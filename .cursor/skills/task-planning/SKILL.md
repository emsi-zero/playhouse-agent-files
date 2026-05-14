---
name: task-planning
description: Analyze and plan a new task across the multi-repo workspace. Use when user describes a new feature, fix, or task that needs scoping. Works in both Plan and Agent mode.
---

# Task Planning (Multi-Repo)

This skill analyzes the codebase to scope a task and produce a plan.

## When to Use

- User describes a new task/feature/fix
- Before creating task manifest
- When unsure which repos are affected
- When breaking down a large task into checklist items

## Planning Process

### Step 1: Understand the Task

Clarify with user if needed:
- What is the goal?
- What type of task? (feature, fix, refactor, etc.)
- Any constraints or requirements?

### Step 2: Analyze Codebase

Use subagent or direct search to find relevant code:

#### Option A: Use Explore Subagent (Recommended for Complex Tasks)

```
Task(
  subagent_type: "explore",
  description: "Scope <task-summary>",
  prompt: """
    Analyze where <task-description> would be implemented in the PlayHouse codebase.
    
    Search for:
    1. Related existing code (keywords: <relevant-terms>)
    2. Similar features for patterns to follow
    3. Test files that might need updates
    
    For each repo, identify:
    - playhouse-server: relevant files in apps/
    - playhouse-web: relevant files in src/features/ or src/components/
    - playhouse-control-plane: relevant files in cmd/ or web/
    
    Return structured analysis:
    - Repos needed: [list with reasoning]
    - Files to modify per repo: [specific paths]
    - Suggested checklist: [ordered work items]
    - Dependencies or risks: [notes]
  """,
  readonly: true
)
```

#### Option B: Direct Search (For Simpler Tasks)

```bash
# Search for related code
rg "<keyword>" /home/emad/Projects/Pilche/playhouse-* --type-add 'code:*.{js,jsx,ts,tsx,py,go}' -t code -l

# Find related files
find /home/emad/Projects/Pilche -name "*<term>*" -type f
```

### Step 3: Identify Affected Repos

Based on analysis, determine which repos need changes:

| If task involves... | Repos needed |
|---------------------|--------------|
| API/backend changes | `playhouse-server` |
| Customer UI changes | `playhouse-web` |
| Admin UI changes | `playhouse-control-plane` (web/) |
| Go microservices | `playhouse-control-plane` (cmd/) |
| Deployment/infra | `playhouse-cicd` |
| Documentation | `playhouse-docs` |

### Step 4: Create Checklist

Break down the task into ordered work items:

1. **Backend first** (if applicable):
   - Models/migrations
   - API endpoints
   - Business logic
   - Tests

2. **Frontend second** (if applicable):
   - API client/hooks
   - Components
   - Pages/routes
   - Tests

3. **Infrastructure** (if applicable):
   - Docker changes
   - Environment variables
   - Deployment configs

4. **Documentation** (if applicable):
   - API docs
   - User guides
   - README updates

### Step 5: Output Plan

#### In Plan Mode

Output the plan as YAML for the user to review:

```yaml
# Proposed Task Plan
# Review and adjust, then switch to Agent mode and say "create task"

task:
  type: feature
  branch: "feature/description"
  description: "Human-readable description"

repos:
  playhouse-server:
    needed: true
    reason: "API endpoints for X"
  playhouse-web:
    needed: true
    reason: "UI components for X"
  playhouse-control-plane:
    needed: false
  playhouse-cicd:
    needed: false
  playhouse-docs:
    needed: false

checklist:
  - id: server-models
    repo: playhouse-server
    description: "Add X model with fields..."
  - id: server-api
    repo: playhouse-server
    description: "Add X endpoint at /api/..."
  - id: server-tests
    repo: playhouse-server
    description: "Add tests for X API"
  - id: web-hook
    repo: playhouse-web
    description: "Add useX hook"
  - id: web-component
    repo: playhouse-web
    description: "Add X component"
  - id: web-integration
    repo: playhouse-web
    description: "Integrate X into page Y"

notes: |
  - Consideration 1
  - Consideration 2

---
To proceed: Switch to Agent mode and say "create task" or "create task from plan"
```

#### In Agent Mode

If already in Agent mode, proceed to create the manifest using the task-management skill.

## Parallel Subagent Analysis

For large tasks spanning multiple repos, analyze in parallel:

```
# Spawn multiple explore subagents simultaneously
Task(subagent_type: "explore", prompt: "Analyze playhouse-server for <task>", ...)
Task(subagent_type: "explore", prompt: "Analyze playhouse-web for <task>", ...)
Task(subagent_type: "explore", prompt: "Analyze playhouse-control-plane for <task>", ...)
```

Combine results to build complete picture.

## Refining a Plan

If user wants to adjust the plan:

1. Read existing plan/manifest
2. Discuss changes with user
3. Update checklist items, repos, or notes
4. In Agent mode: Update manifest file
5. In Plan mode: Output updated YAML for user

## Integration with Task Management

After planning is complete and user approves:

1. **In Plan mode**: Tell user to switch to Agent mode
2. **In Agent mode**: 
   - Read the task-management skill
   - Create the manifest file
   - Create branches in all needed repos
   - Update manifest phase to `implementing`

## Quick Reference

**New task flow:**
1. User describes task
2. Analyze codebase (subagent or direct search)
3. Identify repos and create checklist
4. Output plan (Plan mode) or create manifest (Agent mode)
5. Create branches
6. Start implementing

**Keywords that trigger this skill:**
- "plan", "scope", "analyze"
- "new feature", "new task", "I want to..."
- "what repos", "which files", "where should"
