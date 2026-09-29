# Day 10 — Attach an Elastic IP to an EC2 Instance

A KodeKloud "100 Cloud/AWS" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

---

## 1. Scenario

The Nautilus DevOps team has an existing EC2 instance and an existing
Elastic IP, both already created, that need to be connected. Task:
attach the Elastic IP `xfusion-ec2-eip` to the EC2 instance
`xfusion-ec2`, in `us-east-1`.

**Credentials/environment** (lab-specific, rotates per session): via
`showcreds` on the `aws-client` host, same as every prior lab in this
series.

Desired relationship:

```text
Elastic IP (xfusion-ec2-eip)
        │
        │ association
        ▼
EC2 instance (xfusion-ec2)
```

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 The networking layers between "the internet" and an instance

```text
Internet
    │
    ▼
Public IPv4 / Elastic IP
    │
    ▼
AWS networking layer
    │
    ▼
Private IPv4
    │
    ▼
EC2 network interface
    │
    ▼
EC2 instance
```

An instance's **private** IP (e.g. `172.31.38.29`) is only reachable
inside the VPC — the internet has no route to it directly. A **public**
address (auto-assigned or Elastic) is what actually makes the instance
reachable from outside, and AWS's networking layer handles the
translation between the two; nothing inside the instance's own OS
configures or even necessarily knows about this mapping (§2.8).

### 2.2 Auto-assigned public IP vs. Elastic IP — the distinction this whole lab is about

```text
Automatically assigned public IPv4        Elastic IP (EIP)
        │                                       │
   tied to the instance's                reserved against
   current lifecycle —                    YOUR AWS ACCOUNT,
   can change on stop/start               independent of any
   (Day 7's EC2 lesson)                    one instance's lifecycle
```

Before this lab's change, `xfusion-ec2` already had an auto-assigned
public IP (`54.88.238.188`) — that alone doesn't satisfy the requirement.
The task specifically wants the **Elastic IP** attached, which is a
different, more stable kind of address, held by the account and
explicitly associated (or re-associated) to whichever instance needs it.

### 2.3 Allocation vs. association — two genuinely separate operations

This is the single most important distinction in the lab:

```text
ALLOCATE  → obtain a static public IPv4 address for your AWS account
              (already done for xfusion-ec2-eip before this task starts)

ASSOCIATE → connect an already-allocated EIP to a specific resource
              (this is the actual task)
```

```text
AWS Account
     │
     └── allocate ──► Elastic IP exists, unattached to anything
                              │
                              └── associate ──► EIP ←→ EC2 instance
```

An EIP can exist, fully allocated and billed, while being attached to
**nothing** — this isn't a broken state, it's simply "allocated but not
yet associated," and it's exactly the starting state this lab's own
`describe-addresses` check reveals (§2.5).

### 2.4 Find both resources by tag first

```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=xfusion-ec2" \
  --query 'Reservations[].Instances[].{InstanceId:InstanceId,State:State.Name,PrivateIP:PrivateIpAddress,PublicIP:PublicIpAddress}' \
  --output table

aws ec2 describe-addresses \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=xfusion-ec2-eip" \
  --query 'Addresses[].{PublicIP:PublicIp,AllocationId:AllocationId,InstanceId:InstanceId}' \
  --output table
```

Same tag-lookup pattern as every prior EC2/EBS lab in this series
(Days 5–9) — resolve the human-assigned `Name` tag into the actual
identifiers (`InstanceId`, `AllocationId`) that the mutating API call
actually needs.

### 2.5 Reading the "before" state correctly — `InstanceId: None` is informative, not broken

```text
AllocationId   eipalloc-03804f2cd3a72ad04
InstanceId     None
PublicIP       54.162.142.57
```

`InstanceId: None` here is the direct confirmation of §2.3's allocation-
vs-association distinction: the EIP fully exists and has its own
identity (`eipalloc-...`), but currently belongs to no instance. This is
the correct, expected pre-task state — not evidence something is wrong
with the EIP.

### 2.6 The identifiers involved — four different things, easy to conflate

| Object | Identifier | Looks like |
|---|---|---|
| EC2 instance | Instance ID | `i-047e991048f29a1df` |
| Elastic IP | Allocation ID | `eipalloc-03804f2cd3a72ad04` |
| Elastic IP | Public IP (human-facing) | `54.162.142.57` |
| The *relationship* itself | Association ID | `eipassoc-02ca3647c2985fa6f` |

The association ID is worth calling out specifically — it's not an
identifier for either the instance or the IP, it identifies the *link
between them*, generated only once `associate-address` succeeds. Getting
these four confused is the most likely source of a wrong `--query` or a
wrong argument to a later command.

### 2.7 Associating the EIP

```bash
aws ec2 associate-address \
  --region us-east-1 \
  --instance-id i-047e991048f29a1df \
  --allocation-id eipalloc-03804f2cd3a72ad04
```

```json
{
    "AssociationId": "eipassoc-02ca3647c2985fa6f"
}
```

