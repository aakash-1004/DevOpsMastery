---
title: "Load Balancer, App Gateway, Front Door"
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
order: 6
up: "[[Azure-MOC]]"
prev: "[[Azure-05-Blob-Storage]]"
next: "[[Azure-07-VMSS-Auto-Scaling]]"
origin: DevOps-Mastery/11-Azure/Azure Mastery.md
---

# Load Balancer, App Gateway, Front Door

<!-- nav -->
↑ [[Azure-MOC|Azure Mastery — Complete DevOps Notes]] · ← [[Azure-05-Blob-Storage|Blob Storage]] · → [[Azure-07-VMSS-Auto-Scaling|VMSS — Auto Scaling]]
<!-- /nav -->

Azure fragments what AWS bundles into ALB/NLB across **four distinct services**. Interviews test whether you know which to pick.

## The Four Options

| Service | Layer | Scope | AWS Equivalent | Use Case |
|---|---|---|---|---|
| **Load Balancer (Basic/Standard)** | L4 (TCP/UDP) | Regional | AWS NLB | High-perf TCP/UDP, internal services |
| **Application Gateway** | L7 (HTTP/HTTPS) | Regional | AWS ALB | Web apps, path/host routing, WAF |
| **Front Door** | L7 (HTTP/HTTPS) | Global | AWS CloudFront + Global Accelerator | Global apps, CDN, edge routing |
| **Traffic Manager** | DNS-level | Global | AWS Route 53 with routing policies | DNS-based geo/failover routing |

**Rule of thumb:**
- HTTP web traffic in one region → **Application Gateway**
- HTTP web traffic globally → **Front Door**
- TCP/UDP (databases, custom protocols) → **Load Balancer**
- DNS-based routing across regions → **Traffic Manager**

## Load Balancer (Standard SKU) — L4

Regional, high-performance L4 load balancer. Handles TCP and UDP. Millions of connections per second, sub-millisecond latency.

- **Public** — internet-facing (frontend gets public IP)
- **Internal** — private, front-end IP inside a VNet

**Two SKUs:** Basic (legacy, being deprecated) and Standard. Always use Standard for new deployments — it's the one with AZ support, HA, and modern features.

## Application Gateway — L7

Regional L7 load balancer. Understands HTTP/HTTPS, does SSL termination, path-based routing, host-based routing, WAF integration.

**Direct AWS ALB analogue.** If you know ALB deeply, App Gateway is a smooth transition — same mental model.

Key features:
- Path routing: `/api/*` → api backend pool, `/admin/*` → admin pool
- Host routing: `api.myapp.com` vs `www.myapp.com`
- WAF v2 built-in (OWASP rules, custom rules)
- Session affinity
- SSL termination + end-to-end TLS
- URL redirect / rewrite

## Front Door — Global L7

Global anycast network. **This is the killer service without a direct AWS single-service equivalent** — it combines CloudFront (CDN) + Global Accelerator (anycast routing) + ALB (L7 routing) + WAF into one.

Use cases:
- Global web apps needing low latency worldwide
- Multi-region active-active with automatic failover
- Static content CDN with dynamic content acceleration
- WAF at edge

## Real Configuration

**Application Gateway (basic web app):**
```bash
# Create the App Gateway (this takes 5-10 mins)
az network application-gateway create \
  --name agw-prod-web \
  --resource-group rg-prod-web-eastus \
  --location eastus \
  --sku Standard_v2 \
  --priority 100 \
  --capacity 2 \
  --vnet-name vnet-prod-eastus \
  --subnet snet-appgw \
  --public-ip-address pip-agw-prod \
  --servers 10.0.1.10 10.0.1.11 10.0.1.12 \
  --http-settings-port 80 \
  --http-settings-protocol Http

# Enable WAF (upgrade to WAF_v2 SKU)
az network application-gateway update \
  --name agw-prod-web \
  --resource-group rg-prod-web-eastus \
  --sku WAF_v2

# Add a path-based routing rule for /api
az network application-gateway url-path-map create \
  --gateway-name agw-prod-web \
  --resource-group rg-prod-web-eastus \
  --name api-path-map \
  --paths /api/* \
  --address-pool api-pool \
  --http-settings apiHttpSettings \
  --default-address-pool webPool \
  --default-http-settings appGatewayBackendHttpSettings
```

