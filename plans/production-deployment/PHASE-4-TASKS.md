# Phase 4 - Task Breakdown

This phase is split into 4 focused tasks. Complete them sequentially.

## Phase 4 Overview
- **Goal**: Upgrade server-a to v1.0.0, retire v0.x.x, and stabilize production
- **Duration**: 1 week
- **Repos**: playhouse-cicd
- **Prerequisites**: Phase 0, 1, 2, 3 complete, 100% of tenants on v1.0.0 (server-b)
- **End State**: Both servers on v1.0.0, v0.x.x retired, production stable

## Tasks

### Task 4.1: Pre-Upgrade Planning & Validation
- **Goal**: Plan upgrade and verify readiness
- **Repos**: playhouse-cicd
- **Work**:
  - [ ] **Choose upgrade strategy**:
    - [ ] Blue-green upgrade (recommended):
      - Server-a taken offline
      - Upgraded to v1.0.0 offline
      - All traffic routed from server-b → server-a
      - ~10 minute downtime during upgrade window
    - [ ] OR rolling upgrade (more complex, less downtime)
  - [ ] **Schedule upgrade window**:
    - [ ] Choose low-traffic time (2-4 AM UTC)
    - [ ] Avoid business hours for key regions
    - [ ] Avoid other maintenance windows
    - [ ] Notify customers 48h in advance
  - [ ] **Prepare rollback plan**:
    - If upgrade fails: re-route to server-b (already v1.0.0)
    - Keep server-b running until server-a proven stable
  - [ ] **Pre-flight checks**:
    - [ ] Verify current state (all on server-b)
    - [ ] Extra backup of all data
    - [ ] Take metrics baseline screenshot
    - [ ] Confirm team ready and standing by
  - **Acceptance**: Upgrade strategy documented, window scheduled, team ready
- **Duration**: 1-2 days
- **Next**: Task 4.2

---

### Task 4.2: Server-A Upgrade Execution
- **Goal**: Upgrade server-a from v0.x.x to v1.0.0
- **Repos**: playhouse-cicd
- **Work**:
  - [ ] **T-5 minutes (1:55 AM UTC)**:
    - [ ] Final verification
    - [ ] Announce in Slack: "Upgrade starting in 5 minutes"
  - [ ] **T-0 (2:00 AM UTC) - Stop Server-A**:
    - [ ] Graceful shutdown: `docker-compose stop server-a`
    - [ ] Wait 30s for termination
    - [ ] Remove container: `docker-compose rm server-a`
    - [ ] Verify: all routing now goes to server-b
  - [ ] **T+2 min (2:02 AM UTC) - Pull v1.0.0 Image**:
    - [ ] `docker pull playhouse-server:v1.0.0`
  - [ ] **T+3 min (2:03 AM UTC) - Start Server-A**:
    - [ ] `docker-compose -f docker-compose.prod.yml up -d server-a`
    - [ ] Image: playhouse-server:v1.0.0
  - [ ] **T+5 min (2:05 AM UTC) - Wait for Health Check**:
    - [ ] Monitor: `docker-compose ps | grep server-a`
    - [ ] Expected: status "healthy" within 30-60s
    - [ ] Direct test: `curl http://localhost:8000/health`
  - [ ] **T+6 min (2:06 AM UTC) - Verify Server-A Ready**:
    - [ ] Check logs: no critical errors
    - [ ] Test database connection
    - [ ] Verify no startup issues
  - [ ] **T+6 min (2:06 AM UTC) - Update Routing**:
    - [ ] Via Admin UI or API: route all tenants from server-b → server-a
    - [ ] Set server-a status: "active"
    - [ ] Set server-b status: "idle"
  - [ ] **T+7 min (2:07 AM UTC) - Wait for Cache TTL**:
    - [ ] Sleep 60s (routing cache TTL)
  - [ ] **T+8 min (2:08 AM UTC) - Verify Routing Changed**:
    - [ ] Sample verify: `curl /api/v1/route?tenant=tenant-1`
    - [ ] Expected: routing to server-a:8000
  - [ ] **T+9 min (2:09 AM UTC) - Spot Check Tenants**:
    - [ ] Test tenant access:
      - `curl http://tenant-1.pilche.ir/api/v1/health`
      - Expected: 200 OK
  - [ ] **T+10 min (2:10 AM UTC) - Announce Complete**:
    - [ ] Slack: "Upgrade complete. All on server-a (v1.0.0)"
    - [ ] Status: ✓ Successful
    - [ ] Downtime: ~10 minutes (planned maintenance)
  - **Acceptance**: Server-a upgraded, all tenants routed to server-a, downtime ~10 min
- **Duration**: Execution: 10-15 min, Plus pre/post: ~1 hour
- **Next**: Task 4.3

---

