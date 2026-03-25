# SSH Agents Plugin — Queue Performance Patches

Patches against `origin/master` (`jenkinsci/ssh-agents-plugin`). 1 new file, ~40 lines.

---

## How This Relates to Queue Performance

The ssh-agents-plugin is an SSH `ComputerLauncher` implementation — it handles the SSH connection protocol for launching Jenkins agents. Unlike the [ec2-plugin](../ec2-plugin/) (which had `RetentionStrategy` code making AWS API calls under the Queue lock) or the [job-restrictions-plugin](../job-restrictions-plugin/) (which evaluated restrictions in the innermost Queue maintenance loop), the ssh-agents-plugin has **zero code paths that execute under the Queue lock**.

All heavy SSH work — TCP connection with retries, host key verification, authentication, SFTP/SCP file transfer, remote process launch, and disconnect cleanup — runs on `Computer.threadPoolForRemoting` or a per-launcher `ExecutorService`. This is the correct architecture for a `ComputerLauncher`: Jenkins core calls `launcher.launch()` from `SlaveComputer._connect()` on the remoting pool, never under the Queue lock.

The plugin's only Queue performance gap is the absence of `scheduleMaintenance()` triggers. When an SSH agent comes online or goes offline, the Queue should re-evaluate immediately rather than waiting for the next 5-second timer tick. Now that Queue maintenance completes in milliseconds (thanks to the [ec2-plugin](../ec2-plugin/), [leastload-plugin](../leastload-plugin/), and [job-restrictions-plugin](../job-restrictions-plugin/) patches), triggering it on every capacity-changing event is essentially free.

---

## Files Changed

### SSHComputerListener.java — Queue Maintenance Triggers (NEW)

A `@Extension ComputerListener` that triggers `scheduleMaintenance()` when SSH-launched agents change state.

- **`onOnline(Computer, TaskListener)`** — Calls `scheduleMaintenance()` when an SSH agent is fully online. While Jenkins core's `SlaveComputer._setChannel()` already calls `scheduleMaintenance()`, the `onOnline` callback fires *after* all `ComputerListener`s have run — making it a more reliable "everything ready" signal. This is the same pattern used by the ec2-plugin's `EC2ComputerListener.onOnline()`.
- **`onOffline(Computer, OfflineCause)`** — Calls `scheduleMaintenance()` when an SSH agent goes offline. This ensures the Queue immediately re-evaluates: cloud plugins can provision replacements, and queued items can be rerouted to other available executors.
- **`onTemporarilyOnline(Computer)`** — Calls `scheduleMaintenance()` when an SSH agent is brought back online after being temporarily taken offline. The restored executor capacity should be utilized immediately.
- **`onTemporarilyOffline(Computer, OfflineCause)`** — Calls `scheduleMaintenance()` when an SSH agent is temporarily taken offline. Queued items targeting this agent's labels should be rerouted immediately.

All callbacks filter on `SlaveComputer` instances using `SSHLauncher` to avoid firing for non-SSH agents. Since `scheduleMaintenance()` submits to `AtmostOneTaskExecutor`, duplicate calls coalesce — multiple triggers in quick succession cause at most one `maintain()` run.

---

## Existing Code — No Changes Needed

All existing code paths are already off the Queue lock hot path:

| Path | Type | Thread | Status |
|------|------|--------|--------|
| `SSHLauncher.launch()` | HEAVYWEIGHT | `Computer.threadPoolForRemoting` | Acceptable — SSH connect, auth, SFTP, exec |
| `SSHLauncher.afterDisconnect()` | HEAVYWEIGHT | `Computer.threadPoolForRemoting` | Acceptable — session/connection teardown |
| `SSHLauncher.openConnection()` | HEAVYWEIGHT | Per-launcher `ExecutorService` | Acceptable — TCP + retries + authentication |
| `SSHLauncher.copyAgentJar()` | HEAVYWEIGHT | Per-launcher `ExecutorService` | Acceptable — SFTP/SCP file transfer |
| `SSHLauncher.startAgent()` | HEAVYWEIGHT | Per-launcher `ExecutorService` | Acceptable — remote exec + channel setup |
| Host key verification strategies | MODERATE | Per-launcher `ExecutorService` | Acceptable — file I/O for known_hosts/XML |
| `SSHLauncher.cleanupConnection()` | LIGHTWEIGHT | Any (async submit) | Acceptable — non-blocking |
| `SSHLauncher.isLaunchSupported()` | LIGHTWEIGHT | Any | Returns `true` |
| `SSHConnector.launch()` | LIGHTWEIGHT | Caller | Object construction only |
| `PluginImpl.register/unregister()` | LIGHTWEIGHT | Any | Brief synchronized ArrayList ops |
| `MissingVerificationStrategyAdministrativeMonitor` | LIGHTWEIGHT | Admin monitor | Off hot path |

---

## Background Analysis

- [SSH_QUEUE_AUDIT.md](background/SSH_QUEUE_AUDIT.md) — Full audit of every code path with execution context and verdict
- [SSH_HEAVY_LIGHTWEIGHT_ANALYSIS.md](background/SSH_HEAVY_LIGHTWEIGHT_ANALYSIS.md) — Heavy vs lightweight classification and comparison with other audited plugins
