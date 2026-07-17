# Day 16 — AWS IAM Identity Center: Federated Multi-Account Access

`Difficulty: Intermediate` · `Focus Area: Identity Federation, Multi-Account Governance`

---

## 🎯 Objective

Configure AWS IAM Identity Center to provide centralized, single sign-on 
access to an AWS account, demonstrating the federation pattern used by 
organizations to avoid managing individual IAM user credentials per 
account.

## 🧩 Environment

| | |
|---|---|
| **Platform** | AWS Free Tier |
| **Region** | us-east-1 |
| **Prerequisite** | AWS Organizations (Day 13) |

## 📋 Problem Statement

Managing individual IAM users across multiple AWS accounts does not 
scale — each new account requires separately provisioning and 
deprovisioning credentials per employee. IAM Identity Center solves 
this by providing one central identity that can be granted temporary, 
role-based access across any number of member accounts, without a 
static IAM user existing in each one.

## 🧠 Core Concepts

**Identity Federation** — Authenticating once with a central identity 
provider and receiving access to multiple downstream systems, rather 
than maintaining separate credentials per system.

**IAM Identity Center vs. IAM Users** — An Identity Center user is 
distinct from an IAM user (Day 1): it exists at the organization level 
and is granted temporary, assumed access to specific accounts via 
Permission Sets, rather than holding a static credential within any one 
account.

**Permission Sets** — The Identity Center equivalent of an IAM policy, 
but designed to be applied consistently across multiple accounts rather 
than scoped to a single account's IAM users.

## ✅ Implementation

### 1. Enabled IAM Identity Center
- Linked to the existing AWS Organization (Day 13)

### 2. Created a Federated User and Group
- Created user `ahmad-sso-test`
- Created group `Developers-SSO` and added the user as a member

### 3. Created a Permission Set
- Used the predefined `ReadOnlyAccess` permission set, defining the 
  maximum access level grantable through this assignment

### 4. Assigned Account Access
- Assigned both the individual user and the group to the management 
  account with the `ReadOnlyAccess` permission set
- Confirmed via the "Assigned users and groups" view showing both 
  assignments active

### 5. Cleanup
- Removed account access assignments
- Deleted the permission set, group, and user

## 📌 Key Takeaway

> IAM Identity Center inverts the typical access model: instead of 
> creating a static IAM user in every account a person needs access to, 
> a single federated identity is granted temporary, revocable access to 
> any number of accounts via Permission Sets. This is the standard 
> pattern for human access in multi-account AWS organizations — static 
> IAM users (Day 1) are reserved for service accounts and automation, 
> not routine human login.

## 🌍 Real-World Relevance

Enterprises with dozens of AWS accounts rely on this pattern to ensure 
offboarding an employee is a single action (removing their Identity 
Center access) rather than requiring credential cleanup across every 
individual account they had access to.

## 🔗 References
- AWS Documentation — *AWS IAM Identity Center User Guide*
- AWS Documentation — *Permission Sets*

---
*Previous: [← Day 15 — Least Privilege, RBAC & ABAC](./day-15-abac-least-privilege.md)*
