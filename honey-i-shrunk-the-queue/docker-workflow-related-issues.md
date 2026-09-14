# docker-workflow-plugin: issues and PRs affected by the off-CPS `WithContainerStep` change

**Repository:** [jenkinsci/docker-workflow-plugin](https://github.com/jenkinsci/docker-workflow-plugin)
**Survey date:** 13 September 2026
**Scope surveyed:** all 256 open items (211 issues, 45 pull requests) plus targeted searches of closed items
**Local patch:** `docker-workflow-plugin/` in this workspace, branch `off-cps-start`, based on `634.vedc7242b_eda_7`

---

## 1. What the change actually does

`WithContainerStep.Execution` is converted from `AbstractStepExecutionImpl` to `GeneralNonBlockingStepExecution`. This is the change upstream already has a placeholder for:

```114:115:docker-workflow-plugin/src/main/java/org/jenkinsci/plugins/docker/workflow/WithContainerStep.java
    // TODO switch to GeneralNonBlockingStepExecution
    public static class Execution extends AbstractStepExecutionImpl {
```

Concretely, after the patch:

- `start()` calls `run(this::doStart)` and returns `false`. The Docker CLI work (`docker version`, `docker run`, `docker top` / `listProcess`, image inspect, `whoAmI`) happens on a background thread instead of the CPS VM thread.
- Container teardown (`docker stop` / `docker rm`) runs off CPS too, from a body callback rather than `TailCall`.
- `stop(Throwable)` calls `super.stop()` and then destroys the container asynchronously with the caller's `ACL` restored, because `run()` is a no-op once `stopCause` is set.

Three user-visible consequences follow, and they are the lens for the whole triage below:

1. **Throughput.** Other `parallel` branches keep executing while one branch starts or stops a container. Previously one `docker run` serialized the entire build's interpreter.
2. **Interruptibility.** A build can be aborted, and `timeout` can fire, while container start/stop is in flight. Previously the CPS thread was parked in `Proc.joinWithTimeout` and could not process the interrupt.
3. **Diagnosability.** The **Thread Dump** link shows the step busy on a named background thread rather than showing `WorkflowScript` stuck on the closure's last line.

**What the change does not do** — important, because several open issues look related but are not:

- It does not change the hardcoded `docker stop --time=1`. That is [PR #743](https://github.com/jenkinsci/docker-workflow-plugin/pull/743).
- It does not fix `docker top` failures, ENTRYPOINT problems, or "container started but didn't run the expected command".
- It does not fix fingerprint write amplification, only the fact that the wait happens on the interpreter.
- It does not fix `sh` / `bat` hangs *inside* a running container; those are durable-task issues.
- It does not improve pipeline resume across a controller restart.

---

## 2. Precedent: this is finishing a job started in 2019

[PR #158 "Using GeneralNonBlockingStepExecution"](https://github.com/jenkinsci/docker-workflow-plugin/pull/158) was **merged on 2019-04-09**. It applied exactly this treatment to `withDockerRegistry` and `withDockerServer` (the classes now in `AbstractEndpointStepExecution2`). Its description is the same argument being made here:

> Previously these operations were run inside the CPS VM thread, preventing other `parallel` branches from making progress for example. Now the program Groovy logic should continue while these operations are in progress, the **Thread Dump** link should show the step as busy in a specific background thread, and interrupting the build should attempt to cleanly abort the task.

`withDockerContainer` was left out of that PR, and the `TODO` above has been in the file ever since. Two older closed issues are the historical record of the same pain:

| Item | State | Relevance |
|---|---|---|
| [#433 JENKINS-37719](https://github.com/jenkinsci/docker-workflow-plugin/issues/433) — Build cannot be interrupted if `docker stop` hangs | closed 2017 | Exact symptom (2). Closed via [PR #86](https://github.com/jenkinsci/docker-workflow-plugin/pull/86), which added CLI *timeouts* rather than moving work off CPS |
| [#480 JENKINS-42322](https://github.com/jenkinsci/docker-workflow-plugin/issues/480) — Docker rm/stop killed by the timeout, failing builds | closed 2017 | The direct fallout of that timeout-based fix: `ERROR: Timeout after 10 seconds` now failed builds |

That pair matters for framing the PR. The 2017 fix bounded the blocking call; it did not stop it from blocking the interpreter, and it converted hangs into build failures. The non-blocking conversion is the fix the timeout was standing in for.

---

## 3. Tier A — strong candidates to close with this change

These describe the blocking/interruptibility mechanism directly. Recommend linking the PR and asking the reporter to retest on a build with the patch.

### [#559 — JENKINS-49710 Pipelines run under heavy load sometimes hang running Docker](https://github.com/jenkinsci/docker-workflow-plugin/issues/559)

The single best match. About 50 concurrent tests, roughly 1% hang forever and **"must be manually killed"**. Both halves are consequences of CPS occupancy: contention appears only under concurrency, and the inability to abort is precisely symptom (2). Components listed are docker-workflow, durable-task and pipeline, which is consistent with a build wedged between the container lifecycle and the interpreter.

### [#719 — JENKINS-73567 Failed to kill container](https://github.com/jenkinsci/docker-workflow-plugin/issues/719)

The reproducer is a `parallel` map of `docker.image(...).inside { }` branches. The reporter is explicit about the scaling curve: 0–5 parallel jobs succeed 100% of the time, 35 parallel jobs produce 1–2 failures 100% of the time, **after** the workload itself succeeded. Failure is in teardown, and the container exits 137, meaning Docker had to `kill -9` after the stop deadline.

Caveat worth stating in the close comment: the hardcoded `--time=1` is a contributing factor, so this issue is best resolved by this change *together with* [PR #743](https://github.com/jenkinsci/docker-workflow-plugin/pull/743). With teardown off the interpreter, the stop no longer has to compete with 34 other branches for the CPS thread before it is even issued.

### [#714 — JENKINS-72729 InterruptedException when executing within docker on remote worker](https://github.com/jenkinsci/docker-workflow-plugin/issues/714)

Same scaling signature: a small set of concurrently generated pipelines passes reliably, the full suite mostly fails. The reporter already tried raising `DurableTaskStep.REMOTE_TIMEOUT`, which fixed a different problem and not this one — consistent with the delay being controller-side interpreter contention rather than agent-side channel latency. This is the closest public analogue to the 100-way parallel stall documented in `ai_analysis_report.md`.

### [PR #145 — [WIP] Docker.stop continually hangs (and times out builds)](https://github.com/jenkinsci/docker-workflow-plugin/pull/145)

Open WIP from 2018, empty description, untouched since. It targets exactly the teardown hang that the async `stop()` path addresses. Recommend closing as superseded once the non-blocking PR lands.

---

## 4. Tier B — real improvement, but the root cause is elsewhere

Do not claim these as fixed. The change removes the interpreter-wide blast radius; the underlying Docker-level defect stays. These are good candidates for a comment rather than a close.

### [#641 — JENKINS-60898 Contention on fingerprint files when using Docker steps](https://github.com/jenkinsci/docker-workflow-plugin/issues/641)

The most precise report in the whole tracker. The reporter links the exact line inside `WithContainerStep.java` where every job "gets to ... and hangs", and the attached thread dump shows threads blocked in `hudson.model.Fingerprint.save`. With the patch, that wait happens on a background thread, so short Docker jobs running every minute no longer stall each other's interpreters. The write amplification into the `fingerprints` directory is untouched and needs a separate fix (see also [#638 JENKINS-60570](https://github.com/jenkinsci/docker-workflow-plugin/issues/638), fingerprint data never cleaned up).

### [#678 — JENKINS-67089 Fails to kill the docker container after build completion](https://github.com/jenkinsci/docker-workflow-plugin/issues/678)

The stack trace is the teardown path this patch rewrites:

```
DockerClient.stop → WithContainerStep.destroy → WithContainerStep$Callback.finished
  → BodyExecutionCallback$TailCall.onSuccess → CpsBodyExecution$SuccessAdapter.receive
```

`TailCall.onSuccess` on the CPS thread is exactly what the patch replaces with a callback that runs destroy off CPS. The reported failure is random and correlated with load, so moving it off the interpreter is likely to help, but `Failed to kill container` can still occur.

### [#718 — JENKINS-73561 Docker pipeline hangs when container has problems starting](https://github.com/jenkinsci/docker-workflow-plugin/issues/718)

The first build works, subsequent builds hang at `sh` after `docker top` reports "the container started but didn't run the expected command". The *hang* becomes abortable and stops holding the interpreter; the ENTRYPOINT/`cat` root cause does not change. Partial.

### [#375 — JENKINS-28606 Track down WithContainerStepTest.stop() hang](https://github.com/jenkinsci/docker-workflow-plugin/issues/375)

Filed by jglick. Two asks: the `ps`/`COOKIE` lookup fails to find container processes (not addressed), and **"we should do a hard kill on the container after 10+ seconds"** (the async stop path makes a bounded, cancellable teardown feasible). Worth a comment, not a close.

### [#511 — JENKINS-46062 Timeout timer inside a pipeline with a docker stage runs out too soon](https://github.com/jenkinsci/docker-workflow-plugin/issues/511)

Low confidence, but timer accuracy inside a Docker stage is plausibly affected by the interpreter being unable to service the timeout while parked in a Docker CLI join. Worth re-testing rather than claiming.

---

## 5. Tier C — adjacent work to coordinate with, not close

| Item | Relationship |
|---|---|
| [PR #743 — Make docker stop timeout configurable via `STOP_TIMEOUT`](https://github.com/jenkinsci/docker-workflow-plugin/pull/743) | Complementary. Pairs with the async teardown for #719. Both touch the stop path; expect a merge conflict |
| [#674 JENKINS-66595 — Allow configuration of stop timeout](https://github.com/jenkinsci/docker-workflow-plugin/issues/674) | The issue behind PR #743 |
| [#607 JENKINS-57136 — Allow users to customize docker timeouts](https://github.com/jenkinsci/docker-workflow-plugin/issues/607) | Same family. Non-blocking execution reduces the need to tune these, but does not remove it |
| [#733 JENKINS-76036 — `--time` deprecated, use `--timeout`](https://github.com/jenkinsci/docker-workflow-plugin/issues/733) | Same code path in `DockerClient.stop` |
| [#697 JENKINS-69852 — Use docker-java instead of docker cli](https://github.com/jenkinsci/docker-workflow-plugin/issues/697) | Competing direction. A Java API client would remove `Proc.join` entirely. Worth noting in the PR that the non-blocking conversion does not block that rewrite |
| [#461 JENKINS-40170 — Separate execution functionality of `image.inside()` into distinct steps](https://github.com/jenkinsci/docker-workflow-plugin/issues/461) | Feature request. Unrelated mechanism, but the refactor touches the same class |
| [#651 JENKINS-63151 — Execute steps before starting docker container](https://github.com/jenkinsci/docker-workflow-plugin/issues/651) | Feature request in `start()`. Only relevant as a merge-conflict risk |
| [PR #370 — Fix `WithContainerStep` when `hasWorkdir` is false for Windows Containers](https://github.com/jenkinsci/docker-workflow-plugin/pull/370) | Touches the same file's `Decorator`. Conflict risk |
| [PR #366 — Migrate tests to JUnit5](https://github.com/jenkinsci/docker-workflow-plugin/pull/366) | Conflicts with the new `containerStartDoesNotBlockCpsVm` test, which is written for JUnit 4 |

---

## 6. Tier D — looks related, is not fixed

Listing these explicitly so the PR does not over-claim. Each is a hang or failure in or around `docker.inside`, but none is caused by CPS occupancy.

| Item | Actual cause |
|---|---|
| [#671 JENKINS-65749](https://github.com/jenkinsci/docker-workflow-plugin/issues/671), [#629 JENKINS-59893](https://github.com/jenkinsci/docker-workflow-plugin/issues/629) | `bat` hangs in Windows containers. Durable-task / Windows process handling |
| [#653 JENKINS-63253](https://github.com/jenkinsci/docker-workflow-plugin/issues/653) | `dir()` causes "process apparently never started". Durable-task working directory |
| [#734](https://github.com/jenkinsci/docker-workflow-plugin/issues/734), [#722 JENKINS-73979](https://github.com/jenkinsci/docker-workflow-plugin/issues/722) | `docker top` error / ENTRYPOINT race. Same error, different thread after the patch |
| [#681 JENKINS-67443](https://github.com/jenkinsci/docker-workflow-plugin/issues/681) | `stop`/`rm` on an already-removed container. Needs an existence check |
| [#552 JENKINS-49365](https://github.com/jenkinsci/docker-workflow-plugin/issues/552) | Resume after controller restart. If anything this needs extra care under `GeneralNonBlockingStepExecution`, since a restart during `doStart` fails the step |
| [#430 JENKINS-37069](https://github.com/jenkinsci/docker-workflow-plugin/issues/430) | Workspace permissions |
| [#484 JENKINS-42863](https://github.com/jenkinsci/docker-workflow-plugin/issues/484) | Step result reporting |
| [#393 JENKINS-32859](https://github.com/jenkinsci/docker-workflow-plugin/issues/393), [PR #312](https://github.com/jenkinsci/docker-workflow-plugin/pull/312) | PID 1 zombie reaping, `--init` |
| [#641's sibling #638 JENKINS-60570](https://github.com/jenkinsci/docker-workflow-plugin/issues/638) | Fingerprint files never cleaned up |

---

## 7. Suggested PR framing

A PR that lands this will be easier to review if it makes the continuity argument rather than presenting a new idea:

> Completes the work started in #158 by applying `GeneralNonBlockingStepExecution` to `withDockerContainer`, resolving the `// TODO switch to GeneralNonBlockingStepExecution` left on `WithContainerStep.Execution`. Container start (`docker version`, `docker run`, `docker top`) and teardown (`docker stop`/`rm`) no longer occupy the CPS VM thread, so sibling `parallel` branches continue to execute and the build remains interruptible while the Docker CLI is in flight. Relates to #559, #714, #719, #641, #678. Supersedes #145.

Include the regression test already written locally, `WithContainerStepTest.containerStartDoesNotBlockCpsVm`, which parks a hook immediately before `docker run` and asserts that a sibling `parallel` branch can still log. That test is the artifact reviewers will want, because it encodes the property rather than the symptom.

Two review risks to disclose up front: the step class changes from `AbstractStepImpl` to `Step` (descriptor and serial-form implications), and the body callback is a custom `BodyExecutionCallback` rather than `TailCall`, because destroy must run before `onSuccess` / `onFailure`.

---

## 8. Method and caveats

Data was collected from the public, unauthenticated GitHub REST API on 13 September 2026:

```bash
curl -sS "https://api.github.com/repos/jenkinsci/docker-workflow-plugin/issues?state=open&per_page=100&page=N"
curl -sS "https://api.github.com/search/issues?q=repo:jenkinsci/docker-workflow-plugin+GeneralNonBlockingStepExecution"
```

All 256 open items were indexed by number, type, labels and title, then their bodies were searched locally for CPS, blocking, deadlock, hang, parallel, heavy-load and scalability language. Candidates were read in full. Closed items were reached through three targeted searches rather than a full crawl.

Caveats worth keeping in mind:

- Most issues in this tracker are Jira imports from 2015–2025 and many reporters are long gone, so "close" in Tier A realistically means "close as likely fixed, invite retest" rather than "confirmed fixed".
- Issue comment threads were only read for the four prior-art items. A maintainer triaging for real should read the comments on #559, #714 and #719, which have 2, 1 and 12 comments respectively.
- Only open items were enumerated exhaustively. There may be further closed duplicates of the CPS-blocking symptom that these three searches did not surface.
