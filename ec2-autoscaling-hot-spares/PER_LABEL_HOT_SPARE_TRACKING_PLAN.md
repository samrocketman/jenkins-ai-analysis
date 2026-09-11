# Plan: Track hot spare scaling per requested label

Target: `/workspace/queue-debugging/ec2-plugin` (branch `hot-spares-by-label`).

A hot spare rule currently owns one pool: the union of every template carrying any label the rule
names. Demand measured anywhere in that union is satisfied anywhere in that union. This plan makes
the *label a job asked for* the unit of scaling, and leaves the rule as the policy that applies to
the labels it covers.

---

## 1. The defect, as observed

A pull request build for x64 medium hardware on `Jenkins NG staging green agents` provisioned the
instance it asked for, `spot medium x64 c5a.2xlarge`, and then six idle `medium arm64 c6g.2xlarge`
agents and three idle `small arm64 c6g.xlarge` agents that nothing could use.

Resolving the rules of `jenkins-ng/configs/aws-cloud-config.groovy` against the generated templates
shows why:

```
rule 'x86_64_medium || arm64_medium' covers 10 templates
    medium x64 c6a.2xlarge     [x86_64_medium_pr x86_64_medium jervis_generator]   ... 5 of these
    medium arm64 c6g.2xlarge   [arm64_medium_pr arm64_medium]                      ... 5 of these
    queued labels that raise this rule's target:
        [x86_64_medium_pr, x86_64_medium, arm64_medium_pr, jervis_generator]

rule 'x86_64_small || arm64_small || jervis_generator' covers 15 templates
    medium x64 (5), small x64 (5), small arm64 (5)
    queued labels that raise this rule's target:
        [x86_64_medium_pr, x86_64_medium, jervis_generator, x86_64_small, arm64_small]
```

Three code paths produce this:

- `MinimumInstanceChecker.countQueueItemsForLabel` counts a queued item when **any** template of the
  group could serve it, so an `x86_64_medium_pr` build raises the medium rule's target, and (via the
  `jervis_generator` label on the x64 medium templates) the small rule's target as well.
- `provisionAcrossLabelGroup` satisfies the shortfall from **any** template of the group, walking
  `orderTemplatesForLabel`. Each compute type is ranked `10..1` independently, so `medium x64 c6a`
  and `medium arm64 c6g` tie at weight 5 and form one band that rotates: roughly every other pass
  leads with arm64.
- `agentsForLabel` counts spares, busy agents and in-flight agents across the whole group, so the
  arm64 spares look like progress towards the x64 demand that created them.

The x64 spares are then consumed by builds and retire at `max_total_uses: 1`, while the arm64 spares
can never be taken by an x64 build, so only the unusable half accumulates.

This is not a `_pr` problem. `_pr` is an admin convention; the plugin must not know about it. The
problem is that a rule naming several labels declares them one interchangeable pool.

---

## 2. Target model

**A rule is a policy, not a pool.** `HotSpareConfigByLabel` keeps its fields (`scalingFactor`,
`baseHotSpares`, `maxHotSpares`, `idleTimeoutMinutes`, `gracePeriodMinutes`, the override flags) and
its label expression now says *which labels the policy applies to*.

**The unit of scaling is a tracked label**, and each one gets its own `HotSpareDemand` entry, its own
prediction, and its own provisioning group:

| Aspect | Today | Planned |
| --- | --- | --- |
| Target key | cloud + rule expression | cloud + requested label |
| Group | templates matching the rule expression | templates matching the requested label |
| Queued input | items any group template could serve | items whose assigned label *is* this label |
| Busy input | agents of the group running anything | agents running work that asked for this label |
| Spare input | idle agents of the group | idle agents that can serve this label (unchanged in kind) |
| Consumed | attributed to the rule | attributed to the label the task asked for |

With the rules in `aws-cloud-config.groovy` as they now stand, rule one governs four independent
targets — `x86_64_medium`, `arm64_medium`, `x86_64_medium_pr`, `arm64_medium_pr` — and an x64 pull
request can only ever warm templates carrying `x86_64_medium_pr`.

