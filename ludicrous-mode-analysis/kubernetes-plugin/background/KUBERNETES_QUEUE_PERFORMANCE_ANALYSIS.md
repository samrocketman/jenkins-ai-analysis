# Kubernetes Plugin — Queue Performance Patch Plan

Phased approach to optimize kubernetes-plugin for Queue performance.

---

## Executive Summary

The kubernetes-plugin is **already well-architected for Queue performance**. Unlike the EC2 and Azure VM Agents plugins, it:

1. **Has no heavyweight code under the Queue lock** — `canTake()` and `onEnterBuildable()` are pure in-memory
2. **Does not block the provisioner thread** — `provision()` returns immediately-completed futures
3. **Defers all Kubernetes API calls to background threads** — pod creation, waiting, and deletion all happen on `Computer.threadPoolForRemoting`

The **only significant gap** is the absence of `scheduleMaintenance()` triggers. When agents come online, go offline, or pods are deleted/failed, the Queue is not notified immediately. This causes up to a 5-second delay (the `maintain()` interval) before the Queue re-evaluates.

With `scheduleMaintenance()` being fast (because `maintain()` itself is fast when all plugins cooperate), adding these triggers will make the Queue respond sub-second to capacity changes.

---

## Phase 1: scheduleMaintenance() Triggers (HIGH IMPACT, LOW RISK)

### 1A. Add KubernetesComputerListener

Create a new `ComputerListener` that triggers `scheduleMaintenance()` on all agent lifecycle events. This covers the most common capacity change events.

```java
@Extension
public class KubernetesComputerListener extends ComputerListener {

    @Override
    public void onOnline(Computer c, TaskListener listener) {
        if (c instanceof KubernetesComputer) {
            scheduleMaintenance();
        }
    }

    @Override
    public void onOffline(@NonNull Computer c, @CheckForNull OfflineCause cause) {
        if (c instanceof KubernetesComputer) {
            scheduleMaintenance();
        }
    }

    @Override
    public void onTemporarilyOnline(Computer c) {
        if (c instanceof KubernetesComputer) {
            scheduleMaintenance();
        }
    }

    @Override
    public void onTemporarilyOffline(Computer c, OfflineCause cause) {
        if (c instanceof KubernetesComputer) {
            scheduleMaintenance();
        }
    }

    private static void scheduleMaintenance() {
        Jenkins j = Jenkins.getInstanceOrNull();
        if (j != null) {
            j.getQueue().scheduleMaintenance();
        }
    }
}
```

**Events covered:**
| Event | Trigger Source |
|-------|---------------|
| Agent comes online | `KubernetesLauncher.launch()` completes → `setAcceptingTasks(true)` → `onOnline()` |
| Agent goes offline | Pod deletion, agent disconnect, `_terminate()` |
| Agent temporarily offline | Manual mark offline, health check failure |
| Agent temporarily online | Manual mark online |

### 1B. Add scheduleMaintenance() to Reaper Event Handlers

The `Reaper` handles pod events from the Kubernetes watch API. Some of these events cause node removal or termination **before** the `ComputerListener` callbacks fire (because the node is removed from Jenkins before the computer has a chance to go through the offline path).

Add `scheduleMaintenance()` to each `Reaper.Listener` implementation:

| Listener | After Which Operation |
|----------|----------------------|
| `RemoveAgentOnPodDeleted` | After `Jenkins.get().removeNode(node)` |
| `TerminateAgentOnContainerTerminated` | After `logAndCleanUp()` |
| `TerminateAgentOnPodFailed` | After `logAndCleanUp()` |
| `TerminateAgentOnImagePullBackOff` | After `node.terminate()` |

### 1C. Add scheduleMaintenance() to GarbageCollection

After deleting orphan pods in `PeriodicGarbageCollection.garbageCollect()`, trigger `scheduleMaintenance()` so the Queue can re-provision replacements immediately.

### Summary of Phase 1 Changes

