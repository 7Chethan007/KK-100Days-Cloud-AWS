# Day 24 — Configure an AWS Application Load Balancer for Nginx

A KodeKloud "100 Cloud/AWS" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

Day 21 launched an EC2 instance behind an Elastic IP, and Day 22 opened
a single security-group path to port 22 on it. Both labs stopped at
"the instance is reachable directly." This lab inserts a new layer in
front of the instance: a public **Application Load Balancer** that
accepts client HTTP and forwards it to the same kind of backend EC2 —
which is the first lab in this series where *two* resources' state
(ALB target-group health **and** an end-to-end HTTP response) must
agree before the task is considered done.

---

## 1. Scenario

The Nautilus DevOps team needs a public HTTP entry point for an
application running on an EC2 instance with Nginx. The application
itself is deployed later, so the immediate task is to route public
HTTP traffic through an AWS Application Load Balancer (ALB) to the
existing Nginx server.

| Requirement | Verified result |
|---|---|
| ALB | `datacenter-alb`, `active` |
| ALB endpoint | `datacenter-alb-119349275.us-east-1.elb.amazonaws.com` |
| Listener | HTTP :80 → forward to `datacenter-tg` |
| Target group | `datacenter-tg`, type `instance`, HTTP :80, health path `/` |
| Registered target | `i-073ee26be1cb4c5ff:80` — `healthy` |
| ALB security group | `datacenter-sg` — `sg-057fe0eed85448c5f` |
| Backend security group | EC2 default — `sg-05b981d99638148bd` |
| End-to-end HTTP test | `200 OK`, Nginx welcome page |

```text
Client / browser
      │  HTTP :80
      ▼
Application Load Balancer  ── datacenter-alb ── (internet-facing, listener HTTP :80)
      │  forward
      ▼
Target group  ── datacenter-tg ── (HTTP :80, health checks on /)
      │
      ▼
EC2 instance  ── datacenter-ec2 (i-073ee26be1cb4c5ff) ── Nginx :80

datacenter-sg  :  inbound TCP 80 from 0.0.0.0/0        (public clients → ALB)
EC2 default SG :  inbound TCP 80 from datacenter-sg    (ALB → Nginx)
```

The two security-group rules above are the security story of this
lab: the ALB is publicly reachable, but the backend server is *not*
opened to the internet directly — it only accepts HTTP from the ALB's
security group.

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 An ALB is a *stable public name* in front of a *changeable* backend

```text
Client ──▶ datacenter-alb-119349275.us-east-1.elb.amazonaws.com ──▶ backend(s)
             the part clients depend on            the part that can change
```

The DNS name is the contract with clients; the targets behind it can be
added, removed, or replaced without clients noticing. This is the same
"stable address" idea as Day 21's Elastic IP, but one layer higher:
Day 21 made a *single instance's* address stable, whereas an ALB makes
an *entire fleet's* entry point stable. This is why the task adds an
ALB rather than just exposing the instance.

### 2.2 ALB, listener, and target group are three separate objects answering three separate questions

```text
ALB           : where does public traffic arrive?      (the front door + its DNS name)
Listener      : what protocol/port is accepted, and where is it FORWARDED? (:80 → datacenter-tg)
Target group  : which backends receive it, and are they HEALTHY enough to?  (i-...:80, health path /)
```

Creating an ALB does not, by itself, route anything anywhere — a
listener must exist on it, and that listener must reference a target
group, and that target group must have healthy registered targets.
Every "the ALB is up but returns 503" symptom in §6 is one of these
three links being wrong, which is exactly why the runbook verifies
each one rather than only the ALB's own state.

### 2.3 A listener is an *entry rule*, not a server

```text
Listener (ALB side) :  "connections to :80 are accepted and forwarded to datacenter-tg"
Nginx    (backend)  :  the process that actually produces the HTTP response
```

It is easy to mentally merge these, but they fail independently: the
listener can exist and forward perfectly while Nginx is down (→ `502`),
and Nginx can be healthy while the listener forwards to the wrong
target group (→ `503`) or the wrong port. Keeping the two concepts
separate is what makes the §6 troubleshooting table diagnosable.

### 2.4 Two security-group rules, two different *sources* — and the backend intentionally has no public rule

```text
datacenter-sg   inbound TCP 80 from 0.0.0.0/0          → anyone on the internet can reach the ALB
EC2 default SG  inbound TCP 80 from sg-057fe0eed85448c5f → only the ALB can reach Nginx
```

