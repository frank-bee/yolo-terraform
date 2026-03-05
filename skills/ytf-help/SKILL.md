---
name: ytf-help
description: "Show available Yolo Terraform commands and usage. Use when: (1) user says /ytf:help, (2) user asks what ytf/yolotf commands are available, (3) user needs help with yolotf."
---

# Yolo Terraform — Command Reference

Display the following help overview to the user:

## Commands

| Command | Description |
|---------|-------------|
| `/ytf [project]` | Run terraform plan/apply with DynamoDB locking and PR lifecycle. Auto-detects affected projects if no argument given. |
| `/ytf:setup` | First-time setup: verify prerequisites, auto-discover projects, generate `.yolotf.yaml`, verify DynamoDB lock table exists. |
| `/ytf:status` | Show all projects with lock status, PR state, holder, and age. Flags stale locks. |
| `/ytf:unlock [project]` | Release a project lock. Shows your locks if no project specified. Supports force-unlock for other users' locks. |
| `/ytf:help` | Show this help overview. |

## Workflows

After planning, you choose one of three paths:

### A) Apply Now (no PR)
For quick changes that don't need review. Plan → apply → done.

### B) Review First, Apply Later
For changes that need team sign-off before applying:
1. Plan → create PR with plan output
2. Re-plan as needed (you're told what changed since last plan)
3. Get review
4. Apply → result posted to PR
5. Lock auto-cleans when PR is merged/closed

### C) Apply First, Review Later
For confident changes where you want an audit trail:
1. Plan → create PR → apply immediately
2. Apply result posted to PR
3. PR stays open for review
4. Lock auto-cleans when PR is merged/closed

## Quick Examples

- `ytf` — auto-detect changes and plan affected projects
- `ytf k8s-dev` — plan a specific project
- `ytf:status` — see who holds locks and PR states
- `ytf:unlock` — release your locks (interactive picker)
- `ytf:unlock k8s-dev` — unlock a specific project
- `ytf:setup` — reconfigure or first-time setup

## Notes

- All terraform commands run with `AWS_PROFILE` set to `terraform.profile` from config
- DynamoDB lock commands use `dynamodb.profile`, `dynamodb.region`, and `dynamodb.tableName` from config
- Locks are automatically cleaned up when PRs are merged or closed
- Config lives in `.yolotf.yaml` at repo root
