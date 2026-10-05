# Day 15 — Create an EBS Volume Snapshot

A KodeKloud "100 Cloud/AWS" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

---

## 1. Scenario

Create an EBS snapshot of an existing volume, in `us-east-1`:

| Field | Value |
|---|---|
| Volume | `xfusion-vol` |
| Snapshot Name tag | `xfusion-vol-ss` |
| Description | `xfusion Snapshot` |
| Required final state | `completed` |

```text
EC2
 │
 └── EBS Volume
       xfusion-vol
          │
          ▼
      EBS Snapshot
      xfusion-vol-ss
      "xfusion Snapshot"
```

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 A snapshot is a separate, independently-existing resource — not a copy sitting "inside" the volume

```text
EBS Volume (xfusion-vol)
        │
        │ create-snapshot
        ▼
EBS Snapshot (xfusion-vol-ss)   ← its own resource, its own ID (snap-...),
                                   stored by AWS independently of the
                                   volume's own lifecycle
```

The volume can later be deleted, resized, or modified, and the snapshot
remains — it's a point-in-time capture, not a live reference back to the
volume. This is the same "AMI is a template built FROM an instance, not
a live copy OF it" distinction from Day 13 — a snapshot is the storage-
level analog, and in fact an AMI's creation (Day 13) is literally built
on top of exactly this snapshot mechanism underneath.

### 2.2 Snapshot Name tag vs. Description — two genuinely different fields

```text
Name tag     → xfusion-vol-ss     (a human-readable LABEL, like every
                                     other resource's Name tag in this series)
Description  → "xfusion Snapshot"  (free-text metadata ABOUT the snapshot,
                                     a separate field entirely)
```

These are set through different parts of the console (and different API
parameters — `--description` on `create-snapshot` vs. a separate
`create-tags` call for `Name`). Conflating them — e.g. putting
`xfusion-vol-ss` into the description field, or vice versa — is the most
likely way to fail this task's exact-match verification despite having
genuinely created a snapshot.

### 2.3 Find the volume by tag, same pattern as every prior EBS lab

```bash
aws ec2 describe-volumes \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=xfusion-vol" \
  --query 'Volumes[].{ID:VolumeId,State:State,AZ:AvailabilityZone}' \
  --output table
```

Same tag-lookup pattern from Day 5 (volume creation), Day 12 (volume
attachment) — resolve the human-assigned `Name` tag to the actual
`VolumeId` the `create-snapshot` call needs.

### 2.4 `create-snapshot` — description goes in at creation time; the Name tag is applied separately

```bash
aws ec2 create-snapshot \
  --region us-east-1 \
  --volume-id <vol-id> \
  --description "xfusion Snapshot" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=xfusion-vol-ss}]'
```

`--tag-specifications` at creation time is the direct equivalent of the
console's "add the Name tag after creation" step described in the
draft — doing it in one call avoids the two-step
create-then-separately-tag sequence, though both approaches produce the
identical final resource.

### 2.5 `pending` → `completed` — another intermediate state, same recurring lesson

```text
create-snapshot called
        │
        ▼
   State: pending     ← AWS is still copying/writing data to the snapshot
        │
        ▼
   State: completed    ← what this task's acceptance criterion requires
```

Identical shape to every "response accepted ≠ resource ready" case in
this series — EBS volume creation (Day 5), EC2 stop/start (Day 7), AMI
creation (Day 13). A snapshot can take meaningfully longer to reach
`completed` than a volume reaches `available`, since AWS is actually
copying the volume's data in the background — **don't submit the lab
while it's still `pending`**, exactly as the draft's own warning states.

### 2.6 Verify with `describe-snapshots`, checking the specific field the task cares about

```bash
aws ec2 describe-snapshots \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=xfusion-vol-ss" \
  --query 'Snapshots[].{ID:SnapshotId,State:State,Description:Description,VolumeId:VolumeId}' \
  --output table
```

