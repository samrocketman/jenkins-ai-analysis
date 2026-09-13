# Parallel Pipeline Stall Analysis

**Job:** `cicd-examples/hello-world-groovy/PR-338#29` (and later 100-way reproductions of the same Jenkinsfile)
**Controller:** Jenkins 2.555.1
**Date of investigation:** 12 September 2026
**Reproducer:** `.ci/Jenkinsfile` in this repo, via Jervis `pullRequestAndTagPipeline`

This report covers the full investigation: the pre-`node()` dead spot, the later “100 agents assigned but still slow” phase, thread-dump evidence, shared-library occupancy, plugin bugs, patches produced in this workspace, and remaining work.

---

## 1. Executive summary

There are **two separate bottlenecks**, both caused by the same architectural fact: a Pipeline build has **one CPS VM thread**. Anything that does I/O, process waits, Groovy parsing, or a full flow-graph walk on that thread serializes every parallel branch.

1. **Before agents are requested (20–60s dead spot).** The Blue Ocean / pipeline-graph UI draws all parallel branches immediately. `node()` has not run yet, so the EC2 plugin cannot see queued work. The CPS runner is occupied by GitHub Checks progress publishing (`checks-api` `ChecksGraphListener` → `github-checks` HTTP) and, originally, by shared-library GitHub commit statuses inside `stage()`. Enabling **“Suppress progress updates in job check”** (`skipProgressUpdates`) removed this gap.

2. **After 100 agents are assigned (~2 minutes for a trivial parallel block).** Consecutive `sh 'echo'` steps look slow, but the shell is not the problem. The CPS runner is blocked inside `docker.inside` / `sshagent` start (and teardown): both plugins wait on `Proc.joinWithTimeout` in `StepExecution.start()`, which the Step API forbids. Durable `sh` *inside* the container yields; the next branch cannot run until the current branch finishes `docker run` / `ssh-agent` spawn.

Patches in this workspace:

| Plugin (installed version) | Clone | Change | Status |
|---|---|---|---|
| `github-checks` `679.v74133da_b_435a_` | `github-checks-plugin/` | Async enqueue IN_PROGRESS; ACL restore; completed waits for in-flight progress | Built for 2.555.1; needs HPI redeploy |
| `docker-workflow` `634.vedc7242b_eda_7` | `docker-workflow-plugin/` | `WithContainerStep` → `GeneralNonBlockingStepExecution` | Source patched; Maven package was aborted |
| `ssh-agent` `396.vcc7d84e622ec` | `ssh-agent-plugin/` | `SSHAgentStepExecution` → `GeneralNonBlockingStepExecution` | `mvn -DskipTests package` succeeded |

The remaining plugin-level fix of the same class as github-checks is **`checks-api`**: `extractOutput()` still walks the full flow graph on the CPS thread even after github-checks publish is async. That is not cloned here.

---

## 2. Environment

### 2.1 Demo job

`.ci/Jenkinsfile` launches a 100-way scripted `parallel()` of `stageWithAgent`:

```groovy
pullRequestAndTagPipeline {
    Map parallelAgents = [failFast: false]
    for (int i = 1; i <= 100; i++) {
        String agentNum = "${i}"
        parallelAgents["Agent ${agentNum}"] = { ->
            stageWithAgent(stage_name: "Agent ${agentNum}", command: "echo hello from agent ${agentNum}")
        }
    }
    parallel(parallelAgents)
    // ...
}
```

The original report was 20-way; the Jenkinsfile was later raised to 100 to make CPS occupancy obvious.

`.jervis.yml` relevant bits: `jdk: corretto21`, `jenkins.vault_secrets: true`, `os: amazonlinux2023`, cache on stage `Tests` / extract on `Release to Nexus`.

### 2.2 Call chain

