# Day 20 — Detection Engineering: Sigma Rules & MITRE ATT&CK

`Difficulty: Intermediate` · `Focus Area: Detection Logic, Threat Modeling`

---

## 🎯 Objective

Understand Detection Engineering as the discipline of authoring custom, 
organization-specific detection logic — distinct from relying solely on 
managed detection services (GuardDuty, Day 14) — and author a Sigma 
rule mapped to MITRE ATT&CK for the privilege escalation technique 
demonstrated in Day 2.

## 📋 Problem Statement

Managed detection services like GuardDuty cover broad, generic threat 
patterns applicable across all AWS customers. Organization-specific 
risks — unusual access patterns unique to a company's own IAM 
structure, applications, or workflows — require custom detection logic 
authored by the organization itself. Detection Engineering is the 
practice of writing, testing, and maintaining this custom logic.

## 🧠 Core Concepts

**Sigma** — A vendor-agnostic, YAML-based format for writing detection 
rules once and converting them for use across multiple SIEM platforms 
(Splunk, Elastic, Microsoft Sentinel), avoiding the need to author 
duplicate logic per tool.

**MITRE ATT&CK Framework** — An industry-standard taxonomy of adversary 
tactics and techniques, providing common terminology for describing 
attacker behavior. The privilege escalation technique demonstrated in 
Day 2 corresponds to **T1548 (Abuse Elevation Control Mechanism)**.

**True Positive vs. False Positive:**

| | Definition |
|---|---|
| True Positive | The rule correctly identifies genuine malicious activity |
| False Positive | The rule triggers on legitimate, benign activity |
| False Negative | Malicious activity occurs but the rule fails to trigger |

Rule quality is measured not just by detection capability but by the 
ratio of true to false positives — an overly broad rule generates 
"alert fatigue," causing analysts to deprioritize or ignore alerts 
entirely, including genuine incidents.

## ✅ Implementation

### Authored a Sigma Rule for the Day 2 Privilege Escalation Technique

Based on the exact CloudTrail event structure captured in Day 2's 
`CreatePolicyVersion` attack, the following detection rule was written:

```yaml
title: AWS IAM Privilege Escalation via CreatePolicyVersion
id: 8f5c9a21-3b7e-4c1d-9e8f-1a2b3c4d5e6f
status: experimental
description: >
  Detects when a principal creates a new policy version with 
  overly permissive Action/Resource combinations, which may 
  indicate privilege escalation activity.
references:
  - https://attack.mitre.org/techniques/T1548/
author: Ahmad - Cloud Security Lab
date: 2026/07/25
tags:
  - attack.privilege_escalation
  - attack.t1548

logsource:
  product: aws
  service: cloudtrail

detection:
  selection:
    eventName: 'CreatePolicyVersion'
    eventSource: 'iam.amazonaws.com'
  suspicious_content:
    requestParameters|contains:
      - '"Action":"*"'
      - '"Resource":"*"'
  condition: selection and suspicious_content

falsepositives:
  - Legitimate administrative policy updates by authorized 
    security/platform teams (should be rare and well-documented)

level: high
```

### Design Rationale

- **Two-stage condition** (`selection` + `suspicious_content`): Matching 
  only on the event name (`CreatePolicyVersion`) alone would generate 
  excessive false positives, since this action is routinely performed 
  by legitimate administrators. Requiring the *combination* of the 
  event name **and** a wildcard `Action`/`Resource` payload narrows 
  detection to the specific pattern observed in the Day 2 attack.
- **Documented false positives**: A legitimate platform team performing 
  an intentional, broad policy update would also match this rule. 
  Documenting this explicitly is a deliberate part of rule authorship — 
  it informs the analyst reviewing an alert what to rule out first, 
  rather than leaving them to guess.
- **MITRE ATT&CK reference**: Tagging the rule with `T1548` allows this 
  detection to be cross-referenced against any other tooling or threat 
  intelligence organized by the same framework, rather than relying on 
  ad-hoc, non-standardized naming.

## 🧠 Key Concepts Applied

**Detection as a Complement to Response** — Day 2 demonstrated the 
attack and its manual forensic reconstruction after the fact. This rule 
represents the proactive counterpart: detecting the same technique in 
near-real-time were it to occur again, rather than relying solely on 
after-the-fact log review.

**Precision Over Recall (Initially)** — The rule was deliberately scoped 
narrowly (exact API call + specific payload pattern) rather than broadly 
(e.g., alerting on all `CreatePolicyVersion` calls), reflecting the 
practical reality that an unusable, noisy rule is often worse than no 
rule at all.

## 📌 Key Takeaway

> Detection Engineering extends beyond consuming managed detections 
> (GuardDuty) to authoring precise, well-documented, organization-aware 
> rules for specific attack techniques. The value of a detection rule 
> is inseparable from its false-positive profile — a rule's detection 
> logic and its documented exceptions must be designed together, not 
> as an afterthought.

## 🌍 Real-World Relevance

Mature security teams maintain a library of Sigma rules mapped to MITRE 
ATT&CK, version-controlled alongside application code, allowing 
detection logic to evolve through the same review and testing discipline 
as any other engineering artifact — rather than existing as 
undocumented, tribal knowledge inside a single SIEM console.

## 🔗 References
- Sigma Project — *Sigma Rule Specification*
- MITRE ATT&CK — *T1548: Abuse Elevation Control Mechanism*

---
*Previous: [← Day 19 — DNS Security & Route 53](./day-19-dns-security-route53.md)*
