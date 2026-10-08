---
title: Azure CDN
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
order: 13
up: "[[Azure-MOC]]"
prev: "[[Azure-12-Azure-DNS]]"
next: "[[Azure-14-AKS]]"
origin: DevOps-Mastery/11-Azure/Azure Mastery.md
---

# Azure CDN

<!-- nav -->
↑ [[Azure-MOC|Azure Mastery — Complete DevOps Notes]] · ← [[Azure-12-Azure-DNS|Azure DNS]] · → [[Azure-14-AKS|AKS — Azure Kubernetes Service]]
<!-- /nav -->

## Concept

Content delivery — cache content at edge locations close to users.

## AWS Equivalent

**Azure has TWO products in this space, which is confusing:**

- **Azure CDN** — traditional CDN, older product. Multiple vendor backends (Microsoft, Akamai, Verizon).
- **Azure Front Door** — modern global L7 with CDN + WAF + Load Balancing bundled.

**AWS CloudFront ≈ Azure Front Door** (for new work)
**AWS CloudFront ≈ Azure CDN** (legacy positioning)

**Recommendation:** For anything new, use Front Door. Azure CDN is being deprecated in phases — the Akamai and Verizon tiers are already retired or retiring, Microsoft tier remains but Front Door is the strategic direction.

## Front Door Recap (covered earlier in Section 6)

Since we covered Front Door in the load-balancing section, this is a shorter recap focused on CDN aspects.

**Front Door as a CDN:**
- 100+ edge POPs globally
- Cache static and dynamic content with configurable TTLs
- Compression (gzip, brotli)
- Query string caching modes
- Custom cache rules per path pattern
- Automatic HTTPS with managed certificates
- WAF at edge for security
- Custom domains supported

## Real Configuration

```bash
# Create Front Door profile (Standard SKU is fine for most cases; Premium adds more WAF + private link)
az afd profile create \
  --profile-name fd-prod-web \
  --resource-group rg-prod-web-eastus \
  --sku Standard_AzureFrontDoor

# Create an endpoint (the hostname users hit)
az afd endpoint create \
  --resource-group rg-prod-web-eastus \
  --profile-name fd-prod-web \
  --endpoint-name myapp \
  --enabled-state Enabled

# Endpoint URL: myapp-xxxxxx.z01.azurefd.net

# Create an origin group and add origins (your App Gateways or Blob endpoints)
az afd origin-group create \
  --resource-group rg-prod-web-eastus \
  --profile-name fd-prod-web \
  --origin-group-name web-origins \
  --probe-request-type GET \
  --probe-protocol Https \
  --probe-interval-in-seconds 30 \
  --probe-path /health \
  --sample-size 4 \
  --successful-samples-required 3 \
  --additional-latency-in-milliseconds 50

# Create a route matching all paths
az afd route create \
  --resource-group rg-prod-web-eastus \
  --profile-name fd-prod-web \
  --endpoint-name myapp \
  --route-name default \
  --origin-group web-origins \
  --supported-protocols Http Https \
  --https-redirect Enabled \
  --forwarding-protocol MatchRequest \
  --link-to-default-domain Enabled \
  --patterns-to-match /*
```

## Production Use Case

**Global website: static + dynamic split via Front Door:**

```
Front Door Profile: fd-prod-web (Premium SKU)
├── Endpoint: myapp.azurefd.net (also CNAMEd to www.aakashrao.dev)
├── Origins:
│   ├── Origin group: static-origins
│   │   └── Origin: stprodstatic.z13.web.core.windows.net (Blob $web endpoint)
│   └── Origin group: dynamic-origins
│       ├── Origin: agw-eastus.eastus.cloudapp.azure.com
│       └── Origin: agw-westeurope.westeurope.cloudapp.azure.com
├── Routes:
│   ├── /static/*, /images/*, /css/*, /js/* → static-origins, cache 7 days
│   ├── /api/* → dynamic-origins, no cache
│   └── /* → dynamic-origins, cache 5 min if 200 response
└── WAF policy: OWASP + custom rules

Managed certificate for www.aakashrao.dev.
Rules engine: strip cookies from static requests to improve cache hit ratio.
```

## Memory Hint

- **Front Door** = CloudFront + WAF + Global Accelerator, one service
- **Azure CDN** = older, being deprecated, don't start new projects on it
- **Endpoints** live in profiles; profiles have routes; routes point to origin groups; origin groups contain origins. Deep nesting compared to CloudFront distributions.

## Interview Q&A — CDN

**Q: You inherit a workload using Azure CDN Standard from Microsoft. Would you migrate to Front Door?**

> "Yes, planned migration. Azure CDN is on a phased sunset path — Verizon and some tiers already retired, Microsoft tier still supported but Front Door is the strategic direction. Front Door offers a superset of capabilities: global anycast, WAF v2, rules engine, private link support in Premium.
>
> Migration approach:
>
> First, provision Front Door in parallel. Point origin(s) to the same backend as the CDN.
>
> Second, test on a non-production subdomain to validate cache behavior, TTLs, and any custom rules. CDN-to-Front-Door cache behavior differs subtly — verify your caching semantics still work.
>
> Third, DNS cutover with reduced TTL. Update CNAME from `cdn-endpoint.azureedge.net` to `frontdoor-endpoint.azurefd.net`. Monitor for anomalies.
>
> Fourth, keep the old CDN endpoint alive for a week for rollback safety.
>
> Watch out for: WAF policies aren't portable directly. If you had a WAF on the CDN via a separate service, you'll rebuild it in Front Door's native WAF. Custom cache rules likewise need translation.
>
> The upside: better performance from Front Door's Microsoft-owned edge network, integrated WAF, and one less service to manage."
