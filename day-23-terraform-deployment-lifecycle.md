# Day 23 — Terraform Deployment Lifecycle (Init → Plan → Apply → Verify → Destroy)

`Difficulty: Hands-On` · `Focus Area: Terraform Execution Lifecycle, IAM Credential Hygiene, Incident Response`

---

## 🎯 Objective

Close the gap left at the end of Day 22, where the S3 architecture was
written and scanned with Checkov but never actually deployed. Run the
full Terraform execution lifecycle — `init`, `plan`, `apply` — against
real AWS infrastructure, verify the result in the Console, and tear it
back down with `destroy`. Along the way, respond to a real credential
exposure incident using AWS root keys, the exact way it would be
handled in a production environment.

## 🧩 Environment

| | |
|---|---|
| **Platform** | AWS Free Tier |
| **Host OS** | Windows 10 + WSL2 (Ubuntu 24.04) |
| **Tools** | Terraform v1.15.8, AWS CLI v2.36.14 (installed fresh inside WSL2) |
| **Scope** | Deploying the Day 22 architecture: 3 S3 buckets, 1 IAM role, 1 IAM role policy, 1 SNS topic, 1 SNS topic policy, plus supporting sub-resources (21 resources total) |

## 📋 Problem Statement

A Terraform configuration that only ever passes `checkov -d .` proves
the *definition* is compliant — it proves nothing about whether that
definition can actually be applied against a real AWS account: correct
provider authentication, correct IAM permissions for the identity
running Terraform, and correct resource dependency ordering are all
untested until `terraform apply` is actually run. This lab runs that
missing half of the lifecycle.

## 🧠 Core Concepts

### The Terraform Execution Lifecycle

```
terraform init      → downloads provider plugins, sets up backend
      │
      ▼
terraform plan       → dry-run: shows what WOULD change, creates nothing
      │
      ▼
terraform apply       → executes the plan, creates real resources
      │
      ▼
   [resources verified in AWS Console]
      │
      ▼
terraform destroy    → tears down everything Terraform created
```

`plan` and `apply` are deliberately separate commands. `plan` is safe
to run at any time — it never changes infrastructure. `apply` is the
only command in this lifecycle that costs money or creates a security
exposure, which is why it requires an explicit typed `yes` confirmation
rather than accepting a flag like `-y` by default.

### Why Terraform CLI Needs Its Own Credentials in WSL2

AWS CLI credentials configured in Windows PowerShell are stored under
the Windows user profile and are **not** visible inside WSL2, which
runs as a separate Linux filesystem and user context. Terraform
authenticates through the AWS CLI's credential chain, so a completely
separate `aws configure` step was required inside the WSL2 shell before
`terraform plan` could authenticate at all — the first `terraform plan`
attempt failed with `No valid credential sources found` for exactly
this reason.

### Root Credentials vs. IAM User Credentials

| | Root Account | IAM User |
|---|---|---|
| **Scope of access** | Everything, unconditionally — cannot be restricted by any policy | Only what's explicitly granted |
| **Intended use** | Account setup, billing, a handful of account-level tasks | All day-to-day operations, including CLI/Terraform |
| **Blast radius if leaked** | Total account compromise | Limited to whatever permissions were attached |

AWS's own guidance is unambiguous: root credentials should not be used
for programmatic access at all, and access keys for the root user
should generally not exist. This lab surfaced that exact anti-pattern
in real time.

## ✅ Implementation

### 1. `terraform init`
Ran from the lab directory (`~/cloud-security-labs/day22-iac-security`),
downloading the `hashicorp/aws` provider and generating
`.terraform.lock.hcl` to pin the provider version for reproducibility.

### 2. `terraform plan`
First attempt failed on a leftover syntax error from Day 22 (an
invalid `bucket` argument inside `aws_sns_topic`, previously flagged
but not yet removed from the file — removed before proceeding).

