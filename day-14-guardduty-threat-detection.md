# Day 14 — GuardDuty: Automated Threat Detection

`Difficulty: Foundational` · `Focus Area: Threat Detection, Security Monitoring`

---

## 🎯 Objective

Enable Amazon GuardDuty and review its automated threat detection 
findings, understanding how it analyzes CloudTrail, VPC Flow Logs, and 
DNS logs using machine learning to surface anomalous activity without 
manual log review — directly contrasting with the manual CloudTrail 
analysis performed in Day 2 and Day 7.

## 🧩 Environment

| | |
|---|---|
| **Platform** | AWS Free Tier |
| **Region** | eu-north-1 |

## 📋 Problem Statement

Manually reviewing CloudTrail logs (as demonstrated in Day 2) does not 
scale — production environments generate far more log volume than can 
be practically reviewed by a human analyst. GuardDuty addresses this by 
continuously analyzing account activity against machine learning 
models and known threat intelligence, surfacing only findings that 
warrant investigation.

## 🧠 Core Concepts

**Continuous ML-Based Analysis** — GuardDuty ingests CloudTrail 
management events, VPC Flow Logs, and DNS query logs, comparing 
observed behavior against learned baselines and known malicious 
indicators (e.g., known malicious IP lists), rather than requiring 
manually authored detection rules for every scenario.

**Findings** — The output of GuardDuty's analysis: a structured record 
describing a specific detected anomaly, including type, severity, 
affected resource, and supporting evidence (e.g., which API calls 
triggered the detection).

**Severity Levels** — Findings are categorized (Low/Medium/High) to help 
prioritize investigation and response effort, rather than treating every 
finding as equally urgent.

## ✅ Implementation

### 1. Enabled GuardDuty
- Activated via the GuardDuty console with default settings

### 2. Generated Sample Findings

Used the built-in **"Generate sample findings"** feature to populate 
the Findings dashboard with representative examples of real-world 
detection categories, without requiring an actual malicious event to 
occur in the account.

### 3. Analyzed a Sample Finding

Reviewed a `CredentialAccess:IAMUser/AnomalousBehavior` finding in 
detail:

| Field | Value |
|---|---|
| Type | `CredentialAccess:IAMUser/AnomalousBehavior` |
| Severity | Medium |
| Description | An IAM user invoked APIs commonly associated with credential access attack patterns |
| Anomalous APIs | 4 API calls flagged as inconsistent with the user's established behavioral baseline |

**Analyst interpretation:** This finding type mirrors the underlying 
logic of the Day 2 privilege escalation lab — an identity performing 
actions inconsistent with its normal usage pattern (e.g., a 
read-only-scoped user suddenly invoking IAM policy-modification APIs) is 
exactly the kind of signal GuardDuty is designed to surface 
automatically, rather than requiring a human analyst to notice it while 
manually reviewing CloudTrail Event History.

### 4. Cleanup
- Disabled GuardDuty via **Settings → Disable GuardDuty** to avoid 
  ongoing per-event analysis charges beyond the free trial period

## 🧠 Key Concepts Applied

**Automated vs. Manual Detection** — Day 2 and Day 7 demonstrated manual 
CloudTrail log analysis to reconstruct a specific known event. GuardDuty 
demonstrates the complementary automated approach: continuously scanning 
all activity for anomalies without requiring a human to know what to 
look for in advance.

**Behavioral Baselining** — Rather than relying solely on static rules 
(e.g., "block this specific IP"), GuardDuty's anomaly detection 
approach can surface novel attack patterns that deviate from an 
identity's established behavior, which static rule-based detection 
would miss entirely.

## 📌 Key Takeaway

> Manual log analysis (Day 2, Day 7) and automated detection (GuardDuty) 
> are complementary, not competing, approaches. GuardDuty surfaces 
> candidate incidents at scale using behavioral analysis; manual 
> CloudTrail review remains necessary for deep forensic reconstruction 
> once a finding — automated or otherwise — warrants investigation.

## 🌍 Real-World Relevance

Security teams typically route GuardDuty findings into a centralized 
alerting pipeline (e.g., via EventBridge to Slack, PagerDuty, or a 
SIEM), enabling 24/7 automated monitoring coverage that would be 
impractical to achieve through manual log review alone.

## 🔗 References
- AWS Documentation — *Amazon GuardDuty User Guide*
- AWS Documentation — *GuardDuty Finding Types*

---
*Previous: [← Day 13 — AWS Organizations](./day-13-aws-organizations.md)*