### 2.1 Which labels are tracked

A label is tracked for a pass when it is one of:

1. a label a buildable queue item is waiting for (`Queue.Item.getAssignedLabel()`);
2. a label a running build asked for (`Executor.getCurrentWorkUnit().work.getAssignedLabel()`), so a
   label stays predicted while its work runs rather than only while it queues;
3. a label that already has a non-zero target, so a label that has gone quiet keeps fading a step per
   idle timeout instead of dropping to zero the moment its queue empties.

A tracked label is **governed** by the first rule, in configuration order, that matches at least one
of the templates serving that label. Ungoverned labels are ignored, exactly as today.

Nothing here assumes a tracked label is a single atom. A job may ask for any label expression, and
`Label.get(String)` parses it into a `Label` with a printed name and an evaluator
(`Label.matches(Collection<LabelAtom>)`), which is all the model needs: the name is the demand key
and the evaluator resolves the group. So a job requesting `foo && bar` is tracked in its own right
whenever a rule covers it:

- **served by** the templates carrying both `foo` and `bar`, since those are the only ones that can
  run the work — narrower than either atom's group, which is the point;
- **governed by** a policy such as `foo || bar`, because that expression matches those templates;
- **tracked separately** from `foo` and from `bar`, so a burst of `foo && bar` work warms only
  hardware that satisfies both, while an idle agent carrying both counts as a spare for all three
  targets because it genuinely can serve all three.

A compound expression with no template satisfying it has an empty group and is skipped, which is
correct: nothing this cloud can launch would run that work.

### 2.2 The rule floor stays with the rule

`baseHotSpares` is the admin saying "keep this many warm for this rule even when nothing is asking".
It stays a rule-level floor held across the rule's own templates, because spreading it over every
tracked label would multiply it by the number of labels named. A spare provisioned for the floor
still counts as a spare for every label it can serve, so the per-label shortfalls shrink accordingly
and nothing is provisioned twice.

*As implemented:* the floor needs no pass of its own. The label a rule **names** is itself a tracked
label, and it is the only one charged the floor (`base = labelName.equals(config.getLabel()) ?
config.getBaseHotSpares() : 0`, passed to a new `updateTarget` overload). Its target is therefore the
floor, provisioned across the rule's whole group by the ordinary per-label path, and — because the
target is what protects an agent from the idle timeout — the floor's agents are retained for the same
reason every other spare is. A single-label rule is exactly today's behaviour, since the label it
names and the label jobs ask for are the same entry.

`maxHotSpares` becomes a ceiling **per tracked label**. This is a documented behaviour change for
multi-label rules; single-label rules are unaffected.

### 2.3 Overlapping groups are intentional

`x86_64_medium_pr` is served by the five spot templates plus the five on-demand x64 templates;
`x86_64_medium` is served by the five on-demand x64 templates. An idle on-demand x64 agent is warm
capacity for both and counts for both, so the two targets share it rather than each provisioning its
own. Only the shortfall the shared agents do not cover reaches the spot templates.

A consequence worth stating plainly: because the `jenkins-ng` rules now name the `_pr` labels, and
because cost ranking puts spot at the top of the `_pr` groups, pull request hot spares will be warm
*spot* instances. That follows from the admin's rules, which is where the decision belongs.

---

## 3. Code changes

### 3.1 `MinimumInstanceChecker`

Rewrite `checkForLabelHotSpares(EC2Cloud)`:

