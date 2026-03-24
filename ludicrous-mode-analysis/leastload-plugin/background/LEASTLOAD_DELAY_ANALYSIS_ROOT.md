# Why Least-Load Plugin Can Cause Long Delays (1+ Minute) or No Scheduling

This document explains how the interaction between Jenkins core `Queue`/`LoadBalancer` and the **leastload-plugin** can produce long delays when selecting nodes by label, and why work sometimes is not scheduled even when executors are available.

---

## 1. When does scheduling actually run?

### Queue maintenance is periodic and serialized

- **`Queue.maintain()`** is the only place that assigns buildable items to executors. It:
  - Builds a snapshot of *parked* (idle) executors.
  - Iterates over **buildables** and, for each item, builds a `MappingWorksheet` from the current `candidates` and calls **`loadBalancer.map(task, ws)`**.
  - If `map()` returns **null**, the item is left in buildables and tried again in a **later** maintenance run.

- **When does `maintain()` run?**
  - **Periodic timer**: `MaintainTask.periodic()` uses `Timer.get().scheduleWithFixedDelay(this, interval, interval, TimeUnit.MILLISECONDS)` with **`hudson.model.Queue.maintainInterval`** (default **5000 ms**).
  - **On events**: `scheduleMaintenance()` is called when items are added/updated (e.g. `scheduleInternal`, queue updates). That only **submits** a run to the maintainer thread; it does not run immediately.

- **Single-threaded maintenance**: The maintainer is an **`AtmostOneTaskExecutor`**. Only one `maintain()` runs at a time. If a run is already in progress, the next one (from the timer or from `scheduleMaintenance()`) is queued and runs **after** the current run finishes.

**Implications:**

- There is at least a **0–5 second** delay between “task becomes buildable” and the next time it is considered (next `maintain()`).
- With **`scheduleWithFixedDelay`**, the next run is scheduled **after** the previous run **completes** plus the interval. So if one `maintain()` takes **60 seconds** (e.g. many items, slow `getCauseOfBlockageForItem` or label/node checks), the next run can start **~65 seconds** later. That easily produces **1+ minute** gaps between scheduling attempts.

---

## 2. Why does the leastload-plugin return null and defer work?

The plugin’s **`LeastLoadBalancer.map()`** can return **null** in two main situations. When it returns null, the task **stays in buildables** and is only retried in the **next** `maintain()` cycle.

### 2.1 Round-robin “available nodes this round” and label mismatch

The plugin keeps a **persistent** set:

```java
private final Set<String> availableNodeNamesThisRound = new HashSet<>();
```

- **When it’s empty**: it **refreshes** from Jenkins (nodes that are online, accepting tasks, and have **at least one idle executor**).
- It only **assigns** to nodes that are in `availableNodeNamesThisRound`.
- After assigning a task to a node, it **removes** that node from the set (**`markNodesUsed(m)`**).
- It **only refreshes again** when:
  - the set is empty at the start of `map()`, or
  - after filtering, **`chunksForThisRound.isEmpty()`**.

So across **all** `map()` calls (all buildable items, across all `maintain()` cycles), the set is **not** reset at the start of each cycle. It only gets repopulated when empty or when no chunk is available for the current task.

**Scenario that causes long delay or “no scheduling”:**

- Suppose only **one node (e.g. A)** has the **label** required by the task.
- In an earlier `map()` call (same or previous `maintain()`), the plugin assigned some work to **A** and **marked A as used**.
- For the **next** task that also needs label A:
  - **`useableChunks`** = chunks for node A (only A matches the label).
  - **`chunksForThisRound`** = `filterToAvailableNodesThisRound(useableChunks)` = **empty**, because **A is no longer in `availableNodeNamesThisRound`**.
  - Plugin then **refreshes**: `refreshAvailableNodes()` only adds nodes with **`countIdle() > 0`**. If **A’s executor was just assigned** and is busy, **A is not added**.
  - So **`chunksForThisRound`** stays empty → **`map()` returns null** → task remains in buildables.

