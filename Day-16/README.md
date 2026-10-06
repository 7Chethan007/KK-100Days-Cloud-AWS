# Day 16 — Create an IAM User

A KodeKloud "100 Cloud/AWS" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

---

## 1. Scenario

Create an IAM user named `iamuser_kareem`, through the console, and
verify it exists via the AWS CLI.

```text
AWS Account
    │
    └── IAM
         └── Users
              └── iamuser_kareem
```

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 Identity and authorization are two separate concepts — the entire lesson of this lab

```text
Creating a user   → establishes an IDENTITY (a "who")
Attaching a policy → establishes AUTHORIZATION (a "what they can do")
```

`aws iam create-user --user-name iamuser_kareem` creates an identity
with a username, user ID, and ARN — it grants **zero** permissions by
default. A newly-created IAM user can authenticate (once credentials
exist) but cannot call a single AWS API until some policy explicitly
grants it. This lab only asked for the identity to exist, which is why
no policy attachment step is part of the requirement — conflating
"user created" with "user can do things" would be the main conceptual
trap here.

### 2.2 Console and CLI both hit the same underlying IAM API

```text
Console: IAM → Users → Create user → enter username → Create user
CLI:     aws iam create-user --user-name iamuser_kareem
```

Same relationship as every Console-vs-CLI pair in this series (EC2,
EBS, ENI) — the console is a UI wrapper over the same API call. Since
the lab specifies doing the creation via console, the CLI's role here
is purely verification, not an alternate creation path you need to
also run.

### 2.3 Verify by querying for exactly the field that matters

```bash
aws iam get-user --user-name iamuser_kareem
```
```json
{
    "User": {
        "UserName": "iamuser_kareem",
        "UserId": "AIDA36OX6JRWKMVTN44RQ",
        "Arn": "arn:aws:iam::821328497772:user/iamuser_kareem"
    }
}
```

`get-user` returning a result at all is already meaningful (the user
exists), but the requirement is specifically about the **name** — a
tighter, scriptable check extracts just that field:

```bash
aws iam get-user --user-name iamuser_kareem --query 'User.UserName' --output text
```
```text
iamuser_kareem
```

Same "project down to the one field the requirement actually specifies"
discipline as every `--query` usage throughout this series.

### 2.4 Reading an ARN — a structured, predictable identifier, not an opaque string

```text
arn:aws:iam::821328497772:user/iamuser_kareem
 │   │   │         │              │
 │   │   │         │              └── resource type/name: user/<username>
 │   │   │         └── AWS Account ID
 │   │   └── service: iam
 │   └── partition: aws
 └── literal prefix
```

**ARN = Amazon Resource Name** — AWS's universal way of uniquely
identifying any resource across any service. Once you can read one
ARN's structure, you can read any AWS ARN — the same `arn:partition:
service:region:account-id:resource` shape (region is empty here since
IAM is a global, not regional, service) recurs across every AWS
resource type.

### 2.5 IAM is global, not region-scoped — a deliberate departure from every prior lab's region caveat

Every prior AWS lab in this series (EC2, EBS, ENI, AMI, snapshots)
carried an explicit "confirm the region" step, because those resources
are region-scoped. IAM users are **account-wide**, not tied to any
particular region — there's no `--region` flag on `create-user`/
`get-user`, and a user created "in" one region-selected console session
is visible identically regardless of which region the console happens
to display. Worth noting explicitly since the instinct to check region
first is otherwise correct muscle memory from this whole series — it
just doesn't apply here.

### 2.6 Finding a user without knowing their exact ARN

```bash
aws iam list-users --query "Users[?UserName=='iamuser_kareem'].UserName" --output text
```

Same tag/name-based resource-discovery pattern as every EC2/EBS lab in
this series, applied to IAM — `list-users` plus a JMESPath filter on
`UserName` resolves a human-known name to confirmed existence, the IAM
equivalent of `describe-instances --filters "Name=tag:Name,..."`.

### 2.7 Cleaning up an accidental extra resource — the same instinct as any lab

If an extra, unintended user gets created along the way (easy to do
while clicking through a console flow), `aws iam delete-user
--user-name <name>` removes it — the same "don't leave unintended
resources lying around" hygiene as Day 15's EBS snapshot lab or any
other resource-creation task. Delete only the *extra* one; never the
one the task actually requires.

### 2.8 The compressed reasoning chain

```text
Requirement (create IAM user iamuser_kareem, verify it exists)
   → Console: IAM → Users → Create user → iamuser_kareem → Create user
   → No policy attachment — identity only is required (§2.1)
   → Verify: aws iam get-user --user-name iamuser_kareem
   → Extract the specific field: --query 'User.UserName' --output text
   → (if unsure of exact name) aws iam list-users --query "Users[?UserName=='...']..."
   → Confirm ARN structure matches arn:aws:iam::<account-id>:user/iamuser_kareem
   → Clean up any accidentally-created extra user, leave the required one intact
