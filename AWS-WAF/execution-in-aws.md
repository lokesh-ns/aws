# AWS WAF (Web Application Firewall) with ALB — Console Step-by-Step Guide

## 🎯 What We'll Build

```
                        User (Browser)
                             │
                             │ http://<alb-dns-name>
                             ▼
                  ┌─────────────────────┐
                  │  AWS WAF            │
                  │  aws-waf-demo-ec2   │
                  │                     │
                  │  Rule: block-my-    │
                  │  laptop-access      │
                  │  (IP Set based)     │
                  └──────────┬──────────┘
                             │
                    ┌────────┴────────┐
                    │ Allow ✅        │ Block ❌
                    │ (other IPs)     │ (blocked IP)
                    ▼                 ▼
          Application Load Balancer  403 Forbidden
          lb-waf-demo
          internet-facing | port 80
                    │
                    ▼
             Target Group
          target-group-ec2-instances
                    │
                    ▼
            EC2 Instance
          test-ec2-aws-waf-demo
          Apache running
          (prints hostname + IP)
```

> **How it works:** WAF sits in front of the ALB and evaluates every incoming request against rules. If the request matches a block rule (e.g. your laptop IP), WAF returns 403 Forbidden before the request even reaches the ALB or EC2. If allowed, the request passes through to the ALB which forwards it to EC2.

---

## ⚠️ Before You Start

- AWS account ready
- WAF is a **regional resource** — always verify your region matches the region where your ALB is deployed
- All resources created inside `test-vpc`
- Key pair `.pem` file downloaded and `chmod 400` applied

---

## 📍 PART 1 — Create VPC

### Step 1 — Create VPC

> VPC is the isolated private network where all resources live. CIDR `12.0.0.0/16` gives ~65,000 IPs to work with.

- Search **VPC** → **Your VPCs** → **Create VPC**

| Field | Value |
|---|---|
| Resources to create | VPC only |
| Name tag | test-vpc |
| IPv4 CIDR | 12.0.0.0/16 |
| Tenancy | Default |

- Click **Create VPC**

✅ Verify: click **Your VPCs** → `test-vpc` visible with CIDR `12.0.0.0/16`

---

## 📍 PART 2 — Create Internet Gateway

### Step 2 — Create and Attach Internet Gateway

> IGW connects the VPC to the public internet. Without it, EC2 instances cannot be reached from outside and the ALB cannot receive traffic.

- **VPC dashboard** → **Internet Gateways** → **Create internet gateway**

| Field | Value |
|---|---|
| Name tag | internet-gateway-test |

- Click **Create internet gateway**
- **Actions** → **Attach to VPC** → select `test-vpc` → **Attach internet gateway**

✅ State changes from `detached` → `attached`

---

## 📍 PART 3 — Create Two Public Subnets

### Step 3 — Create Subnet 1 (1A)

> Two subnets across two AZs required — ALB needs at least 2 AZs to work. Each subnet gets its own CIDR range from the VPC's IP space.

- **Subnets** → **Create subnet** → VPC: `test-vpc`

| Field | Value |
|---|---|
| Subnet name | test-public-subnet-1a |
| Availability Zone | eu-central-1a |
| IPv4 CIDR | 12.0.1.0/24 |

---

### Step 4 — Create Subnet 2 (1B)

- Click **Add new subnet**

| Field | Value |
|---|---|
| Subnet name | test-public-subnet-1b |
| Availability Zone | eu-central-1b |
| IPv4 CIDR | 12.0.2.0/24 |

- Click **Create subnet**

✅ Two subnets created across two AZs

---

## 📍 PART 4 — Create Route Table

### Step 5 — Create and Configure Public Route Table

> Route table tells traffic where to go. `0.0.0.0/0 → IGW` makes subnets public — instances inside can reach and be reached from the internet.

- **Route Tables** → **Create route table**

| Field | Value |
|---|---|
| Name | test-public-RT |
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

## 📍 PART 5 — Create EC2 Instance

### Step 6 — Create Key Pair

> Key pair needed for SSH access. Private key downloads once — store it safely.

- **EC2** → **Key Pairs** → **Create key pair**

| Field | Value |
|---|---|
| Name | aws-waf-ec2-demo |
| Type | RSA |
| Format | .pem |

```bash
chmod 400 aws-waf-ec2-demo.pem
```

---

### Step 7 — Launch EC2 Instance

> Apache is installed via user data and prints hostname + IP so we can verify requests are reaching the EC2 after passing through WAF and ALB.

- **EC2** → **Launch instance**

**Basic config:**

