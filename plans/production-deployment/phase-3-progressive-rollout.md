# Production Deployment - Phase 3: Progressive Rollout

## Goal

Gradually migrate all remaining tenants from v0.x.x (server-a) to v1.0.0 (server-b) in 4 batches over 1-2 weeks, monitoring at each step for stability before proceeding.

## Repos

- `playhouse-cicd` (deployment scripts, monitoring)
- `playhouse-control-plane` (registry, routing updates)

## Duration

1-2 weeks

## Overview

This phase:
1. Migrates tenants in 4 batches (10% → 35% → 85% → 100%)
2. Monitors health at each batch
3. Applies go/no-go criteria before each batch
4. Rolls back any batch if issues detected
5. Maintains zero downtime throughout
6. Documents progress and decisions

## Prerequisites (From Phase 2)

- ✓ Production infrastructure ready
- ✓ Monitoring dashboards live
- ✓ Canary deployment successful (5-10% of tenants on v1.0.0)
- ✓ Canary go/no-go decision: GO
- ✓ Baseline metrics captured
- ✓ Team confident and trained

## Batch Strategy

**Total tenants (example): 50**
- Canary (already done): 5 tenants (10%)
- Batch 1: 5 tenants (10% of remaining 45 = 10%) → 10 total (20%)
- Batch 2: 11 tenants (25%) → 21 total (42%)
- Batch 3: 22 tenants (50%) → 43 total (86%)
- Batch 4: 7 tenants (14%) → 50 total (100%)

**Timing:**
- Batch 1: Monday
- Batch 2: Wednesday (after 48h monitoring of batch 1)
- Batch 3: Friday (after 48h monitoring of batch 2)
- Batch 4: Monday of week 2 (after 48h monitoring of batch 3)

---

## Work Items

### 1. Batch Planning & Tenant Segmentation

**What:** Plan all 4 batches upfront, identifying tenants for each.

**Changes:**
- [ ] Create batch assignment spreadsheet:
  ```
  Tenant ID | Customer Name | Plan Type | Batch | Migration Date | Status
  --------  | ------------- | --------- | ----- | -------------- | ------
  tenant-1  | ACME Corp     | Pro       | 1     | Mon 08:00      | pending
  tenant-2  | XYZ Ltd       | Free      | 1     | Mon 08:00      | pending
  ...
  ```

- [ ] Batch 1 (Mon): 10% of remaining
  - [ ] Select diverse mix: different plan types, sizes, regions
  - [ ] Avoid: any with recent support tickets, near renewal dates

- [ ] Batch 2 (Wed): 25% of remaining
  - [ ] Slightly less conservative, include some mid-tier customers
  - [ ] Still avoid: large contracts, critical accounts

- [ ] Batch 3 (Fri): 50% of remaining
  - [ ] More diverse mix
  - [ ] Can include larger customers now

- [ ] Batch 4 (Mon+): Final 14%
  - [ ] Can include largest customers
  - [ ] By now, stability proven across broad user base

- [ ] Review with product/customer success:
  - [ ] Any customers to exclude? (temporarily large usage, API integration, etc.)
  - [ ] Notify major customers upfront (transparency)

**Acceptance Criteria:**
- All batches planned and assigned
- Tenants selected strategically (low to high risk)
- Customer success notified for large accounts
- Team has batch assignment list

---

### 2. Pre-Batch 1 Notification

**What:** Notify team and prepare for first progressive batch.

**Changes:**
- [ ] **Day before Batch 1 (Sunday)**:
  - [ ] Send team notification: "Batch 1 deployment tomorrow"
  - [ ] Post batch assignment in Slack
  - [ ] Ensure on-call engineer confirmed
  - [ ] Verify monitoring dashboards ready
  - [ ] All scripts tested one final time

**Acceptance Criteria:**
- Team notified and ready
- On-call rotation confirmed
- Pre-flight checks passed

---

### 3. Batch 1 Deployment (Monday)

**What:** Deploy first progressive batch (10% of tenants) via control-plane admin UI.

