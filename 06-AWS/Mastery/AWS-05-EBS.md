---
title: EBS — Elastic Block Store
type: reference
module: aws
status: ✅ Active Reference
interview-relevance: 🔴 Maximum
tags:
  - aws
  - devops
  - cloud
  - interview
  - masterreview
order: 5
up: "[[AWS-Mastery-MOC]]"
prev: "[[AWS-04-EC2]]"
next: "[[AWS-06-AMI]]"
origin: DevOps-Mastery/06-AWS/AWS Mastery — Complete DevOps Notes.md
---

# EBS — Elastic Block Store

<!-- nav -->
↑ [[AWS-Mastery-MOC|AWS Mastery — Complete DevOps Notes]] · ← [[AWS-04-EC2|EC2 — Elastic Compute Cloud]] · → [[AWS-06-AMI|AMI — Amazon Machine Image]]
<!-- /nav -->

## Concept

EBS provides **durable, high-performance block storage** for EC2 instances. Think of it as a virtual hard drive attached to your VM.

**Key characteristics:**

- Persistent — data survives even when EC2 stops/terminates (unless configured otherwise)
- Attached to ONE instance at a time (with exceptions like io2 Multi-Attach)
- Region and AZ specific — can only attach to instances in the same AZ

## EBS vs Instance Store

```
EBS
    ├── Persistent (data survives instance termination)
    ├── Detachable, reattachable
    ├── Snapshots for backup
    ├── Type: gp3, gp2, io1/io2, st1, sc1
    └── Use for: root volumes, databases, application data

Instance Store
    ├── Ephemeral (data GONE on stop/terminate)
    ├── Physically attached to host — fastest possible
    ├── No snapshots
    └── Use for: temporary caches, scratch space
```

**Rule:** If it's data you can't lose, use EBS (or S3, or RDS).

## EBS Volume Types

```
gp3 (Newest general-purpose SSD)
    ├── Default choice for most workloads
    ├── 3,000 IOPS baseline, scalable to 16,000
    ├── Cheaper than gp2
    └── Use for: root volumes, most apps

gp2 (Older general-purpose SSD)
    ├── Legacy — use gp3 instead
    ├── IOPS scales with volume size
    └── Being phased out

io1 / io2 (Provisioned IOPS SSD)
    ├── High-performance, mission-critical workloads
    ├── Up to 64,000 IOPS (io2 Block Express: 256,000 IOPS)
    ├── More expensive
    └── Use for: high-transaction databases, latency-sensitive apps

st1 (Throughput-optimized HDD)
    ├── Big data, log processing, sequential I/O
    ├── Low cost per GB
    └── Use for: data warehouses, log analytics

sc1 (Cold HDD)
    ├── Lowest cost
    ├── Infrequently accessed
    └── Use for: cold archives, backup
```

**Real world context:** Default choice = `gp3`. Only switch to `io2` for high-transaction databases where you can measure the IOPS requirement.

## Key Features

**1. Built-in Redundancy**

- Every EBS volume is automatically replicated within the same AZ
- Protects against hardware failure
- BUT — a single AZ failure = your volume is unavailable
- Solution: cross-AZ replication via snapshots

**2. Encryption**

- Enable encryption at rest with a single checkbox
- Uses AWS KMS keys
- No performance penalty
- Encrypt everything by default in production

**3. Snapshots (Backup)**

- Point-in-time copy of your volume
- Stored in S3 (behind the scenes)
- Incremental — only changed blocks are stored after the first snapshot
- Can copy to another AZ or region
- Can create a new volume from a snapshot

**4. Scalable**

- Resize the volume without stopping the instance
- Increase size, change type, adjust IOPS/throughput — all online
- After resizing, run `growpart` + `resize2fs` on Linux to expand filesystem

## Practical: Attaching EBS to EC2

