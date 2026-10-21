# Day 28 — From Dockerfile to Private Container Registry

A KodeKloud "100 Cloud/AWS" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

This is the first container lab in the series. The reasoning habits
are familiar — exact URIs matter (Day 20's policy name, Day 26's
`None` user-data check), configuration vs. outcome are separate
claims (Day 26's working-website-vs-verified-user-data), bidirectional
verification (Day 10/21's EIP checks) — but the objects are new:
images, tags, registries, and digests, and the one genuinely new
concept is that a *tag* is mutable while a *digest* is a permanent
content fingerprint.

---

## 1. Scenario

Create a private ECR repository `devops-ecr`, build a Docker image
from `/root/pyapp/Dockerfile` on `aws-client`, and push it tagged
`latest`.

| Property | Value |
|---|---|
| Repository | `devops-ecr`, private, `us-east-1` |
| Build context | `/root/pyapp` |
| Required tag | `latest` |
| Final image URI | `081866666028.dkr.ecr.us-east-1.amazonaws.com/devops-ecr:latest` |
| Verified digest | `sha256:9cbef5c52d2ab49d986d5fe1f7b360e497a597cc2693e977622c14117f321135` |

```text
Dockerfile ──docker build──▶ local image (devops-ecr:latest)
                                     │
                    aws ecr get-login-password | docker login
                                     │
                              (Docker now authenticated to ECR)
                                     │
                    docker tag  devops-ecr:latest  →  <account>.dkr.ecr.<region>.amazonaws.com/devops-ecr:latest
                                     │
                              docker push ──▶ ECR repository: devops-ecr
                                     │
                    verify: console + aws ecr describe-images (tag + digest)
```

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 Six separate stages, six separate questions — none substitutes for another

```text
Create repository → WHERE will the image live?
Build image        → WHAT artifact are we distributing?
Authenticate        → is Docker ALLOWED to talk to this registry?
Tag                 → WHICH registry+repo+tag does THIS local image map to?
Push                 → transfer the bytes
Verify               → did the EXPECTED image actually arrive?
```

Same "composable, independently-checkable building blocks" pattern
as every prior multi-service lab (IAM's four pieces, EC2/CloudWatch/
SNS in Day 25, VPC's five networking pieces in Day 27) — a repository
existing says nothing about whether any image is in it; a successful
local build says nothing about whether it was ever pushed; a
successful push says nothing about whether it landed at the *intended*
destination if the tag/URI was wrong.

### 2.2 Creating the repository doesn't create or affect the image — same "identity vs. content" split as every registry-style lab

```text
aws ecr create-repository (or Console)  → an EMPTY shelf — no image exists yet
docker build                             → the ARTIFACT, built entirely LOCALLY, unrelated to the repo existing
```

This is the exact structural pattern from Day 25 MLOps's registered
model (`fraud-detector` existing ≠ any version registered under it)
and Day 18's Git tag (creating a reference doesn't push/rerun
anything) — creating the destination and creating the thing that will
go there are always two separate, separately-verifiable actions.

### 2.3 The registry hostname encodes exactly four facts, each independently meaningful

```text
081866666028  .  dkr.ecr  .  us-east-1  .  amazonaws.com  /  devops-ecr
   ^               ^             ^                            ^
account ID    ECR Docker    AWS Region                  repository name
              endpoint
```

Getting any one of these wrong (wrong account, wrong region, wrong
repo name) produces a *different, specific* failure — e.g. pushing to
`us-west-2`'s endpoint when the repository was created in
`us-east-1` produces "repository does not exist," not a silent
success somewhere else. This is the same exact-identifier discipline
as every ARN/URI throughout this series (Day 20's IAM policy ARN, Day
22's SSH key path) — a URI isn't one opaque string, it's several
independently-checkable fields concatenated together.

### 2.4 `docker tag` adds a *second name* to an existing local image — it never rebuilds or uploads anything

```text
Before: devops-ecr:latest → image ID abc123
After:  devops-ecr:latest → abc123
        <full ECR URI>:latest → abc123   (SAME image, now with an ADDITIONAL reference)
```

This is purely local bookkeeping — exactly the same "configuring an
alias doesn't transfer anything" principle as Day 26 DevOps's
`git remote add` (registers a destination, moves zero commits) and
Day 18's `git tag -a` (names a commit, doesn't push or rerun
anything). `docker tag` is what makes the subsequent `docker push`
know *which* registry/repo/tag the already-built image should go to —
without it, `docker push` would have no ECR-shaped reference to push.

### 2.5 `docker login` via a piped token is authentication; it proves nothing about authorization to push to a *specific* repository

```text
aws ecr get-login-password | docker login ...   → "Login Succeeded" — Docker now holds valid, SHORT-LIVED credentials
docker push <repo>                                → may STILL fail with AccessDenied if the AWS identity lacks
                                                      the specific ECR push permission for THIS repository
```

This is the same "authentication establishes identity; authorization
determines what that identity may do" split drawn out explicitly in
Day 20's IAM role lab — a successful login is a necessary, not
sufficient, condition for a successful push. The token itself is also
deliberately short-lived and registry-scoped, which is why it's piped
directly into `docker login` via stdin rather than ever being stored
or displayed — exposing it would create a credential that's sensitive
specifically because it grants registry access, not console access.

### 2.6 `docker push` is layer-aware — "nothing to push" for unchanged layers is expected, not a bug

```text
Docker checks which layers the registry ALREADY HAS
→ only uploads layers the registry is MISSING
→ final metadata/manifest update associates the pushed content with the requested tag
```

A second push of an image sharing most layers with something already
in the registry can be fast specifically because of this
layer-deduplication — seeing fewer upload lines than expected on a
repeat push isn't evidence the push silently failed, it's the designed
efficiency behavior.

### 2.7 A tag is mutable; a digest is a permanent content fingerprint — this is the one genuinely new concept in the lab

```text
latest  (tag)    → can be RE-POINTED to a different image by a later push with the same tag
sha256:9cbef5...  (digest) → IDENTIFIES this exact content — CANNOT be reused for different content
```

`latest` referring to image A today and image B tomorrow (after
someone else pushes with the same tag) is completely valid, expected
behavior for a mutable tag — which is exactly why a deployment
pipeline that needs reproducibility pins to a *digest* instead, the
same precision-over-convenience tradeoff as every prior lab's "exact
identifier, not an approximate one" lesson, here made explicit as a
deliberate design choice rather than a mistake to avoid.

### 2.8 Verifying via both the Console and the CLI is the bidirectional check, applied to a registry instead of a network resource

```text
Console: ECR → devops-ecr → Images → tag + digest VISIBLE
CLI:     aws ecr describe-images --query '...'    → tag + digest CONFIRMED PROGRAMMATICALLY
```

Neither alone is as strong as both agreeing — the same "verify from
both sides" discipline as Day 10/21's EIP association (checked from
both the address record and the instance record) and Day 22's SSH
verification (interactive login, then `BatchMode=yes`) — here applied
to confirming a pushed image actually landed where (and as) intended.

### 2.9 The compressed reasoning chain

```text
Requirement: devops-ecr repo, built image pushed as :latest
   → create the repository FIRST — establishes the destination URI, creates NOTHING else (§2.2)
   → cd into the build context, inspect (ls -la) — confirm the Dockerfile is actually there
   → docker build -t devops-ecr:latest .  — LOCAL artifact only, unrelated to ECR so far
   → aws ecr get-login-password | docker login ...  — AUTHENTICATION, not authorization (§2.5)
   → docker tag devops-ecr:latest <full ECR URI>:latest  — adds a SECOND name, uploads NOTHING (§2.4)
   → docker push <full ECR URI>:latest  — layer-aware transfer (§2.6)
   → verify via BOTH Console AND aws ecr describe-images — tag AND digest (§2.8)
   → remember: tag = mutable convenience, digest = permanent content identity (§2.7)
```

---

## 3. Concepts (reference)

### 3.1 Dockerfile, image, container — three different things
A Dockerfile is build instructions. An image is the resulting
packaged artifact. A container is a running instance created from an
image. Building produces the second from the first; running produces
the third from the second.

### 3.2 Registry, repository, tag — the addressing hierarchy
A registry (e.g. ECR) hosts repositories; a repository (e.g.
`devops-ecr`) groups related images; a tag (e.g. `latest`) is a
human-readable pointer to one specific image within that repository.

### 3.3 `docker tag` as a local alias operation
Adds an additional name/reference to an existing local image without
rebuilding it or transferring any data — purely local metadata,
exactly like Git's tag/remote-add operations elsewhere in this series.

### 3.4 Authentication vs. authorization, for Docker/ECR specifically
`docker login` (via a piped ECR token) establishes that Docker is
speaking to the registry as a recognized AWS identity. Whether that
identity can actually push to a *specific* repository is a separate,
IAM-permission-governed question.

### 3.5 Layer-based push efficiency
Docker only uploads image layers the registry doesn't already have —
a repeat or incremental push can complete quickly without indicating
failure.

### 3.6 Tag mutability vs. digest permanence
A tag can be repointed to different image content by a later push
with the same tag name. A digest is a cryptographic hash of the
content itself and can never refer to different content — the basis
for reproducible, pinned deployment references.

---

## 4. Runbook

### 4.1 Create the private ECR repository
```text
ECR → Private registry → Repositories → Create repository
  → Visibility: Private
  → Name: devops-ecr
  → Image tag mutability: Mutable
  → Create repository
```
```text
081866666028.dkr.ecr.us-east-1.amazonaws.com/devops-ecr
```

### 4.2 Inspect the build context
```bash
cd /root/pyapp
ls -la
```
```text
app.py
Dockerfile
requirements.txt
```

### 4.3 Build the image
```bash
docker build -t devops-ecr:latest .
docker images devops-ecr
```

### 4.4 Authenticate Docker to ECR
```bash
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  081866666028.dkr.ecr.us-east-1.amazonaws.com
```
```text
Login Succeeded
```

### 4.5 Tag the image with the full ECR destination
```bash
docker tag devops-ecr:latest \
  081866666028.dkr.ecr.us-east-1.amazonaws.com/devops-ecr:latest
```

### 4.6 Push the image
```bash
docker push 081866666028.dkr.ecr.us-east-1.amazonaws.com/devops-ecr:latest
```
```text
latest: digest: sha256:9cbef5c52d2ab49d986d5fe1f7b360e497a597cc2693e977622c14117f321135 size: ...
```

### 4.7 Verify in the Console
```text
ECR → Private repositories → devops-ecr → Images
  → tag: latest
  → digest: sha256:9cbef5c52d2ab49d986d5fe1f7b360e497a597cc2693e977622c14117f321135
```

### 4.8 Verify via the CLI
```bash
aws ecr describe-images \
  --repository-name devops-ecr \
  --region us-east-1 \
  --query 'imageDetails[*].[imageTags,imageDigest]' \
  --output table
```
```text
latest
sha256:9cbef5c52d2ab49d986d5fe1f7b360e497a597cc2693e977622c14117f321135
```

---

## 5. Final state

| Requirement | Result |
|---|---|
| Private repository `devops-ecr` created, `us-east-1` | ✅ |
| Image built locally as `devops-ecr:latest` | ✅ |
| Docker authenticated to ECR | ✅ |
| Image tagged with the full ECR URI | ✅ |
| Image pushed successfully | ✅ |
| `latest` tag confirmed in Console + CLI | ✅ |
| Digest confirmed in Console + CLI | ✅ |

```text
081866666028.dkr.ecr.us-east-1.amazonaws.com/devops-ecr
 └── latest → sha256:9cbef5c52d2ab49d986d5fe1f7b360e497a597cc2693e977622c14117f321135
```

---

## 6. Troubleshooting

| Symptom | Layer | Cause | Fix |
|---|---|---|---|
| `Dockerfile not found` | Build | Wrong working directory | `pwd`, `ls -la` before building |
| `no basic auth credentials` | Auth | Docker never authenticated, or token expired | Re-run the `get-login-password \| docker login` pipeline |
| `AccessDeniedException` | Authorization | Identity authenticated but lacks the specific ECR permission | Check IAM policy for this identity/repository |
| `repository does not exist` | Addressing | Wrong account, region, or repo name in the URI | Re-verify all four URI components (§2.3) |
| `tag does not exist` locally | Tagging | Local image reference typo'd | `docker images`, `docker image inspect` |
| Push succeeds but image missing from ECR | Verification | Pushed to the wrong URI, or verification checked the wrong repo/region | Re-check the exact URI used in `docker push` vs. what's being inspected |
| Repeat push uploads almost nothing | — | Expected — layer deduplication (§2.6) | Not a failure; confirm the final tag/digest regardless |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.9) out loud
      from "create the repository" to "verify via Console + CLI."
- [ ] Explain, in one sentence, why creating the ECR repository has
      no effect on whether any Docker image exists.
- [ ] Explain why `docker tag` doesn't upload anything, and what
      command actually does the uploading.
- [ ] Explain why a successful `docker login` doesn't guarantee a
      successful `docker push`.
- [ ] Explain the difference between a tag and a digest, and why a
      production deployment might prefer pinning to the digest.
- [ ] Decode a full ECR image URI into its four components from
      memory, without looking at §2.3.
- [ ] From memory, build a second image, push it under a *version*
      tag (not `latest`), then push the *same* image again under
      `latest`, and confirm via `describe-images` that both tags now
      point to the same digest — without copying any command from
      this file.

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
