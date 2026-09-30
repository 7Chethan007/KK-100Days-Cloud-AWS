# Day 11 — Attach an Elastic Network Interface (ENI) to an EC2 Instance

A KodeKloud "100 Cloud/AWS" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

---

## 1. Scenario

The Nautilus DevOps team has an existing EC2 instance and an existing,
unattached Elastic Network Interface (ENI), both already created. Task:
attach `datacenter-eni` to `datacenter-ec2`, in `us-east-1`.

Two explicit preconditions the task calls out:
1. The ENI's status must be **attached** before submitting.
2. The instance's **initialization must be complete** before submitting
   — i.e. don't just check that it's `running`, confirm its status
   checks have finished too.

Desired state:

```text
                       VPC
                        │
                        ▼
                 datacenter-ec2
                  /           \
                 ▼             ▼
         Primary ENI        datacenter-eni
         device 0            device 1
                              (in-use)
```

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 What an ENI actually is

An EC2 instance doesn't own an IP address directly — it communicates
through a network interface, the same way a physical server communicates
through a NIC:

```text
EC2 Instance
      │
      ▼
Network Interface (ENI)
      │
      ├── Private IP
      ├── MAC address
      ├── Security Groups
      ├── Subnet
      └── Network connectivity
```

Every instance starts with a **primary ENI** (device index `0`). A
**secondary ENI** (device index `1`, `2`, ...) can be attached later —
useful for giving an instance an additional network identity: a second
private IP, a different subnet, a different security-group association,
or traffic separation for network appliances and failover setups.

### 2.2 Instance initialization vs. instance state — the same distinction as Day 7, applied here

The task explicitly requires confirming initialization is complete, not
just that the instance is `running`:

```text
Instance STATE     → pending → running → stopping → stopped
Status CHECKS       → SystemStatus, InstanceStatus (AWS's own health probes)
```

A freshly-launched instance can show `running` while its status checks
are still `Initializing` — exactly the same distinction covered in Day
7's instance-type-change lab. Attaching a new network interface to an
instance that hasn't finished initializing risks acting on a resource
whose networking stack isn't fully ready yet, which is precisely why
this task calls the precondition out explicitly rather than leaving it
implicit.

```bash
aws ec2 describe-instance-status \
  --region us-east-1 \
  --instance-ids <instance-id> \
  --query 'InstanceStatuses[0].{Instance:InstanceState.Name,InstanceStatus:InstanceStatus.Status,SystemStatus:SystemStatus.Status}' \
  --output table
```

Wait for both `InstanceStatus` and `SystemStatus` to read `ok` before
proceeding — the same `aws ec2 wait instance-status-ok` pattern from
Day 7 applies directly here.

### 2.3 An ENI's lifecycle — `available` vs. `in-use`

```text
Create ENI
    │
    ▼
available          (exists, attached to nothing)
    │
    │ attach
    ▼
in-use              (attached to a specific instance, at a specific device index)
    │
    │ detach
    ▼
available
```

Before this task's change, `describe-network-interfaces` on
`datacenter-eni` should show `Status: available` and
`Attachment.InstanceId: None` — this is the expected starting state, not
a sign anything is broken (same "allocated but not yet associated"
framing as Day 10's Elastic IP lab, one layer down the networking
stack).

### 2.4 Two hard constraints an ENI attachment must satisfy

```text
1. Availability Zone   → the ENI and the instance must be in the SAME AZ
2. Subnet               → the ENI's subnet determines its AZ and IP range
```

```text
us-east-1a
 ├── EC2:  datacenter-ec2
 └── ENI:  datacenter-eni
```

An ENI is physically bound to the AZ its subnet lives in — you cannot
attach an ENI from `us-east-1b` to an instance running in `us-east-1a`.
Comparing both resources' AZs *before* attempting the attach is what
prevents an avoidable API rejection.

### 2.5 Find both resources by tag, resolve to real IDs

