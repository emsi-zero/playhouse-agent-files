# Production Deployment - Phase 0: Routing Service & Infrastructure

## Goal

Establish the foundational infrastructure for blue-green multi-version deployments with dynamic tenant routing and dual static assets.

## Repos

- `playhouse-control-plane` (routing service, registry schema, API extensions, admin UI)
- `playhouse-cicd` (docker-compose, nginx, deployment scripts)
- `playhouse-docs` (runbooks, architecture)

## Overview

This phase implements:
1. **Routing Service**: Lightweight Go service that queries registry to determine tenant→server mapping
2. **Registry Schema Upgrades**: Tables to track routing policies and deployment pools
3. **Control-Plane API Extensions**: New endpoints for managing rollouts, releases, and deployments
4. **Admin UI Extensions**: UI for managing canary rollouts, progressive deployments, and tenant migrations
5. **Nginx Configuration**: Auth request module integration for dynamic routing
6. **Static Asset Structure**: Versioned paths for web assets

## Work Items

### 1. Registry Schema Extensions

**Location**: `playhouse-control-plane/migrations/`

Add three tables to support multi-version deployments:

```sql
-- Track which version each tenant is routed to
CREATE TABLE tenant_releases (
    id SERIAL PRIMARY KEY,
    tenant_id UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    target_version VARCHAR(50) NOT NULL,     -- e.g. "v1.0.0", "v0.x.x"
    db_schema_version INT NOT NULL DEFAULT 0, -- 0, 1, 2...
    status VARCHAR(20) DEFAULT 'pending',    -- pending|in_progress|completed|rolled_back
    migrated_at TIMESTAMP,
    notes TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(tenant_id)
);

-- Track deployment pool (server version → instance mapping)
CREATE TABLE deployment_pools (
    id SERIAL PRIMARY KEY,
    version VARCHAR(50) NOT NULL UNIQUE,    -- e.g. "v1.0.0"
    server_host VARCHAR(255) NOT NULL,      -- e.g. "server-b:8000"
    server_port INT NOT NULL,
    web_assets_path VARCHAR(255),           -- e.g. "/assets/v1.0.0"
    status VARCHAR(20) DEFAULT 'active',    -- active|draining|idle
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Track rollout batches (for gradual rollout progress)
CREATE TABLE rollout_batches (
    id SERIAL PRIMARY KEY,
    release_version VARCHAR(50) NOT NULL,
    batch_number INT NOT NULL,
    tenant_count INT NOT NULL,
    percentage INT NOT NULL,                 -- e.g. 10, 50, 100
    status VARCHAR(20) DEFAULT 'pending',   -- pending|in_progress|completed|rolled_back
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    notes TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(release_version, batch_number)
);

-- Index for quick lookups
CREATE INDEX idx_tenant_releases_tenant_id ON tenant_releases(tenant_id);
CREATE INDEX idx_tenant_releases_status ON tenant_releases(status);
CREATE INDEX idx_deployment_pools_status ON deployment_pools(status);
CREATE INDEX idx_rollout_batches_version ON rollout_batches(release_version, status);
```

**Changes**:
- [ ] Create migration file `migrations/NNNN_add_routing_tables.sql`
- [ ] Update migration runner to apply new schema
- [ ] Add schema versioning documentation

### 2. Routing Service (Go)

**Location**: `playhouse-control-plane/cmd/routing-service/`

Lightweight HTTP service that queries registry and returns routing decisions:

```go
// main.go
// GET /route?tenant=<subdomain>
// Returns: {target_server, web_version, db_schema_version}

// GET /health
// Returns: {status: "ok"}

// Cache with 60s TTL on routing decisions
// Queries registry_db for tenant_releases + deployment_pools
```

**Changes**:
- [ ] Create routing service binary in `cmd/routing-service/main.go`
- [ ] Implement registry query layer (GET tenant routing + deployment pool info)
- [ ] Add in-memory cache with TTL
- [ ] Add Dockerfile for routing-service image
- [ ] Health check endpoint
- [ ] Metrics endpoint (for monitoring)

**Acceptance Criteria**:
- Service returns correct server mapping for each tenant
- Cache invalidation works (60s TTL verified)
- Handles missing tenant gracefully (returns error or default)
- Starts in <2s

### 3. Nginx Configuration Updates

**Location**: `playhouse-cicd/nginx/`

Update nginx to use auth_request for dynamic routing:

**nginx.conf changes**:
```nginx
# Enable auth_request module
# Query routing service before proxying
```

