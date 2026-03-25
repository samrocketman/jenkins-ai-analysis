# Ludicrous Mode: Jenkins Queue Performance

A coordinated patch set across Jenkins plugins that transforms Queue maintenance from a 30+ second blocking operation into a millisecond-fast non-blocking pipeline.

---

## Problem

Jenkins core runs a single-threaded `Queue.maintain()` loop under a global `ReentrantLock`. Every 5 seconds, this loop iterates over all buildable items, checks every candidate executor, and assigns work. Nothing else in Jenkins can schedule, cancel, or inspect the queue while this lock is held.

The real damage came from plugins hooking into this maintenance pipeline. The EC2 plugin made live AWS API calls (describeInstances, runInstances) while holding the Queue lock. The job-restrictions plugin evaluated regex and class-name restrictions on every (node, item) pair — and after XStream deserialization, silently threw NullPointerExceptions that Jenkins core interpreted as "block this item." The leastload plugin's round-robin logic returned null for label-restricted work when a single node had already been used, deferring the task to the next 5-second cycle.

Together, these plugins turned each `Queue.maintain()` cycle into a 30–35 second ordeal:

```mermaid
%%{init: {'theme':'base', 'themeVariables': {
  'pie1':'#b3e5fc',
  'pie2':'#c8e6c9',
  'pie3':'#fff9c4',
  'pie4':'#ffccbc',
  'pie5':'#d1c4e9',
  'pieSectionColors':'#b3e5fc,#fff9c4,#c8e6c9',
  'pieStrokeColor':'#e0e0e0',
  'pieOuterStrokeWidth':'2',
  'pieLegendTextColor':'#f5f5f5',
  'pieTitleTextColor':'#f5f5f5'
}}}%%
pie showData
    title Total 35 seconds (values in ms)
    "Queue maintenance (other)" : 29990
    "leastload selects agents for queued work" : 10
    "Executors take builds from queue" : 5000
```

The leastload plugin itself took only ~10ms to select agents. The overwhelming majority of time was spent in plugin code making blocking API calls and evaluating restrictions under the Queue lock. The 5-second "Executors take builds from queue" slice represents the time executors waited for the lock to release before they could start their assigned work.

Because `Queue.maintain()` uses `scheduleWithFixedDelay`, the next cycle didn't start until 5 seconds *after* the previous 30-second cycle finished — meaning builds could wait 35+ seconds between scheduling attempts. With hundreds of buildable items and hundreds of agents, each cycle got slower, and the gap between cycles grew. Once Jenkins surpassed ~800 agents, Queue maintenance took so long that Jenkins itself became unresponsive — the UI hung, API calls timed out, and the system entered a death spiral where provisioning more agents made everything worse.

---

## Solution

The three plugin patches work together to ensure that everything called under the Queue lock returns in milliseconds. No AWS API calls, no expensive restriction evaluations, no blocking I/O — just fast in-memory lookups and instant returns.

### EC2 Plugin: Lightweight Queue Path, Heavyweight Background Work

The EC2 plugin's retention strategy (`check()`, `start()`) was called under the Queue lock and made live EC2 API calls for every agent. The patch splits all EC2 operations into two categories:

- **Lightweight** (runs under Queue lock): Returns immediately with no AWS interaction. Schedules heavy work on background thread pools and returns.
- **Heavyweight** (runs outside Queue lock): EC2 API calls (describeInstances, getState, connect), idle timeout, reconnect — all run on dedicated `HEAVY_WORK_EXECUTOR` and `PROVISIONING_EXECUTOR` thread pools.

Provisioning now returns `PlannedNode` futures immediately instead of blocking on `runInstances`. Instance counts are cached with a 30-second TTL to avoid repeated `describeInstances` calls. Dead-node detection uses batch `describeInstances` (one API call per cloud) instead of per-node calls. When agents come online, `scheduleMaintenance()` is called to trigger an immediate Queue re-evaluation rather than waiting for the next 5-second timer tick.

See [ec2-plugin/README.md](ec2-plugin/) for the full patch breakdown.

### Leastload Plugin: Pipeline Support and Immediate Label Assignment

The leastload plugin's load balancer is called for every buildable item during `Queue.maintain()`. Two bugs caused it to return null (deferring work to the next cycle) far more often than necessary:

1. **Pipeline tasks were ignored.** The plugin checked `subTask instanceof Job`, but pipeline work uses `PlaceholderTask` with a `Run` owner. All pipeline `node('label')` blocks fell back to the default consistent-hash balancer, bypassing least-load distribution entirely.

2. **Label-restricted work was deferred.** When a single node had the required label and was marked "used" by the round-robin tracker, the plugin returned null even when that node had idle executors. This caused 10–60+ second delays for label-restricted work.

The patch fixes both: pipeline tasks are resolved to their parent `Job` via `Run.getParent()`, and label-restricted work is assigned immediately when idle executors exist — even if the node was already used this round. Per-label round-robin tracking ensures high-demand labels aren't starved by a global round shared with other labels. Load prediction is disabled (returns empty) to skip the O(computers × predictors × FutureLoads) overhead per buildable item.

