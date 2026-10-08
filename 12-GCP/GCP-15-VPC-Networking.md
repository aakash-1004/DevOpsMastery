---
title: VPC Networking
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
order: 16
up: "[[GCP-MOC]]"
prev: "[[GCP-14-GKE]]"
next: "[[GCP-16-BigQuery]]"
origin: DevOps-Mastery/12-GCP/GCP Mastery.md
---

# VPC Networking

<!-- nav -->
↑ [[GCP-MOC|GCP Mastery — Complete DevOps Notes]] · ← [[GCP-14-GKE|GKE — Google Kubernetes Engine]] · → [[GCP-16-BigQuery|BigQuery]]
<!-- /nav -->

> **Tags:** #gcp #vpc #networking #mental-model-shift
> **Status:** Critical — this is THE section where GCP mental model diverges most from AWS and Azure. Get this wrong in an interview and it signals you haven't really used GCP.
> **Interview Relevance:** ⭐⭐⭐⭐⭐

## Concept

A **GCP VPC (Virtual Private Cloud)** is a **global** software-defined network. Subnets inside a VPC are regional, but the VPC itself spans every GCP region. A single VPC can have subnets in `asia-south1`, `us-central1`, and `europe-west3` simultaneously — all natively connected, no peering, no VPN.

This is the single biggest architectural divergence between GCP and AWS/Azure. Getting it wrong wastes weeks in a migration.

## AWS Equivalent

- **VPC (Virtual Private Cloud)** — regional (bound to a single region), subnets are AZ-scoped.

## Azure Equivalent

- **VNet (Virtual Network)** — regional (bound to a single region), subnets are inside the VNet region.

## Key Differences from BOTH

| Aspect | AWS VPC | Azure VNet | GCP VPC |
|---|---|---|---|
| Scope | Regional | Regional | **Global** |
| Subnet scope | AZ (one AZ per subnet) | Regional (spans zones in region) | Regional (one subnet spans all zones in a region) |
| Cross-region connectivity | VPC Peering or Transit Gateway | VNet Peering (regional or global) | **Native — subnets in different regions in the same VPC talk directly** |
| Firewall rules | Security Groups (stateful, per-ENI) + NACLs (stateless, per-subnet) | NSGs (stateful, per-NIC or per-subnet) | **VPC Firewall Rules (stateful, VPC-wide, filtered by tags/SAs)** |
| Firewall attachment | Per-instance or per-subnet | Per-NIC or per-subnet | **VPC-level, applied by tag/service account/CIDR** |
| Route propagation | Route tables per subnet | Route tables per subnet | Routes are VPC-level, filtered by tags |
| Default deny | SGs default-deny inbound | NSGs default-deny inbound | **Default VPC has default-allow rules; custom VPCs default-deny** |
| Pod IPs (K8s) | VPC CNI assigns VPC IPs | Azure CNI or overlay | **Alias IPs — dedicated pod CIDR block per node** |

**The mental shifts to internalize:**

1. **VPC is global.** You almost never need multiple VPCs for multi-region deployments. In AWS, a global app is 5 VPCs + peering + Transit Gateway. In GCP, it's 1 VPC.
2. **Firewall rules aren't attached to instances or subnets** — they're VPC-level, and they select targets by **network tags** or **service accounts**. Add tag `web-server` to your firewall rule, add tag `web-server` to any instance, done.
3. **No default NACL equivalent.** GCP firewall rules are stateful only. No stateless layer. Simpler, but you lose the belt-and-suspenders defense-in-depth pattern.
4. **Auto mode vs Custom mode VPCs.** Auto mode auto-creates one subnet per region with pre-defined CIDRs. Convenient for demos, banned in production because CIDRs overlap across projects. **Always create Custom mode VPCs in production.**
5. **Shared VPC** — host project owns the VPC, service projects consume subnets. Cross-project networking without peering. Very common in enterprise GCP setups.

## Real Configuration

