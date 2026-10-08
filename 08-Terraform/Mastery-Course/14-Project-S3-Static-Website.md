---
title: "14 — Project: S3 Static Website"
type: project
module: terraform
status: complete
interview-relevance: ★★★☆☆
tags:
  - terraform
  - s3
  - project
  - aws
  - static-website
config: "aws-s3/, proj-static-website/"
up: "[[Terraform-Mastery-MOC]]"
prev: "[[13-AWS-Networking-VPC-Concepts]]"
next: "[[15-Project-VPC-EC2-Nginx]]"
origin: Terraform-Mastery/14-Project-S3-Static-Website.md
---

# 14 — Project: S3 Static Website

← Back to [[Terraform-Mastery-MOC]]

---

## Part 1 — S3 basics (`aws-s3/`)

```hcl
terraform {
  required_providers {
    aws    = { source = "hashicorp/aws",    version = "5.54.1" }
    random = { source = "hashicorp/random", version = "3.6.2"  }
  }
}

provider "aws" {
  region = "eu-north-1"
}

resource "random_id" "rand_id" {
  byte_length = 8
}

resource "aws_s3_bucket" "demo-bucket" {
  bucket = "demo-bucket-${random_id.rand_id.hex}"
}

resource "aws_s3_object" "bucket-data" {
  bucket = aws_s3_bucket.demo-bucket.bucket
  source = "./myfile.txt"      # local file path
  key    = "mydata.txt"        # object key in S3
}

output "name" {
  value = random_id.rand_id.hex
}
```

### Why the `random` provider

**S3 bucket names are globally unique across all AWS accounts on earth.** `demo-bucket` was taken years ago. `random_id` generates a suffix so the name can't collide.

```hcl
resource "random_id" "rand_id" {
  byte_length = 8      # 8 bytes → 16 hex characters
}
# .hex → "9419a72476f569ef"
```

**Key property: `random_id` is stored in state and stays stable across applies.** It regenerates only if the resource is destroyed and recreated, or if `keepers` change. If it regenerated every plan, every apply would replace your bucket.

Other random provider resources worth knowing: `random_password` (for generated secrets), `random_string`, `random_uuid`, `random_pet` (human-readable names like `tidy-badger`).

**This config also demonstrates multi-provider setup** — two entries in `required_providers`, and Terraform manages both plugin downloads.

### `source` vs `key`
- `source` = the **local** file path Terraform reads from
- `key` = the **S3 object key** (the path/filename inside the bucket)

They can differ, as here: `./myfile.txt` is uploaded as `mydata.txt`.

---

## Part 2 — Static website (`proj-static-website/`)

Six resources, each doing one thing. This separation is the modern AWS provider design.

```hcl
resource "random_id" "rand_id" {
  byte_length = 8
}

# 1. The bucket
resource "aws_s3_bucket" "mywebapp-bucket" {
  bucket = "mywebapp-bucket-${random_id.rand_id.hex}"
}

# 2. Unblock public access (required before a public bucket policy will work)
resource "aws_s3_bucket_public_access_block" "example" {
  bucket = aws_s3_bucket.mywebapp-bucket.id

  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}

# 3. Bucket policy allowing anonymous GetObject
resource "aws_s3_bucket_policy" "mywebapp" {
  bucket = aws_s3_bucket.mywebapp-bucket.id
  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Sid       = "PublicReadGetObject",
        Effect    = "Allow",
        Principal = "*",
        Action    = "s3:GetObject",
        Resource  = "${aws_s3_bucket.mywebapp-bucket.arn}/*"
      }
    ]
  })
}

# 4. Turn on website hosting
resource "aws_s3_bucket_website_configuration" "mywebapp" {
  bucket = aws_s3_bucket.mywebapp-bucket.id

  index_document {
    suffix = "index.html"
  }
}

# 5 & 6. Upload the files
resource "aws_s3_object" "index_html" {
  bucket       = aws_s3_bucket.mywebapp-bucket.bucket
  source       = "./index.html"
  key          = "index.html"
  content_type = "text/html"        # CRITICAL
}

resource "aws_s3_object" "styles_css" {
  bucket       = aws_s3_bucket.mywebapp-bucket.bucket
  source       = "./styles.css"
  key          = "styles.css"
  content_type = "text/css"         # CRITICAL
}

output "name" {
  value = aws_s3_bucket_website_configuration.mywebapp.website_endpoint
}
```

---

## Key learnings

### `content_type` is not optional
Without it, S3 defaults to `binary/octet-stream` and **the browser downloads the file instead of rendering it.** The site appears completely broken — you click the URL and get a download prompt. This is the number one "my S3 static site doesn't work" cause.

For many files, derive it:
```hcl
resource "aws_s3_object" "site" {
  for_each = fileset("${path.module}/site", "**/*")

  bucket       = aws_s3_bucket.web.id
  key          = each.value
  source       = "${path.module}/site/${each.value}"
  etag         = filemd5("${path.module}/site/${each.value}")
  content_type = lookup({
    html = "text/html"
    css  = "text/css"
    js   = "application/javascript"
    png  = "image/png"
    svg  = "image/svg+xml"
  }, split(".", each.value)[length(split(".", each.value)) - 1], "application/octet-stream")
}
```
`fileset()` + `for_each` uploads a whole directory. `etag = filemd5(...)` makes Terraform detect content changes — **without it, editing a file's contents produces no diff and the change never uploads.** That's a genuinely nasty gotcha.

### `public_access_block` must come first
AWS blocks public access by default on new buckets (four separate settings). A bucket policy granting public read is **silently overridden** unless these are turned off. Terraform's dependency graph handles the ordering because both reference the same bucket.

