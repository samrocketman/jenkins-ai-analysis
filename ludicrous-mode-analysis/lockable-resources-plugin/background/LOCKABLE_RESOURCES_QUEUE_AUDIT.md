# Lockable Resources Plugin — Queue Lock & Dispatcher Audit

Audit of every code path in the lockable-resources-plugin that runs under the Jenkins Queue lock or contends with the Queue maintenance pipeline.

---

## How Jenkins Invokes Plugin Code

The lockable-resources-plugin hooks into the Queue maintenance pipeline via a **`QueueTaskDispatcher`**, which is called **under the Queue lock** for every item during `Queue.maintain()`. It also has a **`RunListener`**, a **`ComputerListener`**, and a **Pipeline Step** that interact with a shared plugin-level lock (`syncResources`).

| Extension Point | Class | Runs Under Queue Lock? |
|----------------|-------|----------------------|
| `QueueTaskDispatcher.canRun(Item)` | `LockableResourcesQueueTaskDispatcher` | **Yes** — called for every queue item during `maintain()` |
| `RunListener.onStarted()` | `LockRunListener` | **No** — but acquires `syncResources` |
| `RunListener.onCompleted()` | `LockRunListener` | **No** — but acquires `syncResources` |
| `RunListener.onDeleted()` | `LockRunListener` | **No** — but acquires `syncResources` |
| `ComputerListener.onConfigurationChange()` | `NodesMirror` | **No** — but acquires `syncResources` |
| Pipeline `lock` step | `LockStepExecution.start()` | **No** — but acquires `syncResources` |

---

## 1. LockableResourcesQueueTaskDispatcher.canRun() — UNDER QUEUE LOCK

**This is the critical hot path.** Jenkins calls `QueueTaskDispatcher.canRun(Queue.Item)` during `Queue.maintain()` under the global Queue lock. It is called for **every** blocked, waiting, and buildable item on **every** maintenance cycle. With N queue items, this runs N times per cycle while the Queue lock is held.

### Step-by-step trace:

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | `item.task.getClass().getName().equals("hudson.matrix.MatrixProject")` | String comparison | LIGHTWEIGHT |
| 2 | `Utils.getProject(item)` | `instanceof Job` check | LIGHTWEIGHT |
| 3 | `Utils.requiredResources(project)` | `project.getProperty(RequiredResourcesProperty.class)` | LIGHTWEIGHT (in-memory property lookup) |
| 4 | Early return if no resources required | Boolean checks | LIGHTWEIGHT |
| 5 | `Integer.parseInt(resources.requiredNumber)` | String parse | LIGHTWEIGHT |
| 6 | `item.getActions(ParametersAction.class)` | In-memory action list | LIGHTWEIGHT |
| 7 | Parameter extraction loop | In-memory iteration | LIGHTWEIGHT |
| 8 | `ExtensionList.lookup(Utils.MatrixAssist.class)` | Extension list access | LIGHTWEIGHT |
| 9 | `LockableResourcesManager.get()` | Singleton lookup via Jenkins | LIGHTWEIGHT |

Then the path diverges into two branches:

### Branch A: Label/Script/Number path (lines 71–128)

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 10a | **`tryQueue(resources, item.getId(), ...)`** | Enters `synchronized(syncResources)` | **CONTENTION POINT** |
| 11a | `checkCurrentResourcesStatus(selected, ...)` | Iterates ALL resources checking `getQueueItemProject()` | O(total_resources) — MODERATE |
| 12a | `getResourcesMatchingScript(systemGroovyScript, params)` | **Evaluates Groovy script per resource** | **HEAVYWEIGHT** |
| 12b | `getResourcesWithLabel(requiredResources.label)` | Iterates resources, `isValidLabel()` per resource | MODERATE |
| 13a | `cachedCandidates.getIfPresent(queueItemId)` | Guava cache read | LIGHTWEIGHT |
| 14a | Free status checks (`!rs.isReserved() && !rs.isLocked() && !rs.isQueued()`) | Boolean field reads | LIGHTWEIGHT |
| 15a | `rsc.setQueued(queueItemId, queueItemProject)` | Field sets | LIGHTWEIGHT |

