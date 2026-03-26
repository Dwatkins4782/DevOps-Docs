# Data Platform & Data Lake Infrastructure - Interview Q&A

**Prepared by**: Davonte Watkins
**Last Updated**: March 2026
**Context**: Recruiter screening question — "Describe a recent project where you supported or built infrastructure for a data platform or data lake."

---

## Full Answer (Navy Federal Credit Union)

At Navy Federal, I supported the infrastructure buildout for a centralized data platform used by analytics and application teams to process financial-grade data workloads.

### Infrastructure Provisioning (Terraform/Terragrunt + Azure)
- Built and managed cloud infrastructure using **Terraform and Terragrunt** with DRY module patterns, remote state management, and environment-specific overrides
- Provisioned **AKS clusters**, Azure networking (VNets, subnets, NSGs, private endpoints), **Azure Storage accounts** (Blob/ADLS Gen2 for data lake layers), and **Azure Key Vault** for secrets and encryption key management
- All infrastructure was version-controlled and deployed through **Azure DevOps YAML pipelines** with plan/apply stages and approval gates

### Data Storage & Database Layer (MongoDB + Azure Data Services)
- Built and managed **MongoDB clusters** supporting the platform's operational data layer — including replica set configuration, index optimization, performance tuning, backup strategies, and security hardening for financial-grade workloads
- Provisioned and configured **Azure Data Lake Storage Gen2** containers with hierarchical namespace enabled, organizing raw/curated/enriched data zones following a **medallion architecture** (bronze/silver/gold) pattern for the analytics team

### Kubernetes Platform (AKS)
- Architected and managed **AKS clusters** hosting data processing microservices — including node pool scaling, networking (Ingress, Service Mesh), RBAC, and multi-environment configuration across dev, staging, and production
- Data pipeline workloads (ETL/ELT jobs) ran as containerized services on these clusters, with **Horizontal Pod Autoscaling** configured to handle variable data processing loads

### Observability & Monitoring
- Designed the **observability stack** (Prometheus, Grafana, OpenTelemetry) with custom dashboards tracking data pipeline health, processing latency, throughput metrics, and SLO-based alerting
- Gave the data engineering team visibility into pipeline failures, data freshness, and processing bottlenecks

### Security & Secrets Management
- Deployed **External Secrets Operator** integrated with **Azure Key Vault** to manage database credentials, storage access keys, and API tokens — eliminating hardcoded secrets
- Implemented **Sealed Secrets (Bitnami)** for GitOps-compliant secret lifecycle management
- Enforced **zero-trust API access** via Kong API Gateway with JWT/OAuth2 authentication for data service endpoints

### CI/CD & GitOps
- All infrastructure and application deployments managed through **ArgoCD** for GitOps-based continuous delivery with automated sync, drift detection, and rollback capabilities
- Built fully automated **release pipelines** with release gates, security scan thresholds, and automated Jira/ServiceNow ticket updates

### Tools Summary
Terraform, Terragrunt, Azure (AKS, ADLS Gen2, Key Vault, VMSS, Azure Monitor), MongoDB, Kubernetes, ArgoCD, Prometheus, Grafana, OpenTelemetry, Azure DevOps, Docker, Helm, Kong API Gateway, External Secrets Operator, Python, Bash, PowerShell

---

## Condensed Version (For Short-Form Responses)

At Navy Federal, I built infrastructure for a centralized data platform on Azure using Terraform/Terragrunt for IaC, AKS for containerized data processing workloads, ADLS Gen2 organized in a medallion architecture (bronze/silver/gold), and MongoDB clusters for the operational data layer. I designed the observability stack (Prometheus, Grafana, OpenTelemetry) for pipeline health monitoring, implemented secrets management via External Secrets Operator + Azure Key Vault, and managed GitOps delivery through ArgoCD. All infrastructure was deployed through Azure DevOps pipelines with security gates and approval workflows.

---

## Follow-Up Questions to Prepare For

### Q: What is the medallion architecture?
**A:** A data organization pattern with three layers:
- **Bronze (Raw)**: Raw ingested data as-is from source systems — no transformation
- **Silver (Curated)**: Cleaned, deduplicated, conformed data — joins, type casting, validation applied
- **Gold (Enriched)**: Business-level aggregations, KPIs, and analytics-ready datasets

At Navy Federal, we stored each layer in separate ADLS Gen2 containers with lifecycle policies — bronze data retained 90 days, silver 1 year, gold indefinitely.

### Q: How did you handle data security and compliance?
**A:**
- **Encryption**: ADLS Gen2 encrypted at rest (Azure-managed keys); TLS 1.2 in transit
- **Access control**: Azure RBAC + ACLs on ADLS Gen2 containers; service principals with minimum permissions
- **Network isolation**: Private endpoints for storage accounts; VNet service endpoints; no public access
- **Secrets rotation**: Conjur/ESO managed all credentials with automated rotation
- **Audit logging**: Azure Monitor diagnostic logs + Prometheus metrics for all data access
- **Compliance**: Financial-grade controls aligned with internal security policies

### Q: How did you scale the data processing workloads?
**A:**
- **HPA**: Kubernetes Horizontal Pod Autoscaler based on CPU/memory and custom metrics (queue depth)
- **Node pools**: Dedicated node pools for data workloads with appropriate VM sizes (memory-optimized for ETL)
- **Cluster autoscaler**: AKS cluster autoscaler to add/remove nodes based on pending pod demand
- **Batch scheduling**: Used Kubernetes CronJobs for scheduled ETL; Jobs for ad-hoc processing
- **Resource quotas**: Per-namespace quotas to prevent data jobs from starving API workloads

### Q: How did you monitor data pipeline health?
**A:** Custom Grafana dashboards tracking:
- **Data freshness**: Time since last successful pipeline run per data source
- **Processing latency**: End-to-end time from ingestion to gold layer availability
- **Error rates**: Failed pipeline runs, data validation failures, schema drift alerts
- **Throughput**: Records processed per minute/hour across bronze/silver/gold layers
- **Resource usage**: Pod CPU/memory, node utilization, storage growth trends
- **Alerting**: PagerDuty integration for critical failures (stale data >2 hours, pipeline failure >3 consecutive runs)

### Q: What challenges did you face?
**A:**
1. **Schema evolution**: Source systems changing schemas broke downstream pipelines — implemented schema registry and validation at bronze layer
2. **Cost optimization**: ADLS Gen2 storage costs grew quickly — implemented lifecycle policies, tiered storage (hot/cool/archive), and data retention policies
3. **Network latency**: Cross-region data movement was slow — implemented regional storage accounts with Azure Data Factory for replication
4. **Security reviews**: Financial compliance required extensive documentation — automated compliance evidence collection via Prometheus metrics and Azure Policy
