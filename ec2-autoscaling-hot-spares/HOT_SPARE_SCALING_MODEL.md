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
| In use, but with less work than the target holds | `queued + busy < target` for a whole idle timeout | target gives up one step, no lower than the load |
| Nothing running, queued, or starting | quiet for one idle timeout | target gives up one step, no lower than the base |

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

## The target owns retention, not the idle timeout

An earlier version let the idle timeout reclaim any spare that sat unused for its timeout, and
lowered the target a step each time one was reclaimed. That produced exactly the churn the feature
exists to avoid: a label holding the five spares its target asked for would time one out, the next
pass would see a shortfall of one and launch a replacement, and a build arriving in between waited
for a boot that had just been thrown away. With a base count above zero it never even settled — the
target could not fall below the base, so the pool recycled itself forever.

So the target decides how many spares a label keeps, and an agent is reclaimed only once the target
has come down past it:

- `MinimumInstanceChecker.isSpareStillWanted` ranks the label's spares by how long each has been
  idle and keeps the freshest `target` of them. Ranking rather than comparing counts is what makes
  several agents checking at once agree on which are the extras: each one asks where it sits in the
  same order, so exactly `spares - target` are released instead of all of them deciding they are
  surplus. The longest idle goes first, which is the behaviour an idle timeout implies.
- `EC2RetentionStrategy` consults it on both idle paths, the plain timeout and the billing-period
  one, and skips the termination while the label still wants the agent. The idle clock is not
  reset, so the agent goes promptly once the target does fall past it.
- Agents that cannot take work are not spares and are never protected by this: `isSpare` requires
  idle, online, accepting tasks and not temporarily offline, so an agent that has drained its
  maximum uses is reclaimed as before rather than satisfying the target while being useless. That
  also fixes a counting bug of its own, since such an agent used to count towards the target and
  suppress the provisioning of a real spare.

## Template schedules

`SlaveTemplate.minimumNumberOfInstancesTimeRangeConfig` - *Only apply minimum number of instances
during specific time range* - was read by the per-template minimum-instances loop and by the
retention strategy, but not by the label pass, so a rule warmed a template around the clock and
undid the schedule. That is the workaround issue 2028 describes: a template per shift, offset so the
spares are only paid for during working hours.

The schedule stays a property of the template, which is what makes that arrangement work as one
pool:

- `provisionAcrossLabelGroup` skips a template outside its range and offers its share to the next
  one, the same way it treats a template at its instance cap. A label with a template per shift
  therefore warms up on whichever one is on duty.
- `isSpare` returns false for an agent whose template is outside its range, so those agents neither
  count towards the target nor are protected from the idle timeout. When a range closes, its agents
  are released at their idle timeout while the open template warms up, and the pool migrates.
- Nothing here gates the *load* signals. Builds queued for the label and agents running one are
  measurements of demand and are counted whatever the schedule says; the schedule decides where
  spares may be held, not whether the label is busy.

A label whose every template is off duty simply holds no spares, and builds for it provision on
their own account as they always did - the schedule is about warm capacity nobody asked for, not
about refusing work.

Making retention follow the target means the target has to come down on its own, which is what the
in-use fade above is for. Without it a single burst would teach a label a peak, and the guard would
hold that peak for as long as the label stayed busy at all.

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
- Load drops but does not stop: the pool is left alone for an idle timeout, then gives up a step at
  a time down to the work still in sight. Nothing is recycled while the target still wants it.
- Load stops: the target gives up a step per idle timeout down to the base count, and the spares
  above it are reclaimed as it passes them. With a base of 0 the label scales to nothing.

## Coverage

- `HotSpareDemandTest`: warm-up from nothing on an executor taken, a shortfall per executor taken,
  a 20-branch build taking the target to 20 in one pass, the target following the load up with the
  step as its floor, one step of cover and no more when the pool keeps running dry, a single build
  bounded to its concurrency plus a step, holding while spares are taken with warm ones left, fading
  once the builds stop, settling once in-flight covers the queue, repeated checks not climbing, fade
  to base, fade no further than base, the fade towards the load while the label stays busy, work
  coming back mid-fade stopping it, ceiling, zero step, agents that never time out,
  billing-period timeouts, per-label isolation, cover bounded by work in sight.
- `LabelHotSpareCheckerTest` also covers the schedules: no spares outside a template's time range,
  the same rule provisioning inside it, and a label with a day and a night template keeping its
  spares on the one that is on duty. `EC2RetentionStrategyTest` covers the other half, an idle spare
  released once its template's range closes.
- `LabelHotSpareCheckerTest`: queueing a build provisions spares with nothing but `Queue.maintain()`
  (no sweep, no manual pass), taking an executor starts the next spare the same way, a 20-wide
  parallel build warming 20 spares through the real queue, spares replaced as they are consumed (the
  reported defect), and a quiet label stops replacing them. A `QueueTaskDispatcher` holds the builds
  in the queue so a branch being picked up cannot move the counts mid-assertion, and Jenkins' own
  `NodeProvisioner.Strategy` extensions are removed so every agent counted is one the hot spare pass
  launched.
- `EC2RetentionStrategyTest`: a spare the target still wants outlives its idle timeout, and one the
  target has given up on is reclaimed.
- `EC2QueueMaintenanceLatencyTest`: the queue-lock paths stay fast, which is what the split between
  the executor thread and the checker thread protects.

## Configuration wording

`scalingFactor` keeps its name for compatibility but is presented as *Hot spare step*; the help for
the base count, the step, the ceiling and the idle timeout all describe the loop, including that the
idle timeout is both the agents' timeout and the rate the prediction fades at.
