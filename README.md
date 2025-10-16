# terraform-remote

## Using the AWS remote Terraform state

1) In your Terraform stack, declare an empty S3 backend:

    terraform {
      backend "s3" {}
    }

2) Initialize Terraform pointing at this repo’s backend config:

    terraform init -reconfigure -backend-config=../backend/dev.hcl

This stores state in a versioned, KMS-encrypted S3 bucket with DynamoDB state locking.
