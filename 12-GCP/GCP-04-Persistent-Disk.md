---
title: Persistent Disk
type: reference
module: gcp
status: ✅ Active Reference
interview-relevance: 🔴 Maximum
tags:
  - gcp
  - devops
  - cloud
  - multicloud
  - interview
  - masterreview
order: 5
up: "[[GCP-MOC]]"
prev: "[[GCP-03-Compute-Engine]]"
next: "[[GCP-05-Cloud-Storage]]"
origin: DevOps-Mastery/12-GCP/GCP Mastery.md
---

# Persistent Disk

<!-- nav -->
↑ [[GCP-MOC|GCP Mastery — Complete DevOps Notes]] · ← [[GCP-03-Compute-Engine|Compute Engine]] · → [[GCP-05-Cloud-Storage|Cloud Storage]]
<!-- /nav -->

## Concept

Persistent Disk is GCP's network-attached block storage — durable, resizable, snapshotable, attaches to Compute Engine VMs.

## AWS Equivalent

**Persistent Disk = AWS EBS**

## Azure Equivalent

**Persistent Disk = Azure Managed Disk**

Same category. Same lifecycle model. Different tier names, different regional/zonal design.

## Three-Way Comparison

| GCP | AWS | Azure | Use Case |
|---|---|---|---|
| `pd-standard` | `st1` / `sc1` | Standard HDD | Cheap, sequential, backup |
| `pd-balanced` | `gp3` (baseline) | Standard SSD | Modern default for general |
| `pd-ssd` | `gp3` (higher IOPS) / `io2` | Premium SSD | High-performance |
| `pd-extreme` | `io2 Block Express` | Ultra Disk | Extreme IOPS |
| `hyperdisk-*` | `io2 Block Express` | Ultra Disk | Newest tier, decoupled IOPS/GB |

## Key Differences from BOTH

**1. Regional vs Zonal Persistent Disks**

Unique GCP feature: **regional persistent disks** replicate synchronously across two zones. AWS EBS is zonal-only (you can snapshot cross-AZ but not have live sync). Azure has ZRS disks (three-AZ sync).

Regional PDs give you HA at the disk level — if a zone goes down, you can attach the disk to a VM in the other zone. Great for databases needing fast failover.

**2. Snapshots Work Cross-Region Natively**

`gcloud compute snapshots create` creates snapshots that are automatically globally accessible. Restore in any region.

AWS: EBS snapshots are per-region unless you explicitly copy across regions.
Azure: similar — snapshots are regional unless explicitly copied.

GCP snapshot model is more geo-flexible out of the box.

**3. Hyperdisk — Decouple IOPS from Size**

Newest tier (2023+): with Hyperdisk, you provision size and IOPS/throughput separately. Need a 100GB disk with 20,000 IOPS? Configure independently.

AWS `gp3` has similar decoupling. Azure Ultra Disk does too. Hyperdisk is GCP's answer.

**4. Automatic Snapshot Schedules**

GCP lets you attach a snapshot schedule policy directly to a disk. Daily/hourly snapshots run automatically without external scheduling. AWS uses AWS Backup or Lifecycle Manager. Azure uses Azure Backup. GCP's is built in and simpler for basic needs.

## Real Configuration

```bash
# Create a 100GB balanced PD
gcloud compute disks create data-disk-01 \
  --project=myapp-prod \
  --zone=us-central1-a \
  --size=100GB \
  --type=pd-balanced

# Attach to VM
gcloud compute instances attach-disk vm-web-01 \
  --disk=data-disk-01 \
  --zone=us-central1-a

# On the VM: format and mount
sudo mkfs.ext4 -F /dev/sdb
sudo mkdir /data
sudo mount /dev/sdb /data
echo '/dev/sdb /data ext4 defaults,nofail 0 0' | sudo tee -a /etc/fstab

# Resize a disk online (no downtime for supported OS)
gcloud compute disks resize data-disk-01 \
  --size=200GB \
  --zone=us-central1-a
# Then on VM: sudo resize2fs /dev/sdb

# Create a snapshot
gcloud compute snapshots create data-disk-01-snap-$(date +%Y%m%d) \
  --source-disk=data-disk-01 \
  --source-disk-zone=us-central1-a

# Create a snapshot schedule policy
gcloud compute resource-policies create snapshot-schedule daily-backup-policy \
  --region=us-central1 \
  --max-retention-days=30 \
  --daily-schedule \
  --start-time=03:00

# Attach policy to disk
gcloud compute disks add-resource-policies data-disk-01 \
  --zone=us-central1-a \
  --resource-policies=daily-backup-policy

# Create a REGIONAL PD (sync-replicated across 2 zones)
gcloud compute disks create data-disk-regional \
  --replica-zones=us-central1-a,us-central1-b \
  --size=200GB \
  --type=pd-balanced \
  --region=us-central1
```

## Production Use Case

**Database VM with regional PD for HA:**

```
VM: db-primary (us-central1-a)
├── Boot disk: pd-balanced 50GB
├── Data disk: pd-ssd 500GB REGIONAL (replicated to us-central1-b)
├── Log disk: pd-balanced 200GB (zonal, less critical)
├── Snapshot schedule: hourly for data disk, daily for logs
└── Failover: standby VM in us-central1-b can attach regional disk within 1 min if primary zone fails
```

Same failover time as Azure SQL DB HA. Cheaper than running two full DB instances with async replication.

## Memory Hint

- **pd-standard** = HDD (cheap, sequential)
- **pd-balanced** = default production choice (like AWS gp3)
- **pd-ssd** = high performance
- **Regional PD** = 2-zone sync replication (uniquely GCP)
- **Snapshots are global** by default — no cross-region copying needed
- **Attach snapshot schedule policy** to disks — set-and-forget backups

## Interview Q&A — Persistent Disk

**Q: Compare Persistent Disk types across the three clouds. When would you pick GCP for storage-heavy workloads?**

> "Baseline mapping: pd-standard ≈ st1 ≈ Standard HDD (cheap sequential). pd-balanced ≈ gp3 ≈ Standard SSD (modern default). pd-ssd ≈ io2 ≈ Premium SSD (high-performance). pd-extreme and Hyperdisk ≈ io2 Block Express ≈ Ultra Disk (extreme IOPS).
>
> GCP-specific wins for storage-heavy workloads:
>
> First, regional PDs — synchronous replication across two zones at the disk layer. If you need sub-minute failover for a stateful workload without running a full replicated database, GCP gives you this natively. AWS engineers usually solve it with DRBD or database-level replication; Azure has ZRS disks but the failover story is different. Regional PDs give you the simplest 'attach disk to VM in surviving zone' failover story of the three.
>
> Second, cross-region snapshots by default. GCP snapshots are stored in a Google-managed multi-region location and can be restored anywhere. AWS and Azure both require explicit cross-region copies which are extra cost and effort.
>
> Third, decoupled IOPS via Hyperdisk. Similar to gp3 and Ultra Disk, but GCP tends to price aggressively for these workloads.
>
> When to choose GCP for storage: databases that need multi-zone HA without complex replication setup, workloads with backup/DR requirements across regions, and analytics workloads with unpredictable IOPS needs where Hyperdisk shines."