See [leastload-plugin/README.md](leastload-plugin/) for the full patch breakdown.

### Job-Restrictions Plugin: Cached Restriction Evaluation

The job-restrictions plugin evaluates regex patterns and class-name checks during `Node.canTake()`, which is called for every (node, item) pair during Queue maintenance. With hundreds of items and hundreds of nodes, this runs tens of thousands of times per cycle.

The patch adds a TTL cache so that repeated checks for the same (node, item) pair return instantly from the cache instead of re-evaluating restrictions. Only "allow" results are cached (never "block") to avoid stale blockages. The cache key includes both the item ID and the task's full display name to prevent collisions when pipeline flyweight tasks reuse queue item IDs.

Critically, the cache's `ConcurrentHashMap` is declared `transient volatile` with lazy double-checked-locking initialization. This fixes a silent bug where XStream deserialization (which bypasses constructors and field initializers) left the map `null`. The resulting `NullPointerException` was caught by Jenkins core's `Node.canTake()` and silently treated as "block this item" — permanently preventing all items from being dispatched to any node with job restrictions configured. This single bug was the root cause of the "Waiting for next available executor" hang that started this investigation.

See [job-restrictions-plugin/README.md](job-restrictions-plugin/) for the full patch breakdown.

### Azure VM Agents Plugin: scheduleMaintenance() Triggers and Async Provisioning

The Azure VM agents plugin provisions Jenkins agents on Azure VMs. Unlike the EC2 plugin, its retention strategies already defer Azure API calls to background thread pools — the Queue lock path is lightweight. However, the plugin never calls `scheduleMaintenance()`, meaning the Queue waits up to 5 seconds before noticing new or removed agents. Additionally, `Cloud.provision()` makes synchronous Azure API calls on the NodeProvisioner thread: template verification lists all VMs in the resource group, and node reuse checks call `getByResourceGroup()` per offline Azure VM.

The proposed patches add:

1. **Comprehensive scheduleMaintenance() triggers** at 10 capacity-change events: agent online, VM reuse complete, new VM provisioned, SSH reconnect, deployment submitted, pool provisioning, agent deprovisioned, agent shutdown, dead node cleaned, excess VMs deleted. Immediate triggers (0ms) for "agent ready" events, delayed triggers (~1s) for "capacity planned/removed" events.

2. **Async template verification** — `provision()` submits verification to a background thread instead of blocking the provisioner. The next NodeProvisioner cycle finds the template verified.

3. **Optimistic node reuse** — `virtualMachineExists()` moves inside the PlannedNode future, eliminating N sequential Azure API calls from the provisioner thread.

4. **VM existence caching** — Caffeine TTL cache (30s default) for `virtualMachineExists()` results, cutting redundant API calls during cleanup and reuse.

See [azure-vm-agents-plugin/README.md](azure-vm-agents-plugin/) for the full analysis and patch plan.

### Lockable Resources Plugin: scheduleMaintenance() Triggers, Contention Reduction, and Script Caching

The lockable-resources-plugin hooks into the Queue via a `QueueTaskDispatcher` — its `canRun(Queue.Item)` method is called **under the Queue lock** for every item on every `Queue.maintain()` cycle. Unlike cloud plugins that use `RetentionStrategy` or `Cloud`, a `QueueTaskDispatcher` sits in the absolute innermost loop of Queue maintenance.

Three issues were found:

1. **No `scheduleMaintenance()` triggers anywhere.** When resources are freed (build completes, pipeline lock body finishes, user unreserves via UI), items waiting in the Jenkins Queue for those resources wait up to 5 seconds for the next timer tick. The proposed patch adds immediate `scheduleMaintenance()` calls at 6 resource-freeing events: `unlockResources()`, `unreserve()`, `reset()`, `LockRunListener.onCompleted()`, `LockRunListener.onDeleted()`, and `FreeDeadJobs.freePostMortemResources()`.

2. **`syncResources` lock contention under Queue lock.** The plugin uses a `synchronized(syncResources)` monitor in `tryQueue()` that is also held by threads doing disk I/O (`save()`). When `onCompleted()` holds `syncResources` during a config write, `canRun()` under the Queue lock blocks waiting — transitively extending Queue lock hold time by disk write latency. The proposed fix makes `save()` asynchronous and narrows the `syncResources` scope.

3. **Groovy script evaluation under Queue lock.** When jobs use `resourceMatchScript`, `canRun()` evaluates a Groovy script for every lockable resource on every cache miss. The existing `cachedCandidates` Guava cache mitigates repeated evaluations, but cache misses (first call, or after resource state change) are heavyweight. The proposed fix adds a per-resource script result TTL cache.

See [lockable-resources-plugin/README.md](lockable-resources-plugin/) for the full analysis and patch plan.

### SSH Agents Plugin: Immediate Queue Maintenance on Agent State Changes

The SSH agents plugin (`ssh-slaves`) is the `ComputerLauncher` implementation used to launch Jenkins agents over SSH. Unlike cloud plugins, it has **zero code paths under the Queue lock** — all heavy SSH work (connection, authentication, SFTP/SCP file transfer, remote process launch, disconnect cleanup) runs on `Computer.threadPoolForRemoting` or a per-launcher `ExecutorService`.

