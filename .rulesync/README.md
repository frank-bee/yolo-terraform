# Yolo Terraform — AI-Driven Terraform Orchestrator

> **ALPHA** — This is an early-stage experiment. Expect rough edges, breaking changes, and missing features. Use at your own risk.

**Yolo Terraform replaces Atlantis.** Instead of running a dedicated server that watches PRs and executes Terraform via webhook-triggered workflows, it moves plan/apply execution to the developer's machine — orchestrated by their AI coding agent.

The agent reads skill files (plain markdown) that teach it exactly how to run `terraform`, coordinate locks via DynamoDB, and manage the PR lifecycle via `gh` CLI. No server to operate, no webhook infrastructure, no container to maintain. The developer's AI agent becomes the Terraform execution engine.

## Skills

| Skill | Command | Purpose |
|-------|---------|---------|
| [ytf](skills/ytf/SKILL.md) | `/ytf [project]` | Main orchestrator — auto-detects affected projects from git changes, acquires DynamoDB locks, runs terraform plan/apply, manages PR lifecycle |
| [ytf-setup](skills/ytf-setup/SKILL.md) | `/ytf:setup` | First-time setup — checks prerequisites, verifies DynamoDB lock table, generates `.yolotf.yaml` config |
| [ytf-status](skills/ytf-status/SKILL.md) | `/ytf:status` | Status overview — lists all projects and current DynamoDB locks with holder, PR state, and age |
| [ytf-unlock](skills/ytf-unlock/SKILL.md) | `/ytf:unlock [project]` | Release a project lock — shows your locks if no project specified, supports force-unlock |
| [ytf-help](skills/ytf-help/SKILL.md) | `/ytf:help` | Show available commands and usage |

## How It Works

1. Developer invokes `/ytf` in their AI agent (Claude Code, Cursor, etc.)
2. Agent detects which terraform projects have changes (git diff vs main)
3. Acquires a DynamoDB lock to prevent concurrent operations on the same project
4. Runs `terraform init` + `plan`, shows output
5. Developer chooses: create PR (with plan as comment), apply directly, or quit
6. On apply success, lock is released; on PR creation, lock tracks the PR number
7. Stale locks from merged/closed PRs are auto-cleaned on every run

## What It Preserves from Atlantis

- Project-level locking (no concurrent plan/apply on the same stack)
- Plan output posted to PRs for team review
- Apply results posted to PRs for audit trail
- YAML-driven project config (`.yolotf.yaml` replaces `atlantis.yaml`)

## What It Eliminates

- Atlantis server deployment and maintenance
- Webhook infrastructure
- Server-side credentials management
- Atlantis-specific CI/CD pipeline

## Prerequisites

- `terraform`, `git`, `gh`, `aws` CLI in PATH
- AWS profile with permissions for terraform plan/apply (configured as `terraform.profile` in `.yolotf.yaml`)
- AWS profile with DynamoDB access for the lock table (configured as `dynamodb.profile` in `.yolotf.yaml`)
- Run `/ytf:setup` once to generate config and verify the lock table exists

## Rulesync

Skills are managed via [rulesync](https://github.com/dyoshikawa/rulesync). Source of truth is in `.rulesync/skills/`. Generated output goes to `.claude/skills/`, `.github/skills/`, and `.cursor/skills/`.

### Generating skills

Generate for all configured targets (claudecode, copilot, cursor):
```bash
rulesync generate
```

Generate for your tool only:
```bash
rulesync generate --targets claudecode   # Claude Code
rulesync generate --targets copilot      # GitHub Copilot
rulesync generate --targets cursor       # Cursor
```
