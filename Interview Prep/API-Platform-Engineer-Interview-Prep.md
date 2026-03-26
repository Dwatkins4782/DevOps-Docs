# API Platform Engineer (Java/OpenShift/3Scale) - Interview Preparation Guide

**Role**: API Development & Platform Engineer (85% API Dev / 15% Jira Documentation)
**Tech Stack**: Java, Maven, JBoss, RESTful APIs, Red Hat OpenShift, 3Scale, Jenkins/Tekton, Kubernetes, Bitbucket (Git), Confluence
**Prepared by**: Davonte Watkins
**Last Updated**: March 2026

---

## Table of Contents
- [Role Requirements Mapping](#role-requirements-mapping)
- [API Development & RESTful Services](#api-development--restful-services)
- [Red Hat OpenShift Configuration & Management](#red-hat-openshift-configuration--management)
- [3Scale API Management](#3scale-api-management)
- [JBoss / EAP Configuration & Management](#jboss--eap-configuration--management)
- [CI/CD with Jenkins & Tekton](#cicd-with-jenkins--tekton)
- [Monitoring & Observability](#monitoring--observability)
- [Security Management](#security-management)
- [Outage Response & Rollback Strategies](#outage-response--rollback-strategies)
- [RDBMS & Database Operations](#rdbms--database-operations)
- [Jira & Confluence Documentation](#jira--confluence-documentation)
- [Agile & Team Collaboration](#agile--team-collaboration)
- [Scenario-Based Questions](#scenario-based-questions)
- [My Experience Talking Points](#my-experience-talking-points)

---

## Role Requirements Mapping

| Requirement | My Direct Experience |
|-------------|---------------------|
| Java / RESTful Service APIs | Built and supported Java-based microservices on JBoss/OpenShift at Fineos and Bank of America |
| RDBMS experience | SQL Server, PostgreSQL, MySQL, Oracle - queries, stored procs, performance tuning |
| Maven | Managed Maven builds in Jenkins/Tekton pipelines; POM configuration, dependency management, artifact publishing |
| Jenkins or Tekton | 10+ years Jenkins (Groovy pipelines); Tekton on OpenShift at Fineos |
| Red Hat OpenShift | Deployed and managed OpenShift clusters at Fineos and Bank of America; OCP 4.x operations |
| 3Scale API Management | Configured 3Scale for API gateway, rate limiting, OAuth/OIDC, developer portal |
| CI/CD | End-to-end pipeline design across Jenkins, Tekton, Azure DevOps, GitHub Actions, ArgoCD |
| JBoss | Deployed and managed JBoss/EAP applications at Macy's, Disney, Bank of America |
| Jira / Confluence | Daily Jira workflow management, Confluence documentation, Agile ceremonies |
| Bitbucket (Git) | Git flow strategies, PR workflows, branch policies across Bitbucket, GitHub, Azure Repos |
| Kubernetes | AKS, EKS, OpenShift - cluster management, upgrades, networking, RBAC, troubleshooting |
| Agile practices | Scrum/Kanban across every role; daily standups, sprint planning, backlog grooming, retrospectives |

---

## API Development & RESTful Services

### Q: How do you configure RESTful APIs on OpenShift?

**A:** My approach to deploying RESTful APIs on OpenShift:

**1. Application Configuration:**
```yaml
# DeploymentConfig for Java API on OpenShift
apiVersion: apps.openshift.io/v1
kind: DeploymentConfig
metadata:
  name: payment-api
  namespace: production
spec:
  replicas: 3
  strategy:
    type: Rolling
    rollingParams:
      maxUnavailable: 1
      maxSurge: 1
  template:
    spec:
      containers:
        - name: payment-api
          image: registry.internal/payment-api:1.2.3
          ports:
            - containerPort: 8080
              protocol: TCP
          env:
            - name: JAVA_OPTS
              value: "-Xms512m -Xmx1024m -XX:+UseG1GC"
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: payment-db-secret
                  key: host
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
            limits:
              cpu: 1000m
              memory: 2Gi
          readinessProbe:
            httpGet:
              path: /api/health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /api/health
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 15
```

**2. Service & Route Configuration:**
```yaml
# Service
apiVersion: v1
kind: Service
metadata:
  name: payment-api-svc
spec:
  selector:
    app: payment-api
  ports:
    - port: 8080
      targetPort: 8080
  type: ClusterIP

# Route (OpenShift-specific)
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: payment-api-route
spec:
  host: payment-api.apps.cluster.internal
  to:
    kind: Service
    name: payment-api-svc
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

**3. 3Scale Integration:**
- Configure the API in 3Scale admin portal
- Create application plans with rate limits
- Set up API key or OAuth2 authentication
- Map the backend to the OpenShift service URL
- Deploy APIcast gateway on OpenShift connecting to 3Scale

### Q: How do you design RESTful APIs for enterprise use?

**A:** Key design principles I follow:

1. **Resource-oriented URLs**: `/api/v1/payments/{id}`, `/api/v1/customers/{id}/orders`
2. **Proper HTTP methods**: GET (read), POST (create), PUT (full update), PATCH (partial update), DELETE (remove)
3. **Versioning**: URL-based versioning (`/api/v1/`, `/api/v2/`) for backward compatibility
4. **Error handling**: Consistent error response format with HTTP status codes, error codes, and messages
5. **Pagination**: Cursor-based or offset pagination for list endpoints
6. **HATEOAS**: Hypermedia links for API discoverability
7. **Idempotency**: Idempotency keys for POST/PUT operations to handle retries safely
8. **Documentation**: OpenAPI/Swagger spec maintained alongside code

```json
// Standard error response format
{
  "status": 400,
  "error": "VALIDATION_ERROR",
  "message": "Invalid payment amount",
  "details": [
    { "field": "amount", "issue": "Must be greater than 0" }
  ],
  "traceId": "abc-123-def-456",
  "timestamp": "2025-03-25T10:30:00Z"
}
```

### Q: How do you handle API versioning and backward compatibility?

**A:**
- **URL versioning**: `/api/v1/` -> `/api/v2/` for breaking changes
- **Deprecation policy**: v1 continues running for 6-12 months after v2 launch
- **Feature flags**: Toggle new behavior without versioning for non-breaking changes
- **Contract testing**: Pact tests to ensure backward compatibility
- **3Scale routing**: Route traffic between API versions using 3Scale mapping rules

---

## Red Hat OpenShift Configuration & Management

### Q: How do you configure an OpenShift cluster for API workloads?

**A:** At Fineos and Bank of America, my OpenShift configuration covered:

**Cluster Setup:**
1. **Node sizing**: Worker nodes sized for Java workloads (4 vCPU / 16GB RAM minimum for JBoss apps)
2. **Namespace strategy**: Separate namespaces per environment (dev, staging, prod) and per team
3. **RBAC**: OpenShift Groups mapped to AD groups; Role/ClusterRole bindings for least-privilege
4. **Network policies**: Default-deny between namespaces; explicit allow rules for API traffic
5. **Image registry**: Internal OpenShift registry + JFrog Artifactory for vetted base images
6. **Resource quotas**: Per-namespace CPU/memory quotas to prevent noisy neighbor issues

**Project/Namespace Configuration:**
```bash
# Create project with resource quotas
oc new-project payment-api-prod
oc apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: api-quota
  namespace: payment-api-prod
spec:
  hard:
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.cpu: "16"
    limits.memory: 32Gi
    pods: "20"
    services: "10"
EOF

# LimitRange for default container limits
oc apply -f - <<EOF
apiVersion: v1
kind: LimitRange
metadata:
  name: api-limits
spec:
  limits:
    - default:
        cpu: 1000m
        memory: 2Gi
      defaultRequest:
        cpu: 500m
        memory: 1Gi
      type: Container
EOF
```

### Q: How do you manage OpenShift upgrades?

**A:**
1. **Pre-upgrade**: Review release notes, check deprecated APIs, test in staging cluster first
2. **Backup**: etcd snapshot, persistent volume backups
3. **Upgrade path**: Follow Red Hat's supported upgrade path (e.g., 4.12 -> 4.13 -> 4.14)
4. **Rolling upgrade**: Control plane first, then worker nodes in batches
5. **Canary nodes**: Upgrade a subset of workers, validate workloads, then proceed
6. **Validation**: Run smoke tests against all critical APIs after each batch
7. **Rollback plan**: etcd restore if control plane fails; node rollback if worker issues

### Q: How do you handle OpenShift node failures?

**A:**
1. **Detection**: Prometheus alerts on node NotReady status
2. **Impact assessment**: Which pods were on the failed node? Are replicas running on other nodes?
3. **Automatic recovery**: Pod rescheduling to healthy nodes (if resource headroom exists)
4. **Manual intervention**: If node won't recover - cordon, drain, replace
5. **Root cause**: Check node logs, kubelet status, underlying VM/hardware health
6. **Post-incident**: Update capacity planning if resource headroom was insufficient

```bash
# Troubleshooting node issues
oc get nodes                           # Check node status
oc describe node <node-name>           # Check conditions, events
oc adm top nodes                       # Resource usage
oc get pods --all-namespaces --field-selector spec.nodeName=<node> # Pods on node
oc adm cordon <node>                   # Prevent new scheduling
oc adm drain <node> --ignore-daemonsets --delete-emptydir-data  # Evacuate pods
```

---

## 3Scale API Management

### Q: How do you configure 3Scale for API management?

**A:** 3Scale has three main components I configure:

**1. Admin Portal (API Configuration):**
- **Create API Product**: Define the API (payment-api), set backend URL (OpenShift service)
- **Application Plans**: Define rate limits, features, and pricing tiers
  - Free tier: 100 requests/hour
  - Standard: 10,000 requests/hour
  - Premium: unlimited with SLA guarantees
- **Mapping Rules**: Map URL paths to methods for analytics tracking
- **Authentication**: Configure OAuth2, API key, or OIDC based on security requirements

**2. APIcast Gateway (Traffic Management):**
```yaml
# APIcast deployment on OpenShift
apiVersion: apps.openshift.io/v1
kind: DeploymentConfig
metadata:
  name: apicast-production
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: apicast
          image: registry.redhat.io/3scale-amp2/apicast-gateway-rhel8
          env:
            - name: THREESCALE_PORTAL_ENDPOINT
              value: "https://access-token@admin.3scale.internal"
            - name: APICAST_CONFIGURATION_CACHE
              value: "300"  # Cache config for 5 minutes
            - name: APICAST_RESPONSE_CODES
              value: "true"  # Track response codes
          ports:
            - containerPort: 8080  # HTTP
            - containerPort: 8443  # HTTPS
```

**3. Developer Portal:**
- Publish API documentation (OpenAPI/Swagger)
- Self-service API key provisioning
- Usage analytics dashboard for API consumers
- Interactive API testing console

### Q: How do you monitor 3Scale?

**A:**
- **3Scale Analytics**: Built-in dashboard showing hits, response codes, latency per API/plan
- **Prometheus metrics**: APIcast exposes metrics at `/metrics` endpoint
  - `apicast_status` - HTTP response codes
  - `total_response_time_seconds` - Latency histograms
  - `upstream_response_time_seconds` - Backend latency
- **Grafana dashboards**: Custom dashboards for:
  - API traffic patterns (requests/second, peak hours)
  - Error rates by API endpoint
  - Latency percentiles (p50, p95, p99)
  - Rate limit hits per application plan
  - Top consumers by usage
- **Alerting**: Prometheus alerts for error rate spikes, latency degradation, rate limit exhaustion

### Q: How do you handle API rate limiting and throttling?

**A:** In 3Scale:
1. **Application Plans**: Define rate limits per plan (hourly, daily, monthly)
2. **Custom policies**: Lua-based policies for more granular control (per-endpoint limits)
3. **Backend rate limiting**: Additional limits at the application level (e.g., Spring Cloud Gateway)
4. **Response headers**: Return `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`
5. **Graceful degradation**: Return `429 Too Many Requests` with `Retry-After` header
6. **Monitoring**: Alert when consumers consistently hit rate limits (may need plan upgrade)

---

## JBoss / EAP Configuration & Management

### Q: How do you configure JBoss/EAP for production API workloads?

**A:** At Macy's and Bank of America, I managed JBoss/EAP extensively:

**Standalone Configuration (standalone.xml):**
```xml
<!-- JVM tuning for API workloads -->
<jvm name="default">
  <heap size="1024m" max-size="2048m"/>
  <jvm-options>
    <option value="-XX:+UseG1GC"/>
    <option value="-XX:MaxGCPauseMillis=200"/>
    <option value="-XX:+HeapDumpOnOutOfMemoryError"/>
    <option value="-XX:HeapDumpPath=/var/log/jboss/heapdump"/>
  </jvm-options>
</jvm>

<!-- Datasource configuration -->
<datasource jndi-name="java:jboss/datasources/PaymentDS"
            pool-name="PaymentDS" enabled="true">
  <connection-url>jdbc:sqlserver://db-host:1433;databaseName=PaymentDB</connection-url>
  <driver>sqlserver</driver>
  <security>
    <user-name>${env.DB_USER}</user-name>
    <password>${env.DB_PASSWORD}</password>
  </security>
  <pool>
    <min-pool-size>10</min-pool-size>
    <max-pool-size>50</max-pool-size>
  </pool>
  <validation>
    <valid-connection-checker class-name="org.jboss.jca.adapters.jdbc.extensions.mssql.MSSQLValidConnectionChecker"/>
    <background-validation>true</background-validation>
    <background-validation-millis>60000</background-validation-millis>
  </validation>
  <timeout>
    <blocking-timeout-millis>30000</blocking-timeout-millis>
    <idle-timeout-minutes>5</idle-timeout-minutes>
  </timeout>
</datasource>
```

**Containerized JBoss on OpenShift:**
```dockerfile
FROM registry.redhat.io/jboss-eap-7/eap74-openjdk11-openshift-rhel8
COPY target/payment-api.war /opt/eap/standalone/deployments/
COPY standalone-openshift.xml /opt/eap/standalone/configuration/
ENV JAVA_OPTS_APPEND="-Xms512m -Xmx1024m -XX:+UseG1GC"
```

### Q: How do you monitor JBoss/EAP?

**A:**

| What to Monitor | How | Alert Threshold |
|----------------|-----|-----------------|
| JVM Heap usage | JMX + Prometheus JMX Exporter | >80% sustained |
| Thread pool usage | JMX / management CLI | Active threads >90% of max |
| Datasource pool | JMX (`ActiveCount`, `AvailableCount`) | Available <10% of max |
| HTTP response times | Access logs + Prometheus | p95 >2s |
| Error rates | Access logs + application logs | >1% 5xx rate |
| GC pauses | GC logs + Prometheus | GC pause >500ms |
| Deployment status | Management API | Deployment failed |
| Connection leaks | Datasource metrics | Leak count >0 |

**JBoss CLI monitoring commands:**
```bash
# Check datasource pool stats
/subsystem=datasources/data-source=PaymentDS/statistics=pool:read-resource(include-runtime=true)

# Check active deployments
/deployment=*:read-resource

# Check server status
:read-attribute(name=server-state)

# Check thread stats
/core-service=platform-mbean/type=threading:read-resource(include-runtime=true)
```

### Q: How do you troubleshoot JBoss performance issues?

**A:**
1. **Thread dumps**: `jstack <pid>` or JBoss CLI `/core-service=platform-mbean/type=threading:dump-all-threads`
   - Look for: blocked threads, deadlocks, stuck I/O operations
2. **Heap dumps**: `jmap -dump:live,format=b,file=heap.bin <pid>`
   - Analyze with Eclipse MAT for memory leaks
3. **GC analysis**: Enable `-verbose:gc -Xloggc:/var/log/jboss/gc.log`
   - Use GCViewer to identify GC pressure
4. **Slow queries**: Enable datasource statistics, check `AverageGetTime` and `MaxWaitTime`
5. **Access logs**: Analyze response time distribution, identify slow endpoints
6. **Connection pool exhaustion**: Check `WaitCount` - if high, increase pool or find leaking connections

---

## CI/CD with Jenkins & Tekton

### Q: How do you configure a CI/CD pipeline for Java API deployments on OpenShift?

**A:**

**Jenkins Pipeline (Groovy):**
```groovy
pipeline {
    agent any
    environment {
        MAVEN_OPTS = '-Xmx1024m'
        OPENSHIFT_CLUSTER = 'https://api.cluster.internal:6443'
        IMAGE_REGISTRY = 'image-registry.openshift-image-registry.svc:5000'
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://bitbucket.org/team/payment-api.git'
            }
        }
        stage('Build & Test') {
            steps {
                sh 'mvn clean package -DskipTests=false'
                junit '**/target/surefire-reports/*.xml'
            }
        }
        stage('Code Quality') {
            parallel {
                stage('SonarQube') {
                    steps {
                        sh 'mvn sonar:sonar -Dsonar.host.url=$SONAR_URL'
                    }
                }
                stage('Dependency Check') {
                    steps {
                        sh 'mvn org.owasp:dependency-check-maven:check'
                    }
                }
            }
        }
        stage('Build Image') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject('payment-api-dev') {
                            def bc = openshift.selector('bc', 'payment-api')
                            bc.startBuild('--from-dir=.', '--wait')
                        }
                    }
                }
            }
        }
        stage('Deploy to Dev') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject('payment-api-dev') {
                            openshift.selector('dc', 'payment-api').rollout().latest()
                            openshift.selector('dc', 'payment-api').rollout().status()
                        }
                    }
                }
            }
        }
        stage('Integration Tests') {
            steps {
                sh 'mvn verify -Pintegration-test -Dapi.url=https://payment-api-dev.apps.cluster.internal'
            }
        }
        stage('Deploy to Staging') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.tag('payment-api-dev/payment-api:latest', 'payment-api-staging/payment-api:latest')
                        openshift.withProject('payment-api-staging') {
                            openshift.selector('dc', 'payment-api').rollout().latest()
                        }
                    }
                }
            }
        }
        stage('Deploy to Prod') {
            when { branch 'main' }
            input { message 'Deploy to production?' }
            steps {
                script {
                    openshift.withCluster() {
                        openshift.tag('payment-api-staging/payment-api:latest', 'payment-api-prod/payment-api:latest')
                        openshift.withProject('payment-api-prod') {
                            openshift.selector('dc', 'payment-api').rollout().latest()
                            openshift.selector('dc', 'payment-api').rollout().status()
                        }
                    }
                }
            }
        }
    }
    post {
        failure {
            slackSend channel: '#api-team', message: "Build FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
        success {
            jiraComment issueKey: env.JIRA_TICKET, body: "Deployed ${env.BUILD_NUMBER} to production"
        }
    }
}
```

**Tekton Pipeline (OpenShift Pipelines):**
```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: payment-api-pipeline
spec:
  workspaces:
    - name: source
    - name: maven-settings
  params:
    - name: git-url
    - name: git-revision
    - name: image-name
  tasks:
    - name: fetch-source
      taskRef:
        name: git-clone
        kind: ClusterTask
      workspaces:
        - name: output
          workspace: source
      params:
        - name: url
          value: $(params.git-url)
        - name: revision
          value: $(params.git-revision)

    - name: maven-build
      taskRef:
        name: maven
        kind: ClusterTask
      runAfter: [fetch-source]
      workspaces:
        - name: source
          workspace: source
        - name: maven-settings
          workspace: maven-settings
      params:
        - name: GOALS
          value: ["clean", "package", "-DskipTests=false"]

    - name: build-image
      taskRef:
        name: buildah
        kind: ClusterTask
      runAfter: [maven-build]
      workspaces:
        - name: source
          workspace: source
      params:
        - name: IMAGE
          value: $(params.image-name)

    - name: deploy
      taskRef:
        name: openshift-client
        kind: ClusterTask
      runAfter: [build-image]
      params:
        - name: SCRIPT
          value: |
            oc set image dc/payment-api payment-api=$(params.image-name)
            oc rollout latest dc/payment-api
            oc rollout status dc/payment-api
```

### Q: Jenkins vs Tekton - when do you use each?

**A:**
| Aspect | Jenkins | Tekton |
|--------|---------|--------|
| Best for | Complex orchestration, legacy integrations | Cloud-native, OpenShift-native pipelines |
| Execution | Runs on Jenkins agents (VMs/pods) | Runs as K8s pods natively |
| Configuration | Groovy Jenkinsfile | YAML CRDs |
| State | Persistent (Jenkins controller stores history) | Ephemeral (logs in pods, results in CRDs) |
| Scalability | Agent-based scaling | K8s pod autoscaling |
| My usage | Fineos, Bank of America, Wells Fargo, Keysight, Macy's | Fineos (OpenShift Pipelines) |

---

## Monitoring & Observability

### Q: What do you monitor when supporting API services in production?

**A:** I use the **RED method** (Rate, Errors, Duration) plus infrastructure metrics:

**API-Level Metrics:**
| Metric | Tool | Alert Threshold | Action |
|--------|------|-----------------|--------|
| Request rate (RPS) | Prometheus + 3Scale analytics | >50% deviation from baseline | Investigate traffic spike or drop |
| Error rate (4xx, 5xx) | Prometheus + access logs | >1% 5xx, >5% 4xx | Check app logs, DB connectivity |
| Response latency (p50/p95/p99) | Prometheus histograms | p95 >2s, p99 >5s | Thread dumps, slow query analysis |
| Throughput | 3Scale + Grafana | Below SLA threshold | Scale pods, check bottlenecks |

**Infrastructure Metrics:**
| Metric | Tool | Alert Threshold | Action |
|--------|------|-----------------|--------|
| Pod CPU/memory | Prometheus + cAdvisor | >80% sustained | HPA scaling or resource increase |
| JVM heap usage | JMX Exporter | >85% after GC | Heap dump analysis, memory leak investigation |
| DB connection pool | JMX Exporter | Available <10% | Find connection leaks, increase pool |
| Pod restarts | kube-state-metrics | >3 in 15 min | Check OOMKilled, liveness probe failures |
| Node resources | node-exporter | >85% CPU or memory | Node scaling, pod redistribution |
| Disk usage | node-exporter | >80% capacity | Log rotation, PV expansion |

**Grafana Dashboard Layout:**
1. **Overview panel**: Total RPS, overall error rate, average latency
2. **Per-endpoint panel**: Breakdown by API path showing individual performance
3. **Infrastructure panel**: Pod count, CPU/memory, HPA status
4. **JVM panel**: Heap, GC pauses, thread count, class loading
5. **Database panel**: Query time, connection pool, slow queries
6. **3Scale panel**: Rate limit utilization, top consumers, plan usage

### Q: How do you set up alerting?

**A:**
```yaml
# Prometheus alert rules for API services
groups:
  - name: api-alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
          /
          sum(rate(http_server_requests_seconds_count[5m])) > 0.01
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "API error rate above 1% for 5 minutes"
          runbook: "https://confluence.internal/runbooks/high-error-rate"

      - alert: HighLatency
        expr: |
          histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "API p95 latency above 2 seconds"

      - alert: PodRestarting
        expr: |
          increase(kube_pod_container_status_restarts_total{namespace="payment-api-prod"}[15m]) > 3
        labels:
          severity: critical
        annotations:
          summary: "Pod restarting frequently - possible crash loop"

      - alert: JVMHeapHigh
        expr: |
          jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} > 0.85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "JVM heap usage above 85% for 10 minutes"
