# Day 9 — Enable Termination Protection for an EC2 Instance

A KodeKloud "100 Cloud/AWS" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

This is the direct sibling of Day 8 (stop protection) — same shape of
problem, a different attribute entirely. Read them back to back and the
distinction should stop being confusing.

---

## 1. Scenario

The Nautilus DevOps team created an EC2 instance during the migration but
forgot to enable **termination protection**. Task: enable it for the
existing instance named `datacenter-ec2`, in `us-east-1`.

**Credentials/environment** (lab-specific, rotates per session):

| Field | Value |
|---|---|
| Console URL | `https://408499177606.signin.aws.amazon.com/console?region=us-east-1` |
| Username | `kk_labs_user_343541` |
| Region constraint | `us-east-1` only |
| Access | via `aws-client` host; run `showcreds` there to retrieve credentials |

Desired state:

```text
EC2 Instance
    │
    ├── Name: datacenter-ec2
    ├── Region: us-east-1
    └── Termination protection: ENABLED
```

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 Stop vs. terminate — two destinations, not two names for one thing

```text
                    EC2 Instance
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
             STOP                TERMINATE
              │                     │
       instance PAUSES        instance is DELETED
       (can be started        (the resource itself
        again)                 stops existing)
```

Stopping is reversible — the instance and its root volume (typically)
persist, ready to be started again. Terminating is destructive — the
instance resource is gone, and depending on each volume's
`DeleteOnTermination` setting, attached storage can be deleted along with
it. This lab protects against the *destructive* one.

### 2.2 Termination protection is a distinct EC2 attribute, not a variant of stop protection

```text
                    EC2 instance
                         │
             ┌───────────┴───────────┐
             ▼                       ▼
       Stop protection       Termination protection
             │                       │
      DisableApiStop        DisableApiTermination
             │                       │
        blocks STOP             blocks TERMINATE
             │                       │
        (Day 8's lab)           (today's lab)
```

These are independently-set attributes on the same instance —
`DisableApiStop = true` says nothing about `DisableApiTermination`, and
vice versa. Day 8 protected `nautilus-ec2` from being *stopped*; today
protects `datacenter-ec2` from being *terminated*. Conflating the two
attribute names is the single most likely mistake in this lab, precisely
because they read so similarly.

### 2.3 Reading `DisableApiTermination` correctly — same naming trap as Day 8

```text
DisableApiTermination = true
```