The second rule uses a **security-group reference**, not a CIDR. That
means the backend accepts traffic from network interfaces associated
with the ALB's security group — regardless of what IP the ALB happens
to come from (ALB node IPs are not static). This is the concrete
technique behind "public ALB, private backend," and it is the same
source-vs-destination discipline as Day 2's security-group lab and
Day 22's port-22 rule. A CIDR of `0.0.0.0/0` on the *backend* would
have "worked" too — and in production would usually be the wrong
answer, because it removes the whole point of the ALB as the sole
gateway.

### 2.5 Subnet + Availability Zone placement is part of *routing eligibility*, not a deployment detail

```text
ALB must span >= 2 AZs (AWS requirement for an ALB)
A registered target whose AZ the ALB does not span → registered, but UNUSABLE
```

This is the failure this lab actually hit: the ALB was active in
`us-east-1b` / `us-east-1e` while EC2 sat in `us-east-1c`, so the
target returned `Target.NotInUse` — *not* `unhealthy`. The distinction
matters: `unhealthy` means "the ALB can route to it but the health
check failed"; `NotInUse` means "the ALB is not even eligible to route
to it." Fixing Nginx or the security groups would never have moved
that state. The ALB simply had to span the target's AZ. Same line of
reasoning as Day 3's subnet lab: the subnet an ENI lives in silently
constrains everything layered on top of it.

### 2.6 A correct CLI command can still fail — IAM authorization is its own layer

```text
aws elbv2 set-subnets ...   → "AccessDenied: not authorized to perform elasticloadbalancing:SetSubnets"
```

The command was syntactically correct and the *intent* was right, but
the active identity was not permitted to call it. This is a fourth,
independent layer (identity/permission) on top of the resource,
network, and application layers — the lab's workaround was to
**recreate** the ALB with the correct subnets instead of modifying it,
because recreation only needs create/delete permissions that the
identity did have. The cost of that workaround is real and worth
naming: a new ALB gets a **new ARN and a new DNS name**, and access is
briefly interrupted. It was acceptable here only because this is a
lab with no live clients.

### 2.7 A created resource is not a working resource — the decisive check is the full HTTP path

```text
describe-load-balancers → active        (resource exists)
describe-target-health  → healthy       (target is eligible + passing)
curl http://<alb-dns>/  → 200 OK        (traffic ACTUALLY flows end-to-end)
```

Each line verifies a strictly stronger claim than the one above it.
"ALB active" says nothing about targets; "target healthy" says nothing
about whether the listener forwards to *that* target group or whether
Nginx responds correctly. Only the end-to-end request exercises the
entire chain — DNS → ALB :80 → listener → target group → EC2 :80 →
Nginx — which is why the lab's completion criterion is the `200 OK`,
not the successful creation of any single resource.

### 2.8 The compressed reasoning chain

```text
Requirement: public HTTP entry point → ALB → existing EC2 Nginx
   → export AWS_DEFAULT_REGION=us-east-1; confirm identity (sts) + inspect EC2 (VPC/subnet/AZ) (§2.5)
   → ensure datacenter-sg allows inbound TCP 80 from 0.0.0.0/0 (§2.4)
   → create target group datacenter-tg (instance, HTTP :80, health "/"); register i-...:80 (§2.2)
   → create internet-facing ALB in >= 2 public subnets, INCLUDING the target's AZ (§2.5)
   → create a listener HTTP :80 → forward datacenter-tg (§2.2, §2.3)
   → ensure EC2 default SG allows inbound TCP 80 from the ALB's SECURITY GROUP, not a CIDR (§2.4)
   → if SetSubnets is denied, recreate the ALB with correct subnets (§2.6)
   → verify: describe-load-balancers active → describe-target-health healthy → curl → 200 OK (§2.7)
```

---

## 3. Concepts (reference)

### 3.1 Application Load Balancer (ALB)
A Layer 7 (HTTP/HTTPS) load balancer that terminates client
connections on a stable DNS name and forwards requests to registered
targets according to listener rules. It provides health-based routing
and lets backends change without changing the public entry point.

### 3.2 Listener
A rule on the load balancer defining the protocol and port it accepts
connections on and the default action to take (here: forward to a
target group). A listener is the *entry rule*, not the backend itself.

### 3.3 Target group
The set of destinations that receive forwarded requests, plus the
health-check configuration that decides whether each destination is
eligible. Defines target type (`instance` here), protocol, port, VPC,
and health-check path.

