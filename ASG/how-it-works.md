# How it works

An Auto Scaling Group (ASG) automatically manages a fleet of EC2 instances,
scaling out when demand increases and scaling in when demand drops — ensuring
your application stays available without over-provisioning.

## Real World Flow

In production, an ASG sits behind an ALB. When traffic spikes, the ALB health
checks start seeing increased request load. CloudWatch picks up the CPU or
request count metric breaching the threshold, triggers a scaling policy, and
the ASG launches new EC2 instances into the target group. The ALB automatically
starts routing traffic to them. When traffic drops, the reverse happens —
instances are terminated gracefully, with the ALB draining connections first
before the ASG removes them.

## Key Concepts

- **Launch Template** — defines what each instance looks like: AMI, instance
  type, key pair, security groups, user data script
- **Desired / Min / Max** — desired is your current target count, min/max are
  the hard boundaries the ASG will never cross
- **Scaling Policies** — three types in practice:
  - *Target Tracking* — most common, e.g. keep average CPU at 50%
  - *Step Scaling* — scale by fixed increments at defined thresholds
  - *Scheduled Scaling* — pre-planned scaling for known traffic patterns
    (e.g. scale up every weekday at 9am)
- **Health Checks** — ASG can use EC2 status checks or ALB health checks.
  ALB-based is preferred in production — if an instance fails the ALB check,
  ASG terminates and replaces it automatically
- **Cooldown Period** — prevents ASG from launching or terminating instances
  too rapidly back-to-back after a scaling event

## In a Banking Context

At ANZ-style environments, ASGs are typically used with:
- Golden AMIs baked via a CI/CD pipeline (no config drift at launch)
- User data scripts pulling secrets from Secrets Manager at boot
- Multi-AZ deployment across at least 2 subnets for fault tolerance
- ALB target group attachment for zero-downtime rolling replacements
- CloudWatch alarms feeding scaling policies with SNS notifications to ops teams