---
title: CloudFront — CDN
type: reference
module: aws
status: ✅ Active Reference
interview-relevance: 🔴 Maximum
tags:
  - aws
  - devops
  - cloud
  - interview
  - masterreview
order: 15
up: "[[AWS-Mastery-MOC]]"
prev: "[[AWS-14-Route-53]]"
next: "[[AWS-16-ECS]]"
origin: DevOps-Mastery/06-AWS/AWS Mastery — Complete DevOps Notes.md
---

# CloudFront — CDN

<!-- nav -->
↑ [[AWS-Mastery-MOC|AWS Mastery — Complete DevOps Notes]] · ← [[AWS-14-Route-53|Route 53 — DNS]] · → [[AWS-16-ECS|ECS — Elastic Container Service]]
<!-- /nav -->

## Concept

**AWS CloudFront** is a Content Delivery Network (CDN) — speeds up delivery of web content by caching it at servers close to users (edge locations).

**Why CDN:** Without CDN, all users hit your origin server. User in Europe accessing a server in Mumbai = 250ms+ latency.

With CDN:

```
User in Europe → CloudFront Edge in London (10ms latency)
    ├── If cached: instant response from edge
    └── If not cached: edge fetches from origin, caches for next user
```

## How CloudFront Works

```
1. User requests myapp.com/logo.png
2. DNS routes to nearest CloudFront edge location
3. Edge checks cache:
   ├── HIT: return cached content (fast)
   └── MISS: fetch from origin, cache locally, return
4. Subsequent users at that edge get cached response
5. TTL expires → edge refetches from origin
```

**Result:** Static content served from 400+ edge locations worldwide with millisecond latency.

## What Gets Cached

**By default:**

- Static content (images, CSS, JS, videos)
- Configurable via cache policies

**Can also cache:**

- Dynamic content (HTML, API responses)
- Configure via cache behaviors and TTLs

**Never cached:**

- Sensitive/user-specific data
- Backend logic
- Configurable via cache policies

## Cache Behavior Controls

```
TTL (Time to Live)
    ├── How long CloudFront caches content before refetching
    ├── Min TTL, Default TTL, Max TTL
    └── Example: cache images for 24 hours

Cache Behaviors
    ├── Different rules for different URL patterns
    ├── /images/* → long cache (1 day)
    ├── /api/* → no cache (dynamic API)
    └── /* → default (1 hour)

Origin Headers
    ├── Cache-Control from origin can override CloudFront
    └── Vary header controls per-user caching
```

## Browser Cache vs CloudFront

```
Browser Cache
    ├── Files cached on USER'S device
    ├── Only helps that individual user
    ├── First visit slow, repeat visits fast
    └── Local optimization

CloudFront (CDN)
    ├── Files cached at edge locations
    ├── Helps ALL users in that region
    ├── First user slow, subsequent users fast
    └── Global optimization

Both work together — browser cache + CDN = best performance
```

## Origin Types

**Where CloudFront fetches content from:**

```
S3 Bucket
    ├── Most common origin
    ├── Static websites
    └── Media files

EC2 Instance
    ├── Direct to EC2 with public IP
    └── Not recommended (no HA)

ALB (Application Load Balancer)
    ├── Preferred for dynamic content
    ├── Load balanced across multiple instances
    └── Multi-AZ HA

Custom Origin
    ├── Any HTTP/HTTPS endpoint
    ├── On-premises servers, other clouds
    └── Any accessible URL
```

## Practical Setup — Static Website with CloudFront

**Full flow:**

```
1. S3 bucket with static website files (index.html, css/, js/, images/)
2. Configure S3 as CloudFront origin
3. Create CloudFront distribution
4. Attach SSL certificate from ACM (free)
5. Configure cache behaviors:
   ├── /images/* → long cache (1 week)
   ├── /css/*, /js/* → medium cache (1 day)
   └── /* → default (1 hour)
6. Point Route 53 alias to CloudFront distribution
7. Done — global CDN, HTTPS, custom domain
```

## Real Production Use Cases

**E-commerce Site:**

