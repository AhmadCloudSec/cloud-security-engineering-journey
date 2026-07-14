# Day 9 — AWS KMS: Encryption Key Management

`Difficulty: Intermediate` · `Focus Area: Cryptography, Key Management`

---

## 🎯 Objective

Understand AWS Key Management Service (KMS) fundamentals — symmetric key 
encryption, AWS-managed vs. customer-managed keys, and envelope 
encryption — and empirically verify the encrypt/decrypt lifecycle using 
a self-created Customer Managed Key (CMK).

## 🧩 Environment

| | |
|---|---|
| **Platform** | AWS Free Tier |
| **Region** | us-east-1 |
| **Tools** | AWS Management Console, AWS CLI, PowerShell |

## 📋 Problem Statement

Encryption is only as trustworthy as the key management behind it. AWS 
KMS centralizes key generation, storage, and usage auditing without 
ever exposing raw key material to the caller — every cryptographic 
operation happens as an API call, not a local computation on an 
extractable key. This lab creates a Customer Managed Key and verifies 
this behavior directly.

## 🧠 Core Concepts

**Symmetric Encryption** — The same key is used to both encrypt and 
decrypt data. KMS's default key type (`SYMMETRIC_DEFAULT`) uses AES-256.

**AWS Managed Keys vs. Customer Managed Keys (CMK):**

| | AWS Managed Key | Customer Managed Key |
|---|---|---|
| Created by | AWS automatically (e.g., `aws/s3`) | Account owner, explicitly |
| Policy control | Not editable | Fully editable key policy |
| Cost | Free | ~$1/month + usage |
| Typical use | Default service encryption | Compliance, audit, cross-account sharing |

**Envelope Encryption** — Since KMS API calls are limited to ~4KB of 
data, large objects are encrypted using a locally-generated **data key**, 
which is itself encrypted by KMS and stored alongside the encrypted 
data. This is the mechanism underlying SSE-KMS in S3, EBS, and RDS.

**Key Deletion Safety** — KMS enforces a mandatory waiting period 
(7–30 days) before permanently deleting a key, since any data encrypted 
under a deleted key becomes permanently unrecoverable.

## ✅ Implementation

### 1. Created a Customer Managed Key
- Key type: Symmetric, Encrypt/Decrypt usage
- Alias: `demo-security-lab-key`
- Key administrators and usage permissions scoped to `ahmad-admin`

### 2. Verified Key Metadata

```bash
$ aws kms describe-key --key-id alias/demo-security-lab-key --profile ahmad-admin
{
    "KeyId": "92c0a9a2-5eff-461b-941d-f128724b8a35",
    "KeyState": "Enabled",
    "KeyUsage": "ENCRYPT_DECRYPT",
    "KeyManager": "CUSTOMER",
    "CustomerMasterKeySpec": "SYMMETRIC_DEFAULT"
}
```

### 3. Encrypted a Plaintext Value

```bash
$ aws kms encrypt --key-id alias/demo-security-lab-key --plaintext "Hello Security Lab" --profile ahmad-admin --output text --query CiphertextBlob
AQICAHiYanxbfBOrXH1yb/HSF0jPSPlecdfigdfx6/nhvenj9gGLEP0E72qAba+Zj...
```

The resulting ciphertext is opaque and unrecoverable without a 
corresponding `kms:Decrypt` call against the same key — the raw key 
material was never exposed at any point in this process.

### 4. Decrypted the Ciphertext

```bash
$ aws kms decrypt --ciphertext-blob fileb://encrypted.bin --profile ahmad-admin --query Plaintext --output text
HelloSecurityLab
```

✅ Confirms the full encrypt → decrypt lifecycle: the AWS CLI's `--output 
text` automatically base64-decodes the `Plaintext` field, so no 
additional manual decoding was required to recover the original value.

### 5. Cleanup
- Deleted the local ciphertext file
- Scheduled key deletion with the minimum 7-day recovery window

## 🧠 Key Concepts Applied

**Key Material Isolation** — At no point in this lab was the actual 
cryptographic key material visible or exportable; all operations were 
performed as authenticated API requests to KMS, which is the core 
security guarantee of the service (backed by FIPS 140-2 validated 
hardware security modules).

**Least-Privilege Key Policies** — Key administrator and key usage 
permissions were scoped explicitly to a single IAM identity, rather than 
being left at an overly permissive default.

## 📌 Key Takeaway

> KMS's security model depends on the key never leaving AWS's control — 
> callers request operations (`Encrypt`, `Decrypt`, `GenerateDataKey`), 
> not the key itself. This is why services like S3 (Day 4), Secrets 
> Manager (Day 8), and EBS/RDS all layer on top of KMS rather than 
> implementing their own encryption schemes.

## 🌍 Real-World Relevance

Regulated industries (finance, healthcare) typically mandate Customer 
Managed Keys over AWS Managed Keys specifically for the audit trail and 
key policy control CMKs provide — every `Encrypt`/`Decrypt` call against 
a CMK is individually logged in CloudTrail, enabling granular usage 
auditing.

## 🔗 References
- AWS Documentation — *AWS Key Management Service*
- AWS Documentation — *Envelope Encryption*

---
*Previous: [← Day 8 — Secrets Manager](./day-08-secrets-manager.md)*
