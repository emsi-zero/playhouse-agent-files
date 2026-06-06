# Production Deployment - Phase 4: Retire Old Version

## Goal

Upgrade server-a from v0.x.x to v1.0.0, re-route all tenants back to server-a, and retire the old version from production.

## Repos

- `playhouse-cicd` (server upgrade, docker-compose updates)

## Duration

1 week

## Overview

This phase:
1. Migrates all tenants back to server-a (now v1.0.0)
2. Verifies stability with both servers on same version
3. Decommissions server-b and v0.x.x code
4. Documents final deployment
5. Stabilizes for production

## Prerequisites (From Phase 3)

- ✓ 100% of tenants on v1.0.0 (server-b)
- ✓ Phase 3 progressive rollout completed successfully
- ✓ All batches stable for 24-48h each
- ✓ Zero customer issues reported
- ✓ Server-b proven production-ready

## Architecture After Phase 4

**Before (End of Phase 3):**
```
server-a (v0.x.x): idle
server-b (v1.0.0): 100% of tenants
```

**After Phase 4:**
```
server-a (v1.0.0): 100% of tenants (all re-routed from server-b)
server-b (v1.0.0): idle (available for v2.0.0 in future)
```

---

## Work Items

### 1. Pre-Upgrade Planning

**What:** Plan upgrade strategy and prepare for transition.

**Changes:**
- [ ] **Decide upgrade strategy** (choose one):

  **Option A: Blue-Green Upgrade (Recommended)**
  - Server-a stays running (v0.x.x)
  - Upgrade server-a offline → (v1.0.0)
  - Switch all routing from server-b → server-a
  - Downtime: ~5-10 minutes
  
  ```
  Monday 2:00 AM (low traffic window):
  1. Take server-a offline (graceful shutdown)
  2. Pull v1.0.0 image
  3. Start server-a with v1.0.0
  4. Wait for health checks (2-3 min)
  5. Route all tenants from server-b → server-a
  6. Verify routing (5 min)
  7. Server-b now idle
  8. Downtime: ~10 minutes
  ```

  **Option B: Rolling Upgrade (More complex)**
  - Gradual shift traffic from server-a to server-b to server-a
  - Zero downtime but more moving parts
  - Requires DNS updates, circuit breakers, etc.

  **Recommendation:** Option A (blue-green) is simpler, lower risk, acceptable 10-min downtime

- [ ] **Schedule upgrade window**:
  - [ ] Choose low-traffic time (e.g., 2:00-4:00 AM UTC Monday)
  - [ ] Avoid business hours for key customer regions
  - [ ] Avoid maintenance windows of other systems
  - [ ] Notify customers 48h in advance

- [ ] **Prepare rollback plan**:
  - If upgrade fails, immediately re-route to old server-b
  - Keep server-b running until server-a upgrade proven stable

**Acceptance Criteria:**
- Upgrade strategy documented
- Upgrade window scheduled and communicated
- Rollback plan ready

---

### 2. Pre-Upgrade Validation

**What:** Verify everything ready before upgrade.

**Changes:**
- [ ] **Verify current state**:
  ```bash
  # Check all tenants routed to server-b
  curl http://routing-service:9000/route?tenant=any-tenant
  # Expected: "target_server": "server-b:8000"
  
  # Verify server-a is running but idle
  docker-compose logs server-a | tail -20
  
  # Verify server-b is healthy
  docker-compose logs server-b | tail -20
  ```

- [ ] **Backup all data** (extra precaution):
  ```bash
  scripts/backup-registry-db.sh
  scripts/backup-tenant-db.sh
  ```

- [ ] **Take metrics baseline**:
  - [ ] Screenshot all dashboards (before upgrade)
  - [ ] Save query logs (for comparison)

- [ ] **Alert team**:
  - [ ] Final confirmation from tech lead: Ready to proceed?
  - [ ] On-call engineer standing by
  - [ ] Support team aware (no customer comms for planned 10-min downtime)

