# SSH Agents Plugin — Queue Lock & Launcher Code Path Audit

Audit of every code path in the ssh-agents-plugin that could affect Jenkins Queue performance.

---

## How Jenkins Invokes ComputerLauncher Code

The ssh-agents-plugin implements `ComputerLauncher` (via `SSHLauncher`) and `ComputerConnector` (via `SSHConnector`). Unlike cloud plugins that implement `RetentionStrategy`, `Cloud`, or `NodeProvisioner.Strategy`, a `ComputerLauncher` has a fundamentally different relationship to the Queue lock:

| Extension Point | Called From | Runs Under Queue Lock? |
|----------------|------------|----------------------|
| `ComputerLauncher.launch()` | `SlaveComputer._connect()` via `Computer.threadPoolForRemoting` | **No** |
| `ComputerLauncher.afterDisconnect()` | `SlaveComputer.disconnect()` via `Computer.threadPoolForRemoting` | **No** |
| `ComputerLauncher.isLaunchSupported()` | UI and reconnect logic | **No** |
| `ComputerConnector.launch()` | Cloud plugins calling `ComputerConnector.launch(host, listener)` | **No** (caller-dependent, typically provisioner thread) |

Jenkins core's `SlaveComputer.connect(boolean)` submits the `_connect()` task to `Computer.threadPoolForRemoting`. Inside `_connect()`, `launcher.launch(this, taskListener)` runs entirely on the remoting thread pool. Similarly, `afterDisconnect()` runs from the disconnect path on the same pool.

**This means the ssh-agents-plugin has zero code paths that execute under the Queue lock.** All SSH work (connection, authentication, file transfer, agent startup) runs on background threads.

---

## Audit Results by Class

### 1. SSHLauncher — SSH ComputerLauncher

**File:** `SSHLauncher.java` (1413 lines)

#### `launch(SlaveComputer, TaskListener)` — REMOTING POOL (not Queue lock)

The main entry point. Called by Jenkins core on `Computer.threadPoolForRemoting`.

| Step | Code (line) | Operation | Verdict |
|------|-------------|-----------|---------|
| 1 | 420 | `computer.getNode()` | In-memory lookup — LIGHTWEIGHT |
| 2 | 423 | `checkConfig()` | Descriptor validation, `Jenkins.get().getDescriptor()`, form validation — LIGHTWEIGHT |
| 3 | 424 | `synchronized (this)` | Serializes concurrent launch attempts per launcher instance — ACCEPTABLE |
| 4 | 429 | `new Connection(host, port)` | Trilead SSH2 object creation — LIGHTWEIGHT |
| 5 | 430–431 | `Executors.newSingleThreadExecutor(...)` | Creates per-launcher thread pool — LIGHTWEIGHT |
| 6 | 436–441 | `getPreferredKeyAlgorithms(computer)` | In-memory algorithm list — LIGHTWEIGHT |
| 7 | 446 | `openConnection(listener, computer)` | **SSH TCP connect + retries + authentication** — **HEAVYWEIGHT** |
| 8 | 448 | `verifyNoHeaderJunk(listener)` | `connection.exec("exit 0", ...)` over SSH — HEAVYWEIGHT |
| 9 | 449 | `reportEnvironment(listener)` | `connection.exec("set", ...)` over SSH — HEAVYWEIGHT |
| 10 | 462 | `copyAgentJar(listener, workingDirectory)` | SFTP/SCP file transfer — **HEAVYWEIGHT** |
| 11 | 464 | `startAgent(computer, listener, java, workingDirectory)` | SSH session + `computer.setChannel()` — **HEAVYWEIGHT** |
| 12 | 466 | `PluginImpl.register(connection)` | `synchronized` add to ArrayList — LIGHTWEIGHT |
| 13 | 493 | `srv.invokeAll(callables)` | Blocks until launch callable completes — HEAVYWEIGHT (blocking wait) |
| 14 | 497 | `results.get(0).get()` | Retrieves launch result — LIGHTWEIGHT (already complete) |
| 15 | 512 | `shutdownAndAwaitTerminationOfLauncher()` | `awaitTermination(10s)` — blocks up to 10s — HEAVYWEIGHT |
| 16 | 515–517 | `CredentialsProvider.track(node, credentials)` | Credential tracking — LIGHTWEIGHT |

**Verdict: HEAVYWEIGHT** — But all operations run on `Computer.threadPoolForRemoting`, never under the Queue lock. No Queue performance impact.

