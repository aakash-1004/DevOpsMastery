---
title: Virtual Machines
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
order: 3
up: "[[Azure-MOC]]"
prev: "[[Azure-02-Entra-ID-and-RBAC]]"
next: "[[Azure-04-Managed-Disks]]"
origin: DevOps-Mastery/11-Azure/Azure Mastery.md
---

# Virtual Machines

<!-- nav -->
↑ [[Azure-MOC|Azure Mastery — Complete DevOps Notes]] · ← [[Azure-02-Entra-ID-and-RBAC|Entra ID & RBAC]] · → [[Azure-04-Managed-Disks|Managed Disks]]
<!-- /nav -->

## Concept

Azure Virtual Machines are IaaS compute — rent a VM by the hour, install what you want.

## AWS Equivalent

**Azure VM = AWS EC2 Instance**

Same fundamental service. Same use cases. Different terminology and defaults.

## Key Differences from AWS

**1. VMs Have Multiple Companion Resources**
When you create an AWS EC2 instance, you get: the instance, an EBS volume, and (optionally) an ENI. That's roughly one thing conceptually.

When you create an Azure VM, you get *separate resources*:
- Virtual Machine (compute)
- Network Interface Card (NIC)
- Public IP address (separate resource, not attached to NIC by default)
- OS Disk (managed disk, separate resource)
- Data Disks (separate resources)
- Network Security Group (attached to NIC or subnet)

**Implication:** deleting the VM does NOT delete its disk or NIC by default. You have to clean up separately, or use the "delete with VM" option, or nuke the RG. Common gotcha for AWS engineers — you'll accumulate orphaned resources fast.

**2. VM Sizes Instead of Instance Types**
AWS uses `t3.micro`, `m5.large`, `c5.xlarge`. Azure uses `Standard_B1s`, `Standard_D2s_v5`, `Standard_E4ds_v5`.

Series letter meanings:
- **B** = Burstable (like AWS `t` family) — cheap, low CPU baseline with bursts
- **D** = General purpose (like AWS `m`) — balanced
- **E** = Memory optimized (like AWS `r`) — DB workloads
- **F** = Compute optimized (like AWS `c`) — CPU heavy
- **N** = GPU (like AWS `p`, `g`)
- **L** = Storage optimized (like AWS `i`)

Number = vCPU count roughly. Suffix `s` = premium storage support. `v5` = generation.

**3. Availability Sets vs Availability Zones**
Azure has TWO HA constructs:
- **Availability Set**: spreads VMs across fault domains WITHIN one datacenter — protects against rack failure, not AZ failure. Older concept.
- **Availability Zone**: spreads VMs across separate physical datacenters — real HA. Newer, better, use this.

For anything new, use AZs. Availability Sets are a legacy pattern from before AZs existed in Azure.

**4. Pricing Model**
- Pay-as-you-go (like AWS on-demand)
- **Reserved Instances** — commit 1 or 3 years, up to 72% off
- **Spot VMs** — like AWS Spot, up to 90% off, evictable
- **Azure Hybrid Benefit** — reuse on-prem Windows/SQL licenses, huge for enterprises

## Real Configuration

```bash
# Create a resource group first
az group create --name rg-vm-lab --location eastus

# Create a Linux VM (this creates NIC, public IP, disk, NSG automatically)
az vm create \
  --resource-group rg-vm-lab \
  --name vm-web-01 \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Standard

# List VMs
az vm list --output table

# Get public IP
az vm show -d --resource-group rg-vm-lab --name vm-web-01 --query publicIps -o tsv

# SSH in
ssh azureuser@<public-ip>

# Stop (still billed for allocated resources)
az vm stop --resource-group rg-vm-lab --name vm-web-01

# Deallocate (stops billing for compute — different from stop!)
az vm deallocate --resource-group rg-vm-lab --name vm-web-01

# Start
az vm start --resource-group rg-vm-lab --name vm-web-01

# Delete
az vm delete --resource-group rg-vm-lab --name vm-web-01 --yes
```

## The Stop-vs-Deallocate Gotcha

This trips up every AWS engineer. In Azure:

- **Stop** = OS shuts down, but VM is still "Allocated" — you're STILL BEING BILLED for compute
- **Deallocate** = VM released from the underlying host, compute billing STOPS (storage still charged)

In AWS, stopping an EC2 instance immediately stops compute billing. Not in Azure.

**Always deallocate, not stop, for cost savings on non-running VMs.** Missing this has caused real bill shock — imagine 20 "stopped" dev VMs racking up $5k/month.

## User Data → Custom Script Extension / cloud-init

AWS user_data has two Azure equivalents:

**cloud-init** — for Linux, runs at first boot, same concept as AWS user_data:
```bash
az vm create \
  --resource-group rg-vm-lab \
  --name vm-web-01 \
  --image Ubuntu2204 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --custom-data cloud-init.yaml
```

**Custom Script Extension** — runs scripts on already-running VMs:
```bash
az vm extension set \
  --resource-group rg-vm-lab \
  --vm-name vm-web-01 \
  --name customScript \
  --publisher Microsoft.Azure.Extensions \
  --settings '{"fileUris": ["https://storage.blob.core.windows.net/scripts/install.sh"], "commandToExecute": "./install.sh"}'
```

