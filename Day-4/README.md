# Day 4 — S3 Bucket Versioning

A KodeKloud "100 Cloud/AWS" lab. Each task in this series is treated as a
**practical exercise used to teach the underlying engineering concept**,
not a checklist of commands — read through §2–§4 before the runbook; the
commands should be *derivable*, not memorized.

---

## 1. Objective

**Problem statement (as given):**

> Data protection and recovery are fundamental aspects of data
> management. It's essential to have systems in place to ensure that
> data can be recovered in case of accidental deletion or corruption.
> The DevOps team has received a requirement for implementing such
> measures for one of the S3 buckets they are managing.
>
> The S3 bucket name is `xfusion-s3-436888308`, enable versioning for
> this bucket.
>
> Notes: Create the resources only in the `us-east-1` region.

Desired state:

```text
S3 Bucket
└── xfusion-s3-436888308
      └── Versioning: Enabled
```

**Credentials/environment** (lab-specific, rotates per session):

| Field | Value |
|---|---|
| Console URL | `https://320539308557.signin.aws.amazon.com/console?region=us-east-1` |
| Username | `kk_labs_user_557705` |
| Region constraint | `us-east-1` only |
| Access | via `aws-client` host; run `showcreds` there to retrieve credentials |

---

## 2. Why does this exist?

Imagine an S3 bucket contains:

```text
config.yaml
```

Without versioning:

```text
Upload v1
   ↓
config.yaml = v1

Upload v2
   ↓
config.yaml = v2      ← v1 is gone, not recoverable
```

The previous object isn't available as a normal object version for
recovery. With versioning:

```text
config.yaml
│
├── Version 1
└── Version 2 ← current
```

Now if an application accidentally overwrites the file, the previous
version can still be retrieved.

### Core intuition

Versioning changes the storage model from:

```text
key → one current object
```

to:

```text
key → multiple object versions
```

That's the important concept to remember.

---

## 3. Mental model

### 3.1 What happens on delete (the most important versioning concept)

Suppose:

```text
report.csv
Version A
Version B ← current
```

Someone runs:

```bash
aws s3 rm s3://bucket/report.csv
```

With versioning enabled, S3 does **not** simply destroy Version B.
Instead, S3 creates a **delete marker**:

```text
report.csv
│
├── Version A
├── Version B
└── Delete Marker ← current
```

The object appears deleted from a normal listing, but the previous
versions still exist underneath. This is why versioning is useful for
**accidental deletion and overwrite recovery**.

### 3.2 Versioning ≠ backup

Versioning protects against:

- accidental overwrite
- accidental deletion
- application mistakes

It is **not** a complete backup strategy. For needs like:

- independent copies
- cross-region recovery
- long-term retention
- protection against broader infrastructure/account failures

you need additional mechanisms layered on top:

```text
S3 Versioning
      +
Lifecycle policies
      +
Replication
      +
Backup/retention strategy
```

Mental model: **versioning provides object-level historical versions; it
is one component of a broader data-protection strategy, not the whole
strategy.**

### 3.3 Versioning states (one-way door)

```text
Never enabled  →  Enabled  ⇄  Suspended
     (default)      ↑____________|
                (cannot go back to "never enabled")
```

Once turned on, a bucket can never return to "never versioned" — only
`Enabled` or `Suspended`. Existing versions are retained even after
suspending.

---

## 4. Inspect — before changing anything

This reinforces the CLI habit from previous days: check current state
before writing.

```bash
aws s3api get-bucket-versioning \
  --bucket xfusion-s3-436888308
```

Conceptually:

```text
aws
 └── s3api
      └── get-bucket-versioning
```

We're asking: *"What is the current versioning configuration of this
bucket?"*

Possible responses:

```json
{
    "Status": "Enabled"
}
```

or:

```json
{}
```

An empty object means no versioning state has ever been set (never
`Enabled` or `Suspended`).

Note the service name: **`s3api`**, not `s3`. The `aws s3` family
(`cp`, `sync`, `ls`, `rm`) is a high-level, file-manager-style wrapper
for moving objects. `aws s3api` is the low-level, 1:1 mapping to the S3
REST API — every configuration operation (versioning, policy, lifecycle,
CORS, encryption, replication) lives here. Rule of thumb: **moving
objects → `s3`; configuring bucket behavior → `s3api`.**

---

## 5. Reason

```text
What resource am I modifying?        → an existing bucket
What subresource controls this?      → bucket "versioning configuration"
What region?                         → us-east-1 (stated constraint)
What's the current state?            → checked in §4 — never set
What do I need it to become?         → Enabled
```

The versioning subresource is a small structured object
(`{Status: ..., MFADelete: ...}`), not a boolean — that's why the CLI
takes shorthand key=value syntax (`Status=Enabled`) instead of a plain
`--enable-versioning` flag. The only valid `Status` values are `Enabled`
or `Suspended` — there is no `Disabled` (§3.3).

---

## 6. Implement

### CLI
```bash
aws s3api put-bucket-versioning \
  --bucket xfusion-s3-436888308 \
  --versioning-configuration Status=Enabled
```

This call returns no output on success — that's expected for `put-*`
calls with no response body; absence of an error is the success signal.

Notice the pattern:

```text
GET
 ↓
inspect current state

PUT
 ↓
change desired state
```

This is a general AWS CLI pattern that applies far beyond S3:

```text
describe/get
     ↓
understand current state
     ↓
create/put/update
     ↓
describe/get
     ↓
verify
```

