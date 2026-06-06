# Phase 1 - Task Breakdown

This phase is split into 8 focused testing tasks. Complete them sequentially in staging.

## Phase 1 Overview
- **Goal**: Validate blue-green infrastructure with dual servers in staging
- **Duration**: 1-2 weeks
- **Repos**: playhouse-server, playhouse-web, playhouse-cicd, playhouse-control-plane
- **Prerequisites**: Phase 0 complete
- **End State**: All infrastructure tested, procedures validated, team trained

## Tasks

### Task 1.1: Schema Backward Compatibility Analysis
- **Goal**: Verify v0.x and v1.0 servers can coexist with shared schema
- **Repos**: playhouse-server
- **Work**:
  - [ ] Audit all v1 migrations: check for no deletions
  - [ ] List new columns added in v1
  - [ ] Verify all new columns have DEFAULT values
  - [ ] Review ORM code for type changes
  - [ ] Document breaking changes (if any) and mitigation
  - [ ] Create compatibility matrix (v0 with v1 schema, v1 with v0 schema)
  - **Acceptance**: No breaking changes found or documented with mitigation
- **Duration**: 1-2 days
- **Next**: Task 1.2

---

### Task 1.2: Web Asset Versioning Setup
- **Goal**: Build versioned asset bundles for v0.x.x and v1.0.0
- **Repos**: playhouse-web
- **Work**:
  - [ ] Create `scripts/build-versioned.sh` script
  - [ ] Update Dockerfile to build assets for specific version
  - [ ] Create `public/version-meta.json` template
  - [ ] Update web entry point to read version metadata
  - [ ] Test build produces `dist/v0.x.x/` and `dist/v1.0.0/` directories
  - [ ] Verify web app detects version at runtime
  - **Acceptance**: Both version asset directories created, version detected at runtime
- **Duration**: 1 day
- **Next**: Task 1.3

---

### Task 1.3: Build Release Images (v0.x.x & v1.0.0)
- **Goal**: Build Docker images for both versions
- **Repos**: playhouse-cicd
- **Work**:
  - [ ] Create `scripts/build-release-images.sh`
  - [ ] Build playhouse-server:v0.x.x
  - [ ] Build playhouse-server:v1.0.0
  - [ ] Build playhouse-web:v0.x.x
  - [ ] Build playhouse-web:v1.0.0
  - [ ] Build playhouse-control-plane:v0.x.x
  - [ ] Build playhouse-control-plane:v1.0.0
  - [ ] Tag all images correctly
  - [ ] Verify each image starts without errors (basic health check)
  - **Acceptance**: All images tagged and verified
- **Duration**: 1 day
- **Next**: Task 1.4

---

### Task 1.4: Staging Dual-Server Deployment
- **Goal**: Deploy both servers in staging environment
- **Repos**: playhouse-cicd
- **Work**:
  - [ ] Create `docker-compose.staging-dual-servers.yml`
  - [ ] Configure server-a (v0.x.x) and server-b (v1.0.0)
  - [ ] Add routing-service for dynamic routing
  - [ ] Add nginx with auth request integration
  - [ ] Deploy to staging: `docker-compose up`
  - [ ] Verify all services healthy
  - [ ] Create 3 test tenants (A, B, C)
  - [ ] Initialize deployment pools in registry
  - **Acceptance**: Full stack running, all services healthy, test tenants created
- **Duration**: 1 day
- **Next**: Task 1.5

---

### Task 1.5: Routing Integration Tests
- **Goal**: Test tenant routing correctness
- **Repos**: playhouse-cicd
- **Work**:
  - [ ] Test basic routing: tenant → server-a
  - [ ] Test routing change: tenant → server-b after update
  - [ ] Test static assets routing: correct version served
  - [ ] Test X-Version header: set correctly per tenant
  - [ ] Test cache TTL: updates within 60s
  - [ ] Test fallback: works if routing service down
  - [ ] Test cross-tenant isolation: no bleed
  - [ ] Create `tests/routing/test-routing.sh` script
  - **Acceptance**: All routing tests pass
