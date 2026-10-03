# Day 14 — Terminate an EC2 Instance

A KodeKloud "100 Cloud/AWS" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

---

## 1. Scenario

Terminate the EC2 instance specified by the lab and verify its final
state is `terminated`, in `us-east-1`.

```text
pending → running → stopping → stopped
                                    │
                                    ▼
                               terminated   ← also reachable directly
                                               from running/stopped
```

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 Stop vs. terminate — recap, now applied to the irreversible side

This is the direct counterpart to every stop-protection/termination-
protection lab earlier in this series (Days 8 and 9): stopping powers an
instance down while the resource itself persists and can be started
again; terminating destroys the instance resource permanently.

```text
STOP       → instance exists, powered off → can start again
TERMINATE  → instance resource destroyed    → cannot be restarted
```

Day 9's lab was about *preventing* this exact action
(`DisableApiTermination = true`). Today's task does the opposite —
actually performing the termination — which is precisely why
understanding that earlier protection mechanism matters here: if
`DisableApiTermination` were still `true` on this instance, the
terminate call would be rejected, and that's the first thing worth
checking if termination unexpectedly fails (§2.4).

### 2.2 Find the instance by tag, resolve to the real ID

```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=<INSTANCE_NAME>" \
  --query 'Reservations[].Instances[].[InstanceId,State.Name]' \
  --output table
```

Same tag-lookup pattern used for every EC2/EBS/ENI/EIP/AMI resource
throughout this series — resolve the human-assigned `Name` tag into the
actual `InstanceId`, since that's what every subsequent API call
actually needs, not the name.

### 2.3 Terminating via the console vs. the CLI — same underlying API call

```text
Console: EC2 → Instances → select instance → Instance state → Terminate instance → Confirm
CLI:     aws ec2 terminate-instances --instance-ids <id>
```

Both paths invoke the same underlying `TerminateInstances` API — the
console is just a UI wrapper. Worth knowing the CLI equivalent even if
you perform the action through the console, since the CLI is what you'd
actually use for verification either way (§2.5).

### 2.4 Termination is a one-way door — and AWS actively guards it in two ways

```text
1. Instance-level: DisableApiTermination (Day 9)
2. Console/CLI: an explicit confirmation step before the action proceeds
```