So the task is **not** scheduled until a **future** `maintain()` run when:
- It is considered again in the buildables order, and
- When the plugin refreshes, node A has an idle executor again (e.g. previous work finished).

With a **5 second** (or longer) interval between `maintain()` runs, this easily produces **10–60+ second** delays. If the single node with that label is often busy or the buildables order is unfavorable, the same task can be skipped for many cycles and appear as “doesn’t schedule at all” for a long time.

### 2.2 assignGreedily fails

Even when `chunksForThisRound` is non-empty, **`assignGreedily(m, chunksForThisRound, 0)`** can fail (e.g. constraints, capacity, or ordering). Then the plugin returns **null** and the item stays in buildables for the next cycle.

---

## 3. How label selection affects the worksheet

- **Candidates** passed to `MappingWorksheet` are built in **`Queue.maintain()`** by filtering **parked** executors with **`JobOffer.getCauseOfBlockage(p)`**. That uses **`node.canTake(item)`**, which checks **label** (and other constraints). So the worksheet only contains executors on nodes that **can** take the task (e.g. match the assigned label).
- **`MappingWorksheet`** then groups those offers by computer and applies load prediction; **`WorkChunk.applicableExecutorChunks()`** further filters by **`assignedLabel`** and capacity.

So for a **label-restricted** task:

- If only one node has that label, the worksheet has only that node’s chunks.
- The leastload plugin’s **round-robin** logic then restricts to **“available this round”**. If that node was already marked used and hasn’t been refreshed (e.g. because it has no idle executor after the last assignment), **chunksForThisRound** is empty and **map() returns null**.

So **label selection** doesn’t add delay by itself; the combination of **single-node-per-label** and **“available nodes this round”** plus **refresh only when idle** causes the plugin to frequently return null for that task until the next cycle when the node is idle again.

---

## 4. Summary: why delays can be “over one minute” and why work sometimes never runs

| Cause | Effect |
|-------|--------|
| **Default `maintainInterval` = 5 s** | At least 0–5 s between scheduling attempts for a buildable item. |
| **`scheduleWithFixedDelay` + slow `maintain()`** | If one `maintain()` run is slow (e.g. many items, slow `getCauseOfBlockageForItem`, label/node logic), the **next** run starts only after the previous one **finishes** plus the interval. One long run (e.g. 60 s) → next run ~65 s later → **1+ minute** between attempts. |
| **Single maintainer thread** | `scheduleMaintenance()` does not run immediately; it queues behind the current `maintain()`. So “event-driven” maintenance still waits for the current run to finish. |
| **Leastload returns null** | Task stays in buildables and is only retried in the **next** `maintain()`. So each null multiplies delay by one full cycle (5 s or more). |
| **Round-robin + single node for label** | Node A (only one with the label) is marked “used”; refresh doesn’t add A back because it has no idle executor; **chunksForThisRound** is empty → **null** for every task that needs that label until A is idle again in a later cycle. |
| **No refresh between cycles for “used” nodes** | `availableNodeNamesThisRound` is only cleared/refreshed when empty or when no chunk is available. So a task that needs a node that was used in a previous assignment can keep getting null until a refresh happens and that node has idle executors again. |

Together, these explain:

- **Long delays**: 5 s per cycle × many cycles where the plugin returns null (round-robin + label), plus possible long `maintain()` runs extending the actual period between runs.
- **Work not scheduled despite free executors**: Executors may be on nodes that are either (a) not in `availableNodeNamesThisRound` (already used this “round”) or (b) not matching the task’s label. After refresh, nodes with no idle executors are excluded, so the only node with the right label might not reappear until it becomes idle in a later cycle—so the task can sit in buildables for a long time or appear to “never” run.

---

## 5. References in code

