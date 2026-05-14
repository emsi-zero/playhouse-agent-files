---
name: task-management
description: Manage multi-repo task lifecycle with persistent state. Use when creating a task manifest, updating task progress, finishing a task, or merging. Also use when user says "create task", "save plan", "update task", "finish task", or "merge task".
---

# Task Management (Multi-Repo)

This skill manages the task lifecycle using a persistent manifest file.

**Manifest location:** `/home/emad/Projects/Pilche/.cursor/current-task.yaml`
**Archive location:** `/home/emad/Projects/Pilche/.cursor/tasks/`

## Creating a Task Manifest

When starting a new task (after planning or when user describes work):

### Step 1: Gather Information

Required:
- Task description (from user)
- Task type: `feature`, `fix`, `hotfix`, `refactor`, `chore`, `docs`, `test`
- Affected repos (from analysis or user input)
- Initial checklist items

### Step 2: Generate Identifiers

```bash
# Task ID: date + slug
TASK_ID="$(date +%Y-%m-%d)-short-description"

# Branch name: type/description
BRANCH="feature/short-description"
```

### Step 3: Write Manifest

Create `/home/emad/Projects/Pilche/.cursor/current-task.yaml`:

```yaml
task:
  id: "YYYY-MM-DD-description"
  type: feature
  branch: "feature/description"
  description: "Human-readable task description"
  created: "YYYY-MM-DDTHH:MM:SSZ"
  phase: planning
  # Multi-tenancy / long phases: merge task branches here until the phase is done; then merge this → main.
  # Omit or null when merging tasks straight to main.
  integration_branch: "multi-tenancy/phase-0"

repos:
  playhouse-server:
    needed: true
    base_branch: main           # Ultimate merge target when phase completes (usually main)
    task_branch: null           # Actual branch name once created
    branch_created: false
    has_changes: false
    committed: false
    pushed: false
  playhouse-web:
    needed: true
    base_branch: main
    task_branch: null
    branch_created: false
    has_changes: false
    committed: false
    pushed: false
  playhouse-control-plane:
    needed: false
    base_branch: dev            # Note: control-plane uses dev as base
    task_branch: null
    branch_created: false
    has_changes: false
    committed: false
    pushed: false
  playhouse-cicd:
    needed: false
    base_branch: main
    task_branch: null
    branch_created: false
    has_changes: false
    committed: false
    pushed: false
  playhouse-docs:
    needed: false
    base_branch: main
    task_branch: null
    branch_created: false
    has_changes: false
    committed: false
    pushed: false

checklist:
  - id: item-1
    repo: playhouse-server
    description: "First work item"
    status: pending
  - id: item-2
    repo: playhouse-web
    description: "Second work item"
    status: pending

notes: |
  Any additional context, decisions, or constraints.
```

### Step 4: Announce

```
Created task manifest:
- ID: <task.id>
- Branch: <task.branch>
- Repos: <list of needed repos>
- Checklist: <count> items

Ready to create branches? (say "create branches" or "start implementing")
```

## Creating Branches

After manifest exists, create branches in all needed repos:

```bash
PILCHE=/home/emad/Projects/Pilche
TASK_BRANCH="<task.branch from manifest>"

# For each repo where needed=true:
# Use the repo's base_branch (main, dev, etc.) as the starting point
for repo in <repos where needed=true>; do
  cd "$PILCHE/$repo"
  BASE_BRANCH="<repos.$repo.base_branch>"  # e.g., main or dev
  git checkout "$BASE_BRANCH" && git pull origin "$BASE_BRANCH"
  git checkout -b "$TASK_BRANCH"
  echo "Created branch $TASK_BRANCH from $BASE_BRANCH in $repo"
done
```

After creating branches, update manifest for EACH repo:
1. Set `repos.<name>.task_branch: "<task.branch>"`
2. Set `repos.<name>.branch_created: true`
3. Set `task.phase: implementing`

**Important:** The `task_branch` field records the actual branch name used in that repo. This allows verification that you're on the correct branch before making changes.

### Multi-tenancy / phase integration (`integration_branch`)

When `task.integration_branch` is set (e.g. `multi-tenancy/phase-0`):

| Step | Action |
|------|--------|
| Start task | Create `task.branch` **from** `integration_branch` (after `git pull`), not only from `main`, so the task includes prior phase work. |
| Finish task (merge) | **Only after explicit user authorization:** merge `task.branch` → `integration_branch` in each needed repo (and push if approved). **Do not** merge to `main` until the **phase** is complete. |
| Phase complete | After explicit user approval, merge `integration_branch` → `main` (then optionally tag, delete old task branches, create `multi-tenancy/phase-1` from `main` for the next phase). |

**Push:** `git push origin <integration_branch>` requires **explicit user approval** in the conversation — same as merging **into** that branch. Merging `integration_branch` → `main` also requires **explicit** approval.

## Updating Task Progress

### After Completing a Checklist Item (with Checkpoint Commit)

1. **Verify on task branch** (not protected)
2. **Stage and commit** with checkpoint message:
   ```bash
   git add -A
   git commit -m "checkpoint(<item.id>): <item.description>"
   ```
