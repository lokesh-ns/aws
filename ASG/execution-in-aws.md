# EC2 Auto Scaling Group — AWS Console Step-by-Step Guide

## 🎯 What We'll Build

```
Internet
    │
    ▼
Application Load Balancer (ALB)
    │  internet-facing, port 80
    ▼
Target Group (target-ec2-apache2)
    │
    ├──────────────────────────┐
    ▼                          ▼
EC2 (12.0.1.x)           EC2 (12.0.3.x)
public subnet 1a          public subnet 1b
    ▲                          ▲
    └──────────────────────────┘
         Auto Scaling Group
         min=1, desired=2, max=3
```

---

## 💡 Why Auto Scaling?

| Traditional Setup | Auto Scaling Group |
|---|---|
| Fixed number of EC2 instances | Dynamic — scales up and down |
| Pay for idle instances at night | Only pay for what you need |
| Manual intervention to add instances | Automatically provisions new instances |
| No recovery if instance fails | Auto-replaces unhealthy instances |

> **Real-world example:** During the day you get high traffic, at night very few requests. Auto Scaling reduces instances at night and scales back up in the morning — saving significant cost.

---

## ⚠️ Before You Start

- AWS account ready
- Always check the region selector (top right) before creating anything
- All resources in this guide go inside the same `test-vpc`

---

## 📍 PART 1 — Create VPC & Networking

### Step 1 — Create VPC

> A VPC is your private isolated network. All resources — subnets, EC2 instances, load balancer — live inside this VPC.

- Search **VPC** → **Your VPCs** → **Create VPC**

| Field | Value |
|---|---|
| Resources to create | VPC only |
| Name | test-vpc |
| IPv4 CIDR | 12.0.0.0/16 |
| Tenancy | Default |

- Click **Create VPC**

---

### Step 2 — Create Internet Gateway

> Internet Gateway provides internet access to resources in public subnets. Without it, the load balancer cannot receive traffic from the internet.

- **Internet Gateways** → **Create internet gateway**
- Name: `internet-gateway-test`
- Click **Create internet gateway**
- **Actions** → **Attach to VPC** → select `test-vpc` → **Attach internet gateway**

✅ State changes from `detached` to `attached`

---

### Step 3 — Create Two Public Subnets

> Two subnets across two Availability Zones is required for high availability. ALB also requires subnets in at least 2 AZs.

- **Subnets** → **Create subnet** → VPC: `test-vpc`

**Subnet 1:**

| Field | Value |
|---|---|
| Subnet name | test-public-subnet-1a |
| Availability Zone | eu-central-1a |
| IPv4 CIDR | 12.0.1.0/24 |

Click **Add new subnet:**

**Subnet 2:**

| Field | Value |
|---|---|
| Subnet name | test-public-subnet-1b |
| Availability Zone | eu-central-1b |
| IPv4 CIDR | 12.0.3.0/24 |

- Click **Create subnet**

✅ Two public subnets created across two AZs

---

### Step 4 — Create Public Route Table

> Adding `0.0.0.0/0 → IGW` makes these subnets public — any resource inside can reach the internet and be reached from the internet.

- **Route Tables** → **Create route table**

| Field | Value |
|---|---|
| Name | route-table-test-public |
| VPC | test-vpc |

- Click **Create route table**

**Associate both subnets:**
- **Subnet associations tab** → **Edit subnet associations**
- ✅ Select `test-public-subnet-1a`
- ✅ Select `test-public-subnet-1b`
- Click **Save associations**

**Add internet route:**
- **Routes tab** → **Edit routes** → **Add route**

| Destination | Target |
|---|---|
| 0.0.0.0/0 | Internet Gateway → internet-gateway-test |

- Click **Save changes**

---

## 📍 PART 2 — Create Security Groups

### Step 5 — Create ALB Security Group

> The ALB needs its own security group to allow HTTP traffic from the internet.

- **VPC** → **Security Groups** → **Create security group**

| Field | Value |
|---|---|
| Name | alb-sg-http |
| Description | Allow HTTP requests to ALB |
| VPC | test-vpc |

