# Production Deployment - Phase 2: Production Preparation & First Canary

## Goal

Prepare real production infrastructure, set up monitoring and backups, execute first canary deployment with 5-10% of real tenants, and gather baseline metrics before progressive rollout.

## Repos

- All (production infrastructure setup)

## Duration

1 week

## Overview

This phase:
1. Provisions production servers
2. Sets up automated backups
3. Configures monitoring and alerting
4. Backfills tenant data
5. Executes first canary deployment
6. Monitors canary for 24-48 hours
7. Makes go/no-go decision for Phase 3

## Prerequisites (From Phase 1)

- ✓ Dual server setup tested in staging
- ✓ Routing verified working
- ✓ Database migrations tested
- ✓ Backward compatibility validated
- ✓ Rollback procedure tested
- ✓ Deployment scripts working
- ✓ Runbooks complete
- ✓ Team trained and confident

## Work Items

### 1. Production Server Infrastructure

**What:** Set up production hardware/cloud infrastructure.

**Changes:**
- [ ] Provision production server(s):
  - [ ] Linux server (Ubuntu 22.04 LTS or CentOS 8+)
  - [ ] Minimum specs: 8 CPU, 32GB RAM, 500GB SSD
  - [ ] Docker + Docker Swarm installed
  - [ ] Security hardening applied (firewall, SSH, sudo, etc.)
  - [ ] Automatic security updates configured

- [ ] Configure networking:
  - [ ] Private network for internal services (registry-db, server, pgbouncer)
  - [ ] Public network for nginx (80/443)
  - [ ] Security groups/firewall rules limiting access

- [ ] Set up persistent storage:
  - [ ] Mount for registry DB data: `/data/pgdata-registry/`
  - [ ] Mount for tenant DB data: `/data/pgdata-tenant/`
  - [ ] Mount for backups: `/data/backups/`
  - [ ] All mounted on separate filesystems for performance

- [ ] Configure system limits:
  - [ ] Open file descriptors: 65535+
  - [ ] Max connections: 4096+
  - [ ] Swap: minimal or disabled

**Acceptance Criteria:**
- Server provisioned and accessible
- Docker running and ready
- Storage mounts tested with I/O benchmarks
- Security hardening validated

---

### 2. Database Backups

**What:** Automate registry and tenant database backups.

**Changes:**
- [ ] Set up registry DB backup script: `scripts/backup-registry-db.sh`
  ```bash
  #!/bin/bash
  # Daily backup at 2 AM UTC
  pg_dump -h registry-db -U registry -d registry \
    > /data/backups/registry-$(date +%Y%m%d-%H%M%S).sql.gz
  
  # Retention: keep 30 days
  find /data/backups/ -name 'registry-*.sql.gz' -mtime +30 -delete
  ```

- [ ] Set up tenant DB backup script: `scripts/backup-tenant-db.sh`
  ```bash
  #!/bin/bash
  # Daily incremental backup at 3 AM UTC
  pg_basebackup -D /data/backups/tenant-$(date +%Y%m%d-%H%M%S) \
    -U postgres -X stream -P
  ```

- [ ] Configure backup rotation and retention policies
  - [ ] Daily full backups
  - [ ] Keep 7 daily + 4 weekly + 12 monthly
  - [ ] Off-site replication to S3 or backup service

- [ ] Add backup verification job:
  ```bash
  #!/bin/bash
  # Weekly restore test (Wednesday midnight)
  # Restore to temporary instance
  # Run basic connectivity tests
  # Verify data integrity
  # Report results
  ```

- [ ] Add monitoring for backup success:
  - [ ] Alert if backup fails
  - [ ] Alert if backup doesn't complete within SLA (e.g., 1h)
  - [ ] Alert if backup size anomaly (too large/small)

**Acceptance Criteria:**
- Backups run daily without errors
- Restore procedure tested and working
- Offsite replication working
- Monitoring alerting on backup failures

---

### 3. Monitoring & Alerting Setup

**What:** Configure comprehensive monitoring for production.

**Changes:**
- [ ] Choose monitoring tool (Datadog, Prometheus, New Relic, etc.)

