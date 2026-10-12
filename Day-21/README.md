# Day 21 — Create an EC2 Instance with an Elastic IP

A KodeKloud "100 Cloud/AWS" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

Day 10 covered attaching an Elastic IP to an EC2 instance that already
existed. This lab builds both resources from scratch — instance first,
Elastic IP second — then performs the exact same association, so the
"response accepted ≠ resource ready" and "verify from both sides"
discipline from Day 10 applies again, just with one more resource to
stand up first.

---

## 1. Scenario

Launch a Linux EC2 instance named `devops-ec2`, allocate an Elastic IP
named `devops-eip`, and associate the two.

| Resource | Required configuration |
|---|---|
| EC2 instance name | `devops-ec2` |
| Instance type | `t2.micro` |
| Elastic IP name | `devops-eip` |
| Association | Elastic IP attached to `devops-ec2` |

```text
Client / Internet
      │ connects to public IP
      ▼
Elastic IP: 34.204.146.196  ──(association)──▶  devops-ec2
  (eipalloc-0e0881583ae238cc0)   (eipassoc-0a254ea0f7d8411ac)   i-0c49c05fa521f523c
                                                   private IP: 172.31.32.190
```

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 Launching an instance and allocating an Elastic IP are two independent, order-flexible resources

```text
EC2 instance:  compute + OS + networking, exists the moment it's launched
Elastic IP:    a static public IPv4, exists the moment it's allocated
Association:   a THIRD, separate action connecting the two
```

Neither resource depends on the other to be *created* — you could
allocate the Elastic IP first and launch the instance second, or vice
versa, as this lab does. What *does* depend on both existing is the
association step, which is why it's correctly sequenced last in both
the console and CLI procedures.

### 2.2 Allocating an Elastic IP only reserves it — same "accepted ≠ attached" split as Day 10

```text
Allocate Elastic IP  → reserved to your account, AllocationId assigned
Associate Elastic IP → AssociationId assigned, InstanceId populated
```

This is the identical two-step structure Day 10 worked through for an
*existing* Elastic IP — allocation and association are separate API
calls with separate, separately-checkable results. An allocated but
unassociated Elastic IP is a valid, billable, completely normal
intermediate state, not a half-finished mistake.

### 2.3 Tagging the Elastic IP `Name=devops-eip` is metadata, not identity

```text
AllocationId: eipalloc-0e0881583ae238cc0   ← the actual AWS identifier
Tag Name:      devops-eip                   ← a human-readable label, not unique by AWS rule
```

AWS resources are addressed by their IDs; the `Name` tag exists purely
for human/CLI filtering convenience (`--filters "Name=tag:Name,..."`).
Nothing stops two Elastic IPs from sharing the same `Name` tag in
theory — the filter is a search convenience over tags you chose to
set, not a system-enforced unique key, the same distinction drawn for
EC2/EBS "Name" tags throughout this series since Day 3.

### 2.4 Verifying from the instance's `describe-instances` *and* the address's `describe-addresses` is the bidirectional check

```text
describe-addresses   → does this EIP record say it's attached to this instance?
describe-instances   → does this instance's network interface show this public IP?
```

Checking only one side risks a stale or partially-propagated API
response confirming something the other side hasn't caught up to yet
— exactly the "verify bidirectionally / from the decisive side"
principle carried over from Day 10 (EIP), Day 11 (ENI), and Day 12
(EBS attach). Here it's made explicit as Step 4 of the CLI procedure
rather than assumed.

### 2.5 `--filters` for discovery, `--query` for projection — same split as every CLI lab since Day 5

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-ec2" \
  --query 'Reservations[].Instances[].{...}' \
  --output table
