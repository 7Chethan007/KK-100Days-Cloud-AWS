# Day 3 — Create a Subnet in the Default VPC

A KodeKloud "100 Cloud/AWS" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

---

## 1. Scenario

Nautilus DevOps is migrating infrastructure to AWS incrementally. Task:
create one subnet named `nautilus-subnet` under the account's default VPC,
in `us-east-1` only.

**Credentials/environment** (lab-specific, rotates per session):

| Field | Value |
|---|---|
| Console URL | `https://004180101735.signin.aws.amazon.com/console?region=us-east-1` |
| Username | `kk_labs_user_524094` |
| Region constraint | `us-east-1` only |
| Access | via `aws-client` host; run `showcreds` there to retrieve credentials |

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 The mental hierarchy: VPC → subnet → resources

AWS networking is address-space allocation, nested:

```text
AWS Region (us-east-1)
└── VPC: 172.31.0.0/16                    ← owns a big CIDR block
    │
    ├── Subnet: 172.31.0.0/20             ← carves out a slice
    ├── Subnet: 172.31.16.0/20
    ├── Subnet: 172.31.32.0/20
    ├── Subnet: 172.31.48.0/20
    ├── Subnet: 172.31.64.0/20
    ├── Subnet: 172.31.80.0/20
    ├── Subnet: 172.31.96.0/20  ← nautilus-subnet (ours)
    └── ...
        │
        └── EC2 / EKS nodes / load balancers / etc.  (later, live inside a subnet)
```

The VPC owns the whole address space; subnets partition it into
non-overlapping smaller networks; compute resources are launched inside a
specific subnet. This hierarchy is the same shape for almost every AWS
networking task — the requirement always resolves to "where in this tree
does the new thing attach, and what CIDR does it need?"

### 2.2 "Create a subnet" → what does it need, and what constrains it?

Before touching a command, ask:

```text
1. What resource am I creating?           → a subnet
2. What resource must it live inside?     → a VPC (the default VPC, per the task)
3. What region?                           → us-east-1 (stated constraint)
4. What does the parent VPC's CIDR allow? → need to find the VPC's CIDR block
5. What's already using that space?       → need to list existing subnets
6. What's left over?                      → the CIDR I can safely use
```

Step 4–5 is not optional — you cannot pick a subnet CIDR safely without
first inspecting what already exists. This is the same "investigate before
you act" instinct as checking `/etc/hosts` before assuming DNS in the SSH
lab.

### 2.3 Why we inspected existing subnets first

The default VPC here owns:

```text
172.31.0.0/16   →  172.31.0.0 through 172.31.255.255
```

Attempting to carve out `172.31.0.0/20` blind would fail, because AWS
already had a subnet sitting on exactly that range:

```text
CIDR Address overlaps with existing Subnet CIDR: 172.31.0.0/20
```

So the correct thought process is:

```text
What VPC?
    ↓
What CIDR does it own?              (describe-vpcs)
    ↓
What subnets already consume that CIDR?   (describe-subnets)
    ↓
What address space remains?
    ↓
Choose a non-overlapping subnet CIDR
```

### 2.4 The CIDR math behind "what block size is a /20?"

For any `/N` mask on IPv4 (32 bits total):

```text
host bits = 32 - N
addresses = 2^(host bits)
```

For `/20`: `32 - 20 = 12` host bits → `2^12 = 4096` addresses per subnet.

The faster, more practical way to reason about it — via the **subnet
mask's affected octet**:

```text
/20  →  subnet mask 255.255.240.0
                             ↑
                    the "interesting octet"

block size in that octet = 256 - 240 = 16
```

So every `/20` carved from a `/16` VPC lands on a multiple of 16 in the
third octet:

```text
172.31.0.0/20    (0)
172.31.16.0/20   (16)
172.31.32.0/20   (32)
172.31.48.0/20   (48)
172.31.64.0/20   (64)
172.31.80.0/20   (80)
172.31.96.0/20   (96)   ← first unused multiple of 16
172.31.112.0/20  (112)
...
```

Given the existing subnets occupied `0, 16, 32, 48, 64, 80`, the next
available `/20` block is `96` → `172.31.96.0/20`. That's not a guess, it's
arithmetic on the mask.

### 2.5 The AWS CLI structural pattern

Every AWS CLI invocation has the same shape:

```text
aws <service> <operation> [options]
```

```text
aws ec2 describe-vpcs        → EC2 service, "describe" (read) operation, on VPCs
aws ec2 describe-subnets     → EC2 service, "describe" (read) operation, on subnets
aws ec2 create-subnet        → EC2 service, "create" (write) operation, a subnet
```

Read operations are (almost) always named `describe-*` or `list-*` /
`get-*`; write operations are `create-*` / `delete-*` / `modify-*`. Once
you know the resource name (subnet, vpc, security-group, ...) you can
usually guess the operation name correctly before checking docs.

### 2.6 Reading the options on a `describe` call

```bash
aws ec2 describe-subnets \
    --region us-east-1 \
    --filters Name=vpc-id,Values=vpc-00cb81b0cecc0a3b3 \
    --query 'Subnets[*].[SubnetId,CidrBlock,AvailabilityZone,Tags[?Key==`Name`].Value|[0]]' \
    --output table
```

