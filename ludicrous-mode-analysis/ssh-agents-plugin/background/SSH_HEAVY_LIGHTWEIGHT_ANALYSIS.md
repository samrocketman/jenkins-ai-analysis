# SSH Agents Plugin: Heavy vs Lightweight Classification

**Date:** March 2026
**Goal:** Classify all ssh-agents-plugin operations to ensure Queue maintenance is as fast as possible, and identify where `scheduleMaintenance()` triggers should be added.

---

## Executive Summary

The ssh-agents-plugin is a `ComputerLauncher` implementation, not a cloud or retention strategy plugin. It has **zero code paths under the Queue lock**. All heavy SSH work (connection, authentication, file transfer, agent startup, disconnect cleanup) runs on `Computer.threadPoolForRemoting` or a per-launcher `ExecutorService`.

The only recommended change is adding a `ComputerListener` to trigger `scheduleMaintenance()` on SSH agent online/offline events, so the Queue re-evaluates immediately when executor capacity changes.

---

## 1. Classification: Under Queue Lock

**There are no ssh-agents-plugin code paths under the Queue lock.**

The plugin does not implement any extension points called by `Queue.maintain()`:

| Extension Point | Called Under Queue Lock | Implemented by ssh-agents-plugin? |
|----------------|----------------------|----------------------------------|
| `RetentionStrategy.check()` | Yes | No |
| `RetentionStrategy.start()` | Yes | No |
| `Cloud.canProvision()` | No (provisioner thread) | No |
| `Cloud.provision()` | No (provisioner thread) | No |
| `NodeProvisioner.Strategy.apply()` | No (provisioner thread) | No |
| `Node.canTake()` / `NodeProperty.canTake()` | Yes | No |
| `QueueTaskDispatcher.canRun()` / `canTake()` | Yes | No |
| `ComputerListener` callbacks | No | **Not yet** (recommended) |
| `ExecutorListener` callbacks | Varies | No |
| `QueueListener` callbacks | No | No |

---

## 2. Classification: Remoting Pool (Heavyweight — Acceptable)

These operations are heavyweight but run on background threads, never blocking Queue maintenance.

### 2.1 SSHLauncher.launch() — Full Launch Sequence

**Thread:** `Computer.threadPoolForRemoting` → per-launcher `ExecutorService`

| Operation | Location | Cost | Blocking? |
|-----------|----------|------|-----------|
| `openConnection()` — SSH TCP connect | SSHLauncher.java:858 | Network I/O, up to `launchTimeoutMillis` (default 60s) | Yes — TCP + SSH handshake |
| `connection.connect()` retries | SSHLauncher.java:855–883 | Up to `maxNumRetries` × `retryWaitTime` (default 10 × 15s = 150s) | Yes — `Thread.sleep()` between retries |
| SSH authentication | SSHLauncher.java:890 | `SSHAuthenticator.authenticate()` — password or key auth | Yes — network round-trip |
| `verifyNoHeaderJunk()` | SSHLauncher.java:601 | `connection.exec("exit 0", ...)` | Yes — remote command |
| `reportEnvironment()` | SSHLauncher.java:843 | `connection.exec("set", ...)` | Yes — remote command |
| `copyAgentJar()` — SFTP path | SSHLauncher.java:679 | SFTP stat + read + MD5 + write (agent jar ~1.5MB) | Yes — file transfer |
| `copySlaveJarUsingSCP()` — SCP fallback | SSHLauncher.java:818 | SCP transfer when SFTP unavailable | Yes — file transfer |
| `startAgent()` — remote exec | SSHLauncher.java:629 | `session.execCommand()` + `computer.setChannel()` | Yes — process launch + channel setup |
| `invokeAll(callables)` | SSHLauncher.java:493 | Blocks until launch callable completes | Yes — waits for all above |
| `shutdownAndAwaitTerminationOfLauncher()` | SSHLauncher.java:991 | `awaitTermination(10s)` × 2 | Yes — up to 20s |

**Total worst-case duration:** Minutes (dominated by retry loop). All on remoting pool.

### 2.2 SSHLauncher.afterDisconnect() — Disconnect Cleanup

**Thread:** `Computer.threadPoolForRemoting`

| Operation | Location | Cost | Blocking? |
|-----------|----------|------|-----------|
| `shutdownAndAwaitTerminationOfLauncher()` | SSHLauncher.java:943 | Up to 20s | Yes |
| `session.waitForCondition(..., 3000)` | SSHLauncher.java:1031 | Up to 3s | Yes |
| `session.getStdout().close()` / `session.close()` | SSHLauncher.java:976–977 | SSH session teardown | Yes — network |
| `PluginImpl.unregister(connection)` | SSHLauncher.java:984 | Synchronized ArrayList remove | Brief |
| `cleanupConnection()` → async close | SSHLauncher.java:985 | Non-blocking submit | No |

**Total worst-case duration:** ~23 seconds. All on remoting pool.

### 2.3 Host Key Verification

**Thread:** Per-launcher `ExecutorService` (inside launch callable)

| Strategy | Cost | Blocking? |
|----------|------|-----------|
| `NonVerifyingKeyVerificationStrategy` | Returns `true` immediately | No |
| `ManuallyProvidedKeyVerificationStrategy` | In-memory key comparison | No |
| `ManuallyTrustedKeyVerificationStrategy` | File I/O: `XmlFile.read()` on cache miss, `XmlFile.write()` on new key | Yes — disk I/O |
| `KnownHostsFileKeyVerificationStrategy` | **Parses `~/.ssh/known_hosts` from disk** on every call | Yes — disk I/O |

