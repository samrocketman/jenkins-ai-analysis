# Azure VM Agents Plugin — Heavy vs Lightweight Classification

Detailed classification of every operation in the azure-vm-agents-plugin that could affect Jenkins Queue performance, with proposed fixes for heavyweight operations.

---

## Classification Key

- **LIGHTWEIGHT**: Returns in microseconds. No network I/O, no Azure API calls, no blocking locks. Safe to run under the Queue lock.
- **HEAVYWEIGHT**: Makes Azure API calls, blocks on network I/O, or holds locks for extended periods. Must NOT run under the Queue lock or on the provisioner thread synchronously.
- **DEFERRED**: Heavy work exists but is already submitted to a background thread pool. The call site itself is lightweight.

---

## 1. AzureVMCloudRetensionStrategy

### `check(AzureVMComputer)` — LIGHTWEIGHT

The idle-timeout retention strategy. Called under the Queue lock once per Azure VM computer per maintenance cycle.

**What it does under the lock:**
- Reads `isIdle()`, `isOnline()`, `idleTerminationMillis`, `getIdleStartMilliseconds()` — all in-memory field reads.
- If the VM should be recycled, creates a `Callable` and submits it via `executionEngine.executeAsync()` to `AzureVMCloud.getThreadPool()`.
- Returns `1` (re-check in 1 minute).

**What runs on the background thread (DEFERRED):**
- `agent.shutdown()` → `serviceDelegate.shutdownVirtualMachine()` → `azureClient.virtualMachines().getByResourceGroup().deallocate()`
- `agent.deprovision()` → `terminateVirtualMachine()` → `azureClient.virtualMachines().deleteByResourceGroup()` + disk cleanup

**No changes needed** for the Queue lock path. The deferral pattern is correct.

**Missing:** No `scheduleMaintenance()` after the background shutdown/deprovision completes.

### `start(AzureVMComputer)` — LIGHTWEIGHT

- `azureComputer.connect(false)` — returns `Future<?>` immediately (non-blocking).
- `resetShutdownVMStatus()` — submits shutdown check to `Computer.threadPoolForRemoting`.

**No changes needed.**

---

## 2. AzureVMCloudPoolRetentionStrategy

### `check(AzureVMComputer)` — LIGHTWEIGHT

The pool retention strategy. Called under the Queue lock.

**What it does under the lock:**
- `agentComputer.getNode()` — in-memory.
- `agentNode.getCloud()` → `Jenkins.get().getCloud(name)` — in-memory map lookup.
- Iterates `currentCloud.getVmTemplates()` calling `TemplateUtil.checkSame()` — string comparisons on a `CopyOnWriteArrayList`. O(templates) but each comparison is fast.
- Checks retention time expiry — timestamp comparison.
- Submits `tryDeleteWhenIdle` or `checkPoolSizeAndDelete` to `Computer.threadPoolForRemoting`.
- Returns `1`.

**What runs on the background thread (DEFERRED):**
- `tryDeleteWhenIdle()` → `agent.deprovision()` → Azure API calls.
- `checkPoolSizeAndDelete()` — `static synchronized`, iterates `Jenkins.get().getComputers()`, counts matching VMs, may call `tryDeleteWhenIdle()`.

**Minor concern:** `checkPoolSizeAndDelete()` is `static synchronized`, meaning ALL pool retention checks across all templates serialize. This is a bottleneck if many templates use pool retention, but it runs off the Queue lock so it doesn't block `maintain()`.

**Proposed fix:** Replace `static synchronized` with per-template locking or use `ConcurrentHashMap` for the count. Low priority since it's off the Queue lock.

**Missing:** No `scheduleMaintenance()` after `tryDeleteWhenIdle()` removes VMs.

### `start(AzureVMComputer)` — LIGHTWEIGHT

Same pattern as `AzureVMCloudRetensionStrategy.start()`. No changes needed.

### `taskAccepted(Executor, Queue.Task)` — LIGHTWEIGHT