Getting an `AssociationId` back confirms AWS *accepted and performed*
the association — but per the "response ≠ verified state" discipline
running through this whole series (Days 5, 7, 8, 9), the response alone
is not the final verification step (§2.8).

### 2.8 Bidirectional verification — check the relationship from both ends

```text
EIP → EC2:  describe-addresses  → "which instance is this IP attached to?"
EC2 → EIP:  describe-instances  → "what public IP does this instance have now?"
```

```bash
aws ec2 describe-addresses \
  --region us-east-1 \
  --allocation-ids eipalloc-03804f2cd3a72ad04 \
  --query 'Addresses[].{ElasticIP:PublicIp,InstanceId:InstanceId,AssociationId:AssociationId}' \
  --output table

aws ec2 describe-instances \
  --region us-east-1 \
  --instance-ids i-047e991048f29a1df \
  --query 'Reservations[0].Instances[0].{State:State.Name,PrivateIP:PrivateIpAddress,PublicIP:PublicIpAddress}' \
  --output table
```

Checking the relationship from *both* resources independently is
meaningfully stronger evidence than checking either alone — if only one
side showed the expected value, that would itself be a sign something is
inconsistent (a caching delay, a wrong ID used somewhere). Agreement
from both directions is the actual proof of the desired end state.

### 2.9 An EIP changes reachability, not access control

```text
Internet ──► 54.162.142.57 ──► EC2 instance
                                     │
                              still gated by its
                              SECURITY GROUP rules
```

Attaching an Elastic IP makes the instance reachable *at that address* —
it says nothing about which ports/protocols are actually allowed through.
SSH (22), HTTP (80), HTTPS (443), or any other service still needs an
explicit security group rule permitting it, exactly as covered for EC2
launches in Day 6. An EIP is an address, not a firewall exception.

### 2.10 The compressed reasoning chain

```text
Requirement (attach xfusion-ec2-eip to xfusion-ec2)
   → Find the instance by Name tag        → i-047e991048f29a1df
   → Find the EIP by Name tag              → eipalloc-03804f2cd3a72ad04, PublicIP 54.162.142.57
   → Inspect current EIP state             → InstanceId: None (allocated, NOT associated — §2.3)
   → Associate: associate-address --instance-id ... --allocation-id ...
   → Capture the AssociationId returned    → eipassoc-02ca3647c2985fa6f
   → Verify EIP → EC2: describe-addresses  → InstanceId now matches
   → Verify EC2 → EIP: describe-instances  → PublicIP now matches the Elastic IP
   → Both directions agree → association confirmed
```

---

## 3. Concepts (reference)

### 3.1 Elastic IP (EIP)
A static, account-owned public IPv4 address that can be associated with
(and re-associated to a different) EC2 instance or network interface,
independent of any one instance's own lifecycle. Distinct from an
auto-assigned public IP, which is tied to the instance and can change
across a stop/start cycle (the same caveat flagged in Day 7's instance-
type-change lab).

### 3.2 Why EIPs matter operationally
Any external system hardcoding an instance's address (DNS records,
firewall allowlists, partner integrations, monitoring configs) breaks if
that address silently changes. An EIP gives you one stable address you
can move between instances as needed — useful for bastion hosts, NAT
gateways, and legacy systems that expect a fixed address; modern
architectures more often front instances with a load balancer and DNS
instead, but EIPs remain common in exactly those specific use cases.

### 3.3 Allocation vs. association (recap)
Allocation obtains the address for your account. Association attaches
that already-allocated address to a specific resource. They're separate
API calls (`allocate-address` vs. `associate-address`), separate billing
considerations, and separate points of failure — an EIP can be correctly
allocated while association is still missing, which is exactly this
lab's starting state.

### 3.4 Where the EIP mapping actually lives
The public-to-private address translation is performed by AWS's
networking layer (ultimately tied to the instance's Elastic Network
Interface), not configured inside the instance's own operating system.
You never SSH in and edit a network config file to "set" the Elastic
IP — this is worth knowing before assuming a connectivity issue is an
in-instance networking problem.

### 3.5 EIP ≠ open access
Reachability (does an address route to this instance) and authorization
(is this specific traffic allowed through) are separate AWS mechanisms —
an EIP handles the former; security groups handle the latter. Both must
be correctly configured for an application to actually be reachable.

### 3.6 Resource lifecycle hygiene
An allocated-but-unassociated EIP still exists as a billed resource.
Real environments should periodically review for unused EIPs (along with
unused EBS volumes, snapshots, load balancers, NAT gateways) rather than
letting them accumulate silently.

---

## 4. Runbook

### 4.1 Identify the EC2 instance
```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=xfusion-ec2" \
  --query 'Reservations[].Instances[].{InstanceId:InstanceId,State:State.Name,PrivateIP:PrivateIpAddress,PublicIP:PublicIpAddress}' \
  --output table
```
```text
InstanceId  i-047e991048f29a1df
PrivateIP   172.31.38.29
PublicIP    54.88.238.188
State       running
```

