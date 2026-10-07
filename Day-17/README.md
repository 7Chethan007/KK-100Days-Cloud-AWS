# Day 17 — Create an IAM Group

A KodeKloud "100 Cloud/AWS" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

This is the direct sequel to Day 16 (IAM user) — same global, non-
regional IAM service, now one layer up the hierarchy: grouping users
instead of creating one.

---

## 1. Scenario

Create an IAM group named `iamgroup_mark`.

```text
              IAM Group
            iamgroup_mark
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
      User A    User B    User C
                  │
                  ▼
            (shared policy,
             not required by
             this specific lab)
```

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 The problem a group actually solves

```text
Without a group:                  With a group:
User A ── Policy                  iamgroup_mark
User B ── Policy                        │
User C ── Policy                  ┌─────┼─────┐
  (repeat the SAME                ▼     ▼     ▼
   attachment N times)         UserA UserB UserC
                                        │
                                   ONE policy
                                   attached ONCE
```

Attaching the same policy to every user individually doesn't scale —
every new hire means another manual attachment, and every policy
change means updating N users instead of one group. A group is purely
an organizational/management construct: it holds no permissions of its
own inherently, but anything attached to the group is inherited by
every member.

### 2.2 Group creation has the exact same identity-only nature as Day 16's user creation

```text
create-group  → establishes an IDENTITY for the group (name, GroupId, ARN)
attach-group-policy → establishes AUTHORIZATION (separate, optional, not required here)
```

Same distinction as Day 16: creating `iamgroup_mark` gives it an
existence and an ARN, nothing more. An empty group with no attached
policy and no members is a completely valid, successfully-created
resource — this lab only asks for the group to exist, so stopping
there is correct, not incomplete.

### 2.3 `"Users": []` in the verification response is informative, not an error

```bash
aws iam get-group --group-name iamgroup_mark
```
```json
{
    "Users": [],
    "Group": { "GroupName": "iamgroup_mark", ... }
}
```

An empty `Users` array means exactly what it says: no users have been
added to this group yet — which is the expected state for a freshly
created group the lab never asked you to populate. Reading an empty
array as a failure signal would be a misdiagnosis; the field to check
for success is `Group.GroupName`, not `Users`.

### 2.4 `get-group` requires an exact name — no wildcard matching

```bash
aws iam get-group --group-name iamgroup*
```
```text
ValidationError: The specified value for groupName is invalid.
```

`get-group` is an exact lookup, not a search — it has no concept of
shell-style glob expansion (and the shell itself wouldn't expand
`iamgroup*` here either, since it's inside a quoted/literal CLI
argument, not a filesystem path). The CLI genuinely needs one precise
name. Wanting to *search* for groups matching a pattern is a different
operation entirely:

```bash
aws iam list-groups \
  --query "Groups[?starts_with(GroupName, 'iamgroup')].GroupName" \
  --output table
```

`list-groups` plus a JMESPath filter (`starts_with`, or an exact `==`
test) is the correct tool when you don't already know the one exact
name to look up — the same "exact lookup vs. filtered search" split as
`--volume-ids` vs. `--filters` throughout the AWS CLI (Days 5, 8, 11,
12).

### 2.5 Cleaning up an accidental extra group — same hygiene as Day 16's extra user

```bash
aws iam delete-group --group-name iamgroup_mark_test
```

If an extra group gets created while exploring the CLI (easy to do
while testing commands), deleting the *extra* one and re-verifying the
*required* one is the same resource hygiene as cleaning up an
accidental extra IAM user in Day 16 — delete only what wasn't asked
for, never touch the one the task actually requires.

### 2.6 Verify by extracting the specific field, same pattern as every prior lab

```bash
aws iam get-group --group-name iamgroup_mark --query 'Group.GroupName' --output text
```
```text
iamgroup_mark
```

Same "project down to exactly the field the requirement names" habit
as `get-user --query 'User.UserName'` in Day 16, and every `--query`
usage in this series — a scriptable, unambiguous confirmation instead
of eyeballing a full JSON blob.

### 2.7 The compressed reasoning chain

```text
Requirement (create IAM group iamgroup_mark)
   → aws iam create-group --group-name iamgroup_mark
   → Verify: aws iam get-group --group-name iamgroup_mark
   → "Users": [] is expected — group exists, simply has no members yet (§2.3)
   → Extract the specific field: --query 'Group.GroupName' --output text
   → (if searching, not already knowing the name) list-groups + JMESPath filter, NOT a wildcard on get-group
   → Clean up any accidentally-created extra group, leave the required one intact
```

---

## 3. Concepts (reference)

### 3.1 IAM Group
A named collection of IAM users, used to apply shared policies to all
members at once rather than attaching the same policy to each user
individually. A group has no inherent permissions of its own — it's a
management/organizational construct, not a permission source until a
policy is attached to it.

### 3.2 IAM hierarchy: Users, Groups, Roles, Policies
```text
Users  → individual identities
Groups → collections of users, for shared permission management
Roles  → assumable, temporary identities (not covered by this lab)
Policies → the actual documents defining what's permitted
```
Groups and Roles both relate to Policies, but solve different
problems — Groups organize *existing* long-term users; Roles provide
*temporary*, assumable permission sets, often for services or
cross-account access.

