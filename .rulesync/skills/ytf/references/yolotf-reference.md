# yolotf Reference

## Config File

Location: repo-level `.yolotf.yaml` (takes precedence) or `~/.yolotf.yaml`.

```yaml
version: 1
user: jane                         # lock holder identity
lockTTL: "168h"                    # DynamoDB TTL (default 7 days, 0 = no expiry)
terraform:
  profile: MyTerraformProfile      # AWS profile for terraform plan/apply
dynamodb:
  tableName: yolotf-locks
  region: us-east-1
  profile: MyDynamoDBProfile       # AWS profile with DynamoDB access
projects:
  - name: app-dev
    dir: stacks/dev
  - name: app-prod
    dir: stacks/prod
  - name: networking
    dir: networking
```

## DynamoDB Lock Table

The table name, region, and profile are read from `dynamodb` config.

This is NOT Terraform state locking. It's application-level locking to prevent two developers from planning/applying the same project simultaneously.

### Schema

| Attribute | Type | Description |
|-----------|------|-------------|
| `LockID` (PK) | String | `yolotf-lock-<8char-sha256-of-repo-url>-<project>` |
| `Holder` | String | User identity (from config `user` field) |
| `Repo` | String | Repo slug (e.g., `github.com/my-org/infra`) |
| `Project` | String | Project name |
| `PR` | Number | PR number (0 = none) |
| `AcquiredAt` | String | ISO 8601 |
| `ExpiresAt` | Number | Unix epoch (DynamoDB TTL). Omit for no expiry |

### LockID Construction

```bash
# Repo URL normalized (strip .git, protocol)
REPO_SLUG="github.com/my-org/infra"
HASH=$(echo -n "$REPO_SLUG" | shasum -a 256 | cut -c1-8)
LOCK_ID="yolotf-lock-${HASH}-${PROJECT_NAME}"
```

### Lock Operations

All commands use `--table-name <tableName> --region <region> --profile <profile>` from the `dynamodb` config section.

**Acquire** (conditional put — fails if another user holds lock):
```bash
aws dynamodb put-item \
  --table-name <tableName> \
  --region <region> --profile <profile> \
  --item '{"LockID":{"S":"<id>"},"Holder":{"S":"<user>"},"Repo":{"S":"<repo>"},"Project":{"S":"<project>"},"PR":{"N":"0"},"AcquiredAt":{"S":"<iso8601>"},"ExpiresAt":{"N":"<epoch>"}}' \
  --condition-expression "attribute_not_exists(LockID) OR Holder = :h" \
  --expression-attribute-values '{":h":{"S":"<user>"}}'
```

**Update PR** (set PR number on existing lock):
```bash
aws dynamodb update-item \
  --table-name <tableName> \
  --region <region> --profile <profile> \
  --key '{"LockID":{"S":"<id>"}}' \
  --update-expression "SET PR = :pr" \
  --condition-expression "Holder = :h" \
  --expression-attribute-values '{":h":{"S":"<user>"},":pr":{"N":"<pr_number>"}}'
```

**Release** (conditional delete — only if you hold it):
```bash
aws dynamodb delete-item \
  --table-name <tableName> \
  --region <region> --profile <profile> \
  --key '{"LockID":{"S":"<id>"}}' \
  --condition-expression "Holder = :h" \
  --expression-attribute-values '{":h":{"S":"<user>"}}'
```

**Force-unlock** (unconditional delete):
```bash
aws dynamodb delete-item \
  --table-name <tableName> \
  --region <region> --profile <profile> \
  --key '{"LockID":{"S":"<id>"}}'
```

**Scan all locks**:
```bash
aws dynamodb scan --table-name <tableName> \
  --region <region> --profile <profile>
```

### Table Creation (one-time)

```bash
aws dynamodb create-table \
  --table-name <tableName> \
  --region <region> --profile <profile> \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST

aws dynamodb update-time-to-live \
  --table-name <tableName> \
  --region <region> --profile <profile> \
  --time-to-live-specification Enabled=true,AttributeName=ExpiresAt
```

## Terraform Execution

**Profile**: Use the `terraform.profile` value from config for all terraform commands.

```bash
export AWS_PROFILE=<terraform.profile>
terraform -chdir=<project_dir> init
terraform -chdir=<project_dir> plan -out=tfplan
terraform -chdir=<project_dir> show tfplan   # human-readable output
terraform -chdir=<project_dir> apply tfplan
```

Do NOT modify `backend.tf` or `provider.tf` files — they contain `assume_role` configurations for cross-account access.

## TTL Calculation

Parse `lockTTL` from config (e.g., `"168h"`) and compute `ExpiresAt` as current Unix epoch + TTL seconds. If `lockTTL` is `"0"` or omitted, do not set `ExpiresAt` attribute.
