# Azure VM Agents Plugin — Queue Lock & Provisioner Audit

Audit of every code path in the azure-vm-agents-plugin that runs under the Jenkins Queue lock or blocks the NodeProvisioner thread.

---

## How Jenkins Invokes Plugin Code

Jenkins core's `Queue.maintain()` runs under a global `ReentrantLock` every 5 seconds (configurable via `hudson.model.Queue.maintainInterval`). During this cycle, the following plugin extension points are called **while holding the Queue lock**:

| Extension Point | Called From | Frequency |
|----------------|------------|-----------|
| `RetentionStrategy.check(Computer)` | `Queue.maintain()` → `Computer.getRetentionStrategy().check()` | Once per Computer per cycle |
| `RetentionStrategy.start(Computer)` | `Queue.maintain()` → when Computer is first created | Once per Computer lifecycle |
| `ExecutorListener.taskAccepted()` | Executor starts a task | Per task start |
| `ExecutorListener.taskCompleted()` | Executor finishes a task | Per task completion |
| `QueueListener.onEnterBuildable()` | Item becomes buildable | Per queue item |

The `NodeProvisioner` runs on its **own** timer thread (not under the Queue lock). It calls:

| Extension Point | Called From | Lock? |
|----------------|------------|-------|
| `Cloud.canProvision(CloudState)` | `NodeProvisioner.Strategy.apply()` | No Queue lock |
| `Cloud.provision(CloudState, int)` | `NodeProvisioner.Strategy.apply()` | No Queue lock |
| `NodeProvisioner.Strategy.apply()` | `NodeProvisioner.update()` | No Queue lock |

Although `provision()` does not hold the Queue lock, it blocks the provisioner thread. A slow `provision()` delays all cloud provisioning decisions.

---

## Audit Results by Class

### 1. AzureVMCloudRetensionStrategy — Idle Timeout Retention

**File:** `AzureVMCloudRetensionStrategy.java`

#### `check(AzureVMComputer)` — UNDER QUEUE LOCK

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | `agentNode.isIdle()` | In-memory flag | LIGHTWEIGHT |
| 2 | `agentNode.isOnline()` | In-memory flag | LIGHTWEIGHT |
| 3 | `idleTerminationMillis > 0` | Field comparison | LIGHTWEIGHT |
| 4 | `System.currentTimeMillis() - agentNode.getIdleStartMilliseconds()` | Timestamp math | LIGHTWEIGHT |
| 5 | `agentNode.getNode()` | In-memory lookup | LIGHTWEIGHT |
| 6 | `executionEngine.executeAsync(task, ...)` | Submits Callable to `AzureVMCloud.getThreadPool()` | LIGHTWEIGHT (non-blocking) |
| 7 | Return `1` | Constant | LIGHTWEIGHT |

**Heavy work deferred:** The Callable inside `executeAsync` calls `agent.shutdown()` → `serviceDelegate.shutdownVirtualMachine()` → `azureClient.virtualMachines().getByResourceGroup().deallocate()`, or `agent.deprovision()` → `terminateVirtualMachine()` → `azureClient.virtualMachines().deleteByResourceGroup()`. All Azure API calls run on the background thread pool.

**Verdict: LIGHTWEIGHT** — Only boolean checks and async task submission under the Queue lock.

#### `start(AzureVMComputer)` — UNDER QUEUE LOCK

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | `azureComputer.connect(false)` | Returns `Future<?>` immediately | LIGHTWEIGHT |
| 2 | `resetShutdownVMStatus(azureComputer.getNode())` | Submits to `Computer.threadPoolForRemoting` | LIGHTWEIGHT |

**Verdict: LIGHTWEIGHT** — `connect(false)` is non-blocking (launches connection asynchronously). `resetShutdownVMStatus` submits to remoting pool.

---

### 2. AzureVMCloudPoolRetentionStrategy — Pool Retention

**File:** `AzureVMCloudPoolRetentionStrategy.java`