```

---

## Security Management

### Q: How do you manage security for API services?

**A:** Multi-layered security approach:

**1. Authentication & Authorization:**
- **3Scale**: API key validation, OAuth2/OIDC token validation at gateway level
- **JWT tokens**: Validated at both 3Scale gateway and application level
- **RBAC**: OpenShift RBAC for cluster access; application-level role-based access
- **Service accounts**: Dedicated SA per application with minimum permissions

**2. Secrets Management:**
- **CyberArk Conjur**: Pipeline credentials, DB passwords, API keys - never in code or config maps
- **OpenShift Secrets**: Encrypted at rest, mounted as volumes or env vars
- **External Secrets Operator**: Syncs from Conjur/Vault to K8s Secrets automatically
- **Rotation**: Automated credential rotation via Conjur with zero-downtime (app reconnects with new creds)

**3. Network Security:**
- **Network Policies**: Default-deny between namespaces; explicit ingress/egress rules
- **TLS everywhere**: Edge termination at Route, re-encryption to backend
- **3Scale mTLS**: Mutual TLS between APIcast and backend services
- **Egress controls**: Whitelist outbound connections via NetworkPolicy or OpenShift Egress

**4. Container Security:**
- **Image scanning**: Trivy/Prisma Cloud in pipeline - block on High/Critical CVEs
- **Non-root execution**: SecurityContext enforcing non-root, read-only rootfs
- **SCC (Security Context Constraints)**: Restricted SCC applied to all API pods
- **Image source**: Only pull from approved internal registry

**5. SAST/DAST/SCA in Pipeline:**
- **SAST**: Checkmarx/SonarQube scanning Java code for OWASP Top 10
- **SCA**: Black Duck / OWASP Dependency Check for vulnerable Maven dependencies
- **DAST**: OWASP ZAP against staging endpoints
- **Security gate**: Pipeline fails if High/Critical findings exist

### Q: How do you handle a security vulnerability in a running API?

**A:**
1. **Assess severity**: CVSS score, exploitability, internet-facing vs internal
2. **Immediate mitigation**: If actively exploited:
   - 3Scale rate limiting or IP blocking
   - WAF rule to block exploit pattern
   - Network policy to restrict access
3. **Patch**: Develop fix, test in staging, expedited deployment
4. **Verify**: Confirm vulnerability is resolved with targeted scan
5. **Post-incident**: Update dependencies, add regression test, update Jira with full timeline
6. **Communication**: Notify stakeholders via Confluence incident report

### Q: How do you handle a compromised API key?

**A:**
1. **Revoke immediately**: Disable the key in 3Scale admin portal
2. **Rotate**: Generate new key, distribute to legitimate consumer
3. **Audit**: Check 3Scale analytics for unauthorized usage patterns
4. **Block**: If malicious actor identified, block their IP/account in 3Scale
5. **Investigate**: How was the key compromised? Code repo, logs, config file?
6. **Prevent**: Move to Conjur-managed secrets, implement key expiration policies

---

## Outage Response & Rollback Strategies

### Q: What steps do you take during an API outage?

**A:** I follow a structured incident response process:

**1. Detection & Assessment (0-5 minutes):**
```bash
# Quick health check
oc get pods -n payment-api-prod                    # Pod status
oc get events -n payment-api-prod --sort-by=.lastTimestamp  # Recent events
oc logs -f dc/payment-api -n payment-api-prod      # Application logs
oc adm top pods -n payment-api-prod                # Resource usage
```
- Check Grafana dashboards for error rate, latency spikes
- Check 3Scale analytics for traffic patterns
- Identify scope: single pod, all pods, entire namespace, cluster-wide?

**2. Initial Triage (5-15 minutes):**

| Symptom | Likely Cause | Quick Fix |
|---------|-------------|-----------|
| Pods in CrashLoopBackOff | OOM, config error, dependency failure | Check logs, describe pod, check resource limits |
| High error rate, pods running | DB connection issue, external dependency down | Check DB connectivity, dependent services |
| High latency | Resource contention, GC pressure, slow queries | Thread dump, DB query analysis |
| No traffic reaching pods | Route/Service misconfiguration, 3Scale issue | Check route, service endpoints, APIcast logs |
| All pods evicted | Node failure, resource quota exceeded | Check node status, resource quotas |

**3. Communication:**
- Update incident Jira ticket with status
- Notify team via Slack/Teams channel
- If customer-impacting: notify stakeholders and update status page

**4. Resolution & Rollback (if needed):**
- Apply fix if root cause is clear
- If fix is unclear: **rollback to last known good version**

**5. Post-Incident:**
- Blameless post-mortem document in Confluence
- Update runbooks with new failure scenario
- Create Jira tickets for preventive measures

### Q: How do you roll back a bad deployment?

**A:** Multiple rollback strategies depending on the deployment method:

**OpenShift DeploymentConfig Rollback:**
```bash
# View deployment history
oc rollout history dc/payment-api -n payment-api-prod

