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
