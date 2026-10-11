# Day 20 — Create an IAM Role for EC2 with a Permissions Policy

A KodeKloud "100 Cloud/AWS" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

This is the fifth IAM lab in the series — Day 16 created a user, Day
17 a group, Day 18 a policy, Day 19 attached that policy to a user.
This lab introduces a fourth IAM building block, the **role**, and its
own two-sided configuration: *who* may assume it (trust policy) versus
*what* it can do once assumed (permissions policy) — the role
equivalent of Day 19's "attachment exists ≠ attached to the specific
resource you meant."

---

## 1. Scenario

Create an IAM role `iamrole_john`, trusted by the EC2 service, with
the managed policy `iampolicy_john` attached — using the AWS Console
for creation, then the CLI for precise verification.

| Property | Required value |
|---|---|
| IAM role name | `iamrole_john` |
| Trusted entity | AWS service — EC2 |
| Permissions policy | `iampolicy_john` |
| AWS account | `905418329946` |

```text
EC2 service (ec2.amazonaws.com)
        │  sts:AssumeRole  (trust policy — WHO)
        ▼
   iamrole_john
        │  attached managed policy (permissions policy — WHAT)
        ▼
   iampolicy_john
```

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 A role answers two independent questions, not one

```text
Trust policy:        WHO may assume this role?        → Principal: ec2.amazonaws.com
Permissions policy:  WHAT can the role do once assumed? → iampolicy_john's statements
```

These are two separate documents attached to the same role, and
neither substitutes for the other. A role trusted by EC2 with zero
permissions policies is a valid, creatable resource — it would just
let EC2 assume an identity that can do nothing. This split is the
role-specific version of the "identity vs. authorization" distinction
carried through every IAM lab since Day 16: creating the identity
(user/group/role) and attaching authorization (policy) are always
separate, separately-verifiable steps.

### 2.2 Why a role instead of embedding credentials

```text
Bad:  application code contains a long-lived access key + secret key
Good: EC2 instance assumes a role → receives short-lived, auto-rotated credentials
```

A role exists specifically to avoid embedding permanent IAM user
credentials inside application code or instance configuration — the
EC2 instance obtains temporary credentials by assuming the role, via
the instance metadata service, rather than by having a secret baked in
anywhere. This lab only builds the role itself; actually wiring it to
a running instance (an instance profile) is a separate, later step
that wasn't required here (§8 scopes this explicitly).

### 2.3 Choosing "EC2" as the trusted entity sets the `Principal`, nothing else

```json
{
  "Effect": "Allow",
  "Principal": { "Service": "ec2.amazonaws.com" },
  "Action": "sts:AssumeRole"
}
```

Selecting "AWS service → EC2" in the console wizard only determines
the trust policy's `Principal.Service` value — it does not attach any
instance, grant any permission, or do anything beyond define who is
*allowed to ask* to assume this role. This is a narrower, more literal
reading of what a console wizard step actually changes, the same
discipline as Day 17's "`create-group` establishes identity only,
nothing about membership or permissions."

### 2.4 Selecting the exact policy name matters — a lookalike name doesn't satisfy the requirement

```text
Required:  iampolicy_john
Not:       any similarly-named or default-looking policy
```

The lab requirement names one specific managed policy by exact name.
Attaching a different policy — even one that looks plausible in a long
console list — produces a role that exists and has *some* permissions,
but not the one the requirement actually asks for. Same exact-name
discipline as Day 18's "exact lookup vs. filtered search" distinction,
just applied to a console selection instead of a CLI argument.

### 2.5 Creation and attachment are separate console actions — and separately verifiable facts

```text
Create role (with EC2 trust) → role exists, trust policy correct
Attach iampolicy_john          → separate action, can be missed or skipped
```

The lab's own narrative demonstrates this directly: the role was
created, but the first verification pass found it with **no**
attached policies — the attachment step hadn't actually completed
during initial creation. This is the console equivalent of Day 19's
"`AttachmentCount` alone doesn't tell you who" — *a role existing* and
*a role having the required policy attached* are two separate claims,
and only checking the first one would have missed the gap entirely.

### 2.6 `get-role` and `list-attached-role-policies` verify different things

