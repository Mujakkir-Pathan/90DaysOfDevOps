# Day 64 -- Terraform State Management and Remote Backends

## Task
The state file is the single most important thing in Terraform. It is the source of truth -- the map between your `.tf` files and what actually exists in the cloud. Lose it and Terraform forgets everything. Corrupt it and your next apply could destroy production.

Today you learn to manage state like a professional -- remote backends, locking, importing existing resources, and handling drift.

---

## All tf files

[terrform files](https://github.com/Mujakkir-Pathan/terraform-terraweak-90day/tree/main/day-4)

---

## All sceenshots

[screenshots of task](screenshots/)

---

## Challenge Tasks

### Task 1: Inspect Your Current State
Used the Day 63 configuration and explored the Terraform state.

```bash
terraform show
terraform state list
terraform state show aws_instance.main
terraform state show aws_vpc.main
```

Answer:
1. How many resources does Terraform track?

Terraform tracked **10 state entries**: **8 managed AWS resources and 2 data sources**.

2. What attributes does the state store for an EC2 instance? (hint: way more than what you defined)

The state stored many attributes for the EC2 instance, including its **instance ID and ARN, AMI, availability zone, instance type, CPU details, networking information, security group IDs, tags, storage details, metadata options, monitoring settings, tenancy, lifecycle settings, and other provider-generated attributes**. It contained much more information than was explicitly defined in the `.tf` configuration.

3. Open `terraform.tfstate` in an editor -- find the `serial` number. What does it represent?

---

### Task 2: Set Up S3 Remote Backend
Moved Terraform state from local storage to an S3 remote backend with DynamoDB locking.

1. Created the backend infrastructure manually or in a separate Terraform configuration.

2. Added the S3 backend block to the Terraform configuration.

3. Ran `terraform init` and migrated the existing state to the new backend.

4. Verified the migrated state in S3 and confirmed that `terraform plan` showed no changes.

---

### Task 3: Test State Locking
Tested state locking by using two terminals in the same project directory.

1. Opened two terminals in the same project directory.

2. Ran `terraform apply` in Terminal 1.

3. Ran `terraform plan` in Terminal 2 while Terminal 1 was waiting for confirmation.

4. Confirmed that Terminal 2 displayed a lock error with a Lock ID.

**Document:** What is the error message? Why is locking critical for team environments?

5. Used `terraform force-unlock <LOCK_ID>` only when a stale lock needed to be removed and no other Terraform operation was running.

---

### Task 4: Import an Existing Resource
Imported an existing S3 bucket into Terraform management.

1. Manually created an S3 bucket named `terraweek-import-test-<yourname>`.

2. Added a `resource "aws_s3_bucket"` block for the bucket with its name.

3. Imported the bucket into Terraform state.

4. Ran `terraform plan` and updated the configuration until it matched the existing resource with no unexpected changes.

5. Ran `terraform state list` and confirmed that the imported bucket appeared alongside the other resources.

**Document:** What is the difference between `terraform import` and creating a resource from scratch?

---

### Task 5: State Surgery -- mv and rm
Used Terraform state commands to rename a resource and remove it from state without destroying it in AWS.

1. **Renamed a resource in state:**

Used `terraform state list` to identify the current resource name and used `terraform state mv` to rename the resource in state.

Updated the `.tf` file to match the new name and confirmed with `terraform plan` that there were no changes.

2. **Removed a resource from state (without destroying it):**

Used `terraform state rm` to remove the bucket from Terraform state without destroying it in AWS.

Ran `terraform plan` and confirmed that Terraform no longer tracked the bucket while the bucket still existed in AWS.

3. **Re-imported it** to bring it back:

Imported the bucket back into Terraform state.

**Document:** When would you use `state mv` in a real project? When would you use `state rm`?

---

### Task 6: Simulate and Fix State Drift
Simulated infrastructure drift by changing an AWS resource outside Terraform and reconciled the configuration.

1. Applied the full configuration so the infrastructure was in sync.

2. Went to the **AWS console** and manually changed the Name tag of the EC2 instance to `"ManuallyChanged"` and changed the instance type if it was stopped or added a new tag.

3. Ran `terraform plan` and detected the difference between the desired configuration and the actual infrastructure.

4. Considered the two available choices:
   - **Option A:** Ran `terraform apply` to force reality back to match the configuration (reconcile)
   - **Option B:** Updated the `.tf` files to match the manual change (accept the drift)

5. Chose Option A, applied the configuration, and verified that the tags were restored.

6. Ran `terraform plan` again and confirmed that it showed "No changes."

**Document:** How do teams prevent state drift in production? (hint: restrict console access, use CI/CD for all changes)

---

### 1. Diagram: Local State vs Remote State Setup

```text
Local State
Terraform → terraform.tfstate → Local machine

Remote State
Terraform → S3 Bucket
              ↓
        terraform.tfstate
              ↓
       DynamoDB Locking
```

### 2. Steps followed for `terraform import` and the result

* Created the S3 bucket manually in AWS.
* Added the `aws_s3_bucket` resource to the Terraform configuration.
* Ran `terraform import` to bring the bucket into Terraform state.
* Ran `terraform plan` and checked for differences.
* Successfully added the bucket to Terraform state.

### 3. Explanation of State Drift with the real example

State drift happened when I changed the EC2 instance's **Name tag** directly in the AWS console to `ManuallyChanged`. Terraform detected the difference during `terraform plan`. I ran `terraform apply`, which restored the tag to the value defined in my Terraform configuration.

### 4. When to use: `state mv`, `state rm`, `import`, `force-unlock`, `refresh`

| Command        | When to use                                                                                              |
| -------------- | -------------------------------------------------------------------------------------------------------- |
| `state mv`     | When renaming or moving a resource in Terraform state without recreating it.                             |
| `state rm`     | When removing a resource from Terraform management without deleting it from AWS.                         |
| `import`       | When bringing an existing AWS resource under Terraform management.                                       |
| `force-unlock` | When a stale state lock remains and no Terraform operation is running.                                   |
| `refresh`      | When updating Terraform state to reflect the current infrastructure without changing the infrastructure. |

