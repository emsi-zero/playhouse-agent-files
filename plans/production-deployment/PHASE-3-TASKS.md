# Phase 3 - Task Breakdown

This phase is split into 5 focused tasks. Complete them sequentially.

## Phase 3 Overview
- **Goal**: Gradually migrate all remaining tenants from v0.x.x to v1.0.0 in batches
- **Duration**: 1-2 weeks
- **Repos**: playhouse-cicd, playhouse-control-plane
- **Prerequisites**: Phase 0, 1, 2 complete with GO decision
- **End State**: 100% of tenants on v1.0.0

## Tasks

### Task 3.1: Batch Planning & Tenant Segmentation
- **Goal**: Plan all 4 batches and segment tenants strategically
- **Repos**: playhouse-cicd
- **Work**:
  - [ ] Create batch assignment spreadsheet:
    - [ ] Tenant ID, Customer, Plan, Batch, Scheduled Date
    - [ ] ~50 tenants total (adjust per your data)
  - [ ] Segment into 4 batches:
    - [ ] **Batch 1 (Mon)**: 10% remaining (~5 tenants)
    - [ ] **Batch 2 (Wed)**: 25% remaining (~11 tenants) → 35% total
    - [ ] **Batch 3 (Fri)**: 50% remaining (~22 tenants) → 85% total
    - [ ] **Batch 4 (Mon+1)**: Final 14% (~7 tenants) → 100% total
  - [ ] Batch strategy (conservative to aggressive):
    - [ ] Batch 1: Test/small customers (low risk)
    - [ ] Batch 2: Mix of sizes
    - [ ] Batch 3: Larger customers introduced
    - [ ] Batch 4: Largest customers (by now proven stable)
  - [ ] Review with product/customer success:
    - [ ] Any exclusions? (large usage, API integration, etc.)
    - [ ] Notify major customers?
  - **Acceptance**: All batches planned, assigned, reviewed
- **Duration**: 1 day
- **Next**: Task 3.2

---

### Task 3.2: Batch 1 Deployment (Monday)
- **Goal**: Deploy and validate Batch 1 (10% of tenants)
- **Repos**: playhouse-cicd, playhouse-control-plane
- **Work**:
  - [ ] **Morning**:
    - [ ] Announce in Slack: "Batch 1 starting"
    - [ ] Take baseline metrics screenshot
    - [ ] Backup Batch 1 tenant DBs
  - [ ] **Via Admin UI**:
    - [ ] Navigate to `/admin-ui/deployments/`
    - [ ] Click "Deploy Batch 1"
    - [ ] Confirm tenants (pre-filled from planning)
    - [ ] Click "Deploy"
  - [ ] **CP-Controller orchestrates** (automatic):
    - [ ] Migrations run for all Batch 1 tenants
    - [ ] Admin UI shows progress bar
    - [ ] Monitor progress: 0% → 100%
  - [ ] **After deployment**:
    - [ ] Verify routing changed
    - [ ] Spot check tenant access
    - [ ] Verify no errors in logs
    - [ ] Announce: "Batch 1 deployed"
  - [ ] **Monitoring** (24-48h):
    - [ ] Start continuous monitoring
    - [ ] Admin UI shows real-time metrics
    - [ ] Team watches for anomalies
  - **Acceptance**: Batch 1 deployed, all tenants on v1.0.0, initial validation passed
- **Duration**: Deployment: <1h, Monitoring: 24-48h (parallel with Batch 2 planning)
- **Next**: Task 3.3

---

### Task 3.3: Batch 1 Go/No-Go & Batch 2 Planning
- **Goal**: Decide whether to proceed to Batch 2, plan Batch 2 while monitoring Batch 1
- **Repos**: playhouse-cicd
- **Work**:
  - [ ] **While monitoring Batch 1** (parallel activity):
    - [ ] Continue monitoring Batch 1 every 4-8 hours
    - [ ] Prepare Batch 2 (same planning as Task 3.1)
    - [ ] Pre-migrate Batch 2 DBs (can start day 2 evening)
  - [ ] **Batch 1 Go/No-Go** (Tuesday morning, ~36-48h after deployment):
    - [ ] Review all Batch 1 metrics
    - [ ] Compare vs canary baseline
    - [ ] Check zero customer issues
    - [ ] Team votes: GO or ROLLBACK?
    - [ ] Document decision
  - [ ] **If GO**:
    - [ ] Announce: "Proceeding to Batch 2"
    - [ ] Update Batch 2 status: "Ready to deploy"
    - [ ] Proceed to Batch 2 deployment (same day or next morning)
  - [ ] **If NO-GO**:
    - [ ] Click "Rollback Batch 1" in Admin UI
    - [ ] Verify recovery complete
    - [ ] Post-mortem analysis
    - [ ] Plan retry for next week
  - **Acceptance**: Explicit go/no-go decision, Batch 2 ready if GO
