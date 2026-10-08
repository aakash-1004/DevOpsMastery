---
title: "11 — Dependencies, Lifecycle & Validations"
type: concept
module: terraform
status: complete
interview-relevance: ★★★★☆
tags:
  - terraform
  - lifecycle
  - depends_on
  - validation
  - preconditions
config: tf-test/
up: "[[Terraform-Mastery-MOC]]"
prev: "[[10-Modules]]"
next: "[[12-Workspaces]]"
origin: Terraform-Mastery/11-Dependencies-Lifecycle-Validations.md
---

# 11 — Dependencies, Lifecycle & Validations

← Back to [[Terraform-Mastery-MOC]]

---

## Config built — `tf-test/`

```hcl
resource "aws_security_group" "main" {
  name = "my-sg"
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "my-vpc" }
}

resource "aws_subnet" "private" {
  vpc_id     = aws_vpc.main.id      # implicit dependency
  cidr_block = "10.0.0.0/24"
}

resource "aws_instance" "myserver" {
  ami                         = "ami-07c8c1b18ca66bb07"
  instance_type               = "t3.micro"
  subnet_id                   = aws_subnet.private.id
  associate_public_ip_address = false
  depends_on                  = [aws_security_group.main]    # explicit dependency

  lifecycle {
    #create_before_destroy = true
    #prevent_destroy       = true
    #replace_triggered_by  = [aws_security_group.main, aws_security_group.main.ingress]

    precondition {
      condition     = aws_security_group.main.id != ""
      error_message = "Security Group ID must not be blank"
    }

    postcondition {
      condition     = self.public_ip != ""
      error_message = "Public IP is not present."
    }
  }
}
```

> **Note the deliberate failure in this config:** `associate_public_ip_address = false` in a private subnet, with a postcondition asserting `self.public_ip != ""`. That postcondition **will fail** — which is exactly the point of the exercise. It demonstrates that postconditions catch bad outcomes after creation.

---

## Dependencies

### Implicit — the normal case

```hcl
resource "aws_subnet" "private" {
  vpc_id = aws_vpc.main.id      # this reference IS the dependency
}
```

Terraform builds a **DAG (directed acyclic graph)** from every reference between resources. It derives the creation order automatically and parallelises anything independent (default 10 concurrent operations, tunable with `-parallelism=n`).

**This is why Terraform is declarative** — you never write ordering, it's inferred from data flow.

```bash
terraform graph | dot -Tpng > graph.png   # visualise the DAG
```

### Explicit — `depends_on`

For dependencies Terraform **cannot see** because there's no data reference between the resources:

```hcl
resource "aws_instance" "myserver" {
  depends_on = [aws_security_group.main]
}
```

