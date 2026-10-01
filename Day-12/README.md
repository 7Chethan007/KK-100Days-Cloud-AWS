# Day 12 — Attach an EBS Volume to an EC2 Instance

A KodeKloud "100 Cloud/AWS" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

---

## 1. Scenario

Attach EBS volume `xfusion-volume` to EC2 instance `xfusion-ec2`, in
`us-east-1`, using device name `/dev/sdb`.

```text
                    AWS Region: us-east-1
                           │
                           ▼
                    Availability Zone
                       us-east-1a
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
        EC2 Instance               EBS Volume
       xfusion-ec2              xfusion-volume
       i-01fd...                vol-036a...
              │                         │
              └────────── attach ──────┘
                         │
                         ▼
                     /dev/sdb
```

The crucial boundary for this task: **attaching** a volume is not the
same thing as **mounting** a filesystem. This lab only requires the
AWS-level attachment — no `mkfs`, no `mount`, no `/etc/fstab`.

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 EBS is storage decoupled from compute

An EC2 instance has CPU, RAM, and network interfaces — but its block
storage doesn't have to live physically "inside" it:

```text
EC2
│
├── CPU
├── RAM
├── Network Interface
└── Storage  ──►  can be an independently-existing EBS volume
```

An EBS volume (**Elastic Block Store**) can exist with no instance
attached at all — exactly the lab's starting state: `xfusion-volume`
already exists, `xfusion-ec2` already exists, and nothing connects them
yet.

```text
Before attachment:

EC2                         EBS
┌──────────┐                ┌──────────┐
│ Instance │                │  Volume  │
└──────────┘                └──────────┘

After attachment:

EC2 ─────── attach ──────── EBS
```

### 2.2 Attach vs. mount — two entirely separate layers

This is the single most important distinction in the lab:

```text
AWS layer                          Linux layer
─────────                          ───────────
EBS Volume                         Block device
    │ attach                           │ format (mkfs)
    ▼                                  ▼
EC2 Instance                       Filesystem
    │                                  │ mount
    ▼                                  ▼
/dev/sdb (AWS attachment)          /data (a mounted directory)
```

The AWS-level "attach" establishes a device mapping the instance can
*see* — it says nothing about whether that block device has a
filesystem on it, or whether anything inside the OS has mounted it
anywhere. This lab stops at the first layer; `mkfs`/`mount`/`fstab`
would be a separate, later task (§2.8 walks through what that would
look like, for context, without doing it here).

### 2.3 `/dev/sdb` is an EC2 attachment name, not a guaranteed Linux device name

```text
AWS says: attach this volume as /dev/sdb
                    │
                    ▼
Linux MAY show: /dev/sdb   (older/Xen-based instances)
            OR: /dev/nvme1n1   (Nitro-based instances)
```

On modern Nitro-based EC2 instance types, the kernel commonly exposes
EBS volumes as NVMe devices (`/dev/nvme1n1`, etc.) regardless of the
name you specified at attach time — the AWS-level device name you pass
to `attach-volume`/the console is the **attachment identifier AWS
tracks**, not a promise about what `lsblk` will show inside the
instance. This lab's acceptance criterion is specifically about the EC2
attachment configuration (`Device: /dev/sdb` in `describe-volumes`), not
about what the Linux kernel happens to name it.

### 2.4 Availability Zone compatibility is a hard constraint, checked before anything else

```text
us-east-1
│
├── us-east-1a  →  EC2 (xfusion-ec2) + EBS (xfusion-volume)   ✅ compatible
├── us-east-1b  →  ...
└── us-east-1c  →  ...
```

A standard EBS volume is pinned to one AZ and can only attach to
instances in that same AZ — this is why the AWS console's "Attach
volume" screen only lists compatible instances. If an expected instance
doesn't show up there, AZ mismatch is the first thing worth checking,
not a sign the console is malfunctioning.

