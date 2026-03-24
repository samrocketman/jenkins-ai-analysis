# EC2 Plugin: Heavy vs Lightweight Pattern Analysis (Phase 2)

**Date:** February 2026  
**Goal:** Identify all remaining opportunities to split heavy and lightweight patterns so build processing is **as fast as possible**.

---

## Executive Summary

The previous refactor addressed **Queue lock** paths (`EC2RetentionStrategy.check()` and `start()`). This analysis identifies **additional** paths that can block or delay build scheduling, provisioning, and execution. The highest-impact opportunities are:

1. **taskAccepted → MinimumInstanceChecker** – Heavy provisioning runs synchronously when a build is accepted
2. **NoDelayProvisionerStrategy** – Full EC2 provisioning blocks the NodeProvisioner strategy round
3. **EC2Cloud.provision** – Synchronous EC2 API calls (describeInstances, runInstances) block provisioning thread
4. **EC2RetentionStrategy** – `connect()` still runs under `Queue.withLock` in async path
5. **EC2SlaveMonitor** – Per-node EC2 API calls; could batch or stagger

---

## 1. Paths Already Optimized (Queue Lock)

| Path | Status |
|------|--------|
| `EC2RetentionStrategy.check()` | ✅ Returns instantly; heavy work on `HEAVY_WORK_EXECUTOR` |
| `EC2RetentionStrategy.start()` | ✅ Returns instantly; heavy work on `HEAVY_WORK_EXECUTOR` |
| `JobOffer.getCauseOfBlockage` → `isAcceptingTasks` | ✅ Light (default true) |
| `Node.canTake()` for EC2AbstractSlave | ✅ Light (default impl) |

---

## 2. High-Impact: Build Assignment Adjacent

### 2.1 EC2RetentionStrategy.taskAccepted → MinimumInstanceChecker

**Location:** `EC2RetentionStrategy.java:414-432`

**Invocation:** `ExecutorListener.taskAccepted()` when an executor accepts a build from the queue.

**Current behavior:**
- When `maxTotalUses <= 1`, calls `MinimumInstanceChecker.checkForMinimumInstances()` **synchronously**
- `checkForMinimumInstances()` is `synchronized`, scans clouds/templates, calls `Queue.getInstance().getBuildableItems()`, and may call `cloud.provision()` (full EC2 provisioning)

**Impact:** Blocks the thread that accepted the build. With multiple EC2 agents accepting tasks, this can serialize and delay.

**Split recommendation:**
```
LIGHTWEIGHT (sync):  setAcceptingTasks(false), maybe update maxTotalUses
HEAVY (async):       schedule MinimumInstanceChecker.checkForMinimumInstances() on executor
```
- Use `HEAVY_WORK_EXECUTOR` or a dedicated single-thread executor
- Defer `checkForMinimumInstances()` so it does not block `taskAccepted`

---

### 2.2 MinimumInstanceChecker.checkForMinimumInstances

**Location:** `MinimumInstanceChecker.java:84-152`

**Invocation:** `taskAccepted` (above), `EC2SlaveMonitor.execute` (periodic).

**Current behavior:**
- `synchronized` on entire method – serializes all callers
- `Queue.getInstance().getBuildableItems()` – may contend with Queue
- `cloud.provision(template, n)` – full EC2 path (describeInstances, runInstances, etc.)

**Split recommendation:**
- **Decision:** Fast path – check if any template has min requirements; if not, return immediately (already exists)
- **Provision:** Run `cloud.provision()` on **async executor** instead of caller thread
- **Lock:** Narrow `synchronized` scope or use `ReentrantLock` with tryLock; avoid blocking other strategies

---

### 2.3 MinimumInstanceChecker.countQueueItemsForAgentTemplate

**Location:** `MinimumInstanceChecker.java:69-75`

**Current behavior:** `Queue.getInstance().getBuildableItems().stream()...` – iterates Queue

**Note:** Queue access may require lock. If called from async path, wrap in `Queue.withLock` for brief read. Consider cached snapshot with short TTL if available in Jenkins baseline.

