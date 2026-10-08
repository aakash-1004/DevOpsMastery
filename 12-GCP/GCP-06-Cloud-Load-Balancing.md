---
title: Cloud Load Balancing
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
order: 7
up: "[[GCP-MOC]]"
prev: "[[GCP-05-Cloud-Storage]]"
next: "[[GCP-07-MIG-Managed-Instance-Groups]]"
origin: DevOps-Mastery/12-GCP/GCP Mastery.md
---

# Cloud Load Balancing

<!-- nav -->
↑ [[GCP-MOC|GCP Mastery — Complete DevOps Notes]] · ← [[GCP-05-Cloud-Storage|Cloud Storage]] · → [[GCP-07-MIG-Managed-Instance-Groups|MIG — Managed Instance Groups]]
<!-- /nav -->

**This is where GCP diverges most from AWS/Azure.** In AWS you learn ALB, NLB, GWLB. In Azure you learn Load Balancer, App Gateway, Front Door, Traffic Manager. In GCP, there's **one service — Cloud Load Balancing — with multiple modes**.

## Concept

Cloud Load Balancing is a single product spanning L4 and L7, regional and global, external and internal — you configure the mode at deploy time.

## AWS Equivalent

Depends on mode:
- **Global external HTTPS LB** ≈ CloudFront + ALB combined
- **Regional external HTTPS LB** ≈ ALB
- **Network LB** (passthrough) ≈ NLB
- **Internal HTTPS LB** ≈ Internal ALB

## Azure Equivalent

Depends on mode:
- **Global external HTTPS LB** ≈ Front Door + Application Gateway
- **Regional external HTTPS LB** ≈ Application Gateway
- **Network LB** ≈ Azure Load Balancer (Standard)
- **Internal HTTPS LB** ≈ Internal Application Gateway

## Cloud Load Balancer Modes — The Matrix

| Mode | Layer | Scope | Traffic Type |
|---|---|---|---|
| **Global External Application LB** | L7 | Global | HTTP/HTTPS from internet |
| **Regional External Application LB** | L7 | Regional | HTTP/HTTPS from internet |
| **Internal Application LB** | L7 | Regional | HTTP/HTTPS internal |
| **External Passthrough Network LB** | L4 | Regional | TCP/UDP from internet |
| **Internal Passthrough Network LB** | L4 | Regional | TCP/UDP internal |
| **External Proxy Network LB** | L4 | Global or Regional | TCP with proxy |

**Rule of thumb:**
- Global web app → **Global External Application LB** (this is the standout — one product with anycast IP)
- Regional web app → **Regional External Application LB**
- TCP/UDP internal → **Internal Passthrough Network LB**
- TCP internet-facing → **External Passthrough Network LB**

## Key Differences from BOTH AWS and Azure

**1. Global Load Balancer with a Single Anycast IP**

The Global External Application LB is unique to GCP at the depth GCP does it. One IP address, announced from every Google edge POP worldwide via anycast. Users hit the closest POP automatically, traffic then routes over Google's private backbone to the healthiest backend region.

AWS: you'd combine CloudFront (edge) + ALB (regional). Two services, two configurations, no single anycast IP.

Azure: you'd combine Front Door (edge) + Application Gateway (regional). Same pattern as AWS.

GCP: one Global External Application LB does all of it. Single IP, single config, edge caching + global routing + regional load balancing.

**2. Backend Types Are Extremely Flexible**

Backends can be: MIGs (like ASG), zonal NEGs, serverless NEGs (Cloud Run/Functions/App Engine), internet NEGs (on-prem or another cloud), buckets (static content). You can mix backend types in one LB.

Real example: a single Global External Application LB can route `/api/*` to a MIG running Compute Engine VMs, `/functions/*` to Cloud Functions, and `/static/*` to a Cloud Storage bucket — all with one URL map.

AWS and Azure require more service composition for equivalent architectures.

**3. Cloud Armor for WAF**

