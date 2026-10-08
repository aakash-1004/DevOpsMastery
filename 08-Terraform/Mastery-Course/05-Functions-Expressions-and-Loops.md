---
title: "05 — Functions, Expressions & Loops"
type: concept
module: terraform
status: complete
interview-relevance: ★★★☆☆
tags:
  - terraform
  - functions
  - expressions
  - loops
  - hcl
config: "tf-functions/, tf-operators-exps/"
up: "[[Terraform-Mastery-MOC]]"
prev: "[[04-Variables-Deep-Dive]]"
next: "[[06-Data-Sources]]"
origin: Terraform-Mastery/05-Functions-Expressions-and-Loops.md
---

# 05 — Functions, Expressions & Loops

← Back to [[Terraform-Mastery-MOC]]

---

## Concept

Terraform ships built-in functions callable inside any expression. **You cannot define your own functions** — that's a hard limitation and a genuine interview question. If the built-ins don't cover it, you either restructure the data or drop to an external data source.

Test functions interactively without touching AWS:
```bash
terraform console
> max(5, 12, 9)
12
> upper("hello")
"HELLO"
```
`terraform console` is the fastest way to debug an expression. Underused.

---

## Config built — `tf-functions/`

```hcl
terraform {}

locals {
  value = "Hello World!"
}

variable "string_list" {
  type    = list(string)
  default = ["serv1", "serv2", "serv3", "serv1"]
}

output "output" {
  #value = upper(local.value)
  #value = startswith(local.value, "hello")
  #value = split(" ", local.value)
  #value = min(1, 2, 3, 4, 5)
  #value = abs(-15.123)
  #value = length(var.string_list)
  #value = join(":", var.string_list)
  #value = contains(var.string_list, "serv1")
  #value = toset(var.string_list)   # removes duplicates
}
```

Note `terraform {}` empty and no provider — **function experiments need no cloud provider at all.** Good pattern for scratch testing.

---

## Function catalogue by category

### String
| Function | Example | Result |
|---|---|---|
| `upper` / `lower` | `lower("ABC")` | `"abc"` |
| `trimspace` | `trimspace("  x  ")` | `"x"` |
| `split` | `split("-", "a-b-c")` | `["a","b","c"]` |
| `join` | `join("-", ["a","b"])` | `"a-b"` |
| `startswith` / `endswith` | `startswith("hello","he")` | `true` |
| `replace` | `replace("a-b","-","_")` | `"a_b"` |
| `format` | `format("%s-%d","web",1)` | `"web-1"` |
| `substr` | `substr("hello",0,2)` | `"he"` |

### Collection
| Function | Purpose |
|---|---|
| `length(x)` | count of list/map/string |
| `contains(list, val)` | membership test — **use in validation blocks** |
| `merge(map1, map2)` | combine maps, later wins on collision |
| `concat(list1, list2)` | join lists |
| `element(list, idx)` | index access that **wraps around** past the end |
| `index(list, val)` | position of a value |
| `keys(map)` / `values(map)` | extract keys or values |
| `lookup(map, key, default)` | safe map access with fallback |
| `flatten(list_of_lists)` | collapse nesting one level — **critical for the IAM project** |
| `distinct(list)` | remove duplicates, preserve order |
| `slice(list, from, to)` | sublist |
| `alltrue` / `anytrue` | boolean reduce over a list — **used in validation** |

### Type conversion
| Function | Purpose |
|---|---|
| `toset(list)` | list → set, **removes duplicates** |
| `tolist(set)` | set → list |
| `tomap`, `tostring`, `tonumber`, `tobool` | explicit casts |

### Numeric
`max`, `min`, `abs`, `ceil`, `floor`, `parseint`

### Encoding
| Function | Purpose |
|---|---|
| `jsonencode(obj)` | HCL object → JSON string — **how IAM policies are written** |
| `jsondecode(str)` | JSON string → HCL object |
| `yamldecode(str)` | YAML string → HCL object — **the IAM project's core trick** |
| `base64encode` / `base64decode` | e.g. for `user_data_base64` |

### Filesystem
| Function | Purpose |
|---|---|
| `file(path)` | read file contents as string |
| `templatefile(path, vars)` | read + interpolate variables — **the right way to do user_data** |
| `fileexists(path)` | guard before reading |

### Network
| Function | Purpose |
|---|---|
| `cidrsubnet(prefix, newbits, netnum)` | carve subnets out of a VPC CIDR programmatically |
| `cidrhost(prefix, hostnum)` | nth IP in a range |
| `cidrnetmask(prefix)` | netmask — **used with `can()` for CIDR validation** |

`cidrsubnet` is the one to actually remember:
```hcl
cidrsubnet("10.0.0.0/16", 8, 0)  # → "10.0.0.0/24"
cidrsubnet("10.0.0.0/16", 8, 1)  # → "10.0.1.0/24"
cidrsubnet("10.0.0.0/16", 8, 2)  # → "10.0.2.0/24"
```
Combined with `count.index` or a for expression, this generates an arbitrary number of subnets with zero hardcoded CIDRs. That's the "senior" way to write a VPC module.

### Special
| Function | Purpose |
|---|---|
| `can(expr)` | returns bool instead of erroring — **validation workhorse** |
| `try(a, b, c)` | first expression that doesn't error |
| `coalesce(a, b)` | first non-null/non-empty |

---

## Config built — `tf-operators-exps/`

