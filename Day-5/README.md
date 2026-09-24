# Day 5 — Create a GP3 EBS Volume

A KodeKloud "100 Cloud/AWS" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

---

## 1. Scenario

The Nautilus DevOps team is migrating infrastructure to AWS in small,
controlled increments rather than one large cutover. Task: create an EBS
volume with:

| Requirement | Value             |
|---|---|
| Name        | `nautilus-volume` |
| Type        | `gp3`             |
| Size        | `2 GiB`           |
| Region      | `us-east-1`       |

We also created a second volume, `test-ui`, through the AWS Console —
deliberately, to see the UI workflow and then verify it landed correctly
using the CLI.

**Credentials/environment** (lab-specific, rotates per session):

| Field | Value |
|---|---|
| Console URL | `https://769217380749.signin.aws.amazon.com/console?region=us-east-1` |
| Username | `kk_labs_user_517905` |
| Region constraint | `us-east-1` only |
| Access | via `aws-client` host; run `showcreds` there to retrieve credentials |

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 What are we actually creating?

An **EBS volume (Elastic Block Store)** is persistent block storage — a
virtual disk you can attach to an EC2 instance. It is **not** an EC2
instance itself, and it is not automatically attached to one:

```text
              AWS Region
             us-east-1
                  │
                  ▼
          Availability Zone
            us-east-1a
                  │
                  ▼
             EBS Volume
        ┌──────────────────┐
        │  gp3             │
        │  2 GiB           │
        │  Persistent disk │
        └──────────────────┘
```

For this task we only create the disk — no instance, no attachment.

### 2.2 "Create a volume" → what does it need, and what constrains it?

Same four questions as every AWS task:

```text
1. What resource am I creating?         → an EBS volume
2. What does it live inside?            → an Availability Zone (not just a region)
3. What region/AZ?                      → us-east-1, we chose us-east-1a
4. What type/size?                      → gp3, 2 GiB (stated requirements)
```

### 2.3 Region vs. Availability Zone — a distinction this task exposes

The task says "region = us-east-1," but `create-volume` requires an
**Availability Zone**, not just a region:

```text
us-east-1
│
├── us-east-1a   ← we placed the volume here
├── us-east-1b
├── us-east-1c
└── ...
```

A region is a collection of physically separate, isolated AZs. EBS
volumes live in exactly one AZ — this matters later, because an EC2
instance can only attach volumes that live in *its own* AZ. Picking
`us-east-1a` here isn't arbitrary; it's the first AZ available, and there
was no stated constraint pinning it to a specific one.

### 2.4 `gp3` — what it actually is

`gp3` is an EBS **General Purpose SSD** volume type. Unlike its
predecessor `gp2` (where IOPS scaled with size), `gp3` decouples size from
performance — every `gp3` volume gets a baseline performance regardless of
size, adjustable independently if needed:

```text
gp3 baseline (unless explicitly overridden):
  IOPS       = 3000
  Throughput = 125 MiB/s
```

The lab didn't require tuning either, so both were left at AWS's defaults
— confirmed in the `create-volume` response, not assumed.

### 2.5 The AWS CLI structural pattern

```text
aws <service> <operation> [options]

aws ec2 describe-volumes   → EC2 service, read operation, on volumes
aws ec2 create-volume      → EC2 service, write operation, a volume
```

Read operations are `describe-*`/`list-*`/`get-*`; write operations are
`create-*`/`delete-*`/`modify-*` — the same pattern as every prior lab in
this series (Day 3's subnets, Day 4's S3 versioning).

### 2.6 Reading `create-volume`'s options

```bash
aws ec2 create-volume \
  --region us-east-1 \
  --availability-zone us-east-1a \
  --volume-type gp3 \
  --size 2 \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=nautilus-volume}]'
```

