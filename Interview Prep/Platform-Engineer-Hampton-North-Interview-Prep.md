# Platform Engineer (Hampton North - Remote) - Interview Preparation Guide

**Role**: Platform Engineer — Design infrastructure, build backend services, own AWS, support product engineering teams
**Company**: Hampton North (high-growth startup, remote, globally distributed)
**Tech Stack**: Terraform, TypeScript/NestJS/Node.js, MongoDB, PostgreSQL, AWS (EC2, ECS, RDS, S3, CloudWatch), IaC, CI/CD
**Prepared by**: Davonte Watkins
**Last Updated**: March 2026

---

## Table of Contents
- [Requirements Mapping](#requirements-mapping)
- [Terraform Infrastructure Design](#terraform-infrastructure-design)
- [AWS Infrastructure (EC2, ECS, RDS, S3, CloudWatch)](#aws-infrastructure-ec2-ecs-rds-s3-cloudwatch)
- [Backend Services (TypeScript/NestJS/Node.js)](#backend-services-typescriptnestjsnodejs)
- [MongoDB & PostgreSQL Data Stores](#mongodb--postgresql-data-stores)
- [Reusable Modules & Internal Tooling](#reusable-modules--internal-tooling)
- [Design Through Deployment to Production Support](#design-through-deployment-to-production-support)
- [Monitoring & Observability](#monitoring--observability)
- [Security & Automation](#security--automation)
- [Outage Response & Rollback](#outage-response--rollback)
- [Async Collaboration on Distributed Teams](#async-collaboration-on-distributed-teams)
- [Startup / Fast-Paced Environment](#startup--fast-paced-environment)
- [Scenario-Based Questions](#scenario-based-questions)
- [STAR Stories for This Role](#star-stories-for-this-role)

---

## Requirements Mapping

| Requirement | My Direct Experience |
|-------------|---------------------|
| Scalable infrastructure with Terraform | 10+ years IaC — Terraform/Terragrunt modules for AKS, AWS (VPC, EC2, S3, Route53), Azure; DRY patterns, remote state, drift detection |
| Backend services (TypeScript/NestJS/Node.js) | Built Python/Bash automation services; supported Node.js and Java microservices on Kubernetes/OpenShift; understand REST API design, service-oriented architecture |
| AWS (EC2, ECS, RDS, S3, CloudWatch) | Deployed AWS infrastructure at Keysight (EC2, VPC, S3, Route53, CloudWatch), Fineos (AWS + OpenShift hybrid), Bank of America (AWS + OpenShift) |
| MongoDB & Postgres | Built and managed MongoDB clusters at Navy Federal (replica sets, indexing, performance tuning, backups); PostgreSQL at Fineos (HIPAA compliance queries) |
| Reusable modules & internal tooling | Built IDPs at Fineos (50% faster onboarding), reusable Terraform modules, pipeline template libraries, self-service tooling |
| Design through deployment to production | End-to-end ownership at every role — architecture, IaC, CI/CD, deploy, monitor, incident response |
| Globally distributed remote team | Fully remote at Navy Federal, Fineos — async communication, Confluence docs, Slack, cross-timezone collaboration |
| Cloud-native frameworks | Kubernetes (AKS/EKS/OpenShift), Docker, Helm, ArgoCD, microservices, service mesh, API gateways |
| IaC, automation, platform improvements | Core of my career — Terraform, Ansible, Packer, Python automation, ephemeral agents, Gen AI tooling |
| Proactive communicator | Flagged AKS capacity risks proactively, presented alternatives to unrealistic timelines, led toolchain assessments |

---

## Terraform Infrastructure Design

### Q: How do you design scalable, reproducible infrastructure with Terraform?

**A:** My approach at Navy Federal and across roles:

**Module Architecture:**
```
terraform/
├── modules/
│   ├── networking/          # VPC, subnets, security groups, NAT GW
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── compute/             # EC2, ECS, ASG, launch templates
│   ├── database/            # RDS (Postgres), DocumentDB (MongoDB-compat)
│   ├── storage/             # S3 buckets, lifecycle policies
│   ├── monitoring/          # CloudWatch dashboards, alarms, SNS
│   └── security/            # IAM roles, KMS keys, security groups
├── environments/
│   ├── dev/
│   │   ├── main.tf          # Calls modules with dev-specific vars
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   ├── staging/
│   └── prod/
└── terragrunt.hcl           # DRY config with Terragrunt
```

**Key Patterns I Enforce:**
```hcl
# 1. Remote state with locking
terraform {
  backend "s3" {
    bucket         = "hampton-terraform-state"
    key            = "prod/platform/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}

# 2. Reusable VPC module
module "vpc" {
  source = "../../modules/networking"

  vpc_cidr           = var.vpc_cidr
  availability_zones = var.azs
  private_subnets    = var.private_subnet_cidrs
  public_subnets     = var.public_subnet_cidrs
  enable_nat_gateway = true
  single_nat_gateway = var.environment == "dev" ? true : false  # Cost optimization

  tags = merge(var.common_tags, {
    Environment = var.environment
  })
}

# 3. ECS cluster with autoscaling
module "ecs" {
  source = "../../modules/compute"

  cluster_name     = "${var.project}-${var.environment}"
  vpc_id           = module.vpc.vpc_id
  private_subnets  = module.vpc.private_subnet_ids
  desired_count    = var.environment == "prod" ? 3 : 1
  cpu              = 512
  memory           = 1024
  container_image  = "${var.ecr_repo}:${var.image_tag}"
  container_port   = 3000

  environment_variables = {
    NODE_ENV     = var.environment
    DATABASE_URL = "postgresql://${module.rds.endpoint}:5432/${var.db_name}"
    MONGO_URI    = module.documentdb.connection_string
  }
}
```

**Security Emphasis:**
- All S3 buckets: encryption at rest (SSE-S3 or KMS), public access blocked, versioning enabled
- All RDS: encryption at rest, TLS in transit, private subnet only (no public access)
- IAM: Least-privilege roles per service, no wildcard permissions
- Security groups: Minimal ingress rules, deny by default
- KMS: Customer-managed keys for sensitive data encryption
- Secrets: AWS Secrets Manager integrated via data sources (never hardcoded)

### Q: How do you handle Terraform drift and state management?

**A:**
- **Scheduled drift detection**: CI pipeline runs `terraform plan` nightly — alerts on unexpected changes
- **State locking**: DynamoDB table prevents concurrent modifications
- **Import brownfield resources**: `terraform import` for existing resources not yet managed
- **State surgery**: `terraform state mv` for refactoring modules without destroying/recreating
- **Workspace separation**: Separate state files per environment to prevent cross-env impact
- **Rollback**: Git revert the change -> pipeline re-applies previous known-good state

---

## AWS Infrastructure (EC2, ECS, RDS, S3, CloudWatch)

### Q: How do you own and optimize AWS infrastructure?

**A:**

**EC2:**
- Right-sizing with CloudWatch CPU/memory metrics — downsize over-provisioned instances
- Reserved Instances for steady-state workloads, Spot Instances for batch/CI jobs
- Launch Templates with hardened AMIs (Packer-built, CIS benchmarked)
- Auto Scaling Groups with target tracking policies

**ECS (Fargate):**
```hcl
# ECS Service with auto-scaling
resource "aws_ecs_service" "api" {
  name            = "backend-api"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.api.arn
  desired_count   = 3
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = var.private_subnets
    security_groups  = [aws_security_group.ecs.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.api.arn
    container_name   = "backend-api"
    container_port   = 3000
  }
}

# Auto-scaling based on CPU
resource "aws_appautoscaling_target" "ecs" {
  max_capacity       = 10
  min_capacity       = 3
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.api.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "cpu" {
  name               = "cpu-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value = 70.0
  }
}
```

**RDS (PostgreSQL):**
- Multi-AZ for production (automatic failover)
- Read replicas for read-heavy workloads
- Automated backups with 7-day retention (prod) / 1-day (dev)
- Performance Insights enabled for query analysis
- Parameter groups tuned for workload (connection limits, shared_buffers, work_mem)

**S3:**
- Lifecycle policies: transition to IA after 30 days, Glacier after 90 days
- Versioning + MFA Delete for critical buckets
- Cross-region replication for DR
- Access logging enabled, bucket policies restricting to VPC endpoints

**CloudWatch:**
- Custom dashboards per service (CPU, memory, request count, error rate, latency)
- Composite alarms combining multiple metrics
- Log Insights queries for error analysis
- Anomaly detection for baseline-based alerting

### Q: How do you monitor AWS infrastructure?

**A:**

| Resource | Key Metrics | Alert Threshold |
|----------|------------|-----------------|
| EC2/ECS | CPUUtilization, MemoryUtilization | >80% sustained 5min |
| RDS | FreeableMemory, CPUUtilization, ReadLatency, Connections | Memory <500MB, CPU >75%, latency >100ms |
| S3 | BucketSizeBytes, NumberOfObjects, 4xxErrors | Unusual growth, error spike |
| ALB | TargetResponseTime, HTTPCode_Target_5XX, HealthyHostCount | p95 >2s, 5xx >1%, healthy <2 |
| ECS | RunningTaskCount, DesiredTaskCount | Running < Desired for >5min |

```python
# CloudWatch alarm example (Terraform)
resource "aws_cloudwatch_metric_alarm" "api_errors" {
  alarm_name          = "backend-api-5xx-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "HTTPCode_Target_5XX_Count"
  namespace           = "AWS/ApplicationELB"
  period              = 300
  statistic           = "Sum"
  threshold           = 10
  alarm_description   = "API 5xx errors above threshold"
  alarm_actions       = [aws_sns_topic.alerts.arn]
}
```

---

## Backend Services (TypeScript/NestJS/Node.js)

### Q: How do you build and support backend services?

**A:** While my primary coding languages are Python, Bash, and Groovy, I have strong experience supporting and deploying backend services and understand the full lifecycle:

**What I bring to NestJS/Node.js:**
- **Containerization**: I Dockerize Node.js applications with multi-stage builds, non-root users, and optimized layers
- **CI/CD**: Build pipelines for `npm install` -> `npm test` -> `npm run build` -> Docker build -> push to ECR -> deploy to ECS
- **Infrastructure**: Provision all supporting infra (ECS, ALB, RDS, MongoDB, S3, IAM roles, security groups)
- **Monitoring**: CloudWatch logs, custom metrics, X-Ray distributed tracing
- **Debugging production**: Log analysis, container exec for troubleshooting, resource monitoring

**Dockerfile for NestJS:**
```dockerfile
# Multi-stage build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force
COPY . .
RUN npm run build

FROM node:20-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./
USER appuser
EXPOSE 3000
HEALTHCHECK CMD wget -qO- http://localhost:3000/health || exit 1
CMD ["node", "dist/main"]
```

**ECS Task Definition for Node.js:**
```json
{
  "containerDefinitions": [{
    "name": "backend-api",
    "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/backend-api:latest",
    "portMappings": [{ "containerPort": 3000 }],
    "environment": [
      { "name": "NODE_ENV", "value": "production" },
      { "name": "PORT", "value": "3000" }
    ],
    "secrets": [
      { "name": "DATABASE_URL", "valueFrom": "arn:aws:secretsmanager:us-east-1:123:secret:db-url" },
      { "name": "MONGO_URI", "valueFrom": "arn:aws:secretsmanager:us-east-1:123:secret:mongo-uri" }
    ],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/backend-api",
        "awslogs-region": "us-east-1",
        "awslogs-stream-prefix": "api"
      }
    },
    "healthCheck": {
      "command": ["CMD-SHELL", "wget -qO- http://localhost:3000/health || exit 1"],
      "interval": 30,
      "timeout": 5,
      "retries": 3
    }
  }],
  "cpu": "512",
  "memory": "1024",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"]
}
```

### Q: How do you support product engineering teams?

**A:** This is platform engineering — my specialty:

1. **Self-service pipeline templates**: Reusable CI/CD templates devs plug into (GitHub Actions workflows, Jenkinsfiles)
2. **Infrastructure modules**: Terraform modules devs use to provision their own environments (`terraform apply` with sensible defaults)
3. **Internal tooling**: CLI tools and scripts that automate common tasks (create new service, provision DB, set up monitoring)
4. **Documentation**: Confluence/GitHub wiki with getting-started guides, architecture decisions, runbooks
5. **Developer experience**: Fast feedback loops — build times <5 min, deploy times <10 min, instant rollback

**Example: Self-service new service template:**
```bash
# Developer runs:
./platform-cli create-service --name order-api --type nestjs --db postgres

# This generates:
# - Terraform module for ECS + RDS + ALB
# - Dockerfile + docker-compose for local dev
# - GitHub Actions CI/CD pipeline
# - CloudWatch dashboard + alarms
# - README with getting-started instructions
```

---

## MongoDB & PostgreSQL Data Stores

### Q: How do you configure and manage MongoDB and Postgres?

**A:**

**MongoDB (Navy Federal experience):**
- **Replica sets**: 3-node replica set (1 primary, 2 secondaries) for HA and read scaling
- **Index optimization**: Compound indexes based on query patterns, covered queries for performance
- **Performance tuning**: Connection pool sizing, WiredTiger cache tuning, query profiling (`db.setProfilingLevel(1, { slowms: 100 })`)
- **Backup**: Continuous backup with oplog replay for point-in-time recovery
- **Security**: SCRAM authentication, TLS, network encryption, role-based access
- **Monitoring**: MongoDB Atlas metrics or custom Prometheus exporters — ops counters, replication lag, connection count, WiredTiger stats

**On AWS**: Use DocumentDB (MongoDB-compatible) or self-managed on EC2:
```hcl
resource "aws_docdb_cluster" "main" {
  cluster_identifier      = "backend-mongo"
  engine                  = "docdb"
  master_username         = "admin"
  master_password         = data.aws_secretsmanager_secret_version.db.secret_string
  backup_retention_period = 7
  preferred_backup_window = "02:00-04:00"
  vpc_security_group_ids  = [aws_security_group.docdb.id]
  db_subnet_group_name    = aws_docdb_subnet_group.main.name
  storage_encrypted       = true
  kms_key_id              = aws_kms_key.db.arn
}
```

**PostgreSQL (Fineos, Bank of America experience):**
- **RDS Multi-AZ**: Automatic failover for production
- **Read replicas**: Offload read traffic for reporting/analytics
- **Parameter tuning**: `shared_buffers` (25% of RAM), `work_mem`, `effective_cache_size`, `max_connections`
- **Connection pooling**: PgBouncer sidecar or RDS Proxy for connection management
- **Performance**: `pg_stat_statements` for slow query identification, `EXPLAIN ANALYZE` for query plans
- **Migrations**: Flyway or TypeORM migrations in CI/CD pipeline

```sql
-- Performance monitoring queries
-- Top slow queries
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC LIMIT 10;

-- Connection usage
SELECT count(*), state FROM pg_stat_activity GROUP BY state;

-- Table bloat
SELECT schemaname, tablename, pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename))
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC LIMIT 10;
```

### Q: How do you handle database outages?

**A:**
1. **RDS Multi-AZ failover**: Automatic — DNS endpoint switches to standby (60-120s downtime)
2. **Application retry logic**: Exponential backoff with circuit breaker (Resilience4j / NestJS equivalent)
3. **Connection pool recovery**: Pool validates connections on checkout, evicts stale ones
4. **Read replica promotion**: If primary is gone, promote read replica to primary
5. **Point-in-time recovery**: Restore from automated backup to specific timestamp
6. **Post-incident**: Check CloudWatch RDS metrics, review Performance Insights, update connection pool config if needed

---

## Reusable Modules & Internal Tooling

### Q: How do you develop reusable infrastructure modules that reduce friction?

**A:** This is what I did at Fineos when building the Internal Developer Platform (IDP):

**Terraform Module Standards:**
- **Versioned modules**: Git tags (`v1.0.0`, `v1.1.0`) so teams can pin versions
- **Sensible defaults**: Works out of the box for 80% of use cases; override for the rest
- **Validation**: Input variable validation with custom error messages
- **Outputs**: Export everything downstream modules need (IDs, endpoints, ARNs)
- **Documentation**: README with usage examples, required variables, and architecture diagram
- **Testing**: Terratest for automated module validation

```hcl
# Reusable ECS service module with validation
variable "container_port" {
  type        = number
  default     = 3000
  description = "Port the container listens on"
  validation {
    condition     = var.container_port > 0 && var.container_port < 65536
    error_message = "Container port must be between 1 and 65535"
  }
}

variable "environment" {
  type = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod"
  }
}
```

**Internal Tooling Examples:**
1. **Service scaffolding CLI**: Generates new NestJS service with Terraform, CI/CD, monitoring pre-configured
2. **Secret rotation automation**: Python script rotating DB credentials via Secrets Manager with zero-downtime
3. **Cost reporting**: Weekly Slack bot posting per-service AWS costs with trend analysis
4. **Environment provisioning**: One-command ephemeral staging environments for PR testing
5. **Deployment dashboard**: Internal web app showing all services, versions, health, last deploy time

---

## Design Through Deployment to Production Support

### Q: How do you take features from design through deployment and ongoing production support?

**A:** Full lifecycle ownership:

**1. Design:**
- Review requirements with product engineering team
- Design infrastructure architecture (draw.io / Lucidchart diagrams)
- Write ADR (Architecture Decision Record) documenting trade-offs
- Estimate effort, identify risks, create Jira stories

**2. Build:**
- Write Terraform modules for new infrastructure
- Build CI/CD pipeline (GitHub Actions: lint -> test -> build -> deploy)
- Configure monitoring (CloudWatch dashboards, alarms, log groups)
- Write documentation (README, runbook, architecture doc)

**3. Deploy:**
- Deploy to dev -> run integration tests
- Deploy to staging -> run load tests + security scans
- Deploy to prod -> canary or blue-green with automated rollback
- Verify health via monitoring dashboards

**4. Production Support:**
- Monitor dashboards daily for anomalies
- Respond to alerts (PagerDuty on-call rotation)
- Performance tuning based on production metrics
- Capacity planning based on growth trends
- Regular patching and dependency updates

**5. Iterate:**
- Gather feedback from product engineering team
- Identify bottlenecks and improvements
- Implement and measure impact
- Document learnings

---

## Outage Response & Rollback

### Q: How do you deal with outages? What steps do you take?

**A:**

**Immediate (0-5 min):**
```bash
# 1. Assess scope
aws ecs describe-services --cluster prod --services backend-api
aws cloudwatch get-metric-statistics --metric-name HTTPCode_Target_5XX_Count ...

# 2. Check recent changes
aws ecs describe-task-definition --task-definition backend-api  # Current version
git log --oneline -5  # Recent commits

# 3. Check logs
aws logs tail /ecs/backend-api --since 15m --filter-pattern "ERROR"
```

**Triage (5-15 min):**
| Symptom | Check | Fix |
|---------|-------|-----|
| Tasks failing to start | Task stopped reason, image pull errors | Fix image, rollback task def |
| High error rate | Application logs, DB connectivity | Fix or rollback deployment |
| High latency | RDS Performance Insights, connection pool | Scale up, add read replica |
| No traffic | ALB target health, security groups, DNS | Fix routing, SG rules |
| Memory/CPU spike | CloudWatch container insights | Scale out, fix memory leak |

**Rollback:**
```bash
# ECS: Rollback to previous task definition
aws ecs update-service --cluster prod --service backend-api \
  --task-definition backend-api:42  # Previous version

# Terraform: Revert and re-apply
git revert HEAD
git push  # CI/CD re-applies previous infra state

# Database: Point-in-time restore
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier prod-db \
  --target-db-instance-identifier prod-db-restore \
  --restore-time 2026-03-25T10:00:00Z
```

---

## Async Collaboration on Distributed Teams

### Q: How do you collaborate asynchronously across a globally distributed remote team?

**A:** This has been my working model at Navy Federal and Fineos:

- **Written-first communication**: Detailed Confluence pages and GitHub PR descriptions — anyone in any timezone can understand the context without a meeting
- **Async standups**: Slack channel updates with what I did, what I'm doing, blockers — not just in live meetings
- **Self-documenting work**: PRs include architecture decisions, screenshots of monitoring before/after, and rollback instructions
- **Overlap hours**: Identify 2-3 hours of timezone overlap for real-time discussions; use async for everything else
- **Decision logs**: Document decisions in ADRs so team members joining later understand the "why"
- **Video recordings**: Record demos and walkthroughs for team members who can't attend live
- **Clear ownership**: Jira assignments and PR reviewers clearly defined so nothing falls through timezone gaps

---

## Startup / Fast-Paced Environment

### Q: How do you operate in a high-growth startup environment?

**A:** My approach:

- **Bias toward action**: Ship iteratively — MVP infrastructure first, optimize later. Don't gold-plate
- **Pragmatic trade-offs**: Use managed services (RDS, ECS Fargate, DocumentDB) over self-managed to move faster
- **Automation from day 1**: Even in a fast environment, I automate deploys and monitoring early — it saves more time than it costs within a week
- **Reduce blast radius**: Feature flags, canary deployments, and fast rollback so shipping fast doesn't mean shipping dangerously
- **Clear communication**: Surface problems early. "This will take 2 weeks, not 2 days" is better said on day 1 than day 10
- **Wear multiple hats**: Comfortable doing infra, CI/CD, some backend code, monitoring, security, and on-call — startup reality

**How I've demonstrated this:**
- **Navy Federal**: Simultaneously managed 3 major platform initiatives while supporting day-to-day operations
- **Fineos**: Built an entire Internal Developer Platform from scratch while maintaining existing infrastructure
- **Keysight**: Ramped up quickly (5-month contract) and delivered automated vSphere provisioning and dynamic Jenkins pipelines

---

## Scenario-Based Questions

### Q: A new product feature requires a new microservice with a Postgres database. Walk me through your approach.

**A:**
1. **Design**: Meet with product eng to understand data model, traffic expectations, and SLA requirements
2. **Terraform**: Create ECS service module + RDS Postgres module, S3 for any file storage, CloudWatch for monitoring
3. **Networking**: Private subnet, security group allowing only ECS -> RDS on port 5432, ALB for external traffic with TLS
4. **CI/CD**: GitHub Actions workflow — `npm test` -> Docker build -> push to ECR -> deploy to ECS (dev -> staging -> prod)
5. **Secrets**: DB credentials in AWS Secrets Manager, referenced in ECS task definition
6. **Monitoring**: CloudWatch dashboard (ECS metrics + RDS metrics + ALB metrics), alarms for error rate and latency
7. **Documentation**: README, architecture diagram, runbook
8. **Deploy**: Dev first, integration tests, staging with load test, prod with canary
9. **Handoff**: Walk product eng through monitoring dashboard, share runbook, add to on-call rotation

### Q: How would you reduce friction for product engineering teams?

**A:**
1. **One-command environment creation**: Terraform module that provisions a complete dev environment in minutes
2. **Pipeline templates**: GitHub Actions reusable workflows — devs just reference the template and set vars
3. **Self-service secrets**: Devs can provision secrets via PR to a secrets config repo (approved by platform team)
4. **Fast deployments**: Target <10 min from merge to production (with automated tests and canary)
5. **Clear documentation**: Every module has a README with copy-paste examples
6. **Office hours**: Weekly platform team office hours for questions and feature requests
7. **Metrics**: Track developer experience metrics — deploy frequency, lead time, build wait time — and optimize

---

## STAR Stories for This Role

**Best stories to use for Hampton North interviews:**

| Question | Story |
|----------|-------|
| "Building platform tooling for engineering teams" | Fineos IDP — reduced onboarding 50% with self-service templates |
| "Handling infrastructure outages" | Navy Federal AKS Meltdown — 23-min recovery, implemented preventive monitoring |
| "Fast-paced environment" | Navy Federal — 3 concurrent initiatives delivered on schedule |
| "Innovation" | Ephemeral VMSS agents — replaced legacy static agents, 50% queue time reduction |
| "Working with distributed teams" | Navy Federal/Fineos — fully remote, async communication, Confluence-first docs |
| "Terraform at scale" | Navy Federal — Terragrunt DRY modules across multi-env AKS + networking + storage |
| "Proactive communication" | Flagged AKS capacity risk in sprint planning; presented data + solution before it became an incident |
