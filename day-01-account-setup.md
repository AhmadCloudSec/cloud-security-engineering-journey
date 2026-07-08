 Day 1 — AWS Account Foundations & Root Account Hardening

🎯 Objective
Establish a secure baseline for a new AWS account by hardening the root 
identity, enabling proactive cost monitoring, and creating a properly 
scoped administrative identity for daily operations.

 🧩 Environment
 Platform: AWS Free Tier
 Region: us-east-1 (N. Virginia)
 Access Method: AWS Management Console

 📋 Summary

New AWS accounts are frequently compromised due to two common failures: 
unprotected root credentials and lack of cost visibility. This lab 
addresses both by implementing MFA on the root identity, provisioning 
a dedicated administrative IAM user for routine use, and configuring 
automated billing alerts.

 ✅ Tasks Completed

 1. Root Account Security
 Enabled Multi-Factor Authentication (MFA) on the root user using a 
  virtual authenticator app (avoided SMS-based MFA due to SIM-swap risk)
 Verified root account is not used for any operational task going forward

 2. Cost Visibility & Governance
 Enabled AWS Free Tier usage alerts and billing alerts in Billing Preferences
 Created two CloudWatch billing alarms via SNS notification:
  Threshold: **$1 USD** — early warning
  Threshold: **$5 USD** — escalation warning
 Confirmed SNS email subscription for alarm delivery

 3. IAM Administrative User
 Created a dedicated IAM user (`ahmad-admin`) for all daily operations
 Attached `AdministratorAccess` policy (temporary — to be scoped down 
  under least-privilege principles in later phases)
 Enabled MFA on the IAM user (separate device from root)
 Confirmed root account is no longer used for console access

 🧠 Key Concepts Applied

Shared Responsibility Model AWS secures the underlying infrastructure; 
the account owner is responsible for identity configuration, access control, 
and data protection within the account.

Root Account Risk — The root identity has unrestrictable, full account 
control. Industry best practice restricts its use to a small set of 
account-level tasks (e.g., closing the account, changing the root email), 
never routine administration.

🐞 Issue Encountered & Resolution

Issue: SNS email subscription for the billing alarm topic remained in 
`Pending confirmation` status despite multiple confirmation requests.

Resolution: Deleted the stale subscription and created a fresh SNS 
topic with a manually re-typed (non-pasted) email endpoint, which resolved 
the delivery issue. Confirmed via subscription status changing to `Confirmed`.

 ✔️ Validation

- IAM Security Status dashboard shows root MFA: enabled
- `aws sts get-caller-identity` confirms operations run under `ahmad-admin`, not root
- CloudWatch Alarms show `billing-alarm-1-dollar` and `billing-alarm-5-dollar` 
  in `OK` / monitoring state with actions enabled

 📌 Key Takeaway

> A properly hardened AWS account separates the *account owner identity* 
> (root, locked away) from the *operational identity* (IAM admin user, 
> MFA-protected, used daily) — reducing blast radius if any single 
> credential is compromised.
