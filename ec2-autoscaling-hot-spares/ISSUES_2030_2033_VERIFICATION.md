# Verification against ec2-plugin issues 2030 and 2033

Post-implementation check of the hot-spares-by-label / grace-period / weighted-round-robin work
against two open upstream bug reports, plus the follow-up fix that issue 2030 turned out to need.

## Issue 2033 — async provisioning exceptions never fall through

Reported: a capacity failure raised inside the async provisioning future returns `null` without
trying the next template that matches the label, so the primary template is retried every cycle.

Already fixed by this work. `EC2Cloud.provisionFromGroup` runs inside the async task and walks the
remaining templates of the label group, so the failure is handled where it happens rather than
being handed back to the `NodeProvisioner` as a null node. Capacity failures are classified through
`SlaveTemplate.isInsufficientCapacityError`.

Covered by `EC2CloudProvisionFailoverTest`:

- capacity failure falls over to the second template, with the round-robin flag on and off,
- a non-capacity failure also falls over but records no cooldown,
- two consecutive requests each deliver an instance while the first template keeps failing.

Nuance worth knowing: the cooldown the reporter proposed exists, but only under the opt-in cloud
option `roundRobinTemplatesByLabel`. With the flag off, the exhausted template still leads every
cycle; it just no longer costs a cycle, because the fallback happens inside the same request.

## Issue 2030 — instances exceed the template cap under concurrent provisioning

Reported: two builds triggered within seconds of each other each launch a spot instance from a
template capped at one. The reporter found `-Djenkins.ec2.instanceCountCacheTtlMs=0` avoids it,
pointing at the instance count cache added in the Ludicrous Mode work.

**Not** fixed by this work. Reproduced with four simultaneous `provision(Label, 1)` calls against a
template capped at one instance: two instances were launched. A cloud-wide cap of two under the
same load produced three.

Cause: both launch paths hold `slaveCountingLock` while re-checking the cap, but the count they
check comes from a cache that was only dropped after the launch completed, outside the lock. A
request arriving in that window reads counts taken before either request launched anything. Waiting
for EC2 to report the launch is racy in its own right, which is what the reporter observed with
spot requests.

Fixed by counting launches this cloud has committed but EC2 has not reported yet:

- `EC2Cloud.recordInFlight` records the launched instance ids while the counting lock is still
  held, so the next request to reach the cap check already counts them,
- `getPossibleNewSlavesCount` subtracts those records from both the template and the cloud
  headroom, on top of whatever EC2 (or the cache) reports,
- `observeCountedInstances` drops a record once a count reports that instance, and
  `jenkins.ec2.inFlightInstanceTtlMs` (two minutes) discards records for instances that never
  appear, so a failed launch cannot block provisioning permanently,
- an instance the last cloud-wide count already reported is not recorded, because adopting an
  orphaned or stopped instance is not a new launch and must not be charged twice.

This keeps the count cache warm: correctness no longer depends on invalidating it, so the change
adds no EC2 calls. The reporter's `instanceCountCacheTtlMs=0` workaround is no longer needed.

Covered by `EC2CloudConcurrentProvisionTest` (template cap and cloud cap under four simultaneous
requests, both verified to fail without the fix) and by the orphan-adoption headroom case in
`EC2CloudInstanceCapTest`.

Residual limitation: the legacy spot path requests an instance (`sir-...`) rather than launching
one, and its id only becomes an instance id once fulfilled. Such a record expires by TTL and the
cap then relies on `describeSpotInstanceRequests` reporting the request, which it normally does
within seconds.

## Queue maintenance latency

`Queue.maintain()` runs under the Queue lock, so none of the above may reach an EC2 call from it.
The in-flight records are in-memory and are read under the counting lock, which is off that path;
the cap check itself is only reached from the `NodeProvisioner` thread and the provisioning
executor.

`EC2QueueMaintenanceLatencyTest` measures this with every EC2 call stubbed to take five seconds:
`Queue.maintain()`, `EC2RetentionStrategy.check`, `MinimumInstanceChecker.scheduleCheck` and
`EC2Cloud.canProvision` each stay under two seconds, while a control shows that provisioning does
pay the five seconds, so the budgets are discriminating.