### 3. AWS CLI Setup Inside WSL2
Installed AWS CLI v2 fresh inside the WSL2 environment (separate from
the Windows PowerShell installation), since Terraform's provider
authentication runs entirely inside the WSL2 shell:
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install
```

### 4. `terraform plan` (retry) — Successful, But With Root Credentials
`aws configure` was run using an existing Access Key — which turned
out to belong to the **AWS root account**, confirmed by
`aws sts get-caller-identity` returning
`"Arn": "arn:aws:iam::185529490401:root"`. See Incident section below.

### 5. `terraform apply`
Executed after credential remediation (see below), with a scoped IAM
user. Plan showed 21 resources to add, 0 to change, 0 to destroy.
Typed `yes` at the confirmation prompt.

**Result:** `Apply complete! Resources: 21 added, 0 changed, 0 destroyed.`

### 6. Verification in AWS Console
Confirmed all three S3 buckets existed in their respective regions
(`us-east-1` for the primary and log buckets, `us-west-2` for the
replica bucket — required manually switching the Console's region
selector to see it), the IAM replication role under **IAM → Roles**,
and the SNS topic under **SNS → Topics**.

### 7. Post-Verification Cleanup
```bash
rm -rf aws/
rm -f awscliv2.zip*
```
The AWS CLI installer artifacts had been extracted directly inside the
Terraform lab folder (since the install commands were run from that
directory) and needed to be removed before the folder was fit to
commit to version control.

### 8. `.gitignore` Created
```
*.tfstate
*.tfstate.backup
.terraform/
*.tfvars
```
The state file contains resource ARNs, IDs, and potentially sensitive
configuration values in plaintext — it must never be committed to a
public (or even private) repository.

### 9. `terraform destroy`
Ran to tear down all 21 resources after verification was complete, to
avoid any ongoing cost exposure or orphaned lab resources.

**Result:** `Destroy complete! Resources: 21 destroyed.`

## 🐛 Troubleshooting Log

### Issue: `terraform plan` failed with `No valid credential sources found`

**Root cause identified.**
WSL2 is a fully separate environment from Windows — AWS CLI
credentials configured via PowerShell on the Windows side do not
propagate into the WSL2 filesystem or shell session. Terraform's AWS
provider had no credential chain to draw from inside WSL2 at all.

**Fix:** Installed AWS CLI v2 separately inside WSL2 and ran
`aws configure` in that environment specifically.

### 🚨 Security Incident: Root Access Key Used for CLI Authentication

**What happened.**
When configuring AWS CLI inside WSL2, an existing Access Key/Secret
Key pair was entered into `aws configure`. Running
`aws sts get-caller-identity` immediately afterward revealed the
identity behind that key was:
```json
{
    "Arn": "arn:aws:iam::185529490401:root"
}
```
This is the AWS account's root user — the single most sensitive
credential in the entire account, with unrestrictable full access to
every service, every resource, and billing.

**Compounding risk.**
The key was also pasted directly into a chat conversation during
troubleshooting — meaning the credential existed in at least two
places it should never have been: a root-level key, exposed in a
plaintext channel outside of AWS's own credential storage.

**Immediate remediation (in order):**
1. Navigated to root account **Security credentials** in the Console
   and deleted the exposed access key immediately — treating it as
   compromised the moment it left a secure context, regardless of
   whether it was actually misused.
2. Created a dedicated IAM user, `terraform-lab-user`, with no Console
   login access (CLI-only), scoped to `AmazonS3FullAccess`,
   `IAMFullAccess`, and `AmazonSNSFullAccess` — matching only the
   services this specific Terraform configuration touches, rather than
   `AdministratorAccess`.
3. Generated a **new** access key under that IAM user and re-ran
   `aws configure` inside WSL2 with the new credentials.
4. Re-verified identity: `aws sts get-caller-identity` now correctly
   returned `arn:aws:iam::185529490401:user/terraform-lab-user`.

**Why this matters more than the lab itself.**
Recognizing a leaked/misused root credential and executing an
immediate, correct rotation — delete first, re-provision second, never
attempt to "clean up" a compromised key by editing its permissions —
is a core incident-response reflex for any Cloud Security Engineer.
This was not a scripted exercise; it was a genuine near-miss caught
and corrected in real time.

### Issue: AWS CLI installer files left inside the Terraform project directory

**Root cause identified.**
Because the `curl` / `unzip` / `sudo ./aws/install` sequence was run
from inside `~/cloud-security-labs/day22-iac-security` rather than the
home directory, the extracted `aws/` folder and the 70MB
`awscliv2.zip` archive were created directly inside the Terraform
project — files with no relationship to the actual infrastructure code,
which would have bloated a Git commit significantly.

**Fix:** Removed both after confirming the CLI install had completed
successfully (`aws --version` had already been validated), and added
a `.gitignore` to prevent the equivalent problem with Terraform's own
generated files (`.terraform/`, `*.tfstate`) going forward.

## 🧠 Key Concepts Applied

**`plan` Before `apply`, Always** — The 21-resource plan was reviewed
in full before typing `yes`. Terraform's confirmation step exists
specifically so that a reviewed plan and an executed plan cannot
silently diverge — skipping straight to `apply -auto-approve` in a
learning context (or in early-stage automation) removes the one
built-in checkpoint against unintended changes.

**Least-Privilege Even Under Time Pressure** — After discovering the
root-key incident, the remediation path was to build a properly scoped
IAM user rather than reach for `AdministratorAccess` "to get unblocked
quickly." The correct fix and the fast fix were the same action here,
which is generally true — least privilege is rarely actually slower to
set up.

**State File as a Security Artifact, Not Just a Bookkeeping File** —
`terraform.tfstate` is functionally a snapshot of the account's
resource inventory and configuration. Treating it with the same
"never commit this" discipline as a `.env` file or credentials file is
standard practice, formalized here via `.gitignore` before the first
`git add`.

**Verify in the Console, Not Just the CLI Exit Code** — `Apply
complete` in the terminal is a claim; the S3/IAM/SNS console views are
the evidence. This same principle — never trust a single tool's
success message as final confirmation — echoes the API Gateway
debugging lesson from Day 20, where the JSON error body and the actual
HTTP status code told two different parts of the same story.

## 📌 Key Takeaway

> A Terraform configuration is only as trustworthy as the credentials
> used to run it. The most valuable lesson from this lab wasn't the
> successful `apply` — it was catching a root-level access key mid-use,
> rotating it immediately and correctly, and never treating "it hasn't
> caused a problem yet" as a reason to delay remediation. In a real
> environment, that same key sitting active for even a few extra
> minutes is the difference between a near-miss and an incident report.

## 🌍 Real-World Relevance

Root key exposure is one of the most common real-world cloud incident
patterns — checked into a public GitHub repo, pasted into a support
ticket, or left in a CI/CD log — and it's exactly why AWS Config Rules,
GuardDuty, and IAM Access Analyzer (all covered earlier in this
journey) specifically flag root account activity and unused/overly
broad access keys as high-priority findings. The correct response
sequence practiced here — **revoke first, re-provision with least
privilege second, verify identity before proceeding** — is identical to
what a real incident response runbook prescribes, just executed here
at lab scale before it ever became a real breach.

## 🔗 References
- AWS Documentation — *AWS account root user*
- AWS Documentation — *Security best practices in IAM*
- HashiCorp Documentation — *Terraform CLI: init, plan, apply, destroy*
- AWS Documentation — *Terraform on AWS provider authentication*

---
*Previous: [← Day 22 — Infrastructure as Code Security (Terraform + Checkov)](./day-22-iac-security-terraform.md)*
