# AWS Application Load Balancer — Brief Overview & How It Works in Real Time

## What is an Application Load Balancer?

An ALB is a managed AWS service that sits in front of your application and distributes incoming HTTP/HTTPS traffic across multiple targets (EC2 instances, containers, Lambda functions) based on rules you define. It operates at **Layer 7** of the OSI model — meaning it understands the actual content of HTTP requests, not just raw TCP packets.

---

## How it Works in Real Time

Think of the ALB as a **smart traffic cop** standing at the entrance of your application.

### Without ALB

```
User → directly hits EC2 instance
         ↓
  if that instance crashes → app is down ❌
  if traffic spikes → one instance overloaded ❌
  no way to route /api vs /web differently ❌
```

### With ALB

```
User → hits ALB DNS
         ↓
  ALB checks which instances are healthy
         ↓
  routes request to the best available instance
         ↓
  if one crashes → automatically stops sending to it
  if traffic spikes → spreads load across all instances
```

---

## Real Time Flow — What Happens in Milliseconds

When a user opens `http://your-alb-dns.amazonaws.com`:

```
1. DNS Resolution (< 1ms)
   Browser asks DNS: what IP is this ALB?
   DNS returns ALB's public IP

2. Request reaches ALB (network latency)
   ALB security group checks: is port 80 allowed? ✅
   If not → request dropped immediately

3. ALB evaluates listener rules (< 1ms)
   Listener on port 80 says:
   → forward to target group: my-target-group

4. ALB picks a healthy target (< 1ms)
   Checks internal health table:
   EC2-1 → healthy ✅
   EC2-2 → healthy ✅
   Round-robin → send to EC2-1 this time

5. Request forwarded to EC2-1 (network latency)
   EC2 security group checks: is port 80 allowed? ✅
   Apache receives and processes the request

6. Response flows back through ALB to user
   Total time: typically 10–50ms end to end
```

---

## Key Concepts in Plain English

**Listener** — the "ear" of the ALB. It sits on a port (80 or 443) and listens for incoming requests. When a request arrives it checks its rules to decide where to send it.

**Rules** — conditions the ALB evaluates. For example:
```
if path = /api/*    → send to backend-servers target group
if path = /images/* → send to static-servers target group
if path = /*        → send to default target group
```

**Target Group** — the "team" of servers the ALB sends traffic to. Each target group has its own health check running continuously.

**Health Check** — ALB silently pings each EC2 instance every 30 seconds (default) on the configured path. If an instance fails 2 consecutive checks it is marked unhealthy and removed from rotation automatically — no human needed.

**Round Robin** — default algorithm. Request 1 → EC2-1, Request 2 → EC2-2, Request 3 → EC2-1, and so on evenly across all healthy instances.

---

## Real World Scenarios Where ALB Helps

### Scenario 1 — Instance Crashes at 2am

```
EC2-1 crashes        at 2:00:00am
Health check fails   at 2:00:30am
EC2-1 marked unhealthy at 2:01:00am
All traffic routes to EC2-2 automatically
No one is paged, no downtime for users ✅
```

### Scenario 2 — Traffic Spike (Black Friday)

```
Normal:  100 req/sec  → 2 instances handle it
Spike:   10,000 req/sec → ALB + ASG spins up 20 more instances
ALB automatically distributes across all 22 instances
Spike ends → ASG scales back down, ALB adjusts routing
```

### Scenario 3 — Microservices Routing

```
https://myapp.com/api/users   → users-service target group
https://myapp.com/api/orders  → orders-service target group
https://myapp.com/            → frontend target group

One domain, one ALB, three different backend services ✅
```

---

## ALB vs NLB — When to Use Which

| | ALB | NLB |
|---|---|---|
| OSI Layer | Layer 7 (HTTP/HTTPS) | Layer 4 (TCP/UDP) |
| Understands | URLs, headers, cookies | Just IP and port |
| Use case | Web apps, APIs, microservices | Gaming, IoT, real-time streaming |
| Latency | ~1-5ms overhead | Ultra low ~100 microseconds |
| Path-based routing | ✅ Yes | ❌ No |
| Cost | Slightly higher | Slightly lower |

> **Simple rule:** If your app speaks HTTP/HTTPS → use ALB. If you need raw TCP speed → use NLB.

---

## What You Verified in Your Lab

When you refreshed the browser and saw the IP change between `12.0.1.142` and `12.0.3.160` — that was the ALB round-robin working in real time. Each refresh was a new HTTP request and the ALB was sending alternate requests to each instance exactly as designed.

```
Refresh 1 → ALB → EC2-1 → IP: 12.0.1.142
Refresh 2 → ALB → EC2-2 → IP: 12.0.3.160
Refresh 3 → ALB → EC2-1 → IP: 12.0.1.142
Refresh 4 → ALB → EC2-2 → IP: 12.0.3.160
```

---

## Interview Quick Reference

| Question | Answer |
|---|---|
| What layer does ALB operate at? | Layer 7 — Application layer (HTTP/HTTPS) |
| What is a listener? | Port on the ALB that receives incoming requests and forwards based on rules |
| What is a target group? | Logical group of EC2 instances ALB routes traffic to with health checks |
| What is round-robin? | Default routing — requests distributed evenly across healthy instances in order |
| How does ALB handle a failed instance? | Health check detects failure → removes instance from rotation automatically |
| What is path-based routing? | Route `/api/*` to one target group and `/web/*` to another using the same ALB |
| Why two security groups? | ALB SG controls internet → ALB traffic. EC2 SG controls ALB → instance traffic |
| What is health check grace period? | Time ALB waits before checking a new instance — allows app to fully start up |
| Can ALB route to Lambda? | Yes — ALB supports EC2, ECS containers, IP addresses, and Lambda as targets |
| What is sticky session? | ALB sends same user's requests to same instance using a cookie — useful for session data |