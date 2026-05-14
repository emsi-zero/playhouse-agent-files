# Code Review

Review the current branch by running tests and analyzing diffs against main.

## Step 0: Identify Stack

First, detect which stack you're working with based on the current directory:

```bash
pwd
```

Then determine the test command:
- If `package.json` exists → **JavaScript/React** (npm test)
- If `manage.py` exists → **Python/Django** (pytest)
- If `go.mod` exists → **Go** (go test)
- Otherwise → Ask user which repo to review

## Step 1: Run Tests

### JavaScript/React (playhouse-web)
```bash
npm test -- --watchAll=false --coverage 2>&1 || true
```

### Python/Django (playhouse-server)
```bash
./.venv/bin/python -m pytest --tb=short -v 2>&1 || true
# For coverage:
./.venv/bin/python -m pytest --cov=apps --cov-report=term-missing 2>&1 || true
```

### Go (playhouse-control-plane)
```bash
go test ./... -v 2>&1 || true
# For coverage:
go test ./... -cover 2>&1 || true
```

Report pass/fail count and coverage.

## Step 2: Get Diff Against Main

```bash
git fetch origin main
git diff origin/main...HEAD
```

## Step 3: Review the Diff

Focus on (in priority order):

### JavaScript/React
- Bugs: logic errors, null/undefined handling, race conditions
- Behavioral regressions: changes that break existing functionality
- Security issues: XSS, injection, exposed secrets, auth bypasses
- Missing tests: new code paths without coverage

### Python/Django
- Bugs: logic errors, None handling, race conditions, ORM misuse
- Behavioral regressions: changes that break existing functionality
- Security issues: SQL injection, auth bypasses, exposed secrets, IDOR
- Missing tests: new code paths without coverage

### Go
- Bugs: logic errors, nil pointer dereferences, race conditions, goroutine leaks
- Behavioral regressions: changes that break existing functionality
- Security issues: SQL injection, auth bypasses, exposed secrets
- Missing tests: new code paths without coverage

## Output Format

```markdown
## Repository
- **Repo**: [playhouse-web | playhouse-server | playhouse-control-plane]
- **Branch**: [current branch name]

## Test Results
- **Status**: ✅ All passing / ❌ X failing
- **Coverage**: X%
- **Failing tests**: (list if any)

## Code Review Findings

### 🔴 Critical (must fix)
- [Finding with file:line reference]

### 🟡 Important (should fix)
- [Finding with file:line reference]

### 🟢 Suggestions (consider)
- [Finding with file:line reference]

## Summary
[1-2 sentence assessment]
```