3. **Update manifest:**
   - Set `checklist[id=<item.id>].status: done`
   - Set `repos.<repo>.has_changes: true`
   - Set `repos.<repo>.committed: true`
4. Save manifest

### After Making Changes in a Repo (No Commit Yet)

1. Update `repos.<name>.has_changes: true`
2. Save manifest

### Before Switching to Another Repo

1. If current repo has uncommitted changes:
   ```bash
   git add -A
   git commit -m "wip(<task.id>): context switch to <target-repo>"
   ```
2. Update `repos.<name>.committed: true`
3. Save manifest

### At End of Session

For each repo with uncommitted changes:
```bash
git add -A
git commit -m "wip(<task.id>): session checkpoint"
```

Update manifest for each committed repo.

### After Pushing

1. Update `repos.<name>.pushed: true`
2. Save manifest

## Syncing with Git State

Run this to verify manifest matches reality:

```bash
PILCHE=/home/emad/Projects/Pilche

for repo in playhouse-server playhouse-control-plane playhouse-web playhouse-cicd playhouse-docs playhouse-integration-driver; do
  echo "=== $repo ==="
  cd "$PILCHE/$repo"
  
  # Get expected branch from manifest (repos.<repo>.task_branch)
  EXPECTED_BRANCH="<repos.$repo.task_branch>"  # null if not created yet
  
  current_branch=$(git branch --show-current)
  echo "current branch: $current_branch"
  echo "expected branch: $EXPECTED_BRANCH"
  
  # Check if on protected branch
  if [ "$current_branch" = "main" ] || [ "$current_branch" = "master" ] || [ "$current_branch" = "dev" ]; then
    echo "⛔ WARNING: On protected branch!"
  elif [ "$EXPECTED_BRANCH" != "null" ] && [ "$current_branch" != "$EXPECTED_BRANCH" ]; then
    echo "⚠️  Branch mismatch!"
  else
    echo "✓ Branch OK"
  fi
  
  changes=$(git status --porcelain | wc -l)
  echo "uncommitted changes: $changes"
  
  # Check if ahead of remote
  ahead=$(git log @{u}..HEAD --oneline 2>/dev/null | wc -l || echo "no upstream")
  echo "unpushed commits: $ahead"
  echo ""
done
```

Update manifest if diverged from reality. Specifically:
- If `task_branch` in manifest doesn't match actual current branch, update it
- If `branch_created: false` but branch exists, set it to `true`
- If changes exist but `has_changes: false`, update it

## Finishing a Task

When all checklist items are done:

### Step 1: Verify Completion

```bash
# Check all repos are committed and pushed
cat /home/emad/Projects/Pilche/.cursor/current-task.yaml | grep -A5 "repos:"
```

Ensure for each needed repo:
- `has_changes: true` → `committed: true`
- `committed: true` → `pushed: true`

### Step 2: Clean Up Commit History

Before creating a PR, clean up checkpoint/WIP commits for a clean history.

For each repo with changes:

```bash
PILCHE=/home/emad/Projects/Pilche
REPO="<repo-name>"
BASE_BRANCH="<repos.$repo.base_branch>"
TASK_BRANCH="<repos.$repo.task_branch>"

cd "$PILCHE/$REPO"

# Fetch latest base branch
git fetch origin $BASE_BRANCH

# Count commits to clean up
COMMIT_COUNT=$(git log origin/$BASE_BRANCH..$TASK_BRANCH --oneline | wc -l)
echo "Found $COMMIT_COUNT commits to review"

# Interactive rebase
git rebase -i origin/$BASE_BRANCH
```

**In the rebase editor:**

| Action | When to Use |
|--------|-------------|
| `pick` | Keep commit as-is |
| `reword` (r) | Change checkpoint message to conventional format |
| `squash` (s) | Combine with previous commit |
| `fixup` (f) | Combine with previous, discard message |

**Recommended cleanup strategy:**

1. **Squash all WIP commits** (`wip(...)`) into related checkpoints
2. **Reword checkpoints** to conventional commit format:
   - `checkpoint(child-cards): ...` → `feat(entry-flow): add child cards with metadata`
3. **Keep logical separation** if changes are independent
4. **Squash everything** if it's a single logical change

After rebase:
```bash
# Force push (safe on task branch)
git push --force-with-lease origin $TASK_BRANCH
```

### Step 3: Update Phase

Set `task.phase: reviewing`

### Step 4: Generate Summary

Output a summary for PR/review:
```
## Task Complete: <description>

### Changes by Repo

**playhouse-server:**
- <list of changes>

**playhouse-web:**
- <list of changes>

### Checklist
- [x] All items completed

### Commit History (cleaned)
- <final commit messages after rebase>

### Ready for Review
Branch `<branch>` is ready to merge into <base_branch>.
```

## Merge and push to protected branches — human approval required (MANDATORY)

**The agent MUST NOT** merge into `main`, `dev`, `master`, or any other **production/integration policy** branch without matching approval below.

