# Phase 1 - Server Foundation (PR1)

## Scope

Port the low-risk tenancy foundation from `playhouse-server` into current codebase.

## Include

- `apps/tenancy/context.py`
- `apps/tenancy/exceptions.py`
- `apps/tenancy/registry.py` (core read patterns only)
- `apps/tenancy/tests/test_context.py`
- `apps/tenancy/tests/test_registry.py`
- `apps/tenancy/tests/test_imports.py`
- `apps/tenancy/apps.py`
- `apps/tenancy/README.md` (updated wording)
- Optional dev helper: `scripts/create_test_tenants.py`

## Exclude

- Middleware, DB router, settings wiring, URL wiring, flags API.
- Any production routing behavior.

## Acceptance Criteria

- Tenancy foundation modules import and unit tests pass.
- No runtime request path behavior changes yet.
- Minimal settings changes only for safe initialization.

## Suggested Verification

- `./.venv/bin/python -m pytest apps/tenancy/tests/test_context.py`
- `./.venv/bin/python -m pytest apps/tenancy/tests/test_registry.py`
- `./.venv/bin/python -m pytest apps/tenancy/tests/test_imports.py`

## Risks

- Import path drift with current app layout.
- Registry helper assumptions that no longer match current settings.