### 3.4 Security-group reference vs. CIDR source
An inbound rule can name a source security group instead of a CIDR
block, meaning "traffic from network interfaces associated with *that*
group." This is how the backend accepts traffic from the ALB without
knowing any of the ALB's (non-static) node IPs.

### 3.5 Availability Zones and subnets
An ALB is deployed into subnets, and those subnets determine its
enabled AZs. AWS requires at least two AZs. A target in an AZ the ALB
does not span is registered but `unused` (`Target.NotInUse`) — a
routing-eligibility problem, not a health problem.

### 3.6 IAM authorization as an independent layer
Resource state, network reachability, and application health can all
be correct while the AWS *identity* still lacks permission to perform
a given API call. A denied action is an authorization result, not a
configuration bug.

### 3.7 End-to-end verification
A created resource, an eligible target, and an actual successful HTTP
response are three progressively stronger claims. Only the last one
proves the full request path works.

---

## 4. Runbook

> The commands below document the successful configuration and are for
> learning/verification. They are not meant to recreate resources that
> already exist.

### 4.1 Confirm identity, region, and the target instance
```bash
export AWS_DEFAULT_REGION=us-east-1

aws sts get-caller-identity && aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=datacenter-ec2" \
            "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].{ID:InstanceId,VPC:VpcId,Subnet:SubnetId,AZ:Placement.AvailabilityZone}' \
  --output table
```
```text
ID                   VPC                     Subnet                  AZ
i-073ee26be1cb4c5ff  vpc-0651d1807418ccc95   subnet-01183de671b9c1d9d  us-east-1c
```
`sts get-caller-identity` confirms the account/identity; the EC2 query
fixes the VPC, subnet, and AZ that constrain the ALB (§2.5).

### 4.2 Verify the ALB security group (public HTTP)
```bash
aws ec2 describe-security-groups \
  --filters Name=group-name,Values=datacenter-sg \
            Name=vpc-id,Values=vpc-0651d1807418ccc95 \
  --query 'SecurityGroups[].{Name:GroupName,ID:GroupId}' --output table
```
Add the rule if missing (errors if it already exists):
```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-057fe0eed85448c5f \
  --protocol tcp --port 80 --cidr 0.0.0.0/0
```

### 4.3 Create the target group and register EC2
```bash
aws elbv2 create-target-group \
  --name datacenter-tg \
  --protocol HTTP --port 80 \
  --vpc-id vpc-0651d1807418ccc95 \
  --target-type instance

TG_ARN=$(aws elbv2 describe-target-groups \
  --names datacenter-tg \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

aws elbv2 register-targets \
  --target-group-arn "$TG_ARN" \
  --targets Id=i-073ee26be1cb4c5ff,Port=80
```
Health-check path `/`, served by Nginx; target type `instance` means
the registration is an instance ID + port (§3.3).

### 4.4 Create the ALB across two AZs — including the target's AZ
```bash
aws elbv2 create-load-balancer \
  --name datacenter-alb \
  --subnets subnet-01183de671b9c1d9d subnet-05f6898fc19cea5ba \
  --security-groups sg-057fe0eed85448c5f \
  --scheme internet-facing \
  --type application
```
Subnets: `subnet-01183de671b9c1d9d` (`us-east-1c`, same as EC2) and
`subnet-05f6898fc19cea5ba` (`us-east-1b`). Note the final ALB was
obtained by **recreating** it with these subnets after `SetSubnets`
was denied (§2.6); the earlier attempt spanned `us-east-1b`/`us-east-1e`
and left the target `NotInUse`.

### 4.5 Create the HTTP listener
```bash
ALB_ARN=$(aws elbv2 describe-load-balancers \
  --names datacenter-alb \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)

aws elbv2 create-listener \
  --load-balancer-arn "$ALB_ARN" \
  --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn="$TG_ARN"
```
Default action `forward` → `datacenter-tg` (§2.2, §2.3).

### 4.6 Allow the ALB to reach Nginx (EC2 default SG)
```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-05b981d99638148bd \
  --protocol tcp --port 80 \
  --source-group sg-057fe0eed85448c5f
```
Source is the **ALB security group**, not `0.0.0.0/0` (§2.4).

