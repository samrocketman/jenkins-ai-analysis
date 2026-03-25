# Lockable Resources Plugin — Queue Performance Patches

Analysis of `jenkinsci/lockable-resources-plugin` for Queue lock contention and dispatcher thread blocking. Proposed patches across 3 phases, ~3–5 files changed.

---

## How This Plugin Differs from Cloud Plugins

Unlike the EC2 and Azure VM plugins, the lockable-resources-plugin does not implement `RetentionStrategy`, `Cloud`, or `NodeProvisioner.Strategy`. Instead, it hooks into the Queue via a **`QueueTaskDispatcher`** — the `canRun(Queue.Item)` method is called **under the Queue lock** for every item (blocked, waiting, and buildable) during every `Queue.maintain()` cycle. This means the plugin's code runs in the innermost loop of the Queue maintenance pipeline.

The plugin also manages its own internal pipeline queue (`queuedContexts`) for `lock` step waiting, separate from the Jenkins Queue. When resources are freed, the plugin tries to satisfy waiting pipeline contexts directly, but never triggers `scheduleMaintenance()` to notify the Jenkins Queue.

---

## Issues Found

### Issue 1: No `scheduleMaintenance()` Triggers Anywhere

The plugin never calls `scheduleMaintenance()`. When resources are freed (build completes, pipeline lock body finishes, user unreserves via UI, etc.), items waiting in the Jenkins Queue for those resources won't be dispatched until the next 5-second timer tick. This adds up to 5 seconds of unnecessary latency to every resource release.

### Issue 2: Groovy Script Evaluation Under Queue Lock

When jobs use `resourceMatchScript` (Groovy-based resource selection), `canRun()` evaluates the Groovy script for every lockable resource on every cache miss. `SecureGroovyScript.evaluate()` involves Groovy compilation, sandbox interception, and script execution — all under the Queue lock. With 50 resources and 5 scripted items, that's 250 Groovy evaluations per cycle.

The existing `cachedCandidates` Guava cache (5-minute TTL) mitigates this on subsequent cycles, but every cache miss (first evaluation, or after `uncacheIfFreeing()` invalidation) is heavyweight.

### Issue 3: `syncResources` Lock Contention Under Queue Lock

The `tryQueue()` method acquires a plugin-level `synchronized(syncResources)` monitor while running under the Queue lock. Other threads (build lifecycle via `LockRunListener`, pipeline steps via `LockStepExecution`, UI actions) also hold `syncResources` while performing disk I/O (`save()` writes the config XML to disk). This means the Queue lock is transitively extended by disk write latency on unrelated threads.

### Issue 4: Label Expression Parsing Under Queue Lock

`isValidLabel()` calls `Label.parseExpression()` and creates `LabelAtom` objects for every resource on every label-based check. For simple label names, a fast `List.contains()` short-circuits this. For label expressions (e.g. `label1 && label2`), parsing and evaluation happens for every resource per item per cycle.

---

## Proposed Patches

### Phase 1: Comprehensive `scheduleMaintenance()` Triggers

Since Queue maintenance is now millisecond-fast (with the other Ludicrous Mode patches), trigger it at every event where lockable resource availability changes.

**New infrastructure:**

```java
// In LockableResourcesManager:
public static void scheduleQueueMaintenance() {
    Jenkins j = Jenkins.getInstanceOrNull();
    if (j != null) {
        j.getQueue().scheduleMaintenance();
    }
}
```

**Trigger points (6 total, all immediate):**

| Location | Event |
|----------|-------|
| `unlockResources()` after `save()` | Resources freed (build complete, pipeline unlock, UI unlock) |
| `unreserve()` after `save()` | Resources unreserved (UI or API) |
| `reset()` after `save()` | Resources reset (UI or API) |
| `LockRunListener.onCompleted()` after `unlockBuild()` | Freestyle/matrix build completed |
| `LockRunListener.onDeleted()` after `unlockBuild()` | Build deleted |
| `FreeDeadJobs.freePostMortemResources()` after cleanup loop | Orphaned resources freed at startup |

The call should be placed **outside** the `synchronized(syncResources)` block to avoid holding the plugin lock while triggering Queue maintenance.

`scheduleMaintenance()` coalesces via `AtmostOneTaskExecutor` — multiple triggers in quick succession do not cause multiple `maintain()` runs.

### Phase 2: Reduce `syncResources` Contention Under Queue Lock

**Goal:** Minimize the time the Queue lock waits for `syncResources`.

1. **Async `save()`** — Replace synchronous disk writes with a coalesced background save. State mutations happen in-memory under a brief lock; disk persistence is deferred (e.g. via `jenkins.util.Timer.get().schedule()`). This prevents `onCompleted()` → `save()` from blocking `canRun()` → `tryQueue()`.

2. **Narrower lock scope in `tryQueue()`** — Currently the entire method body runs under `syncResources`. The lock is only needed for reading resource state and setting queued flags — the parameter extraction, cache read, and free-status checks don't need exclusive access.

### Phase 3: Cache Heavyweight Computations

1. **Groovy script result TTL cache** — Add a per-`(resourceName, scriptHash)` cache with 30s TTL for `scriptMatches()` results. Only "match=true" results are cached (similar to the job-restrictions plugin pattern of only caching "allow" results). This bounds per-cycle cost to O(expired_or_new_entries) instead of O(all_resources).

2. **Label expression cache** — Cache `Label.parseExpression()` results per candidate string and `Set<LabelAtom>` per resource (invalidated on `setLabels()`). This avoids repeated AST construction and object creation.

3. **Project-to-resource index** — Replace the O(total_resources) `checkCurrentResourcesStatus()` scan with a `Map<String, Set<LockableResource>>` index updated on `setQueued()`/`unqueue()`.

---

## Configuration

| System Property | Default | Phase | Description |
|-----------------|---------|-------|-------------|
| `lockable-resources.scriptCacheTtlMs` | 30000 | 3 | Groovy script evaluation cache TTL (ms) |
| `lockable-resources.asyncSave` | true | 2 | Enable async disk persistence |
| `lockable-resources.saveCoalesceMs` | 1000 | 2 | Coalesce window for async saves (ms) |

---

## Files Changed Summary

### Phase 1 (scheduleMaintenance triggers)

| File | Change |
|------|--------|
| **LockableResourcesManager.java** | Add `scheduleQueueMaintenance()` static helper. Add calls in `unlockResources()`, `unreserve()`, `reset()` |
| **LockRunListener.java** | Add `scheduleQueueMaintenance()` calls in `onCompleted()` and `onDeleted()` |
| **FreeDeadJobs.java** | Add `scheduleQueueMaintenance()` after orphan cleanup |

### Phase 2 (contention reduction)

| File | Change |
|------|--------|
| **LockableResourcesManager.java** | Async `save()`, narrower `syncResources` scope in `tryQueue()` |

### Phase 3 (computation caching)

| File | Change |
|------|--------|
| **LockableResource.java** | Groovy script result cache, label expression cache |
| **LockableResourcesManager.java** | Project-to-resource index for `checkCurrentResourcesStatus()` |

---

## Background Analysis

- [LOCKABLE_RESOURCES_QUEUE_AUDIT.md](background/LOCKABLE_RESOURCES_QUEUE_AUDIT.md) — Full audit of all code paths under Queue lock and `syncResources` contention
- [LOCKABLE_RESOURCES_HEAVY_LIGHTWEIGHT_ANALYSIS.md](background/LOCKABLE_RESOURCES_HEAVY_LIGHTWEIGHT_ANALYSIS.md) — Detailed heavy vs lightweight classification with proposed fixes
