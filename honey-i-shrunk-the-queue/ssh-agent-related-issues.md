# `jenkinsci/ssh-agent-plugin` issue triage for the off-CPS patch

Scope: which open issues and pull requests in `jenkinsci/ssh-agent-plugin`
are addressed, improved, or affected by converting `SSHAgentStepExecution`
from `StepExecution` to `GeneralNonBlockingStepExecution`.

Local patch: branch `off-cps-start-master`, on top of upstream `122f120`
("Extend PATH with `<git-home>\usr\bin` if git's ssh-agent is used", #316).

Surveyed 2026-09-13 against the GitHub REST API: 17 open items (16 issues,
1 pull request), plus keyword searches across closed items for
`parallel`, `slow`/`performance`, `CPS`/`blocking`, and `timeout`.

## What the patch actually changes

`start()` now returns immediately and `initRemoteAgent()` runs on a
background thread, so launching `ssh-agent` and running `ssh-add` no longer
occupy the CPS VM thread. Teardown moves off the CPS VM thread too, via
`GeneralNonBlockingStepExecution.TailCall` for the normal path and an
explicit `Computer.threadPoolForRemoting` submission with the caller's
authentication restored for the `stop()` path.

This is a concurrency change. It does not alter how the agent is discovered,
how credentials are resolved, how host keys are verified, or how the
executable is located. That distinction drives the triage below.

## Verified locally

| Check | Result |
| --- | --- |
| `mvn -B test` (full suite) | 63 run, 0 failures, 2 skipped |
| `agentStartDoesNotBlockCpsVm` (new) | PASS in 2.2 s |
| same test on unpatched `122f120` | FAIL — sibling branch blocked until t = 63.7 s |
| `sshAgentAvailableAfterRestart` | PASS (restart during the block still works) |

## Honest headline

**This patch does not outright close any open issue in the tracker.** The
open backlog is dominated by agent-provider discovery, Windows, credential,
and host-key problems, none of which are threading bugs. Its one strong
candidate is #253, and its real value is preventing a class of timeout that
the project has so far worked around by making the timeout configurable.

## Plausibly improved

### #253 — [JENKINS-65857] `docker exec containerID ssh-agent -k` times out after 1 minute

> If several pipelines pull gitlab code at the same time on the same machine,
> `docker exec ssh-agent -k` will fail.

The strongest match in the tracker. The reporter's trigger is explicitly
concurrency on a single machine, and the failure is in teardown. Before the
patch every `sshagent` teardown across every parallel branch is serialized
onto the one CPS VM thread, so the one-minute launcher timeout is measuring
queueing delay as much as it is measuring `ssh-agent -k`. The patch runs
teardown on a background thread, which removes that queueing.

Worth linking as "likely addressed", not "fixes". The reporter is on ssh-agent
1.20 / Jenkins 2.277.3 from 2021 and cannot realistically confirm, and a
`docker exec` that is genuinely slow would still time out.

## Adjacent, already closed — useful context for the PR description

These were resolved by making the timeout configurable rather than by
removing the contention that causes it. They are the best evidence that the
one-minute timeout is a recurring, real-world pain point.

| Item | Title | Outcome |
| --- | --- | --- |
| #277 | [JENKINS-74823] ERROR: Timeout after 1 minutes | closed; timeout made configurable |
| #268 | [JENKINS-71537] ssh-add is timing out in a Fargate worker | closed |
| #286 | Make the agent command timeout configurable via the `sshagent` step | merged |
| #294 / #300 / #299 | follow-ups to the configurable timeout | merged / closed |

Framing the patch as attacking a contributing cause of #277 and #268, rather
than claiming to fix them, is both accurate and the more persuasive argument.

## Interacts with the patch — call these out

### #197 — [JENKINS-29810] `sshagent` step to survive Jenkins restarts

`GeneralNonBlockingStepExecution.onResume()` fails the step with
`SynchronousResumeNotSupportedException` when a restart lands while
background work is in flight. The patch therefore introduces a narrow new
window: a controller restart *during agent launch* now fails the build.

This is narrow and the existing `sshAgentAvailableAfterRestart` test still
passes, confirming a restart while the block body runs is unaffected. It does
mean the patch moves #197 slightly further away, not closer, and the PR should
say so rather than leave a reviewer to find it.

### #242 — [JENKINS-55256] `ssh-agent` leaves defunct processes on swarm client

Teardown now happens on a different thread, and on the `stop()` path it is
submitted to `Computer.threadPoolForRemoting`. That changes when and from
where `ssh-agent -k` is reaped. The root cause here looks like process
reaping on the swarm client rather than step threading, so do not claim a fix,
but this is the open issue most likely to see changed behaviour.

Also note the teardown path can now run twice on an abort: `stop()` submits
its own teardown while the body callback may already have submitted one, since
`run()` only short-circuits once `stopCause` is set. `ExecRemoteAgent.stop`
tolerates this today, and `teardownDoesNotFailBuildWhenAgentAlreadyStopped`
covers it, but a single-shot guard would be tidier.

### #228 — [JENKINS-43050] doesn't work well with docker pipelines

Two distinct complaints. The first — `ssh-agent -k` appearing to run before
the body's `sh` — is an ordering bug from 2017 that the current callback
structure no longer exhibits. The second — the agent socket living in `/tmp`
on the host and not being visible inside the container — is a volume-mounting
concern this patch does not touch. jglick already triaged both in the Jira
comments. Probably closeable on its own merits, but not *by* this patch.

## Not addressed

All of these are functional bugs or feature requests with no threading
component. Listing them explicitly so the PR does not over-claim.

| Item | Title |
| --- | --- |
| #273 | [JENKINS-73018] Getting "script returned exit code 128" (host keys) |
| #272 | [JENKINS-72254] Add support for SSH Certificates |
| #265 | [JENKINS-69264] Use SSH host key verification strategies from git-client-plugin |
| #263 | [JENKINS-68647] Could not `ssh-add` a private key created as a credential |
| #262 | [JENKINS-68325] FATAL: Could not find a suitable ssh-agent provider |
| #255 | [JENKINS-66285] Can't get file content through subshell `$(command)` |
| #254 | [JENKINS-66240] Cannot find ssh-agent provider after upgrade (k8s `container` block) |
| #240 | [JENKINS-53387] `ssh-add` — Text file busy |
| #227 | [JENKINS-42346] Environment not considered by exec launcher — see PR #317 |
| #218 | [JENKINS-38470] User-specific keys can't be found by the `sshagent` step |
| #192 | [JENKINS-29168] SSH credential not used by Windows slave |
| #187 | [JENKINS-27657] Perforce SCM workspace checkout can't access the ssh-agent |

`#262`, `#254`, `#259`, `#261`, `#266`, `#267` form a long-running
"could not find a suitable ssh-agent provider" cluster. It is the single
largest theme in the tracker and is entirely unrelated to this work.

## Prior art and conflict risk

No previous pull request has attempted the `GeneralNonBlockingStepExecution`
migration, so there is no duplicate to close or rebase onto.

**PR #317 (open) — "honor a custom PATH and pass step/build env to ssh-agent,
ssh-add"** is the only open PR and it does touch
`SSHAgentStepExecution.java`. Conflict risk is low: its change to that file is
a single line inside `initRemoteAgent()`, adding
`getContext().get(EnvVars.class)` to the `ExecRemoteAgent` constructor call,
and this patch leaves the body of `initRemoteAgent()` untouched. The two
should merge in either order.

One semantic interaction is worth verifying if #317 lands first: under this
patch `initRemoteAgent()` runs on a background thread, so #317's
`getContext().get(EnvVars.class)` would be called off the CPS VM thread. That
is exactly what `StepContext.get` is designed for, so it should be fine, but
it is worth a test run rather than an assumption.

## Suggested PR wording

Link #253 as "likely addressed, needs confirmation". Cite #277 and #268 as
closed evidence that the one-minute timeout keeps biting users and that the
project's current answer is a configurable timeout rather than removing the
contention. Explicitly disclaim the provider-discovery cluster. Mention the
#197 resume regression up front.
