# Phase 2 - Task Breakdown

This phase is split into 6 focused tasks. Complete them sequentially.

## Phase 2 Overview
- **Goal**: Prepare production infrastructure and execute first canary deployment
- **Duration**: 1 week
- **Repos**: All
- **Prerequisites**: Phase 0 & 1 complete
- **End State**: Production ready with canary successfully deployed (5-10% of tenants)

## Tasks

### Task 2.1: Production Infrastructure Setup
- **Goal**: Provision and configure production servers and networks
- **Repos**: playhouse-cicd
- **Work**:
  - [ ] Provision production server(s):
    - [ ] Linux (Ubuntu 22.04+)
    - [ ] 8+ CPU, 32GB RAM, 500GB SSD
    - [ ] Docker + Docker Swarm installed
    - [ ] Security hardening applied
  - [ ] Configure networking:
    - [ ] Private network for internal services
    - [ ] Public network for nginx (80/443)
    - [ ] Security groups/firewall rules
  - [ ] Set up storage:
    - [ ] `/data/pgdata-registry/` mount
    - [ ] `/data/pgdata-tenant/` mount
    - [ ] `/data/backups/` mount
  - [ ] Configure system limits:
    - [ ] Open file descriptors: 65535+
    - [ ] Max connections: 4096+
    - [ ] Verify with benchmark test
  - **Acceptance**: Server ready, storage mounts tested, limits verified
- **Duration**: 1-2 days
- **Next**: Task 2.2

---

### Task 2.2: Backup & Disaster Recovery Setup
- **Goal**: Automate database backups and test recovery
- **Repos**: playhouse-cicd
- **Work**:
  - [ ] Create `scripts/backup-registry-db.sh`:
    - [ ] Daily full backup
    - [ ] 30-day retention
    - [ ] Off-site replication (S3/cloud)
  - [ ] Create `scripts/backup-tenant-db.sh`:
    - [ ] Daily incremental backup
    - [ ] 30-day retention
  - [ ] Create backup verification job:
    - [ ] Weekly restore test
    - [ ] Data integrity checks
    - [ ] Report generation
  - [ ] Add monitoring for backup success:
    - [ ] Alert if backup fails
    - [ ] Alert if backup slow
    - [ ] Alert if size anomaly
  - [ ] Test backup restore procedure
  - [ ] Document recovery procedure
  - **Acceptance**: Backups running daily, restore tested, monitoring working
- **Duration**: 1-2 days
- **Next**: Task 2.3

---

### Task 2.3: Monitoring & Alerting Setup
- **Goal**: Configure comprehensive production monitoring
- **Repos**: playhouse-cicd
- **Work**:
  - [ ] Choose monitoring tool (Datadog, Prometheus, etc.)
  - [ ] Set up dashboards for:
    - [ ] Service health (routing, servers, APIs)
    - [ ] Server metrics (CPU, memory, disk, network)
    - [ ] Application metrics (latency, errors, requests)
    - [ ] Database metrics (connections, query latency)
    - [ ] Entitlements metrics (denials, cache hit rate)
  - [ ] Configure alert thresholds:
    - [ ] Routing latency >500ms: WARNING
    - [ ] Error rate >5%: WARNING
    - [ ] Error rate >10%: CRITICAL
    - [ ] DB pool >80%: WARNING
    - [ ] Backup failed: CRITICAL
  - [ ] Set up logging aggregation
  - [ ] Enable distributed tracing (optional)
  - [ ] Create runbook for interpreting metrics
  - **Acceptance**: Dashboards live, alerts configured, team can interpret
- **Duration**: 2 days
- **Next**: Task 2.4

---