#### `openConnection(TaskListener, SlaveComputer)` — REMOTING POOL

| Step | Code (line) | Operation | Verdict |
|------|-------------|-----------|---------|
| 1 | 852 | `connection.setTCPNoDelay(...)` | Socket option — LIGHTWEIGHT |
| 2 | 858–862 | `connection.connect(verifier, timeout, ...)` | **SSH TCP handshake + host key exchange** — **HEAVYWEIGHT** |
| 3 | 864–882 | Retry loop with `connection.close()` + logging | Error handling — LIGHTWEIGHT per iteration |
| 4 | 883 | `Thread.sleep(TimeUnit.SECONDS.toMillis(getRetryWaitTime()))` | **Blocks 15s default between retries** — **HEAVYWEIGHT** |
| 5 | 886 | `getCredentials()` | Credential lookup from Jenkins store — LIGHTWEIGHT |
| 6 | 890–891 | `SSHAuthenticator.newInstance(...).authenticate(listener)` | **SSH authentication (password/key)** — **HEAVYWEIGHT** |

With default settings (10 retries, 15s wait), a failing connection can block the remoting thread for up to **150 seconds**. This is acceptable because it runs on the remoting pool, not under the Queue lock.

#### `copyAgentJar(TaskListener, String)` — REMOTING POOL

| Step | Code (line) | Operation | Verdict |
|------|-------------|-----------|---------|
| 1 | 685 | `new SFTPClient(connection)` | SFTP subsystem init — HEAVYWEIGHT |
| 2 | 688 | `sftpClient._stat(workingDirectory)` | Remote stat over SFTP — HEAVYWEIGHT |
| 3 | 698 | `new Slave.JnlpJar(AGENT_JAR).readFully()` | Read agent jar from local disk — MODERATE |
| 4 | 702–711 | MD5 hash comparison of existing jar | Remote read + local hash — HEAVYWEIGHT |
| 5 | 723 | `sftpClient.writeToFile(fileName)` | Remote file write over SFTP — HEAVYWEIGHT |
| 6 | 744 | Fallback to `copySlaveJarUsingSCP(...)` | SCP transfer — HEAVYWEIGHT |

#### `startAgent(SlaveComputer, TaskListener, String, String)` — REMOTING POOL

| Step | Code (line) | Operation | Verdict |
|------|-------------|-----------|---------|
| 1 | 631 | `connection.openSession()` | SSH session open — HEAVYWEIGHT |
| 2 | 640 | `session.execCommand(cmd)` | Remote process launch — HEAVYWEIGHT |
| 3 | 645 | `computer.setChannel(stdout, stdin, logger, null)` | Establish remoting channel — HEAVYWEIGHT |

`computer.setChannel()` is the critical moment: Jenkins core's `SlaveComputer._setChannel()` calls `scheduleMaintenance()` internally. This is the point where the Queue first learns about the new executor.

#### `afterDisconnect(SlaveComputer, TaskListener)` — REMOTING POOL

| Step | Code (line) | Operation | Verdict |
|------|-------------|-----------|---------|
| 1 | 939–941 | `connection == null` check | In-memory flag — LIGHTWEIGHT |
| 2 | 943 | `shutdownAndAwaitTerminationOfLauncher()` | Blocks up to 20s — **HEAVYWEIGHT** |
| 3 | 945–953 | `tearingDownConnection` check | Volatile flag — LIGHTWEIGHT |
| 4 | 955 | `tearDownConnection(slaveComputer, listener)` | Synchronized session/connection cleanup — HEAVYWEIGHT |

**Verdict: HEAVYWEIGHT** — But runs on remoting pool. No Queue lock impact.

#### `tearDownConnectionImpl(SlaveComputer, TaskListener)` — REMOTING POOL

| Step | Code (line) | Operation | Verdict |
|------|-------------|-----------|---------|
| 1 | 968 | `reportTransportLoss(connection, listener)` | In-memory cause check — LIGHTWEIGHT |
| 2 | 975 | `getSessionOutcomeMessage(session, connectionLost)` | `session.waitForCondition(..., 3000)` — blocks up to 3s — HEAVYWEIGHT |
| 3 | 976–977 | `session.getStdout().close()` + `session.close()` | SSH session teardown — HEAVYWEIGHT |
| 4 | 984 | `PluginImpl.unregister(connection)` | `synchronized` remove from ArrayList — LIGHTWEIGHT |
| 5 | 985 | `cleanupConnection(listener)` | Async `connection.close()` submit — LIGHTWEIGHT |