**Timeline:**

**Morning (8:00 AM UTC):**
- [ ] Take baseline metrics screenshot
- [ ] Announce in Slack: "Batch 1 migration starting"
- [ ] Send customer notifications (if any): "Maintenance window 8:00-10:00"
- [ ] Log in to Admin UI: `/admin-ui/deployments/`

**Pre-Migration (8:00-8:15 AM):**
- [ ] Backup all Batch 1 tenant DBs (automated or manual):
  ```bash
  for tenant in batch_1_tenant_1 batch_1_tenant_2 ... ; do
    pg_dump -h playhouse-db -U postgres -d tenant_${tenant} | gzip \
      > /data/backups/tenant-${tenant}-$(date +%Y%m%d-%H%M%S).sql.gz
  done
  ```

**Via Admin UI - Start Batch 1 Deployment (8:15 AM):**
- [ ] Navigate to `/admin-ui/deployments/`
- [ ] Click "Continue Rollout" or "Deploy Next Batch"
- [ ] Wizard Step 1: Confirm version v1.0.0
- [ ] Wizard Step 2: Display pre-selected Batch 1 tenants (from batch planning)
- [ ] Wizard Step 3: Review and confirm deployment
- [ ] Click "Deploy Batch 1"
- [ ] Behind UI: Control-plane API triggers:
  ```json
  POST /api/v1/releases/v1.0.0/rollouts
  {
    "batch_number": 1,
    "tenant_ids": ["batch_1_tenant_1", "batch_1_tenant_2", ...],
    "target_server": "server-b:8000",
    "db_schema_version": 1,
    "scheduled_start": "2026-06-03T08:15:00Z",
    "monitoring_window_hours": 48
  }
  ```

**Control-Plane Orchestration (8:15-8:45 AM)** (automatic):
1. cp-controller receives rollout request
2. For each tenant in batch_1_tenant_ids:
   - Creates db migration job
   - Waits for job completion
   - Tracks progress in UI
3. After all migrations complete:
   - Updates all tenant_releases entries to v1.0.0
   - Sets status to 'in_progress'
   - Triggers routing service cache invalidation
4. Tenants routed to server-b automatically (no manual routing needed)

**Monitor Progress via Admin UI (8:15-8:45 AM):**
- [ ] Navigate to `/admin-ui/releases/v1.0.0/rollouts/1`
- [ ] View real-time migration progress:
  ```
  Batch 1 Migration Progress
  ████████░░░░░░░░░░░░ 40% complete (4/10 tenants migrated)
  
  Migrations:
  ✓ batch_1_tenant_1 - completed
  ✓ batch_1_tenant_2 - completed
  ✓ batch_1_tenant_3 - completed
  ✓ batch_1_tenant_4 - completed
  ⏳ batch_1_tenant_5 - in_progress
  ```
- [ ] All migrations should complete within 30 min

**Validation (8:45-9:00 AM):**
- [ ] Via Admin UI `/admin-ui/deployments/`:
  - [ ] Release status shows: "Batch 1 deploying"
  - [ ] Progress bar shows: 10% of tenants on v1.0.0
  - [ ] Real-time metrics updating
- [ ] Spot check tenant access:
  ```bash
  curl -u admin:password http://batch-1-tenant-1.pilche.ir/api/v1/health
  # Expected: 200 OK
  ```
- [ ] Verify no errors in logs

**Post-Deployment (9:00 AM):**
- [ ] Announce in Slack: "Batch 1 deployed successfully"
- [ ] Update Admin UI release notes: "Batch 1 deployed at 9:00 AM UTC"
- [ ] Start continuous monitoring (via Admin UI or dashboards)
- [ ] Admin UI shows release status: "monitoring_batch_1" (20% of tenants on v1.0.0)

**Acceptance Criteria:**
- All Batch 1 tenants successfully migrated via control-plane
- All tenants routed to v1.0.0
- Admin UI shows accurate progress
- Initial validation passed
- Monitoring running

