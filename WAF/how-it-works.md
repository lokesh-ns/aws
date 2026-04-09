# AWS WAF — How It Works in Real Time

## What is WAF?

AWS WAF (Web Application Firewall) is a security layer that sits in front of your application and inspects every incoming HTTP/HTTPS request before it reaches your ALB, EC2, or any backend service. It evaluates requests against rules you define and either allows, blocks, or challenges them.

---

## Real Time Flow

```
User sends request
      │
      ▼
AWS WAF intercepts (before ALB)
      │
      ├── Does request match any rule?
      │
      ├── YES → apply rule action
      │              ├── Block  → 403 Forbidden returned immediately
      │              ├── Allow  → request passes through
      │              └── CAPTCHA → challenge served to browser
      │
      └── NO → apply default action (usually Allow)
                     │
                     ▼
             Request reaches ALB
                     │
                     ▼
             EC2 / Backend serves response
```

---

## What WAF Checks in Real Time

Every request that hits your ALB is instantly evaluated by WAF against these conditions:

| What WAF Checks | Example |
|---|---|
| Source IP address | Block requests from `106.51.197.44/32` |
| Geographic location | Block all traffic from a specific country |
| Request URI path | Block requests to `/admin/*` |
| HTTP headers | Block if `User-Agent` contains `bot` |
| Query string | Block if query param contains SQL injection pattern |
| Request body | Block if POST body contains XSS script |
| Rate limit | Block if same IP sends more than 100 req/5min |

---

## Real World Scenarios

### Scenario 1 — IP Block
```
Hacker at IP 106.51.x.x → sends request to ALB
WAF checks IP set → match found
Action: Block
Result: 403 Forbidden — request never reaches EC2 ❌
```

### Scenario 2 — Normal User
```
User at IP 72.21.x.x → sends request to ALB
WAF checks all rules → no match
Default action: Allow
Result: Request passes to ALB → EC2 serves page ✅
```

### Scenario 3 — Bot Detection
```
Bot sends 1000 requests in 1 minute from same IP
WAF rate-based rule triggers at 100 req/5min
Action: Block
Result: Bot throttled — legitimate users unaffected ✅
```

### Scenario 4 — CAPTCHA Challenge
```
Suspicious IP sends request
WAF matches CAPTCHA rule
Result: Browser gets puzzle → human solves → access granted ✅
        Bot cannot solve   → access denied ❌
```

---

## Key Points to Remember

- WAF evaluates **before** the request reaches ALB or EC2 — blocked traffic never touches your servers
- Rules are evaluated **top to bottom** by priority — first match wins
- **Default action** applies when no rule matches — usually set to Allow
- WAF works at **Layer 7** (HTTP) — it understands URLs, headers, body content
- **Count action** logs matching requests without blocking — useful for testing rules before enforcing
- WAF integrates with **CloudWatch** — you can see blocked requests, metrics, and sampled logs in real time

---

## WAF vs Security Group — Key Difference

| | WAF | Security Group |
|---|---|---|
| Layer | Layer 7 (HTTP/HTTPS) | Layer 4 (TCP/UDP) |
| What it inspects | URL, headers, body, IP, geo | Only IP and port |
| Can block SQL injection | ✅ Yes | ❌ No |
| Can block by country | ✅ Yes | ❌ No |
| Rate limiting | ✅ Yes | ❌ No |
| Works with | ALB, CloudFront, API Gateway | EC2, RDS, Lambda |

---

## What You Tested in the Lab

```
Step 1 → Created IP set with your laptop IP (curl -4 ifconfig.me)
Step 2 → Added Block rule → got 403 Forbidden ❌
Step 3 → Changed to Allow → Apache page loaded ✅
Step 4 → Changed to CAPTCHA → puzzle appeared in browser 🧩
```

> Every change took effect within seconds — that is WAF working in real time with no server restarts or deployments needed.