```

`--filters` narrows *which* resources the API returns (server-side);
`--query` reshapes *what fields* appear in the already-filtered
response (client-side, JMESPath). Reservations wrapping Instances is
EC2's API shape, not something this lab introduces — `Reservations[].
Instances[]` has to be flattened the same way in every `describe-
instances` call regardless of what's being asked.

### 2.6 A static Elastic IP solves a problem an auto-assigned public IP doesn't

```text
Auto-assigned public IP:  can change across a stop/start cycle
Elastic IP:                stays allocated to your account until explicitly released
```

This is why the lab specifically asks for an Elastic IP rather than
relying on the instance's default public IPv4 — a stable address is
the whole point when an application needs to be reachable at a
consistent endpoint. It's also why an unused Elastic IP should be
released rather than left allocated indefinitely — AWS bills for
Elastic IPs that aren't actively associated with a running instance
under current pricing rules.

### 2.7 A stable public IP is necessary but not sufficient for reachability

```text
Elastic IP associated → instance has a stable public address
          +
Security group rules  → must allow the specific inbound port/protocol
Route table            → subnet must route 0.0.0.0/0 to an internet gateway
Application/service    → must actually be listening
```

Associating the Elastic IP only guarantees the *address* is stable and
reachable at the network layer the IP itself controls — whether a
client can actually connect to an application on that address depends
on layers this lab didn't touch (security groups, NACLs, route
tables, the service itself). Troubleshooting a failed connection means
checking those layers separately, not re-checking the EIP association
again.

### 2.8 The compressed reasoning chain

```text
Requirement: devops-ec2 (t2.micro) with devops-eip attached
   → launch the instance first (or allocate the EIP first — order between these two is flexible)
   → allocate an Elastic IP, tag it devops-eip (tag = convenience label, not identity, §2.3)
   → associate the EIP with devops-ec2 — a third, separate action (§2.1, §2.2)
   → verify from the address side: describe-addresses → InstanceId + AssociationId populated
   → verify from the instance side: describe-instances → NetworkInterfaces[].Association.PublicIp
   → cross-check both sides agree on the same public IP (§2.4)
   → if unreachable despite correct association, check security groups/routes/app — NOT the EIP again (§2.7)
```

---

## 3. Concepts (reference)

### 3.1 EC2 instance, AMI, instance type
An EC2 instance is a virtual server; the AMI is the template (OS +
baseline software) used to launch it; the instance type determines
its compute/memory capacity. None of these three choices affects
networking directly — that's a separate configuration layer.

### 3.2 Elastic IP allocation vs. association
Allocation reserves a static public IPv4 to your account
(`AllocationId`). Association connects that reserved address to a
specific resource (`AssociationId`, and the resource's `InstanceId`
appears in the address record). These are two separate, sequential
API actions.

### 3.3 Public IP (auto-assigned) vs. Elastic IP
An auto-assigned public IPv4 is ephemeral — it can change when an
instance stops and restarts. An Elastic IP is a static address that
persists under your account's control until explicitly released,
independent of the instance's running state.

### 3.4 `Name` tags as filtering convenience, not identity
AWS resources are uniquely identified by their own IDs
(`InstanceId`, `AllocationId`, etc.). `Name` tags exist for
human-readability and CLI `--filters` convenience; they carry no
uniqueness guarantee from AWS itself.

### 3.5 Bidirectional verification
Confirming a relationship (like an EIP-to-instance association) from
both participating resources' own API views, rather than trusting a
single side's response, guards against stale or partially-propagated
state.

---

## 4. Runbook

### 4.1 Launch the EC2 instance (Console)
```text
EC2 → Instances → Launch instances
  → Name: devops-ec2
  → AMI: Ubuntu or Amazon Linux
  → Instance type: t2.micro
  → Configure key pair / network settings per lab requirements
  → Launch instance
  → Wait for State: running
```

### 4.2 Allocate and tag the Elastic IP (Console)
```text
EC2 → Network & Security → Elastic IP addresses → Allocate Elastic IP address
  → Keep default Amazon IPv4 pool → Allocate
  → Select the new address → Actions → Manage tags
  → Add tag: Name = devops-eip → Save
```

### 4.3 Associate the Elastic IP (Console)
```text
Select the Elastic IP → Actions → Associate Elastic IP address
  → Resource type: Instance
  → Instance: devops-ec2
  → Associate
```

### 4.4 Verify the EC2 instance (CLI)
```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-ec2" \
  --query 'Reservations[].Instances[].{Name:Tags[?Key==`Name`]|[0].Value,InstanceId:InstanceId,State:State.Name,Type:InstanceType,PublicIP:PublicIpAddress,PrivateIP:PrivateIpAddress}' \
  --output table
