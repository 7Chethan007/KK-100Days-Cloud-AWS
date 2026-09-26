# Day 7 — Change an EC2 Instance Type

A KodeKloud "100 Cloud/AWS" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

---

## 1. Scenario

During the AWS migration, the Nautilus DevOps team discovered one EC2
instance was underutilized. Task:

1. Ensure `devops-ec2`'s status checks are complete (if still
   `Initializing`) before making any change.
2. Change its instance type from `t2.micro` to `t2.nano`.
3. Ensure `devops-ec2` is back in `running` state after the change.

**Credentials/environment** (lab-specific, rotates per session):

| Field | Value |
|---|---|
| Access | via `aws-client` host; run `showcreds` there to retrieve credentials |
| Region constraint | `us-east-1` (per this series' recurring convention) |

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 Find the resource by tag, not by memorized ID

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-ec2" \
  --query 'Reservations[].Instances[].{ID:InstanceId,Name:Tags[?Key==`Name`].Value|[0],State:State.Name,Type:InstanceType,AZ:Placement.AvailabilityZone}' \
  --output table
```

In real automation you rarely know a resource's ID ahead of time — you
know its human-assigned name. Filtering by the `Name` tag to *discover*
the instance ID is the same `--filters` pattern from Day 5's `test-ui`
EBS volume lookup: search by what a human labeled it, then operate on
the ID that search returns.

```text
Name tag  →  describe-instances --filters  →  InstanceId  →  perform the actual operation
```

### 2.2 Instance type vs. AMI — two independent axes, again

Same distinction as Day 6's EC2 launch lab, now showing up as a *change*
instead of a launch-time choice:

```text
AMI            → what OS/software is on the disk (unaffected by this task)
Instance type  → what virtual hardware backs it (t2.micro → t2.nano)
```

Changing `t2.micro` to `t2.nano` never touches the operating system or
its filesystem — it only changes the CPU/memory/network allocation the
same disk image runs on.

### 2.3 Why the instance must be stopped before the type can change

An instance type isn't a setting you can hot-swap on live virtual
hardware — AWS has to re-provision the instance onto hardware matching
the new type. The required lifecycle is:

```text
Running
   │
   ▼
Stop
   │
   ▼
Stopped
   │
   ▼
Change instance type
   │
   ▼
Start
   │
   ▼
Running
```

This is a concrete example of a broader pattern: **some infrastructure
changes require downtime because the underlying resource must be
re-provisioned, not just reconfigured.** Recognizing which category a
change falls into (in-place vs. requires-a-restart) is itself a skill —
here, the task statement telling you to check status checks *first* is a
hint that a stop/start cycle is coming.

### 2.4 Status checks vs. instance state — two different questions

The task specifically says to wait for status checks before touching
anything. These are genuinely different facts about the instance:

```text
Instance STATE            →  pending → running → stopping → stopped
                              (the EC2 lifecycle position)

Status CHECKS              →  SystemStatus, InstanceStatus
                              (AWS's own infrastructure health checks)
```

An instance can be `running` while its status checks are still
`Initializing` — the lifecycle says "the instance exists and is powered
on," while status checks say "AWS has actually confirmed the underlying
hardware and instance OS are responding correctly." Modifying an
instance that's still initializing risks acting on a resource whose true
health AWS hasn't confirmed yet — which is exactly why the task calls
this out as a precondition, not just a nice-to-have.

Critically: **`ok` status checks are not proof your application is
healthy** — they confirm AWS's infrastructure layer, nothing about
whatever you're running on top of it:

```text
AWS infrastructure
        │
   System check     ok
   Instance check    ok
        │
        ▼
   EC2 is running
        │
        ▼
Your application could still be completely broken
```

### 2.5 `aws ec2 wait` — synchronizing with asynchronous operations

Every EC2 state-changing API call (`stop-instances`, `start-instances`)
returns immediately with an acknowledgment — it does **not** block until
the instance actually reaches the target state. That's the same
"response received ≠ resource ready" lesson as Day 5's EBS volume
(`"State": "creating"` in the response, not yet `available`), now shown
for instance lifecycle transitions instead of volume creation:

```bash
aws ec2 wait instance-status-ok --instance-ids <id>
aws ec2 wait instance-stopped --instance-ids <id>
aws ec2 wait instance-running --instance-ids <id>
```

Instead of hand-rolling:

```bash
sleep 30; sleep 30; sleep 30   # guessing how long is "enough"
```

`aws ec2 wait <condition>` polls on your behalf and returns exactly when
the real condition is met — the general pattern for scripting against
any asynchronous cloud API:

```text
Start asynchronous operation
        │
        ▼
   aws ec2 wait ...
        │
        ▼
Continue only once the condition is genuinely satisfied
```

### 2.6 `modify-instance-attribute` — changing one property without touching others

```bash
aws ec2 modify-instance-attribute \
  --instance-id <id> \
  --instance-type '{"Value":"t2.nano"}'
```

This changes exactly the instance-type attribute, leaving the AMI,
network config, security groups, tags, and EBS volumes completely
untouched — the smallest possible change that satisfies the requirement,
rather than re-launching a new instance from scratch (which would also
change the instance ID, breaking anything referencing it).

### 2.7 The compressed reasoning chain

```text
Requirement (devops-ec2: t2.micro → t2.nano, ending in running state)
   → Find the instance by Name tag                 → describe-instances --filters
   → Check status checks BEFORE touching anything    → describe-instance-status
   → (if Initializing) wait for it                   → aws ec2 wait instance-status-ok
   → Recognize: instance type change requires a stop  → not an in-place edit
   → Stop                                             → stop-instances
   → Wait for the stop to actually complete            → wait instance-stopped
   → Change the type                                  → modify-instance-attribute
   → Start                                            → start-instances
   → Wait for it to actually be running                → wait instance-running
   → Verify: describe-instances                        → State=running, Type=t2.nano
   → Verify: describe-instance-status                  → InstanceStatus=ok, SystemStatus=ok
```

---

## 3. Concepts (reference)

### 3.1 Instance state vs. status checks
State (`pending`/`running`/`stopping`/`stopped`) is the instance's
position in the EC2 lifecycle. Status checks (`SystemStatus`,
`InstanceStatus`) are separate, ongoing health probes AWS runs against
the underlying hardware and the instance itself. A `running` instance can
still have failing or still-initializing status checks — don't treat
"state says running" as "fully healthy and ready."

### 3.2 Why instance-type changes require a stop
Instance type determines the physical hardware class the instance runs
on. Changing it means AWS must move the instance onto different
underlying hardware — something that can't happen while the instance is
actively running on the old hardware. This is fundamentally different
from, say, adding a tag or a security group, which are metadata/network
changes that never require touching the underlying compute allocation.

### 3.3 `aws ec2 wait`
A family of subcommands (`instance-running`, `instance-stopped`,
`instance-status-ok`, and others) that block the CLI invocation until the
named condition becomes true, polling internally rather than requiring
the caller to guess a sleep duration. The general-purpose antidote to
"is it done yet" guesswork in any script driving AWS's asynchronous
APIs.

### 3.4 Side effects of a stop/start cycle
- **EBS-backed root/data volumes** — persist through stop/start; data is
  not lost.
- **Public IPv4 (auto-assigned)** — can change after a stop/start, since
  it's allocated fresh on start unless the instance uses an Elastic IP.
- **Instance-store (ephemeral) volumes** — can be lost on stop, since
  that storage is physically tied to the original underlying host.

Before doing this in anything beyond a disposable lab, worth explicitly
asking: can this workload tolerate the downtime, does anything hardcode
the current public IP, and is any data living only on instance-store
volumes?

### 3.5 `describe-instances` vs. `describe-instance-status`
- `describe-instances` — configuration and lifecycle state (type, AMI,
  tags, state, network details).
- `describe-instance-status` — the two AWS-run health checks
  (`SystemStatus`, `InstanceStatus`) plus the current state, focused
  specifically on instance health rather than configuration.

Use the first to confirm *what* the instance is configured as; the
second to confirm AWS considers it *healthy*.

---

## 4. Runbook

### 4.1 Find the instance by its Name tag
```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-ec2" \
  --query 'Reservations[].Instances[].{ID:InstanceId,Name:Tags[?Key==`Name`].Value|[0],State:State.Name,Type:InstanceType,AZ:Placement.AvailabilityZone}' \
  --output table
```
```text
Name  : devops-ec2
ID    : i-0d72e642c3b3837f8
State : running
Type  : t2.micro
AZ    : us-east-1a
```

### 4.2 Check status checks before touching anything
```bash
aws ec2 describe-instance-status \
  --instance-ids i-0d72e642c3b3837f8 \
  --query 'InstanceStatuses[0].{Instance:InstanceState.Name,InstanceStatus:InstanceStatus.Status,SystemStatus:SystemStatus.Status}' \
  --output table
```
If either status shows `Initializing`, wait for both to reach `ok`
before proceeding:
```bash
aws ec2 wait instance-status-ok --instance-ids i-0d72e642c3b3837f8
```

### 4.3 Stop the instance
```bash
aws ec2 stop-instances --instance-ids i-0d72e642c3b3837f8
aws ec2 wait instance-stopped --instance-ids i-0d72e642c3b3837f8
```

### 4.4 Change the instance type
```bash
aws ec2 modify-instance-attribute \
  --instance-id i-0d72e642c3b3837f8 \
  --instance-type '{"Value":"t2.nano"}'
```

### 4.5 Start the instance back up
```bash
aws ec2 start-instances --instance-ids i-0d72e642c3b3837f8
aws ec2 wait instance-running --instance-ids i-0d72e642c3b3837f8
```

### 4.6 Verify — configuration
```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-ec2" \
  --query 'Reservations[].Instances[].{ID:InstanceId,Name:Tags[?Key==`Name`].Value|[0],State:State.Name,Type:InstanceType,AZ:Placement.AvailabilityZone}' \
  --output table
```
```text
----------------------------------
|        DescribeInstances       |
+--------+-----------------------+
|  AZ    |  us-east-1a           |
|  ID    |  i-0d72e642c3b3837f8  |
|  Name  |  devops-ec2           |
|  State |  running              |
|  Type  |  t2.nano              |
+--------+-----------------------+
```

### 4.7 Verify — health
```bash
aws ec2 describe-instance-status \
  --instance-ids i-0d72e642c3b3837f8 \
  --query 'InstanceStatuses[0].{Instance:InstanceState.Name,InstanceStatus:InstanceStatus.Status,SystemStatus:SystemStatus.Status}' \
  --output table
```
```text
Instance        running
InstanceStatus  ok
SystemStatus    ok
```

### 4.8 Click "Check" in the lab UI to validate.

---

## 5. Final state

| Requirement | Result |
|---|---|
| `devops-ec2` located by Name tag | ✅ `i-0d72e642c3b3837f8` |
| Status checks confirmed before modifying | ✅ |
| Instance type `t2.micro` → `t2.nano` | ✅ |
| Instance state after change | ✅ `running` |
| Status checks after change | ✅ `ok` / `ok` |

```text
Instance : devops-ec2 (i-0d72e642c3b3837f8)
Before   : t2.micro, running
                │
                ▼
              stop → wait
                │
                ▼
        modify-instance-attribute
                │
                ▼
             start → wait
                │
                ▼
After    : t2.nano, running, status checks ok
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `An error occurred (IncorrectInstanceState)` on `modify-instance-attribute` | Instance is still `running` (or `stopping`) — type changes require a fully `stopped` instance | `stop-instances`, then `wait instance-stopped`, before modifying |
| Script proceeds to modify the instance while status checks still show `Initializing` | Checked instance *state* (`running`) but never checked *status checks* separately | `describe-instance-status`, and `wait instance-status-ok` if not yet `ok` |
| Automation "hangs" or fails intermittently right after `stop-instances`/`start-instances` | Treated the API call's acknowledgment as if the transition were already complete | Use `aws ec2 wait instance-stopped` / `instance-running` instead of a fixed `sleep` |
| Public IP changed unexpectedly after the stop/start cycle | Auto-assigned public IPv4 is not guaranteed stable across a stop/start | Use an Elastic IP if a stable public address is required |
| `describe-instances` shows the new type, but the app on the instance seems unreachable/broken | Status checks passing only confirms AWS's infrastructure layer, not application health | Check the application separately — status checks are not an app-health proxy (§2.4) |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.7) out loud
      from "devops-ec2 needs t2.micro → t2.nano" to "verified running with
      status checks ok."
- [ ] Explain, in one sentence, why an instance type change requires the
      instance to be stopped first, unlike a tag or security-group change.
- [ ] Explain the difference between instance *state* and instance
      *status checks*, and why the task explicitly calls out the latter.
- [ ] Explain what `aws ec2 wait instance-stopped` does differently from
      just calling `stop-instances` and immediately continuing.
- [ ] Name two things that can change or be lost across a stop/start
      cycle, and how you'd mitigate each in a real (non-lab) environment.
- [ ] Repeat the full CLI sequence from memory on a fresh instance,
      verifying both `describe-instances` and `describe-instance-status`
      afterward — not just one or the other.

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
in the order you'd actually discover it: "what resource, what does it live
inside, what state must it be in, what existing state constrains my
choice, what CLI service+operation performs the read, what performs the
write, how do I verify." End with a compressed step-chain.>

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