# Rollback to previous version
oc rollout undo dc/payment-api -n payment-api-prod

# Rollback to specific revision
oc rollout undo dc/payment-api --to-revision=5 -n payment-api-prod

# Verify rollback
oc rollout status dc/payment-api -n payment-api-prod
oc get pods -n payment-api-prod
```

**ArgoCD GitOps Rollback:**
```bash
# Revert the Git commit that caused the issue
git revert HEAD
git push origin main
# ArgoCD auto-syncs to the reverted state

# Or manual ArgoCD rollback
argocd app rollback payment-api
```

**3Scale API Rollback (if API config change caused issue):**
- Promote previous staging config to production in 3Scale admin portal
- Or revert APIcast deployment to previous image tag

**Database Migration Rollback:**
```bash
# Flyway rollback (if using Flyway for DB migrations)
mvn flyway:undo

# Or apply a reverse migration script
psql -h db-host -d payment_db -f rollback_v2.1.sql
```

### Q: How do you ensure zero-downtime deployments?

**A:**
1. **Rolling updates**: `maxUnavailable: 0, maxSurge: 1` - new pods start before old ones terminate
2. **Readiness probes**: New pods only receive traffic after health check passes
3. **Pre-stop hooks**: Graceful shutdown allowing in-flight requests to complete
4. **Database compatibility**: Forward-compatible migrations - new code works with old schema and vice versa
5. **Feature flags**: Toggle new features independently of deployment
6. **Blue-green**: Maintain two production environments; switch Route to new version after validation
7. **Canary**: Route 10% of traffic to new version via 3Scale; monitor before full rollout

```yaml
# Zero-downtime deployment config
spec:
  strategy:
    type: Rolling
    rollingParams:
      maxUnavailable: 0      # Never take pods away before new ones are ready
      maxSurge: 1             # Add one pod at a time
      timeoutSeconds: 600
  template:
    spec:
      containers:
        - name: payment-api
          readinessProbe:
            httpGet:
              path: /api/health/ready
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 5
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 10"]  # Allow LB to drain connections
```

---

## RDBMS & Database Operations

### Q: How do you manage database operations for API services?

**A:**

**Connection Management:**
- Connection pooling via JBoss/EAP datasource (min 10, max 50 per pod)
- Connection validation with background checker to remove stale connections
- Separate read/write connection pools for read-replica architectures

**Performance Monitoring:**
```sql
-- SQL Server: Find slow queries
SELECT TOP 10
    qs.total_elapsed_time / qs.execution_count AS avg_elapsed_time,
    qs.execution_count,
    SUBSTRING(qt.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(qt.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2)+1) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
