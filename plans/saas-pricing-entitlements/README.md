# SaaS Pricing and Entitlements Implementation Plan

This plan introduces pricing plans for customer tenants and enforces feature/limit access based on purchased plans.

## Objectives

- Model plans as entitlement bundles.
- Enforce access in backend and frontend.
- Keep tenant-level override flexibility for support/sales.
- Preserve safe rollout and rollback behavior in multi-tenant production.

## Phases

1. `phase-0-contract-and-governance.md`
2. `phase-1-control-plane-registry.md`
3. `phase-2-server-entitlements-runtime.md`
4. `phase-3-web-entitlements-consumption.md`
5. `phase-4-rollout-observability-and-ops.md`
6. `phase-5-validation-and-launch.md`

## Current Architecture Fit

- Registry already has `tenants.plan`, `feature_flags`, `tenant_feature_flags`, and `config_version`.
- Server already resolves tenant from subdomain and exposes tenant-scoped `GET /api/v1/flags`.
- Control-plane already updates `config_version` when tenant feature flags are changed.

This plan extends these existing capabilities instead of replacing them.

