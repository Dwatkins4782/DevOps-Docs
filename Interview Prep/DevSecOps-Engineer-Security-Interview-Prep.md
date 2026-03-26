# DevSecOps Engineer (Security) - Interview Preparation Guide

**Role**: DevSecOps Engineer (Security)
**Prepared by**: Davonte Watkins
**Last Updated**: March 2026

---

## Table of Contents
- [Role Requirements Mapping](#role-requirements-mapping)
- [Pipeline Security Integration](#pipeline-security-integration)
- [Infrastructure as Code Security](#infrastructure-as-code-security)
- [IAM, Encryption & VPC Security](#iam-encryption--vpc-security)
- [Container & Kubernetes Security](#container--kubernetes-security)
- [Vulnerability Management](#vulnerability-management)
- [Threat Modeling](#threat-modeling)
- [Compliance-as-Code](#compliance-as-code)
- [Python Gen AI for Security Automation](#python-gen-ai-for-security-automation)
- [SQL for Security Operations](#sql-for-security-operations)
- [Scenario-Based Questions](#scenario-based-questions)
- [My Experience Talking Points](#my-experience-talking-points)

---

## Role Requirements Mapping

| Requirement | My Direct Experience |
|-------------|---------------------|
| Pipeline Security (SAST, DAST, SCA) | Integrated Checkmarx (SAST), OWASP ZAP (DAST), Black Duck (SCA) into Jenkins, ADO, GitHub Actions pipelines |
| IaC Security (Checkov, Terrascan, OPA) | Deployed OPA Gatekeeper on OpenShift/AKS; IaC scanning with Checkov/tfsec in CI/CD |
| IAM, Encryption, VPC Security | Azure AD RBAC, Key Vault, AWS IAM, VPC design, encryption at rest/transit |
| Container & K8s Security (EKS/AKS) | Image hardening (CIS), Twistlock/Prisma Cloud, non-root enforcement, runtime security |
| Vulnerability Management | Triage-to-remediation workflows; CVSS scoring; developer remediation SLAs |
| Threat Modeling | Architectural risk identification; STRIDE methodology; secure design reviews |
| Compliance-as-Code (SOC2, ISO 27001, HIPAA) | PCI-DSS (Bank of America, Wells Fargo), HIPAA (Fineos), CIS Benchmarks |
| Enterprise DevOps/DevSecOps | 10+ years across Navy Federal, Fineos, Bank of America, Wells Fargo, Keysight |
| Scripting (Python, Bash, PowerShell) | Extensive automation including Gen AI-powered security tooling |
| IaC (Terraform, CloudFormation, Ansible) | Terraform/Terragrunt, Bicep, CloudFormation, Ansible, Packer, Chef |

---

## Pipeline Security Integration

### Q: How do you integrate security into CI/CD pipelines?
**A:** I follow a "shift-left" approach embedding security at every pipeline stage:

```
[Code Commit] -> [Pre-Commit Hooks] -> [Build] -> [SAST] -> [SCA] -> [Container Scan] -> [Deploy to Staging] -> [DAST] -> [Security Gate] -> [Deploy to Prod]
```

**My implementation across roles:**

| Stage | Tool | What It Catches | My Experience |
|-------|------|-----------------|---------------|
| Pre-commit | git-secrets, detect-secrets | Hardcoded secrets, API keys | Navy Federal, Fineos |
| SAST | Checkmarx, SonarQube, Fortify | SQL injection, XSS, insecure code patterns | Navy Federal, Bank of America |
| SCA/Dependency | Black Duck, Snyk, OWASP Dependency-Check | Known CVEs in libraries, license violations | Navy Federal, Fineos, Bank of America |
| Container Scan | Twistlock/Prisma Cloud, Trivy | Base image vulnerabilities, misconfigurations | Fineos, Bank of America |
| DAST | OWASP ZAP, Contrast | Runtime vulnerabilities, auth bypass, injection | Bank of America |
| IaC Scan | Checkov, tfsec, OPA | Misconfigurations, insecure defaults | Navy Federal, Fineos |
| Secrets Scan | CyberArk Conjur, detect-secrets | Leaked credentials in code/configs | All roles |

### Q: Walk me through a secure pipeline you built.
**A:** At Navy Federal (Azure DevOps):

```yaml
# Simplified secure pipeline structure
trigger:
  branches:
    include: [main, release/*]

stages:
  - stage: SecurityScan
    jobs:
      - job: SAST
        steps:
          - task: Checkmarx@1
            inputs:
              projectName: 'payment-service'
              preset: 'ASA Premium'
              highSeverityThreshold: 0  # Fail on any High

      - job: SCA
        steps:
          - task: BlackDuck@1
            inputs:
              policyCheck: true
              failOnPolicyViolation: true

      - job: ContainerScan
        steps:
          - script: |
              trivy image --severity HIGH,CRITICAL --exit-code 1 $(imageTag)

      - job: IaCScan
        steps:
          - script: |
              checkov -d ./terraform --framework terraform --check HIGH

  - stage: SecurityGate
    dependsOn: SecurityScan
    condition: succeeded()
    jobs:
      - job: GateApproval
        steps:
          - script: |
              # Aggregate scan results
              python scripts/security-gate.py \
                --sast-report checkmarx-results.json \
                --sca-report blackduck-results.json \
                --container-report trivy-results.json \
                --threshold "HIGH:0,MEDIUM:5"

  - stage: DeployStaging
    dependsOn: SecurityGate
    jobs:
      - job: Deploy
        steps:
          - task: CyberArkConjur@1  # Secrets from Conjur, not pipeline vars
            inputs:
              secretIds: |
                db-password=secrets/staging/db/password

  - stage: DAST
    dependsOn: DeployStaging
    jobs:
      - job: DynamicScan
        steps:
          - script: |
              zap-cli --quick-scan https://staging.app.internal

  - stage: DeployProd
    dependsOn: DAST
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
```

### Q: How do you handle security gate failures?
**A:**
1. **Hard gates** (zero tolerance): High/Critical SAST findings, known exploited CVEs (CISA KEV), secrets detected - pipeline fails immediately
2. **Soft gates** (threshold-based): Medium findings with a threshold (e.g., <5 allowed), low findings documented but don't block
3. **Exception process**: Developers can request a time-boxed exception via Jira with security team approval
4. **Remediation SLAs**: Critical = 24 hours, High = 7 days, Medium = 30 days, Low = 90 days
5. **Tracking**: All findings tracked in a centralized dashboard (Grafana) with team-level metrics

At Navy Federal, implementing these gates reduced production security incidents by 60% within the first quarter.

---

## Infrastructure as Code Security

### Q: How do you enforce security in Terraform/IaC?
**A:**

**Pre-deployment scanning:**
```bash
# Checkov - comprehensive IaC scanning
checkov -d ./terraform --framework terraform \
  --check CKV_AZURE_1,CKV_AZURE_2 \  # Specific checks
  --skip-check CKV_AZURE_35 \         # Documented exceptions
  --output json > checkov-report.json

# tfsec - Terraform-specific
tfsec ./terraform --format json --tfvars-file prod.tfvars

# Terrascan - policy-as-code
terrascan scan -t azure -i terraform -d ./terraform
```

**OPA/Gatekeeper enforcement (runtime):**
```yaml
# ConstraintTemplate - enforce resource limits
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels
        violation[{"msg": msg}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("Missing required labels: %v", [missing])
        }
```

**My experience:**
- At Navy Federal: Deployed OPA Gatekeeper on AKS clusters enforcing resource limits, image registries, labeling standards, and security contexts
- At Fineos: OPA Gatekeeper on OpenShift enforcing HIPAA-compliant configurations
- At Bank of America: OPA policies enforcing PCI-DSS pod security policies and network segmentation

### Q: How do you prevent IaC drift and misconfigurations?
**A:**
1. **GitOps**: ArgoCD continuously reconciles desired state (Git) with actual state (cluster)
2. **Drift detection**: Terraform plan in CI shows any drift from checked-in state
3. **Policy enforcement**: OPA Gatekeeper rejects non-compliant resources at admission time
4. **Automated remediation**: Self-healing via ArgoCD sync policies
5. **Regular audits**: Scheduled Checkov scans against running infrastructure

---

## IAM, Encryption & VPC Security

### Q: How do you implement IAM best practices?
**A:**

**Azure (my primary experience):**
- **RBAC**: Custom role definitions with least-privilege; no standing admin access
- **Managed Identities**: Workload identity for AKS pods, eliminating stored credentials
- **PIM (Privileged Identity Management)**: Just-in-time elevation for admin access
- **Conditional Access**: MFA enforcement, location-based restrictions
- **Service Principals**: Scoped to minimum required permissions; credentials rotated via CyberArk Conjur

**AWS:**
- **IAM Roles**: Instance roles for EC2, task roles for ECS/EKS
- **STS AssumeRole**: Cross-account access with session policies
- **SCPs**: Service Control Policies at organization level
- **Permission boundaries**: Prevent privilege escalation

### Q: How do you handle encryption at rest and in transit?
**A:**

| Layer | At Rest | In Transit |
|-------|---------|------------|
| Storage | Azure Storage SSE (AES-256), AWS S3 SSE-KMS | TLS 1.2+ enforced |
| Database | TDE for SQL Server, encrypted EBS for PostgreSQL | SSL/TLS connections required |
| Kubernetes | etcd encryption enabled, Sealed Secrets for manifests | mTLS via service mesh (Istio/Linkerd) |
| Secrets | Conjur AES-256 encrypted, Key Vault HSM-backed | TLS to Conjur API, encrypted in memory |
| Backups | Encrypted with customer-managed keys | Transfer via private endpoints |

### Q: How do you secure VPC/VNet architecture?
**A:** At Navy Federal I implemented:
- **Hub-and-spoke** network topology with centralized Azure Firewall
- **Private endpoints** for all PaaS services (Key Vault, ACR, SQL)
- **NSG rules**: Default deny-all, explicit allow rules per service
- **Service mesh**: Istio for pod-to-pod mTLS within AKS
- **Ingress**: Kong API Gateway with JWT/OAuth2 authentication
- **Egress**: All outbound traffic routed through Azure Firewall with FQDN filtering
- **Network policies**: Kubernetes NetworkPolicies for namespace isolation

---

## Container & Kubernetes Security

### Q: How do you harden container images?
**A:**

1. **Base image selection**: Minimal images (Alpine, distroless) from approved registry only
2. **CIS Benchmark hardening**:
   - Non-root user enforcement
   - Read-only root filesystem
   - No privilege escalation (allowPrivilegeEscalation: false)
   - Drop all capabilities, add only required ones
3. **Image scanning**: Trivy/Prisma Cloud in CI pipeline - fail on High/Critical
4. **Image signing**: Cosign for image provenance validation
5. **Registry restrictions**: OPA Gatekeeper policy allowing only approved ACR registries

```yaml
# Hardened pod security context
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault
```

### Q: How do you implement runtime security in Kubernetes?
**A:**
- **Pod Security Standards**: Enforced "restricted" profile via namespace labels
- **OPA Gatekeeper**: Admission controller rejecting non-compliant workloads
- **Falco/Prisma Cloud**: Runtime threat detection for anomalous behavior (shell spawns, unexpected network connections, file modifications)
- **Network Policies**: Default deny ingress/egress per namespace; explicit allow rules
- **Service mesh mTLS**: All pod-to-pod communication encrypted and authenticated
- **Audit logging**: Kubernetes audit logs forwarded to SIEM

At Bank of America, I enforced CIS Benchmark container security standards across OpenShift clusters, ensuring all deployed workloads passed internal InfoSec review before production release. At Fineos, I deployed Prisma Cloud (Twistlock) for container image scanning and runtime protection.

---

## Vulnerability Management

### Q: Describe your triage-to-remediation workflow.
**A:**

```
[Scan Detected] -> [Auto-Triage] -> [Severity Classification] -> [Developer Assignment] -> [Remediation] -> [Verification Scan] -> [Close]
```

**Detailed process I implemented:**

1. **Detection**: SAST/DAST/SCA tools detect vulnerability in pipeline
2. **Auto-triage**: Python script classifies severity using CVSS score + exploitability + business context
3. **Jira ticket creation**: Automated ticket with vulnerability details, affected code, remediation guidance
4. **Developer notification**: Slack/Teams alert to the owning team
5. **Remediation support**: Security team provides fix guidance, code snippets, or pairing sessions
6. **SLA tracking**: Dashboard tracks remediation against SLAs (Critical:24h, High:7d, Medium:30d)
7. **Verification**: Re-scan in next pipeline run confirms fix
8. **Metrics**: Track MTTR (mean time to remediate), open vulnerability counts, team compliance rates

### Q: How do you handle false positives?
**A:**
- **Suppression file**: Maintain a documented suppression list (e.g., `.checkmarx-suppress.json`) with justification for each suppression
- **Review process**: Suppressions require security team approval via PR review
- **Regular review**: Quarterly review of all suppressions to ensure they're still valid
- **Tool tuning**: Feed false positives back to tool configuration to improve accuracy
- **Custom rules**: Write custom SAST/SCA rules to reduce noise for your tech stack

---

## Threat Modeling

### Q: How do you perform threat modeling?
**A:** I use **STRIDE** methodology applied to our architecture:

| Threat | Category | How I Mitigate |
|--------|----------|----------------|
| **S**poofing | Identity | Azure AD MFA, Conjur host identity, mTLS |
| **T**ampering | Data integrity | Signed container images, immutable infrastructure, Git-signed commits |
| **R**epudiation | Audit | Conjur audit logs, K8s audit logs, centralized SIEM |
| **I**nformation Disclosure | Confidentiality | Encryption at rest/transit, secrets management, network segmentation |
| **D**enial of Service | Availability | Rate limiting (Kong), autoscaling, HA/DR design |
| **E**levation of Privilege | Authorization | Least-privilege RBAC, pod security contexts, no standing admin |

**My approach at Navy Federal:**
1. **Architecture review**: Before any new service deployment, review the architecture diagram
2. **Data flow analysis**: Map all data flows and identify trust boundaries
3. **Threat identification**: Apply STRIDE to each component and data flow
4. **Risk scoring**: Likelihood x Impact matrix
5. **Mitigation design**: Security controls mapped to each identified threat
6. **Validation**: Verify mitigations through DAST, penetration testing, and code review

---

## Compliance-as-Code

### Q: How do you implement Compliance-as-Code?
**A:**

**PCI-DSS (Bank of America, Wells Fargo):**
- OPA policies enforcing network segmentation between CDE and non-CDE workloads
- Automated secret rotation evidence via Conjur audit logs (Requirement 3, 8)
- Container hardening policies enforcing CIS Benchmarks
- Encrypted data transmission enforced via service mesh mTLS

**HIPAA (Fineos):**
- Deployment pipeline gates checking for PHI handling compliance
- Encryption enforcement policies (at rest and transit)
- Access control auditing via Conjur and K8s RBAC audit logs
- Automated compliance reports generated from infrastructure state

**SOC 2 / ISO 27001 patterns:**
```yaml
# Example: OPA policy enforcing encryption at rest
package compliance.soc2

deny[msg] {
  input.resource_type == "azurerm_storage_account"
  not input.resource.properties.encryption.services.blob.enabled
  msg := "SOC2 CC6.1: Storage account must have blob encryption enabled"
}

deny[msg] {
  input.resource_type == "azurerm_kubernetes_cluster"
  not input.resource.properties.disk_encryption_set_id
  msg := "SOC2 CC6.1: AKS must use customer-managed disk encryption"
}
```

**Automated compliance evidence collection:**
- Terraform state as infrastructure evidence
- Conjur audit logs as credential management evidence
- OPA decision logs as policy enforcement evidence
- Pipeline logs as change management evidence
- All collected automatically and stored for audit retrieval

---

## Python Gen AI for Security Automation

### Q: How have you used Python and Gen AI in security?
**A:** At Navy Federal and Fineos, I built Python-based Gen AI automation tools:

1. **Pipeline Log Anomaly Detection**: Built a Python tool using LLM APIs to analyze CI/CD pipeline logs, automatically classifying failures as security-related, infrastructure-related, or code-related, and generating incident summaries with recommended remediation steps.

2. **Infrastructure Documentation Generation**: Python scripts leveraging Gen AI to parse Terraform modules and automatically generate security-focused documentation including threat models, data flow diagrams descriptions, and compliance mapping.

3. **Vulnerability Report Summarization**: Gen AI-powered tool that ingests SAST/DAST/SCA scan results and produces executive summaries, developer-friendly remediation guides, and compliance impact assessments.

4. **Runbook Auto-Generation**: Python automation that reads incident history and infrastructure configs to generate operational runbooks for common security incidents (secret compromise, unauthorized access, container runtime alerts).

5. **HIPAA Audit Documentation**: At Fineos, built Gen AI scripts to auto-generate HIPAA-compliant audit documentation from CI/CD telemetry data, reducing audit preparation time by 70%.

---

## SQL for Security Operations

### Q: How have you used SQL in your DevSecOps work?
**A:**

1. **Credential Rotation Tracking (Navy Federal)**:
   - Wrote SQL Server queries to track credential rotation history, identify stale credentials, and generate rotation compliance reports
   - Stored procedures for automated rotation status dashboards

2. **Vulnerability Tracking Database (Bank of America)**:
   - SQL queries against vulnerability management database for trend analysis, MTTR calculations, and team-level compliance scoring
   - Aggregate reporting across SAST/DAST/SCA findings

3. **Audit Reporting (Fineos)**:
   - PostgreSQL queries for HIPAA compliance reporting - who accessed what data, when, and from where
   - MySQL queries for application health dashboards and deployment tracking

4. **Configuration Management (Navy Federal)**:
   - SQL Server queries supporting application configuration management across managed environments
   - Database credential rotation verification queries confirming Conjur-managed rotations completed successfully

```sql
-- Example: Credential rotation compliance report
SELECT
    s.secret_name,
    s.last_rotated,
    s.rotation_interval_days,
    DATEDIFF(day, s.last_rotated, GETDATE()) AS days_since_rotation,
    CASE
        WHEN DATEDIFF(day, s.last_rotated, GETDATE()) > s.rotation_interval_days
        THEN 'NON-COMPLIANT'
        ELSE 'COMPLIANT'
    END AS compliance_status
FROM secrets_inventory s
WHERE s.environment = 'production'
ORDER BY days_since_rotation DESC;
```

---

## Scenario-Based Questions

### Q: A critical CVE is discovered in a base image used across 50+ services. What do you do?
**A:**
1. **Assess scope**: Query container registry to identify all images using the vulnerable base
2. **Classify severity**: Check CVSS score, exploitability, and whether it's in CISA KEV
3. **Triage**: Determine which services are internet-facing vs internal-only
4. **Immediate mitigation**: If exploitable, deploy WAF rules or network restrictions as temporary mitigation
5. **Patch**: Update base image, rebuild all affected images via automated pipeline
6. **Validate**: Run security scans confirming remediation
7. **Deploy**: Prioritize internet-facing services, then roll out to all services
8. **Verify**: Confirm no services are running vulnerable images
9. **Post-incident**: Update processes to prevent recurrence (e.g., automated base image updates)

### Q: A developer pushes a hardcoded API key to a public repo. What do you do?
**A:**
1. **Revoke immediately**: Rotate/revoke the compromised API key on the target service
2. **Remove from history**: git filter-branch or BFG to remove the secret from Git history
3. **Audit usage**: Check API provider logs for unauthorized usage of the key
4. **Implement prevention**:
   - Pre-commit hooks with detect-secrets
   - Server-side secret scanning (GitHub Advanced Security)
   - Pipeline gate with secret detection
5. **Migrate to Conjur**: Move the credential to CyberArk Conjur with proper access policies
6. **Educate**: Conduct security awareness session with the team

### Q: How would you secure a new microservice from scratch?
**A:**
1. **Threat model**: STRIDE analysis before writing code
2. **Secure base image**: Approved, hardened, scanned base image from internal registry
3. **Pipeline security**: SAST, SCA, container scan gates from first commit
4. **Secrets management**: All credentials via Conjur from day one
5. **Network security**: Default-deny network policies, service mesh enrollment
6. **Authentication**: OAuth2/OIDC via Kong API Gateway
7. **Authorization**: RBAC with least-privilege
8. **Observability**: Security-focused logging (auth events, permission failures, data access)
9. **Compliance**: OPA policies enforcing organizational standards
10. **Documentation**: Threat model, security architecture, runbooks

---

## My Experience Talking Points

### Opening Statement
"I'm a Senior DevOps/DevSecOps Engineer with 10+ years of hands-on experience integrating security across the entire development lifecycle in highly regulated enterprise environments. I've implemented SAST/DAST/SCA pipeline security, IaC security enforcement with OPA/Checkov, container hardening, vulnerability management workflows, and Compliance-as-Code for PCI-DSS, HIPAA, and CIS standards. I bring direct experience securing Kubernetes (AKS/EKS/OpenShift), managing secrets with CyberArk Conjur, and building Gen AI-powered security automation tools."

### Key Differentiators
1. **Regulated environments**: PCI-DSS (Bank of America, Wells Fargo), HIPAA (Fineos), enterprise financial security (Navy Federal)
2. **Full security pipeline**: Pre-commit to runtime - SAST, DAST, SCA, container scanning, IaC scanning, secrets detection
3. **Policy-as-code expertise**: OPA Gatekeeper on AKS and OpenShift for admission control and compliance enforcement
4. **Secrets management**: CyberArk Conjur Enterprise deployment, configuration, and CI/CD integration
5. **Automation-first**: Python Gen AI tools for security automation, anomaly detection, and compliance documentation
6. **Kubernetes security depth**: Image hardening, runtime security, network policies, pod security contexts, service mesh mTLS
7. **Vulnerability management**: Built triage-to-remediation workflows reducing MTTR by 50%+ across engineering teams
