# Production Deployment Plan for Pilche

## Overview

This plan outlines the deployment of Pilche to production with a blue-green multi-version strategy supporting safe, progressive tenant migration from v0.x.x to v1.0.0 and beyond.

## Architecture

**Key Principles:**
- Database per tenant (true isolation)
- Shared runtime (few server instances)
- Multi-version servers (Server A, Server B with different versions)
- Dynamic routing (tenants gradually migrate A → B)
- Database migrations per tenant (coordinated with routing)
- Dual web asset versions (static files for each version)

**Routing Flow:**
```
Browser → nginx (query routing service) 
       → Server A (v0.x.x) OR Server B (v1.0.0)
       ↓
    Registry DB stores routing policy
    (tenant_id → target_version)
```

## Phases

### Phase 0: Routing Service & Infrastructure (2-3 weeks)
Foundation for blue-green deployments. Implements routing service, registry schema updates, nginx configuration, and versioned static assets.

**Status:** Ready for implementation
**Repos:** playhouse-control-plane, playhouse-cicd, playhouse-docs

→ See: [phase-0-routing-and-infrastructure.md](phase-0-routing-and-infrastructure.md)

### Phase 1: Dual Server Setup & Testing (1-2 weeks)
Deploy both server versions in staging, validate routing, test database migrations, and rehearse rollback procedures.

**Status:** Ready for implementation (after Phase 0)
**Repos:** playhouse-server, playhouse-web, playhouse-cicd, playhouse-control-plane

→ See: [phase-1-dual-server-setup-and-testing.md](phase-1-dual-server-setup-and-testing.md)

### Phase 2: Production Preparation & First Canary (1 week)
Prepare production infrastructure, set up backups and monitoring, execute first canary deployment (5-10% of tenants).

**Status:** Ready for implementation (after Phase 1)
**Repos:** All

→ See: [phase-2-production-preparation-and-canary.md](phase-2-production-preparation-and-canary.md)

### Phase 3: Progressive Rollout (1-2 weeks)
Gradually migrate all tenants to v1.0.0 in 4 batches, monitoring at each step.

**Status:** Ready for implementation (after Phase 2 succeeds)
**Repos:** playhouse-cicd, playhouse-control-plane

→ See: [phase-3-progressive-rollout.md](phase-3-progressive-rollout.md)

### Phase 4: Retire Old Version (1 week)
Upgrade server-a to v1.0.0, retire v0.x.x, stabilize production.

**Status:** Ready for implementation (after Phase 3)
**Repos:** playhouse-cicd

→ See: [phase-4-retire-old-version.md](phase-4-retire-old-version.md)

## Timeline

| Phase | Duration | Cumulative |
|-------|----------|-----------|
| Phase 0 | 2-3 weeks | 2-3 weeks |
| Phase 1 | 1-2 weeks | 3-5 weeks |
| Phase 2 | 1 week | 4-6 weeks |
| Phase 3 | 1-2 weeks | 5-8 weeks |
| Phase 4 | 1 week | 6-9 weeks |

**Total: 6-9 weeks to production**

## Control-Plane Orchestration

**All deployments are triggered via the Admin UI and orchestrated by control-plane.**

→ See: [CONTROL-PLANE-ORCHESTRATION.md](CONTROL-PLANE-ORCHESTRATION.md)

**Key workflow:**
1. Admin UI: User selects version and tenants for deployment
2. CP-API: Receives request, validates, creates jobs
3. CP-Controller: Orchestrates DB migrations, updates routing
4. Routing Service: Redirects tenants to new server version
5. Admin UI: Shows real-time progress and metrics
6. Team: Monitors and makes go/no-go decisions

**No manual scripts needed** — all operations via Admin UI for safety and auditability.

---

## Key Concepts

### Tenant Releases
Track which version each tenant is routed to:
- tenant_id: Unique tenant
- target_version: "v1.0.0" or "v0.x.x"
- db_schema_version: Current schema version for tenant's DB
- status: pending, in_progress, completed, rolled_back
- migrated_at: When migration happened

### Deployment Pools
Track where each version is running:
- version: "v1.0.0", "v0.x.x"
- server_host: "server-b:8000"
- server_port: 8000
- web_assets_path: "/assets/v1.0.0"
- status: active, draining, idle

### Routing Service
HTTP service that queries registry and returns routing decision:
```
GET /route?tenant=acme-corp
→ {
    target_server: "server-b:8000",
    web_version: "v1.0.0",
    db_schema_version: 1
  }
```

In-memory cache (60s TTL) for performance.

### Schema Versioning
Database schemas evolved without deletions, only additions with defaults:
- v0: Original schema (playhouse-server/migrations/0001_initial.py)
- v1: Added new columns (entitlements, subscriptions)
- Both v0 and v1 servers can read v1 schema

### Static Assets
Versioned asset paths for web app:
- `/assets/v0.x.x/app-abc123.js`
- `/assets/v1.0.0/app-def456.js`

Browser requests go to nginx, which routes to correct asset version based on tenant.

## Success Criteria

✓ All phases complete
✓ 100% of tenants on v1.0.0
✓ Zero data loss/corruption
✓ Production stable for 7 days
✓ Team confident in procedures
✓ Runbooks updated and tested

## Risks & Mitigations

| Risk | Mitigation |
|------|-----------|
| Routing service becomes bottleneck | Cache heavily (60s TTL), use Redis cache for v2 |
| Schema migration causes data loss | Test with production-scale data, migration rollback tested |
| Version mismatch API errors | Comprehensive backward compat tests |
| Cross-tenant data bleed | Unit tests, isolation tests, audit logs |
| DNS/SSL certificate issues | Test with production domain before deploy |
| Tenant DB connection pool exhaustion | Monitor connection pool, implement circuit breaker |
| Rollout causes tenant downtime | Gradual rollout, health checks at each batch |

## Checklist: Ready to Start

- [ ] Phase 0 plan reviewed and understood
- [ ] Team assigned to each phase
- [ ] Production server/infrastructure budgeted
- [ ] Backup storage provisioned
- [ ] Monitoring tool selected (Datadog, Prometheus, etc.)
- [ ] SSL certificates obtained or Let's Encrypt configured
- [ ] DNS provider access verified
- [ ] Disaster recovery plan documented
- [ ] Rollback procedures practiced in staging
- [ ] Go/no-go meeting scheduled for Phase 0 kickoff

## Next Steps

1. Start Phase 0 implementation
2. Set up code review process for infrastructure changes
3. Schedule weekly sync meetings during each phase
4. Create incident response channel
5. Document all deployment decisions for team wiki

## Questions?

Refer to individual phase documentation or ask the team.
