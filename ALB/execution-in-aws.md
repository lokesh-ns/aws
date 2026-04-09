# AWS Application Load Balancer (ALB) — Console Step-by-Step Guide

## 🎯 What We'll Build

```
                        User (Browser)
                             │
                             │ http://<alb-dns-name>
                             ▼
                 Application Load Balancer (ALB)
                   test-LB-EC2-demo
                   internet-facing | port 80
                             │
                             ▼
                      Target Group
                  test-EC2-ALB-demo-TG
                             │
               ┌─────────────┴─────────────┐
               ▼                           ▼
    EC2 Instance 1                EC2 Instance 2
    test-EC2-instance-1           test-EC2-instance-2
    test-public-subnet-1a         test-public-subnet-1b
    IP: 12.0.1.142                IP: 12.0.3.160
    Apache running                Apache running
    (prints hostname + IP)        (prints hostname + IP)
```

> **How it works:** User hits the ALB DNS. ALB checks the target group. Target group has 2 healthy EC2 instances. ALB routes each request in round-robin — first request goes to EC2-1, next to EC2-2, and so on. Refreshing the browser shows a different IP each time, confirming load balancing is working.

---

## ⚠️ Before You Start

- AWS account ready
- Region selector (top right) — check before every resource creation
- All resources created inside `test-vpc`
- Key pair `.pem` file downloaded and permissions set (`chmod 400`)

---

## 📍 PART 1 — Create VPC

### Step 1 — Create VPC

> A VPC is your private isolated network inside AWS. Every resource in this guide — subnets, EC2 instances, ALB — lives inside this VPC. The CIDR `12.0.0.0/16` gives us a large IP range to carve subnets from.

- Search **VPC** in console → **Your VPCs** → **Create VPC**

| Field | Value |
|---|---|
| Resources to create | VPC only |
| Name tag | test-vpc |
| IPv4 CIDR | 12.0.0.0/16 |
| Tenancy | Default |

- Click **Create VPC**

✅ VPC created — verify by clicking the VPC ID and confirming CIDR is `12.0.0.0/16`

---

## 📍 PART 2 — Create Internet Gateway

### Step 2 — Create and Attach Internet Gateway

> Internet Gateway is the bridge between your VPC and the public internet. Without it, your EC2 instances have no internet access and the ALB cannot receive requests from outside. Two actions are needed — create it, then attach it to the VPC.

- **VPC dashboard** → left sidebar → **Internet Gateways** → **Create internet gateway**

| Field | Value |
|---|---|
| Name tag | internet-gateway-test |

- Click **Create internet gateway**
- Click **Actions** → **Attach to VPC** → select `test-vpc` → **Attach internet gateway**

✅ State changes from `detached` → `attached`

---

## 📍 PART 3 — Create Two Public Subnets

### Step 3 — Create Public Subnet 1 (1A)

> Subnets are segments of the VPC tied to one Availability Zone. We create two subnets in two different AZs so the ALB can distribute traffic across zones. ALB requires at least 2 subnets in 2 different AZs to work.

- **VPC dashboard** → **Subnets** → **Create subnet**
- VPC: select `test-vpc`

| Field | Value |
|---|---|
| Subnet name | test-public-subnet-1a |
| Availability Zone | eu-central-1a |
| IPv4 CIDR | 12.0.1.0/24 |

---

### Step 4 — Create Public Subnet 2 (1B)

> Second subnet in a different AZ. EC2 instance 2 will go here. Using different AZs means if one AZ goes down, traffic is still served from the other.

- Click **Add new subnet** (in the same create flow)

| Field | Value |
|---|---|
| Subnet name | test-public-subnet-1b |
| Availability Zone | eu-central-1b |
| IPv4 CIDR | 12.0.3.0/24 |

- Click **Create subnet**

✅ Two subnets created:
- `test-public-subnet-1a` → `12.0.1.0/24` → eu-central-1a
- `test-public-subnet-1b` → `12.0.3.0/24` → eu-central-1b

---

## 📍 PART 4 — Create Route Table

### Step 5 — Create Public Route Table

