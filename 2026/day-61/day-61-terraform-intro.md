# Day 61 — Introduction to Terraform and Your First AWS Infrastructure

## Task

Completed the introduction to Infrastructure as Code by installing Terraform, configuring and verifying AWS CLI access, creating an S3 bucket and EC2 instance through Terraform, inspecting Terraform state, modifying an EC2 tag, and destroying the infrastructure through Terraform.

---

## All screenshots
[screenshots](screenshots/)

---

## tf files

[terraform_files](https://github.com/Mujakkir-Pathan/terraform-terraweak-90day/tree/main)

---

# Challenge Tasks

## Task 1: Understand Infrastructure as Code

Before working with Terraform, I reviewed the basic concepts of Infrastructure as Code.

### What is Infrastructure as Code (IaC)? Why does it matter in DevOps?

Infrastructure as Code means creating and managing infrastructure such as servers, storage, networks, and databases using configuration files instead of manually creating everything through a cloud console.

IaC is important in DevOps because infrastructure becomes repeatable, consistent, version-controlled, and easier to automate. The same configuration can be used to recreate infrastructure without manually repeating every step.

### What problems does IaC solve compared to manually creating resources in the AWS console?

Manual infrastructure creation can lead to human errors, inconsistent configurations, slow provisioning, and difficulty reproducing the same environment.

IaC solves these problems by keeping infrastructure configuration as code. The code can be reviewed, stored in Git, reused, and executed repeatedly.

### How is Terraform different from AWS CloudFormation, Ansible, and Pulumi?

| Tool | Main Purpose / Difference |
|---|---|
| Terraform | Infrastructure provisioning and management across multiple cloud providers using HCL |
| AWS CloudFormation | AWS-native Infrastructure as Code service mainly focused on AWS resources |
| Ansible | Primarily used for configuration management, server automation, and application setup |
| Pulumi | Infrastructure as Code using general-purpose programming languages such as Python, TypeScript, Go, and C# |

### What does it mean that Terraform is "declarative" and "cloud-agnostic"?

**Declarative** means I describe the infrastructure I want, and Terraform determines what actions are required to reach that desired state.

**Cloud-agnostic** means Terraform is not limited to one cloud provider. It can manage infrastructure across AWS, Azure, Google Cloud, Kubernetes, and many other providers.

---

## Task 2: Install Terraform and Configure AWS

### Install Terraform

Terraform was installed on the Ubuntu Linux amd64 machine using the HashiCorp APT repository.

The HashiCorp repository configuration initially contained an incorrectly formatted URL, so it was corrected before updating APT.

Terraform was then installed successfully.

### Verify

Ran:

```bash
terraform -version
```

Result:

```text
Terraform v1.15.9
on linux_amd64
```

Terraform was installed and working correctly.

### Install and configure the AWS CLI

AWS CLI was not initially installed, so it was installed using APT.

Verified with:

```bash
aws --version
```

Result:

```text
aws-cli/2.31.35 Python/3.14.4 Linux/7.0.0-1006-aws source/x86_64.ubuntu.26
```

AWS CLI was already authenticated on the machine, so `aws sts get-caller-identity` was used to verify access.

### Verify AWS access

Ran:

```bash
aws sts get-caller-identity
```

The command returned the AWS account and IAM identity:

```text
Account: 874841217451
Arn: arn:aws:iam::874841217451:user/stephane
```

AWS access was successfully verified.

The AWS CLI region was verified as:

```text
us-east-2
```

---

## Task 3: Your First Terraform Config -- Create an S3 Bucket

Created the Terraform project directory:

```bash
mkdir terraform-basics && cd terraform-basics
```

Created `main.tf` with:

- A `terraform` block specifying the AWS provider.
- An AWS provider configured for `us-east-2`.
- An `aws_s3_bucket` resource with a globally unique bucket name.

The S3 bucket created during the task was:

```text
terraweek-mujakkir-2026
```

### Run the Terraform lifecycle

Initialized Terraform with:

```bash
terraform init
```

Terraform downloaded the required AWS provider plugin and created the `.terraform/` working directory.

Created a plan using:

```bash
terraform plan
```

Applied the configuration using:

```bash
terraform apply
```

Terraform successfully created the S3 bucket:

```text
aws_s3_bucket.my_s3_bucket
```

The bucket was then verified successfully in the AWS S3 Console.

### Document: What did `terraform init` download?

`terraform init` downloaded the AWS provider required by the Terraform configuration.

The provider allows Terraform to communicate with AWS and create, read, update, and destroy AWS resources.

### What does the `.terraform/` directory contain?

The `.terraform/` directory contains Terraform's local working files, including the downloaded provider plugin required by the configuration.

The project also contained:

```text
.terraform/
.terraform.lock.hcl
main.tf
terraform.tfstate
terraform.tfstate.backup
```

The `.terraform.lock.hcl` file records provider version and checksum information so provider dependencies remain consistent.

---

## Task 4: Add an EC2 Instance

Added an `aws_instance` resource to the same `main.tf`.

The original AMI in the task was for `ap-south-1`, so it was not used directly because AMI IDs are region-specific.

The Terraform environment was using `us-east-2`, so a valid Amazon Linux 2 AMI for the environment was used:

```text
ami-07f0e6f6330bf9233
```

The account did not allow `t2.micro` as a Free Tier eligible instance type, so `t3.micro` was used instead.

The EC2 resource was configured with:

```hcl
resource "aws_instance" "my_instance" {
  ami           = "ami-07f0e6f6330bf9233"
  instance_type = "t3.micro"

  tags = {
    Name = "TerraWeek-Day1"
  }
}
```

### Run Terraform plan

Ran:

```bash
terraform plan
```

Terraform reported:

```text
Plan: 1 to add, 0 to change, 0 to destroy.
### Terraform Apply Screenshot

The `terraform apply` command successfully created both the S3 bucket and EC2 instance.

![Terraform apply creating S3 bucket and EC2 instance](screenshots/terraform-apply.png)

### AWS Console Screenshot

The created S3 bucket and EC2 instance were verified successfully in the AWS Console.

![AWS resources created by Terraform](screenshots/aws-resources.png)
```

This showed that the S3 bucket was already being managed by Terraform and only the EC2 instance needed to be created.

### Run Terraform apply

Ran:

```bash
terraform apply
```

The EC2 instance was successfully created.

Instance ID:

```text
i-0c03955952ea3449a
```

Instance type:

```text
t3.micro
```

Name tag:

```text
TerraWeek-Day1
```

Region:

```text
us-east-2
```

The EC2 instance was verified successfully in the AWS EC2 Console.

### Document: How does Terraform know the S3 bucket already exists and only the EC2 instance needs to be created?

Terraform knows about the S3 bucket because it is recorded in `terraform.tfstate`.

When `terraform plan` was executed, Terraform compared the desired configuration in `main.tf` with the resources recorded in the state file.

The S3 bucket was already tracked, so Terraform planned no change for it and planned only one new resource: the EC2 instance.

---

## Task 5: Understand the State File

Terraform created the state file:

```text
terraform.tfstate
```

The state file contains information about the real infrastructure Terraform manages.

### `terraform show`

Ran:

```bash
terraform show
```

This displayed a human-readable representation of the current Terraform state, including both:

```text
aws_s3_bucket.my_s3_bucket
aws_instance.my_instance
```

### `terraform state list`

Ran:

```bash
terraform state list
```

Result:

```text
aws_instance.my_instance
aws_s3_bucket.my_s3_bucket
```

This confirmed that Terraform was managing two resources.

### `terraform state show aws_s3_bucket.my_s3_bucket`

Ran:

```bash
terraform state show aws_s3_bucket.my_s3_bucket
```

The state showed information including:

```text
bucket        = "terraweek-mujakkir-2026"
id            = "terraweek-mujakkir-2026"
arn           = "arn:aws:s3:::terraweek-mujakkir-2026"
bucket_region = "us-east-1"
```

The state also contained information about encryption, versioning, tags, domain names, and other AWS attributes.

### `terraform state show aws_instance.my_instance`

Ran:

```bash
terraform state show aws_instance.my_instance
```

Important information recorded for the EC2 instance included:

```text
id             = "i-0c03955952ea3449a"
ami            = "ami-07f0e6f6330bf9233"
instance_type  = "t3.micro"
instance_state = "running"
region         = "us-east-2"
availability_zone = "us-east-2b"
private_ip     = "172.31.22.99"
public_ip      = "3.129.61.192"
tags           = {
  "Name" = "TerraWeek-Day1"
}
```

### What information does the state file store about each resource?

The state file stores information Terraform needs to track the real infrastructure, including resource IDs, ARNs, regions, configuration attributes, networking information, tags, and other values returned by the provider.

For example, the EC2 state contained the instance ID, AMI, instance type, region, IP addresses, security groups, and tags.

### Why should you never manually edit the state file?

The state file is managed by Terraform. Manually changing it can make Terraform's understanding of the infrastructure inconsistent with the actual AWS resources.

This can result in incorrect plans or unexpected infrastructure changes.

### Why should the state file not be committed to Git?

Terraform state can contain sensitive infrastructure information and resource details. It can also contain sensitive values depending on the resources being managed.

For this reason, state files should not normally be committed to Git. In team environments, Terraform state should be stored securely using an appropriate remote backend with controlled access and locking.

---

## Task 6: Modify, Plan, and Destroy

Changed the EC2 instance Name tag from:

```text
TerraWeek-Day1
```

to:

```text
TerraWeek-Modified
```

Ran:

```bash
terraform plan
```

Terraform reported:

```text
~ update in-place
```

and:

```text
Plan: 0 to add, 1 to change, 0 to destroy.
```

### What do the `~`, `+`, and `-` symbols mean?

| Symbol | Meaning |
|---|---|
| `+` | Resource will be created |
| `-` | Resource will be destroyed |
| `~` | Existing resource will be updated in-place |

### Is this an in-place update or a destroy-and-recreate?

It was an **in-place update**.

Terraform changed:

```text
TerraWeek-Day1
```

to:

```text
TerraWeek-Modified
```

without destroying and recreating the EC2 instance.

The instance ID remained:

```text
i-0c03955952ea3449a
```

### Apply the change

Ran:

```bash
terraform apply
```

The tag change was successfully applied.

### Verify the tag changed in the AWS console

Verified the EC2 instance in the AWS Console.

The Name tag changed successfully to:

```text
TerraWeek-Modified
```

### Finally, destroy everything

Ran:

```bash
terraform destroy
```

Terraform destroyed both Terraform-managed resources:

```text
aws_s3_bucket.my_s3_bucket
aws_instance.my_instance
```

Final result:

```text
Destroy complete! Resources: 2 destroyed.
```

### Verify in the AWS console

Verified that the S3 bucket and EC2 instance were removed from AWS.

---

## Key Learnings

### Infrastructure as Code (IaC)

Infrastructure as Code (IaC) means creating and managing infrastructure using code instead of manually creating resources through a cloud console. It makes infrastructure repeatable, consistent, and easier to manage because the same configuration can be used again. In DevOps, IaC also allows infrastructure changes to be reviewed, version-controlled, and automated. Terraform is one of the popular tools used to implement IaC.


### What Each Terraform Command Does

| Command | What it does |
|---|---|
| `terraform init` | Initializes the Terraform project and downloads the required provider plugins. |
| `terraform plan` | Compares the Terraform configuration with the current state and shows what changes Terraform will make. |
| `terraform apply` | Applies the planned changes and creates or updates the infrastructure. |
| `terraform destroy` | Removes the infrastructure managed by Terraform. |
| `terraform show` | Displays the current Terraform state in a human-readable format. |
| `terraform state list` | Lists all resources currently managed by Terraform in the state file. |

### Terraform State File

Terraform stores information about the infrastructure it manages in `terraform.tfstate`. The state file contains details such as resource IDs, ARNs, regions, IP addresses, tags, and other attributes returned by AWS. It matters because Terraform uses this information to understand which resources already exist and compare the real infrastructure with the desired configuration in `main.tf`. The state file should not be manually edited or committed to Git because it can contain sensitive infrastructure information and manually changing it can cause Terraform's state to become inconsistent.