`KnownHostsFileKeyVerificationStrategy` re-reads and re-parses the known_hosts file on every SSH connection and every `getPreferredKeyAlgorithms()` call. With many agents, this could benefit from caching, but it runs on the remoting pool so it does not affect Queue performance.

---

## 3. Classification: Lightweight

These operations are fast and have no Queue performance impact.

| Operation | Location | Cost |
|-----------|----------|------|
| `SSHLauncher.isLaunchSupported()` | SSHLauncher.java:358 | Returns `true` — constant |
| `SSHLauncher.getCredentials()` | SSHLauncher.java:334 | In-memory credential lookup |
| `SSHLauncher.cleanupConnection()` | SSHLauncher.java:545 | Async submit of `connection.close()` to remoting pool |
| `SSHConnector.launch(host, listener)` | SSHConnector.java:179 | Object construction — `new SSHLauncher(...)` |
| `PluginImpl.register(connection)` | PluginImpl.java:82 | Synchronized ArrayList add |
| `PluginImpl.unregister(connection)` | PluginImpl.java:93 | Synchronized ArrayList remove |
| `PluginImpl.start()` | PluginImpl.java:49 | Log message |
| `MissingVerificationStrategyAdministrativeMonitor.isActivated()` | MissingVerificationStrategyAdministrativeMonitor.java:48 | Iterates computers array, instanceof checks |
| `SSHLauncher.readResolve()` | SSHLauncher.java:314 | Field defaults |
| `SSHLauncher.logConfiguration()` | SSHLauncher.java:1391 | StringBuilder |
| All `DescriptorImpl` form validation methods | SSHLauncher.java:1228+, SSHConnector.java:303+ | UI validation, off hot path |

---

## 4. Missing: scheduleMaintenance() Triggers

The ssh-agents-plugin currently fires **zero** `scheduleMaintenance()` calls. Jenkins core's `SlaveComputer._setChannel()` calls it when the remoting channel is established, but no other lifecycle events trigger queue re-evaluation.

### Events That Should Trigger scheduleMaintenance()

| Event | Why It Matters | Currently Triggered? |
|-------|---------------|---------------------|
| SSH agent comes online | New executor capacity available for queued work | Partially — `_setChannel()` calls it in core, but `onOnline` fires later after all listeners complete |
| SSH agent goes offline | Executor capacity lost; Queue should re-evaluate for replacement provisioning | No |
| SSH agent temporarily online | Executor capacity restored | No |
| SSH agent temporarily offline | Executor capacity temporarily unavailable | No |

### Recommended Implementation

Add a new `@Extension ComputerListener` that triggers `scheduleMaintenance()` for SSH-launched agents:

**File:** `src/main/java/hudson/plugins/sshslaves/SSHComputerListener.java`

The listener should filter on `SlaveComputer` instances using `SSHLauncher`, and call `Jenkins.get().getQueue().scheduleMaintenance()` on:
- `onOnline(Computer, TaskListener)` — fires after all ComputerListeners complete; definitive "ready for work" signal
- `onOffline(Computer, OfflineCause)` — capacity lost; triggers re-evaluation
- `onTemporarilyOnline(Computer)` — capacity restored
- `onTemporarilyOffline(Computer, OfflineCause)` — capacity temporarily unavailable

Since `scheduleMaintenance()` submits to `AtmostOneTaskExecutor`, duplicate calls coalesce — multiple triggers in quick succession do not cause multiple `maintain()` runs.

---

## 5. Comparison with Other Audited Plugins

| Plugin | Queue Lock Paths | Provisioner Blocking | Missing scheduleMaintenance() |
|--------|-----------------|---------------------|------------------------------|
| **ec2-plugin** (before patch) | HEAVY — `check()`/`start()` made EC2 API calls under lock | HEAVY — `provision()` blocked on `runInstances` | Yes — 3 triggers added |
| **azure-vm-agents-plugin** | LIGHTWEIGHT — all check/start defer to background | HEAVY — `provision()` calls `verify()` synchronously | Yes — 13 missing triggers identified |
| **job-restrictions-plugin** (before patch) | HEAVY — `canTake()` evaluated restrictions per (node, item) pair | N/A | N/A |
| **ssh-agents-plugin** | **NONE** — no Queue lock code paths | N/A — not a provisioner | Yes — 4 triggers recommended |

---

## 6. References

- [SSH_QUEUE_AUDIT.md](SSH_QUEUE_AUDIT.md) — Detailed per-method audit
- `SSHLauncher.java` — Main launcher implementation (1413 lines)
- `SSHConnector.java` — ComputerConnector factory (362 lines)
- `PluginImpl.java` — Connection registry (101 lines)
- `verifiers/` — 7 host key verification files
- Jenkins core `SlaveComputer._connect()` — Calls `launcher.launch()` on remoting pool
- Jenkins core `SlaveComputer._setChannel()` — Calls `scheduleMaintenance()` on channel establishment
- [EC2 Plugin Audit](../ec2-plugin/background/EC2_QUEUE_AUDIT.md) — Comparable audit for cloud plugin
- [Azure VM Agents Audit](../azure-vm-agents-plugin/background/AZURE_QUEUE_AUDIT.md) — Comparable audit for cloud plugin
