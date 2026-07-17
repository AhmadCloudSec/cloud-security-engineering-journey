# Day 15 — Least Privilege, RBAC & ABAC

`Difficulty: Intermediate` · `Focus Area: Access Control Models, Policy Debugging`

---

## 🎯 Objective

Understand Least Privilege as a formal access control principle, 
compare Role-Based (RBAC) and Attribute-Based (ABAC) access control 
models, and implement an ABAC policy using IAM tag-based conditions — 
including systematic debugging of the resulting access issue using 
AWS's native policy evaluation tooling.

## 🧩 Environment

| | |
|---|---|
| **Platform** | AWS Free Tier |
| **Region** | us-east-1 |
| **Tools** | IAM, S3, AWS CLI, IAM Policy Simulator |

## 📋 Problem Statement

RBAC (role-based grouping, covered practically in Day 3) scales poorly 
as the number of distinct project/resource combinations grows — each 
new combination potentially requiring a new role ("role explosion"). 
ABAC addresses this by granting access based on matching attributes 
(tags) between the requesting principal and the target resource, 
allowing a single policy to govern access across an arbitrary number of 
projects.

## 🧠 Core Concepts

**Least Privilege** — Every identity should hold only the permissions 
required for its specific function, no more. This principle was 
previously demonstrated adversarially in Day 2 (a narrow permission set 
enabling privilege escalation); here it is approached as a proactive 
design discipline.

**RBAC vs. ABAC:**

| | RBAC | ABAC |
|---|---|---|
| Access basis | Fixed role membership | Matching resource/principal attributes (tags) |
| Scaling new projects | Requires a new role per scenario | Requires only new tags; policy is unchanged |
| Example | Day 3's `Developers` group with `ReadOnlyAccess` | A single policy matching `Project` tags across all resources |

**ABAC Policy Condition Structure** — Access is granted only when a 
specified resource tag matches the corresponding principal tag, using 
IAM policy variables:
```json
"Condition": {
  "StringEquals": {
    "aws:ResourceTag/Project": "${aws:PrincipalTag/Project}"
  }
}
```

## ✅ Implementation

### 1. Provisioned Tagged Test Resources
- Created two S3 buckets: `abac-test-project-alpha` (tagged 
  `Project=Alpha`) and a corresponding `Project=Beta` bucket
- Created an IAM user (`abac-test-user`) tagged `Project=Alpha`
- Attached a custom ABAC policy conditioning `s3:*` actions on 
  `aws:ResourceTag/Project` matching `aws:PrincipalTag/Project`

### 2. Encountered and Debugged an Access Failure

Testing via AWS CLI produced a consistent `AccessDenied` result:
