---
name: git-branching
description: Manage git branches for tasks with proper naming conventions in multi-repo workspace. Use when creating branches, switching branches, or merging. Integrates with task manifest for repo tracking.
---

# Git Branching Workflow (Multi-Repo)

This workspace contains multiple repositories. Branch operations must be repo-aware and integrated with the task manifest.

**Task manifest:** `/home/emad/Projects/Pilche/.cursor/current-task.yaml`

## Before Any Branch Operation

Always check the task manifest first:

```bash
cat /home/emad/Projects/Pilche/.cursor/current-task.yaml 2>/dev/null || echo "NO_ACTIVE_TASK"
```

- If manifest exists: Use `task.branch` for branch name, check `repos.<name>.needed` for which repos
- If no manifest: Create one first using task-management skill, OR ask user which repos are involved

## Creating Branches (From Task Manifest)

### Step 1: Read Task Manifest

Get branch name and needed repos from manifest:
- Branch name: `task.branch`
- Repos: all entries where `repos.<name>.needed: true`

### Step 2: Create Branch in Each Needed Repo

Use the SAME branch name across all affected repos:

```bash
# Single repo example
cd /home/emad/Projects/Pilche/playhouse-web
git checkout main && git pull origin main
git checkout -b feature/user-authentication

# Cross-repo example (web + server)
for repo in playhouse-web playhouse-server; do
  cd /home/emad/Projects/Pilche/$repo
  git checkout main && git pull origin main
  git checkout -b feature/user-authentication
  echo "Created branch in $repo"
done
```

### Step 3: Update Task Manifest

After creating branches, update manifest for each repo:
- Set `repos.<name>.branch_created: true`
- Set `task.phase: implementing` (if was `planning`)

### Step 4: Announce

Always announce which branches were created:
> "Created branch `feature/user-authentication` in `playhouse-web` and `playhouse-server`"
> "Updated task manifest: phase → implementing"

## Multi-tenancy program: phase integration branches

For **multi-tenancy / Phase 0+** work, use a long-lived integration line **per phase** instead of merging every task straight to `main`.

| Branch | Purpose |
|--------|---------|
| `multi-tenancy/phase-0` | Phase 0 (Foundation): integrates task branches until Phase 0 is done |
| `multi-tenancy/phase-1` | Phase 1 (Alpha): create from `main` after Phase 0 merged, or continue naming as you add phases |

**Rules:**

1. **Create the phase branch from `main`** when starting a phase (or refresh it from `main` if policy requires).
2. **Task / feature branches** carry day-to-day work. **Do not** merge a completed task into `multi-tenancy/phase-N` (or `git push` that branch) unless the user **explicitly authorizes** that merge in the conversation — same requirement as merging to `main`. Default: stop on the task branch and offer merge commands or a PR. Until the phase is finished, **do not** merge phase work to `main` either.
3. **Branch new tasks from the phase branch** after pulling latest: `git checkout multi-tenancy/phase-0 && git pull && git checkout -b feature/my-task`.
4. **When the phase is complete and reviewed:** merge `multi-tenancy/phase-N` → `main` only with **explicit user approval** (see task-management skill).
5. **Next phase:** create `multi-tenancy/phase-(N+1)` from updated `main` (after prior phase merged).

Treat `multi-tenancy/phase-*` as a **protected integration line**: no direct commits; integrate via merge commits from task branches **only after explicit merge approval**. Pushing `origin multi-tenancy/phase-N` also requires explicit approval.

## Branch Naming Convention

| Type | Prefix | Use For | Example |
|------|--------|---------|---------|
| Feature | `feature/` | New functionality | `feature/user-authentication` |
| Integration | `multi-tenancy/phase-N` | Multi-tenancy phase integration (merge tasks here until phase done) | `multi-tenancy/phase-0` |
| Fix | `fix/` | Bug fixes | `fix/login-redirect-loop` |
| Hotfix | `hotfix/` | Urgent production fixes | `hotfix/payment-crash` |
| Refactor | `refactor/` | Code improvements without behavior change | `refactor/api-client-cleanup` |
| Chore | `chore/` | Maintenance, deps, configs | `chore/update-dependencies` |
| Docs | `docs/` | Documentation only | `docs/api-readme` |
| Test | `test/` | Adding/fixing tests | `test/auth-coverage` |

### Naming Rules

- Use lowercase letters, numbers, and hyphens only
- Keep descriptions short (2-4 words max)
- Be specific: `fix/cart-total` not `fix/bug`
- Use SAME name across repos for related changes

## Status Check Across All Repos

Before any commit or merge, check status across all repos:

```bash
echo "=== Git Status Across Workspace ==="
for d in /home/emad/Projects/Pilche/playhouse-*; do
  name=$(basename "$d")
  branch=$(git -C "$d" branch --show-current 2>/dev/null)
  status=$(git -C "$d" status --porcelain 2>/dev/null)
  if [ -n "$status" ] || [ "$branch" != "main" ]; then
    echo ""
    echo "[$name] on branch: $branch"
    [ -n "$status" ] && echo "$status" | head -5
  fi
done
```

## Merging (Rebase and Merge)

### Human approval required (MANDATORY)

Do **not** `git checkout main` (or `dev`) / `git merge` / `git push origin main` unless the user **explicitly approved** that merge or push in this conversation. The same applies to **`multi-tenancy/phase-*`**: do **not** merge a task branch into the phase branch or `git push origin multi-tenancy/phase-N` without explicit authorization. See the **task-management** skill: "Merge and push to protected branches — human approval required".