| Field | Value |
|---|---|
| Name | test-ec2-aws-waf-demo |
| AMI | Ubuntu Server 22.04 LTS |
| Instance type | t2.micro |
| Key pair | aws-waf-ec2-demo |

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
- Wait for status: **Running** + `2/2 checks passed`

**Verify Apache:**
- Copy **Public IPv4** → open in browser
- ✅ Should show: `Server: ip-12-0-1-xxx | IP: 12.0.1.xxx`

---

## 📍 PART 6 — Create Target Group

### Step 8 — Create Target Group and Register EC2

> Target group is the logical container ALB routes traffic to. EC2 instance is registered here — ALB points to the group, not directly to instances.

- **EC2** → **Load Balancing** → **Target Groups** → **Create target group**

| Field | Value |
|---|---|
| Target type | Instances |
| Target group name | target-group-ec2-instances |
| Protocol | HTTP |
| Port | 80 |
| VPC | test-vpc |
| Protocol version | HTTP1 |

- Health checks: leave defaults (path: `/`)
- Click **Next**

**Register targets:**
- ✅ Select `test-ec2-aws-waf-demo`
- Click **Include as pending below**
- Click **Create target group**

✅ Target group created with EC2 instance registered

---

## 📍 PART 7 — Create ALB Security Group and ALB

### Step 9 — Create ALB Security Group

> ALB needs its own security group to accept HTTP traffic from the internet. Without port 80 open here, the ALB cannot receive any requests.

- **VPC** → **Security Groups** → **Create security group**

| Field | Value |
|---|---|
| Name | security-group-aws-waf-demo |
| Description | Allow SSH and HTTP requests |
| VPC | test-vpc |

**Inbound rules:**

| Type | Protocol | Port | Source |
|---|---|---|---|
| SSH | TCP | 22 | Anywhere (0.0.0.0/0) |
| HTTP | TCP | 80 | Anywhere (0.0.0.0/0) |

> ⚠️ Make sure VPC is set to `test-vpc` not the default VPC

- Click **Create security group**

---

### Step 10 — Create Application Load Balancer

> ALB receives user requests and forwards to the target group. WAF will be attached to this ALB in the next part.

- **EC2** → **Load Balancers** → **Create load balancer**
- Choose **Application Load Balancer** → **Create**

| Field | Value |
|---|---|
| Name | lb-waf-demo |
| Scheme | Internet-facing |
| IP address type | IPv4 |
| VPC | test-vpc |
| Subnets | ✅ test-public-subnet-1a + ✅ test-public-subnet-1b |
| Security groups | security-group-aws-waf-demo (remove default) |

**Listeners and routing:**

| Listener | Port | Forward to |
|---|---|---|
| HTTP | 80 | target-group-ec2-instances |

- Click **Create load balancer**

> ⚠️ Wait 2-3 minutes until state changes from `provisioning` → `active`

**Verify ALB:**
- Copy **DNS name** → open in browser
- ✅ Apache homepage loads with EC2 hostname and IP

---

## 📍 PART 8 — Create AWS WAF

### Step 11 — Navigate to WAF Console

> WAF is a global service but rules are applied regionally. You must select the correct region matching your ALB before creating anything.

- Search **WAF** in console → click **WAF & Shield**
- ⚠️ **Change region to match your ALB** (e.g. Europe Frankfurt / eu-central-1)
- Left sidebar → **Web ACLs** → **Create web ACL**

---

### Step 12 — Configure Web ACL

> Web ACL (Access Control List) is the main WAF resource. It contains rules and is attached to the ALB. Every request through the ALB is evaluated against the rules in this ACL.

| Field | Value |
|---|---|
| Name | aws-waf-demo-ec2 |
| Description | WAF for EC2 demo with ALB |
| Resource type | Regional resources |
| Region | eu-central-1 (Europe Frankfurt) |

**Add AWS resource:**
- Click **Add AWS resources**
- Select **Application Load Balancer**
- Select `lb-waf-demo`
- Click **Add**

- Click **Next**

---

## 📍 PART 9 — Create IP Set

### Step 13 — Create IP Set with Your Laptop IP

> Before adding a rule to the WAF, you need to create an IP Set — a logical container of IP addresses to block or allow. Here we add your laptop's public IP so WAF knows which IP to block.

- Left sidebar (in a new tab) → **IP sets** → **Create IP set**

| Field | Value |
|---|---|
| IP set name | my-laptop-ip |
| Region | eu-central-1 (same as WAF) |
| IP version | IPv4 |

**Find your public IP:**

> ⚠️ Always use your **public IPv4** — not your private/local IP (like `192.168.x.x` or `10.x.x.x`). Private IPs are not routable on the internet and WAF will never see them — the rule simply won't work.

**On Mac — run this in terminal:**
```bash
curl -4 ifconfig.me
# Output example: 106.51.197.44
```