### Branch B: Named resources path (lines 130–138)

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 10b | `queue(resources.required, item.getId(), ...)` | No `synchronized` — **BUG**: unsynchronized access to shared state | LIGHTWEIGHT but UNSAFE |
| 11b | `r.isReserved() || r.isQueued(queueItemId) || r.isLocked()` | Boolean field reads | LIGHTWEIGHT |
| 12b | `r.setQueued(queueItemId, queueProjectName)` | Field sets | LIGHTWEIGHT |

---

## 2. Detailed Analysis of Heavyweight Operations Under Queue Lock

### 2.1 Groovy Script Evaluation — HEAVYWEIGHT

**Location:** `LockableResourcesManager.getResourcesMatchingScript()` → `LockableResource.scriptMatches()`

When a job uses `resourceMatchScript` (Groovy-based resource selection), `canRun()` evaluates the script for **every** lockable resource, **every** queue item that uses scripts, **every** maintenance cycle.

```
scriptMatches():
  Binding binding = new Binding(params);       // Object creation
  binding.setVariable("resourceName", name);   // 4 variable bindings
  script.evaluate(classLoader, binding, null); // FULL GROOVY EVALUATION
```

`SecureGroovyScript.evaluate()` compiles and executes a Groovy script in a sandboxed environment. Cost depends on script complexity but is inherently heavyweight (classloader operations, sandbox checks, script execution).

**Impact:** With R resources and Q queue items using scripts, this runs R × Q script evaluations per `maintain()` cycle, all under the Queue lock. At 100 resources × 10 scripted items = 1,000 Groovy evaluations per cycle.

**Mitigation:** The plugin has a `cachedCandidates` Guava cache with 5-minute TTL that caches the script results per `queueItemId`. On cache hit, the script is NOT re-evaluated — but `candidates.retainAll(this.resources)` is called to prune stale entries. The cache only helps on subsequent cycles for the same queue item; the first evaluation (cache miss) is always heavyweight.

### 2.2 `synchronized(syncResources)` Contention — CONTENTION

**Location:** `LockableResourcesManager.tryQueue()` line 498

The `tryQueue()` method acquires a plugin-level monitor (`syncResources`) while the caller already holds the Queue lock. This creates a lock ordering dependency:

- **Queue.maintain() path:** Queue lock → `syncResources`
- **LockRunListener.onStarted():** `syncResources` → (no Queue lock needed)
- **LockRunListener.onCompleted():** calls `unlockBuild()` → `unlockNames()` → `syncResources`
- **LockStepExecution.start():** `syncResources`

If `onStarted()` or another thread holds `syncResources` while the Queue lock is held by `maintain()`, the Queue maintenance thread blocks waiting for `syncResources`, extending the total Queue lock hold time by however long the other thread holds `syncResources`.

The `syncResources` lock is held during:
- `lock()` → calls `save()` (disk I/O)
- `unlockResources()` → calls `freeResources()` → `save()` (disk I/O)
- `unreserve()` → calls `save()` (disk I/O)

**Impact:** The Queue lock is transitively extended by disk I/O operations from concurrent unlock/reserve operations on a completely separate thread.

### 2.3 Label Expression Parsing — MODERATE

**Location:** `LockableResource.isValidLabel()` → `Label.parseExpression(candidate)`

When resources are matched by label, `isValidLabel()` is called for every resource. It first checks the simple `labelsContain()` (list `.contains()` — fast). If that misses, it calls `Label.parseExpression(candidate)` which:

1. Parses the label expression string into an AST
2. Creates `LabelAtom` objects for each resource label
3. Evaluates the expression against the atoms

This is repeated for every resource for every label-based queue item per cycle. With simple labels (no expressions), the fast `labelsContain()` path short-circuits. With complex label expressions (e.g. `label1 && label2 || label3`), every resource incurs a parse + evaluate.

### 2.4 `save()` via `tryQueue()` — NOT CALLED

Notably, `tryQueue()` does **not** call `save()`. It only sets `queued` state on resources. This is correct for the Queue lock path — disk I/O is avoided.

### 2.5 `uncacheIfFreeing()` — LIGHTWEIGHT (but called from unlock paths)

Iterates `cachedCandidates.asMap()` entries and invalidates matching entries. O(cache_size × candidates_per_entry). Only called from unlock/unreserve paths, not from `canRun()`.

---

## 3. LockRunListener — Build Lifecycle Events