Unlike `stop`/`start`, which are freely reversible, `terminate` is
AWS's one deliberately-hard-to-trigger-by-accident EC2 action — both the
console's confirmation dialog and the optional `DisableApiTermination`
attribute exist specifically to prevent a single misclick from
destroying a resource permanently. If a `terminate-instances` call is
unexpectedly rejected, checking `DisableApiTermination` (via
`describe-instance-attribute`, Day 9's lab) is the first diagnostic
step, not assuming the CLI command itself is wrong.

### 2.5 The lifecycle has intermediate states — verify the *final* one, not the first response

```text
terminate-instances called
        │
        ▼
   State: shutting-down     ← intermediate, NOT the final state
        │
        ▼
   State: terminated          ← what the task actually requires
```

Same "response accepted ≠ resource in its final state" pattern running
through this entire series (EBS creation in Day 5, stop/start in Day 7,
AMI creation in Day 13) — a `terminate-instances` call returning
`shutting-down` is expected and correct mid-flight, but it is **not**
the state the task's acceptance criterion checks for. The verification
step has to re-query and confirm `terminated` specifically.

### 2.6 Verify with `describe-instances`, checking exactly the one field that matters

```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --instance-ids <INSTANCE_ID> \
  --query 'Reservations[0].Instances[0].[InstanceId,State.Name]' \
  --output table
```

A terminated instance still shows up in `describe-instances` output for
some time afterward (AWS retains the record briefly rather than
immediately erasing all trace of it) — don't mistake "the instance is
still visible" for "termination didn't work." The only field that
actually answers the question is `State.Name`, and the only value that
satisfies the task is `terminated`, not `shutting-down`, `stopped`, or
anything else.

### 2.7 The compressed reasoning chain

```text
Requirement (terminate the specified instance, verify State: terminated)
   → Find the instance by Name tag             → resolve to InstanceId
   → (if termination is ever rejected)          → check DisableApiTermination (Day 9) first
   → Terminate via console or CLI terminate-instances
   → Immediate response may show shutting-down   → intermediate, not final
   → Re-run describe-instances                    → confirm State.Name == terminated
   → Confirm region was us-east-1 throughout       → AWS resources are region-scoped
```

---

## 3. Concepts (reference)

### 3.1 The full EC2 instance lifecycle
```text
pending → running → stopping → stopped → (back to pending/running via start)
                                    │
running/stopped ────────────────────┴──→ shutting-down → terminated
```
`terminated` is a terminal state reachable from either `running` or
`stopped` — there is no path back out of it.

### 3.2 Why `DisableApiTermination` exists (recap from Day 9)
A per-instance attribute that, when `true`, causes AWS to reject
`terminate-instances` calls against that instance — a deliberate,
reversible safeguard distinct from the console's one-time confirmation
dialog, meant to protect specific instances from termination even by
automation that would otherwise succeed.

### 3.3 Stop vs. terminate — resource persistence
Stopping preserves the instance (and, by default, its root EBS volume)
for a future restart. Terminating destroys the instance resource
itself; by default the root volume is also deleted on termination
(governed by its own `DeleteOnTermination` attribute), though
additional attached volumes may or may not be, depending on their own
configuration.

### 3.4 Intermediate vs. final lifecycle states
`shutting-down` is the expected state immediately after a
`terminate-instances` call succeeds — it is not evidence of failure, and
it is not the state the task's verification should check for.
`terminated` is the actual terminal, final state.

### 3.5 A terminated instance remains briefly queryable
AWS does not instantly scrub a terminated instance from
`describe-instances` output — it typically remains visible (with
`State.Name: terminated`) for a period afterward. Seeing the instance ID
still appear in a query is not itself informative; only the `State.Name`
value tells you anything.

---

## 4. Runbook

### 4.1 Find the instance
```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=<INSTANCE_NAME>" \
  --query 'Reservations[].Instances[].[InstanceId,State.Name]' \
  --output table
```
```text
i-xxxxxxxxxxxxxxxxx    running
```

### 4.2 Terminate — Console path
```text
EC2 → Instances → select the instance
    → Instance state → Terminate instance → confirm
```

### 4.2b Terminate — CLI equivalent
```bash
aws ec2 terminate-instances \
  --region us-east-1 \
  --instance-ids i-xxxxxxxxxxxxxxxxx
```
```json
{
    "TerminatingInstances": [
        {
            "InstanceId": "i-xxxxxxxxxxxxxxxxx",
            "CurrentState": { "Name": "shutting-down" },
            "PreviousState": { "Name": "running" }
        }
    ]
}
```
`shutting-down` here is expected and intermediate (§2.5) — proceed to
verify the final state.

### 4.3 Verify the final state
```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --instance-ids i-xxxxxxxxxxxxxxxxx \
  --query 'Reservations[0].Instances[0].[InstanceId,State.Name]' \
  --output table
```
```text
-----------------------------------
|       DescribeInstances         |
+----------------------+----------+
| i-xxxxxxxxxxxxxxxx   | terminated|
+----------------------+----------+
```

### 4.4 Click "Check" in the lab UI to validate.

---

## 5. Final state

| Requirement | Result |
|---|---|
| Correct region (`us-east-1`) | ✅ |
| Correct instance identified by Name tag | ✅ |
| Termination issued | ✅ |
| Final verified state: `terminated` | ✅ |

```text
Instance (running/stopped)
        │
        │ terminate-instances / console Terminate
        ▼
   shutting-down   (intermediate — don't stop verifying here)
        │
        ▼
    terminated     (final, irreversible)
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `terminate-instances` rejected | `DisableApiTermination` is `true` on this instance (Day 9's protection) | `describe-instance-attribute --attribute disableApiTermination`; if confirmed, disable it first with `--no-disable-api-termination`, then retry |
| `State.Name` shows `shutting-down`, task check still fails | This is an intermediate state, not the final one | Wait briefly and re-run `describe-instances`; the required value is specifically `terminated` |
| Instance still appears in `describe-instances` output after termination | Expected — AWS retains terminated instance records visibly for a period | Check `State.Name` specifically; the instance's continued visibility is not itself a problem |
| Wrong instance terminated | Resolved the wrong `InstanceId` from an ambiguous or incorrect `Name` tag filter | Always re-confirm the `InstanceId` via `describe-instances --filters` before terminating; Name tags aren't guaranteed unique |
| Resource not found when verifying | Wrong region specified | Confirm `--region us-east-1` explicitly on every call |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.7) out loud
      from "terminate this instance" to "verified State.Name:
      terminated."
- [ ] Explain, in one sentence, the difference between `stopped` and
      `terminated`, and why only one of them is reversible.
- [ ] Explain why `shutting-down` appearing in the immediate response
      isn't sufficient proof the task is complete.
- [ ] Explain how Day 9's `DisableApiTermination` attribute could
      cause this exact task to fail, and how you'd diagnose that if it
      happened.
- [ ] Explain why an instance still showing up in `describe-instances`
      after termination isn't itself evidence something went wrong.
- [ ] Repeat the sequence on a different instance from memory, using
      the CLI path end to end (find → terminate → verify
      `State.Name == terminated`) without touching the console.

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
in the order you'd actually discover it: "what resource, what existing
protections/attributes could block this, what CLI service+operation
performs the read, what performs the write, what intermediate states are
expected vs. final, how do I verify the FINAL state specifically." End
with a compressed step-chain.>

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