> The `-4` flag forces IPv4. Without it you might get an IPv6 address which won't match the IPv4 IP set.

**On Windows — run this in Command Prompt:**
```cmd
curl -4 ifconfig.me
```

**Alternative — browser method:**
- Google: `what is my IP address`
- Copy the IP shown (e.g. `106.51.197.44`)

> ⚠️ Do NOT use `ifconfig` or `ipconfig` — those show your **private** local network IP, not the public IP that AWS WAF sees.

**Enter IP with CIDR notation:**
```
106.51.197.44/32
```

> `/32` means exactly this one IP address. `/24` would mean all 256 IPs in that subnet range.

- Click **Create IP set**

✅ IP set `my-laptop-ip` created with your laptop's IP

---

## 📍 PART 10 — Add Rule to WAF

### Step 14 — Add IP Set Rule to Block Your Laptop

> Rules are the heart of WAF. Each rule defines a condition (what to match) and an action (block, allow, or CAPTCHA). Here we match by IP set and block those requests.

- Go back to Web ACL creation → **Add rules and rule groups**
- Click **Add rules** → **Add my own rules and rule groups**
- Select **IP set**

| Field | Value |
|---|---|
| Rule name | block-my-laptop-access |
| IP set | my-laptop-ip |
| IP address to use | Source IP address |
| Action | **Block** |

- Click **Add rule**
- Click **Next**

**Set rule priority:**
- Leave default order
- Click **Next**

**CloudWatch metrics:**
- Leave default metric name
- Click **Next**

**Review and create:**
- Click **Create web ACL**

> ⚠️ Wait 1-2 minutes for WAF to be created and attached to the ALB

✅ WAF created with block rule for your laptop IP

---

## 📍 PART 11 — Test WAF Rules

### Step 15 — Verify WAF is Blocking Your Request

> Now test if WAF is correctly blocking traffic from your laptop IP. The ALB DNS remains the same — WAF intercepts before the request reaches the ALB.

- Go to **EC2** → **Load Balancers** → `lb-waf-demo`
- Copy the **DNS name**
- Open in browser: `http://<alb-dns-name>`

✅ You should see:
```
403 Forbidden
```

> WAF recognized your laptop IP matched the block rule and returned 403 before the request reached EC2.

---

### Step 16 — Verify in WAF Console

- **WAF & Shield** → **Web ACLs** → click `aws-waf-demo-ec2`
- You can see **blocked requests** showing up in the sampled requests section

---

### Step 17 — Change Rule Action to Allow

> Now test the Allow action — changing the rule action from Block to Allow should immediately restore access.

- **Web ACLs** → `aws-waf-demo-ec2` → **Rules tab**
- Click the rule `block-my-laptop-access` → **Edit**
- Change **Action** from `Block` → `Allow`
- Click **Save rule**
- Click **Save** on the Web ACL page as well

**Verify:**
- Refresh browser with ALB DNS
- ✅ Apache homepage loads again — IP and hostname visible

---

### Step 18 — Test CAPTCHA Action

> WAF also supports CAPTCHA — instead of blocking outright, it challenges the user to prove they are human. Useful for bot protection.

- **Web ACLs** → `aws-waf-demo-ec2` → **Rules tab**
- Click the rule → **Edit**
- Change **Action** to `CAPTCHA`
- Click **Save rule** → **Save**

**Verify:**
- Refresh browser with ALB DNS
- ✅ CAPTCHA puzzle appears — "Please confirm you are human"
- Solving it grants access; failing keeps you blocked

---

## ✅ Summary — What Each Resource Does

| Resource | Purpose |
|---|---|
| VPC `12.0.0.0/16` | Isolated private network |
| Internet Gateway | Public internet access for VPC |
| Public subnet 1a `12.0.1.0/24` | EC2 instance location |
| Public subnet 1b `12.0.2.0/24` | Required second AZ for ALB |
| Route Table | Routes `0.0.0.0/0` to IGW |
| EC2 Instance | Apache web server — serves homepage |
| EC2 Security Group | Port 22 + 80 open on instance |
| Target Group | Registers EC2 — ALB routes to this |
| ALB Security Group | Port 80 + 22 open on load balancer |
| Application Load Balancer | Receives traffic, forwards to target group |
| IP Set | Contains your laptop IP to block |
| WAF Web ACL | Evaluates requests, enforces rules, attached to ALB |

---

## 📋 Traffic Flow — Step by Step

