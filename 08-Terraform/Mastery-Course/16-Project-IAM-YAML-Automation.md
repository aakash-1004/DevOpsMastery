---
title: "16 — Project: IAM Management from YAML"
type: project
module: terraform
status: complete
interview-relevance: ★★★★☆
tags:
  - terraform
  - iam
  - yaml
  - flatten
  - for_each
  - project
config: aws-IAM-management/
up: "[[Terraform-Mastery-MOC]]"
prev: "[[15-Project-VPC-EC2-Nginx]]"
next: "[[17-Terraform-Cloud-and-Best-Practices]]"
origin: Terraform-Mastery/16-Project-IAM-YAML-Automation.md
---

# 16 — Project: IAM Management from YAML

← Back to [[Terraform-Mastery-MOC]]

> **The most technically interesting config in the course.** It combines `yamldecode`, nested for expressions, `flatten`, `toset`, `for_each`, and a `lifecycle` block to solve a real many-to-many problem. This is the one to walk through when an interviewer asks for something non-trivial.

---

## The problem

Provision IAM users from a config file. Each user has **one or more** roles. Terraform needs one `aws_iam_user` per user and one `aws_iam_user_policy_attachment` per **user-role pair**.

That's a many-to-many relationship, and Terraform's `for_each` needs a flat map with unique keys. Getting from nested input to flat pairs is the whole exercise.

**Input — `users.yaml`:**
```yaml
users:
  - username: raju
    roles: [AmazonEC2FullAccess]
  - username: sham
    roles: [AmazonS3ReadOnlyAccess]
  - username: baburao
    roles: [AmazonS3ReadOnlyAccess, AmazonEC2FullAccess]
```

3 users → 3 user resources, but **4** policy attachments (baburao has two roles).

---

## The config

```hcl
locals {
  users_data = yamldecode(file("./users.yaml")).users

  user_role_pair = flatten([
    for user in local.users_data : [
      for role in user.roles : {
        username = user.username
        role     = role
      }
    ]
  ])
}

output "output" {
  value = local.user_role_pair
}

# Create the users
resource "aws_iam_user" "users" {
  for_each = toset(local.users_data[*].username)
  name     = each.value
}

# Generate console passwords
resource "aws_iam_user_login_profile" "profile" {
  for_each        = aws_iam_user.users
  user            = each.value.name
  password_length = 12

  lifecycle {
    ignore_changes = [
      password_length,
      password_reset_required,
      pgp_key,
    ]
  }
}

# Attach policies — one resource per user-role pair
resource "aws_iam_user_policy_attachment" "main" {
  for_each = {
    for pair in local.user_role_pair :
    "${pair.username}-${pair.role}" => pair
  }

  user       = aws_iam_user.users[each.value.username].name
  policy_arn = "arn:aws:iam::aws:policy/${each.value.role}"
}
```

---

## Breaking it down

### Step 1 — read external config
```hcl
local.users_data = yamldecode(file("./users.yaml")).users
```
- `file()` reads the file as a raw string
- `yamldecode()` parses it into an HCL data structure
- `.users` pulls the top-level key

Result:
```hcl
[
  { username = "raju",    roles = ["AmazonEC2FullAccess"] },
  { username = "sham",    roles = ["AmazonS3ReadOnlyAccess"] },
  { username = "baburao", roles = ["AmazonS3ReadOnlyAccess", "AmazonEC2FullAccess"] },
]
```

**Why this pattern is genuinely useful:** the YAML file can be edited by people who don't know Terraform. Onboarding a new engineer becomes a three-line YAML PR, not an HCL change. The config code never changes — only data does.

`jsondecode()` and `csvdecode()` work identically for other formats.

### Step 2 — nested for expression → list of lists
```hcl
[
  for user in local.users_data : [
    for role in user.roles : {
      username = user.username
      role     = role
    }
  ]
]
```
Produces:
```hcl
[
  [ {username="raju", role="AmazonEC2FullAccess"} ],
  [ {username="sham", role="AmazonS3ReadOnlyAccess"} ],
  [ {username="baburao", role="AmazonS3ReadOnlyAccess"},
    {username="baburao", role="AmazonEC2FullAccess"} ],
]
```
Nested — one inner list per user. Not usable by `for_each`.