---

## 3. High-Impact: NodeProvisioner / Provisioning

### 3.1 NoDelayProvisionerStrategy.apply

**Location:** `NoDelayProvisionerStrategy.java:26-75`

**Invocation:** `NodeProvisioner.Strategy` when evaluating excess workload.

**Current behavior:**
```java
Collection<PlannedNode> plannedNodes = cloud.provision(new Cloud.CloudState(label, 0), numToProvision);
strategyState.recordPendingLaunches(plannedNodes);
```
- `cloud.provision()` runs **synchronously** – blocks until EC2 instances are launched (or fail)
- Includes `getPossibleNewSlavesCount` (describeInstances, describeSpotInstanceRequests), `SlaveTemplate.provision` (runInstances, etc.)

**Impact:** Blocks the NodeProvisioner strategy round. Other strategies wait. Build scheduling can stall.

**Split recommendation:**
- **Option A:** Return `PlannedNode`s that represent **future** capacity – e.g. `createPlannedNode`-style futures that complete when instance is ready. Provisioner records "planned" immediately; actual EC2 work runs in background.
- **Option B:** Defer `cloud.provision()` to executor; strategy returns `CONSULT_REMAINING_STRATEGIES` and a separate task does provisioning. Complexity: coordination with NodeProvisioner expectations.
- **Option C:** Use **cached** instance count with short TTL; avoid `describeInstances` on every strategy round. Refresh cache asynchronously.

---

### 3.2 EC2Cloud.provision(Label, int)

**Location:** `EC2Cloud.java:1026-1085`

**Current behavior:**
- For each template: `getNewOrExistingAvailableSlave(t, number, false)`
- Holds `slaveCountingLock`, calls `getPossibleNewSlavesCount` → `countCurrentEC2Slaves` (paginated EC2 API), `countCurrentEC2SpotSlaves`, then `t.provision()` (runInstances, etc.)

**Split recommendation:**
- **Async cap refresh:** Maintain cached `possibleSlavesCount` per template; refresh on background thread with TTL (e.g. 30–60 s). Provisioning uses cache for fast "can we provision?" check.
- **Lighter provision path:** `runInstances` is inherently slow; consider returning `PlannedNode` that completes when instance ID exists, then background thread handles SSH/WinRM connect.

---

### 3.3 EC2Cloud.getNewOrExistingAvailableSlave

**Location:** `EC2Cloud.java:990-1023`

**Current behavior:**
- `slaveCountingLock.lock()` for entire block
- `getPossibleNewSlavesCount` → EC2 API
- `t.provision(number, provisionOptions)` → EC2 API

**Split:** Same as 3.2 – cache counts; narrow lock scope.

---

### 3.4 EC2Cloud.attachSlavesToJenkins

**Location:** `EC2Cloud.java:1087-1100`

**Current behavior:**
```java
if (slave.getStopOnTerminate() && c != null) {
    c.connect(false);  // SSH/WinRM – synchronous
}
jenkins.addNode(slave);
```

**Split recommendation:**
- Add node first; let `RetentionStrategy.start()` (already async) drive `connect()`.
- Or fire-and-forget `connect()` on executor so caller does not block.

---

## 4. Medium-Impact: Retention Heavy Path

### 4.1 Queue.withLock(() -> computer.connect(false))

**Location:** `EC2RetentionStrategy.java` – `attemptReconnectIfOffline`, `start`

**Current behavior:** `connect()` runs **inside** `Queue.withLock`. Connection can trigger substantial work (SSH, launcher) while holding the Queue lock.

**Split recommendation:**
- Schedule `connect()` on **remoting/launcher pool** or `Computer.threadPoolForRemoting`; use `Queue.withLock` only for tiny state updates (e.g. marking "connecting") if needed.
- Or: ensure `connect()` is invoked from a context that does not hold Queue lock; Jenkins core may already support this.

---

### 4.2 itemsInQueueForThisSlave

**Location:** `EC2RetentionStrategy.java:345-364`