```
Jenkinsfile parallel()
  → vars/parallel.groovy          wraps every branch in stage(task_name)
  → stageWithAgent
  → deployStage(..., parallel_agents: true, stage_with_agent: true)
  → buildAgent
  → adminJervisBuildNode
  → node(getAgentLabel(settings))
       checkout scm  or  unstash 'git-cache'
       docker.image(...).inside("--privileged ...")
         sshagent(agent_credentials)
           jervisRepoCache
             vaultSecrets
               body → runToolChainsSh
```

`parallel_agents: true` only skips `lock()` in `deployStage.doParallelAgent`. It does not make `node()`, Docker, or SSH agent start concurrent on the controller.

`vars/parallel.groovy` wraps **every** parallel branch in another `stage(task_name)`. That labeled FlowNode is what `checks-api` and the library GitHub status helper react to.

### 2.3 Plugin inventory

Plugin versions come from `dependencies.gradle` (copied into this repo for investigation; not used by the Groovy demo build). Controller is Jenkins 2.555.1 (`org.jenkins-ci.main:jenkins-war:2.555.1`).

Plugins that actually matter to this stall:

| Plugin | Version |
|---|---|
| `workflow-cps` | `4315.va_e456c4e7f4f` |
| `checks-api` | `402.vca_263b_f200e3` |
| `github-checks` | `679.v74133da_b_435a_` |
| `pipeline-graph-view` | `873.v8cb_25b_5e95f4` |
| `docker-workflow` | `634.vedc7242b_eda_7` |
| `ssh-agent` | `396.vcc7d84e622ec` |

Shared library lives at `/jenkins-pipeline-scripts` (readable; not writable from this workspace).

---

## 3. Symptom evolution

### Phase A — UI has parallels, no agents requested

- Pipeline graph showed all parallel branches immediately.
- EC2 plugin did not see queued work for 20–60 seconds.
- Capacity was not the issue: EC2 can only react after `node()` enqueues an `Executor`/`Queue.Task`.
- The dead spot was **after** the parallel structure was drawn and **before** `node()`.

### Phase B — skip-progress A/B

Enabling GitHub Checks **“Suppress progress updates in job check”** (`skipProgressUpdates`) **solved the original 20s stall**. That confirmed `ChecksGraphListener.onNewHead` as the first bottleneck.

A library change to skip GitHub commit statuses inside parallel (draft `stage.groovy` in this repo) was applied first. The 20s gap remained, because checks-plugin still ran at the `parallel()` fork even when library statuses were skipped.

### Phase C — 100 agents assigned, parallel still ~2 minutes

- All 100 agents were assigned at once.
- The parallel block still took ~2 minutes.
- Consecutive `sh 'echo'` steps on an agent were slow.
- Not Maven, not shell, not EC2 capacity, not `lock()` serializing work.

Thread dumps for this phase are `dump1.txt` … `dump6.txt` (Jenkins UI truncates stacks at ~8 frames).

---

## 4. Why one Java thread serializes 100 branches

Scripted/declarative Pipeline is **CPS** (`workflow-cps`). Each build has a single `Running CpsFlowExecution[...]` thread that interprets Groovy continuations.

- Durable steps (`sh`, `node` wait, `checkout` via SNB, `stash`/`unstash`) **yield**: `start()` returns `false`, the VM parks, other branches can run.
- `@NonCPS` methods, `GraphListener` callbacks invoked from `notifyListeners`, and `StepExecution.start()` that does I/O **do not yield**. The interpreter is stuck until that work returns.
- `parallel()` does not spawn 100 CPS VMs. It schedules 100 program threads onto **one** runner. If the runner is in `Proc.join` on agent 37’s `docker run`, agents 1–36 and 38–100 cannot even start their next Pipeline step.

That is the “node walking” blockage in the Jenkins Java runtime.

---

## 5. Root cause 1: GitHub Checks progress on every labeled head

### 5.1 Path

For every labeled FlowNode (`stage`, parallel branch name):