### Step 3 — `flatten()` → flat list
```hcl
flatten([...])
```
```hcl
[
  {username="raju",    role="AmazonEC2FullAccess"},
  {username="sham",    role="AmazonS3ReadOnlyAccess"},
  {username="baburao", role="AmazonS3ReadOnlyAccess"},
  {username="baburao", role="AmazonEC2FullAccess"},
]
```
**4 pairs from 3 users.** This is the many-to-many expansion.

> **This flatten-a-nested-for-expression pattern is the canonical Terraform idiom for many-to-many relationships.** Recognising it — users→roles, subnets→route tables, repos→branch protections — is a strong signal.

### Step 4 — users via `toset` + splat
```hcl
for_each = toset(local.users_data[*].username)
name     = each.value
```
- `local.users_data[*].username` — splat, extracts `["raju","sham","baburao"]`
- `toset()` — `for_each` needs a set or map, not a list; also guarantees uniqueness
- With a set, `each.key == each.value`

Addresses: `aws_iam_user.users["raju"]`, `["sham"]`, `["baburao"]`

### Step 5 — `for_each` over another resource
```hcl
resource "aws_iam_user_login_profile" "profile" {
  for_each = aws_iam_user.users     # iterating a RESOURCE, not a variable
  user     = each.value.name
}
```
A `for_each`-created resource **is itself a map**, so you can iterate it directly. Keys carry over, and `each.value` is the full user object.

This also creates an implicit dependency — login profiles can't be created before their users.

### Step 6 — building a map with a composite key
```hcl
for_each = {
  for pair in local.user_role_pair :
  "${pair.username}-${pair.role}" => pair
}
```
`for_each` needs **unique keys**. Username alone collides (baburao appears twice). Role alone collides. `username-role` is unique.

Addresses:
```
aws_iam_user_policy_attachment.main["raju-AmazonEC2FullAccess"]
aws_iam_user_policy_attachment.main["baburao-AmazonS3ReadOnlyAccess"]
aws_iam_user_policy_attachment.main["baburao-AmazonEC2FullAccess"]
```

**Why the composite key matters beyond uniqueness:** these keys are the resource identity in state. Remove one role from one user's YAML list and exactly that one attachment is destroyed — nothing else shifts. With `count` this would be an index-shift disaster → [[07-Count-vs-ForEach]]

### Step 7 — building ARNs
```hcl
policy_arn = "arn:aws:iam::aws:policy/${each.value.role}"
```
AWS-managed policies use the literal account placeholder `aws` in the ARN. For **customer-managed** policies you need the real account ID:
```hcl
policy_arn = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:policy/${each.value.role}"
```

### Step 8 — `ignore_changes` on the login profile
```hcl
lifecycle {
  ignore_changes = [password_length, password_reset_required, pgp_key]
}
```
Without it, every plan shows a spurious diff because the generated password doesn't round-trip cleanly through state. → [[11-Dependencies-Lifecycle-Validations]]

---

## The security problem with this config

**`aws_iam_user_login_profile` stores the generated password in the state file in plaintext.** Anyone with read access to the state bucket has console credentials for every user created this way.

**The proper approaches:**

1. **PGP encryption** — supply `pgp_key`, the password is encrypted with the user's public key and only they can decrypt it
2. **Secrets Manager** — generate with `random_password`, write to Secrets Manager, hand the user a secret ARN
3. **Don't create console users at all** — federate through SSO/Identity Center. IAM users with passwords are legacy; modern AWS uses identity federation with short-lived credentials.

> **This is the highest-value thing to say about this project in an interview.** Being able to explain what you built *and* what's wrong with it security-wise is what distinguishes someone who understands from someone who followed a tutorial. Directly relevant given my own credential-hygiene history — the same failure mode, different surface.

Also worth noting: attaching policies **directly to users** is an anti-pattern. Production uses groups (`aws_iam_group_membership`) or, better, roles that users assume. Directly-attached user policies don't scale and are hard to audit.

---

