---
name: ytf-status
description: "Show Yolo Terraform status overview: lists all projects from config, scans DynamoDB locks showing holder, PR state, and age. Use when: (1) checking who has locks on which projects, (2) user says /ytf:status, (3) reviewing project lock state before planning."
---

# Yolo Terraform — Project & Lock Overview

Read `../ytf/references/yolotf-reference.md` for config and DynamoDB details.

## Steps

### 1. Read Config

Read `.yolotf.yaml` (repo root, fallback `~/.yolotf.yaml`). If missing, tell user to run setup.

### 2. Scan Locks

```bash
aws dynamodb scan --table-name <tableName> \
  --region <region> --profile <profile>
```

### 3. Display Status

For each lock with PR > 0, check PR state: `gh pr view <N> --json state -q .state`

Format as table:

```
Project            Holder   PR     State    Age
──────────────────────────────────────────────────
app-dev            jane     #42    OPEN     2h ago
networking         bob      #55    MERGED   1d ago  ⚠ stale
app-prod           —        —      —        —
```

- Projects with no lock show `—` in all columns.
- Mark MERGED/CLOSED locks as `⚠ stale` — suggest running `/ytf` to auto-clean.