**proxy-locations.conf changes**:
```nginx
# Route to server based on routing service response
# Serve versioned static assets (/assets/vX.X.X/*)
# Set X-Version header for web app
```

**New file: nginx/routing-auth.conf**:
```nginx
# Auth request to routing service
# Set upstream variables based on response headers
```

**Changes**:
- [ ] Add `auth_request` directive to route requests through routing service
- [ ] Add location blocks for `/assets/vX.X.X/` (versioned assets)
- [ ] Add location `/auth/route` (internal auth request)
- [ ] Configure response header parsing (X-Target-Server, X-Web-Version)
- [ ] Update proxy_pass to use dynamic upstream ($target_server)

**Acceptance Criteria**:
- Requests to `tenant.pilche.ir` are routed to correct server
- Static assets served from versioned paths
- X-Version header present in responses
- Fallback to default version if routing service unavailable

### 4. Docker Compose Production File

**Location**: `playhouse-cicd/docker-compose.prod.yml`

Create production deployment file supporting dual servers (A and B):

**Changes**:
- [ ] Add routing-service container
- [ ] Add server-a service (pinned to v0.x.x image tag)
- [ ] Add server-b service (pinned to v1.0.0 image tag, initially disabled or idle)
- [ ] Add deployment pool initialization script
- [ ] All services have health checks
- [ ] Proper restart policies
- [ ] Resource limits defined

**Structure**:
```yaml
services:
  routing-service:
    image: playhouse-routing-service:${PILCHE_TAG}
    environment:
      REGISTRY_DATABASE_URL: ...
    ports:
      - "9000:9000"  # Internal routing service port
    
  server-a:
    image: playhouse-server:v0.x.x  # Current stable
    environment:
      TENANT_ROUTING_MODE: "active"
    
  server-b:
    image: playhouse-server:v1.0.0  # New version (initially idle)
    environment:
      TENANT_ROUTING_MODE: "idle"
    deploy:
      replicas: 0  # Start with 0
```

**Acceptance Criteria**:
- Compose file is valid (docker-compose config passes)
- Can start full stack with `docker-compose up`
- Routing service queries registry correctly

### 5. Deployment Pool Initialization Script

**Location**: `playhouse-cicd/scripts/init-deployment-pools.sh`

Initialize deployment pools in registry when starting production:

```bash
#!/bin/bash
# Create deployment pool entries for current versions
# Called after registry-migrate completes
# Ensures routing table knows where to find each version
```

**Changes**:
- [ ] Script queries current versions from docker-compose
- [ ] Creates deployment_pools entries via cp-api
- [ ] Sets default pool (current version)
- [ ] Validates pool creation

### 5A. Control-Plane API Extensions

**Location**: `playhouse-control-plane/internal/api/`

Add new endpoints for managing rollouts and deployments.

**New Endpoints**:

- [ ] `POST /api/v1/releases` - Create a new release
  ```json
  {
    "version": "v1.0.0",
    "status": "pending",
    "target_tenants": ["canary1", "canary2"],
    "notes": "Initial canary deployment"
  }
  ```

- [ ] `GET /api/v1/releases` - List all releases with their status

- [ ] `PATCH /api/v1/releases/{version}` - Update release (change status, target tenants)
  ```json
  {
    "status": "in_progress",
    "current_batch": 1,
    "batch_progress": "35%"
  }
  ```

- [ ] `POST /api/v1/releases/{version}/rollouts` - Start or update rollout
  ```json
  {
    "batch_number": 1,
    "tenant_ids": ["tenant1", "tenant2", "tenant3"],
    "target_server": "server-b:8000",
    "db_schema_version": 1,
    "scheduled_start": "2026-06-02T08:00:00Z",
    "monitoring_window_hours": 24
  }
  ```

- [ ] `GET /api/v1/releases/{version}/rollouts` - Get rollout status

- [ ] `PATCH /api/v1/releases/{version}/rollouts/{batch_number}` - Update batch status
  ```json
  {
    "status": "monitoring",
    "health_check_passed": true,
    "metrics": {
      "error_rate": 0.85,
      "latency_p95": 128,
      "uptime": 99.95
    }
  }
  ```

- [ ] `POST /api/v1/releases/{version}/rollback` - Trigger rollback
  ```json
  {
    "batch_number": 1,
    "reason": "Error rate spike detected",
    "rollback_to_version": "v0.x.x"
  }
  ```

- [ ] `GET /api/v1/deployments` - View current deployment state
  ```json
  {
    "current_version": "v1.0.0",
    "server_a": {
      "image": "playhouse-server:v1.0.0",
      "status": "active",
      "tenant_count": 50
    },
    "server_b": {
      "image": "playhouse-server:v1.0.0",
      "status": "idle",
      "tenant_count": 0
    }
  }
  ```