**Current behavior:** Called from `internalCheck` inside `Queue.withLock(() -> itemsInQueueForThisSlave(computer))`. Scans `Jenkins.get().getQueue().getItems()`.

**Note:** Already wrapped in `Queue.withLock` for brief read. Consider coarser cache with short TTL outside lock if correctness allows (careful with race conditions).

---

## 5. Medium-Impact: Periodic Monitors

### 5.1 EC2SlaveMonitor.removeDeadNodes

**Location:** `EC2SlaveMonitor.java:41-62`

**Invocation:** `AsyncPeriodicWork` every 10 min (default).

**Current behavior:** For each EC2 node, `isAlive(true)` → `fetchLiveInstanceData` → `CloudHelper.getInstanceWithRetry` (EC2 API per node).

**Split recommendation:**
- **Batch:** Single `describeInstances` with filters for all EC2 instance IDs; avoid N per-node API calls.
- **Stagger:** Spread checks over the recurrence period to avoid burst.

---

### 5.2 EC2SlaveMonitor.execute → MinimumInstanceChecker

**Location:** `EC2SlaveMonitor.java:37-39`

**Current behavior:** After `removeDeadNodes`, calls `MinimumInstanceChecker.checkForMinimumInstances()`.

**Note:** Already async (periodic work). If `checkForMinimumInstances` is deferred to executor (see 2.2), this path benefits as well.

---

### 5.3 EC2ConnectionUpdater

**Location:** `EC2Cloud.java:1726-1759`

**Invocation:** PeriodicWork every 60s.

**Current behavior:** Filtered `describeInstances` keepalive per cloud.

**Note:** Already periodic. Tune interval via property if API quotas are an issue.

---

### 5.4 EC2CleanupOrphanedNodes

**Location:** `EC2CleanupOrphanedNodes.java:71-106`

**Invocation:** PeriodicWork every 1 hour.

**Current behavior:** Paginated `describeInstances`, `createTags`, `terminateInstances`.

**Note:** Off hot path. Tune recurrence or disable if not needed.

---

## 6. Lower Impact / Acceptable

| Path | Location | Notes |
|------|----------|-------|
| `EC2ComputerListener.onOnline` | EC2ComputerListener | Light; sets flag |
| `EC2ComputerLauncher.launch` | EC2ComputerLauncher | Heavy; expected for remoting |
| `EC2Step.Execution.run` | EC2Step | Heavy; explicit pipeline step |
| `SshHostKeyVerificationStrategy.getHostKeyFromConsole` | EC2Computer | `getState()` → EC2; could cache |
| `EC2Computer.getState` | EC2Computer | Always refreshes; callers avoid on hot path |

---

## 7. Implementation Priority

| Priority | Path | Effort | Impact |
|----------|------|--------|--------|
| **P1** | taskAccepted → defer MinimumInstanceChecker | Low | High – unblocks build acceptance |
| **P2** | MinimumInstanceChecker → async provision | Medium | High – avoids synchronized + heavy provision |
| **P3** | NoDelayProvisionerStrategy → async/cached provision | High | High – unblocks NodeProvisioner |
| **P4** | EC2Cloud.provision → cached instance count | Medium | Medium – faster provisioning |
| **P5** | attachSlavesToJenkins → defer connect | Low | Medium |
| **P6** | EC2RetentionStrategy connect under Queue lock | Medium | Medium – reduces lock hold |
| **P7** | EC2SlaveMonitor batch describeInstances | Medium | Low–medium |

---

## 8. References

- `../leastload-plugin/` – Queue maintenance analysis
- `EC2_QUEUE_AUDIT.md` – EC2 Queue lock audit
- `EC2_QUEUE_PERFORMANCE_ANALYSIS.md` – Phase 1 implementation summary
- `EC2Cloud.java` – Provisioning, countCurrentEC2Slaves
- `MinimumInstanceChecker.java` – checkForMinimumInstances
- `NoDelayProvisionerStrategy.java` – NodeProvisioner strategy
