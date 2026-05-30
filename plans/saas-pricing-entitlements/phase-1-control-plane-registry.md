# Phase 1 - Control Plane and Registry

## Goal

Add registry schema and cp-api capabilities to manage plans, limits, and tenant overrides.

## Primary Repo

- `playhouse-control-plane`

## Schema Changes

Add tables:

- `plans(plan_key, display_name, status, metadata_json, created_at, updated_at)`
- `plan_feature_flags(plan_key, flag_key, enabled, payload_json, updated_at)`
- `plan_limits(plan_key, limit_key, limit_value, unit, updated_at)`
- `tenant_limits(tenant_id, limit_key, limit_value, reason, expires_at, updated_at)`

Notes:

- Keep existing `tenants.plan`, `feature_flags`, and `tenant_feature_flags`.
- Add FK/indexes for efficient entitlement resolution.

## cp-api Additions

- Plan catalog:
  - `GET /api/v1/plans`
  - `POST /api/v1/plans`
  - `PATCH /api/v1/plans/:key`
- Plan defaults:
  - `PUT /api/v1/plans/:key/feature-flags`
  - `PUT /api/v1/plans/:key/limits`
- Tenant overrides:
  - keep existing tenant feature-flags API
  - add `GET/PUT /api/v1/tenants/:id/limits`
- Effective preview:
  - `GET /api/v1/tenants/:id/entitlements` (resolved flags + limits + version)

## Behavior Requirements

- Any effective entitlement change must increment `tenants.config_version`.
- Audit all mutations in `audit_log`.
- Validate all incoming keys and values (`limit_value >= 0`, known keys only unless explicitly open).

## Checklist

- [ ] Create DB migration files for new tables and indexes.
- [ ] Add data access layer and API handlers.
- [ ] Add validation and audit logging for all mutation endpoints.
- [ ] Add `config_version` bump rules for plan/override changes.
- [ ] Add API tests for CRUD, validation, and version bump behavior.

## Exit Criteria

- cp-api can fully manage plan catalog/defaults/tenant overrides.
- Integration tests confirm deterministic entitlement resolution and version updates.

