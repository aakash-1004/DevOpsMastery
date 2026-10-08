---
title: "Networking — VNets, Subnets, NSGs"
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
order: 15
up: "[[Azure-MOC]]"
prev: "[[Azure-14-AKS]]"
next: "[[Azure-16-Cross-Cloud-Interview-Master-QA]]"
origin: DevOps-Mastery/11-Azure/Azure Mastery.md
---

# Networking — VNets, Subnets, NSGs

<!-- nav -->
↑ [[Azure-MOC|Azure Mastery — Complete DevOps Notes]] · ← [[Azure-14-AKS|AKS — Azure Kubernetes Service]] · → [[Azure-16-Cross-Cloud-Interview-Master-QA|Cross-Cloud Interview Master Q&A]]
<!-- /nav -->

## Concept

Azure Virtual Networks (VNets) are isolated network environments — your private space in Azure where resources communicate.

## AWS Equivalent

**VNet = AWS VPC**

Same category. Regional. Contains subnets. Rules for traffic control. Similar mental model.

## Key Differences from AWS

**1. VNet is Regional (Same as VPC)**

Unlike GCP where VPCs are global, Azure VNets are regional. Same as AWS VPCs. If you know VPCs, you know this half of VNets.

**2. Subnet Semantics Are Simpler**

AWS distinguishes public vs private subnets by whether they have a route to the Internet Gateway. Azure subnets don't have this hard distinction — a subnet has internet access if:
- Its resources have public IPs, OR
- It routes through NAT Gateway, OR
- Default outbound is enabled (deprecated pattern for new subnets)

More flexible, but slightly less structured than AWS's public/private subnet convention.

**3. NSGs vs Security Groups**

Network Security Group ≈ AWS Security Group + NACL merged.

- **Stateful** like AWS SGs
- Can be attached to **subnet** OR **NIC** (or both — combined)
- Both allow AND deny rules (SGs are allow-only)
- Higher-priority rules override lower-priority

Rule format: priority (100-4096), source, source port, destination, destination port, protocol, action.

Combined attachment: if NSG is on both the subnet and the NIC, traffic must be allowed by BOTH. Restrictive AND, not OR.

**4. Peering + Virtual Network Gateway**

VNet-to-VNet connectivity:
- **VNet Peering** — direct link, low latency, no gateway needed. Cross-region peering supported (extra cost). AWS equivalent: VPC Peering.
- **VNet Gateway** — for hybrid connectivity (VPN or ExpressRoute back to on-prem). AWS equivalent: Site-to-Site VPN / Direct Connect.

**5. Azure Bastion vs Bastion Host EC2**

Azure Bastion is a managed jump box service — provisioned in a VNet, provides browser-based RDP/SSH to VMs without exposing them to internet. AWS you'd typically DIY with a bastion EC2 or use Systems Manager Session Manager. Azure Bastion is cleaner and more secure.

**6. Private Endpoint = PrivateLink**

Azure Private Endpoint gives a resource (SQL, Storage, Cosmos, etc.) a private IP inside your VNet. AWS equivalent: PrivateLink / VPC endpoints. Both essentially remove the need to traverse public internet for accessing PaaS services.

## Real Configuration

```bash
# Create a VNet with an initial subnet
az network vnet create \
  --resource-group rg-prod-network-eastus \
  --name vnet-prod-eastus \
  --address-prefix 10.0.0.0/16 \
  --subnet-name snet-web \
  --subnet-prefix 10.0.1.0/24 \
  --location eastus

# Add more subnets
az network vnet subnet create \
  --resource-group rg-prod-network-eastus \
  --vnet-name vnet-prod-eastus \
  --name snet-app \
  --address-prefix 10.0.2.0/24

az network vnet subnet create \
  --resource-group rg-prod-network-eastus \
  --vnet-name vnet-prod-eastus \
  --name snet-db \
  --address-prefix 10.0.3.0/24

# Create an NSG
az network nsg create \
  --resource-group rg-prod-network-eastus \
  --name nsg-web

# Add rules
az network nsg rule create \
  --resource-group rg-prod-network-eastus \
  --nsg-name nsg-web \
  --name allow-https \
  --priority 100 \
  --source-address-prefixes '*' \
  --destination-port-ranges 443 \
  --access Allow \
  --protocol Tcp \
  --direction Inbound

az network nsg rule create \
  --resource-group rg-prod-network-eastus \
  --nsg-name nsg-web \
  --name allow-ssh-from-bastion \
  --priority 110 \
  --source-address-prefixes 10.0.100.0/24 \
  --destination-port-ranges 22 \
  --access Allow \
  --protocol Tcp \
  --direction Inbound

# Attach NSG to subnet
az network vnet subnet update \
  --resource-group rg-prod-network-eastus \
  --vnet-name vnet-prod-eastus \
  --name snet-web \
  --network-security-group nsg-web

# Peer two VNets
az network vnet peering create \
  --resource-group rg-prod-network-eastus \
  --name peer-prod-to-shared \
  --vnet-name vnet-prod-eastus \
  --remote-vnet /subscriptions/.../vnet-shared-eastus \
  --allow-vnet-access

# Create a NAT Gateway (for outbound internet from private subnets)
az network public-ip create \
  --resource-group rg-prod-network-eastus \
  --name pip-nat-eastus \
  --sku Standard

az network nat gateway create \
  --resource-group rg-prod-network-eastus \
  --name natgw-eastus \
  --public-ip-addresses pip-nat-eastus \
  --idle-timeout 10

az network vnet subnet update \
  --resource-group rg-prod-network-eastus \
  --vnet-name vnet-prod-eastus \
  --name snet-app \
  --nat-gateway natgw-eastus
```

