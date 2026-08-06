# Day 25 — Secrets Management (git-secrets + AWS Secrets Manager)

`Difficulty: Hands-On` · `Focus Area: Pre-Commit Secret Scanning, Secrets-as-Data, Terraform Sensitive Values`

---

## 🎯 Objective

Build a two-layer defense against credential leakage directly connected
to the real incident from Day 23, where an AWS root access key was
briefly exposed in a plaintext channel. Install `git-secrets` to block
credential patterns at commit time, then use AWS Secrets Manager with
Terraform's `data` source pattern to prove a secret's *value* never
needs to exist in version-controlled code — only its *name* does.

## 🧩 Environment

| | |
|---|---|
| **Platform** | AWS Free Tier (Secrets Manager) |
| **Host OS** | Windows 10 + WSL2 (Ubuntu 24.04) |
| **Tools** | `git-secrets` (AWS Labs), Terraform v1.15.8, AWS CLI v2.36.14 |
| **Repo** | `terraform-security-pipeline` |

## 📋 Problem Statement

Day 23 demonstrated, in real time, how easily a credential can end up
somewhere it shouldn't — a chat window, in that case. The exact same
failure mode applies to source code: a password or access key typed
into a `.tf`, `.env`, or config file and committed becomes permanently
part of the repository's history the moment it's pushed, discoverable
by anyone with read access (or, for public repos, by automated scanning
bots that operate continuously). The fix is not "be more careful" — it's
building tooling that makes the mistake structurally difficult to make.

## 🧠 Core Concepts

### Two Independent Layers of Defense

```
Layer 1 — git-secrets (prevents the leak from ever being committed)
    Developer types a secret into a file
        │
        ▼
    git add . && git commit
        │
        ▼
    pre-commit hook runs git-secrets --scan automatically
        │
        ├── pattern match (e.g. AKIA...) → commit BLOCKED
        └── no match → commit proceeds

Layer 2 — AWS Secrets Manager + Terraform data source
    (prevents the secret from ever needing to be in the file at all)
    Terraform code contains only a secret NAME
        │
        ▼
    terraform plan/apply resolves the actual value from AWS at runtime
        │
        ▼
    Value never appears in .tf files, git history, or (by default)
    plan/apply output
```

These layers solve different problems. Layer 1 catches a mistake after
it's made but before it becomes permanent. Layer 2 removes the
opportunity for the mistake in the first place by giving the secret a
home outside the codebase entirely.

### Why `git-secrets --register-aws` Allow-Lists a Specific Example Key

Testing the pre-commit hook with AWS's own documentation example key
(`AKIAIOSFODNN7EXAMPLE`) did **not** trigger a block — not because the
tool failed, but because `git-secrets --register-aws` deliberately adds
that exact string (and its matching secret-key example) to the
"allowed" list on registration. AWS's own official tutorials use that
key constantly, and without the exception, every engineer following an
AWS tutorial verbatim would trip a false positive. This is precisely
why a second test with a non-whitelisted fake key
(`AKIAZZZZZZZZZZZZZZZZ`) was necessary to actually prove enforcement
was working.

### Terraform's Sensitive-Value Guard

AWS's Terraform provider marks the `secret_string` attribute returned
by `aws_secretsmanager_secret_version` as sensitive by default.
Attempting to reference that value in a plain `output` block — without
explicitly adding `sensitive = true` — causes Terraform to refuse to
plan at all, with an explicit error rather than a silent redaction.
This is a structural safeguard against accidentally printing a secret
to a terminal, a CI log, or a state file diff.

## ✅ Implementation

### 1. `git-secrets` Installation and Setup
```bash
git clone https://github.com/awslabs/git-secrets.git
cd git-secrets
sudo make install
cd ~/terraform-security-pipeline
git secrets --install
git secrets --register-aws
```

### 2. Verification Test — False Positive (Expected to Pass)
```bash
echo 'aws_access_key = "AKIAIOSFODNN7EXAMPLE"' > test-secret.txt
git add test-secret.txt
git commit -m "test commit with fake secret"
```
Commit succeeded — confirmed via `git secrets --list` that this exact
string is a pre-registered `secrets.allowed` entry.

