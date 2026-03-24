# EC2 Plugin Queue Code Path Audit

Based on the performance audit in `leastload-plugin/docs`, this document identifies all EC2 plugin code paths invoked from or under `hudson.model.Queue` lock, and the split between lightweight (instant-return) and heavyweight (deferred) operations.

---

## 1. Queue Lock Context

- **`Queue.maintain()`** holds the Queue lock for the entire loop over buildables.
- **`ComputerRetentionWork.doAperiodicRun()`** calls `Queue.withLock(() -> c.getRetentionStrategy().check(c))` for each computer—so **`RetentionStrategy.check()` runs under the Queue lock**.
- **`RetentionStrategy.start()`** calls `Queue.withLock(() -> check(c))`—same lock.
- Any slow work in these paths blocks the entire Queue and delays scheduling.

---

## 2. EC2 Plugin Code Paths Under Queue Lock

### 2.1 EC2RetentionStrategy.check() [IMPLEMENTED: LIGHTWEIGHT]

**Invoked from:** `ComputerRetentionWork` (under Queue lock).

**Implemented behavior:** Returns immediately with `CHECK_INTERVAL_MINUTES`. Uses `tryLock()` and `nextCheckAfter` to avoid rescheduling. Schedules heavy work on `HEAVY_WORK_EXECUTOR` and returns. **No EC2 API calls under the lock.**

Heavy work (getState, getUptime, idle timeout, reconnect) runs in `runHeavyCheck()` on a background thread. State-changing operations (disconnect, idleTimeout, launchTimeout) use brief `Queue.withLock` sections.

### 2.2 EC2RetentionStrategy.start() [IMPLEMENTED: LIGHTWEIGHT]

**Invoked from:** When a new EC2Computer is introduced (e.g. agent added). Runs under Queue lock via `RetentionStrategy.start()`.

**Implemented behavior:** Returns immediately. Schedules `getState()` and `connect()` on `HEAVY_WORK_EXECUTOR`. **`connect()` runs outside the Queue lock** (no `Queue.withLock` wrapper) so SSH/connection work does not block the queue.

### 2.3 JobOffer.getCauseOfBlockage() → EC2Computer.isAcceptingTasks() [LIGHT]

**Invoked from:** `Queue.maintain()` when building candidates for each buildable.

**Current behavior:** EC2RetentionStrategy does **not** override `isAcceptingTasks()`, so it returns `true` (default). **No EC2 API calls** on this path. Acceptable.

### 2.4 Node.canTake() for EC2AbstractSlave [LIGHT]

**Invoked from:** `JobOffer.getCauseOfBlockage()` → `node.canTake(item)`.

**Current behavior:** EC2AbstractSlave does not override `canTake()`. Uses default `Node.canTake()` (label, permissions, node properties). **No EC2-specific logic.** Acceptable.

---

## 3. Additional High-Impact Paths (Build Assignment Adjacent)

### 3.1 EC2RetentionStrategy.taskAccepted → MinimumInstanceChecker [IMPLEMENTED]

**Invocation:** `ExecutorListener.taskAccepted()` when an executor accepts a build.

**Implemented behavior:** Calls `MinimumInstanceChecker.scheduleCheck()` instead of `checkForMinimumInstances()` directly. Heavy provisioning runs on `MinimumInstanceChecker.EXECUTOR`; caller returns immediately.

### 3.2 EC2SlaveMonitor → MinimumInstanceChecker [IMPLEMENTED]

**Invocation:** `AsyncPeriodicWork.execute()` every 10 min (default).

**Implemented behavior:** Calls `MinimumInstanceChecker.scheduleCheck()` instead of `checkForMinimumInstances()` directly. Monitor returns immediately; checker runs asynchronously.

### 3.3 EC2Cloud Instance Count Cache [IMPLEMENTED]

**Location:** `EC2Cloud.getPossibleNewSlavesCount()`

