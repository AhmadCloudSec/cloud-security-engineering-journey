# Day 24 — CI/CD Pipeline Security (GitHub Actions + Automated Checkov Gate)

`Difficulty: Hands-On` · `Focus Area: CI/CD Security, Automated Policy Enforcement, Git Authentication`

---

## 🎯 Objective

Move the manual Checkov scanning practiced in Day 22 into an automated
GitHub Actions pipeline, so that every `push` or pull request against
`main` is scanned without requiring a human to remember to run the
command. Experience a real automated-gate failure and resolve it the
way a pull request reviewer would in production — by fixing the
underlying code, not by disabling the check.

## 🧩 Environment

| | |
|---|---|
| **Platform** | GitHub Actions (hosted runners) |
| **Host OS** | Windows 10 + WSL2 (Ubuntu 24.04) for local git/Terraform work |
| **Repo** | `terraform-security-pipeline` (new, dedicated repo) |
| **Tools** | Git, GitHub CLI-less HTTPS auth (Personal Access Token), `bridgecrewio/checkov-action@v12` |

## 📋 Problem Statement

A manually-run scanner only catches what a human remembers to run
before pushing. A team of any size beyond one person cannot rely on
every contributor remembering `checkov -d .` before every commit. The
fix is to move the scan into the platform itself — a pipeline that
triggers automatically on every push and blocks merges when policy is
violated, regardless of whether the contributor remembered to check
locally.

## 🧠 Core Concepts

### GitHub Actions Execution Model

```
git push
    │
    ▼
GitHub detects push event on `main`
    │
    ▼
Spins up a temporary Ubuntu VM (hosted runner)
    │
    ▼
actions/checkout@v4  →  clones the repo onto that VM
    │
    ▼
bridgecrewio/checkov-action@v12  →  installs & runs Checkov
    │
    ├── soft_fail: false + FAILED check → job fails, red ❌
    │
    └── all checks PASSED/SKIPPED → job succeeds, green ✅
```

The runner is entirely GitHub-managed infrastructure — it exists only
for the duration of the job and is destroyed afterward. Nothing about
the pipeline depends on the contributor's own machine being on or
configured correctly.

### `soft_fail`: The Difference Between a Gate and a Suggestion

`soft_fail: false` was a deliberate configuration choice. With
`soft_fail: true`, Checkov would report findings but the job would
still report success — turning the pipeline into a passive dashboard
rather than an enforced gate. `soft_fail: false` means a single
unresolved `FAILED` check blocks the pipeline from reporting success,
which is what makes this a genuine control rather than a suggestion.

### Personal Access Tokens and Scoped Permissions

GitHub no longer accepts account passwords for git operations over
HTTPS — a Personal Access Token (PAT) is required instead, and critically,
**that token's scopes limit what it can do**. A token scoped only to
`repo` was sufficient to push ordinary file changes, but GitHub
specifically rejected the push once it detected changes under
`.github/workflows/`, requiring the additional `workflow` scope. This
is a deliberate extra guardrail: workflow files can execute arbitrary
code on every future push, so GitHub treats creating or modifying them
as a higher-privilege action than an ordinary file change.

## ✅ Implementation

### 1. Repository and Local Clone
Created `terraform-security-pipeline` on GitHub with a README and a
Terraform `.gitignore`, then cloned it into WSL2. The Day 22 `main.tf`
was copied into the new repo as the subject of the pipeline.

### 2. Workflow File
```yaml
name: Terraform Security Scan

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  checkov-scan:
    name: Checkov IaC Security Scan
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Run Checkov
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: .
          framework: terraform
          output_format: cli
          soft_fail: false
```

### 3. First Push — Pipeline Failed as Designed
The copied `main.tf` did not include the final `#checkov:skip`
justification comments and the replica-bucket lifecycle fix that had
been applied at the end of Day 22 — meaning the exact same 5 findings
resolved previously were still present in this file. The pipeline
correctly caught them and reported **Failing** on the commit.

### 4. Fix and Re-Push
Rewrote `main.tf` to include:
- A lifecycle configuration for the replica bucket
  (`abort_incomplete_multipart_upload`)
- `#checkov:skip` justification comments on the log bucket
  (`CKV2_AWS_62`, `CKV_AWS_144`) and replica bucket
  (`CKV2_AWS_62`, `CKV_AWS_18`)

Verified locally with `checkov -d .` before pushing again — confirming
a clean scan is something to check *before* relying on the pipeline to
catch it, not instead of.

### 5. Second Push — Pipeline Passed
Commit `23e4f14` showed a green checkmark in the repository's commit
history, confirming the automated gate now passes on the corrected
configuration.

