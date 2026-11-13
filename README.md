# OpenTofu AWS Network (Isolated State Architecture)

## Purpose & Problem Solved
This repository provisions foundational AWS networking infrastructure (VPC, public and private subnets, Internet Gateway, and route tables) in `ap-southeast-2` using an **isolated S3 state key** (`network/terraform.tfstate`).

**The Problem Solved:**
In multi-repository OpenTofu/Terraform architectures, sharing a remote state file across root modules causes state overwrites and destructive deletion races. By isolating the network stack in its own S3 state key and exporting outputs (`vpc_id`, `public_subnet_ids`), downstream repositories like `tofu-aws-compute` can read network IDs read-only via `terraform_remote_state` without risking state corruption.

## Architecture & State Configuration
- **Region:** `ap-southeast-2`
- **Backend Bucket:** `<YOUR-S3-BUCKET-NAME>`
- **Isolated State Key:** `network/terraform.tfstate`
- **Lock Table:** `tofu-lab-locks`
- **Exported Outputs:** `vpc_id`, `public_subnet_ids`, `private_subnet_ids`

## How to Deploy (Step 1)
1. Authenticate your AWS SSO session (`aws sso login` and `export AWS_PROFILE=<YOUR-PROFILE>`).
2. Initialize and apply the network layer:
~~~bash
tofu init
tofu plan
tofu apply
~~~

## How to Verify Deployment
- **Check state objects:** Run `tofu state list` — confirms the VPC, 4 subnets, Internet Gateway, and route tables are tracked.
- **Check exported outputs:** Run `tofu output` — confirms `vpc_id` and `public_subnet_ids` are populated for `tofu-aws-compute` to consume.
- **Verify S3 key:** Run `aws s3 ls s3://<YOUR-S3-BUCKET-NAME>/network/` — confirms `terraform.tfstate` exists under the `network/` prefix.

## How to Clean Up Resources
Always destroy downstream compute resources in `tofu-aws-compute` first before destroying the network layer:
1. Destroy the network stack:
~~~bash
tofu destroy
~~~
2. **Verify cleanup:**
   - Run `tofu state list` — returns zero resources.
   - Run `aws ec2 describe-vpcs --region ap-southeast-2` — confirms the lab VPC has been deleted.
