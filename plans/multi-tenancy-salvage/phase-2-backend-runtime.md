# Phase 2 - Backend Runtime Path (PR2)

## Scope

Implement end-to-end backend tenancy runtime on top of current `main`, using selective logic from old branches.

## Include

- Rebuild tenant middleware behavior (self-hosted and managed mode).
- Rebuild DB router behavior with explicit tenant-scoped model policy.
- Integrate registry client behavior (timeouts, fallbacks, caching policy as needed).
- Add/refresh tenancy API endpoints:
  - `GET /api/v1/flags`
  - `GET /api/v1/config`
- Port and adapt tenancy integration tests.

## Key Source References

- `config/middleware.py` from `origin/multi-tenancy-implementation`
- `config/db_router.py` from `origin/multi-tenancy-implementation`
- `config/registry_client.py` from `origin/multi-tenancy-implementation`
- `config/tests/test_tenancy.py` from `origin/multi-tenancy-implementation`
- `apps/tenancy/*` foundations from phase-0 branch

## Exclude

- Frontend consumption and UI gating.
- Broader infra rollout and DNS/proxy changes (separate ops track).

## Acceptance Criteria

- Tenant resolution from host works for expected formats.
- Tenant-scoped models route correctly in managed mode.
- Flags/config endpoints return stable contracts.
- Integration tests cover happy path and key negative cases.

## Suggested Verification

- `./.venv/bin/python -m pytest config/tests/test_tenancy.py`
- `./.venv/bin/python manage.py check`
- Targeted API smoke checks against `/api/v1/config` and `/api/v1/flags`

## Risks

- Settings and middleware order differences in current code.
- DB credential/host assumptions under Docker or pgbouncer.
- Drift between old tenant model lists and current domain models.