### 2.5 Find both resources by tag, resolve to real IDs

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=xfusion-ec2" \
  --query 'Reservations[].Instances[].{ID:InstanceId,AZ:Placement.AvailabilityZone,State:State.Name}' \
  --output table

aws ec2 describe-volumes \
  --filters "Name=tag:Name,Values=xfusion-volume" \
  --query 'Volumes[].{ID:VolumeId,AZ:AvailabilityZone,State:State}' \
  --output table
```

Same tag-lookup pattern as every EC2/EBS/ENI/EIP lab so far in this
series — resolve the human-assigned `Name` tags into the actual
`InstanceId`/`VolumeId` the `attach-volume` call needs, and confirm both
AZs match in the same pass (§2.4).

### 2.6 `attach-volume` returns `attaching` — an intermediate state, not the finish line

```bash
aws ec2 attach-volume \
  --volume-id vol-036a9018dff64e0f3 \
  --instance-id i-01fd19209732add67 \
  --device /dev/sdb
```
```json
{
    "VolumeId": "vol-036a9018dff64e0f3",
    "InstanceId": "i-01fd19209732add67",
    "Device": "/dev/sdb",
    "State": "attaching"
}
```

`attaching` is exactly the same "response received ≠ resource ready"
pattern seen throughout this series (EBS *creation* in Day 5,
`stop`/`start` in Day 7, ENI attachment in Day 11) — the call succeeding
means AWS accepted and started the operation, not that the attachment
has actually completed yet.

### 2.7 Two different "state" fields that are easy to conflate

```text
Volume's own State        →  available / in-use / etc.    (the WHOLE volume's status)
Attachment's own State     →  attaching / attached / detaching   (THIS SPECIFIC relationship's status)
```

```text
Volume
  │
  ├── overall state   → in-use
  │
  └── Attachments[0]
          ├── State    → attached
          ├── InstanceId → i-01fd19209732add67
          └── Device    → /dev/sdb
```

Seeing `State: in-use` at the top level and expecting the word
"attached" to appear there is a natural but incorrect expectation —
`in-use` is the volume-level status; `attached` lives one level deeper,
inside the specific attachment record. Querying the wrong field and
concluding "it's not attached yet" when it actually is would be a
self-inflicted false negative.

### 2.8 Bidirectional verification — from the volume's side, and from the instance's side

```bash
# From the volume's perspective
aws ec2 describe-volumes \
  --volume-ids vol-036a9018dff64e0f3 \
  --query 'Volumes[0].Attachments[0].{State:State,Device:Device,InstanceId:InstanceId}' \
  --output json

# From the instance's perspective
aws ec2 describe-instances \
  --instance-ids i-01fd19209732add67 \
  --query 'Reservations[0].Instances[0].BlockDeviceMappings[*].{Device:DeviceName,VolumeId:Ebs.VolumeId,State:Ebs.Status}' \
  --output table
```

Same bidirectional-check discipline as Day 10 (Elastic IP) and Day 11
(ENI) — confirming the relationship from both resources independently is
stronger evidence than trusting either query alone, and the instance-
side view additionally shows the primary root volume (`/dev/xvda`)
alongside the newly-attached `/dev/sdb`, giving you the instance's
*complete* storage picture in one call.

### 2.9 Why formatting/mounting is deliberately out of scope here (context, not required)

```text
EBS → attach → EC2 → lsblk → blkid → mkfs (only if genuinely empty) → mount → /etc/fstab
```

Running `mkfs` on a volume that already contains data destroys it — the
lab's explicit scope (attach only) matters because the natural next
instinct ("now let me mount it") is exactly the kind of action that's
dangerous to take reflexively on a volume whose contents you haven't
confirmed. If a future task *does* ask for mounting, the correct first
step is always inspecting (`blkid`) before formatting, never formatting
by default.

### 2.10 The compressed reasoning chain

```text
Requirement (attach xfusion-volume to xfusion-ec2 as /dev/sdb)
   → Find the instance by Name tag        → InstanceId, AZ
   → Find the volume by Name tag           → VolumeId, AZ, current State (available)
   → Compare AZs                            → must match
   → Attach: attach-volume --device /dev/sdb
   → Response shows State: attaching          → intermediate, not final
   → Verify from the volume's side             → Attachments[0].State == attached
   → Verify from the instance's side            → BlockDeviceMappings shows /dev/sdb, attached
   → Both directions agree → attachment confirmed
   → STOP — no mkfs, no mount, no fstab; those are a separate task