> Route table tells traffic inside the VPC where to go. By adding a route `0.0.0.0/0 → IGW`, we make both subnets public — instances inside can reach the internet and be reached from the internet.

- **VPC dashboard** → **Route Tables** → **Create route table**

| Field | Value |
|---|---|
| Name | test-public-RT |
| VPC | test-vpc |

- Click **Create route table**

---

### Step 6 — Associate Both Subnets with Route Table

> Without this association, subnets still use the default main route table which has no internet route. This step connects our custom route table to both public subnets.

- Select `test-public-RT` → **Subnet associations tab** → **Edit subnet associations**
- ✅ Select `test-public-subnet-1a`
- ✅ Select `test-public-subnet-1b`
- Click **Save associations**

---

### Step 7 — Add Internet Route

> This is what makes subnets truly "public". The route `0.0.0.0/0 → internet-gateway-test` tells the route table: for any destination on the internet, send traffic through the IGW.

- **Routes tab** → **Edit routes** → **Add route**

| Destination | Target |
|---|---|
| 0.0.0.0/0 | Internet Gateway → internet-gateway-test |

- Click **Save changes**

✅ Route table now has:
- Local route: `12.0.0.0/16 → local` (auto-created)
- Internet route: `0.0.0.0/0 → igw-xxx`

---

## 📍 PART 5 — Create Two EC2 Instances

### Step 8 — Create Key Pair

> Key pair is required for SSH access into EC2 instances. Both instances will use the same key pair for convenience. Download and secure the `.pem` file immediately — it cannot be downloaded again.

- **EC2** → **Key Pairs** → **Create key pair**

| Field | Value |
|---|---|
| Name | test-ALB-demo-key-pair |
| Key pair type | RSA |
| Private key format | .pem |

- Click **Create key pair** — `.pem` file downloads automatically

```bash
# Set permissions on your local terminal
chmod 400 test-ALB-demo-key-pair.pem
```

---

### Step 9 — Launch EC2 Instance 1 (Subnet 1A)

> First EC2 instance goes in subnet-1a (eu-central-1a). Apache is installed via user data and configured to print its hostname and IP — this lets us verify which instance the ALB is routing to when we refresh the browser.

- **EC2** → **Launch instance**

**Basic config:**

| Field | Value |
|---|---|
| Name | test-EC2-instance-1 |
| AMI | Ubuntu Server 22.04 LTS |
| Architecture | x86 |
| Instance type | t2.micro |
| Key pair | test-ALB-demo-key-pair |

**Network settings → Edit:**

| Field | Value |
|---|---|
| VPC | test-vpc |
| Subnet | test-public-subnet-1a |
| Auto-assign public IP | Enable |

**Security group — Create new:**

| Type | Protocol | Port | Source |
|---|---|---|---|
| SSH | TCP | 22 | Anywhere (0.0.0.0/0) |
| HTTP | TCP | 80 | Anywhere (0.0.0.0/0) |

> SSH allows you to log into the instance. HTTP allows the ALB and users to reach Apache on port 80.

**Storage:** 8 GB (default)

**Advanced details → User data:**

```bash
#!/bin/bash
apt-get update -y
apt-get install -y apache2
echo "<h1>Server: $(hostname) | IP: $(hostname -I)</h1>" > /var/www/html/index.html
systemctl start apache2
systemctl enable apache2
```

- Click **Launch instance**
- Wait until status shows **Running** and status checks pass `2/2`

**Verify Apache on Instance 1:**
- Copy **Public IPv4 address** → open in browser
- ✅ Should show: `Server: ip-12-0-1-xxx | IP: 12.0.1.142`
- IP starts with `12.0.1.x` confirming it's from subnet-1a

---

### Step 10 — Launch EC2 Instance 2 (Subnet 1B)

> Second instance goes in subnet-1b (eu-central-1b). Same configuration but different subnet and AZ. Apache prints a different IP (12.0.3.x) confirming this is a separate instance in a separate subnet.

- **EC2** → **Launch instance**

**Basic config:**

