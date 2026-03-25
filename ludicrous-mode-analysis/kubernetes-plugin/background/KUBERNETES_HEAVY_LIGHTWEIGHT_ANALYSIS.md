# Kubernetes Plugin — Heavy vs Lightweight Classification

Complete classification of all operations that affect Queue and provisioner performance.

---

## Classification Key

| Category | Definition | Example |
|----------|-----------|---------|
| **LIGHTWEIGHT** | < 1ms, in-memory only, no I/O | Field read, instanceof, map lookup |
| **MODERATE** | 1–50ms, may do minor I/O or iteration | Node list iteration, synchronized counter |
| **HEAVYWEIGHT** | 50ms+, network I/O, disk I/O, or blocking | K8s API call, pod creation, waiting |
| **CONTENTION** | Holds lock that can block critical threads | `Queue.withLock()`, `synchronized` |

---

## Queue Lock Paths (Called During `Queue.maintain()`)

### KubernetesQueueTaskDispatcher.canTake()

```
canTake(Node, BuildableItem)
├── node instanceof KubernetesSlave          LIGHTWEIGHT (instanceof)
├── item.task.getOwnerTask()                 LIGHTWEIGHT (in-memory)
├── KubernetesFolderProperty.isAllowed()
│   ├── agent.getKubernetesCloud()           LIGHTWEIGHT (Jenkins.get().getCloud())
│   ├── targetCloud.isUsageRestricted()      LIGHTWEIGHT (boolean field)
│   ├── collectAllowedClouds(parent)         LIGHTWEIGHT (folder traversal ≤10 levels)
│   └── allowedClouds.contains(name)         LIGHTWEIGHT (Set.contains)
└── return null / new CauseOfBlockage        LIGHTWEIGHT
```

**Total: ~0.01ms — LIGHTWEIGHT. No changes needed.**

Runs O(kubernetes_nodes × buildable_items) times per `maintain()` cycle. Cost is negligible because all operations are in-memory field reads and collection lookups.

---

### NoDelayProvisionerStrategy.FastProvisioning.onEnterBuildable()

```
onEnterBuildable(BuildableItem)
├── DISABLE_NODELAY_PROVISING check          LIGHTWEIGHT (boolean)
├── Jenkins.get().clouds iteration           LIGHTWEIGHT (list iteration)
├── cloud instanceof KubernetesCloud         LIGHTWEIGHT (instanceof)
├── cloud.canProvision(CloudState)
│   └── getTemplate(label)
│       └── PodTemplateUtils.getTemplateByLabel()
│           └── iterate templates, label match   LIGHTWEIGHT (O(templates))
└── provisioner.suggestReviewNow()           LIGHTWEIGHT (async trigger)
```

**Total: ~0.1ms — LIGHTWEIGHT. No changes needed.**

---

## Provisioner Thread Paths (Called from NodeProvisioner, NOT under Queue lock)

### NoDelayProvisionerStrategy.apply()

```
apply(StrategyState)
├── strategyState.getSnapshot()              LIGHTWEIGHT (cached snapshot)
├── Jenkins.get().clouds iteration           LIGHTWEIGHT
├── cloud.canProvision(cloudState)           LIGHTWEIGHT (template matching)
├── cloud.provision(cloudState, workload)    LIGHTWEIGHT (see below)
├── fireOnStarted(cloud, label, planned)     LIGHTWEIGHT (listener notification)
├── strategyState.recordPendingLaunches()    LIGHTWEIGHT
└── Timer schedule suggestReviewNow          LIGHTWEIGHT
```

**Total: ~0.5ms — LIGHTWEIGHT. No changes needed.**

### KubernetesCloud.provision()

This is the critical finding that differentiates kubernetes-plugin from EC2 and Azure plugins:

```
provision(CloudState, int)
├── new LimitRegistrationResults(this)       LIGHTWEIGHT
├── state.getLabel()                         LIGHTWEIGHT
├── InProvisioning.getAllInProvisioning(label)
│   └── label.getNodes().stream().filter()   MODERATE (O(label_nodes))
├── getTemplatesFor(label)
│   └── PodTemplateFilter.applyAll()         LIGHTWEIGHT (O(templates))
├── loop: toBeProvisioned > 0
│   ├── getUnwrappedTemplate(podTemplate)    LIGHTWEIGHT (template merge)
│   ├── limitRegistrationResults.register()
│   │   └── KubernetesProvisioningLimits
│   │       ├── initInstance() (first call)  CONTENTION (Queue.withLock, one-time)
│   │       └── synchronized increment       LIGHTWEIGHT
│   └── PlannedNodeBuilderFactory.build()
│       └── StandardPlannedNodeBuilder.build()
│           ├── KubernetesSlave.builder().build()  LIGHTWEIGHT (in-memory)
│           └── CompletableFuture.completedFuture  LIGHTWEIGHT (already complete!)
├── Metrics counter increment                LIGHTWEIGHT
└── return plannedNodes                      LIGHTWEIGHT
```