Cloud Armor is the WAF layer, integrated with the Application LB. OWASP Top 10 pre-configured rules, custom rules with CEL expressions, rate limiting per IP or per user identity, geo-blocking, DDoS protection.

Similar to AWS WAF or Azure Front Door WAF. Same category, tightly integrated with the LB.

**4. Cloud CDN Integration**

Cloud CDN is enabled with a checkbox on the LB — same LB serves as origin, edge caching is added on top. Fewer services to configure than CloudFront-in-front-of-ALB.

## Real Configuration — Global External HTTPS LB

Building a global HTTPS LB is 6+ resources. Here's the minimum:

```bash
# Assume you have a MIG (web-mig) with an instance template already

# 1. Reserve a global static IP
gcloud compute addresses create web-lb-ip --global

# 2. Managed SSL certificate (Google manages renewal)
gcloud compute ssl-certificates create web-ssl-cert \
  --domains=www.aakashrao.dev \
  --global

# 3. Health check
gcloud compute health-checks create http web-health-check \
  --port=8080 \
  --request-path=/health

# 4. Backend service (defines how to send traffic to backends)
gcloud compute backend-services create web-backend \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=web-health-check \
  --global \
  --enable-cdn

# 5. Add MIG as a backend
gcloud compute backend-services add-backend web-backend \
  --instance-group=web-mig \
  --instance-group-region=us-central1 \
  --balancing-mode=UTILIZATION \
  --max-utilization=0.8 \
  --global

# 6. URL map (routing rules)
gcloud compute url-maps create web-url-map \
  --default-service=web-backend

# 7. HTTPS proxy (attaches URL map to SSL cert)
gcloud compute target-https-proxies create web-https-proxy \
  --url-map=web-url-map \
  --ssl-certificates=web-ssl-cert

# 8. Forwarding rule (attaches proxy to IP)
gcloud compute forwarding-rules create web-forwarding-rule \
  --address=web-lb-ip \
  --global \
  --target-https-proxy=web-https-proxy \
  --ports=443
```

Yes, it's more moving parts than an AWS ALB or Azure App Gateway. The benefit is composability — you can build almost any load balancing topology from these primitives.

## Cloud Armor Attachment

```bash
# Create WAF policy
gcloud compute security-policies create web-armor-policy \
  --description="Web WAF policy"

# Add OWASP rule
gcloud compute security-policies rules create 1000 \
  --security-policy=web-armor-policy \
  --expression="evaluatePreconfiguredExpr('sqli-v33-stable')" \
  --action="deny-403"

# Rate limit: 100 req/min per IP
gcloud compute security-policies rules create 2000 \
  --security-policy=web-armor-policy \
  --expression="true" \
  --action="rate-based-ban" \
  --rate-limit-threshold-count=100 \
  --rate-limit-threshold-interval-sec=60 \
  --ban-duration-sec=600 \
  --conform-action=allow \
  --exceed-action=deny-429 \
  --enforce-on-key=IP

# Attach to backend service
gcloud compute backend-services update web-backend \
  --security-policy=web-armor-policy \
  --global
```

## Production Use Case

**Global three-tier app on one Global External Application LB:**

```
Global External Application LB (anycast IP: 34.120.x.x)
├── SSL cert: managed, auto-renewing, covers www + apex
├── Cloud Armor: OWASP rules + rate limit + geo-block sanctioned countries
├── Cloud CDN: enabled with 5-min TTL for /static/*
│
├── URL Map:
│   ├── /api/* → api-backend (MIGs across us-central1 + europe-west1)
│   ├── /functions/webhook → functions-neg-backend (Cloud Functions in us-central1)
│   ├── /static/* → static-bucket-backend (Cloud Storage bucket, CDN cached)
│   └── /* → web-backend (MIG in us-central1 with cross-region failover to us-east1)
│
└── Backend services:
    ├── web-backend, api-backend: cross-region MIGs, latency-based routing
    ├── functions-neg-backend: serverless NEG
    └── static-bucket-backend: Cloud Storage bucket
```

