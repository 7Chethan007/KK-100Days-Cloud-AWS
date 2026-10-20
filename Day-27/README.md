# Day 27 — Build a Public AWS VPC from First Principles

A KodeKloud "100 Cloud/AWS" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

Every prior EC2 lab in this series (Days 21, 25, 26) launched into
networking that already existed. This lab builds the networking
*itself* — VPC, subnet, Internet Gateway, route table — from nothing,
and in doing so makes explicit a distinction those earlier labs could
take for granted: a subnet isn't "public" because of its CIDR, its
auto-assign setting, or even an attached instance having a public
IP — it's public because its route table, specifically, routes
`0.0.0.0/0` to an Internet Gateway, and that route table is
*explicitly associated* with the subnet.

---

## 1. Scenario

Build a public VPC from scratch and launch an EC2 instance into it,
reachable over SSH from the internet.

| Resource | Config |
|---|---|
| VPC | `devops-pub-vpc`, `10.0.0.0/16` |
| Subnet | `devops-pub-subnet`, `10.0.1.0/24`, auto-assign public IPv4 enabled |
| Internet Gateway | `devops-pub-igw`, attached to the VPC |
| Route table | `devops-pub-rt`, `0.0.0.0/0 → igw`, associated with the subnet |
| EC2 instance | `devops-pub-ec2`, `t2.micro` |
| Security group | `devops-pub-sg`, SSH/22 from `0.0.0.0/0` |
| Region | `us-east-1` |

```text
Internet
   │
   ▼
Internet Gateway (devops-pub-igw) ── attached to ──▶ devops-pub-vpc (10.0.0.0/16)
   ▲                                                         │
   │ route: 0.0.0.0/0 → igw                                  ▼
Route table (devops-pub-rt) ── associated with ──▶ devops-pub-subnet (10.0.1.0/24)
                                                              │
                                                              ▼
                                            devops-pub-ec2 (10.0.1.20 / 3.92.66.212)
                                                     ▲
                                        security group: SSH/22 ← 0.0.0.0/0
```

---

## 2. Reasoning model — how to *derive* the configuration, not memorize it

### 2.1 Five separate components, five separate responsibilities

```text
VPC:               address space + isolation boundary
Subnet:            a slice of that address space, in one AZ
Internet Gateway:  the connection point to the public internet
Route table:       the decision of WHERE traffic goes
Security group:    the decision of WHAT traffic is ALLOWED
```

None of these substitutes for another — the same "composable,
independently-creatable building blocks" pattern as IAM's User/Group/
Policy/Role (Days 16-20) and EC2/CloudWatch/SNS (Day 25), now applied
to networking. Creating a VPC gives you address space; it says
nothing about internet reachability. Attaching an Internet Gateway
gives the VPC *a* connection to the internet; it says nothing about
whether any specific subnet actually uses it.

### 2.2 "Public subnet" is a *routing* fact, not a CIDR, auto-assign, or instance-IP fact

```text
10.x.x.x CIDR                        → just means PRIVATE address space (RFC 1918) — says nothing about public reachability
auto-assign public IPv4 enabled       → means NEW INSTANCES in this subnet GET a public IP — doesn't route anything
instance has a public IPv4 address    → an ADDRESS exists — still doesn't mean traffic can reach it
subnet's route table: 0.0.0.0/0 → igw → THIS is what actually makes the subnet "public"
```

This is the single most important distinction in the lab: three
things that *look* like they should make a subnet public (private
CIDR, auto-assign setting, an assigned public IP) are all orthogonal
to the one thing that actually does — the route table's default
route pointing at an attached Internet Gateway. A subnet with
auto-assign enabled and instances holding public IPs, but *no* route
to an IGW, is still not internet-reachable.

### 2.3 CIDR sizing: `/16` and `/24` both just answer "how many addresses," via the same formula

```text
VPC:    10.0.0.0/16  → 32-16=16 host bits → 2^16 = 65,536 addresses
Subnet: 10.0.1.0/24  → 32-24=8  host bits → 2^8  = 256 addresses, minus 5 AWS-reserved = 251 usable
```