```

---

## 3. Concepts (reference)

### 3.1 EBS volumes exist independently of EC2 instances
A volume can be created, exist, and sit `available` with zero instances
attached — attachment is an explicit, separate operation connecting an
already-existing volume to an already-existing instance, not something
that happens automatically at creation time.

### 3.2 Attach vs. mount (the core distinction)
Attach = AWS-level: makes a block device visible to an instance. Mount =
OS-level: makes a filesystem on that block device accessible at a
directory path. A volume can be fully, correctly attached while
remaining completely unmounted and unformatted — this lab's entire
scope sits at the first layer only.

### 3.3 AWS device name vs. Linux kernel device name
The name passed to `attach-volume`/the console (e.g. `/dev/sdb`) is how
AWS tracks the attachment. The actual device name the Linux kernel
exposes inside the instance can differ, especially on Nitro-based
instance types (commonly `/dev/nvme*n1`). Don't conflate the two when
verifying — this lab's check is the AWS-side attachment record, not
`lsblk` output.

### 3.4 Volume-level state vs. attachment-level state
`describe-volumes` returns both a top-level `State` for the volume as a
whole (`available`/`in-use`) and a nested `Attachments[].State` for each
specific attachment relationship (`attaching`/`attached`/`detaching`).
They answer different questions and must be read from the correct
nesting level.

### 3.5 Common EBS/attachment states
```text
available   → volume exists, not attached to anything
attaching   → AWS is in the process of establishing the attachment
in-use      → volume (overall) is attached to an instance
attached    → (attachment-level) this specific relationship is established
detaching   → AWS is in the process of removing an attachment
```

### 3.6 Why you shouldn't format/mount reflexively
Formatting (`mkfs`) a volume that already contains data is destructive
and irreversible without a backup/snapshot. Inspecting first (`lsblk`,
`blkid`) to confirm a volume's actual state is the required habit before
ever running a formatting command, even if a task eventually does ask
for it.

---

## 4. Runbook

### 4.1 Find the EC2 instance
```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=xfusion-ec2" \
  --query 'Reservations[].Instances[].{ID:InstanceId,AZ:Placement.AvailabilityZone,State:State.Name}' \
  --output table
```
```text
ID                    AZ
i-01fd19209732add67   us-east-1a
```

### 4.2 Find the EBS volume
```bash
aws ec2 describe-volumes \
  --filters "Name=tag:Name,Values=xfusion-volume" \
  --query 'Volumes[].{ID:VolumeId,AZ:AvailabilityZone,State:State}' \
  --output table
```
```text
ID                      AZ           State
vol-036a9018dff64e0f3   us-east-1a   available
```
AZs match (§2.4) — proceed.

### 4.3 Attach the volume
```bash
aws ec2 attach-volume \
  --volume-id vol-036a9018dff64e0f3 \
  --instance-id i-01fd19209732add67 \
  --device /dev/sdb
```
```json
{
    "VolumeId": "vol-036a9018dff64e0f3",
    "InstanceId": "i-01fd19209732add67",
    "Device": "/dev/sdb",
    "State": "attaching"
}
```

### 4.4 Verify — from the volume's side (attachment-level state, not volume-level)
```bash
aws ec2 describe-volumes \
  --volume-ids vol-036a9018dff64e0f3 \
  --query 'Volumes[0].Attachments[0].{State:State,Device:Device,InstanceId:InstanceId}' \
  --output json
