# Phase 4 - Rollout, Observability, and Operations

## Goal

Roll out pricing entitlements safely in production with visibility and rollback controls.

## Repos

- `playhouse-control-plane`
- `playhouse-server`
- `playhouse-cicd`

## Rollout Strategy

- Start with non-blocking observability mode:
  - compute and log would-be denials without enforcing limits for selected tenants
- Enable enforcement for canary tenants.
- Gradually expand to all tenants.

## Observability

Track per tenant and per key:

- feature denials count
- quota exceeded count
- quota utilization percentage (where measurable)
- entitlement fetch errors and latency
- stale version/caching anomalies

## Operational Controls

- Emergency kill switch flags in registry for fast disable.
- Runbook for:
  - reverting enforcement to observe-only mode
  - reverting specific feature gates
  - temporary tenant overrides for support incidents

## Checklist

- [ ] Add metrics and structured logs for entitlement checks.
- [ ] Add dashboards and alerts for denial spikes.
- [ ] Add CICD/env wiring for rollout toggles.
- [ ] Document rollback and emergency override procedures.
- [ ] Dry-run incident scenario in staging.

## Exit Criteria

- Canary rollout completed without critical regressions.
- Ops team can monitor, override, and rollback confidently.

