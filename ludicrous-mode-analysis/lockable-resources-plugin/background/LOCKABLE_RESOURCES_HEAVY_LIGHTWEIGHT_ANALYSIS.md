# Lockable Resources Plugin — Heavy vs Lightweight Classification

Detailed classification of every operation in the lockable-resources-plugin that could affect Jenkins Queue performance, with proposed fixes for heavyweight operations.

---

## Classification Key

- **LIGHTWEIGHT**: Returns in microseconds. No network I/O, no disk I/O, no script execution, no lock contention. Safe to run under the Queue lock.
- **HEAVYWEIGHT**: Executes Groovy scripts, performs disk I/O, or holds contended locks for extended periods. Must be optimized or eliminated from the Queue lock path.
- **MODERATE**: In-memory operations that scale with resource count. Acceptable for small resource sets but can accumulate at scale.
- **CONTENTION**: Acquires a shared lock that may be held by threads doing I/O, transitively extending Queue lock hold time.

---

## 1. LockableResourcesQueueTaskDispatcher

### `canRun(Queue.Item)` — UNDER QUEUE LOCK

Called for every queue item during every `Queue.maintain()` cycle.

#### Fast-exit path (no lockable resources configured on job) — LIGHTWEIGHT

If `Utils.requiredResources(project)` returns null (no `RequiredResourcesProperty` on the job), `canRun()` returns null immediately. This is the majority case for most Jenkins jobs.

| Step | Cost | Classification |
|------|------|---------------|
| `item.task.getClass().getName()` comparison | O(1) | LIGHTWEIGHT |
| `Utils.getProject(item)` | O(1) | LIGHTWEIGHT |
| `Utils.requiredResources(project)` | O(1) property lookup | LIGHTWEIGHT |
| Return null | O(1) | LIGHTWEIGHT |

**No changes needed** for jobs without lockable resources.

#### Named resources path (Branch B, lines 130–138) — LIGHTWEIGHT

When a job specifies resources by name (no label, no script, no quantity):

| Step | Cost | Classification |
|------|------|---------------|
| `LockableResourcesManager.get().queue(resources.required, ...)` | O(required_resources) | LIGHTWEIGHT |
| Per-resource: `isReserved()`, `isQueued()`, `isLocked()` | O(1) per resource | LIGHTWEIGHT |
| Per-resource: `setQueued(queueItemId, queueProjectName)` | O(1) per resource | LIGHTWEIGHT |

**No changes needed** — this path is already fast. Note: `queue()` is NOT synchronized on `syncResources`, which is a race condition but keeps the Queue lock path fast.

#### Label/Script/Number path (Branch A, lines 71–128) — MODERATE to HEAVYWEIGHT

When a job specifies resources by label, script, or quantity:

| Step | Cost | Classification |
|------|------|---------------|
| Parameter extraction | O(params) | LIGHTWEIGHT |
| **`tryQueue(...)` enters `synchronized(syncResources)`** | **Waits for concurrent holders** | **CONTENTION** |
| `checkCurrentResourcesStatus(...)` | O(total_resources) | MODERATE |
| **`getResourcesMatchingScript(script, params)`** (cache miss) | **O(total_resources × script_cost)** | **HEAVYWEIGHT** |
| `getResourcesMatchingScript(script, params)` (cache hit) | O(cached_candidates) for `retainAll` | LIGHTWEIGHT |
| `getResourcesWithLabel(label)` | O(total_resources × label_cost) | MODERATE |
| `cachedCandidates` access | O(1) | LIGHTWEIGHT |
| Free status iteration | O(candidates) | LIGHTWEIGHT |
| `setQueued()` per selected | O(selected) | LIGHTWEIGHT |

---

## 2. Heavyweight Operations — Detailed Analysis

### 2.1 Groovy Script Evaluation (`scriptMatches`)

**Location:** `LockableResource.scriptMatches()` (lines 331–355)

**Called from:** `getResourcesMatchingScript()` → called from `tryQueue()` → called from `canRun()` under Queue lock.

