# Day 26 — Configure an EC2 Instance as a Web Server with Nginx (via User Data)

A KodeKloud "100 Cloud/AWS" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

Day 21 launched a bare EC2 instance; Day 24 put an ALB in front of
one; Day 25 attached a CloudWatch alarm to one. This lab introduces
**user data** — a mechanism for automating an instance's *first-boot*
configuration — and the specific trap of recovering an instance that
was launched *without* it: the website can end up working perfectly
while the one thing a user-data-focused check actually verifies stays
empty.

---

## 1. Scenario

Launch an Ubuntu EC2 instance `devops-ec2` that serves the Nginx
welcome page over HTTP, installed automatically via EC2 user data,
reachable from the internet on port 80.

| Requirement | Value |
|---|---|
| Instance name | `devops-ec2` |
| AMI | Ubuntu Server LTS |
| Security group | HTTP (TCP 80) from `0.0.0.0/0` |
| User data | installs + starts Nginx at first boot |

```text
Client ── HTTP :80 ──▶ Security group (allow 80 from 0.0.0.0/0) ──▶ devops-ec2
                                                                          │
                                                             user data (first boot only):
                                                             apt-get install nginx
                                                             systemctl enable --now nginx
                                                                          │
                                                                          ▼
                                                              Nginx welcome page
```

---

## 2. Reasoning model — how to *derive* the configuration, not memorize it

### 2.1 Four separate layers each have to work — none substitutes for another

```text
EC2 instance        → exists, is running
Security group       → allows the TRAFFIC to even reach the instance
Nginx                → is INSTALLED and RUNNING, listening on 80
User data             → is what AUTOMATED the install, as opposed to a manual fix
```

A website being reachable over HTTP only proves the first three
layers are correct — it says nothing about *how* Nginx got there. This
is the central distinction the lab is built around: Method B (manual
install) can satisfy layers 1-3 completely while leaving layer 4
(user data) exactly as empty as it was before — the same "a resource
working correctly says nothing about whether it was configured the
required *way*" principle as every other lab in this series that
separates configuration-correctness from outcome-correctness.

### 2.2 User data runs *once*, at first boot — it is not a configuration-management system

```text
Launch instance (user data present) → cloud-init runs the script ONCE, at that first boot
Edit the stored user-data value LATER → does NOT automatically rerun anything
```

This is why Method A (configure user data *before* launch) and Method
B (the instance already exists without it) are genuinely different
situations requiring different remediation — you cannot "add" user
data to an already-running instance and expect it to retroactively
execute; the install either happened automatically at boot or it
didn't, and fixing that after the fact means either terminating and
relaunching with user data configured, or installing manually and
accepting that the user-data *attribute* will stay empty.

### 2.3 A security group is a firewall, not an installer — it answers a completely separate question from "is the software running"

```text
Security group: "is 0.0.0.0/0 ALLOWED to reach port 80 on this instance?"
Nginx service:   "IS there anything actually LISTENING on port 80?"
```

A server could have Nginx running perfectly and still be completely
unreachable if the security group blocks port 80; conversely, a
security group could correctly allow port 80 while nothing is
actually listening there. These are two independent failure modes —
diagnosing "website unreachable" means checking *both*, not assuming
one implies the other, the same two-sided check as every prior
"configuration vs. notification," "trust vs. permissions" split in
this series (Day 20's IAM role, Day 22's SSH authorized_keys/
known_hosts, Day 25's alarm threshold/action).

### 2.4 `running` state and the status checks don't verify the *application* at all — same gap as Day 25's alarm states

```text
Instance state: running   → the VM is powered on
System status check: ok   → AWS's underlying infrastructure is healthy
Instance status check: ok → the guest OS is reachable/responsive
                              NONE of these say anything about Nginx specifically
```

This is the EC2 analogue of Day 25's "a successful push/an empty dry
run doesn't prove the exact requirement" pattern — three distinct,
genuinely green signals that still leave the actual question (is the
web server serving content) completely unanswered. The only check
that answers *that* question is an actual HTTP request.

### 2.5 `describe-instance-attribute --attribute userData` returning `None` is informative, not a tooling failure

```text
aws ec2 describe-instance-attribute --attribute userData --query 'UserData.Value' --output text
→ None
```

A `None` result means exactly what it says: no user data was
configured for this instance at launch — it is not an error in the
command, and piping `None` through `base64 --decode` would just
produce garbage/empty output, not a sign the decode step failed. This
is the decisive check for whether Method A's requirement (user-data
configured) is actually satisfied, independent of whether the website
happens to work.