#### `shutdownAndAwaitTerminationOfLauncher()` — REMOTING POOL

| Step | Code (line) | Operation | Verdict |
|------|-------------|-----------|---------|
| 1 | 996 | `srv.shutdown()` | Disables new task submission — LIGHTWEIGHT |
| 2 | 1000 | `srv.awaitTermination(10, TimeUnit.SECONDS)` | **Blocks up to 10s** — HEAVYWEIGHT |
| 3 | 1001 | `srv.shutdownNow()` | Interrupts running tasks — LIGHTWEIGHT |
| 4 | 1003 | `srv.awaitTermination(10, TimeUnit.SECONDS)` | **Blocks up to 10s** — HEAVYWEIGHT |

Total potential blocking time: **20 seconds**. Runs on remoting pool only.

#### `cleanupConnection(TaskListener)` — ANY THREAD

| Step | Code (line) | Operation | Verdict |
|------|-------------|-----------|---------|
| 1 | 549 | `Computer.threadPoolForRemoting.submit(_connection::close)` | Non-blocking async submit | LIGHTWEIGHT |
| 2 | 550 | `connection = null` | Field clear — LIGHTWEIGHT |

**Verdict: LIGHTWEIGHT** — Connection close is deferred to remoting pool.

#### `isLaunchSupported()` — ANY THREAD

Returns `true` unconditionally. **Verdict: LIGHTWEIGHT**

---

### 2. SSHConnector — ComputerConnector Factory

**File:** `SSHConnector.java` (362 lines)

#### `launch(String host, TaskListener)` — CALLER THREAD (typically provisioner)

| Step | Code (line) | Operation | Verdict |
|------|-------------|-----------|---------|
| 1 | 180–194 | `new SSHLauncher(...)` + setters | Object construction — LIGHTWEIGHT |

**Verdict: LIGHTWEIGHT** — Pure factory method. Returns immediately.

---

### 3. Host Key Verification Strategies

All verification strategies run inside the `connection.connect()` callback during `openConnection()`, which executes on the remoting pool inside the launch callable.

#### `NonVerifyingKeyVerificationStrategy.verify()` — REMOTING POOL

Always returns `true`. **Verdict: LIGHTWEIGHT**

#### `ManuallyTrustedKeyVerificationStrategy.verify()` — REMOTING POOL

| Step | Operation | Verdict |
|------|-----------|---------|
| 1 | `HostKeyHelper.getInstance().getHostKey(computer)` | WeakHashMap cache check, then file I/O (`XmlFile.read()`) on miss | MODERATE |
| 2 | Key comparison | In-memory byte comparison | LIGHTWEIGHT |
| 3 | `HostKeyHelper.getInstance().saveHostKey(computer, hostKey)` | `XmlFile.write()` — disk I/O | MODERATE |

**Verdict: MODERATE** — File I/O on the remoting pool. No Queue lock impact.

#### `ManuallyTrustedKeyVerificationStrategy.getPreferredKeyAlgorithms()` — REMOTING POOL

| Step | Operation | Verdict |
|------|-----------|---------|
| 1 | `super.getPreferredKeyAlgorithms()` | In-memory algorithm list | LIGHTWEIGHT |
| 2 | `HostKeyHelper.getInstance().getHostKey(computer)` | Cache or file I/O | MODERATE |
| 3 | Algorithm reordering | In-memory list manipulation | LIGHTWEIGHT |

#### `KnownHostsFileKeyVerificationStrategy.verify()` — REMOTING POOL

| Step | Operation | Verdict |
|------|-----------|---------|
| 1 | `KNOWN_HOSTS_FILE.exists()` | Filesystem check | LIGHTWEIGHT |
| 2 | `new KnownHosts(KNOWN_HOSTS_FILE)` | **Parses known_hosts file from disk** | HEAVYWEIGHT |
| 3 | `knownHosts.verifyHostkey(host, algorithm, key)` | In-memory search | LIGHTWEIGHT |

**Verdict: HEAVYWEIGHT** — Parses `~/.ssh/known_hosts` from disk on every verification. Runs on remoting pool only.

#### `KnownHostsFileKeyVerificationStrategy.getPreferredKeyAlgorithms()` — REMOTING POOL