One IP, one config, one certificate for everything. This is genuinely simpler than the AWS equivalent (CloudFront + ALB + Route 53 + WAF + Lambda@Edge separately).

## Console Path

- Console → **Network Services** → **Load balancing**
- **CREATE LOAD BALANCER** wizard walks through: type (App/Network), scope (Global/Regional), external/internal, backends, frontend
- Existing LB detail: **Frontend**, **Host and path rules** (URL map), **Backend configuration**, **Security** (Armor + SSL)

## Memory Hint

- **One product, many modes** — GCP unifies what AWS/Azure split
- **Global External Application LB** = anycast + edge + WAF + CDN in one thing (AWS: CloudFront+ALB, Azure: Front Door+App Gateway)
- **Backend types** = MIG + serverless NEG + bucket + internet NEG (mix in one LB)
- **URL map** = the routing logic (like ALB rules or App Gateway path routing)
- **Cloud Armor** = WAF layer

## Interview Q&A — Cloud Load Balancing

**Q: You're designing a globally distributed web app. Compare how you'd architect the load balancing layer in AWS, Azure, and GCP.**

> "The architectural pattern is the same — anycast edge, regional origins, cross-region failover — but the number of services differs.
>
> AWS: CloudFront in front for edge caching and anycast, ALB in each region as origin, Route 53 with health-checked latency-based routing for regional failover, AWS WAF attached to CloudFront. Four services composed. Well-understood pattern, but four service configurations to keep in sync.
>
> Azure: Front Door in front for edge and global routing, Application Gateway in each region as regional LB, Azure DNS for the domain, WAF policies on Front Door. Three-to-four services. Slightly cleaner than AWS because Front Door bundles edge + WAF + global routing.
>
> GCP: Global External Application Load Balancer with cross-region backends. One service. Anycast IP is announced from every Google edge POP globally, users hit closest POP automatically, traffic routes over Google's private backbone to the healthiest backend region. Cloud CDN adds edge caching to the same LB. Cloud Armor adds WAF. All configured on one LB resource.
>
> The GCP architecture is genuinely simpler for global apps. Fewer moving parts, one certificate to manage, one URL map to edit, one place to look for logs and metrics.
>
> AWS wins on maturity and ecosystem — more third-party integrations, more documentation, larger community. Azure wins on WAF sophistication and rules engine at edge. GCP wins on architectural simplicity for global apps.
>
> For a genuinely global workload where you don't need AWS-specific integrations, GCP's LB is the cleanest choice."

**Q: A team complains that CloudFront + ALB feels 'over-engineered' for their needs. How does GCP simplify this?**

> "That team is the perfect candidate for GCP's Global External Application LB.
>
> The mental model shift: instead of thinking 'CDN in front of load balancer,' think 'load balancer with edge and CDN built in.'
>
> On GCP, you create one Global External Application LB. Toggle Cloud CDN on the backend service — done, edge caching added. Attach a managed SSL certificate — done, HTTPS with auto-renewal. Attach a Cloud Armor policy — done, WAF. Add backends from any region — done, cross-region failover. Point your DNS at the LB's anycast IP — done, deployed.
>
> Fewer resources to reason about. Fewer places for misconfiguration.
>
> Trade-offs:
> - Vendor lock-in is real. This design is GCP-specific and doesn't port cleanly to AWS/Azure.
> - Less granular pricing. You can't independently optimize edge vs origin cost like you can with CloudFront + ALB separately.
> - Some advanced features exist only on the CloudFront side (Lambda@Edge equivalents like Cloud Functions on the edge exist but are less mature).
>
> Practical recommendation: for teams starting fresh on GCP with a global web app, use this LB. It's cleaner and faster to build. For teams porting from AWS with heavy CloudFront customization, plan the architecture more carefully — some patterns don't translate 1:1."
