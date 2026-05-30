# Phase 0 - Contract and Governance

## Goal

Define a stable product and technical contract for plans, flags, and limits before schema and code changes.

## Repos

- `playhouse-docs` (source-of-truth docs)
- `playhouse-control-plane` (API contract consumers/producers)
- `playhouse-server` (runtime consumer)
- `playhouse-web` (UI consumer)

## Deliverables

- Canonical entitlement vocabulary:
  - `flag_key` for boolean/tunable features
  - `limit_key` for numeric quotas
- Plan catalog proposal (`starter`, `growth`, `enterprise` or equivalents).
- Precedence rules:
  - feature flags: global default -> plan default -> tenant override
  - limits: plan default -> tenant override
- API contract for tenant app bootstrap response:
  - includes `flags`, `limits`, `version`
- Error contract for enforcement:
  - `feature_not_enabled`
  - `quota_exceeded`

## Checklist

- [ ] Finalize initial plan matrix (features and limits).
- [ ] Define naming conventions for `flag_key` and `limit_key`.
- [ ] Define response and error payload formats.
- [ ] Define ownership workflow (product/ops/engineering) for entitlement changes.
- [ ] Define backward compatibility policy for plan migrations.

## Exit Criteria

- Architecture decision record approved.
- Contract doc published in `playhouse-docs`.
- All downstream phases reference the approved keys/contracts only.