```text
get-role                       → proves the role EXISTS, and shows its trust policy
list-attached-role-policies    → proves WHICH managed policies are attached (or none)
```

`get-role` succeeding says nothing about attached permissions —
that's a structurally separate piece of the role's state, retrieved by
a different API call entirely. Treating a successful `get-role` as
proof the lab requirement is met would be exactly the kind of
"response accepted ≠ fully configured" mistake flagged throughout the
AWS side of this series (EBS, EC2, EIP, AMI).

### 2.7 `--query` on `AssumeRolePolicyDocument` extracts the trust relationship precisely

```bash
aws iam get-role --role-name iamrole_john \
  --query 'Role.AssumeRolePolicyDocument.Statement[*].Principal.Service' \
  --output json
```

Rather than eyeballing the full role JSON for the right nested field,
JMESPath projects straight to `Principal.Service` across every trust
statement — the same "project to the exact field the requirement
names" habit as every `--query` usage since Day 16's
`get-user --query 'User.UserName'`.

### 2.8 Filtering `list-attached-role-policies` by exact name closes the loop

```bash
aws iam list-attached-role-policies --role-name iamrole_john \
  --query "AttachedPolicies[?PolicyName=='iampolicy_john'].PolicyName" \
  --output table
```

An empty `AttachedPolicies` array is unambiguous; a non-empty one
still needs the exact-name filter to confirm it's *this specific*
policy and not some other managed policy that happens to also be
attached. This mirrors Day 19's `list-attached-user-policies` being
the decisive check, just scoped to a role instead of a user.

### 2.9 Confirm the identity making the CLI calls, not just the resource being inspected

```bash
aws sts get-caller-identity
```

IAM resources are account-scoped — verifying against the wrong account
would either show nothing or, worse, show a different, unrelated role
that happens to share a name. Checking the calling identity first is
the same "confirm context before trusting a lookup" discipline as
every `aws sts get-caller-identity` call used for dynamic account IDs
throughout the AWS CLI series.

### 2.10 The compressed reasoning chain

```text
Requirement: EC2-trusted role iamrole_john with iampolicy_john attached
   → Console: Create role → AWS service → EC2 (sets trust Principal only, §2.3)
   → Console: attach the EXACT policy iampolicy_john (§2.4)
   → Console: name the role iamrole_john, create
   → Verify identity first: aws sts get-caller-identity (§2.9)
   → aws iam get-role → confirms role EXISTS + trust policy (§2.6)
   → aws iam list-attached-role-policies → confirms attachment SEPARATELY (§2.5, §2.6)
   → if empty: Console → Permissions → Add permissions → Attach policies → retry
   → --query to extract the specific Principal.Service / PolicyName fields (§2.7, §2.8)
```

---

## 3. Concepts (reference)

### 3.1 IAM role vs. IAM user
A role has no permanent credentials of its own — it's assumed
temporarily by an authorized principal (a service, another account, a
federated identity), which then receives short-lived credentials. A
user has long-lived credentials tied directly to it. Both can have
permissions policies attached the same way.

### 3.2 Trust policy (`AssumeRolePolicyDocument`)
Defines which principals are allowed to call `sts:AssumeRole` against
this role. It is a separate document from the role's permissions
policies and answers only "who may assume this," never "what can they
do afterward."

### 3.3 Permissions policy
Defines what actions are allowed or denied once the role is assumed —
the same kind of managed/inline policy document used for users and
groups in Days 18-19, just attached to a role instead.

### 3.4 `get-role` vs. `list-attached-role-policies`
`get-role` returns the role's own metadata and trust policy.
`list-attached-role-policies` returns a separate list of managed
policies attached to it. A role can exist with a correct trust policy
and still have zero attached permissions policies — these are
independently-queried, independently-failable facts.

### 3.5 Role creation vs. role usage (instance profile)
Creating a role and configuring its trust/permissions is distinct from
actually associating it with a running EC2 instance via an instance
profile, which is in turn distinct from the application successfully
using the resulting temporary credentials. This lab only required the
first of those three stages.

---

## 4. Runbook

### 4.1 Create the role via the AWS Console
```text
AWS Console → IAM → Roles → Create role
  → Trusted entity type: AWS service
  → Use case: EC2
  → Permissions: search and select iampolicy_john
  → Role name: iamrole_john
  → Create role
```