### `jsonencode` for policies
```hcl
policy = jsonencode({ ... })
```
Cleaner than a heredoc JSON string — you get HCL syntax highlighting, interpolation, and syntax validation. The alternative is `data "aws_iam_policy_document"`, which is more verbose but better for complex policies with conditions.

### Resource decomposition
Older AWS provider versions had everything as arguments on `aws_s3_bucket` (`acl`, `website`, `versioning`, `server_side_encryption_configuration`). **Provider v4+ split them into separate resources.** If you find a tutorial using `website { }` inside `aws_s3_bucket`, it's outdated. Knowing this signals you've dealt with provider upgrades.

---

## What's wrong with this for production

The course config is a teaching artifact. The honest critique — and being able to give it is more valuable in an interview than the config itself:

| Issue | Production answer |
|---|---|
| Bucket is publicly readable | **CloudFront + Origin Access Control**, bucket stays private. Only CloudFront can read it. |
| No HTTPS | S3 website endpoints are **HTTP only**. HTTPS requires CloudFront + ACM. |
| No custom domain | Route53 alias record → CloudFront distribution |
| No caching/CDN | CloudFront — faster globally and cheaper than serving everything from S3 |
| No `error_document` | Add an `error_document { key = "error.html" }` block |
| No versioning | `aws_s3_bucket_versioning` — recover from a bad deploy |
| No encryption | `aws_s3_bucket_server_side_encryption_configuration` |
| No `etag` on objects | Content changes don't trigger re-upload |

**The proper architecture:**
```
Route53 (DNS) → CloudFront (CDN + HTTPS via ACM cert in us-east-1)
                     ↓ Origin Access Control
                S3 bucket (PRIVATE)
```

> **Interview line:** "The S3 website endpoint approach works but it's HTTP-only and requires a public bucket. For anything real I'd front it with CloudFront using Origin Access Control, which keeps the bucket private and gives you HTTPS, caching, and a custom domain. The ACM certificate has to be in us-east-1 because that's the only region CloudFront accepts certificates from — which is a good use of a provider alias."

---

## Real-world mapping

| Concept | Job reality |
|---|---|
| S3 static hosting | Frontend SPA deploys (React/Vue build output), documentation sites |
| `random_id` for unique names | Any globally-unique resource name in a multi-environment setup |
| `jsonencode` for policies | Every IAM policy, bucket policy, ECS task definition you'll write |
| `fileset` + `for_each` upload | Deploying a built frontend directory |
| `public_access_block` | Security baseline — almost always the opposite of this config: all four set to `true` |

**Connection to my own work:** `terraform-fullstack` already provisions S3 for remote state. The interesting contrast is that the state bucket should have **every** public access block set to `true` plus encryption plus versioning, while this course config sets them all to `false`. Same resource type, opposite security posture, driven entirely by purpose.

---

## Interview Q&A

**Q: How would you host a static website on AWS with Terraform?**
> The minimal version is an S3 bucket with website configuration enabled, a bucket policy allowing public GetObject, the public access block settings relaxed, and the files uploaded as S3 objects with correct content types. But I wouldn't ship that. For anything production-facing I'd keep the bucket completely private and put CloudFront in front of it using Origin Access Control, because the S3 website endpoint is HTTP-only and can't do a custom domain properly. CloudFront gives HTTPS through an ACM certificate, edge caching, and a Route53 alias for the domain. The one wrinkle is that the ACM certificate has to live in us-east-1 regardless of where everything else is, so that needs an aliased provider.

**Q: Why does the S3 bucket name need a random suffix?**
> S3 bucket names are globally unique across every AWS account in the world, not just within my account, so any obvious name is already taken. I use the `random_id` resource to generate a suffix and interpolate it into the name. The important property is that the random value is stored in state and stays stable across applies — it only regenerates if the resource is destroyed and recreated. If it regenerated on every plan, every apply would try to replace the bucket.

**Q: You deployed a static site to S3 and the browser downloads the HTML instead of rendering it. What happened?**
> Missing content type. If `content_type` isn't set on the S3 object, S3 serves it as binary/octet-stream and the browser treats it as a download rather than a page. Setting `content_type = "text/html"` on the HTML objects and `text/css` on the stylesheets fixes it. When I'm uploading a whole directory with `fileset` and `for_each`, I derive the content type from the file extension with a lookup map. The related gotcha in that setup is that I also need `etag = filemd5(...)` on each object, because without it Terraform doesn't notice when a file's contents change and the update never actually uploads.

**Q: What's the difference between a bucket policy and the public access block settings?**
> The bucket policy is the grant — it says who can do what. The public access block is an override that sits above it, and by default AWS enables all four settings on new buckets, which means a bucket policy granting public read is silently ignored. So to make a bucket public you have to relax the public access block first, and Terraform orders that correctly because both resources reference the same bucket. It's worth noting the default exists for a good reason — publicly readable buckets have caused a lot of data breaches — so in almost every case other than a deliberate static site I'd want all four settings left at true.

---

## Gotchas & flags

- **`content_type` missing → browser downloads instead of rendering.** The most common failure.
- **`etag` missing → file content changes produce no Terraform diff.** The most confusing failure.
- Bucket names are globally unique and **must be lowercase**, 3–63 chars, no underscores.
- `aws_s3_bucket` in provider v4+ no longer accepts `acl`, `website`, `versioning` inline — those are separate resources now.
- `terraform destroy` on a non-empty bucket **fails** unless `force_destroy = true` is set on the bucket.
- The S3 website endpoint (`bucket.s3-website-region.amazonaws.com`) is **HTTP only** — it will never serve HTTPS.

---

Next: [[15-Project-VPC-EC2-Nginx]]
