# Production Deployment - Phase 1: Dual Server Setup & Testing

## Goal

Deploy and validate blue-green infrastructure with two server versions running in parallel. Test routing correctness, validate database migration procedures, and rehearse all rollback scenarios in staging environment before touching production.

## Repos

- `playhouse-server` (backward-compatible schema changes)
- `playhouse-web` (dual static asset builds)
- `playhouse-cicd` (deployment scripts, test scenarios)
- `playhouse-control-plane` (registry testing, routing service validation)

## Duration

1-2 weeks in staging environment

## Overview

This phase:
1. Ensures schema backward compatibility
2. Builds versioned web assets for v0.x.x and v1.0.0
3. Deploys both servers in staging with routing
4. Tests routing between servers
5. Tests database migration procedures
6. Tests rollback scenarios
7. Validates rollout scripts
8. Trains team on deployment procedures

## Prerequisites (From Phase 0)

- ✓ Routing service implemented and tested
- ✓ Registry schema supports tenant_releases and deployment_pools
- ✓ Nginx configured with auth_request integration
- ✓ Docker-compose.prod.yml ready
- ✓ All Phase 0 documentation complete

## Work Items

### 1. Ensure Schema Backward Compatibility

**Repo:** playhouse-server

**What:** Verify v0.x can read v1 schema and v1.0 can read v0 schema without errors.

**Changes:**
- [ ] Audit all v1 migrations: check no column deletions, only additions with defaults
- [ ] Review each migration file for breaking changes:
  - [ ] No `ALTER TABLE ... DROP COLUMN`
  - [ ] All new columns have `DEFAULT` values or `null`
  - [ ] No type changes on existing columns
- [ ] Add backward compatibility tests in `tests/test_backward_compat.py`:
  ```python
  # Test v0.x ORM code works with v1 schema
  # (new columns present but unused by v0 code)
  
  # Test v1.0 ORM code works with v0 schema
  # (missing new columns, defaults used)
  ```
- [ ] Document schema evolution in `docs/SCHEMA_VERSIONING.md`
  - Which migrations added new columns
  - Why each change is safe
  - Any known compatibility issues

**Acceptance Criteria:**
- All backward compat tests pass
- No API breaking changes
- Rollback path documented

---

### 2. Build Versioned Web Assets

**Repo:** playhouse-web

**What:** Create separate asset bundles for v0.x.x and v1.0.0, each in its own path.

**Changes:**
- [ ] Create `scripts/build-versioned.sh`:
  ```bash
  #!/bin/bash
  VERSION=${1:-local}
  npm run build
  mkdir -p dist/web/${VERSION}
  cp -r build/* dist/web/${VERSION}/
  echo "Built assets for version ${VERSION}"
  ```

- [ ] Update `Dockerfile`:
  ```dockerfile
  # Build stage
  FROM node:18 AS builder
  WORKDIR /app
  COPY . .
  ARG PILCHE_TAG=local
  RUN npm run build
  RUN mkdir -p /assets/${PILCHE_TAG} && cp -r build/* /assets/${PILCHE_TAG}/
  
  # Serve stage
  FROM nginx:1.25-alpine
  COPY --from=builder /assets /usr/share/nginx/html/assets
  COPY nginx.conf /etc/nginx/nginx.conf
  ```

- [ ] Create `public/version-meta.json`:
  ```json
  {
    "version": "${PILCHE_TAG}",
    "buildDate": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
    "assetPath": "/assets/${PILCHE_TAG}",
    "apiVersion": "v1"
  }
  ```

- [ ] Update `src/App.js` or entry point to read version meta:
  ```javascript
  const versionMeta = await fetch('/version-meta.json').then(r => r.json());
  console.log('App version:', versionMeta.version);
  // Use versionMeta.assetPath for asset loading
  ```

- [ ] Update `FlagsContext.js` to use versioned asset paths

**Acceptance Criteria:**
- Build produces `dist/web/v0.x.x/` and `dist/web/v1.0.0/` directories
- Docker image contains both versions under `/assets/`
- Web app correctly identifies version at runtime
- Assets load from correct path

---

### 3. Build v0.x.x and v1.0.0 Docker Images

**Repo:** playhouse-cicd

**What:** Build and tag all components for both versions.

