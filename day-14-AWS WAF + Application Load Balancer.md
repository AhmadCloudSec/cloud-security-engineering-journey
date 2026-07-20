# AWS WAF + Application Load Balancer Security Lab

A hands-on cloud security project demonstrating Layer 7 (application-layer) protection for a load-balanced web application on AWS. This lab simulates a production-style architecture — public/private subnet segmentation, health-checked EC2 targets behind an Application Load Balancer, and an AWS WAF Web ACL enforcing managed rule sets against common web exploits (SQL Injection, XSS, and known bad inputs).

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Objective](#objective)
- [Tech Stack](#tech-stack)
- [Architecture Diagram](#architecture-diagram)
- [Implementation Steps](#implementation-steps)
- [Security Group Configuration](#security-group-configuration)
- [Attack Simulation & Testing](#attack-simulation--testing)
- [Troubleshooting Log](#troubleshooting-log)
- [Lessons Learned](#lessons-learned)
- [Screenshots](#screenshots)
- [Future Improvements](#future-improvements)

---

## Architecture Overview

This project provisions a small but realistic web infrastructure on AWS and places a Web Application Firewall in front of it to inspect and filter incoming HTTP traffic before it ever reaches the application servers.

**Core components:**

| Component | Purpose |
|---|---|
| VPC (custom) | Isolated network environment |
| 2x Public Subnets (Multi-AZ) | Host EC2 instances across two Availability Zones for high availability |
| Internet Gateway | Provides internet connectivity to the public subnets |
| Route Table | Routes `0.0.0.0/0` traffic to the Internet Gateway |
| 2x EC2 Instances (Amazon Linux 2023) | Run Apache (`httpd`) web servers — `waf-web-server-1` and `waf-web-server-2` |
| Application Load Balancer (ALB) | Distributes incoming traffic across both EC2 targets |
| Target Group | Performs health checks on the EC2 instances (HTTP, port 80) |
| AWS WAF (Web ACL) | Inspects HTTP requests and blocks malicious patterns before they reach the ALB targets |
| Security Groups | Enforce least-privilege inbound/outbound network access |

---

## Objective

The goal of this lab was to:

1. Build a fault-tolerant, load-balanced web architecture from scratch
2. Understand and correctly configure Security Groups (inbound vs. outbound rules)
3. Deploy AWS WAF with managed rule groups to protect against OWASP-style attacks
4. Validate the WAF is actually blocking malicious traffic through real attack simulation (not just configuration — actual testing)
5. Document a real, messy troubleshooting process (not a "happy path" tutorial)

---

## Tech Stack

- **AWS EC2** — Amazon Linux 2023, t3.micro
- **AWS VPC** — Custom VPC with public subnets across 2 Availability Zones
- **AWS Application Load Balancer (ALB)**
- **AWS Target Groups** — Health check configuration
- **AWS WAF (Web ACL)** — Regional resource type, Managed Rule Groups
- **Apache HTTP Server (httpd)**
- **AWS Security Groups & Network ACLs**

---

## Architecture Diagram

```
                              Internet
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │       AWS WAF            │
                    │  (Web ACL - Regional)     │
                    │                           │
                    │  • Core Rule Set (CRS)    │
                    │  • SQL Database Rules     │
                    │  • Known Bad Inputs       │
                    └────────────┬──────────────┘
                                 │ (filtered traffic only)
                                 ▼
                    ┌─────────────────────────┐
                    │  Application Load        │
                    │  Balancer (waf-lab-alb)  │
                    │  Internet-facing          │
                    └────────────┬──────────────┘
                                 │
                         ┌───────┴────────┐
                         ▼                ▼
              ┌──────────────────┐ ┌──────────────────┐
              │  Target Group     │ │  Target Group     │
              │  Health Check: /  │ │  Health Check: /  │
              │  Port 80 (HTTP)   │ │  Port 80 (HTTP)   │
              └─────────┬─────────┘ └─────────┬─────────┘
                        ▼                     ▼
           ┌────────────────────┐  ┌────────────────────┐
           │  EC2 Instance 1     │  │  EC2 Instance 2     │
           │  waf-web-server-1   │  │  waf-web-server-2   │
           │  AZ: us-east-1a     │  │  AZ: us-east-1b     │
           │  Apache (httpd)     │  │  Apache (httpd)     │
           │  Public Subnet 1    │  │  Public Subnet 2    │
           └────────────────────┘  └────────────────────┘
                        │                     │
                        └──────────┬──────────┘
                                   ▼
                        ┌────────────────────┐
                        │  Internet Gateway   │
                        └────────────────────┘
```

---

## Implementation Steps

### 1. Network Foundation
- Created a custom VPC with CIDR `10.0.0.0/16`
- Created two public subnets (`10.0.1.0/24` and `10.0.2.0/24`) in separate Availability Zones for high availability
- Attached an Internet Gateway and configured the route table with a `0.0.0.0/0 → igw-xxxx` route

### 2. EC2 Instance Provisioning
- Launched two Amazon Linux 2023 (`t3.micro`) instances, one per public subnet
- Connected via **EC2 Instance Connect** (browser-based SSH)
- Installed and configured Apache:
  ```bash
  sudo dnf install -y httpd
  sudo systemctl start httpd
  sudo systemctl enable httpd
  echo '<h1>WAF Lab - Server is working</h1>' | sudo tee /var/www/html/index.html
  ```

### 3. Load Balancer & Target Group
- Created a Target Group (`waf-lab-tg`) — HTTP, port 80, health check path `/`
- Registered both EC2 instances as targets
- Created an Internet-facing Application Load Balancer (`waf-lab-alb`) and attached the target group as its default listener rule (port 80)

### 4. AWS WAF Deployment
- Created a Web ACL (`waf-lab-webacl`) scoped to **Regional resources**
- Associated the Web ACL with the ALB
- Added AWS Managed Rule Groups:
  - **Core Rule Set (CRS)** — general OWASP Top 10 protections (XSS, common exploits)
  - **SQL Database** — SQL injection–specific protections
  - **Known Bad Inputs** — blocks known malicious request patterns
- Set default action for each rule group to **Block** (not Count)

### 5. Validation
- Confirmed both targets reached a **Healthy** status in the Target Group
- Sent normal traffic to the ALB DNS name and confirmed successful page load
- Simulated SQL Injection and XSS attacks against the ALB endpoint and confirmed AWS WAF returned **403 Forbidden**

---

## Security Group Configuration

| Direction | Type | Port | Source/Destination | Purpose |
|---|---|---|---|---|
| Inbound | SSH | 22 | My IP | EC2 Instance Connect access |
| Inbound | HTTP | 80 | ALB Security Group / 0.0.0.0/0 | Allow ALB health checks + traffic to reach instances |
| Outbound | HTTP | 80 | 0.0.0.0/0 | Package repository access |
| Outbound | HTTPS | 443 | 0.0.0.0/0 | Required for `dnf`/`yum` package downloads (S3-backed repos) |
| Outbound | SSH | 22 | My IP | Return traffic for SSH session |

---

## Attack Simulation & Testing

To validate that AWS WAF was functioning — not just configured — the following requests were sent directly to the ALB's public DNS name:

| Test | Request | Expected Result | Actual Result |
|---|---|---|---|
| Baseline | `GET /` | 200 OK | ✅ 200 OK |
| SQL Injection | `GET /?id=1' OR '1'='1` | 403 Forbidden | ✅ 403 Forbidden |
| Cross-Site Scripting (XSS) | `GET /?search=<script>alert(1)</script>` | 403 Forbidden | ✅ 403 Forbidden |

This confirms the WAF is actively inspecting query string parameters and blocking requests matching known attack signatures **before** they reach the EC2 targets — the load balancer logs show these requests never reached the application layer.

---

## Troubleshooting Log

Real-world cloud engineering is rarely a straight path. Below are the actual issues encountered during this build and how each was diagnosed and resolved — documented deliberately, since debugging methodology is a core skill this lab was meant to develop.

### Issue 1: SSH connection to EC2 failed ("Error establishing SSH connection")
- **Diagnosis:** Checked instance state (Running, status checks passed) → checked Security Group inbound rules
- **Root cause:** No inbound rule for port 22 (SSH) existed on the instance's security group
- **Fix:** Added an inbound rule — Type: SSH, Port: 22, Source: My IP

### Issue 2: `sudo dnf install -y httpd` timed out
- **Diagnosis:** Ran `curl localhost` (failed — no server), checked security group outbound rules
- **Root cause:** No outbound rule for port 443 (HTTPS). Amazon Linux package repositories are served over HTTPS from S3, so without an outbound 443 rule, `dnf` could not reach the mirror servers, resulting in a connection timeout
- **Fix:** Added an outbound rule — Type: HTTPS, Port: 443, Destination: 0.0.0.0/0

### Issue 3: Target Group health checks failing ("Request timed out") after Apache was confirmed running
- **Diagnosis:** Confirmed `curl localhost` worked on both instances (Apache serving correctly) → confirmed Route Table had a valid IGW route → confirmed Network ACLs were set to default Allow-All → isolated the issue to the instance-level Security Group
- **Root cause:** No **inbound** rule for port 80 (HTTP) existed. Outbound HTTP/HTTPS had been added (for package installs) but the inbound direction — required for the ALB to actually reach the instance — was still missing
- **Fix:** Added an inbound rule — Type: HTTP, Port: 80, Source: 0.0.0.0/0 (ALB security group in a production setting)
- **Key takeaway:** Outbound and inbound rules serve entirely different purposes and must both be configured for full bidirectional functionality — this was the most valuable lesson of the lab

### Issue 4: XSS payload was not blocked on first WAF test
- **Diagnosis:** Reviewed the Web ACL's associated rules
- **Root cause:** The initial "Essentials" protection pack was created without AWS Managed Rule Groups attached — only the base pack shell existed
- **Fix:** Manually added the AWS-Managed Rule Group option and enabled Core Rule Set, SQL Database, and Known Bad Inputs, with the default action set to Block

---

## Lessons Learned

- **Security Groups are stateful but directional** — an outbound allow rule does not imply the corresponding inbound traffic is allowed, and vice versa. Each direction must be deliberately configured for its specific purpose (e.g., inbound 80 for ALB→instance traffic, outbound 443 for instance→internet package downloads).
- **"Request timed out" health check errors** almost always point to a network-layer block (Security Group, NACL, or routing) rather than an application-layer problem — the systematic elimination order that proved effective was: **Compute (is the app running?) → Security Group Outbound → Security Group Inbound → Route Table → NACL**.
- **A WAF Web ACL being "associated" with a resource does not guarantee protection is active** — the managed rule groups must be explicitly added and their default action explicitly set to Block; the pack UI can create an empty shell if managed rules aren't selected during setup.
- **Testing beats assuming.** Configuring a control (like a WAF rule) is not the same as validating it. Simulated attack traffic was essential to prove the control worked as intended, rather than trusting the console configuration alone.

---

## Screenshots

> Add screenshots to a `/screenshots` folder in this repo and reference them below.

- `screenshots/target-group-healthy.png` — Target Group showing both instances Healthy
- `screenshots/waf-webacl-overview.png` — WAF Web ACL associated with the ALB
- `screenshots/xss-blocked-403.png` — 403 Forbidden response for simulated XSS payload
- `screenshots/sql-injection-blocked-403.png` — 403 Forbidden response for simulated SQL Injection payload
- `screenshots/normal-request-200.png` — Baseline successful request through the ALB

---

## Future Improvements

- [ ] Move EC2 instances into private subnets with a NAT Gateway for outbound-only internet access (removing direct public IPs from application servers)
- [ ] Enable WAF logging to a CloudWatch Log Group / S3 bucket for long-term analysis
- [ ] Add a rate-based WAF rule to mitigate brute-force and basic DDoS-style traffic
- [ ] Enable HTTPS on the ALB listener using an ACM certificate
- [ ] Add CloudWatch Alarms on WAF blocked-request metrics for real-time alerting
- [ ] Automate this entire architecture with Terraform for repeatable deployment
