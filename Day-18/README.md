# Day 18 — Create a Read-Only IAM Policy for EC2 Console Access

A KodeKloud "100 Cloud/AWS" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

This is the third IAM lab in this series — Day 16 created a user,
Day 17 created a group; today creates the third core IAM building
block, a customer-managed **policy**, and is the first of the three
where the actual *content* (the JSON permission document) matters as
much as the resource's mere existence.

---

## 1. Scenario

Create a customer-managed IAM policy named `iampolicy_anita` that
allows viewing EC2 instances, AMIs, tags, and snapshots — without
granting any ability to create, modify, start/stop, or delete EC2
resources.

```text
iampolicy_anita
        │
        └── Allow
             ├── ec2:DescribeInstances
             ├── ec2:DescribeImages
             ├── ec2:DescribeTags
             └── ec2:DescribeSnapshots
```

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 A policy is not an identity — recap, now the third IAM building block

```text
User   (Day 16)  → an identity
Group  (Day 17)  → a collection of identities
Policy (today)    → a document of PERMISSIONS, attached to neither by default
```

Same identity-vs-authorization distinction from Days 16–17, now
viewed from the authorization side specifically: creating
`iampolicy_anita` produces a standalone permissions document with
`AttachmentCount: 0` — correct, since the task only asks for the
policy to *exist*, not to be attached to anything. A policy sitting
unattached is a fully valid, complete answer to this specific
requirement.

### 2.2 Customer-managed vs. AWS-managed — why create one at all

```text
AWS-managed policy    → AWS owns and maintains it, generic/broad coverage
Customer-managed policy → you own it, precise to your exact requirement
```

The task needs an exact four-action permission set
(`DescribeInstances`/`DescribeImages`/`DescribeTags`/
`DescribeSnapshots`) — no AWS-managed policy would match that precise
boundary, which is exactly why a customer-managed policy (one you
author yourself) is the correct tool.

### 2.3 Reading the policy document structurally

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ec2:DescribeInstances", "ec2:DescribeImages", "ec2:DescribeTags", "ec2:DescribeSnapshots"],
      "Resource": "*"
    }
  ]
}
```

```text
Version    → the IAM POLICY LANGUAGE version (a fixed date string, not
              a creation timestamp — "2012-10-17" is simply the current
              policy-language schema version, used in virtually every
              IAM policy regardless of when it was authored)
Effect     → Allow or Deny
Action     → which API calls this statement applies to
Resource   → which specific resources this applies to ("*" = all,
              scoped implicitly by what the listed actions can even act on)
```

### 2.4 Why `Describe*` actions are inherently read-only, by naming convention

```text
ec2:DescribeInstances   → read
ec2:DescribeImages        → read
ec2:DescribeTags           → read
ec2:DescribeSnapshots       → read

ec2:RunInstances            → write (never listed)
ec2:TerminateInstances        → write (never listed)
ec2:DeleteSnapshot              → write (never listed)
```

AWS API actions follow a broadly consistent naming convention:
`Describe*`/`List*`/`Get*` are read operations; verbs like `Run`/
`Create`/`Modify`/`Terminate`/`Delete` are write operations. The
read-only property of this policy isn't something you have to prove
by exhaustively listing every *excluded* action — it follows directly
from only including `Describe*` actions and nothing else.

### 2.5 The important caveat: this policy alone doesn't guarantee an identity is globally read-only

```text
This policy: Allow Describe* only
        +
Some OTHER policy also attached to the same identity: Allow ec2:TerminateInstances
        =