**Changes:**
- [ ] Create `scripts/build-release-images.sh`:
  ```bash
  #!/bin/bash
  set -e
  
  TAG=${1:-local}
  REGISTRY=${2:-docker.io}
  
  echo "Building Pilche v${TAG}..."
  
  # Build playhouse-server
  docker build \
    -t ${REGISTRY}/playhouse-server:${TAG} \
    --build-arg PILCHE_TAG=${TAG} \
    playhouse-server/
  
  # Build playhouse-web
  docker build \
    -t ${REGISTRY}/playhouse-web:${TAG} \
    --build-arg PILCHE_TAG=${TAG} \
    playhouse-web/
  
  # Build playhouse-control-plane
  docker build \
    -t ${REGISTRY}/playhouse-control-plane:${TAG} \
    --build-arg PILCHE_TAG=${TAG} \
    playhouse-control-plane/
  
  # Build routing service
  docker build \
    -t ${REGISTRY}/playhouse-routing-service:${TAG} \
    -f playhouse-control-plane/Dockerfile.routing \
    playhouse-control-plane/
  
  echo "✓ Built images for v${TAG}"
  ```

- [ ] Build both v0.x.x and v1.0.0 tags
- [ ] Test basic image health check (service starts without errors)

**Acceptance Criteria:**
- Both v0.x.x and v1.0.0 images build successfully
- Images run without startup errors
- Health checks pass

---

### 4. Staging Deployment: Dual Servers

**Repo:** playhouse-cicd

**What:** Deploy full stack with both servers to staging environment.

**Changes:**
- [ ] Create `docker-compose.staging-dual-servers.yml`:
  - registry-db
  - registry-migrate
  - cp-api
  - cp-controller
  - playhouse-db
  - pgbouncer
  - routing-service (NEW)
  - server-a (v0.x.x)
  - server-b (v1.0.0)
  - web-v0 (v0.x.x assets)
  - web-v1 (v1.0.0 assets)
  - nginx (with auth_request routing)

- [ ] Deploy to staging:
  ```bash
  export PILCHE_TAG_A=v0.x.x
  export PILCHE_TAG_B=v1.0.0
  docker-compose -f docker-compose.staging-dual-servers.yml up -d
  ```

- [ ] Create test tenants (A, B, C, D)
- [ ] Initialize deployment pools (routing table)
- [ ] Verify all services healthy

**Acceptance Criteria:**
- All services start successfully
- Health checks pass for all services
- Routing service accessible at localhost:9000
- Registry initialized with test tenants

---

### 5. Routing Tests

**Repo:** playhouse-cicd

**What:** Test matrix for tenant → server routing correctness.

**Test File:** `tests/routing/test_routing.sh`

**Test Cases:**
- [ ] **Basic routing**: Tenant A requests route endpoint, gets server-a
  ```bash
  curl http://routing-service:9000/route?tenant=tenant-a
  # Expected: {"target_server": "server-a:8000", "version": "v0.x.x"}
  ```

- [ ] **Routing change**: Update tenant A routing in registry, new requests go to server-b
  ```bash
  curl -X PATCH http://cp-api:8080/tenants/tenant-a \
    -d '{"target_version": "v1.0.0"}'
  
  # After cache TTL (60s):
  curl http://routing-service:9000/route?tenant=tenant-a
  # Expected: {"target_server": "server-b:8000", "version": "v1.0.0"}
  ```

- [ ] **Static assets routing**: Correct asset version served based on tenant
  ```bash
  curl http://localhost/assets/app.js -H "Host: tenant-a.localhost"
  # Serves from /assets/v0.x.x/app.js if tenant-a → v0.x.x
  # Serves from /assets/v1.0.0/app.js if tenant-a → v1.0.0
  ```

- [ ] **X-Version header**: Header set correctly for each tenant
  ```bash
  curl -i http://localhost/ -H "Host: tenant-a.localhost" \
    | grep X-Version
  # Expected: X-Version: v0.x.x (if routing to v0.x.x)
  ```

- [ ] **Routing cache TTL**: Cache invalidation works after 60s
  ```bash
  # Request 1: Gets v0.x.x
  curl http://routing-service:9000/route?tenant=tenant-a
  
  # Update routing
  curl -X PATCH http://cp-api:8080/tenants/tenant-a \
    -d '{"target_version": "v1.0.0"}'
  
  # Request 2 (within 60s): Still v0.x.x (cached)
  curl http://routing-service:9000/route?tenant=tenant-a
  
  # Wait 60s, Request 3: Now v1.0.0 (cache expired)
  curl http://routing-service:9000/route?tenant=tenant-a
  ```

