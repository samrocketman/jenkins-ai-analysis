# EC2 Plugin: Queue Performance Implementation Status

**Date:** February–March 2026  
**Goal:** Make build processing as fast as possible by splitting heavy and lightweight patterns.

---

## 1. Phase 1: Queue Lock Paths (Completed)

| Path | Before | After |
|------|--------|-------|
| `EC2RetentionStrategy.check()` | EC2 API calls under Queue lock | Returns instantly; heavy work on `HEAVY_WORK_EXECUTOR` |
| `EC2RetentionStrategy.start()` | `connect()` under Queue lock | Returns instantly; `connect()` on executor, no `Queue.withLock` |
| `connect()` in retention path | Wrapped in `Queue.withLock` | Called directly in async task |

**Files modified:** `EC2RetentionStrategy.java`

---

## 2. Phase 2: High-Impact Fixes (Completed)

### 2.1 taskAccepted → MinimumInstanceChecker

| Before | After |
|--------|-------|
| `checkForMinimumInstances()` called synchronously when `maxTotalUses` drains | `MinimumInstanceChecker.scheduleCheck()` – returns immediately |

**Files modified:** `EC2RetentionStrategy.java`

### 2.2 MinimumInstanceChecker Scheduling

| Before | After |
|--------|-------|
| `checkForMinimumInstances()` blocks caller (taskAccepted, EC2SlaveMonitor) | `scheduleCheck()` submits to single-thread executor; callers return immediately |

**Files modified:** `MinimumInstanceChecker.java`, `EC2SlaveMonitor.java`

### 2.3 EC2Cloud Instance Count Cache

| Before | After |
|--------|-------|
| `getPossibleNewSlavesCount()` calls `countCurrentEC2Slaves()` (EC2 API) on every provision | Cached with 30s TTL; configurable via `jenkins.ec2.instanceCountCacheTtlMs` |

**Files modified:** `EC2Cloud.java`

### 2.4 attachSlavesToJenkins connect

| Before | After |
|--------|-------|
| `c.connect(false)` blocks provisioning thread for stopOnTerminate nodes | Connect scheduled on `PROVISIONING_EXECUTOR`; addNode first, connect async |

**Files modified:** `EC2Cloud.java`

---

## 3. Phase 3: Aggressive Queue Maintenance (Completed)

### 3.1 Problem

Jenkins core's `Queue.maintain()` runs on a 5-second timer (`Queue.maintainInterval`). After an EC2 agent finishes provisioning and comes online, there is up to a 5-second delay before the queue notices the new executor and dispatches work. Under heavy load, `maintain()` can take longer than 5 seconds, extending the gap further since the timer uses `scheduleWithFixedDelay`.

### 3.2 Solution

Call `scheduleMaintenance()` at three key moments during EC2 provisioning:

| Trigger | Location | Delay | When it fires |
|---------|----------|-------|---------------|
| **Provision future completes** | `EC2Cloud.provision()` `whenComplete` callback | 1s (configurable) | `runInstances` returns from AWS; node not ready yet |
| **Instance reaches RUNNING** | `EC2Cloud.waitForRunningAndConnectAsync()` | 1s (configurable) | Instance is RUNNING, `c.connect(false)` initiated; agent not yet online |
| **Agent fully online** | `EC2ComputerListener.onOnline()` | **Immediate** | `SlaveComputer` channel established, agent accepting tasks |

The first two triggers fire before the node can actually accept work, so they use a 1-second delay (configurable via `jenkins.ec2.scheduleMaintenanceDelayMs`) to let in-progress builds complete their state transitions. The delay is implemented via `jenkins.util.Timer.get().schedule()` — non-blocking.

The `onOnline` trigger fires **immediately** because the agent is definitively ready to take work at that point — there is no reason to wait.

`scheduleMaintenance()` submits to `AtmostOneTaskExecutor` — duplicate calls coalesce, so multiple triggers in quick succession do not cause multiple `maintain()` runs.

**Files modified:** `EC2Cloud.java`, `EC2ComputerListener.java`

### 3.3 Why three triggers?

- **Provision complete** — Other items already in the queue may be dispatchable to existing executors; the provisioning event is a signal that capacity has changed.
- **Instance RUNNING** — Earliest moment the agent could accept work. Connection is in progress but not yet complete.
- **onOnline** — Definitive "ready" signal. The agent's channel is up, all `ComputerListener`s have run, and the node is accepting tasks. `SlaveComputer._setChannel()` already calls `scheduleMaintenance()` internally, but `onOnline` fires *after* all listeners complete, making it a more reliable "everything ready" signal.

---

## 4. Configuration

| Property | Default | Description |
|----------|---------|-------------|
| `jenkins.ec2.checkIntervalMinutes` | 1 | Minutes between retention checks per computer |
| `jenkins.ec2.instanceCountCacheTtlMs` | 30000 | Instance count cache TTL (ms) |
| `jenkins.ec2.scheduleMaintenanceDelayMs` | 1000 | Delay before triggering queue maintenance after provisioning events (ms) |
| `jenkins.ec2.checkAlivePeriod` | 600000 | EC2SlaveMonitor recurrence (ms, 10 min) |
| `EC2RetentionStrategy.disabled` | false | Disable retention strategy |
| `hudson.plugins.ec2.EC2Computer.instanceCacheTTLMs` | 30000 | TTL for getState/getUptime cache during SSH verification (ms) |