### 3. Verification Test — Real Pattern Match (Expected to Block)
```bash
echo 'aws_access_key = "AKIAZZZZZZZZZZZZZZZZ"' > test-secret.txt
git add test-secret.txt
git commit -m "test commit with fake secret v2"
```
**Result:** `[ERROR] Matched one or more prohibited patterns` — commit
blocked, confirming the hook is active and enforcing.

### 4. Creating a Secret in AWS Secrets Manager
```bash
aws secretsmanager create-secret \
  --name "day25-lab/demo-password" \
  --description "Demo secret for Day 25 lab" \
  --secret-string "MyDemoPassword2026!"
```

### 5. Referencing the Secret in Terraform Without Hardcoding It
```hcl
data "aws_secretsmanager_secret" "demo" {
  name = "day25-lab/demo-password"
}

data "aws_secretsmanager_secret_version" "demo_version" {
  secret_id = data.aws_secretsmanager_secret.demo.id
}

output "secret_name_confirmation" {
  value = data.aws_secretsmanager_secret.demo.name
}
```
`terraform plan` output showed only the secret's **name** in the
`Changes to Outputs` section — the actual password value was never
displayed, logged, or written into any `.tf` file.

### 6. Confirming Terraform's Sensitive-Value Protection
Added a second output that attempted to expose the actual secret
value:
```hcl
output "secret_value_test" {
  value = data.aws_secretsmanager_secret_version.demo_version.secret_string
}
```
`terraform plan` refused to proceed:
```
Error: Output refers to sensitive values
...
Terraform requires that any root module output containing sensitive
data be explicitly marked as sensitive, to confirm your intent.
```

### 7. Cleanup
```bash
rm secrets-demo.tf
aws secretsmanager delete-secret \
  --secret-id "day25-lab/demo-password" \
  --force-delete-without-recovery
```

## 🐛 Troubleshooting Log

### Issue: `sudo make install` failed with repeated "Sorry, try again" password prompts

**Root cause identified.**
The WSL2 user's password had been forgotten/mistyped repeatedly.

**Fix:** Reset the password from outside the WSL2 session entirely,
using Windows PowerShell to drop into the WSL2 instance as `root`
(which requires no password, since Windows itself already trusts that
level of access), then ran `passwd <username>` to set a new password
before returning to the normal user session.

### Issue: `terraform init`/`plan` failed with syntax errors referencing `cat > main.tf << 'EOF'` and a trailing `EOF`

**Root cause identified.**
A prior heredoc command (`cat > main.tf << 'EOF' ... EOF`) intended to
rewrite the entire file had, at some point, been captured literally as
the file's first and last lines rather than executing as a shell
command — leaving Terraform trying to parse `cat > main.tf << 'EOF'`
itself as HCL syntax, which produced cascading "Argument or block
definition required" and "Invalid character" errors.

**Fix:** Verified the exact problem lines with `head -3 main.tf` /
`tail -3 main.tf`, then surgically removed just the first and last
lines:
```bash
sed -i '1d' main.tf
sed -i '$d' main.tf
```
Re-verified with `head`/`tail` before re-running `terraform init`.

**Lesson:** When a heredoc-based file rewrite is pasted into a terminal
that's mid-way through processing other output, the boundary lines
(`cat > file << 'EOF'` and `EOF`) are exactly the kind of artifact that
can end up literally inside the file rather than being interpreted as
shell syntax — checking the file's first and last lines directly is a
faster diagnostic than reading the full HCL error stack.

### Issue: `terraform init` failed with a registry connection timeout

**Root cause identified.**
A single `terraform init` attempt failed with
`could not connect to registry.terraform.io... Client.Timeout exceeded`.
Rather than assuming a persistent network or firewall problem,
connectivity was verified independently and incrementally: basic
internet (`ping google.com` — succeeded) and then the specific
registry endpoint directly (`curl -I https://registry.terraform.io` —
returned `HTTP/2 200`). Since the direct connection test succeeded, the
original failure was concluded to be a transient timeout rather than a
structural blocker, and `terraform init` was simply re-run successfully.

