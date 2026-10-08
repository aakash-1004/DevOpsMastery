---
title: Azure DNS
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
order: 12
up: "[[Azure-MOC]]"
prev: "[[Azure-11-ARM-and-Bicep]]"
next: "[[Azure-13-Azure-CDN]]"
origin: DevOps-Mastery/11-Azure/Azure Mastery.md
---

# Azure DNS

<!-- nav -->
↑ [[Azure-MOC|Azure Mastery — Complete DevOps Notes]] · ← [[Azure-11-ARM-and-Bicep|ARM & Bicep]] · → [[Azure-13-Azure-CDN|Azure CDN]]
<!-- /nav -->

## Concept

Azure DNS is a managed DNS hosting service. Hosts DNS zones and records with global anycast infrastructure.

## AWS Equivalent

**Azure DNS = AWS Route 53** (for DNS hosting specifically)

**Traffic Manager = Route 53 routing policies**

The equivalence is clean here. Same fundamental service. Some feature gaps and additions on each side.

## Key Differences from AWS

**1. Public vs Private Zones — Same as Route 53**

- **Public DNS zone** — internet-resolvable, for your domain
- **Private DNS zone** — only resolvable inside linked VNets, for internal service discovery

**2. Alias Records**

Azure DNS supports alias records that automatically track the IP of Azure resources — Traffic Manager profiles, Front Door endpoints, public IPs, etc. If the underlying resource IP changes, DNS updates automatically.

Similar to Route 53 alias records. Same concept, similar limitations (only for supported Azure resource types).

**3. No Domain Registration in Azure DNS Itself**

Route 53 handles both registration and hosting. Azure DNS only does hosting — for registration, you use **App Service Domains** (part of App Service) or buy elsewhere and point NS records to Azure.

Minor annoyance. Most enterprises register domains through a corporate registrar anyway (Cloudflare, GoDaddy, Google Domains, etc.) and point them at Azure DNS for hosting.

**4. Traffic Manager Handles Routing Policies**

Route 53 combines hosting + routing (weighted, latency, failover, geo). Azure splits: **Azure DNS** for basic hosting, **Traffic Manager** for policy-based routing.

## Real Configuration

```bash
# Create a DNS zone
az network dns zone create \
  --resource-group rg-dns-prod \
  --name aakashrao.dev

# Get the nameservers (point your registrar to these)
az network dns zone show \
  --resource-group rg-dns-prod \
  --name aakashrao.dev \
  --query nameServers

# Create an A record
az network dns record-set a add-record \
  --resource-group rg-dns-prod \
  --zone-name aakashrao.dev \
  --record-set-name www \
  --ipv4-address 20.62.146.10

# Create a CNAME
az network dns record-set cname set-record \
  --resource-group rg-dns-prod \
  --zone-name aakashrao.dev \
  --record-set-name blog \
  --cname www.aakashrao.dev

# Create an alias record pointing to a Front Door endpoint
az network dns record-set a create \
  --resource-group rg-dns-prod \
  --zone-name aakashrao.dev \
  --name @ \
  --target-resource /subscriptions/.../frontdoor-name

# Private DNS zone for internal service discovery
az network private-dns zone create \
  --resource-group rg-network-prod \
  --name internal.aakashrao.dev

# Link private zone to VNet
az network private-dns link vnet create \
  --resource-group rg-network-prod \
  --zone-name internal.aakashrao.dev \
  --name link-vnet-prod \
  --virtual-network vnet-prod-eastus \
  --registration-enabled false
```

## Production Use Case

**Public + private DNS split-brain for internal microservices:**

```
Public zone: aakashrao.dev (Azure DNS)
├── A record: www → Front Door IP
├── CNAME: blog → hosted Ghost blog
└── MX records: Google Workspace mail servers

Private zone: internal.aakashrao.dev (linked to prod VNet)
├── A record: db.internal → 10.0.20.5 (private SQL endpoint)
├── A record: cache.internal → 10.0.20.10 (private Redis endpoint)
└── A record: search.internal → 10.0.20.15 (private Elasticsearch)

Apps in the VNet resolve db.internal.aakashrao.dev to the private IP.
Public users can't resolve *.internal.aakashrao.dev at all.
```

Same split-brain DNS pattern as AWS Route 53 with private hosted zones — different implementation, same outcome.

## Memory Hint

- **Azure DNS** = Route 53 hosting (without domain registration built-in)
- **Traffic Manager** = Route 53 routing policies (separate service)
- **Alias record** = auto-tracks Azure resource IPs
- **Private DNS zone** + VNet link = internal service discovery

## Interview Q&A — Azure DNS

**Q: How would you build the DNS equivalent of a Route 53 latency-based routing setup on Azure?**

> "Two-service composition: Azure DNS for zone hosting + Traffic Manager for routing.
>
> Azure DNS hosts your zone — `aakashrao.dev` — with static records for things that don't need routing (MX, TXT, etc.).
>
> For the smart routing, create a Traffic Manager profile with the Performance routing method. Add endpoints for each region — say, `agw-eastus.eastus.cloudapp.azure.com`, `agw-westeurope.westeurope.cloudapp.azure.com`, `agw-southeastasia.southeastasia.cloudapp.azure.com`. Configure health probes.
>
> In Azure DNS, create a CNAME `www.aakashrao.dev → myapp.trafficmanager.net`. Or better, use an Alias record if the apex needs routing.
>
> User in India resolves `www.aakashrao.dev` → CNAME to `myapp.trafficmanager.net` → Traffic Manager returns the endpoint with lowest latency from the user's DNS resolver → user connects to closest App Gateway.
>
> Failover behavior: Traffic Manager health probes detect a failed endpoint and stops returning it for future queries. TTLs matter — set them low (30-60s) so failover propagates fast.
>
> This is more services stitched together than Route 53's single-service approach with routing policies built in. But it works well and gives you the flexibility to swap DNS provider without touching routing logic. Feature parity, more wiring."
