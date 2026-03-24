# Job-Restrictions Plugin — Queue Performance Patches

Patches against `origin/master` (`jenkinsci/job-restrictions-plugin`). 7 files changed, +141 / -47 lines.

---

## How This Helps Unblock the Queue

During `Queue.maintain()`, Jenkins calls `Node.canTake(item)` for every candidate executor on every buildable item. For each node with job restrictions configured, this invokes `JobRestrictionProperty.canTake()` which evaluates regex patterns, class-name checks, and other restriction logic. With hundreds of buildable items and hundreds of nodes, this runs tens of thousands of times per maintenance cycle.

Node properties are a potential performance bottleneck in the Queue maintenance pipeline because they sit in the innermost loop — O(buildables × executors × nodeProperties). The patch introduces a TTL cache so that repeated `canTake()` calls for the same (node, item) pair return instantly from memory instead of re-evaluating restrictions. This matters most in combination with the [EC2 plugin](../ec2-plugin/) patches (which ensure the Queue lock is held for microseconds during retention checks) and the [leastload plugin](../leastload-plugin/) fixes (which ensure work is assigned on the first attempt). Together, the entire Queue maintenance cycle completes in milliseconds because every plugin in the pipeline returns fast.

The patch also fixes a critical silent bug: after XStream deserialization, the cache's `ConcurrentHashMap` was `null` (XStream bypasses constructors and field initializers). The resulting `NullPointerException` was silently caught by Jenkins core's `Node.canTake()` and treated as "block this item" — permanently preventing all items from being dispatched to any node with job restrictions. This was the root cause of the original "Waiting for next available executor on built-in" hang that started the entire investigation.

---

## Files Changed (src/main)

### JobRestrictionProperty.java — TTL Cache and XStream Fix

The main patch. Adds a cache layer around restriction evaluation.

- **`canTake()`** — Computes a composite cache key, checks the `ConcurrentHashMap` for a fresh result (within TTL), and returns on hit. On miss, delegates to `computeCanTake()`, caches "allow" results (null `CauseOfBlockage`), and returns. "Block" results are never cached to avoid stale blockages — if a restriction configuration changes or a pipeline flyweight task reuses a queue item ID, the block will be re-evaluated fresh.
- **`getCache()`** — Double-checked locking lazy initializer. The `ConcurrentHashMap` is `transient volatile` so XStream deserialization leaves it `null` (transient) and `getCache()` initializes it on first access (volatile + synchronized). This is the fix for the silent NPE bug.
- **`cacheKey()`** — Composite key `itemId + "|" + taskFullDisplayName`. The task name is included because pipeline flyweight tasks can reuse queue item IDs after a blocked item leaves — without the name, a cached "allow" from a previous item could be returned for a different task.
- **`evictIfNeeded()`** — When the cache exceeds `CACHE_MAX_ENTRIES`, evicts expired entries first (older than TTL), then removes entries until the cache is at half capacity.
- **`CachedResult`** — Inner class holding the nullable `CauseOfBlockage` and timestamp.

### RegexNameRestriction.java / JobClassNameRestriction.java

Extracted intermediate values to local variables for debuggability. No behavioral change.

---

## Files Changed (src/test)

### UserIdCauseRestrictionTest.java / JobClassNameRestrictionTest.java

Downgraded from JUnit 5 (`@WithJenkins`, `@BeforeEach`, `jupiter.api.Test`) to JUnit 4 (`@Rule JenkinsRule`, `@Before`, `org.junit.Test`) for compatibility with the plugin's Jenkins baseline. `ACL.as2()` replaced with `ACL.impersonate()`. Diamond inference replaced with explicit type arguments.

### JobRestrictionPropertyBuider.java / JobRestrictionTestHelper.java

Diamond inference replaced with explicit type arguments for Java compatibility.

---

## Configuration

| System Property | Default | Description |
|-----------------|---------|-------------|
| `JobRestrictionProperty.cacheDisabled` | false | Disable canTake cache entirely |
| `JobRestrictionProperty.cacheTtlMs` | 30000 | Cache TTL in milliseconds |
| `JobRestrictionProperty.cacheMaxEntries` | 500 | Max cache entries per node property instance |

---

## Background Analysis

- [QUEUE_PERFORMANCE.md](background/QUEUE_PERFORMANCE.md) — Cache design, XStream transient fix, Jenkins core Queue context
