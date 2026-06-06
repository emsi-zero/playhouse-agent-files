# Pilche Production Deployment - Summary of Changes

## Updated Plans for Control-Plane Orchestration

You were right! The plans have been revised to properly reflect the **control-plane** as the orchestration engine with **admin-ui** providing the user interface for managing all deployments.

### Key Changes

#### Phase 0: Routing & Infrastructure
- **NEW**: Control-plane API extensions (`5A`)
  - POST `/api/v1/releases` - Create releases
  - POST `/api/v1/releases/{version}/rollouts` - Start rollout
  - PATCH `/api/v1/releases/{version}/rollouts/{batch}` - Update batch status
  - POST `/api/v1/releases/{version}/rollback` - Trigger rollback
  - GET `/api/v1/deployments` - View current state

- **NEW**: Admin UI extensions (`5B`)
  - Deployments Dashboard
  - Release Management
  - Canary Deployment Wizard
  - Progressive Rollout UI
  - Tenant Migration Control
  - Real-time metrics WebSocket updates

#### Phase 2: Canary Deployment
- **CHANGED**: Canary now triggered via Admin UI wizard
  - User navigates to `/admin-ui/deployments/`
  - Selects version and canary tenants
  - CP-API creates jobs for each tenant
  - CP-Controller orchestrates migrations
  - Tenants automatically routed after migration
  - Admin UI shows real-time progress

- **CHANGED**: Rollback via Admin UI button
  - Click "Rollback Canary"
  - CP-API triggers batch rollback
  - CP-Controller updates registry
  - Tenants re-route to v0.x.x automatically
  - Estimated time: <5 minutes

#### Phase 3: Progressive Rollout
- **CHANGED**: Each batch deployed via Admin UI
  - User clicks "Deploy Batch 1" (pre-selected from planning)
  - CP-Controller orchestrates all migrations
  - Admin UI shows migration progress bar
  - Automatic routing after migrations complete

- **CHANGED**: Batch rollback via Admin UI
  - No manual scripts needed
  - One-click rollback for any batch
  - Automatic tenant re-routing

---

## Architecture

```
┌─────────────────────────────────────────┐
│           Admin UI                      │
│  (User Interface for Deployments)       │
│  - Deployment Dashboard                 │
│  - Release Management                   │
│  - Canary/Batch Wizards                 │
│  - Monitoring/Metrics                   │
└──────────────┬──────────────────────────┘
               │ (User clicks Deploy/Rollback)
               ↓
┌─────────────────────────────────────────┐
│        Control-Plane API (cp-api)       │
│   (REST Endpoints for Operations)       │
│  - /api/v1/releases                     │
│  - /api/v1/releases/{v}/rollouts        │
│  - /api/v1/releases/{v}/rollback        │
│  - /api/v1/deployments                  │
└──────────────┬──────────────────────────┘
               │ (API calls)
               ↓
┌─────────────────────────────────────────┐
│     Control-Plane Controller (cp-ctrl)  │
│   (Orchestration Engine)                │
│  - Create migration jobs                │
│  - Poll job status                      │
│  - Update Registry DB                   │
│  - Trigger routing service cache clear  │
│  - Track rollout progress               │
└──────────────┬──────────────────────────┘
               │ (Updates)
               ↓
┌─────────────────────────────────────────┐
│         Registry Database               │
│   (Source of Truth)                     │
│  - tenant_releases                      │
│  - deployment_pools                     │
│  - rollout_batches                      │
│  - migration_jobs                       │
└──────────────┬──────────────────────────┘
               │ (Routing queries)
               ↓
┌─────────────────────────────────────────┐
│      Routing Service                    │
│  (Tenant → Server Mapper)               │
│  - Queries Registry DB                  │
│  - In-memory cache (60s TTL)            │
│  - HTTP endpoint for nginx              │
└──────────────┬──────────────────────────┘
               │ (Routing decisions)
               ↓
        ┌──────┴──────┐
        ↓             ↓
    ┌─────────┐   ┌──────────┐
    │ Server A│   │ Server B │
    │ v0.x.x  │   │ v1.0.0   │
    │ Tenants │   │ Tenants  │
    └─────────┘   └──────────┘
```

---

## Deployment Flow

### Example: Canary Deployment