| Field | Value |
|---|---|
| Name | test-EC2-instance-2 |
| AMI | Ubuntu Server 22.04 LTS |
| Architecture | x86 |
| Instance type | t2.micro |
| Key pair | test-ALB-demo-key-pair |

**Network settings → Edit:**

| Field | Value |
|---|---|
| VPC | test-vpc |
| Subnet | test-public-subnet-1b |
| Auto-assign public IP | Enable |

**Security group — Create new:**

| Type | Protocol | Port | Source |
|---|---|---|---|
| SSH | TCP | 22 | Anywhere (0.0.0.0/0) |
| HTTP | TCP | 80 | Anywhere (0.0.0.0/0) |

**Advanced details → User data:**

```bash
#!/bin/bash
apt-get update -y
apt-get install -y apache2
echo "<h1>Server: $(hostname) | IP: $(hostname -I)</h1>" > /var/www/html/index.html
systemctl start apache2
systemctl enable apache2
```

- Click **Launch instance**
- Wait until status shows **Running** and `2/2` checks passed

**Verify Apache on Instance 2:**
- Copy **Public IPv4 address** → open in browser
- ✅ Should show: `Server: ip-12-0-3-xxx | IP: 12.0.3.160`
- IP starts with `12.0.3.x` confirming it's from subnet-1b

---

## 📍 PART 6 — Create Target Group

### Step 11 — Create Target Group and Register Both Instances

> A Target Group is a logical container that groups EC2 instances together. The ALB routes traffic to this group, not directly to individual instances. Health checks run against each instance in the group — unhealthy ones are automatically excluded from routing.

- **EC2** → left sidebar → **Load Balancing** → **Target Groups** → **Create target group**

**Target group config:**

| Field | Value |
|---|---|
| Target type | Instances |
| Target group name | test-EC2-ALB-demo-TG |
| Protocol | HTTP |
| Port | 80 |
| IP address type | IPv4 |
| VPC | test-vpc |
| Protocol version | HTTP1 |

**Health checks:**

| Field | Value |
|---|---|
| Health check protocol | HTTP |
| Health check path | / (root — default) |

> Health check path `/` works here because our Apache serves the homepage at the root URL. Leave all advanced health check settings at default.

- Click **Next**

**Register targets:**
- ✅ Select `test-EC2-instance-1`
- ✅ Select `test-EC2-instance-2`
- Click **Include as pending below**

> Both instances appear in the "Review targets" section on port 80

- Click **Create target group**

✅ Target group created — health status shows `unused` for now (becomes `healthy` after ALB is created)

---

## 📍 PART 7 — Create ALB Security Group

### Step 12 — Create Security Group for ALB

> The ALB needs its own security group separate from the EC2 instances. This controls what traffic is allowed INTO the load balancer from the internet. Without port 80 open here, users cannot reach the ALB even if EC2 instances are healthy.

- **VPC** → **Security Groups** → **Create security group**

| Field | Value |
|---|---|
| Name | security-group-for-ALB-demo |
| Description | Allow access from internet on port 80 |
| VPC | test-vpc |

**Inbound rules:**

| Type | Protocol | Port | Source |
|---|---|---|---|
| HTTP | TCP | 80 | 0.0.0.0/0 (anywhere) |

**Outbound rules:** leave default (all traffic)

- Click **Create security group**
- Copy the security group name/ID — needed in the next step

---

## 📍 PART 8 — Create Application Load Balancer

### Step 13 — Create ALB

> The ALB is the entry point for all user traffic. It listens on port 80, receives HTTP requests, and forwards them to the target group. It automatically skips unhealthy instances and distributes requests evenly across healthy ones.

- **EC2** → **Load Balancers** → **Create load balancer**
- Choose **Application Load Balancer** → **Create**

**Basic configuration:**

| Field | Value |
|---|---|
| Load balancer name | test-LB-EC2-demo |
| Scheme | Internet-facing |
| IP address type | IPv4 |

**Network mapping:**

| Field | Value |
|---|---|
| VPC | test-vpc |
| Subnets | ✅ eu-central-1a (test-public-subnet-1a) + ✅ eu-central-1b (test-public-subnet-1b) |

