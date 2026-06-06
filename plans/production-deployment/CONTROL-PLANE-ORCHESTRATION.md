# Control-Plane Orchestration for Pilche Deployment

## Overview

The production deployment of Pilche uses the **control-plane** as the orchestration layer for all canary rollouts and progressive tenant migrations. The **admin-ui** provides the user interface for managing deployments.

## Architecture

```
Admin UI (User Interface)
    ↓ (User clicks "Deploy Batch 1")
cp-api (REST API Layer)
    ↓ (Accepts deployment requests)
cp-controller (Orchestration Engine)
    ↓ (Manages jobs, migrations, routing)
Registry DB (Persistent State)
    ├── tenant_releases (tenant → version mapping)
    ├── deployment_pools (version → server mapping)
    ├── rollout_batches (batch progress tracking)
    └── migration_jobs (job status)
    ↓
Routing Service (Reads Registry)
    ↓ (Invalidates cache on changes)
Nginx (Routes Tenants)
```

## How Deployments Work

### Phase 1: Admin UI Initiates Deployment

**User navigates to Admin UI**:
```
/admin-ui/deployments/
```

**UI shows current state**:
- v0.x.x: 100% of tenants (server-a)
- v1.0.0: 0% of tenants (server-b idle)
- Release: v1.0.0 (pending)

**User clicks "Deploy Canary" or "Deploy Batch 1"**:
- Pre-filled tenants from planning phase
- Review and confirm deployment

### Phase 2: CP-API Receives Request

**Admin UI calls**:
```json
POST /api/v1/releases/v1.0.0/rollouts
{
  "batch_number": 1,
  "tenant_ids": ["batch-1-tenant-1", "batch-1-tenant-2", ...],
  "target_server": "server-b:8000",
  "db_schema_version": 1,
  "scheduled_start": "2026-06-03T08:15:00Z",
  "monitoring_window_hours": 48
}
```

**CP-API validates request**:
- Verify version exists
- Verify tenants exist
- Verify authorization (admin-only)
- Store rollout request in database

**CP-API responds**:
```json
{
  "rollout_id": "rollout-v1.0.0-batch-1",
  "status": "scheduled",
  "batch_number": 1,
  "tenant_count": 10,
  "estimated_duration_minutes": 30
}
```

**Admin UI shows**:
```
Batch 1 Deployment Scheduled
- Tenants: 10
- Estimated duration: 30 minutes
- Status: waiting_to_start
- Starting in: T-00:05:00
```

### Phase 3: CP-Controller Executes Deployment

**When scheduled time arrives**:

1. **cp-controller polls** for pending rollouts (every 10 seconds)

2. **For each tenant in batch**:
   ```
   Migrate tenant-batch-1-tenant-1:
   - Create job: migrate_schema v0→v1
   - Job runs: Django migrations
   - Poll job status
   - When complete: mark tenant as ready_to_route
   ```

3. **Update Registry** (after all migrations complete):
   ```sql
   UPDATE tenant_releases
   SET target_version = 'v1.0.0', status = 'in_progress'
   WHERE tenant_id IN (...batch_1_tenants...);
   ```

4. **Invalidate Routing Cache**:
   - Routing service receives notification
   - Cache TTL expires (60s)
   - Next request for updated tenant goes to new server

5. **Update Rollout Status**:
   ```json
   {
     "status": "in_progress",
     "progress": "100% (10/10 tenants migrated)",
     "tenants_ready": 10,
     "current_server": "server-b:8000"
   }
   ```

### Phase 4: Admin UI Displays Real-Time Progress

**During deployment**:
```
Batch 1 Deployment
████████░░░░░░░░░░░░ 80% complete

Migration Progress:
✓ tenant-1 - 2m ago
✓ tenant-2 - 1m 30s ago
✓ tenant-3 - 1m ago
⏳ tenant-4 - in_progress (1m 20s elapsed)
⏳ tenant-5 - pending

Estimated remaining: 5 minutes
```

