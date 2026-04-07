# Execution in Terraform

## 🧠 1. How CloudFront improves security & reduces S3 cost

### 🔐 Security Improvement

Without CloudFront:

* S3 bucket must be **public**
*   Anyone can directly hit:

    ```
    https://s3.amazonaws.com/bucket/file
    ```
* Risks:
  * Data exposure
  * No control over access patterns
  * Easy target for abuse / DDoS

***

With CloudFront + OAC:

* S3 bucket is **PRIVATE**
* Only CloudFront can access it

👉 Flow:

```
User → CloudFront → S3
```

NOT:

```
User → S3 ❌
```

#### ✅ What enforces this?

* **Origin Access Control (OAC)**
* **Bucket Policy with SourceArn restriction**

👉 Example concept:

```
Condition:
  AWS:SourceArn = CloudFront Distribution ARN
```

💡 Meaning:

> Only THIS CloudFront distribution can access S3 — nobody else.

***

### 💰 Cost Optimization

Without CloudFront:

* Every request hits S3 (global traffic)
* High:
  * Data transfer cost
  * Latency
  * Repeated fetch cost

***

With CloudFront:

#### 🚀 Edge Caching

* Files cached at edge locations (Mumbai, US, etc.)
* Same request served locally

👉 Example:

* 1000 users from India:
  * First request → S3
  * Next 999 → Edge cache (FREE-ish compared to origin)

***

#### 📉 Cost Impact

| Scenario   | Cost                                         |
| ---------- | -------------------------------------------- |
| Direct S3  | High data transfer (cross-region)            |
| CloudFront | Reduced origin fetch + cheaper edge delivery |

***

## 🧠 2. Why OAC is better than OAI

### ❌ Old: Origin Access Identity (OAI)

* Legacy mechanism
* IAM-based identity
* Limited flexibility
* No modern security features

***

### ✅ New: Origin Access Control (OAC)

#### Key advantages:

#### 🔒 1. SigV4 Signing

* Requests are **cryptographically signed**
* Stronger authentication

***

#### 🧩 2. Fine-grained control

* Works better with:
  * Bucket policies
  * Conditional access

***

#### 🚀 3. Supports modern AWS architecture

* Required for newer features
* AWS recommendation (OAI is effectively legacy)

***

#### ⚠️ Interview Answer (short)

> OAC is preferred over OAI because it provides SigV4-based secure authentication, better policy control, and aligns with modern AWS security practices.

***

## 🧠 3. How to automate static website using Terraform

Now let’s convert what you saw into **clean DevOps thinking**.

***

### 🏗️ Components you must automate

#### 1. S3 Bucket

* Private bucket
* Static files storage

```
resource "aws_s3_bucket" "site" {
  bucket = var.bucket_name
}
```

***

#### 2. Block Public Access

```
resource "aws_s3_bucket_public_access_block" "block" {
  bucket = aws_s3_bucket.site.id

  block_public_acls   = true
  block_public_policy = true
  restrict_public_buckets = true
}
```

***

#### 3. Upload Files (Important DevOps pattern)

```
resource "aws_s3_object" "files" {
  for_each = fileset("${path.module}/www", "**")

  bucket = aws_s3_bucket.site.id
  key    = each.value
  source = "${path.module}/www/${each.value}"

  etag = filemd5("${path.module}/www/${each.value}")
}
```

💡 This is **real production pattern**:

* Auto-upload all files
* No manual steps

***

#### 4. Origin Access Control (OAC)

```
resource "aws_cloudfront_origin_access_control" "oac" {
  name                              = "oac-demo"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}
```

***

#### 5. Bucket Policy (Critical)

👉 This is where most people fail (you saw errors in video)

```
resource "aws_s3_bucket_policy" "policy" {
  bucket = aws_s3_bucket.site.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Service = "cloudfront.amazonaws.com"
      }
      Action = "s3:GetObject"
      Resource = "${aws_s3_bucket.site.arn}/*"
      Condition = {
        StringEquals = {
          "AWS:SourceArn" = aws_cloudfront_distribution.cdn.arn
        }
      }
    }]
  })
}
```

***

#### 6. CloudFront Distribution

Key parts:

* Origin → S3
* OAC attached
* HTTPS redirect
* Cache config

***

### ⚠️ Real DevOps Learnings from This Video

#### 🔥 1. Provider changes break code

You saw:

> `aws_s3_bucket_object` → deprecated → `aws_s3_object`

👉 Lesson:

* Always **lock provider version**

```
required_providers {
  aws = {
    source  = "hashicorp/aws"
    version = "~> 5.0"
  }
}
```

***

#### 🔥 2. IAM Policy errors are common

Errors you saw:

* Invalid Principal
* Malformed policy
* Wrong action/resource

👉 Real skill:

> Debugging policies is a **must-have DevOps skill**

***

#### 🔥 3. Implicit vs Explicit dependency

```
bucket = aws_s3_bucket.site.id  # implicit
```

vs

```
depends_on = [aws_s3_bucket_public_access_block.block]
```

***

#### 🔥 4. Cache behavior understanding

* TTL = performance vs freshness tradeoff
* Cache invalidation = extra cost

***

### 💡 Interview Question You’ll Get

👉 “How do you securely host a static website on AWS using Terraform?”

Answer structure:

1. Private S3 bucket
2. CloudFront distribution
3. OAC for secure access
4. Bucket policy with SourceArn
5. Cache optimization
6. Optional: Route53 + ACM