**Total: ~1ms — LIGHTWEIGHT. No Kubernetes API calls!**

The `PlannedNode` future is **already completed** at return time. This means:
1. **No provisioner thread blocking** (unlike EC2 and Azure plugins)
2. The actual pod creation happens later in `KubernetesLauncher.launch()` on `Computer.threadPoolForRemoting`
3. The provisioner can immediately proceed to the next round

### KubernetesProvisioningLimits.register()

```
register(cloud, podTemplate, numExecutors)
├── initInstance() (first call only)
│   └── Queue.withLock()                     CONTENTION (one-time)
│       └── Jenkins.get().getNodes().stream()
│           └── count KubernetesSlaves       MODERATE (O(nodes))
├── synchronized(this)
│   ├── getGlobalCount() + compare           LIGHTWEIGHT (ConcurrentMap.get)
│   ├── getPodTemplateCount() + compare      LIGHTWEIGHT (ConcurrentMap.get)
│   └── cloudCounts.put() / podTemplateCounts.put()  LIGHTWEIGHT
└── return boolean                           LIGHTWEIGHT
```

**First call: CONTENTION (Queue.withLock) — but one-time initialization.**
**Subsequent calls: LIGHTWEIGHT (~0.01ms).**

Proposed fix: The first-call `Queue.withLock()` could be avoided by initializing counts from a `ComputerListener.onOnline()` or `@Initializer` callback instead, but the impact is minimal since it only happens once per Jenkins lifecycle.

---

## Remoting Pool Paths (Background Threads — No Queue Lock Concern)

### KubernetesLauncher.launch()

```
launch(SlaveComputer, TaskListener)
├── Reaper.getInstance().maybeActivate()     MODERATE (one-time activation)
├── node.getTemplate()                       LIGHTWEIGHT
├── cloud.connect()                          HEAVYWEIGHT (K8s client creation, cached)
├── template.build(node)                     MODERATE (pod spec construction)
├── cloud.registerPodInformer(node)          HEAVYWEIGHT (K8s watch setup per namespace)
├── client.pods().get() [check existing]     HEAVYWEIGHT (K8s API)
├── client.pods().create(pod)                HEAVYWEIGHT (K8s API)
├── volume.createVolume(client, metadata)    HEAVYWEIGHT (PVC creation)
├── client.pods().waitUntilReady(timeout)    HEAVYWEIGHT (blocking, up to 600s)
├── poll loop: isOnline() + sleep(1000)      HEAVYWEIGHT (blocking, up to 600s)
├── computer.setAcceptingTasks(true)         LIGHTWEIGHT
└── node.save()                              MODERATE (disk I/O)
```

**Total: seconds to minutes. Acceptable — runs on remoting thread pool.**

**MISSING scheduleMaintenance():** After `setAcceptingTasks(true)` and `launched = true`, the agent is ready to accept work but the Queue is not notified. The Queue will only notice on the next periodic `maintain()` cycle (up to 5 seconds later).

### KubernetesSlave._terminate()

```
_terminate(TaskListener)
├── getKubernetesCloud()                     LIGHTWEIGHT
├── cloud.connect()                          HEAVYWEIGHT (K8s client)
├── getPodRetention().shouldDeletePod()
│   └── KubernetesCloud.getPodResource().get()  HEAVYWEIGHT (K8s API)
├── computer.getChannel()                    LIGHTWEIGHT
├── ch.callAsync(SlaveDisconnector)          HEAVYWEIGHT (remote call, 5s timeout)
├── deleteSlavePod(listener, client)
│   └── client.pods().delete()               HEAVYWEIGHT (K8s API)
└── listener.getLogger().println()           LIGHTWEIGHT
```

**Total: 1–10 seconds. Acceptable — runs from background thread.**

**MISSING scheduleMaintenance():** After pod deletion and agent removal, Queue not notified.

---

## Reaper Event Handlers (Pod Watcher Callbacks — Background Thread)

### RemoveAgentOnPodDeleted.onEvent()

```
onEvent(DELETED, node, pod, reasons)
├── Jenkins.get().removeNode(node)           MODERATE (node list modification)
└── disconnectComputer(node, cause)          MODERATE
```

**MISSING scheduleMaintenance()** after removing the node. When a pod is deleted externally (e.g., `kubectl delete pod`), the Queue should be notified immediately so it can:
1. Re-provision a replacement pod
2. Re-evaluate buildable items that were assigned to this agent

### TerminateAgentOnContainerTerminated.onEvent()