```bash
# ==========================================
# CUSTOM VPC — production-grade
# ==========================================

# Create the VPC (custom mode — no auto subnets)
gcloud compute networks create prod-vpc \
  --subnet-mode=custom \
  --bgp-routing-mode=regional \
  --mtu=1460

# --bgp-routing-mode=regional  : Cloud Router only advertises routes from its region
#                                (use 'global' if you want cross-region route advertisement)

# Add subnets — one per region
gcloud compute networks subnets create prod-subnet-asia \
  --network=prod-vpc \
  --region=asia-south1 \
  --range=10.10.0.0/20 \
  --secondary-range=pods-range=10.100.0.0/16,services-range=10.200.0.0/20 \
  --enable-private-ip-google-access

gcloud compute networks subnets create prod-subnet-us \
  --network=prod-vpc \
  --region=us-central1 \
  --range=10.20.0.0/20

# --enable-private-ip-google-access : instances without external IPs can reach
#                                     Google APIs (Cloud Storage, BigQuery, etc.)
# --secondary-range                 : for GKE VPC-native pod/service IPs

# ==========================================
# FIREWALL RULES — tag-based
# ==========================================

# Rule 1: Allow SSH from IAP (Google's identity-aware proxy)
gcloud compute firewall-rules create allow-iap-ssh \
  --network=prod-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=35.235.240.0/20 \
  --target-tags=ssh-allowed

# 35.235.240.0/20 = IAP's source range. IAP proxies SSH from the browser,
# so opening SSH ONLY to this range means no public SSH exposure.

# Rule 2: Allow HTTP/HTTPS from anywhere to web servers
gcloud compute firewall-rules create allow-web \
  --network=prod-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:80,tcp:443 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=web-server

# Rule 3: Allow web servers to reach DB — using service account, not tag
gcloud compute firewall-rules create allow-web-to-db \
  --network=prod-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:5432 \
  --source-service-accounts=web-sa@my-project-956179.iam.gserviceaccount.com \
  --target-service-accounts=db-sa@my-project-956179.iam.gserviceaccount.com

# Service-account-based firewalling is the strongest pattern:
# Even if someone spoofs a tag (unlikely but possible via impersonation),
# they can't spoof running as a different service account.

# Rule 4: Explicit deny SSH from internet (defense in depth)
gcloud compute firewall-rules create deny-public-ssh \
  --network=prod-vpc \
  --direction=INGRESS \
  --action=DENY \
  --rules=tcp:22 \
  --source-ranges=0.0.0.0/0 \
  --priority=900

# Lower priority number = higher precedence. Priority 900 beats default rules.

# ==========================================
# CLOUD NAT — outbound internet from private VMs
# ==========================================

gcloud compute routers create prod-router \
  --network=prod-vpc \
  --region=asia-south1

gcloud compute routers nats create prod-nat \
  --router=prod-router \
  --region=asia-south1 \
  --nat-all-subnet-ip-ranges \
  --auto-allocate-nat-external-ips

# ==========================================
# VPC PEERING (for cross-VPC connectivity)
# ==========================================

# Peer prod-vpc with a shared services VPC
gcloud compute networks peerings create prod-to-shared \
  --network=prod-vpc \
  --peer-network=shared-services-vpc \
  --peer-project=shared-services-project

# Peering is non-transitive: A↔B, B↔C does NOT give A↔C.
```

## Production Use Case

**Scenario:** Company runs workloads in `asia-south1` (Mumbai) and `us-central1` (Iowa). They're migrating from AWS where they had two VPCs (one per region) connected via Transit Gateway. Asks: "Do we need two GCP VPCs?"

**Answer:**