The identity CAN terminate instances — this policy didn't prevent it
```

IAM evaluates *all* policies attached to an identity together — the
absence of a permission in *this* policy says nothing about whether
some other attached policy grants it. This policy is accurately
described as "read-only" in isolation; whether an identity *using* it
ends up actually being read-only depends on everything else attached
to that identity too. Worth stating precisely rather than overclaiming
what one policy, by itself, guarantees.

### 2.6 Why `Resource: "*"` is appropriate here, not a sign of over-broad access

```text
Resource: "*"   → applies to every resource these specific DESCRIBE actions can return
```

`Describe*` EC2 actions are inherently account/region-wide read
operations — AWS doesn't support scoping `DescribeInstances` down to
"only these specific instance ARNs" the way it might for a write
action on a specific resource. `"*"` here isn't carelessly broad; it's
the only resource scope these particular read actions actually
support.

### 2.7 IAM is global — the task's region mention refers to the EC2 resources being viewed, not the policy itself

```text
IAM policy   → GLOBAL (same recurring exception from Day 16)
EC2 resources it lets you view → regional (us-east-1, or any other region)
```

Same "IAM has no region dimension" exception flagged in Day 16 — the
policy document itself isn't created "in" `us-east-1`; it's a global
resource that happens to grant visibility into resources that *are*
regional. Don't look for a region selector when creating the policy
itself.

### 2.8 Dynamic account ID in an ARN — reusability over hardcoding

```bash
arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):policy/iampolicy_anita
```

Same "derive, don't hardcode" discipline as every scripted AWS CLI
usage in this series — asking AWS which account you're actually
authenticated to, inline, makes the exact same command work
unmodified across different accounts/sessions rather than requiring a
manual account-ID substitution each time.

### 2.9 Verifying progressively deeper — existence, then version, then actual content

```text
Level 1: aws iam get-policy                                → does it exist at all?
Level 2: Policy.DefaultVersionId                              → which version is active?
Level 3: aws iam get-policy-version --version-id <that one>    → what does it ACTUALLY contain?
```

Confirming the policy exists is the weakest claim; confirming its
*actual stored JSON statement* matches the required four actions is
the strongest. Same "verify the specific field/content the requirement
names, not just resource existence" discipline running through every
AWS lab in this series — a policy existing under the right name with
the wrong actions inside it would still "exist" at Level 1.

### 2.10 Why querying the *version* matters, not just the policy

```bash
aws iam get-policy --query 'Policy.DefaultVersionId' --output text
```
```text
v1
```

IAM policies are versioned — editing a policy's document creates a new
version rather than overwriting the old one, and `DefaultVersionId`
tracks which version is currently *active*. Querying this explicitly
before fetching the actual document (`get-policy-version
--version-id <id>`) avoids accidentally inspecting a stale, non-active
version if the policy had ever been edited — not a concern for a
freshly-created policy (always `v1`), but the correct habit for any
policy that might have a history.

### 2.11 The compressed reasoning chain

```text
Requirement (customer-managed policy iampolicy_anita, Describe*-only, unattached)
   → Author the JSON: Effect=Allow, 4 specific Describe* actions, Resource="*"
   → Create via console: IAM → Policies → Create policy → JSON editor → name it
   → Confirm NOT attaching it to any user/group/role (not required — §2.1)
   → Verify existence: get-policy --policy-arn arn:aws:iam::<dynamic-account-id>:policy/iampolicy_anita
   → Verify active version: Policy.DefaultVersionId == v1
   → Verify ACTUAL content: get-policy-version --version-id v1 --query '...Statement[].Action'
   → Confirm exactly the 4 required actions, nothing more, nothing less
   → Confirm AttachmentCount: 0 is CORRECT, not a gap (§2.1)
```

---

## 3. Concepts (reference)

### 3.1 The three core IAM building blocks (Days 16–18)
User (an identity), Group (a collection of users), Policy (a document
of permissions) — three independently-creatable resources, each
useful on its own and combinable in various ways (policies attached
to users directly, or to groups, or to roles).

### 3.2 Policy document structure
`Version` (policy-language schema version, a fixed string — not a
creation date), `Statement` (one or more permission rules), each with
`Effect` (Allow/Deny), `Action` (which API calls), and `Resource`
(which targets).

### 3.3 `Describe*`/`List*`/`Get*` vs. write-verb actions
A broadly consistent AWS naming convention distinguishing read
operations from state-changing ones — useful for quickly assessing
whether a policy is read-only by scanning its action list, without
needing to cross-reference an exhaustive "these are all the dangerous
actions" list.

### 3.4 Why one `Allow`-only, read-only policy doesn't guarantee an identity is globally read-only
IAM authorization is evaluated across *every* policy attached to an
identity — a second, separately-attached policy granting write access
would still permit it, regardless of what this one policy says. Precise
language matters: "this policy grants read-only access" ≠ "any
identity holding this policy can only ever read."

### 3.5 `Resource: "*"` is sometimes the only valid scope, not automatically "too broad"
Certain read (`Describe*`) actions don't support resource-level
scoping at all — `"*"` for those specific actions is the complete,
correct, and only available scope, not a shortcut taken in place of a
narrower option that existed.

### 3.6 IAM policy versioning
Editing a managed policy's document creates a new version rather than
mutating the existing one; `DefaultVersionId` tracks which version is
currently active. Relevant for any policy that's been edited since
creation — always confirm the active version before inspecting its
content.

### 3.7 IAM as a global service (recap from Day 16)
The policy itself has no region; only the resources it grants
visibility into (here, EC2 instances/AMIs/snapshots) are regional.

---

## 4. Runbook

### 4.1 Author the policy document
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeImages",
        "ec2:DescribeTags",
        "ec2:DescribeSnapshots"
      ],
      "Resource": "*"
    }
  ]
}
```

