# Day 11 — EBS Volumes & Snapshot-Based Backup

`Difficulty: Foundational` · `Focus Area: Storage, Backup & Recovery`

---

## 🎯 Objective

Understand Amazon EBS (Elastic Block Store) as persistent, independent 
block storage for EC2 instances, and demonstrate point-in-time backup 
creation via EBS Snapshots.

## 🧩 Environment

| | |
|---|---|
| **Platform** | AWS Free Tier |
| **Region** | us-east-1 |
| **Volume Type** | gp3, 1 GiB |

## 📋 Problem Statement

Compute (EC2) and storage (EBS) are independent resources in AWS — an 
EBS volume's lifecycle is not necessarily tied to the instance it's 
attached to. Understanding this independence, and how snapshots provide 
point-in-time recovery, is fundamental to designing a resilient backup 
strategy.

## 🧠 Core Concepts

**EBS as Independent Storage** — An EBS volume can persist beyond the 
life of the EC2 instance it was attached to, provided the "Delete on 
Termination" setting is disabled. A persisted volume can subsequently be 
attached to a different instance, with all its data intact.

**Snapshots as Point-in-Time Backups** — A snapshot captures the state 
of an EBS volume at a specific moment, stored durably in Amazon S3 
(managed transparently by AWS). Snapshots are incremental after the 
first full snapshot — only changed blocks are stored in subsequent 
snapshots, reducing storage cost.

**Recovery Workflow** — If a volume is lost or corrupted, a new volume 
can be created directly from a snapshot, restoring data to the exact 
state at the time the snapshot was taken.

## ✅ Implementation

### 1. Provisioned a Standalone EBS Volume
- Created a 1 GiB `gp3` volume independent of any EC2 instance, to 
  isolate the storage lifecycle from compute for this demonstration

### 2. Created a Snapshot
- Initiated snapshot creation from the volume via **Actions → Create 
  snapshot**
- Observed snapshot progress transition from in-progress (`Pending`, 
  99%) to `Completed`

### 3. Cleanup
- Deleted the snapshot
- Deleted the underlying volume

## 📌 Key Takeaway

> EBS volumes are not implicitly backed up — a deliberate snapshot 
> strategy is required for recoverability. Combined with the earlier 
> observation that EC2 termination can optionally preserve or delete 
> attached volumes, a complete backup strategy requires explicitly 
> managing both volume persistence (`Delete on Termination`) and 
> scheduled snapshot creation, rather than assuming either happens 
> automatically.

## 🌍 Real-World Relevance

Production environments typically automate snapshot creation on a 
schedule (e.g., via AWS Backup or Data Lifecycle Manager) rather than 
relying on manual snapshots, ensuring consistent recovery points exist 
without requiring an operator to remember to create them.

## 🔗 References
- AWS Documentation — *Amazon EBS Volumes*
- AWS Documentation — *Amazon EBS Snapshots*

---
*Previous: [← Day 10 — RDS Database Security](./day-10-rds-database-security.md)*
