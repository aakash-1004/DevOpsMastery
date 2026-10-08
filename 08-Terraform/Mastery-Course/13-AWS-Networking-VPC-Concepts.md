---
title: "13 — AWS Networking & VPC Concepts"
type: concept
module: terraform
status: complete
interview-relevance: ★★★★★
tags:
  - aws
  - vpc
  - networking
  - subnets
  - cidr
  - terraform
config: aws-vpc/
up: "[[Terraform-Mastery-MOC]]"
prev: "[[12-Workspaces]]"
next: "[[14-Project-S3-Static-Website]]"
related:
  - "[[AWS-MOC]]"
origin: Terraform-Mastery/13-AWS-Networking-VPC-Concepts.md
---

# 13 — AWS Networking & VPC Concepts

← Back to [[Terraform-Mastery-MOC]]

> **VPC fundamentals get asked in every AWS-touching interview**, and networking troubleshooting scenarios ("EC2 in a private subnet can't reach the internet") are a standard scenario question.

---

## Hierarchy

```
REGION (ap-south-1)
  └── AVAILABILITY ZONE (ap-south-1a, 1b, 1c)   ← physically separate datacentres
        └── SUBNET (10.0.1.0/24)                 ← lives in exactly ONE AZ
              └── EC2 / RDS / ENI
```

- **Region** — a geographic location (Mumbai, N. Virginia, Stockholm). Fully independent; resources don't cross regions without explicit setup.
- **Availability Zone** — one or more discrete datacentres within a region, with independent power, cooling, and networking. **Multi-AZ is how you survive a datacentre failure.**
- **VPC** — a private, isolated virtual network inside one region. Spans all AZs in that region.
- **Subnet** — an IP range within the VPC, pinned to **exactly one AZ**. This is why "multi-AZ" in practice means "subnets in multiple AZs".

> **The rule that gets tested:** a VPC spans AZs; a subnet does not. To be highly available you need one subnet per AZ.

---

## CIDR

**CIDR (Classless Inter-Domain Routing)** — a way of expressing an IP range as `address/prefix-length`.

`10.0.1.0/24`:
- `/24` = first **24 bits are the network portion**
- Remaining 8 bits = host addresses → 2⁸ = 256 addresses
- Range: `10.0.1.0` → `10.0.1.255`

| CIDR | Total addresses | Usable in AWS |
|---|---|---|
| /16 | 65,536 | 65,531 |
| /20 | 4,096 | 4,091 |
| /24 | 256 | **251** |
| /28 | 16 | 11 |

