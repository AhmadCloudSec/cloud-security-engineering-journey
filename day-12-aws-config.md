# Day 12 — AWS Config: Resource Configuration Tracking

`Difficulty: Foundational` · `Focus Area: Configuration Auditing, Compliance`

---

## 🎯 Objective

Understand AWS Config's role in tracking resource configuration history 
over time, distinct from CloudTrail's action-based audit logging, and 
its use in continuous compliance evaluation via Config Rules.

## 🧩 Environment

| | |
|---|---|
| **Platform** | AWS Free Tier |
| **Region** | us-east-1 |

## 📋 Problem Statement

Knowing *who performed an action* (CloudTrail) is different from knowing 
*what a resource's configuration was at a given point in time*. AWS 
Config addresses the latter, maintaining a configuration history for 
supported resource types and enabling automated compliance checks 
against defined rules (e.g., "no S3 bucket should be publicly 
readable").

## 🧠 Core Concepts

**CloudTrail vs. AWS Config:**

| | CloudTrail | AWS Config |
|---|---|---|
| Tracks | API actions (who did what, when) | Resource configuration state over time |
| Answers | "Who changed this?" | "What was this resource's configuration before/after a change?" |

**Config Rules** — Automated checks that continuously evaluate whether 
resources comply with a defined policy (e.g., encryption enabled, no 
public access). Non-compliant resources are automatically flagged 
without requiring manual auditing.

**Configuration Recorder** — The underlying mechanism that must be 
explicitly enabled to begin tracking resource configurations; this 
incurs a per-configuration-item cost, distinct from CloudTrail's 
inclusion in the free tier for basic event history.

## ✅ Implementation

### 1. Enabled AWS Config
- Configured the recorder to track all supported resource types in the 
  region
- Provisioned an S3 bucket for configuration history storage
- Confirmed recorder status: `Recording is on`

### 2. Reviewed the Compliance Dashboard
- Observed Dashboard metrics: 0 compliant/non-compliant rules, 0 
  compliant/non-compliant resources
- **Analysis:** This zero-state was expected — no Config Rules were 
  deployed during setup, so no compliance evaluation was being performed. 
  Config Rules must be explicitly defined (e.g., using AWS-managed rules 
  like `s3-bucket-public-read-prohibited`) before compliance data 
  populates.

### 3. Cleanup
- Disabled the configuration recorder (`Stop recording`) to halt 
  ongoing per-item charges

## 📌 Key Takeaway

> AWS Config's value is unlocked specifically through Config Rules — 
> without them, the service only passively tracks configuration history 
> without evaluating it against any policy. A meaningful AWS Config 
> deployment requires defining rules aligned to organizational security 
> baselines (e.g., mandatory encryption, no public S3 access, required 
> tagging) rather than enabling the recorder alone.

## 🌍 Real-World Relevance

Organizations use AWS Config Rules to continuously enforce security 
baselines at scale — for example, automatically flagging (or, combined 
with EventBridge and Lambda, automatically remediating) any S3 bucket 
that becomes publicly accessible, without waiting for a periodic manual 
audit.

## 🔗 References
- AWS Documentation — *AWS Config Developer Guide*
- AWS Documentation — *AWS Config Managed Rules*

---
*Previous: [← Day 11 — EBS Volumes & Snapshots](./day-11-ebs-snapshots.md)*
