---
name: ytf-setup
description: "First-time setup for Yolo Terraform. Generates .yolotf.yaml config with auto-discovered projects, verifies prerequisites (terraform, git, gh, aws), and ensures DynamoDB lock table exists (creates it if missing). Use when: (1) setting up yolotf for the first time, (2) user says /ytf:setup, (3) .yolotf.yaml is missing."
---

# Yolo Terraform — First-Time Setup

Read `../ytf/references/yolotf-reference.md` for table creation commands and config schema.

## Steps

### 1. Check Prerequisites

Verify each is in PATH and functional:
- `terraform --version`
- `git --version`
- `gh auth status`
- `aws sts get-caller-identity` (basic AWS CLI works)

Report missing/failing tools and stop if any fail.

### 2. Generate Config

Ask user for:
- **Username** (suggest: `git config user.name` output)
- **Lock TTL** (default: `168h`)
- **Terraform AWS profile** (the profile used for `terraform plan/apply`)
- **DynamoDB AWS profile** (the profile with access to the lock table)
- **DynamoDB region** (default: `us-east-1`)

Auto-discover projects by scanning for `backend.tf`:
```bash
find . -name backend.tf -not -path '*/.terraform/*' | sort
```

For each `backend.tf`, derive:
- `dir`: directory path relative to repo root
- `name`: path with `/` replaced by `-`

Present discovered projects for confirmation. Allow add/remove/rename.

Write `.yolotf.yaml` to repo root.

### 3. Ensure DynamoDB Table Exists

Check if the lock table exists using the profile and region from the config just generated:
```bash
aws dynamodb describe-table --table-name <tableName> \
  --region <region> --profile <dynamodb.profile> 2>&1
```

If the table exists, report it and continue to step 4.

If the table does **not** exist, offer to create it:
```
DynamoDB table "<tableName>" not found in <region>.

  A) Create it now (via AWS CLI)
  B) Skip — I'll create it myself
```

**Option A — Create table:**
```bash
aws dynamodb create-table \
  --table-name <tableName> \
  --region <region> --profile <dynamodb.profile> \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST

aws dynamodb update-time-to-live \
  --table-name <tableName> \
  --region <region> --profile <dynamodb.profile> \
  --time-to-live-specification Enabled=true,AttributeName=ExpiresAt
```

Wait for the table to become ACTIVE:
```bash
aws dynamodb wait table-exists --table-name <tableName> \
  --region <region> --profile <dynamodb.profile>
```

Report: `"Created DynamoDB table <tableName> with TTL on ExpiresAt."`

### 4. Ensure tfplan in .gitignore

Append if missing:
```
# yolotf plan files
tfplan
```

### 5. Verify

Test scan to confirm access:
```bash
aws dynamodb scan --table-name <tableName> \
  --region <region> --profile <dynamodb.profile> --max-items 1
```

Then verify the terraform profile works:
```bash
aws sts get-caller-identity --profile <terraform.profile>
```

Report success with config summary.
