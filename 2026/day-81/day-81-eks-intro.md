# Day 81 -- Introduction to Amazon EKS with Terraform

## Task
You had been running Kubernetes locally with Kind. That worked for learning, but the AI-BankApp needed a production environment -- managed control plane, auto-scaling nodes, persistent EBS storage, and IAM integration.

Amazon EKS (Elastic Kubernetes Service) is AWS's managed Kubernetes offering. The AI-BankApp project (https://github.com/TrainWithShubham/AI-BankApp-DevOps, branch `feat/gitops`) already had a complete Terraform configuration in its `terraform/` directory that provisions a production-grade EKS cluster. Today you understood EKS architecture, studied the Terraform configs, provisioned the cluster, and connected to it.

---

## All project

[AI-BankApp-DevOps](https://github.com/Mujakkir-Pathan/AI-BankApp-DevOps)

---

## All screenshot's

[screenshots of task](screenshots/)

---

## Challenge Tasks

### Task 1: Understand EKS Architecture
Research and write notes on:

1. **What does "managed Kubernetes" mean?**
   - AWS managed the **control plane** (API server, etcd, scheduler, controller manager)
   - I managed the **data plane** (worker nodes where my pods ran)
   - AWS handled control plane upgrades, patching, and high availability across multiple AZs

2. **EKS components:**
   - **EKS Control Plane** -- was managed by AWS, ran in AWS-owned VPC, and was accessible via API endpoint
   - **Node Groups** -- EC2 instances that ran my pods
     - **Managed Node Groups** -- AWS handled provisioning, scaling, and updates
     - **Self-Managed Nodes** -- I managed the EC2 instances myself
     - **Fargate Profiles** -- provided serverless compute with no nodes to manage
   - **VPC and Networking** -- EKS ran inside my VPC with subnets across AZs
   - **IAM Integration** -- EKS used IAM roles for cluster access and pod-level permissions (IRSA)

3. **EKS add-ons the AI-BankApp uses** (from `terraform/eks.tf`):
   - `coredns` -- provided DNS resolution inside the cluster
   - `kube-proxy` -- provided network routing for services
   - `vpc-cni` -- provided AWS VPC CNI networking and assigned VPC IPs to pods
   - `eks-pod-identity-agent` -- enabled pod-level IAM roles
   - `aws-ebs-csi-driver` -- allowed pods to use EBS volumes for MySQL and Ollama storage
   - `metrics-server` -- enabled `kubectl top` and HPA

---

### Task 2: Study the AI-BankApp Terraform Configuration
I examined the `terraform/` directory.

```text
argocd.tf
eks.tf
outputs.tf
provider.tf
terraform.tfvars
variables.tf
vpc.tf
```

**`variables.tf` and `terraform.tfvars`:**

The original configuration used:

```hcl
aws_region         = "us-west-2"
cluster_name       = "bankapp-eks"
cluster_version    = "1.35"
node_instance_type = "t3.medium"
node_desired_count = 3
node_max_count     = 5
```

`t3.medium` was not Free Tier eligible in this environment, so I changed the worker-node instance type to `m7i-flex.large`.

**`vpc.tf`** -- I reviewed the networking foundation:
- It used the `terraform-aws-modules/vpc/aws` module.
- It configured 3 Availability Zones.
- It configured public subnets `10.0.1-3.0/24`.
- It configured private subnets `10.0.4-6.0/24`.
- It configured intra subnets `10.0.7-9.0/24`.
- It enabled a NAT Gateway.
- Public subnets were tagged with `kubernetes.io/role/elb`.
- Private subnets were tagged with `kubernetes.io/role/internal-elb`.

**`eks.tf`** -- I reviewed the cluster configuration:
- It used the `terraform-aws-modules/eks/aws` module `~> 21.0`.
- It configured Amazon Linux 2023 nodes.
- It configured 3 desired/minimum nodes and 5 maximum nodes.
- The final worker-node instance type was `m7i-flex.large`.
- It installed all 6 EKS add-ons.
- It configured IRSA for the EBS CSI driver.
- It enabled public and private API endpoint access.

**`argocd.tf`** -- I reviewed the ArgoCD configuration:
- It installed ArgoCD using the `argo-cd` Helm chart.
- It exposed ArgoCD using a LoadBalancer service.
- It configured ArgoCD to depend on the EKS module.

**`outputs.tf`** -- I reviewed the outputs:
- It provided the `aws eks update-kubeconfig` command.
- It provided the ArgoCD initial password retrieval command.

**Document:** The architecture was:

```text
VPC
├── Public Subnets
├── Private Subnets
│   └── EKS Node Group
│       └── Pods
└── Intra Subnets
    └── EKS Control Plane

EKS Add-ons:
CoreDNS
kube-proxy
VPC CNI
EKS Pod Identity Agent
AWS EBS CSI Driver
Metrics Server
```

---

### Task 3: Provision the EKS Cluster
I verified the required tools:

```text
Terraform 1.16.4
AWS CLI 2.31.35
kubectl 1.37.0
Helm 3.22.0
```

I configured AWS credentials using `aws configure` and verified them with:

```bash
aws sts get-caller-identity
```

I initialized Terraform with:

```bash
terraform init -upgrade
```

I reviewed the Terraform plan.

The first apply attempt failed because `t3.medium` was not Free Tier eligible. I changed the worker-node type to `m7i-flex.large` and applied the configuration successfully.

The final result was:

```text
Apply complete! Resources: 6 added, 0 changed, 1 destroyed.
```

The running cluster was:

```text
Cluster: bankapp-eks
Region: us-west-2
Kubernetes version: 1.35
Worker nodes: 3
Worker type: m7i-flex.large
Minimum/desired nodes: 3
Maximum nodes: 5
```

---

### Task 4: Connect to Your Cluster
I updated kubeconfig:

```bash
aws eks update-kubeconfig --name bankapp-eks --region us-west-2
```

I verified:

```bash
kubectl config current-context
kubectl cluster-info
kubectl get nodes -o wide
```

The cluster had 3 `Ready` nodes running Kubernetes `v1.35.8-eks-f4fc4f1`.

I explored the cluster with:

```bash
kubectl get pods -n kube-system
kubectl get daemonsets -n kube-system
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
kubectl top nodes
```

The EKS system components and add-ons were running successfully.

`kubectl top nodes` returned:

```text
Node 1: 45m CPU, 2% CPU, 635Mi memory, 8% memory
Node 2: 32m CPU, 1% CPU, 897Mi memory, 12% memory
Node 3: 19m CPU, 0% CPU, 644Mi memory, 9% memory
```

I checked ArgoCD:

```bash
kubectl get pods -n argocd
kubectl get svc -n argocd
```

The ArgoCD Pods were running and `argocd-server` was exposed through a LoadBalancer.

I retrieved the admin password using:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

I accessed the ArgoCD LoadBalancer URL in the browser and successfully logged in with the `admin` account.

---

### Task 5: Deploy the AI-BankApp Manually (Before ArgoCD)
I returned to the repository root and applied the raw manifests from the `k8s/` directory:

```bash
kubectl apply -f k8s/namespace.yml
kubectl apply -f k8s/pv.yml
kubectl apply -f k8s/pvc.yml
kubectl apply -f k8s/configmap.yml
kubectl apply -f k8s/secrets.yml
kubectl apply -f k8s/mysql-deployment.yml
kubectl apply -f k8s/service.yml
kubectl apply -f k8s/ollama-deployment.yml
kubectl apply -f k8s/bankapp-deployment.yml
kubectl apply -f k8s/hpa.yml
```

The MySQL, Ollama, and BankApp Pods became `Running`.

I checked the PVCs:

```bash
kubectl get pvc -n bankapp
```

The PVCs were:

```text
mysql-pvc   Bound   5Gi    RWO   gp3
ollama-pvc  Bound   10Gi   RWO   gp3
```

I checked the PVs:

```bash
kubectl get pv
```

Both PVs were `Bound` to the correct PVCs.

I accessed the application using:

```bash
kubectl port-forward svc/bankapp-service -n bankapp 8080:8080
```

I opened `http://localhost:8080`, saw the AI-BankApp login page, and successfully logged in.

**Verify the HPA:**

```bash
kubectl get hpa -n bankapp
```

The result was:

```text
NAME          REFERENCE            TARGETS       MINPODS   MAXPODS   REPLICAS
bankapp-hpa   Deployment/bankapp   cpu: 1%/70%   2         4         2
```

---

### Task 6: Understand EKS Costs and Clean Up Strategy
I reviewed the EKS cost components:

| Component | Cost (approximate) |
|-----------|-------------------|
| EKS Control Plane | $0.10/hr (~$73/month) |
| t3.medium nodes (3x) | ~$0.042/hr each (~$91/month total) |
| NAT Gateway | ~$0.045/hr + data transfer (~$33/month) |
| EBS volumes (15Gi total) | ~$1.50/month |
| LoadBalancer (ArgoCD) | ~$0.025/hr (~$18/month) |
| **Total for this lab** | **~$220/month (~$7/day)** |

The actual worker nodes used `m7i-flex.large`, so the task's `t3.medium` node cost was only an approximate reference for the original configuration.

**Document:** The NAT Gateway was expensive because it had an hourly charge and additional data-processing charges while providing outbound internet access for resources in private subnets.

I deleted the BankApp workload while keeping the EKS cluster for Days 82-83:

```bash
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
```

All resources were deleted successfully.

I did not run:

```bash
cd terraform
terraform destroy
```

because the cluster was required for Days 82-83.

---

## EKS architecture diagram

```text
AWS
└── VPC
    ├── Public Subnets
    │   └── Load Balancer
    │
    ├── Private Subnets
    │   └── EKS Managed Node Group
    │       ├── Node 1
    │       ├── Node 2
    │       └── Node 3
    │           └── Kubernetes Pods
    │
    └── Intra Subnets
        └── EKS Control Plane

EKS Add-ons:
├── CoreDNS
├── kube-proxy
├── VPC CNI
├── EKS Pod Identity Agent
├── AWS EBS CSI Driver
└── Metrics Server
```

## Terraform files explained in my own words

| File | Explanation |
|---|---|
| `variables.tf` | Defined the Terraform input variables. |
| `terraform.tfvars` | Provided the values for the Terraform variables. |
| `provider.tf` | Configured the AWS and Helm providers and local values. |
| `vpc.tf` | Created the VPC, subnets, and NAT Gateway. |
| `eks.tf` | Created the EKS cluster, node group, add-ons, and EBS CSI IAM integration. |
| `argocd.tf` | Installed ArgoCD through Helm and exposed it with a LoadBalancer. |
| `outputs.tf` | Provided cluster information and helper commands. |

## EKS cost breakdown table

| Component | Approximate cost |
|---|---:|
| EKS Control Plane | ~$73/month |
| 3 × `t3.medium` nodes | ~$91/month |
| NAT Gateway | ~$33/month |
| EBS volumes | ~$1.50/month |
| ArgoCD LoadBalancer | ~$18/month |
| **Total for task estimate** | **~$220/month (~$7/day)** |

## ArgoCD login URL and confirmation it is accessible

```text
http://ad58dfaf2b4ec45b1b3be017d86c1392-1942367843.us-west-2.elb.amazonaws.com
```

The ArgoCD URL was accessible, and I successfully logged in with the `admin` account.
