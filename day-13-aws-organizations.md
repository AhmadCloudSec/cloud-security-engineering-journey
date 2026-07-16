# Day 13 — AWS Organizations: Multi-Account Governance

`Difficulty: Foundational` · `Focus Area: Multi-Account Management, Policy Governance`

---

## 🎯 Objective

Understand AWS Organizations as a governance layer for managing multiple 
AWS accounts under a single management structure, and examine Service 
Control Policies (SCPs) as an account-wide permission boundary distinct 
from IAM policies.

## 🧩 Environment

| | |
|---|---|
| **Platform** | AWS Free Tier |
| **Scope** | Single account (Management Account) |

## 📋 Problem Statement

As organizations scale, managing security, billing, and access across 
multiple isolated AWS accounts (e.g., separate accounts for 
development, production, and testing) individually becomes impractical. 
AWS Organizations addresses this by centralizing account management, 
consolidated billing, and policy enforcement under a single management 
account.

## 🧠 Core Concepts

**Management Account** — The account used to create the organization; 
responsible for consolidated billing across all member accounts and the 
root of the organizational hierarchy.

**Organizational Units (OUs)** — Logical groupings of accounts (e.g., 
"Production", "Development") that allow policies to be applied to 
multiple accounts simultaneously rather than individually.

**Service Control Policies (SCPs)** — Account-wide (or OU-wide) 
permission guardrails that define the *maximum* permissions available 
within an account — even to that account's own Administrator. This is a 
critical distinction from IAM policies:

| | IAM Policy | Service Control Policy |
|---|---|---|
| Scope | Individual user/role/group | Entire account or OU |
| Function | Grants specific permissions | Sets the ceiling on what can ever be granted, regardless of IAM policy |
| Can an Account Admin
