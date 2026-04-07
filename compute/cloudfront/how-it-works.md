# How it works

## 🌐 Amazon CloudFront

### 🚀 What is CloudFront?

* A **CDN (Content Delivery Network)**
* Delivers content from **edge locations** closer to users
* Reduces latency and improves speed





### 🔁 Complete Flow

#### 1️⃣ User Requests Content

*   A user opens your website:

    ```
    https://example.com
    ```

***

#### 2️⃣ Request Goes to Nearest Edge Location

* CloudFront routes request to **closest edge server**
* Example:
  * User in Bangalore → hits nearby edge location

👉 This is what reduces latency

***

#### 3️⃣ CloudFront Checks Cache

### ✅ Case 1: Cache HIT (fast ⚡)

* File already exists in edge cache
* CloudFront returns immediately

👉 Response time: milliseconds

***

### ❌ Case 2: Cache MISS (slow first time)

* File not in cache
* CloudFront forwards request to **origin**

***

#### 4️⃣ Origin Fetch

Origin can be:

* Amazon S3 (static files)
* Amazon EC2 (apps)
* Elastic Load Balancing (backend services)

👉 CloudFront fetches data from origin

***

#### 5️⃣ Cache the Response

* CloudFront stores the file in edge location
* Based on **TTL (Time To Live)**

***

#### 6️⃣ Return Response to User

* User gets response
* Next requests → faster (cache hit)

***

## 🧠 Visual Flow

```
User
 ↓
Nearest Edge Location (CloudFront)
 ↓ (if cache miss)
Origin (S3 / ALB / EC2)
 ↓
Edge caches response
 ↓
User gets content
```

***

## 🔐 Security Layer (Important)

CloudFront also acts as a **shield**

***

### 🔒 Features:

* HTTPS (SSL termination)
* Integration with AWS WAF
* Restrict direct access to origin

👉 Example:

* Block users from directly accessing S3

***

## 🔥 Real DevOps Scenario

***

### 🎯 Example: React App Deployment

```
User
 ↓
CloudFront (CDN + HTTPS)
 ↓
S3 (React build files)
```

***

### 🎯 Example: Microservices App

```
User
 ↓
CloudFront
 ↓
ALB
 ↓
EKS / EC2 backend
```

***

## 🎯 Interview-Ready Answer (Short)

If asked:

👉 **“How does CloudFront work?”**

You say:

> CloudFront routes user requests to the nearest edge location. If the content is cached, it returns immediately. If not, it fetches from the origin like S3 or ALB, caches it based on TTL, and then serves it to users, improving latency and reducing load on the backend.