```java
Map<String, HotSpareConfigByLabel> tracked = trackedLabels(cloud);   // §2.1, insertion ordered
for (Map.Entry<String, HotSpareConfigByLabel> entry : tracked.entrySet()) {
    Label label = Label.get(entry.getKey());
    Collection<SlaveTemplate> group = cloud.getTemplates(label);
    if (group.isEmpty()) { continue; }

    int spares        = countCurrentNumberOfSpareAgentsForLabel(cloud, label);      // unchanged
    int provisioning  = countCurrentNumberOfProvisioningAgentsForLabel(cloud, label); // unchanged
    int busy          = countAgentsRunningWorkFor(cloud, label);                     // new
    int queued        = countQueueItemsRequesting(label);                            // new
    int base          = namedByRule ? entry.getValue().getBaseHotSpares() : 0;       // §2.2

    int target   = HotSpareDemand.of(cloud, label.getName())
                       .updateTarget(entry.getValue(), base, spares, provisioning, queued, busy);
    int toLaunch = target - (spares + provisioning);
    if (toLaunch > 0) { provisionAcrossLabelGroup(cloud, label, group, toLaunch); }
}
HotSpareDemand.forgetZeroed(cloud, namedByRules);   // §3.2
```

New counting helpers, both keyed on the label a job asked for rather than on what a template could
serve:

```java
private static int countQueueItemsRequesting(@NonNull Label label) {
    return (int) Queue.getInstance().getBuildableItems().stream()
            .map(Queue.Item::getAssignedLabel)
            .filter(label::equals)
            .count();
}

private static int countAgentsRunningWorkFor(@NonNull Label label) {
    // Executor.getCurrentWorkUnit().work.getAssignedLabel(); a work unit with no label counts
    // for nothing, and a flyweight task never occupies an EC2 executor.
}
```

`isSpareStillWanted` loses its `HotSpareConfigByLabel` parameter and asks the question across every
tracked label the computer can serve, keeping the agent when any of them still wants it:

```java
public static boolean isSpareStillWanted(@NonNull EC2Cloud cloud, @NonNull EC2Computer computer)
```

The ranking within a label (freshest `target` spares kept, longest-idle released first) is unchanged;
it is simply evaluated per label.

`countQueueItemsForLabel` and the group-wide busy counter become unused by the checker. Keep them if
anything else needs them; otherwise delete rather than leave two counting models in the file.
*As implemented:* both were replaced outright by the label-attributed pair above.

### 3.2 `HotSpareDemand`

Already keyed `cloud.name + '\0' + label`, so the storage needs no change. Add:

- `static Set<String> trackedLabels(EC2Cloud cloud)` — the labels with a live entry, for §2.1 case 3;
- `static void spareConsumed(EC2Cloud cloud, String label)` — the attribution the retention strategy
  now has; the `HotSpareConfigByLabel` overload goes away;
- `static void forgetZeroed(EC2Cloud cloud, Set<String> keep)` — drop entries that have faded to
  zero, so a controller that has run for a month does not hold an entry per label ever requested. An
  entry above zero is never dropped, because its spares are still being kept for it and it must fade
  rather than vanish; nor is one with a spare taken since the last pass, which is a label about to
  want something. *As implemented:* `keep` is the set of labels the rules **name** rather than the
  labels the pass visited — a zeroed entry is visited by every pass precisely because it still
  exists, so keying the pruning on what was visited would never prune anything.

The prediction itself (`updateTarget`, growth interval, decay, ceiling) is unchanged. It becomes
*more* accurate simply because its inputs stop describing other people's work.

### 3.3 `EC2RetentionStrategy`

`taskAccepted(Executor, Queue.Task)` already has the task, so `noteConsumedSpare` takes the label it
asked for and records against that:

```java
Label assigned = task.getAssignedLabel();
if (assigned != null && cloud.getHotSpareConfigForLabel(assigned) != null) {
    HotSpareDemand.spareConsumed(cloud, assigned.getName());
    MinimumInstanceChecker.scheduleCheck();
}
```

`isWantedAsHotSpare` drops its loop over rules and calls the new single-argument
`isSpareStillWanted`.

### 3.4 `EC2Cloud`

Add `getHotSpareConfigForLabel(Label)`: the first rule matching any template that serves the label.
`getHotSpareConfigFor(SlaveTemplate)` stays as it is — retention and grace period resolution are
per template and do not change.

---

## 4. Worked example against the current `jenkins-ng` config

One pull request build queued for `x86_64_medium_pr`, nothing else running:

