# Day 22 — Configure Secure SSH Access to an EC2 Instance

A KodeKloud "100 Cloud/AWS" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

Day 21 launched an EC2 instance and attached an Elastic IP, ending at
"the instance has a stable address" (Day 21 §2.7 explicitly flagged
that reachability still depends on layers beyond the IP itself). This
lab is one of those layers made concrete: getting actual
authenticated access to the instance via SSH public-key
authentication, bootstrapped through EC2 Instance Connect rather than
an EC2 key pair selected at launch.

---

## 1. Scenario

Create EC2 instance `datacenter-ec2`, generate an SSH key pair on the
`aws-client` landing host, install the public key into the instance's
`root` account, and establish passwordless SSH access.

| Requirement | Verified result |
|---|---|
| EC2 instance | `datacenter-ec2`, `i-0d216f7b7c36e3f45`, `t2.micro`, running |
| Public IP | `34.229.227.10` |
| SSH key pair | generated on `aws-client`, `/root/.ssh/id_rsa` + `.pub` |
| Target account | `root` on EC2 |
| Authentication | public-key SSH succeeded |

```text
aws-client                              datacenter-ec2 (34.229.227.10)
  /root/.ssh/id_rsa      (private)         /root/.ssh/authorized_keys
  /root/.ssh/id_rsa.pub  (public) ───copy──▶  (contains the public key)
  /root/.ssh/known_hosts (server's host key)
         │
         └── ssh -i /root/.ssh/id_rsa root@34.229.227.10 ──▶ root shell
```

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 Private key stays with the client; only the public key ever travels to the server

```text
aws-client:  /root/.ssh/id_rsa       ← stays here, NEVER copied anywhere
aws-client:  /root/.ssh/id_rsa.pub   ← the ONLY thing that gets copied, to the server
```

The server verifies a cryptographic proof that the client holds the
matching private key — it never needs, and should never receive, a
copy of that private key itself. This is why the key pair was
generated directly *on* `aws-client` rather than on the EC2 instance
and copied over: generating it at the destination that will actually
hold the private key avoids ever having the secret exist anywhere it
would need to be transferred away from.

### 2.2 An EC2 launch-time key pair and this lab's generated key solve the same category of problem differently

```text
EC2 key pair at launch:  AWS injects a public key into the instance's INITIAL account
This lab's key:          generated independently, installed MANUALLY afterward
```

The task specifically required the key to be generated at
`/root/.ssh/id_rsa` on `aws-client` — not whatever key pair AWS might
auto-provision. "Proceed without a key pair" at launch plus EC2
Instance Connect for initial browser-based access is the mechanism
that lets the *required* key be the one actually installed, rather
than conflating AWS's own key-provisioning feature with the specific
key pair this lab's requirement names.

### 2.3 Directory and file permissions are a hard prerequisite, not a formality

```text
chmod 700 /root/.ssh                      → only the owner can enter this directory
chmod 600 /root/.ssh/authorized_keys      → only the owner can read/write this file
chmod 600 /root/.ssh/id_rsa               → only the owner can read the private key
```

SSH daemons actively check these permissions and will refuse key-based
authentication if they're too permissive — this isn't advisory
hardening, it's a functional requirement for the mechanism to work at
all. A key pair generated and installed correctly can still fail
authentication purely because a directory or file was left world- or
group-readable.

### 2.4 `authorized_keys` and `known_hosts` answer two completely different questions — same two-sided pattern as every "both ends must agree" lab in this series

```text
authorized_keys (on the SERVER):  which public keys may log in as THIS account?
known_hosts     (on the CLIENT):  do I recognize THIS server's identity?
```

Neither file substitutes for the other. `authorized_keys` governs
authorization (who can get in); `known_hosts` governs server identity
verification (is this the server I think it is) — the same "two
separate, independently-checkable facts" structure as Day 20's
trust-policy-vs-permissions-policy split for IAM roles, just applied
to SSH instead of IAM. Accepting an unknown host key for the first
time (`yes` at the prompt) only answers the second question; it has
no bearing on whether the key-based login itself will succeed.

### 2.5 The exact target account determines the exact file path — same exact-path discipline as every prior lab

```text
Target account: root  → /root/.ssh/authorized_keys
(if it were ubuntu):   → /home/ubuntu/.ssh/authorized_keys
```

