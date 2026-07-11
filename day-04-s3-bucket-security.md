# Day 4 — S3 Bucket Security & Public Access Misconfiguration

`Difficulty: Foundational` · `Focus Area: Data Security, Misconfiguration Risk`

---

## 🎯 Objective

Demonstrate, in a controlled lab, how a common S3 misconfiguration — 
disabling Block Public Access combined with an overly permissive bucket 
policy — exposes stored objects to unauthenticated, public internet 
access, and how to correctly remediate it.

## 🧩 Environment

| | |
|---|---|
| **Platform** | AWS Free Tier |
| **Region** | us-east-1 |
| **Service** | Amazon S3 |

## 📋 Problem Statement

Misconfigured S3 buckets are among the most frequently cited root causes 
in publicly disclosed cloud data breaches. Unlike many cloud 
misconfigurations, exposing an S3 bucket requires no credential theft or 
exploit — only a permissive setting change, making it a critical control 
to understand and audit.

## ✅ Implementation

### 1. Baseline Secure Bucket
- Created a new S3 bucket with default settings retained:
  - **Block Public Access:** fully enabled (all four settings)
  - **Default encryption:** SSE-S3 (Amazon S3 managed keys)
- Uploaded a test object and confirmed the object is downloadable by the 
  bucket owner via the console

### 2. Baseline Access Control Verification
- Retrieved the object's public URL and accessed it in an unauthenticated 
  (incognito) browser session
- **Result:** `AccessDenied` — confirms the bucket is not publicly 
  accessible by default, as expected

### 3. Simulated Misconfiguration (Attacker Perspective)

To reproduce a real-world exposure scenario, the following changes were 
made deliberately, in isolation, and reverted immediately after 
verification:

**Step A** — Disabled all four Block Public Access settings at the 
bucket level.

**Step B** — Attached the following bucket policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::<BUCKET_NAME>/*"
    }
  ]
}
```

**Result:** The same object URL, when accessed from an unauthenticated 
incognito session, returned the file content directly — no 
authentication, login, or credentials required.

**Root cause analysis:** The `Principal: "*"` statement grants the 
specified action (`s3:GetObject`) to *any* AWS principal, authenticated 
or not, effectively making every object in the bucket world-readable via 
direct URL.

### 4. Remediation
- Removed the bucket policy entirely
- Re-enabled all four Block Public Access settings
- Re-verified via the object URL that access reverted to `AccessDenied`

## 🧠 Key Concepts Applied

**Block Public Access (Account/Bucket Level)** — A safeguard layer that 
overrides even a permissive bucket policy; AWS enables this by default 
on new buckets specifically to prevent accidental exposure.

**Principal Wildcarding Risk** — A bucket policy `Principal` value of 
`"*"` should only ever be used deliberately, for content explicitly 
intended to be public (e.g., static website assets), never as a default 
or convenience setting.

**Defense in Depth** — This lab required *both* disabling Block Public 
Access *and* adding a permissive policy to expose the bucket — 
illustrating why Block Public Access exists as an independent, 
overriding safeguard rather than relying on policy authors alone.

## 📌 Key Takeaway

> Public S3 exposure requires no exploit — only a permissive 
> configuration change. Every new bucket should retain Block Public 
> Access enabled by default, with public access enabled only 
> deliberately, per-bucket, and only for content explicitly intended to 
> be public.

## 🌍 Real-World Relevance

This misconfiguration pattern mirrors publicly documented breaches 
involving exposed cloud storage buckets, where sensitive customer or 
internal data was left accessible via direct URL due to overly 
permissive bucket policies combined with disabled public access 
protections.

## 🔗 References
- AWS Documentation — *Blocking Public Access to Your Amazon S3 Storage*
- AWS Documentation — *Amazon S3 Bucket Policies*

---
*Previous: [← Day 3 — IAM Groups & Roles](./day-03-iam-groups-and-roles.md)*