**What it does:**
```
for each resource in this.resources:
    Binding binding = new Binding(params);
    binding.setVariable("resourceName", name);
    binding.setVariable("resourceDescription", description);
    binding.setVariable("resourceLabels", labelsAsList);
    binding.setVariable("resourceNote", note);
    Object result = script.evaluate(classLoader, binding, null);
    return (Boolean) result;
```

**Cost per resource:**
- `new Binding(params)` — HashMap copy
- 4 × `setVariable()` — HashMap puts
- `script.evaluate()` — **Full Groovy script compilation + sandbox execution**. Even cached/compiled scripts have per-invocation overhead from sandbox checks (method call interception, property access interception).

**Scaling:** With R resources and S scripted queue items: R × S evaluations per `maintain()` cycle. At 50 resources × 5 scripted items = 250 evaluations per cycle, each taking potentially 1–10ms = 0.25–2.5 seconds under the Queue lock.

**Current mitigation:** `cachedCandidates` Guava cache (5-minute TTL, keyed by `queueItemId`). On cache hit, the script is NOT re-evaluated. However:
- First evaluation (cache miss) is always heavyweight
- Cache is invalidated by `uncacheIfFreeing()` when resources are freed, causing re-evaluation on the next cycle
- `candidates.retainAll(this.resources)` on cache hit creates garbage and modifies the cached list

**Proposed fix:** Cache script evaluation results per resource with a short TTL (e.g. 30s), separate from the candidate cache. Only re-evaluate when the cache expires or when a resource's state changes. This bounds the per-cycle cost to O(expired_entries) instead of O(all_resources).

### 2.2 `synchronized(syncResources)` Contention

**Location:** `LockableResourcesManager.tryQueue()` line 498

**The problem:** The `tryQueue()` method acquires `syncResources` while running under the Queue lock. Other threads (build lifecycle, pipeline steps, UI actions) also acquire `syncResources` and perform disk I/O (`save()`) while holding it.

**Worst case:** A build completes → `onCompleted()` acquires `syncResources` → calls `unlockResources()` → calls `save()` (disk write). Meanwhile, `Queue.maintain()` calls `canRun()` → `tryQueue()` → blocks on `syncResources` until the disk write completes. The Queue lock is held for the entire wait.

**Scale factor:** With many concurrent build completions and lock/unlock operations, contention on `syncResources` directly amplifies Queue lock hold time.

**Proposed fix:**
1. **Short-term:** Reduce the scope of `syncResources` in `tryQueue()`. Only hold it for the state-reading portion, not for the entire method.
2. **Medium-term:** Use a `ReadWriteLock`. `canRun()` takes a read lock (concurrent with other readers). `lock()`/`unlock()`/`save()` take a write lock. This eliminates contention between concurrent `canRun()` calls.
3. **Long-term:** Make `save()` asynchronous. State mutations happen in-memory under a brief lock; disk persistence is deferred to a background thread.

### 2.3 Label Expression Parsing (`isValidLabel`)

**Location:** `LockableResource.isValidLabel()` (lines 281–298)

**Called from:** `getResourcesWithLabel()` → called from `tryQueue()` → called from `canRun()` under Queue lock.

**What it does:**
```
if (labelsContain(candidate)) return true;  // fast: list.contains()
Label labelExpression = Label.parseExpression(candidate);  // parse expression
Set<LabelAtom> atomLabels = new HashSet<>();  // create atom set
for (String label : getLabelsAsList()) atomLabels.add(new LabelAtom(label));
return labelExpression.matches(atomLabels);  // evaluate
```

**Cost per resource:**
- `labelsContain(candidate)` — O(labels_per_resource) list scan. If the candidate is a simple label name and is present in the list, this returns `true` immediately — LIGHTWEIGHT.
- `Label.parseExpression(candidate)` — Parses the label expression string. For simple atoms this is fast; for compound expressions (`A && B || C`) it builds an AST.
- `new LabelAtom(label)` per label — Object creation per label per resource.
- `labelExpression.matches(atomLabels)` — Expression evaluation.