The subnet's CIDR (`10.0.1.0/24`) is carved *out of* the VPC's larger
range (`10.0.0.0/16`) — not a separate, independent address space.
AWS reserves exactly 5 addresses *per subnet* (network address,
VPC router, DNS, future use, broadcast-equivalent), which is why the
251-usable-address figure is scoped to this one subnet, not divided
out of the VPC's 65,536 as a single flat pool.

### 2.4 Longest-prefix matching is *why* adding a default route doesn't redirect internal traffic

```text
Destination 10.0.1.20  → matches BOTH 10.0.0.0/16 (local) AND 0.0.0.0/0 (igw)
                           → /16 is MORE SPECIFIC → local route wins
Destination 8.8.8.8     → matches ONLY 0.0.0.0/0
                           → igw route applies
```

A route table doesn't pick "whichever route was added most recently"
or "the first match" — it picks the *most specific* matching
destination CIDR. This is exactly why the pre-existing `local` route
and the newly-added `0.0.0.0/0 → igw` route coexist without conflict:
traffic to another resource inside the VPC always matches the more
specific `/16` first, regardless of what the default route points at.

### 2.5 Creating a route table and associating it with a subnet are two separate, sequential actions

```text
Create devops-pub-rt                           → exists, has ONLY the default local route, associated with NOTHING yet
Add 0.0.0.0/0 → igw                              → the RULE now exists inside the table
Associate devops-pub-subnet with devops-pub-rt   → the subnet now actually USES this table's rules
```

A subnet not explicitly associated with a custom route table silently
falls back to the VPC's *main* route table — meaning a perfectly
correct custom route table, sitting right there with the correct
default route, has zero effect on a subnet that was never told to use
it. This is the direct networking analogue of Day 20's
"`AttachmentCount` alone doesn't tell you who" — the rule existing
isn't the same as the rule applying to the resource you care about.

### 2.6 Attaching an Internet Gateway to a VPC is a prerequisite for the route table to reference it — not optional, not implicit

```text
Create IGW (devops-pub-igw)         → exists, unattached, unusable as a route target yet
Attach IGW to devops-pub-vpc         → NOW a route table in that VPC can target it
Add route: 0.0.0.0/0 → devops-pub-igw → only meaningful because the attachment already happened
```

The dependency order matters: VPC → IGW → attach IGW to VPC → route
table references the *attached* IGW. Skipping the attachment step (or
attaching to the wrong VPC) would leave the route table pointing at a
target that can't actually serve as an egress path.

### 2.7 Private IPv4 and public IPv4 are two separate addresses serving two separate networks — neither replaces the other

```text
Private IPv4 (10.0.1.20)   → used for communication WITHIN the VPC (and anything routable to it)
Public IPv4 (3.92.66.212)   → used for DIRECT internet communication, mapped by AWS to the private address
```

An instance always has its private address; the public address is an
*additional* mapping AWS maintains on top of it, not a replacement.
This is why both addresses appear simultaneously in the console —
they answer "how do I reach this instance from inside the VPC" and
"how do I reach this instance from the internet" respectively, and
both can be true at once.

### 2.8 Security groups are stateful — response traffic to an allowed connection doesn't need its own rule

```text
Inbound rule: SSH/22 from 0.0.0.0/0  →  allows the CONNECTION in
Response packets (SSH server → client)  →  AUTOMATICALLY allowed back, no outbound rule needed
```

This is different from a stateless mechanism (like a network ACL)
where inbound and outbound traffic must each be explicitly permitted.
Not understanding statefulness could lead to unnecessarily adding
outbound rules that accomplish nothing, or — worse — assuming a
missing outbound rule is blocking return traffic when it isn't.

### 2.9 Reachability is a four-layer chain — failure at any one layer produces "unreachable," and each layer needs its own distinct check

```text
Layer 1 — Addressing:     does the instance have the expected public IP, in the expected VPC/subnet?
Layer 2 — Routing:         is the IGW attached? does the route table have 0.0.0.0/0 → igw? is THIS subnet associated with THAT table?
Layer 3 — Network controls: does the security group allow the port? does the NACL (stateless) allow both directions?
Layer 4 — OS/service:      is the SSH daemon actually running, listening, and correctly configured?
```