> ⚠️ Both subnets must be selected — ALB requires at least 2 AZs. Missing one will cause an error.

**Security groups:**

| Action | Value |
|---|---|
| Remove | default security group |
| Add | security-group-for-ALB-demo |

**Listeners and routing:**

| Listener | Port | Forward to |
|---|---|---|
| HTTP | 80 | test-EC2-ALB-demo-TG |

**Summary before creating:**

| Setting | Value |
|---|---|
| Type | Internet-facing |
| IP type | IPv4 |
| Security group | security-group-for-ALB-demo |
| VPC | test-vpc |
| Subnets | test-public-subnet-1a + test-public-subnet-1b |
| Target group | test-EC2-ALB-demo-TG |

- Click **Create load balancer**

✅ ALB created — state is `provisioning`

> ⚠️ Wait 2-3 minutes until state changes to **Active**

---

## 📍 PART 9 — Verify Everything is Working

### Step 14 — Check Target Group Health

> Once ALB is created, it starts sending health checks to EC2 instances. Status changes from `unused` → `healthy` if Apache is responding on port 80.

- **EC2** → **Target Groups** → click `test-EC2-ALB-demo-TG`
- **Targets tab**

✅ Both instances should show **Healthy**

If showing **Unhealthy:**
- SSH into instance and run `curl localhost:80`
- If `Connection refused` → Apache not running → run `sudo systemctl start apache2`
- Check security group has port 80 open

---

### Step 15 — Test Load Balancing via ALB DNS

> The ALB DNS name is the public endpoint users access. It is NOT an IP — it's a hostname that resolves to the ALB. Every request through this DNS gets load balanced across both EC2 instances.

- **EC2** → **Load Balancers** → click `test-LB-EC2-demo`
- Copy the **DNS name** (e.g. `test-LB-EC2-demo-xxx.eu-central-1.elb.amazonaws.com`)
- Open in browser: `http://<alb-dns-name>`

**First request:**
```
Server: ip-12-0-1-xxx | IP: 12.0.1.142
```
→ Routed to EC2 Instance 1 (subnet-1a)

**Refresh browser:**
```
Server: ip-12-0-3-xxx | IP: 12.0.3.160
```
→ Routed to EC2 Instance 2 (subnet-1b)

**Refresh again:**
```
Server: ip-12-0-1-xxx | IP: 12.0.1.142
```
→ Back to Instance 1

✅ Load balancing confirmed — requests alternate between both instances

---

## ✅ Summary — What Each Resource Does

| Resource | Purpose |
|---|---|
| VPC `12.0.0.0/16` | Isolated private network for all resources |
| Internet Gateway | Connects VPC to public internet |
| Public subnet 1a `12.0.1.0/24` | EC2 instance 1 in eu-central-1a |
| Public subnet 1b `12.0.3.0/24` | EC2 instance 2 in eu-central-1b |
| Route Table | Routes `0.0.0.0/0` to IGW — makes subnets public |
| EC2 Instance 1 | Apache web server — prints IP `12.0.1.142` |
| EC2 Instance 2 | Apache web server — prints IP `12.0.3.160` |
| EC2 Security Group | Port 22 (SSH) + Port 80 (HTTP) on instances |
| Target Group | Groups both EC2 instances — ALB routes to this |
| ALB Security Group | Port 80 open from internet to ALB |
| Application Load Balancer | Receives user traffic, distributes to target group |

---

## 📋 Traffic Flow — Step by Step

```
1. User opens browser
         → http://test-LB-EC2-demo-xxx.elb.amazonaws.com

2. DNS resolves ALB hostname → ALB public IP

3. Request reaches ALB
         → ALB security group checks: is port 80 allowed? ✅

4. ALB checks target group
         → Are instances healthy? ✅
         → Round-robin: send to Instance 1

5. Instance 1 receives request
         → EC2 security group checks: is port 80 allowed? ✅
         → Apache serves HTML page
         → Returns: IP: 12.0.1.142

6. User refreshes browser
         → ALB round-robin: send to Instance 2
         → Returns: IP: 12.0.3.160

7. Instance becomes unhealthy
         → ALB health check fails
         → ALB stops routing to that instance
         → All traffic goes to remaining healthy instance
```