- [ ] **Cross-tenant isolation**: Tenant A routing doesn't affect tenant B
  ```bash
  # Route tenant-a to v1.0.0
  # Tenant B should still route to v0.x.x (or whatever assigned)
  # Verify no cross-tenant leakage
  ```

- [ ] **Missing tenant handling**: Request for non-existent tenant returns error or default
  ```bash
  curl http://routing-service:9000/route?tenant=unknown-tenant
  # Expected: 404 or default pool
  ```

**Acceptance Criteria:**
- All routing tests pass
- Routing changes reflected within 60s
- No cross-tenant bleed
- Cache working as expected

---

### 6. Database Migration Testing

**Repo:** playhouse-server

**What:** Test tenant DB migration procedures (v0 → v1 → v0).

**Management Command:** `management/commands/migrate_tenant_schema.py`

**Changes:**
- [ ] Create migration management command:
  ```python
  # management/commands/migrate_tenant_schema.py
  class Command(BaseCommand):
      def add_arguments(self, parser):
          parser.add_argument('tenant_id')
          parser.add_argument('--target-version', default='latest')
          parser.add_argument('--rollback', action='store_true')
      
      def handle(self, tenant_id, target_version, rollback, **options):
          # Get tenant DB config from registry
          # Run migrations via Django migration framework
          # Log progress
          # Validate post-migration
  ```

- [ ] Create rollback command: `migrate_tenant_schema_rollback.py`

- [ ] Create test fixtures in `tests/fixtures/tenant_migration_data.json`
  - Sample data that exercises all DB tables
  - Data that will change/be affected by v1 schema

- [ ] Add migration test suite in `tests/test_migrations.py`:
  ```python
  def test_migrate_v0_to_v1_preserves_data():
      # Create tenant, populate with v0 schema data
      # Run migration to v1
      # Verify all data intact
      # Verify new v1 columns have correct defaults
  
  def test_rollback_v1_to_v0_works():
      # Start with v1 schema and data
      # Rollback to v0
      # Verify all v0 queries still work
  
  def test_migration_cycle_v0_v1_v0():
      # Start v0, migrate to v1, rollback to v0
      # Verify no data loss in round-trip
  ```

**Test Procedure:**
```bash
# Day 2 of Phase 1
1. Spin up staging stack with server-a (v0.x.x)
2. Create test tenants with sample data
3. Run: python manage.py migrate_tenant_schema tenant-test-1 --target-version=v1
4. Verify migration success
5. Query test data from both servers (server-a on v0, server-b on v1)
6. Run rollback: python manage.py migrate_tenant_schema_rollback tenant-test-1
7. Verify rollback success and data integrity
```

**Acceptance Criteria:**
- Migration v0 → v1 succeeds without errors
- No data loss during migration
- Rollback v1 → v0 works cleanly
- Pre-migration and post-migration data consistent

---

### 7. Backward Compatibility Tests

**Repos:** playhouse-server, playhouse-web

**What:** Ensure cross-version compatibility.

**Server Tests:** `tests/test_backward_compat.py`

- [ ] Test v0.x code queries work with v1 schema
  ```python
  # New columns in v1 have defaults
  # v0 code ignores new columns
  # All v0 queries still work
  ```

- [ ] Test v1.0 code works with v0 schema
  ```python
  # v1 code gracefully handles missing columns
  # Uses sensible defaults
  # No crashes or 500 errors
  ```

- [ ] Test API responses compatible
  ```python
  # v0 client can parse v1 API responses
  # v1 client can parse v0 API responses
  # Feature flags work cross-version
  # Entitlements enforced correctly
  ```

**Web Tests:** `tests/backward-compat.test.js`

- [ ] Test old web app works with new server API
  ```javascript
  // v0 web queries new endpoints
  // Gracefully handles new response fields
  // Feature flags still work
  ```

- [ ] Test new web app works with old server API
  ```javascript
  // v1 web queries old endpoints
  // Missing fields handled
  // UI degrades gracefully
  // No 500 errors
  ```

**Acceptance Criteria:**
- All backward compatibility tests pass
- No API breaking changes
- Graceful feature degradation
- Cross-version integration works

---

### 8. Rollback Simulation

**Repo:** playhouse-cicd

**What:** Test full rollback scenario (tenant → v1.0 → back to v0.x.x).

