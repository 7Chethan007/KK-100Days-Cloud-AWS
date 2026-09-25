# Day 6 — Launch an EC2 Instance

A KodeKloud "100 Cloud/AWS" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each component exists and *how you'd derive the
launch parameters yourself*.

> **Note on this write-up:** unlike prior days in this series, this
> runbook was authored from the task spec and general AWS behavior
> rather than a captured terminal transcript with exact IDs and output.
> Treat IDs like `ami-0fef201115eefe936` below as illustrative of the
> *shape* of real output — your actual AMI ID, instance ID, and IP will
> differ per session. Verify against your own `describe-*` output before
> trusting any specific value here.

---

## 1. Scenario

The Nautilus DevOps team is migrating infrastructure to AWS in small,
controlled increments. Task: launch an EC2 instance with:

| Requirement | Value |
|---|---|
| Name | `nautilus-ec2` |
| AMI | Amazon Linux (Amazon Linux 2023) |
| Instance type | `t2.micro` |
| Key pair | new RSA key pair named `nautilus-kp` |
| Security group | the default (available-by-default) security group |

This lab looks like "launch one instance," but it exercises six
independent AWS concepts stacked on top of each other — the goal is
understanding what each layer does, not memorizing the console wizard's
click-path.

---

## 2. Reasoning model — how to *derive* the launch parameters, not memorize them

### 2.1 The six questions every EC2 launch answers

```text
1. WHAT am I launching?            → AMI
2. HOW POWERFUL should it be?      → Instance type
3. WHERE should it live?           → VPC + Subnet
4. WHAT NETWORK TRAFFIC is allowed? → Security Group
5. WHO can log into it?            → Key Pair
6. WHAT NAME identifies it?        → Tags
```

Every EC2 launch — console or CLI — is answering these same six
questions. Once you can answer them for a given task, the actual
`run-instances` call (or console form) is just transcription.

### 2.2 AMI — the template, not the machine

**AMI = Amazon Machine Image.** It's a template containing an operating
system, boot configuration, and root filesystem — EC2 boots a fresh
instance *from* it, the same way a VM is created from a disk image:

```text
AMI (template)
Amazon Linux 2023
       │
       │ launch
       ▼
EC2 instance (a running copy)
```

```text
AMI       = blueprint / template
EC2       = the actual running machine
```