**Inbound rules:**

| Type | Port | Source |
|---|---|---|
| HTTP | 80 | 0.0.0.0/0 |

- Click **Create security group**

---

### Step 6 — Create EC2 Security Group

> EC2 instances need port 80 for HTTP traffic from the ALB, and port 22 for SSH access.

- **VPC** → **Security Groups** → **Create security group**

| Field | Value |
|---|---|
| Name | lt-sg-ec2-apache2 |
| Description | Allow SSH and HTTP for EC2 instances |
| VPC | test-vpc |

**Inbound rules:**

| Type | Port | Source |
|---|---|---|
| HTTP | 80 | 0.0.0.0/0 |
| SSH | 22 | 0.0.0.0/0 |

- Click **Create security group**

---

## 📍 PART 3 — Create Target Group

### Step 7 — Create Target Group

> A Target Group is a logical group of EC2 instances the load balancer routes traffic to. Created empty now — ASG registers instances automatically.

- **EC2** → **Load Balancing** → **Target Groups** → **Create target group**

| Field | Value |
|---|---|
| Target type | Instances |
| Target group name | target-ec2-apache2 |
| Protocol | HTTP |
| Port | 80 |
| VPC | test-vpc |
| Protocol version | HTTP1 |

- Health checks: leave defaults
- Click **Next** → **Create target group**

✅ Empty target group created — ASG will populate it

---

## 📍 PART 4 — Create Application Load Balancer

### Step 8 — Create ALB

> The ALB distributes incoming HTTP requests across EC2 instances and automatically routes away from unhealthy ones.

- **EC2** → **Load Balancers** → **Create load balancer**
- Choose **Application Load Balancer** → **Create**

| Field | Value |
|---|---|
| Name | ALB-EC2-autoscaling-group |
| Scheme | Internet-facing |
| IP address type | IPv4 |
| VPC | test-vpc |
| Subnets | ✅ test-public-subnet-1a + ✅ test-public-subnet-1b |
| Security groups | alb-sg-http (remove default) |

**Listeners and routing:**

| Listener | Port | Default action |
|---|---|---|
| HTTP | 80 | Forward to → target-ec2-apache2 |

- Click **Create load balancer**

> ⚠️ Wait 2-3 minutes until state changes from `provisioning` to `active`

---

## 📍 PART 5 — Create Auto Scaling Group (No Launch Template Yet)

> ⚠️ **This is exactly what was done initially** — ASG created without user data in the launch template. This caused 502 errors because instances launched with nothing running on port 80. The fix using nginx user data came later.

### Step 9 — Create Launch Template (Without User Data)

- **EC2** → **Launch Templates** → **Create launch template**

| Field | Value |
|---|---|
| Name | LT-EC2-instances-apache2 |
| AMI | Ubuntu Server 22.04 LTS |
| Instance type | t2.micro |
| Key pair | select or create |
| Security groups | lt-sg-ec2-apache2 |
| Auto-assign public IP | Enable |
| **User data** | **Left empty ← this caused the 502** |

- Click **Create launch template**

---

### Step 10 — Create Auto Scaling Group

- **EC2** → **Auto Scaling Groups** → **Create Auto Scaling group**

**Step 1 — Name and template:**

| Field | Value |
|---|---|
| Name | autoscaling-group-EC2-test-demo |
| Launch template | LT-EC2-instances-apache2 |
| Version | 1 (Latest) |

**Step 2 — Network:**

| Field | Value |
|---|---|
| VPC | test-vpc |
| Subnets | ✅ test-public-subnet-1a + ✅ test-public-subnet-1b |

**Step 3 — Load balancing:**

| Field | Value |
|---|---|
| Load balancing | Attach to an existing load balancer |
| Target group | target-ec2-apache2 |
| Health check type | ✅ Enable Elastic Load Balancing health checks |
| Health check grace period | 200 seconds |

**Step 4 — Capacity:**