#### `check(AzureVMComputer)` — UNDER QUEUE LOCK

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | `agentComputer.getNode()` | In-memory | LIGHTWEIGHT |
| 2 | `agentNode.getCloud()` | `Jenkins.get().getCloud(name)` — in-memory lookup | LIGHTWEIGHT |
| 3 | Template iteration (`currentCloud.getVmTemplates()`) | Iterates `CopyOnWriteArrayList` | LIGHTWEIGHT (O(templates)) |
| 4 | `TemplateUtil.checkSame(template, agentNode.getTemplate())` | String comparisons | LIGHTWEIGHT |
| 5 | Retention time check | Timestamp comparison | LIGHTWEIGHT |
| 6 | `Computer.threadPoolForRemoting.submit(...)` | Async submit of `tryDeleteWhenIdle` or `checkPoolSizeAndDelete` | LIGHTWEIGHT |
| 7 | Return `1` | Constant | LIGHTWEIGHT |

**Heavy work deferred:** `tryDeleteWhenIdle()` → `agent.deprovision()` and `checkPoolSizeAndDelete()` → `Jenkins.get().getComputers()` iteration both run on `Computer.threadPoolForRemoting`.

**Verdict: LIGHTWEIGHT** — All Azure API calls and node iteration deferred to background threads.

#### `start(AzureVMComputer)` — UNDER QUEUE LOCK

Same pattern as `AzureVMCloudRetensionStrategy.start()`.

**Verdict: LIGHTWEIGHT**

#### `taskAccepted(Executor, Queue.Task)` — UNDER QUEUE LOCK (ExecutorListener)

Empty method body. **Verdict: LIGHTWEIGHT**

#### `taskCompleted(Executor, Queue.Task, long)` — UNDER QUEUE LOCK (ExecutorListener)

Calls `done(executor)` → `done(AzureVMComputer)` which sets `computer.setAcceptingTasks(false)` and `agent.setCleanUpAction(...)`. All in-memory flag operations.

**Verdict: LIGHTWEIGHT**

---

### 3. AzureVMCloudOnceRetentionStrategy — Single-Use Retention

**File:** `AzureVMCloudOnceRetentionStrategy.java`

#### `check(AzureVMComputer)` — UNDER QUEUE LOCK

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | `agentComputer.isIdle()` | In-memory flag | LIGHTWEIGHT |
| 2 | Timestamp calculations | `getIdleStartMilliseconds()`, `getConnectTime()` | LIGHTWEIGHT |
| 3 | `done(agentComputer)` | Sets `setAcceptingTasks(false)`, `setCleanUpAction(...)` | LIGHTWEIGHT |

**Verdict: LIGHTWEIGHT** — Pure in-memory flag checks and sets.

#### `start(AzureVMComputer)` — UNDER QUEUE LOCK

Same pattern as other strategies.

**Verdict: LIGHTWEIGHT**

#### `taskAccepted` / `taskCompleted` — UNDER QUEUE LOCK (ExecutorListener)

Same as pool strategy: empty `taskAccepted`, `done()` sets flags.

**Verdict: LIGHTWEIGHT**

---

### 4. AzureVMCloud — Cloud Implementation

**File:** `AzureVMCloud.java`

#### `canProvision(CloudState)` — PROVISIONER THREAD (no Queue lock)

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | `getAzureAgentTemplate(label)` | Iterates `vmTemplates` list, label matching | LIGHTWEIGHT |
| 2 | `template.isTemplateDisabled()` | Boolean flag | LIGHTWEIGHT |
| 3 | `template.retrieveTemplateProvisionStrategy().isEnabled()` | In-memory timestamp check | LIGHTWEIGHT |

**Verdict: LIGHTWEIGHT**

#### `provision(CloudState, int)` — PROVISIONER THREAD (no Queue lock, but blocks provisioner)

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | `Jenkins.get().isQuietingDown()` | In-memory flag | LIGHTWEIGHT |
| 2 | `getAzureAgentTemplate(label)` | Template lookup | LIGHTWEIGHT |
| 3 | `template.retrieveTemplateProvisionStrategy().isVerifiedPass()` | In-memory | LIGHTWEIGHT |
| 4 | **`AzureVMCloudVerificationTask.verify()`** | Synchronous call that may invoke `updateCloudVirtualMachineCounts()` → `getVirtualMachineCountsByTemplate()` → `azureClient.virtualMachines().listByResourceGroup()` | **HEAVYWEIGHT** |
| 5 | **`Jenkins.get().getComputers()` iteration for node reuse** | Iterates all computers | MODERATE (O(computers)) |
| 6 | **`AzureVMManagementServiceDelegate.virtualMachineExists(agentNode)`** per reuse candidate | `azureClient.virtualMachines().getByResourceGroup()` per VM | **HEAVYWEIGHT** (O(offline_azure_vms) API calls) |
| 7 | `Computer.threadPoolForRemoting.submit(reuse future)` | Async (reuse work runs in future) | LIGHTWEIGHT |
| 8 | `synchronized(this) { calculateNumberOfAgentsToRequest(...) }` | In-memory count checks | LIGHTWEIGHT |
| 9 | `doProvision(...)` | Submits deployment to thread pool, creates PlannedNode futures | LIGHTWEIGHT |

