# Day 8 — Enable Stop Protection for an EC2 Instance

A KodeKloud "100 Cloud/AWS" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

---

## 1. Scenario

As part of the migration, the team wants to protect an existing EC2
instance from being accidentally stopped. Task: enable **stop
protection** for the instance named `nautilus-ec2`, in `us-east-1`.

**Credentials/environment** (lab-specific, rotates per session):
retrieved via `showcreds` on the `aws-client` host, same as every prior
lab in this series.

Desired state:

```text
Instance : nautilus-ec2
Region   : us-east-1
Stop protection : Enabled
```

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 What "stop protection" actually is

EC2 exposes an instance attribute called `DisableApiStop`. When it's
`true`, AWS refuses stop requests made through the API/console against
that instance:

```text
Normal instance
    │
    └── stop-instances ──► instance stops

Stop protection enabled (DisableApiStop = true)
    │
    └── stop-instances ──► AWS rejects the request
```

The instance keeps running exactly as before — this is a guard on one
specific *operation* (stop), not a change to the instance's own runtime
behavior.

### 2.2 Stop protection ≠ termination protection — two independent switches

```text
                    EC2 instance
                         │
           ┌─────────────┴─────────────┐
           ▼                           ▼
     Stop protection            Termination protection
           │                           │
     DisableApiStop            DisableApiTermination
           │                           │
      blocks STOP                blocks TERMINATE
```

These guard two different lifecycle operations and are set
independently — enabling one says nothing about the other. This lab asks
specifically for **stop** protection; don't reach for the termination
attribute by pattern-matching the name.

### 2.3 Find the instance by tag, not a memorized ID

```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=nautilus-ec2" \
  --query 'Reservations[].Instances[].{ID:InstanceId,Name:Tags[?Key==`Name`].Value|[0],State:State.Name}' \
  --output table
```

Same `--filters Name=tag:Name,Values=...` pattern as Day 5's EBS volume
lookup and Day 7's `devops-ec2` lookup — search by the human-assigned
label, then act on the ID that search resolves to. This recurs constantly
because you rarely know a resource's generated ID ahead of time in real
work.

### 2.4 Enabling the attribute — the CLI flag maps directly onto the API attribute

```bash
aws ec2 modify-instance-attribute \
  --region us-east-1 \
  --instance-id <id> \
  --disable-api-stop
```

```text
--disable-api-stop   (CLI flag)
        │
        ▼
DisableApiStop = true   (the underlying EC2 attribute)
        │
        ▼
stop-instances requests against this instance are now rejected
```

The name reads oddly at first — `--disable-api-stop` doesn't mean
"disable [stop protection]," it means "**disable [the API's ability to
stop]** this instance." Read it as disabling the *stop operation itself*,
not disabling a protection feature — the protection is the *result* of
disabling the operation, not something separately named.

### 2.5 A successful `modify-instance-attribute` call is not yet verification

`modify-instance-attribute` returning with no error confirms AWS
*accepted* the request — it doesn't, by itself, prove the attribute is
now set the way you expect. This is the same "response received ≠
resource state confirmed" discipline as every mutating call in this
series (Day 5's EBS volume `"State": "creating"`, Day 7's EC2 stop/start)
— the only way to know the real state is to query it afterward, with the
right query.

### 2.6 Why `describe-instances` is the wrong tool to verify this specific attribute

A first instinct might be to check the new state the same way you found
the instance:

```bash
aws ec2 describe-instances --query '...DisableApiStop...'
```

`describe-instances` is built to answer broad questions about an
instance's general shape — state, type, AMI, subnet, tags. Certain
narrower, less commonly-needed attributes (like `DisableApiStop`,
`DisableApiTermination`, `SourceDestCheck`,
`InstanceInitiatedShutdownBehavior`) aren't reliably surfaced through its
general query shape and can appear as `None`/empty even when correctly
set — this is a case where the query returning an unexpected value is a
signal to question the *verification method*, not to assume the change
failed and revert it.

### 2.7 The correct, authoritative check: `describe-instance-attribute`

```bash
aws ec2 describe-instance-attribute \
  --region us-east-1 \
  --instance-id <id> \
  --attribute disableApiStop
```

```text
describe-instances            → "tell me broadly about this instance"
describe-instance-attribute   → "tell me the authoritative value of
                                  ONE specific named attribute"
```

This is the API that actually owns and directly reports the value of a
single named instance attribute — the general lesson worth keeping is
that AWS often has both a broad "describe everything about X" call and a
narrower "describe this one specific property of X precisely" call, and
the second is the one to trust when the first gives a confusing or
seemingly-wrong answer for a specific field.

### 2.8 The compressed reasoning chain

```text
Requirement (enable stop protection on nautilus-ec2)
   → Find the instance by Name tag                        → i-0f7a4aa85758e3780
   → Confirm this is STOP protection, not termination       → DisableApiStop, not DisableApiTermination
   → Enable it: modify-instance-attribute --disable-api-stop
   → First verification attempt: describe-instances          → shows "None" — misleading, not the right API
   → Recognize: query the API, not the change                → describe-instance-attribute --attribute disableApiStop
   → Re-verify                                                → {"DisableApiStop": {"Value": true}}
   → Confirmed: stop protection enabled, instance still running
```

---

## 3. Concepts (reference)