### `onStarted(Run, TaskListener)` — NOT under Queue lock

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | Matrix build check | String comparison | LIGHTWEIGHT |
| 2 | `LockableResourcesManager.get()` | Singleton lookup | LIGHTWEIGHT |
| 3 | **`synchronized(lrm.syncResources)`** | Acquires plugin lock | **CONTENTION with canRun()** |
| 4 | `Utils.requiredResources(proj)` | Property lookup | LIGHTWEIGHT |
| 5 | `lrm.getResourcesFromProject(proj.getFullName())` | O(resources) scan | MODERATE |
| 6 | `lrm.lock(required, build)` | Sets build, calls `save()` | **DISK I/O** |
| 7 | Parameter variable setup | In-memory | LIGHTWEIGHT |
| 8 | `build.addAction(...)` | In-memory | LIGHTWEIGHT |

**Issue:** Holds `syncResources` during `save()` (disk I/O). If `canRun()` is called concurrently, the Queue lock is blocked waiting for `syncResources` to be released after disk I/O completes.

**Missing:** No `scheduleMaintenance()` call. After locking resources, other items waiting for those resources won't be re-evaluated until the next 5-second timer tick.

### `onCompleted(Run, TaskListener)` — NOT under Queue lock

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | Matrix build check | String comparison | LIGHTWEIGHT |
| 2 | `LockableResourcesManager.get().unlockBuild(build)` | See below | See below |

`unlockBuild()` → `unlockNames()` → `synchronized(syncResources)` → `unlockResources()`:
- `freeResources()` — iterates resources, clears state, calls `uncacheIfFreeing()`, calls `save()` (DISK I/O)
- `proceedNextContext()` loop — iterates `queuedContexts`, checks validity, tries to lock next context
- Final `save()` — DISK I/O

**Missing:** No `scheduleMaintenance()` call. When resources are freed, items waiting in Jenkins Queue for those resources won't know until the next timer tick.

### `onDeleted(Run)` — NOT under Queue lock

Same as `onCompleted()`. **Missing:** No `scheduleMaintenance()` call.

---

## 4. LockStepExecution — Pipeline `lock` Step

### `start()` — NOT under Queue lock, but acquires `syncResources`

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | **`synchronized(LockableResourcesManager.syncResources)`** | Plugin lock | **CONTENTION** |
| 2 | `step.validate(...)` | In-memory | LIGHTWEIGHT |
| 3 | `lrm.createResource(resource.resource)` | May add to list + `save()` | DISK I/O (if new resource) |
| 4 | `lrm.getAvailableResources(...)` | Resource scanning | MODERATE |
| 5 | `lrm.lock(available, run)` | Sets state + `save()` | DISK I/O |
| 6 | `LockStepExecution.proceed(...)` | Sets up body invoker | LIGHTWEIGHT |

If resources aren't available:
| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 7 | `lrm.queueContext(...)` | Adds to `queuedContexts` + `save()` | DISK I/O |

**Missing:** No `scheduleMaintenance()` call when resources are locked or queued.

### `Callback.finished()` — Releases resources when pipeline lock body completes

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | `LockableResourcesManager.get().unlockNames(...)` | `syncResources` → `freeResources()` → `save()` | DISK I/O |

**Missing:** No `scheduleMaintenance()` call. This is a critical miss — when a pipeline lock body completes and resources are freed, items waiting in the Jenkins Queue for those resources won't be dispatched for up to 5 seconds.

### `stop(Throwable)` — Pipeline cancelled/aborted

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | `LockableResourcesManager.get().unqueueContext(getContext())` | `syncResources` → list remove + `save()` | DISK I/O |

**Missing:** No `scheduleMaintenance()` call.

---

## 5. LockableResourcesRootAction — UI Actions

All UI actions (unlock, reserve, steal, reassign, unreserve, reset) call through to `LockableResourcesManager` methods that acquire `syncResources` and call `save()`.

| Action | Method Called | Frees Resources? | scheduleMaintenance()? |
|--------|-------------|-------------------|----------------------|
| Unlock | `unlockResources()` | **Yes** | **No** |
| Reserve | `reserve()` | No (takes resource) | **No** |
| Steal | `steal()` | **Yes** (unlocks + re-reserves) | **No** |
| Reassign | `reassign()` | Partial (re-reserves) | **No** |
| Unreserve | `unreserve()` | **Yes** | **No** |
| Reset | `reset()` | **Yes** | **No** |

Every action that frees resources should trigger `scheduleMaintenance()`.

---

## 6. NodesMirror — ComputerListener