If the user has not approved a merge, stop after work on the **task branch** and offer commands or a PR instead.

### When merge is explicitly approved

When the user says "merge" (or clearly approves), perform a rebase-merge workflow.

### Single Repo Merge

```bash
cd /home/emad/Projects/Pilche/playhouse-web

# Ensure working tree is clean
git status

# Fetch latest and rebase onto main
git fetch origin
git rebase origin/main

# If conflicts occur, resolve them, then:
# git add <resolved-files>
# git rebase --continue

# Switch to main and merge
git checkout main
git pull origin main
git merge --no-ff <branch-name>

# Push
git push origin main
```

### Cross-Repo Merge (Dependency Order)

When merging across repos, follow this order:
1. Backend first: `playhouse-server`, `playhouse-control-plane`
2. Frontend second: `playhouse-web`
3. Infrastructure last: `playhouse-cicd`

**During a multi-tenancy phase:** the **intended** merge target for a finished task is `multi-tenancy/phase-N` (not `main`) until the phase completes — but performing that merge (or push) still requires **explicit user approval**, same as `main`. Only merge the phase branch to `main` when the phase is complete and the user explicitly approves.

```bash
# Example: merging feature/user-auth across web + server
BRANCH="feature/user-auth"

# 1. Merge server first (backend)
cd /home/emad/Projects/Pilche/playhouse-server
git fetch origin && git rebase origin/main
git checkout main && git pull origin main
git merge --no-ff $BRANCH
git push origin main
echo "Merged $BRANCH into main in playhouse-server"

# 2. Merge web second (frontend)
cd /home/emad/Projects/Pilche/playhouse-web
git fetch origin && git rebase origin/main
git checkout main && git pull origin main
git merge --no-ff $BRANCH
git push origin main
echo "Merged $BRANCH into main in playhouse-web"
```

**Example: merge task branch into phase integration (multi-tenancy):**

```bash
INTEGRATION="multi-tenancy/phase-0"
TASK_BRANCH="feature/multi-tenancy-core"

for repo in playhouse-server playhouse-cicd; do
  cd /home/emad/Projects/Pilche/$repo
  git fetch origin
  git checkout "$INTEGRATION"
  git pull origin "$INTEGRATION" 2>/dev/null || true
  # Only when the user explicitly authorized merging into the phase branch:
  git merge --no-ff "$TASK_BRANCH" -m "Merge $TASK_BRANCH into $INTEGRATION"
  # git push origin "$INTEGRATION"  # only after explicit approval to push
done
```

### Merge to dev

If merging to dev instead of main:

```bash
git fetch origin
git rebase origin/dev
git checkout dev
git pull origin dev
git merge --no-ff <branch-name>
git push origin dev
```

## Workflow Checklist (Task-Manifest Integrated)

### Starting a task
- [ ] Check if task manifest exists
- [ ] If no manifest: create one first (use task-management skill)
- [ ] Read branch name and repos from manifest
- [ ] Create branch in EACH repo where `needed: true`
- [ ] Update manifest: `branch_created: true` for each repo
- [ ] Update manifest: `phase: implementing`
- [ ] Announce: "Created branch `<name>` in `<repo1>`, `<repo2>`"

### Before Committing
- [ ] Read manifest to confirm which repos have changes
- [ ] Run status check across workspace
- [ ] Ensure you're in the correct repo directory
- [ ] Commit with clear message
- [ ] Update manifest: `committed: true` for the repo

### After Pushing
- [ ] Update manifest: `pushed: true` for the repo

### Merging (only after explicit user approval)
- [ ] **Confirm user explicitly approved** merge to target branch — including **`multi-tenancy/phase-*`** (not inferred from "finish task")
- [ ] Read manifest for branch name and repo list
- [ ] Run status check - ensure all changes are committed
- [ ] Rebase onto target branch in each repo
- [ ] Resolve any conflicts
- [ ] Merge in dependency order (backend → frontend → infra)
- [ ] Push to remote
- [ ] Update manifest: `phase: done`
- [ ] Archive manifest to `.cursor/tasks/<id>.yaml`
- [ ] Announce: "Merged `<branch>` into `<target>` in `<repo1>`, `<repo2>`"

## Quick Reference

**Check task manifest:**
```bash
cat /home/emad/Projects/Pilche/.cursor/current-task.yaml 2>/dev/null || echo "NO_ACTIVE_TASK"
```

**Check workspace status:**
```bash
for d in /home/emad/Projects/Pilche/playhouse-*; do echo "=== $(basename $d) ===" && git -C "$d" status -sb; done
```

**Create branches from manifest:**
```bash
PILCHE=/home/emad/Projects/Pilche
BRANCH="<read from task.branch in manifest>"

# For each repo where needed: true
for repo in playhouse-server playhouse-web; do
  cd "$PILCHE/$repo"
  git checkout main && git pull && git checkout -b "$BRANCH"
  echo "Created $BRANCH in $repo"
done
# Then update manifest: branch_created: true for each
```

**Sync manifest with git reality:**
```bash
PILCHE=/home/emad/Projects/Pilche
for repo in playhouse-server playhouse-control-plane playhouse-web playhouse-cicd playhouse-docs playhouse-integration-driver; do
  echo "=== $repo ==="
  branch=$(git -C "$PILCHE/$repo" branch --show-current 2>/dev/null)
  changes=$(git -C "$PILCHE/$repo" status --porcelain 2>/dev/null | wc -l)
  echo "branch: $branch, uncommitted: $changes"
done
```