**Legitimate uses:**
- An IAM role policy must exist before a resource that uses it at runtime (the reference is inside a JSON policy string Terraform can't parse as a dependency)
- An S3 bucket policy must be in place before objects are uploaded
- An application on EC2 needs a NAT gateway route to exist before user_data runs — the instance doesn't reference the NAT gateway, but it needs it

**When it's a smell:** if you're adding `depends_on` because something "just doesn't work otherwise" and you can't name the hidden dependency, you probably have a missing reference. Restructure to create a real reference instead — `depends_on` is a blunt instrument that also forces the dependency on *every* attribute, making plans more conservative than needed.

> **Interview answer shape:** "Implicit is the default and almost always right. `depends_on` is for dependencies that exist in the real world but aren't visible in the configuration as a data reference."

`depends_on` also works on modules:
```hcl
module "app" {
  source     = "./modules/app"
  depends_on = [module.network]
}
```

---

## Lifecycle block

### `create_before_destroy`

Default behaviour for a replacement is destroy-then-create → **downtime**. This inverts it:

```hcl
lifecycle {
  create_before_destroy = true
}
```

**Requirement:** the resource must tolerate two existing simultaneously. Anything with a unique name (a security group name, an IAM role name, an S3 bucket) will **collide**. Fix with `name_prefix` instead of `name`, letting AWS generate unique names.

Real use: replacing a launch configuration or an instance behind a load balancer without a gap in service.

### `prevent_destroy`

```hcl
lifecycle {
  prevent_destroy = true
}
```

Any plan that would destroy this resource **errors out**. Use on: RDS instances, S3 buckets with data, DynamoDB tables, anything stateful.

**Limits worth knowing:** it blocks `terraform destroy` and destructive plans, but it does not stop someone deleting the resource in the console, and removing the resource block from the config entirely bypasses it. It's a guardrail against accident, not a security control.

### `ignore_changes`

```hcl
lifecycle {
  ignore_changes = [tags, ami]
}

lifecycle {
  ignore_changes = all
}
```

Terraform stops caring about drift on those attributes.

**Real example from the IAM project:**
```hcl
resource "aws_iam_user_login_profile" "profile" {
  for_each        = aws_iam_user.users
  user            = each.value.name
  password_length = 12

  lifecycle {
    ignore_changes = [password_length, password_reset_required, pgp_key]
  }
}
```
Without this, every plan shows a diff because the generated password state doesn't round-trip cleanly.

**Legitimate uses:** an ASG's `desired_count` adjusted by autoscaling, tags written by an external tagging system, an ECS task definition image tag updated by a separate deploy pipeline.

**The trap:** `ignore_changes` makes drift invisible. Use it surgically on specific attributes; `ignore_changes = all` means the resource is effectively unmanaged and you've turned Terraform into a very expensive resource creator.

### `replace_triggered_by` (Terraform 1.2+)

```hcl
lifecycle {
  replace_triggered_by = [
    aws_security_group.main,              # replace me if the whole SG is replaced
    aws_security_group.main.ingress       # or if just this attribute changes
  ]
}
```

Forces replacement of this resource when a referenced resource or attribute changes. Use when an instance must be rebuilt because its dependency changed — e.g. rebuild instances when a launch template version changes.

---

## Preconditions & postconditions (Terraform 1.2+)

Both live in `lifecycle`. The distinction is **timing**:

| | Runs | Checks |
|---|---|---|
| `precondition` | **before** the resource is created/updated | Assumptions that must hold going in |
| `postcondition` | **after** creation | The outcome is what you expected |

```hcl
lifecycle {
  precondition {
    condition     = aws_security_group.main.id != ""
    error_message = "Security Group ID must not be blank"
  }

  postcondition {
    condition     = self.public_ip != ""
    error_message = "Public IP is not present."
  }
}
```

**`self` is only valid inside postconditions** — it refers to the resource being created, whose attributes don't exist yet at precondition time.

Practical examples:
```hcl
precondition {
  condition     = data.aws_ami.selected.architecture == "x86_64"
  error_message = "AMI must be x86_64 to match the chosen instance type."
}

postcondition {
  condition     = self.instance_state == "running"
  error_message = "Instance did not reach running state."
}
```

**Precondition vs variable validation — the interview distinction:**

| | `variable` validation | `precondition` |
|---|---|---|
| Scope | One variable, in isolation | Any expression, including other resources and data sources |
| Timing | Very early in plan | At resource evaluation |
| Use for | "Is this input well-formed?" | "Is the world in a state where creating this makes sense?" |

Validation can't reference another resource. Preconditions can. That's the dividing line.

---

## `check` blocks (Terraform 1.5+)

```hcl
check "ec2_instance_validation" {
  description = "Ensure EC2 instance is using an approved AMI and instance type."

  assert {
    condition     = var.ami_id != ""
    error_message = "AMI ID must not be empty."
  }

  assert {
    condition     = contains(var.production_instance_type, var.instance_type)
    error_message = "Instance type must be one of the approved types for production: ${join(", ", var.production_instance_type)}."
  }
}
```

**The key difference from preconditions: `check` blocks produce a WARNING, not an error.** A failed check does not stop the apply.

That's deliberate. Checks are for **continuous validation** — assertions you want reported on every plan without blocking deploys. A precondition failing should stop you; a check failing should tell you. `check` blocks can also contain a scoped `data` block to verify something live, like whether an endpoint responds after deployment.

> **Memory hook for all three:** validation = "is this input sane?" (blocks). precondition = "is the world ready?" (blocks). postcondition = "did it come out right?" (blocks). check = "is anything off?" (warns).

---

## Real-world mapping

| Feature | Job reality |
|---|---|
| `prevent_destroy` | Mandatory on production RDS, state buckets, anything with data |
| `ignore_changes` | ASG desired_count, ECS image tags, externally-managed tags |
| `create_before_destroy` | Zero-downtime instance/launch-template replacement |
| `depends_on` | IAM policy propagation, NAT route availability before user_data |
| `precondition` | Guardrails: AMI architecture matches instance type, subnet is in the right AZ |
| `check` | Continuous validation dashboards — assertions surfaced without blocking |

---

## Interview Q&A

**Q: How does Terraform know what order to create resources in?**
> It builds a dependency graph from the references between resources. When a subnet references `aws_vpc.main.id`, that reference is the dependency — Terraform knows the VPC has to exist first, and it derives the whole ordering from those data flows. Anything independent runs in parallel, ten operations at a time by default. That's what makes it declarative: I never write the ordering, it falls out of the data. For dependencies that exist in reality but aren't visible as a reference, there's `depends_on` as the explicit escape hatch.

**Q: When would you use `depends_on`?**
> When the dependency is real but invisible to Terraform. The usual case is IAM — a resource needs a role policy to be attached before it works at runtime, but the policy ARN is buried in a JSON string so there's no reference Terraform can follow. Similarly, an EC2 instance whose user_data needs internet access depends on the NAT gateway route existing, even though the instance never references the NAT gateway. I'm cautious with it though — if I'm adding `depends_on` because something "just doesn't work otherwise" and I can't name the hidden dependency, that usually means I'm missing a reference that should exist, and restructuring to create a real reference is better than papering over it.

**Q: Explain `create_before_destroy` and when it breaks.**
> By default, when Terraform has to replace a resource it destroys the old one first, which means a gap in service. `create_before_destroy` inverts that — new one up, then old one down — which is how you replace an instance behind a load balancer without downtime. Where it breaks is any resource with a unique name constraint: two security groups or IAM roles with the same name can't coexist, so the create half fails. The fix is using `name_prefix` instead of `name` so AWS generates a unique suffix. It's also worth knowing that `create_before_destroy` propagates to dependencies, which can pull more resources into the replacement than you expected.

**Q: What's the difference between variable validation and a precondition?**
> Scope and what they can see. Variable validation checks one variable in isolation and can only reference that variable — it's for "is this input well-formed", like whether a CIDR parses or an instance type is on the approved list. A precondition lives in a resource's lifecycle block and can reference anything: other resources, data sources, the whole configuration. So it's for "is the world in a state where creating this makes sense" — for example asserting that the AMI I looked up is the right architecture for the instance type I'm using. Validation is earlier and narrower; preconditions are later and can see everything.

**Q: What's the difference between a precondition and a `check` block?**
> A failing precondition is an error that stops the apply. A failing check is a warning that's reported but doesn't block anything. That's deliberate — checks are for continuous validation, assertions you want surfaced on every plan without gating deploys, and they can include a scoped data block to verify something live like whether an endpoint actually responds after deployment. So preconditions are for conditions where proceeding would be wrong, and checks are for conditions you want visibility into.

**Q: How do you stop someone accidentally destroying your production database?**
> `prevent_destroy` in the lifecycle block — any plan that would destroy that resource errors out. But I'd be honest about its limits in an interview: it only protects against Terraform-initiated destruction, it doesn't stop someone deleting the instance in the console, and it's bypassed entirely if the resource block is removed from the configuration. So it's an accident guardrail, not a security control. The real protections are IAM policies denying delete on production resources, deletion protection enabled on the RDS instance itself, and required PR review on the Terraform repository.

---

## Gotchas & flags

- `self` is valid **only** in postconditions, and only refers to the resource being created.
- `prevent_destroy` fails on `terraform destroy` too — you must remove the flag first, which is intentional friction.
- `ignore_changes = all` effectively unmanages the resource. Almost never correct.
- `create_before_destroy` propagates to dependencies and can widen the replacement set unexpectedly.
- `lifecycle` arguments must be **literal values** — no variables, no interpolation. You can't do `prevent_destroy = var.is_prod`.
- The course's `tf-test` postcondition is designed to fail (private subnet, no public IP). That's the lesson, not a bug.

---

Next: [[12-Workspaces]]