**Test File:** `tests/rollback/rollback-scenario.sh`

**Scenario:**
```bash
# Setup
1. Deploy staging dual-servers
2. Create 2 test tenants with data
3. Migrate test tenants to v1 schema
4. Route tenants to server-b (v1.0.0)

# Simulate failure
5. Inject error on server-b (e.g., bad entitlements config)
6. Observe tenant errors/failed requests

# Rollback
7. Re-route tenants back to server-a (v0.x.x)
8. Verify services recover immediately
9. Verify all data intact, no data loss
10. Verify entitlements work again

# Validate
11. Check both server error logs
12. Verify DB consistency
13. Verify no cross-tenant contamination
```

**Success Metrics:**
- [ ] Rollback completes in <5 minutes
- [ ] All tenant data intact
- [ ] Zero data loss/corruption
- [ ] Tenants working on server-a after rollback
- [ ] No errors in logs

---

### 9. Deployment Scripts

**Repo:** playhouse-cicd/scripts/

**What:** Create reusable operational scripts.

**Changes:**
- [ ] `deploy-server-b.sh`: Start server-b with v1.0.0 image
  ```bash
  # Usage: deploy-server-b.sh v1.0.0
  # - Pulls v1.0.0 image
  # - Starts server-b container
  # - Verifies health check
  # - Logs output
  ```

- [ ] `route-tenants-to-version.sh`: Update registry routing for batch
  ```bash
  # Usage: route-tenants-to-version.sh tenant-a tenant-b v1.0.0
  # - Updates tenant_releases in registry
  # - Sets status to in_progress
  # - Waits for routing cache TTL
  # - Validates routing change
  ```

- [ ] `migrate-tenant-db.sh`: Run schema migration for tenant
  ```bash
  # Usage: migrate-tenant-db.sh tenant-a v1
  # - Backs up tenant DB
  # - Runs Django migrations
  # - Validates migration success
  # - Logs progress
  ```

- [ ] `check-routing-status.sh`: Show current routing state
  ```bash
  # Usage: check-routing-status.sh
  # - Query registry for all tenant_releases
  # - Show tenant → version mapping
  # - Show deployment pool status
  # - Show batch rollout progress
  ```

- [ ] `rollback-routing.sh`: Revert routing to previous version
  ```bash
  # Usage: rollback-routing.sh tenant-a
  # - Query previous routing from registry history
  # - Update tenant_releases
  # - Validate rollback
  ```

- [ ] `validate-deployment.sh`: Health checks for all services
  ```bash
  # Usage: validate-deployment.sh
  # - Check all containers running
  # - Check health endpoints responding
  # - Check routing service latency
  # - Check DB connections
  # - Report status
  ```

**Acceptance Criteria:**
- All scripts are executable and idempotent
- Error handling for common issues
- Logging for audit trail
- Scripts work in both staging and production

---

### 10. Operational Runbooks & Documentation

**Repo:** playhouse-docs/

**What:** Document all deployment procedures for team.

**Files to create:**
- [ ] `CANARY-DEPLOYMENT.md`: Step-by-step canary rollout
  - Pre-flight checks
  - Selecting canary tenants
  - Pre-migrating tenant DBs
  - Deploying server-b
  - Routing tenants
  - Monitoring checklist
  - Go/no-go criteria
  - Rollback procedure if needed

- [ ] `PROGRESSIVE-ROLLOUT.md`: Gradual tenant migration
  - Batch strategy (10%, 25%, 50%, 100%)
  - Timing between batches
  - Health checks between batches
  - Monitoring dashboard interpretation
  - Stopping/pausing rollout
  - Rollback from any batch

- [ ] `DATABASE-MIGRATION.md`: Tenant DB migration procedures
  - When to migrate
  - Pre-migration backup procedure
  - Running migration command
  - Validating post-migration
  - Rollback migration
  - Troubleshooting common issues

- [ ] `ROLLBACK.md`: Emergency rollback procedures
  - Detecting when rollback needed
  - Quick rollback steps (re-route tenants)
  - Verifying rollback successful
  - Post-rollback analysis
  - Prevention for next time

- [ ] `TROUBLESHOOTING.md`: Common issues and fixes
  - Routing service not responding
  - Server-b health check failing
  - Tenant DB migration failing
  - Cross-tenant data bleed
  - DNS/SSL certificate issues
  - Connection pool exhaustion