### Console (equivalent path, used in this run)
S3 → bucket `xfusion-s3-436888308` → **Properties** tab → **Bucket
Versioning** → Edit → Enable → Save changes. Same effect as the CLI call
above, just via the UI.

---

## 7. Verify

### CLI verification actually run for this lab:

```text
aws-client ~ ➜  aws s3api get-bucket-versioning \
  --bucket xfusion-s3-436888308
{
    "Status": "Enabled"
}
```

### Console verification:
Properties tab confirms:

```text
Bucket Versioning
Status: Enabled
```

Full chain:

```text
Requirement
     ↓
Enable versioning
     ↓
AWS configuration changed
     ↓
CLI verification  → {"Status": "Enabled"}
     ↓
Console verification → Bucket Versioning: Enabled
     ↓
Lab complete
```

---

## 8. Troubleshooting

| Symptom                                                | Cause                                                          | Fix                                                                                   |
| --------------------------------------------------------| ----------------------------------------------------------------| ---------------------------------------------------------------------------------------|
| `An error occurred (NoSuchBucket)`                     | Bucket name typo, or wrong account/region                      | Double-check bucket name matches exactly; S3 names are case-sensitive                |
| `get-bucket-versioning` returns `{}`                   | Versioning was never enabled (this is normal, not an error)    | Proceed to `put-bucket-versioning`                                                    |
| `An error occurred (AccessDenied)`                     | Credentials not loaded/expired, or IAM policy lacks `s3:PutBucketVersioning` | Re-run `showcreds` on `aws-client`; confirm the lab user has S3 permissions |
| `put-bucket-versioning` succeeds but lab check fails   | Wrong bucket targeted, or `Status` typo (`enabled` vs `Enabled`) | Re-run §7 verification; `Status` is case-sensitive, must be exactly `Enabled`        |
| Used `aws s3` instead of `aws s3api` and command not found | Versioning is not exposed under the high-level `s3` command family | Use `aws s3api put-bucket-versioning` (§4)                                       |

---

## 9. CLI concepts learned

Don't memorize just this command:

```bash
aws s3api put-bucket-versioning ...
```

Learn to derive it. When a task says *"enable X on resource Y,"* the
thought process should be:

```text
What AWS service?        → S3
What resource?           → Bucket
What configuration?      → Versioning
Can I inspect it?        → get-bucket-versioning
How do I modify it?      → put-bucket-versioning
How do I prove it worked? → get-bucket-versioning
```

Other reusable patterns from this lab:
- `aws <service> <operation>` — read ops are `get-*`/`describe-*`/`list-*`;
  write ops are `put-*`/`create-*`/`update-*`/`delete-*`.
- Structured subresources (versioning, lifecycle, policy) take
  shorthand `Key=Value` syntax on the CLI, not plain booleans.
- `s3` (file-manager convenience) vs `s3api` (raw REST API bindings) —
  configuration always lives under `s3api`.

---

## 10. Generalization

This "inspect → reason → implement → verify" rhythm is identical to Day
3's subnet lab (`describe-vpcs` → reason about CIDR → `create-subnet` →
`describe-subnets` to verify) — the same shape recurs across almost every
AWS CLI task regardless of service:

```text
1. Understand  — what resource, what subresource, what constraints?
2. Inspect     — get/describe/list (read current state)
3. Reason      — what does current state imply about what to change?
4. Implement   — put/create/update (apply the change)
5. Verify      — get/describe/list again (confirm it landed)
```

Beyond S3, the same "versioning-like" one-way-door pattern shows up
elsewhere in AWS: DynamoDB point-in-time recovery, RDS backup retention,
CloudTrail log file validation — all are "turn on a protective history/audit
mechanism on an existing resource," inspected and toggled the same way.

---

## 11. Final state

```text
S3 Bucket: xfusion-s3-436888308
└── Versioning: Enabled
      ├── Future overwrites  → create new versions (old ones retained)
      └── Future deletes     → create delete markers (data retrievable)
```

---

## 12. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§5–§6) out loud
      from "enable versioning on bucket X" to the final verify step.
- [ ] Explain, in one sentence, the difference between `aws s3` and
      `aws s3api` and why versioning lives under the latter.
- [ ] Explain what a delete marker is and why it's not the same as data
      loss.
- [ ] Explain why `Status=Suspended` exists but `Status=Disabled` does
      not.
- [ ] Name one real-world mechanism you'd pair with versioning in
      production to build a complete data-protection strategy (§3.2).
- [ ] Repeat the full runbook end-to-end via the CLI only, confirming
      state before and after with `get-bucket-versioning`.

---

## 13. Runbook format for future lab days

Use this skeleton for `day-N/README.md`:

```markdown
# Day N — <Task>

## 1. Objective
What does the lab ask us to accomplish?

## 2. Why does this exist?
Underlying infrastructure/DevOps concept.

## 3. Mental model
Diagram explaining how the components relate.

## 4. Inspect
Commands to understand the existing state.

## 5. Reason
How we determine what needs to change.

## 6. Implement
AWS Console / CLI commands.

## 7. Verify
How we prove the desired state exists.

## 8. Troubleshooting
Errors encountered + how to reason about them.

## 9. CLI concepts learned
Important commands/options/patterns.

## 10. Generalization
How the concept applies outside this specific lab.

## 11. Final state
What the infrastructure looks like after completion.

## 12. Do it yourself (checklist)
Comprehension questions + "redo without copy-pasting" prompt.
```