- [ ] **Test upgrade steps in parallel environment** (optional but recommended):
  - [ ] If possible, spin up test server-a with v1.0.0
  - [ ] Verify startup sequence and health checks
  - [ ] Time the upgrade (should be <5 min)

**Acceptance Criteria:**
- All verification steps passed
- Backups completed
- Team ready and standing by

---

### 3. Server-a Upgrade (Execution)

**What:** Upgrade server-a from v0.x.x to v1.0.0.

**Upgrade Timeline:**

**T-5 min (1:55 AM UTC Monday):**
- [ ] Last-minute verification
- [ ] All systems checked
- [ ] Announce in Slack: "Upgrade starting in 5 minutes"

**T-0 (2:00 AM UTC):**
- [ ] **Graceful shutdown of server-a**:
  ```bash
  docker-compose stop server-a
  # Wait 30s for graceful termination
  sleep 30
  docker-compose rm server-a
  ```

- [ ] **Verify all routing now goes to server-b**:
  ```bash
  # All requests should go to server-b during upgrade
  # Server-b should handle 100% load (already tested in Phase 3)
  ```

**T+2 min (2:02 AM UTC):**
- [ ] **Pull v1.0.0 image**:
  ```bash
  docker pull playhouse-server:v1.0.0
  ```

**T+3 min (2:03 AM UTC):**
- [ ] **Start server-a with v1.0.0**:
  ```bash
  docker-compose -f docker-compose.prod.yml up -d server-a
  # Image: playhouse-server:v1.0.0
  ```

- [ ] **Wait for health check**:
  ```bash
  # Docker health check runs every 10s
  # Waits up to 30s for first success
  # Expected: healthy within 30-60s
  
  docker-compose ps | grep server-a
  # Status should show: "healthy" (not "starting" or "unhealthy")
  
  # Alternatively, direct health check:
  curl http://localhost:8000/health
  # Expected: 200 OK, response contains "healthy": true
  ```

**T+5 min (2:05 AM UTC):**
- [ ] **Verify server-a is ready**:
  ```bash
  # Test database connection
  curl http://localhost:8000/admin/api/health
  
  # Check logs for errors
  docker-compose logs server-a | tail -30
  # Look for: no "ERROR" lines, should see startup logs
  ```

**T+6 min (2:06 AM UTC):**
- [ ] **Update routing to server-a**:
  ```bash
  # Move all tenants from server-b back to server-a
  scripts/route-all-tenants-to-version.sh v1.0.0 server-a
  
  # Or update registry directly:
  UPDATE deployment_pools SET status = 'active' WHERE version = 'v1.0.0' AND server_host = 'server-a:8000';
  UPDATE deployment_pools SET status = 'idle' WHERE version = 'v1.0.0' AND server_host = 'server-b:8000';
  ```

- [ ] **Wait 60s for routing cache TTL**:
  ```bash
  sleep 60
  ```

**T+7 min (2:07 AM UTC):**
- [ ] **Verify routing changed**:
  ```bash
  # Sample verify:
  for i in {1..5}; do
    curl http://routing-service:9000/route?tenant=tenant-$i
    # Expected: "target_server": "server-a:8000"
  done
  ```

**T+8 min (2:08 AM UTC):**
- [ ] **Spot check tenant access**:
  ```bash
  # Test a few real tenant subdomains
  curl http://acme-corp.pilche.ir/api/v1/health
  # Expected: 200 OK
  
  curl http://admin.pilche.ir/api/v1/config
  # Expected: 200 OK
  ```

**T+10 min (2:10 AM UTC):**
- [ ] **Announce upgrade complete**:
  - [ ] Slack message: "Upgrade complete. All tenants on server-a (v1.0.0)."
  - [ ] Status: ✓ Successful
  - [ ] Downtime: ~10 minutes (planned maintenance window)

**Acceptance Criteria:**
- Server-a successfully upgraded to v1.0.0
- Health checks passing
- All tenants routed to server-a
- Initial access tests passed
- Downtime: approximately 10 minutes

---

