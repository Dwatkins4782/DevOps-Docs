# Recruiter Screening Questions - Quick Reference

**Prepared by**: Davonte Watkins
**Last Updated**: March 2026

These are common Yes/No + proof-point questions recruiters ask. Each maps to direct production experience.

---

## 1. AWS + Terraform (Production)

**Q: Do you have hands-on experience building and managing cloud infrastructure in AWS using Terraform in a production environment?**

**Yes.**

| Where | What I Built | Scale |
|-------|-------------|-------|
| **Keysight** | VPC, EC2, Route 53, CloudWatch, S3, security groups (Terraform + Ansible) | Production test automation platform |
| **Fineos** | AWS hybrid cloud (OpenShift on AWS), EC2, S3, IAM, Secrets Manager | HIPAA healthcare platform |
| **Bank of America** | AWS networking, EC2, security groups, IAM (Terraform + Ansible) | PCI-DSS financial platform |
| **Navy Federal** | Azure (AKS, networking, storage, Key Vault) via Terraform/Terragrunt — same patterns | Enterprise financial platform |

**My Terraform workflow:**
- Modules for reusable patterns (VPC, subnets, SGs, EC2, S3, IAM)
- Remote state (S3 + DynamoDB locking on AWS; Azure Storage on Azure)
- Terragrunt for DRY multi-environment overrides
- CI/CD: `terraform plan` on PR, `terraform apply` on merge with approval gates
- Drift detection via scheduled plan runs
- Rollback: git revert -> pipeline re-applies previous state

---

## 2. Kubernetes + Helm (Production)

**Q: Have you deployed and managed containerized applications in Kubernetes using Helm in a production environment?**

**Yes.**

| Where | Platform | Helm Usage |
|-------|----------|------------|
| **Navy Federal** | AKS (built end-to-end) | Helm charts for microservices, ArgoCD app-of-apps |
| **Fineos** | OpenShift 4.x | Jenkins JCasC via Helm, automated Helm deploys to hybrid cloud |
| **Bank of America** | OpenShift | Helm charts for standardized app packaging |

**My Helm workflow:**
- Custom charts with per-env values (dev/staging/prod)
- Helm repos in JFrog Artifactory
- Helm hooks for DB migrations (pre-install, pre-upgrade)
- ArgoCD integration for GitOps delivery
- Rollback: `helm rollback <release> <revision>` or git revert for ArgoCD

---

## 3. Python Automation (Production)

**Q: Have you written and maintained Python scripts for automation or infrastructure support in a production environment?**

**Yes.**

| Where | Use Case | Impact |
|-------|----------|--------|
| **Navy Federal** | Build agent automation (provisioning, health monitoring, config) | Managed agents for thousands of developers |
| **Navy Federal** | Pipeline Log Anomaly Detection (LLM-powered) | Reduced MTTR for pipeline failures |
| **Navy Federal** | Terraform doc generator (auto-generates security docs from modules) | Automated compliance documentation |
| **Fineos** | Jira API integration (ticket creation, status updates from pipelines) | End-to-end traceability |
| **Keysight** | Test framework refactoring (superclass/subclass -> generic modules) | Simplified test automation |
| **Bank of America** | Glue scripts (Jenkins + Git/SVN + Artifactory + OpenShift) | Multi-tool pipeline orchestration |

**Types I write:** Infrastructure wrappers, secret rotation, pipeline health monitoring, API integrations, Gen AI tools (log analysis, doc generation)

---

## 4. Data Platform / Data Lake

**Q: Briefly describe a recent project where you supported or built infrastructure for a data platform or data lake.**

At **Navy Federal** (2022-2025), I built infrastructure for a centralized data platform:
- **ADLS Gen2** organized in medallion architecture (bronze/silver/gold)
- **AKS clusters** hosting containerized ETL/ELT with HPA
- **Terraform/Terragrunt** for all IaC with modular patterns
- **MongoDB clusters** for operational data layer
- **Prometheus + Grafana + OpenTelemetry** for pipeline health monitoring
- **ESO + Azure Key Vault** for secrets; **ArgoCD** for GitOps delivery
- **Azure DevOps YAML pipelines** with security gates and approval workflows

**Tools:** Terraform, Terragrunt, Azure (AKS, ADLS Gen2, Key Vault), MongoDB, Kubernetes, ArgoCD, Prometheus, Grafana, OpenTelemetry, Azure DevOps, Docker, Helm, Python, Bash, PowerShell
