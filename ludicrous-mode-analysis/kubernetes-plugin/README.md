# Kubernetes Plugin — Ludicrous Mode Analysis

## Summary

The kubernetes-plugin is **already well-architected for Queue performance**. All code paths under the Queue lock are lightweight (pure in-memory), the provisioner thread is never blocked (futures are already-completed), and all Kubernetes API calls are correctly deferred to background threads.

The **only significant gap** is the absence of `scheduleMaintenance()` triggers — the Queue is not notified when agents come online, go offline, or pods are deleted/failed, causing up to a 5-second delay before the Queue re-evaluates capacity.

## Plugin Architecture (Queue Perspective)

| Extension Point | Class | Queue Lock? | Verdict |
|----------------|-------|------------|---------|
| `Cloud.provision()` | `KubernetesCloud` | No | **LIGHTWEIGHT** — no K8s API calls, returns already-completed futures |
| `Cloud.canProvision()` | `KubernetesCloud` | No | **LIGHTWEIGHT** — in-memory template matching |
| `QueueTaskDispatcher.canTake()` | `KubernetesQueueTaskDispatcher` | **Yes** | **LIGHTWEIGHT** — in-memory folder property check |
| `QueueListener.onEnterBuildable()` | `FastProvisioning` | **Yes** | **LIGHTWEIGHT** — async provisioner trigger |
| `RetentionStrategy.check()` | `OnceRetentionStrategy`/`CloudRetentionStrategy` | **Yes** | **LIGHTWEIGHT** — timestamp check (not plugin code) |
| `ComputerLauncher.launch()` | `KubernetesLauncher` | No | **HEAVYWEIGHT** — K8s API (runs on remoting pool, acceptable) |
| `ComputerListener.preLaunch()` | `Reaper` | No | MODERATE — may set up K8s watch |

### Key Design Advantage Over EC2/Azure Plugins

Unlike EC2 and Azure VM Agents plugins, `KubernetesCloud.provision()` does **not** make any API calls. It only:
1. Checks provisioning limits (in-memory counters)
2. Creates `KubernetesSlave` objects (in-memory)
3. Returns `PlannedNode` futures that are **already completed**

Pod creation happens later in `KubernetesLauncher.launch()` on `Computer.threadPoolForRemoting`.

## Proposed Changes

### Phase 1: scheduleMaintenance() Triggers (HIGH IMPACT)

Add `scheduleMaintenance()` calls so the Queue responds immediately to capacity changes instead of waiting up to 5 seconds:

| Change | Location | Event |
|--------|----------|-------|
| **New** `KubernetesComputerListener` | New file | `onOnline`, `onOffline`, `onTemporarilyOnline`, `onTemporarilyOffline` |
| Reaper: `RemoveAgentOnPodDeleted` | After `removeNode()` | Pod deleted externally |
| Reaper: `TerminateAgentOnContainerTerminated` | After `logAndCleanUp()` | Container terminated |
| Reaper: `TerminateAgentOnPodFailed` | After `logAndCleanUp()` | Pod failed |
| Reaper: `TerminateAgentOnImagePullBackOff` | After `terminate()` | Image pull backoff |
| `GarbageCollection` | After orphan deletion | Orphan cleanup |

**Expected impact:** Queue responds to capacity changes in milliseconds instead of up to 5 seconds.

### Phase 2: Minor Optimizations (LOW PRIORITY)

- `KubernetesProvisioningLimits.initInstance()` — avoid `Queue.withLock()` on first call (one-time cost, negligible impact)

### No Changes Needed

- All Queue lock code paths are already lightweight
- All provisioner thread code paths are already lightweight
- All heavyweight K8s API operations already run on background threads
- Retention strategies delegate to core/durable-task (both lightweight)

## Background Documents

- [Queue Lock Audit](background/KUBERNETES_QUEUE_AUDIT.md) — every code path under Queue lock or blocking provisioner
- [Heavy vs Lightweight Classification](background/KUBERNETES_HEAVY_LIGHTWEIGHT_ANALYSIS.md) — detailed classification of all operations
- [Queue Performance Patch Plan](background/KUBERNETES_QUEUE_PERFORMANCE_ANALYSIS.md) — phased approach with comparison to other plugins