| File | Change |
|------|--------|
| **NEW** `KubernetesComputerListener.java` | `ComputerListener` with `scheduleMaintenance()` on online/offline events |
| `Reaper.java` — `RemoveAgentOnPodDeleted.onEvent()` | Add `scheduleMaintenance()` after `removeNode()` |
| `Reaper.java` — `TerminateAgentOnContainerTerminated.onEvent()` | Add `scheduleMaintenance()` in `logAndCleanUp()` |
| `Reaper.java` — `TerminateAgentOnPodFailed.onEvent()` | Same — covered by `logAndCleanUp()` |
| `Reaper.java` — `TerminateAgentOnImagePullBackOff.onEvent()` | Add `scheduleMaintenance()` after `node.terminate()` |
| `GarbageCollection.java` — `garbageCollect()` | Add `scheduleMaintenance()` after orphan deletion |

**Expected Impact:**
- Queue responds to capacity changes within milliseconds instead of up to 5 seconds
- Combined with fast `maintain()` (from other Ludicrous Mode patches), builds start sub-second after agent is ready

---

## Phase 2: Minor Optimizations (LOW PRIORITY)

### 2A. KubernetesProvisioningLimits First-Call Contention

The `initInstance()` method acquires `Queue.withLock()` on the first `register()` call. This is a one-time cost but could be avoided by pre-initializing counts from a `@Initializer(after = InitMilestone.COMPLETED)` callback.

**Impact: Negligible** — only delays the very first provisioning cycle after Jenkins startup by the time to iterate all nodes.

### 2B. InProvisioning.getAllInProvisioning() Efficiency

`DefaultInProvisioning.getInProvisioning()` iterates all nodes for a label and checks `isNotAcceptingTasks()`. For labels with many nodes, this could be slow.

**Impact: Low** — Kubernetes agents are typically one-shot, so there are usually few nodes per label.

---

## What Does NOT Need Changing

### Queue Lock Code Paths

| Code Path | Why It's Already Fast |
|-----------|--------------------|
| `KubernetesQueueTaskDispatcher.canTake()` | Pure in-memory folder property check |
| `FastProvisioning.onEnterBuildable()` | In-memory cloud check + async provisioner trigger |
| `OnceRetentionStrategy.check()` | Core strategy, timestamp comparison only |
| `CloudRetentionStrategy.check()` | Core strategy, timestamp comparison only |

### Provisioner Thread Code Paths

| Code Path | Why It's Already Fast |
|-----------|--------------------|
| `KubernetesCloud.canProvision()` | In-memory template matching |
| `KubernetesCloud.provision()` | No API calls, returns immediately-completed futures |
| `NoDelayProvisionerStrategy.apply()` | Orchestrates above, all in-memory |
| `StandardPlannedNodeBuilder.build()` | Creates KubernetesSlave in-memory, wraps in completed future |

### Background Thread Code Paths

| Code Path | Why It's Acceptable |
|-----------|--------------------|
| `KubernetesLauncher.launch()` | Runs on `Computer.threadPoolForRemoting`, expected to be slow |
| `KubernetesSlave._terminate()` | Runs from background tasks |
| `Reaper` watcher callbacks | Runs on K8s client watch thread |
| `GarbageCollection` | Runs on `AsyncPeriodicWork` thread |

---

## Comparison with Other Cloud Plugins

| Aspect | EC2 Plugin (Before) | Azure VM Agents (Before) | Kubernetes Plugin (Current) |
|--------|--------------------|-----------------------|---------------------------|
| Queue lock paths | HEAVYWEIGHT (AWS API in RetentionStrategy) | LIGHTWEIGHT | LIGHTWEIGHT |
| Provisioner blocking | HEAVYWEIGHT (synchronous EC2 API) | HEAVYWEIGHT (synchronous Azure API) | LIGHTWEIGHT (no API calls) |
| `provision()` returns | Futures that block on launch | Futures that block on VM create | **Already-completed futures** |
| `scheduleMaintenance()` triggers | Added in patches | Missing (proposed) | **Missing (proposed)** |
| Background thread usage | Added in patches | Partially done | **Already correct** |

The kubernetes-plugin's architecture is fundamentally better than the other cloud plugins because:

1. **Pod creation is deferred to the launcher** — `provision()` just creates the `KubernetesSlave` object in memory
2. **No retention strategy code** — delegates to existing lightweight strategies
3. **Watch-based reaping** — uses K8s watch API instead of polling (efficient for pod lifecycle tracking)
4. **Already uses Caffeine caching** — for Reaper's backoff tracking

The only gap is responsiveness — the missing `scheduleMaintenance()` triggers.
