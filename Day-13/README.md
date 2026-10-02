# Day 13 — Create an AMI from an EC2 Instance

A KodeKloud "100 Cloud/AWS" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

---

## 1. Scenario

Create an **Amazon Machine Image (AMI)** from an existing EC2 instance:

```text
Source instance: devops-ec2 (i-0e7eaf3f2b7f9fa6e)
AMI name:        devops-ec2-ami
Required state:  available
```

Desired final state:

```text
AMI ID:  ami-025ed55ea9499ddb6
Name:    devops-ec2-ami
State:   available
```

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 An AMI is a template, not a copy of a running instance

```text
                 devops-ec2
                     │
               Create AMI
                     │
                     ▼
             ┌──────────────┐
             │ EBS Snapshot │
             └──────┬───────┘
                     │
                     ▼
                    AMI
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
         EC2-1      EC2-2      EC2-3
```

Creating an AMI doesn't "clone the running server" the way a VM
snapshot tool might — AWS takes a point-in-time snapshot of the
instance's EBS volume(s) and packages that together with launch
metadata (root device, block-device mappings) into a reusable image
definition. The AMI is the *recipe*; each future instance launched from
it is a fresh instance built from that recipe, not a live copy of
`devops-ec2` itself.

### 2.2 AMI vs. EBS snapshot — related, not interchangeable

```text
EBS Volume
    │
    ▼
Snapshot        →  a storage-level, point-in-time backup of ONE volume

AMI
 │
 ├── Image metadata
 ├── Root-device information
 ├── Block-device mappings
 └── EBS snapshot(s)      →  an EC2-level LAUNCH TEMPLATE that references
                              one or more snapshots
```

A snapshot alone can't be launched as an instance — it has no concept of
instance type compatibility, root device naming, or launch metadata. An
AMI is the layer above snapshots that makes "launch a new EC2 instance
from this" meaningful. Confusing the two matters in practice: deleting a
snapshot an AMI still references, or deregistering an AMI while assuming
its snapshots vanish too, are two different, independently-tracked
operations (§3.3).

### 2.3 Why AMIs exist — amortizing manual setup across many instances

```text
Configured EC2 (manually set up once)
        │
        ▼
       AMI
        │
   ┌────┼────┐
   ▼    ▼    ▼
  EC2  EC2  EC2    (each launched already matching the baseline)
```

Doing OS setup, package installs, and application deployment by hand
once per server doesn't scale — baking that configuration into an AMI
lets every future instance start from an identical, already-correct
baseline. This is the foundation Auto Scaling Groups, golden-image
patterns, and immutable-infrastructure deployments are all built on.

### 2.4 The reboot option — a filesystem-consistency tradeoff

```text
Running application
       │
       ▼
Filesystem writes in progress
       │
       ▼
   Reboot              ← brings the filesystem to a quiescent state
       │
       ▼
Snapshot taken at rest  ← consistent point-in-time capture
```

If the snapshot is taken while the application is actively writing,
the resulting image can capture an inconsistent, torn mid-write state.
Rebooting first flushes writes and quiesces the filesystem before the
snapshot happens — the AWS console's "Reboot instance" checkbox is
offering exactly this tradeoff: brief downtime in exchange for a
guaranteed-consistent image. Leaving it enabled (the default, and what
this lab kept) is the safer choice unless you have a specific reason
the instance absolutely cannot reboot.

### 2.5 `pending` is an expected, intermediate AMI state — not a failure

```text
Create image
     │
     ▼
  pending    ← AWS is still processing the snapshot + image metadata
     │
     ▼
 available   ← AMI is now launch-ready
```

Same "response accepted ≠ resource ready" pattern as every mutating AWS
call throughout this series (EBS volume creation in Day 5, EC2
stop/start in Day 7, ENI/EIP attachment in Days 10–11) — seeing
`pending` immediately after `Create image` is the expected first
observation, not a sign anything went wrong. The task's actual
requirement is reaching `available`, which means waiting and
re-checking, not treating the first query's result as final.