```
Static assets (images, CSS, JS)  → CloudFront (cached long)
Product pages (HTML)              → CloudFront (cached briefly)
API calls                         → CloudFront (cache disabled or short TTL)
User dashboard                    → CloudFront (bypass cache — user-specific)
```

**Video Streaming:**

```
Video segments (HLS)  → CloudFront (aggressive caching)
Video manifests       → CloudFront (short TTL)
Live streams          → CloudFront with low-latency streaming
```

**Global App:**

```
Users worldwide → CloudFront (nearest edge)
                → Origin: ALB in one primary region

Effect: users in Europe/Asia see near-local response times
        without needing to deploy in every region
```

## CloudFront Security Features

```
HTTPS Enforcement
    ├── Redirect HTTP to HTTPS
    ├── Modern TLS versions
    └── Free SSL cert via ACM

Signed URLs / Signed Cookies
    ├── Restrict access to paid content
    └── Time-limited access tokens

WAF Integration
    ├── AWS Web Application Firewall
    ├── Block malicious traffic at edge
    └── SQL injection, XSS protection

Geo-Restriction
    ├── Block/allow specific countries
    └── Compliance and content licensing

DDoS Protection
    ├── AWS Shield Standard (free with CloudFront)
    └── AWS Shield Advanced (paid, more protection)

Origin Access Identity (OAI) / Origin Access Control (OAC)
    ├── Restrict S3 access — only CloudFront can access
    └── Users MUST go through CloudFront
```

## Pricing

**Based on:**

- Data transfer OUT to internet
- HTTPS requests
- Origin fetches

**Free tier:** 1TB data transfer + 10M requests per month for first 12 months.

**Real cost example:** Small static website (50GB/month traffic) = ~$5-10/month with CloudFront.

## Memory Hints

- **CloudFront** = "AWS's CDN" (delivery network)
- **Edge Location** = "mini data center close to users"
- **Origin** = "source of truth" (S3, ALB, EC2)
- **TTL** = "how long to cache before refresh"
- **Cache Hit** = "found at edge" (fast)
- **Cache Miss** = "not at edge, fetch from origin" (slower first time)

## Interview Q&A — CloudFront

**Q: How does CloudFront improve performance for a global user base?**

> "Three mechanisms.
>
> First, geographic proximity. CloudFront has 400+ edge locations worldwide. A user in London requesting content from an origin in Mumbai would normally see 200-300ms latency. With CloudFront, they hit a London edge in 5-10ms. Massive latency reduction.
>
> Second, caching. Once content is cached at an edge, subsequent users at that edge get instant responses. The origin isn't hit again until TTL expires. This offloads origin traffic and reduces load.
>
> Third, network optimization. AWS uses its private network backbone between edges and origins — often faster than the public internet. TCP connections are reused. HTTPS handshakes are optimized.
>
> Combined effect: 10x latency reduction and 90% origin load reduction is typical for static content.
>
> For dynamic content, CloudFront can still help with connection pooling and geographic proximity even without caching. And with the new 'Origin Shield' feature, CloudFront can consolidate multiple edge requests to reduce origin load."

**Q: Design a static website deployment on AWS with global availability, HTTPS, and low latency.**

> "The reference architecture: Route 53 + CloudFront + S3 + ACM.
>
> S3 bucket hosts the static files (HTML, CSS, JS, images). Block Public Access enabled — only CloudFront can access it via Origin Access Control (OAC).
>
> CloudFront distribution sits in front of S3. It caches content at 400+ edge locations globally. Cache behaviors: /images/* long TTL, /css/_/js/_ medium TTL, /* default 1 hour.
>
> SSL certificate from AWS Certificate Manager (ACM). Free, auto-renews, attached to CloudFront. Users get HTTPS with a proper certificate.
>
> Route 53 hosted zone for the domain. Alias record for `www.myapp.com` points to CloudFront distribution. Optionally, apex record (`myapp.com`) uses Alias to CloudFront.
>
> Result: global static site, HTTPS, low latency worldwide, DDoS protection via CloudFront + AWS Shield Standard. Total cost for a small site: under $5/month.
>
> If we needed dynamic content later: add API Gateway + Lambda behind CloudFront using the /api/* cache behavior with cache disabled. Backend Node.js on Lambda scales automatically."
