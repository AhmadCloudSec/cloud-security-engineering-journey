# Day 8 — AWS Secrets Manager: Credential-less Application Secrets

`Difficulty: Foundational` · `Focus Area: Secrets Management, Credential Hygiene`

---

## 🎯 Objective

Demonstrate secure storage and retrieval of application credentials 
using AWS Secrets Manager, eliminating the need to hardcode sensitive 
values (passwords, API keys) directly in application code or 
configuration files.

## 🧩 Environment

| | |
|---|---|
| **Platform** | AWS Free Tier |
| **Region** | us-east-1 |
| **Tools** | AWS Management Console, AWS CLI |

## 📋 Problem Statement

Hardcoded credentials in source code are a persistent, well-documented 
root cause of security incidents — particularly when code is 
accidentally committed to public repositories. AWS Secrets Manager 
addresses this by centralizing secret storage, encrypting secrets at 
rest, and allowing applications to retrieve credentials at runtime via 
IAM-authenticated API calls rather than embedded plaintext values.

## ✅ Implementation

### 1. Secret Creation
- Created a secret (`demo-app-db-credentials`) of type "Other type of 
  secret" containing a `username`/`password` key-value pair
- Secret encrypted at rest using the default AWS-managed KMS key 
  (`aws/secretsmanager`)

### 2. Retrieval via AWS CLI

```bash
$ aws secretsmanager get-secret-value --secret-id demo-app-db-credentials --profile ahmad-admin
{
    "ARN": "arn:aws:secretsmanager:us-east-1:<ACCOUNT_ID>:secret:demo-app-db-credentials-xxxxx",
    "Name": "demo-app-db-credentials",
    "VersionId": "<version-id>",
    "SecretString": "{\"username\":\"admin\",\"password\":\"REDACTED\"}",
    "VersionStages": ["AWSCURRENT"]
}
```

This mirrors how an application (e.g., an EC2 instance with an attached 
IAM Role, as demonstrated in Day 6) would retrieve credentials at 
runtime — via an authenticated API call, not a value stored in code.

### 3. Deletion (Safety Mechanism Observed)

```bash
$ aws secretsmanager delete-secret --secret-id demo-app-db-credentials --recovery-window-in-days 7 --profile ahmad-admin
{
    "ARN": "arn:aws:secretsmanager:us-east-1:<ACCOUNT_ID>:secret:demo-app-db-credentials-xxxxx",
    "Name": "demo-app-db-credentials",
    "DeletionDate": "2026-07-20T12:33:52+05:00"
}
```

**Observation:** Secrets Manager enforces a minimum recovery window 
(here, 7 days) before permanent deletion — a deliberate safeguard 
against accidental or malicious secret deletion, giving administrators 
a window to cancel the deletion if needed.

## 🧠 Key Concepts Applied

**Credential-less Application Design** — Combined with IAM Roles (Day 
6), an application never needs to store a static secret value; it 
authenticates as itself (via its role) and retrieves the actual secret 
value only at runtime.

**Encryption at Rest** — Secret values are encrypted using AWS KMS by 
default, meaning even direct access to the underlying storage would not 
expose plaintext secret values without the corresponding decryption 
permissions.

**Deletion Safety Windows** — Unlike many AWS resources that delete 
immediately, Secrets Manager enforces a mandatory recovery period, 
reflecting the high blast-radius risk of losing a production credential.

## 📌 Key Takeaway

> Secrets Manager, combined with IAM Roles, enables a credential-less 
> application architecture: no password, API key, or connection string 
> ever needs to exist in source code, configuration files, or version 
> control — only an authenticated runtime API call.

## 🌍 Real-World Relevance

This pattern directly prevents one of the most common causes of 
credential leaks: secrets committed to source control (public or 
private). Combined with automatic secret rotation (not exercised in 
this lab but natively supported), Secrets Manager also reduces the 
blast radius of a leaked credential by limiting its useful lifetime.

## 🔗 References
- AWS Documentation — *AWS Secrets Manager*
- AWS Documentation — *Rotating Secrets*

---
*Previous: [← Day 7 — VPC Routing Deep Dive](./day-07-vpc-routing-deep-dive.md)*
