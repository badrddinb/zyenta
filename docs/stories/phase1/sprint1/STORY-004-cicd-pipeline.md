# Story 004: CI/CD Pipeline

## Story Information

| Field | Value |
|-------|-------|
| **Story ID** | STORY-004 |
| **Sprint** | Phase 1 - Sprint 1 |
| **Title** | GitHub Actions CI/CD Pipeline |
| **Priority** | High |
| **Story Points** | 5 |
| **Status** | Ready for Development |
| **Dependencies** | STORY-001 |

## User Story

**As a** developer
**I want** automated CI/CD pipelines with GitHub Actions
**So that** code quality is validated on every push and deployment is automated

## Acceptance Criteria

### AC1: CI Workflow Triggers
- [ ] Workflow triggers on push to `main` and `develop` branches
- [ ] Workflow triggers on pull requests to `main` and `develop`
- [ ] Concurrency is configured to cancel in-progress runs

### AC2: Environment Configuration
- [ ] Node.js version 20 is used
- [ ] pnpm version 8 is used
- [ ] Python version 3.11 is used for Python tests

### AC3: Lint & Type Check Job
- [ ] Checkout code
- [ ] Setup pnpm and Node.js with caching
- [ ] Install dependencies with frozen lockfile
- [ ] Generate Prisma client
- [ ] Run lint
- [ ] Run build (type check)
- [ ] Check formatting

### AC4: Node.js Test Job
- [ ] Depends on lint job
- [ ] PostgreSQL 15 service container is configured
- [ ] Redis 7 service container is configured
- [ ] Health checks for service containers
- [ ] Run database migrations
- [ ] Run tests with proper environment variables

### AC5: Python Test Job
- [ ] Depends on lint job
- [ ] PostgreSQL and Redis service containers
- [ ] Python 3.11 is setup
- [ ] Install Python dependencies from genesis-engine
- [ ] Run pytest with coverage
- [ ] Uses `continue-on-error: true` (for initial setup)

### AC6: Build Job
- [ ] Depends on test-node job
- [ ] Build all packages and applications
- [ ] Verify build output

### AC7: Deploy Workflow (Placeholder)
- [ ] Triggers on push to main
- [ ] Triggers on manual workflow_dispatch
- [ ] Uses production environment
- [ ] Placeholder steps for future deployment

## Technical Details

### Workflow Structure

```
CI Workflow (.github/workflows/ci.yml)
├── Lint & Type Check Job
│   ├── Checkout
│   ├── Setup pnpm + Node.js
│   ├── Install dependencies
│   ├── Generate Prisma
│   ├── Lint
│   ├── Build (type check)
│   └── Format check
│
├── Test Node.js Job (depends: lint)
│   ├── Services: postgres, redis
│   ├── Checkout
│   ├── Setup pnpm + Node.js
│   ├── Install dependencies
│   ├── Generate Prisma
│   ├── Run migrations
│   └── Run tests
│
├── Test Python Job (depends: lint)
│   ├── Services: postgres, redis
│   ├── Checkout
│   ├── Setup Python
│   ├── Install dependencies
│   └── Run pytest
│
└── Build Job (depends: test-node)
    ├── Checkout
    ├── Setup pnpm + Node.js
    ├── Install dependencies
    ├── Generate Prisma
    └── Build all

Deploy Workflow (.github/workflows/deploy.yml)
├── Triggers: push to main, workflow_dispatch
├── Environment: production
└── Placeholder steps
```

### Files to Create

1. **`.github/workflows/ci.yml`**
   - Complete CI workflow with all jobs
   - Service containers for testing
   - Proper caching and dependencies

2. **`.github/workflows/deploy.yml`**
   - Placeholder deployment workflow
   - Production environment configuration

### Service Container Configuration

**PostgreSQL:**
```yaml
image: postgres:15-alpine
env:
  POSTGRES_USER: test
  POSTGRES_PASSWORD: test
  POSTGRES_DB: test
ports:
  - 5432:5432
options: --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
```

**Redis:**
```yaml
image: redis:7-alpine
ports:
  - 6379:6379
options: --health-cmd "redis-cli ping" --health-interval 10s --health-timeout 5s --health-retries 5
```

### Environment Variables for Tests

```yaml
env:
  DATABASE_URL: postgresql://test:test@localhost:5432/test
  REDIS_URL: redis://localhost:6379
```

## Definition of Done

- [ ] CI workflow file is created and valid YAML
- [ ] Deploy workflow file is created and valid YAML
- [ ] Workflow runs successfully on push (may have expected failures for missing tests)
- [ ] All jobs execute in correct order (dependency chain)
- [ ] Service containers start and pass health checks
- [ ] Caching is working for pnpm
- [ ] Concurrency cancellation works
- [ ] Code review completed

## Testing

1. Push to a feature branch
2. Create PR to develop/main
3. Verify workflow is triggered
4. Check all jobs run in order
5. Verify service containers are healthy
6. Check build artifacts

## Notes

- Python test job uses `continue-on-error` since requirements.txt doesn't exist yet
- Actual tests will be added in later sprints
- Deploy workflow is a placeholder - actual deployment steps added later

## Related Stories

- Depends on: STORY-001 (Monorepo Setup)
- Related to: STORY-002 (Database Package - for generate/migrate)