**AWS reserves 5 addresses in every subnet:**
- `.0` — network address
- `.1` — VPC router
- `.2` — DNS (AmazonProvidedDNS)
- `.3` — reserved for future use
- `.255` — broadcast (AWS doesn't support broadcast but reserves it anyway)

> **"How many usable IPs in a /24 subnet in AWS?" → 251, not 254.** This is a favourite gotcha because the Linux answer (254) is wrong here.

**Quick maths:** usable = 2^(32−prefix) − 5

**Design rules:**
- VPC typically `/16` — room to grow
- Subnets typically `/24` — 251 usable, plenty for most workloads
- **CIDRs must not overlap** if you ever want VPC peering or a transit gateway between them. Plan the address space before you build, because changing a VPC CIDR means rebuilding it.
- Use RFC 1918 private ranges: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`

---

## Core components

### Internet Gateway (IGW)
Horizontally-scaled, redundant AWS-managed component allowing communication between VPC instances and the internet. **One per VPC.** Must be created *and attached* to the VPC.

**A subnet is "public" if and only if its route table has a `0.0.0.0/0` route pointing at an IGW.** Nothing else makes a subnet public — not a name, not a tag, not `map_public_ip_on_launch`.

### Route Table
Rules for where traffic goes. Every subnet is associated with exactly one route table (the VPC's main route table if not explicitly associated).

```
Public route table:
  10.0.0.0/16  → local          (implicit, always present, can't be removed)
  0.0.0.0/0    → igw-xxxxx      ← this line makes it public

Private route table:
  10.0.0.0/16  → local
  0.0.0.0/0    → nat-xxxxx      ← outbound only, via NAT
```

### NAT Gateway
Lets instances in a **private** subnet initiate outbound connections to the internet (yum updates, API calls) while **preventing the internet from initiating connections inbound**.

- Lives in a **public** subnet, needs an Elastic IP
- Charged hourly **plus** per GB processed — often a surprisingly large line item
- **Not highly available by default** — one per AZ for real HA, which triples the cost
- NAT Instance is the cheap DIY alternative (an EC2 doing NAT), but you manage HA and patching yourself

> **The classic troubleshooting scenario:** "EC2 in a private subnet can't reach the internet." Cause is almost always no NAT gateway, or a route table missing the `0.0.0.0/0 → nat` entry, or the NAT gateway sitting in a private subnet by mistake.

### Security Group (SG)
Virtual firewall **at the instance/ENI level**.

- **Stateful** — allow inbound, the return traffic is automatically allowed. You don't write egress rules for responses.
- **Allow rules only** — you cannot write a deny rule
- Default: **all inbound denied, all outbound allowed**
- Can reference **another security group** as a source — this is the good pattern: "allow 3306 from the app-tier SG" rather than from a CIDR

### Network ACL (NACL)
Optional firewall **at the subnet level**.

- **Stateless** — you must explicitly allow both inbound *and* the outbound return traffic (including ephemeral ports 1024–65535)
- Supports **allow and deny** rules
- Rules are **numbered and evaluated in order**, first match wins
- Default NACL allows everything both ways

**SG vs NACL — the guaranteed interview question:**

| | Security Group | NACL |
|---|---|---|
| Level | Instance / ENI | Subnet |
| State | **Stateful** | **Stateless** |
| Rules | Allow only | Allow + **deny** |
| Evaluation | All rules together | Numbered, first match wins |
| Default | Deny in, allow out | Allow all both ways |
| Typical use | Primary control | Blanket IP blocks, extra layer |

> **Memory hook:** SG = **bouncer who remembers your face** (stateful). NACL = **numbered checklist at the building entrance, no memory** (stateless). SG is the one you actually configure day to day.

### VPC Peering
Private routing between two VPCs. **Not transitive** — if A peers B and B peers C, A cannot reach C. CIDRs must not overlap. Routes must be added on both sides.

### VPC Endpoints
Private connectivity to AWS services without going over the internet.
- **Gateway endpoint** — S3 and DynamoDB only. Free. Added as a route table entry.
- **Interface endpoint** (PrivateLink) — most other services. Creates an ENI in your subnet. Costs per hour + per GB.

**Why they matter:** a private subnet reaching S3 through a NAT gateway pays NAT data-processing charges. A gateway endpoint is free and keeps the traffic off the internet entirely. Cost-optimisation questions love this.

### Bastion Host (Jump Box)
A hardened instance in a public subnet used to SSH into private-subnet instances. **The modern replacement is AWS Systems Manager Session Manager** — no open port 22, no bastion to patch, full audit logging via CloudTrail. Saying "I'd use SSM Session Manager rather than a bastion" is a current-practice signal.

### Elastic IP
A static public IPv4 address you own. Needed because **an instance's auto-assigned public IP changes on stop/start.**

> **Hard-won lesson from my own environment:** EC2 public IPs drift on stop/start, which breaks DuckDNS records and anything pinned to the IP. Elastic IPs (or a DNS name) are the fix. AWS now charges for all public IPv4 addresses, including in-use EIPs, so this is also a cost line.

### VPC Flow Logs
Capture IP traffic metadata (not payloads) for network interfaces. Sent to CloudWatch Logs or S3. **The tool for "is traffic reaching this instance at all, and is it being ACCEPTed or REJECTed?"** — the definitive way to resolve a security-group-versus-routing argument.

### Direct Connect
Dedicated physical network link from on-premises to AWS. Consistent latency, higher bandwidth, avoids the public internet. Weeks to provision.

### Transit Gateway
A hub-and-spoke network transit hub connecting many VPCs and on-premises networks. **Solves VPC peering's non-transitivity and its n² mesh problem** — with 10 VPCs, full peering needs 45 connections; a transit gateway needs 10.

---

## Build order (memorise this sequence)

```
1.  Create VPC
2.  Create public subnet
3.  Create private subnet
4.  Create Internet Gateway
5.  Attach IGW to VPC
6.  Create route table for public subnet
7.  Add 0.0.0.0/0 → IGW route
8.  Associate public subnet with public route table
9.  (If NAT needed) EIP + NAT gateway in the public subnet
10. Create private route table, 0.0.0.0/0 → NAT
11. Associate private subnet with private route table
```

---

## Config built — `aws-vpc/`

```hcl
resource "aws_vpc" "my_vpc" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "my_vpc" }
}