## 🐛 Troubleshooting Log

### Issue: `git clone` prompted for a username that was actually a full URL

**Root cause identified.**
When Git's credential prompt asked `Username for 'https://github.com':`,
the full repository URL was entered instead of just the account
username, producing a malformed credential string and an authentication
failure. The correct value for that prompt is the bare GitHub username
(confirmed by checking the GitHub profile directly, which also revealed
the actual username was `AhmadCyberSec`, not the previously assumed
`AhmadCloudSec`).

**Lesson:** Verifying the exact identity string directly from the
source (the GitHub profile page) rather than working from memory
avoided a second layer of confusion stacked on top of the credential
issue.

### Issue: `git push` rejected with "Password authentication is not supported"

**Root cause identified.**
GitHub deprecated password authentication for git operations over
HTTPS in 2021. A Personal Access Token must be used in place of the
account password.

**Fix:** Generated a token via **Settings → Developer settings →
Personal access tokens (classic)**, scoped initially to `repo`.

### Issue: Push rejected specifically for the workflow file — `refusing to allow a Personal Access Token to create or update workflow... without 'workflow' scope`

**Root cause identified.**
The `repo` scope alone is insufficient for creating or modifying files
under `.github/workflows/`. GitHub requires the additional `workflow`
scope specifically because workflow files are executable automation,
not passive configuration.

**Fix:** Updated the token to include the `workflow` scope alongside
`repo`, then re-ran `git push origin main` successfully.

### Issue: Terminal output was repeatedly pasted back in as commands

**Root cause identified.**
Several confirmation messages (`git status` output, prior success
messages) were pasted into the terminal as if they were new commands,
producing a cascade of `command not found` errors (`mmit`, `create`,
etc.) that had nothing to do with the actual git or Checkov state.

**Fix:** Re-ran `git status` cleanly with no preceding paste to
re-establish ground truth before continuing — a reminder that when
error messages look unrelated to the intended command, the first
question to ask is whether the terminal actually received only the
intended command.

## 🧠 Key Concepts Applied

**A Pipeline Gate Is Only as Strict as Its Failure Mode** —
`soft_fail: false` was the single line of configuration that made this
a real control. The same pipeline with `soft_fail: true` would have
run identically, reported the same findings, and still shown green —
looking functional while enforcing nothing.

**Scope Tokens to the Minimum Required Action, Then Expand Only When
Blocked** — The token was created with `repo` first, not
`repo + workflow` preemptively. GitHub's specific rejection reason
(workflow file modification) was the signal to add exactly the
permission that was missing, rather than requesting broad scopes
up front "to avoid issues later."

**Verify Locally Before Trusting the Remote Gate** — After fixing the
Checkov findings, `checkov -d .` was run locally before pushing again,
rather than pushing directly and waiting to see if the pipeline caught
anything else. The pipeline is the enforced backstop; it should not be
the primary feedback loop during active development.

**Automated Failures Are Information, Not Setbacks** — The first
pipeline failure was the exact intended behavior of the system being
built: the gate caught real, un-remediated findings the moment the
code reached a shared branch. Treating a red ❌ as "the pipeline is
broken" versus "the pipeline is working" is a mindset distinction that
matters — this pipeline was never broken; it was doing its job.

## 📌 Key Takeaway

> A CI/CD security gate only has teeth if a failing check actually
> blocks something and if getting past it requires fixing the
> underlying issue rather than adjusting the tool. This lab's pipeline
> failed once, correctly, on real findings — and passed only after
> those findings were genuinely resolved with either a fix or a
> justified suppression, never by loosening the pipeline itself.

## Real-World Relevance

This is the exact mechanism behind every "required status check" seen
on pull requests at companies using policy-as-code: a red X blocking a
merge button until Checkov, tfsec, Snyk, or an equivalent scanner
reports clean. The token-scope friction encountered here is also a
real, common first-time stumbling block for engineers setting up their
first GitHub Actions workflow that touches its own `.github/workflows/`
directory — recognizing the specific rejection reason and resolving it
by adding exactly the needed scope (rather than over-granting
permissions) is itself a small but real least-privilege decision.

## 🔗 References
- GitHub Documentation — *Personal access tokens (classic)*
- GitHub Documentation — *Managing GitHub Actions settings for a repository*
- Checkov / Bridgecrew — *checkov-action GitHub Marketplace listing*
- GitHub Documentation — *Deprecation of password authentication for Git operations*

---
*Previous: [← Day 23 — Terraform Deployment Lifecycle](./day-23-terraform-deployment-lifecycle.md)*
