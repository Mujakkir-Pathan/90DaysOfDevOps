# Day 63 -- Variables, Outputs, Data Sources and Expressions

## Task

Refactored the Day 62 Terraform configuration to make it dynamic, reusable, and environment-aware by replacing hardcoded values with variables, variable files, outputs, data sources, locals, built-in functions, and conditional expressions.

---

## All tf files

[terrform files](https://github.com/Mujakkir-Pathan/terraform-terraweak-90day/tree/main/day-3)

---

## All sceenshots

[screenshots of task](screenshots/)

---

## Challenge Tasks

### Task 1: Extract Variables

Created a `variables.tf` file with input variables for:

- `region` as a string with a default value
- `vpc_cidr` as a string with default value `"10.0.0.0/16"`
- `subnet_cidr` as a string with default value `"10.0.1.0/24"`
- `instance_type` as a string with default value `"t2.micro"`
- `project_name` as a string without a default value
- `environment` as a string with default value `"dev"`
- `allowed_ports` as a list of numbers with default value `[22, 80, 443]`
- `extra_tags` as a map of strings with default value `{}`

Replaced the hardcoded values in `main.tf` with `var.<name>` references.

Ran `terraform plan`, which prompted for `project_name` because it did not have a default value.

**Document:** What are the five variable types in Terraform?

- `string` stores text values such as `"dev"` or `"t2.micro"`.
- `number` stores numeric values such as `80` or `443`.
- `bool` stores true or false values.
- `list` stores an ordered collection of values.
- `map` stores key-value pairs.

---

### Task 2: Variable Files and Precedence

Created `terraform.tfvars`:

```hcl
project_name  = "terraweek"
environment   = "dev"
instance_type = "t2.micro"
```

Created `prod.tfvars`:

```hcl
project_name  = "terraweek"
environment   = "prod"
instance_type = "t3.small"
vpc_cidr      = "10.1.0.0/16"
subnet_cidr   = "10.1.1.0/24"
```

Used `terraform plan` with the default `terraform.tfvars` file.

Used `terraform plan -var-file="prod.tfvars"` to apply the production variables.

Used `terraform plan -var="instance_type=t2.nano"` to override the instance type from the CLI.

Set `TF_VAR_environment="staging"` and used `terraform plan` to test environment-variable precedence.

**Document:** Write the variable precedence order from lowest to highest priority.

1. Variable defaults
2. Environment variables (`TF_VAR_*`)
3. `terraform.tfvars` and automatically loaded `.auto.tfvars` files
4. Explicit `-var-file` values
5. CLI `-var` values

Higher-priority values override lower-priority values when the same variable is defined multiple times.

---

### Task 3: Add Outputs

Created an `outputs.tf` file with outputs for:

1. `vpc_id` -- the VPC ID
2. `subnet_id` -- the public subnet ID
3. `instance_id` -- the EC2 instance ID
4. `instance_public_ip` -- the public IP of the EC2 instance
5. `instance_public_dns` -- the public DNS name
6. `security_group_id` -- the security group ID

Applied the configuration and verified that the outputs were printed at the end of the apply.

Used `terraform output` to show all outputs.

Used `terraform output instance_public_ip` to show the specific public IP.

Used `terraform output -json` to display the outputs in JSON format.

**Verify:** Does `terraform output instance_public_ip` return the correct IP?

Yes. `terraform output instance_public_ip` returned the public IP address assigned to the EC2 instance.

---

### Task 4: Use Data Sources

Added a `data "aws_ami"` block to dynamically find an Amazon Linux 2 AMI.

Configured the AMI data source to:

- Filter for Amazon Linux 2 images
- Filter for `hvm` virtualization
- Filter for `gp2` root devices
- Use `owners = ["amazon"]`
- Set `most_recent = true`

Replaced the hardcoded AMI ID in the EC2 instance with `data.aws_ami.amazon_linux.id`.

Added a `data "aws_availability_zones"` block to fetch the available Availability Zones in the selected region.

Used `data.aws_availability_zones.available.names[0]` for the subnet Availability Zone.

Applied the configuration and verified that the AMI and Availability Zone were selected dynamically.

**Document:** What is the difference between a `resource` and a `data` source?

A `resource` creates and manages infrastructure through Terraform. A `data` source reads existing information from a provider without creating or managing that object.

---

### Task 5: Use Locals for Dynamic Values

Added a `locals` block:

```hcl
locals {
  name_prefix = "${var.project_name}-${var.environment}"
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}
```

Replaced the Name tags with `local.name_prefix`:

- VPC: `"${local.name_prefix}-vpc"`
- Subnet: `"${local.name_prefix}-subnet"`
- Instance: `"${local.name_prefix}-server"`

Merged common tags with resource-specific tags:

```hcl
tags = merge(local.common_tags, {
  Name = "${local.name_prefix}-server"
})
```

Applied the configuration and checked the tags in the AWS console.

---

### Task 6: Built-in Functions and Conditional Expressions

Practiced Terraform built-in functions in `terraform console`.

Tested string functions including `upper`, `join`, and `format`.

Tested collection functions including `length`, `lookup`, and `toset`.

Tested the networking function `cidrsubnet`.

Added a conditional expression to select the instance type based on the environment.

Applied the configuration with `environment = "prod"` and verified that the instance type changed to `t3.small`.

**Document:** Pick five functions you find most useful and explain what each does.

1. `upper()` converts a string to uppercase. It is useful for standardizing text values.
2. `join()` combines multiple strings using a specified separator. It is useful for creating names and paths dynamically.
3. `length()` returns the number of elements in a collection or characters in a string. It is useful when working with dynamic collections.
4. `lookup()` retrieves a value from a map using a key. It is useful for selecting environment-specific values.
5. `cidrsubnet()` calculates a subnet CIDR from a larger network CIDR. It is useful for dynamically creating network ranges.

---

### Variable Precedence

Terraform uses different sources to assign values to variables. When the same variable is defined in multiple places, the higher-priority value takes precedence.

The precedence from lowest to highest is:

1. Variable defaults
2. Environment variables
3. Automatically loaded variable files
4. Explicit `-var-file`
5. CLI `-var`

For example, `instance_type = "t2.micro"` in a variable file can be overridden with:

```bash
terraform plan -var="instance_type=t2.nano"
```

### Five Useful Built-in Functions

- `upper()` converted text to uppercase.
- `join()` combined multiple strings with a separator.
- `length()` returned the number of elements or characters.
- `lookup()` retrieved a value from a map using a key.
- `cidrsubnet()` calculated subnet CIDRs dynamically.

### Difference Between `variable`, `local`, `output`, and `data`

| Terraform Block | Purpose |
| --- | --- |
| `variable` | Accepted input values from the user or environment |
| `local` | Created reusable calculated values inside the configuration |
| `output` | Displayed useful values after Terraform applied the configuration |
| `data` | Read existing information from a provider without creating it |