```
BLOCKED scenario (your laptop IP):
────────────────────────────────────
User (laptop) → http://<alb-dns>
    │
    ▼
WAF evaluates request
    │
    ├── IP matches my-laptop-ip set?  YES
    │
    ▼
Action: Block
    │
    ▼
403 Forbidden returned ❌
(request never reaches ALB or EC2)


ALLOWED scenario (other IPs):
──────────────────────────────
User (other IP) → http://<alb-dns>
    │
    ▼
WAF evaluates request
    │
    ├── IP matches block rule?  NO
    │
    ▼
Default action: Allow
    │
    ▼
Request passes to ALB
    │
    ▼
ALB forwards to target group
    │
    ▼
EC2 instance serves Apache page ✅


CAPTCHA scenario:
──────────────────
User → http://<alb-dns>
    │
    ▼
WAF evaluates → IP matches CAPTCHA rule
    │
    ▼
CAPTCHA challenge served to browser
    │
    ├── User solves → request passes through ✅
    └── User fails  → request blocked ❌
```

---

## 🔑 WAF Concepts Explained

**Web ACL (Access Control List)** — the main WAF resource. Contains a list of rules and is attached to an ALB, CloudFront, or API Gateway. Every request is evaluated against rules top-down.

**Rule** — a condition + action pair. Condition defines what to match (IP, header, path, body). Action defines what to do (allow, block, CAPTCHA, count).

**IP Set** — a reusable list of IP addresses or CIDR ranges. Used in rules to match traffic by source IP. You can add/remove IPs from the set without changing the rule itself.

**Rule Priority** — rules are evaluated in order. Lower number = evaluated first. If a request matches rule 1 → that action is taken and no further rules are checked.

**Default Action** — what WAF does if NO rules match. Usually set to `Allow` so only specifically listed IPs/patterns are blocked.

**Count action** — instead of blocking, WAF counts matching requests and logs them. Useful for testing rules before enforcing them.

---

## 🚨 Common Issues and Fixes

| Issue | Symptom | Fix |
|---|---|---|
| WAF not blocking | 200 OK still returned | Check WAF is attached to ALB — go to Web ACL → Associated AWS resources |
| Wrong region for WAF | No ALBs visible to attach | Change region in WAF console to match your ALB region |
| 403 after changing to Allow | Still getting blocked | Click Save on the rule AND Save on the Web ACL — both saves required |
| IP set wrong format | Creation fails | Always suffix IP with /32 for a single IP (e.g. `103.x.x.x/32`) |
| ALB not visible in WAF | Cannot attach resource | Verify ALB and WAF are in the same region |
| CAPTCHA not showing | Getting 403 instead | Ensure browser supports JavaScript — CAPTCHA requires JS |

---

## 🔑 Key Interview Questions

| Question | Answer |
|---|---|
| What is AWS WAF? | Web Application Firewall that filters HTTP/HTTPS traffic based on rules before it reaches your application |
| What is a Web ACL? | Main WAF resource — a set of rules attached to ALB/CloudFront that evaluates every incoming request |
| What is an IP Set? | Reusable list of IPs or CIDR ranges used in WAF rules to match traffic by source IP |
| How does WAF integrate with ALB? | WAF Web ACL is attached to the ALB — every request to the ALB is first evaluated by WAF |
| What are WAF actions? | Allow, Block, CAPTCHA, Count — Count is useful for testing rules without enforcing them |
| What is /32 in CIDR notation? | Represents exactly one IP address — /24 would be 256 IPs, /16 would be 65,536 IPs |
| Difference between WAF block and security group deny? | WAF = Layer 7, can match on URL paths, headers, body, IPs. SG = Layer 4, only IP and port |
| What is WAF rule priority? | Order in which rules are evaluated — lower number = higher priority, first match wins |
| Can WAF protect against SQL injection? | Yes — AWS Managed Rules include SQL injection, XSS, and other OWASP top 10 protections |
| What is the default action in WAF? | What happens when no rule matches — usually Allow so only explicitly blocked traffic is stopped |

---


## 🔜 Next Step — Terraform Equivalent

| Console Action | Terraform Resource |
|---|---|
| Create VPC | `aws_vpc` |
| Create Internet Gateway | `aws_internet_gateway` |
| Create Subnet | `aws_subnet` |
| Create Route Table + Route | `aws_route_table` + `aws_route` |
| Associate Subnet | `aws_route_table_association` |
| Create Security Group | `aws_security_group` |
| Launch EC2 | `aws_instance` |
| Create Target Group | `aws_lb_target_group` |
| Register EC2 to Target Group | `aws_lb_target_group_attachment` |
| Create ALB | `aws_lb` |
| Create ALB Listener | `aws_lb_listener` |
| Create IP Set | `aws_wafv2_ip_set` |
| Create WAF Web ACL | `aws_wafv2_web_acl` |
| Attach WAF to ALB | `aws_wafv2_web_acl_association` |