- [ ] Set up dashboards for:
  - [ ] **Service Health**:
    - [ ] Routing service latency (p50, p95, p99)
    - [ ] Routing service error rate
    - [ ] Server-a availability (uptime %)
    - [ ] Server-b availability (uptime %)
    - [ ] Registry DB connections
    - [ ] Tenant DB connections (per server)
  
  - [ ] **Server Metrics**:
    - [ ] CPU usage (server-a vs server-b)
    - [ ] Memory usage
    - [ ] Disk I/O
    - [ ] Network traffic (ingress/egress)
  
  - [ ] **Application Metrics**:
    - [ ] API response times (by endpoint, by server)
    - [ ] Request rate (requests/sec)
    - [ ] Error rate (500s, 400s, etc.)
    - [ ] Database query latency (by server)
    - [ ] Feature denial count (observability mode)
  
  - [ ] **Entitlements**:
    - [ ] Entitlement cache hit rate
    - [ ] Config version fetch latency
    - [ ] Stale cache detection (version mismatch)
  
  - [ ] **Deployment Progress**:
    - [ ] % of tenants on v1.0.0
    - [ ] Batch rollout progress
    - [ ] Rollback trigger alerts

- [ ] Configure alert thresholds:
  - [ ] Routing service latency >500ms: WARNING
  - [ ] Routing service latency >2s: CRITICAL (trigger rollback)
  - [ ] Server error rate >5%: WARNING
  - [ ] Server error rate >10%: CRITICAL (trigger rollback)
  - [ ] DB connection pool >80%: WARNING
  - [ ] DB connection pool exhausted: CRITICAL (page on-call)
  - [ ] Feature denial spike >2x baseline: CRITICAL (investigate)
  - [ ] Backup failed: CRITICAL (page on-call)

- [ ] Set up logging aggregation:
  - [ ] All container logs → centralized logging (ELK, Loki, CloudWatch, etc.)
  - [ ] Application logs tagged with:
    - [ ] Service name (server-a, server-b, routing-service)
    - [ ] Tenant ID (for audit trail)
    - [ ] Request ID (for tracing)
  - [ ] Log retention: 30 days minimum

- [ ] Enable distributed tracing (optional but recommended):
  - [ ] Request spans from nginx → routing service → server → DB
  - [ ] Latency breakdown per component

**Acceptance Criteria:**
- Dashboards live and showing real data
- Alerts configured with appropriate thresholds
- Logging aggregation working
- Team can interpret metrics

---

### 4. SSL/TLS Setup

**What:** Configure HTTPS for production domains.

**Changes:**
- [ ] Obtain SSL certificates:
  - [ ] Option A: Let's Encrypt (free, auto-renew)
    ```bash
    certbot certonly --standalone \
      -d pilche.ir \
      -d admin.pilche.ir \
      -d '*.pilche.ir'
    ```
  - [ ] Option B: Purchased certificates (DigiCert, etc.)

- [ ] Configure nginx with SSL:
  ```nginx
  server {
    listen 443 ssl http2;
    server_name pilche.ir admin.pilche.ir *.pilche.ir;
    
    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;
    
    # Security best practices
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
  }
  ```