Empty method body.

### `taskCompleted` / `taskCompletedWithProblems` — LIGHTWEIGHT

Calls `done(executor)` → `done(AzureVMComputer)`:
- `computer.setAcceptingTasks(false)` — in-memory flag.
- `agent.setCleanUpAction(...)` — in-memory flag.

**No changes needed** for Queue lock. These correctly set flags for the cleanup task to handle later.

### `calculateEffectivePoolSize(int, int)` — LIGHTWEIGHT

Pure arithmetic. No I/O.

---

## 3. AzureVMCloudOnceRetentionStrategy

### `check(AzureVMComputer)` — LIGHTWEIGHT

- Checks `isIdle()`, `getIdleStartMilliseconds()`, `getConnectTime()` — all in-memory.
- Calls `done(agentComputer)` which sets `setAcceptingTasks(false)` and `setCleanUpAction(...)`.

**No changes needed.**

### `start(AzureVMComputer)` — LIGHTWEIGHT

Same pattern. No changes needed.

### `taskAccepted` / `taskCompleted` — LIGHTWEIGHT

Same pattern as pool strategy.

---

## 4. AzureVMCloud

### `canProvision(CloudState)` — LIGHTWEIGHT

- `getAzureAgentTemplate(label)` — iterates `vmTemplates`, matches labels. O(templates).
- `template.isTemplateDisabled()` — boolean flag.
- `template.retrieveTemplateProvisionStrategy().isEnabled()` — in-memory timestamp check.

**No changes needed.**

### `provision(CloudState, int)` — HEAVYWEIGHT (blocks provisioner thread)

This is the primary source of provisioner-thread blocking. Three issues:

#### Issue 1: Synchronous template verification (HEAVYWEIGHT)

```
provision() line 654:
  if (!template.retrieveTemplateProvisionStrategy().isVerifiedPass()) {
      AzureVMCloudVerificationTask.verify(this.name, template.getTemplateName());
  }
```

`verify()` → `verifyCloud()` → `updateCloudVirtualMachineCounts()` → `getVirtualMachineCountsByTemplate()`:
- `azureClient.virtualMachines().listByResourceGroup(resourceGroupName)` — lists ALL VMs in the resource group. Can take seconds.
- `agentTemplate.verifyTemplate()` — may make additional Azure API calls for image verification, network checks, etc.

**Proposed fix:** Do not call `verify()` synchronously from `provision()`. If the template is not yet verified, return empty and let the background `AzureVMCloudVerificationTask` (hourly) or a more frequent background check handle it. Alternatively, cache the verification result with a TTL so repeated `provision()` calls don't re-verify.

#### Issue 2: Per-VM `virtualMachineExists()` for node reuse (HEAVYWEIGHT)

```
provision() line 677:
  if (AzureVMManagementServiceDelegate.virtualMachineExists(agentNode)) {
```

For every offline `AzureVMComputer`, this calls `azureClient.virtualMachines().getByResourceGroup()`. With N offline Azure VMs, this is N sequential Azure API calls on the provisioner thread.

**Proposed fix:** Move the `virtualMachineExists()` check inside the `PlannedNode` future. The reuse path already submits work to `Computer.threadPoolForRemoting` — the existence check should be the first thing the future does, not a synchronous gate in `provision()`. Alternatively, maintain a cached set of known-alive VMs updated by the cleanup task or verification task.

#### Issue 3: `Jenkins.get().getComputers()` iteration for reuse (MODERATE)

The reuse loop at line 667 iterates ALL Jenkins computers (not just Azure ones) to find reuse candidates. With thousands of agents across multiple cloud providers, this iteration itself is O(total_computers) even though each iteration step is cheap.

**Proposed fix:** Low priority. Could maintain a separate index of Azure VMs eligible for reuse, but the iteration itself is fast (in-memory pointer chasing).

### `doProvision(int, List, AzureVMAgentTemplate, boolean)` — LIGHTWEIGHT (DEFERRED)

