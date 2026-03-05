---
name: ytf
description: "Yolo Terraform — plan/apply orchestrator with DynamoDB locking and PR lifecycle management. Replaces Atlantis with local AI-driven execution. Use when: (1) running terraform plan or apply, (2) managing terraform project locks, (3) creating PRs with terraform plan output, (4) applying terraform and posting results to PRs, (5) user says /ytf or mentions yolotf. Detects affected projects from git changes, acquires DynamoDB locks, runs terraform, and manages the full PR lifecycle via gh CLI."
---

# Yolo Terraform — Plan/Apply Orchestrator

Read `references/yolotf-reference.md` for DynamoDB lock operations, config schema, and terraform execution details.

## Argument

Optional `[project]` name. If omitted, auto-detect affected projects from git changes.

## Flow

### Phase 0: Config + Housekeeping (every run)

1. Read config from `.yolotf.yaml` (repo root, fallback `~/.yolotf.yaml`). If missing, tell user to run setup.
2. Construct repo slug: normalize git remote URL (strip protocol, `.git` suffix).
3. Scan DynamoDB for all locks. For each lock held by current user with PR > 0:
   - Check PR state: `gh pr view <N> --json state -q .state`
   - If MERGED or CLOSED → release lock, report: `"Housekeeping: released lock for <project> (PR #N <state>)"`
4. Report: `"Housekeeping: no stale locks found."` if none released.

### Phase 1: Project Selection

**If project specified**: validate it exists in config, proceed to Phase 2.

**If no argument**: detect affected projects:
1. Collect changed files (union, deduplicated):
   ```bash
   git diff --name-only main...HEAD 2>/dev/null
   git diff --cached --name-only
   git diff --name-only
   ```
2. Match each changed file against project `dir` paths (prefix match).
3. Present affected projects for selection (with file count per project).
   - Offer "plan all affected" and individual selection.
   - If no projects affected, show all projects as fallback.

### Phase 2: State Detection + Lock + Plan

For each selected project, check DynamoDB for existing lock:

| State | Condition | Action |
|-------|-----------|--------|
| Fresh | No lock | Acquire lock → plan → Phase 3 |
| Locked by you, no PR | Lock exists, PR=0 | → Phase 3 (offer re-plan) |
| Locked by you, PR open | Lock exists, PR>0 | → Phase 4 |
| Locked by someone else | Different holder | Show holder, offer force-unlock |

When planning (use `terraform.profile` from config):
```bash
export AWS_PROFILE=<terraform.profile>
terraform -chdir=<dir> init
terraform -chdir=<dir> plan -out=tfplan
terraform -chdir=<dir> show tfplan
```

### Phase 3: Post-Plan — Choose Your Workflow

After showing the plan, present the three workflow options clearly:

```
What would you like to do?

  A) Apply now (no PR)     — apply immediately, release lock, done
  B) Review first, apply later — create PR → get review → apply → merge
  C) Apply first, review later — create PR → apply → get review → merge

  Other: Show plan | Quit (keep lock)
```

**Option A — Apply now (no PR)**:
`terraform apply tfplan` → release lock on success. On failure: offer retry/re-plan/unlock.

**Option B — Review first, apply later**:
Create PR → Phase 4B (review-first menu).

**Option C — Apply first, review later**:
Create PR → apply immediately → Phase 4C (review-after menu).

**Create PR** (shared by B and C):
- If on main/master: create branch `yolotf/<project>`, commit, push
- `gh pr create --draft`, post plan as PR comment wrapped in ```hcl code fence
- Update DynamoDB lock PR attribute

### Phase 4B: Review First, Apply Later

The user wants review before applying. Present this menu:

```
PR #<N> is open for <project>. Waiting for review before apply.

  1) Re-plan       — run plan again (will report changes vs previous plan)
  2) Apply         — apply the current plan
  3) Show plan     — display current plan output
  4) Wait          — exit, keep lock (resume later with /ytf <project>)
  5) Abandon       — release lock (PR stays open)
```

**Re-plan**: run plan again. Compare new plan against the previous one and **report what changed** (new/removed/modified resources). Post updated plan as new PR comment. Return to Phase 4B.

**Apply**: `terraform apply tfplan` → post apply result as PR comment → report success → keep lock and PR open for review/merge. Lock will auto-clean when PR is merged or closed.

### Phase 4C: Apply First, Review Later

The user already applied. The PR is open for review/documentation.

```
PR #<N> is open for <project>. Apply completed — PR is open for review.

  1) Wait for review  — exit, lock will auto-clean when PR is merged/closed
  2) Unlock now       — release lock manually (PR stays open)
  3) Show apply result
```

**Wait for review**: exit. Lock stays. Housekeeping auto-cleans when the PR is merged or closed.

**Unlock now**: release lock immediately. PR remains open for review.

## Re-plan Change Detection

When re-planning (in Phase 4B), compare the new plan output against the previous plan and clearly inform the user:

- **If no changes**: `"Re-plan complete — no changes from previous plan."`
- **If changes detected**: summarize what changed:
  ```
  Re-plan complete — changes detected vs previous plan:
    + aws_s3_bucket.new_bucket (new resource)
    ~ aws_iam_role.app_role (modified)
    - aws_security_group.old_sg (no longer in plan)
  ```
  Post the updated plan as a new PR comment (do not edit the old one — keep the history).

## Rules

- `AWS_PROFILE=<terraform.profile>` from config for all terraform commands.
- DynamoDB commands use `--table-name`, `--region`, and `--profile` from `dynamodb` config.
- Never modify `backend.tf` or `provider.tf`.
- Wrap plan/apply output in ```hcl fences when posting to PRs.
- Ensure `tfplan` is in `.gitignore`.
- Process multiple selected projects sequentially with summary at end.
- All user interaction is menu-driven — no free-text interpretation.
