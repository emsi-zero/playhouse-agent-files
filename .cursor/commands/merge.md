# Merge Task Branch

Squash-merge the current task branch into its base branch following the checkpoint commit strategy.

## Human approval (MANDATORY)

This command is for when the **user explicitly runs it or clearly asked** to merge into the base branch. The agent must **not** perform the merge/push steps below on its own initiative—only after the user has approved merging in this conversation (or is executing this command themselves).

If approval is unclear, stop and ask: "Should I merge `<branch>` into `<base>` and push?"

When `task.integration_branch` is set (e.g. `multi-tenancy/phase-0`), the merge **target** for finishing a phase task is usually that integration branch — **not** `main` — until the phase completes. Merging or pushing **`multi-tenancy/phase-*`** still requires the **same explicit user approval** as merging to `main` (see `rules/task-context.mdc`).

## Prerequisites

Before running this command:
1. All checklist items should be completed
2. All changes should be committed (no uncommitted work)
3. Code review should be done (`/review` command)
4. User has **approved** merging to the target base branch

## Step 0: Load Task Context

Read the task manifest to understand what needs to be merged:

```bash
cat /home/emad/Projects/Pilche/.cursor/current-task.yaml
```

Extract:
- `task.id` — for commit message reference
- `task.type` — for conventional commit prefix
- `task.branch` — the branch pattern
- `task.description` — for commit message
- `repos.<name>.needed` — which repos have changes
- `repos.<name>.base_branch` — target branch for each repo (main, dev)
- `repos.<name>.task_branch` — source branch to merge

## Step 1: Verify Pre-Merge Conditions

For each repo where `needed: true`:

```bash
PILCHE=/home/emad/Projects/Pilche
REPO="<repo-name>"
TASK_BRANCH="<repos.$repo.task_branch>"
BASE_BRANCH="<repos.$repo.base_branch>"

cd "$PILCHE/$REPO"

# 1. Verify on task branch
current=$(git branch --show-current)
if [ "$current" != "$TASK_BRANCH" ]; then
  echo "⛔ Not on task branch. Current: $current, Expected: $TASK_BRANCH"
  exit 1
fi

# 2. Check for uncommitted changes
if [ -n "$(git status --porcelain)" ]; then
  echo "⛔ Uncommitted changes exist. Commit or stash first."
  exit 1
fi

# 3. Fetch latest base branch
git fetch origin $BASE_BRANCH

# 4. Check if branch is up to date with base
behind=$(git rev-list --count HEAD..origin/$BASE_BRANCH)
if [ "$behind" -gt 0 ]; then
  echo "⚠️  Branch is $behind commits behind origin/$BASE_BRANCH"
  echo "   Recommend: git rebase origin/$BASE_BRANCH"
fi
```

## Step 2: Generate Squash Commit Message

Based on task type, generate a conventional commit message:

| Task Type | Commit Prefix |
|-----------|---------------|
| feature | `feat` |
| fix | `fix` |
| hotfix | `fix` |
| refactor | `refactor` |
| chore | `chore` |
| docs | `docs` |
| test | `test` |

**Format:**
```
<type>(<scope>): <description>

<detailed description from task>

Task-ID: <task.id>
```

**Example:**
```
refactor(entry-flow): improve child selection stage UI

- Compact customer summary with edit icon
- Child cards with age and special conditions display
- Add Child as dashed card
- Simplified actions row (Back + Continue only)

Task-ID: 2026-04-13-entry-flow-child-selection-ui
```

## Step 3: Rebase and Squash (Interactive)

For each repo with changes:

```bash
PILCHE=/home/emad/Projects/Pilche
REPO="<repo-name>"
BASE_BRANCH="<repos.$repo.base_branch>"
TASK_BRANCH="<repos.$repo.task_branch>"

cd "$PILCHE/$REPO"

# Rebase onto latest base branch
git fetch origin $BASE_BRANCH
git rebase origin/$BASE_BRANCH

# If conflicts, resolve them and continue:
# git add <resolved-files>
# git rebase --continue

# Interactive rebase to squash all commits into one
COMMIT_COUNT=$(git rev-list --count origin/$BASE_BRANCH..HEAD)
if [ "$COMMIT_COUNT" -gt 1 ]; then
  echo "Squashing $COMMIT_COUNT commits..."
  git rebase -i origin/$BASE_BRANCH
  # In editor: change all 'pick' to 'squash' except the first
  # Then edit the final commit message
fi
```