### 4.2 Create via the console
```text
IAM → Policies → Create policy → JSON editor → paste the document above
  → Next → Policy name: iampolicy_anita → Create policy
```

### 4.3 Verify the policy exists
```bash
aws iam get-policy \
  --policy-arn arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):policy/iampolicy_anita
```
```text
PolicyName:       iampolicy_anita
PolicyId:         ANPA2V2VL2XZ6G4MHWJ4F
Arn:              arn:aws:iam::734081570291:policy/iampolicy_anita
DefaultVersionId: v1
AttachmentCount:  0
IsAttachable:     true
```

### 4.4 Verify the active version
```bash
aws iam get-policy \
  --policy-arn arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):policy/iampolicy_anita \
  --query 'Policy.DefaultVersionId' \
  --output text
```
```text
v1
```

### 4.5 Verify the actual stored permissions
```bash
aws iam get-policy-version \
  --policy-arn arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):policy/iampolicy_anita \
  --version-id v1 \
  --query 'PolicyVersion.Document.Statement[].Action' \
  --output json
```
```json
[
  [
    "ec2:DescribeInstances",
    "ec2:DescribeImages",
    "ec2:DescribeTags",
    "ec2:DescribeSnapshots"
  ]
]
```

### 4.6 Click "Check" in the lab UI to validate.

---

## 5. Final state

| Requirement | Result |
|---|---|
| Customer-managed policy `iampolicy_anita` created | ✅ |
| `Effect: Allow` | ✅ |
| Exactly the 4 required `Describe*` actions, nothing more | ✅ (verified via `get-policy-version`) |
| `Resource: "*"` | ✅ (the only valid scope for these actions) |
| `AttachmentCount: 0` | ✅ (correct — not required to attach) |

```text
AWS Account
│
└── IAM (global)
     └── Customer Managed Policy: iampolicy_anita
            │
            ├── Version: v1 (active)
            ├── Effect: Allow
            ├── Actions: DescribeInstances, DescribeImages, DescribeTags, DescribeSnapshots
            ├── Resource: *
            └── Attachments: 0 (unattached — correct for this task)
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `NoSuchEntity` when running `get-policy` | Wrong AWS account/profile currently authenticated | `aws sts get-caller-identity` to confirm which account you're actually in |
| Policy exists but lab check still fails | Extra/missing actions in the JSON, or a typo in an action name | `get-policy-version` and compare `Statement[].Action` character-for-character against the requirement |
| Attached the policy to a user "just in case" | Conflated "create the policy" with "make it immediately useful" — not required by this task | Detach if the task specifically checks for `AttachmentCount: 0`; re-read the actual requirement |
| Assumed this policy alone makes an identity fully read-only | Didn't account for other policies potentially also attached to the same identity | Precisely: this policy grants only read access; an identity's TOTAL effective permissions depend on every attached policy combined (§2.5) |
| Queried `get-policy-version` without first checking `DefaultVersionId` | Risk of inspecting a stale version if the policy had ever been edited | Always query `DefaultVersionId` first, then fetch that exact version |
| Looked for a region selector when creating the policy | Treated IAM as region-scoped like EC2/EBS | IAM is global; no region applies to the policy resource itself (§2.7) |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.11) out loud
      from "create a read-only EC2 policy named iampolicy_anita" to
      "verified via `get-policy-version` that exactly 4 actions exist."
- [ ] Explain, in one sentence, why `AttachmentCount: 0` is a correct,
      successful result for this specific task.
- [ ] Explain why this policy being "read-only" doesn't guarantee an
      identity holding it can never perform a write action.
- [ ] Explain why `Resource: "*"` is appropriate here rather than being
      a sign of overly broad access.
- [ ] Explain the difference between confirming a policy *exists* and
      confirming its *actual stored content* — which AWS CLI calls
      answer each question?
- [ ] Write a second policy from memory granting read-only access to a
      different service (e.g. S3's `ListBucket`/`GetObject`), verify
      its exact action list via `get-policy-version`, and explain why
      you chose the specific actions you did.

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
service+operation performs the read, what performs the write, how do I
verify the SPECIFIC content/field the requirement names — not just
existence." End with a compressed step-chain.>

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
