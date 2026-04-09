# How it works

An Auto Scaling Group (ASG) automatically manages a fleet of EC2 instances,
scaling out when demand increases and scaling in when demand drops.

## Real World Flow

- Traffic hits the ALB, which routes requests to EC2 instances in the ASG
- CloudWatch monitors metrics — CPU, memory, or request count
- When a metric breaches the threshold, a scaling policy triggers
- ASG launches new EC2 instances and registers them with the ALB target group
- ALB starts routing traffic to new instances once health checks pass
- When traffic drops, ASG terminates excess instances after ALB drains connections

## Key Concepts

- **Launch Template** — defines AMI, instance type, key pair, security groups,
  and user data script for every instance the ASG launches
- **Desired / Min / Max** — desired is the current target count, min and max
  are the hard boundaries ASG will never cross
- **Target Tracking** — most common policy, e.g. keep average CPU at 50%
- **Step Scaling** — scale by fixed increments at defined thresholds
- **Scheduled Scaling** — pre-planned scaling for known traffic patterns
- **Health Checks** — ALB-based health checks preferred over EC2 checks in
  production; failed instances are terminated and replaced automatically
- **Cooldown Period** — prevents rapid back-to-back scaling events after a
  trigger

## In Production

- ASGs are deployed across multiple AZs for fault tolerance
- Instances are launched from golden AMIs baked via CI/CD pipeline
- Secrets pulled from Secrets Manager via user data at boot time
- CloudWatch alarms feed scaling policies with SNS alerts to ops teams
- ALB connection draining ensures zero downtime during scale-in