---

### 4. Batch 1 Monitoring (24-48 hours)

**What:** Intensive monitoring of Batch 1 for 24-48 hours.

**Monitoring Checklist** (every 2-4 hours for first 24h, then every 4-8h):

**Routing & Availability:**
- [ ] Batch 1 tenants routed to server-b consistently
- [ ] Non-batch tenants still routed to server-a
- [ ] Server-a and server-b both healthy

**Performance:**
- [ ] API response times for Batch 1: within baseline ±10%
- [ ] Database query latency: normal
- [ ] CPU/memory utilization: normal
- [ ] No connection pool saturation

**Errors & Logging:**
- [ ] Error rate for Batch 1: v1.0.0 ± 0.5% (no increase)
- [ ] 5xx errors: <1% (same as server-a)
- [ ] 4xx errors: normal (user errors)
- [ ] Logs: no critical errors related to Batch 1

**Data Integrity:**
- [ ] No data loss or corruption
- [ ] Cross-tenant isolation maintained
- [ ] Audit logs clean

**Customer Reports:**
- [ ] Support tickets for Batch 1: 0
- [ ] Customer complaints: 0
- [ ] Batch 1 tenants reporting normal service

**Entitlements** (observability mode):
- [ ] Feature denials logged normally
- [ ] Config version fetch latency: normal
- [ ] Cache hit rate: >95%
- [ ] No version mismatches

**Metrics vs. Canary:**
- [ ] Batch 1 metrics similar to Canary (same version)
- [ ] No anomalies vs. Canary baseline

**Monitoring Log** (example entries):
```
Mon 10:00 AM: Batch 1 deployed, all healthy. Monitoring started.
Mon 02:00 PM: 4h check-in, no issues. Error rate 0.8%, latency 120ms p95.
Mon 10:00 PM: 12h check-in, overnight traffic normal. Still healthy.
Tue 10:00 AM: 24h assessment, ready for go/no-go.
```

---

### 5. Batch 1 Go/No-Go Decision (Tuesday AM)

**What:** After 24-48h, decide whether to proceed to Batch 2.

**Go/No-Go Criteria** (all must pass):

- ✓ **Availability**: Batch 1 server availability = 99.9%+
- ✓ **Error Rate**: No significant increase vs. baseline
- ✓ **Performance**: Response times within ±10% of baseline
- ✓ **Data Integrity**: Zero data loss/corruption
- ✓ **Customer Reports**: Zero issues from Batch 1 tenants
- ✓ **Support Tickets**: Zero Batch 1 related tickets
- ✓ **Logs**: No critical errors specific to Batch 1 tenants

**Decision Process:**

```
Tuesday, 10 AM (24-48h after Batch 1):
1. Tech lead reviews all Batch 1 metrics
2. Compare Batch 1 vs. Canary (should be similar)
3. On-call engineer confirms logs clean
4. Team votes: GO to Batch 2 or ROLLBACK Batch 1?
5. Decision documented in ticket/Slack
```

**If GO → Proceed to Batch 2** (same procedure as Batch 1)

**If NO-GO → Execute Batch 1 Rollback** (via Admin UI):
- [ ] Navigate to `/admin-ui/releases/v1.0.0/rollouts/1`
- [ ] Click "Rollback Batch 1" button
- [ ] Confirm in dialog: "Rollback Batch 1 to v0.x.x?"
- [ ] Behind UI: Control-plane API triggers:
  ```json
  POST /api/v1/releases/v1.0.0/rollback
  {
    "batch_number": 1,
    "reason": "Error rate spike detected",
    "rollback_to_version": "v0.x.x"
  }
  ```

- [ ] Control-plane orchestration:
  1. cp-controller receives rollback request
  2. Updates all Batch 1 tenant_releases: target_version = 'v0.x.x'
  3. Sets status to 'rolled_back'
  4. Routing service invalidates cache
  5. Tenants automatically re-route to server-a