### Task 2.4: SSL/TLS, DNS, and Data Initialization
- **Goal**: Configure SSL certificates, DNS, and backfill production data
- **Repos**: playhouse-cicd
- **Work**:
  - [ ] SSL/TLS setup:
    - [ ] Obtain certificates (Let's Encrypt or purchased)
    - [ ] Install on nginx
    - [ ] Configure auto-renewal
    - [ ] Redirect HTTP → HTTPS
  - [ ] DNS configuration:
    - [ ] Set A records for pilche.ir, admin.pilche.ir
    - [ ] Set wildcard *.pilche.ir
    - [ ] Verify DNS propagation
  - [ ] Data initialization:
    - [ ] Backfill tenants with plan values
    - [ ] Create tenant_releases entries for all tenants
    - [ ] Create deployment_pools entries
    - [ ] Validate all tenants initialized
  - [ ] Canary tenant selection:
    - [ ] Choose 5-10% as canary
    - [ ] Document canary tenant list
    - [ ] Notify canary customers (optional)
  - **Acceptance**: SSL working (A+ rating), DNS resolving, all tenants initialized
- **Duration**: 1-2 days
- **Next**: Task 2.5

---

### Task 2.5: Pre-Canary Validation
- **Goal**: Final checks before canary deployment
- **Repos**: All
- **Work**:
  - [ ] Infrastructure verification:
    - [ ] All servers healthy
    - [ ] Storage performance good
    - [ ] Backups running successfully
    - [ ] Monitoring dashboards live
    - [ ] Logging aggregation working
  - [ ] Team readiness:
    - [ ] Team trained on procedures
    - [ ] On-call rotation set up
    - [ ] Incident response channel ready
    - [ ] Runbooks reviewed and accessible
  - [ ] Pre-flight check:
    - [ ] v0.x.x image available
    - [ ] v1.0.0 image available
    - [ ] docker-compose.prod.yml valid
    - [ ] Test: images pull successfully
  - [ ] Go/no-go meeting:
    - [ ] Review all checklist items
    - [ ] Team votes: proceed or delay?
    - [ ] Document decision
  - **Acceptance**: All checks passed, team confident, explicit go/no-go decision
- **Duration**: 1 day
- **Next**: Task 2.6

---

### Task 2.6: Canary Deployment & Monitoring
- **Goal**: Deploy canary to 5-10% of tenants and monitor for 24-48h
- **Repos**: playhouse-cicd, playhouse-control-plane
- **Work**:
  - [ ] **Deployment (Day 1)**:
    - [ ] Announce: "Canary deployment starting"
    - [ ] Backup canary tenant DBs
    - [ ] Login to Admin UI (`/admin-ui/deployments/`)
    - [ ] Click "Deploy Canary"
    - [ ] Select canary tenants
    - [ ] Confirm deployment
    - [ ] Monitor: migrations complete
    - [ ] Verify: tenants routed to server-b
    - [ ] Spot check: test tenant access
  - [ ] **Monitoring (24-48 hours)**:
    - [ ] Every 2-4 hours (first 24h), then 4-8h:
      - [ ] Check server health
      - [ ] Check error rates
      - [ ] Check latency
      - [ ] Check DB performance
      - [ ] Check logs for issues
      - [ ] Check customer reports
  - [ ] **Go/No-Go Decision (Day 3)**:
    - [ ] Review all metrics
    - [ ] Compare canary vs baseline
    - [ ] Team votes: proceed to Phase 3 or rollback?
    - [ ] If rollback: click "Rollback" in Admin UI
    - [ ] Document decision
  - **Acceptance**: Canary deployed 48h+, decision made with confidence
- **Duration**: 3 days
- **Next**: Phase 3

---

## Phase 2 Exit Criteria

- ✓ Production infrastructure provisioned
- ✓ Backups automated and tested
- ✓ Monitoring dashboards live
- ✓ SSL/TLS configured
- ✓ DNS working
- ✓ All tenants backfilled and initialized
- ✓ Pre-flight checks passed
- ✓ Canary deployed (5-10% of tenants)
- ✓ 24-48h monitoring completed
- ✓ Go/no-go decision made: GO
- ✓ Post-canary documentation complete

---

## Timeline

| Task | Duration | Total |
|------|----------|-------|
| 2.1 | 1-2 days | 1-2 days |
| 2.2 | 1-2 days | 2-4 days |
| 2.3 | 2 days | 4-6 days |
| 2.4 | 1-2 days | 5-8 days |
| 2.5 | 1 day | 6-9 days |
| 2.6 | 3 days | **1 week** |

---

## Critical Notes

⚠️ **Production traffic begins with canary deployment**
- Canary scope intentionally small (5-10%)
- Monitoring is critical during canary
- Ready to rollback if needed (<5 minutes)
- Team on-call and alert
- Document everything

---

## Rollback Procedure (If Needed)

If canary has issues:
1. Click "Rollback" in Admin UI
2. CP-Controller re-routes canary tenants to server-a
3. Verify services recover
4. All tenants back on v0.x.x
5. Complete in <5 minutes
6. Investigate root cause
7. Plan retry for next week
