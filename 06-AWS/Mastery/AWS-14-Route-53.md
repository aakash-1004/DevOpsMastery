---
title: Route 53 — DNS
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
order: 14
up: "[[AWS-Mastery-MOC]]"
prev: "[[AWS-13-CloudFormation]]"
next: "[[AWS-15-CloudFront]]"
origin: DevOps-Mastery/06-AWS/AWS Mastery — Complete DevOps Notes.md
---

# Route 53 — DNS

<!-- nav -->
↑ [[AWS-Mastery-MOC|AWS Mastery — Complete DevOps Notes]] · ← [[AWS-13-CloudFormation|CloudFormation — IaC]] · → [[AWS-15-CloudFront|CloudFront — CDN]]
<!-- /nav -->

## Concept

**AWS Route 53** is a scalable DNS service. Named after DNS port 53. Handles:

- Domain registration
- DNS routing
- Health checking

## DNS Refresher

**DNS (Domain Name System)** translates human-friendly domain names to IP addresses.

```
User types: www.example.com
    ↓
DNS lookup: www.example.com → 192.168.0.2
    ↓
Browser connects to 192.168.0.2
```

Without DNS, we'd have to remember IPs. DNS is the internet's phone book.

**Default port:** 53 (both UDP and TCP)

## Route 53 Features

```
Domain Registration
    ├── Register new domains directly
    └── Manage renewal, transfers, WHOIS privacy

Hosted Zones
    ├── Container for DNS records for a domain
    ├── Public hosted zone: internet-facing
    └── Private hosted zone: internal VPC

DNS Records
    ├── A record: domain → IPv4
    ├── AAAA record: domain → IPv6
    ├── CNAME: domain → another domain
    ├── MX: mail server records
    ├── TXT: verification, SPF, DKIM
    ├── NS: nameserver records
    └── SRV: service location records

Routing Policies
    ├── Simple: one record, one target
    ├── Weighted: split traffic (10% to new version)
    ├── Latency-based: route to closest region
    ├── Failover: primary/secondary for DR
    ├── Geolocation: route based on user's country
    └── Multi-value: return multiple IPs (basic load balancing)

Health Checks
    ├── Monitor endpoints
    ├── Trigger DNS failover
    └── Alert via SNS
```

## DNS Record Types Deep Dive

```
A Record (IPv4)
    Example: www.google.com → 12.34.56.78
    Most common — direct IP mapping

AAAA Record (IPv6)
    Example: www.example.com → 2001:db8::1
    IPv6 equivalent of A record

CNAME Record (Canonical Name)
    Example: blog.example.com → www.example.com
    Alias for another domain
    CANNOT be used at zone apex (root domain)

MX Record (Mail Exchange)
    Example: example.com → mail.example.com (Priority 10)
    Directs email to your mail server

TXT Record (Text)
    Example: example.com → "v=spf1 include:_spf.example.com ~all"
    Used for verification, SPF (email spam prevention), DKIM

NS Record (Name Server)
    Example: example.com → ns-123.awsdns-45.org
    Authoritative name servers for the domain

SRV Record (Service)
    Example: _sip._tcp.example.com → 10 60 5060 sipserver.example.com
    Service location — used for VoIP, LDAP, etc.
```

## How Route 53 Works

**Workflow:**

```
1. Domain Registration
   Register a domain (myexample.com) via Route 53 or transfer existing

2. Hosted Zone Creation
   Route 53 creates a hosted zone with 4 nameservers (NS records)

3. DNS Records
   Add A, CNAME, MX, TXT records as needed
   Point them to AWS resources (EC2 IP, ALB, CloudFront, S3)

4. Routing Policies
   Configure how DNS resolves queries (simple, weighted, latency, etc.)

5. Health Checks (optional)
   Monitor endpoints
   Route away from unhealthy endpoints
```

## Routing Policies — Use Cases

**Simple Routing:**

```
myexample.com → 54.123.45.67

One record, one destination. Basic.
```

**Weighted Routing (Canary Deployments):**

```
myexample.com:
    ├── 90% traffic → ALB v1 (old version)
    └── 10% traffic → ALB v2 (new version)

Gradually shift traffic to new version. If issues, revert.
```

**Latency-Based Routing (Global Apps):**

```
User in Europe    → EU-West ALB (lowest latency)
User in Asia      → Mumbai ALB (lowest latency)
User in America   → US-East ALB (lowest latency)

Automatically route to region with best latency.
```