A `Running` instance with a public IP and a correct-looking security
group can still be completely unreachable if layer 2 (routing) is
wrong — which is exactly why this lab's troubleshooting discipline is
"verify the layer that should provide the behavior," not "change
settings until something works." This generalizes the layered
verification pattern from every prior lab (Day 19 DevOps's Nginx+PHP
chain, Day 20 MLOps's four-layer autolog check) to five networking
layers instead of application layers.

### 2.10 The compressed reasoning chain

```text
Requirement: a public VPC with an internet-reachable EC2 instance
   → verify Region FIRST — VPC/subnet/IGW/route-table are all regional/AZ-scoped resources
   → create the VPC (10.0.0.0/16) — just address space + isolation, nothing routed yet (§2.1)
   → create the subnet (10.0.1.0/24) INSIDE that VPC's range (§2.3)
   → enable auto-assign public IPv4 on the subnet — NOT what makes it "public" by itself (§2.2)
   → create the IGW, ATTACH it to the VPC — unattached IGW is unusable as a route target (§2.6)
   → create the route table, add 0.0.0.0/0 → igw — the rule now exists (§2.4)
   → EXPLICITLY associate the subnet with that route table — the rule now APPLIES (§2.5)
   → launch the instance into that subnet, with a security group allowing SSH/22 from 0.0.0.0/0
   → verify private AND public IP both present — two separate addresses, not one replacing the other (§2.7)
   → if unreachable, diagnose layer by layer: addressing → routing → security/NACL → OS/service (§2.9)
```

---

## 3. Concepts (reference)

### 3.1 VPC, subnet, route table, Internet Gateway, security group — five independent building blocks
Each solves one specific problem: isolation/address space (VPC),
subdivision by AZ (subnet), traffic direction (route table),
internet connectivity (IGW), and traffic permission (security group).
None is created or modified as a side effect of configuring another.

### 3.2 What actually makes a subnet "public"
Specifically and only: its associated route table contains a default
route (`0.0.0.0/0`) targeting an Internet Gateway that is attached to
the same VPC. Private-looking CIDRs, auto-assign settings, and
instance public IPs are separate, non-defining attributes.

### 3.3 CIDR sizing and the AWS reserved-address count
Usable addresses in an IPv4 CIDR block = `2^(32 - prefix length)`,
minus 5 AWS-reserved addresses per *subnet* specifically (not per
VPC).

### 3.4 Longest-prefix matching
A route table always routes by the most specific (longest-prefix)
matching destination CIDR, which is why a default route coexisting
with the VPC's local route never redirects intra-VPC traffic.

### 3.5 Route table association is explicit, not inherited by default
A subnet uses the VPC's main route table unless explicitly associated
with a different one — creating a custom route table with the
correct rules has no effect on a subnet that was never associated
with it.

### 3.6 Private IPv4 vs. public IPv4
Two separately-maintained addresses on the same instance: the private
address for intra-VPC (and connected-network) communication, the
public address for direct internet communication, mapped to the
private address by AWS's networking layer.

### 3.7 Stateful security groups vs. stateless network ACLs
Security groups automatically permit response traffic for an allowed
connection. Network ACLs do not — both inbound and outbound rules
must be explicitly configured for a NACL to permit a full exchange.

---

## 4. Runbook

### 4.1 Verify the Region
```text
Console Region selector → US East (N. Virginia) — us-east-1
```

### 4.2 Create the VPC
```text
VPC → Your VPCs → Create VPC → VPC only
  → Name: devops-pub-vpc
  → IPv4 CIDR: 10.0.0.0/16
  → Create VPC
```

### 4.3 Create the subnet and enable auto-assign public IPv4
```text
VPC → Subnets → Create subnet → VPC: devops-pub-vpc
  → Name: devops-pub-subnet
  → AZ: any available in us-east-1
  → IPv4 CIDR: 10.0.1.0/24
  → Create subnet
  → select subnet → Actions → Edit subnet settings
  → Enable auto-assign public IPv4 address → Save
```

### 4.4 Create and attach the Internet Gateway
```text
VPC → Internet gateways → Create internet gateway
  → Name: devops-pub-igw → Create
  → select it → Actions → Attach to a VPC → devops-pub-vpc → Attach
```

### 4.5 Create the route table, add the default route, associate the subnet
```text
VPC → Route tables → Create route table
  → Name: devops-pub-rt → VPC: devops-pub-vpc → Create

Open it → Routes → Edit routes → Add route
  → Destination: 0.0.0.0/0 → Target: Internet Gateway → devops-pub-igw → Save

Open it → Subnet associations → Edit subnet associations
  → select devops-pub-subnet → Save associations
```

### 4.6 Launch the EC2 instance
```text
EC2 → Launch instances
  → Name: devops-pub-ec2
  → AMI: Amazon Linux 2023
  → Instance type: t2.micro
  → VPC: devops-pub-vpc, Subnet: devops-pub-subnet
  → Auto-assign public IP: Enable
  → Security group: create new — devops-pub-sg
      → Inbound: SSH, TCP, 22, Source 0.0.0.0/0
  → Launch
```

### 4.7 Verify the instance
```text
EC2 → Instances → devops-pub-ec2
```
```text
Instance ID     i-01c69500e2f81e908
State           Running
Instance type   t2.micro
VPC             devops-pub-vpc
Subnet          devops-pub-subnet
Private IPv4    10.0.1.20
Public IPv4     3.92.66.212
```

### 4.8 Verify the security group
```text
Instance → Security tab → inspect devops-pub-sg
  → confirm SSH/TCP/22 from 0.0.0.0/0
```

---

## 5. Final state

| Requirement | Result |
|---|---|
| `devops-pub-vpc` (10.0.0.0/16) | ✅ |
| `devops-pub-subnet` (10.0.1.0/24), auto-assign public IPv4 enabled | ✅ |
| `devops-pub-igw` created and attached | ✅ |
| `devops-pub-rt`: `0.0.0.0/0 → igw`, associated with the subnet | ✅ |
| `devops-pub-ec2` (`t2.micro`) running | ✅ |
| Private `10.0.1.20` + public `3.92.66.212` both assigned | ✅ |
| `devops-pub-sg`: SSH/22 from `0.0.0.0/0` | ✅ |

```text
us-east-1
 └── devops-pub-vpc (10.0.0.0/16)
       ├── devops-pub-igw (attached)
       └── devops-pub-subnet (10.0.1.0/24)
              ├── route table: devops-pub-rt (0.0.0.0/0 → igw, local → 10.0.0.0/16)
              └── devops-pub-ec2 (t2.micro, Running)
                     private: 10.0.1.20   public: 3.92.66.212
                     security group: devops-pub-sg (SSH/22 ← 0.0.0.0/0)
```

---

## 6. Troubleshooting

| Symptom | Layer | Likely cause | Check |
|---|---|---|---|
| No public IPv4 address on the instance | Addressing | Auto-assign disabled, or launch override | Subnet settings; instance launch config |
| Public IP exists but no internet connectivity | Routing | IGW not attached, missing default route, or subnet not associated with the right route table | `describe-route-tables`, `describe-internet-gateways`, subnet association |
| SSH times out | Routing / Network controls | Any of the above, or security group/NACL blocking | Work through layers 2→3 in order |
| SSH says permission denied | OS/service | Wrong username/key, not a network issue | Check credentials, not routing |
| Internal (intra-VPC) traffic fails | Addressing/Routing | Wrong CIDR, security rules, or resource placement | Confirm both resources' subnet/CIDR |
| Wrong resources shown in console | — | Wrong Region or active filters | Check Region selector first |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.10) out
      loud from "verify Region" to "layered troubleshooting if
      unreachable."
- [ ] Explain, in one sentence, why a `10.x.x.x` CIDR and an
      auto-assigned public IP do NOT by themselves make a subnet
      public.
- [ ] Explain why adding a `0.0.0.0/0 → igw` route doesn't redirect
      traffic destined for another instance inside the same VPC.
- [ ] Explain why a subnet needs to be *explicitly* associated with a
      custom route table, rather than automatically using whichever
      one was most recently edited.
- [ ] Explain the difference between a security group being
      "stateful" and a network ACL being "stateless," using the SSH
      response traffic as the example.
- [ ] Given an instance with a public IP that's still unreachable,
      name the four layers you'd check, in order, and why that order
      makes sense.
- [ ] From memory, build a second, completely separate public VPC
      with a different CIDR, launch an instance into it, and verify
      reachability — without copying any step from this file.

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