Submits deployment `Callable` to `AzureVMCloud.getThreadPool()` and creates `PlannedNode` futures on `Computer.threadPoolForRemoting`. Returns immediately.

**Missing:** No `scheduleMaintenance()` after the PlannedNode future resolves.

### `createProvisionedAgent(...)` — HEAVYWEIGHT (runs inside PlannedNode future)

Polls deployment status in a `Thread.sleep(5s)` loop:
- `azureClient.deployments().getByResourceGroup()` per iteration
- `azureClient.virtualMachines().getByResourceGroup()` when VM is ready

This runs inside the PlannedNode future (off the Queue lock), but it ties up a thread pool thread for the entire deployment timeout (potentially 20+ minutes).

**Proposed fix:** Low priority for Queue lock concerns since it's already async. Could improve thread pool utilization by using `ScheduledExecutorService` with periodic callbacks instead of `Thread.sleep()` polling.

### `getThreadPool()` — LIGHTWEIGHT but CONTENDED

`static synchronized` lazy initialization of `Executors.newCachedThreadPool()`. After first initialization, the synchronized block is just a volatile read. Not a Queue lock concern.

---

## 5. AzureVMNoDelayProvisionerStrategy

### `apply(StrategyState)` — HEAVYWEIGHT (inherits from provision())

Calls `cloud.provision()` which has the heavyweight issues described above.

**Fix:** Fixing `AzureVMCloud.provision()` fixes this automatically.

### `AzureVMFastProvisioning.onEnterBuildable(BuildableItem)` — LIGHTWEIGHT

Runs under Queue lock (QueueListener). Only calls `canProvision()` (lightweight) and `suggestReviewNow()` (async trigger).

**No changes needed.**

---

## 6. AzureVMAgentCleanUpTask

### `cleanVMs()` — HEAVYWEIGHT (background, NOT under Queue lock)

- Iterates `Jenkins.get().getComputers()`.
- Calls `AzureVMManagementServiceDelegate.virtualMachineExists(agentNode)` per offline VM — Azure API.
- Calls `Jenkins.get().removeNode(agentNode)` for dead VMs.
- Submits `deprovision`/`shutdown` via `executionEngine.executeAsync()`.

**Not a Queue lock concern** — runs on `AzureVMCloud.getThreadPool()`.

**Missing:** No `scheduleMaintenance()` after `removeNode()` for dead VMs.

### `cleanDeployments()` — HEAVYWEIGHT (background)

Azure deployment API calls. Not a Queue lock concern.

### `cleanLeakedResources()` — HEAVYWEIGHT (background)

Azure resource listing and deletion. Not a Queue lock concern.

---

## 7. AzureVMMaintainPoolTask

### `maintain(AzureVMCloud, AzureVMAgentTemplate)` — MODERATE (background)

- `DynamicBufferCalculator.calculateBufferMetrics()` → iterates `Jenkins.get().getComputers()` — O(total_computers).
- `retentionStrategy.calculateEffectivePoolSize()` — arithmetic.
- `provisionNodes()` → may call `AzureVMCloudVerificationTask.verify()` synchronously (Azure API calls on pool maintainer thread).

**Not a Queue lock concern** — runs on its own timer.

**Missing:** No `scheduleMaintenance()` after `provisionNodes()`.

---

## 8. AzureVMCloudVerificationTask

### `verify(String, String)` — HEAVYWEIGHT (can block provisioner thread)

When called from `AzureVMCloud.provision()`, this runs synchronously on the provisioner thread:
- `verifyCloud()` → `updateCloudVirtualMachineCounts()` → `getVirtualMachineCountsByTemplate()` → `azureClient.virtualMachines().listByResourceGroup()`.
- `agentTemplate.verifyTemplate()` → various Azure API calls for template validation.

When called from the periodic `execute()` method, runs on its own timer (not a Queue lock concern).

**Proposed fix:** Never call `verify()` synchronously from `provision()`. See Issue 1 above.

