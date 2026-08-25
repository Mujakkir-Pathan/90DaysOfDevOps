# Day 62 -- Providers, Resources and Dependencies

## Task

Completed the task by exploring the AWS provider, building a complete VPC networking stack, understanding implicit and explicit dependencies, adding a security group and EC2 instance, using `depends_on`, visualizing the dependency graph, applying lifecycle rules, and destroying the infrastructure in dependency order.

---

## screenshots

[screenshots of task](screenshots/)

---

## tf file

[main.tf with explanation](https://github.com/Mujakkir-Pathan/terraform-terraweak-90day/blob/main/day-2/main.tf)

---

## Challenge Tasks

### Task 1: Explore the AWS Provider

1. Created a new project directory named `terraform-aws-infra`.
2. Created a `providers.tf` file.
3. Defined the `terraform` block with `required_providers` and pinned the AWS provider to version `~> 5.0`.
4. Defined the `provider "aws"` block with the AWS region.
5. Ran `terraform init` and checked the installed AWS provider version.
6. Read the `.terraform.lock.hcl` file and reviewed the provider version and checksums.

**Document:** What does `~> 5.0` mean? How is it different from `>= 5.0` and `= 5.0.0`?

- `~> 5.0` allowed Terraform to use version `5.x` releases while preventing an upgrade to version `6.0` or higher.
- `>= 5.0` allowed version `5.0` and any newer compatible version, including future major versions.
- `= 5.0.0` allowed only the exact `5.0.0` version.
- The `~>` constraint was useful when allowing compatible updates while preventing unexpected major-version changes.

---

### Task 2: Build a VPC from Scratch

Created a `main.tf` file and defined the following resources:

1. Created an `aws_vpc` resource with CIDR block `10.0.0.0/16` and tagged it `TerraWeek-VPC`.
2. Created an `aws_subnet` resource with CIDR block `10.0.1.0/24`, referenced the VPC ID, enabled public IP assignment, and tagged it `TerraWeek-Public-Subnet`.
3. Created an `aws_internet_gateway` resource and attached it to the VPC.
4. Created an `aws_route_table` resource in the VPC and added a default route `0.0.0.0/0` through the internet gateway.
5. Created an `aws_route_table_association` resource to associate the route table with the subnet.
6. Ran `terraform plan` and reviewed the resources Terraform planned to create.
7. Applied the configuration and verified the networking resources in AWS.

**Verify:** Apply and check the AWS VPC console. Can you see all five resources connected?

Yes. The VPC, public subnet, internet gateway, route table, and route table association were created and connected through their resource references.

---

### Task 3: Understand Implicit Dependencies

1. Identified the subnet's reference to `aws_vpc.main.id` as an implicit dependency.
2. Identified the internet gateway's reference to the VPC ID as an implicit dependency.
3. Identified the route table association's references to both the route table and subnet as implicit dependencies.

**How does Terraform know to create the VPC before the subnet?**

Terraform analyzes references between resources. Because the subnet uses `aws_vpc.main.id`, Terraform knows that the VPC must exist before the subnet can be created.

**What would happen if you tried to create the subnet before the VPC existed?**

The subnet could not be created because it requires a valid VPC ID. Terraform prevents this situation by creating resources according to their dependency graph.

**Find all implicit dependencies in your config and list them**

- `aws_subnet.public` → `aws_vpc.main`
- `aws_internet_gateway.main` → `aws_vpc.main`
- `aws_route_table.main` → `aws_vpc.main`
- `aws_route_table_association.public` → `aws_route_table.main`
- `aws_route_table_association.public` → `aws_subnet.public`
- `aws_security_group.main` → `aws_vpc.main`
- `aws_instance.main` → `aws_subnet.public`
- `aws_instance.main` → `aws_security_group.main`

---

### Task 4: Add a Security Group and EC2 Instance

Added the following resources to the configuration:

1. Created an `aws_security_group` inside the VPC.
2. Added an ingress rule allowing SSH traffic on port `22` from `0.0.0.0/0`.
3. Added an ingress rule allowing HTTP traffic on port `80` from `0.0.0.0/0`.
4. Added an egress rule allowing all outbound traffic.
5. Tagged the security group as `TerraWeek-SG`.
6. Created an `aws_instance` in the public subnet.
7. Used an Amazon Linux 2 AMI for the configured region.
8. Set the instance type to `t2.micro`.
9. Associated the security group with the EC2 instance.
10. Enabled `associate_public_ip_address = true`.
11. Tagged the instance as `TerraWeek-Server`.
12. Applied the configuration and verified the EC2 instance and networking resources.

---

### Task 5: Explicit Dependencies with depends_on

1. Added a second `aws_s3_bucket` resource for application logs.
2. Added `depends_on = [aws_instance.main]` to the S3 bucket.
3. Used `depends_on` to create an explicit dependency even though the S3 bucket had no direct reference to the EC2 instance.
4. Ran `terraform plan` and reviewed the dependency order.
5. Generated the Terraform dependency graph using `terraform graph`.

```text
terraform graph | dot -Tpng > graph.png
```

6. Used the Terraform graph output to understand how Terraform connected the resources and determined their creation order.

**Document:** When would you use `depends_on` in real projects? Give two examples.

`depends_on` was useful when Terraform could not automatically detect a dependency but one resource still needed another resource to exist first.

**Example 1:** An application resource could depend on an IAM role or policy being fully created before the application was deployed.

**Example 2:** A database or application resource could explicitly depend on another infrastructure component when the dependency existed operationally but was not represented by a direct Terraform reference.

---

### Task 6: Lifecycle Rules and Destroy

1. Added a `lifecycle` block to the EC2 instance.

```hcl
lifecycle {
  create_before_destroy = true
}
```

2. Changed the AMI ID and ran `terraform plan`.
3. Observed the replacement behavior caused by `create_before_destroy`.
4. Ran `terraform destroy`.
5. Observed that Terraform destroyed resources in reverse dependency order.
6. Verified that the infrastructure was cleaned up from AWS.

**Document:** What are the three lifecycle arguments (`create_before_destroy`, `prevent_destroy`, `ignore_changes`) and when would you use each?

| Lifecycle Argument | Meaning | When to Use |
|---|---|---|
| `create_before_destroy` | Creates the replacement resource before destroying the existing resource. | Used when downtime should be minimized during resource replacement. |
| `prevent_destroy` | Prevents Terraform from destroying the resource through Terraform. | Used for critical resources such as important databases or production infrastructure. |
| `ignore_changes` | Tells Terraform to ignore changes to selected resource attributes. | Used when an attribute can be changed outside Terraform and those changes should not trigger updates. |

---

## Dependency Graph

The dependency graph was based on the resource references and the explicit `depends_on` relationship.

```text
aws_vpc.main
├── aws_subnet.public
│   ├── aws_route_table_association.public
│   └── aws_instance.main
│       └── aws_s3_bucket.logs
├── aws_internet_gateway.main
│   └── aws_route_table.public
│       └── aws_route_table_association.public
└── aws_security_group.main
    └── aws_instance.main
        └── aws_s3_bucket.logs
```

The graph showed that Terraform did not simply create resources from top to bottom. It analyzed the dependencies and created resources in an order that satisfied those dependencies.

---

## Explanation of Implicit vs Explicit Dependencies

An **implicit dependency** was created automatically when one Terraform resource referenced another resource. For example, `aws_subnet.public` referenced `aws_vpc.main.id`, so Terraform automatically knew that the VPC had to be created before the subnet.

An **explicit dependency** was created manually with `depends_on`. It was useful when Terraform could not see a direct reference but there was still a real operational dependency. In this task, the S3 bucket was explicitly configured to depend on the EC2 instance.

The main difference was that **implicit dependencies came from resource references, while explicit dependencies were manually declared with `depends_on`**.