- [ ] Verify rollback (via Admin UI):
  - [ ] Status shows: "Batch 1 rolled back"
  - [ ] Batch tenants show: 0% on v1.0.0
  - [ ] All tenants back on v0.x.x
  - [ ] Latency/error rates recovering
  - [ ] Estimated time: <5 minutes total

- [ ] Post-mortem:
  - [ ] Identify root cause
  - [ ] Document issue in ticket
  - [ ] Plan fix and retry (next week)

**Acceptance Criteria:**
- Explicit GO/NO-GO decision made
- Decision documented with reasoning
- Team aligned on next steps

---

### 6-8. Batch 2, 3, 4 (Same Pattern)

**Repeat the same process for Batches 2, 3, and 4:**

1. **Pre-Batch Notification** (day before)
2. **Batch Deployment** (morning)
   - Backup, migrate DBs, update routing, validate
3. **Batch Monitoring** (24-48h)
   - Check every 2-4h for first 24h
4. **Go/No-Go Decision** (next morning)
   - Proceed or rollback

**Key Differences by Batch:**

| Batch | Size | Risk Level | Includes |
|-------|------|-----------|----------|
| 1 | 10% | Low | Test, small customers |
| 2 | 25% | Low-Medium | Mix of sizes |
| 3 | 50% | Medium | Larger customers introduced |
| 4 | 14% | Medium-High | Largest customers |

**Batch 2 (Wednesday):**
- Deploy after Batch 1 go decision
- Slightly more diverse mix
- Still conservative risk

**Batch 3 (Friday):**
- Deploy after Batch 2 go decision
- Can include larger customers
- Still some caution

**Batch 4 (Monday of Week 2):**
- Deploy after Batch 3 go decision
- Final 14% of tenants
- Includes largest accounts (but now proven stable)

---

### 9. Monitoring Dashboard Throughout

**What:** Central monitoring view for entire progressive rollout.

**Rollout Progress Dashboard:**
```
Total Tenants: 50

Canary (Already Done):     ████░░░░░░░░░░░░░░░░ 5 (10%)
Batch 1 (Mon):             ████░░░░░░░░░░░░░░░░ 5 (10%) ✓ DEPLOYED
Batch 2 (Wed):             ░░░░░░░░░░░░░░░░░░░░ 0 (0%)  ⏳ PENDING
Batch 3 (Fri):             ░░░░░░░░░░░░░░░░░░░░ 0 (0%)  ⏳ PENDING
Batch 4 (Mon+):            ░░░░░░░░░░░░░░░░░░░░ 0 (0%)  ⏳ PENDING

Current Status:
- v0.x.x (server-a): 40 tenants (80%) - HEALTHY
- v1.0.0 (server-b): 10 tenants (20%) - HEALTHY

Last Check: Tue 10:15 AM
- Error Rate: 0.8% (both servers)
- Latency p95: 125ms
- DB Connections: 45/100
- Backups: ✓ All current
```

**Shared Dashboard for Team:**
- Real-time metrics (Datadog, Grafana, etc.)
- Annotation for each batch deployment
- Alert notifications for any anomalies
- Post-mortem notes for each batch

---

### 10. Batch Success Criteria

**For Each Batch:**

- ✓ All tenants in batch successfully migrated
- ✓ All tenants routed to v1.0.0
- ✓ 24-48h monitoring completed
- ✓ Metrics healthy (error rate, latency normal)
- ✓ Zero customer issues
- ✓ Go decision made
- ✓ Proceed to next batch or complete Phase 3

---

### 11. Documentation During Rollout

**What:** Document decisions and learnings throughout Phase 3.

**Changes:**
- [ ] **Daily standup** (10:00 AM UTC):
  - [ ] Current batch status
  - [ ] Metrics update
  - [ ] Any issues/anomalies
  - [ ] Go/no-go confidence level (1-10)

- [ ] **Batch completion notes** (for each batch):
  - [ ] Deployment time
  - [ ] Migration time per tenant
  - [ ] Any delays or issues
  - [ ] Monitoring highlights
  - [ ] Unexpected behaviors
  - [ ] Lessons learned