- `--region` — where the API call is executed against.
- `--availability-zone` — which AZ physically hosts the volume (§2.3).
- `--volume-type` / `--size` — the two stated requirements, directly.
- `--tag-specifications` — attaches metadata at creation time. AWS
  identifies the volume by a generated ID (`vol-...`), never by name — the
  `Name` tag is a human-friendly label layered on top, not the resource's
  real identity:

```text
Volume ID (AWS's real identity):  vol-01f5e6e5f8a6b5157
Name tag (human label):           nautilus-volume
```

### 2.7 A successful response is not the finished state

```text
create-volume response → "State": "creating"
```

Getting a response back means the API call succeeded, not that the
resource is ready. The lifecycle is:

```text
create-volume
      │
      ▼
   creating
      │
      ▼
   available   ← the state that actually means "usable"
```

This is the same lesson as Day 2's JupyterLab lab (success = the
documented ready-state, not "the command didn't error") — here it's a
resource *state* rather than a log line, but the habit is identical.

### 2.8 `--volume-ids` (exact lookup) vs `--filters` (search)

```bash
--volume-ids vol-01f5e6e5f8a6b5157        # "give me exactly this volume"
--filters "Name=tag:Name,Values=test-ui"  # "find volumes matching this tag"
```

Use `--volume-ids` when you already have the authoritative ID (e.g. just
returned by `create-volume`). Use `--filters` when you only know a
human-assigned property (like a `Name` tag) and need to search for it —
this is exactly the situation after creating something through the
**Console**, where you never see a CLI response to grab an ID from.

### 2.9 Console and CLI are two interfaces to the same API

Creating `test-ui` in the Console and then finding it via
`describe-volumes --filters` in the CLI proves a point worth
internalizing:

```text
             AWS Console
                  │
                  ▼
             AWS API
                  ▲
                  │
              AWS CLI
```

Neither interface creates a different "kind" of resource — both are
front-ends over the identical underlying API and the identical resource
store. Anything created in one is fully visible and manageable from the
other.

### 2.10 The full task rhythm

```text
1. Identify   — aws sts get-caller-identity (who am I, which account?)
2. Confirm    — aws configure get region (right region?)
3. Inspect    — aws ec2 describe-volumes (existing state)
4. Act        — aws ec2 create-volume (apply the change)
5. Observe    — note the returned VolumeId + State=creating
6. Verify     — aws ec2 describe-volumes --volume-ids ... (State=available?)
```

Same `inspect → reason → act → verify` shape as every prior lab, with one
addition at the front: confirming *identity* (§3.1) before confirming
*state*, since this lab explicitly called out checking the account first.

---

## 3. Concepts (reference)

### 3.1 Why check identity before touching infrastructure
`aws sts get-caller-identity` answers "who am I operating as, in which
AWS account?" before any resource is touched:

```text
{
    "UserId": "AIDA3GGHNYGGR4N3JFJQX",
    "Account": "769217380749",
    "Arn": "arn:aws:iam::769217380749:user/kk_labs_user_517905"
}
```

This guards against a real, common mistake: running the right commands
against the wrong account (e.g. a stale AWS CLI profile pointed somewhere
else). Worth running as a first step whenever entering an unfamiliar or
freshly-provisioned AWS environment, lab or otherwise.

### 3.2 EBS volumes are zonal, not regional
A region is a set of independent Availability Zones; an EBS volume is
created in, and stays pinned to, exactly one AZ. This becomes load-bearing
the moment you try to attach it to an EC2 instance — the instance and the
volume must be in the *same* AZ, not just the same region.

### 3.3 `gp3` vs `gp2`
`gp2`'s IOPS scaled with volume size (bigger volume = more IOPS,
automatically). `gp3` decouples the two — a fixed baseline (3000 IOPS,
125 MiB/s) regardless of size, independently adjustable up to a point if a
workload needs more, without needing to over-provision size just to buy
performance.

### 3.4 Resource ID vs. Name tag
AWS resources are identified internally by a generated, immutable ID
(`vol-...`, `vpc-...`, `i-...`). A `Name` tag is ordinary metadata, exactly
like any other tag — it exists purely for human readability and search,
and multiple resources could technically share the same `Name` tag value
(though that defeats its purpose).

