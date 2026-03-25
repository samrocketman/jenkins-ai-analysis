# Kubernetes Plugin — Queue Lock & Provisioner Code Path Audit

Audit of every code path in the kubernetes-plugin that runs under the Jenkins Queue lock or blocks the NodeProvisioner thread.

---

## How Jenkins Invokes Plugin Code

The kubernetes-plugin implements `Cloud` (via `KubernetesCloud`), a `QueueTaskDispatcher` (via `KubernetesQueueTaskDispatcher`), a `NodeProvisioner.Strategy` (via `NoDelayProvisionerStrategy`), a `ComputerLauncher` (via `KubernetesLauncher`), and a `ComputerListener` (via `Reaper`). It does **not** implement a custom `RetentionStrategy` — it uses `OnceRetentionStrategy` (from durable-task plugin) or `CloudRetentionStrategy` (from Jenkins core).

| Extension Point | Class | Runs Under Queue Lock? |
|----------------|-------|----------------------|
| `Cloud.canProvision(CloudState)` | `KubernetesCloud` | **No** — called from NodeProvisioner thread |
| `Cloud.provision(CloudState, int)` | `KubernetesCloud` | **No** — called from NodeProvisioner thread |
| `NodeProvisioner.Strategy.apply()` | `NoDelayProvisionerStrategy` | **No** — NodeProvisioner thread |
| `QueueTaskDispatcher.canTake(Node, BuildableItem)` | `KubernetesQueueTaskDispatcher` | **Yes** — called for every (node, item) pair during `Queue.maintain()` |
| `QueueListener.onEnterBuildable(BuildableItem)` | `NoDelayProvisionerStrategy.FastProvisioning` | **Yes** — called under Queue lock when item becomes buildable |
| `ComputerListener.preLaunch()` | `Reaper` | **No** — called from launcher thread |
| `ComputerLauncher.launch()` | `KubernetesLauncher` | **No** — called from `Computer.threadPoolForRemoting` |
| `RetentionStrategy.check()` | `OnceRetentionStrategy` / `CloudRetentionStrategy` | **Yes** — called under Queue lock by `ComputerRetentionWork` |
| `NodeListener.onDeleted()` | `KubernetesProvisioningLimits.NodeListenerImpl` | **No** — called outside Queue lock |

---

## 1. KubernetesQueueTaskDispatcher.canTake() — UNDER QUEUE LOCK

