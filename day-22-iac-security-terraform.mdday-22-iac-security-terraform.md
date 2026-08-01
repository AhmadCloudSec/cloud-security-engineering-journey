# Day 22 — Infrastructure as Code (IaC) Security with Terraform & Checkov

`Difficulty: Hands-On` · `Focus Area: Shift-Left Security, Policy-as-Code, S3 Hardening, Cross-Region DR`

---

## 🎯 Objective

Move from *reactive* cloud security (finding misconfigurations after a
resource is already live, as in Days 1–21) to *proactive* security —
scanning Infrastructure-as-Code (IaC) for misconfigurations **before**
any resource is ever created in AWS. Build a Terraform-defined S3
architecture from an intentionally insecure baseline through to a
production-grade, fully-compliant configuration using Checkov as the
policy-as-code scanning engine.

## 🧩 Environment

| | |
|---|---|
| **Platform** | AWS Free Tier |
| **Host OS** | Windows 10 + WSL2 (Ubuntu 24.04 "noble") |
| **Tools** | Terraform v1.15.8, Checkov v3.3.8 (installed via `pipx`) |
| **Scope** | S3 bucket (primary), S3 bucket (access logs), S3 bucket (cross-region replica), IAM replication role, SNS event notifications |

## 📋 Problem Statement

Every security control implemented in Phase 1–2 (GuardDuty, WAF,
IAM Identity Center, etc.) was applied **after** a resource already
existed in the AWS Console. This is the industry's "Shift Right"
pattern — security bolted on after the fact, which means a
misconfigured resource can be live (and exploitable) for the entire
window between creation and remediation.

DevSecOps inverts this: infrastructure is defined as code (Terraform),
and that code is scanned by a policy engine (Checkov) **before**
`terraform apply` ever runs. A non-compliant resource is caught on a
developer's laptop, not in production. This lab builds that workflow
from scratch and walks through the exact iteration cycle a real
engineer goes through — write code, scan, read findings, fix, rescan —
across three full iterations of the same S3 architecture.

## 🧠 Core Concepts

### Shift-Left Security

```
Traditional (Shift Right):          IaC Security (Shift Left):

  Write code                          Write code
      │                                   │
      ▼                                   ▼
  terraform apply                    checkov -d .   ← scan BEFORE apply
      │                                   │
      ▼                                   ▼
  Resource is LIVE                   Findings reviewed
      │                                   │
      ▼                                   ▼
  Manual audit finds issue            Code fixed
      │                                   │
      ▼                                   ▼
  Fix applied (resource               terraform apply
  was exposed the whole time)             │
                                           ▼
                                     Resource is LIVE (already compliant)
```

### Checkov's Three Verdicts

| Verdict | Meaning |
|---|---|
| **PASSED** | The resource's configuration satisfies the policy |
| **FAILED** | The resource's configuration violates the policy — must be fixed or explicitly suppressed |
| **SKIPPED** | A `#checkov:skip=` annotation exists, with a documented justification |

A `FAILED` check left unresolved and unexplained is treated the same
as an unreviewed vulnerability in a compliance audit. A `SKIPPED`
check with a justification comment is an **accepted risk** — a
deliberate, documented engineering decision, not an oversight.

### Public Access Block vs. Bucket Policy vs. ACL

These three S3 controls are often confused:

| Control | What It Does |
|---|---|
| **Public Access Block** | Account/bucket-level switch that overrides ACLs and policies — the strongest control |
| **Bucket ACL** | Legacy, object/bucket-level read/write grants |
| **Bucket Policy** | JSON-based fine-grained access rules |

`aws_s3_bucket_public_access_block` with all four flags set to `true`
is the baseline every production bucket should have, regardless of
what its policy or ACL says — this was the single highest-severity
finding in this lab's first scan.

## ✅ Implementation

### Iteration 1 — Intentionally Insecure Baseline

Created a minimal S3 bucket with a public-access-block resource whose
flags were all explicitly set to `false`, to observe Checkov's
baseline findings on a "beginner mistake" configuration:

```hcl
resource "aws_s3_bucket" "insecure_bucket" {
  bucket = "ahmad-security-lab-demo-bucket-2026"
}

resource "aws_s3_bucket_public_access_block" "example" {
  bucket = aws_s3_bucket.insecure_bucket.id
  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}
```

