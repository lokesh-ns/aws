# Revision Guide

Revise in this order — each service builds on the previous one.

## Foundation
- [ ] [IAM](IAM/README.md) — users, roles, policies (everything depends on this)
- [ ] [KMS](KMS/README.md) — encryption keys, used by almost every service
- [ ] [VPC](VPC/README.md) — networking foundation
- [ ] [VPC Peering](VPC-Peering/README.md) — cross-VPC connectivity

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

## Interview Preparation
- [ ] [Overview](interview-prep/README.md)
- [ ] [EC2](interview-prep/ec2.md)
- [ ] [Lambda](interview-prep/lambda.md)
- [ ] [ASG](interview-prep/asg.md)
- [ ] [EKS](interview-prep/eks.md)
- [ ] [ECS & Fargate](interview-prep/ecs-fargate.md)
- [ ] [VPC](interview-prep/vpc.md)
- [ ] [ALB](interview-prep/alb.md)
- [ ] [NAT Gateway](interview-prep/nat-gateway.md)
- [ ] [CloudFront](interview-prep/cloudfront.md)
- [ ] [S3](interview-prep/s3.md)
- [ ] [EBS](interview-prep/ebs.md)
- [ ] [IAM](interview-prep/iam.md)
- [ ] [KMS](interview-prep/kms.md)
- [ ] [CloudWatch](interview-prep/cloudwatch.md)
- [ ] [CloudTrail](interview-prep/cloudtrail.md)
- [ ] [DevOps & CI/CD](interview-prep/devops-cicd.md)
- [ ] [Architecture & Scenarios](interview-prep/architecture-scenarios.md)
- [ ] [Behavioral](interview-prep/behavioral.md)