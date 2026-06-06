# Phase 0 - Task Breakdown

This phase is split into 8 focused tasks. Complete them sequentially.

## Phase 0 Overview
- **Goal**: Build routing infrastructure and control-plane orchestration layer
- **Duration**: 2-3 weeks
- **Repos**: playhouse-control-plane, playhouse-cicd, playhouse-docs
- **End State**: Routing service, registry schema, CP-API, Admin UI, and nginx all working

## Tasks

### Task 0.1: Registry Schema Design & Migrations
- **Goal**: Design and implement registry database schema for multi-version deployments
- **Repos**: playhouse-control-plane (migrations/)
- **Work**:
  - [ ] Create `tenant_releases` table (tenant_id → target_version mapping)
  - [ ] Create `deployment_pools` table (version → server instance)
  - [ ] Create `rollout_batches` table (batch progress tracking)
  - [ ] Create `migration_jobs` table (job status tracking)
  - [ ] Add appropriate indexes for quick lookups
  - [ ] Write migration file
  - [ ] Test migration applies without errors
  - **Acceptance**: All tables created, indexes present, migration idempotent
- **Duration**: 2-3 days
- **Next**: Task 0.2

---

### Task 0.2: Routing Service Implementation
- **Goal**: Build Go service that queries registry and provides tenant→server routing
- **Repos**: playhouse-control-plane (cmd/routing-service/)
- **Work**:
  - [ ] Create `cmd/routing-service/main.go`
  - [ ] Implement `GET /route?tenant=X` endpoint
  - [ ] Implement `GET /health` endpoint
  - [ ] Add in-memory cache with 60s TTL
  - [ ] Query registry_db for routing decisions
  - [ ] Add error handling and logging
  - [ ] Create Dockerfile for routing-service
  - [ ] Add metrics endpoint for monitoring
  - **Acceptance**: Service starts, responds to requests, caches work, <2s startup
- **Duration**: 2-3 days
- **Next**: Task 0.3

---

### Task 0.3: Control-Plane API Endpoints
- **Goal**: Add REST endpoints for managing releases, rollouts, and deployments
- **Repos**: playhouse-control-plane (internal/api/)
- **Work**:
  - [ ] Create `internal/api/releases.go` with CRUD endpoints
  - [ ] Create `internal/api/rollouts.go` with rollout management
  - [ ] Create `internal/api/deployments.go` with status queries
  - [ ] Implement POST `/api/v1/releases`
  - [ ] Implement GET `/api/v1/releases` and `/api/v1/releases/{version}`
  - [ ] Implement PATCH `/api/v1/releases/{version}`
  - [ ] Implement POST `/api/v1/releases/{version}/rollouts`
  - [ ] Implement POST `/api/v1/releases/{version}/rollback`
  - [ ] Add request validation and authorization checks
  - [ ] Add audit logging for all operations
  - **Acceptance**: All endpoints functional, tested, documented
- **Duration**: 3-4 days
- **Next**: Task 0.4

---

### Task 0.4: Admin UI - Deployment Dashboard & Release Management
- **Goal**: Build UI for viewing and managing deployments
- **Repos**: playhouse-control-plane (web/)
- **Work**:
  - [ ] Create `/admin-ui/deployments/` page
    - [ ] Show current server versions and tenant distribution
    - [ ] Real-time metrics (error rates, latency, uptime)
    - [ ] Quick action buttons (Deploy, Pause, Rollback)
  - [ ] Create `/admin-ui/releases/` page
    - [ ] List all releases with status
    - [ ] View release details
    - [ ] Create new release form
  - [ ] Create Canary Deployment Wizard component
    - [ ] Step 1: Select version
    - [ ] Step 2: Select canary tenants
    - [ ] Step 3: Review and confirm
  - [ ] Add real-time WebSocket for metric updates
  - [ ] Add confirmation dialogs for destructive actions
  - **Acceptance**: UI functional, forms work, real-time updates flowing
- **Duration**: 3-5 days
- **Next**: Task 0.5

---

