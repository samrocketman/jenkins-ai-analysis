# Azure VM Agents Plugin — Queue Performance Patch Plan

Phased patch plan to optimize the azure-vm-agents-plugin for millisecond-fast Queue maintenance.

---

## Current State

The azure-vm-agents-plugin has **good fundamentals** — all three retention strategies already defer Azure API calls to background thread pools. The Queue lock path is lightweight. However, two categories of issues remain:

1. **Provisioner thread blocking** — `AzureVMCloud.provision()` makes synchronous Azure API calls that block the NodeProvisioner.
2. **Missing scheduleMaintenance() triggers** — The plugin never notifies the Queue when capacity changes, causing up to 5-second delays before work is dispatched to newly available agents.

---

## Phase 1: Comprehensive scheduleMaintenance() Triggers

**Goal:** Trigger `Queue.scheduleMaintenance()` at every event where scheduling capacity changes, so work dispatches immediately instead of waiting for the 5-second timer tick.

### 1.1 New Infrastructure

#### New class: `AzureVMComputerListener`

```java
package com.microsoft.azure.vmagent;

import hudson.Extension;
import hudson.model.Computer;
import hudson.slaves.ComputerListener;
import jenkins.model.Jenkins;

import java.util.logging.Level;
import java.util.logging.Logger;

@Extension
public class AzureVMComputerListener extends ComputerListener {
    private static final Logger LOGGER = Logger.getLogger(AzureVMComputerListener.class.getName());

    @Override
    public void onOnline(Computer c, TaskListener listener) {
        if (c instanceof AzureVMComputer) {
            LOGGER.log(Level.FINE, "Azure VM agent online: {0}, triggering queue maintenance",
                    c.getName());
            AzureVMCloud.scheduleQueueMaintenance(0);
        }
    }
}
```

#### New static helper: `AzureVMCloud.scheduleQueueMaintenance(long delayMs)`

Add to `AzureVMCloud.java`:

```java
private static final long DEFAULT_MAINTENANCE_DELAY_MS =
        SystemProperties.getLong("azure.vm.scheduleMaintenanceDelayMs", 1000L);

public static void scheduleQueueMaintenance() {
    scheduleQueueMaintenance(DEFAULT_MAINTENANCE_DELAY_MS);
}

public static void scheduleQueueMaintenance(long delayMs) {
    Jenkins j = Jenkins.getInstanceOrNull();
    if (j == null) return;
    if (delayMs <= 0) {
        j.getQueue().scheduleMaintenance();
    } else {
        jenkins.util.Timer.get().schedule(
                () -> j.getQueue().scheduleMaintenance(),
                delayMs,
                java.util.concurrent.TimeUnit.MILLISECONDS);
    }
}
```

### 1.2 Trigger Points — Capacity Added (Immediate, 0ms delay)

These events mean an executor is definitively ready for work:

#### `AzureVMComputerListener.onOnline()` (new class)

Agent fully online, channel established, all `ComputerListener`s have run.

```java
AzureVMCloud.scheduleQueueMaintenance(0);
```

#### `AzureVMCloud.provision()` — reuse path (line ~707)

After `azureComputer.setAcceptingTasks(true)` and `agentNode.clearCleanUpAction()`:

```java
azureComputer.setAcceptingTasks(true);
agentNode.clearCleanUpAction();
agentNode.setEligibleForReuse(false);
AzureVMCloud.scheduleQueueMaintenance(0); // <-- ADD
```

Note: This runs inside a PlannedNode future on `Computer.threadPoolForRemoting`, not under the Queue lock.

#### `AzureVMCloud.doProvision()` — inner PlannedNode future (line ~901)

After `agent.clearCleanUpAction()` in the finally block, the agent is connected and ready:

```java
} finally {
    agent.clearCleanUpAction();
}
AzureVMCloud.scheduleQueueMaintenance(0); // <-- ADD (after the try-finally)
```

#### `AzureVMAgentSSHLauncher` — after successful reconnect (line ~257)

After `agent.clearCleanUpAction()` and `successful = true`:

