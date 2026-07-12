# Day 5 — VPC Networking & Security Group Hardening

`Difficulty: Foundational` · `Focus Area: Network Segmentation, Access Control`

---

## 🎯 Objective

Understand AWS network isolation fundamentals (VPC, Subnets) and 
demonstrate, in a controlled lab, the risk of an overly permissive 
Security Group rule commonly seen in real-world misconfigurations.

## 🧩 Environment

| | |
|---|---|
| **Platform** | AWS Free Tier |
| **Region** | us-east-1 |
| **VPC** | Default VPC |

## 📋 Summary

Network-layer misconfigurations — particularly overly permissive Security 
Group rules — are a common root cause of unauthorized access attempts 
against cloud-hosted servers. This lab reviews AWS network isolation 
architecture and demonstrates the configuration of both a safe and a 
deliberately risky inbound rule.

## 🧠 Core Concepts

**VPC (Virtual Private Cloud)** — An isolated, private network 
environment within AWS, fully controlled by the account owner.

**Subnets** — Segments within a VPC:
- **Public subnet:** Resources with direct internet-facing connectivity 
  (e.g., web servers)
- **Private subnet:** Resources isolated from direct internet access 
  (e.g., databases), typically reachable only from within the VPC

**Security Groups** — Stateful, instance-level firewalls controlling 
inbound/outbound traffic to specific resources. Unlike Network ACLs 
(subnet-level, stateless), Security Groups only support "Allow" rules 
and automatically permit return traffic for any allowed connection.

## ✅ Implementation

### 1. Reviewed Default VPC Architecture
- Identified the account's Default VPC and its associated subnets
- Reviewed the default Security Group's baseline (no open inbound rules)

### 2. Created a Security Group Demonstrating Risk

Created `demo-risky-sg` with an inbound rule:

| Type | Protocol | Port | Source |
|---|---|---|---|
| SSH | TCP | 22 | `0.0.0.0/0` (Anywhere) |

**Risk analysis:** A `0.0.0.0/0` source on a management port (SSH/22) 
permits connection attempts from any IPv4 address globally, exposing the 
instance to continuous automated brute-force login attempts — a pattern 
frequently observed in real-world compromise cases.

### 3. Correct Configuration (Applied in Day 6 EC2 Lab)

When launching a live EC2 instance for hands-on testing, the SSH inbound 
rule was scoped to **"My IP"** only, restricting SSH access to a single, 
known source address rather than the entire internet.

## 📌 Key Takeaway

> Management ports (SSH/22, RDP/3389) should never be exposed to 
> `0.0.0.0/0`. Access should be scoped to a specific IP, a corporate VPN 
> range, or — preferably — eliminated in favor of session-based access 
> tools (e.g., AWS Systems Manager Session Manager) that require no 
> open inbound port at all.

## 🧹 Cleanup Performed
- Deleted `demo-risky-sg`

## 🔗 References
- AWS Documentation — *Security Groups for Your VPC*
- AWS Documentation — *Network ACLs*

---
*Previous: [← Day 4 — S3 Bucket Security](./day-04-s3-bucket-security.md)*