This distinction matters practically the moment you need more than one
identical server: instead of manually installing an OS and dependencies
N times, you launch N instances from **one** AMI (or a custom "golden"
AMI you've baked with your own software preinstalled) — this is the
foundation autoscaling groups and immutable-infrastructure deployments
build on.

### 2.3 Instance type — the hardware, independent of the OS

`t2.micro` specifies **virtual hardware** (CPU allocation, memory,
network baseline) — completely independent of which AMI you chose:

```text
AMI        → what OS/software is on the disk?
Instance type → what CPU/RAM/network does it run on?
```

"Amazon Linux + t2.micro" reads as: *"boot this OS image on this class
of virtual hardware."* Same AMI can be launched on `t2.micro`,
`t3.large`, or `m5.xlarge` — the OS doesn't change, only the resources
backing it.

### 2.4 VPC and subnet — where the instance lives on the network

```text
                VPC
        172.31.0.0/16
              │
       ┌──────┴──────┐
       │             │
    Subnet A      Subnet B
       │             │
     EC2           EC2
```

A **VPC** is your private network inside AWS; a **subnet** is a smaller,
zone-pinned slice of that network (exactly the CIDR-carving exercise from
Day 3's subnet lab — an instance launched here lands in one of those
existing `/20` blocks, e.g. `172.31.96.0/20`). The task doesn't specify a
custom VPC/subnet, so the default VPC's default subnet is the correct,
unstated-but-implied choice — same reasoning as accepting AWS defaults
whenever a lab doesn't override them.

```text
AWS Region
   │
   ▼
  VPC
   │
   ▼
 Subnet
   │
   ▼
  EC2
```

### 2.5 Security group — the stateful firewall in front of the instance

A **Security Group** controls what network traffic reaches the
instance — a virtual firewall attached to the instance's network
interface, not to the subnet:

```text
Internet
   │
   │ TCP 22
   ▼
Security Group
   │  allow SSH, port 22?
   ▼
  EC2
```

Critically, it's **stateful**: allowing an inbound connection
automatically allows the matching response traffic back out, with no
separate outbound rule needed for replies:

```text
Client → SSH request → Security Group (ALLOW 22) → EC2
Client ←────────────── response flows back automatically ──┘
```

The task says to attach "the default (available by default) security
group" — every VPC ships with one named literally `default` — so this
requirement is "use what's already there," not "create a new one."

**Security group ≠ subnet.** A subnet answers "where does this instance
live on the network" (an address-space question); a security group
answers "what traffic can reach it" (a firewall-rules question) — two
independent, layered concerns, not two names for the same thing.

**Security group ≠ your Linux firewall.** A security group operates at
the AWS network layer, in front of the instance; `iptables`/`nftables`
inside the OS is a second, independent layer behind it:

```text
AWS network → Security Group → EC2 network interface → Linux → iptables/nftables → Application
```

Traffic must clear *both* layers — a common troubleshooting trap is
fixing one and forgetting the other still blocks it.

### 2.6 Key pair — who's allowed to authenticate

An EC2 key pair is a public/private key pair used for SSH
authentication, not a password:

```text
Your computer                      EC2 instance
     │                                  │
  private key ──── cryptographic ──── public key
                    proof                (in ~/.ssh/authorized_keys)
```

**RSA** is the underlying public-key algorithm — the public key can be
freely shared (it's baked into the instance at launch), the private key
must stay secret (only you ever hold it; AWS lets you download it
exactly once at creation). This avoids ever sending a password over the
network: possessing the private key *is* the proof of identity.

`.pem` vs `.ppk` is a common point of confusion worth heading off:
they're **file formats for the same kind of key**, tied to different
tooling (`.pem` for OpenSSH/most Linux and Mac clients; `.ppk` for
PuTTY on Windows) — not two different cryptographic algorithms. An RSA
key can be represented in either format; converting between them (e.g.
via `puttygen`) doesn't change the underlying key.

### 2.7 Why `chmod 400` on a downloaded `.pem` file

```bash
chmod 400 nautilus-kp.pem
```

```text
400 → owner: read only; group: nothing; others: nothing
```

The private key is a credential, not an ordinary file — if it's group-
or world-readable, any other local user could read it, and OpenSSH
itself typically refuses to use a private key file whose permissions are
that loose (`Permissions 0644 for 'nautilus-kp.pem' are too open`). This
is the same "credential file, not ordinary data" reasoning as
Day-4-DevOps's `.env` handling — anything that authenticates you gets
the tightest permission that still lets *you* read it.

### 2.8 How SSH actually reaches the instance — every layer has to work

```bash
ssh -i nautilus-kp.pem ec2-user@<public-ip>
```

```text
ssh
 ├── -i nautilus-kp.pem   → the private key to authenticate with
 ├── ec2-user             → the Linux username baked into Amazon Linux AMIs
 └── <public-ip>          → the instance's public address
```

But this only works if every layer between you and the instance is
correctly configured:

```text
Internet
   │
Security Group (TCP/22 allowed?)
   │
EC2 network interface
   │
SSH daemon listening on port 22
   │
Public-key authentication (your private key ↔ the key pair's public key)
   │
Shell
```

"SSH doesn't work" is never a single-cause problem — it requires walking
this chain layer by layer (§2.9), not guessing.

### 2.9 The compressed reasoning chain

```text
Requirement (nautilus-ec2, Amazon Linux, t2.micro, nautilus-kp RSA, default SG)
   → Choose AMI                    → Amazon Linux 2023, x86_64
   → Choose instance type          → t2.micro
   → Accept default VPC/subnet     → no custom networking specified
   → Create RSA key pair           → nautilus-kp (download the .pem exactly once)
   → Attach default security group → the pre-existing "default" SG, not a new one
   → Tag Name=nautilus-ec2
   → Launch
   → Verify: describe-instances, describe-images, describe-key-pairs
        (don't trust the console wizard's "success" screen alone)
```

---

## 3. Concepts (reference)

### 3.1 The full AWS hierarchy this lab sits inside
```text
AWS
│
├── Region
│    └── Availability Zone
│           └── Subnet
│                  └── EC2
│
└── VPC
      ├── Subnets
      ├── Route Tables
      ├── Internet Gateway
      ├── NAT Gateway
      └── Security Groups
```
An EC2 instance is the leaf of this tree — every layer above it
(region → AZ → subnet → VPC's routing/gateways) has to be correctly
configured for the instance to be reachable at all.

### 3.2 Security Group vs. Network ACL
Two different, layered network-control mechanisms you'll meet as you go
further into VPC networking:

| | Security Group | Network ACL |
|---|---|---|
| Applies to | the resource (ENI) | the subnet |
| Stateful | Yes (return traffic auto-allowed) | No (must allow both directions explicitly) |
| Rule types | Allow only | Allow **and** Deny |
| Typical use | per-instance traffic control | subnet-level boundary |

Don't memorize every detail yet — just retain that SG and NACL are not
the same mechanism, and both can independently block traffic.

### 3.3 Golden AMIs and immutable infrastructure
A "golden AMI" is a custom image baked with your application, its
dependencies, and configuration already installed — so launching N
servers means launching N instances from one known-good image, rather
than configuring each instance after boot. This underpins autoscaling
groups (which launch new instances from a fixed AMI on demand) and the
broader "immutable infrastructure" pattern (replace instances rather
than patch them in place).

### 3.4 Stateful firewalls and why you rarely write outbound SSH rules
Because security groups are stateful, an allowed inbound connection's
response traffic is automatically permitted — you write an inbound rule
for SSH (port 22) and never need a matching outbound rule just for
replies. This differs from a traditional stateless ACL, where inbound
and outbound must each be explicitly allowed.

### 3.5 Layer-by-layer debugging for "instance unreachable"
When `http://<ec2-ip>:8080` (or SSH) doesn't respond, trace the whole
path rather than guessing:
```text
User → Internet → Route → Internet Gateway → VPC → Subnet
     → Security Group → EC2 → OS firewall → Application → Application port
```
This same layer-by-layer instinct transfers directly to debugging
Kubernetes/EKS networking, Docker networking, and load balancers later
in this series.

---

## 4. Runbook

### 4.1 Identify current AWS identity and region (habit from prior labs)
```bash
aws sts get-caller-identity
aws configure get region
```

### 4.2 Find the Amazon Linux AMI to launch from
```bash
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-*-x86_64" "Name=state,Values=available" \
  --query 'sort_by(Images,&CreationDate)[-1].[ImageId,Name]' \
  --output table
```
Picks the most recently published Amazon Linux 2023 (x86_64) AMI owned
by Amazon — avoids hardcoding a specific AMI ID, which changes over
time and differs per region.

### 4.3 Confirm the default VPC and subnet exist (no custom networking required)
```bash
aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[*].[VpcId,CidrBlock]' \
  --output table

aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=<default-vpc-id>" \
  --query 'Subnets[*].[SubnetId,CidrBlock,AvailabilityZone]' \
  --output table
```

### 4.4 Confirm the default security group
```bash
aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=default" "Name=vpc-id,Values=<default-vpc-id>" \
  --query 'SecurityGroups[*].[GroupId,GroupName]' \
  --output table
```

### 4.5 Create the RSA key pair, saving the private key immediately
```bash
aws ec2 create-key-pair \
  --key-name nautilus-kp \
  --key-type rsa \
  --query 'KeyMaterial' \
  --output text > nautilus-kp.pem

chmod 400 nautilus-kp.pem
```
The private key material is only ever returned **once**, at creation —
if this step's output is lost, the key pair must be deleted and
recreated; AWS never lets you re-download an existing private key.

### 4.6 Launch the instance
```bash
aws ec2 run-instances \
  --image-id <ami-id-from-4.2> \
  --instance-type t2.micro \
  --key-name nautilus-kp \
  --security-group-ids <default-sg-id-from-4.4> \
  --subnet-id <subnet-id-from-4.3> \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=nautilus-ec2}]' \
  --query 'Instances[0].[InstanceId,State.Name]' \
  --output table
```

### 4.7 Verify — don't trust a console "launch successful" screen alone
```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=nautilus-ec2" \
  --query 'Reservations[].Instances[].{ID:InstanceId,Name:Tags[?Key==`Name`].Value|[0],State:State.Name,Type:InstanceType,AMI:ImageId,AZ:Placement.AvailabilityZone,Key:KeyName,SG:SecurityGroups[].GroupName}' \
  --output table
```
Expect `State: running` (after the initial `pending` — same "response
received ≠ resource ready" lesson as Day 5's EBS volume lab), `Type:
t2.micro`, `Key: nautilus-kp`, and `SG` including `default`.

Independently confirm the AMI and key pair too, rather than trusting a
single combined query:
```bash
aws ec2 describe-images \
  --image-ids <ami-id> \
  --query 'Images[0].{Name:Name,Architecture:Architecture,Platform:PlatformDetails}' \
  --output table

aws ec2 describe-key-pairs \
  --key-names nautilus-kp \
  --query 'KeyPairs[0].{Name:KeyName,Type:KeyType}' \
  --output table
```

### 4.8 (Optional) Confirm SSH reachability end to end
```bash
ssh -i nautilus-kp.pem ec2-user@<public-ip-from-4.7>
```
If this fails, walk §2.8/§3.5's layer chain rather than guessing at a
fix.

### 4.9 Click "Check" in the lab UI to validate.

---

## 5. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `run-instances` fails with an AMI/subnet/AZ mismatch | AMI is region-specific; a hardcoded AMI ID from a different region won't exist here | Re-run the `describe-images` lookup (§4.2) in the target region rather than reusing an ID from elsewhere |
| `Permissions 0644 for 'nautilus-kp.pem' are too open` | `.pem` file left at default, world-readable permissions | `chmod 400 nautilus-kp.pem` |
| Downloaded/generated `.pem` is gone or lost | AWS only returns private key material once, at creation | Delete the key pair and create a new one — there is no way to recover the original private key |
| `ssh` connects but hangs or times out | Security group doesn't allow inbound TCP/22 from your IP, or wrong public IP used | `describe-security-groups` to confirm the SSH rule; re-check the instance's current public IP |
| `ssh` reaches the host but authentication fails | Wrong username for the AMI (e.g. `ubuntu` vs `ec2-user`), or wrong/mismatched private key | Amazon Linux uses `ec2-user`; confirm the `.pem` matches the key pair name attached to the instance |
| Instance shows `running` but the app on it is unreachable | A layer above the OS (security group, route table, IGW) or the app's own port/firewall is blocking it | Walk the full chain in §3.5, layer by layer, rather than only checking the security group |

---

## 6. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.9) out loud
      from "launch an EC2 instance named X" to "verified via
      `describe-instances`."
- [ ] Explain, in one sentence, the difference between what an AMI
      determines and what an instance type determines.
- [ ] Explain why a security group is described as "stateful," and what
      that means for outbound SSH response traffic.
- [ ] Explain the difference between `.pem` and `.ppk`, and why it's a
      file-format distinction, not a cryptographic-algorithm one.
- [ ] Explain why `chmod 400` is required on a downloaded private key,
      using the same reasoning as a `.env` file's permissions.
- [ ] Launch a second instance from memory, choosing a fresh AMI lookup
      and a newly created key pair rather than reusing this session's
      values, then verify every field independently (AMI, type, key,
      security group) rather than trusting one combined query.

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