- [ ] `MONITORING.md`: Metrics to track during rollout
  - Routing latency (p50, p95, p99)
  - Server error rates (per version)
  - Feature denial rates (observability mode)
  - API response times (per endpoint)
  - Database query latency
  - Connection pool saturation
  - Alert thresholds
  - Dashboard setup

- [ ] `TEAM-READINESS.md`: Training checklist for team
  - Who needs to know what
  - Training sessions
  - Procedure walkthroughs
  - Dry-run deployments
  - On-call setup

**Acceptance Criteria:**
- All runbooks complete and detailed
- Step-by-step procedures (no ambiguity)
- Troubleshooting guides for common issues
- Team has read and signed off on procedures

---

## Testing Sequence (Day by Day)

### Day 1-2: Preparation & Routing Validation
- Deploy staging dual-servers
- Verify all services healthy
- Run routing test suite
- Document any routing issues
- Fix and re-test

### Day 3: Database Migrations
- Test v0 → v1 migration for test tenant
- Verify backward compatibility
- Test rollback v1 → v0
- Repeat cycle 3x to verify consistency

### Day 4: Backward Compatibility
- Run full backward compatibility test suite
- Test v0 server with v1 schema
- Test v1 server with v0 schema
- Test API cross-version
- Document any incompatibilities

### Day 5: Canary Simulation
- Route 2 test tenants to server-b (v1.0.0)
- Run for 24h with monitoring
- Collect baseline metrics
- Validate entitlements enforced
- Check observability mode working

### Day 6: Rollback Scenario
- Inject failure on server-b
- Execute rollback procedure
- Verify all data intact
- Verify zero downtime
- Document rollback time

### Day 7: Full Progressive Rollout Simulation
- Route 25% to server-b → wait 12h → 50% → 85% → 100%
- Monitor at each batch
- Validate go/no-go criteria
- Document metrics

### Day 8-10: Performance & Load Testing
- Run load testing (simultaneous requests to both servers)
- Test routing latency under load
- Test database connection pool saturation
- Identify bottlenecks
- Document resource requirements

### Day 11-14: Team Training & Dry-Runs
- Run deployment procedure dry-run with team
- Each team member practices critical steps
- Troubleshoot together
- Update runbooks based on learnings
- Final sign-off from team

---

## Exit Criteria

- ✓ Dual server infrastructure working in staging
- ✓ Routing between servers accurate and reliable (100% correct)
- ✓ Database migrations successful in both directions (v0 → v1 → v0)
- ✓ Backward compatibility verified (all tests pass)
- ✓ Rollback tested and working (<5 min, zero data loss)
- ✓ Deployment scripts functional and tested
- ✓ Runbooks complete and tested (dry-run with team)
- ✓ Team trained and confident in deployment procedures
- ✓ Phase 1 sign-off from tech lead
- ✓ Go/no-go meeting: Ready to proceed to Phase 2

---

## Risks & Mitigations

| Risk | Mitigation |
|------|-----------|
| Schema migration causes data loss | Test with production-scale data; implement migration rollback; validate post-migration |
| Version mismatch causes API errors | Comprehensive backward compat tests; API versioning if needed; graceful error handling |
| Routing service failure mid-test | Cache at nginx level; implement fallback pool; circuit breaker pattern |
| DNS/SSL certificate issues | Test with production domain in staging; wildcard cert validation |
| Tenant DB connection pool exhaustion | Monitor pool during tests; implement circuit breaker; set reasonable limits |
| Cross-tenant data contamination | Isolation tests; audit logs; data validation queries post-migration |
| Rollback doesn't work cleanly | Test rollback multiple times; verify data consistency after rollback |

---

## Success Metrics

| Metric | Target |
|--------|--------|
| Routing test pass rate | 100% |
| DB migration success rate | 100% |
| Migration time (per tenant) | <30 min |
| Rollback time | <5 min |
| Backward compatibility test pass rate | 100% |
| Zero data loss in any test | 100% |
| Team procedure dry-run success | 100% |

---

## Next Phase

Phase 2 focuses on production preparation:
- Provision production servers
- Set up backups and monitoring
- Execute first canary deployment (5-10% of real tenants)

**Phase 2 can only start after Phase 1 exit criteria all met.**

---

## Notes

- All tests run in staging, NOT production
- No production traffic during Phase 1
- Document all findings and learnings
- Update runbooks based on Phase 1 experience
- Celebrate successful completion before moving to Phase 2