### 2.6 Finding and verifying the AMI via the CLI

```bash
aws ec2 describe-images \
  --image-ids ami-025ed55ea9499ddb6 \
  --query 'Images[0].[ImageId,Name,State,StateReason]' \
  --output table
```

```text
aws ec2 describe-images   →  the read operation for AMI metadata
--image-ids <id>           →  target exactly this AMI, not every AMI you own
--query '...'               →  project down to just the fields that matter
--output table              →  render as a human-readable table
```

Same `--image-ids`-for-exact-lookup vs. `--filters`-for-search pattern
established across this series (Day 8's `--volume-ids` vs. `--filters`,
Day 11's ENI lookup) — here, once you already have the AMI ID from the
console or from `Create image`'s response, an exact ID lookup is more
direct than re-searching by name.

### 2.7 `StateReason` — the field that explains a non-`available` state

```bash
aws ec2 describe-images \
  --image-ids ami-025ed55ea9499ddb6 \
  --query 'Images[0].[State,StateReason]' \
  --output table
```

If an AMI ever lands in `failed` instead of progressing to `available`,
`StateReason` is the field that actually explains *why* — checking only
`State` and seeing something other than `available` tells you *that*
something's wrong, but not *what*. This is the AMI-creation equivalent
of reading a full error message instead of just noting that a command
failed.

### 2.8 The compressed reasoning chain

```text
Requirement (AMI "devops-ec2-ami" from devops-ec2, State=available)
   → Confirm region/account context before anything else
   → Select devops-ec2 → Actions → Image and templates → Create image
   → Set Image name: devops-ec2-ami; keep storage config as-is
   → Leave "Reboot instance" enabled (filesystem consistency, §2.4)
   → Create image → AMI ID returned, initial State = pending
   → describe-images --image-ids <id>                → confirm State: pending (expected, not a failure)
   → Re-check after AWS finishes processing            → State: available
   → (if ever not available) check StateReason           → diagnose the actual cause
   → Confirm final: ImageId, Name, State all match requirement
```

---

## 3. Concepts (reference)

### 3.1 AMI structure
An AMI bundles together: a root-volume (and optionally additional
volume) EBS snapshot, block-device mapping metadata, and launch-relevant
configuration (architecture, virtualization type, root device name).
Launching an instance from an AMI recreates volumes from its referenced
snapshots and boots from them.

### 3.2 Why region matters for AMIs specifically
AMIs are region-scoped resources, same as EC2 instances and EBS volumes
— an AMI created in `us-east-1` simply does not exist in `us-west-2`
until explicitly copied there (`aws ec2 copy-image`). This is the same
recurring "confirm region before assuming a resource exists" discipline
from every AWS lab in this series.

### 3.3 Deregistering an AMI vs. deleting its snapshots
Deregistering an AMI removes the image definition itself, but does
**not** automatically delete the underlying EBS snapshot(s) it
referenced — these are two separate operations with two separate
cleanup steps. A snapshot could still be referenced by other AMIs or
retained deliberately for recovery; deleting it casually alongside a
deregister can be an unrecoverable mistake if that assumption is wrong.

### 3.4 AMI in Auto Scaling architectures
An Auto Scaling Group's launch template/configuration typically
references a specific AMI — when demand increases and new instances are
launched, every new instance starts from that same known-good baseline,
which is the practical payoff of investing in a correct, tested AMI
rather than configuring each instance individually after boot.

### 3.5 AMI is not a complete backup strategy
An AMI captures a point-in-time image suitable for *launching new
instances*, but real production backup strategy additionally considers
retention policies, RPO/RTO targets, database-specific backup
mechanisms, and cross-region redundancy — an AMI is one tool among
several, not a substitute for a full backup/DR plan.

---

## 4. Runbook

### 4.1 Confirm region and locate the source instance
```text
AWS Console → EC2 → Instances → confirm devops-ec2 is visible in the
current region
```

### 4.2 Create the image
```text
Select devops-ec2
  → Actions → Image and templates → Create image
  → Image name: devops-ec2-ami
  → Storage: keep existing configuration (gp3, unchanged)
  → Reboot instance: leave ENABLED (§2.4)
  → Create image
```

### 4.3 Capture the returned AMI ID
```text
ami-025ed55ea9499ddb6
devops-ec2-ami
pending
```

### 4.4 Check initial state via CLI
```bash
aws ec2 describe-images \
  --image-ids ami-025ed55ea9499ddb6 \
  --query 'Images[0].[ImageId,Name,State,StateReason]' \
  --output table
```
```text
ami-025ed55ea9499ddb6
devops-ec2-ami
pending
None
```
Expected — `pending` is the normal intermediate state (§2.5).

### 4.5 Re-check until `available`
```bash
aws ec2 describe-images \
  --image-ids ami-025ed55ea9499ddb6 \
  --query 'Images[0].[ImageId,Name,State,StateReason]' \
  --output table
```
```text
ami-025ed55ea9499ddb6
devops-ec2-ami
available
None
```

### 4.6 Confirm in the console
```text
EC2 → Images → AMIs → devops-ec2-ami → State: available
```

### 4.7 (Optional) Search by name instead of ID
```bash
aws ec2 describe-images \
  --owners self \
  --filters "Name=name,Values=devops-ec2-ami" \
  --query 'Images[*].[ImageId,Name,State,CreationDate]' \
  --output table
```

### 4.8 Click "Check" in the lab UI to validate.

---

## 5. Final state

| Requirement | Result |
|---|---|
| AMI created from `devops-ec2` | ✅ |
| Name: `devops-ec2-ami` | ✅ |
| State: `available` | ✅ |
| `StateReason`: none (no failure) | ✅ |

```text
devops-ec2 (i-0e7eaf3f2b7f9fa6e)
        │
        │ Create image (reboot enabled)
        ▼
   EBS Snapshot
        │
        ▼
ami-025ed55ea9499ddb6
devops-ec2-ami
        │
        ▼
     available
        │
        ▼
ready to launch new instances from
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| AMI not visible in console/CLI | Wrong AWS region selected | Confirm `--region`/console region matches where `devops-ec2` actually lives |
| `State: pending` immediately after `Create image` | Normal intermediate state — AWS hasn't finished processing yet | Wait and re-run `describe-images`; don't treat this as a failure (§2.5) |
| `State: failed` | An actual error occurred during snapshot/image creation | Check `StateReason` specifically — it names the cause `State` alone doesn't explain (§2.7) |
| Launched an instance from the AMI and it's missing recent data | The source instance had writes in-flight when the snapshot was taken (reboot disabled, or data written after image creation) | Keep "Reboot instance" enabled for consistency; re-create the AMI if data changed afterward |
| Deregistered an AMI expecting its snapshot to also disappear | Deregistering an AMI and deleting its snapshot are separate operations | Explicitly delete the snapshot afterward if it's genuinely no longer needed by anything else |
| AMI doesn't appear in a different region | AMIs are region-scoped; nothing copies them automatically | Use `aws ec2 copy-image` to explicitly copy it to another region if needed |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.8) out loud
      from "create an AMI named devops-ec2-ami from devops-ec2" to
      "verified State: available."
- [ ] Explain, in one sentence, the difference between an AMI and an EBS
      snapshot.
- [ ] Explain why the "Reboot instance" option exists, and what
      specifically it protects against.
- [ ] Explain why seeing `State: pending` right after creating an image
      isn't cause for concern.
- [ ] Explain why deregistering an AMI doesn't automatically delete its
      underlying snapshot, and why that matters operationally.
- [ ] Launch an instance from this AMI (conceptually, without
      necessarily doing it) and describe, step by step, what AWS does
      from AMI reference to running instance.

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
in the order you'd actually discover it: "what resource, what does it
represent conceptually, what intermediate states are expected vs. actual
failures, what CLI service+operation performs the read, what performs the
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
