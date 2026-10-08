---
title: Cloud DNS
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
order: 13
up: "[[GCP-MOC]]"
prev: "[[GCP-11-IaC-Deployment-Manager-Config-Connector-Terraform]]"
next: "[[GCP-13-Cloud-CDN]]"
origin: DevOps-Mastery/12-GCP/GCP Mastery.md
---

# Cloud DNS

<!-- nav -->
↑ [[GCP-MOC|GCP Mastery — Complete DevOps Notes]] · ← [[GCP-11-IaC-Deployment-Manager-Config-Connector-Terraform|IaC — Deployment Manager, Config Connector, Terraform]] · → [[GCP-13-Cloud-CDN|Cloud CDN]]
<!-- /nav -->

## Concept

Cloud DNS is GCP's managed DNS hosting service. Global anycast, high availability, programmatic API.

## AWS Equivalent

**Cloud DNS ≈ Route 53** (DNS hosting only — Route 53 does more)

## Azure Equivalent

**Cloud DNS = Azure DNS**

## Three-Way Comparison

| Feature | Route 53 | Azure DNS | Cloud DNS |
|---|---|---|---|
| DNS hosting | Yes | Yes | Yes |
| Domain registration | Yes | No (App Service Domains) | Yes (Cloud Domains) |
| Health checks + routing policies | Built-in | Traffic Manager (separate) | (via Load Balancer health checks) |
| Private zones | Yes | Yes | Yes |
| Alias records | Yes (Route 53-specific) | Yes (Azure-specific) | (via LB integration) |

## Key Differences from BOTH

**1. Cloud DNS Doesn't Have Native Routing Policies**

AWS Route 53 has weighted, latency-based, geolocation, and failover routing built in. Azure has Traffic Manager as a separate service. Cloud DNS is DNS only — no routing policies.

For routing intelligence on GCP, you use the Global External Application Load Balancer instead (see [[GCP-06-Cloud-Load-Balancing]]). The LB's anycast IP handles latency-based routing at the load balancer layer, not DNS. Different mental model.

**2. Cloud Domains for Registration**

`Cloud Domains` is GCP's domain registrar service — buy a domain and point it at Cloud DNS with one click. Comparable to Route 53 Domains.

**3. Private DNS Zones**

Same as AWS Route 53 Private Hosted Zones and Azure Private DNS Zones. Resolve internal names within a VPC.

## Real Configuration

```bash
# Create a public DNS zone
gcloud dns managed-zones create aakashrao-dev-zone \
  --dns-name=aakashrao.dev. \
  --description="Public zone for aakashrao.dev" \
  --visibility=public

# Get name servers (point your registrar here if not using Cloud Domains)
gcloud dns managed-zones describe aakashrao-dev-zone --format="value(nameServers)"

# Create an A record
gcloud dns record-sets create www.aakashrao.dev. \
  --zone=aakashrao-dev-zone \
  --type=A \
  --ttl=300 \
  --rrdatas=34.120.10.5

# Update via transaction
gcloud dns record-sets transaction start --zone=aakashrao-dev-zone
gcloud dns record-sets transaction add 34.120.10.6 \
  --name=api.aakashrao.dev. --ttl=300 --type=A --zone=aakashrao-dev-zone
gcloud dns record-sets transaction execute --zone=aakashrao-dev-zone

# Create a private DNS zone
gcloud dns managed-zones create internal-zone \
  --dns-name=internal.aakashrao.dev. \
  --visibility=private \
  --networks=prod-vpc \
  --description="Internal service discovery"
```

## Console Path

- Console → **Network Services** → **Cloud DNS**
- Zone detail: **Record sets**, **Registrations** (if using Cloud Domains)

## Memory Hint

- **Cloud DNS** = "Route 53 without the routing policies" (routing lives at the LB layer)
- **Public zone** = internet-resolvable
- **Private zone + VPC binding** = internal service discovery
- **Cloud Domains** = the registrar-side product (like Route 53 Domains)
