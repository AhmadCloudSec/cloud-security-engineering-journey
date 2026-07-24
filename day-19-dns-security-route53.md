# Day 19 — DNS Security & Amazon Route 53

`Difficulty: Conceptual` · `Focus Area: DNS Architecture, Traffic Routing, DNS Attack Vectors`

---

## 🎯 Objective

Understand DNS as a foundational internet protocol, analyze common DNS-based 
attack vectors (spoofing, cache poisoning, amplification DDoS, tunneling), 
and review Amazon Route 53's security and traffic-routing capabilities — 
including DNSSEC, DNS Firewall, and weighted load balancing — as 
mitigations against these threats.

## 🧩 Environment

| | |
|---|---|
| **Platform** | AWS Free Tier |
| **Service** | Amazon Route 53 |
| **Scope** | Conceptual review (no domain registration required) |

## 📋 Problem Statement

DNS was designed in the 1980s without security as a primary 
consideration — responses are, by default, unauthenticated and 
unencrypted. This makes DNS a high-value target: compromising DNS 
resolution allows an attacker to silently redirect users to malicious 
infrastructure without needing to compromise the destination service 
itself. Understanding DNS attack vectors and their corresponding 
mitigations is foundational to securing any internet-facing 
architecture, including the ALB-based application reviewed in Day 17.

## 🧠 Core Concepts

### DNS Resolution Process

DNS translates human-readable domain names into IP addresses through a 
hierarchical lookup:
### DNS Attack Vectors

| Attack | Mechanism | Impact |
|---|---|---|
| **DNS Spoofing / Cache Poisoning** | Attacker injects a forged DNS response into a resolver's cache | Users are silently redirected to attacker-controlled infrastructure (e.g., a phishing clone of a legitimate site) |
| **DNS Hijacking** | Attacker compromises domain registrar credentials and modifies authoritative DNS records directly | Complete redirection of all traffic for the domain, globally |
| **DNS Amplification (DDoS)** | Attacker spoofs the victim's IP as the source of a small DNS query; the (much larger) response is sent to the victim | Volumetric DDoS using minimal attacker bandwidth |
| **DNS Tunneling** | Data is encoded into DNS query subdomains to exfiltrate data through a channel often excluded from deep packet inspection | Covert data exfiltration or C2 channel |

### DNS Security Mitigations

**DNSSEC (DNS Security Extensions)** — Cryptographically signs DNS 
records, allowing resolvers to verify a response has not been tampered 
with in transit. Conceptually parallel to the KMS-based integrity 
guarantees explored in Day 9: a signature proves authenticity, not just 
correctness.

**DNS Firewall (Route 53 Resolver DNS Firewall)** — Blocks resolution of 
known-malicious domains at the resolver level, preventing infected 
hosts from reaching command-and-control infrastructure even if malware 
is already present.

**Private Hosted Zones** — DNS zones resolvable only from within a 
specified VPC, preventing internal resource names from being resolvable 
(and therefore discoverable) from the public internet.

## ✅ Implementation

### 1. Reviewed Route 53 Hosted Zone Architecture
- Examined the Hosted Zones interface and DNS record structure
- Reviewed DNS Firewall's domain list mechanism for blocking malicious 
  domain resolution

### 2. Analyzed Weighted Routing Policy

Route 53 supports multiple routing policies beyond simple A-record 
resolution. **Weighted routing** was reviewed as a mechanism directly 
applicable to the multi-instance architecture built in Day 17:
**Use case:** Weighted routing enables canary deployments and gradual 
traffic shifting — for example, directing a small percentage of traffic 
to a new application version before a full cutover, limiting the blast 
radius of a faulty deployment. This complements, rather than replaces, 
the Application Load Balancer's own round-robin distribution across 
healthy targets (Day 17) — Route 53 weighting typically operates one 
layer above the ALB, controlling traffic between entirely separate 
endpoints (e.g., different regions or environments), while the ALB 
handles distribution across targets within a single endpoint.

### 3. Compared Routing Policies

| Policy | Use Case |
|---|---|
| Simple | Single-endpoint domains, no load distribution |
| Weighted | Gradual traffic shifting, canary releases, A/B testing |
| Latency-based | Route users to the lowest-latency regional endpoint |
| Failover | Automatic redirection to a standby endpoint on health check failure |
| Geolocation | Route based on the requester's geographic location |

## 🧠 Key Concepts Applied

**Defense Beyond the Application Layer** — Day 17's WAF lab protected 
the application layer (HTTP requests); this lab extends that 
perspective to the layer that precedes it entirely — if DNS resolution 
itself is compromised, traffic never reaches the WAF-protected ALB in 
the first place. A complete security review must consider each layer 
independently.

**Weighted Routing as Risk Mitigation** — Rather than an all-or-nothing 
deployment (Day 17's architecture sent 100% of traffic to both healthy 
targets equally via the ALB), DNS-level weighted routing provides a 
mechanism to limit exposure when introducing change at the 
infrastructure or application-version level.

## 📌 Key Takeaway

> DNS sits outside the security perimeter of any individual application 
> — a WAF (Day 17), IAM policies (Day 2, 15), and encryption (Day 9) all 
> assume traffic correctly reaches the intended endpoint in the first 
> place. DNSSEC and DNS Firewall address the layer beneath all of these 
> controls: ensuring the name resolution itself cannot be silently 
> subverted to redirect users before any other control has a chance to 
> apply.

## 🌍 Real-World Relevance

Financial institutions and other high-value targets implement DNSSEC 
specifically because DNS spoofing bypasses application-layer defenses 
entirely — a user visiting a spoofed banking domain never reaches the 
real, WAF-protected, encrypted application, making DNS integrity a 
prerequisite for every other security control's effectiveness.

## 🔗 References
- AWS Documentation — *Amazon Route 53 Developer Guide*
- AWS Documentation — *Route 53 Resolver DNS Firewall*
- AWS Documentation — *Choosing a Routing Policy*
- IETF RFC 4033 — *DNS Security Introduction and Requirements (DNSSEC)*

---
*Previous: [← Day 18 — Zero Trust Architecture](./day-18-zero-trust-architecture.md)*