**Verdict: HEAVYWEIGHT** — Steps 4 and 6 make synchronous Azure API calls that block the provisioner thread:
- Template verification lists ALL VMs in the resource group
- Node reuse checks call `getByResourceGroup()` per eligible offline VM

#### `doProvision(int, List, AzureVMAgentTemplate, boolean)` — PROVISIONER THREAD

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | `getThreadPool().submit(callableTask)` | Submits deployment to thread pool | LIGHTWEIGHT |
| 2 | `Computer.threadPoolForRemoting.submit(PlannedNode future)` | Submits per-VM provisioning future | LIGHTWEIGHT |

The heavy work (ARM deployment, `createProvisionedAgent` polling, `addNode`, `connect`) runs entirely inside the futures.

**Verdict: LIGHTWEIGHT** (the method itself returns immediately)

---

### 5. AzureVMNoDelayProvisionerStrategy — NodeProvisioner Strategy

**File:** `AzureVMNoDelayProvisionerStrategy.java`

#### `apply(StrategyState)` — PROVISIONER THREAD (no Queue lock)

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | `Jenkins.get().clouds` iteration | In-memory | LIGHTWEIGHT |
| 2 | `cloud.canProvision(cloudState)` | In-memory (see above) | LIGHTWEIGHT |
| 3 | `cloud.provision(cloudState, workload)` | See `provision()` above | **HEAVYWEIGHT** (inherits provision issues) |
| 4 | `Timer.get().schedule(suggestReviewNow, 1s)` | Async scheduling | LIGHTWEIGHT |

**Verdict: HEAVYWEIGHT** — Inherits all issues from `AzureVMCloud.provision()`.

#### `AzureVMFastProvisioning.onEnterBuildable(BuildableItem)` — UNDER QUEUE LOCK (QueueListener)

| Step | Code | Operation | Verdict |
|------|------|-----------|---------|
| 1 | `jenkins.clouds` iteration | In-memory | LIGHTWEIGHT |
| 2 | `cloud.canProvision(...)` | In-memory (see above) | LIGHTWEIGHT |
| 3 | `provisioner.suggestReviewNow()` | Triggers review on provisioner thread | LIGHTWEIGHT |

**Verdict: LIGHTWEIGHT** — Only in-memory checks and async triggers. Does NOT call `provision()`.

---

### 6. AzureVMAgentCleanUpTask — Periodic Cleanup (AsyncPeriodicWork, 5 min)

**File:** `AzureVMAgentCleanUpTask.java`

Runs on its own timer via `AzureVMCloud.getThreadPool()`. Does NOT run under the Queue lock.

| Method | Azure API Calls | Queue Lock? |
|--------|----------------|-------------|
| `cleanVMs()` | `virtualMachineExists()` per offline VM, then async `deprovision`/`shutdown` | No |
| `cleanDeployments()` | `deployments().getByResourceGroup()`, `deleteByResourceGroup()` | No |
| `cleanLeakedResources()` | `genericResources().listByResourceGroup()`, `virtualMachines().getById()`, `deleteById()` | No |

**Verdict: NOT A QUEUE LOCK CONCERN** — All work runs on background thread pool.

However, `cleanVMs()` calls `Jenkins.get().removeNode(agentNode)` which modifies the node list but does NOT trigger `scheduleMaintenance()`.

---

### 7. AzureVMMaintainPoolTask — Pool Maintenance (AsyncPeriodicWork, 5 min)

**File:** `AzureVMMaintainPoolTask.java`

Runs on its own timer. Does NOT run under the Queue lock.

| Method | Operation | Queue Lock? |
|--------|-----------|-------------|
| `maintain()` | `DynamicBufferCalculator.calculateBufferMetrics()` → iterates `Jenkins.get().getComputers()` | No |
| `provisionNodes()` | `AzureVMCloudVerificationTask.verify()` then `cloud.doProvision()` | No |