```

---

## 3. Concepts (reference)

### 3.1 IAM User
A long-term identity within an AWS account — has a username, a unique
`UserId`, and an ARN. Exists independently of whether any credentials
(password, access keys) or permissions (policies) have been attached
to it.

### 3.2 Identity vs. authorization
Creating a user (`create-user`) establishes *who* — an identity AWS can
recognize. Attaching a policy (`attach-user-policy`) establishes
*what that identity may do*. These are deliberately separate API
operations, and a user with zero attached policies can authenticate
but call no APIs successfully.

### 3.3 ARN structure
`arn:<partition>:<service>::<account-id>:<resource>` (region is empty
for global services like IAM). Reading this structure generically is a
transferable skill — every AWS resource, across every service, is
identified this same way.

### 3.4 IAM as a global service
Unlike EC2/EBS/VPC resources (region-scoped), IAM users/groups/roles/
policies exist account-wide, with no region dimension — a departure
from the "always confirm region first" habit built up across the rest
of this series, worth recognizing as the exception rather than
forgetting the habit entirely.

### 3.5 `list-users` with a JMESPath filter vs. `get-user`
`get-user --user-name <name>` is a direct, exact lookup — use it when
you already know the username. `list-users` with a `--query` filter is
a search — useful when confirming a user exists among many, or when
constructing a script that shouldn't assume the exact name is already
known.

---

## 4. Runbook

### 4.1 Create the user via the console
```text
AWS Console → IAM → Access management → Users → Create user
  Username: iamuser_kareem
  → Create user
```

### 4.2 Verify via CLI — full detail
```bash
aws iam get-user --user-name iamuser_kareem
```
```json
{
    "User": {
        "Path": "/",
        "UserName": "iamuser_kareem",
        "UserId": "AIDA36OX6JRWKMVTN44RQ",
        "Arn": "arn:aws:iam::821328497772:user/iamuser_kareem"
    }
}
```

### 4.3 Verify via CLI — the specific field
```bash
aws iam get-user --user-name iamuser_kareem --query 'User.UserName' --output text
```
```text
iamuser_kareem
```

### 4.4 Confirm via a list-based search (alternative check)
```bash
aws iam list-users --query "Users[?UserName=='iamuser_kareem'].UserName" --output text
```
```text
iamuser_kareem
```

### 4.5 (If an extra user was accidentally created) clean it up
```bash
aws iam delete-user --user-name iamuser_kareem2
```

### 4.6 (CLI-only equivalent, for reference)
```bash
aws iam create-user --user-name iamuser_kareem
```

### 4.7 Click "Check" in the lab UI to validate.

---

## 5. Final state

| Requirement | Result |
|---|---|
| IAM user `iamuser_kareem` exists | ✅ |
| `get-user` confirms `UserName: iamuser_kareem` | ✅ |
| ARN correctly structured (`arn:aws:iam::<account-id>:user/iamuser_kareem`) | ✅ |
| No unintended extra users left behind | ✅ |
| No policy attached (not required by this task) | ✅ (identity only) |

```text
AWS Account (821328497772)
    │
    └── IAM (global — no region)
         └── User: iamuser_kareem
                │
                ├── UserId: AIDA36OX6JRWKMVTN44RQ
                ├── Arn: arn:aws:iam::821328497772:user/iamuser_kareem
                └── Policies: (none attached — identity only)
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `get-user` returns `NoSuchEntity` | Username typo'd during creation, or wrong AWS account/credentials in use | `list-users` to see what actually exists; confirm `aws sts get-caller-identity` points at the right account |
| Assumed region mattered and spent time checking it | IAM is a global service, not region-scoped (§2.5) | No `--region` needed for any `aws iam` command |
| Lab check fails despite the user existing | An extra, unintended user was also created and something about verification is ambiguous, or the required username has a subtle typo (case, underscore vs. hyphen) | `list-users --query "Users[*].UserName"` to see the exact spelling of everything that exists |
| Attached a policy "just in case" | Conflated identity creation with authorization — not required by this task | Remove the unnecessary policy attachment if the task specifically checks for identity only; re-read the actual requirement |
| Accidentally created a duplicate/extra user | Clicked through the console flow twice, or typo'd and recreated | `delete-user` on the extra one specifically — never the required one |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.8) out loud
      from "create IAM user iamuser_kareem" to "verified via `get-user`
      and `list-users`."
- [ ] Explain, in one sentence, the difference between creating a user
      and granting that user permissions.
- [ ] Explain why IAM commands don't take a `--region` flag, unlike
      every other AWS CLI command used so far in this series.
- [ ] Break down a given ARN (any AWS resource's) into its component
      parts from memory, without looking at a reference.
- [ ] Explain the difference between `get-user --user-name <name>` and
      `list-users` with a JMESPath filter — when would you reach for
      each?
- [ ] Create a second IAM user from memory via the CLI only
      (`create-user`), verify it with `get-user`, then delete it
      cleanly with `delete-user`.

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
verify the SPECIFIC field the requirement names." End with a compressed
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