```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=datacenter-ec2" \
  --query 'Reservations[].Instances[].{InstanceId:InstanceId,State:State.Name,AZ:Placement.AvailabilityZone}' \
  --output table

aws ec2 describe-network-interfaces \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=datacenter-eni" \
  --query 'NetworkInterfaces[].{ENI:NetworkInterfaceId,Status:Status,AZ:AvailabilityZone,Instance:Attachment.InstanceId}' \
  --output table
```

Same tag-lookup pattern as every prior EC2/EBS/EIP lab in this series —
resolve human-assigned `Name` tags into the actual `InstanceId` and
`NetworkInterfaceId` the mutating call needs, and get both resources'
AZs in the same query so the §2.4 comparison is immediate.

### 2.6 `--device-index` — telling AWS which network slot this is

```bash
aws ec2 attach-network-interface \
  --region us-east-1 \
  --network-interface-id <eni-id> \
  --instance-id <instance-id> \
  --device-index 1
```

```text
device index 0   → primary ENI (always present)
device index 1   → first secondary ENI
device index 2   → second secondary ENI
...
```

Analogous to `eth0`/`eth1`/`eth2` in traditional Linux networking — the
index is how AWS (and the instance's OS) distinguishes multiple attached
interfaces. Since the primary ENI already occupies index `0`, the first
attached secondary ENI uses `1`.

### 2.7 A successful `attach-network-interface` call is not yet "attached" in the sense the task means

```json
{
    "AttachmentId": "eni-attach-xxxxxxxxx"
}
```

Getting an `AttachmentId` back confirms AWS *accepted* the attach
request — the same "response received ≠ verified state" pattern as
every mutating call across this series (Days 5, 7, 8, 9, 10). The task's
actual requirement ("make sure status is attached before submitting")
means querying the ENI's `Status` field afterward and confirming it
reads `in-use`, not just checking that the attach command didn't error.

### 2.8 Bidirectional verification — from the ENI, and from the instance

```text
ENI → EC2:  describe-network-interfaces  → "what instance is THIS ENI attached to, and at what status?"
EC2 → ENI:  describe-instances (NetworkInterfaces[])  → "what interfaces does THIS instance now have?"
```

Same bidirectional-check discipline as Day 10's Elastic IP lab —
confirming the relationship from both ends is stronger evidence than
checking either resource alone. From the EC2 side specifically, you
should see the instance's `NetworkInterfaces` array grow from one entry
(the primary) to two (primary + `datacenter-eni`), both showing
`attached`.

### 2.9 The compressed reasoning chain

```text
Requirement (attach datacenter-eni to datacenter-ec2, confirmed in-use, instance fully initialized)
   → Find the instance by Name tag           → InstanceId, AZ
   → Confirm instance status checks are ok    → describe-instance-status, wait if Initializing
   → Find the ENI by Name tag                 → NetworkInterfaceId, AZ, current Status
   → Compare AZs                               → must match; if not, this ENI can't attach here
   → Confirm ENI status = available            → not already attached elsewhere
   → Attach: attach-network-interface --device-index 1
   → Capture AttachmentId returned
   → Verify from the ENI side                  → describe-network-interfaces: Status = in-use, correct InstanceId
   → Verify from the EC2 side                  → describe-instances: NetworkInterfaces[] now has 2 entries
   → Both directions agree → attachment confirmed, task requirements satisfied
```

---

## 3. Concepts (reference)

### 3.1 Primary vs. secondary ENI
Every instance has exactly one primary ENI, created at launch, at device
index `0` — it cannot be detached while the instance is running.
Secondary ENIs are optional, attached/detached independently, and give
an instance additional network identities (extra private IPs, different
subnets, separate security groups).

### 3.2 Instance state vs. status checks (recap from Day 7)
State (`running`) reflects lifecycle position; status checks
(`SystemStatus`, `InstanceStatus`) reflect AWS's own confirmed health of
the underlying hardware and instance. A `running` instance can still be
`Initializing` — this task's precondition exists specifically because
network configuration changes are sensitive to exactly this gap.

### 3.3 ENI AZ/subnet binding
An ENI's subnet fixes its Availability Zone permanently for that ENI's
lifetime — this is why AZ compatibility must be checked *before*
attempting an attach, not discovered via a rejected API call.

### 3.4 `describe-network-interfaces` vs. `describe-instances` for ENI info
- `describe-network-interfaces --network-interface-ids <id>` — the
  authoritative, ENI-centric view: status, attachment, AZ, subnet,
  private IP.
- `describe-instances ... .NetworkInterfaces[]` — the instance-centric
  view: every ENI currently attached to a given instance, useful for
  confirming the instance's *complete* network picture after a change.

### 3.5 Device index as AWS's interface-ordering mechanism
Conceptually parallel to `eth0`/`eth1` naming in traditional Linux
networking — `--device-index` tells AWS (and ultimately the instance's
OS) which network slot a given ENI occupies. Index `0` is always the
primary; secondary ENIs take the next available index.

### 3.6 Why "attach succeeded" and "status is in-use" are different claims
`attach-network-interface` returning an `AttachmentId` confirms AWS
accepted and is processing the request; `Status: in-use` on a subsequent
`describe-network-interfaces` call confirms the attachment actually
completed. The task explicitly asks you to verify the second, not just
trust the first.

---

## 4. Runbook

### 4.1 Find the EC2 instance
```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=datacenter-ec2" \
  --query 'Reservations[].Instances[].{InstanceId:InstanceId,State:State.Name,AZ:Placement.AvailabilityZone,PrivateIP:PrivateIpAddress,InstanceType:InstanceType}' \
  --output table
```
```text
InstanceId    i-0cc7b52ed0dcea99c
State         running
AZ            us-east-1a
PrivateIP     172.31.x.x
InstanceType  t2.micro
```

### 4.2 Confirm instance initialization is complete (status checks, not just state)
```bash
aws ec2 describe-instance-status \
  --region us-east-1 \
  --instance-ids i-0cc7b52ed0dcea99c \
  --query 'InstanceStatuses[0].{Instance:InstanceState.Name,InstanceStatus:InstanceStatus.Status,SystemStatus:SystemStatus.Status}' \
  --output table
```
If either shows `Initializing`:
```bash
aws ec2 wait instance-status-ok --region us-east-1 --instance-ids i-0cc7b52ed0dcea99c
```

### 4.3 Find the ENI
```bash
aws ec2 describe-network-interfaces \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=datacenter-eni" \
  --query 'NetworkInterfaces[].{ENI:NetworkInterfaceId,Status:Status,AZ:AvailabilityZone,Subnet:SubnetId,PrivateIP:PrivateIpAddress,Instance:Attachment.InstanceId}' \
  --output table
```
```text
ENI        eni-0abc123def456789a
Status     available
AZ         us-east-1a
Subnet     subnet-xxxxxxxx
PrivateIP  172.31.x.x
Instance   None
```
Confirms: allocated, unattached, in the correct AZ (§2.3, §2.4).

### 4.4 Compare AZs
```text
EC2 AZ:  us-east-1a
ENI AZ:  us-east-1a
→ compatible, proceed
```

### 4.5 Attach the ENI
```bash
aws ec2 attach-network-interface \
  --region us-east-1 \
  --network-interface-id eni-0abc123def456789a \
  --instance-id i-0cc7b52ed0dcea99c \
  --device-index 1
```
```json
{
    "AttachmentId": "eni-attach-0123456789abcdef0"
}
```

### 4.6 Verify — from the ENI's side
```bash
aws ec2 describe-network-interfaces \
  --region us-east-1 \
  --network-interface-ids eni-0abc123def456789a \
  --query 'NetworkInterfaces[0].{ENI:NetworkInterfaceId,Status:Status,InstanceId:Attachment.InstanceId,DeviceIndex:Attachment.DeviceIndex}' \
  --output table
```
```text
Status       in-use
InstanceId   i-0cc7b52ed0dcea99c
DeviceIndex  1
```

### 4.7 Verify — from the EC2 instance's side
```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --instance-ids i-0cc7b52ed0dcea99c \
  --query 'Reservations[].Instances[].NetworkInterfaces[].{ENI:NetworkInterfaceId,DeviceIndex:Attachment.DeviceIndex,Status:Attachment.Status,PrivateIP:PrivateIpAddress}' \
  --output table
```
```text
ENI              DeviceIndex   Status
eni-primary...   0             attached
eni-0abc123...   1             attached
```
Both directions agree — attachment confirmed.

### 4.8 (Equivalent Console path, for reference)
```text
EC2 → Instances → datacenter-ec2
    → confirm State: Running AND status checks passed (not just "running")
EC2 → Network Interfaces → datacenter-eni
    → confirm Status: Available, correct AZ
    → Actions → Attach → Instance: datacenter-ec2 → Attach
EC2 → Network Interfaces → datacenter-eni
    → confirm Status: In-use
EC2 → Instances → datacenter-ec2 → Networking → Network interfaces
    → confirm count went from 1 → 2, second interface = datacenter-eni, attached
```

### 4.9 Click "Check" in the lab UI to validate.

---

## 5. Final state

| Requirement | Result |
|---|---|
| `datacenter-ec2` located | ✅ |
| Instance initialization confirmed complete (status checks `ok`) | ✅ |
| `datacenter-eni` located | ✅ |
| AZ compatibility confirmed | ✅ both `us-east-1a` |
| ENI attached (`attach-network-interface`) | ✅ |
| ENI status verified `in-use` | ✅ |
| Verified from EC2 side (2 network interfaces) | ✅ |

```text
                       VPC
                        │
                        ▼
                 datacenter-ec2
                  /           \
                 ▼             ▼
         Primary ENI        datacenter-eni
         device 0            device 1
         (attached)          Status: in-use
                              Attachment: eni-attach-...
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `attach-network-interface` rejected with an AZ/subnet error | ENI and instance are in different Availability Zones | Compare `Placement.AvailabilityZone` (instance) against `AvailabilityZone` (ENI) before attaching; an ENI can't cross AZs |
| ENI already shows `Status: in-use` before you've attached anything | It's already attached to a different instance | `describe-network-interfaces ... Attachment` to see which instance owns it; don't blindly detach it in a real environment |
| `attach-network-interface` rejected for interface-count reasons | The instance type has reached its maximum number of network interfaces | Check the instance type's ENI limit; this isn't something to retry past |
| Resource "not found" for instance or ENI | Wrong region selected | Confirm `us-east-1` explicitly, in both console and every `--region` flag |
| Attached while the instance was still `Initializing` | Checked `State: running` but never checked status checks separately | `describe-instance-status`, and `wait instance-status-ok` if not yet `ok`, before attaching (§2.2) |
| Unsure whether the attachment "really" completed | Only checked the `attach-network-interface` response, not the resulting state | Verify bidirectionally: `describe-network-interfaces` AND `describe-instances` (§2.8) |
| Multiple resources sharing a similar `Name` tag | Names are just tags, not guaranteed unique | Always resolve to the actual resource ID before operating, rather than trusting a name match alone |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.9) out loud
      from "attach datacenter-eni to datacenter-ec2" to "verified from
      both directions, with initialization confirmed complete."
- [ ] Explain, in one sentence, why an ENI's Availability Zone can't
      differ from the instance it's being attached to.
- [ ] Explain the difference between an instance being `running` and an
      instance having completed initialization — why does this task
      care about the difference?
- [ ] Explain what `--device-index 1` actually tells AWS, and why the
      primary ENI is always index `0`.
- [ ] Explain why an `AttachmentId` in the response isn't sufficient
      proof the task's "status is attached" requirement is satisfied.
- [ ] Repeat the full sequence on a different instance/ENI pair from
      memory, verifying bidirectionally and confirming status checks
      before attaching — not just after.

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
verify — ideally from both directions of the relationship." End with a
compressed step-chain.>

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