**Alternative: Soft reset and recommit (simpler):**

```bash
# Get the merge base
MERGE_BASE=$(git merge-base origin/$BASE_BRANCH HEAD)

# Soft reset to merge base (keeps changes staged)
git reset --soft $MERGE_BASE

# Create single squash commit with proper message
git commit -m "<generated commit message>"
```

## Step 4: Merge into Base Branch

**Option A: Fast-forward merge (preferred for clean history):**

```bash
cd "$PILCHE/$REPO"

# Switch to base branch
git checkout $BASE_BRANCH
git pull origin $BASE_BRANCH

# Fast-forward merge (after squash, should be linear)
git merge --ff-only $TASK_BRANCH

# Push
git push origin $BASE_BRANCH
```

**Option B: Squash merge (if not rebased):**

```bash
cd "$PILCHE/$REPO"

# Switch to base branch
git checkout $BASE_BRANCH
git pull origin $BASE_BRANCH

# Squash merge
git merge --squash $TASK_BRANCH
git commit -m "<generated commit message>"

# Push
git push origin $BASE_BRANCH
```

## Step 5: Clean Up Task Branch

After successful merge:

```bash
cd "$PILCHE/$REPO"

# Delete local task branch
git branch -d $TASK_BRANCH

# Delete remote task branch
git push origin --delete $TASK_BRANCH
```

## Step 6: Update Task Manifest

After all repos are merged:

1. Set `task.phase: done`
2. Archive the manifest:

```bash
TASK_ID="<task.id>"
mv /home/emad/Projects/Pilche/.cursor/current-task.yaml \
   /home/emad/Projects/Pilche/.cursor/tasks/$TASK_ID.yaml
```

## Merge Order (Multi-Repo)

When task spans multiple repos, merge in dependency order:

1. **playhouse-server** (backend API first)
2. **playhouse-control-plane** (backend services)
3. **playhouse-web** (frontend depends on API)
4. **playhouse-cicd** (deployment config)
5. **playhouse-docs** (documentation last)

## Output Format

After completing the merge:

```markdown
## Merge Complete

### Task
- **ID**: <task.id>
- **Description**: <task.description>

### Merged Repos

| Repo | Base Branch | Commit |
|------|-------------|--------|
| playhouse-web | main | `abc1234` |
| playhouse-server | main | `def5678` |

### Commit Message
```
<type>(<scope>): <description>
```

### Cleanup
- [x] Task branches deleted (local)
- [x] Task branches deleted (remote)
- [x] Manifest archived to `.cursor/tasks/<task.id>.yaml`

### Next Steps
- No active task. Ready for next task.
```

## Error Handling

### Merge Conflicts

If conflicts occur during rebase:

```bash
# Show conflicting files
git status

# After resolving conflicts:
git add <resolved-files>
git rebase --continue

# Or abort if needed:
git rebase --abort
```

### Protected Branch Push Rejected

If push to base branch fails:

1. Verify you have push permissions
2. Check if branch protection rules require PR
3. If PR required, create PR instead of direct merge:

```bash
gh pr create --base $BASE_BRANCH --head $TASK_BRANCH \
  --title "<type>(<scope>): <description>" \
  --body "<detailed description>"
```

### Uncommitted Changes

If uncommitted changes exist:

```bash
# Option 1: Commit as WIP
git add -A
git commit -m "wip(<task.id>): pre-merge checkpoint"

# Option 2: Stash temporarily
git stash push -m "pre-merge stash"
# After merge:
git stash pop
```

## Quick Reference

```bash
# Full squash-merge workflow for single repo:
PILCHE=/home/emad/Projects/Pilche
REPO="playhouse-web"
BASE="main"
BRANCH="feature/my-feature"

cd "$PILCHE/$REPO"
git fetch origin $BASE
git rebase origin/$BASE
MERGE_BASE=$(git merge-base origin/$BASE HEAD)
git reset --soft $MERGE_BASE
git commit -m "feat(scope): description"
git checkout $BASE
git pull origin $BASE
git merge --ff-only $BRANCH
git push origin $BASE
git branch -d $BRANCH
git push origin --delete $BRANCH
```
