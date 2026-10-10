# Day 19 — Attach an IAM Policy to an IAM User

A KodeKloud "100 Cloud/AWS" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

This is the fourth IAM lab in this series and the one that finally
connects the other three: Day 16 created a user, Day 17 created a
group, Day 18 created a policy — all three existed independently,
unattached to anything. Today performs the actual **attachment** that
makes a policy's permissions apply to an identity.

---

## 1. Scenario

Attach the existing customer-managed policy `iampolicy_jim` to the
existing IAM user `iamuser_jim`. Both resources already exist — this
task is purely about the association between them.

```text
IAM User                 IAM Policy
iamuser_jim      ───►    iampolicy_jim
    │                          │
    │                          ▼
    └──────────────► permissions become available to iamuser_jim
```

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 Attachment is the missing link Days 16–18 never created

```text
Day 16: iamuser_kareem        (identity, unattached)
Day 17: iamgroup_mark           (collection, empty)
Day 18: iampolicy_anita          (permissions document, AttachmentCount: 0)
Day 19: attach a user TO a policy  ← the actual connecting operation
```

Every prior IAM lab deliberately stopped at "the resource exists" —
this lab is the first one where the task is specifically the
*relationship* between two already-existing resources, not the
creation of a new one. Recognizing that no new user or policy needs
to be created here (both already exist) is itself the first correct
move — don't recreate what's already there.

### 2.2 Managed policy vs. inline policy — why this matters for *how* you verify

```text
Customer-managed policy  → a standalone resource, reusable, attachable
                             to MANY identities; this lab's iampolicy_jim
Inline policy              → embedded directly INTO one identity, not a
                             separate resource, not reusable elsewhere
```

Because `iampolicy_jim` is a managed policy (created separately, Day
18's pattern), verifying the attachment means checking a *relationship*
between two independent resources (`list-attached-user-policies`) —
an inline policy would instead be queried as part of the user's own
embedded policy list (`list-user-policies`), a different API entirely.
Knowing which kind of policy you're dealing with determines which
verification command is even correct.

### 2.3 Attaching doesn't create new permissions — it associates an existing document with an identity

```text
Before: iampolicy_jim exists, AttachmentCount: 0
         iamuser_jim exists, no attached policies

Attach-user-policy

After:  iampolicy_jim exists, AttachmentCount: 1
         iamuser_jim exists, iampolicy_jim now in its attached list
```

The policy's *content* never changes during this operation — only the
*relationship* changes. This is the same "identity vs. authorization"
distinction from Day 16, now viewed from the opposite direction:
Day 16 was "create an identity with zero permissions"; today is
"connect an identity to permissions that already fully exist
elsewhere."

### 2.4 `AttachmentCount` alone doesn't tell you *who* — it only tells you *how many*

```bash
aws iam get-policy --policy-arn ...
```
```json
{ "AttachmentCount": 1, "IsAttachable": true, "DefaultVersionId": "v1" }
```

`AttachmentCount: 1` confirms the policy is attached to exactly one
identity — it does **not** confirm that identity is `iamuser_jim`
specifically. This is a precise, important limitation: `get-policy`
answers "how many things is this attached to," never "is it attached
to *this* one." Confirming the specific pairing requires a different
query entirely (§2.5).

### 2.5 `list-attached-user-policies` is the decisive check — querying from the user's side

```bash
aws iam list-attached-user-policies --user-name iamuser_jim
```

This asks the exact question the task cares about: *"what managed
policies does THIS specific user have attached?"* — querying from the
user's side (rather than the policy's side) is what actually proves
the specific pairing `iamuser_jim` ↔ `iampolicy_jim`, which
`get-policy`'s `AttachmentCount` alone could never prove by itself.

### 2.6 Dynamic account ID in the ARN — same reusability habit as Day 18

```bash
arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):policy/iampolicy_jim
```

Same pattern as Day 18 — deriving the account ID inline rather than
hardcoding it keeps the same command correct regardless of which
account/session you're actually authenticated to.

### 2.7 "Attached" doesn't mean "guaranteed working access" — recap and extension of Day 18's caveat

```text
Attached policy  →  one input to AWS's overall authorization decision
                     (other policies, explicit Denies, SCPs, etc. ALSO apply)
```

Same precise-language caveat from Day 18, now applied to the
*attachment* step specifically: confirming the attachment exists is
confirming one necessary fact, not confirming that every action
`iampolicy_jim` describes will definitely succeed for `iamuser_jim` in
every circumstance — other applicable policies/controls remain part
of the full picture.

### 2.8 Why you shouldn't modify the policy's content for this task

```text
Task: ATTACH iampolicy_jim to iamuser_jim
NOT the task: edit iampolicy_jim's permissions, create a new policy, etc.
```

The requirement is purely relational — attach exactly the existing
policy to exactly the existing user, nothing else. Editing the
policy's JSON, creating a new version, or attaching additional
unrelated policies would all be scope creep beyond what was actually
asked.

### 2.9 The compressed reasoning chain

```text
Requirement (attach iampolicy_jim to iamuser_jim — both already exist)
   → Confirm the user exists: get-user --user-name iamuser_jim
   → Confirm the policy exists: get-policy --policy-arn arn:...:policy/iampolicy_jim
   → Attach via console: IAM → Users → iamuser_jim → Permissions →
     Add permissions → Attach policies directly → iampolicy_jim
   → Verify from the POLICY side: get-policy → AttachmentCount: 1 (necessary, NOT sufficient)
   → Verify from the USER side (decisive): list-attached-user-policies --user-name iamuser_jim
   → Confirm iampolicy_jim appears in that specific user's attached list
   → Don't modify the policy's content or attach anything beyond what was asked
```

---

## 3. Concepts (reference)