**After deployment**:
```
Batch 1 Deployed ✓
- All tenants routed to v1.0.0
- Current: 10 tenants on server-b
- Monitoring started
- Next check: in 4 hours
```

### Phase 5: Monitoring & Decision

**Admin UI shows monitoring dashboard**:
```
Batch 1 Metrics (Last 2 hours)
- Error Rate: 0.85% (baseline: 0.8%) ✓
- Latency p95: 128ms (baseline: 125ms) ✓
- Uptime: 99.95% ✓
- Support tickets: 0 ✓

Status: HEALTHY - Ready for next batch?
```

**Team reviews and decides**:
- If GO: Click "Deploy Batch 2" → Repeat from Phase 1
- If NO-GO: Click "Rollback Batch 1" → Proceed to Phase 6

### Phase 6: Rollback (if needed)

**If issues detected**:

```
Batch 1 Issues Detected
- Error rate spike: 5.2% (threshold: 1.0%)
- Support tickets: 3 (threshold: 0)

Recommend: ROLLBACK
```

**User clicks "Rollback"**:

**Admin UI calls**:
```json
POST /api/v1/releases/v1.0.0/rollback
{
  "batch_number": 1,
  "reason": "Error rate spike detected",
  "rollback_to_version": "v0.x.x"
}
```

**CP-Controller executes rollback**:
1. Update tenant_releases: target_version = v0.x.x, status = rolled_back
2. Invalidate routing cache
3. Tenants automatically re-route to server-a
4. Update rollout status: "rolled_back"

**Admin UI shows**:
```
Batch 1 Rolled Back ✓
- All tenants back on v0.x.x (server-a)
- Rollback completed in: 2 minutes
- Metrics recovering to baseline
```

---

## Key Responsibilities

### Admin UI
- User interface for viewing deployment status
- Forms for initiating deployments
- Real-time monitoring dashboard
- Progress tracking
- Rollback buttons
- Audit log display

### CP-API
- REST endpoints for deployment requests
- Request validation and authorization
- Database persistence
- Job creation
- Status queries

### CP-Controller
- Orchestration engine
- Job execution (migrations)
- Database state management
- Registry updates
- Tenant routing updates
- Rollout progress tracking

### Routing Service
- Queries registry for tenant → server mapping
- Caches decisions (60s TTL)
- Handles cache invalidation
- Provides nginx with routing decisions

### Registry Database
- Stores tenant_releases (tenant → version)
- Stores deployment_pools (version → server)
- Stores rollout_batches (batch progress)
- Stores migration_jobs (job status)

---

## API Endpoints (CP-API)

### Releases Management

```
POST   /api/v1/releases
GET    /api/v1/releases
GET    /api/v1/releases/{version}
PATCH  /api/v1/releases/{version}
DELETE /api/v1/releases/{version}
```

### Rollout Management

```
POST   /api/v1/releases/{version}/rollouts
GET    /api/v1/releases/{version}/rollouts
GET    /api/v1/releases/{version}/rollouts/{batch_number}
PATCH  /api/v1/releases/{version}/rollouts/{batch_number}
POST   /api/v1/releases/{version}/rollback
```

### Deployment Status

```
GET    /api/v1/deployments
GET    /api/v1/deployments/status
GET    /api/v1/deployments/metrics
```

---

## Data Model

### tenant_releases
```
tenant_id (PK, FK tenants)
target_version (v0.x.x, v1.0.0, etc.)
db_schema_version (0, 1, 2, ...)
status (pending, in_progress, completed, rolled_back)
migrated_at (timestamp)
```

### deployment_pools
```
version (v0.x.x, v1.0.0, etc.)
server_host (server-a:8000, server-b:8001)
server_port
web_assets_path (/assets/vX.X.X)
status (active, draining, idle)
```

### rollout_batches
```
release_version (v1.0.0)
batch_number (0=canary, 1=first batch, ...)
tenant_count
percentage (10%, 25%, 50%, 100%)
status (pending, in_progress, completed, rolled_back)
started_at, completed_at
```

