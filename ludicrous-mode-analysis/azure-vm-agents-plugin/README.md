# Azure VM Agents Plugin — Queue Performance Patches

Analysis of `jenkinsci/azure-vm-agents-plugin` for Queue lock contention and provisioner thread blocking. Proposed patches across 3 phases, ~10 files changed.

---

## How This Helps Unblock the Queue

Jenkins core's `Queue.maintain()` runs under a global `ReentrantLock`. Plugins hooking into this loop can block all scheduling if they perform slow operations. The azure-vm-agents-plugin has **three retention strategies** (`AzureVMCloudRetensionStrategy`, `AzureVMCloudPoolRetentionStrategy`, `AzureVMCloudOnceRetentionStrategy`), a **Cloud implementation** with `provision()`, and a **no-delay provisioner strategy** that all interact with the Queue.

**Good news:** The retention strategies are already well-designed — all three `check()` methods defer Azure API calls to background thread pools (`AzureVMCloud.getThreadPool()` and `Computer.threadPoolForRemoting`). The Queue lock path is lightweight.

**Issues found:**

1. **No `scheduleMaintenance()` triggers anywhere.** The plugin never notifies the Queue when capacity changes. Agents that come online, get deprovisioned, or finish tasks don't trigger immediate queue re-evaluation. Work waits up to 5 seconds for the next timer tick.

2. **`provision()` makes synchronous Azure API calls on the provisioner thread.** Template verification calls `azureClient.virtualMachines().listByResourceGroup()` and node reuse checks call `azureClient.virtualMachines().getByResourceGroup()` per candidate — all blocking the NodeProvisioner.

3. **No VM state caching.** Every `virtualMachineExists()` call is a live Azure API call with no TTL cache.

---

## Phase 1: Comprehensive scheduleMaintenance() Triggers

Since Queue maintenance is now millisecond-fast, trigger it aggressively at every event where scheduling capacity changes.

### New Files

- **AzureVMComputerListener.java** — `ComputerListener.onOnline()` triggers immediate `scheduleMaintenance()` when an Azure VM agent is fully online.

### New Infrastructure

- **`AzureVMCloud.scheduleQueueMaintenance(long delayMs)`** — Static helper using `jenkins.util.Timer` + `Queue.scheduleMaintenance()`. Two flavors: immediate (0ms) for "agent ready" events, delayed (configurable, default 1000ms) for "capacity planned/removed" events.

### Trigger Points (10 total)

**Immediate (agent definitively ready for work):**

| Location | Event |
|----------|-------|
| `AzureVMComputerListener.onOnline()` | Agent online, channel established |
| `AzureVMCloud.provision()` reuse path | Reused VM back and accepting tasks |
| `AzureVMCloud.doProvision()` PlannedNode future | New VM connected |
| `AzureVMAgentSSHLauncher` post-reconnect | Recovered agent ready |

**Delayed ~1s (capacity planned or removed):**

| Location | Event |
|----------|-------|
| `AzureVMCloud.doProvision()` after deployment submit | Capacity planned |
| `AzureVMMaintainPoolTask.provisionNodes()` | Pool adding capacity |
| `AzureVMAgent.deprovision()` after `removeNode()` | Agent removed |
| `AzureVMAgent.shutdown()` after `setEligibleForReuse(true)` | Agent going offline |
| `AzureVMAgentCleanUpTask.cleanVMs()` after `removeNode()` | Dead node cleaned |
| `AzureVMCloudPoolRetentionStrategy.checkPoolSizeAndDelete()` | Excess VMs deleted |

---

## Phase 2: Fix Provisioner Thread Blocking

### Async Template Verification

`AzureVMCloud.provision()` currently calls `AzureVMCloudVerificationTask.verify()` synchronously, which lists all VMs via `azureClient.virtualMachines().listByResourceGroup()`. The fix submits verification to a background thread and returns empty from `provision()`. The next NodeProvisioner cycle finds the template verified.

### Move virtualMachineExists() Into PlannedNode Future

The node reuse path in `provision()` calls `virtualMachineExists()` per offline Azure VM — N sequential Azure API calls. The fix moves the existence check inside the PlannedNode future, making `provision()` return in microseconds. Reuse candidates are counted optimistically; if a VM no longer exists, the future throws and the provisioner retries.

### VM Existence Cache

Add a `Caffeine` TTL cache (30s default) for `virtualMachineExists()` results in `AzureVMManagementServiceDelegate`. The plugin already depends on `caffeine-api`.

---

## Phase 3: Batch Operations and Minor Optimizations

- **Batch VM queries for cleanup** — Replace per-VM `getByResourceGroup()` calls in `cleanVMs()` with a single `listByResourceGroup()` per cloud.
- **Per-template locking** — Replace `static synchronized` in `checkPoolSizeAndDelete()` with `ConcurrentHashMap`-based per-template locks.
- **Scheduled polling for createProvisionedAgent()** — Replace `Thread.sleep(5s)` loop with `ScheduledExecutorService` callbacks to free thread pool threads during deployment polling.

---

## Configuration

| System Property | Default | Phase | Description |
|-----------------|---------|-------|-------------|
| `azure.vm.scheduleMaintenanceDelayMs` | 1000 | 1 | Delay before `scheduleMaintenance()` for non-immediate events |
| `azure.vm.existsCacheTtlMs` | 30000 | 2 | VM existence cache TTL |

---

## Background Analysis

- [AZURE_QUEUE_AUDIT.md](background/AZURE_QUEUE_AUDIT.md) — Full audit of all code paths under Queue lock and provisioner thread
- [AZURE_HEAVY_LIGHTWEIGHT_ANALYSIS.md](background/AZURE_HEAVY_LIGHTWEIGHT_ANALYSIS.md) — Detailed heavy vs lightweight classification with proposed fixes
- [AZURE_QUEUE_PERFORMANCE_ANALYSIS.md](background/AZURE_QUEUE_PERFORMANCE_ANALYSIS.md) — Phased patch plan with specific code changes and snippets
