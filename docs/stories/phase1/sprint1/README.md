# Phase 1 - Sprint 1: Foundation

## Sprint Overview

| Field | Value |
|-------|-------|
| **Sprint** | 1 |
| **Phase** | 1 - "Creator" MVP |
| **Duration** | 2 weeks |
| **Total Story Points** | 39 |
| **Status** | Ready for Development |

## Sprint Goal

Establish the complete project foundation including monorepo configuration, database schema, local development infrastructure, CI/CD pipeline, and shared packages that all subsequent development depends on.

## Stories

| ID | Title | Priority | Points | Dependencies | Status |
|----|-------|----------|--------|--------------|--------|
| [STORY-001](./STORY-001-monorepo-setup.md) | Monorepo Setup | Critical | 8 | None | Ready |
| [STORY-002](./STORY-002-database-package.md) | Database Package | Critical | 8 | STORY-001 | Ready |
| [STORY-003](./STORY-003-docker-compose.md) | Docker Compose Infrastructure | High | 5 | STORY-001 | Ready |
| [STORY-004](./STORY-004-cicd-pipeline.md) | CI/CD Pipeline | High | 5 | STORY-001 | Ready |
| [STORY-005](./STORY-005-queue-package.md) | Queue Package | High | 5 | STORY-001, STORY-003 | Ready |
| [STORY-006](./STORY-006-shared-types-package.md) | Shared Types Package | High | 5 | STORY-001 | Ready |
| [STORY-007](./STORY-007-ui-package.md) | UI Package Foundation | Medium | 3 | STORY-001 | Ready |

## Dependency Graph

```
STORY-001 (Monorepo Setup)
    │
    ├──► STORY-002 (Database Package)
    │
    ├──► STORY-003 (Docker Compose) ──► STORY-005 (Queue Package)
    │                                        │
    ├──► STORY-004 (CI/CD Pipeline)          │
    │                                        │
    ├──► STORY-005 (Queue Package) ◄─────────┘
    │
    ├──► STORY-006 (Shared Types Package)
    │
    └──► STORY-007 (UI Package Foundation)
```

## Suggested Implementation Order

1. **STORY-001** - Monorepo Setup (blocks all others)
2. **STORY-003** - Docker Compose Infrastructure (needed for database testing)
3. **STORY-002** - Database Package (needs Docker for testing)
4. **STORY-004** - CI/CD Pipeline (can run in parallel with Story 2)
5. **STORY-005** - Queue Package (needs Redis from Docker)
6. **STORY-006** - Shared Types Package (can run in parallel)
7. **STORY-007** - UI Package Foundation (can run in parallel)

## Definition of Done for Sprint

- [ ] All 7 stories completed and code reviewed
- [ ] `pnpm install` works from fresh clone
- [ ] `pnpm build` builds all packages successfully
- [ ] `pnpm lint` passes without errors
- [ ] `pnpm test` runs (may have no tests yet)
- [ ] Docker infrastructure starts with `./scripts/dev-infra.sh start`
- [ ] Database migrations run successfully
- [ ] CI pipeline passes on push
- [ ] All packages can be imported by consuming applications

## Technical Stack Summary

| Component | Technology |
|-----------|------------|
| Package Manager | pnpm 8.15+ |
| Build Orchestration | Turborepo 2.0+ |
| Language | TypeScript 5.3+, Python 3.11+ |
| Database | PostgreSQL 15 (via Prisma) |
| Cache/Queue | Redis 7 (via BullMQ) |
| Message Broker | RabbitMQ 3 |
| Object Storage | MinIO (S3-compatible) |
| CI/CD | GitHub Actions |
| Validation | Zod |
| Styling Utilities | Tailwind CSS utilities |

## Verification Checklist

After completing all stories:

### Monorepo
- [ ] `pnpm install` completes without errors
- [ ] `pnpm build` builds all packages
- [ ] `pnpm lint` runs without errors
- [ ] Husky pre-commit hook is triggered on commit

### Database
- [ ] `pnpm db:generate` creates Prisma client
- [ ] `pnpm db:push` syncs schema to database
- [ ] `pnpm db:studio` opens Prisma Studio

### Infrastructure
- [ ] `./scripts/dev-infra.sh start` launches all services
- [ ] PostgreSQL is accessible on port 5432
- [ ] Redis is accessible on port 6379
- [ ] RabbitMQ management UI is accessible on port 15672
- [ ] MinIO console is accessible on port 9001

### CI/CD
- [ ] GitHub Actions workflow runs on push
- [ ] All CI jobs pass

## Next Sprint

After completing Sprint 1, proceed to **Sprint 2: Genesis Engine Core**:
- FastAPI project setup
- Base agent class implementation
- Brand Agent with LLM integration
- Product Scout Agent with supplier integration
- Copywriter Agent

See `SPRINT2_GENESIS_ENGINE_ACTION_PLAN.md` for details.

## Notes

- This sprint has no external dependencies
- Focus on getting the foundation right - all future work depends on it
- Test each component thoroughly before moving to the next
- Document any deviations from the plan