The plugin's Queue performance gap is the absence of `scheduleMaintenance()` calls. When an SSH agent comes online or goes offline, the Queue should re-evaluate immediately. Now that Queue maintenance completes in milliseconds, triggering it on every capacity-changing event is essentially free.

The patch adds a `ComputerListener` that calls `scheduleMaintenance()` on four SSH agent lifecycle events: `onOnline` (agent ready — fires after all listeners complete, making it a more reliable signal than core's `_setChannel()` call alone), `onOffline` (capacity lost — triggers replacement provisioning), `onTemporarilyOnline` (capacity restored), and `onTemporarilyOffline` (capacity temporarily unavailable).

See [ssh-agents-plugin/README.md](ssh-agents-plugin/) for the full audit and patch details.

---

## Results

With all three patches applied:

- **Queue maintenance completes in milliseconds** instead of 30+ seconds. All plugin code under the Queue lock performs only in-memory lookups and instant returns.
- **Jenkins remains responsive** at scale. The UI, API, and build scheduling all function normally because the Queue lock is held for trivial durations.
- **No performance cliff at 800+ agents.** Previously, Jenkins became unresponsive once agent count exceeded ~800 because each Queue maintenance cycle grew linearly with agent count (due to per-agent API calls under the lock). With the patches, agent count has minimal impact on Queue lock hold time.
- **Real-world pressure testing** was performed with over 1,000 AWS Jenkins agents with Jenkins remaining responsive and Queue maintenance staying fast.
- **Hypothetical scaling** into tens of thousands of agents is now feasible, since the Queue maintenance path no longer makes any network calls or expensive computations under the lock.

---

## Plugin Patch Details

Each plugin subfolder has a README summarizing the actual Java patches (diffed against `origin/master`) with references to background analysis:

- **[ec2-plugin/](ec2-plugin/)** — 8 files, +470/-185 lines. Async retention, async provisioning, instance count cache, batch describeInstances, queue maintenance triggers.
- **[azure-vm-agents-plugin/](azure-vm-agents-plugin/)** — ~10 files proposed. Comprehensive scheduleMaintenance() triggers (10 events), async template verification, optimistic node reuse, VM existence cache.
- **[lockable-resources-plugin/](lockable-resources-plugin/)** — ~3–5 files proposed. scheduleMaintenance() triggers (6 events), async save to reduce syncResources contention, Groovy script result caching, label expression caching. QueueTaskDispatcher runs in innermost Queue lock loop.
- **[ssh-agents-plugin/](ssh-agents-plugin/)** — 1 new file, ~40 lines. ComputerListener for scheduleMaintenance() triggers on SSH agent online/offline/temporarily-online/temporarily-offline events. All existing code paths already run outside Queue lock — no changes needed.
- **[leastload-plugin/](leastload-plugin/)** — 1 file, +203/-38 lines. Pipeline support, per-label round-robin, single-node immediate assign, load prediction skip.
- **[job-restrictions-plugin/](job-restrictions-plugin/)** — 7 files, +141/-47 lines. canTake TTL cache, transient volatile fix for XStream deserialization.
- **[jenkins/](jenkins/)** — Jenkins core Queue analysis (no code changes). Queue lock contention, MappingWorksheet performance, queue time breakdown pie chart.

---

## Configuration

All optimizations are configurable via system properties:

| Property | Default | Plugin | Description |
|----------|---------|--------|-------------|
| `jenkins.ec2.checkIntervalMinutes` | 1 | ec2 | Minutes between retention checks |
| `jenkins.ec2.instanceCountCacheTtlMs` | 30000 | ec2 | Instance count cache TTL |
| `jenkins.ec2.scheduleMaintenanceDelayMs` | 1000 | ec2 | Delay before queue maintenance on early provisioning events |
| `hudson.plugins.ec2.EC2Computer.instanceCacheTTLMs` | 30000 | ec2 | SSH verification state cache TTL |
| `azure.vm.scheduleMaintenanceDelayMs` | 1000 | azure-vm-agents | Delay before queue maintenance on non-immediate events |
| `azure.vm.existsCacheTtlMs` | 30000 | azure-vm-agents | VM existence cache TTL |
| `JobRestrictionProperty.cacheDisabled` | false | job-restrictions | Disable canTake cache |
| `JobRestrictionProperty.cacheTtlMs` | 30000 | job-restrictions | canTake cache TTL |
| `JobRestrictionProperty.cacheMaxEntries` | 500 | job-restrictions | Max cache entries per node |
| `lockable-resources.scriptCacheTtlMs` | 30000 | lockable-resources | Groovy script evaluation cache TTL |
| `lockable-resources.asyncSave` | true | lockable-resources | Enable async disk persistence |
| `lockable-resources.saveCoalesceMs` | 1000 | lockable-resources | Coalesce window for async saves (ms) |
| `hudson.model.Queue.maintainInterval` | 5000 | jenkins core | Queue maintenance interval (ms) |