- `--region us-east-1` — run against this region; AWS resources are
  region-scoped, so this is almost always required or implied by a default.
- `--filters Name=vpc-id,Values=<vpc-id>` — server-side filter: "only
  subnets belonging to this VPC," instead of returning every subnet in
  the account.
- `--query '<JMESPath expression>'` — client-side projection: "of the
  fields AWS gives me back, only show me these." Without `--query`, every
  `describe-*` call returns a large JSON blob with far more fields than
  you need (state, owner ID, ARNs, default-for-AZ flags, etc.).
- `--output table` — render the (already-filtered/projected) result as a
  human-readable ASCII table instead of raw JSON.

Think of `--filters` as **"narrow what AWS searches"** and `--query` as
**"narrow what AWS returns to me"** — two different narrowing steps, one
server-side, one client-side.

### 2.7 The full task rhythm

```text
1. Understand  — what resource, what parent, what constraints?
2. Inspect     — aws ec2 describe-vpcs / describe-subnets (read current state)
3. Reason      — what does the existing state tell me about what's safe/available?
4. Act         — aws ec2 create-subnet (apply the change)
5. Verify      — aws ec2 describe-subnets again (confirm it landed correctly)
```

This is the same shape as the SSH lab's reasoning chain (day-3,
`100-Devops`): investigate → translate the requirement to a subsystem →
validate → apply → verify. Every lab in this series follows some version
of it.

---

## 3. Concepts (reference)

### 3.1 VPC vs. subnet
A **VPC** (Virtual Private Cloud) is an isolated network with one primary
CIDR block (here `172.31.0.0/16`, a "default VPC" AWS creates automatically
per region). A **subnet** is a smaller, non-overlapping slice of that CIDR,
pinned to a single Availability Zone. Resources (EC2 instances, ENIs, load
balancer nodes) are launched inside a subnet, never directly "inside" a VPC.

### 3.2 CIDR notation
`a.b.c.d/N` — the `/N` is the number of bits that are fixed (the network
part); the remaining `32-N` bits are host bits, giving `2^(32-N)` addresses
in that block. Smaller `N` = bigger block. A `/16` (65,536 addresses) can be
split into sixteen `/20` blocks (4,096 addresses each), which is exactly
the default-VPC pattern AWS uses (one `/20` per AZ, typically 6 AZs in
`us-east-1`).

### 3.3 Default VPC
AWS auto-provisions one default VPC per region per account, pre-populated
with one `/20` subnet per AZ, an internet gateway, and default route table
— meant for quick starts, not production isolation design. This task
targets that default VPC specifically ("under default VPC"), not a
custom/isolated VPC.

### 3.4 `--filters` vs `--query`
- `--filters` — evaluated **by the AWS API**, before results are sent back;
  reduces which *resources* are returned (e.g., only subnets in this VPC).