### 2.6 `curl -I` proves HTTP reachability end-to-end; it says nothing about the installation *method*

```text
curl -I http://PUBLIC-IP → HTTP/1.1 200 OK, Server: nginx/...
```

This is strong, decisive evidence that the full chain (security
group → instance → Nginx) works — but it is evidence about *outcome*
only. A lab check that specifically inspects the user-data attribute
is checking a *different* fact entirely, and a `200 OK` here provides
zero information about what that check will find — the gap Method
B explicitly cannot close.

### 2.7 Duplicate `Name` tags turn a single-instance assumption into a silent multi-match bug

```text
INSTANCE_ID=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=devops-ec2" \
  --query '...InstanceId' --output text)
```

If more than one instance shares the `devops-ec2` tag (e.g. from a
retried launch), this command's output could contain *multiple* IDs
space-separated in one string — assigning that directly to a variable
and using it in a follow-up command could silently operate against the
wrong instance, or fail in a confusing way. Confirming the returned
value is a single, expected ID before using it downstream is the same
exact-target discipline as filtering by tag then verifying the
specific resource throughout this series.

### 2.8 The compressed reasoning chain

```text
Requirement: devops-ec2 serving Nginx over HTTP, installed via user data
   → (Method A, preferred) configure the install script in User data BEFORE launch (§2.2)
   → configure the security group: HTTP/80 from 0.0.0.0/0 — a SEPARATE concern from Nginx itself (§2.3)
   → launch, wait for running + BOTH status checks ok — none of which verify Nginx (§2.4)
   → curl -I http://<public-ip> — the decisive OUTCOME check (§2.6)
   → describe-instance-attribute --attribute userData — the decisive CONFIGURATION-METHOD check (§2.5)
   → (Method B, recovery only) manually apt-get install + enable --now nginx —
     fixes the OUTCOME, leaves userData EMPTY — know which requirement this actually satisfies (§2.1, §2.2)
   → before trusting a filtered InstanceId, confirm it's a SINGLE match, not several (§2.7)
```

---

## 3. Concepts (reference)

### 3.1 EC2 user data
A script (or cloud-init config) supplied at launch time that the
instance's first boot executes automatically via `cloud-init` — a
mechanism for reproducible, automated initial configuration, not an
ongoing configuration-management tool.

### 3.2 Security group as a stateful firewall
Controls which network traffic is permitted to reach an instance.
Entirely independent of whether any particular service is installed
or running on that instance — reachability and service health are
separate facts.

### 3.3 Instance state vs. status checks vs. application health
`running` reflects power state; system/instance status checks reflect
AWS infrastructure and guest-OS reachability respectively. None of
these three signals inspects any specific application running inside
the instance.

### 3.4 Configuration method vs. outcome
Whether a requirement is satisfied by *how* something was configured
(e.g. via user data specifically) versus merely *whether* it currently
works are two different checks — a correct outcome achieved by an
unintended method may satisfy one and fail the other.

### 3.5 `describe-instance-attribute` for user data
The authoritative way to inspect what user-data script (if any) was
actually configured for a running instance — a `None` result is a
definitive, meaningful answer, not a tooling error.

---

## 4. Runbook

### 4.1 Method A — launch with user data configured (preferred)
```text
EC2 → Launch instances
  → Name: devops-ec2
  → AMI: Ubuntu Server LTS
  → Instance type: t2.micro (if permitted)
  → Auto-assign Public IP: enabled
  → Security group: HTTP (TCP 80) from 0.0.0.0/0
  → Advanced details → User data:
```
```bash
#!/bin/bash
apt-get update
apt-get install -y nginx
systemctl enable --now nginx
```
```text
  → Launch instance
```

### 4.2 Wait and verify instance health
```bash
aws ec2 describe-instance-status \
  --instance-ids "$INSTANCE_ID" --include-all-instances \
  --query 'InstanceStatuses[].{State:InstanceState.Name,System:SystemStatus.Status,Instance:InstanceStatus.Status}' \
  --output table
```
Expect `running` / `ok` / `ok` — none of which verify Nginx (§2.4).

