# Phase 2 - Server Entitlements Runtime

## Goal

Extend backend tenancy runtime so tenant app calls receive resolved flags and limits, then enforce entitlements server-side.

## Primary Repo

- `playhouse-server`

## Integration with Registry

Current integration already exists for:

- tenant resolution (`TenantMiddleware` + registry lookup)
- feature flags retrieval (`get_tenant_flags`)

This phase extends runtime to fetch **effective limits** and return them with flags.

## Runtime Work

- Add a registry-runtime resolver for entitlements:
  - `get_tenant_entitlements(tenant_subdomain_or_id)` -> `flags`, `limits`, `version`
- Extend tenant bootstrap endpoint:
  - either keep `/api/v1/flags` and add `limits`
  - or add `/api/v1/entitlements` and keep `/api/v1/flags` for compatibility
- Cache keys must include tenant + `config_version`.

## Backend Enforcement

Add guard utilities:

- `require_feature(flag_key)`
- `enforce_limit(limit_key, current_usage, requested_delta)`

Apply first to high-value endpoints:

- premium-only features (feature gates)
- creation/update endpoints that consume quota (limit gates)

## Error Semantics

- Feature denied -> HTTP 403 with `code=feature_not_enabled`
- Limit exceeded -> HTTP 429 (or 403 if preferred globally) with `code=quota_exceeded`

## Checklist

- [ ] Implement registry runtime query for limits.
- [ ] Add/extend API endpoint for entitlements bootstrap.
- [ ] Implement centralized enforcement helpers.
- [ ] Integrate helpers into first wave of protected endpoints.
- [ ] Add unit and API tests for precedence, cache behavior, and error semantics.

## Exit Criteria

- Tenant bootstrap returns complete effective entitlements.
- Protected backend endpoints enforce plans regardless of UI state.