- [ ] Redirect HTTP → HTTPS
- [ ] Set up auto-renewal (if Let's Encrypt)
- [ ] Add HSTS header (Strict-Transport-Security)

**Acceptance Criteria:**
- SSL certificates obtained
- HTTPS working for all domains
- SSL test passes (A+ rating on SSL Labs)
- Auto-renewal working

---

### 5. DNS Configuration

**What:** Configure DNS for production domains.

**Changes:**
- [ ] Set up DNS A records:
  - [ ] pilche.ir → production IP
  - [ ] admin.pilche.ir → production IP (or subdomain)
  - [ ] *.pilche.ir → production IP (wildcard for tenant subdomains)

- [ ] Verify DNS propagation:
  ```bash
  dig pilche.ir
  dig admin.pilche.ir
  dig test.pilche.ir  # tenant subdomain
  ```

- [ ] Test subdomain routing:
  ```bash
  curl http://test.pilche.ir:2003 -H "Host: test.pilche.ir"
  curl http://admin.pilche.ir:2003 -H "Host: admin.pilche.ir"
  ```

**Acceptance Criteria:**
- DNS records pointing to production IP
- Wildcard subdomain resolution working
- All domains resolve within 100ms

---

### 6. Data Backfill & Tenant Initialization

**What:** Prepare registry with all existing tenants and routing policies.

**Changes:**
- [ ] Backfill existing tenants with plan values (from Phase 5 requirements):
  ```bash
  # For each existing tenant:
  # - Create plan assignment
  # - Create entitlements snapshot
  # - Mark as initialized
  ```

- [ ] Create tenant_releases entries for ALL tenants:
  ```sql
  INSERT INTO tenant_releases (tenant_id, target_version, db_schema_version, status)
  SELECT tenant_id, 'v0.x.x', 0, 'completed'
  FROM tenants;
  ```

- [ ] Create deployment_pools entries:
  ```sql
  INSERT INTO deployment_pools (version, server_host, server_port, web_assets_path, status)
  VALUES
    ('v0.x.x', 'server-a:8000', 8000, '/assets/v0.x.x', 'active'),
    ('v1.0.0', 'server-b:8000', 8001, '/assets/v1.0.0', 'idle');
  ```

- [ ] Validate all tenants have:
  - [ ] tenant_releases entry (routing assigned)
  - [ ] Explicit plan value (from pricing tables)
  - [ ] No unresolved entitlements

**Acceptance Criteria:**
- All existing tenants in database
- All tenants have routing policy assigned
- All tenants have explicit plan value
- Data validation checks pass

---

### 7. Canary Tenant Selection

**What:** Choose 5-10% of real tenants for canary deployment.

**Changes:**
- [ ] Identify canary tenants:
  - [ ] 5-10% of customer base
  - [ ] Mix of:
    - [ ] Early adopters (comfortable with beta)
    - [ ] Internal test tenants
    - [ ] Small/medium customers (lower risk)
  - [ ] Exclude: Large customers, critical accounts, recently on-boarded

- [ ] Example: 3 canary tenants if 30-50 total tenants
  - [ ] tenant-1 (small, early adopter)
  - [ ] tenant-2 (internal test)
  - [ ] tenant-3 (medium, willing to test)

- [ ] Document canary tenant list (for reference during rollout)

**Acceptance Criteria:**
- Canary tenants identified and documented
- Team aware of canary scope
- No surprises during rollout

---

### 8. Pre-Canary Checklist

**What:** Final checks before deploying canary.

**Changes:**
- [ ] Verify all infrastructure ready:
  - [ ] Production servers provisioned and healthy
  - [ ] Networks and firewalls configured
  - [ ] Storage mounts tested (I/O performance good)
  - [ ] Backups running successfully
  - [ ] Monitoring dashboards live
  - [ ] Logging aggregation working
  - [ ] SSL certificates installed
  - [ ] DNS propagated

- [ ] Verify all team readiness:
  - [ ] Team trained on deployment procedures
  - [ ] On-call rotation set up
  - [ ] Incident response channel ready
  - [ ] All runbooks reviewed and accessible
  - [ ] Escalation contacts known

- [ ] Verify all data ready:
  - [ ] Tenants backfilled with plan values
  - [ ] Routing policies initialized
  - [ ] Canary tenants identified
  - [ ] Backup validation passed

- [ ] Verify all images ready:
  - [ ] v0.x.x image tagged and available
  - [ ] v1.0.0 image tagged and available
  - [ ] Routing service image available
  - [ ] All images pull successfully in production

**Acceptance Criteria:**
- Go/no-go meeting scheduled
- All checklist items confirmed
- Team confident to proceed

---

### 9. Canary Deployment Execution

**What:** Deploy canary batch (5-10% of tenants) to v1.0.0 via control-plane API and admin-ui.

**Changes:**
- [ ] **Pre-Migration (Day 1 morning)**:
  - [ ] Backup all canary tenant DBs
  - [ ] Announce to team: "Canary deployment starting"
  - [ ] Take baseline metrics screenshot (for comparison)
  - [ ] Log in to Admin UI: `/admin-ui/deployments/`

- [ ] **Via Admin UI - Create Release**:
  - [ ] Navigate to `/admin-ui/releases/`
  - [ ] Click "Create New Release"
  - [ ] Select version: v1.0.0
  - [ ] Set status: "pending"
  - [ ] Add notes: "First canary deployment"
  - [ ] Click "Create"

- [ ] **Via Admin UI - Start Canary Deployment**:
  - [ ] Navigate to `/admin-ui/deployments/`
  - [ ] Click "Deploy v1.0.0" button
  - [ ] Wizard Step 1: Confirm version v1.0.0
  - [ ] Wizard Step 2: Select canary tenants (canary_tenant_1, canary_tenant_2, canary_tenant_3)
  - [ ] Wizard Step 3: Review and confirm
  - [ ] Behind UI: Control-plane API triggers:
    ```json
    POST /api/v1/releases/v1.0.0/rollouts
    {
      "batch_number": 0,
      "tenant_ids": ["canary_tenant_1", "canary_tenant_2", "canary_tenant_3"],
      "target_server": "server-b:8000",
      "db_schema_version": 1,
      "scheduled_start": "2026-06-01T16:30:00Z",
      "monitoring_window_hours": 48
    }
    ```

- [ ] **Control-Plane Orchestration** (automatic):
  1. cp-controller receives rollout request
  2. Creates db migration job for each canary tenant:
     ```
     Job: migrate_tenant_schema
     Tenant: canary_tenant_1
     Target Schema: v1
     ```
  3. Waits for all migrations to complete
  4. Updates tenant_releases in registry:
     ```sql
     UPDATE tenant_releases
     SET target_version = 'v1.0.0', status = 'in_progress'
     WHERE tenant_id IN ('canary_tenant_1', 'canary_tenant_2', 'canary_tenant_3');
     ```
  5. Routing service picks up changes (cache invalidates after 60s)
  6. Tenants now routed to server-b

- [ ] **Verify Deployment** (via Admin UI):
  - [ ] Navigate to `/admin-ui/deployments/`
  - [ ] View "Release v1.0.0" status: "canary"
  - [ ] Canary tenants column shows 3 tenants
  - [ ] Real-time metrics updating
  - [ ] All services healthy (green indicators)

- [ ] **Start Monitoring** (Day 1 evening):
  - [ ] Admin UI dashboard shows real-time metrics
  - [ ] All team members watching dashboards
  - [ ] Log aggregation streaming
  - [ ] Alert notifications enabled
  - [ ] On-call engineer ready

**Acceptance Criteria**:
- Canary tenants successfully routed to server-b via control-plane orchestration
- All services healthy
- Initial requests/responses normal
- Monitoring data flowing
- Admin UI accurately shows deployment status

---

### 10. Canary Monitoring (24-48 hours)

**What:** Intensive monitoring of canary deployment.

**Canary Monitoring Checklist** (check every 2-4 hours):

- [ ] **Server Metrics**:
  - [ ] Server-a CPU/memory normal
  - [ ] Server-b CPU/memory normal (should be similar)
  - [ ] Disk I/O normal
  - [ ] Network traffic within expectations

- [ ] **Routing**:
  - [ ] Canary tenants consistently routed to server-b
  - [ ] Non-canary tenants still routed to server-a
  - [ ] Routing latency <200ms p95

- [ ] **Application Health**:
  - [ ] No increase in error rates (5xx errors <1%)
  - [ ] API response times stable (no >2x increase)
  - [ ] Database queries performing (no slowdown)
  - [ ] Web app loading correctly for canary tenants

- [ ] **Entitlements** (observability mode):
  - [ ] Feature denials logged correctly (not enforced yet)
  - [ ] Config version fetch latency normal
  - [ ] Cache hit rate >95%
  - [ ] No version mismatches

- [ ] **Data Integrity**:
  - [ ] No data loss or corruption
  - [ ] Cross-tenant isolation maintained
  - [ ] Audit logs clean (no unexpected errors)

- [ ] **Customer Impact**:
  - [ ] Canary customers reporting no issues
  - [ ] Support tickets for canary tenants: 0
  - [ ] Performance complaints: 0

**Monitoring Timeline:**
- **Hour 0**: Deployment complete, all systems green
- **Hour 1**: Initial spike normal, everything settling
- **Hour 4**: First full cycle of activity, validate stability
- **Hour 8**: Overnight activity, monitor for any nocturnal issues
- **Hour 24**: Full day cycle complete, make interim assessment
- **Hour 48**: Full 48h cycle, ready for go/no-go decision

---

### 11. Canary Go/No-Go Decision

**What:** Decide whether to proceed with Phase 3 (progressive rollout).

**Go/No-Go Criteria** (all must pass):

- ✓ **Availability**: Canary server availability = 99.9%+ (max 43s downtime)
- ✓ **Error Rate**: Canary server error rate = v0.x.x ± 0.5% (no significant increase)
- ✓ **Latency**: API response times within baseline ±10%
- ✓ **Data Integrity**: Zero data loss or corruption detected
- ✓ **Customer Reports**: Zero critical issues from canary tenants
- ✓ **Support Tickets**: Zero canary-related support tickets
- ✓ **Entitlements**: Observability metrics show expected pattern
- ✓ **Monitoring**: All dashboards show healthy metrics

**Decision Process:**
```
Day 3, 10 AM (48h after canary):
1. Tech lead reviews all metrics
2. On-call engineer confirms no issues in logs
3. Product owner checks with canary customers: "How's it going?"
4. Team votes: proceed or rollback?
5. Decision documented in Slack/ticket
```

**If GO → Proceed to Phase 3** (progressive rollout)

**If NO-GO → Execute Rollback** (via Admin UI):
- [ ] Navigate to `/admin-ui/releases/v1.0.0/`
- [ ] Click "Rollback Canary" button
- [ ] Confirm in dialog: "Rollback canary tenants to v0.x.x?"
- [ ] Behind UI: Control-plane API triggers:
  ```json
  POST /api/v1/releases/v1.0.0/rollback
  {
    "batch_number": 0,
    "reason": "Critical issue detected in canary",
    "rollback_to_version": "v0.x.x"
  }
  ```

- [ ] Control-plane orchestration:
  1. cp-controller updates all canary tenant_releases
  2. Sets target_version back to 'v0.x.x'
  3. Sets status to 'rolled_back'
  4. Routing service invalidates cache
  5. Tenants re-route to server-a automatically

- [ ] Verify rollback (via Admin UI):
  - [ ] Release status shows: "Canary rolled back"
  - [ ] All canary tenants show: 0% on v1.0.0
  - [ ] Metrics recovering to baseline
  - [ ] Estimated time: <5 minutes

- [ ] Post-mortem:
  - [ ] Identify root cause
  - [ ] Document issue
  - [ ] Fix and plan retry (next week)

**Acceptance Criteria:**
- Explicit go/no-go decision made
- Decision documented
- Team aligned on next steps

---

### 12. Post-Canary Documentation

**What:** Document learnings and update runbooks.

**Changes:**
- [ ] Create canary post-mortem (even if successful):
  - [ ] What went well?
  - [ ] What could be better?
  - [ ] Any surprises?
  - [ ] Metrics vs. expectations?

- [ ] Update runbooks based on experience:
  - [ ] Timing estimates (were they accurate?)
  - [ ] Step clarity (any confusion?)
  - [ ] Rollback procedure (did it work as expected?)
  - [ ] Monitoring dashboards (helpful or confusing?)

- [ ] Capture metrics baseline:
  - [ ] v0.x.x baseline (from server-a)
  - [ ] v1.0.0 baseline (from server-b canary)
  - [ ] Comparison chart for Phase 3 batches

- [ ] Team feedback:
  - [ ] Collect what the team learned
  - [ ] Any training gaps?
  - [ ] Update procedures accordingly

**Acceptance Criteria:**
- Post-mortem documented
- Runbooks updated
- Baseline metrics captured
- Team feedback incorporated

---

## Exit Criteria

- ✓ Production infrastructure provisioned and tested
- ✓ Automated backups running daily
- ✓ Monitoring dashboards live and healthy
- ✓ SSL/TLS configured
- ✓ DNS working
- ✓ All tenants backfilled and initialized
- ✓ Canary deployment successful
- ✓ 24-48h monitoring completed
- ✓ Go/no-go decision made (GO)
- ✓ Post-canary documentation complete
- ✓ Team confident to proceed to Phase 3

---

## Risks & Mitigations

| Risk | Mitigation |
|------|-----------|
| Production infrastructure issues | Pre-flight checks, staging mirror |
| Backup/restore fails in production | Tested backup restore procedure before canary |
| Monitoring too noisy (false alerts) | Tuned alert thresholds in staging |
| Canary deployment uncovers unknown bugs | Staged rollout (canary first) |
| Customer impact from canary bugs | Small canary size (5-10%), ready rollback |
| On-call team overwhelmed | Limit canary scope, have backup on-call |

---

## Success Metrics

| Metric | Target |
|--------|--------|
| Infrastructure setup time | <2 days |
| Backup automation success | 100% |
| Monitoring alert accuracy | >95% (false positives <5%) |
| Canary deployment time | <2 hours |
| Canary availability | 99.9%+ |
| Canary error rate delta | <0.5% vs v0.x.x |
| Customer issues during canary | 0 |
| Go/no-go decision time | <1 hour after 48h observation |

---

## Next Phase

Phase 3 focuses on progressive rollout across all remaining tenants in 4 batches.

**Phase 3 can only start after Phase 2 exit criteria all met (GO decision).**

---

## Timeline

| Day | Activity |
|-----|----------|
| 1 | Infrastructure setup, monitoring config |
| 2 | SSL/TLS, DNS, data backfill |
| 3 | Canary tenant selection, pre-flight checks |
| 4 | Canary deployment (DBs migrate, server-b deploy, routing) |
| 5-6 | Canary monitoring (24h full cycle) |
| 6 | Canary monitoring (continued, 48h total) |
| 7 | Go/no-go decision, post-canary doc |

---

## Notes

- Production traffic starts with canary deployment
- Canary scope intentionally small (5-10%) to limit blast radius
- Monitoring is critical — tune thresholds in staging first
- Ready rollback procedure within 5 minutes
- Team on-call and alert throughout
- Document everything for audit trail