```java
agent.clearCleanUpAction();
successful = true;
AzureVMCloud.scheduleQueueMaintenance(0); // <-- ADD
```

### 1.3 Trigger Points — Capacity Planned (Delayed, ~1s)

These events mean capacity is being added but isn't ready yet:

#### `AzureVMCloud.doProvision()` — after deployment future submitted (line ~834)

```java
final Future<AzureVMDeploymentInfo> deploymentFuture = getThreadPool().submit(callableTask);
AzureVMCloud.scheduleQueueMaintenance(); // <-- ADD (default 1s delay)
```

#### `AzureVMMaintainPoolTask.provisionNodes()` — after `doProvision()` (line ~76)

```java
cloud.doProvision(newAgents, new ArrayList<>(), template, true);
AzureVMCloud.scheduleQueueMaintenance(); // <-- ADD (default 1s delay)
```

### 1.4 Trigger Points — Capacity Removed (Delayed, ~1s)

These events reduce capacity. The Queue should reassign any work that was targeting the removed agent:

#### `AzureVMAgent.deprovision()` — after `removeNode()` (line ~722)

```java
Jenkins.get().removeNode(this);
AzureVMCloud.scheduleQueueMaintenance(); // <-- ADD
```

#### `AzureVMAgent.shutdown()` — after `setEligibleForReuse(true)` (line ~615)

```java
setEligibleForReuse(true);
AzureVMCloud.scheduleQueueMaintenance(); // <-- ADD
```

#### `AzureVMAgentCleanUpTask.cleanVMs()` — after `removeNode()` for dead VMs (line ~522)

```java
Jenkins.get().removeNode(agentNode);
AzureVMCloud.scheduleQueueMaintenance(); // <-- ADD
```

#### `AzureVMCloudPoolRetentionStrategy.checkPoolSizeAndDelete()` — after deletion

After `tryDeleteWhenIdle()` call inside `checkPoolSizeAndDelete()`:

```java
if (count > effectiveLimit) {
    tryDeleteWhenIdle(agentComputer);
    AzureVMCloud.scheduleQueueMaintenance(); // <-- ADD
}
```

### 1.5 Configuration

| System Property | Default | Description |
|-----------------|---------|-------------|
| `azure.vm.scheduleMaintenanceDelayMs` | 1000 | Delay (ms) before triggering `scheduleMaintenance()` for non-immediate events |

---

## Phase 2: Fix Provisioner Thread Blocking

**Goal:** Remove synchronous Azure API calls from the `provision()` method so the NodeProvisioner thread returns quickly.

### 2.1 Remove synchronous template verification from provision()

**Current (HEAVYWEIGHT):**

```java
// AzureVMCloud.provision() line 654
if (!template.retrieveTemplateProvisionStrategy().isVerifiedPass()) {
    AzureVMCloudVerificationTask.verify(this.name, template.getTemplateName());
}
if (template.retrieveTemplateProvisionStrategy().isVerifiedFailed()) {
    return new ArrayList<>();
}
```

`verify()` calls `updateCloudVirtualMachineCounts()` → `azureClient.virtualMachines().listByResourceGroup()`.

**Proposed fix:**

If the template is not verified, submit an async verification and return empty. The next provisioner cycle will find the template verified and proceed.

```java
if (!template.retrieveTemplateProvisionStrategy().isVerifiedPass()) {
    // Submit async verification instead of blocking
    getThreadPool().submit(() ->
        AzureVMCloudVerificationTask.verify(this.name, template.getTemplateName()));
    LOGGER.log(Level.INFO, "Template {0} not yet verified, async verification submitted",
            template.getTemplateName());
    return Collections.emptyList();
}
```

This means the first `provision()` call for an unverified template returns empty, but verification completes in the background and the next NodeProvisioner cycle (triggered by `suggestReviewNow()` in `AzureVMFastProvisioning`) will find it verified.

### 2.2 Move virtualMachineExists() into PlannedNode future

**Current (HEAVYWEIGHT):**