> "No — and this is one of the biggest wins moving from AWS to GCP.
>
> Create **one custom-mode VPC** globally. Add two subnets: one in each region with non-overlapping CIDRs. Instances in Mumbai and Iowa can talk to each other over Google's backbone, natively, no peering, no Transit Gateway equivalent, no per-GB inter-region peering cost (though egress between regions still applies).
>
> Firewall rules apply once, VPC-wide, filtered by tag or service account. Instead of duplicating security groups across two VPCs and keeping them in sync, we define `allow-app-to-db` once and it works everywhere.
>
> Where I would use a second VPC: strict tenant isolation (a separate customer's workload), or environments that legally can't be on the same network (compliance boundary). Otherwise, one global VPC with well-designed subnet CIDRs is the pattern.
>
> Migration mapping:
> - AWS VPC (Mumbai) → subnet in `asia-south1`, CIDR 10.10.0.0/20
> - AWS VPC (Ohio) → subnet in `us-central1`, CIDR 10.20.0.0/20
> - AWS Transit Gateway → gone. Native cross-region routing.
> - AWS Security Groups → GCP Firewall Rules with `--target-service-accounts`
> - AWS NACLs → no equivalent. Rely on defense-in-depth through firewall rules and IAM."

## Console Path

- Console → **VPC network** → **VPC networks**
- **VPC network** → **Firewall** for rules
- **VPC network** → **Routes** for custom routes
- **Hybrid Connectivity** → **Cloud Routers / VPN / Interconnect**

## Memory Hint

- **VPC is GLOBAL, subnets are regional** — the opposite of AWS/Azure
- **Firewall rules are VPC-level, targeted by tag or SA** — not attached to instances
- **Always custom-mode VPCs in production** — auto-mode is for demos
- **Service-account-based firewalling** > tag-based > CIDR-based (strongest to weakest)
- **Shared VPC** = one team owns the network, other teams consume subnets

## Interview Q&A — VPC Networking

**Q: A candidate mentioned GCP VPCs are "just like AWS VPCs but with different names." How would you correct that?**

> "That framing misses the most important architectural difference — GCP VPCs are global, AWS VPCs are regional.
>
> In AWS, if you have workloads in Mumbai and Ohio, you have two VPCs and you need Transit Gateway or VPC Peering to connect them. Each VPC has its own route tables, its own NACLs, its own security groups that you have to keep in sync.
>
> In GCP, one VPC spans both regions. Subnets are regional, but the VPC — the security boundary, the firewall rule scope, the routing plane — is global. That single design choice cascades: fewer VPCs to manage, no cross-region peering resources, firewall rules defined once, cheaper networking topology.
>
> The second big difference is how firewall rules attach. AWS security groups attach to ENIs — every instance has its own set. GCP firewall rules exist at the VPC level and select their targets via network tags or service accounts. Instead of maintaining N security groups for N tiers of workload, you maintain N firewall rules that apply based on labels.
>
> So they're not 'the same with different names' — they're fundamentally different network models, and the GCP model is generally simpler at scale."

**Q: When would you deliberately choose to create multiple VPCs in GCP?**

> "Three legitimate reasons:
>
> 1. **Tenant isolation** — hosting workloads for different customers or business units that must not have any network path between them. Even with well-crafted firewall rules, having them in the same VPC means a misconfiguration could bridge them. Separate VPCs make bridging require explicit peering, which is easier to audit.
>
> 2. **Environment boundary (rare)** — some organizations require prod and non-prod to be on entirely different networks. Personally I prefer separate projects with separate VPCs over separate subnets in one VPC for prod/non-prod — cleaner IAM boundaries, easier billing separation.
>
> 3. **Compliance requirement** — PCI-scoped workloads on their own network, HIPAA data on its own network. Auditors like clean boundaries.
>
> For everything else — multi-region app, microservices, dev/stage — I'd stay in one VPC with well-designed subnet CIDRs and firewall rules targeted by service account."

**Q: Explain the practical difference between using network tags vs service accounts as firewall rule targets.**

> "Both work; service accounts are the stronger choice.
>
> Network tags are string labels applied to instances. Anyone with `compute.instances.setTags` permission on an instance can change its tags — and thus change what firewall rules apply to it. If a tag says 'db-client-allowed,' a compromised or misconfigured instance can add that tag to itself and gain database access.
>
> Service accounts are attached to instances at creation and (mostly) not changeable at runtime — changing an instance's service account requires stopping the instance, which shows up in audit logs. Firewall rules targeting a service account can only be matched by instances actually running as that service account.
>
> In production: use service account targeting for anything security-sensitive — database access, secret manager access, cross-tier traffic. Reserve tags for broad, non-sensitive rules like 'allow health checks to any web tier.' If I'm reviewing someone's GCP setup and I see tag-based firewalls guarding databases, I flag it."