## Production Use Case

**Multi-region active-active web app with Front Door:**
```
Front Door Profile (global, single entry point: myapp.azurefd.net)
├── Route: /*  →  Origin Group: web-origins
│                 ├── Origin: agw-eastus.eastus.cloudapp.azure.com (priority 1)
│                 └── Origin: agw-westeurope.westeurope.cloudapp.azure.com (priority 1)
│                 (equal priority = latency-based routing to closest)
│
└── WAF Policy: block SQL injection, XSS, rate-limit /login to 10/min per IP

Each Application Gateway:
├── SSL termination with managed cert
├── Backend pool: 3 VMs across 3 AZs
└── Health probe /health, 15s interval
```

User in India hits Front Door edge in Chennai → routed to eastus origin (closer than westeurope) → App Gateway → backend VM. Failover if eastus origin unhealthy.

## Console Path

- **Load balancer** — Portal → "Load balancers"
- **Application Gateway** — Portal → "Application gateways"
- **Front Door** — Portal → "Front Door and CDN profiles"

Each has: Overview, Frontend IPs, Backend pools, Listeners/Routes, Rules, Health probes, Diagnostic settings.

## Memory Hint

- **Load Balancer** = layer 4, regional, fast (AWS NLB)
- **App Gateway** = layer 7, regional, WAF (AWS ALB)
- **Front Door** = layer 7, GLOBAL, CDN (AWS CloudFront + more)
- "Regional web? App Gateway. Global web? Front Door. TCP? Load Balancer."

## Interview Q&A — Load Balancing

**Q: You need to load-balance a globally distributed web application on Azure. Walk me through your choice.**

> "Front Door, not Application Gateway. Front Door is Azure's global L7 with anycast edge routing — 100+ points of presence worldwide, users hit the nearest edge, traffic then routes to the healthiest and closest backend origin.
>
> Application Gateway is regional — great for a single-region web app, but for global you'd need to deploy App Gateway in each region and then need something in front to route users to the right region. Front Door replaces both layers.
>
> Design: single Front Door profile globally. Origin group contains one App Gateway per region — East US, West Europe, Southeast Asia. Front Door does two things: (1) latency-based routing so users hit the closest region, and (2) automatic failover if a region's App Gateway fails health probes.
>
> Add WAF at the Front Door layer — protects all regions with one policy. Add CDN caching for static assets so `/static/*` serves from edge without hitting origins at all.
>
> AWS equivalent: this is CloudFront + AWS Global Accelerator + ALB per region. Azure bundles them into Front Door + App Gateway pairs. Slightly cleaner Azure architecture in my opinion, though AWS gives you more discrete control."

**Q: When would you use Traffic Manager instead of Front Door?**

> "Traffic Manager is DNS-based routing — it returns different DNS answers based on your routing method (geographic, weighted, priority, performance). Front Door does actual traffic proxying at the L7 level.
>
> Use Traffic Manager when: you're routing to non-HTTP endpoints (databases, custom TCP services). Or when clients need to connect directly to the backend for performance reasons and you can't tolerate the extra proxy hop. Or when you need DNS-only routing without any acceleration.
>
> Use Front Door when: HTTP/HTTPS traffic where you want L7 features — WAF, path-based routing, session affinity, caching, SSL offload.
>
> Cost matters too — Traffic Manager is cheaper because it's just DNS. Front Door prices per rule and per GB proxied. For a small internal DNS failover setup, Traffic Manager makes sense. For customer-facing web apps, Front Door.
>
> AWS analogue: Traffic Manager is closest to Route 53 with routing policies. Front Door is CloudFront-like.
>
> Real practical note: some enterprises use BOTH. Traffic Manager routes between fundamentally different backends (like AWS vs Azure vs on-prem for hybrid setups), Front Door handles the web edge within Azure."