resource "aws_subnet" "private-subnet" {
  cidr_block = "10.0.1.0/24"
  vpc_id     = aws_vpc.my_vpc.id
  tags = { Name = "private-subnet" }
}

resource "aws_subnet" "public-subnet" {
  cidr_block              = "10.0.2.0/24"
  vpc_id                  = aws_vpc.my_vpc.id
  map_public_ip_on_launch = true            # auto-assign public IPs here
  tags = { Name = "public-subnet" }
}

resource "aws_internet_gateway" "my-igw" {
  vpc_id = aws_vpc.my_vpc.id                # attaching is implicit in Terraform
  tags = { Name = "my-igw" }
}

resource "aws_route_table" "my-rt" {
  vpc_id = aws_vpc.my_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.my-igw.id
  }
}

resource "aws_route_table_association" "public-sub" {
  route_table_id = aws_route_table.my-rt.id
  subnet_id      = aws_subnet.public-subnet.id     # THIS makes the subnet public
}
```

**Two things to notice:**
1. `map_public_ip_on_launch = true` assigns public IPs but **does not make the subnet public** — the route table association does. A subnet with public IPs and no IGW route is a subnet full of unreachable instances with wasted public IPs.
2. `availability_zone` is not specified, so AWS picks. **In real code always pin the AZ** — otherwise a "multi-AZ" design can silently land both subnets in the same AZ.

**Production-grade version would add:**
```hcl
resource "aws_subnet" "public" {
  for_each                = var.public_subnets
  vpc_id                  = aws_vpc.main.id
  cidr_block              = each.value.cidr
  availability_zone       = each.value.az      # explicit AZ
  map_public_ip_on_launch = true
}

