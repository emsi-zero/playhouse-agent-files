---
name: workspace-navigation
description: Navigate and understand the PlayHouse multi-repo workspace. Use when orienting to the codebase, finding files across repos, or checking workspace state.
---

# Workspace Navigation

This skill helps navigate the PlayHouse multi-repo workspace effectively.

## Quick Orientation

Run this to understand the current state of all repos:

```bash
echo "=== PlayHouse Workspace Status ==="
for d in /home/emad/Projects/Pilche/playhouse-*; do
  name=$(basename "$d")
  branch=$(git -C "$d" branch --show-current 2>/dev/null || echo "N/A")
  changes=$(git -C "$d" status --porcelain 2>/dev/null | wc -l)
  echo "$name: branch=$branch, uncommitted=$changes"
done
```

## Repository Purposes

| Repo | What Lives Here | Key Entry Points |
|------|-----------------|------------------|
| `playhouse-web` | Customer-facing React app | `src/App.js`, `src/features/` |
| `playhouse-server` | Django REST API | `apps/`, `manage.py`, `config/urls.py` |
| `playhouse-control-plane` | Admin UI + Go microservices | `cmd/`, `web/src/` |
| `playhouse-cicd` | Docker Compose, deployment configs | `docker-compose*.yml` |
| `playhouse-docs` | Documentation, backlogs, specs | `*.md` |
| `playhouse-integration-driver` | On-prem Go driver (WebSocket + hardware) | `cmd/`, `internal/`, `go.mod` |

## Finding Code by Feature

### Customer-Facing Features
- **UI Components**: `playhouse-web/src/features/` or `playhouse-web/src/components/`
- **API Endpoints**: `playhouse-server/apps/<app_name>/views.py`
- **Models/Data**: `playhouse-server/apps/<app_name>/models.py`

### Admin Features
- **Admin UI**: `playhouse-control-plane/web/src/`
- **Admin API**: `playhouse-control-plane/cmd/cp-api/`

### Infrastructure
- **Local Dev**: `playhouse-cicd/docker-compose.local.yml`
- **Staging**: `playhouse-cicd/docker-compose.self-hosted-staging.yml`

### On-prem driver (Go)
- **Driver service**: `playhouse-integration-driver/` (WebSocket client, registration, locker adapters)
- **API contract / server**: `playhouse-server/apps/integrations/` (registration, execute, health)

## Search Across Workspace

### Find a file by name
```bash
find /home/emad/Projects/Pilche -name "*.js" -path "*/src/*" | grep -i "component"
```

### Find text across all repos
```bash
rg "searchTerm" /home/emad/Projects/Pilche/playhouse-* --type-add 'code:*.{js,jsx,ts,tsx,py,go}' -t code
```

### Find API endpoints
```bash
rg "path\(|@api_view|router\." /home/emad/Projects/Pilche/playhouse-server/apps/
```

### Find React components
```bash
rg "export (default )?(function|const) \w+" /home/emad/Projects/Pilche/playhouse-web/src/ -g "*.js"
```

## Common Cross-Repo Patterns

### Feature spans web + server
1. Check API endpoint: `playhouse-server/apps/<feature>/views.py`
2. Check frontend hook: `playhouse-web/src/features/<feature>/hooks/`
3. Check frontend component: `playhouse-web/src/features/<feature>/components/`

### Deployment change
1. Modify compose file: `playhouse-cicd/docker-compose*.yml`
2. Update env vars: `playhouse-cicd/.env.example`
3. Document: `playhouse-docs/`

## Directory Structure Quick View

```bash
# See structure of a specific repo
tree -L 2 -d /home/emad/Projects/Pilche/playhouse-web/src/

# See all top-level directories across workspace
for d in /home/emad/Projects/Pilche/playhouse-*; do
  echo "=== $(basename $d) ==="
  ls -1 "$d" | head -10
done
```

## Working Directory Safety

Always verify cwd before running commands:

```bash
# Check current directory
pwd

# Navigate explicitly
cd /home/emad/Projects/Pilche/playhouse-web

# Or use -C flag with git
git -C /home/emad/Projects/Pilche/playhouse-server status
```
