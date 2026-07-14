# Day 10 — RDS Database Security Fundamentals

`Difficulty: Foundational` · `Focus Area: Database Security, Network Isolation`

---

## 🎯 Objective

Provision a managed MySQL database using Amazon RDS and verify its 
network isolation configuration, confirming the database has no direct 
path to the public internet.

## 🧩 Environment

| | |
|---|---|
| **Platform** | AWS Free Tier |
| **Region** | us-east-1 |
| **Engine** | MySQL Community |
| **Instance Class** | db.t4g.micro |

## 📋 Problem Statement

Databases typically hold an organization's highest-value data, making 
them a primary target in any breach scenario. A database that is 
directly reachable from the public internet represents a significant 
and unnecessary risk. This lab provisions an RDS instance and verifies, 
via the console, that no internet-facing path to the database exists.

## 🧠 Core Concepts

**Managed Database Service** — RDS handles underlying OS patching, 
engine installation, and infrastructure maintenance, reducing the 
operational security surface to network configuration, access control, 
and credential management.

**Public Accessibility Setting** — At creation time, RDS instances can 
be set to `Publicly accessible: No`, which prevents the database from 
being assigned a route to an Internet Gateway, regardless of Security 
Group configuration. This mirrors the Route Table mechanism reviewed in 
Day 7 — the database sits in a network position with no path to the 
public internet.

**Security Groups (Database-Level)** — Even within the VPC, access to 
the database instance is further restricted by a Security Group, which 
by default only permits inbound connections from resources within the 
same VPC (e.g., an application-tier EC2 instance) — not from arbitrary 
external sources.

## ✅ Implementation

### 1. Provisioned a MySQL RDS Instance
- Created via RDS "Standard create" using the Free Tier template
- Engine: MySQL Community, instance class `db.t4g.micro`
- Master credentials configured with auto-generated password

### 2. Configured Network Isolation at Creation
- **Public access:** `No`
- **VPC:** Default VPC
- **VPC Security Group:** Newly created, scoped to VPC-internal traffic

### 3. Verified Network Isolation Post-Deployment

Reviewed the **Connectivity & security** tab and confirmed:

| Setting | Value | Interpretation |
|---|---|---|
| Internet access gateway | `Disabled` | The database has no route to an Internet Gateway — it cannot be reached from the public internet under any circumstance, independent of Security Group rules |
| Security group rules | EC2 Security Group (inbound), CIDR/IP (outbound) | Inbound access is scoped to resources within the VPC, not `0.0.0.0/0` |

**Key verification:** `Internet access gateway: Disabled` is a stronger 
guarantee than a restrictive Security Group alone — even if a Security 
Group rule were misconfigured to be overly permissive, the database 
would still be unreachable from the internet because no network path to 
an Internet Gateway exists at all.

### 4. Cleanup
- Deleted the RDS instance without a final snapshot (non-production lab)

## 🧠 Key Concepts Applied

**Defense in Depth** — Two independent layers were verified: the 
absence of an internet-facing route (network layer) and a scoped 
Security Group (access-control layer). Either control alone reduces 
risk; together, they provide redundant protection against 
misconfiguration in one layer.

**Least Exposure by Default** — Setting `Publicly accessible: No` at 
creation time ensures the database is isolated by default, rather than 
requiring a separate, error-prone step to "lock it down" after the fact.

## 📌 Key Takeaway

> A database's safety from internet-based attacks should not rely 
> solely on Security Group rules. Confirming there is no route to an 
> Internet Gateway at all (`Internet access gateway: Disabled`) provides 
> a stronger, network-layer guarantee — consistent with the Route Table 
> mechanics reviewed in Day 7.

## 🌍 Real-World Relevance

Publicly accessible databases with permissive Security Groups are a 
recurring root cause in disclosed data breaches. Defaulting new 
databases to `Publicly accessible: No` and placing them in a private 
network context eliminates this entire class of exposure by design.

## 🔗 References
- AWS Documentation — *Amazon RDS User Guide*
- AWS Documentation — *Working with a DB Instance in a VPC*

---
*Previous: [← Day 9 — KMS Encryption](./day-09-kms-encryption.md)*
