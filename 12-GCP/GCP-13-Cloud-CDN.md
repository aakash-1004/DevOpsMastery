---
title: Cloud CDN
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
order: 14
up: "[[GCP-MOC]]"
prev: "[[GCP-12-Cloud-DNS]]"
next: "[[GCP-14-GKE]]"
origin: DevOps-Mastery/12-GCP/GCP Mastery.md
---

# Cloud CDN

<!-- nav -->
↑ [[GCP-MOC|GCP Mastery — Complete DevOps Notes]] · ← [[GCP-12-Cloud-DNS|Cloud DNS]] · → [[GCP-14-GKE|GKE — Google Kubernetes Engine]]
<!-- /nav -->

## Concept

Cloud CDN caches HTTP(S) content at Google's edge locations globally.

## AWS Equivalent

**Cloud CDN ≈ CloudFront** (but with different attachment model)

## Azure Equivalent

**Cloud CDN ≈ Azure CDN / Front Door CDN** (Front Door is the more comparable modern service)

## Key Differences from BOTH

**1. Cloud CDN Is a Feature of the Load Balancer, Not a Separate Service**

Unlike CloudFront and Front Door which are standalone products, Cloud CDN is enabled with a **checkbox on a Load Balancer backend**. Same LB, same URL map, edge caching added.

Simpler configuration — one resource. Less flexibility — can't have CDN without an LB.

**2. Cache Invalidation**

Same concept as CloudFront invalidations:
```bash
gcloud compute url-maps invalidate-cdn-cache web-url-map \
  --host=www.aakashrao.dev \
  --path="/static/*"
```

## Cloud CDN Configuration

```bash
# Enable Cloud CDN on an existing backend service
gcloud compute backend-services update web-backend \
  --enable-cdn \
  --cache-mode=CACHE_ALL_STATIC \
  --default-ttl=3600 \
  --max-ttl=86400 \
  --global

# Cache modes:
# - CACHE_ALL_STATIC: cache static content, respect Cache-Control on dynamic
# - USE_ORIGIN_HEADERS: fully rely on origin's Cache-Control
# - FORCE_CACHE_ALL: cache everything regardless of headers (dangerous — use carefully)

# Invalidate cache for a path
gcloud compute url-maps invalidate-cdn-cache web-url-map \
  --host=www.aakashrao.dev \
  --path="/static/app.js"
```

## Console Path

- Console → **Network Services** → **Cloud CDN**
- Or directly: Load Balancer → Backend service → **Enable Cloud CDN** toggle

## Memory Hint

- **Cloud CDN** = "CloudFront but bundled into the LB config"
- **One checkbox** on a backend service — that's the entire configuration surface
- **CACHE_ALL_STATIC** default — reasonable starting point

## Interview Q&A — Cloud CDN

**Q: A team wants to add a CDN in front of their existing GCP web application. How does the setup differ from AWS?**

> "Structurally different but conceptually simpler on GCP.
>
> AWS: create a CloudFront distribution, configure origins (usually ALB or S3), set up cache behaviors per path pattern, attach WAF, update DNS to point to CloudFront. Separate resource, separate ID, separate metrics.
>
> GCP: on the existing Global External Application Load Balancer, go to the backend service and toggle 'Enable Cloud CDN.' That's it. Same LB serves as origin, same URL map defines cache behavior per path (via backend service selection), same domain, same certificate, same everything.
>
> Advantages of GCP's approach:
> - Fewer moving parts
> - Cache behavior per path is defined in the same URL map you already use for routing
> - Metrics and logs are unified with the LB
>
> Trade-offs:
> - Less flexibility — CDN and LB are one resource, can't scale them independently
> - CloudFront has more edge features (Lambda@Edge, Origin Shield) that Cloud CDN doesn't match
> - Some enterprise features (custom certificates in specific edge locations, private CDN) are stronger in CloudFront
>
> For most web app CDN needs, Cloud CDN's simplicity wins. For heavy edge computing or specific enterprise CDN requirements, CloudFront remains stronger."