### 3.1 Policy attachment as a relationship, not a resource creation
Attaching a managed policy to a user creates an *association* between
two already-existing resources — it doesn't create a new resource, and
it doesn't modify either existing resource's own content.

### 3.2 Managed policy vs. inline policy, and why it affects verification
Managed policies are standalone, reusable, and queried via
attachment-list APIs (`list-attached-user-policies`,
`list-entities-for-policy`). Inline policies are embedded in one
identity and queried differently (`list-user-policies`,
`get-user-policy`). Knowing which type you're working with determines
which verification command actually applies.

### 3.3 `AttachmentCount` vs. `list-attached-*-policies`
`AttachmentCount` (from the policy's perspective) answers "how many
identities is this attached to" — a count, not an identity list.
`list-attached-user-policies`/`list-attached-group-policies`/
`list-attached-role-policies` (from the identity's perspective) answer
"which specific policies does THIS identity have" — the correct tool
for confirming a specific pairing.

### 3.4 Attachment ≠ guaranteed effective access (recap from Day 18)
IAM's authorization decision considers every applicable policy,
explicit denies, and other controls together — confirming one
attachment exists confirms one input to that decision, not the final
outcome for every possible action.

### 3.5 Verification direction matters
The same relationship can be checked from either side (the policy's
`AttachmentCount`, or the user's attached-policies list) — but only
one direction actually answers "is this SPECIFIC pairing correct."
Checking from the wrong side can give a technically-true but
insufficient answer.

---

## 4. Runbook

### 4.1 Confirm both resources already exist
```bash
aws iam get-user --user-name iamuser_jim
```
```bash
aws iam get-policy \
  --policy-arn arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):policy/iampolicy_jim
```

### 4.2 Attach the policy via the console
```text
IAM → Users → iamuser_jim → Permissions → Add permissions
  → Attach policies directly → search/select iampolicy_jim → Add permissions
```

### 4.3 Verify from the policy's side (necessary, not sufficient)
```bash
aws iam get-policy \
  --policy-arn arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):policy/iampolicy_jim
```
```json
{
  "AttachmentCount": 1,
  "IsAttachable": true,
  "DefaultVersionId": "v1"
}
```
Confirms *something* is now attached — not yet confirming it's
`iamuser_jim` specifically.

### 4.4 Verify from the user's side (the decisive check)
```bash
aws iam list-attached-user-policies --user-name iamuser_jim
```
```text
PolicyName: iampolicy_jim
PolicyArn:  arn:aws:iam::363158248168:policy/iampolicy_jim
```
`iampolicy_jim` appears in `iamuser_jim`'s own attached-policies list —
this is the proof the specific pairing is correct.

### 4.5 Click "Check" in the lab UI to validate.

---

## 5. Final state

| Requirement | Result |
|---|---|
| `iamuser_jim` confirmed to already exist | ✅ |
| `iampolicy_jim` confirmed to already exist | ✅ |
| Policy attached to the user via the console | ✅ |
| `get-policy` shows `AttachmentCount: 1` | ✅ (necessary check) |
| `list-attached-user-policies --user-name iamuser_jim` shows `iampolicy_jim` | ✅ (decisive check) |
| No unrelated changes made to either resource | ✅ |

```text
AWS Account
│
└── IAM (global)
     ├── User: iamuser_jim
     │       │
     │       └── Attached policies: [iampolicy_jim]  ← verified HERE (decisive)
     │
     └── Policy: iampolicy_jim
             │
             └── AttachmentCount: 1  ← verified HERE (necessary, not sufficient alone)
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `NoSuchEntity` on `get-user` or `get-policy` | Wrong username/policy name, or wrong AWS account currently authenticated | `aws sts get-caller-identity`; double-check exact spelling of both names |
| `AttachmentCount: 1` but the task still fails verification | The policy is attached to a *different* identity, not `iamuser_jim` | Always verify from the user's side (`list-attached-user-policies`), never rely on the count alone (§2.4) |
| Policy not appearing in `list-attached-user-policies` | Attachment never actually completed in the console, or attached to the wrong user by mistake | Redo the attach flow in the console; re-verify immediately after |
| `AccessDenied` running any of the verification commands | The CLI's own current identity lacks IAM read permissions | Check the credentials/role currently active for the CLI session itself |
| Edited the policy's JSON "to be thorough" | Scope creep — the task only asked for attachment, not content changes | Revert any unrequested policy edits; the task is purely relational (§2.8) |
| Attached additional, unrelated policies to the user | Misread the task as "give the user more access" rather than "attach exactly this one policy" | Detach anything beyond `iampolicy_jim` if it wasn't requested |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.9) out loud
      from "attach iampolicy_jim to iamuser_jim" to "verified from the
      user's side via `list-attached-user-policies`."
- [ ] Explain, in one sentence, why `AttachmentCount: 1` on the policy
      doesn't by itself prove it's attached to `iamuser_jim`
      specifically.
- [ ] Explain the difference between a managed policy and an inline
      policy, and why that difference changes which CLI command you'd
      use to verify an attachment.
- [ ] Explain why attaching a policy doesn't change the policy's own
      content or create a new resource.
- [ ] Explain why confirming an attachment exists isn't the same as
      guaranteeing every action the policy describes will succeed for
      that user.
- [ ] Attach a second, different policy to a different user from
      memory, then verify the specific pairing from the user's side
      — not just by checking the policy's `AttachmentCount`.

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
in the order you'd actually discover it: "what resources already exist vs.
need creating, what's the actual relationship being requested, what
existing state constrains my choice, what CLI service+operation performs
the read, what performs the write, how do I verify from the DECISIVE
side, not just any side that happens to return a number." End with a
compressed step-chain.>

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