```java
// AzureVMCloud.provision() line 677
if (AzureVMManagementServiceDelegate.virtualMachineExists(agentNode)) {
    numberOfAgents--;
    plannedNodes.add(new PlannedNode(...));
}
```

This calls `getByResourceGroup()` per offline VM — N sequential Azure API calls.

**Proposed fix:**

Move the existence check inside the PlannedNode future. Create the PlannedNode optimistically and check existence as the first step of the future:

```java
if (agentNode != null && isNodeEligibleForReuse(agentNode, template)) {
    numberOfAgents--;
    plannedNodes.add(new PlannedNode(agentNode.getNodeName(),
            Computer.threadPoolForRemoting.submit(() -> {
                // Check existence inside the future (off provisioner thread)
                if (!AzureVMManagementServiceDelegate.virtualMachineExists(agentNode)) {
                    throw AzureCloudException.create("VM no longer exists: " + agentNode.getNodeName());
                }
                final Object agentLock = getLockForAgent(agentNode);
                try {
                    synchronized (agentLock) {
                        // ... existing reuse logic ...
                    }
                } finally {
                    releaseLockForAgent(agentNode);
                }
            }), template.getNoOfParallelJobs()));
}
```

This eliminates all synchronous Azure API calls from `provision()`, making it return in microseconds.

**Trade-off:** The NodeProvisioner will optimistically count reuse candidates as planned capacity. If a VM no longer exists, the PlannedNode future throws and the provisioner retries. This is acceptable because:
- The `isNodeEligibleForReuse()` check (in-memory) filters out most ineligible VMs.
- VMs that were eligible for reuse rarely disappear between checks.
- A failed PlannedNode triggers a new provisioner cycle.

### 2.3 Add VM existence caching

Add a TTL cache for `virtualMachineExists()` results to avoid redundant API calls during cleanup, reuse checks, and retries.

In `AzureVMManagementServiceDelegate`:

```java
private static final long VM_EXISTS_CACHE_TTL_MS =
        SystemProperties.getLong("azure.vm.existsCacheTtlMs", 30_000L);

private final Cache<String, Boolean> vmExistsCache = Caffeine.newBuilder()
        .expireAfterWrite(VM_EXISTS_CACHE_TTL_MS, TimeUnit.MILLISECONDS)
        .maximumSize(1000)
        .build();

private boolean virtualMachineExists(String vmName, String resourceGroupName) throws AzureCloudException {
    String cacheKey = resourceGroupName + "/" + vmName;
    Boolean cached = vmExistsCache.getIfPresent(cacheKey);
    if (cached != null) {
        return cached;
    }

    boolean exists;
    try {
        azureClient.virtualMachines().getByResourceGroup(resourceGroupName, vmName);
        exists = true;
    } catch (ManagementException e) {
        if (e.getResponse().getStatusCode() == 404) {
            exists = false;
        } else {
            throw e;
        }
    } catch (Exception e) {
        throw AzureCloudException.create(e);
    }

    vmExistsCache.put(cacheKey, exists);
    return exists;
}
```

The plugin already has `caffeine-api` as a dependency, so the `Caffeine` cache is available.

| System Property | Default | Description |
|-----------------|---------|-------------|
| `azure.vm.existsCacheTtlMs` | 30000 | VM existence cache TTL (ms) |

---

## Phase 3: Batch Operations and Minor Optimizations

### 3.1 Batch VM queries for cleanup

`AzureVMAgentCleanUpTask.cleanVMs()` calls `virtualMachineExists()` per offline VM. Replace with a single `listByResourceGroup()` per cloud:

```java
// Batch: list all VMs once per cloud
Map<String, Set<String>> vmsByCloud = new HashMap<>();
for (AzureVMCloud cloud : Jenkins.get().clouds.getAll(AzureVMCloud.class)) {
    Set<String> existingVMs = new HashSet<>();
    try {
        for (VirtualMachine vm : cloud.getAzureClient().virtualMachines()
                .listByResourceGroup(cloud.getResourceGroupName())) {
            existingVMs.add(vm.name());
        }
    } catch (Exception e) {
        LOGGER.log(Level.WARNING, "Failed to list VMs for cloud " + cloud.getCloudName(), e);
    }
    vmsByCloud.put(cloud.getCloudName(), existingVMs);
}

// Then check existence from the cached set
```

