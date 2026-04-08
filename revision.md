# Revision Guide

Revise in this order — each service builds on the previous one.

## Foundation
- [ ] [IAM](IAM/README.md) — users, roles, policies (everything depends on this)
- [ ] [KMS](KMS/README.md) — encryption keys, used by almost every service
- [ ] [VPC](VPC/README.md) — networking foundation

## Compute
- [ ] [EC2](EC2/README.md)
- [ ] [ASG](ASG/README.md) — EC2 at scale
- [ ] [Lambda](Lambda/README.md) — serverless compute
- [ ] [EKS](EKS/README.md) — managed Kubernetes
- [ ] [ECS](ECS/README.md) — managed containers
- [ ] [Fargate](Fargate/README.md) — serverless containers (ECS/EKS without nodes)

## Networking
- [ ] [ALB](ALB/README.md) — load balancing
- [ ] [NAT Gateway](NAT-gateway/README.md) — private subnet internet access
- [ ] [CloudFront](CloudFront/README.md) — CDN and edge caching

## Storage
- [ ] [S3](S3/README.md) — object storage
- [ ] [EBS](EBS/README.md) — block storage for EC2

## Observability & Governance
- [ ] [CloudWatch](CloudWatch/README.md) — logs and metrics
- [ ] [CloudTrail](CloudTrail/README.md) — API audit trail

## CI/CD
- [ ] [CICD](CICD/README.md) — pipelines and automation