```
1. Admin logs into /admin-ui/deployments/

2. Clicks "Deploy Canary"
   - Shows pre-filled canary tenants
   - Shows version: v1.0.0
   - Shows estimated time: ~20 min

3. Confirms deployment
   ↓
   POST /api/v1/releases/v1.0.0/rollouts
   {
     batch_number: 0,
     tenant_ids: [canary_1, canary_2, canary_3],
     db_schema_version: 1
   }

4. CP-Controller receives request
   - Creates 3 migration jobs
   - For each job:
     * Run Django migrate for tenant DB
     * Monitor job status
     * Mark complete when done
   - After all jobs complete:
     * Update tenant_releases (target_version = v1.0.0)
     * Routing service clears cache
     * Tenants automatically re-route to server-b

5. Admin UI shows real-time progress
   ████████░░░░░░░░░░░░ 80% complete
   - Tenant 1: ✓
   - Tenant 2: ✓
   - Tenant 3: ⏳

6. After completion (T+20 min)
   - All canary tenants on server-b
   - Admin UI shows metrics dashboard
   - Team monitors for 24-48h

7. Go/No-Go Decision
   - If HEALTHY: Click "Deploy Batch 1" → Repeat from step 2
   - If ISSUES: Click "Rollback" → Tenants back on server-a in <5 min
```

---

## Benefits of Control-Plane Orchestration

✅ **Safety**: All state in Registry DB, rollback always available  
✅ **Automation**: No manual scripts, no human errors  
✅ **Visibility**: Admin UI shows real-time progress  
✅ **Auditability**: Every action logged with who/what/when  
✅ **Reliability**: Jobs are idempotent, migrations retry on failure  
✅ **Reversibility**: Rollback any batch at any time  
✅ **Scalability**: Same procedure for canary to 100% of tenants  
✅ **Maintainability**: Reusable for v2.0.0, v3.0.0, etc.

---

## Files Modified/Created

### New Files
- `CONTROL-PLANE-ORCHESTRATION.md` - Complete orchestration guide

### Updated Files
- `phase-0-routing-and-infrastructure.md`
  - Added: Control-plane API extensions (section 5A)
  - Added: Admin UI extensions (section 5B)
  - Updated: Repos list to include admin-ui updates

- `phase-2-production-preparation-and-canary.md`
  - Updated: Canary deployment now via Admin UI (section 9)
  - Updated: Rollback now via Admin UI (section 11)

- `phase-3-progressive-rollout.md`
  - Updated: Batch 1 deployment now via Admin UI (section 3)
  - Updated: Batch rollback now via Admin UI (section 5)

- `README.md`
  - Added: Section on Control-Plane Orchestration
  - Referenced: CONTROL-PLANE-ORCHESTRATION.md

---

## How to Use the Updated Plans

1. **Read first**: `CONTROL-PLANE-ORCHESTRATION.md`
   - Understand how control-plane orchestrates deployments
   - Understand Admin UI user flows

2. **Follow phases in order**:
   - **Phase 0**: Implement routing + CP API + Admin UI
   - **Phase 1**: Test in staging
   - **Phase 2**: Canary deployment via Admin UI
   - **Phase 3**: Progressive rollout via Admin UI
   - **Phase 4**: Server upgrade

3. **During deployment**: Use Admin UI
   - No manual scripts or shell commands
   - All operations tracked and auditable
   - Real-time progress monitoring

4. **If issues arise**: Click "Rollback"
   - CP-Controller handles re-routing
   - <5 minute recovery time
   - No manual intervention needed

---

## Key Points

🔑 **Control-plane is the orchestration engine**
- Manages all deployment operations
- Ensures consistent state across system
- Provides reliable rollback capability

🔑 **Admin UI is the user interface**
- No command-line needed
- User-friendly deployment wizards
- Real-time monitoring and metrics

🔑 **All deployments are automated**
- Migrations run automatically per tenant
- Routing updates happen automatically
- Rollback is one-click

🔑 **Everything is reversible**
- Any batch can be rolled back
- Registry DB is source of truth
- Historical state always available

🔑 **Safety first**
- Small batches first (canary 5-10%)
- 24-48h monitoring between batches
- Go/no-go decision gates
- Automatic rollback on issues

---

## Next Steps

1. **Review** the updated plans with your team
2. **Validate** control-plane architecture with your implementation
3. **Start Phase 0** - implement CP API endpoints and Admin UI
4. **Test Phase 1** - staging deployment with both server versions
5. **Launch Phase 2** - first canary on production
6. **Execute Phases 3-4** - progressive rollout and finalization

All procedures are now **control-plane driven** with **admin-ui** as the single source of user interaction.

---

**Updated**: June 1, 2026  
**Status**: Ready for implementation