does **not** mean "disable [termination protection]." It means "disable
[the API's ability to terminate]" this instance:

```text
Normal instance
terminate request ──► EC2 instance ──► TERMINATED

Termination protection enabled (DisableApiTermination = true)
terminate request ──► EC2 instance ──► BLOCKED
```

Read the attribute name as *"disable [verb: terminate]"*, not *"disable
[protection]"* — the same parsing trap Day 8 covers for
`DisableApiStop`, and worth internalizing once so it stops tripping you
up on either attribute.

### 2.4 Find the instance by tag — the same recurring pattern

```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=datacenter-ec2" \
  --query 'Reservations[].Instances[].{ID:InstanceId,Name:Tags[?Key==`Name`].Value|[0],State:State.Name}' \
  --output table
```

```text
Human-readable name (datacenter-ec2)
        │
        ▼
--filters "Name=tag:Name,Values=..."
        │
        ▼
Instance ID (i-xxxxxxxxxxxxxxxxx)
        │
        ▼
Every subsequent API call operates on the ID, not the name
```

Fourth appearance of this exact pattern in the series (EBS volumes Day 5,
EC2 launch Day 6/instance-type change Day 7, stop protection Day 8) —
by now this should be closer to muscle memory than something you derive
from scratch each time.

### 2.5 Modify the attribute

```bash
aws ec2 modify-instance-attribute \
  --region us-east-1 \
  --instance-id <id> \
  --disable-api-termination
```

Same shape as Day 8's `--disable-api-stop`, targeting the sibling
attribute. As with any `modify-instance-attribute` call, a clean return
confirms AWS *accepted* the request — not that the attribute now holds
the value you expect (§2.6).

### 2.6 Verify with the authoritative attribute API, not the broad one

```bash
aws ec2 describe-instance-attribute \
  --region us-east-1 \
  --instance-id <id> \
  --attribute disableApiTermination
```

Exactly the lesson from Day 8: `describe-instances`'s general query shape
doesn't reliably surface this specific attribute — `describe-instance-
attribute` is the API that actually owns and precisely reports it:

```text
describe-instances            → "tell me broadly about this instance"
describe-instance-attribute   → "tell me the authoritative value of
                                  ONE specific named attribute"
```

Expected result:

```json
{
    "InstanceId": "i-...",
    "DisableApiTermination": {
        "Value": true
    }
}
```

### 2.7 What enabling protection does *not* do

```text
Enable termination protection
       │
       ├── does NOT stop the instance
       ├── does NOT restart the instance
       ├── does NOT change instance type
       └── DOES prevent a terminate-instances call from succeeding
```

Confirming the instance is still `running` afterward isn't a formality —
it's proof the change was scoped to exactly the one operation the task
asked about, with no unintended side effect on the instance's runtime
state (the same "verify the *whole* picture, not just the one field you
changed" habit as Day 7's post-type-change check).

### 2.8 The compressed reasoning chain

```text
Requirement (enable termination protection on datacenter-ec2)
   → Identify the operation to protect             → TERMINATE, not STOP
   → Identify the controlling attribute             → DisableApiTermination, not DisableApiStop
   → Find the instance by Name tag                  → describe-instances --filters
   → Enable it                                       → modify-instance-attribute --disable-api-termination
   → Verify with the AUTHORITATIVE API                → describe-instance-attribute --attribute disableApiTermination
   → Confirm Value == true
   → Confirm the instance is still running unaffected → describe-instances (state only)
```

---

## 3. Concepts (reference)

### 3.1 Stop vs. terminate — the underlying lifecycle distinction
Stopping shuts the instance down while preserving the resource (and,
typically, its root EBS volume) for a later restart. Terminating
irreversibly deletes the instance resource, and — depending on each
attached volume's `DeleteOnTermination` flag — can delete storage along
with it. Protection mechanisms exist for each because the *consequences*
of each operation are so different: one is a pause, the other can be data
loss.

### 3.2 `DisableApiStop` vs. `DisableApiTermination`
Two independent instance attributes, each guarding a different
lifecycle operation. Day 8 covers the first; today covers the second.
Setting one has zero effect on the other — an instance can have
termination protection on and stop protection off (as here), or any
other combination.

### 3.3 `describe-instances` vs. `describe-instance-attribute` (recap from Day 8)
- `describe-instances` — broad, general resource description; not
  reliable for this specific narrow attribute.
- `describe-instance-attribute --attribute <name>` — the precise,
  authoritative lookup for exactly one named attribute.

### 3.4 Reversing the change
```bash
aws ec2 modify-instance-attribute \
  --region us-east-1 \
  --instance-id <id> \
  --no-disable-api-termination
```
Sets `DisableApiTermination` back to `false`. **Do not run this in the
current lab** — the required end state is protection *enabled*. Included
here only because "how do I undo this" is a fair question once you
understand the attribute, and the answer is the `--no-` prefix on the
same flag, not a different attribute or a different command.

### 3.5 Tag-based discovery as a recurring, now-familiar pattern
`--filters "Name=tag:Name,Values=<value>"` against `describe-instances`
to resolve a human-assigned name into the ID every subsequent call
actually needs — the same mechanism used for EBS volumes (Day 5) and EC2
instances (Days 6–8).

---

## 4. Runbook

### 4.1 Find the instance
```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=datacenter-ec2" \
  --query 'Reservations[].Instances[].{ID:InstanceId,Name:Tags[?Key==`Name`].Value|[0],State:State.Name}' \
  --output table
```
```text
ID    : i-0123456789abcdef0
Name  : datacenter-ec2
State : running
```

### 4.2 Check current termination-protection state (before changing anything)
```bash
aws ec2 describe-instance-attribute \
  --region us-east-1 \
  --instance-id i-0123456789abcdef0 \
  --attribute disableApiTermination
```
```json
{
    "InstanceId": "i-0123456789abcdef0",
    "DisableApiTermination": {
        "Value": false
    }
}
```
Confirms the starting state: `Desired = true`, `Current = false`.

### 4.3 Enable termination protection

**Console path:**
```text
EC2 → Instances → datacenter-ec2 (select)
  → Actions → Instance settings → Change termination protection
  → Enable → Save
```

**CLI equivalent:**
```bash
aws ec2 modify-instance-attribute \
  --region us-east-1 \
  --instance-id i-0123456789abcdef0 \
  --disable-api-termination
```
No output on success — expected; proceed to verify.

### 4.4 Verify with the authoritative attribute API
```bash
aws ec2 describe-instance-attribute \
  --region us-east-1 \
  --instance-id i-0123456789abcdef0 \
  --attribute disableApiTermination
```
```json
{
    "InstanceId": "i-0123456789abcdef0",
    "DisableApiTermination": {
        "Value": true
    }
}
```

### 4.5 Confirm the instance is unaffected otherwise
```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --instance-ids i-0123456789abcdef0 \
  --query 'Reservations[0].Instances[0].State.Name' \
  --output text
```
```text
running
```

### 4.6 Click "Check" in the lab UI to validate.

---

## 5. Final state

| Requirement | Result |
|---|---|
| `datacenter-ec2` located | ✅ |
| Region `us-east-1` | ✅ |
| `DisableApiTermination` = `true` | ✅ (confirmed via `describe-instance-attribute`) |
| Instance still `running` | ✅ |

```text
EC2
 │
 └── datacenter-ec2
       │
       ├── Region: us-east-1
       ├── State: running
       └── DisableApiTermination: true
                              │
                              ▼
                   Termination protection
                         ENABLED
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `datacenter-ec2` not found | Wrong region selected | Confirm `us-east-1` explicitly, both in console and `--region` |
| Multiple instances returned for the same Name tag | Name tags aren't guaranteed unique | Inspect each returned ID/state individually before picking one |
| `describe-instances` doesn't clearly show the attribute | Wrong API for this specific narrow attribute | Use `describe-instance-attribute --attribute disableApiTermination` |
| Console shows "Enabled" but CLI shows `Value: false` | Checking the wrong account, region, or instance ID | Re-confirm with `aws sts get-caller-identity`, region, and the exact instance ID |
| Enabled stop protection instead of termination protection | Confused `DisableApiStop` with `DisableApiTermination` — similarly-named, different operations (§2.2) | Re-check with `describe-instance-attribute --attribute disableApiTermination` specifically |
| A later `terminate-instances` call is unexpectedly rejected | Termination protection is working exactly as intended | If termination is genuinely required, disable it first via `--no-disable-api-termination` (§3.4) |
| Accidentally disabled protection while testing | Ran the reversal command (§3.4) during this lab | Re-run `modify-instance-attribute --disable-api-termination` and re-verify |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.8) out loud
      from "enable termination protection on datacenter-ec2" to "verified
      via `describe-instance-attribute`."
- [ ] Explain, in one sentence, the difference between what stopping and
      terminating an instance each do to the underlying resource.
- [ ] Explain why `DisableApiTermination = true` means protection is
      *enabled*, not disabled — using the same "disable [the verb]"
      reading as Day 8's `DisableApiStop`.
- [ ] Explain why `describe-instances` is not the right tool to verify
      this attribute, and name the correct one.
- [ ] If the requirement instead said "prevent the instance from being
      *stopped*," which attribute would you use, and how confident are
      you that's not the one you just configured?
- [ ] Repeat the full sequence on a different instance from memory,
      verifying with `describe-instance-attribute` specifically — then
      explain exactly what CLI command would reverse it (without running
      it).

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
in the order you'd actually discover it: "what resource, what attribute,
what existing state constrains my choice, what CLI service+operation
performs the read, what performs the write, how do I verify — with the
RIGHT API, not just any describe call." End with a compressed step-chain.>

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
