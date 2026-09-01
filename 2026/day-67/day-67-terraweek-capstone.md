# Day 67 -- TerraWeek Capstone: Multi-Environment Infrastructure with Workspaces and Modules

## Task

Completed a multi-environment AWS infrastructure project using custom Terraform modules and Terraform workspaces.

The project used one Terraform codebase for three environments:

- `dev`
- `staging`
- `prod`

The infrastructure was built using reusable VPC, security group, and EC2 modules. Each environment had its own Terraform workspace, state, VPC, subnet, security group, and EC2 instance.

The infrastructure was successfully deployed, verified, and completely destroyed after testing.

---

## All tf files

[terrform files](https://github.com/Mujakkir-Pathan/terraform-terraweak-90day/tree/main/day-7)

---

## All sceenshots

[screenshots of task](screenshots/)

---

## Challenge Tasks

### Task 1: Learn Terraform Workspaces

Created the Terraform project directory and initialized Terraform.

```bash
mkdir terraweek-capstone && cd terraweek-capstone
terraform init
```

Terraform was initialized successfully.

The initial workspace was:

```text
default
```

Created three environment workspaces:

```bash
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod
```

Verified the workspaces:

```bash
terraform workspace list
```

The workspaces created were:

```text
default
dev
staging
prod
```

Also practiced switching between workspaces:

```bash
terraform workspace select dev
terraform workspace select staging
terraform workspace select prod
```

#### Answers

**1. What does `terraform.workspace` return inside a config?**

`terraform.workspace` returns the name of the currently selected Terraform workspace.

For example, when the `dev` workspace is selected:

```hcl
terraform.workspace
```

returns:

```text
dev
```

This was used to automatically identify the current environment.

**2. Where does each workspace store its state file?**

With the local backend, Terraform stores workspace state separately under:

```text
terraform.tfstate.d/
├── dev/
├── staging/
└── prod/
```

Each workspace therefore maintains its own state.

**3. How is this different from using separate directories per environment?**

Workspaces allow multiple environments to use the same Terraform configuration while maintaining separate state.

Separate directories provide physically separate Terraform configurations and are often easier to manage when environments require significantly different infrastructure.

For this capstone, workspaces were useful because the three environments shared the same infrastructure structure and mainly differed in configuration values.

---

### Task 2: Set Up the Project Structure

Created the Terraform project using separate files for different responsibilities and reusable modules.

```text
terraweek-capstone/
  main.tf
  variables.tf
  outputs.tf
  providers.tf
  locals.tf
  dev.tfvars
  staging.tfvars
  prod.tfvars
  .gitignore
  modules/
    vpc/
      main.tf
      variables.tf
      outputs.tf
    security-group/
      main.tf
      variables.tf
      outputs.tf
    ec2-instance/
      main.tf
      variables.tf
      outputs.tf
```

Created `.gitignore` to ignore Terraform state, generated files, and environment variable files:

```gitignore
.terraform/
*.tfstate
*.tfstate.backup
*.tfvars
.terraform.lock.hcl
```

#### Why is this file structure considered best practice?

The configuration was separated according to responsibility instead of keeping everything in one large Terraform file.

- `providers.tf` contained provider configuration.
- `variables.tf` contained input variables.
- `main.tf` called the child modules.
- `outputs.tf` exposed important infrastructure values.
- `locals.tf` contained workspace-aware local values.
- `.tfvars` files contained environment-specific values.
- `modules/` contained reusable infrastructure components.

This made the Terraform project easier to read, maintain, reuse, and troubleshoot.

---

### Task 3: Build the Custom Modules

Created three focused custom Terraform modules.

#### Module 1: `modules/vpc/`

The VPC module accepted:

- `cidr`
- `public_subnet_cidr`
- `environment`
- `project_name`

It created:

- VPC
- Public subnet
- Internet gateway
- Route table
- Route table association

The module returned:

- `vpc_id`
- `subnet_id`

Resources were tagged with the project and environment.

#### Module 2: `modules/security-group/`

The security group module accepted:

- `vpc_id`
- `ingress_ports`
- `environment`
- `project_name`

It created:

- Security group
- Dynamic ingress rules
- Allow-all egress

The module returned:

- `sg_id`

#### Module 3: `modules/ec2-instance/`

The EC2 module accepted:

- `ami_id`
- `instance_type`
- `subnet_id`
- `security_group_ids`
- `environment`
- `project_name`

It created:

- EC2 instance

The module returned:

- `instance_id`
- `public_ip`

Validated the Terraform configuration successfully:

```bash
terraform validate
```

Result:

```text
Success! The configuration is valid.
```

---

### Task 4: Wire It All Together with Workspace-Aware Config

Used `terraform.workspace` to identify the active environment.

The local configuration used:

```hcl
locals {
  environment = terraform.workspace
  name_prefix = "${var.project_name}-${local.environment}"

  common_tags = {
    Project     = var.project_name
    Environment = local.environment
    ManagedBy   = "Terraform"
    Workspace   = terraform.workspace
  }
}
```

The root variables included:

- `project_name`
- `vpc_cidr`
- `subnet_cidr`
- `instance_type`
- `ingress_ports`

The root `main.tf` called the three custom modules:

```hcl
module "vpc" {
  source = "./modules/vpc"

  cidr               = var.vpc_cidr
  public_subnet_cidr = var.subnet_cidr
  environment        = local.environment
  project_name       = var.project_name
}

module "security_group" {
  source = "./modules/security-group"

  vpc_id        = module.vpc.vpc_id
  ingress_ports = var.ingress_ports
  environment   = local.environment
  project_name  = var.project_name
}

module "ec2_instance" {
  source = "./modules/ec2-instance"

  ami_id             = data.aws_ami.amazon_linux.id
  instance_type      = var.instance_type
  subnet_id          = module.vpc.subnet_id
  security_group_ids = [module.security_group.sg_id]
  environment        = local.environment
  project_name       = var.project_name
}
```

#### Environment-specific values

**`dev.tfvars`:**

```hcl
vpc_cidr      = "10.0.0.0/16"
subnet_cidr   = "10.0.1.0/24"
instance_type = "t3.micro"
ingress_ports = [22, 80]
```

**`staging.tfvars`:**

```hcl
vpc_cidr      = "10.1.0.0/16"
subnet_cidr   = "10.1.1.0/24"
instance_type = "t3.small"
ingress_ports = [22, 80, 443]
```

**`prod.tfvars`:**

```hcl
vpc_cidr      = "10.2.0.0/16"
subnet_cidr   = "10.2.1.0/24"
instance_type = "t3.micro"
ingress_ports = [80, 443]
```

The production instance used `t3.micro` instead of the original `t3.small` specified in the task because the available instance options were considered to avoid unnecessary AWS costs.

Dev allowed SSH access on port `22`, while production did not.

Different CIDR ranges were used to prevent overlap between the environments.

---

### Task 5: Deploy All Three Environments

#### Dev

Selected the `dev` workspace and created a plan:

```bash
terraform workspace select dev
terraform plan -var-file="dev.tfvars"
```

Terraform planned:

```text
Plan: 7 to add, 0 to change, 0 to destroy.
```

Applied the configuration:

```bash
terraform apply -var-file="dev.tfvars"
```

Result:

```text
Apply complete! Resources: 7 added, 0 changed, 0 destroyed.
```

Dev outputs:

```text
instance_id       = "i-04bbc5fbf1b53a2a2"
public_ip         = "3.23.100.71"
security_group_id = "sg-06e34a8103d8653de"
subnet_id         = "subnet-06c5bd7375238bf13"
vpc_id            = "vpc-007e640492b997871"
```

Verified the workspace and outputs:

```text
dev
```

The outputs matched the deployed Dev infrastructure.

#### Staging

Selected the `staging` workspace:

```bash
terraform workspace select staging
```

Created a plan:

```bash
terraform plan -var-file="staging.tfvars"
```

Terraform planned:

```text
Plan: 7 to add, 0 to change, 0 to destroy.
```

Applied the configuration:

```bash
terraform apply -var-file="staging.tfvars"
```

Result:

```text
Apply complete! Resources: 7 added, 0 changed, 0 destroyed.
```

Staging outputs:

```text
instance_id       = "i-0e4b22d49957b8b27"
public_ip         = "18.188.227.143"
security_group_id = "sg-0f50a870b3613c1af"
subnet_id         = "subnet-03a5b1b388f2eb1db"
vpc_id            = "vpc-0a264d7d54dc41149"
```

Verified the workspace:

```text
staging
```

The outputs matched the deployed Staging infrastructure.

#### Prod

Selected the `prod` workspace:

```bash
terraform workspace select prod
```

Created a plan:

```bash
terraform plan -var-file="prod.tfvars"
```

Terraform planned:

```text
Plan: 7 to add, 0 to change, 0 to destroy.
```

Applied the configuration:

```bash
terraform apply -var-file="prod.tfvars"
```

Result:

```text
Apply complete! Resources: 7 added, 0 changed, 0 destroyed.
```

Prod outputs:

```text
instance_id       = "i-08a3c478a410d2b2a"
public_ip         = "18.119.103.137"
security_group_id = "sg-0c9c3489ed56fef5e"
subnet_id         = "subnet-0161f6b1dbae12c17"
vpc_id            = "vpc-02c59acc0e14b3fc4"
```

### Environment comparison

| Environment | VPC CIDR | Subnet CIDR | Instance Type | Ingress |
|---|---|---|---|---|
| Dev | `10.0.0.0/16` | `10.0.1.0/24` | `t3.micro` | 22, 80 |
| Staging | `10.1.0.0/16` | `10.1.1.0/24` | `t3.small` | 22, 80, 443 |
| Prod | `10.2.0.0/16` | `10.2.1.0/24` | `t3.micro` | 80, 443 |

Each environment received a different Name tag:

```text
terraweek-dev-server
terraweek-staging-server
terraweek-prod-server
```

**Verify: Are all three environments completely isolated from each other?**

Yes. Each environment used a separate Terraform workspace and separate state. Each environment also had its own VPC, subnet, security group, and EC2 instance.

The VPC CIDR ranges were different, so the environments did not have overlapping network ranges.

For this capstone, the environments were isolated as required.

---

### Task 6: Document Best Practices

#### 1. File structure

Terraform configuration was separated into providers, variables, outputs, main configuration, locals, environment values, and reusable modules.

This made the project easier to understand and maintain.

#### 2. State management

Terraform state keeps track of the relationship between Terraform configuration and real infrastructure.

For production environments, remote state should be used with:

- Remote storage
- State locking
- State versioning
- Encryption
- Restricted access

This capstone used local workspace state to demonstrate Terraform workspaces.

#### 3. Variables

Variables were used instead of hardcoding environment-specific values.

Separate `.tfvars` files were used for:

- Dev
- Staging
- Prod

Production configurations should also use variable validation blocks where appropriate.

#### 4. Modules

Infrastructure was divided into three focused modules:

- VPC
- Security Group
- EC2 Instance

Each module defined its own inputs and outputs.

Terraform Registry modules should be pinned to known versions in production to avoid unexpected changes.

#### 5. Workspaces

Terraform workspaces were used to maintain separate state for:

- `dev`
- `staging`
- `prod`

The configuration referenced:

```hcl
terraform.workspace
```

to identify the active environment.

#### 6. Security

Terraform state and environment files can contain sensitive information.

Best practices include:

- Ignore state files in Git.
- Ignore `.tfvars` files when they contain secrets.
- Encrypt remote state at rest.
- Restrict access to state storage.
- Never hardcode credentials.
- Follow least-privilege IAM.

The project used `.gitignore` to prevent Terraform state and `.tfvars` files from being committed.

#### 7. Commands

A good Terraform workflow is:

```text
terraform fmt
      ↓
terraform validate
      ↓
terraform plan
      ↓
terraform apply
```

`terraform plan` was run before each environment was applied.

#### 8. Tagging

Resources were tagged using project and environment information.

Common tags included:

```text
Project
Environment
ManagedBy
Workspace
```

This made AWS resources easier to identify and manage.

#### 9. Naming

A consistent naming pattern was used:

```text
<project>-<environment>-<resource>
```

Examples:

```text
terraweek-dev-server
terraweek-staging-server
terraweek-prod-server
```

#### 10. Cleanup

Unused infrastructure should be destroyed to avoid unnecessary cloud costs.

All three environments were destroyed after verification.

---

### Task 7: Destroy All Environments

The environments were destroyed in reverse order.

#### Prod

```bash
terraform workspace select prod
terraform destroy -var-file="prod.tfvars"
```

Result:

```text
Destroy complete! Resources: 7 destroyed.
```

#### Staging

```bash
terraform workspace select staging
terraform destroy -var-file="staging.tfvars"
```

Result:

```text
Destroy complete! Resources: 7 destroyed.
```

#### Dev

```bash
terraform workspace select dev
terraform destroy -var-file="dev.tfvars"
```

Result:

```text
Destroy complete! Resources: 7 destroyed.
```

A total of:

```text
7 resources × 3 environments = 21 resources
```

were successfully destroyed.

Verified each workspace state using:

```bash
terraform state list
```

All three workspace states were empty.

The workspaces were then deleted:

```bash
terraform workspace select default
terraform workspace delete dev
terraform workspace delete staging
terraform workspace delete prod
```

Final workspace verification:

```text
* default
```

Only the `default` workspace remained.

**Verify: Is your AWS account completely clean?**

Yes, the Terraform-managed resources created for this capstone were completely destroyed. The Dev, Staging, and Prod Terraform states were empty, and their workspaces were deleted.

The final Terraform workspace was:

```text
default
```

---

## Terraform Best Practices Summary

| Practice | Implementation |
|---|---|
| File structure | Separate files by responsibility |
| State | Separate state per workspace; remote backend recommended for production |
| Variables | Environment-specific `.tfvars` files |
| Modules | Focused VPC, security group, and EC2 modules |
| Workspaces | `dev`, `staging`, and `prod` |
| Security | State and `.tfvars` excluded through `.gitignore` |
| Validation | `terraform fmt` and `terraform validate` |
| Planning | `terraform plan` before `apply` |
| Tagging | Project, environment, managed-by, and workspace |
| Naming | `<project>-<environment>-<resource>` |
| Cleanup | Destroyed all test infrastructure |

---

## TerraWeek Concepts Learned

| Day | Concepts |
| --- | --- |
| 61 | IaC, HCL, init/plan/apply/destroy, state basics |
| 62 | Providers, resources, dependencies, lifecycle |
| 63 | Variables, outputs, data sources, locals, functions |
| 64 | Remote backend, locking, import, drift |
| 65 | Custom modules, registry modules, versioning |
| 66 | EKS with modules, real-world provisioning |
| 67 | Workspaces, multi-env, capstone project |

---

## Final Result

The TerraWeek capstone successfully brought together the Terraform concepts learned throughout the week.

A single Terraform codebase was used to provision three separate environments with reusable modules and workspace-specific configuration.

All three environments were successfully:

```text
Planned
  ↓
Applied
  ↓
Verified
  ↓
Destroyed
  ↓
Workspaces deleted
```

The project demonstrated how Terraform can be used to build repeatable, modular, and environment-aware infrastructure.