**Scaling:** With R resources, L labels per resource, and Q label-based queue items: R × Q evaluations per cycle. If labels are simple atoms and match via `labelsContain()`, the expensive path is avoided.

**Proposed fix:** Cache the parsed `Label` expression per candidate string. Cache the `Set<LabelAtom>` per resource (invalidated when resource labels change). This avoids repeated parsing and object creation.

### 2.4 `checkCurrentResourcesStatus()` — MODERATE

**Location:** `LockableResourcesManager.checkCurrentResourcesStatus()` (lines 577–597)

Iterates ALL resources to find ones queued for the same project. O(total_resources) per call.

**Proposed fix:** Maintain a `Map<String, List<LockableResource>>` index from project name to resources. This turns O(total_resources) into O(1) lookup.

---

## 3. Operations That Free Resources (scheduleMaintenance Candidates)

### 3.1 `unlockResources()` — Frees locked resources

**Location:** `LockableResourcesManager.unlockResources()` (lines 713–726)

```java
synchronized (syncResources) {
    this.freeResources(resourcesToUnLock, build);
    while (proceedNextContext()) { }  // process pipeline queue
    save();  // DISK I/O
}
```

`freeResources()` clears lock state and calls `uncacheIfFreeing()`. Then `proceedNextContext()` loops through the plugin's internal pipeline queue, locking resources for the next waiting pipeline context.