**Changes**:
- [ ] Create `internal/api/releases.go` with release CRUD operations
- [ ] Create `internal/api/rollouts.go` with rollout management
- [ ] Update `internal/api/tenants.go` to add rollout status fields
- [ ] Add database migrations for rollout tracking tables
- [ ] Add request validation and error handling
- [ ] Add audit logging for all rollout actions

**Acceptance Criteria**:
- All endpoints functional and tested
- Proper authorization checks (admin-only)
- Audit trail for all operations
- Clear error messages

### 5B. Admin UI Extensions

**Location**: `playhouse-control-plane/web/`

Add UI for managing canary rollouts and progressive deployments.

**New Pages/Components**:

- [ ] **Deployments Dashboard** (`/admin-ui/deployments/`)
  - Current deployment status (which server versions active)
  - Tenant distribution (how many on each version)
  - Real-time metrics (error rates, latency)
  - Quick action buttons (Deploy, Rollback, Pause)

- [ ] **Release Management** (`/admin-ui/releases/`)
  - List all releases with status (pending, canary, in_progress, completed, rolled_back)
  - Create new release (upload image, set initial targets)
  - View release details and history
  - Start/pause/resume rollout

- [ ] **Canary Deployment Wizard**
  - Step 1: Select version to deploy
  - Step 2: Choose canary tenants (with autocomplete/search)
  - Step 3: Review and confirm
  - Step 4: Monitor deployment progress
  - Option to rollback at any time

- [ ] **Progressive Rollout UI**
  - Show current batch progress (e.g., "10% of 50 tenants")
  - List tenants in each batch (editable)
  - Start next batch button (with go/no-go confirmation)
  - Monitoring dashboard with key metrics
  - Pause/Resume/Rollback buttons at each step

- [ ] **Tenant Migration Control**
  - Search tenant and see current version
  - Manually move tenant to specific version (for support/edge cases)
  - Schedule migration for specific time
  - View migration history/audit log

**Changes**:
- [ ] Create `web/src/pages/Deployments.tsx` - Deployment status dashboard
- [ ] Create `web/src/pages/ReleaseManagement.tsx` - Release CRUD
- [ ] Create `web/src/components/CanaryWizard.tsx` - Canary deployment flow
- [ ] Create `web/src/components/RolloutMonitor.tsx` - Progress/metrics display
- [ ] Create `web/src/components/TenantMigration.tsx` - Tenant version control
- [ ] Add forms with validation, error handling, loading states
- [ ] Add real-time WebSocket connection for live metric updates
- [ ] Add confirmation dialogs for destructive actions (rollback, pause)
- [ ] Add audit log view showing all deployment actions

**Acceptance Criteria**:
- All UI components functional
- Forms validate input properly
- Real-time metric updates working
- Confirmation dialogs for critical actions
- Audit log captures all operations

### 6. Documentation

**Location**: `playhouse-docs/` (new section)

**Changes**:
- [ ] Add `PRODUCTION-DEPLOYMENT.md` overview
- [ ] Add `ROUTING-ARCHITECTURE.md` (how routing works)
- [ ] Add `DEPLOYMENT-POOLS.md` (pool lifecycle)
- [ ] Add `DATABASE-SCHEMA-VERSIONING.md` (tenant DB migrations)

## Testing in Phase 0

- [ ] Routing service resolves tenants to correct servers (unit tests)
- [ ] Nginx auth_request integration works (integration test)
- [ ] Static assets served from versioned paths (manual test)
- [ ] Docker-compose.prod.yml starts cleanly
- [ ] Registry schema migrations apply without errors

## Exit Criteria

- Routing infrastructure in place and tested
- Registry schema supports multi-version deployments
- Control-plane API endpoints for rollouts complete and tested
- Admin UI for managing deployments complete
- Nginx correctly routes based on tenant→server mapping
- Static asset versioning structure ready
- Deployment pools initialized
- Documentation complete
- All Phase 0 work items marked complete

## Risks & Mitigations

| Risk | Mitigation |
|------|-----------|
| Routing service becomes bottleneck | Cache heavily (60s TTL), consider distributed cache (Redis) later |
| Auth request fails, nginx falls back to error | Add fallback server (default to A) |
| Tenant not in routing table | Initialize new tenants with default pool on creation |
| Version mismatch between code + registry | Validation script checks consistency |

## Next Phase

Phase 1 focuses on testing this routing + versioning infrastructure with dual servers before any production traffic.
