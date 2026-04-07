# Execution in AWS

## 🚀 Lab: Deploy Website using Amazon CloudFront + Amazon S3

***

## 🎯 Goal

* Host a static website globally
* Serve it via CDN (CloudFront)
* Secure S3 (no public access)

***

## 🧱 Architecture

```
User
 ↓
CloudFront (CDN + HTTPS)
 ↓
S3 (private bucket)
```

***

## ⚠️ Before You Start

* AWS account ready
* Region doesn’t matter (CloudFront is global)

***

## 🧪 Step 1: Create S3 Bucket

#### 👉 Actions:

1. Go to S3 → Create bucket
2.  Bucket name:

    ```
    your-unique-site-name
    ```
3. Region: any
4. ❗ Uncheck:
   * **Block all public access** (for now)
5. Create bucket

***

## 📂 Step 2: Upload Website Files

Create a simple file locally:

#### `index.html`

```
<html>
  <h1>Hello from CloudFront 🚀</h1>
</html>
```

👉 Upload to S3 bucket

***

## 🌐 Step 3: Enable Static Website Hosting

1. Go to bucket → Properties
2. Scroll → Static website hosting
3. Enable:
   * Index document: `index.html`

👉 You’ll get:

```
http://bucket-name.s3-website-...
```

***

## 🌍 Step 4: Create CloudFront Distribution

***

### 👉 Go to CloudFront → Create Distribution

#### Origin Settings:

* Origin domain → select your S3 bucket
* ⚠️ Choose:
  * **S3 bucket (NOT website endpoint)**

***

#### 🔐 Important (Security Setup)

Set:

* Origin access → **Origin Access Control (OAC)**
* Click:
  * “Create new OAC”

👉 This ensures:

* S3 is private
* Only CloudFront can access\
  <br>

OAC is auto created by AWS, while creating the distribution.

***

#### Viewer Settings:

* Viewer protocol policy:
  * ✅ Redirect HTTP → HTTPS

***

#### Default Root Object:

```
index.html
```

***

👉 Click **Create Distribution**

⏳ Wait 5–10 minutes (deployment time)

***

## 🔐 Step 5: Fix S3 Permissions (CRITICAL)

After creating distribution:

1. Go to S3 → Permissions
2. Update bucket policy (CloudFront will suggest)

👉 Allow CloudFront access

***

## 🚫 Step 6: Make Bucket Private

Now:

* Enable **Block all public access**

👉 This is production best practice

***

## 🌍 Step 7: Access Your Website

Go to:

```
https://dxxxxx.cloudfront.net
```

🎉 Your site is live globally

***

## ⚡ Step 8: Test Caching

***

### 👉 Update your `index.html`

Change text:

```
Hello Version 2
```

Upload again

***

#### ❗ You’ll still see OLD version

👉 Why?

* Cached by CloudFront

***

## 🔄 Step 9: Invalidate Cache

1. Go to CloudFront → Distributions
2. Click your distribution
3. Go to **Invalidations**
4. Create:

```
/*
```

👉 Now refresh → updated content

***

## 🎯 What You Learned

* CDN working (CloudFront)
* Secure S3 access (OAC)
* Cache behavior
* Invalidation (very important)

***

## 🔥 DevOps Upgrade (Must Try Next)

***

### ✅ Add Custom Domain

Use:

* Amazon Route 53

***

### ✅ Add SSL Certificate

Use:

* AWS Certificate Manager

***

### ✅ CI/CD Integration

* Upload build to S3 automatically
* Trigger CloudFront invalidation

***

## 🚨 Common Errors (You WILL Face)

***

#### ❌ Access Denied

👉 Fix:

* Check bucket policy
* Check OAC

***

#### ❌ Old content showing

👉 Fix:

* Invalidation

***

#### ❌ Wrong origin selected

👉 Fix:

* Use S3 bucket endpoint (not website URL)

***

## 🎯 Interview Question (From This Lab)

👉 **How do you securely host a static website using CloudFront?**

Answer:

* Use S3 as origin
* Enable OAC
* Block public access
* Serve via CloudFront