**First scan result:** 5 passed, 11 failed. Failures grouped into
three categories:
- **Public access** (`CKV_AWS_53/54/55/56`, `CKV2_AWS_6`) — bucket
  explicitly open to the internet
- **Encryption** (`CKV_AWS_145`) — no KMS encryption at rest
- **Resilience/visibility** (`CKV_AWS_18`, `CKV_AWS_21`, `CKV_AWS_144`,
  `CKV2_AWS_61`, `CKV2_AWS_62`) — no logging, versioning, lifecycle,
  replication, or event notifications

### Iteration 2 — Baseline Hardening

Rebuilt the bucket with public access blocked, KMS encryption,
versioning, a dedicated log-target bucket, and a lifecycle rule.
Reduced findings from 11 to 3 remaining gaps:
`CKV_AWS_300` (no abort-incomplete-upload rule), `CKV2_AWS_62` (no
event notifications), `CKV_AWS_144` (no cross-region replication).

### Iteration 3 — Production-Grade, Multi-Region Architecture

Rather than suppress the two remaining "enterprise-only" findings,
implemented them properly to reflect a real production posture:

- **Second AWS provider alias** (`aws.west`, `us-west-2`) for
  disaster-recovery replication
- **`aws_s3_bucket_replication_configuration`** with a dedicated
  **IAM role** scoped to only the replication actions it needs
  (`GetReplicationConfiguration`, `ReplicateObject`,
  `ReplicateDelete` — no wildcard permissions)
- **SNS topic + topic policy** wired to `aws_s3_bucket_notification`
  for `ObjectCreated`/`ObjectRemoved` events, with the topic
  encrypted (`CKV_AWS_26`) and its policy scoped to only accept
  publishes from the specific bucket ARN (`CKV_AWS_169`, `CKV_AWS_385`)
- **`abort_incomplete_multipart_upload`** added to both bucket
  lifecycle rules

**Result:** 54 passed, 5 failed — all 5 remaining failures were on the
**log bucket** and **replica bucket**, not the primary bucket.

### Final Step — Documented Risk Acceptance

Rather than force every secondary bucket into full architectural
parity with the primary bucket, applied targeted fixes and
justified suppressions:

- **Fixed:** added a lifecycle rule to the replica bucket
  (legitimate, low-cost gap)
- **Suppressed with justification** — log bucket and replica bucket
  event notifications (`CKV2_AWS_62`): no consumer exists for
  "a log file was written" events
- **Suppressed with justification** — replica bucket access logging
  (`CKV_AWS_18`): primary bucket is already logged; duplicating
  logging on a DR replica adds cost without security value
- **Suppressed with justification** — log bucket cross-region
  replication (`CKV_AWS_144`): logs already have a 365-day lifecycle
  expiration; replication is not required for this workload's
  compliance posture

```hcl
#checkov:skip=CKV2_AWS_62:Log/replica buckets don't need event notifications - no application consumes these events
#checkov:skip=CKV_AWS_18:DR replica bucket - primary bucket already has access logging; duplicate logging adds cost without security value
resource "aws_s3_bucket" "replica_bucket" {
  provider = aws.west
  bucket   = "ahmad-security-lab-replica-2026"
}
```

**Final scan target:** 0 FAILED, remaining findings shown as SKIPPED
with justification text visible in the scan output.

## 🐛 Troubleshooting Log

### Issue: Checkov scan showed three buckets (`insecure_bucket`, `secure_bucket`, `log_bucket`) when only one was expected

**Root cause identified.**
The lab folder was created with `mkdir -p ~/cloud-security-labs/day22-iac-security`
and populated with the insecure baseline via `cd` into that folder.
On the *next* edit, `nano main.tf` was run from the **home directory**
(`~`) instead of the lab folder — creating a second, separate
`~/main.tf` containing the hardened version, while the original
insecure file remained untouched in the lab folder. Running
`checkov -d .` from `~` recursively scanned **both** files, merging
unrelated findings from two different iterations into one confusing
report.