**Verdict: NOT A QUEUE LOCK CONCERN** — But `verify()` makes synchronous Azure API calls on the pool maintainer thread, slowing pool replenishment.

---

### 8. AzureVMCloudVerificationTask — Periodic Verification (AsyncPeriodicWork, 1 hour)

**File:** `AzureVMCloudVerificationTask.java`

Runs hourly on its own timer AND called synchronously from `provision()`.

| Method | Azure API Calls | Queue Lock? |
|--------|----------------|-------------|
| `verify()` | `verifyConfiguration()` (string validation only), `updateCloudVirtualMachineCounts()` → `getVirtualMachineCountsByTemplate()` → `listByResourceGroup()`, `agentTemplate.verifyTemplate()` → multiple Azure API calls | No (when periodic), but **called synchronously from provision()** on the provisioner thread |

**Verdict: HEAVYWEIGHT when called from provision()** — Blocks the provisioner thread with Azure API calls.

---

### 9. AzureVMAgent — Slave Implementation

**File:** `AzureVMAgent.java`

#### `deprovision(Localizable)` — CALLED FROM BACKGROUND THREADS

| Step | Azure API Calls | Notes |
|------|----------------|-------|
| `isVMAliveOrHealthy()` | `getVirtualMachineStatus()` → `azureClient.virtualMachines().getByResourceGroup()` | Polls in a loop (up to 120 retries × 10s) |
| SSH terminate script execution | SSH commands via `AzureVMAgentSSHLauncher` | Network I/O |
| `computer.disconnect()` | Channel disconnect | Network I/O |
| `terminateVirtualMachine()` | `azureClient.virtualMachines().deleteByResourceGroup()`, disk cleanup | Azure API |
| `Jenkins.get().removeNode(this)` | Modifies Jenkins node list | No `scheduleMaintenance()` call |

**Verdict: HEAVYWEIGHT** — But called from background threads (retention async tasks, cleanup task), never under Queue lock.

**Missing:** No `scheduleMaintenance()` after `removeNode()`.

#### `shutdown(Localizable)` — CALLED FROM BACKGROUND THREADS

| Step | Azure API Calls |
|------|----------------|
| `serviceDelegate.shutdownVirtualMachine(this)` | `azureClient.virtualMachines().getByResourceGroup().deallocate()` |

**Verdict: HEAVYWEIGHT** — But called from background threads only.

**Missing:** No `scheduleMaintenance()` after `setEligibleForReuse(true)`.

---

## Summary

### Under Queue Lock — All Lightweight

| Method | Class | Verdict |
|--------|-------|---------|
| `check()` | AzureVMCloudRetensionStrategy | LIGHTWEIGHT |
| `check()` | AzureVMCloudPoolRetentionStrategy | LIGHTWEIGHT |
| `check()` | AzureVMCloudOnceRetentionStrategy | LIGHTWEIGHT |
| `start()` | All three strategies | LIGHTWEIGHT |
| `taskAccepted()` | Pool + Once strategies | LIGHTWEIGHT |
| `taskCompleted()` | Pool + Once strategies | LIGHTWEIGHT |
| `onEnterBuildable()` | AzureVMFastProvisioning | LIGHTWEIGHT |

### Blocking Provisioner Thread — Heavyweight Issues

| Method | Class | Issue |
|--------|-------|-------|
| `provision()` step 4 | AzureVMCloud | Synchronous `verify()` → `listByResourceGroup()` |
| `provision()` step 6 | AzureVMCloud | Per-VM `virtualMachineExists()` → `getByResourceGroup()` × N |

### Missing scheduleMaintenance() Triggers

| Location | Event |
|----------|-------|
| No `ComputerListener` exists | Agent comes online |
| `doProvision()` PlannedNode future | New VM provisioned and connected |
| `provision()` reuse path | Reused VM back online |
| `deprovision()` | Agent removed from Jenkins |
| `shutdown()` | Agent eligible for reuse |
| `cleanVMs()` | Dead node removed |
| `checkPoolSizeAndDelete()` | Excess VMs deleted |
| `provisionNodes()` | Pool adding capacity |
| `AzureVMAgentSSHLauncher` | Agent reconnected |