```bash
# Create a new volume (via CLI)
aws ec2 create-volume \
  --volume-type gp3 \
  --size 20 \
  --availability-zone ap-south-1a

# Attach to an instance
aws ec2 attach-volume \
  --volume-id vol-xxxxx \
  --instance-id i-xxxxx \
  --device /dev/xvdf

# On the EC2 instance (Linux):
sudo lsblk                              # confirm new disk (e.g., /dev/xvdf)
sudo mkfs.ext4 /dev/xvdf                # format it
sudo mkdir /data                        # create mount point
sudo mount /dev/xvdf /data              # mount it
echo '/dev/xvdf /data ext4 defaults 0 0' | sudo tee -a /etc/fstab  # persist across reboots
```

## Snapshot Workflow

```
Create Snapshot
    ├── aws ec2 create-snapshot --volume-id vol-xxx
    ├── First snapshot = full backup
    └── Subsequent snapshots = incremental (only changed blocks)

Copy to Another AZ
    ├── Create snapshot → snapshot lives in region (multi-AZ)
    ├── Create new volume from snapshot in target AZ
    └── Attach new volume to instance in target AZ

Copy to Another Region
    ├── aws ec2 copy-snapshot --source-region ap-south-1 --destination-region us-east-1
    ├── Full copy across regions
    └── Used for disaster recovery
```

## Real Production Use Case

**Database on EBS:**

```
MySQL/PostgreSQL on EC2
    ↓
Root volume:  gp3 20GB (OS)
Data volume:  io2 500GB with 20,000 IOPS (database files)
    ↓
Daily snapshots via AWS Backup
    ↓
Cross-region copy for DR
```

**Storage extension scenario:**

```
Application logs filling /var/log
    ↓
Attach new 100GB EBS volume as /dev/xvdf
    ↓
mkfs.ext4 → mount at /var/log
    ↓
Move existing logs, restart app
    ↓
Root volume freed up, no downtime
```

## Memory Hints

- **EBS** = external hard drive for your VM
- **Snapshot** = backup photograph of your drive at a moment in time
- **gp3** = default choice (most cases)
- **io2** = expensive, only for high-IOPS databases
- **AZ-specific** = same AZ as your instance, always

## Interview Q&A — EBS

**Q: Difference between EBS and Instance Store?**

> "EBS is persistent block storage attached over the network — data survives instance stop and terminate. You can snapshot it, detach it, reattach it to another instance in the same AZ.
>
> Instance Store is ephemeral — physically attached to the host, fastest possible I/O, but wiped when the instance stops or terminates. No snapshots, no detach.
>
> Rule of thumb: EBS for anything you can't afford to lose, Instance Store for scratch space, temp files, or high-performance caching that you can rebuild.
>
> Common pattern: root volume on EBS (persistent), Instance Store for a Redis cache that can be rebuilt from RDS if lost."

**Q: Your database is running out of storage. Walk me through extending EBS.**

> "Zero downtime EBS extension flow. First, in AWS console or CLI, modify the volume — increase the size. AWS handles this online, no instance restart needed. Takes a few minutes for the modification to complete.
>
> Then on the Linux instance: `lsblk` confirms the new size at the block device level. Run `growpart /dev/xvda 1` to extend the partition to use the new space. Finally, `resize2fs /dev/xvda1` extends the ext4 filesystem online.
>
> Total time — 5 minutes. Zero downtime, zero data loss. This is why we use LVM in some cases too — same concept, different layer of abstraction.
>
> Common mistake: extending the volume in AWS but forgetting to resize the filesystem — the disk shows more space but the filesystem still thinks it's the old size. Two-step process, never skip step two."

**Q: How do you back up EBS volumes and restore in another region?**

> "EBS snapshots are the answer. Snapshots are incremental — first one is full backup, subsequent ones only store changed blocks. Stored in S3 behind the scenes, extremely durable.
>
> Backup workflow: `aws ec2 create-snapshot --volume-id vol-xxx` — or better, use AWS Backup with a scheduled backup plan. AWS Backup handles retention policies, tagging, and cross-region replication automatically.
>
> Restore in another region: `aws ec2 copy-snapshot --source-region ap-south-1 --destination-region us-east-1 --source-snapshot-id snap-xxx`. Then in the target region, create a new volume from that snapshot, attach to an instance.
>
> For production DR: daily automated backups via AWS Backup, cross-region copy to a DR region, retention of 30 days for daily backups plus 12 months of monthly backups. RTO under 30 minutes for restore, RPO 24 hours max."