This reduces N API calls per cleanup cycle to 1 per cloud.

### 3.2 Replace `static synchronized` in `checkPoolSizeAndDelete()`

Replace the global static lock with per-template granularity:

```java
private static final ConcurrentHashMap<String, Object> templateLocks = new ConcurrentHashMap<>();

private static void checkPoolSizeAndDelete(AzureVMComputer agentComputer, int poolSize, int effectiveMaxVMs) {
    AzureVMAgent templateAgentNode = agentComputer.getNode();
    if (templateAgentNode == null) return;

    String templateKey = templateAgentNode.getTemplate().getTemplateName();
    Object lock = templateLocks.computeIfAbsent(templateKey, k -> new Object());

    synchronized (lock) {
        // ... existing counting and deletion logic ...
    }
}
```

### 3.3 Improve createProvisionedAgent() polling

Replace `Thread.sleep(5s)` polling with `ScheduledExecutorService` callbacks:

```java
// Instead of blocking a thread pool thread for 20+ minutes:
ScheduledFuture<?> poller = Timer.get().scheduleWithFixedDelay(() -> {
    try {
        Deployment dep = azureClient.deployments().getByResourceGroup(resourceGroup, deploymentName);
        if ("succeeded".equalsIgnoreCase(dep.provisioningState())) {
            // Complete the CompletableFuture
            future.complete(createAgentFromDeployment(dep));
        } else if (isTerminalFailure(dep)) {
            future.completeExceptionally(new AzureCloudException("Deployment failed"));
        }
    } catch (Exception e) {
        future.completeExceptionally(e);
    }
}, 5, 5, TimeUnit.SECONDS);
```

This frees the thread pool thread during the polling interval. Lower priority since it's already off the Queue lock.

---

## Files Changed Summary

### Phase 1 (scheduleMaintenance triggers)

| File | Change |
|------|--------|
| **AzureVMComputerListener.java** (NEW) | `ComputerListener.onOnline()` → `scheduleQueueMaintenance(0)` |
| **AzureVMCloud.java** | Add `scheduleQueueMaintenance()` static helper. Add triggers in reuse path and after deployment future submit |
| **AzureVMAgent.java** | Add trigger after `removeNode()` in `deprovision()` and after `setEligibleForReuse(true)` in `shutdown()` |
| **AzureVMAgentCleanUpTask.java** | Add trigger after `removeNode()` for dead VMs |
| **AzureVMCloudPoolRetentionStrategy.java** | Add trigger after `tryDeleteWhenIdle()` in `checkPoolSizeAndDelete()` |
| **AzureVMMaintainPoolTask.java** | Add trigger after `provisionNodes()` |
| **AzureVMAgentSSHLauncher.java** (remote package) | Add trigger after successful reconnect |

### Phase 2 (provisioner fixes)

| File | Change |
|------|--------|
| **AzureVMCloud.java** | Async template verification in `provision()`, move `virtualMachineExists()` into PlannedNode future |
| **AzureVMManagementServiceDelegate.java** | Add `Caffeine` TTL cache for `virtualMachineExists()` |

### Phase 3 (batch + minor)

| File | Change |
|------|--------|
| **AzureVMAgentCleanUpTask.java** | Batch `listByResourceGroup()` instead of per-VM checks |
| **AzureVMCloudPoolRetentionStrategy.java** | Per-template locking instead of `static synchronized` |
| **AzureVMCloud.java** | Optional: `ScheduledExecutorService` polling for `createProvisionedAgent()` |

---

## Configuration Summary

| System Property | Default | Phase | Description |
|-----------------|---------|-------|-------------|
| `azure.vm.scheduleMaintenanceDelayMs` | 1000 | 1 | Delay before `scheduleMaintenance()` for non-immediate events |
| `azure.vm.existsCacheTtlMs` | 30000 | 2 | VM existence cache TTL |