**Fix:** Removed the lab directory entirely, recreated it, and moved
the correct file into place explicitly:
```bash
cd ~
rm -rf cloud-security-labs/day22-iac-security
mkdir -p cloud-security-labs/day22-iac-security
mv main.tf cloud-security-labs/day22-iac-security/main.tf
cd cloud-security-labs/day22-iac-security
```

**Lesson:** `checkov -d .` scans **every** `.tf` file under the target
directory recursively — always confirm the current working directory
(`pwd`) before scanning, especially after any file was edited from a
different shell location. A stray file from an earlier iteration is a
silent source of misleading scan results.

### Issue: `apt install terraform` reported "command not found," suggesting `snap install --classic`

**Root cause identified.**
The HashiCorp APT repository and GPG key were added correctly, and
`apt update` succeeded, but the actual `apt install terraform -y`
install command was skipped — jumping straight to `terraform -version`
after only adding the repository. Since the package was never
installed, `apt` had nothing to run, and Ubuntu suggested the `snap`
package as a fallback.

**Fix:** Ran the missing install step explicitly:
```bash
sudo apt install terraform -y
```
Deliberately avoided `snap install terraform --classic` — classic
confinement snaps run outside the standard sandbox with broader system
access, which is unnecessary risk for a CLI tool with a proper APT
distribution channel available.

## 🧠 Key Concepts Applied

**Scanning Is Not a One-Time Gate** — This lab went through three full
scan-fix-rescan cycles on the same resource, each surfacing a
different tier of finding (basic public access → encryption/lifecycle
→ enterprise resilience features). Real Terraform codebases evolve the
same way; a clean scan today does not guarantee a clean scan after the
next feature is added.

**Least-Privilege IAM, Even for Automation Roles** — The replication
IAM role was scoped to exactly three action sets against exactly two
bucket ARNs, rather than a broad `s3:*`. Checkov's IAM checks
(`CKV_AWS_62`, `CKV_AWS_63`) specifically look for wildcard actions —
policy-as-code catches over-permissioned automation roles the same way
it catches open buckets.

**Not Every Resource Needs Identical Controls** — The primary bucket,
log bucket, and replica bucket have different risk profiles and
therefore different appropriate controls. Applying identical hardening
to all three regardless of purpose is not more secure — it's a sign
the engineer didn't reason about each resource's actual role. The
suppression comments exist precisely to make that reasoning visible
to a reviewer or auditor.

**Working Directory State Is Part of the Attack Surface for Tooling**
— A misplaced file didn't cause a security issue in this lab, but it
did cause an entire scan cycle to report incorrect, merged results.
The same class of mistake in a CI/CD pipeline (scanning a stale
checkout, or a leftover file from a previous branch) produces false
confidence — a "PASSED" pipeline that never actually scanned the
current code.

## 📌 Key Takeaway

> Shift-left security means the scanner runs against the *blueprint*,
> not the *building*. Every finding in this lab — public access,
> missing encryption, missing replication — was caught and fixed
> without a single AWS resource ever being created insecurely. The
> five final "failures" were not oversights but resources with
> genuinely different risk profiles, resolved via either a real fix
> (replica lifecycle) or a documented, justified suppression — not a
> blanket exemption.

## 🌍 Real-World Relevance

Production Terraform codebases at any real company go through this
exact cycle inside a CI/CD pipeline: a pull request triggers Checkov
(or a commercial equivalent like Bridgecrew/Prisma Cloud), and a
`FAILED` check on a critical policy blocks the merge until fixed or
explicitly suppressed with a reviewed justification. The distinction
between "fix it" and "suppress it with a reason" made in this lab for
the log and replica buckets is precisely the judgment call a Cloud
Security Engineer is expected to make and document during real code
review — blanket-fixing everything is as much a red flag in an audit
as blanket-ignoring everything.

## 🔗 References
- HashiCorp Documentation — *Terraform AWS Provider*
- Checkov Documentation — *Prisma Cloud / Bridgecrew Policy Index*
- AWS Documentation — *Amazon S3 Cross-Region Replication*
- AWS Documentation — *Blocking public access to your Amazon S3 storage*

---
*Previous: [← Day 21 — Detection Engineering with Sigma](./day-21-detection-engineering-sigma.md)*