- **Queue maintenance and interval**: `jenkins/core/src/main/java/hudson/model/Queue.java`  
  - `maintain()` (around 1491), `scheduleMaintenance()` (1200), `MaintainTask.periodic()` (2921–2924), `maintainInterval` default 5000L.
- **Single-threaded maintainer**: `Queue` line 333 (`AtmostOneTaskExecutor`), `scheduleMaintenance()` → `maintainerThread.submit()`.
- **Buildables loop and null handling**: `Queue.maintain()` around 1622–1681 (buildables loop, `loadBalancer.map()`, `continue` when `m == null`).
- **Leastload round-robin and refresh**: `leastload-plugin/.../LeastLoadBalancer.java`  
  - `availableNodeNamesThisRound` (84–85), refresh when empty (112–114), filter (118), refresh when chunksForThisRound empty (119–123), return null (125–126, 134–137), `refreshAvailableNodes()` (184–186: `countIdle() <= 0` skips node), `markNodesUsed()` (211–218).

---

## 6. Why a 15–20 second delay can still occur (e.g. x86_64_medium with plenty of capacity)

Even with per-label tracking and immediate-assign, you can still see 15–20 s between “task becomes buildable” and “task is assigned” when capacity exists. The main reasons:

### 6.1 `scheduleWithFixedDelay` and slow `maintain()`

The timer uses **fixed delay**, not fixed rate:

- **Next run** is scheduled **5 s after the previous run finishes** (default `maintainInterval`).
- So **effective period = duration of `maintain()` + 5 s**.

If one `maintain()` run takes **10–15 s** (see below), the next run starts **15–20 s** after the start of the previous one. A task that becomes buildable right after a run starts may not be considered until that next run → **15–20 s delay** even with free executors.

### 6.2 What makes `maintain()` slow

Inside each `maintain()` run, for **every** buildable item the Queue:

1. Calls **`getCauseOfBlockageForItem(p)`** (can touch labels, snapshot, etc.).
2. Builds **candidates**: for each idle executor (or per node, cached), calls **`JobOffer.getCauseOfBlockage(p)`** → **`node.canTake(item)`** (label and permission checks).
3. Constructs **`MappingWorksheet(p, candidates, loadBalancer.getLoadPredictors())`**.
4. Calls **`loadBalancer.map(p.task, ws)`**.

With **many buildables** (e.g. 20–40) and **many nodes** (e.g. 38), the cost is:

- **Buildables × (getCauseOfBlockageForItem + nodes × getCauseOfBlockage + worksheet + map)**.

If each buildable costs ~0.5–1 s (label lookups, `getNodes()`, etc.), 20–30 items can make a single `maintain()` run **10–20 s**. So the **next** run is 15–25 s later → you see a 15–20 s gap even when the balancer would assign quickly for x86_64_medium.

### 6.3 Position in the buildables list

Buildables are processed **in order**. If your x86_64_medium task is e.g. 25th and each item takes ~0.5 s, the Queue reaches it only after **~12 s** in that run. So delay can be “one long run” rather than “several 5 s cycles”.

### 6.4 What to do: reduce the maintenance interval

To bring the worst-case delay down toward **~6 s or less**:

- Set the **queue maintenance interval** to **2–3 s** so the delay after each run is 2–3 s instead of 5 s:
  - **System property**: `hudson.model.Queue.maintainInterval`
  - **Default**: 5000 (ms)
  - **Example**: `-Dhudson.model.Queue.maintainInterval=2000` (2 s) or `3000` (3 s)

Then:

- **Effective period** = duration of `maintain()` + 2–3 s.
- If `maintain()` is 10 s, you still get a run every **12–13 s** instead of 15 s; if it’s 5 s, you get a run every **7–8 s** instead of 10 s.

To go toward “instant or within ~6 s”, combine:

1. **Lower `maintainInterval`** (e.g. 2000–3000 ms).
2. **Reduce cost of `maintain()`** (fewer buildables, or faster label/canTake logic and snapshots) so each run is shorter and the next run starts sooner.