- [ ] **Rollout dashboard** (shared with team):
  - [ ] Progress vs. plan
  - [ ] Metrics trending
  - [ ] Any warnings
  - [ ] Team confidence level

**Acceptance Criteria:**
- Daily documentation
- Batch completion notes
- Team aligned on progress

---

### 12. Contingency Plans

**If Rollback Needed Mid-Rollout:**

```
Example: Batch 2 discovers critical issue with v1.0.0

1. Stop Batch 2 rollout immediately
2. Revert Batch 2 tenants to v0.x.x:
   scripts/rollback-routing.sh batch_2_tenant_1 batch_2_tenant_2 ...
3. Keep Batch 1 on v1.0.0 (already stable)
4. Keep Canary on v1.0.0
5. Investigate root cause
6. Fix issue in v1.0.0
7. Retry Batch 2 (next day or later)

Total estimated impact: ~1-2 hours downtime for affected tenants
```

**Maintain Safety Gates:**
- Maximum 1-2 tenants can be down at once (keep batch size manageable)
- Always have 50%+ tenants on stable version (can revert majority if needed)
- Hourly monitoring alerts (not daily) if any anomaly detected

---

## Exit Criteria

- ✓ Batch 1 successfully deployed and stable (24-48h)
- ✓ Batch 2 successfully deployed and stable (24-48h)
- ✓ Batch 3 successfully deployed and stable (24-48h)
- ✓ Batch 4 successfully deployed and stable (24-48h)
- ✓ 100% of tenants now on v1.0.0
- ✓ Server-a (v0.x.x) now idle/unused
- ✓ Zero data loss or corruption across all batches
- ✓ All customers happy and reporting normal service
- ✓ Monitoring shows v1.0.0 production-ready
- ✓ Phase 3 sign-off from tech lead

---

## Risks & Mitigations

| Risk | Mitigation |
|------|-----------|
| Issue found mid-batch | Rollback procedure ready; maximum 1h impact per batch |
| Cascading failures across batches | Conservative batch sizes; 24-48h monitoring between batches |
| Customer impact during migration | Transparent communication; support on standby |
| Database connection pool exhaustion | Monitored closely; circuit breaker implemented |
| Routing service failure during rollout | Fallback routing; cache at nginx level |
| Network issues during large batch | Staged rollout (10% → 50% → 100%) reduces blast radius |

---

## Success Metrics

| Metric | Target |
|--------|--------|
| Batch 1 migration success rate | 100% |
| Batch 2 migration success rate | 100% |
| Batch 3 migration success rate | 100% |
| Batch 4 migration success rate | 100% |
| Error rate delta for each batch | <0.5% vs baseline |
| Latency delta for each batch | <10% vs baseline |
| Customer issues per batch | 0 |
| Unplanned rollbacks | 0 |
| Total Phase 3 duration | 1-2 weeks |

---

## Timeline

| Day | Batch | Activity |
|-----|-------|----------|
| Mon | 1 | Deploy (8:00 AM) → Monitor 24h |
| Tue | 1 | Go/no-go decision (10:00 AM) |
| Wed | 2 | Deploy (8:00 AM) → Monitor 24h |
| Thu | 2 | Go/no-go decision (10:00 AM) |
| Fri | 3 | Deploy (8:00 AM) → Monitor 24h |
| Sat-Sun | 3 | Monitor (continued) |
| Mon | 3 | Go/no-go decision (10:00 AM) |
| Mon+ | 4 | Deploy → Monitor 24h |
| Tue+ | 4 | Go/no-go decision |
| Final | - | 100% of tenants on v1.0.0 ✓ |

---

## Notes

- Conservative batch strategy minimizes blast radius
- Each batch must prove stability before next batch
- Monitoring is continuous (every 2-4 hours minimum)
- Rollback procedure practiced in staging beforehand
- Team familiar with decision-making process
- Customer communication transparent throughout
- Documentation for audit trail and future deployments