### 4.7 Verify ALB state and enabled AZs
```bash
aws elbv2 describe-load-balancers \
  --names datacenter-alb \
  --query 'LoadBalancers[0].{State:State.Code,DNS:DNSName,Zones:AvailabilityZones[].ZoneName}' \
  --output table
```
```text
State   active
DNS     datacenter-alb-119349275.us-east-1.elb.amazonaws.com
Zones   us-east-1b, us-east-1c
```

### 4.8 Verify target health
```bash
aws elbv2 describe-target-health --target-group-arn "$TG_ARN" --output table
```
```text
Id                   Port  State
i-073ee26be1cb4c5ff  80    healthy
```

### 4.9 Decisive end-to-end test
```bash
DNS=$(aws elbv2 describe-load-balancers \
  --names datacenter-alb \
  --query 'LoadBalancers[0].DNSName' --output text)

curl -i --connect-timeout 10 --max-time 20 "http://$DNS/"
```
```text
HTTP/1.1 200 OK
Content-Type: text/html
Server: nginx/1.24.0
```
Body: the Nginx welcome page.

---

## 5. Final state

| Check | Expected | Observed | Status |
|---|---|---|---|
| ALB `datacenter-alb` state | `active` | `active` | Pass |
| ALB AZs include target's AZ (`us-east-1c`) | `us-east-1b`, `us-east-1c` | ✅ | Pass |
| Listener HTTP :80 → `datacenter-tg` | forward | ✅ | Pass |
| Target `i-073ee26be1cb4c5ff:80` health | `healthy` | `healthy` | Pass |
| `datacenter-sg` inbound TCP 80 from `0.0.0.0/0` | present | ✅ | Pass |
| EC2 default SG inbound TCP 80 from `datacenter-sg` | present | ✅ | Pass |
| `curl http://<alb-dns>/` | `200 OK` | `200 OK` | Pass |

```text
Internet
   │  HTTP :80
   ▼
datacenter-alb  (active · us-east-1b + us-east-1c)
   │  listener :80 → forward
   ▼
datacenter-tg   (HTTP :80 · health "/")
   │  registered target
   ▼
i-073ee26be1cb4c5ff :80   (healthy)  ──▶ Nginx ──▶ 200 OK
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| ALB is `provisioning` | AWS still provisioning | Wait, then re-check `describe-load-balancers` |
| DNS does not resolve | ALB not ready, or DNS misconfigured | Confirm state `active`, then retry DNS/`curl` |
| Target `unused` / `Target.NotInUse` | Target's AZ not enabled on the ALB, or TG not used by a listener | Compare target AZ with ALB enabled AZs; align subnets (§2.5) |
| Target `initial` | Registration / initial health check in progress | Wait, then re-query target health |
| Target `unhealthy` | Health-check failure, blocked port/path, or app down | Inspect `Reason`/`Description`, SGs, and Nginx |
| HTTP `503` | No usable healthy target, or wrong listener/TG config | Inspect target health and listener forwarding action |
| HTTP `502` | Target reachable but returns no valid response | Check Nginx on :80 and backend SG source |
| `AccessDenied` on `SetSubnets` | IAM policy lacks the action | Check permissions; recreate the ALB with correct subnets rather than retrying (§2.6) |

### The key failure encountered

The ALB was active in `us-east-1b`/`us-east-1e` while EC2 sat in
`us-east-1c`, so the registered target reported:

```text
Target.NotInUse
Target is in an Availability Zone that is not enabled for the load balancer
```

`SetSubnets` was denied, so `datacenter-alb` was **recreated** with a
`us-east-1c` + `us-east-1b` subnet pair and its listener recreated.
The target then became `healthy` and `curl` returned `200 OK`. Lesson:
a *registered* target is not necessarily an *eligible* or *healthy*
one (§2.5).

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.8) out loud
      from "confirm the instance's AZ" to "verified via `curl` 200."
- [ ] Explain, in one sentence, the difference between a target that is
      `unhealthy` and one that is `Target.NotInUse`.
- [ ] Explain why the backend inbound rule references the ALB's
      security group instead of a CIDR, and what breaks if it is wrong.
- [ ] Explain why "ALB is `active`" is weaker evidence than "`curl`
      returns `200 OK`."
- [ ] Explain why recreating the ALB was a valid workaround here but a
      poor choice for a production load balancer carrying live traffic.
- [ ] From memory, stand up a second ALB in front of a different
      instance with a different target-group name, and verify it the
      same way — then deliberately place the ALB in the *wrong* AZ and
      observe `Target.NotInUse` before fixing it — without copying any
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
