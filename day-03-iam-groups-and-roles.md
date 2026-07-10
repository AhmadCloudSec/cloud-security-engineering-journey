# Day 3 — IAM Groups, IAM Roles & CloudTrail Forensic Analysis

`Difficulty: Intermediate` · `Focus Area: Identity Architecture, Log Forensics`



 Objective

Extend IAM identity architecture beyond individual users by implementing 
group-based permission inheritance and service-assumable roles, and 
perform manual forensic analysis of a prior security event using 
CloudTrail Event History.

 Environment

| | |
|---|---|
| **Platform** | AWS Free Tier |
| **Region** | us-east-1 |
| **Access Method** | AWS Management Console |

## 📋 Summary

Individually attaching policies to IAM users does not scale and is 
error-prone to maintain. This lab implements two standard IAM patterns 
used in production environments — **group-based permission inheritance** 
and **service-assumable roles** — and separately walks through manual 
CloudTrail log analysis of the privilege escalation event from Day 2, 
demonstrating the raw evidence a security analyst would review during 
incident response.

 Implementation

 1. IAM Groups — Permission Inheritance

**Rationale:** Rather than attaching policies to individual users, 
permissions should be attached to a group, with users added as members. 
This centralizes permission management — updating one group updates 
access for every member.

Steps performed:
- Created an IAM group (`test-developer`)
- Attached the AWS managed policy `ReadOnlyAccess` to the group
- Created a new IAM user (`test-developer-user`) and added it directly 
  to the group during creation
- Verified via the user's **Permissions** tab that `ReadOnlyAccess` was 
  listed as inherited *"via group test-developer"* rather than attached 
  directly to the user

Validation:
- Group's **Users** column updated from `0` to `1` after user assignment
- User's Permissions tab confirmed policy attribution to the group, 
  not the user itself

### 2. IAM Roles — Service-Assumable Temporary Access

Rationale: Hardcoding long-lived access keys into compute resources 
(e.g., EC2 instances) is a common source of credential leakage. IAM 
Roles eliminate this by allowing AWS services to request short-lived, 
automatically-rotated credentials at runtime.

Steps performed:
- Created an IAM role (`EC2-S3-ReadOnly-Role`)
- Configured **EC2** as the trusted entity (`ec2.amazonaws.com`) — 
  meaning only EC2 instances can assume this role
- Attached the `AmazonS3ReadOnlyAccess` managed policy
- Reviewed the generated trust policy and instance profile ARN
**Trust policy (excerpt):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Key distinction from IAM Users/Groups: A role has no long-term 
credentials and cannot log in on its own — it is *assumed* temporarily 
by a trusted entity, with AWS automatically issuing and rotating 
short-lived session credentials in the background.

 3. CloudTrail Forensic Analysis (Day 2 Event Review)

Revisited the `CreatePolicyVersion` event generated during the Day 2 
privilege escalation lab, using CloudTrail Event History filtering by 
event name, to practice manual log analysis independent of automated 
detection tooling.

Analyst workflow used:
1. Filtered Event History by `Event name = CreatePolicyVersion`
2. Opened the matching event and extracted the full JSON record
3. Cross-referenced `userIdentity.userName`, `sourceIPAddress`, and 
   `requestParameters.policyDocument` to reconstruct the full attack 
   narrative without relying on GuardDuty or any automated alerting

This mirrors real SOC analyst workflows performed manually before or 
alongside automated detection tooling (e.g., GuardDuty, SIEM correlation 
rules).

 Key Concepts Applied

Group-Based Access Control — Permissions attached at the group 
level scale predictably; adding/removing a user from a group is 
sufficient to grant/revoke access, with no per-user policy editing.

IAM Roles vs. Users/Groups — Roles are identity-less, credential-less 
constructs assumed temporarily by a trusted principal (a service, 
account, or federated identity), in contrast to the permanent, 
directly-authenticated nature of IAM users.

Manual Log Forensics — Automated detection (e.g., GuardDuty) is 
valuable but should be complemented by the ability to manually interpret 
raw CloudTrail records, since not every environment or event type will 
have automated coverage.

 Key Takeaway

> Scalable IAM architecture separates *how permissions are granted* 
> (directly to a user, inherited via a group, or temporarily assumed via 
> a role) based on *who or what* needs access and for *how long* — and 
> the underlying CloudTrail record for any of these actions provides 
> sufficient detail for full forensic reconstruction without additional 
> tooling.

 Cleanup Performed
 Deleted `test-developer-user`
- Deleted `test-developer` group
- Deleted `EC2-S3-ReadOnly-Role`

 References
- AWS Documentation — *IAM User Groups*
- AWS Documentation — *IAM Roles for EC2*
- AWS Documentation — *CloudTrail Event History*

---
*Previous: [← Day 2 — CloudTrail & Privilege Escalation](./day-02-privesc-lab.md)*