| Field | Value |
|---|---|
| Desired capacity | 2 |
| Minimum capacity | 1 |
| Maximum capacity | 3 |
| Scaling policies | None |

- Click through → **Create Auto Scaling group**

✅ ASG created — 2 EC2 instances automatically provisioned

---

## 🚨 PART 6 — The 502 Error (What Went Wrong)

### What Happened

After ASG created the instances, the ALB DNS returned **502 Bad Gateway** and the target group showed both targets as **Unhealthy**.

```
ALB sends health check → EC2:80
EC2 responds           → ❌ Connection refused
Result                 → Health check fails → 502 Bad Gateway
```

### Step 11 — Diagnose the Issue

SSH into one of the EC2 instances:

```bash
# SSH into the EC2 instance
ssh -i "your-key.pem" ubuntu@<ec2-public-ip>

# Check if anything is running on port 80
curl localhost:80
# Output: curl: (7) Failed to connect to localhost port 80: Connection refused ❌

# Check Apache
systemctl status apache2
# Output: Unit apache2.service could not be found ❌
```

### Root Cause Confirmed

```
No user data in launch template
    ↓
EC2 instances launch with bare Ubuntu OS
    ↓
Nothing installed, nothing running on port 80
    ↓
ALB health check hits port 80 → Connection refused
    ↓
Target marked Unhealthy
    ↓
ALB returns 502 Bad Gateway ❌
```

---

## 🔧 PART 7 — Quick Fix: Manually Install Nginx

> This was the immediate fix applied to get the existing instances working. Repeated for both EC2 instances.

### Step 12 — SSH into Each EC2 and Install Nginx

```bash
# SSH into EC2 instance
ssh -i "your-key.pem" ubuntu@<ec2-public-ip>

# Install and start Nginx
sudo apt update
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx

# Verify it is running
curl localhost:80
# ✅ Should return Nginx HTML
```

Repeat the same for the second EC2 instance.

### Verify Fix in Console

- **EC2** → **Target Groups** → `target-ec2-apache2` → **Targets tab**
- Both instances should now show **Healthy** ✅

- Open ALB DNS in browser:
```
http://<your-alb-dns>
```
✅ Nginx welcome page loads

Refresh a few times — IP in response changes as ALB load balances between both instances.

---

## ✅ PART 8 — Proper Fix: Update Launch Template with Nginx User Data

> Manual fix only works for existing instances. New instances created by ASG would still launch empty. The proper fix is updating the launch template so all future instances auto-install nginx.

### Step 13 — Create New Launch Template Version with User Data

- **EC2** → **Launch Templates** → select `LT-EC2-instances-apache2`
- **Actions** → **Modify template (Create new version)**
- **Advanced details** → **User data** → add:

```bash
#!/bin/bash
apt-get update -y
apt-get install -y nginx
systemctl start nginx
systemctl enable nginx
echo "<h1>Server: $(hostname) | IP: $(hostname -I)</h1>" > /var/www/html/index.html
```

- Click **Create template version**

---

### Step 14 — Update ASG to Use New Launch Template Version

> ASG must be updated to use the new version — otherwise new instances still use the old empty version.

- **EC2** → **Auto Scaling Groups** → select your ASG
- Click **Edit**
- Launch template version → select **Latest**
- Click **Update**

✅ All new instances created by ASG from now will automatically have Nginx running and pass health checks

---

## 🔄 Before vs After

```
BEFORE (no user data):
──────────────────────
ASG creates EC2
    → bare Ubuntu OS
    → nothing on port 80
    → health check fails
    → Target: Unhealthy ❌
    → User sees 502 Bad Gateway ❌

AFTER (nginx user data added):
───────────────────────────────
ASG creates EC2
    → user data script runs on boot
    → nginx installs automatically
    → nginx starts on port 80
    → health check passes
    → Target: Healthy ✅
    → User sees Nginx page ✅
```

---

## 📍 PART 9 — Test Auto Scaling Recovery

### Step 15 — Terminate One Instance and Watch ASG Replace It