### 3.1 `DisableApiStop` vs. `DisableApiTermination`
Two independent instance attributes guarding two independent lifecycle
operations. Setting one to `true` has no effect on the other — an
instance can have stop protection on and termination protection off, or
vice versa, or both, or neither.

### 3.2 `describe-instances` vs. `describe-instance-attribute`
- `describe-instances` — broad, general-purpose resource description;
  good for state, type, tags, network placement.
- `describe-instance-attribute --attribute <name>` — narrow, precise
  lookup of exactly one named attribute's authoritative value; the
  correct tool when a specific attribute's value matters and the general
  call doesn't surface it reliably.

### 3.3 Why an accepted API call isn't proof of the resulting state
AWS's mutating APIs (create/modify/stop/start/etc.) generally return as
soon as the request is validated and queued/applied — not necessarily
after every downstream effect is fully reflected everywhere state can be
queried. Treat "no error returned" as "the request was accepted," and
always follow with a targeted read of the specific state you changed.

### 3.4 Operational risk of protection settings
Stop protection can block *legitimate* automation just as effectively as
accidental human action — a maintenance script calling
`stop-instances` against a protected instance will simply fail. Before
enabling protection settings in a real (non-lab) environment, it's worth
confirming: does any patching/maintenance workflow need to stop this
instance, and who has the access needed to disable protection when that's
genuinely required?

### 3.5 Tag-based resource lookup as a recurring AWS CLI pattern
`--filters "Name=tag:Name,Values=<value>"` against a `describe-*` call is
the general mechanism for "find the resource a human labeled this,"
independent of service — seen now across EBS volumes (Day 5), EC2
instances (Day 7), and again here.

---

## 4. Runbook

### 4.1 Find the instance by its Name tag
```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=nautilus-ec2" \
  --query 'Reservations[].Instances[].{ID:InstanceId,Name:Tags[?Key==`Name`].Value|[0],State:State.Name,StopProtection:DisableApiStop}' \
  --output table
```
```text
ID             i-0f7a4aa85758e3780
Name           nautilus-ec2
State          running
StopProtection None
```
Note `StopProtection: None` here — expected pre-change, and also not the
authoritative way to check this attribute even post-change (§2.6).

### 4.2 Enable stop protection
```bash
aws ec2 modify-instance-attribute \
  --region us-east-1 \
  --instance-id i-0f7a4aa85758e3780 \
  --disable-api-stop
```
No output on success — expected for this call; proceed to verify.

### 4.3 Verify using the authoritative attribute API
```bash
aws ec2 describe-instance-attribute \
  --region us-east-1 \
  --instance-id i-0f7a4aa85758e3780 \
  --attribute disableApiStop
```
```json
{
    "InstanceId": "i-0f7a4aa85758e3780",
    "DisableApiStop": {
        "Value": true
    }
}
```
`Value: true` confirms stop protection is genuinely enabled.

### 4.4 Confirm the instance is still running (protection ≠ a state change)
```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --instance-ids i-0f7a4aa85758e3780 \
  --query 'Reservations[0].Instances[0].State.Name' \
  --output text
```
```text
running
```

### 4.5 Click "Check" in the lab UI to validate.

---

## 5. Final state

| Requirement | Result |
|---|---|
| `nautilus-ec2` located | ✅ `i-0f7a4aa85758e3780` |
| `DisableApiStop` set to `true` | ✅ (confirmed via `describe-instance-attribute`) |
| Instance still `running` | ✅ |

```text
Instance        : nautilus-ec2
Instance ID     : i-0f7a4aa85758e3780
Region          : us-east-1
State           : running
Stop protection : ENABLED
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `describe-instances` shows `StopProtection: None` right after enabling it | Wrong API for verifying this specific attribute — `describe-instances` doesn't reliably surface it | Use `aws ec2 describe-instance-attribute --attribute disableApiStop` instead |
| `stop-instances` fails against this instance later | Stop protection is working as intended (that's the point) | If the stop is genuinely required, first `modify-instance-attribute` with `--no-disable-api-stop` to turn it off |
| Accidentally enabled termination protection instead of stop protection | Confused `DisableApiTermination` with `DisableApiStop` — similar names, different operations | Check with `describe-instance-attribute --attribute disableApiTermination` separately; each is set independently |
| Automation/maintenance script unexpectedly fails to stop an instance | Stop protection was enabled without checking whether any automation depends on being able to stop it | Confirm with stakeholders before enabling protection settings on instances with active automation |
| Unsure whether the `modify-instance-attribute` call actually took effect | No output from the command doesn't confirm anything by itself | Always follow with the read-back verification (§2.5, §4.3) |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.8) out loud
      from "enable stop protection on nautilus-ec2" to "verified via
      `describe-instance-attribute`."
- [ ] Explain, in one sentence, why `DisableApiStop = true` means stop
      protection is *enabled*, not disabled.
- [ ] Explain the difference between `DisableApiStop` and
      `DisableApiTermination`, and why enabling one says nothing about
      the other.
- [ ] Explain why `describe-instances` showing `None` for this attribute
      wasn't proof the change had failed.
- [ ] Explain what you'd need to do to reverse this change and allow the
      instance to be stopped again.
- [ ] Repeat the full sequence on a different instance from memory,
      verifying with `describe-instance-attribute` specifically, not
      `describe-instances`.

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