SSH resolves `authorized_keys` relative to the *account being logged
into*, not some global location — installing the public key under the
wrong user's home directory would leave the required `root` account
still unable to authenticate, even though a key was technically
installed somewhere on the box. Same discipline as Day 19 DevOps's
`appsv`/`apps` typo and Day 21 Git's `demo.gitv`/`demo.git` — the
exact path is part of the requirement, not an approximate target.

### 2.6 An empty passphrase trades convenience for a specific, deliberate security tradeoff

```text
ssh-keygen -t rsa -b 4096 -f /root/.ssh/id_rsa   (no passphrase entered)
```

Leaving the passphrase empty lets the SSH client use the key
non-interactively — necessary for this lab's scripted/automated
connection requirement, but it also means anyone who obtains the raw
private key file can use it immediately with no second factor. This
is an explicit, named tradeoff (convenience vs. an additional secret
layer), not an oversight — production use would generally prefer a
passphrase combined with an SSH agent.

### 2.7 A successful interactive SSH login and a successful *passwordless* login are two different claims

```text
ssh -i ... root@<ip>                                  → succeeded, but a prompt (host key) appeared
ssh -o BatchMode=yes -i ... root@<ip> 'whoami && hostname'  → succeeds with ZERO prompts
```

`BatchMode=yes` disables any interactive fallback — if authentication
depended on something other than the key (a password prompt, an
agent interaction), this command would fail outright instead of
waiting for input. This is the stronger, decisive test that the lab's
actual requirement — *passwordless* access — is genuinely satisfied,
not just that SSH access of *some* kind works.

### 2.8 The compressed reasoning chain

```text
Requirement: datacenter-ec2 + passwordless root SSH via a client-generated key
   → launch EC2 instance, "proceed without a key pair" (§2.2)
   → verify instance via describe-instances (same pattern as Day 21)
   → mkdir -p + chmod 700 /root/.ssh on aws-client (§2.3)
   → ssh-keygen -t rsa -b 4096 -f /root/.ssh/id_rsa, empty passphrase (§2.6)
   → EC2 Instance Connect for initial browser-based access (bootstraps without the new key yet)
   → install the PUBLIC key into /root/.ssh/authorized_keys — exact account, exact path (§2.5)
   → chmod 700 .ssh / 600 authorized_keys on the SERVER (§2.3)
   → ssh -i /root/.ssh/id_rsa root@<ip> from aws-client — accept the host key once (§2.4)
   → stronger check: ssh -o BatchMode=yes ... 'whoami && hostname' — proves PASSWORDLESS, not just working (§2.7)
```

---

## 3. Concepts (reference)

### 3.1 SSH public-key authentication
A key pair where the private key proves identity without ever being
transmitted; the server stores only the public half and verifies a
cryptographic challenge the client can only answer correctly if it
holds the matching private key.

### 3.2 `authorized_keys` vs. `known_hosts`
`authorized_keys`, on the server, lists which public keys may
authenticate as a given account. `known_hosts`, on the client, records
server host keys the client has previously accepted, to detect if a
server's identity changes unexpectedly later. They protect against
different threats and are maintained on opposite ends of the
connection.

### 3.3 EC2 key pair (launch-time) vs. a manually generated/installed key
AWS's EC2 key pair feature is one mechanism for provisioning an
initial key into a new instance's default account. It is independent
of, and doesn't have to be used alongside, any key pair generated and
installed manually after the fact.

### 3.4 SSH directory/file permission requirements
`700` on `.ssh` directories and `600` on private keys and
`authorized_keys` are enforced, not merely recommended, by most SSH
daemon implementations — overly permissive modes can cause
authentication to be silently refused.

### 3.5 `BatchMode=yes` as a decisive non-interactive test
Disabling all interactive prompts turns "does this connection
eventually succeed" into "does this connection succeed with *zero*
additional input" — the stronger, more specific claim this lab's
requirement actually makes.

---

## 4. Runbook

### 4.1 Retrieve lab AWS credentials
```bash
showcreds
```
Console credentials are separate from the SSH key generated later —
two independent authentication systems (AWS Console/API vs. SSH).

### 4.2 Launch the EC2 instance
```text
EC2 → Instances → Launch instances
  → Name: datacenter-ec2
  → AMI: Ubuntu
  → Instance type: t2.micro
  → Key pair: Proceed without a key pair
  → Launch instance
```

### 4.3 Verify the instance via CLI
```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=datacenter-ec2" \
  --query 'Reservations[].Instances[].{Name:Tags[?Key==`Name`]|[0].Value,InstanceId:InstanceId,State:State.Name,Type:InstanceType,PublicIP:PublicIpAddress,KeyName:KeyName}' \
  --output table
```
```text
InstanceId  i-0d216f7b7c36e3f45
Name        datacenter-ec2
PublicIP    34.229.227.10
State       running
Type        t2.micro
KeyName     None
```

