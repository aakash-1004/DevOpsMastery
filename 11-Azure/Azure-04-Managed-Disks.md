---
title: Managed Disks
type: reference
module: azure
status: ✅ Active Reference
interview-relevance: 🔴 Maximum
tags:
  - azure
  - devops
  - cloud
  - multicloud
  - interview
  - masterreview
order: 4
up: "[[Azure-MOC]]"
prev: "[[Azure-03-Virtual-Machines]]"
next: "[[Azure-05-Blob-Storage]]"
origin: DevOps-Mastery/11-Azure/Azure Mastery.md
---

# Managed Disks

<!-- nav -->
↑ [[Azure-MOC|Azure Mastery — Complete DevOps Notes]] · ← [[Azure-03-Virtual-Machines|Virtual Machines]] · → [[Azure-05-Blob-Storage|Blob Storage]]
<!-- /nav -->

## Concept

Azure Managed Disks are the persistent block storage attached to VMs. Called "managed" because Azure handles the underlying storage account layer for you — you just deal with disks.

## AWS Equivalent

**Azure Managed Disk = AWS EBS Volume**

Same concept. Same lifecycle model (persistent, snapshotable, resizable). Different tier names and defaults.

## Key Differences from AWS

**1. Four Performance Tiers (vs AWS's gp3/io2/st1/sc1)**

| Azure Tier | AWS Equivalent | Use Case |
|---|---|---|
| **Standard HDD** | AWS `st1`/`sc1` | Dev/test, backups, cold data |
| **Standard SSD** | AWS `gp2` (lower end) | Web servers, light workloads |
| **Premium SSD** | AWS `gp3` / `io1` | Production apps, databases |
| **Ultra Disk** | AWS `io2 Block Express` | High-transaction databases, latency-critical |

**Premium SSD is the sane default for production.** Standard SSD for dev. Ultra Disk only when you can prove you need it — it's expensive and requires specific VM sizes.

**2. Bursting Behavior**
Premium SSD supports **credit-based bursting** — small disks can burst above their baseline IOPS for a limited time. Same idea as AWS `gp3` but with a credit system. Useful for spiky workloads on small disks.

**3. Zone-Redundant Storage (ZRS) Tier**
For Premium SSD and Standard SSD, you can pick **ZRS** which replicates the disk across three AZs synchronously. AWS EBS doesn't have this — you'd use snapshots + cross-AZ restore. ZRS gives you HA at the disk level directly. Costs more but simplifies HA design.

**4. Snapshots Work Differently**
Azure snapshots are:
- Full or incremental (you choose)
- Stored in either Standard HDD (cheaper) or Premium SSD (faster restore)
- Cross-region: use `az snapshot copy` or Azure Backup vault

AWS snapshots are always incremental after the first, always stored in S3 behind the scenes. Azure gives you more control but also more decisions to make.

## Real Configuration

```bash
# Create a data disk
az disk create \
  --resource-group rg-vm-lab \
  --name disk-data-01 \
  --size-gb 128 \
  --sku Premium_LRS \
  --location eastus

# SKU options: Standard_LRS, StandardSSD_LRS, Premium_LRS, UltraSSD_LRS
# Add _ZRS to any for zone-redundancy: Premium_ZRS, StandardSSD_ZRS

# Attach to VM
az vm disk attach \
  --resource-group rg-vm-lab \
  --vm-name vm-web-01 \
  --name disk-data-01

# On the VM (Linux): initialize and mount
sudo lsblk                     # find the new disk, likely /dev/sdc
sudo mkfs.ext4 /dev/sdc
sudo mkdir /data
sudo mount /dev/sdc /data
echo '/dev/sdc /data ext4 defaults,nofail 0 0' | sudo tee -a /etc/fstab

# Resize (online, no downtime for supported scenarios)
az disk update --resource-group rg-vm-lab --name disk-data-01 --size-gb 256
# Then on the VM: sudo growpart /dev/sdc 1 && sudo resize2fs /dev/sdc1

# Create a snapshot
az snapshot create \
  --resource-group rg-vm-lab \
  --name snap-disk-data-01-$(date +%Y%m%d) \
  --source disk-data-01
```

## Production Use Case

**Database VM disk layout:**
```
vm-mysql-01 (Standard_E4ds_v5, 4 vCPU, 32GB RAM)
├── OS Disk        (Premium SSD, 128GB) — /
├── Data Disk 1    (Premium SSD, 1TB, LRS)  — /var/lib/mysql
├── Data Disk 2    (Premium SSD, 512GB, LRS) — /var/log/mysql
└── Backup target: Azure Backup vault, daily snapshots, 30-day retention
```

Why split data and logs onto separate disks? Same reason as AWS — separate IOPS budgets, easier to snapshot data disk without hitting log I/O, cleaner to resize independently.

## Console Path

- Portal → **Disks** (search bar)
- Or navigate: VM → **Disks** blade → shows OS disk + attached data disks with `Create and attach a new disk` / `Attach existing disks` options
- Snapshots → **Snapshots** (separate resource type in search)

## Memory Hint

- **LRS** = Locally Redundant (single AZ, cheapest)
- **ZRS** = Zone Redundant (3 AZs, HA built-in)
- **Premium SSD** = default production choice, always
- **Ultra Disk** = "I have measured IOPS need > 20,000" or don't bother

## Interview Q&A — Managed Disks

**Q: Compare Azure Managed Disks tiers to AWS EBS. When would you use Ultra Disk?**

> "The tiers map roughly one-to-one. Standard HDD = AWS `st1` for backup and cold workloads. Standard SSD = AWS `gp2` for basic production. Premium SSD is the workhorse tier — equivalent to AWS `gp3` — this is what I'd use by default for anything in production.
>
> Ultra Disk equates to AWS `io2 Block Express`. It's for high-transaction databases where sustained IOPS above 20,000 and sub-millisecond latency matter. Think Oracle RAC, high-frequency trading data, mission-critical OLTP.
>
> The catch with Ultra Disk: it doesn't work with all VM sizes, has to be in a specific zone, can't be the OS disk. It's optimized for specific extreme workloads, not general use.
>
> Practical decision rule: start with Premium SSD, benchmark actual IOPS and latency needs, only move to Ultra Disk if data shows you need it. Ultra costs multiples more per GB and comes with operational complexity."