```
```json
{
    "State": "attached",
    "Device": "/dev/sdb",
    "InstanceId": "i-01fd19209732add67"
}
```

### 4.5 Verify — from the instance's side
```bash
aws ec2 describe-instances \
  --instance-ids i-01fd19209732add67 \
  --query 'Reservations[0].Instances[0].BlockDeviceMappings[*].{Device:DeviceName,VolumeId:Ebs.VolumeId,State:Ebs.Status}' \
  --output table
```
```text
Device      State      VolumeId
/dev/xvda   attached   vol-05816a30a150587f5
/dev/sdb    attached   vol-036a9018dff64e0f3
```
Both directions agree — attachment confirmed.

### 4.6 (Equivalent Console path, for reference)
```text
EC2 → Volumes → xfusion-volume
    → Actions → Attach volume
    → Instance: xfusion-ec2 → Device name: /dev/sdb → Attach
EC2 → Volumes → xfusion-volume → confirm State: in-use, attachment shows xfusion-ec2
```

### 4.7 Click "Check" in the lab UI to validate.

---

## 5. Final state

| Requirement | Result |
|---|---|
| `xfusion-ec2` and `xfusion-volume` located | ✅ |
| Same Availability Zone confirmed | ✅ both `us-east-1a` |
| Volume attached via `attach-volume` | ✅ |
| Attachment state `attached` (volume side) | ✅ |
| `/dev/sdb` confirmed `attached` (instance side) | ✅ |
| No formatting/mounting performed | ✅ (correctly out of scope) |

```text
xfusion-volume
vol-036a9018dff64e0f3
        │
        │ attached
        ▼
xfusion-ec2
i-01fd19209732add67
        │
        │ device
        ▼
     /dev/sdb
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Instance doesn't appear in the "Attach volume" screen | Instance and volume are in different Availability Zones | Compare `Placement.AvailabilityZone` (instance) against `AvailabilityZone` (volume) — they must match |
| `attach-volume` rejected — volume already attached | The volume already has an attachment to a different instance | `describe-volumes ... Attachments` to see the existing owner; don't blindly detach it in a real environment |
| Used the wrong device name (`/dev/sdc`, `/dev/xvdb`, etc.) | Didn't match the task's exact required name | Re-run with exactly `/dev/sdb` as specified |
| `describe-volumes` top-level `State` shows `in-use` but you expected to see "attached" | Looked at the wrong nesting level — `attached` lives inside `Attachments[].State`, not the top-level `State` | Query `Attachments[0].State` specifically (§2.7) |
| Volume shows attached in AWS but Linux `lsblk` doesn't show `/dev/sdb` | Nitro-based instance exposes it as `/dev/nvme1n1` or similar | This isn't a failure — the AWS attachment name and the Linux kernel device name can legitimately differ (§2.3); check `lsblk`/`ls -l /dev/disk/by-id/` |
| Tempted to immediately `mkfs`/`mount` after attaching | Conflating "attach" with "make usable" | Attach and mount are separate layers (§2.2); don't format a volume without first confirming it's actually empty |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.10) out loud
      from "attach xfusion-volume to xfusion-ec2 as /dev/sdb" to
      "verified from both directions."
- [ ] Explain, in one sentence, the difference between attaching an EBS
      volume and mounting a filesystem on it.
- [ ] Explain why the AWS device name you specify at attach time might
      not match what `lsblk` shows inside the instance.
- [ ] Explain the difference between a volume's top-level `State` and
      its `Attachments[].State` — which one answers "is this specific
      attachment complete"?
- [ ] Explain why Availability Zone compatibility has to be checked
      before attempting an attach, not discovered via a rejected API
      call.
- [ ] Repeat the full sequence on a different volume/instance pair from
      memory, verifying bidirectionally — then explain, without running
      it, what you would check before ever running `mkfs` on the
      resulting device.

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
what existing state/compatibility constrains my choice, what CLI
service+operation performs the read, what performs the write, how do I
verify — ideally from both directions of the relationship, and at the
correct nesting level." End with a compressed step-chain.>

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