## Production Use Case — 3-Tier VNet Design

```
VNet: vnet-prod-eastus (10.0.0.0/16, region: East US)
│
├── Subnet: snet-web (10.0.1.0/24)        ← "public" tier (App Gateway lives here)
│   ├── NSG: nsg-web
│   │   ├── Allow 443 from Internet
│   │   ├── Allow 80 from Internet (redirect to 443)
│   │   └── Deny everything else
│   └── App Gateway (agw-prod-web)
│
├── Subnet: snet-app (10.0.2.0/24)        ← private tier (VMs, containers)
│   ├── NSG: nsg-app
│   │   ├── Allow 8080 from nsg-web (App Gateway forwards here)
│   │   ├── Allow 443 outbound to Internet (via NAT Gateway)
│   │   └── Deny all other inbound
│   └── VMSS: vmss-prod-web (3 VMs, zones 1/2/3)
│
├── Subnet: snet-db (10.0.3.0/24)         ← isolated tier (databases)
│   ├── NSG: nsg-db
│   │   ├── Allow 1433 from nsg-app (only app tier can reach SQL)
│   │   └── Deny everything else
│   └── Azure SQL Private Endpoint
│
├── Subnet: snet-bastion (10.0.100.0/27)  ← Bastion subnet (fixed name: AzureBastionSubnet)
│   └── Azure Bastion host
│
├── NAT Gateway: natgw-eastus attached to snet-app (outbound only)
└── VNet Peering: to hub VNet for cross-team access

Route tables enforce that traffic between tiers routes correctly.
```

Same three-tier pattern as classic AWS VPC design (public, private-app, private-db), different naming.

## Console Path

- Portal → **Virtual networks**
- VNet detail:
  - **Address space** — CIDR blocks
  - **Subnets** — list, add, remove subnets
  - **Peerings** — VNet-to-VNet connections
  - **DNS servers** — override with custom DNS
  - **Diagram** — visual topology
- **Network security groups** (separate resource type)
- **Route tables** (separate resource type)
- **Network Watcher** — diagnostics, connection troubleshoot, packet capture

## Memory Hint

- **VNet = VPC** (regional, contains subnets)
- **NSG = SG + NACL merged** (stateful, allow AND deny rules)
- **Attach NSG to subnet OR NIC** (or both — additive AND)
- **Private Endpoint = PrivateLink** (private IPs for PaaS services)
- **Azure Bastion = managed jump box** (better than DIY bastion EC2)

## Interview Q&A — Networking

**Q: Design a production 3-tier app network on Azure. Compare choices to AWS.**

> "Reference architecture follows the same three-tier pattern as AWS with Azure-specific tooling.
>
> VNet with a /16 CIDR — plenty of room. Three subnets: web tier /24, app tier /24, db tier /24. Add a Bastion subnet with the mandatory name AzureBastionSubnet, and a NAT Gateway subnet if needed.
>
> Web tier hosts the ingress — Application Gateway with WAF. NSG allows 443 from Internet, everything else denied.
>
> App tier hosts compute — VMSS or AKS node pool. NSG allows only the app port from the App Gateway's NSG (source is the NSG itself, not an IP range). Outbound to Internet via NAT Gateway for OS updates and API calls. No direct inbound from Internet.
>
> DB tier hosts Azure SQL / Cosmos DB endpoints via Private Endpoint — no public exposure. NSG allows the DB port from the app tier NSG only.
>
> Cross-tier traffic enforced by NSG rules using NSG-as-source instead of IP ranges — more resilient because it survives IP changes.
>
> For hybrid connectivity to on-prem: VNet Gateway with ExpressRoute or Site-to-Site VPN.
>
> Comparison to AWS: the design is identical, just different service names. Application Gateway = ALB, NSG = SG (mostly), NAT Gateway = NAT Gateway, Bastion = Bastion. Azure adds Private Endpoint by default for PaaS services which is cleaner than AWS's VPC endpoints for some services.
>
> Governance: Azure Policy enforces the design at scale — 'no subnet can be created without an NSG,' 'no NSG can allow 22 from Internet,' etc. Analogous to AWS Service Control Policies + Config Rules combined."

**Q: A VM in one VNet can't reach a VM in another VNet — same subscription. What could be wrong?**

> "Ordered checklist, from most likely to least.
>
> First: is there a VNet peering between them? By default VNets are isolated. Create bidirectional peering (`--allow-vnet-access` on both sides). Peering is instant, not a route table update.
>
> Second: NSG rules on either end. Even with peering, NSGs still apply. Source VM's outbound NSG must allow the traffic. Destination VM's inbound NSG must allow it. Check both — I've seen people configure just one and wonder why it fails.
>
> Third: Azure Firewall or NVA in the path. If there's a hub-spoke topology, traffic between spokes might route through a hub firewall via User-Defined Routes. Check UDRs on the source subnet — where does traffic to the destination CIDR route? If it's forced through a firewall, check firewall rules.
>
> Fourth: overlapping IP ranges. Peering doesn't work if the two VNets have overlapping CIDR blocks. If VNet A is 10.0.0.0/16 and VNet B is 10.0.0.0/16, they can't peer meaningfully. Fix requires renumbering one of them.
>
> Fifth: DNS. If VMs use FQDNs, is DNS resolving correctly across VNets? Private DNS zones might not be linked to the source VNet. Or default Azure DNS isn't set up to resolve cross-VNet names.
>
> Diagnostic tool: Azure Network Watcher → Connection troubleshoot. Give it source and destination, it tests actual reachability, tells you where traffic drops. Amazing tool for debugging without SSHing anywhere.
>
> AWS equivalent debugging: VPC Reachability Analyzer. Same idea. Same class of tool."