1. `checks-api` `BuildStatusChecksPublisher.ChecksGraphListener.onNewHead`
2. `FlowExecutionAnalyzer.extractOutput()` — full `FlowGraphTable` walk
3. `ChecksPublisherFactory` → `GitHubChecksPublisher.publish()`
4. GitHub HTTP (OkHttp) for an `IN_PROGRESS` check run

Synchronous `GraphListener`s (and even the non-`Synchronous` checks listener, which still runs on the notify path used here) execute **inline on the CPS runner**.

At a 100-way `parallel()`, that is 100 graph walks + up to 100 GitHub round trips **before** any branch reaches `node()`.

### 5.2 What skip-progress does

`skipProgressUpdates` skips `onNewHead` progress publishing. `QUEUED` / `COMPLETED` still post. That is why the UI still gets a job check, but the 20–60s gap disappears.

This was left as the **old default** (`skipProgressUpdates = false`). The user asked not to make suppress-progress the default.

### 5.3 Shared-library GitHub commit statuses (same class, not a plugin)

`/jenkins-pipeline-scripts/vars/stage.groovy` emulates GitHub autostatus:

- On every `stage(name) { body }`, `@NonCPS` `writeStatusToCommitOnPR` → `callGitHubAPI` (PENDING before the body, SUCCESS/FAILURE after).
- Those POSTs run on the CPS VM **inside `stage()` and before `node()`**.
- `vars/parallel.groovy` already created a stage per branch, so this fired 100 times at fork.

Draft skip-in-parallel lives at `/workspace/hello-world-groovy/stage.groovy` (library is not writable from here). Behavior:

- Skip GitHub commit statuses when already inside a `parallel()` branch, unless the stage name is a required check.
- Detect parallel via `ThreadNameAction` on the current FlowNode or enclosing blocks.

User applied that change. The 20s gap remained because **checks-api still ran at fork**. The library change is still useful once checks progress is off CPS or suppressed; it is not sufficient alone.

---

## 6. github-checks plugin patch

Clone: `/workspace/hello-world-groovy/github-checks-plugin`

### 6.1 Intended behavior

`GitHubChecksPublisher.publish()`:

- **IN_PROGRESS:** enqueue on `Computer.threadPoolForRemoting`, latest-wins coalesce per `(run, check name)`. Caller returns immediately.
- **QUEUED:** still synchronous (`publishNow`).
- **COMPLETED:** mark terminal, wait for / skip in-flight progress, then publish, so a stale in-progress cannot overwrite the conclusion.

Background work restores caller `ACL` (`Jenkins.getAuthentication2()` / `ACL.as2`). Without that, GitHub App credentials failed with `IllegalStateException` and **checks stopped sending**.

### 6.2 Tests added

- `inProgressUpdatesDoNotBlockTheCaller` — publish returns without calling GitHub; coalesced work runs on the provided executor.
- `completedCheckIsNotOverwrittenByStaleProgressUpdate` — completed waits for in-flight progress.

### 6.3 Jenkins 2.555.1 compatibility

- `jenkins.version` `2.555.1`
- BOM `bom-2.504.x` `5983.v443959746f1f`
- `github-branch-source` `1967.1969.v205fd594c821` (already on that controller)

Expected HPI: `github-checks-plugin/target/github-checks.hpi`. Installed production plugin was `679.v74133da_b_435a_`. User still needs to redeploy the patched HPI.

### 6.4 What this patch does **not** fix

`getOutput()` / `FlowExecutionAnalyzer.extractOutput()` still runs on CPS **in checks-api** if skip-progress is off. Async GitHub HTTP only starts after that walk. A full off-CPS `getOutput()` needs a **checks-api** change (not cloned).

---

## 7. Root cause 2: `docker.inside` and `sshagent` occupy CPS after `node()`

### 7.1 `adminJervisBuildNode` after `node()`

From `/jenkins-pipeline-scripts/vars/adminJervisBuildNode.groovy` (approx. 269–377):