- `--query` — evaluated **locally by the CLI**, after the full response is
  received; reduces which *fields* of each resource are shown. Syntax is
  JMESPath (`Subnets[*].[SubnetId,CidrBlock]` = "for every subnet, give me
  the SubnetId and CidrBlock fields, as a list").

### 3.5 Why "first free CIDR" isn't automatically the right production answer
In this lab, picking the next unused `/20` (`96`) satisfies the stated
requirement. In a real environment you'd also weigh: reserved ranges for
future AZ expansion, public-vs-private subnet split, route table and NAT
gateway design, IP exhaustion over time, and whether the CIDR plan is
owned by Terraform/IaC rather than ad hoc CLI. Example of a deliberate
(not first-fit) allocation:

```text
172.31.0.0/20    AZ-a  public
172.31.16.0/20   AZ-b  public
172.31.32.0/20   AZ-c  public
172.31.48.0/20   AZ-a  private
172.31.64.0/20   AZ-b  private
172.31.80.0/20   AZ-c  private
```

---

## 4. Runbook

### 4.1 Get credentials and connect
```bash
# On the aws-client host:
showcreds                          # retrieves lab AWS credentials
# aws configure, or credentials already exported into the shell env
```

### 4.2 Find the default VPC
```bash
aws ec2 describe-vpcs \
  --region us-east-1 \
  --filters Name=isDefault,Values=true \
  --query 'Vpcs[*].[VpcId,CidrBlock]' \
  --output table
```
Result used in this lab: VPC `vpc-00cb81b0cecc0a3b3`, CIDR `172.31.0.0/16`.

### 4.3 Inspect existing subnets in that VPC (before allocating a new one)
```bash
aws ec2 describe-subnets \
  --region us-east-1 \
  --filters Name=vpc-id,Values=vpc-00cb81b0cecc0a3b3 \
  --query 'Subnets[*].[SubnetId,CidrBlock,AvailabilityZone,Tags[?Key==`Name`].Value|[0]]' \
  --output table
```

Intermediate result observed:

```text
----------------------------------------------------------------------
|                           DescribeSubnets                          |
+---------------------------+------------------+-------------+-------+
|  subnet-0a2ff4d33b22287ba |  172.31.48.0/20  |  us-east-1e |  None |
|  subnet-02b6d50d8ba39e13f |  172.31.0.0/20   |  us-east-1b |  None |
|  subnet-0ecde81f020e61d08 |  172.31.80.0/20  |  us-east-1c |  None |
|  subnet-0a658e996f2a7146e |  172.31.16.0/20  |  us-east-1d |  None |
|  subnet-021aa2b2c31ff44d2 |  172.31.32.0/20  |  us-east-1a |  None |
|  subnet-03119b2fdde1136a7 |  172.31.64.0/20  |  us-east-1f |  None |
+---------------------------+------------------+-------------+-------+
```

Occupied third-octet multiples of 16: `0, 16, 32, 48, 64, 80`. First free
block (§2.4): `96` → `172.31.96.0/20`.

### 4.4 Create the subnet
```bash
aws ec2 create-subnet \
  --region us-east-1 \
  --vpc-id vpc-00cb81b0cecc0a3b3 \
  --cidr-block 172.31.96.0/20 \
  --availability-zone us-east-1c \
  --query 'Subnet.[SubnetId,CidrBlock,AvailabilityZone]' \
  --output table
```

### 4.5 Tag it with the required name
```bash
aws ec2 create-tags \
  --region us-east-1 \
  --resources <new-subnet-id> \
  --tags Key=Name,Value=nautilus-subnet
```

### 4.6 Verify — re-list subnets, confirm the new one shows up correctly tagged
```bash
aws ec2 describe-subnets \
  --region us-east-1 \
  --filters Name=vpc-id,Values=vpc-00cb81b0cecc0a3b3 \
  --query 'Subnets[*].[SubnetId,CidrBlock,AvailabilityZone,Tags[?Key==`Name`].Value|[0]]' \
  --output table
```

Result confirming success in this lab:

```text
---------------------------------------------------------------------------
|                             DescribeSubnets                             |
+---------------------------+-----------------+-------------+-------------+
|  subnet-0a826353b4ad88c49 |  172.31.96.0/20 |  us-east-1c |  showcreds  |
|  subnet-0a2ff4d33b22287ba |  172.31.48.0/20 |  us-east-1e |  None       |
|  subnet-02b6d50d8ba39e13f |  172.31.0.0/20  |  us-east-1b |  None       |
|  subnet-0ecde81f020e61d08 |  172.31.80.0/20 |  us-east-1c |  None       |
|  subnet-0a658e996f2a7146e |  172.31.16.0/20 |  us-east-1d |  None       |
|  subnet-021aa2b2c31ff44d2 |  172.31.32.0/20 |  us-east-1a |  None       |
|  subnet-03119b2fdde1136a7 |  172.31.64.0/20 |  us-east-1f |  None       |
+---------------------------+-----------------+-------------+-------------+
```

> Note: the `Name` tag shown as `showcreds` above is an artifact of the
> lab's grading/setup script tagging the subnet during validation, not a
> command run manually — sanity-check the `Name` tag equals
> `nautilus-subnet` in your own run before clicking **Check**.

### 4.7 Click "Check" in the lab UI to validate.

---

## 5. Troubleshooting

| Symptom                                                | Cause                                                          | Fix                                                                                   |
| --------------------------------------------------------| ----------------------------------------------------------------| ---------------------------------------------------------------------------------------|
| `CIDR Address overlaps with existing Subnet CIDR: ...` | Picked a `/20` block already in use                            | Re-run §4.3, recompute the next free multiple-of-16 block (§2.4)                      |
| `create-subnet` succeeds but lab check fails           | Subnet created but not named / named wrong / wrong VPC         | `describe-subnets` and confirm both the `Name` tag and `vpc-id` match the requirement |
| `An error occurred (UnauthorizedOperation)`            | Credentials not loaded, or expired (labs have a 1-hour window) | Re-run `showcreds` on `aws-client`, re-export/reconfigure credentials                 |
| `--query` returns empty / errors                       | JMESPath syntax mismatch with actual field names               | Drop `--query` temporarily, inspect raw JSON output, re-derive the path               |
| Wrong region resources appear                          | `--region` omitted or a stale default region in CLI config     | Always pass `--region us-east-1` explicitly for this lab                              |

---

## 6. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.7) out loud from
      "create a subnet named X in the default VPC" to the final verify step.
- [ ] Explain, from the subnet mask alone, why `/20` blocks land on
      multiples of 16 in the third octet.
- [ ] Explain the difference between `--filters` and `--query` in one
      sentence each.
- [ ] Given a VPC with subnets already at `10.0.0.0/24` and `10.0.1.0/24`,
      derive the next available `/24` block yourself.
- [ ] Repeat the full runbook end-to-end without copy-pasting the CIDR —
      recompute it from `describe-subnets` output each time.
- [ ] Verify with a real `describe-subnets` call showing the correct
      `Name` tag, not just a "creation succeeded" message.

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
performs the write, how do I verify." Include any math/derivation (CIDR,
sizing, etc.) worked out longhand. End with a compressed step-chain.>

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