### 4. Post-Upgrade Validation (First 2 hours)

**What:** Intensive validation after upgrade.

**Changes:**
- [ ] **Continuous monitoring** (first 2 hours):
  ```
  2:10 AM: Upgrade complete
  2:15 AM: Check metrics (5-min mark)
  2:20 AM: Check metrics
  2:30 AM: 30-min assessment
  3:00 AM: 1-hour assessment
  4:10 AM: 2-hour full assessment
  ```

- [ ] **Monitoring checklist** (every 10 min for first 30 min, then every 30 min):
  - [ ] Server-a CPU/memory normal
  - [ ] Server-b CPU/memory idle (no traffic)
  - [ ] Database connections healthy (<80% saturation)
  - [ ] Error rate normal (<1%)
  - [ ] API response times normal (±10% of baseline)
  - [ ] Logs: no critical errors
  - [ ] Routing: all tenants on server-a

- [ ] **Sample tenant access** (every 15 min for first hour):
  ```bash
  for i in {1..5}; do
    TENANT=$(get_random_tenant)
    curl http://${TENANT}.pilche.ir/api/v1/health
    # Expected: 200 OK
  done
  ```

- [ ] **Feature validation** (random sampling):
  - [ ] Web app loads
  - [ ] API calls work
  - [ ] Feature flags/entitlements enforced
  - [ ] Observability mode showing metrics

**Post-Upgrade Metrics** (compare to pre-upgrade baseline):
- [ ] Error rate: within ±0.5%
- [ ] Latency p95: within ±10%
- [ ] Database latency: within ±10%
- [ ] CPU/memory: normal utilization

**Acceptance Criteria:**
- All validation checks passed
- Metrics aligned with baseline
- No errors or anomalies
- Upgrade successful

---

### 5. Extended Monitoring (24-48 hours)

**What:** Monitor production stability after upgrade for full day cycle.

**Changes:**
- [ ] **Day 1 (Tuesday) Morning**:
  - [ ] Full business day traffic pattern
  - [ ] All customer regions active
  - [ ] Monitor for any delayed issues

- [ ] **Day 1 Afternoon**:
  - [ ] Peak traffic hours
  - [ ] Verify performance holds under load

- [ ] **Day 1 Evening / Day 2 Morning**:
  - [ ] Off-peak to peak transitions
  - [ ] Overnight background jobs

- [ ] **Monitoring checklist** (every 4 hours):
  - [ ] Error rates
  - [ ] Response times
  - [ ] Database performance
  - [ ] Memory/CPU trends
  - [ ] Logs: any issues?

- [ ] **Customer feedback**:
  - [ ] Zero support tickets about upgrade
  - [ ] Normal customer activity
  - [ ] No feature regressions reported

**Acceptance Criteria:**
- Full 24-48h cycle completed
- All metrics normal
- No customer issues
- Upgrade proven stable

---

### 6. Server-B Decommissioning

**What:** Safely stop server-b and clean up.

**Changes (after 24-48h post-upgrade validation):**

- [ ] **Stop server-b**:
  ```bash
  docker-compose stop server-b
  docker-compose rm server-b
  ```

- [ ] **Archive old v0.x.x image** (keep for a while):
  ```bash
  docker tag playhouse-server:v0.x.x playhouse-server:v0.x.x-archived
  # Keep in registry for 1 month before deleting
  ```

- [ ] **Clean up old volumes** (if any):
  ```bash
  docker volume ls
  # Remove unused volumes related to old version
  ```

- [ ] **Update docker-compose.prod.yml**:
  - [ ] Remove or comment out server-b definition
  - [ ] Keep commented for 1 month (easy to re-enable if needed)
  - [ ] After 1 month: safely delete

- [ ] **Archive v0.x.x code** (in git):
  - [ ] Tag final v0.x.x release: `git tag v0.x.x-final`
  - [ ] Create release notes documenting v0.x.x EOL
  - [ ] Keep in repo history (never delete)