ORDER BY avg_elapsed_time DESC;

-- PostgreSQL: Active connections and locks
SELECT pid, usename, state, query, query_start
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;

-- Check for blocking locks
SELECT blocked_locks.pid AS blocked_pid,
       blocking_locks.pid AS blocking_pid,
       blocked_activity.query AS blocked_query
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_locks blocking_locks
  ON blocking_locks.locktype = blocked_locks.locktype
WHERE NOT blocked_locks.granted;
```

**Database Migration Strategy:**
- Flyway or Liquibase for version-controlled schema migrations
- Forward-compatible migrations only (additive changes)
- Test migrations in staging with production-like data volumes
- Rollback scripts prepared before every migration
- Blue-green database strategy for major schema changes

### Q: How do you handle database connection failures?

**A:**
1. **Circuit breaker**: Application-level circuit breaker (Resilience4j) prevents cascading failures
2. **Retry logic**: Exponential backoff for transient failures
3. **Connection pool recovery**: JBoss validates connections on checkout; stale connections evicted
4. **Monitoring**: Alert on connection pool exhaustion, query timeouts, replication lag
5. **Failover**: Read replicas for read traffic; automatic failover for primary (Azure SQL HA / RDS Multi-AZ)

---

## Jira & Confluence Documentation

### Q: How do you manage Jira workflows for API development?

**A:** I follow standard Agile Jira workflows:

**Story Lifecycle:**
`Backlog -> Sprint Planning -> In Progress -> Code Review -> QA -> Done`

**Bug Workflow:**
`New -> Triaged -> In Progress -> Fix Verified -> Closed`

**My Jira Practices:**
- **Estimation**: Story points using Fibonacci (1, 2, 3, 5, 8, 13)
- **Labels**: `api`, `security`, `performance`, `tech-debt`, `documentation`
- **Components**: Per API service (payment-api, order-api, customer-api)
- **Sprint reports**: Burndown charts, velocity tracking, carried-over stories
- **Linking**: Stories linked to Bitbucket PRs, Confluence pages, and deployment records
- **Automation**: Jira transitions triggered by pipeline events (PR merged -> In Review, deploy successful -> Done)

**Confluence Documentation I Maintain:**
- API documentation (OpenAPI specs + usage guides)
- Architecture Decision Records (ADRs)
- Runbooks for each service (startup, shutdown, troubleshooting)
- Incident post-mortems
- Onboarding guides for new team members
- Sprint retrospective notes
- Release notes

---

## Agile & Team Collaboration

### Q: How do you work in an Agile environment?

**A:** Across all my roles, I've operated in Scrum and Kanban environments:

**Daily Practices:**
- **Daily standup**: What I did, what I'm doing, blockers - keep it under 2 minutes
- **PR reviews**: Review team members' code within 4 hours; provide constructive feedback
- **Pair programming**: For complex tasks or knowledge transfer

**Sprint Ceremonies:**
- **Sprint planning**: Break epics into stories, estimate, commit to sprint goals
- **Backlog grooming**: Clarify requirements, identify technical dependencies, split large stories
- **Sprint review**: Demo completed work to stakeholders
- **Retrospective**: What went well, what to improve, action items

**Communication Style:**
- Technical discussions: Provide clear, concise explanations with diagrams when needed
- Non-technical stakeholders: Focus on business impact, timelines, and risk
- Written communication: Confluence pages with clear headers, screenshots, and examples
- Remote meetings: Camera on, active participation, follow up with Slack/email summary

### Q: How do you handle disagreements with developers about approach?

**A:**
1. **Listen first**: Understand their perspective and constraints
2. **Data-driven**: Present metrics, benchmarks, or production evidence to support my position
3. **Prototype**: Build a quick POC of both approaches to compare objectively
4. **Compromise**: Find middle ground that addresses both concerns
5. **Escalate constructively**: If no agreement, bring to tech lead/architect with both options documented
6. **Document**: Record the decision and rationale in an ADR (Architecture Decision Record) on Confluence

---

## Scenario-Based Questions

### Q: An API endpoint is returning 503 errors intermittently. How do you investigate?

**A:**
1. **Check pod status**: `oc get pods` - any pods in CrashLoopBackOff or not ready?
2. **Check logs**: `oc logs dc/payment-api` - look for exceptions, connection errors, OOM
3. **Check events**: `oc get events` - any pod evictions, failed scheduling, image pull errors?
4. **Check resource usage**: `oc adm top pods` - CPU/memory near limits?
5. **Check 3Scale**: Is APIcast healthy? Any upstream timeout errors?
6. **Check DB**: Connection pool stats, slow queries, database health
7. **Check dependencies**: Are downstream services healthy?
8. **Thread dump**: If pods are running but slow, take thread dump to find blocked threads
9. **Recent changes**: Was there a recent deployment, config change, or infra change?
10. **Resolution**: Fix root cause, or rollback if recent change caused it

### Q: You need to deploy a new API version that changes the database schema. How?

**A:**
1. **Backward-compatible migration**: Add new columns/tables (don't modify/delete existing)
2. **Deploy migration**: Run Flyway/Liquibase migration in staging, validate
3. **Deploy new code**: New code handles both old and new schema format
4. **Validate**: Run integration tests against new schema
5. **Promote to prod**: Same migration + code deployment with rollback scripts ready
6. **Cleanup**: After validation period, remove old schema support in next release
7. **Rollback plan**: Reverse migration script tested and ready; old code still works with new schema

### Q: How do you handle a sudden 10x traffic spike to your API?

**A:**
1. **HPA kicks in**: Horizontal Pod Autoscaler scales pods based on CPU/memory/custom metrics
2. **3Scale rate limiting**: Protects backend from overload; returns 429 to excess traffic
3. **Connection pool**: May need temporary increase if DB becomes bottleneck
4. **Caching**: Enable/increase API response caching (Redis, 3Scale caching policy)
5. **Monitor**: Watch error rates, latency, DB connections closely during spike
6. **Post-spike**: Review capacity planning, adjust HPA thresholds, consider CDN for static content
7. **Root cause**: Was this expected (marketing campaign)? Or unexpected (bot traffic, DDoS)?

---

## My Experience Talking Points

### Opening Statement
"I'm a Senior DevOps/Platform Engineer with 10+ years of experience supporting Java-based API services on Red Hat OpenShift, JBoss/EAP, and Kubernetes. I've built and managed CI/CD pipelines with Jenkins and Tekton, configured 3Scale API management for enterprise API gateways, and implemented comprehensive monitoring and security across the full API lifecycle. I have deep experience with Maven builds, RDBMS management (SQL Server, PostgreSQL, Oracle), and working in Agile teams using Jira, Confluence, and Bitbucket."

### Key Experiences to Highlight
1. **Fineos**: OpenShift 4.x operations, Tekton pipelines, ArgoCD GitOps, Helm deployments, HIPAA compliance
2. **Bank of America**: OpenShift, Jenkins CI/CD, JBoss deployments, PCI-DSS security, container hardening
3. **Macy's**: JBoss/WebSphere middleware engineering, J2EE deployments, Chef automation, performance tuning
4. **Walt Disney**: JBoss/WebSphere, UrbanCode -> Chef+Jenkins migration, multi-platform CI/CD
5. **Navy Federal**: AKS/Kubernetes, Azure DevOps, CyberArk Conjur secrets management, 3Scale-equivalent API security (Kong)
6. **All roles**: Git/Bitbucket workflows, Jira/Confluence documentation, Agile ceremonies, cross-team collaboration