**This is the only plugin-specific code path under the Queue lock** (other than core's retention strategies).

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | `node instanceof KubernetesSlave` | instanceof check | LIGHTWEIGHT |
| 2 | `item.task.getOwnerTask()` | In-memory task resolution | LIGHTWEIGHT |
| 3 | `KubernetesFolderProperty.isAllowed(slave, job)` | See below | LIGHTWEIGHT |
| 4 | Return null or `KubernetesCloudNotAllowed` | Object creation | LIGHTWEIGHT |

### `KubernetesFolderProperty.isAllowed()`:

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | `agent.getKubernetesCloud()` | `Jenkins.get().getCloud(name)` — in-memory | LIGHTWEIGHT |
| 2 | `targetCloud.isUsageRestricted()` | Boolean field read | LIGHTWEIGHT |
| 3 | `collectAllowedClouds(allowedClouds, parent)` | Recursive folder property traversal | LIGHTWEIGHT (in-memory) |
| 4 | `allowedClouds.contains(targetCloud.name)` | Set lookup | LIGHTWEIGHT |

**Verdict: LIGHTWEIGHT** — Pure in-memory checks. No network I/O, no Kubernetes API calls, no expensive computation. The folder traversal is bounded by folder nesting depth (typically < 10 levels).

---

## 2. NoDelayProvisionerStrategy.FastProvisioning.onEnterBuildable() — UNDER QUEUE LOCK

Called as a `QueueListener` when items become buildable, under the Queue lock.

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | `DISABLE_NODELAY_PROVISING` check | Boolean constant | LIGHTWEIGHT |
| 2 | `Jenkins.get().clouds` iteration | In-memory list | LIGHTWEIGHT |
| 3 | `cloud instanceof KubernetesCloud` | instanceof check | LIGHTWEIGHT |
| 4 | `cloud.canProvision(new Cloud.CloudState(label, 0))` | See below | LIGHTWEIGHT |
| 5 | `provisioner.suggestReviewNow()` | Async trigger to NodeProvisioner | LIGHTWEIGHT |

`canProvision()` calls `getTemplate(label)` → `PodTemplateUtils.getTemplateByLabel(label, getAllTemplates())` which iterates templates in-memory doing label matching.

**Verdict: LIGHTWEIGHT** — Only in-memory operations and an async trigger. Does NOT call `provision()`.

---

## 3. RetentionStrategy.check() — UNDER QUEUE LOCK

The kubernetes-plugin does **not** implement a custom `RetentionStrategy`. Agents use either:

- **`OnceRetentionStrategy`** (from durable-task plugin) — used for single-use pods (`idleMinutes == 0`). `check()` examines idle time and calls `done(computer)` which sets `acceptingTasks(false)`. All in-memory.
- **`CloudRetentionStrategy`** (from Jenkins core) — used for reusable pods (`idleMinutes > 0`). `check()` examines idle time via `CloudSlaveRetentionStrategy`. All in-memory.

**Verdict: LIGHTWEIGHT** — Neither strategy makes network calls. They only check timestamps and set flags.

---

## 4. KubernetesCloud.canProvision() — PROVISIONER THREAD (no Queue lock)

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | `getTemplate(label)` | `PodTemplateUtils.getTemplateByLabel()` — iterates templates, label matching | LIGHTWEIGHT |

**Verdict: LIGHTWEIGHT** — Pure in-memory template matching.

---

## 5. KubernetesCloud.provision() — PROVISIONER THREAD (no Queue lock)

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | `new LimitRegistrationResults(this)` | Object creation | LIGHTWEIGHT |
| 2 | `state.getLabel()` | Field read | LIGHTWEIGHT |
| 3 | `InProvisioning.getAllInProvisioning(label)` | Iterates label's nodes checking `isNotAcceptingTasks` | MODERATE (O(label_nodes)) |
| 4 | `getTemplatesFor(label)` | `PodTemplateFilter.applyAll()` — iterates templates | LIGHTWEIGHT |
| 5 | `getUnwrappedTemplate(podTemplate)` | Template merging — in-memory | LIGHTWEIGHT |
| 6 | `limitRegistrationResults.register(podTemplate, numExecutors)` | `KubernetesProvisioningLimits.register()` — synchronized counter increment | LIGHTWEIGHT |
| 7 | `PlannedNodeBuilderFactory.createInstance().build()` | Creates `KubernetesSlave` + `CompletableFuture.completedFuture()` | LIGHTWEIGHT |

**Verdict: LIGHTWEIGHT** — This is a key finding. Unlike the EC2 and Azure plugins, `KubernetesCloud.provision()` does **not** make any Kubernetes API calls. It only:
- Checks provisioning limits (in-memory counters)
- Creates `KubernetesSlave` objects (in-memory)
- Returns `PlannedNode` futures that are **already completed** with the slave object

The actual pod creation happens later in `KubernetesLauncher.launch()`, which runs on `Computer.threadPoolForRemoting`.

**Note on `KubernetesProvisioningLimits.initInstance()`**: On first call, this acquires `Queue.withLock()` to count existing nodes. This is a one-time initialization cost. Subsequent calls are fast synchronized counter operations.

---

## 6. NoDelayProvisionerStrategy.apply() — PROVISIONER THREAD (no Queue lock)

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | `strategyState.getSnapshot()` | In-memory snapshot | LIGHTWEIGHT |
| 2 | `Jenkins.get().clouds` iteration | In-memory list | LIGHTWEIGHT |
| 3 | `cloud.canProvision(cloudState)` | In-memory label matching | LIGHTWEIGHT |
| 4 | `cloud.provision(cloudState, workloadToProvision)` | See provision() above | LIGHTWEIGHT |
| 5 | `Timer.get().schedule(label.nodeProvisioner::suggestReviewNow, 1L, TimeUnit.SECONDS)` | Async scheduling | LIGHTWEIGHT |

**Verdict: LIGHTWEIGHT** — Inherits the lightweight nature of `KubernetesCloud.provision()`.

---

## 7. KubernetesLauncher.launch() — REMOTING POOL (not Queue lock)

This is where the heavy Kubernetes API work happens. Called by Jenkins core on `Computer.threadPoolForRemoting`.

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | `Reaper.getInstance().maybeActivate()` | Extension lookup, possible one-time activation | MODERATE (first call) |
| 2 | `node.getTemplate()` | In-memory | LIGHTWEIGHT |
| 3 | `node.getKubernetesCloud()` | In-memory lookup | LIGHTWEIGHT |
| 4 | `cloud.connect()` | **Gets/creates Kubernetes client** | MODERATE (cached) |
| 5 | `template.build(node)` | Pod spec construction — in-memory | MODERATE |
| 6 | `cloud.registerPodInformer(node)` | May create a K8s watch — **Kubernetes API** | **HEAVYWEIGHT** (first call per namespace) |
| 7 | `client.pods().inNamespace(namespace).withName(podName).get()` | **Kubernetes API** — check existing pod | **HEAVYWEIGHT** |
| 8 | `client.pods().inNamespace(namespace).create(pod)` | **Kubernetes API** — create pod | **HEAVYWEIGHT** |
| 9 | `template.getWorkspaceVolume().createVolume(client, podMetadata)` | May make Kubernetes API calls for PVCs | **HEAVYWEIGHT** |
| 10 | `client.pods().waitUntilReady(timeout, TimeUnit.SECONDS)` | **Blocking Kubernetes API** — polls pod status | **HEAVYWEIGHT** |
| 11 | Polling loop: `slaveComputer.isOnline()` with `Thread.sleep(1000)` | **Blocks up to slaveConnectTimeout seconds** | **HEAVYWEIGHT** |
| 12 | `computer.setAcceptingTasks(true)` | In-memory flag | LIGHTWEIGHT |
| 13 | `node.save()` | Disk I/O | MODERATE |

**Verdict: HEAVYWEIGHT** — But all operations run on `Computer.threadPoolForRemoting`, never under the Queue lock. No Queue performance impact.

**Total worst-case duration:** Up to `slaveConnectTimeout` seconds (default 600s / 10 minutes). This is expected behavior for a launcher.

---

## 8. Reaper (ComputerListener) — NOT under Queue lock

The `Reaper` is a `ComputerListener` + `Watcher<Pod>` that monitors pod events and removes Jenkins agents when pods are deleted/failed.

### `preLaunch()` — Called before launcher

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | `Timer.get().schedule(this::maybeActivate, 10, TimeUnit.SECONDS)` | Async schedule | LIGHTWEIGHT |
| 2 | `isWatchingCloud()` + `watchCloud()` | May connect to K8s to set up watch | HEAVYWEIGHT (but async, off Queue lock) |

### `RemoveAgentOnPodDeleted.onEvent()` — Pod watcher callback

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | `Jenkins.get().removeNode(node)` | Modifies node list | MODERATE |
| 2 | `computer.disconnect(cause)` | Triggers disconnect | MODERATE |

**Missing:** No `scheduleMaintenance()` call after removing a node.

### `TerminateAgentOnContainerTerminated.onEvent()` — Pod watcher callback

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | `PodUtils.getTerminatedContainers(pod)` | In-memory | LIGHTWEIGHT |
| 2 | `node.terminate()` | Calls `_terminate()` → connects to K8s, deletes pod | HEAVYWEIGHT |
| 3 | `PodUtils.cancelQueueItemFor(pod, ...)` | Cancels queue items | MODERATE |
| 4 | `computer.disconnect(cause)` | Triggers disconnect | MODERATE |

**Missing:** No `scheduleMaintenance()` call after terminating an agent.

### `TerminateAgentOnPodFailed.onEvent()` — Pod watcher callback

Same pattern. **Missing:** No `scheduleMaintenance()` call.

---

## 9. KubernetesSlave._terminate() — CALLED FROM BACKGROUND THREADS

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | `getKubernetesCloud()` | In-memory lookup | LIGHTWEIGHT |
| 2 | `cloud.connect()` | Gets K8s client | MODERATE |
| 3 | `getPodRetention(cloud).shouldDeletePod(...)` | May call K8s API to get pod | **HEAVYWEIGHT** |
| 4 | `computer.getChannel()` + `ch.callAsync(new SlaveDisconnector())` | Remote call, up to 5s timeout | HEAVYWEIGHT |
| 5 | `deleteSlavePod(listener, client)` | `client.pods().delete()` — K8s API | **HEAVYWEIGHT** |

**Not a Queue lock concern** — `_terminate()` is called from retention strategy background tasks, Reaper callbacks, or explicit terminate calls — all off the Queue lock.

**Missing:** No `scheduleMaintenance()` after pod deletion or agent termination.

---

## 10. KubernetesProvisioningLimits — Provisioning Cap Tracking

### `register()` — Called from `provision()` (provisioner thread)

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | `initInstance()` — first call only | `Queue.withLock()` + iterates nodes | **MODERATE** (one-time) |
| 2 | `synchronized(this)` | Synchronized counter operations | LIGHTWEIGHT |
| 3 | Counter increment + cap comparison | In-memory arithmetic | LIGHTWEIGHT |

### `NodeListenerImpl.onDeleted()` — Called when node is removed

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | `instance.unregister(cloud, template, numExecutors)` | Synchronized counter decrement | LIGHTWEIGHT |

---

## Summary: Queue Lock Impact

### Under Queue Lock — All Lightweight

| Method | Class | Verdict |
|--------|-------|---------|
| `canTake(Node, BuildableItem)` | KubernetesQueueTaskDispatcher | LIGHTWEIGHT |
| `onEnterBuildable(BuildableItem)` | NoDelayProvisionerStrategy.FastProvisioning | LIGHTWEIGHT |
| `RetentionStrategy.check()` | OnceRetentionStrategy / CloudRetentionStrategy | LIGHTWEIGHT (not plugin code) |

### On Provisioner Thread — All Lightweight

| Method | Class | Verdict |
|--------|-------|---------|
| `canProvision(CloudState)` | KubernetesCloud | LIGHTWEIGHT |
| `provision(CloudState, int)` | KubernetesCloud | LIGHTWEIGHT |
| `apply(StrategyState)` | NoDelayProvisionerStrategy | LIGHTWEIGHT |

### On Remoting Pool — Heavyweight (Acceptable)

| Method | Class | Cost |
|--------|-------|------|
| `launch(SlaveComputer, TaskListener)` | KubernetesLauncher | Pod creation + wait (seconds to minutes) |
| `_terminate(TaskListener)` | KubernetesSlave | Pod deletion (seconds) |

### Missing scheduleMaintenance() Triggers

| Location | Event | Impact |
|----------|-------|--------|
| `Reaper.RemoveAgentOnPodDeleted` | Pod deleted → agent removed | Queue waits up to 5s to notice capacity change |
| `Reaper.TerminateAgentOnContainerTerminated` | Container terminated → agent terminated | Queue waits up to 5s |
| `Reaper.TerminateAgentOnPodFailed` | Pod failed → agent terminated | Queue waits up to 5s |
| `Reaper.TerminateAgentOnImagePullBackOff` | Image pull backoff → agent terminated | Queue waits up to 5s |
| `KubernetesLauncher.launch()` completion | Agent fully online, accepting tasks | Queue waits up to 5s |
| `KubernetesSlave._terminate()` | Agent pod deleted, node removed | Queue waits up to 5s |
| `GarbageCollection.PeriodicGarbageCollection` | Orphaned pods deleted | Queue waits up to 5s |