### `updateCloudVirtualMachineCounts(AzureVMCloud)` — HEAVYWEIGHT

- `getVirtualMachineCountsByTemplate()` → `azureClient.virtualMachines().listByResourceGroup()`.
- `synchronized(cloud)` — holds cloud-level lock during Azure API call.

This method is the primary source of heavyweight work when called from the provisioner path.

---

## 9. AzureVMAgent

### `deprovision(Localizable)` — HEAVYWEIGHT (background threads only)

Called from retention strategy async tasks and cleanup task. Never under Queue lock.

- `isVMAliveOrHealthy()` → polls `getVirtualMachineStatus()` up to 120 times × 10s sleep.
- SSH terminate script execution.
- `terminateVirtualMachine()` → `azureClient.virtualMachines().deleteByResourceGroup()`.
- `Jenkins.get().removeNode(this)`.

**Missing:** No `scheduleMaintenance()` after `removeNode()`.

### `shutdown(Localizable)` — HEAVYWEIGHT (background threads only)

- `serviceDelegate.shutdownVirtualMachine()` → `azureClient.virtualMachines().getByResourceGroup().deallocate()`.
- `setEligibleForReuse(true)`.

**Missing:** No `scheduleMaintenance()` after shutdown.

---

## 10. AzureVMComputer

### `doDoDelete()` — LIGHTWEIGHT (DEFERRED)

Submits deprovision to `executionEngine.executeAsync()`. Returns HTTP redirect immediately.

**No changes needed.**

---

## 11. AzureVMAgentSSHLauncher (remote package)

### Post-connect — BACKGROUND

After successful SSH connection, calls `agent.clearCleanUpAction()`.

**Missing:** No `scheduleMaintenance()` after successful reconnect.

---

## Summary Table

| Class | Method | Context | Classification | Fix Needed? |
|-------|--------|---------|---------------|-------------|
| RetensionStrategy | `check()` | Queue lock | LIGHTWEIGHT | No |
| PoolRetentionStrategy | `check()` | Queue lock | LIGHTWEIGHT | No |
| OnceRetentionStrategy | `check()` | Queue lock | LIGHTWEIGHT | No |
| All strategies | `start()` | Queue lock | LIGHTWEIGHT | No |
| Pool/Once strategies | `taskAccepted/Completed` | Queue lock | LIGHTWEIGHT | No |
| AzureVMFastProvisioning | `onEnterBuildable()` | Queue lock | LIGHTWEIGHT | No |
| AzureVMCloud | `canProvision()` | Provisioner | LIGHTWEIGHT | No |
| AzureVMCloud | `provision()` step 4 | Provisioner | **HEAVYWEIGHT** | **Yes — async verify** |
| AzureVMCloud | `provision()` step 6 | Provisioner | **HEAVYWEIGHT** | **Yes — cache/defer exists check** |
| AzureVMCloud | `doProvision()` | Provisioner | LIGHTWEIGHT (deferred) | scheduleMaintenance trigger |
| AzureVMCloud | `createProvisionedAgent()` | Background | HEAVYWEIGHT (deferred) | Low priority |
| NoDelayStrategy | `apply()` | Provisioner | HEAVYWEIGHT (inherits) | Fixed by provision() fixes |
| CleanUpTask | `cleanVMs()` | Background | HEAVYWEIGHT (deferred) | scheduleMaintenance trigger |
| MaintainPoolTask | `maintain()` | Background | MODERATE (deferred) | scheduleMaintenance trigger |
| VerificationTask | `verify()` | Background / Provisioner | HEAVYWEIGHT | **Yes — don't call from provision()** |
| AzureVMAgent | `deprovision()` | Background | HEAVYWEIGHT (deferred) | scheduleMaintenance trigger |
| AzureVMAgent | `shutdown()` | Background | HEAVYWEIGHT (deferred) | scheduleMaintenance trigger |
| SSHLauncher | post-connect | Background | LIGHTWEIGHT | scheduleMaintenance trigger |