resource "aws_vpc" "main" {
  cidr_block           = var.cidr
  enable_dns_support   = true       # required for private DNS / VPC endpoints
  enable_dns_hostnames = true
}
```

---

## Troubleshooting decision tree

**"Can't reach my instance"** — work outside-in:

1. **Does the instance have a public IP?** → check `associate_public_ip_address` / `map_public_ip_on_launch`
2. **Is it in a subnet whose route table has `0.0.0.0/0 → IGW`?** → if not, it's a private subnet regardless of what it's named
3. **Does the SG allow the port inbound from your IP?** → SG is stateful, so no egress rule needed for the response
4. **Does the NACL allow it inbound AND outbound (ephemeral ports)?** → stateless, needs both
5. **Is the service actually listening?** → `ss -tlnp` on the host
6. **VPC Flow Logs** → ACCEPT or REJECT tells you definitively whether it's network or host

> **The single most useful diagnostic distinction:** **timeout = packet silently dropped** → security group, NACL, or routing. **connection refused = it reached the host and nothing is listening** → the service is down. This one distinction eliminates half the search space immediately, and it's the same instinct used for JS7/Keycloak connectivity issues.

---

## Interview Q&A

**Q: Explain the relationship between VPC, availability zone, and subnet.**
> A VPC is a private network within a single region and it spans all availability zones in that region. A subnet is an IP range carved out of the VPC's CIDR, and it lives in exactly one availability zone — it can't span AZs. That's the constraint that drives high-availability design: to survive an AZ failure you need a subnet in each AZ, and your workload spread across them. Availability zones themselves are physically separate datacentres with independent power and networking, which is what makes multi-AZ meaningfully different from multi-instance.

**Q: What makes a subnet public?**
> Only one thing — its route table has a `0.0.0.0/0` route pointing at an internet gateway. Not the name, not tags, not `map_public_ip_on_launch`. That last one is the confusing part: enabling auto-assign public IP gives instances a public address, but if the route table has no IGW route, those instances still can't reach the internet and nothing can reach them. So you end up with unreachable instances holding public IPs you're now paying for.

**Q: Security group versus NACL?**
> Security groups operate at the instance level and are stateful — if I allow inbound on 443, the response traffic is automatically permitted, so I don't write a matching egress rule. They only support allow rules, and all rules are evaluated together. NACLs operate at the subnet level and are stateless — I have to explicitly allow both directions, including the ephemeral port range for return traffic, which is where people get caught. NACLs do support deny rules and are evaluated in numbered order with first match winning. In practice security groups are the primary control and I'd reference other security groups as sources rather than CIDR blocks. NACLs I'd use as a coarse second layer, typically to blanket-block a specific IP range.

**Q: An EC2 instance in a private subnet can't reach the internet to download packages. Why?**
> Almost always a missing NAT path. A private subnet by definition has no internet gateway route, so outbound traffic needs a NAT gateway sitting in a public subnet with an Elastic IP, plus a `0.0.0.0/0` route in the private subnet's route table pointing at that NAT gateway. The three failure modes I'd check in order are: no NAT gateway at all, a NAT gateway that was accidentally placed in the private subnet so it has no path out itself, or the route table missing the NAT entry. Worth adding that if the traffic is only going to S3 or DynamoDB, a VPC gateway endpoint is the better answer than NAT — it's free and keeps the traffic off the internet entirely, whereas NAT charges per gigabyte processed.

**Q: How many usable IP addresses in a 10.0.1.0/24 subnet?**
> 251. A /24 gives 256 addresses, and AWS reserves five in every subnet — the network address, the VPC router, the DNS resolver, one held for future use, and the broadcast address. The Linux answer would be 254 because you'd only lose network and broadcast, so this is a spot where AWS differs from general networking knowledge.

**Q: You can SSH to one instance but not another in the same subnet. Where do you look?**
> Same subnet means routing and NACLs are identical, so the difference has to be instance-level. I'd check the security groups first since they're per-instance, then whether the second instance actually has a public IP, then whether sshd is running on it. The fastest signal is the failure mode — if it times out, traffic is being dropped, so it's the security group. If it's connection refused, the packet reached the host and nothing is listening on 22, so it's sshd or the instance failed to boot properly. If I still couldn't tell, VPC Flow Logs would show ACCEPT or REJECT for that flow and settle it definitively.

---

## Gotchas & flags

- Subnets **cannot span AZs.** Always set `availability_zone` explicitly or AWS picks and your "multi-AZ" design may not be.
- `map_public_ip_on_launch` ≠ public subnet. The route table association is what matters.
- VPC CIDR **cannot be changed after creation** (you can add secondary CIDRs, but not resize the primary). Plan the address space up front.
- Overlapping CIDRs permanently block peering between those VPCs.
- NAT gateway is **per-AZ for HA** — one NAT is a single point of failure and a cross-AZ data charge.
- NACLs being stateless is where debugging goes wrong — the ephemeral port range 1024–65535 must be allowed outbound for inbound connections to work.
- Public IPv4 addresses are now **billed**, including EIPs in use. Don't leave them lying around.

---

Next: [[14-Project-S3-Static-Website]]

<!-- related -->
---
**Related:** [[AWS-MOC|AWS]]
<!-- /related -->