**Acceptance Criteria:**
- Server-b stopped and removed
- Old images archived
- docker-compose.prod.yml updated
- No references to v0.x.x in running config

---

### 7. Docker Swarm Preparation (Optional)

**What:** Prepare for Docker Swarm scale-out (future enhancement).

**Changes (Optional for Phase 4):**

- [ ] **Ensure docker-compose.prod.yml is Swarm-ready**:
  ```yaml
  version: '3.9'
  services:
    server:
      deploy:
        replicas: 3  # Can scale to multiple instances
        update_config:
          parallelism: 1
          delay: 30s
  ```

- [ ] **Test stack deployment on Swarm** (if Swarm nodes available):
  ```bash
  docker stack deploy -c docker-compose.prod.yml pilche
  docker stack ps pilche
  ```

- [ ] **Document Swarm scaling procedure** for future use

**Acceptance Criteria:**
- docker-compose.prod.yml Swarm-compatible
- Can scale replicas if needed
- Documentation for future scaling

---

### 8. Final Documentation & Sign-Off

**What:** Document upgrade completion and final production readiness.

**Changes:**
- [ ] **Create Phase 4 completion report**:
  ```
  # Phase 4: Retire Old Version - Completion Report
  
  ## Summary
  - Upgrade Date: [Date]
  - Upgrade Time: [Time]
  - Downtime: ~10 minutes (planned maintenance window)
  - Status: ✓ SUCCESSFUL
  
  ## Metrics
  - Pre-upgrade error rate: 0.8%
  - Post-upgrade error rate: 0.85% (within acceptable ±0.5%)
  - Pre-upgrade latency p95: 125ms
  - Post-upgrade latency p95: 128ms (within acceptable ±10%)
  - Database performance: Normal
  
  ## Issues Encountered
  - [None / List any issues and resolution]
  
  ## Customer Impact
  - Support tickets during/after upgrade: 0
  - Customer complaints: 0
  - Successful user sessions: 100%
  
  ## Learnings
  - [Document any insights for future deployments]
  
  ## Approval
  - Tech Lead: [Name] ✓
  - Product Owner: [Name] ✓
  - Operations: [Name] ✓
  ```

- [ ] **Update production documentation**:
  - [ ] Update PRODUCTION-DEPLOYMENT.md (mark Phase 4 complete)
  - [ ] Update MONITORING.md (now single version metrics)
  - [ ] Archive old runbooks (v0.x.x procedures no longer needed)
  - [ ] Create POST-DEPLOYMENT.md (stability monitoring, next steps)

- [ ] **Create v1.0.0 production readiness checklist**:
  - [ ] ✓ All tenants on v1.0.0
  - [ ] ✓ v0.x.x decommissioned from production
  - [ ] ✓ 48h+ stable production operation
  - [ ] ✓ Monitoring dashboards updated
  - [ ] ✓ Documentation complete
  - [ ] ✓ Team trained on v1.0.0 operations
  - [ ] ✓ Customer feedback positive

- [ ] **Team celebration** 🎉:
  - [ ] Share completion announcement in Slack
  - [ ] Celebrate successful 4-phase deployment
  - [ ] Document team learnings

**Acceptance Criteria:**
- Phase 4 completion report written
- Production documentation updated
- v1.0.0 readiness checklist signed off
- Team notified and celebrated

---

### 9. Post-Deployment Monitoring Setup

**What:** Transition from upgrade mode to normal production monitoring.

**Changes:**
- [ ] **Update monitoring dashboards**:
  - [ ] Remove server version comparison (now single version)
  - [ ] Update rollout progress (now 100% v1.0.0)
  - [ ] Focus on stability metrics instead

- [ ] **Update alerting thresholds**:
  - [ ] Adjust based on v1.0.0 baseline performance
  - [ ] Remove upgrade-specific alerts
  - [ ] Keep canary/rollout alerts for future deployments

- [ ] **Create ongoing monitoring runbook**:
  - [ ] Daily health checks
  - [ ] Weekly performance review
  - [ ] Monthly trend analysis