**Missing:** No `scheduleMaintenance()` after resources are freed. Items in the Jenkins Queue (not the plugin's pipeline queue) waiting for these resources won't be dispatched until the next timer tick.

**Proposed fix:** Call `scheduleMaintenance()` immediately after `unlockResources()` completes.

### 3.2 `unreserve()` — Frees reserved resources

**Location:** `LockableResourcesManager.unreserve()` (lines 996–1010)

```java
synchronized (syncResources) {
    unreserveResources(resources);  // clears reserve state + save()
    proceedNextContext();
    save();
}
```

**Missing:** No `scheduleMaintenance()`.

### 3.3 `reset()` — Clears all state

**Location:** `LockableResourcesManager.reset()` (lines 1020–1028)

**Missing:** No `scheduleMaintenance()`.

### 3.4 `recycle()` — Unlock + unreserve

**Location:** `LockableResourcesManager.recycle()` (lines 1038–1045)

Calls `unlockResources()` then `unreserve()`.

**Missing:** No direct `scheduleMaintenance()` (inherits from callers, but none of them add it either).

### 3.5 `LockRunListener.onCompleted()` / `onDeleted()` — Build lifecycle

Calls `unlockBuild()` → `unlockNames()` → `unlockResources()`.

**Missing:** No `scheduleMaintenance()`.

### 3.6 `LockStepExecution.Callback.finished()` — Pipeline lock body ends

Calls `unlockNames()` → `unlockResources()`.

**Missing:** No `scheduleMaintenance()`.

### 3.7 UI Actions

`doUnlock()`, `doUnreserve()`, `doReset()`, `doSteal()` all free resources.

**Missing:** No `scheduleMaintenance()`.

---

## 4. Proposed Patches — Phased

### Phase 1: Comprehensive `scheduleMaintenance()` Triggers

Add `scheduleMaintenance()` calls at every point where resources become available.

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

**Trigger points (all immediate, 0ms delay):**

| Location | Event |
|----------|-------|
| `unlockResources()` after `save()` | Resources freed after build or pipeline |
| `unreserve()` after `save()` | Resources unreserved |
| `reset()` after `save()` | Resources reset |
| `LockRunListener.onCompleted()` after `unlockBuild()` | Build completed |
| `LockRunListener.onDeleted()` after `unlockBuild()` | Build deleted |
| `FreeDeadJobs.freePostMortemResources()` after cleanup | Startup cleanup |

Unlike the EC2 and Azure plugins, there is no need for delayed triggers — lockable resources are freed instantaneously (no provisioning delay). The call to `scheduleMaintenance()` should be made **outside** the `synchronized(syncResources)` block to avoid holding the plugin lock while triggering Queue maintenance.

### Phase 2: Reduce `syncResources` Contention

**Goal:** Minimize the time `syncResources` is held, especially when disk I/O is involved, so that `canRun()` (under Queue lock) spends less time waiting.

1. **Async save:** Replace synchronous `save()` with a deferred save. State mutations happen in-memory under `syncResources`; disk persistence is coalesced and written asynchronously (e.g. via `Saver` pattern or `AtmostOneTaskExecutor`).

2. **Narrower lock in `tryQueue()`:** Only hold `syncResources` for the state-reading portion of `tryQueue()`. The `setQueued()` calls at the end can use a brief lock or CAS operations.

### Phase 3: Cache Heavyweight computations

1. **Groovy script result caching:** Add a per-resource, per-script TTL cache for `scriptMatches()` results. Key: `(resourceName, scriptHash)`. TTL: 30 seconds (configurable). This ensures the first `canRun()` after resources change re-evaluates, but subsequent cycles use cached results.

2. **Label expression caching:** Cache `Label.parseExpression(candidate)` results per candidate string. Cache `Set<LabelAtom>` per resource (invalidated when `setLabels()` is called).

3. **Project-to-resource index:** Replace `checkCurrentResourcesStatus()` O(N) scan with a `Map<String, Set<LockableResource>>` index.

---

## 5. Summary Table

| Class | Method | Context | Classification | Fix Needed? |
|-------|--------|---------|---------------|-------------|
| QueueTaskDispatcher | `canRun()` fast-exit | Queue lock | LIGHTWEIGHT | No |
| QueueTaskDispatcher | `canRun()` → named resources | Queue lock | LIGHTWEIGHT | No (but race condition exists) |
| QueueTaskDispatcher | `canRun()` → `tryQueue()` lock | Queue lock | **CONTENTION** | **Yes — reduce scope, async save** |
| QueueTaskDispatcher | `canRun()` → Groovy script (miss) | Queue lock | **HEAVYWEIGHT** | **Yes — script result cache** |
| QueueTaskDispatcher | `canRun()` → Groovy script (hit) | Queue lock | LIGHTWEIGHT | No |
| QueueTaskDispatcher | `canRun()` → label matching | Queue lock | MODERATE | Cache label expressions |
| QueueTaskDispatcher | `canRun()` → `checkCurrentResourcesStatus` | Queue lock | MODERATE | Index by project name |
| LockRunListener | `onStarted()` | Executor thread | CONTENTION (holds `syncResources` during save) | Async save |
| LockRunListener | `onCompleted()` | Executor thread | CONTENTION + missing trigger | **Yes — scheduleMaintenance()** |
| LockRunListener | `onDeleted()` | Executor thread | CONTENTION + missing trigger | **Yes — scheduleMaintenance()** |
| LockStepExecution | `start()` | CPS thread | CONTENTION (holds `syncResources`) | Async save |
| LockStepExecution | `Callback.finished()` | CPS thread | CONTENTION + missing trigger | **Yes — scheduleMaintenance()** |
| LockStepExecution | `stop()` | CPS thread | LIGHTWEIGHT + missing trigger | **Yes — scheduleMaintenance()** |
| LockableResourcesManager | `unlockResources()` | Various | Missing trigger | **Yes — scheduleMaintenance()** |
| LockableResourcesManager | `unreserve()` | Various | Missing trigger | **Yes — scheduleMaintenance()** |
| LockableResourcesManager | `reset()` | Various | Missing trigger | **Yes — scheduleMaintenance()** |
| UI actions | `doUnlock/doUnreserve/doReset/doSteal` | HTTP thread | Missing trigger | **Yes — scheduleMaintenance()** |
| NodesMirror | `onConfigurationChange()` | Computer thread | MODERATE | No (infrequent) |
| FreeDeadJobs | `freePostMortemResources()` | Startup | Missing trigger | **Yes — scheduleMaintenance()** |