**Includes:** `main`, `dev`, `master`, and **all** **`multi-tenancy/phase-*`** integration branches for any of: merging **into** them (e.g. task → phase), **`git push`** to them, or merging phase → `main`.

**The agent MUST NOT** merge a task branch into `main`/`dev`/`master` **or into `multi-tenancy/phase-*`**, and MUST NOT `git push` to those branches on the remote, **unless the user has given explicit approval in this conversation** for that merge or push.

**Phase integration:** Merging `multi-tenancy/phase-N` → `main` requires the **same explicit approval** as any merge to `main`. Merging `feature/...` → `multi-tenancy/phase-N` is the **intended** landing place during a phase, but the agent still **must not** do it without **explicit** user authorization (offer commands/PR otherwise).

Examples of **explicit merge approval** (wording may vary; intent must be clear):
- "Merge to main" / "Merge the branch into main"
- "Push the merge" / "Go ahead and merge"
- "Approved — merge and push"

**Not sufficient** to merge on your own (treat as "work is done on the task branch" only):
- "Complete the task" / "Finish the task" / "Don't stop until done" — means finish **implementation, tests, and commits** on the task branch, **not** merge to `main` unless merge was explicitly requested.
- Completing a checklist item named "merge" — stop and **ask** or wait for explicit merge instruction unless the user already said to merge.

**Allowed without merge-to-integration approval:**
- Commits on the **task branch** (checkpoints, final commits)
- `git push origin <task-branch>` when the user asked to push the feature branch
- Rebase/squash **on the task branch** when preparing for review
- Output: exact commands the user can run to merge **into `integration_branch`** (or `main`), or PR creation steps

**Default end state:** Task branch is up to date and tests pass. If `integration_branch` is set, say the branch is **ready to merge into** that integration line **when the user authorizes**; do **not** perform that merge without explicit approval. If no integration branch, same for `base_branch` / `main`.

## Merging a Task

**Only after explicit user approval to merge** (see section above). When the user says "merge" or clearly approves merging after review:

### Step 1: Merge in Dependency Order

1. `playhouse-server` (backend first)
2. `playhouse-control-plane` (backend services)
3. `playhouse-web` (frontend)
4. `playhouse-cicd` (infrastructure last)
5. `playhouse-docs` (documentation)

For each needed repo:
```bash
cd "$PILCHE/$repo"
git fetch origin
git rebase origin/main
git checkout main
git pull origin main
git merge --no-ff "$BRANCH"
git push origin main
```

### Step 2: Update Manifest

Set `task.phase: done`

### Step 3: Archive Manifest

```bash
mv /home/emad/Projects/Pilche/.cursor/current-task.yaml \
   /home/emad/Projects/Pilche/.cursor/tasks/<task.id>.yaml
```

### Step 4: Announce

```
Task merged and archived:
- Merged to main in: <repo list>
- Archived to: .cursor/tasks/<task.id>.yaml

No active task. Ready for next task.
```

## Abandoning a Task

If user wants to abandon current task:

1. Confirm with user
2. Optionally archive with `phase: abandoned`
3. Delete branches in each repo (if requested)
4. Remove or archive manifest

## Manifest Schema Reference

```yaml
task:
  id: string           # Unique: YYYY-MM-DD-slug
  type: string         # feature|fix|hotfix|refactor|chore|docs|test
  branch: string       # Default branch name pattern (same across repos)
  description: string  # Human-readable description
  created: string      # ISO 8601 timestamp
  phase: string        # planning|implementing|reviewing|merging|done|abandoned
  integration_branch: string|null  # e.g. multi-tenancy/phase-0; intended landing branch for the phase — merges into it require explicit user approval

commit_policy:
  auto_checkpoint: boolean     # Auto-commit after checklist items (default: true)
  checkpoint_on_switch: boolean # Commit before switching repos (default: true)
  checkpoint_on_session_end: boolean # Commit at end of session (default: true)
  squash_before_pr: boolean    # Clean up history before PR (default: true)

repos:
  <repo-name>:         # playhouse-server, playhouse-web, etc.
    needed: boolean    # Is this repo part of the task?
    base_branch: string    # Branch to merge into (main, dev, etc.)
    task_branch: string|null  # Actual branch name once created (null before)
    branch_created: boolean
    has_changes: boolean
    committed: boolean
    pushed: boolean

checklist:
  - id: string         # Unique within task
    repo: string       # Which repo this item belongs to
    description: string
    status: string     # pending|in_progress|done|blocked

notes: string          # Free-form notes, decisions, constraints
```

### Branch Fields Explained

| Field | Purpose |
|-------|---------|
| `task.branch` | The intended branch name pattern (e.g., `feature/user-auth`) |
| `repos.<name>.base_branch` | The protected branch to branch from and merge into (`main`, `dev`) |
| `repos.<name>.task_branch` | The actual branch name created in this repo (null until created) |

**Why track per-repo branches?**
1. Different repos may use different base branches (`main` vs `dev`)
2. Allows verification that you're on the correct branch before changes
3. Supports edge cases where branch names might differ slightly per repo
4. Enables accurate sync checking between manifest and git state
