# EC2 Plugin — Queue Performance Patches

Patches against `origin/master` (`jenkinsci/ec2-plugin`). 8 files changed, +470 / -185 lines.

---

## How This Helps Unblock the Queue

Jenkins core calls `RetentionStrategy.check()` and `RetentionStrategy.start()` under the Queue lock for every computer during `Queue.maintain()`. Before these patches, the EC2 plugin made live AWS API calls (describeInstances, getState, getUptime, connect) directly inside these methods. With hundreds of EC2 agents, each `maintain()` cycle spent tens of seconds waiting on AWS responses while holding the lock — blocking all scheduling, cancellation, and queue inspection across the entire Jenkins instance.

The patch splits every EC2 operation into two categories:

- **Lightweight operations** are called by Jenkins core under the Queue lock. They return immediately with no AWS interaction — they check a timestamp or a flag, schedule background work, and return.
- **Heavyweight operations** (AWS API calls, SSH connections, idle timeout logic) run on dedicated background thread pools outside the Queue lock.

This means the Queue lock is held for microseconds per EC2 agent instead of seconds. Combined with the [leastload plugin](../leastload-plugin/) fixes (which ensure the load balancer assigns work without unnecessary deferrals) and the [job-restrictions plugin](../job-restrictions-plugin/) cache (which avoids repeated restriction evaluation), the entire `Queue.maintain()` cycle completes in milliseconds.

---

## Files Changed

### EC2RetentionStrategy.java — Lightweight Retention Checks

The core of the patch. `check()` and `start()` now return immediately.

- **`check()`** — Uses `tryLock()` and a `nextCheckAfter` timestamp to avoid re-scheduling. Submits heavy work to `HEAVY_WORK_EXECUTOR` and returns `CHECK_INTERVAL_MINUTES` instantly. No AWS calls touch the Queue lock path.
- **`runHeavyCheck()`** — New method that runs outside the Queue lock. Performs EC2 API calls (`getState`, `getUptime`), idle timeout evaluation, reconnect attempts. State-changing operations (`disconnect`, `launchTimeout`, `idleTimeout`) use brief `Queue.withLock()` sections only for the final state mutation.
- **`start()`** — Defers `getState()` and `connect()` to `HEAVY_WORK_EXECUTOR`. The caller (Queue lock holder) returns immediately.
- **`taskAccepted()`** — Calls `MinimumInstanceChecker.scheduleCheck()` instead of the synchronous `checkForMinimumInstances()` which could trigger full EC2 provisioning on the caller's thread.
- **`HEAVY_WORK_EXECUTOR`** — New static cached daemon thread pool for all deferred retention work.

### EC2Cloud.java — Async Provisioning and Instance Count Cache

Provisioning no longer blocks the NodeProvisioner strategy thread.

- **`provision()`** — `runInstances` is deferred to `PROVISIONING_EXECUTOR` via `CompletableFuture.supplyAsync()`. Returns `PlannedNode` futures immediately so the NodeProvisioner can record planned capacity and move on. On completion, invalidates the instance count cache and triggers `scheduleQueueMaintenance()`.
- **`waitForRunningAndConnectAsync()`** — New method. Polls instance state on `Computer.threadPoolForRemoting` until RUNNING, then calls `c.connect(false)` and `scheduleQueueMaintenance()`. Runs entirely outside the Queue lock.
- **`getPossibleNewSlavesCount()`** — Caches `countCurrentEC2Slaves()` results with a 30-second TTL. Previously, every provision attempt called `describeInstances` and `describeSpotInstanceRequests` — now it reads from cache when fresh.
- **`scheduleQueueMaintenance()`** — New static helper. Uses `jenkins.util.Timer` to call `scheduleMaintenance()` after a configurable delay (default 1000ms) for early provisioning events. The `onOnline` trigger fires immediately with no delay.
- **`doProvision()` / `attachSlavesToJenkins()`** — `c.connect(false)` deferred to `PROVISIONING_EXECUTOR` instead of blocking the HTTP request or provisioning thread.

### EC2ComputerListener.java — Immediate Queue Notification

- **`onOnline()`** — Calls `scheduleMaintenance()` immediately (no delay) when an EC2 agent is fully online. This is the definitive "ready for work" signal — the agent's channel is established and all `ComputerListener`s have run. Without this, the Queue would wait up to 5 seconds before noticing the new executor.

### EC2Computer.java — Instance State Cache

- **`describeInstance()`** — Returns cached instance data when within TTL (default 30s). Previously, `getState()` always made a fresh API call; now it delegates to `describeInstance()` which uses the cache. This matters during SSH verification retries where `getState()` was called repeatedly.

### EC2SlaveMonitor.java — Batch Dead-Node Detection

- **`removeDeadNodes()`** — Groups EC2 slaves by cloud and calls `CloudHelper.getInstancesBatch()` once per cloud (single `describeInstances` API call for up to 100 instances). Previously made one API call per node. Falls back to per-node `isAlive(true)` if the batch call fails.

### CloudHelper.java — Batch API

- **`getInstancesBatch()`** — New method. Single `describeInstances` call for a list of instance IDs, chunked at 100. Returns `Map<String, Instance>`.

### EC2AbstractSlave.java — Batch Update Support

- **`updateFromFetchedInstance()`** — New method. Updates instance data from a pre-fetched `Instance` object so batch operations can update slave state without per-node API calls.

### MinimumInstanceChecker.java — Async Scheduling

- **`scheduleCheck()`** — New method. Submits `checkForMinimumInstances()` to a single-thread executor. Callers (`taskAccepted`, `EC2SlaveMonitor`) return immediately instead of blocking on EC2 provisioning.

---

## Configuration

| System Property | Default | Description |
|-----------------|---------|-------------|
| `jenkins.ec2.instanceCountCacheTtlMs` | 30000 | Instance count cache TTL (ms) |
| `jenkins.ec2.scheduleMaintenanceDelayMs` | 1000 | Delay before queue maintenance on early provisioning events (ms) |
| `jenkins.ec2.checkIntervalMinutes` | 1 | Minutes between retention checks per computer |
| `hudson.plugins.ec2.EC2Computer.instanceCacheTTLMs` | 30000 | SSH verification state cache TTL (ms) |
| `jenkins.ec2.checkAlivePeriod` | 600000 | EC2SlaveMonitor recurrence (ms) |

---

## Background Analysis

- [EC2_IMPLEMENTATION_STATUS.md](background/EC2_IMPLEMENTATION_STATUS.md) — Implementation status for all optimization phases
- [EC2_QUEUE_AUDIT.md](background/EC2_QUEUE_AUDIT.md) — Audit of all EC2 code paths under Queue lock
- [EC2_PLUGIN_HEAVY_LIGHTWEIGHT_ANALYSIS.md](background/EC2_PLUGIN_HEAVY_LIGHTWEIGHT_ANALYSIS.md) — Phase 2 heavy vs lightweight pattern analysis
- [EC2_QUEUE_PERFORMANCE_ANALYSIS.md](background/EC2_QUEUE_PERFORMANCE_ANALYSIS.md) — Phase 1 retention strategy refactor