---

## 🚨 Common Issues and Fixes

| Issue | Symptom | Fix |
|---|---|---|
| Both targets unhealthy | ALB shows 502 / targets show unhealthy | SSH into EC2 → `curl localhost:80` → if refused, run `sudo systemctl start apache2` |
| ALB not accessible | Browser times out on ALB DNS | Check ALB security group has port 80 inbound from 0.0.0.0/0 |
| EC2 not reachable | Target stays unhealthy | Check EC2 security group has port 80 inbound |
| Same instance every time | Load balancing not working | Clear browser cache or use different browser — some browsers stick to one IP |
| IGW state detached | No internet access | Go to IGW → Actions → Attach to VPC → select test-vpc |
| ALB stuck provisioning | State doesn't change to active | Wait 3-5 mins. If persists, check subnets are in 2 different AZs |
| Only one subnet selected | ALB creation fails | ALB requires subnets in at least 2 AZs — select both subnets |

---

## 🔑 Key Interview Questions

| Question | Answer |
|---|---|
| What is an Application Load Balancer? | Layer 7 load balancer that distributes HTTP/HTTPS traffic across multiple targets based on rules |
| What is a Target Group? | Logical group of EC2 instances the ALB routes traffic to, with health checks per instance |
| Why does ALB need subnets in 2 AZs? | For high availability — if one AZ fails, ALB still routes through the other |
| What happens when an instance is unhealthy? | ALB detects via health checks and stops routing traffic to it automatically |
| What is the difference between ALB and NLB? | ALB = Layer 7 (HTTP/HTTPS, path-based routing). NLB = Layer 4 (TCP/UDP, ultra-low latency) |
| Why does refreshing the browser show different IPs? | ALB uses round-robin — each request goes to the next healthy instance in rotation |
| Why do you need two security groups? | ALB SG controls traffic into the ALB from internet. EC2 SG controls traffic from ALB to instances |
| What is internet-facing scheme? | ALB gets a public DNS name and routes traffic from the public internet |
| What is health check path? | URL path ALB hits on each instance to check if it's responding. `/` checks the homepage |
| What is user data in EC2? | Bootstrap script that runs on instance start — used here to auto-install Apache |

---

## 🧹 Cleanup — Delete in This Order

> ⚠️ ALB costs ~$0.008/hour + EC2 costs — always delete after lab

1. **Application Load Balancer** → Delete
2. **Target Group** → Delete
3. **EC2 Instances** → Select both → **Instance state** → **Terminate**
4. **Security Groups** → Delete ALB SG and EC2 SGs
5. **Internet Gateway** → **Actions** → **Detach from VPC** → then **Delete**
6. **Route Table** → Delete custom `test-public-RT` (not the main one)
7. **Subnets** → Delete both `test-public-subnet-1a` and `test-public-subnet-1b`
8. **VPC** → Delete `test-vpc`
9. **Key Pair** → Delete `test-ALB-demo-key-pair` (optional)

---

## 🔜 Next Step — Terraform Equivalent

Every console step maps directly to a Terraform resource:

| Console Action | Terraform Resource |
|---|---|
| Create VPC | `aws_vpc` |
| Create Internet Gateway | `aws_internet_gateway` |
| Create Subnet | `aws_subnet` |
| Create Route Table | `aws_route_table` |
| Add Route | `aws_route` |
| Associate Subnet | `aws_route_table_association` |
| Create Security Group (ALB) | `aws_security_group` |
| Create Security Group (EC2) | `aws_security_group` |
| Create Key Pair | `aws_key_pair` |
| Launch EC2 Instance | `aws_instance` |
| Create Target Group | `aws_lb_target_group` |
| Register Targets | `aws_lb_target_group_attachment` |
| Create ALB | `aws_lb` |
| Create ALB Listener | `aws_lb_listener` |