```
onEvent(MODIFIED, node, pod, reasons)
├── PodUtils.getTerminatedContainers(pod)    LIGHTWEIGHT
├── logAndCleanUp(...)
│   ├── PodUtils.logLastLines(pod, client)   HEAVYWEIGHT (K8s API for logs)
│   ├── node.terminate()                     HEAVYWEIGHT (pod deletion)
│   ├── PodUtils.cancelQueueItemFor(pod)     MODERATE
│   └── disconnectComputer(node, cause)      MODERATE
```

**MISSING scheduleMaintenance()** after termination.

### TerminateAgentOnPodFailed.onEvent()

Same as above. **MISSING scheduleMaintenance()**.

### TerminateAgentOnImagePullBackOff.onEvent()

```
onEvent(MODIFIED, node, pod, reasons)
├── PodUtils.getContainers(pod, filter)      LIGHTWEIGHT
├── ttlCache.get(podUid, ...)                LIGHTWEIGHT
├── node.terminate() (if limit reached)      HEAVYWEIGHT
└── disconnectComputer(...)                  MODERATE
```

**MISSING scheduleMaintenance()** after termination.

---

## GarbageCollection (AsyncPeriodicWork — Background Thread)

### PeriodicGarbageCollection.execute()

```
execute(TaskListener)
├── annotateLiveAgents(listener)
│   └── for each KubernetesComputer
│       └── kc.annotateTtl(listener)
│           └── cloud.getPodResource().patch()   HEAVYWEIGHT (K8s API per pod)
└── garbageCollect()
    └── for each KubernetesCloud
        └── client.pods().list()                 HEAVYWEIGHT (K8s API)
            └── filter by annotation age
                └── client.resource(pod).delete() HEAVYWEIGHT (K8s API per orphan)
```

**Runs periodically (every 60s). Acceptable — runs on AsyncPeriodicWork thread.**

**MISSING scheduleMaintenance()** after deleting orphan pods.

---

## Summary of Proposed Changes

### Category 1: Missing scheduleMaintenance() Triggers (HIGH PRIORITY)

These are the critical missing pieces. Adding `Jenkins.get().getQueue().scheduleMaintenance()` calls at each of these points will ensure the Queue re-evaluates immediately after capacity changes:

| Location | Event | Proposed Trigger |
|----------|-------|-----------------|
| `KubernetesLauncher.launch()` — after `setAcceptingTasks(true)` | Agent comes online | `scheduleMaintenance()` |
| `Reaper.RemoveAgentOnPodDeleted.onEvent()` — after `removeNode()` | Pod deleted externally | `scheduleMaintenance()` |
| `Reaper.TerminateAgentOnContainerTerminated.onEvent()` — after `terminate()` | Container terminated | `scheduleMaintenance()` |
| `Reaper.TerminateAgentOnPodFailed.onEvent()` — after `terminate()` | Pod failed | `scheduleMaintenance()` |
| `Reaper.TerminateAgentOnImagePullBackOff.onEvent()` — after `terminate()` | Image pull backoff | `scheduleMaintenance()` |
| `KubernetesSlave._terminate()` — after `deleteSlavePod()` | Agent terminated | `scheduleMaintenance()` |
| `GarbageCollection.garbageCollect()` — after deleting orphans | Orphan cleanup | `scheduleMaintenance()` |

Alternatively, a new `ComputerListener` extension can be added to centralize these triggers (similar to what was done for the ssh-agents-plugin):

| ComputerListener Method | When Called | Covers |
|-------------------------|------------|--------|
| `onOnline(Computer, TaskListener)` | Agent comes online | Launch completion |
| `onOffline(Computer, OfflineCause)` | Agent goes offline | Pod deletion, failure |
| `onTemporarilyOnline(Computer)` | Agent temporarily online | State changes |
| `onTemporarilyOffline(Computer, OfflineCause)` | Agent temporarily offline | State changes |

### Category 2: No Code Changes Needed for Queue Lock Paths

All code paths under the Queue lock are already lightweight:
- `KubernetesQueueTaskDispatcher.canTake()` — pure in-memory
- `NoDelayProvisionerStrategy.FastProvisioning.onEnterBuildable()` — pure in-memory + async trigger
- `RetentionStrategy.check()` — delegates to OnceRetentionStrategy/CloudRetentionStrategy (both lightweight)

### Category 3: No Code Changes Needed for Provisioner Thread

- `KubernetesCloud.provision()` is already lightweight — no API calls, returns immediately-completed futures
- `KubernetesCloud.canProvision()` is already lightweight — pure template matching
- `NoDelayProvisionerStrategy.apply()` is already lightweight

### Category 4: Minor Improvement Opportunity

| Issue | Location | Impact |
|-------|----------|--------|
| `KubernetesProvisioningLimits.initInstance()` acquires `Queue.withLock()` | First `register()` call | LOW — one-time cost |