### 3.3 `get-group` vs. `list-groups`
`get-group --group-name <exact-name>` — precise, single-resource
lookup; fails if the name isn't an exact match (no wildcards). `list-
groups` (optionally with a `--query` JMESPath filter) — a search across
every group in the account, the correct tool when you need to match a
pattern rather than a known exact name.

### 3.4 Why an empty `Users` array isn't a failure signal
`get-group`'s response always includes a `Users` field reflecting
current membership — `[]` simply means zero members right now, a
completely normal state for a newly-created group that hasn't had
anyone added to it yet.

### 3.5 Group-to-policy vs. group-to-user relationships
`attach-group-policy` connects a group to a permission document;
`add-user-to-group` connects a group to a member. Both are optional,
separate operations layered on top of the group's own existence —
neither was required for this specific lab, which only asked for the
group itself.

---

## 4. Runbook

### 4.1 Create the group
```bash
aws iam create-group --group-name iamgroup_mark
```
```json
{
    "Group": {
        "Path": "/",
        "GroupName": "iamgroup_mark",
        "GroupId": "AGPARUA3U4CTJXV4RM6YO",
        "Arn": "arn:aws:iam::111727141030:group/iamgroup_mark",
        "CreateDate": "2026-10-07T15:30:47Z"
    }
}
```

### 4.2 Verify — full detail
```bash
aws iam get-group --group-name iamgroup_mark
```
```json
{
    "Users": [],
    "Group": {
        "Path": "/",
        "GroupName": "iamgroup_mark",
        "GroupId": "AGPARUA3U4CTJXV4RM6YO",
        "Arn": "arn:aws:iam::111727141030:group/iamgroup_mark",
        "CreateDate": "2026-10-07T15:30:47Z"
    }
}
```
`"Users": []` is expected (§2.3) — the group exists, with no members
yet.

### 4.3 Verify — the specific field
```bash
aws iam get-group --group-name iamgroup_mark --query 'Group.GroupName' --output text
```
```text
iamgroup_mark
```

### 4.4 Confirm via list-groups (alternative check)
```bash
aws iam list-groups --query "Groups[?GroupName=='iamgroup_mark'].GroupName" --output text
```
```text
iamgroup_mark
```

### 4.5 (If an extra group was accidentally created) clean it up
```bash
aws iam delete-group --group-name iamgroup_mark_test
```

### 4.6 (Console-equivalent path, for reference)
```text
AWS Console → IAM → User groups → Create group → iamgroup_mark → Create group
```

### 4.7 Click "Check" in the lab UI to validate.

---

## 5. Final state

| Requirement | Result |
|---|---|
| IAM group `iamgroup_mark` exists | ✅ |
| `get-group` confirms `GroupName: iamgroup_mark` | ✅ |
| ARN correctly structured (`arn:aws:iam::<account-id>:group/iamgroup_mark`) | ✅ |
| No unintended extra groups left behind | ✅ |
| No users added, no policy attached (not required by this task) | ✅ (identity only) |

```text
AWS Account (111727141030)
    │
    └── IAM (global — no region)
         └── Group: iamgroup_mark
                │
                ├── GroupId: AGPARUA3U4CTJXV4RM6YO
                ├── Arn: arn:aws:iam::111727141030:group/iamgroup_mark
                ├── Users: []  (none yet — expected)
                └── Policies: (none attached — identity only)
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `get-group` fails with a `ValidationError` on a name containing `*` | `get-group` requires an exact name, no wildcard support | Use the exact name, or switch to `list-groups` with a JMESPath filter (§2.4) |
| `"Users": []` in the response | Expected — no members added yet | Not an error; verify via `Group.GroupName`, not `Users` |
| `create-group` fails because the group already exists | Ran creation twice, or the group was already created by a prior attempt | Verify with `get-group` instead of re-creating |
| Lab check fails despite the group existing | An extra, similarly-named group was also created, or the required name has a subtle typo | `list-groups --query 'Groups[*].GroupName'` to see the exact spelling of everything that exists |
| Accidentally created a duplicate/test group | Explored the CLI and created an extra resource along the way | `delete-group` on the extra one specifically — never the required one |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.7) out loud
      from "create IAM group iamgroup_mark" to "verified via `get-group`
      and `list-groups`."
- [ ] Explain, in one sentence, why an IAM group has no permissions of
      its own until a policy is attached to it.
- [ ] Explain why `"Users": []` in a `get-group` response isn't a sign
      something went wrong.
- [ ] Explain why `aws iam get-group --group-name iamgroup*` fails, and
      what command you'd use instead to search for groups matching a
      pattern.
- [ ] Explain the difference between adding a user to a group
      (`add-user-to-group`) and attaching a policy to a group
      (`attach-group-policy`) — are either of these required for this
      lab?
- [ ] Create a second IAM group from memory, add the `iamuser_kareem`
      user from Day 16 to it, verify membership via `get-group`, then
      remove the user and delete the group cleanly.

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
