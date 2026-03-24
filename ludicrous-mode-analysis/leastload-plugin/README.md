# Leastload Plugin — Queue Performance Patches

Patches against `origin/master` (`jenkinsci/leastload-plugin`). 1 file changed, +203 / -38 lines.

---

## How This Helps Unblock the Queue

The leastload plugin's `LeastLoadBalancer.map()` is called for every buildable item during `Queue.maintain()`. When `map()` returns null, the item stays in the buildables list and is retried on the *next* maintenance cycle — at least 5 seconds later (or 35+ seconds later when the cycle itself is slow). Two bugs caused the plugin to return null far more often than necessary, multiplying scheduling delays.

With the [EC2 plugin](../ec2-plugin/) patches ensuring the Queue lock is held for microseconds and the [job-restrictions plugin](../job-restrictions-plugin/) cache eliminating repeated restriction evaluation, the leastload plugin is the final piece: it must actually assign work to executors rather than deferring to the next cycle. These patches ensure that when an idle executor exists for a task's label, the plugin assigns it immediately — no round-robin deferrals, no pipeline exclusions, no unnecessary null returns.

Together, the three plugin patches mean `Queue.maintain()` completes in milliseconds: the EC2 plugin returns instantly from retention checks, the job-restrictions plugin returns cached results, and the leastload plugin assigns work on the first attempt.

---

## Files Changed

All changes are in `LeastLoadBalancer.java`.

### Fix 1: Pipeline Tasks Now Use Leastload

Before this patch, every Jenkins Pipeline `node('label')` block bypassed leastload entirely and used the default consistent-hash load balancer.

The root cause: `isDisabled()` checked `subTask instanceof Job`. For pipeline work, the queue task is `ExecutorStepExecution.PlaceholderTask`, and its `getOwnerTask()` returns the `Run` (e.g. `WorkflowRun`), not the `Job`. The plugin treated any non-Job owner as "disabled" and delegated to the fallback balancer.

The fix adds `resolveJob()` which derives the `Job` from the owner: if it's a `Run`, call `getParent()` (e.g. `WorkflowRun.getParent()` → `WorkflowJob`). Pipeline work now gets least-load distribution. If no Job can be resolved, leastload is applied by default rather than falling back.

### Fix 2: Per-Label Round-Robin Tracking

The plugin maintained a single global set of "available nodes this round." After assigning work to a node, it removed that node from the set. For label-restricted work where only one node matched (common with EC2 auto-provisioned agents), the node was removed after the first assignment and the next task needing the same label saw an empty set — causing `map()` to return null and defer for 5+ seconds.

The fix tracks available nodes per-label in addition to the global set. Each label has its own capacity pool, so assigning `x86_64_medium` work doesn't consume `x86_64_large` capacity. When the per-label set for a task's label is empty, a proactive refresh picks up newly provisioned nodes immediately.

### Fix 3: Label-Restricted Single-Node Immediate Assign

Even after per-label tracking, when a single node has the required label and has been marked "used" this round but still has idle executors, the round-robin logic would filter it out. The worksheet only contains chunks for currently idle executors, so there's no risk of double-assignment.

The fix uses `useableChunks` directly when round-robin filters to empty for label-restricted work. This eliminates 10–60+ second delays when a single node has the required label and multiple executors.

### Load Prediction Disabled

`getLoadPredictors()` now returns an empty collection, so `MappingWorksheet` skips the load prediction block entirely. This avoids O(computers × predictors × FutureLoads) TreeMap operations per buildable item — work that added measurable overhead with many agents and provided negligible benefit.

### Shuffle Skip for Single-Work Tasks

`getApplicableSortedByLoad()` skips `Collections.shuffle()` for single-work tasks (the vast majority of pipeline and freestyle jobs). The shuffle was added for JENKINS-18323 which only affects multi-work (matrix) jobs.

### Observability

Each `map()` call generates a hex trace ID logged with every message, and `logMapDuration()` records elapsed time and outcome at `FINE` level. Diagnostic logging now includes per-label node counts and short-circuits with a targeted message when no executors exist for the task's label.

---

## Background Analysis

- [LEASTLOAD_DELAY_ANALYSIS.md](background/LEASTLOAD_DELAY_ANALYSIS.md) — Root cause analysis of 1+ minute scheduling delays
- [LEASTLOAD_DELAY_ANALYSIS_ROOT.md](background/LEASTLOAD_DELAY_ANALYSIS_ROOT.md) — Earlier delay analysis
- [EXECUTOR_START_DELAY_ANALYSIS.md](../jenkins/background/EXECUTOR_START_DELAY_ANALYSIS.md) — Why executors take ~30s to start after Queue assigns them (Jenkins core)
- [MAPPING_WORKSHEET_PERFORMANCE.md](../jenkins/background/MAPPING_WORKSHEET_PERFORMANCE.md) — Performance issues in MappingWorksheet (Jenkins core)
- [QUEUE_TIME_BREAKDOWN.md](../jenkins/background/QUEUE_TIME_BREAKDOWN.md) — Mermaid pie charts showing 35-second queue delay breakdown