### `onConfigurationChange()` — NOT under Queue lock

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | `isNodeMirrorEnabled()` | System property check | LIGHTWEIGHT |
| 2 | `synchronized(lrm.syncResources)` | Plugin lock | CONTENTION |
| 3 | `Jenkins.get().getNodes()` iteration | In-memory | LIGHTWEIGHT |
| 4 | `mirrorNode()` per node | Label sync, `addResource()` | MODERATE |
| 5 | `deleteNotExistingNodes()` | Resource list iteration | MODERATE |

Only runs when node configuration changes. Not performance-critical for Queue scheduling.

---

## 7. FreeDeadJobs — Jenkins Initializer

### `freePostMortemResources()` — Runs at startup only

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | `synchronized(lrm.syncResources)` | Plugin lock | N/A (startup only) |
| 2 | Iterate resources checking `getBuild().isInProgress()` | O(resources) | MODERATE |
| 3 | `resource.recycle()` for orphans | `unlockResources()` + `unreserve()` | DISK I/O |

**Missing:** No `scheduleMaintenance()` after freeing orphaned resources.

---

## 8. Summary: Queue Lock Impact

### Under Queue Lock — Operations in canRun()

| Operation | Classification | Impact |
|-----------|---------------|--------|
| Project/property lookups | LIGHTWEIGHT | Negligible |
| Parameter extraction | LIGHTWEIGHT | Negligible |
| `synchronized(syncResources)` wait | **CONTENTION** | Blocks Queue lock while waiting for concurrent unlock/save operations |
| `checkCurrentResourcesStatus()` | MODERATE | O(total_resources) per call |
| Groovy script evaluation (cache miss) | **HEAVYWEIGHT** | Full Groovy compilation + sandbox execution per resource |
| Groovy script evaluation (cache hit) | LIGHTWEIGHT | Cache lookup + `retainAll()` |
| Label matching (`isValidLabel`) | MODERATE | Label expression parsing per resource (unless simple label match short-circuits) |
| `cachedCandidates` access | LIGHTWEIGHT | Guava cache get/put |
| Resource state checks | LIGHTWEIGHT | Boolean field reads |
| `setQueued()` | LIGHTWEIGHT | Field sets |

### Contending with Queue Lock — Operations on Other Threads

| Method | Thread | Acquires `syncResources`? | Does I/O? |
|--------|--------|--------------------------|-----------|
| `LockRunListener.onStarted()` | Executor thread | **Yes** | `save()` — disk |
| `LockRunListener.onCompleted()` | Executor thread | **Yes** | `save()` — disk |
| `LockStepExecution.start()` | CPS thread | **Yes** | `save()` — disk |
| `LockStepExecution.Callback.finished()` | CPS thread | **Yes** | `save()` — disk |
| UI actions (unlock, reserve, etc.) | HTTP thread | **Yes** | `save()` — disk |
| `NodesMirror.onConfigurationChange()` | Computer thread | **Yes** | No (but holds lock) |

### Missing `scheduleMaintenance()` Triggers

| Location | Event | Impact |
|----------|-------|--------|
| `unlockResources()` | Resources freed after build completes | Items wait up to 5s for freed resources |
| `unreserve()` | Resources unreserved by user | Items wait up to 5s |
| `reset()` | Resources reset via UI | Items wait up to 5s |
| `recycle()` | Resources recycled | Items wait up to 5s |
| `LockRunListener.onCompleted()` | Build completed, resources unlocked | Items wait up to 5s |
| `LockRunListener.onDeleted()` | Build deleted, resources unlocked | Items wait up to 5s |
| `LockStepExecution.Callback.finished()` | Pipeline lock body completed | Items wait up to 5s |
| `LockStepExecution.stop()` | Pipeline cancelled/aborted | Items wait up to 5s |
| `LockableResourcesRootAction.doUnlock()` | UI unlock | Items wait up to 5s |
| `LockableResourcesRootAction.doUnreserve()` | UI unreserve | Items wait up to 5s |
| `LockableResourcesRootAction.doReset()` | UI reset | Items wait up to 5s |
| `LockableResourcesRootAction.doSteal()` | UI steal (unlocks + re-reserves) | Items wait up to 5s |
| `FreeDeadJobs.freePostMortemResources()` | Post-mortem cleanup at startup | Items wait up to 5s |
| `proceedNextContext()` (inside `unlockResources`) | Pipeline context proceeds after unlock | Items wait up to 5s |
