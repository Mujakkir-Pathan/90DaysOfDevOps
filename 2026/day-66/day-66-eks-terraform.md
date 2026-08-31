# Day 66 -- Provision an EKS Cluster with Terraform Modules

## Task

Provisioned an AWS EKS cluster using Terraform Registry modules instead of creating the Kubernetes infrastructure manually. The setup included a VPC, EKS cluster, managed node group, `kubectl` connectivity, and an Nginx workload.

The complete infrastructure was provisioned through Terraform and successfully destroyed after the exercise.

---

## All tf files

[terrform files](https://github.com/Mujakkir-Pathan/terraform-terraweak-90day/tree/main/day-6)

---

## All sceenshots

[screenshots of task](screenshots/)

---

## Challenge Tasks

### Task 1: Project Setup

Created the `terraform-eks` project with the required Terraform file structure:

```text
terraform-eks/
  providers.tf
  vpc.tf
  eks.tf
  variables.tf
  outputs.tf
  terraform.tfvars
  k8s/
    nginx-deployment.yaml
```

Configured `providers.tf` with the AWS provider pinned to `~> 5.0` and the Kubernetes provider pinned to `~> 2.0`.

Configured the AWS region through the `region` variable and set the value to `us-east-2` in `terraform.tfvars`.

Defined the required variables in `variables.tf`:

- `region` as a string
- `cluster_name` as a string with default `"terraweek-eks"`
- `cluster_version` as a string with default `"1.31"`
- `node_instance_type` as a string with default `"t3.small"`
- `node_desired_count` as a number with default `2`
- `vpc_cidr` as a string with default `"10.0.0.0/16"`

The original task specified `t3.medium`, but the first EKS deployment failed because the instance type was not eligible for the account's Free Tier. I changed the instance type to `t3.small` and successfully created the node group.

---

### Task 2: Create the VPC with Registry Module

Created the VPC using the `terraform-aws-modules/vpc/aws` Terraform Registry module.

Configured:

- VPC CIDR as `10.0.0.0/16`
- 2 availability zones: `us-east-2a` and `us-east-2b`
- 2 public subnets
- 2 private subnets
- A single NAT Gateway to reduce cost
- DNS hostnames enabled
- Required EKS subnet tags

The public subnets used:

```text
10.0.1.0/24
10.0.2.0/24
```

The private subnets used:

```text
10.0.11.0/24
10.0.12.0/24
```

The EKS subnet tags were configured as:

```hcl
public_subnet_tags = {
  "kubernetes.io/role/elb" = 1
}

private_subnet_tags = {
  "kubernetes.io/role/internal-elb" = 1
}
```

Ran `terraform init` successfully and verified the VPC configuration with `terraform plan`.

**Why does EKS need both public and private subnets?**

Public and private subnets serve different purposes. Public subnets have a route to the Internet Gateway and are suitable for resources that need direct internet-facing connectivity, such as internet-facing load balancers.

Private subnets do not have a direct route to the Internet Gateway. EKS worker nodes were placed in the private subnets so that the nodes were not directly exposed to the public internet. The private nodes could still access the internet for required outbound traffic through the NAT Gateway.

**What do the subnet tags do?**

The subnet tags tell Kubernetes/EKS which subnets can be used for different types of load balancers.

```text
kubernetes.io/role/elb
```

identifies subnets suitable for external load balancers.

```text
kubernetes.io/role/internal-elb
```

identifies subnets suitable for internal load balancers.

---

### Task 3: Create the EKS Cluster with Registry Module

Created the EKS cluster using the `terraform-aws-modules/eks/aws` Registry module version `~> 20.0`.

Configured:

- Cluster name: `terraweek-eks`
- Kubernetes version: `1.31`
- Private VPC subnets for the cluster
- Public cluster endpoint access
- One managed node group named `terraweek_nodes`
- Desired node count: `2`
- Minimum nodes: `1`
- Maximum nodes: `3`
- Node instance type: `t3.small`
- Amazon Linux 2 x86_64 AMI
- Terraform, project, and environment tags

The first plan showed:

```text
Plan: 55 to add, 0 to change, 0 to destroy.
```

The first apply attempt failed because `t3.medium` was not eligible for the account's Free Tier:

```text
InvalidParameterCombination - The specified instance type is not eligible for Free Tier.
```

Destroyed the failed infrastructure, changed the node instance type from `t3.medium` to `t3.small`, and ran the deployment again successfully.

The successful Terraform apply created:

```text
Apply complete! Resources: 55 added, 0 changed, 0 destroyed.
```

Later, 2 additional resources were added to configure EKS cluster creator permissions, bringing the total created resources to **57**.

---

### Task 4: Apply and Connect kubectl

Applied the Terraform configuration and successfully created the EKS cluster and managed node group.

Added the required outputs to `outputs.tf`:

```hcl
output "cluster_name" {
  value = module.eks.cluster_name
}

output "cluster_endpoint" {
  value = module.eks.cluster_endpoint
}

output "cluster_region" {
  value = var.region
}
```

Updated the kubeconfig for the EKS cluster:

```text
arn:aws:eks:us-east-2:874841217451:cluster/terraweek-eks
```

Initially, `kubectl` could reach the EKS API server but returned an authentication error:

```text
the server has asked for the client to provide credentials
```

Checked the AWS identity and confirmed the AWS CLI was authenticated correctly.

The problem was that the IAM user was not present as an EKS access entry. Added the following configuration to the EKS module:

```hcl
enable_cluster_creator_admin_permissions = true
```

Applied the change:

```text
Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```

After that, `kubectl` successfully connected to the cluster.

Verified the nodes:

```text
NAME                                       STATUS   ROLES    AGE   VERSION
ip-10-0-11-20.us-east-2.compute.internal   Ready    <none>   ...   v1.31.13-eks-ecaa3a6
ip-10-0-12-71.us-east-2.compute.internal   Ready    <none>   ...   v1.31.13-eks-ecaa3a6
```

Both requested worker nodes were in the `Ready` state.

Verified the Kubernetes system pods:

```text
NAMESPACE     NAME                      READY   STATUS    RESTARTS   AGE
kube-system   aws-node-hxkn2            2/2     Running   0          ...
kube-system   aws-node-r9jfk            2/2     Running   0          ...
kube-system   coredns-bcdd98cfd-82ttr   1/1     Running   0          ...
kube-system   coredns-bcdd98cfd-sx8ts   1/1     Running   0          ...
kube-system   kube-proxy-tfxmc          1/1     Running   0          ...
kube-system   kube-proxy-zkstr          1/1     Running   0          ...
```

**Verify: Do you see 2 nodes in `Ready` state? Can you see the kube-system pods running?**

Yes. Both worker nodes were in the `Ready` state, and all the displayed `kube-system` pods were running successfully.

---

### Task 5: Deploy a Workload on the Cluster

Created:

```text
k8s/nginx-deployment.yaml
```

The manifest contained an Nginx Deployment with 3 replicas and a LoadBalancer Service.

The deployment was first validated using a client-side dry run:

```text
deployment.apps/nginx-terraweek created (dry run)
service/nginx-service created (dry run)
```

Applied the manifest successfully:

```text
deployment.apps/nginx-terraweek created
service/nginx-service created
```

The LoadBalancer Service received an external AWS LoadBalancer hostname:

```text
a895f8706e3ff457a9eadf969dfe976a-104403539.us-east-2.elb.amazonaws.com
```

Accessed the LoadBalancer URL successfully and verified that the standard **Nginx welcome page** was displayed.

This confirmed the complete application path:

```text
Internet
    ↓
AWS Load Balancer
    ↓
nginx-service
    ↓
Nginx Pods
    ↓
EKS Worker Nodes
```

**Verify: Can you access the Nginx welcome page through the LoadBalancer URL?**

Yes. The Nginx welcome page was successfully accessed through the AWS LoadBalancer URL.

---

### Task 6: Destroy Everything

Deleted the Kubernetes resources first so that the AWS LoadBalancer could be removed:

```text
deployment.apps "nginx-terraweek" deleted from default namespace
service "nginx-service" deleted from default namespace
```

Verified that the Nginx service was gone. Only the default Kubernetes service remained:

```text
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   172.20.0.1   <none>        443/TCP   ...
```

Destroyed all Terraform-managed infrastructure with:

```text
terraform destroy
```

Terraform completed the cleanup successfully:

```text
Destroy complete! Resources: 57 destroyed.
```

The final resource count was 57 because 55 resources were created during the successful EKS deployment and 2 additional resources were later added for EKS cluster creator admin permissions.

The EKS cluster, managed node group, VPC, NAT Gateway, subnets, networking resources, IAM resources, and other Terraform-managed infrastructure were destroyed.

**Verify: Is your AWS account completely clean? No leftover resources?**

The Terraform-managed infrastructure was completely destroyed, with all 57 Terraform resources removed. The Kubernetes LoadBalancer was also deleted before running `terraform destroy`, completing the cleanup required for the exercise.

---

## Documentation

### Complete file structure and key config files

```text
terraform-eks/
├── providers.tf
├── vpc.tf
├── eks.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars
├── .terraform.lock.hcl
├── .terraform/
└── k8s/
    └── nginx-deployment.yaml
```

### `providers.tf`

Configured the AWS provider with version `~> 5.0` and the Kubernetes provider with version `~> 2.0`.

### `variables.tf`

Defined variables for:

```text
region
cluster_name
cluster_version
node_instance_type
node_desired_count
vpc_cidr
```

The final node instance type was changed to:

```text
t3.small
```

because `t3.medium` was not eligible for the account's Free Tier.

### `vpc.tf`

Used the `terraform-aws-modules/vpc/aws` Registry module to create the VPC, two public subnets, two private subnets, NAT Gateway, DNS hostnames, and EKS subnet tags.

### `eks.tf`

Used the `terraform-aws-modules/eks/aws` Registry module to create the EKS cluster and managed node group.

The configuration also included:

```hcl
enable_cluster_creator_admin_permissions = true
```

This was added after the initial deployment so the cluster creator could authenticate and administer the Kubernetes cluster through `kubectl`.

### `outputs.tf`

Configured outputs for:

```text
cluster_name
cluster_endpoint
cluster_region
```

### `terraform.tfvars`

Configured the AWS region:

```hcl
region = "us-east-2"
```

### `k8s/nginx-deployment.yaml`

Created:

- An Nginx Deployment
- 3 Nginx replicas
- A LoadBalancer Service

The AWS LoadBalancer successfully exposed the Nginx welcome page.

### How many resources Terraform created in total?

The successful initial EKS deployment created:

```text
55 resources
```

After adding EKS cluster creator admin permissions, Terraform created:

```text
2 additional resources
```

Therefore, the total number of Terraform-managed resources created during the exercise was:

```text
57 resources
```

The final cleanup also destroyed all:

```text
57 resources
```

### Reflection: EKS with Terraform vs manually setting up a cluster with kind

The biggest difference between this EKS exercise and the manual `kind` setup from Day 50 is the amount of infrastructure involved.

With `kind`, the Kubernetes cluster runs locally using Docker containers. It is lightweight and very useful for learning Kubernetes concepts, testing manifests, and experimenting without creating AWS infrastructure.

With EKS and Terraform, the cluster is running on AWS and involves much more infrastructure. Terraform created the VPC, subnets, NAT Gateway, IAM roles, security groups, EKS control plane, managed node group, and supporting resources.

The manual `kind` approach is simpler and faster for local Kubernetes learning, while Terraform + EKS is much closer to how cloud infrastructure can be provisioned in real-world DevOps environments.

The major advantage of Terraform was repeatability. Instead of manually creating all the AWS infrastructure again, the same configuration could provision the environment from scratch and then destroy it cleanly.