**Lesson:** A single failed connection attempt during `init` doesn't
necessarily indicate a systemic network issue — isolating whether
general connectivity, DNS, and the specific endpoint are each
individually reachable prevents chasing a firewall/DNS fix for what
turned out to be a one-off timeout.

### Issue: `aws secretsmanager create-secret` failed with `AccessDeniedException`

**Root cause identified.**
The `terraform-lab-user` IAM identity (created in Day 23, scoped to
S3/IAM/SNS only) had no Secrets Manager permissions at all — an
expected result of following least-privilege scoping rather than
attaching broad access up front.

**Fix:** Attached the `SecretsManagerReadWrite` managed policy to the
user via the IAM console. Flagged as an area for further tightening —
a custom policy scoped to the `day25-lab/*` secret path prefix would be
the more precise, production-appropriate fix rather than blanket
read/write access to every secret in the account.

## 🧠 Key Concepts Applied

**Verify a Security Control With Two Tests, Not One** — The first
git-secrets test (the AWS example key) passing was not, by itself,
proof the tool worked; it could equally have meant the tool wasn't
running at all. Only the second test, with a pattern-matching but
non-whitelisted key, actually distinguished "the control is active and
correctly permissive" from "the control isn't running."

**Secrets-as-Data, Not Secrets-as-Config** — The Terraform pattern used
here (`data "aws_secretsmanager_secret_version"`) treats a secret as
something to be *looked up at plan/apply time*, never as a literal
value living in a file. This is the same category of fix as the
`terraform-lab-user` IAM role created in Day 23: moving the sensitive
material out of anything that gets committed, copied, or shared.

**A Tool Refusing to Run Can Be the Correct Outcome** — Both the
git-secrets commit block and the Terraform sensitive-output error were
"failures" in the sense that a command didn't complete — and both were
exactly the intended behavior. Distinguishing a tool doing its job from
a tool being broken is a recurring theme across this journey (echoing
the CI/CD pipeline's first failing run in Day 24).

**Least Privilege Creates Visible, Fixable Friction** — The
`AccessDeniedException` on secret creation was a direct, expected
consequence of scoping `terraform-lab-user` narrowly back in Day 23.
Rather than treating that friction as a sign the IAM setup was wrong,
it was resolved by adding exactly the new permission needed for this
new capability — the friction is the point.

## 📌 Key Takeaway

> A secret is genuinely safe only when there is no plaintext copy of it
> sitting in a place a human or a scanner could ever read — not in a
> chat window, not in a `.tf` file, not in a `terraform plan` output.
> git-secrets closes the accidental-commit path; Secrets Manager plus
> Terraform's `data` source pattern removes the need for the secret to
> exist in code at all; and Terraform's sensitive-value guard is the
> last backstop against printing it anyway. Layered defenses matter
> here specifically because the Day 23 incident showed that any single
> point of discipline — "just don't paste it anywhere" — is not a
> control, it's a hope.

## 🌍 Real-World Relevance

Every major secret-scanning product used in industry (GitHub secret
scanning, GitGuardian, TruffleHog) is functionally the same idea as
`git-secrets`, just with broader pattern libraries and, in some cases,
automatic revocation integrations with cloud providers. The
allow-listing behavior seen with the AWS example key mirrors exactly
how production secret scanners handle known-safe placeholder values
from documentation, to avoid alert fatigue from false positives — and
understanding that distinction (why a scanner *should* ignore some
matches) is as important as understanding why it blocks others.

## 🔗 References
- AWS Labs — *git-secrets GitHub repository*
- AWS Documentation — *AWS Secrets Manager User Guide*
- HashiCorp Documentation — *Terraform: Sensitive Input Variables and Outputs*
- AWS Documentation — *IAM policies for AWS Secrets Manager*

---
*Previous: [← Day 24 — CI/CD Pipeline Security](./day-24-cicd-pipeline-security.md)*
