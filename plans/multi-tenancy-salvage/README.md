# Multi-Tenancy Salvage Plan

This plan converts old multi-tenancy branches into a clean, reviewable sequence of PRs.

## Goal

- Reuse validated ideas from old branches.
- Avoid reviving stale branches directly.
- Rebuild integration-heavy parts on current code.

## PR Sequence

1. `phase-1-foundation.md` (Server foundation and tests)
2. `phase-2-backend-runtime.md` (Server runtime path and APIs)
3. `phase-3-web-flags.md` (Web flags client and app wiring)

## Source Branches to Mine

- `playhouse-server`: `origin/multi-tenancy/phase-0`
- `playhouse-server`: `origin/multi-tenancy-implementation`
- `playhouse-web`: `origin/multi-tenancy-implementation`

## Done Criteria (overall)

- PR1-PR3 merged in order to the approved integration line.
- Tenant resolution, DB routing, and flags flow work in local and staging smoke tests.
- Legacy stale branches are no longer needed for active development.
