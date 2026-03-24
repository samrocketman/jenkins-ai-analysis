# Jenkins Core — Queue Performance Analysis

Analysis of `hudson.model.Queue` internals. No code changes — these documents describe the core constraints that all plugin patches work around.

---

## How the Queue Works

Jenkins core runs a single-threaded `Queue.maintain()` loop that is the only place buildable items are assigned to executors. It holds a global `ReentrantLock` for the entire duration of the method. While the lock is held, nothing in Jenkins can schedule new builds, cancel queued items, or inspect the queue.

### The maintain() cycle

1. **Lock acquired** at the top of `maintain()`.
2. For each buildable item:
   - `getCauseOfBlockageForItem(item)` — checks task-level blockers and all `QueueTaskDispatcher`s.
   - For each parked (idle) executor: `JobOffer.getCauseOfBlockage(item)` → `Node.canTake(item)` → iterates all `NodeProperty` instances (including job-restrictions plugin).
   - Builds a `MappingWorksheet` from candidates and load predictors.
   - Calls `loadBalancer.map(task, worksheet)` (e.g. leastload plugin).
   - If the mapping succeeds: `Mapping.execute()` → `JobOffer.set(workUnit)` → `Executor.start()`.
3. **Lock released** only after all buildables are processed.

This means:
- Complexity is **O(buildables × executors × nodeProperties)** per cycle.
- Executors assigned in early iterations cannot start until `maintain()` finishes and releases the lock (they block on `Queue.withLock()` in their `run()` method).
- The timer uses `scheduleWithFixedDelay` with a default 5-second interval, so the next cycle starts 5 seconds *after* the previous cycle *finishes*. A slow cycle directly increases the gap.

### Why this caused problems at scale

With plugins making blocking AWS API calls under the lock (EC2 plugin), re-evaluating restrictions tens of thousands of times (job-restrictions plugin), and deferring work to the next cycle (leastload plugin), each `maintain()` cycle took 30+ seconds. The effective period between cycles was 35+ seconds. Past ~800 agents, the cycle grew so long that Jenkins became unresponsive.

---

## Key Bottlenecks

| Bottleneck | Description |
|------------|-------------|
| **Global lock for entire maintain()** | All `schedule()`, `cancel()`, `getItems()`, and `onStartExecuting()` calls block until `maintain()` releases. |
| **getCauseOfBlockageForItem × 3** | Called for every blocked, waiting, and buildable item per cycle. |
| **O(buildables × executors × nodeProperties)** | Each buildable checks each parked executor; each executor checks each NodeProperty. Quadratic with queue depth × executor count. |
| **updateSnapshot() after each transition** | Creates a new `Snapshot` copying all lists after every state change inside maintain(). |
| **5-second scheduleWithFixedDelay** | Even fast cycles have a 5-second floor. Slow cycles push the next run out by cycle duration + 5s. |
| **Executor lock contention** | Executors assigned in early iterations wait for the lock until maintain() exits — up to 30s for the first-assigned executor when cycles are slow. |

---

## MappingWorksheet Performance

`MappingWorksheet` is constructed once per buildable item per cycle. Its cost scales with:

- **Load prediction**: O(computers × predictors × min(100, predictions) × TreeMap ops) when `getEstimatedDuration() > 0`. Eliminated by leastload plugin returning empty `getLoadPredictors()`.
- **applicableExecutorChunks()**: O(work chunks × executor chunks) `canAccept()` calls (label matching + permission checks) with no caching. Called multiple times per `map()`.
- **isPartiallyValid()**: Repeats `canAccept()` for each tentative assignment during greedy search.

---

## Background Analysis

- [EXECUTOR_START_DELAY_ANALYSIS.md](background/EXECUTOR_START_DELAY_ANALYSIS.md) — Why executors take up to ~30s to start after Queue assigns them (Queue lock contention)
- [MAPPING_WORKSHEET_PERFORMANCE.md](background/MAPPING_WORKSHEET_PERFORMANCE.md) — Performance-sensitive behavior in MappingWorksheet (load prediction, canAccept, isPartiallyValid)
- [QUEUE_TIME_BREAKDOWN.md](background/QUEUE_TIME_BREAKDOWN.md) — Mermaid pie charts showing the 35-second queue delay breakdown
