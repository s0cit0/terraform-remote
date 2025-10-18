# Terraform Remote State — Bootstrap Archived (Infra Kept)

The AWS backend infrastructure (**S3 + DynamoDB + KMS**) is **already deployed** and remains running.
We removed the bootstrap IaC code from this repo to keep it clean. You can restore it from the safety tag if needed.

## Restore the bootstrap code from tag
Latest tag (auto-detected): `infra-bootstrap-20251017-211937`

```bash
git fetch --tags
git checkout -b restore-bootstrap infra-bootstrap-20251017-211937
# make changes as needed, then:
# terraform init/plan/apply
```

## Using this backend in other repos
1) Copy the example file to a local config:
```bash
cp backend/dev.example.hcl backend/dev.hcl
```
2) Edit `backend/dev.hcl` with your real values:
- `<STATE_BUCKET_NAME>` (S3 bucket)
- `<STATE_KEY_PATH>` (e.g., repo/dev/terraform.tfstate)
- `<AWS_REGION>` (e.g., us-east-1)
- `<DDB_LOCK_TABLE_NAME>`
- `<KMS_KEY_ARN>`

3) Ensure Terraform has an empty backend block:
```hcl
terraform { backend "s3" {} }
```

4) Initialize:
```bash
terraform init -reconfigure -backend-config=backend/dev.hcl
```

## Notes
- Do **not** commit `backend/dev.hcl` (real values). Commit only `backend/dev.example.hcl`.
- `.terraform.lock.hcl` is safe to commit (pins provider versions).
- Backend enforces KMS encryption and DynamoDB locking.