1. **EC2** → **Instances** → select one → **Instance state** → **Terminate**
2. **Auto Scaling Groups** → your ASG → **Instance management tab**
3. Refresh every 30 seconds — you will see:
   - Terminated instance → `Unhealthy`
   - New instance → `Pending` → `InService`
4. **Target Groups** → new instance appears as `Healthy` ✅
5. New instance already has Nginx running from updated launch template ✅

---

## 📋 Full Execution Flow (What Actually Happened)

```
Step 1  → VPC, IGW, 2 subnets, route table created
Step 2  → Security groups created (ALB + EC2)
Step 3  → Empty target group created
Step 4  → ALB created and pointing to target group
Step 5  → Launch template created WITHOUT user data
Step 6  → ASG created using that launch template
             ↓
             2 EC2 instances auto-provisioned
             ↓
          🚨 502 Bad Gateway — targets unhealthy
             ↓
Step 7  → Diagnosed: curl localhost:80 → Connection refused
             ↓
Step 8  → Manual fix: SSH into each EC2 → apt install nginx
             ↓
          ✅ Both targets healthy — ALB working
             ↓
Step 9  → Launch template updated with nginx user data (v2)
Step 10 → ASG updated to use latest launch template version
             ↓
          ✅ All future instances auto-install nginx on boot
```

---

## ✅ Summary — What Each Resource Does

| Resource | Purpose |
|---|---|
| VPC `12.0.0.0/16` | Isolated private network |
| Public subnet 1a `12.0.1.0/24` | EC2 instances in AZ-1a |
| Public subnet 1b `12.0.3.0/24` | EC2 instances in AZ-1b |
| Internet Gateway | Internet access for ALB and EC2 |
| Route Table | Routes `0.0.0.0/0` to IGW |
| ALB Security Group | Port 80 open for internet traffic |
| EC2 Security Group | Port 80 + 22 open on instances |
| Target Group | Logical group of instances ALB routes to |
| Application Load Balancer | Distributes traffic across instances |
| Launch Template v1 (no user data) | Initial config — caused 502 |
| Launch Template v2 (nginx user data) | Fixed config — auto-installs nginx |
| Auto Scaling Group | Creates/replaces EC2s based on min/desired/max |

---

## 🔑 Key Interview Questions

| Question | Answer |
|---|---|
| Why were targets unhealthy in ALB? | Nothing running on port 80 — no app installed on EC2 |
| How did you find the root cause? | SSH into EC2, ran `curl localhost:80` — got Connection refused |
| What is the quick fix? | Manually SSH in and install nginx |
| What is the proper fix? | Add user data to launch template so every new instance auto-installs nginx |
| Why is user data critical in ASG? | ASG creates instances automatically — without user data they launch completely empty |
| What is min, max, desired in ASG? | Min = never go below, Max = never exceed, Desired = target count to maintain |
| What is a Launch Template? | Blueprint defining AMI, instance type, SG, user data that ASG uses to create EC2s |
| How does ASG self-heal? | Detects instance count below desired, reads launch template, provisions replacement |
| What causes 502 Bad Gateway from ALB? | EC2 running but nothing listening on port 80 |
| Why 2 subnets across 2 AZs? | High availability — if one AZ fails, the other continues serving |
| What is a Target Group? | Group of EC2 instances ALB routes to, with health checks determining routing |

---

## 🔜 Next Step — Terraform Equivalent

| Console Action | Terraform Resource |
|---|---|
| Create VPC | `aws_vpc` |
| Create Subnet | `aws_subnet` |
| Create IGW | `aws_internet_gateway` |
| Create Route Table | `aws_route_table` |
| Associate Subnet | `aws_route_table_association` |
| Create Security Group | `aws_security_group` |
| Create Target Group | `aws_lb_target_group` |
| Create ALB | `aws_lb` |
| Create ALB Listener | `aws_lb_listener` |
| Create Launch Template | `aws_launch_template` |
| Create Auto Scaling Group | `aws_autoscaling_group` |
| Attach ASG to Target Group | `aws_autoscaling_attachment` |