1. `retry(3)` `checkout scm` **or** `stash name: 'git-cache'` / `unstash` (controller as stash hub). These already yield (workflow-scm-step / workflow-basic-steps SNB).
2. `docker.image(docker_image).inside("--privileged -v dind-cache:/var/lib/docker ...")`
3. `sshagent(agent_credentials)` around the body
4. `jervisRepoCache` + `vaultSecrets`
5. Body: `runToolChainsSh`

`Docker.groovy` `inside()` does durable `docker inspect` / `docker pull` **before** `withDockerContainer`; those yield. Comment in Docker.groovy: “withDockerContainer requires the image to be available locally, since its start phase is not a durable task.”

### 7.2 The plugin bugs

**`docker-workflow` `WithContainerStep.Execution.start()`** still does `dockerClient.run(...)` (and version / whoAmI / listProcess) **on CPS** via `Proc.joinWithTimeout`. Upstream source even has:

```java
// TODO switch to GeneralNonBlockingStepExecution
```

Teardown (`docker stop` in the body callback) is the same class of wait ([JENKINS-37719](https://issues.jenkins.io/browse/JENKINS-37719) / [JENKINS-37720](https://issues.jenkins.io/browse/JENKINS-37720)).

**`ssh-agent` `SSHAgentStepExecution.start()`** does `initRemoteAgent()` on CPS (spawn `ssh-agent`, `ssh-add`). Cheaper than privileged `docker run`, still serial.

Durable `sh` **inside** the container yields. Consecutive echos look slow because CPS is inside **another** branch’s `docker.inside` / `sshagent` start/stop.

This matches dumps 2 and 5: CPS **BLOCKED** / `TIMED_WAITING` on `Launcher$RemoteLauncher$ProcImpl.join` → `Proc.joinWithTimeout` on one EC2 agent, while 100 Channel reader threads exist (agents are connected). The interpreter cannot walk the other 99 program threads.

### 7.3 Could the library be rewritten instead?

Yes, but `docker.inside` / `sshagent` are the structural problem.

Library options:

- Skip them on this path, or
- Replace `docker.inside` with durable `docker run` / `docker exec` `sh` steps (those yield).

The correct plugin fix is `GeneralNonBlockingStepExecution`: `start()` returns immediately, I/O runs on `Computer.threadPoolForRemoting`, body starts when `docker run` / `ssh-agent` finish.

`workflow-step-api` already has `GeneralNonBlockingStepExecution`. `docker-commons` is not the Pipeline step.

---

## 8. Thread dump evidence

Dumps: `/workspace/hello-world-groovy/dump1.txt` … `dump6.txt`. All taken while Jenkins was slow on the 100-node Jenkinsfile except dump6 (after parallel completed). `ThreadInfo.toString()` (used by this script and by Manage Jenkins → Thread Dump) truncates each stack at ~8 frames.

### 8.1 How the dumps were taken

Capture the **controller JVM**, not agent JVMs. These dumps were taken **10–30 seconds apart** while the stall was happening (parallels visible, consecutive `sh 'echo'` slow). `dump6.txt` was taken after the parallel block had finished, as a control.

Prefer **Manage Jenkins → Script Console**:

```groovy
import java.lang.management.ManagementFactory
println ManagementFactory.getThreadMXBean().dumpAllThreads(true, true)
    .collect { it.toString() }.join('\n')
```

Paste the console output into `dumpN.txt`. `dumpAllThreads(true, true)` includes locked monitors and synchronizers, which is how dump2/dump3/dump4 show `Number of locked synchronizers`.

Alternatives if Script Console is unavailable: `jstack <controller-pid>` on the controller host, or `kill -3` on the controller process (dump goes to Jenkins stdout). Manage Jenkins → Thread Dump is the same truncated `ThreadInfo` format; it is harder to save as a single file.

### 8.2 What the dumps showed

| Dump | CPS thread | Interpretation |
|---|---|---|
| dump1 | RUNNABLE `GroovyRecognizer` | CPS parsing Groovy (likely a large shared-library script such as `deployStage.groovy`) |
| dump2 | TIMED_WAITING `ProcImpl.join` on one EC2 agent | Waiting for a remote process (`docker run` / `ssh-agent` / similar) on CPS |
| dump3 | RUNNABLE `Jenkins.getDescriptorOrDie` → `MultiBranchProject.getRootDirFor` | Folder/job filesystem lookup on CPS |
| dump4 | RUNNABLE `LoggingInvoker.methodCall` | CPS executing Groovy (stack truncated) |
| dump5 | TIMED_WAITING `ProcImpl.join` → `Proc.joinWithTimeout` | Same as dump2; the smoking gun for docker/sshagent |
| dump6 | **No** `Running CpsFlowExecution[...]` | Parallel over; CPS runner gone |

Other observations:

- Exactly **one** `Running CpsFlowExecution[cicd-examples/hello-world-groovy/PR-338#29]` while the parallel ran.
- dump2: 100 Channel reader threads — 100 agents connected. Allocation is not the stall.
- GitHub OkHttp appeared on a **side** thread in later dumps, not on CPS (consistent with skip-progress or async publish).
- DurableTask watchers were mostly idle — `sh` was not the bottleneck.

---

## 9. Other CPS occupancy on the `stageWithAgent` path

These are **not** `Proc.join` plugin bugs. They still occupy CPS via `@NonCPS` I/O.

| Source | When | Effect on 100-way parallel |
|---|---|---|
| `writeStatusToCommitOnPR` / `callGitHubAPI` | Every `stage()` unless skipped | GitHub HTTP on CPS; library draft mitigates |
| `getKnownJiraIssues()` at start of every `deploy()` | GitHub GraphQL + Jira + artifact-manager-s3 | First branch fills a binding; other 99 can all miss it and repeat |
| `vaultSecrets` (`jenkins.vault_secrets: true`) | Every `adminJervisBuildNode` body | `@NonCPS` Vault HTTP via `VaultService`, 100 times |
| `assumeIamRole` | AWS STS `assumeRole` on CPS, then `sh 'env'` | STS on CPS |
| Groovy parse of huge `deployStage.groovy` | First use / dump1 | One-time parse cost on CPS |
| `checks-api` `ChecksGraphListener` | Every labeled head if skip-progress off | Graph walk + publish; see §5 |

Already yielding (not the stall):

- `checkout scm` (workflow-scm-step + git, SNB)
- `stash` / `unstash` (workflow-basic-steps SNB)
- `withCredentials` (credentials-binding already `GeneralNonBlocking`)
- `node()` wait for an executor
- Durable `sh`
- `lock()` — skipped when `parallel_agents: true`

`ensureStage` typically does **not** add a second stage when already inside `stage` from `vars/parallel.groovy`. `jervisRepoCache` extract/create is mostly a no-op for `"Agent N"` stages (`.jervis.yml` `cache_on: Tests`, `extract_on: Release to Nexus`).

---

## 10. Plugin patches (docker-workflow and ssh-agent)

User asked to clone both into this repo (nested clones are intentional) at the **installed** versions, then “Patch both.” Working branches: `off-cps-start`.

### 10.1 docker-workflow `WithContainerStep`

- Converted from `AbstractStepImpl` to `Step`.
- `Execution` extends `GeneralNonBlockingStepExecution`.
- `start()` calls `run(this::doStart)` and returns `false`.
- Docker CLI (`run`, version, inspect, listProcess) and container destroy run off CPS.
- `stop()` calls `super.stop()` then `destroyContainerAsync` on `Computer.threadPoolForRemoting` with ACL restored (`run()` is a no-op after `stopCause`).
- Body callback is a custom `BodyExecutionCallback` (not TailCall) that `run()`s destroy before `onSuccess`/`onFailure`.
- Test hook: `WithContainerStep.Execution.beforeContainerRun`.
- Test `containerStartDoesNotBlockCpsVm` parks before `docker run` and asserts a parallel sibling can log `"other branch ran"`.

POM left at `jenkins.baseline` 2.479 / `jenkins.version` 2.479.3 (loads on 2.555.1). Core requirement was **not** raised.

Maven `package` was **aborted** before producing an HPI. Source patches remain.

### 10.2 ssh-agent `SSHAgentStepExecution`

- Converted from `AbstractStepExecutionImpl` to `GeneralNonBlockingStepExecution`.
- Same `run(this::doStart)` / async stop pattern.
- Hook `SSHAgentStepExecution.beforeAgentStart` before `new ExecRemoteAgent`.
- Test `sshAgentStartDoesNotBlockCpsVm` uses `sshagent(credentials: [], ignoreMissing: true)`.

`mvn -DskipTests package` **succeeded**. HPI: `ssh-agent-plugin/target/ssh-agent.hpi`. Manifest: `Jenkins-Version: 2.479.1`, `Plugin-Version: 999999-SNAPSHOT (private-cc7d84e6-jenkins)`.

### 10.3 Tests not run after abort

- docker-workflow CPS test needs Docker (`DockerTestUtil.assumeDocker()`).
- ssh-agent CPS test needs `ssh-agent` on the test host.

A parallel Maven + Jervis login-shell race on `/opt/apache-maven` was observed; later builds were sequential. Login shell is noisy and sometimes fails DNS to `nexus.303net.net` during toolchain setup, then recovers.

---

## 11. Plugins similar to GitHub status checks (async enqueue)

The pattern that benefits from async enqueue is: a **GraphListener** (especially `GraphListener.Synchronous`) that does I/O or a full graph walk on every labeled FlowNode.

| Plugin | Behavior | Async enqueue? |
|---|---|---|
| **checks-api** `402.vca_263b_f200e3` | `ChecksGraphListener.onNewHead` walks the full graph then publishes | **Yes — this is the remaining one.** Move `getOutput()` + `publish()` off CPS, latest-wins. github-checks async does not move `getOutput()` off CPS. Skip-progress is the current workaround. |
| **pipeline-graph-view** `LiveGraphPopulator` | `GraphListener.Synchronous` by design; in-memory UI under a monitor; never scans | **No.** Async would lag the UI. |
| **blueocean-events** `PipelineEventListener` | Event on every stage/step to SSE/pubsub | Only if `publishEvent` shows up on CPS dumps. Local SSE, not GitHub HTTP. |
| **github** / **github-branch-source** classic statuses | `RunListener` PENDING/SUCCESS per **build**, not per stage | Small win vs 100-way parallel |
| **pipeline-githubnotify-step** | A step you call, not a listener | Library uses `callGitHubAPI` instead |
| **junit** / **coverage** / **sonar** / **slack** / **email-ext** | Publish at step/build end | Would not speed parallel `node()` allocation |
| **prometheus** / **metrics** | Cheap counters | No |
| **workflow-job** graph persistence | BulkChange save | Durability, not GitHub-like HTTP |

Library GitHub statuses (`writeStatusToCommitOnPR`) are the other per-stage GitHub path. Same idea (outbound HTTP on the interpreter), but it is shared-library code, not a plugin to clone.

---

## 12. What was ruled out

- **EC2 plugin / capacity.** Can only react to queued `node()` work. Phase A is before that. Phase C had 100 agents already assigned.
- **Shell / Maven / image work.** Consecutive `sh 'echo'` is slow because CPS is busy on another branch, not because echo is slow.
- **`lock()`.** Skipped for `stageWithAgent` (`parallel_agents: true`).
- **pipeline-graph-view** as the GitHub-like HTTP stall. It is intentionally sync and in-memory.
- **GitHub OkHttp on CPS** in later dumps (side thread).
- **Need to clone every plugin on the path.** Only docker-workflow and ssh-agent had `Proc.join`-on-CPS in `start()`. `workflow-step-api` already has the helper class.

Suggested control experiment (not required to conclude, still useful): a `parallel` of `node(label) { sh 'echo a'; sh 'echo b' }` **without** `stageWithAgent`. If that is fast, the wrapper (`docker.inside` + `sshagent` + library HTTP) is confirmed.

---

## 13. Remaining work

1. Redeploy patched HPIs on the 2.555.1 controller: github-checks (async IN_PROGRESS + ACL), ssh-agent, and docker-workflow once packaged.
2. Finish `docker-workflow` `mvn -DskipTests package`; run `containerStartDoesNotBlockCpsVm` and `sshAgentStartDoesNotBlockCpsVm`.
3. Optionally clone and patch **checks-api** so `getOutput()` is off CPS; otherwise keep skip-progress enabled for wide parallels.
4. Shared library (not done here; library is not writable):
   - Land `stage.groovy` skip-in-parallel (draft in this repo).
   - Consider moving `writeStatusToCommitOnPR`, Vault, Jira, and STS off CPS (durable `httpRequest` / `sh`, or a non-blocking step).
   - Optional: skip `docker.inside` / `sshagent` on trivial `stageWithAgent` smoke paths.
5. Do not commit unless asked. Nested plugin clones inside this demo repo are investigation artifacts.

---

## 14. Artifacts in this workspace

| Path | Role |
|---|---|
| `.ci/Jenkinsfile` | 100-way `stageWithAgent` reproducer |
| `dependencies.gradle` | Installed plugin list (not used by the Groovy project) |
| `stage.groovy` | Draft: skip GitHub commit statuses inside parallel |
| `github-checks-plugin/` | Async IN_PROGRESS publisher + 2.555.1 POM |
| `docker-workflow-plugin/` | Off-CPS `withDockerContainer` |
| `ssh-agent-plugin/` | Off-CPS `sshagent` |
| `dump1.txt` … `dump6.txt` | Controller thread dumps from PR-338#29 (Script Console `dumpAllThreads`; see §8.1) |

---

## 15. Timeline of the conversation (compressed)

1. Report: 20 parallel branches visible, 30s–1m before `node()` queues agents. Plugin unknown.
2. EC2 ruled out: gap is before the queue.
3. Hypothesis: CPS occupied between graph draw and `node()` (checks + library GitHub statuses).
4. Draft `stage.groovy` to skip GitHub statuses in parallel; user applied it; 20s gap remained.
5. A/B: **Suppress progress updates** fixed the original stall.
6. Cloned `github-checks-plugin`; async IN_PROGRESS; user asked not to default skip-progress.
7. Checks stopped sending → restore ACL on background thread; COMPLETED vs stale IN_PROGRESS race.
8. Lower plugin deps for Jenkins 2.555.1.
9. New symptom: 100 nodes assigned, parallel still ~2 minutes; `sh 'echo'` slow; “node walking” in the JVM.
10. Thread dumps: one CPS runner; `Proc.join` on CPS; 100 agents connected.
11. Traced to `adminJervisBuildNode` → `docker.inside` / `sshagent`.
12. Confirmed plugin bugs (`start()` must not block). Cloned and patched both.
13. Other path occupancy: library `@NonCPS` HTTP + checks-api graph walk.
14. Only remaining GitHub-checks-like plugin for async enqueue: **checks-api**.
)
</tool_call>

</think>

The report is at `ai_analysis_report.md`. It covers the full investigation in one place:

- **Two bottlenecks:** GitHub Checks progress on CPS before `node()`, then `docker.inside` / `sshagent` occupying the single CPS runner after 100 agents are assigned
- **Evidence:** PR-338#29 thread dumps, the skip-progress A/B, and the `stage.groovy` experiment that did not close the first gap
- **Patches:** github-checks async IN_PROGRESS (ACL + completed-vs-stale race), docker-workflow and ssh-agent `GeneralNonBlockingStepExecution`
- **What is still open:** checks-api `getOutput()` still on CPS, docker-workflow HPI not packaged, HPI redeploy, library `@NonCPS` HTTP (Vault / Jira / GitHub statuses)