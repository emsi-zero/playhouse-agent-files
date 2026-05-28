# Phase 3 - Web Flags and Wiring (PR3)

## Scope

Reintroduce and integrate web feature-flags support on top of current `playhouse-web`.

## Include

- Recreate `src/api/flagsApi.js` behavior from old branch:
  - safe default response
  - timeout/abort
  - defensive JSON handling
- Wire flags loading into app bootstrap/store.
- Add feature-gating usage for initial targeted screens/features.
- Add tests for client fallback behavior and malformed payloads.

## Source Reference

- `playhouse-web` `origin/multi-tenancy-implementation` (`src/api/flagsApi.js`)

## Exclude

- Tenant identity resolution logic in frontend (backend/host-driven).
- Broad UI redesign; keep this PR focused on flags integration.

## Acceptance Criteria

- App boots with endpoint available and unavailable.
- Missing or invalid response does not break UI.
- Flags become readable by target feature gates.

## Suggested Verification

- Run web unit tests covering `flagsApi` and bootstrapping path.
- Manual smoke:
  - backend up: flags are fetched and applied
  - backend down/non-JSON: app still runs with defaults

## Risks

- Initialization timing issues during app startup.
- Inconsistent API base URL handling between environments.