### 4.2 Identify the Elastic IP
```bash
aws ec2 describe-addresses \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=xfusion-ec2-eip" \
  --query 'Addresses[].{PublicIP:PublicIp,AllocationId:AllocationId,InstanceId:InstanceId}' \
  --output table
```
```text
AllocationId   eipalloc-03804f2cd3a72ad04
InstanceId     None
PublicIP       54.162.142.57
```
Confirms: allocated, not yet associated (§2.5).

### 4.3 Associate the Elastic IP with the instance
```bash
aws ec2 associate-address \
  --region us-east-1 \
  --instance-id i-047e991048f29a1df \
  --allocation-id eipalloc-03804f2cd3a72ad04
```
```json
{
    "AssociationId": "eipassoc-02ca3647c2985fa6f"
}
```

### 4.4 Verify — from the EIP's side
```bash
aws ec2 describe-addresses \
  --region us-east-1 \
  --allocation-ids eipalloc-03804f2cd3a72ad04 \
  --query 'Addresses[].{ElasticIP:PublicIp,InstanceId:InstanceId,AssociationId:AssociationId}' \
  --output table
```
```text
AssociationId  eipassoc-02ca3647c2985fa6f
ElasticIP      54.162.142.57
InstanceId     i-047e991048f29a1df
```

### 4.5 Verify — from the EC2 instance's side
```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --instance-ids i-047e991048f29a1df \
  --query 'Reservations[0].Instances[0].{State:State.Name,PrivateIP:PrivateIpAddress,PublicIP:PublicIpAddress}' \
  --output table
```
```text
State       running
PrivateIP   172.31.38.29
PublicIP    54.162.142.57
```
Both directions agree — the instance's public IP now matches the
Elastic IP.

### 4.6 (Equivalent Console path, for reference)
```text
EC2 → Instances → xfusion-ec2       (confirm instance/instance ID)
EC2 → Network & Security → Elastic IPs → xfusion-ec2-eip
    → Actions → Associate Elastic IP address
    → Resource type: Instance → xfusion-ec2 → Associate
```
Then re-check both **Elastic IPs** (association shows the instance ID)
and **Instances → xfusion-ec2** (public IPv4 now matches the EIP) — same
bidirectional verification as the CLI path.

### 4.7 Click "Check" in the lab UI to validate.

---

## 5. Final state

| Requirement | Result |
|---|---|
| `xfusion-ec2` located | ✅ `i-047e991048f29a1df` |
| `xfusion-ec2-eip` located | ✅ `eipalloc-03804f2cd3a72ad04` / `54.162.142.57` |
| EIP associated with the instance | ✅ `eipassoc-02ca3647c2985fa6f` |
| Verified from the EIP side | ✅ `InstanceId: i-047e991048f29a1df` |
| Verified from the EC2 side | ✅ `PublicIP: 54.162.142.57` |

```text
                    AWS ACCOUNT
                         │
             ┌───────────┴───────────┐
             ▼                       ▼
         EC2 Instance            Elastic IP
      i-047e991048f29a1df     eipalloc-03804f2cd3a72ad04
             │                       │
             └───────────┬───────────┘
                         ▼
                    Association
                eipassoc-02ca3647c2985fa6f
                         │
                         ▼
                  Public IPv4
                  54.162.142.57
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `xfusion-ec2` or `xfusion-ec2-eip` not found | Wrong region selected | Confirm `us-east-1` explicitly in both console and `--region` |
| EIP shows a public IP but `InstanceId: None` | Allocated but not yet associated — the expected pre-task state | Run `associate-address`; this isn't a broken EIP |
| Assumed the auto-assigned public IP was "the Elastic IP" | Conflated the two kinds of public address (§2.2) | Compare the specific IP shown against the EIP's own `describe-addresses` output, not just "does the instance have a public IP" |
| `associate-address` succeeds but the instance's public IP still shows the old value | Checked before AWS's state propagated, or checked the wrong instance ID | Re-run `describe-instances` for the exact instance ID used in the associate call |
| Application still unreachable after association | Security group doesn't allow the relevant port | EIP only affects reachability of the address, not port-level access (§2.9) — check security group rules separately |
| Unsure whether the association actually "stuck" | Only checked the `associate-address` response, not the resulting state | Verify bidirectionally: `describe-addresses` AND `describe-instances` (§2.8) |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.10) out loud
      from "attach xfusion-ec2-eip to xfusion-ec2" to "verified from
      both directions."
- [ ] Explain, in one sentence, the difference between an auto-assigned
      public IP and an Elastic IP.
- [ ] Explain why `InstanceId: None` on a `describe-addresses` result is
      not itself an error — what does it actually mean?
- [ ] Explain the difference between an Allocation ID, an Association
      ID, and an Instance ID — what does each one identify?
- [ ] Explain why attaching an Elastic IP doesn't, by itself, make an
      application on that instance reachable over HTTP.
- [ ] Repeat the full sequence on a fresh EIP/instance pair from memory,
      verifying bidirectionally rather than trusting only the
      `associate-address` response.

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
performs the read, what performs the write, how do I verify — ideally
from both directions of the relationship." End with a compressed
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
