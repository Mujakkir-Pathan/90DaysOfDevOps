# Day 83 -- EKS Project: Production Deployment of AI-BankApp

## Task
Three days of EKS -- cluster provisioning with Terraform, Gateway API networking, EBS storage, and TLS. Today you put it all together and deploy the AI-BankApp as a production-grade application on EKS. Full stack: Spring Boot app with MySQL and Ollama AI, persistent storage, autoscaling, monitoring, and the complete end-to-end validation.

This is the kind of deployment you would do on the job.

Reference: https://github.com/TrainWithShubham/AI-BankApp-DevOps (branch: `feat/gitops`)

---

## project

[AI-BankApp-DevOps](https://github.com/Mujakkir-Pathan/AI-BankApp-DevOps)

---

## All screenshot's

[screenshots of AI-BankApp-DevOps](screenshots/)

---

## Challenge Tasks

### Task 1: Deploy the Complete AI-BankApp Stack
The EKS cluster was running:

```bash
kubectl get nodes
```

The complete application stack was deployed in order:

```bash
cd AI-BankApp-DevOps

# 1. Namespace and storage
kubectl apply -f k8s/namespace.yml
kubectl apply -f k8s/pv.yml
kubectl apply -f k8s/pvc.yml

# 2. Configuration
kubectl apply -f k8s/configmap.yml
kubectl apply -f k8s/secrets.yml

# 3. Database and AI service
kubectl apply -f k8s/mysql-deployment.yml
kubectl apply -f k8s/service.yml
kubectl apply -f k8s/ollama-deployment.yml

# 4. Wait for dependencies
echo "Waiting for MySQL..."
kubectl wait --for=condition=ready pod -l app=mysql -n bankapp --timeout=120s

echo "Waiting for Ollama (this takes 2-5 minutes for model pull)..."
kubectl wait --for=condition=ready pod -l app=ollama -n bankapp --timeout=600s

# 5. Application
kubectl apply -f k8s/bankapp-deployment.yml
kubectl apply -f k8s/hpa.yml

# 6. Wait for BankApp
echo "Waiting for BankApp..."
kubectl wait --for=condition=ready pod -l app=bankapp -n bankapp --timeout=300s
```

The AI-BankApp stack was deployed with MySQL, Ollama, BankApp, persistent storage, services, and HPA.

---

### Task 2: Set Up Gateway API and Access the App
Envoy Gateway was installed and the Gateway configuration was applied.

The Gateway API routed traffic through the AWS Network Load Balancer to the AI-BankApp.

The application was validated through the Gateway using the health endpoint and browser access.

The full stack was running on EKS: Spring Boot served the UI, MySQL stored accounts and transactions, and Ollama's TinyLlama model powered the AI chatbot.

---

### Task 3: Deploy the Monitoring Stack
Prometheus and Grafana were deployed using `kube-prometheus-stack`.

A ServiceMonitor was created to scrape the BankApp metrics from `/actuator/prometheus`.

```yaml
# bankapp-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: bankapp-monitor
  namespace: monitoring
  labels:
    release: monitoring
spec:
  namespaceSelector:
    matchNames:
      - bankapp
  selector:
    matchLabels:
      app: bankapp
  endpoints:
    - port: "8080"
      path: /actuator/prometheus
      interval: 15s
```

```bash
kubectl apply -f bankapp-servicemonitor.yaml
```

Prometheus metrics were queried successfully.

#### PromQL Queries

```promql
# JVM memory usage
jvm_memory_used_bytes{namespace="bankapp"}

# HTTP request rate
rate(http_server_requests_seconds_count{namespace="bankapp"}[5m])

# HTTP request latency (95th percentile)
histogram_quantile(0.95, rate(http_server_requests_seconds_bucket{namespace="bankapp"}[5m]))
```

The following Grafana dashboards were explored:

- **Kubernetes / Compute Resources / Namespace (Pods)** -- `bankapp` namespace
- **Kubernetes / Compute Resources / Pod** -- individual pods
- **Kubernetes / Compute Resources / Nodes Overview** -- EKS worker node health

`Kubernetes / Compute Resources / Nodes Overview` was used as the available equivalent for the task's `Node Exporter / Nodes` dashboard.

---

### Task 4: End-to-End Validation Checklist

The complete validation checklist was passed.

**Application layer:**

- All BankApp, MySQL, and Ollama pods were running and ready.
- The application health endpoint returned `status: UP`.
- HPA was active with 2 minimum replicas and 4 maximum replicas.
- The Prometheus metrics endpoint was working.

**Data layer:**

- MySQL was healthy.
- MySQL PVC was bound to a 5Gi `gp3` volume.
- Ollama PVC was bound to a 10Gi `gp3` volume.
- Ollama had the `tinyllama:latest` model loaded.

**Infrastructure layer:**

- All 3 EKS worker nodes were `Ready`.
- Node CPU and memory usage were healthy.
- The Gateway was `PROGRAMMED=True` and serving traffic through an AWS NLB.
- Monitoring components were running.

**Security layer:**

- BankApp was running as the non-root `devsecops` user.
- The `bankapp-secret` Kubernetes Secret was present for sensitive configuration.

---

### Task 5: Reflect on the Full EKS Journey
The concepts were mapped to the days they were learned:

| Day | What You Built | AI-BankApp Connection |
|-----|---------------|----------------------|
| 81 | EKS cluster via Terraform, kubectl connection, manual deploy | Used the project's `terraform/` configs to provision infra |
| 82 | Gateway API, Envoy, TLS, EBS storage, session persistence | Used `k8s/gateway.yml`, `k8s/cert-manager.yml`, `k8s/pv.yml` |
| 83 | Full production deployment, monitoring, validation | Complete stack: app + DB + AI + networking + observability |

**What the AI-BankApp's EKS setup includes that you have now seen:**
- Terraform-provisioned VPC with 3-AZ networking
- Managed node group with auto-scaling
- 6 EKS add-ons (CoreDNS, VPC CNI, kube-proxy, Pod Identity, EBS CSI, Metrics Server)
- ArgoCD pre-installed (used on Days 84-86)
- Gateway API with Envoy for traffic management
- cert-manager for automated HTTPS
- Cookie-based session persistence for stateful app
- EBS persistent storage for MySQL and Ollama
- HPA with scale-up/down policies
- Spring Boot Actuator metrics for Prometheus
- Init containers for dependency ordering
- PostStart lifecycle hooks for Ollama model pull

**What you would add for a real production deployment:**
- DNS with Route 53 and ExternalDNS
- Network Policies for pod-to-pod isolation
- Pod Disruption Budgets for safe node draining
- External Secrets Operator for AWS Secrets Manager integration
- Database backups (automated MySQL dumps to S3)
- Log aggregation with Loki (you built this on Day 75)
- Multi-environment clusters (dev + prod)

---

### Task 6: Complete Teardown
**This is critical -- do not leave resources running.**

The workloads were deleted first:

```bash
# Delete monitoring
helm uninstall monitoring -n monitoring

# Delete Gateway resources (releases the NLB)
kubectl delete -f k8s/gateway.yml 2>/dev/null

# Delete the BankApp stack
kubectl delete -f k8s/hpa.yml
kubectl delete -f k8s/bankapp-deployment.yml
kubectl delete -f k8s/ollama-deployment.yml
kubectl delete -f k8s/mysql-deployment.yml
kubectl delete -f k8s/service.yml
kubectl delete -f k8s/secrets.yml
kubectl delete -f k8s/configmap.yml
kubectl delete -f k8s/pvc.yml
kubectl delete -f k8s/pv.yml
kubectl delete -f k8s/namespace.yml

# Delete Envoy Gateway
helm uninstall envoy-gateway -n envoy-gateway-system 2>/dev/null

# Delete cert-manager
helm uninstall cert-manager -n cert-manager 2>/dev/null

# Delete namespaces
kubectl delete namespace monitoring envoy-gateway-system cert-manager 2>/dev/null
```

The lingering resource checks were completed:

```bash
# Check for lingering load balancers
kubectl get svc -A | grep LoadBalancer

# Check for lingering PVCs
kubectl get pvc -A
```

The Gateway LoadBalancer was released. The remaining ArgoCD LoadBalancer was Terraform-managed through `helm_release.argocd`.

No PVCs remained.

The infrastructure was destroyed with Terraform:

```bash
cd terraform
terraform destroy
```

The Terraform teardown was completed.

**Verify in the AWS Console:**
- EKS: no clusters
- EC2: no instances, no load balancers, no EBS volumes
- VPC: the `bankapp-eks` VPC is gone
- CloudFormation: no lingering stacks

**Check your AWS bill** in the Billing Dashboard. All charges should stop within the hour.

**Cost for this 3-day lab (approximate):** $15-25 depending on how long you kept the cluster running.

---

## Full Architecture Diagram

```text
                              Internet
                                  |
                                  v
                         AWS Network Load Balancer
                                  |
                                  v
                         Envoy Gateway / NLB
                                  |
                                  v
                           Gateway API
                                  |
                                  v
                             HTTPRoute
                                  |
                                  v
                         BankApp Service
                                  |
                                  v
                    +-------------------------+
                    |      EKS Cluster        |
                    |                         |
                    |  +-------------------+  |
                    |  | Worker Node 1      |  |
                    |  | BankApp Pod        |  |
                    |  +-------------------+  |
                    |                         |
                    |  +-------------------+  |
                    |  | Worker Node 2      |  |
                    |  | BankApp Pod        |  |
                    |  | MySQL Pod          |  |
                    |  +-------------------+  |
                    |                         |
                    |  +-------------------+  |
                    |  | Worker Node 3      |  |
                    |  | Ollama Pod         |  |
                    |  +-------------------+  |
                    +-------------------------+
                              |
                              v
                     VPC / 3-AZ Networking
                              |
              +---------------+---------------+
              |                               |
              v                               v
       EBS gp3 Persistent Storage       Monitoring Stack
       MySQL PVC / Ollama PVC           Prometheus + Grafana
```

---

## Key Takeaways from the 3-Day EKS Block

- **Day 81:** Provisioned an EKS cluster and supporting AWS infrastructure with Terraform and connected to the cluster using `kubectl`.
- **Day 82:** Implemented Gateway API networking with Envoy, automated TLS with cert-manager, EBS persistent storage, and session persistence.
- **Day 83:** Combined the infrastructure, application, database, AI service, networking, autoscaling, monitoring, validation, and teardown into a complete EKS deployment.
- Learned how Spring Boot Actuator exposes application metrics for Prometheus.
- Used HPA for application autoscaling.
- Used EBS-backed PVCs for persistent MySQL and Ollama storage.
- Used Gateway API and Envoy to expose the application through an AWS NLB.
- Validated application, data, infrastructure, and security layers end to end.
- Completed the full AWS resource teardown with Terraform.

---

## Cost Report for the Lab

**Approximate cost for the 3-day EKS lab:** **$15-25**

The cost depended on how long the EKS cluster and AWS resources remained running.

The lab resources were fully torn down after completion to prevent continued infrastructure charges.

---