Same "verify the exact field the requirement specifies, not just that
the resource exists" discipline as every prior lab — here, `State` must
read `completed`, and `Description` must read exactly `xfusion Snapshot`
(§2.2's distinction made concrete).

### 2.7 The compressed reasoning chain

```text
Requirement (snapshot xfusion-vol-ss, description "xfusion Snapshot", completed)
   → Find the volume by Name tag              → VolumeId
   → create-snapshot --volume-id ... --description "xfusion Snapshot"
        --tag-specifications Name=xfusion-vol-ss
   → Initial response shows State: pending       → intermediate, not final
   → Re-run describe-snapshots                    → wait for State: completed
   → Confirm Description field exactly matches     → separate from the Name tag
   → Confirm region was us-east-1 throughout        → snapshots are region-scoped like volumes
```

---

## 3. Concepts (reference)

### 3.1 EBS snapshot
A point-in-time, incremental backup of an EBS volume's data, stored by
AWS as an independent resource with its own ID (`snap-...`). The first
snapshot of a volume copies all data; subsequent snapshots of the same
volume only store the incremental changes, though each snapshot still
represents the complete volume state at the time it was taken.

### 3.2 Snapshot as the foundation under AMIs (recap from Day 13)
Day 13's AMI creation is built directly on this same snapshot mechanism
— creating an AMI from an instance snapshots its EBS volume(s) and
packages that snapshot together with launch metadata. Today's lab is
the lower-level primitive Day 13 was already using implicitly.

### 3.3 Name tag vs. Description — two separate metadata fields
A resource's `Name` tag is the conventional human-readable label used
for lookup/display (the same `Name` tag pattern used for every resource
in this series). `Description` is a distinct, free-text field some AWS
resources (snapshots, AMIs, security groups) also carry — setting one
does not set the other, and a task that specifies both expects both set
to their own exact values.

### 3.4 `pending` → `completed` for snapshots specifically
Snapshot creation can take longer than many other EBS operations to
leave `pending`, since AWS is physically copying the volume's data.
Treat the initial response exactly like every other "intermediate
state" case in this series — confirm the final state with a follow-up
`describe-snapshots` call before considering the task done.

### 3.5 Snapshots are region-scoped, like every other resource covered so far
Same recurring caveat as volumes, instances, AMIs, and ENIs — a
snapshot created in `us-east-1` doesn't exist in any other region
without an explicit copy operation.

---

## 4. Runbook

### 4.1 Find the volume
```bash
aws ec2 describe-volumes \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=xfusion-vol" \
  --query 'Volumes[].{ID:VolumeId,State:State,AZ:AvailabilityZone}' \
  --output table
```
```text
ID                      State       AZ
vol-xxxxxxxxxxxxxxxxx   in-use      us-east-1a
```

### 4.2 Create the snapshot
```bash
aws ec2 create-snapshot \
  --region us-east-1 \
  --volume-id vol-xxxxxxxxxxxxxxxxx \
  --description "xfusion Snapshot" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=xfusion-vol-ss}]'
```
```json
{
    "SnapshotId": "snap-xxxxxxxxxxxxxxxxx",
    "VolumeId": "vol-xxxxxxxxxxxxxxxxx",
    "State": "pending",
    "Description": "xfusion Snapshot"
}
```
`pending` here is expected — proceed to verify the final state (§2.5).

### 4.3 Wait for and verify the final state
```bash
aws ec2 describe-snapshots \
  --region us-east-1 \
  --snapshot-ids snap-xxxxxxxxxxxxxxxxx \
  --query 'Snapshots[0].{ID:SnapshotId,State:State,Description:Description}' \
  --output table
```
```text
ID                      State       Description
snap-xxxxxxxxxxxxxxxxx  completed   xfusion Snapshot
```

### 4.4 Confirm the Name tag
```bash
aws ec2 describe-snapshots \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=xfusion-vol-ss" \
  --query 'Snapshots[].{ID:SnapshotId,State:State,Description:Description}' \
  --output table
```

### 4.5 (Equivalent Console path, for reference)
```text
EC2 → Elastic Block Store → Volumes → xfusion-vol
    → Actions → Create snapshot → Description: "xfusion Snapshot" → Create snapshot
EC2 → Elastic Block Store → Snapshots → find the new snapshot
    → add tag: Key=Name, Value=xfusion-vol-ss
    → wait for Status: completed (do NOT submit while pending)
```

### 4.6 Click "Check" in the lab UI to validate.

---

## 5. Final state

| Requirement | Result |
|---|---|
| Snapshot created from `xfusion-vol` | ✅ |
| Name tag: `xfusion-vol-ss` | ✅ |
| Description: `xfusion Snapshot` (exact match) | ✅ |
| State: `completed` | ✅ |

```text
xfusion-vol (vol-...)
        │
        │ create-snapshot
        ▼
   pending  ──wait──►  completed
        │
        ▼
xfusion-vol-ss (snap-...)
Description: "xfusion Snapshot"
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Lab check fails despite a snapshot existing | `Name` tag and `Description` values swapped, or one of them left at a default/empty value | Re-check both fields independently — they're separate, both required exactly as specified (§2.2) |
| Submitted the lab while snapshot still shows `pending` | Didn't wait for the actual final state | Re-run `describe-snapshots` and wait for `State: completed` before submitting |
| Snapshot not found when searching by `Name` tag | Tag never actually applied (created via CLI without `--tag-specifications`, or console tag step skipped) | `describe-snapshots --filters "Name=tag:Name,Values=..."`; if empty, add the tag explicitly via `create-tags` |
| Resource not found / wrong volume snapshotted | Wrong region, or wrong volume resolved from an ambiguous Name tag | Confirm `us-east-1` explicitly; re-verify the exact `VolumeId` before calling `create-snapshot` |
| Snapshot taking a long time to leave `pending` | Expected — AWS is physically copying volume data, which can take longer than many other EBS operations | Wait; this isn't a failure, just the nature of snapshot creation (§3.4) |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.7) out loud
      from "create a snapshot of xfusion-vol" to "verified State:
      completed, with the correct Name tag and Description."
- [ ] Explain, in one sentence, the difference between a snapshot's
      `Name` tag and its `Description` field.
- [ ] Explain why a snapshot is a resource independent of the volume it
      was taken from, rather than something stored "inside" the volume.
- [ ] Explain the connection between this lab's snapshot mechanism and
      Day 13's AMI creation — what's the actual relationship?
- [ ] Explain why `pending` right after `create-snapshot` isn't
      evidence of failure.
- [ ] Repeat the full sequence on a different volume from memory,
      setting both the Name tag and Description correctly on the first
      attempt, and verify each field independently.

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
in the order you'd actually discover it: "what resource, what SEPARATE
metadata fields does the requirement actually specify, what existing
state constrains my choice, what CLI service+operation performs the read,
what performs the write, what intermediate states are expected vs. final,
how do I verify EACH field independently." End with a compressed
step-chain.>

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
