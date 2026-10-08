---
title: 07 — count vs for_each
type: concept
module: terraform
status: complete
interview-relevance: ★★★★★
tags:
  - terraform
  - count
  - for_each
  - meta-arguments
config: tf-multi-resources/
up: "[[Terraform-Mastery-MOC]]"
prev: "[[06-Data-Sources]]"
next: "[[08-State-Management-and-Remote-Backend]]"
origin: Terraform-Mastery/07-Count-vs-ForEach.md
---

# 07 — count vs for_each

← Back to [[Terraform-Mastery-MOC]]

> **This is one of the highest-yield Terraform interview topics.** Not because it's hard, but because the `count` shifting problem separates people who've run Terraform in production from people who've only read about it.

---

## Concept

Both are **meta-arguments** — they work on any resource, data source, or module block, regardless of provider. They create multiple instances from a single block.

| | `count` | `for_each` |
|---|---|---|
| Input | a number | a **map** or **set of strings** |
| Instance address | `aws_subnet.main[0]` | `aws_subnet.main["public"]` |
| Loop variable | `count.index` | `each.key`, `each.value` |
| Identity is based on | **position** | **key** |
| Removing a middle item | **shifts everything after it → mass recreation** | only that one is destroyed |

> **Memory hook:** `count` numbers things like a queue — pull someone out of the middle and everyone behind shuffles forward. `for_each` names things like a hotel guest list — one checkout doesn't renumber the other rooms.

---

## Config built — `tf-multi-resources/`

```hcl
locals {
  project = "project-01"
}

resource "aws_vpc" "my-vpc" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "${local.project}-vpc" }
}

# COUNT — two subnets, CIDR derived from the index
resource "aws_subnet" "main" {
  vpc_id     = aws_vpc.my-vpc.id
  cidr_block = "10.0.${count.index}.0/24"     # 10.0.0.0/24 and 10.0.1.0/24
  count      = 2
  tags = { Name = "${local.project}-subnet-${count.index}" }
}

# FOR_EACH — one EC2 per map entry
resource "aws_instance" "main" {
  for_each = var.ec2_map

  ami           = each.value.ami
  instance_type = each.value.instance_type

  subnet_id = element(
    aws_subnet.main[*].id,
    index(keys(var.ec2_map), each.key) % length(aws_subnet.main)
  )

  tags = { Name = "${local.project}-instance-${each.key}" }
}

output "aws_subnet_id" {
  value = aws_subnet.main[1].id     # index access — count-created resources are a list
}
```

**`variables.tf`**
```hcl
variable "ec2_map" {
  type = map(object({
    ami           = string
    instance_type = string
  }))
}
```

**`terraform.tfvars`**
```hcl
ec2_map = {
  "ubuntu" = {
    ami           = "ami-0705384c0b33c194c"
    instance_type = "t3.micro"
  },
  "amazon-linux" = {
    ami           = "ami-052387465d846f3fc"
    instance_type = "t3.micro"
  }
}
```

Resulting addresses:
```
aws_subnet.main[0]                      aws_subnet.main[1]
aws_instance.main["ubuntu"]             aws_instance.main["amazon-linux"]
```

**Unpacking the subnet distribution expression** — this is the trickiest line in the whole course:
```hcl
element(aws_subnet.main[*].id, index(keys(var.ec2_map), each.key) % length(aws_subnet.main))
```
1. `aws_subnet.main[*].id` — splat, gives `["subnet-aaa", "subnet-bbb"]`
2. `keys(var.ec2_map)` — `["amazon-linux", "ubuntu"]` (**lexicographic**, not the tfvars order)
3. `index(..., each.key)` — position of this instance's key in that list
4. `% length(...)` — modulo, so it wraps if there are more instances than subnets
5. `element(...)` — pick that subnet

Net effect: round-robin instances across subnets. **This is clever but fragile** — it depends on map key ordering, which is lexicographic and not obvious. In real code, put the subnet assignment in the map itself:
```hcl
ec2_map = {
  "ubuntu"       = { ami = "...", instance_type = "t3.micro", subnet_key = "public-1" }
  "amazon-linux" = { ami = "...", instance_type = "t3.micro", subnet_key = "public-2" }
}
```
Explicit beats clever. Worth saying in an interview if shown code like the above.

---

## The `count` shifting problem — the core interview point

```hcl
variable "names" {
  default = ["alpha", "bravo", "charlie"]
}

resource "aws_instance" "srv" {
  count = length(var.names)
  tags  = { Name = var.names[count.index] }
}
```

State:
```
aws_instance.srv[0] → alpha
aws_instance.srv[1] → bravo
aws_instance.srv[2] → charlie
```

Now remove `"bravo"` from the middle of the list:
```
Desired:  [0]=alpha  [1]=charlie
State:    [0]=alpha  [1]=bravo    [2]=charlie
```

Terraform's plan: `[1]` changes from bravo to charlie → **update or replace**. `[2]` no longer exists → **destroy**. You asked to remove one server and Terraform modifies two.

With `for_each` over a set:
```hcl
resource "aws_instance" "srv" {
  for_each = toset(var.names)
  tags     = { Name = each.key }
}
```
State addresses are `srv["alpha"]`, `srv["bravo"]`, `srv["charlie"]`. Remove bravo → Terraform destroys `srv["bravo"]` and touches nothing else. **Correct blast radius.**

> **This is the answer to "count vs for_each, which do you use?" — for_each by default, count only for identical, order-irrelevant resources or as an on/off switch.**

---

## When `count` is still the right choice