```hcl
variable "num_list" {
  type    = list(number)
  default = [1, 2, 3, 4, 5]
}

variable "person_list" {
  type = list(object({
    fname = string
    lname = string
  }))
  default = [
    { fname = "Raju", lname = "Rastogi" },
    { fname = "Sham", lname = "Paul" }
  ]
}

variable "map_list" {
  type    = map(number)
  default = { "one" = 1, "two" = 2, "three" = 3 }
}

locals {
  mul = 2 * 8       # 16
  add = 2 + 2       # 4
  eq  = 2 != 3      # true

  # list → list
  double = [for num in var.num_list : num * 2]              # [2,4,6,8,10]

  # list → list with filter
  odd = [for num in var.num_list : num if num % 2 != 0]     # [1,3,5]

  # list of objects → list of one attribute
  fname_list = [for person in var.person_list : person.lname]  # ["Rastogi","Paul"]

  # map → list
  map_info = [for key, value in var.map_list : value * 5]   # [1*5, 3*5, 2*5] — key order!

  # map → map
  double_map = { for key, value in var.map_list : key => value * 2 }
}
```

---

## `for` expression syntax — the shape that matters

**Producing a list** — square brackets, colon:
```hcl
[for item in collection : <expression>]
[for item in collection : <expression> if <condition>]
```

**Producing a map** — curly braces, `key => value`:
```hcl
{ for key, value in collection : key => <expression> }
```

> **Memory hook:** brackets in, brackets out. `[...]` produces a list, `{...}` with `=>` produces a map. The arrow is what makes it a map — without `=>` inside braces it's a syntax error.

**Iterating a map gives two loop variables**: `for key, value in map`. Iterating a list gives one (`for item in list`), or optionally two where the first is the index (`for idx, item in list`).

**Splat expression** — shorthand for the common "extract one attribute" case:
```hcl
var.person_list[*].fname          # same as [for p in var.person_list : p.fname]
aws_subnet.main[*].id             # all subnet IDs from a count-created resource
local.users_data[*].username      # used in the IAM project
```

---

## Real-world usage of these

| Function | Where it earns its keep |
|---|---|
| `merge()` | Combining module-level default tags with caller-supplied tags — every serious module does this |
| `jsonencode()` | IAM policies, ECS task definitions, anything AWS wants as a JSON document |
| `yamldecode() + file()` | Driving Terraform from a config file non-Terraform people can edit → [[16-Project-IAM-YAML-Automation]] |
| `flatten()` | Turning a nested list-of-lists into the flat pairs `for_each` needs |
| `toset()` | Converting a list to a set so `for_each` accepts it |
| `cidrsubnet()` | Generating subnet CIDRs from a VPC CIDR without hardcoding |
| `templatefile()` | user_data scripts with injected values, instead of an unreadable heredoc |
| `can()` | Input validation without the expression blowing up |
| `lookup()` | Region-to-AMI maps and similar per-environment lookup tables |

---

## Interview Q&A

**Q: Can you write custom functions in Terraform?**
> No — Terraform only has built-in functions, and there's no user-defined function mechanism in HCL. If the built-ins don't cover what I need, the options are to restructure the data so a for expression handles it, use an external data source that shells out to a script, or do the computation upstream and feed the result in as a variable. In practice the combination of for expressions, `flatten`, `merge`, and the encoding functions covers almost everything.

**Q: What's the difference between a for expression producing a list versus a map?**
> Square brackets with a colon produce a list — `[for x in list : x * 2]`. Curly braces with a `key => value` arrow produce a map — `{for k, v in map : k => v * 2}`. Both support an `if` clause for filtering. The map form is what I reach for when building the input to `for_each`, because `for_each` needs a map or set with stable keys, and constructing that map explicitly gives me control over what those keys are.

**Q: What does `flatten()` do and when do you need it?**
> It collapses a list of lists into a single flat list, one level deep. The case where I needed it was IAM user management: the input was a list of users, each with a list of roles, and I needed one Terraform resource per user-role *pair*. A nested for expression produces a list of lists, and `flatten` turns that into a flat list of pair objects, which I can then convert into a map for `for_each`. Without it you can't express a many-to-many relationship as individual resources.

**Q: How do you generate subnet CIDRs without hardcoding them?**
> `cidrsubnet()`. Given a VPC CIDR like 10.0.0.0/16, `cidrsubnet("10.0.0.0/16", 8, count.index)` carves out sequential /24s — 10.0.0.0/24, 10.0.1.0/24, and so on. The second argument is how many bits to add to the prefix, the third is which subnet in the sequence. Combined with count or a for expression over availability zones, it means the module caller supplies only the VPC CIDR and the number of subnets, and everything else is derived. Hardcoded CIDR lists are one of the things that make a module non-reusable.

---

## Gotchas & flags

- Map iteration order is **lexicographic by key**, not insertion order. `{one=1, two=2, three=3}` iterates as one, three, two. Don't assume ordering.
- `toset()` on a list of strings removes duplicates **and** discards order.
- `element(list, idx)` wraps around modulo the list length; `list[idx]` errors on out-of-range. `element` is the safe one when the index may exceed the list.
- `lookup(map, key)` without a default errors on a missing key. Always supply the third argument.
- The `tf-functions` config used `terraform {}` with no provider — a valid, useful pattern for pure-expression scratch work.

---

Next: [[06-Data-Sources]]