### 3.5 `--filters` vs `--volume-ids`
- `--volume-ids` — exact lookup by the resource's real identity; fastest
  and most precise when you have it.
- `--filters "Name=tag:Name,Values=..."` — a search across all volumes,
  needed when you only know a tag/property rather than the ID (e.g. after
  creating a resource via the Console).

### 3.6 `--query` vs `--output`
- `--query` — a JMESPath expression, evaluated client-side, that projects
  down to specific fields from the full JSON response.
- `--output table` — controls the *rendering* of whatever `--query` (or
  the full response) returns, as a human-readable table instead of raw
  JSON.

These act on different stages of the same pipeline:

```text
AWS API response (full JSON)
        │
        ▼  --query   (select which fields)
projected fields
        │
        ▼  --output  (choose rendering)
table / json / text
```

---

## 4. Runbook

### 4.1 Identify current AWS identity
```bash
aws sts get-caller-identity
```
```text
{
    "UserId": "AIDA3GGHNYGGR4N3JFJQX",
    "Account": "769217380749",
    "Arn": "arn:aws:iam::769217380749:user/kk_labs_user_517905"
}
```

### 4.2 Confirm configured region
```bash
aws configure get region
```
```text
us-east-1
```

### 4.3 Inspect existing volumes (before creating anything)
```bash
aws ec2 describe-volumes \
  --region us-east-1 \
  --query 'Volumes[*].[VolumeId,Size,VolumeType,State,AvailabilityZone,Tags[?Key==`Name`].Value|[0]]' \
  --output table
```
No existing volumes were visible for this lab user.

### 4.4 Create the volume
```bash
aws ec2 create-volume \
  --region us-east-1 \
  --availability-zone us-east-1a \
  --volume-type gp3 \
  --size 2 \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=nautilus-volume}]'
```
```json
{
    "AvailabilityZoneId": "use1-az6",
    "Iops": 3000,
    "Tags": [
        {
            "Key": "Name",
            "Value": "nautilus-volume"
        }
    ],
    "VolumeType": "gp3",
    "MultiAttachEnabled": false,
    "Throughput": 125,
    "VolumeId": "vol-01f5e6e5f8a6b5157",
    "Size": 2,
    "SnapshotId": "",
    "AvailabilityZone": "us-east-1a",
    "State": "creating",
    "CreateTime": "2026-09-24T14:31:25.000Z",
    "Encrypted": false
}
```
Note `"State": "creating"` — proceed to verify, don't stop here (§2.7).

### 4.5 Verify by exact ID
```bash
aws ec2 describe-volumes \
  --region us-east-1 \
  --volume-ids vol-01f5e6e5f8a6b5157 \
  --query 'Volumes[0].[VolumeId,Size,VolumeType,State,AvailabilityZone,Tags[?Key==`Name`].Value|[0]]' \
  --output table
```
```text
---------------------------
|     DescribeVolumes     |
+-------------------------+
|  vol-01f5e6e5f8a6b5157  |
|  2                      |
|  gp3                    |
|  available              |
|  us-east-1a             |
|  nautilus-volume        |
+-------------------------+
```
All five requirements confirmed: ID assigned, `2` GiB, `gp3`, `available`,
`us-east-1a`, tagged `nautilus-volume`.

### 4.6 (Optional, for practice) Create a second volume via the Console
Navigate: **EC2 → Elastic Block Store → Volumes → Create volume**, then
set:

| Console field | Value | CLI equivalent |
|---|---|---|
| Volume type | General Purpose SSD (gp3) | `--volume-type gp3` |
| Size | 2 GiB | `--size 2` |
| Availability Zone | us-east-1a | `--availability-zone us-east-1a` |
| Tags → Name | `test-ui` | `--tag-specifications ...Value=test-ui` |

Click **Create volume**.

