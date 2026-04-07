# How it works

## What is AWS Lambda?

AWS Lambda is a **serverless compute service** that allows you to run code **without managing servers**.

#### 🔹 Key Concept

* You provide **code**
* Lambda **executes it**
* Returns **output**

#### 🔹 Supported Languages

* Python
* Java
* Node.js
* Go

***

### 🔄 Lambda vs EC2

| Feature           | Lambda            | EC2                |
| ----------------- | ----------------- | ------------------ |
| Server Management | ❌ Not required    | ✅ Required         |
| Scaling           | Auto              | Manual             |
| Billing           | Pay per execution | Pay per uptime     |
| Setup             | Instant           | Needs provisioning |

***

### ✅ Advantages

* No server setup
* No start/stop management
* Pay-as-you-use
* Fully serverless



## ⚙️ How Lambda Works in Real Time

#### 🔄 Execution Flow

```
Event → Trigger → Lambda → Execution → Response → Logs
```

## 🔥 Real-Time Example (Very Important)

### 🧩 Scenario: Image Processing App

1. User uploads image to S3
2. S3 triggers Lambda
3. Lambda:
   * Resizes image
   * Saves new image to another bucket
4. Logs stored in CloudWatch

👉 No server, no scaling issues