## Real-world mapping

| Pattern | Job reality |
|---|---|
| YAML → Terraform | Team/user/permission management from a file non-engineers can PR |
| `flatten` nested for | Any many-to-many: users→roles, subnets→NACL rules, repos→branch rules |
| Composite `for_each` keys | Correct state identity for pair-type resources |
| `for_each` over a resource | Chaining dependent resources, keys carried through |
| `ignore_changes` on generated secrets | Any resource where the provider can't round-trip a value cleanly |

**Honest positioning:** the *technique* here is production-grade. The *security model* is not. Frame it that way.

---

## Interview Q&A

**Q: Tell me about something non-trivial you've built with Terraform.**
> I built an IAM user management system driven by a YAML file. The file lists users and the roles each one should have, and Terraform reads it with `yamldecode` and creates the users and policy attachments. The interesting part is that it's a many-to-many relationship — one user can have several roles — and Terraform's `for_each` needs a flat map with unique keys. So I used a nested for expression to produce one entry per user-role pair, wrapped in `flatten` to collapse the nesting, then built a map keyed on username-plus-role so every pair has a unique stable key. The value of that structure is blast radius: removing one role from one user's YAML destroys exactly that one attachment and nothing else shifts. The other reason I like the pattern is that the YAML is editable by people who don't know Terraform — onboarding becomes a three-line data PR rather than a code change.

**Q: Why did you need `flatten()` there?**
> Because a nested for expression produces a list of lists — one inner list per user, containing that user's role pairs. `for_each` can't consume that; it needs a flat collection. `flatten` collapses one level of nesting, turning it into a single flat list of pair objects, which I then convert into a map. It's the standard Terraform idiom for expressing a many-to-many relationship as individual resources, and the same shape shows up in plenty of other places — subnets to route table associations, repositories to branch protection rules.

**Q: Why a composite key instead of just the username?**
> Because `for_each` keys have to be unique, and a user with two roles would collide on username alone. But the more important reason is that the key is the resource's identity in state. Using username-plus-role means each attachment has a stable address that doesn't depend on ordering, so when someone removes a role from the YAML, Terraform destroys exactly that one attachment. If I'd used `count` over the flattened list instead, removing an item from the middle would shift every subsequent index and Terraform would recreate attachments that didn't need to change.

**Q: What's wrong with this from a security standpoint?**
> Two things. First, `aws_iam_user_login_profile` puts the generated password into the state file in plaintext, so anyone with read access to the state bucket effectively has console credentials for every user. The fixes are to supply a PGP key so the password is encrypted to the user, or generate it with `random_password` and write it to Secrets Manager, or ideally not create console users at all and federate through Identity Center with short-lived credentials instead. Second, attaching policies directly to users doesn't scale and is hard to audit — production should use groups, or better, roles that users assume. So the technique in this config is sound but the security model is a teaching simplification, not something I'd ship.

**Q: How would you extend this to handle groups instead of direct attachments?**
> The YAML would carry group membership rather than role names — each user lists the groups they belong to. Then I'd have `aws_iam_group` resources for the groups, `aws_iam_group_policy_attachment` to attach policies to groups once rather than per user, and `aws_iam_user_group_membership` for the user-to-group mapping. The flatten pattern still applies for the many-to-many part, just on user-to-group pairs instead of user-to-role. The win is that permissions are defined once per group and auditing becomes "what can this group do" rather than enumerating every user's attachments.

---

## Gotchas & flags

- `for_each` requires **a set or a map**. A list from `yamldecode` must go through `toset()` or a for expression.
- `for_each` keys must be **unique and known at plan time**. Derived from YAML data they're static, which is why this works.
- `aws_iam_user_login_profile` writes the password to state in plaintext.
- The `ignore_changes` block is not optional here — without it every plan shows a spurious diff.
- AWS-managed policy ARNs use the literal `aws` account placeholder; customer-managed ones need the real account ID from `data.aws_caller_identity`.
- `file()` paths are relative to the **working directory**, not the module. Inside a module use `"${path.module}/users.yaml"`.

---

Next: [[17-Terraform-Cloud-and-Best-Practices]]