### Task 4.3: Post-Upgrade Validation (First 48 Hours)
- **Goal**: Intensive validation after upgrade
- **Repos**: playhouse-cicd
- **Work**:
  - [ ] **Immediate (First 2 hours)**:
    - [ ] Every 10 min for first 30 min, then every 30 min:
      - [ ] Server-a CPU/memory normal?
      - [ ] Server-b CPU/memory idle?
      - [ ] Database connections healthy?
      - [ ] Error rate normal (<1%)?
      - [ ] API response times normal?
      - [ ] Logs: any errors?
  - [ ] **First 24 hours**:
    - [ ] Monitor every 4 hours:
      - [ ] Error rates vs baseline (±0.5%)
      - [ ] Latency vs baseline (±10%)
      - [ ] Customer reports: zero?
      - [ ] Support tickets: zero?
  - [ ] **Second 24 hours**:
    - [ ] Monitor every 8 hours:
      - [ ] Full business day cycle completed
      - [ ] Off-peak to peak transitions
      - [ ] All metrics normal
  - [ ] **Metrics Comparison**:
    - [ ] Compare to pre-upgrade baseline
    - [ ] Should be within normal range
    - [ ] No anomalies or spikes
  - [ ] **Customer Feedback**:
    - [ ] Zero support tickets
    - [ ] Normal customer activity
    - [ ] No feature regressions
  - **Acceptance**: 48h post-upgrade, all metrics normal, zero customer issues
- **Duration**: 2 days (continuous monitoring)
- **Next**: Task 4.4

---

### Task 4.4: Decommissioning & Finalization
- **Goal**: Retire v0.x.x and finalize production
- **Repos**: playhouse-cicd
- **Work**:
  - [ ] **Stop Server-B** (after 48h validation):
    - [ ] `docker-compose stop server-b`
    - [ ] `docker-compose rm server-b`
    - [ ] Verify server-a handling all traffic
  - [ ] **Archive Old Version**:
    - [ ] Tag final v0.x.x: `docker tag playhouse-server:v0.x.x playhouse-server:v0.x.x-archived`
    - [ ] Keep in registry for 1 month
    - [ ] After 1 month: safe to delete
  - [ ] **Update docker-compose.prod.yml**:
    - [ ] Comment out server-b definition
    - [ ] Keep commented for 1 month (easy re-enable if needed)
    - [ ] After 1 month: delete server-b from compose
  - [ ] **Archive v0.x.x Code** (in git):
    - [ ] Tag final release: `git tag v0.x.x-final`
    - [ ] Create release notes (v0.x.x EOL)
    - [ ] Keep in repo history forever
  - [ ] **Create Phase 4 Completion Report**:
    - [ ] Upgrade Date/Time
    - [ ] Downtime duration
    - [ ] Metrics comparison (before/after)
    - [ ] Issues encountered (if any)
    - [ ] Learnings and improvements
    - [ ] Sign-offs from tech lead, ops, product
  - [ ] **Update Production Documentation**:
    - [ ] Mark Phase 4 complete in README
    - [ ] Update MONITORING.md (now single version)
    - [ ] Archive old runbooks (v0.x.x procedures)
    - [ ] Create POST-DEPLOYMENT.md (ongoing monitoring)
  - [ ] **Team Celebration** 🎉:
    - [ ] Announce: "v1.0.0 production deployment complete!"
    - [ ] Share metrics and success story
    - [ ] Team celebration/debrief
  - [ ] **Schedule v2.0.0 Planning**:
    - [ ] Document that infrastructure reusable
    - [ ] Estimated timeline for v2.0.0 (6-9 weeks)
    - [ ] Same procedure can be repeated
  - **Acceptance**: v0.x.x retired, documentation complete, team celebrated, v2.0.0 path clear
- **Duration**: 1-2 days
- **Next**: Production Stabilization

---

## Phase 4 Exit Criteria

- ✓ Server-a successfully upgraded to v1.0.0
- ✓ All tenants routed to server-a (v1.0.0)
- ✓ Server-b (v1.0.0) now idle
- ✓ Downtime: ~10 minutes (acceptable for planned maintenance)
- ✓ 48h post-upgrade monitoring completed
- ✓ All metrics normal
- ✓ Zero customer issues
- ✓ v0.x.x decommissioned from production
- ✓ Phase 4 completion report signed off
- ✓ All documentation updated
- ✓ Team celebrated
- ✓ v2.0.0 readiness documented

---

## Timeline

| Task | Duration | Total |
|------|----------|-------|
| 4.1 | 1-2 days | 1-2 days |
| 4.2 | ~1 hour execution | 2-3 hours |
| 4.3 | 48 hours monitoring | 2 days |
| 4.4 | 1-2 days | 1-2 days |
| **Total** | | **1 week** |

---

## Production Stabilization (Ongoing)

After Phase 4 completes:
- Monitor production 24/7 for 1-2 weeks
- Daily health checks
- Weekly performance reviews
- Monthly trend analysis
- Document any issues and learnings
- Update runbooks for future deployments

---

## Success Metrics

| Metric | Target |
|--------|--------|
| Upgrade time | <15 min |
| Downtime | ~10 min (planned) |
| Error rate post-upgrade | ±0.5% vs baseline |
| Latency post-upgrade | ±10% vs baseline |
| Customer issues | 0 |
| Post-upgrade monitoring | 48h+ stable |

---

## Future Deployments

This infrastructure is now ready for v2.0.0:
- Server-B idle and available
- Can start v2.0.0 deployment immediately
- Same Phase 0-4 procedure applies
- Team experienced with process
- Estimated: 6-9 weeks per future version

---

## Celebrate! 🎉

You've successfully deployed Pilche v1.0.0 to production:
- Blue-green infrastructure ✓
- Multi-version routing ✓
- Progressive rollout ✓
- Zero customer downtime ✓
- Team trained and confident ✓
