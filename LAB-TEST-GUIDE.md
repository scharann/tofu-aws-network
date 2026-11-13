# End-to-End Lab Test & Verification Guide (Isolated State Architecture)

## Purpose
This guide provides a complete, step-by-step runbook to test the cloned `tofu-aws-network` and `tofu-aws-compute` repositories from scratch—covering AWS SSO authentication, S3/DynamoDB backend provisioning and verification, isolated state deployment, and full resource teardown.

---

## Step 1: Authenticate AWS CLI & Create Backend Prerequisites

1. **Export your AWS profile and verify active SSO credentials:**
~~~bash
export AWS_PROFILE=<YOUR-PROFILE-NAME>
aws sts get-caller-identity || aws sso login
~~~

2. **Confirm the S3 bucket name configured in `backend.tf`:**
~~~bash
grep "bucket" tofu-aws-network/backend.tf
~~~

3. **Create the S3 state bucket (with versioning enabled) and DynamoDB lock table in `ap-southeast-2`:**
~~~bash
aws s3api create-bucket \
  --bucket <YOUR-S3-BUCKET-NAME> \
  --region ap-southeast-2 \
  --create-bucket-configuration LocationConstraint=ap-southeast-2

aws s3api put-bucket-versioning \
  --bucket <YOUR-S3-BUCKET-NAME> \
  --versioning-configuration Status=Enabled

aws dynamodb create-table \
  --table-name tofu-lab-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region ap-southeast-2
~~~

4. **Verify both backend prerequisites exist and are ready:**
~~~bash
aws s3 ls s3://<YOUR-S3-BUCKET-NAME>/
aws dynamodb describe-table --table-name tofu-lab-locks --region ap-southeast-2 --query "Table.TableStatus"
~~~
*Expected:* `aws s3 ls` succeeds without a `NoSuchBucket` error, and DynamoDB returns `"ACTIVE"`.

---

## Step 2: Deploy & Verify the Network Stack (`tofu-aws-network`)

1. **Navigate to `tofu-aws-network` and initialize/apply:**
~~~bash
cd ~/Downloads/Charan/Tofu-Lab-Github-Repositories/tofu-aws-network
tofu init
tofu plan
tofu apply -auto-approve
~~~

2. **Verify network deployment and isolated S3 state:**
~~~bash
tofu state list
tofu output
aws s3 ls s3://<YOUR-S3-BUCKET-NAME>/network/
~~~
*Expected:*
- `tofu state list` shows all 8 network resources (`aws_vpc.main`, 4 subnets, `aws_internet_gateway.igw`, route tables/associations).
- `tofu output` displays populated `vpc_id` and `public_subnet_ids`.
- `aws s3 ls` confirms `terraform.tfstate` exists under the `network/` prefix.

---

## Step 3: Deploy & Verify the Compute Stack (`tofu-aws-compute`)

1. **Switch to `tofu-aws-compute` and re-verify AWS session & backend prerequisites:**
~~~bash
cd ~/Downloads/Charan/Tofu-Lab-Github-Repositories/tofu-aws-compute

export AWS_PROFILE=<YOUR-PROFILE-NAME>
aws sts get-caller-identity || aws sso login

aws s3 ls s3://<YOUR-S3-BUCKET-NAME>/
aws dynamodb describe-table --table-name tofu-lab-locks --region ap-southeast-2 --query "Table.TableStatus"
~~~

2. **Initialize and apply the compute stack:**
~~~bash
tofu init
tofu plan
tofu apply -auto-approve
~~~

3. **Verify compute deployment, isolated S3 state, and running EC2 instance:**
~~~bash
tofu state list
aws s3 ls s3://<YOUR-S3-BUCKET-NAME>/compute/
aws ec2 describe-instances \
  --region ap-southeast-2 \
  --filters "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].[InstanceId,VpcId,SubnetId,PublicIpAddress]" \
  --output table
tofu state show aws_instance.web
~~~
*Expected:*
- `tofu state list` tracks only `aws_security_group.ec2_sg`, `aws_instance.web`, and `data.terraform_remote_state.network` (confirming zero collision with the network stack).
- `aws s3 ls` confirms a separate `terraform.tfstate` file exists under `compute/`.
- `aws ec2 describe-instances` confirms the EC2 instance is running inside the VPC created by `tofu-aws-network`.
- `tofu state show aws_instance.web` displays the full state attributes (`id`, `vpc_security_group_ids`, `subnet_id`, `public_ip`, `instance_state`) recorded in `compute/terraform.tfstate`.

---

## Step 4: Tear Down & Clean Up All Lab Resources

Always destroy resources in **reverse dependency order** (compute first, then network):

1. **Destroy the compute stack first:**
~~~bash
cd ~/Downloads/Charan/Tofu-Lab-Github-Repositories/tofu-aws-compute
tofu destroy -auto-approve
tofu state list
~~~

2. **Destroy the network stack second:**
~~~bash
cd ~/Downloads/Charan/Tofu-Lab-Github-Repositories/tofu-aws-network
tofu destroy -auto-approve
tofu state list
~~~

3. **Verify zero EC2/VPC lab resources remain in AWS (`ap-southeast-2`):**
~~~bash
aws ec2 describe-instances \
  --region ap-southeast-2 \
  --filters "Name=instance-state-name,Values=pending,running,stopping,stopped" \
  --query "Reservations[].Instances[].InstanceId"

aws ec2 describe-vpcs \
  --region ap-southeast-2 \
  --query "Vpcs[?IsDefault==\`false\`].VpcId"
~~~
*Expected:* Both commands return empty arrays `[]`.

4. **(Optional) Remove the S3 state bucket and DynamoDB lock table:**
If you are completely finished testing and want zero S3/DynamoDB footprint left in AWS:
~~~bash
# Delete all versioned objects and delete markers inside the S3 bucket
aws s3api delete-objects --bucket <YOUR-S3-BUCKET-NAME> \
  --delete "$(aws s3api list-object-versions --bucket <YOUR-S3-BUCKET-NAME> \
  --query='{Objects: Versions[].{Key:Key,VersionId:VersionId}}' --output json)"

# Delete the empty S3 bucket and DynamoDB lock table
aws s3api delete-bucket --bucket <YOUR-S3-BUCKET-NAME> --region ap-southeast-2
aws dynamodb delete-table --table-name tofu-lab-locks --region ap-southeast-2
~~~
