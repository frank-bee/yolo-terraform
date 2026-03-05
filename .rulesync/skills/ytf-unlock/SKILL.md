---
name: ytf-unlock
description: "Unlock a Yolo Terraform project lock. Use when: (1) user says /ytf:unlock, (2) user wants to release a terraform project lock, (3) user asks to unlock a project."
---

# Yolo Terraform — Release Project Lock

Read `../ytf/references/yolotf-reference.md` for config and DynamoDB details.

## Argument

Optional `[project]` name. If omitted, show current locks for selection.

## Steps

### 1. Read Config

Read `.yolotf.yaml` (repo root, fallback `~/.yolotf.yaml`). If missing, tell user to run `/ytf:setup`.

Construct repo slug and LockID hash per reference doc.

### 2. Identify Target Lock

**If project specified**: look up that project's lock directly:

```bash
aws dynamodb get-item --table-name <tableName> \
  --key '{"LockID":{"S":"yolotf-lock-<hash>-<project>"}}' \
  --region <region> --profile <profile>
```

**If no argument**: scan all locks and filter to current user's locks:

```bash
aws dynamodb scan --table-name <tableName> \
  --region <region> --profile <profile>
```

- If user holds multiple locks, present list and let them pick (or "all").
- If user holds no locks, report "You have no active locks." and exit.

### 3. Release Lock

For each selected lock:

| Condition | Action |
|-----------|--------|
| Held by you | Conditional release (safe delete with `Holder = :h` condition) |
| Held by someone else | Show holder, ask to confirm **force-unlock** (unconditional delete) |
| No lock exists | Report `"<project> is not locked."` |

If the lock has **PR > 0**, check PR state via `gh pr view <N> --json state -q .state` and warn:
- `"⚠ PR #<N> is still OPEN — the lock will be released but the PR remains open."`

### 4. Confirm

Report result: `"Released lock for <project>."` or `"Force-unlocked <project> (was held by <holder>)."`
