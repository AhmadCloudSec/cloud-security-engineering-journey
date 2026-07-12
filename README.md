 ☁️ Cloud Security Engineering — Hands-On Journey

[AWS](https://img.shields.io/badge/AWS-Cloud-orange)
[Security](https://img.shields.io/badge/Focus-Cloud%20Security-blue)
[Status](https://img.shields.io/badge/Status-In%20Progress-yellow)

📖 About This Repository

This repository documents my structured, hands-on training in Cloud Security 
Engineering, following an enterprise-style bootcamp model (80% practical, 
20% theory). Each entry represents a real lab performed in a live AWS 
environment — including exact commands, console steps, validation checks, 
and (where applicable) simulated attack scenarios with full remediation.

The goal is to build production-relevant skills for roles such as:
- Cloud Security Engineer
- DevSecOps Engineer
- Cloud Security Analyst / SOC Analyst

 🎯 Approach

Rather than following tutorials passively, every lab in this repo was:
1. Performed hands-on in a real AWS Free Tier account
2. Documented with actual command outputs (sensitive data redacted)
3. Analyzed from both an **attacker's** and **defender's** perspective
4. Followed by remediation and cleanup steps

📂 Training Log

| Day | Topic | Key Skills Demonstrated | Status |
|-----|-------|--------------------------|--------|
| [Day 1](./day-01-account-setup.md) | AWS Account Hardening | Root account MFA, billing alarm automation, IAM least-privilege foundations | ✅ Complete |
| [Day 2](./day-02-privesc-lab.md) | CloudTrail & Privilege Escalation | Audit logging, IAM attack simulation, CloudTrail forensic analysis, remediation | ✅ Complete |
| [Day 3](./day-03-iam-groups-and-roles.md) | IAM Groups, Roles & Log Forensics | Group-based access control, service-assumable roles, manual CloudTrail analysis | ✅ Complete |
| [Day 4](./day-04-s3-bucket-security.md) | S3 Bucket Security | Public access misconfiguration, bucket policies, defense-in-depth | ✅ Complete |
| [Day 5](./day-05-vpc-networking-security.md) | VPC Networking & Security Groups | Network segmentation, security group hardening, least-exposure principle | ✅ Complete |
| [Day 6](./day-06-ec2-iam-role-integration.md) | EC2 & IAM Role Integration | Credential-less compute access, live IAM role verification, least-privilege testing | ✅ Complete |
| [Day 7](./day-07-vpc-routing-deep-dive.md) | VPC Routing Deep Dive | Internet Gateway/Route Table mechanics, public vs private subnet verification | ✅ Complete |

(Repository updated as training progresses)

🛠️ Tools & Technologies

`AWS IAM` `AWS CloudTrail` `AWS CloudWatch` `AWS GuardDuty` `AWS CLI` `PowerShell` `JSON Policy Authoring`

 🔐 Security & Redaction Notice

All AWS Access Keys, Secret Keys, and Account IDs referenced in this 
repository have been redacted, rotated, or replaced with placeholder 
values prior to publishing. No live credentials are present in this repo.

 📫 Contact

Ahmad — Aspiring Cloud Security Engineer  
LinkedIn: *[add your LinkedIn URL here]*