---

## 5. Additional Review: Other Areas That Could Slow Builds

### 5.1 Already Lightweight (No Change Needed)

| Path | Location | Notes |
|------|----------|-------|
| `EC2Cloud.canProvision(Label)` | EC2Cloud.java:1289 | Iterates templates only; no EC2 API |
| `EC2Cloud.getTemplates(Label)` | EC2Cloud.java:639 | In-memory filter; no EC2 |
| `EC2RetentionStrategy.postJobAction` | EC2RetentionStrategy.java | Runs after build completes; `terminate()` is EC2 but not on build-start path |

### 5.2 Provisioning Path (NodeProvisioner – Not Under Queue Lock)

| Path | Location | Before | After |
|------|----------|--------|-------|
| **NoDelayProvisionerStrategy.apply** | NoDelayProvisionerStrategy.java:57 | `cloud.provision()` blocks strategy thread | `provision()` returns immediately; `runInstances` deferred to `PROVISIONING_EXECUTOR` |
| **EC2Cloud.provision** | EC2Cloud.java | `getNewOrExistingAvailableSlave` blocks on runInstances | Schedules provision on `PROVISIONING_EXECUTOR`; returns PlannedNodes with futures that complete when instances are RUNNING and connected |
| **EC2Cloud.doProvision** | EC2Cloud.java:716 | `c.connect(false)` blocks HTTP request | Defer connect to `PROVISIONING_EXECUTOR` (same as attachSlavesToJenkins) |

### 5.3 SSH / Launcher Path (Runs on Remoting Thread – Not Queue)

| Path | Location | Before | After |
|------|----------|--------|-------|
| **SshHostKeyVerificationStrategy.getHostKeyFromConsole** | SshHostKeyVerificationStrategy.java:101 | `computer.getState()` → EC2 API every call | `EC2Computer.getState()` uses 30s TTL cache (configurable via `EC2Computer.instanceCacheTTLMs`) |
| **CheckNewHardStrategy** | CheckNewHardStrategy.java:83 | `computer.getUptime()` → uses describeInstance | Same cache; `getUptime()` uses `describeInstance()` which shares the TTL |
| **EC2Computer.getDecodedConsoleOutput** | EC2Computer.java:131 | `getConsoleOutput` → EC2 API | Called during host key verification; could cache console output for retries |
| **EC2Computer.checkIfNitro** | EC2Computer.java:152 | `describeInstanceTypes` → EC2 API | Cached in `isNitro`; first call is heavy |

### 5.4 Periodic / Background (Not on Build-Start Path)

| Path | Location | Impact |
|------|----------|--------|
| **EC2SlaveMonitor.removeDeadNodes** | EC2SlaveMonitor.java:46 | `isAlive(true)` → `CloudHelper.getInstanceWithRetry` per node | Batch `describeInstances` for all instance IDs; stagger checks |
| **EC2AbstractSlave.fetchLiveInstanceData** | EC2AbstractSlave.java:941 | EC2 API per node | Used by isAlive; batching would help |
| **EC2ConnectionUpdater** | EC2Cloud.java | describeInstances every 60s | Off hot path |
| **EC2Cloud.getKeyPair** | EC2Cloud.java:664 | `connect().find(keyPair)` – EC2 API | Used during node creation; not on scheduling path |

### 5.5 MinimumInstanceChecker.countQueueItemsForAgentTemplate

| Path | Location | Notes |
|------|----------|-------|
| **countQueueItemsForAgentTemplate** | MinimumInstanceChecker.java:92 | `Queue.getInstance().getBuildableItems().stream()` | Runs on MinimumInstanceChecker executor (async). Queue.getBuildableItems() reads from snapshot; no explicit lock. If correctness requires lock, wrap in brief `Queue.withLock`. |

---

## 6. Remaining Opportunities (Prioritized) – All Implemented

| Priority | Path | Recommendation | Effort | Status |
|----------|------|----------------|--------|--------|
| **P1** | EC2Cloud.doProvision connect | Defer `c.connect(false)` to PROVISIONING_EXECUTOR | Low | Done |
| **P2** | EC2SlaveMonitor.removeDeadNodes | Batch `describeInstances` for all EC2 instance IDs | Medium | Done |
| **P3** | SshHostKeyVerificationStrategy / CheckNewHardStrategy | Cache getState/getUptime during verification | Low | Done |
| **P4** | NoDelayProvisionerStrategy | runInstances still blocks; would need PlannedNode refactor | High | Done |
| **P5** | EC2Cloud.provision runInstances | Defer to background; return PlannedNode with future | High | Done |
| **P6** | Aggressive scheduleMaintenance | Call scheduleMaintenance on provision complete, RUNNING, and onOnline | Low | Done |

---

## 7. References

- `EC2_QUEUE_AUDIT.md` – Queue lock paths and implementation details
- `EC2_PLUGIN_HEAVY_LIGHTWEIGHT_ANALYSIS.md` – Full heavy vs lightweight analysis
- `EC2_QUEUE_PERFORMANCE_ANALYSIS.md` – Phase 1 summary