```
```text
Name       devops-ec2
InstanceId i-0c49c05fa521f523c
State      running
Type       t2.micro
PublicIP   34.204.146.196
PrivateIP  172.31.32.190
```

### 4.5 Verify the Elastic IP allocation and association (CLI)
```bash
aws ec2 describe-addresses \
  --query 'Addresses[].{Name:Tags[?Key==`Name`]|[0].Value,PublicIP:PublicIp,AllocationId:AllocationId,InstanceId:InstanceId,AssociationId:AssociationId}' \
  --output table
```
```text
Name           devops-eip
PublicIP       34.204.146.196
AllocationId   eipalloc-0e0881583ae238cc0
InstanceId     i-0c49c05fa521f523c
AssociationId  eipassoc-0a254ea0f7d8411ac
```

### 4.6 Verify the specific Elastic IP, filtered by tag (CLI)
```bash
aws ec2 describe-addresses \
  --filters "Name=tag:Name,Values=devops-eip" \
  --query 'Addresses[].{Name:Tags[?Key==`Name`]|[0].Value,PublicIP:PublicIp,InstanceId:InstanceId,AssociationId:AssociationId}' \
  --output table
```
Confirm `InstanceId` matches §4.4's instance, and `AssociationId` is
populated (not empty).

### 4.7 Verify the association from the instance side (CLI)
```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-ec2" \
  --query 'Reservations[].Instances[].{InstanceId:InstanceId,State:State.Name,Type:InstanceType,ElasticIP:NetworkInterfaces[].Association.PublicIp}' \
  --output table
```
```text
34.204.146.196
```
Matches §4.5/§4.6's `PublicIP` — the bidirectional cross-check (§2.4).

---

## 5. Final state

| Requirement | Result |
|---|---|
| EC2 instance `devops-ec2` running, `t2.micro` | ✅ |
| Elastic IP allocated and tagged `devops-eip` | ✅ |
| Elastic IP associated with `devops-ec2` | ✅ |
| `AssociationId` populated on the address record | ✅ |
| Instance's `NetworkInterfaces[].Association.PublicIp` matches the EIP | ✅ |

```text
AWS Region
    └── devops-ec2 (i-0c49c05fa521f523c, t2.micro, running)
           ├── Private IP: 172.31.32.190
           └── Elastic IP: 34.204.146.196
                  ├── AllocationId: eipalloc-0e0881583ae238cc0
                  └── AssociationId: eipassoc-0a254ea0f7d8411ac
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Instance query returns no results | Wrong Region, missing tag, or wrong account | Check Console Region and instance tags; `aws sts get-caller-identity` |
| Instance not `running` | Still pending, stopped, or failed to launch | Check instance state and status checks |
| Elastic IP query returns no results | Not allocated in the current Region, or `Name` tag missing | List all addresses untagged first to confirm allocation happened |
| `InstanceId` empty in `describe-addresses` | Elastic IP allocated but not yet associated | Associate it with `devops-ec2` |
| Instance public IP differs from the Elastic IP | Association incomplete, or the wrong address was attached | Re-check both `describe-addresses` and `describe-instances` |
| SSH/HTTP connection fails despite correct association | Security group, route table, or application/service issue — not an EIP problem | Check security group rules, route table, instance health, and service status (§2.7) |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.8) out loud
      from "launch `devops-ec2`" to "verified bidirectionally."
- [ ] Explain, in one sentence, why allocating an Elastic IP and
      associating it with an instance are two separate API actions,
      not one.
- [ ] Explain why checking only `describe-addresses` (and not also
      `describe-instances`) would be an incomplete verification.
- [ ] Explain why an Elastic IP's `Name` tag is not the same thing as
      its `AllocationId`.
- [ ] Given a correctly-associated Elastic IP that still can't be
      reached over HTTP, name three non-EIP layers you'd check next.
- [ ] From memory, launch a second instance, allocate and associate a
      second Elastic IP, then release it and confirm via CLI that it
      no longer appears in `describe-addresses` — without copying any
      command from this file.

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