**Acceptance Criteria:**
- Monitoring dashboards updated for v1.0.0
- Alert thresholds finalized
- Ongoing monitoring runbook ready

---

### 10. Future Readiness (v2.0.0)

**What:** Prepare infrastructure for next version deployment.

**Changes:**
- [ ] **Architecture now supports v2.0.0 easily**:
  ```
  For v2.0.0 deployment:
  1. Deploy server-b with v2.0.0 image
  2. Repeat Phase 1 testing (staging)
  3. Repeat Phase 2 canary (5-10% of tenants)
  4. Repeat Phase 3 progressive rollout
  5. Repeat Phase 4 upgrade server-a to v2.0.0
  
  This infrastructure is reusable for all future versions!
  ```

- [ ] **Document v2.0.0 readiness**:
  - [ ] Architecture can support continuous deployments
  - [ ] Estimated Phase 0-4 timeline: 6-9 weeks per version
  - [ ] Team experienced with procedure
  - [ ] Automation in place (scripts, monitoring)

**Acceptance Criteria:**
- Infrastructure ready for v2.0.0 whenever needed
- Documentation clear for next deployment cycle

---

## Exit Criteria

- ✓ Server-a successfully upgraded from v0.x.x to v1.0.0
- ✓ All tenants routed back to server-a (v1.0.0)
- ✓ Server-b now idle (available for future versions)
- ✓ 24-48h post-upgrade monitoring completed
- ✓ Zero downtime errors or customer issues
- ✓ Metrics aligned with baseline
- ✓ v0.x.x decommissioned from production
- ✓ Phase 4 completion report signed off
- ✓ Documentation updated
- ✓ Team celebrated successful deployment

---

## Risks & Mitigations

| Risk | Mitigation |
|------|-----------|
| Upgrade fails, server-a won't start | Quick rollback: re-route all to server-b (still running) |
| Load spike during upgrade | Server-b handles 100% traffic during upgrade (capacity tested in Phase 3) |
| Unforeseen compatibility issues | Extended post-upgrade monitoring (24-48h) catches delayed issues |
| Customer impact from 10-min downtime | Planned maintenance window, transparent customer communication |

---

## Success Metrics

| Metric | Target |
|--------|--------|
| Upgrade completion time | <15 minutes |
| Downtime during upgrade | ~10 minutes (planned maintenance) |
| Post-upgrade error rate delta | <1% vs baseline |
| Post-upgrade latency delta | <10% vs baseline |
| Customer issues post-upgrade | 0 |
| Post-upgrade monitoring period | 24-48h stable |

---

## Timeline

| Day | Activity |
|-----|----------|
| Mon | Pre-upgrade planning & validation |
| Tue | Server-a upgrade (T-0 to T+10 min) → Intense monitoring (2h) |
| Tue-Wed | Extended monitoring (24-48h) |
| Wed | Server-b decommissioning |
| Wed-Thu | Final documentation & sign-off |
| Thu | Team celebration & reflection |

---

## Next Steps After Phase 4

After successful v1.0.0 production deployment:

1. **Stabilization Period** (1-2 weeks):
   - Monitor production closely
   - Fix any bugs discovered
   - Gather customer feedback

2. **Retrospective** (Week 2):
   - Team retrospective on entire 6-9 week deployment
   - Document learnings and improvements
   - Update deployment procedures for v2.0.0

3. **Normal Operations** (Ongoing):
   - Maintain current production
   - Plan next features/versions
   - Use this blue-green infrastructure for continuous deployments

4. **Potential Improvements** (Future):
   - Scale to Docker Swarm (multiple server instances)
   - Implement automated rollback triggers
   - Reduce deployment window (faster migrations)
   - Continuous deployment pipeline

---

## Notes

- Phase 4 marks completion of Pilche production deployment
- Total timeline: ~6-9 weeks from Phase 0 to Phase 4
- Infrastructure now supports continuous future deployments
- v1.0.0 production-ready and stable
- Team experienced and confident in procedures
- All documentation complete for handoff and knowledge transfer
