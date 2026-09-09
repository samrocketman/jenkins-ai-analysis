# Hot spare scaling: from a queue multiplier to a load prediction

## What was wrong

`MinimumInstanceChecker.checkForLabelHotSpares` computed the number of spares as

```
desired = max(baseHotSpares, scalingFactor * queuedBuilds)   // capped by maxHotSpares
toLaunch = desired - (idleSpares + provisioning)
```

The target was a function of the queue, and the queue empties the moment the builds start. With a
step of 5 and a base of 0, one burst provisioned five agents; as those agents picked up work,
`queuedBuilds` fell to zero, the target fell to the base count, and nothing was replaced. The label
drained after the first five builds, which is the reported behaviour. There was no growth when the
spares could not keep up and no way for a label to shed spares gradually.

A first attempt at a target that survives the queue emptying grew only on
`queued > spares + provisioning`, and that turned out to be a signal the normal flow never produces:
the agent Jenkins raises for a waiting build is itself in flight for the label, so a single build
waiting for a single instance never looks like starvation. A label with a base of 0 stayed at 0
forever. The thing that reliably says "this label is in use and just lost a spare" is an executor
being taken, which is also the moment the replacement should start booting rather than waiting up to
a minute for the next periodic pass.

## The model now

`HotSpareDemand` keeps a target per cloud and label, in memory, and moves it in steps of
`scalingFactor`:

| Situation | Signal | Effect |
| --- | --- | --- |
| The label is in use at all | an executor taken, or anything queued or running | target is at least one step |
| Work in sight above that | `queued + busy` | target follows it immediately |
| Spares taken the instant they appear, or a queue nothing in flight can serve | `consumed > 0 && spares == 0`, or `queued > spares + provisioning` | target raised to one step above the load, at most once a minute |
| Nothing running, queued, or starting | quiet for one idle timeout | target gives up one step |
| A spare reclaimed for being idle | `EC2RetentionStrategy` idle timeout | target gives up one step |

Bounds: `baseHotSpares` is the floor, `maxHotSpares` the ceiling, and the cover cannot take the
target beyond `queued + busy + step`. That last bound is what stops a label whose templates are all
at their instance caps from accumulating a backlog it would try to launch the moment capacity
appeared, and it means a saturated label settles at its concurrency plus a step rather than climbing
without limit.

The step is the floor and the cover, not the rate. An earlier version grew one step per minute
whatever the load, which meant a build fanning out to 20 branches spent four minutes at 5, 10, 15
before reaching 20 while nineteen branches waited. The load is a measurement, so it is followed at
once and only the cover above it is rate-limited.

Two events drive a pass, and between them they cover both cases that matter for latency:

- `EC2HotSpareQueueListener.onEnterBuildable` — a build is ready to run. This is the only signal
  available for a label with no capacity at all, because the other one needs an executor and there
  is no agent to provide one. Without it the first build of a quiet period waited for the periodic
  sweep before any spare was even requested. Core adds the item to `Queue.buildables` before
  notifying listeners, so the pass this schedules counts it.
- `EC2RetentionStrategy.taskAccepted` — an executor was taken, recorded against every rule whose
  label covers the agent's template.

Both run on the locked queue path, so both do nothing beyond bookkeeping and
`MinimumInstanceChecker.scheduleCheck()`. The provisioning runs on the checker thread, so nothing
about this waits on EC2 where a build is trying to start. `EC2HotSpareChecker`'s one-minute sweep is
now only a backstop, mostly for letting an idle label fade. The top-up counts only idle agents against
the target, so the agent that just went busy is a shortfall of one and its replacement launches while
that build runs. That is what keeps a label warm for a whole busy period rather than for its first
few builds.

Counting what is already on its way had to be tightened for this. `countCurrentNumberOfProvisioning
Agents(ForLabel)` required `Computer.isConnecting()`, which is false between a node being attached
and its launcher starting. With passes now arriving whenever a build queues or starts, two of them
falling either side of that gap launched the same shortfall twice — visible as 24 instances for a
target of 20. Both counts now include any idle, offline agent that is not temporarily offline; an
agent that never comes up is the grace period's problem, not the counter's.

Because a check is now triggered by every build queued and every build started, `scheduleCheck`
drops a request if one is already queued and has not looked at Jenkins yet: an unbounded queue of identical passes behind the
single checker thread would be pointless work. The cover step is separately rate-limited to one per
minute (`jenkins.ec2.hotSpareGrowthIntervalMs`), so a burst cannot add several steps of cover before
the first instances have booted. Following the load needs no rate limit, being idempotent: the same
load seen ten times in a minute asks for the same number.

Decay on reclaimed spares matters as much as the timer: the step is the same size as the growth
step, so it outruns agents timing out one at a time and the label does not re-provision the agents
it is in the middle of giving up.

The target is not persisted. After a restart a label starts at its base count and re-learns within
a few minutes, which is safer than restoring a number describing load that has since gone away.

## Behaviour this produces

- Idle Jenkins, base 0: no spares at all. Queueing the first build asks for a step of spares at
  once, so it starts on whichever agent comes up first instead of waiting for one instance raised on
  its own account, and the builds behind it find the rest warm.
- A wide parallel build: 20 branches queue, the target goes to 20 in that pass. The agents Jenkins
  is already raising for those branches count as in flight, so the spares are launched for the wave
  after them rather than doubling up on the wave in hand.
- Sustained load: every executor taken starts a replacement immediately, the target tracks the
  queued and running work, and a label that keeps emptying its pool gets one step of cover on top.
- Load stops: spares sit idle, each reclamation drops the target a step, and the label returns to
  its base count. With a base of 0 the label scales to nothing.

## Coverage

- `HotSpareDemandTest`: warm-up from nothing on an executor taken, a shortfall per executor taken,
  a 20-branch build taking the target to 20 in one pass, the target following the load up with the
  step as its floor, one step of cover and no more when the pool keeps running dry, a single build
  bounded to its concurrency plus a step, holding while spares are taken with warm ones left, fading
  once the builds stop, settling once in-flight covers the queue, repeated checks not climbing, fade
  to base, fade no further than base, reclaim, ceiling, zero step, agents that never time out,
  billing-period timeouts, per-label isolation, cover bounded by work in sight.
- `LabelHotSpareCheckerTest`: queueing a build provisions spares with nothing but `Queue.maintain()`
  (no sweep, no manual pass), taking an executor starts the next spare the same way, a 20-wide
  parallel build warming 20 spares through the real queue, spares replaced as they are consumed (the
  reported defect), and a quiet label stops replacing them. A `QueueTaskDispatcher` holds the builds
  in the queue so a branch being picked up cannot move the counts mid-assertion, and Jenkins' own
  `NodeProvisioner.Strategy` extensions are removed so every agent counted is one the hot spare pass
  launched.
- `EC2RetentionStrategyTest`: reclaiming an idle spare lowers the target for its label.
- `EC2QueueMaintenanceLatencyTest`: the queue-lock paths stay fast, which is what the split between
  the executor thread and the checker thread protects.

## Configuration wording

`scalingFactor` keeps its name for compatibility but is presented as *Hot spare step*; the help for
the base count, the step, the ceiling and the idle timeout all describe the loop, including that the
idle timeout is both the agents' timeout and the rate the prediction fades at.
