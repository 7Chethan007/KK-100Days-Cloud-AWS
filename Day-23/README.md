# Day 23 — S3 Bucket Data Migration

A KodeKloud "100 Cloud/AWS" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

Every prior resource-creation lab in this series verified a single
resource's state. This lab verifies a *migration* — a bulk operation
across thousands of objects — which means the decisive checks are
aggregate counts/sizes and a dry-run comparison, not a single
`describe-*` call.

---

## 1. Scenario

Create a private destination S3 bucket in `us-east-1` and copy every
object from the source bucket into it via `aws s3 sync`, without
deleting or modifying the source.

| Property | Value |
|---|---|
| Source bucket | `datacenter-s3-96891321` |
| Destination bucket | `datacenter-sync-96891321` |
| Region | `us-east-1` |
| Destination access | Private (public access blocked) |
| Expected object count | `3782` |
| Expected total size | `110077999` bytes |

```text
datacenter-s3-96891321  (source, untouched)
        │  aws s3 sync --dryrun  (preview)
        │  aws s3 sync            (execute)
        ▼
datacenter-sync-96891321  (destination, private)
   3,782 objects · 110,077,999 bytes  ← matches source
```

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 S3's flat namespace means "directory-looking" keys are still a single comparison problem

```text
wp-content/uploads/example.jpg   ← looks like a path, is actually just a KEY STRING
```

S3 has no real directory hierarchy — every object is identified by a
single flat key, and slashes are just characters in that string. This
matters for understanding `aws s3 sync`: it isn't walking a filesystem
tree recursively in the traditional sense, it's comparing two flat
sets of keys (plus metadata like size/timestamp) and copying whatever
differs — the "recursive" behavior is a consequence of iterating over
every key, not of any real directory structure.

### 2.2 `sync` is a *comparison*-driven copy, not an unconditional copy

```text
aws s3 cp --recursive   → copies everything, every time, unconditionally
aws s3 sync              → compares source vs destination, copies only what differs
```

`sync`'s defining property is that rerunning it is safe and
efficient — objects already present and matching at the destination
aren't retransferred. This is also exactly why the *first* `--dryrun`
before migrating and the *final* `--dryrun` after migrating are
different, meaningful checks: the first previews what will actually
move; the final one, returning nothing, is direct evidence the
destination now matches the source under `sync`'s own comparison
rules.

### 2.3 No `--delete` flag is a deliberate choice, not an oversight

```text
aws s3 sync source/ dest/              → only ADDS/UPDATES at destination
aws s3 sync source/ dest/ --delete      → ALSO removes destination objects absent from source
```

The task requires the source untouched and doesn't ask for the
destination to be pruned of anything — omitting `--delete` is what
keeps this a pure one-directional copy. Adding `--delete` here would
be scope creep against what was actually asked, the same
"don't apply more than the task calls for" discipline as every prior
lab's care around exact requirements (Day 20's exact policy name, Day
19's exact socket path).

### 2.4 A single mistyped character produces a `404`, not a silent wrong-target operation — here the typo was caught *before* any data moved

```text
Typed:     datacenter-sync-96891321v
Required:  datacenter-sync-96891321
```