**Implemented behavior:** Caches `countCurrentEC2Slaves` results with 30s TTL (configurable via `jenkins.ec2.instanceCountCacheTtlMs`). Avoids repeated EC2 `describeInstances` / `describeSpotInstanceRequests` during provisioning. Cache invalidated after successful provision.

### 3.4 attachSlavesToJenkins connect [IMPLEMENTED]

**Location:** `EC2Cloud.attachSlavesToJenkins()`

**Implemented behavior:** For `stopOnTerminate` nodes, `c.connect(false)` runs on `PROVISIONING_EXECUTOR` instead of blocking the provisioning thread. Node is added first; connect happens asynchronously.

### 3.5 Aggressive Queue Maintenance on Provisioning Events [IMPLEMENTED]

**Problem:** Jenkins core's `Queue.maintain()` runs on a 5-second timer. After an EC2 agent comes online, there is up to 5 seconds (or longer under load) before the queue notices the new executor.

**Solution:** `scheduleMaintenance()` is called at three points during provisioning:

| Trigger | Location | When |
|---------|----------|------|
| `provisionFuture.whenComplete` | `EC2Cloud.provision()` | `runInstances` returns from AWS |
| Instance RUNNING | `EC2Cloud.waitForRunningAndConnectAsync()` | Instance state is RUNNING, connect initiated |
| Agent online | `EC2ComputerListener.onOnline()` | SlaveComputer channel established, agent accepting tasks |

`scheduleMaintenance()` coalesces via `AtmostOneTaskExecutor` — multiple calls don't cause multiple `maintain()` runs, they just ensure one happens promptly.

**Files modified:** `EC2Cloud.java`, `EC2ComputerListener.java`

---

## 4. Design: Lightweight vs Heavyweight

| Path | Type | Status |
|------|------|--------|
| `RetentionStrategy.check()` | **Lightweight** | ✅ Returns instantly |
| `RetentionStrategy.start()` | **Lightweight** | ✅ Returns instantly |
| `RetentionStrategy.isAcceptingTasks()` | **Lightweight** | ✅ Default true |
| `taskAccepted` → MinimumInstanceChecker | **Lightweight** | ✅ Schedules async |
| EC2SlaveMonitor → MinimumInstanceChecker | **Lightweight** | ✅ Schedules async |
| Provisioning → scheduleMaintenance | **Lightweight** | ✅ Non-blocking submit |
| Idle timeout, reconnect, state refresh | **Heavyweight** | Runs outside Queue lock |
| EC2 API (getState, describeInstances) | **Heavyweight** | Runs on background executors |
| `connect()` | **Heavyweight** | Runs outside Queue lock |

---

## 5. Implementation Summary

1. **EC2RetentionStrategy.check()** – Returns immediately; heavy work on `HEAVY_WORK_EXECUTOR`.
2. **EC2RetentionStrategy.start()** – Returns immediately; `connect()` on `HEAVY_WORK_EXECUTOR` without `Queue.withLock`.
3. **EC2RetentionStrategy.connect()** – No longer wrapped in `Queue.withLock`; runs directly in async task.
4. **taskAccepted** – Uses `MinimumInstanceChecker.scheduleCheck()`.
5. **MinimumInstanceChecker** – `scheduleCheck()` submits to single-thread executor; callers return immediately.
6. **EC2Cloud** – Instance count cache with TTL; `attachSlavesToJenkins` defers connect to executor.
7. **EC2Cloud / EC2ComputerListener** – `scheduleMaintenance()` called on provision complete, instance RUNNING, and agent online.

---

## 6. References

- `../leastload-plugin/` – Queue maintenance and delays
- `EC2_IMPLEMENTATION_STATUS.md` – Full implementation status
- `jenkins/core/.../ComputerRetentionWork.java` – Retention check under Queue lock
- `jenkins/core/.../RetentionStrategy.java` – `@GuardedBy("hudson.model.Queue.lock")` on `check()`