- **Duration**: 1 day (decision point)
- **Next**: Task 3.4 (if GO)

---

### Task 3.4: Batch 2 & 3 Deployments
- **Goal**: Deploy Batches 2 and 3 following same pattern as Batch 1
- **Repos**: playhouse-cicd, playhouse-control-plane
- **Work**:
  - [ ] **Batch 2 (Wednesday)**:
    - [ ] Same procedure as Batch 1
    - [ ] Deploy ~11 tenants
    - [ ] Monitor 24-48h
    - [ ] Go/no-go decision Thursday morning
  - [ ] **Batch 3 (Friday)**:
    - [ ] Same procedure as Batch 1
    - [ ] Deploy ~22 tenants
    - [ ] Monitor 24-48h
    - [ ] Go/no-go decision Monday morning
  - [ ] **Monitoring throughout**:
    - [ ] Dashboard showing progress: Canary (10%) → Batch 1 (20%) → Batch 2 (35%) → Batch 3 (85%)
    - [ ] Each batch shows: tenants on v1.0.0, metrics (error rate, latency, uptime)
    - [ ] Team reviews metrics daily
  - **Acceptance**: Batch 2 deployed, stable, go decision made. Batch 3 deployed, stable, go decision made.
- **Duration**: Wed (Batch 2) → Fri (Batch 3) to Mon (go/no-go), ~5-6 days
- **Next**: Task 3.5

---

### Task 3.5: Batch 4 Deployment & Phase 3 Completion
- **Goal**: Deploy final batch (14% of tenants) and complete progressive rollout
- **Repos**: playhouse-cicd, playhouse-control-plane
- **Work**:
  - [ ] **Batch 4 (Monday+1)**:
    - [ ] Same procedure as previous batches
    - [ ] Deploy final ~7 tenants
    - [ ] Total now 100% on v1.0.0
    - [ ] Monitor 24-48h
  - [ ] **Final Validation**:
    - [ ] 100% of tenants on v1.0.0
    - [ ] All metrics healthy
    - [ ] Zero customer issues
    - [ ] All backups current
  - [ ] **Post-rollout Documentation**:
    - [ ] Document total deployment time
    - [ ] Capture lessons learned
    - [ ] Update runbooks with actual timings
    - [ ] Team retrospective
  - [ ] **Announcement**:
    - [ ] Announce: "Progressive rollout complete"
    - [ ] All tenants now on v1.0.0
    - [ ] Proceed to Phase 4 (retire v0.x.x)
  - **Acceptance**: 100% rollout complete, all batches successful, documentation updated
- **Duration**: Mon-Tue (deployment), 24-48h monitoring
- **Next**: Phase 4

---

## Phase 3 Exit Criteria

- ✓ Batch 1 deployed successfully (10%)
- ✓ Batch 2 deployed successfully (25%)
- ✓ Batch 3 deployed successfully (50%)
- ✓ Batch 4 deployed successfully (14%)
- ✓ 100% of tenants on v1.0.0
- ✓ Zero data loss/corruption across all batches
- ✓ All customers happy, zero critical issues
- ✓ Monitoring shows v1.0.0 production-ready
- ✓ Phase 3 sign-off from tech lead

---

## Timeline

| Task | Duration | Total |
|------|----------|-------|
| 3.1 | 1 day | 1 day |
| 3.2 | <1h deploy + 24-48h monitor | 2-3 days |
| 3.3 | 1 day (decision) | 3-4 days |
| 3.4 | Batch 2 + Batch 3 over 5-6 days | 8-10 days |
| 3.5 | 1 day + 24-48h monitor | **1-2 weeks** |

---

## Batch Monitoring Checklist (Repeat for Each Batch)

Every 2-4 hours (first 24h) then every 4-8h (next 24h):
- [ ] Server A & B CPU/memory normal?
- [ ] Error rates vs baseline? (should be ±0.5%)
- [ ] API latency vs baseline? (should be ±10%)
- [ ] DB connections healthy?
- [ ] Logs: any critical errors?
- [ ] Customer reports: any issues?
- [ ] Support tickets: any batch-related?
- [ ] Entitlements: observability metrics normal?

---

## Rollback Any Batch (If Issues Found)

If problems detected during batch monitoring:
1. Click "Rollback [Batch X]" in Admin UI
2. CP-Controller re-routes tenants to v0.x.x
3. Verify services recovering
4. Complete in <5 minutes
5. Analyze root cause
6. Plan retry with fixes

---

## Key Success Factors

✅ Conservative batch sizes (10% → 50% progression)  
✅ 24-48h monitoring between each batch  
✅ Go/no-go decision gates before proceeding  
✅ Explicit sign-off from tech lead each step  
✅ Team on-call throughout  
✅ Rollback always available  
✅ Transparent communication with customers  
