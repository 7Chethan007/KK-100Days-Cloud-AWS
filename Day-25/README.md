# Day 25 — EC2 Instance and CloudWatch CPU Alarm

A KodeKloud "100 Cloud/AWS" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

Day 24 put an ALB in front of an EC2 instance and verified health
checks plus an end-to-end HTTP response — two resources agreeing. This
lab chains *three* separate AWS services together for the first time:
EC2 (the workload), CloudWatch (observes and evaluates), and SNS
(distributes the notification) — each with its own, independently
verifiable piece of the pipeline.

---

## 1. Scenario

Launch an Ubuntu EC2 instance and configure a CloudWatch alarm that
publishes to an SNS topic when average CPU utilization reaches or
exceeds 90% over one 5-minute evaluation period.

| Resource/setting | Required value |
|---|---|
| EC2 instance | `xfusion-ec2`, Ubuntu |
| Alarm name | `xfusion-alarm` |
| Metric | `AWS/EC2` / `CPUUtilization`, `Average`, 300s |
| Comparison | `GreaterThanOrEqualToThreshold`, 90% |
| Evaluation | 1 datapoint out of 1 |
| Alarm action | SNS topic `xfusion-sns-topic` |
| Region | `us-east-1` |

```text
EC2 (xfusion-ec2)
      │ emits CPUUtilization metric
      ▼
CloudWatch alarm (xfusion-alarm)
      │ evaluates: avg(CPU, 300s) >= 90%, 1-of-1
      ▼ (on ALARM state)
SNS topic (xfusion-sns-topic)
      │ publishes
      ▼
Subscribers (email/SMS — must separately confirm)
```

---

## 2. Reasoning model — how to *derive* the configuration, not memorize it

### 2.1 Three services, three separate responsibilities — none substitutes for another

```text
EC2:        RUNS the workload, emits raw metric data
CloudWatch: OBSERVES metrics, EVALUATES a threshold condition
SNS:        DISTRIBUTES a notification once CloudWatch decides to act
```

EC2 doesn't know or care about any threshold; CloudWatch doesn't
calculate CPU usage itself, only evaluates what EC2 already publishes;
SNS doesn't evaluate anything, it only fans a message out to whoever
subscribed. This separation is *why* the same SNS topic can receive
alarms from many different CloudWatch alarms, and why the same alarm
could in principle notify multiple topics — each service is a
reusable, independent building block, the same "separately-creatable,
separately-attachable" pattern as IAM's User/Group/Policy/Role split
in Days 16-20.

### 2.2 A metric is observed data; an alarm is a *rule* evaluated against that data — two different objects entirely

```text
Metric:  CPUUtilization — a time series of numbers EC2 publishes
Alarm:   xfusion-alarm — a RULE referencing that metric + a threshold + a comparison
```