### 4.4 Create the SSH directory on `aws-client`
```bash
mkdir -p /root/.ssh
chmod 700 /root/.ssh
ls -l /root/.ssh/id_rsa /root/.ssh/id_rsa.pub
```

### 4.5 Generate the RSA key pair on `aws-client`
```bash
ssh-keygen -t rsa -b 4096 -f /root/.ssh/id_rsa
```
```text
Your identification has been saved in /root/.ssh/id_rsa
Your public key has been saved in /root/.ssh/id_rsa.pub
```
Empty passphrase, required for this lab's non-interactive access.

### 4.6 Bootstrap initial access via EC2 Instance Connect
```text
EC2 → Instances → select datacenter-ec2 → Connect → EC2 Instance Connect
```
Opens a browser-based terminal on the instance without needing any
SSH key yet.

### 4.7 Install the public key into the root account
On the instance, via the Instance Connect session:
```bash
mkdir -p /root/.ssh
chmod 700 /root/.ssh
# paste the contents of aws-client's /root/.ssh/id_rsa.pub as one line into:
#   /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
```

### 4.8 Connect from `aws-client` using the new private key
```bash
ssh -i /root/.ssh/id_rsa root@34.229.227.10
```
```text
The authenticity of host '34.229.227.10' can't be established.
Are you sure you want to continue connecting? yes
Warning: Permanently added '34.229.227.10' ... to the list of known hosts.
```
Followed by the Ubuntu login banner and a root shell.

### 4.9 Decisive non-interactive verification
```bash
ssh -o BatchMode=yes \
  -i /root/.ssh/id_rsa \
  root@34.229.227.10 \
  'whoami && hostname'
```
```text
root
ip-172-31-46-42
```

---

## 5. Final state

| Requirement | Result |
|---|---|
| EC2 instance `datacenter-ec2` running, `t2.micro` | ✅ |
| Key pair generated on `aws-client`, not on EC2 | ✅ |
| Public key installed in `/root/.ssh/authorized_keys` on EC2 | ✅ |
| `.ssh` directory and `authorized_keys` permissions correct (`700`/`600`) | ✅ |
| Host key accepted into `aws-client`'s `known_hosts` | ✅ |
| Interactive `ssh -i` login succeeds | ✅ |
| Non-interactive `BatchMode=yes` login succeeds | ✅ |

```text
aws-client                                   datacenter-ec2 (34.229.227.10)
 ├── /root/.ssh/id_rsa       (private, 600)     ├── /root/.ssh (700)
 ├── /root/.ssh/id_rsa.pub   (public)  ────────▶│   └── authorized_keys (600)
 └── /root/.ssh/known_hosts  (server's host key)│
                                                  └── root shell, passwordless
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Permission denied (publickey)` | Public key missing/incorrect, or installed for the wrong account | Check `/root/.ssh/authorized_keys` on the EC2 instance |
| `Connection timed out` | Security group or network path blocks port 22 | Check inbound TCP 22 and the allowed source range |
| `WARNING: UNPROTECTED PRIVATE KEY FILE` | Private key permissions too open | `chmod 600 /root/.ssh/id_rsa` on `aws-client` |
| SSH prompts for a password | Key auth failed, or server configured for a different auth method | Re-check key installation and SSH daemon configuration |
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | Server's host key differs from the saved one | Verify the new fingerprint before updating `known_hosts` |
| `No such file or directory` | Wrong path for the key or target file | Verify the exact absolute path (§2.5) |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.8) out loud
      from "generate the key pair on `aws-client`" to "verified via
      `BatchMode=yes`."
- [ ] Explain, in one sentence, why the private key was generated
      directly on `aws-client` rather than on the EC2 instance and
      copied over.
- [ ] Explain the difference between what `authorized_keys` and
      `known_hosts` each protect against.
- [ ] Explain why installing the public key under
      `/home/ubuntu/.ssh/authorized_keys` would not satisfy this
      lab's requirement, even though the key itself is correct.
- [ ] Explain why a successful `ssh -i ...` login with a host-key
      prompt is weaker evidence than a successful
      `ssh -o BatchMode=yes ...` login.
- [ ] From memory, generate a second key pair, install it for a
      *different* (non-root) user on a fresh instance, and verify
      passwordless access to that account specifically — without
      copying any command from this file.

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
