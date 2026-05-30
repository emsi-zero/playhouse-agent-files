# Phase 3 - Web Entitlements Consumption

## Goal

Use tenant entitlements in the customer web app to gate UX, surface plan limits, and guide upgrades.

## Primary Repo

- `playhouse-web`

## App Bootstrap

- Load entitlements during app initialization.
- Store in a single client state source (context/store) with TTL-aware refresh.
- Ensure graceful fallback for missing entitlements data.

## UI Gating

- Feature gates:
  - hide/disable premium screens/actions when flag is off
- Limit gates:
  - disable create actions when quota exhausted
  - display current usage and plan cap
- Upsell UX:
  - consistent "upgrade required" messaging

## Security Reminder

- Frontend gating is UX only.
- Backend enforcement from Phase 2 remains authoritative.

## Checklist

- [ ] Add entitlements client and bootstrap integration.
- [ ] Implement reusable hook(s): `useFeature`, `useLimit`.
- [ ] Gate first set of premium features and quota-based actions.
- [ ] Add user-facing limit and upgrade messages.
- [ ] Add component/integration tests for gated flows.

## Exit Criteria

- UI behavior reflects effective tenant entitlements at runtime.
- No protected feature appears as enabled when backend would deny it.