**Failover Routing (Disaster Recovery):**

```
Primary:   Mumbai ALB (healthy) → 100% traffic
Standby:   US-East ALB (backup)

If primary health check fails:
    Route 53 automatically switches to US-East

Automated DR at DNS level.
```

**Geolocation Routing (Compliance/Content):**

```
User in EU    → EU servers (GDPR compliance)
User in China → China-specific content
User elsewhere → Default servers
```

## Health Checks

**Monitor endpoints:**

- HTTP/HTTPS endpoints
- TCP endpoints
- Metric-based (CloudWatch alarms)

**Actions on failure:**

- Route DNS to failover target
- Send SNS notification
- Update DNS records

## Multi-Region Deployment Pattern

```
Primary Region: Mumbai (ap-south-1)
    ├── ALB + EC2 in ASG (across 3 AZs)
    └── RDS Multi-AZ

DR Region: US-East (us-east-1)
    ├── ALB + EC2 in ASG (pilot light mode - minimal instances)
    └── RDS Read Replica

Route 53:
    ├── Latency-based routing normally
    ├── Failover to US-East if Mumbai unhealthy
    └── Health checks monitoring both regions
```

## Use Cases Summary

```
Hosting Websites
    ├── Point domain to EC2, ALB, S3, or CloudFront
    └── Manage DNS records for the site

Load Balancing
    ├── Distribute traffic across endpoints
    └── Basic multi-value or weighted routing

Disaster Recovery
    ├── Health checks + failover routing
    └── Automatic failover on region outage

Multi-Region Deployments
    ├── Latency-based or geolocation routing
    └── Route users to nearest region

Email Configuration
    ├── MX records for mail servers
    └── TXT records for SPF/DKIM/DMARC
```

## Pricing

**Not free — billed for:**

- Hosted zones: $0.50 per zone per month
- DNS queries: $0.40 per million queries
- Health checks: $0.50-$0.75 per health check per month
- Domain registration: varies by TLD ($10-40/year typical)

**No free tier** — charges start immediately.

## Memory Hints

- **Route 53** = "AWS's DNS service" (53 = DNS port)
- **A Record** = "domain → IP"
- **CNAME** = "domain → another domain"
- **Hosted Zone** = "container for DNS records"
- **Routing Policy** = "how to answer DNS queries"

## Interview Q&A — Route 53

**Q: Explain a real production DNS setup for a global web app.**

> "Global e-commerce app with 3 regions: Mumbai, US-East, EU-West.
>
> Domain registered via Route 53 (myapp.com). Hosted zone contains multiple record sets.
>
> Primary user-facing records use latency-based routing. Route 53 measures latency from user's DNS resolver to each region and returns the fastest one. User in Delhi gets Mumbai IP, user in California gets US-East, user in Paris gets EU-West.
>
> Each regional endpoint is an ALB. Health checks monitor each ALB every 30 seconds. If Mumbai ALB fails, Route 53 stops returning it — traffic auto-routes to the healthy regions.
>
> For disaster recovery, we also configure failover routing on the primary region. If Mumbai's ALB fails, traffic fails over to US-East as backup.
>
> Additional records: MX for email via Google Workspace, TXT for SPF/DKIM/DMARC verification, CNAME for www subdomain aliasing to main domain, another CNAME pointing to CloudFront distribution for static assets like images.
>
> Result: users always hit the nearest healthy region. Zero downtime during regional outages. Sub-100ms latency globally."

**Q: What's the difference between A record and CNAME?**

> "A record maps a domain directly to an IPv4 address. `www.example.com → 54.123.45.67`. Route 53 returns the IP, browser connects.
>
> CNAME (Canonical Name) is an alias — maps a domain to ANOTHER domain. `blog.example.com → www.example.com`. Route 53 returns another hostname, which then gets resolved to an IP via another DNS lookup.
>
> Two important CNAME limitations: First, CNAME cannot be used at the zone apex (root domain). You can't have `example.com` as CNAME — must be A record. This is a DNS standard, not AWS specific. Second, CNAME adds an extra DNS lookup, slightly slower than A record.
>
> AWS's solution: Route 53 Alias records. Alias behaves like CNAME but at the zone apex. `example.com → ALB DNS name`. Also faster because Route 53 resolves the alias internally.
>
> Use A record for direct IP mapping. Use Alias when pointing to AWS resources (ALB, CloudFront, S3, other Route 53 records). Use CNAME only for pointing to non-AWS domains from non-apex subdomains."