Unlike Day 19 DevOps's `appsv` or Day 21 Git's `demo.gitv` — where the
typo'd target was created successfully and silently — here
`head-bucket` against the non-existent `...96891321v` returned a
clean `404`, because that bucket had never been created at all. The
lesson generalizes slightly differently here: whether a typo is
caught immediately (bucket doesn't exist → explicit error) or silently
succeeds (directory/repo *does* get created at the wrong path)
depends entirely on whether the wrong target already happens to
exist — which is exactly why checking the *exact* name before trusting
any operation's success remains necessary either way.

### 2.5 Public-access-block settings are a specific, narrow guarantee — not a complete security posture

```text
BlockPublicAcls / IgnorePublicAcls / BlockPublicPolicy / RestrictPublicBuckets
    → all true → blocks the COMMON mechanisms for making a bucket public
```

These four settings close off ACL-based and bucket-policy-based
public access specifically — they say nothing about IAM permissions
granted to specific principals, encryption configuration, or other
access paths. Confirming all four are `true` verifies exactly the
property the task asked for ("private") without overclaiming it
proves the bucket is secure in every dimension.

### 2.6 Why aggregate count + size, rather than per-object checks, is the right-scoped verification here

```text
3,782 objects, 110,077,999 bytes  ← SOURCE baseline
3,782 objects, 110,077,999 bytes  ← DESTINATION after sync
                                     + a dry run with NO pending actions
```

Checking every single object's checksum individually would be the
*most* rigorous verification, but matching aggregate counts/sizes
combined with an empty `sync --dryrun` is strong, practically
sufficient evidence for a bulk migration of this kind — three
independent signals (count, size, dry-run) agreeing is a
cost-proportionate verification, not a shortcut. The runbook is
explicit that this isn't a full cryptographic guarantee, which is the
right level of honesty about what was actually checked versus what a
stronger (and more expensive) verification would additionally need.

### 2.7 The compressed reasoning chain

```text
Requirement: copy ALL objects from source → private destination bucket, source untouched
   → export AWS_DEFAULT_REGION=us-east-1; confirm identity via sts get-caller-identity
   → head-bucket BOTH source and destination — confirm EXACT names, catch any typo early (§2.4)
   → aws s3 ls --recursive --summarize on SOURCE — record the baseline count/size (§2.6)
   → get-public-access-block on destination — confirm all four settings true (§2.5)
   → aws s3 sync ... --dryrun — PREVIEW before transferring anything (§2.2)
   → aws s3 sync ... (no --delete, one-directional only) (§2.3)
   → aws s3 ls --recursive --summarize on DESTINATION — compare against source baseline
   → aws s3 sync ... --dryrun AGAIN — empty output confirms synchronization (§2.2, §2.6)
```

---

## 3. Concepts (reference)

### 3.1 S3's flat object namespace
S3 has no real directory tree — objects are identified by flat keys
that may contain `/` characters purely as part of the string. Tools
present this visually as a hierarchy, but the underlying comparison
and iteration is always over flat keys.

### 3.2 `aws s3 sync` vs. `aws s3 cp --recursive`
`sync` compares source and destination and transfers only what
differs, making reruns cheap and safe. `cp --recursive` copies
everything unconditionally every time, with no comparison step.

### 3.3 `--dryrun`
Previews exactly what a command *would* do without performing any
actual transfer — useful both before a migration (to preview impact)
and after (as a verification: an empty dry run means nothing further
needs to move).

### 3.4 `--delete` as an explicit, separate opt-in
Without `--delete`, `sync` only adds/updates at the destination.
Adding `--delete` additionally removes destination objects that no
longer exist at the source — a materially different, more destructive
operation that must be requested explicitly.

### 3.5 Public access block settings
Four independent toggles (`BlockPublicAcls`, `IgnorePublicAcls`,
`BlockPublicPolicy`, `RestrictPublicBuckets`) that together block the
common ACL/policy-based paths to public access — a specific guarantee,
not a stand-in for IAM, encryption, or other access controls.

### 3.6 Aggregate verification vs. per-object verification
Matching counts and total byte sizes, combined with a clean
`sync --dryrun`, is strong evidence a bulk migration completed
correctly — a scoped, cost-proportionate check, distinct from (and
weaker than) verifying every individual object's checksum.

---

## 4. Runbook

### 4.1 Set the region and confirm identity
```bash
export AWS_DEFAULT_REGION=us-east-1
aws sts get-caller-identity
aws configure get region
```

### 4.2 Verify both buckets exist under their exact names
```bash
aws s3api head-bucket --bucket datacenter-s3-96891321
aws s3api head-bucket --bucket datacenter-sync-96891321
```
A typo'd destination (`datacenter-sync-96891321v`) returned `404` —
caught before any sync was attempted.

### 4.3 Record the source baseline
```bash
aws s3 ls s3://datacenter-s3-96891321/ --recursive --summarize
```
```text
Total Objects: 3782
Total Size: 110077999
```

### 4.4 Verify destination public-access blocking
```bash
aws s3api get-public-access-block --bucket datacenter-sync-96891321
```
```json
{
  "PublicAccessBlockConfiguration": {
    "BlockPublicAcls": true,
    "IgnorePublicAcls": true,
    "BlockPublicPolicy": true,
    "RestrictPublicBuckets": true
  }
}
```

### 4.5 Preview the migration
```bash
aws s3 sync \
  s3://datacenter-s3-96891321/ \
  s3://datacenter-sync-96891321/ \
  --dryrun
```

### 4.6 Execute the migration
```bash
aws s3 sync \
  s3://datacenter-s3-96891321/ \
  s3://datacenter-sync-96891321/
```
No `--delete` — one-directional copy only, source untouched.

### 4.7 Verify the destination
```bash
aws s3 ls s3://datacenter-sync-96891321/ --recursive --summarize
```
```text
Total Objects: 3782
Total Size: 110077999
```

### 4.8 Final synchronization check
```bash
aws s3 sync \
  s3://datacenter-s3-96891321/ \
  s3://datacenter-sync-96891321/ \
  --dryrun
```
Empty output — no additional copy actions identified.

---

## 5. Final state

| Check | Expected | Observed | Status |
|---|---|---|---|
| Destination bucket created, `us-east-1` | — | ✅ | Pass |
| Destination public access blocked (all 4 settings) | `true` × 4 | ✅ | Pass |
| Objects | 3,782 | 3,782 | Match |
| Total bytes | 110,077,999 | 110,077,999 | Match |
| Final `sync --dryrun` | no pending actions | empty output | Match |
| Source untouched (no `--delete` used) | — | ✅ | Pass |

```text
us-east-1
 ├── datacenter-s3-96891321     (source — unchanged)
 │       3,782 objects · 110,077,999 bytes
 └── datacenter-sync-96891321   (destination — private)
         3,782 objects · 110,077,999 bytes  ← matches source
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `404 Not Found` from `head-bucket` | Wrong/misspelled bucket name, or it doesn't exist yet | Check the exact spelling; create the bucket if genuinely missing |
| `AccessDenied` | Current identity lacks required S3 permissions | Inspect IAM policy and bucket policy |
| Destination has zero objects after sync | Migration didn't run, or failed silently | Re-check the `sync` command's own output for errors |
| Source/destination totals differ | Some objects didn't copy, or the source changed mid-migration | Rerun `sync`, re-compare `ls --summarize` on both sides |
| Dry run still lists objects after migrating | Transfer incomplete, or comparison conditions (timestamps/sizes) differ | Inspect the specific objects listed before re-running `sync` |
| Public access block shows `false` for any setting | Destination bucket not configured as required | Re-apply `put-public-access-block` with all four settings `true` |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.7) out loud
      from "export the region" to "verified via a clean final dry
      run."
- [ ] Explain, in one sentence, why S3 having a "flat namespace"
      matters for understanding what `sync` is actually comparing.
- [ ] Explain why omitting `--delete` was the correct choice for this
      specific task's requirements.
- [ ] Explain why matching object count and total size, plus an empty
      dry run, is strong evidence of a correct migration but not a
      complete guarantee.
- [ ] Explain what the four public-access-block settings protect
      against, and what they do *not* protect against.
- [ ] From memory, migrate a second (smaller) bucket to a new
      destination, verify it the same way, then additionally compare
      at least one object's ETag/checksum between source and
      destination — without copying any command from this file.

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
