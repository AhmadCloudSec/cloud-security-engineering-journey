# Day 7 — VPC Routing Deep Dive: Internet Gateways & Route Tables

`Difficulty: Intermediate` · `Focus Area: Network Architecture, Traffic Flow`

---

## 🎯 Objective

Move beyond Security Group-level access control to understand the 
network-layer mechanics that actually determine whether a subnet has 
internet connectivity — specifically, the interaction between Internet 
Gateways and Route Tables — and empirically verify the true definition 
of "public" vs. "private" subnets.

## 🧩 Environment

| | |
|---|---|
| **Platform** | AWS Free Tier |
| **Region** | us-east-1 |
| **VPC** | Default VPC (`172.31.0.0/16`) |

## 📋 Problem Statement

"Public subnet" and "private subnet" are frequently treated as fixed 
attributes or toggles in introductory material, which obscures how AWS 
networking actually works. In reality, subnet reachability from the 
internet is determined entirely by **Route Table configuration** — 
specifically, whether a route exists directing `0.0.0.0/0` traffic to 
an Internet Gateway. This lab verifies that mechanism directly against 
a live account rather than accepting it as a stated definition.

## 🧠 Core Concepts

**Internet Gateway (IGW)** — A horizontally scaled, VPC-attached 
component that enables communication between resources in a VPC and the 
public internet. An IGW has no effect on traffic unless a route 
explicitly directs traffic to it.

**Route Table** — A set of rules ("routes") that determine where 
network traffic from a subnet is directed. Every VPC subnet is 
associated with exactly one route table, which contains at minimum a 
`local` route for intra-VPC traffic.

**The Actual Public/Private Definition:**

| Subnet Type | Defining Characteristic |
|---|---|
| **Public** | Associated route table contains a route: `0.0.0.0/0 → igw-xxxx` |
| **Private** | Associated route table contains **no** route to an Internet Gateway (only the `local` route, and optionally a route to a NAT Gateway for outbound-only access) |

There is no separate "public/private" flag on a subnet — the 
classification is a direct consequence of its route table's contents.

**NAT Gateway (Conceptual)** — Enables resources in a private subnet to 
initiate outbound connections to the internet (e.g., for software 
updates) while preventing inbound connections from being initiated by 
external hosts. A NAT Gateway is always deployed in a *public* subnet, 
since it requires its own route to an Internet Gateway.

**Bastion Host (Conceptual)** — A hardened, single point-of-entry 
instance deployed in a public subnet, used to "jump" into private 
subnet resources (e.g., databases) that have no direct route to the 
internet, avoiding direct exposure of those resources.

## ✅ Implementation & Verification

### 1. Confirmed Internet Gateway Attachment

Reviewed the account's Internet Gateway and confirmed:

| Attribute | Value |
|---|---|
| Internet Gateway ID | `igw-0ab84f850de2059ea` |
| State | `Attached` |
| VPC ID | `vpc-0fee13f31d04f9adc` |

### 2. Reviewed Route Table Contents

Inspected the main route table associated with the Default VPC:

| Destination | Target | Interpretation |
|---|---|---|
| `172.31.0.0/16` | `local` | Intra-VPC traffic routed directly, no gateway involved |
| `0.0.0.0/0` | `igw-0ab84f850de2059ea` | All other traffic routed through the Internet Gateway |

**Conclusion:** Any subnet associated with this route table has a valid 
path to the internet and is therefore, by definition, a **public 
subnet** — independent of any naming convention or console label.

### 3. Verified Subnet-to-Route-Table Association

Reviewed all six subnets in the Default VPC (one per Availability Zone) 
and confirmed each is associated with the route table containing the 
IGW route — consistent with AWS's default behavior of provisioning all 
Default VPC subnets as public.

### 4. Conceptual Validation — Private Route Table Behavior

Reasoned through the inverse case: a route table containing only the 
`172.31.0.0/16 → local` route (no IGW route) would result in any 
associated subnet having **no path to the internet**, correctly 
classifying it as private — regardless of whether its addressing scheme 
or naming suggested otherwise.

## 📌 Key Takeaway

> "Public" and "private" subnet classification is not a fixed property 
> or console setting — it is an emergent result of Route Table 
> configuration. A subnet becomes public the moment its route table 
> gains a `0.0.0.0/0 → Internet Gateway` route, and reverts to private 
> the moment that route is removed. This has direct security 
> implications: auditing subnet exposure requires inspecting route 
> tables, not subnet names or tags.

## 🌍 Real-World Relevance

Misunderstanding this mechanism is a common source of accidental 
exposure — for example, associating a subnet intended to be private 
with a route table that includes an IGW route, or provisioning 
resources into a subnet based on its *name* (e.g., "private-subnet-1") 
without verifying its actual route table contents.

## 🔗 References
- AWS Documentation — *Route Tables*
- AWS Documentation — *Internet Gateways*
- AWS Documentation — *NAT Gateways*
- AWS Documentation — *Bastion Hosts on AWS*

---
*Previous: [← Day 6 — EC2 & IAM Role Integration](./day-06-ec2-iam-role-integration.md)*
