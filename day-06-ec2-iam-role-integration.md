# Day 6 — EC2 Instance Provisioning & IAM Role Integration

`Difficulty: Intermediate` · `Focus Area: Compute Security, Credential-less Access`

---

## 🎯 Objective

Provision a live EC2 instance, connect via SSH, and demonstrate — with 
real command output — how IAM Roles provide temporary, credential-less 
AWS access to a compute resource, in contrast to hardcoded access keys.

## 🧩 Environment

| | |
|---|---|
| **Platform** | AWS Free Tier |
| **Region** | us-east-1 |
| **Instance Type** | t3.micro (Free Tier eligible) |
| **AMI** | Amazon Linux 2023 |
| **Access Method** | EC2 Instance Connect (browser-based SSH) |

## 📋 Summary

Hardcoding long-lived AWS access keys onto compute resources is a common 
source of credential leakage in real incidents. This lab provisions a 
live EC2 instance and empirically demonstrates the before/after behavior 
of attaching an IAM Role, confirming that AWS-issued temporary 
credentials — not stored secrets — govern the instance's permissions.

## ✅ Implementation

### 1. Instance Provisioning
- Launched a `t3.micro` instance (Amazon Linux 2023)
- Generated a new RSA key pair for SSH access
- Scoped the associated Security Group's SSH rule to a single trusted 
  source IP (not `0.0.0.0/0`)
- Verified instance reached `Running` state with `3/3` status checks passed

### 2. Baseline Verification — No IAM Role Attached

Connected to the instance via EC2 Instance Connect and executed:

```bash
$ aws sts get-caller-identity
Unable to locate credentials. You can configure credentials by running "aws login".
```

✅ Confirms the instance has no AWS identity or permissions by default — 
no keys are pre-loaded or hardcoded on the AMI.

### 3. IAM Role Attachment

- Created an IAM role (`EC2-S3-Role`) with:
  - Trusted entity: `ec2.amazonaws.com`
  - Attached policy: `AmazonS3ReadOnlyAccess`
- Attached the role to the running instance via **Actions → Security → 
  Modify IAM role** (no instance restart required)

### 4. Post-Attachment Verification

```bash
$ aws sts get-caller-identity
```
Returned a valid identity under `assumed-role/EC2-S3-Role/<instance-id>` 
— confirming the instance is now operating under temporary, 
role-assumed credentials rather than any stored secret.

```bash
$ aws s3 ls
2026-07-10  ahmadcloudsec-security-lab-demo
2026-07-05  aws-cloudtrail-logs-<ACCOUNT_ID>-xxxxxxxx
```
✅ S3 bucket listing succeeded — confirming the role's `ReadOnly` 
permissions are active and being used by the instance in real time.

### 5. Least-Privilege Verification

To confirm the role does not exceed its intended scope, attempted a 
write operation:

```bash
$ aws s3 rm s3://ahmadcloudsec-security-lab-demo/test.txt
delete failed: An error occurred (AccessDenied) when calling the 
DeleteObject operation: User: 
arn:aws:sts::<ACCOUNT_ID>:assumed-role/EC2-S3-Role/<instance-id> 
is not authorized to perform: s3:DeleteObject
```

✅ Confirms the role grants **read-only** access exactly as configured — 
no broader permissions were implicitly or accidentally granted.

### 6. Cleanup
- Terminated the EC2 instance
- Deleted the associated Security Group

## 🧠 Key Concepts Applied

**Credential-less Compute Access** — IAM Roles allow AWS services (like 
EC2) to request short-lived, automatically-rotated credentials at 
runtime, eliminating the need to store long-lived access keys on the 
instance itself.

**Empirical Least-Privilege Verification** — Rather than assuming a 
policy's scope, the role's actual effective permissions were verified 
directly via both an allowed action (`s3 ls`) and a denied action 
(`s3 rm`), producing concrete evidence of the permission boundary.

## 📌 Key Takeaway

> The identity a compute resource operates under should always be 
> assigned via an IAM Role rather than embedded credentials. This was 
> directly verified: identical AWS CLI commands produced categorically 
> different results (credential failure → successful read → denied 
> write) purely based on the role attached to the instance, with zero 
> credentials ever stored on the instance itself.

## 🔗 References
- AWS Documentation — *IAM Roles for Amazon EC2*
- AWS Documentation — *Temporary Security Credentials*

---
*Previous: [← Day 5 — VPC Networking & Security Groups](./day-05-vpc-networking-security.md)*