### Task 0.5: Admin UI - Tenant Migration & Monitoring
- **Goal**: Build UI for tenant migration control and rollout monitoring
- **Repos**: playhouse-control-plane (web/)
- **Work**:
  - [ ] Create Progressive Rollout Monitor component
    - [ ] Show batch progress (e.g., "10% of 50 tenants")
    - [ ] Show individual tenant migration status
    - [ ] Display key metrics per batch
    - [ ] Start/pause/resume controls
  - [ ] Create Tenant Migration Control component
    - [ ] Search and display tenant status
    - [ ] Manual tenant version assignment (for support)
    - [ ] View migration history
  - [ ] Create Rollout Audit Log view
    - [ ] Show all deployment actions
    - [ ] Filter by date, user, release
  - [ ] Add alert displays and error messages
  - **Acceptance**: All components functional, responsive, real-time updates
- **Duration**: 2-3 days
- **Next**: Task 0.6

---

### Task 0.6: Nginx Configuration - Dynamic Routing
- **Goal**: Configure nginx to route requests based on tenant using auth_request
- **Repos**: playhouse-cicd (nginx/)
- **Work**:
  - [ ] Update `nginx/nginx.conf` to enable auth_request module
  - [ ] Create `nginx/routing-auth.conf` with auth request logic
  - [ ] Update `nginx/proxy-locations.conf`:
    - [ ] Add location blocks for `/assets/vX.X.X/`
    - [ ] Add dynamic upstream selection
    - [ ] Set X-Version header based on routing decision
  - [ ] Add error handling (fallback to default version)
  - [ ] Test routing for multiple tenants
  - [ ] Verify static assets served from correct path
  - **Acceptance**: All requests routed correctly, fallback works, assets served correctly
- **Duration**: 2 days
- **Next**: Task 0.7

---

### Task 0.7: Docker Compose Production Setup
- **Goal**: Create production-grade docker-compose with all services
- **Repos**: playhouse-cicd
- **Work**:
  - [ ] Create `docker-compose.prod.yml` with:
    - [ ] routing-service (port 9000, internal)
    - [ ] registry-db (postgres)
    - [ ] registry-migrate (job)
    - [ ] cp-api (port 8080, internal)
    - [ ] cp-controller (internal)
    - [ ] playhouse-db (postgres)
    - [ ] pgbouncer (port 6432, internal)
    - [ ] server-a (v0.x.x, port 8000)
    - [ ] server-b (v1.0.0, port 8001, initially disabled)
    - [ ] web (nginx)
    - [ ] admin-web
    - [ ] nginx (port 80/443)
  - [ ] All services have health checks
  - [ ] All services have restart policies
  - [ ] Resource limits defined
  - [ ] Validate with `docker-compose config`
  - **Acceptance**: Compose file valid, all services include health checks
- **Duration**: 1-2 days
- **Next**: Task 0.8

---

### Task 0.8: Documentation & Integration Testing
- **Goal**: Document architecture and integration, test Phase 0 end-to-end
- **Repos**: playhouse-docs, playhouse-cicd
- **Work**:
  - [ ] Write `PRODUCTION-DEPLOYMENT.md` (overview)
  - [ ] Write `ROUTING-ARCHITECTURE.md` (how routing works)
  - [ ] Write `DEPLOYMENT-POOLS.md` (pool lifecycle)
  - [ ] Write `DATABASE-SCHEMA-VERSIONING.md` (schema evolution)
  - [ ] Create basic integration test:
    - [ ] Start docker-compose stack
    - [ ] Test routing service responds
    - [ ] Test CP-API health check
    - [ ] Test nginx responds
    - [ ] Stop services cleanly
  - [ ] Document any setup/troubleshooting steps
  - **Acceptance**: All docs complete, integration test passes
- **Duration**: 2 days
- **Next**: Phase 1

---

## Phase 0 Exit Criteria

- ✓ Registry schema with 4 tables created
- ✓ Routing service running and responding to requests
- ✓ CP-API endpoints implemented and tested
- ✓ Admin UI complete with deployment dashboard
- ✓ Nginx dynamic routing working
- ✓ docker-compose.prod.yml valid
- ✓ All documentation complete
- ✓ Integration test passing

---

## Notes

- Each task is designed to be worked on by 1-2 people
- Tasks can run in parallel after initial schema work (0.1)
- Test frequently throughout each task
- Document any issues or learnings