## Production Use Case

**Standard 3-tier web app on Azure VMs:**

```
rg-prod-web-eastus
├── VNet: vnet-prod-eastus (10.0.0.0/16)
├── Subnet: snet-web (10.0.1.0/24)
│   ├── VM: vm-web-01 (AZ 1) — Standard_D2s_v5
│   ├── VM: vm-web-02 (AZ 2) — Standard_D2s_v5
│   └── VM: vm-web-03 (AZ 3) — Standard_D2s_v5
├── Load Balancer: lb-web (Public, Standard SKU)
│   └── Backend pool = the 3 VMs
└── NSG: nsg-web (allow 443 from Internet, 22 from bastion only)

Each VM: OS disk (Premium SSD), cloud-init installs Docker + monitoring agent
Managed identity assigned → pulls secrets from Key Vault
```

Deploy via Terraform or Bicep, not portal, for anything real.

## Console Path

- Portal → **Virtual machines** (search bar)
- Create VM: **+ Create** → **Azure virtual machine** → wizard
- VM detail: **Overview** blade → Start / Stop / Restart / Connect buttons
- **Networking** blade shows NIC, NSG rules, public IP
- **Disks** blade shows attached OS and data disks
- **Extensions + applications** blade for post-provisioning scripts

## Memory Hint

- **B-series** = Basic/Burstable (like AWS t3)
- **D-series** = Default (like AWS m5)
- **E-series** = Extra memory (like AWS r5)
- **F-series** = Fast CPU (like AWS c5)
- **Stop ≠ Deallocate** — this is the money-losing trap

## Interview Q&A — Virtual Machines

**Q: Migrating an EC2-based workload to Azure — what surprises AWS engineers?**

> "Three big ones.
>
> First, the resource sprawl. An EC2 instance is one thing. An Azure VM is five to seven separate resources — VM, NIC, public IP, OS disk, data disks, NSG. All need lifecycle management. If you delete the VM through the console, its disk and NIC often stick around costing money. Solution: put all VM-related resources in a dedicated RG and delete the RG when done, or use `--delete-with-vm` flags.
>
> Second, the stop-vs-deallocate distinction. AWS `stop` immediately stops compute billing. Azure has both — `stop` keeps you billed, `deallocate` actually stops billing. Every AWS engineer touching Azure for the first time gets bitten by this. If you have non-prod VMs that don't need to run overnight, deallocate them; use Azure Automation runbooks to schedule this.
>
> Third, Availability Zones aren't universal. Azure has 60+ regions but not all support AZs. If HA design requires multi-AZ, verify region support first — you don't want to design for AZs, deploy, then discover your chosen region only has Availability Sets.
>
> Positive surprise: Managed Identities are cleaner than instance profiles. Enable on the VM, code just works with Azure SDK. No STS credentials fetching in application code."

**Q: How would you handle patching for a fleet of 100 Azure VMs?**

> "For fleet patching I'd use Azure Update Manager — it's the native service, replaces the old Automation Update Management. Enrol the VMs, define maintenance windows per environment, patches apply during the window with automatic reboot handling and pre-scripts if needed.
>
> Structure the fleet with tags so you can group updates: `env=prod`, `env=dev`, `role=web`. Dev environments get patches immediately after Patch Tuesday. Prod gets them a week later with change management approval. Same wave-based rollout pattern as AWS SSM Patch Manager.
>
> For the truly modern approach though — I'd push for replacing patching with immutable infrastructure. VMs never get patched in place. Instead: Packer builds a new VM image weekly with latest patches. VMSS with new image rolls out gradually. Old VMs get destroyed. This is what I'd aim for as a maturity target, not what I'd necessarily start with on day 1.
>
> Monitoring: Azure Monitor + Log Analytics workspace with the Update Compliance workbook. Any VM more than 30 days behind on critical patches fires an alert. Compliance dashboards get exported weekly for security review."

**Q: You need to run a batch job that takes 4 hours, uses 32 cores, and cost matters more than reliability. What Azure compute would you choose?**

> "Spot VM, without question. Up to 90% discount vs pay-as-you-go for the same hardware. Perfect fit — batch job is fault-tolerant, restartable, cost-sensitive, time isn't super critical.
>
> Choose an F-series or D-series VM with 32 vCPUs — `Standard_F32s_v2` for compute-heavy work. Set eviction policy to 'Deallocate' so if evicted you can restart later without losing the disk. Set max price to pay-as-you-go rate so you pay whatever's current spot price up to that ceiling.
>
> Wrap it in a script that checkpoints progress to Blob Storage every 15 minutes. If evicted mid-run, the restart picks up from last checkpoint.
>
> If the batch is a long-term recurring pattern, upgrade the design: use Azure Batch service, which manages spot VM pools automatically, handles retries, and gives you a job queue abstraction. Same underlying spot VMs, less script glue. That's the AWS Batch equivalent.
>
> For interviews, they're checking whether you know Spot exists AND when to use it. The answer isn't 'always Spot to save money' — Spot for fault-tolerant workloads, on-demand for user-facing latency-sensitive stuff, Reserved for known steady workloads."