| | Today | Planned |
| --- | --- | --- |
| Targets raised | `x86_64_medium \|\| arm64_medium` and `x86_64_small \|\| ... \|\| jervis_generator` | `x86_64_medium_pr` only |
| Provisioned from | any of 10 medium templates, and any of 15 small/medium templates | the 10 templates carrying `x86_64_medium_pr` (5 spot, 5 on-demand x64) |
| Leads with | whichever of the tied weight-5 templates the band rotation picked | `spot medium x64 c5a.2xlarge`, weight 10 |
| Unusable capacity | 6 arm64 medium, 3 small arm64 | none |

A branch build for `x86_64_medium` raises only the `x86_64_medium` target, whose group is the five
on-demand x64 templates, so it never warms spot hardware it cannot use.

---

## 5. Compatibility

- A rule naming a single label behaves exactly as it does today: one tracked label, the same group,
  the same target. This is the shape of every example in the help text and in `description.md`.
- A rule naming several labels changes: each label scales independently, and `maxHotSpares` applies
  per label. This is the defect being fixed, and needs a line in the help text and the PR
  description.
- `baseHotSpares` behaviour is unchanged (§2.2).
- No configuration schema change, so no migration and no JCasC change.

---

## 6. Tests

Update:

- `util/LabelHotSpareCheckerTest` — the group-wide expectations become per-label ones.
- `HotSpareDemandTest` — unchanged in substance; add the pruning contract.
- `EC2RetentionStrategyTest` — consumption is attributed by task label; an agent is kept while any
  label it serves wants it.

Add, as the regression tests for this report:

1. Demand for `x86_64_medium_pr` provisions nothing carrying only `arm64_medium_pr`.
2. Demand for a label carried by one template of a rule does not raise the target of another label
   of the same rule (the `jervis_generator` to `small arm64` bleed).
3. An idle agent serving both `x86_64_medium` and `x86_64_medium_pr` satisfies both targets, and the
   two together do not double-provision.
4. A busy agent counts only for the label its build asked for.
5. A label that goes quiet fades a step per idle timeout and its entry is dropped once it reaches
   zero; a label above zero is never dropped.
6. `baseHotSpares` still holds its floor across the rule's templates with an empty queue.
7. A job requesting `foo && bar` under a policy of `foo || bar` is tracked on its own, provisions
   only from templates carrying both atoms, and does not raise the `foo` or `bar` targets; an idle
   agent carrying both counts as a spare for all three.

---

## 7. Risks

- **More warm capacity.** Four tracked labels under one rule can each hold `scalingFactor` spares,
  where the pooled rule held one set. `jenkins-ng` should revisit its factors and consider
  `max_hot_spares` per label. Worth calling out in the config comments, which still describe the
  pooled model.
- **Busy attribution depends on the work unit.** A build whose task exposes no assigned label counts
  for nothing; that is the safe direction (no phantom demand), but it means a label whose jobs are
  scheduled without a label expression will not grow from its running work.
- **Key growth.** Handled by `forgetZeroed`, but the pruning contract needs the test above so it
  cannot silently start dropping labels that still hold spares.
- **Expression spelling.** The demand key is the parsed label's name, so `foo&&bar` and
  `foo && bar` collapse to one entry while `bar && foo` is a second entry describing the same
  requirement. The cost is a duplicated prediction, not duplicated capacity: both entries count the
  same idle agents as their spares, so the second one is largely satisfied by the first. Not worth
  canonicalising unless it shows up in practice.
- **Upstream framing.** For PR #2036 this is a behaviour change to an unreleased feature, so no
  migration is needed, but the help text must state that a multi-label rule scales each label
  independently.

---

## 8. Follow-up outside the plugin

`jenkins-ng/configs/aws-cloud-config.groovy` still carries the comment block explaining that rules
deliberately avoid `_pr` labels because a `_pr` rule would be inflated by branch traffic. That
reasoning no longer holds once demand is tracked per label, and the rules have already been changed
to name the `_pr` labels, so the comment needs rewriting alongside this work.