### 4.3 Verify the website
```bash
PUBLIC_IP=$(aws ec2 describe-instances --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
curl -I --connect-timeout 10 "http://$PUBLIC_IP"
```
```text
HTTP/1.1 200 OK
Server: nginx/1.28.3 (Ubuntu)
Content-Type: text/html
```

### 4.4 Method B — recovery, if the instance was already launched without user data
```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-ec2" \
  --query 'Reservations[].Instances[].{ID:InstanceId,State:State.Name,PublicIP:PublicIpAddress,AMI:ImageId,SecurityGroups:SecurityGroups[].GroupId}' \
  --output table
```
Connect via EC2 Instance Connect, then on the instance:
```bash
sudo apt-get update
sudo apt-get install -y nginx
sudo systemctl enable --now nginx
systemctl is-active nginx
systemctl is-enabled nginx
sudo systemctl status nginx --no-pager
```

### 4.5 Verify the security group
```bash
aws ec2 describe-security-groups --group-ids sg-XXXXXXXX \
  --query 'SecurityGroups[].IpPermissions[].{Protocol:IpProtocol,FromPort:FromPort,ToPort:ToPort,IPv4:IpRanges[].CidrIp}' \
  --output table
```
Confirm TCP 80 from `0.0.0.0/0`.

### 4.6 Verify user data — the decisive check for Method A's specific requirement
```bash
aws ec2 describe-instance-attribute --instance-id "$INSTANCE_ID" \
  --attribute userData --query 'UserData.Value' --output text
```
```text
None
```
`None` means Method B was used — this instance's outcome may be
correct, but its *configuration method* doesn't satisfy a check that
specifically inspects user data (§2.5). If a script IS configured:
```bash
aws ec2 describe-instance-attribute --instance-id "$INSTANCE_ID" \
  --attribute userData --query 'UserData.Value' --output text | base64 --decode
```

### 4.7 Troubleshoot via cloud-init logs, if user data was configured but Nginx still didn't start
```bash
sudo cat /var/log/cloud-init-output.log
sudo cloud-init status --long
```

---

## 5. Final state

| Requirement | Result |
|---|---|
| `devops-ec2` running, Ubuntu AMI | ✅ |
| Both status checks `ok` | ✅ |
| Security group allows HTTP/80 from `0.0.0.0/0` | ✅ |
| Nginx active and enabled | ✅ |
| `curl -I` → `200 OK`, Nginx server header | ✅ |
| User-data attribute contains the install script (Method A) | ✅/⚠️ — depends on which method was used (§2.1) |

```text
devops-ec2
 ├── Security group: HTTP/80 ← 0.0.0.0/0
 ├── Nginx: active, enabled, listening on 80
 └── User data: EITHER the install script (Method A)
                  OR None (Method B — outcome correct, method unverifiable)
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Browser times out | Security group, network path, or Nginx not running | Check port 80 rule, public IP, `systemctl status nginx` |
| `Connection refused` | Nothing listening on port 80 | `systemctl status nginx` on the instance |
| HTTP `403`/`404` | Nginx running but config/document root differs | Inspect Nginx config and document root |
| Running + status `ok`, but no website | Status checks don't verify the application (§2.4) | Check Nginx status and cloud-init logs directly |
| `describe-instance-attribute` returns `None` | User data wasn't configured for this instance (Method B was used) | Use the launch workflow with user data configured *before* launch if that's the actual requirement |
| `UnauthorizedOperation` | IAM permissions don't allow the requested API call | Check the lab role's permissions for the specific action |
| Multiple instance IDs returned for one tag filter | Duplicate `Name` tags from a retried launch | Inspect all returned IDs before acting on any of them (§2.7) |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.8) out loud
      from "configure user data before launch" to "verify via
      `describe-instance-attribute`."
- [ ] Explain, in one sentence, why a working website doesn't prove
      user data was configured.
- [ ] Explain why editing an instance's stored user-data value after
      it's already running has no effect on that instance.
- [ ] Explain why checking the security group and checking Nginx's
      running state are two separate diagnostic steps, not one.
- [ ] Explain why `running` + both status checks `ok` doesn't prove
      the website works.
- [ ] Explain what a `None` result from `describe-instance-attribute
      --attribute userData` specifically tells you.
- [ ] From memory, launch a second instance using Method A with a
      *different* user-data script (e.g. installing Apache instead),
      verify both the HTTP outcome and the user-data attribute
      independently — without copying any command from this file.

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