- **Duration**: 1-2 days
- **Next**: Task 1.6

---

### Task 1.6: Database Migration Testing
- **Goal**: Test tenant DB migration procedures (v0 → v1 → v0)
- **Repos**: playhouse-server
- **Work**:
  - [ ] Create migration management command: `migrate_tenant_schema.py`
  - [ ] Create rollback command: `migrate_tenant_schema_rollback.py`
  - [ ] Create test fixtures with sample data
  - [ ] Test v0 → v1 migration:
    - [ ] Migrate test tenant
    - [ ] Verify all data intact
    - [ ] Verify v1 columns have correct defaults
  - [ ] Test v1 → v0 rollback:
    - [ ] Rollback migration
    - [ ] Verify data integrity
    - [ ] Verify v0 queries still work
  - [ ] Test full cycle: v0 → v1 → v0
  - [ ] Document migration procedure
  - **Acceptance**: All migrations succeed, data integrity maintained, rollback works
- **Duration**: 2 days
- **Next**: Task 1.7

---

### Task 1.7: Backward Compatibility Tests
- **Goal**: Ensure v0 and v1 servers work together
- **Repos**: playhouse-server, playhouse-web
- **Work**:
  - [ ] Create `tests/test_backward_compat.py` (server tests):
    - [ ] v0 code with v1 schema
    - [ ] v1 code with v0 schema
    - [ ] API responses cross-version
  - [ ] Create `tests/backward-compat.test.js` (web tests):
    - [ ] Old web with new server API
    - [ ] New web with old server API
    - [ ] Feature flags/entitlements cross-version
  - [ ] Run full backward compat test suite
  - [ ] Document any incompatibilities
  - **Acceptance**: All backward compat tests pass
- **Duration**: 1-2 days
- **Next**: Task 1.8

---

### Task 1.8: Rollback Simulation & Documentation
- **Goal**: Test rollback procedure end-to-end
- **Repos**: playhouse-cicd
- **Work**:
  - [ ] Create rollback test scenario:
    - [ ] Route 2 test tenants to server-b
    - [ ] Simulate error
    - [ ] Execute rollback
    - [ ] Verify data integrity
  - [ ] Measure rollback time (target: <5 min)
  - [ ] Write deployment runbooks:
    - [ ] `CANARY-DEPLOYMENT.md`
    - [ ] `PROGRESSIVE-ROLLOUT.md`
    - [ ] `DATABASE-MIGRATION.md`
    - [ ] `ROLLBACK.md`
    - [ ] `TROUBLESHOOTING.md`
  - [ ] Train team on procedures
  - [ ] Conduct dry-run deployment with team
  - **Acceptance**: Rollback <5 min, runbooks complete, team trained
- **Duration**: 2-3 days
- **Next**: Phase 2

---

## Phase 1 Exit Criteria

- ✓ Schema backward compatibility verified
- ✓ Web asset versioning working
- ✓ Both v0.x.x and v1.0.0 images built
- ✓ Dual-server staging deployment working
- ✓ All routing tests passing
- ✓ Database migrations tested (forward and backward)
- ✓ Backward compatibility tests passing
- ✓ Rollback tested and working (<5 min)
- ✓ All runbooks written
- ✓ Team trained and confident

---

## Timeline

| Task | Duration | Total |
|------|----------|-------|
| 1.1 | 1-2 days | 1-2 days |
| 1.2 | 1 day | 2-3 days |
| 1.3 | 1 day | 3-4 days |
| 1.4 | 1 day | 4-5 days |
| 1.5 | 1-2 days | 5-7 days |
| 1.6 | 2 days | 7-9 days |
| 1.7 | 1-2 days | 8-11 days |
| 1.8 | 2-3 days | 10-14 days |
| **Total** | | **1-2 weeks** |

---

## Notes

- All work happens in staging, NO production
- Run tests frequently throughout tasks
- Document all findings and issues
- Update runbooks based on learnings
- Each task has clear acceptance criteria