Creating the alarm doesn't create or affect the metric — the metric
exists (or doesn't yet have data) independent of whether any alarm
watches it. This is why a brand-new alarm on a brand-new instance
starts in `INSUFFICIENT_DATA`: the *rule* exists immediately, but the
*metric data* it needs to evaluate against hasn't arrived yet. Neither
object's existence implies anything about the other's state.

### 2.3 "1 out of 1" and "Period 300" are two independent dials, not one combined setting

```text
Period:              how long each MEASUREMENT WINDOW is (300s = 5 min)
Evaluation periods:   how many windows are considered
Datapoints to alarm:  how many of those windows must BREACH
```

"1 out of 1" means exactly one 5-minute window is considered, and that
one window breaching is sufficient — it does *not* mean CPU must stay
above 90% at every single instant inside those 5 minutes, only that
the *average* over the whole window reaches the threshold. Confusing
"one breaching datapoint out of one evaluation period" with "90%
sustained continuously" would lead to misjudging what actually
triggers the alarm.

### 2.4 `GreaterThanOrEqualToThreshold` is a specific, exact comparison — 90% itself qualifies

```text
GreaterThanThreshold            → 90% does NOT breach (needs STRICTLY greater)
GreaterThanOrEqualToThreshold   → 90% DOES breach
```

This is a one-word difference in the comparison operator with a
concrete behavioral consequence at the boundary value — the same
"exact setting, not an approximately-equivalent one" discipline as
every precise flag/value choice throughout this series (Day 20's
exact policy name, Day 19's exact socket path). Picking
`GreaterThanThreshold` by habit instead of reading the requirement
carefully would produce an alarm that silently fails to fire exactly
at the boundary the lab cares about.

### 2.5 The EC2 *name tag* and *instance ID* are different identifiers used at different layers

```text
xfusion-ec2          ← human-readable Name TAG — used to FIND the instance
i-011bc259c78de067d  ← the actual InstanceId — used as the alarm's METRIC DIMENSION
```

CloudWatch's per-instance `CPUUtilization` metric is dimensioned by
`InstanceId`, not by a `Name` tag — so the alarm has to be built using
the *resolved* instance ID, found by filtering on the tag first. This
is the same "discover by filter, then act on the exact ID" two-step
pattern used throughout the AWS CLI side of this series
(`--filters "Name=tag:Name,..."` then operating on the returned
`InstanceId`), here applied to building an alarm's dimension instead
of a `describe-*` query.

### 2.6 A correct threshold configuration and a correctly-wired notification action are two separate, separately-checkable facts

```text
Threshold/period/comparison correct   → the ALARM LOGIC is right
AlarmActions pointing at the topic ARN → the NOTIFICATION PATH is right
```

An alarm could have a flawless threshold configuration and still never
notify anyone if `AlarmActions` is empty or points at the wrong ARN —
these are independent fields on the same resource, and verifying one
says nothing about the other. This is exactly Day 20's "trust policy
vs. permissions policy" split and Day 22's "authorized_keys vs.
known_hosts" split, recurring a third time: two genuinely separate
concerns bundled into what looks like one configuration task.

### 2.7 A topic existing and a subscriber being able to receive notifications are also two separate facts

```text
Topic exists (xfusion-sns-topic)         → the CHANNEL exists
Subscription confirmed (email/SMS, etc.)  → someone is actually LISTENING on it
```

`sns create-topic` creates a channel with zero subscribers by default
— nothing is automatically listening. An email subscription
specifically requires a *separate confirmation step* (clicking a link
in a confirmation email) before it actually receives anything; an
unconfirmed subscription exists in SNS's records but delivers nothing.
This lab's required check only verifies the alarm's action *points at*
the topic — delivery to a real human is a distinct, optional
end-to-end test (§2.8), not part of the core requirement.

### 2.8 Testing SNS delivery directly and testing the alarm's actual trigger path are two different tests

```text
aws sns publish ...                 → tests whether SNS can deliver to subscribers AT ALL
Pushing CPU to 90%+ and waiting      → tests whether the ALARM ITSELF actually fires and acts
```

Manually publishing a test message to the topic proves SNS and its
subscribers work — it says nothing about whether the CloudWatch
alarm's evaluation logic or its action wiring is correct, since that
path was never exercised. Conversely, the configuration checks in
§4.8 (§Check 2/4) prove the alarm and its action are wired correctly
without ever needing to actually spike CPU or wait for a real
notification — each test isolates a different part of the pipeline.

### 2.9 `INSUFFICIENT_DATA` immediately after creation is an expected transient state, not a configuration defect

```text
New alarm, reason: "Unchecked: Initial alarm creation"
```

A brand-new alarm has had zero opportunities to evaluate real metric
data yet — `INSUFFICIENT_DATA` with that specific reason string is the
expected starting state for *any* newly created alarm, before enough
time has passed for a full period's worth of data to arrive and be
evaluated. Treating this as a failure signal and re-editing the alarm
configuration in response would be solving a problem that doesn't
exist.

### 2.10 The compressed reasoning chain

```text
Requirement: EC2 CPU >= 90% over 5 min → notify via SNS
   → verify identity/region first: sts get-caller-identity, region = us-east-1
   → launch xfusion-ec2 (Ubuntu), VERIFY it's running + capture its InstanceId (§2.5)
   → confirm/create the SNS topic xfusion-sns-topic, capture its ARN
   → (optional) subscribe + CONFIRM an email endpoint — separate from the core requirement (§2.7)
   → put-metric-alarm with:
       --dimensions Name=InstanceId,Value=<resolved ID>   (NOT the Name tag, §2.5)
       --statistic Average --period 300                    (one dial, §2.3)
       --evaluation-periods 1 --datapoints-to-alarm 1       (second dial, §2.3)
       --comparison-operator GreaterThanOrEqualToThreshold  (exact operator, §2.4)
       --threshold 90
       --alarm-actions <topic ARN>                          (the SEPARATE notification check, §2.6)
   → verify THRESHOLD CONFIG and ALARM ACTIONS as two SEPARATE describe-alarms checks (§2.6)
   → INSUFFICIENT_DATA right after creation is expected — not a failure (§2.9)
```

---

## 3. Concepts (reference)

### 3.1 EC2 / CloudWatch / SNS responsibility split
EC2 runs the workload and emits metrics; CloudWatch evaluates those
metrics against a configured rule (an alarm); SNS distributes a
notification once an alarm transitions to the `ALARM` state. Each
service is independently created and composable with the others.

### 3.2 Metric vs. alarm
A metric is observed time-series data (e.g. `CPUUtilization`). An
alarm is a separately-created rule referencing a metric, a statistic,
a period, and a threshold comparison — creating one doesn't create or
alter the other.

### 3.3 Period, evaluation periods, and datapoints-to-alarm
Three independent dials: how long each measurement window is
(`Period`), how many windows are considered (`EvaluationPeriods`), and
how many of those windows must breach for the alarm to fire
(`DatapointsToAlarm`).

### 3.4 Comparison operator precision
`GreaterThanThreshold` and `GreaterThanOrEqualToThreshold` differ
specifically at the boundary value — an exact requirement like "90% or
above" maps to the inclusive operator, not the exclusive one.

### 3.5 Resource name tag vs. resource ID as a metric dimension
CloudWatch's EC2 per-instance metrics are dimensioned by `InstanceId`,
not by any tag — tags are for human/CLI discovery, the resolved ID is
what the alarm's dimension actually references.

### 3.6 Alarm threshold logic vs. alarm action wiring
The condition that determines `OK`/`ALARM` state and the action taken
when that state is reached (`AlarmActions`) are independent,
separately-configured and separately-verifiable fields on the same
alarm resource.

### 3.7 SNS topic vs. subscription vs. confirmed subscription
A topic is a channel with no subscribers by default. A subscription
registers an endpoint to that channel. For protocols like email, the
subscription additionally requires explicit confirmation before it
receives real notifications.

### 3.8 Alarm states: `OK`, `ALARM`, `INSUFFICIENT_DATA`
`OK` means the threshold isn't currently breached; `ALARM` means it
is; `INSUFFICIENT_DATA` means CloudWatch hasn't yet evaluated enough
real data to determine either — a normal, expected state immediately
after alarm creation.

---

## 4. Runbook

### 4.1 Verify identity and region
```bash
aws sts get-caller-identity
aws configure get region
aws ec2 describe-regions --region us-east-1 \
  --query 'Regions[?RegionName==`us-east-1`].RegionName' --output text
```

### 4.2 Launch the Ubuntu EC2 instance (Console)
```text
EC2 → Launch instances → Name: xfusion-ec2 → Ubuntu Server LTS AMI
  → t2.micro → configure key pair/networking per lab → Launch
```

### 4.3 Verify the instance
```bash
aws ec2 describe-instances --region us-east-1 \
  --filters "Name=tag:Name,Values=xfusion-ec2" \
  --query 'Reservations[].Instances[].{ID:InstanceId,Name:Tags[?Key==`Name`]|[0].Value,State:State.Name,Type:InstanceType}' \
  --output table
```
```text
ID                   Name          State    Type
i-011bc259c78de067d   xfusion-ec2   running  t2.micro
```

### 4.4 Confirm the SNS topic exists
```bash
aws sns list-topics --region us-east-1 \
  --query "Topics[?contains(TopicArn, 'xfusion-sns-topic')].TopicArn" --output table
```
```text
arn:aws:sns:us-east-1:491620046448:xfusion-sns-topic
```
(If missing: `aws sns create-topic --region us-east-1 --name xfusion-sns-topic`.)

### 4.5 (Optional) subscribe and confirm an email endpoint
```bash
aws sns subscribe --region us-east-1 \
  --topic-arn arn:aws:sns:us-east-1:491620046448:xfusion-sns-topic \
  --protocol email --notification-endpoint your-email@example.com
```
Confirm via the email SNS sends — unconfirmed subscriptions receive
nothing (§2.7).

### 4.6 Create the alarm
```bash
INSTANCE_ID=$(aws ec2 describe-instances --region us-east-1 \
  --filters "Name=tag:Name,Values=xfusion-ec2" \
  --query 'Reservations[].Instances[].InstanceId' --output text)

TOPIC_ARN=$(aws sns list-topics --region us-east-1 \
  --query "Topics[?contains(TopicArn, ':xfusion-sns-topic')].TopicArn | [0]" --output text)

aws cloudwatch put-metric-alarm \
  --region us-east-1 \
  --alarm-name xfusion-alarm \
  --alarm-description "Alert when EC2 average CPU utilization is at least 90% over five minutes" \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions "Name=InstanceId,Value=$INSTANCE_ID" \
  --statistic Average \
  --period 300 \
  --evaluation-periods 1 \
  --datapoints-to-alarm 1 \
  --threshold 90 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --treat-missing-data missing \
  --alarm-actions "$TOPIC_ARN"
```
`put-metric-alarm` creates or updates; no output on success.

### 4.7 Verify the alarm's threshold configuration
```bash
aws cloudwatch describe-alarms --region us-east-1 --alarm-names xfusion-alarm \
  --query 'MetricAlarms[].{Name:AlarmName,Metric:MetricName,Namespace:Namespace,Statistic:Statistic,Period:Period,Operator:ComparisonOperator,Threshold:Threshold,EvaluationPeriods:EvaluationPeriods,DatapointsToAlarm:DatapointsToAlarm,ActionsEnabled:ActionsEnabled}' \
  --output table
```
Expect: `CPUUtilization` / `AWS/EC2` / `Average` / `300` /
`GreaterThanOrEqualToThreshold` / `90.0` / `1` / `1` / `True`.

### 4.8 Verify the alarm's action — a SEPARATE check (§2.6)
```bash
aws cloudwatch describe-alarms --region us-east-1 --alarm-names xfusion-alarm \
  --query 'MetricAlarms[0].AlarmActions' --output json
```
```json
["arn:aws:sns:us-east-1:491620046448:xfusion-sns-topic"]
```

### 4.9 (Optional) inspect recent CPU data if stuck in `INSUFFICIENT_DATA`
```bash
aws cloudwatch get-metric-statistics --region us-east-1 \
  --namespace AWS/EC2 --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-011bc259c78de067d \
  --start-time "$(date -u -v-15M +%Y-%m-%dT%H:%M:%SZ)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --period 300 --statistics Average
```
(`-v-15M` is macOS `date`; use `-d '15 minutes ago'` on Linux.)

### 4.10 (Optional) test SNS delivery directly — not a test of the alarm itself (§2.8)
```bash
aws sns publish --region us-east-1 \
  --topic-arn arn:aws:sns:us-east-1:491620046448:xfusion-sns-topic \
  --subject "SNS delivery test" --message "Testing SNS notification delivery."
```

---

## 5. Final state

| Requirement | Result |
|---|---|
| `xfusion-ec2` running, Ubuntu | ✅ |
| Alarm `xfusion-alarm` references `CPUUtilization`/`AWS/EC2` for the correct `InstanceId` | ✅ |
| `Average`, `Period=300`, `GreaterThanOrEqualToThreshold`, `Threshold=90` | ✅ |
| `EvaluationPeriods=1`, `DatapointsToAlarm=1` | ✅ |
| `ActionsEnabled=True`, `AlarmActions` → `xfusion-sns-topic` ARN | ✅ |
| Initial state `INSUFFICIENT_DATA` ("Unchecked: Initial alarm creation") — expected | ✅ |

```text
us-east-1
 ├── EC2: xfusion-ec2 (i-011bc259c78de067d, running)
 ├── CloudWatch alarm: xfusion-alarm
 │      metric: AWS/EC2 CPUUtilization, Average, 300s
 │      condition: >= 90%, 1-of-1
 │      action → arn:aws:sns:us-east-1:491620046448:xfusion-sns-topic
 └── SNS topic: xfusion-sns-topic
        subscribers: (optional, separately confirmed)
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Alarm stuck in `INSUFFICIENT_DATA` | No metric data evaluated yet — expected right after creation | Confirm instance is running, `InstanceId` matches; wait and recheck; inspect via `get-metric-statistics` |
| SNS topic missing in console | Wrong region/account, or not yet created | Check region = `us-east-1`; `sts get-caller-identity`; `sns list-topics`; create if genuinely missing |
| Alarm exists but no notification received | `AlarmActions` missing/wrong, no subscriber, or email unconfirmed | `describe-alarms` → check `AlarmActions`; `list-subscriptions-by-topic`; confirm the email |
| `AccessDenied` | Identity lacks permission for the specific API call | Check IAM permissions for the exact failing action — don't assume misconfiguration |
| Alarm fires at 89.9% but not 90% exactly | Wrong comparison operator (`GreaterThanThreshold` instead of `GreaterThanOrEqualToThreshold`) | Re-check `ComparisonOperator` via `describe-alarms` (§2.4) |
| Alarm dimension doesn't match the intended instance | Built using the `Name` tag instead of the resolved `InstanceId` | Re-resolve `InstanceId` via `describe-instances --filters`, update the alarm's dimension |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.10) out
      loud from "launch `xfusion-ec2`" to "verify `AlarmActions`
      separately from the threshold config."
- [ ] Explain, in one sentence, why EC2, CloudWatch, and SNS are three
      separate services rather than one combined monitoring feature.
- [ ] Explain why `INSUFFICIENT_DATA` right after creating a new alarm
      is expected, not a sign of misconfiguration.
- [ ] Explain the difference between `GreaterThanThreshold` and
      `GreaterThanOrEqualToThreshold` at exactly the threshold value.
- [ ] Explain why the alarm's dimension uses the EC2 instance ID, not
      its `Name` tag.
- [ ] Explain why checking `AlarmActions` is a separate verification
      step from checking the threshold/period/comparison fields.
- [ ] From memory, create a second alarm on a different metric (e.g.
      `StatusCheckFailed`) pointing at the same SNS topic, and verify
      both its condition and its action independently — without
      copying any command from this file.

---

## 8. Document template for future lab days

Use this skeleton for `day-N/README.md`:

```markdown
# Day N — <task title>

## 1. Scenario
<what was asked, which account/region/resources, constraints, credentials
if relevant>

## 2. Reasoning model — how to derive the commands
<walk the requirement down to a subsystem/resource hierarchy, step by step,
in the order you'd actually discover it: "what resource, is it global or
region-scoped, what existing state constrains my choice, what CLI
service+operation performs the read (exact lookup vs. filtered search),
what performs the write, how do I verify the SPECIFIC field the
requirement names." End with a compressed step-chain.>

## 3. Concepts (reference)
<one subsection per concept the task exercises — explain WHY, not just WHAT>

## 4. Runbook
<copy-pasteable commands in the order run, WITH the actual intermediate
output/results captured inline as code blocks, not just the commands>

## 5. Final state
<table + diagram of what the infrastructure looks like after completion>

## 6. Troubleshooting
<symptom / cause / fix table>

## 7. Do it yourself (checklist)
<"walk the reasoning chain out loud" + comprehension questions +
"redo without copy-pasting, recompute derived values" prompt>
```