Same pattern: `new KnownHosts(KNOWN_HOSTS_FILE)` re-parses the file. **HEAVYWEIGHT** but on remoting pool.

#### `ManuallyProvidedKeyVerificationStrategy.verify()` — REMOTING POOL

In-memory key comparison only. **Verdict: LIGHTWEIGHT**

---

### 4. PluginImpl — Connection Registry

**File:** `PluginImpl.java` (101 lines)

#### `register(Connection)` / `unregister(Connection)` — ANY THREAD

| Step | Operation | Verdict |
|------|-----------|---------|
| 1 | `synchronized` on `PluginImpl.class` | Class-level lock | Brief |
| 2 | `activeConnections.contains(connection)` / `activeConnections.add/remove(connection)` | ArrayList scan | LIGHTWEIGHT |

**Verdict: LIGHTWEIGHT** — Brief synchronized operations on a small list.

#### `closeRegisteredConnections()` — PLUGIN STOP

Iterates and closes all connections. Only called during plugin shutdown. **Not a runtime concern.**

---

### 5. MissingVerificationStrategyAdministrativeMonitor — Admin Monitor

**File:** `MissingVerificationStrategyAdministrativeMonitor.java` (76 lines)

#### `isActivated()` — ADMIN MONITOR THREAD

| Step | Code (line) | Operation | Verdict |
|------|-------------|-----------|---------|
| 1 | 50 | `Jenkins.get().getComputers()` | Returns array snapshot — LIGHTWEIGHT |
| 2 | 50–59 | Iterate all computers, check launcher type | In-memory instanceof + getter | LIGHTWEIGHT |

**Verdict: LIGHTWEIGHT** — Admin monitor, checked periodically by the UI. Off the hot path entirely.

---

## Summary: Queue Lock Impact

### Under Queue Lock — NONE

The ssh-agents-plugin registers **no** extension points that execute under the Queue lock:

| Extension Point | Registered? |
|----------------|------------|
| `RetentionStrategy` (check/start) | No |
| `Cloud` (canProvision/provision) | No |
| `NodeProvisioner.Strategy` | No |
| `ComputerListener` | No |
| `ExecutorListener` | No |
| `QueueListener` | No |
| `QueueTaskDispatcher` | No |
| `NodeProperty` (canTake) | No |

### On Remoting Pool — All Heavy Work

| Method | Cost | Thread |
|--------|------|--------|
| `SSHLauncher.launch()` | SSH connect + auth + SFTP + exec (seconds to minutes) | `Computer.threadPoolForRemoting` |
| `SSHLauncher.afterDisconnect()` | `awaitTermination(10s)` + session close (up to 23s) | `Computer.threadPoolForRemoting` |
| `openConnection()` | TCP + SSH handshake + retries × sleep (up to 150s) | Per-launcher `ExecutorService` |
| `copyAgentJar()` | SFTP/SCP transfer + MD5 comparison | Per-launcher `ExecutorService` |
| `startAgent()` | SSH session + `computer.setChannel()` | Per-launcher `ExecutorService` |
| Host key verification | File I/O (known_hosts, XML) | Per-launcher `ExecutorService` |

### Missing `scheduleMaintenance()` Triggers

The plugin calls **zero** `scheduleMaintenance()`. While Jenkins core's `SlaveComputer._setChannel()` calls it when the channel is established, the plugin does not trigger it for:

| Event | Current Behavior | Recommendation |
|-------|-----------------|----------------|
| SSH agent comes online | Relies on core `_setChannel()` only | Add `ComputerListener.onOnline()` — fires after all listeners complete |
| SSH agent goes offline | No notification | Add `ComputerListener.onOffline()` — triggers re-evaluation for replacement provisioning |
| SSH agent temporarily online | No notification | Add `ComputerListener.onTemporarilyOnline()` |
| SSH agent temporarily offline | No notification | Add `ComputerListener.onTemporarilyOffline()` |

---

## References

- `SSHLauncher.java` — Main launcher implementation
- `SSHConnector.java` — ComputerConnector factory
- `PluginImpl.java` — Connection registry
- `verifiers/` — Host key verification strategies
- Jenkins core `SlaveComputer._connect()` — Calls `launcher.launch()` on remoting pool
- Jenkins core `SlaveComputer._setChannel()` — Calls `scheduleMaintenance()` when channel established