### 4.7 Verify the Console-created volume from the CLI, by tag
```bash
aws ec2 describe-volumes \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=test-ui" \
  --query 'Volumes[*].[VolumeId,Size,VolumeType,State,AvailabilityZone,Tags[?Key==`Name`].Value|[0]]' \
  --output table
```
```text
-----------------------------------------------------------------------------
|                              DescribeVolumes                              |
+------------------------+----+------+------------+-------------+-----------+
|  vol-0ba8bcdb2c1ba4dc3 |  2 |  gp3 |  available |  us-east-1a |  test-ui  |
+------------------------+----+------+------------+-------------+-----------+
```
Confirms the Console and CLI operate on the same underlying resources
(§2.9) — no CLI-side creation was needed to see it here.

### 4.8 Click "Check" in the lab UI to validate.

---

## 5. Troubleshooting

| Symptom                                                | Cause                                                          | Fix                                                                                   |
|-----------------------------------------------------------|-------------------------------------------------------------------|------------------------------------------------------------------------------------------|
| `create-volume` succeeds but lab check fails            | Wrong `Name` tag value, wrong size, or wrong volume type          | Re-run §4.5 and diff every field against the stated requirements exactly               |
| Volume shows `"State": "creating"` and check fails immediately | Checked/submitted before the volume finished provisioning       | Re-run `describe-volumes` and wait for `"State": "available"` before submitting        |
| `describe-volumes --filters` returns nothing for a Console-created volume | Tag key/value typo'd in the Console, or wrong region selected in Console | Re-check the tag exactly (`Name` = `test-ui`), and that the Console's region matches `us-east-1` |
| `An error occurred (UnauthorizedOperation)`             | Credentials not loaded, or expired (labs have a 1-hour window)     | Re-run `showcreds` on `aws-client`, re-export/reconfigure credentials                    |
| Volume created in the wrong Availability Zone            | `--availability-zone` omitted or mistyped                         | Re-run `create-volume` with the intended AZ — note a volume's AZ can't be changed after creation, it must be recreated |
| `--query` returns empty / errors                        | JMESPath syntax mismatch with actual field names                   | Drop `--query` temporarily, inspect raw JSON output, re-derive the path                 |

---

## 6. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.10) out loud
      from "create a 2 GiB gp3 volume named X" to the final verify step.
- [ ] Explain, in one sentence, why an EBS volume is pinned to an
      Availability Zone rather than just a region.
- [ ] Explain why `"State": "creating"` in the `create-volume` response
      isn't sufficient proof the task is done.
- [ ] Explain the difference between `--volume-ids` and `--filters`, and
      when you'd be forced to use the latter.
- [ ] Create a volume through the Console, then verify it purely from the
      CLI using a tag filter, without ever copying an ID from the UI.
- [ ] Repeat the full CLI runbook end-to-end from memory, then verify with
      a real `describe-volumes` call showing `State: available`, not just
      "the create command returned no error."

---

## 7. Document template for future lab days

Use this skeleton for `day-N/README.md`:

```markdown
# Day N — <task title>

## 1. Scenario
<what was asked, which account/region/resources, constraints, credentials
if relevant>

## 2. Reasoning model — how to derive the commands
<walk the requirement down to a subsystem/resource hierarchy, step by step,
in the order you'd actually discover it: "what resource, what does it live
inside, what region/account constraints apply, what existing state
constrains my choice, what CLI service+operation performs the read, what
performs the write, how do I verify." Include any math/derivation worked
out longhand. End with a compressed step-chain.>

## 3. Concepts (reference)
<one subsection per concept the task exercises — explain WHY, not just WHAT>

## 4. Runbook
<copy-pasteable commands in the order run, WITH the actual intermediate
output/results captured inline as code blocks, not just the commands>

## 5. Troubleshooting
<symptom / cause / fix table>

## 6. Do it yourself (checklist)
<"walk the reasoning chain out loud" + comprehension questions +
"redo without copy-pasting, recompute derived values" prompt>
```
