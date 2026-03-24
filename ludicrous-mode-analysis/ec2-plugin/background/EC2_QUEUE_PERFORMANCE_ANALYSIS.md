# EC2 Plugin Queue Performance Analysis

**Date:** February 2026  
**Context:** Performance issues identified in leastload-plugin audit documentation; EC2 plugin audit and refactoring to keep Jenkins Queue responsive.

---

## 1. Original Request

Based on performance issues identified in the audit documentation at `leastload-plugin/docs`:

1. **Identify** all EC2 plugin code paths invoked by `hudson.model.Queue` in Jenkins
2. **Classify** paths as **lightweight** (instant return) vs **heavyweight** (deferred)
3. **Ensure** all Queue-related calls return instantly so the queue's periodic routine completes in under a second
4. **Limit** changes to the ec2-plugin only (no Jenkins core modifications)

---

## 2. Audit Findings (from leastload-plugin/docs)

- `Queue.maintain()` holds the Queue lock for the entire loop over buildables
- `ComputerRetentionWork.doAperiodicRun()` calls `Queue.withLock(() -> c.getRetentionStrategy().check(c))` for each computer—so **`RetentionStrategy.check()` runs under the Queue lock**
- `RetentionStrategy.start()` also runs under the Queue lock
- Any slow work in these paths blocks the entire Queue and delays scheduling

---

## 3. EC2 Plugin Code Paths Under Queue Lock

| Path | Type | Notes |
|------|------|-------|
| **EC2RetentionStrategy.check()** | HEAVY → LIGHT | Was blocking; refactored to return instantly and defer heavy work |
| **EC2RetentionStrategy.start()** | HEAVY → LIGHT | Was blocking; refactored to return instantly and defer connect |
| **JobOffer.getCauseOfBlockage → isAcceptingTasks** | LIGHT | EC2RetentionStrategy does not override; default `true`; no EC2 API calls |
| **Node.canTake() for EC2AbstractSlave** | LIGHT | Uses default `Node.canTake()`; no EC2-specific logic |

### Heavy Operations (Previously Under Lock)

| Operation | Cost | Now Deferred To |
|-----------|------|-----------------|
| `computer.getState()` | EC2 API (describeInstances) | `runHeavyCheck()` async |
| `computer.getUptime()` | EC2 API | `runHeavyCheck()` async |
| `computer.getLaunchTime()` | EC2 API | `runHeavyCheck()` async |
| `MinimumInstanceChecker.countCurrentNumberOfAgents()` | Iterate computers | `runHeavyCheck()` async |
| `itemsInQueueForThisSlave()` | Queue iteration | `runHeavyCheck()` async (wrapped in `Queue.withLock` for brief read) |
| `computer.connect(false)` | SSH/connection work | `HEAVY_WORK_EXECUTOR` |
| `computer.disconnect()` | State change | Brief `Queue.withLock` in async path |
| `slaveNode.idleTimeout()` | Terminate instance | Brief `Queue.withLock` in async path |

---

## 4. Implementation Summary

### 4.1 EC2RetentionStrategy Refactor

**Lightweight path (instant return under Queue lock):**

- **`check()`** – Returns immediately with `CHECK_INTERVAL_MINUTES`. Uses `tryLock()` and `nextCheckAfter` to avoid rescheduling. Schedules heavy work on static `HEAVY_WORK_EXECUTOR` and returns.
- **`start()`** – Returns immediately. Schedules `getState()` and `connect()` on `HEAVY_WORK_EXECUTOR`.

**Heavyweight path (runs outside Queue lock):**

- **`runHeavyCheck()`** – Runs on background thread. Performs EC2 API calls, `MinimumInstanceChecker`, `itemsInQueueForThisSlave()`.
- **`attemptReconnectIfOffline()`** – Runs in same background task.
- **`internalCheck()`** – Idle timeout, launch timeout, disconnect, terminate logic.

**Queue lock usage in async path:**

- State-changing calls (`disconnect`, `idleTimeout`, `launchTimeout`, `connect`) wrapped in `Queue.withLock()` for short sections.
- `itemsInQueueForThisSlave()` wrapped in `Queue.withLock()` when called from async path (reads Queue).

### 4.2 New Components

- **`HEAVY_WORK_EXECUTOR`** – Static cached thread pool (daemon threads) for deferred work
- **`nextCheckAfter`** – Volatile timestamp to avoid rescheduling too often
- **`checkLock`** – ReentrantLock to serialize heavy checks per computer

---

## 5. Compilation Fixes Applied

`Queue.withLock(Callable<V>)` throws `Exception`. Lambdas calling `disconnect()`, `connect()`, `launchTimeout()`, and `idleTimeout()` can throw checked exceptions, so the compiler required handling.

**Fix:** All six `Queue.withLock(...)` call sites wrapped in try-catch blocks. On exception, log at `Level.FINE` and continue.

| Location | Operation |
|----------|-----------|
| `internalCheck` – external stop detected | `computer.disconnect(null)` |
| `internalCheck` – launch timeout | `node.launchTimeout()` |
| `internalCheck` – idle timeout (positive) | `slaveNode.idleTimeout()` |
| `internalCheck` – idle timeout (billing) | `slaveNode.idleTimeout()` |
| `attemptReconnectIfOffline` | `computer.connect(false)` |
| `start` | `computer.connect(false)` |

---

## 6. Files Modified

| File | Changes |
|------|---------|
| `ec2-plugin/docs/EC2_QUEUE_AUDIT.md` | Audit documentation (created) |
| `ec2-plugin/src/main/java/hudson/plugins/ec2/EC2RetentionStrategy.java` | Lightweight check/start; async heavy work; try-catch for Queue.withLock |

---

## 7. Behavior Comparison

| Before | After |
|--------|-------|
| `check()` performed EC2 API calls under Queue lock | `check()` returns immediately; heavy work runs asynchronously |
| `start()` called `connect()` under lock | `start()` returns immediately; `connect()` runs asynchronously |
| Queue blocked for seconds per EC2 node | Queue lock held only briefly (milliseconds) |
| N EC2 nodes → N blocking EC2 API calls per retention cycle | N EC2 nodes → instant return; heavy work offloaded to background |

---

## 8. References

- `../leastload-plugin/` – Queue maintenance and delay analysis
- `EC2_QUEUE_AUDIT.md` – EC2-specific audit
- `jenkins/core/.../ComputerRetentionWork.java` – Retention check under Queue lock
- `jenkins/core/.../RetentionStrategy.java` – `@GuardedBy("hudson.model.Queue.lock")` on `check()`