**1. Conditional creation (the on/off toggle)** — this is `count`'s killer feature and `for_each` has no equivalent:
```hcl
resource "aws_internet_gateway" "main" {
  count  = length(local.public_subnet) > 0 ? 1 : 0
  vpc_id = aws_vpc.main.id
}
```
From the custom VPC module — create an IGW only if at least one subnet is public. Referenced afterwards as `aws_internet_gateway.main[0].id`.

```hcl
resource "aws_eip" "nat" {
  count = var.enable_nat_gateway ? 1 : 0
}
```

**2. N genuinely identical resources** where position carries no meaning and the count only ever grows.

Everything else: `for_each`.

---

## for_each input requirements

`for_each` accepts **a map** or **a set of strings**. Not a list.

```hcl
for_each = toset(["a", "b", "c"])     # list → set conversion
for_each = var.my_map                 # map, natively
for_each = { for u in local.users : u.name => u }   # build a map with a for expression
```

**Why keys must be known at plan time:** Terraform needs resource addresses before making any API calls. If a key derives from an attribute that's `(known after apply)`, plan fails with *"The for_each value depends on resource attributes that cannot be determined until apply."*

**Fix:** derive keys from variables/locals (statically known), not from resource outputs. Common workaround is a two-stage apply with `-target`, but the real fix is restructuring so the keys come from input data.

---

## Referencing instances

```hcl
# count-created — behaves like a list
aws_subnet.main[0].id
aws_subnet.main[*].id          # splat → list of all IDs
length(aws_subnet.main)

# for_each-created — behaves like a map
aws_instance.main["ubuntu"].id
values(aws_instance.main)[*].id     # all IDs
keys(aws_instance.main)             # all keys
[for k, v in aws_instance.main : v.private_ip]
```

---

## `for_each` on modules

Works identically on module blocks — this is how you stamp out N environments or N identical stacks:
```hcl
module "app" {
  for_each = var.environments
  source   = "./modules/app"

  env       = each.key
  instance  = each.value.instance_type
}
# → module.app["dev"], module.app["prod"]
```

---

## Real-world mapping

| Pattern | Job reality |
|---|---|
| `for_each` over a map of subnets | Standard VPC module — public/private subnets keyed by name, not index |
| `count = condition ? 1 : 0` | Optional resources: NAT gateway only in prod, bastion only when a flag is set |
| `for_each` over IAM users | User provisioning from a config file → [[16-Project-IAM-YAML-Automation]] |
| `for_each` on modules | Multi-environment or multi-tenant stamping from one module |
| `count` on subnets with `cidrsubnet(...)` | Acceptable — subnets are positionally identical and the AZ list is stable |

**Why this matters to me specifically:** `terraform-fullstack` uses reusable modules for VPC/ECS/ALB. Any place those modules create N of something, `for_each` keyed by name is the version that won't cause a mass-recreate incident when a subnet is removed from the middle of a list.

---

## Interview Q&A

**Q: What's the difference between count and for_each, and which do you prefer?**
> `count` takes a number and creates instances indexed by position — `resource[0]`, `resource[1]`. `for_each` takes a map or a set and creates instances keyed by a stable string — `resource["public"]`, `resource["private"]`. I default to `for_each`, and the reason is blast radius. With `count`, if I remove an item from the middle of the list, everything after it shifts index, so Terraform sees every subsequent resource as changed and recreates them. With `for_each`, removing an item destroys exactly that one instance and leaves the rest untouched, because identity is the key, not the position.

**Q: So is there any case where you'd still use count?**
> Yes, two. The main one is conditional creation — `count = var.enable_nat ? 1 : 0` is the idiomatic way to make a resource optional, and `for_each` has no equivalent for that. The other is when you genuinely need N identical resources where position has no meaning and the list only grows at the end. But anything with a natural name or key — subnets, environments, users, buckets — gets `for_each`.

**Q: You have a for_each whose keys come from another resource's output and the plan fails. What's happening?**
> Terraform needs the full set of resource addresses before it can plan, and a `for_each` key that derives from an attribute that's only known after apply can't be resolved yet, so it errors with "the for_each value depends on resource attributes that cannot be determined until apply." The workaround people reach for is a targeted two-stage apply, but the real fix is restructuring so the keys come from static input — variables, locals, or a config file — rather than from resource outputs. The keys should be data you already have, not something the cloud hands back.

**Q: You changed a variable and the plan shows five instances being destroyed and recreated when you only meant to remove one. What went wrong?**
> That's the classic count index-shift. The resources were created with `count` over a list, and removing an element from the middle shifted every subsequent index, so Terraform sees each one as a different resource now. The immediate fix if I've already applied is `terraform state mv` to relocate the state entries to their correct new indices before applying. The permanent fix is migrating that resource to `for_each` keyed by something stable — which itself is a state-move exercise, so it's worth doing before the resource set grows.

**Q: What types can for_each accept?**
> A map, or a set of strings. Not a list — if I have a list I convert it with `toset()`, which also drops duplicates. In practice I usually build a map explicitly with a for expression, because that gives me control over what the keys are, and the keys are what determine resource identity in state. Picking good keys is the actual design decision; the mechanics are trivial.

---

## Gotchas & flags

- `for_each` and `count` **cannot both be used on the same block.**
- `toset()` on a list with duplicates silently collapses them — if your list was meant to have duplicates, `for_each` is the wrong tool.
- `keys()` on a map returns **lexicographic** order, not the order written in tfvars. The course's round-robin subnet expression depends on this and it's non-obvious.
- Migrating `count` → `for_each` on existing resources requires `terraform state mv` for every instance, or Terraform will destroy and recreate everything → [[09-State-Manipulation-and-Import]]
- A `count = 1` resource is still referenced as `resource.name[0]`, never `resource.name`. Forgetting the `[0]` is a very common error.

---

Next: [[08-State-Management-and-Remote-Backend]]
