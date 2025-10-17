# Terraform Remote State Backend (AWS S3 + DynamoDB + KMS)

This repository (or any repo using these instructions) connects Terraform to a **remote state** stored in AWS:
- S3 bucket (versioned, encrypted with KMS)
- DynamoDB table (state locking)
- KMS key (encryption)

Everything below uses **generic placeholders** you should replace.

---

## Prerequisites

- The AWS resources already exist (provisioned by a separate “bootstrap” project or manually):
  - S3 bucket: <STATE_BUCKET_NAME>  (example: org-dev-tfstate-123456789012-us-east-1)
  - DynamoDB table: <DDB_LOCK_TABLE_NAME>  (example: org-dev-tf-locks)
  - KMS key ARN: <KMS_KEY_ARN>  (example: arn:aws:kms:us-east-1:123456789012:key/abcd-...)
  - AWS region: <AWS_REGION>  (example: us-east-1)
- Terraform v1.6+ and AWS CLI configured.

---

## 1) Create the backend config file

Create **backend/dev.hcl** with your values (replace the angle-bracketed placeholders):

    bucket         = "<STATE_BUCKET_NAME>"
    key            = "global/dev/terraform.tfstate"
    region         = "<AWS_REGION>"
    dynamodb_table = "<DDB_LOCK_TABLE_NAME>"
    encrypt        = true
    kms_key_id     = "<KMS_KEY_ARN>"

> You may keep different files per environment (e.g., `backend/stage.hcl`, `backend/prod.hcl`) with different keys and tables.

---

## 2) Add the backend stub to your Terraform stack

Add an **empty** S3 backend block to a Terraform file (e.g., `main.tf`):

    terraform {
      backend "s3" {}
    }

Initialize Terraform to point at your backend config:

    terraform init -reconfigure -backend-config=backend/dev.hcl

---

## 3) Validate the backend

Optional checks:

    # Confirm state is readable (prints JSON header lines)
    terraform state pull | head

    # Check the state object in S3 (replace placeholders)
    aws s3api list-objects-v2 \
      --bucket "<STATE_BUCKET_NAME>" \
      --prefix "global/dev/terraform.tfstate"

    # Check the DynamoDB table (replace placeholders)
    aws dynamodb describe-table \
      --table-name "<DDB_LOCK_TABLE_NAME>" \
      --region "<AWS_REGION>" \
      --query 'Table.TableStatus'

    # Check the KMS key (replace placeholders)
    aws kms describe-key \
      --key-id "<KMS_KEY_ARN>" \
      --region "<AWS_REGION>" \
      --query 'KeyMetadata.KeyState'

---

## 4) Security & hygiene

- **Do not commit** `*.tfstate`, `*.tfvars` with secrets, or `.terraform/` directories.
- **Do commit** `.terraform.lock.hcl` to pin provider versions.
- Use provider default tags to enforce ownership/cost attribution (example):

    provider "aws" {
      region = "<AWS_REGION>"
      default_tags {
        tags = {
          owner = "MLR"   # adjust to your policy
        }
      }
    }

---

## 5) Suggested .gitignore

Place this in your repo’s `.gitignore`:

    # Terraform
    **/.terraform/
    *.tfstate
    *.tfstate.*
    crash.log
    *.tfplan

    # Keep the lockfile tracked for deterministic builds
    # .terraform.lock.hcl  <-- DO NOT ignore this file

---

## FAQ

- **Can I change the S3 key path?** Yes. Change `key` in `backend/dev.hcl` (e.g., `workspaces/dev/app1.tfstate`) and re-run `terraform init -reconfigure`.
- **How do I reuse across many repos?** Copy `backend/dev.hcl` (with your values) into each repo, add the backend stub, then run `terraform init -reconfigure -backend-config=backend/dev.hcl`.


## Backend config template (local-only values)
This repo tracks a template at `backend/dev.example.hcl` and **ignores** real backend files (see `.gitignore`).
To use the remote state in a new clone:

1) Copy the template and fill in values:
   cp backend/dev.example.hcl backend/dev.hcl
   # edit backend/dev.hcl and replace:
   #   <STATE_BUCKET_NAME>, <AWS_REGION>, <DDB_LOCK_TABLE_NAME>, <KMS_KEY_ARN>

2) Add an empty S3 backend block to your Terraform stack:
   terraform {
     backend "s3" {}
   }

3) Initialize Terraform against the backend config:
   terraform init -reconfigure -backend-config=backend/dev.hcl