### migration_jobs
```
job_id (unique)
tenant_id
job_type (migrate_schema)
target_schema_version
status (pending, in_progress, completed, failed)
created_at, started_at, completed_at
error_message (if failed)
```

---

## Workflow Summary

1. **Admin logs into Admin UI** (`/admin-ui/deployments/`)
2. **User selects version and tenants** for deployment
3. **System shows review** (tenants, estimated time)
4. **User confirms deployment** (one click)
5. **CP-API receives request** and validates
6. **CP-Controller schedules jobs** for tenant DB migrations
7. **Jobs execute** (run Django migrations per tenant)
8. **After all jobs complete**, registry is updated with new routing
9. **Routing service picks up changes** (cache TTL expires)
10. **Tenants automatically routed** to new server version
11. **Admin UI shows progress** in real-time
12. **Team monitors metrics** (via Admin UI or dashboards)
13. **Team makes go/no-go decision** (Deploy next batch or Rollback)
14. **If Rollback**: CP-API updates registry, tenants re-route to old version
15. **If Go**: Repeat from step 1 for next batch

---

## Benefits of Control-Plane Orchestration

✅ **Automated**: No manual scripts, no shell commands during deployment
✅ **Safe**: All state tracked in registry, rollback always available
✅ **Observable**: Admin UI shows real-time progress and metrics
✅ **Auditable**: All actions logged (who, what, when)
✅ **Reliable**: Jobs retry on failure, migrations are idempotent
✅ **Reversible**: Rollback any batch at any time
✅ **Scalable**: Works for canary (5-10%) to 100% of tenants
✅ **Repeatable**: Same procedure for v1.0.0 → v2.0.0, etc.

---

## Admin UI Screenshots (Conceptual)

### Deployments Dashboard
```
┌────────────────────────────────────────────┐
│ Pilche Deployments                         │
├────────────────────────────────────────────┤
│                                            │
│ Current Release: v1.0.0                   │
│ Status: Progressive Rollout in Progress   │
│                                            │
│ Distribution:                              │
│ ████████░░░░░░░░░░░░ 40% (20 of 50)      │
│                                            │
│ Canary (Done):      ████ 5 tenants        │
│ Batch 1 (Done):     ████ 5 tenants        │
│ Batch 2 (Active):   ███░ 5 tenants ⏳    │
│ Batch 3 (Pending):  ░░░░ 0 tenants       │
│ Batch 4 (Pending):  ░░░░ 0 tenants       │
│                                            │
│ [Deploy Batch 3] [Pause] [Rollback]      │
│                                            │
└────────────────────────────────────────────┘
```

### Release Management
```
┌────────────────────────────────────────────┐
│ Release v1.0.0                            │
├────────────────────────────────────────────┤
│ Status: in_progress                        │
│ Created: 3 hours ago                       │
│ Release Type: Feature Release              │
│                                            │
│ Rollout Progress:                          │
│ Canary: ✓ Complete (5 tenants)            │
│ Batch 1: ✓ Complete (5 tenants)           │
│ Batch 2: ⏳ In Progress (5 tenants)       │
│   └─ 80% complete (4 migrated)            │
│ Batch 3: ⏹ Not started (25 tenants)      │
│ Batch 4: ⏹ Not started (10 tenants)      │
│                                            │
│ Metrics:                                   │
│ Error Rate: 0.9% ✓                        │
│ Latency p95: 130ms ✓                      │
│ Uptime: 99.9% ✓                           │
│                                            │
│ [Deploy Next Batch] [View Logs] [Rollback]│
│                                            │
└────────────────────────────────────────────┘
```

---

## Notes

- All operations are **idempotent** (can be retried without side effects)
- **Rollback is always available** (revert any batch at any time)
- **No manual shell scripts needed** (all via Admin UI)
- **Audit trail** (every action logged in database)
- **Real-time monitoring** (Admin UI pushes metrics via WebSocket)
- **Cross-region ready** (if extending to multiple DCs later)