### 4.2 Fix a missed attachment (as encountered in this lab)
```text
IAM → Roles → iamrole_john → Permissions tab
  → Add permissions → Attach policies
  → Select iampolicy_john → Attach policy
```

### 4.3 Verify the calling identity
```bash
aws sts get-caller-identity
```
```text
arn:aws:iam::905418329946:user/kk_labs_user_183099
```

### 4.4 Verify the role exists and its trust relationship
```bash
aws iam get-role --role-name iamrole_john
```
```json
{
    "RoleName": "iamrole_john",
    "Arn": "arn:aws:iam::905418329946:role/iamrole_john"
}
```
Trust policy excerpt:
```json
{
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
}
```

### 4.5 Verify attached policies (before the fix)
```bash
aws iam list-attached-role-policies --role-name iamrole_john
```
```json
{
    "AttachedPolicies": []
}
```
Empty — the gap this lab's §2.5 is built around.

### 4.6 Verify attached policies (after attaching via Console)
```bash
aws iam list-attached-role-policies --role-name iamrole_john
```
```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "iampolicy_john",
            "PolicyArn": "arn:aws:iam::905418329946:policy/iampolicy_john"
        }
    ]
}
```

### 4.7 Extract the trust principal precisely
```bash
aws iam get-role --role-name iamrole_john \
  --query 'Role.AssumeRolePolicyDocument.Statement[*].Principal.Service' \
  --output json
```
```json
["ec2.amazonaws.com"]
```

### 4.8 Confirm the exact attached policy name
```bash
aws iam list-attached-role-policies --role-name iamrole_john \
  --query "AttachedPolicies[?PolicyName=='iampolicy_john'].PolicyName" \
  --output table
```
```text
--------------------------
|ListAttachedRolePolicies|
+------------------------+
|  iampolicy_john        |
+------------------------+
```

### 4.9 Click "Check" in the lab UI to validate.

---

## 5. Final state

| Requirement | Result |
|---|---|
| Correct AWS account (`905418329946`) | ✅ |
| Role `iamrole_john` exists | ✅ |
| Trust policy principal is `ec2.amazonaws.com` | ✅ |
| `iampolicy_john` attached | ✅ (after fixing the initial miss) |
| Exact-name filter confirms the specific policy | ✅ |

```text
AWS Account (905418329946)
    └── IAM (global — no region)
         └── Role: iamrole_john
                ├── Trust policy → Principal.Service: ec2.amazonaws.com
                └── Attached policy: iampolicy_john
                       (arn:aws:iam::905418329946:policy/iampolicy_john)
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `list-attached-role-policies` returns `[]` | Role created but policy attachment step skipped/missed | Console → role → Permissions → Add permissions → Attach policies |
| `get-role` succeeds but the lab check still fails | Verified only existence, not attached permissions | Separately run `list-attached-role-policies` |
| Trust policy shows the wrong service principal | Wrong "use case" selected during console creation | Re-check `AssumeRolePolicyDocument.Statement[*].Principal.Service`; recreate trust policy if wrong |
| CLI results look unrelated to Console state | CLI authenticated against a different AWS account | `aws sts get-caller-identity` first, confirm account matches |
| Role has the policy but an application using it still gets `AccessDenied` | Policy statements, resource ARNs, explicit denies, or permissions boundaries not satisfying the specific action | Inspect the policy document's statements, not just whether it's attached — out of scope for this lab |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.10) out loud
      from "create role `iamrole_john`" to "verified via `get-role` and
      `list-attached-role-policies`."
- [ ] Explain, in one sentence, why a role's trust policy and
      permissions policy answer two different questions.
- [ ] Explain why `get-role` succeeding doesn't prove the required
      policy is attached.
- [ ] Explain what selecting "EC2" as the trusted entity actually
      changes about the role, and what it does *not* do.
- [ ] Explain the difference between creating this role and actually
      using it from a running EC2 instance (instance profile).
- [ ] From memory, create a second role trusted by a different AWS
      service (e.g. Lambda), attach a policy of your choice, and
      verify both the trust principal and the attachment via the CLI
      — without copying any command from this file.

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
