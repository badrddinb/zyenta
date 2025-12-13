# Story 003: Docker Compose Infrastructure

## Story Information

| Field | Value |
|-------|-------|
| **Story ID** | STORY-003 |
| **Sprint** | Phase 1 - Sprint 1 |
| **Title** | Docker Compose Local Development Infrastructure |
| **Priority** | High |
| **Story Points** | 5 |
| **Status** | Ready for Development |
| **Dependencies** | STORY-001 |

## User Story

**As a** developer
**I want** a Docker Compose configuration for local development infrastructure
**So that** I can run all required services (PostgreSQL, Redis, RabbitMQ, MinIO) locally with a single command

## Acceptance Criteria

### AC1: PostgreSQL Service
- [ ] PostgreSQL 15 Alpine image is used
- [ ] Port 5432 is exposed to host
- [ ] Credentials: user=zyenta, password=zyenta_dev, database=zyenta
- [ ] Data is persisted in a named volume
- [ ] Health check is configured

### AC2: Redis Service
- [ ] Redis 7 Alpine image is used
- [ ] Port 6379 is exposed to host
- [ ] Data persistence is enabled (appendonly)
- [ ] Health check is configured

### AC3: RabbitMQ Service
- [ ] RabbitMQ 3 Management Alpine image is used
- [ ] AMQP port 5672 is exposed
- [ ] Management UI port 15672 is exposed
- [ ] Credentials: user=zyenta, password=zyenta_dev
- [ ] Data is persisted in a named volume
- [ ] Health check is configured

### AC4: MinIO Service
- [ ] MinIO latest image is used
- [ ] API port 9000 is exposed
- [ ] Console port 9001 is exposed
- [ ] Root credentials: minioadmin/minioadmin
- [ ] Data is persisted in a named volume
- [ ] Health check is configured

### AC5: MinIO Setup Service
- [ ] One-time setup container creates default bucket
- [ ] Bucket name: `zyenta-media`
- [ ] Public read access is configured
- [ ] Service waits for MinIO to be healthy before running

### AC6: Network Configuration
- [ ] All services are on a shared bridge network
- [ ] Network name: `zyenta-network`
- [ ] Services can communicate by container name

### AC7: Helper Script
- [ ] `scripts/dev-infra.sh` is created
- [ ] Script supports: start, stop, restart, logs, status, clean commands
- [ ] Script is executable (chmod +x)
- [ ] Clear output messages showing service URLs
- [ ] `clean` command removes all data volumes

## Technical Details

### Service Ports

| Service | Internal Port | Host Port | Description |
|---------|--------------|-----------|-------------|
| PostgreSQL | 5432 | 5432 | Database |
| Redis | 6379 | 6379 | Cache & Queue |
| RabbitMQ | 5672 | 5672 | AMQP |
| RabbitMQ | 15672 | 15672 | Management UI |
| MinIO | 9000 | 9000 | S3 API |
| MinIO | 9001 | 9001 | Web Console |

### Files to Create

1. **`infra/docker/docker-compose.dev.yml`**
   - Complete Docker Compose configuration
   - All five services defined
   - Volumes and networks configured

2. **`scripts/dev-infra.sh`**
   - Bash helper script for infrastructure management
   - Commands: start, stop, restart, logs, status, clean

### Volume Names

- `postgres_data` - PostgreSQL data
- `redis_data` - Redis AOF data
- `rabbitmq_data` - RabbitMQ data
- `minio_data` - MinIO object storage

### Health Checks

| Service | Command | Interval | Timeout | Retries |
|---------|---------|----------|---------|---------|
| PostgreSQL | `pg_isready -U zyenta -d zyenta` | 5s | 5s | 5 |
| Redis | `redis-cli ping` | 5s | 5s | 5 |
| RabbitMQ | `rabbitmq-diagnostics check_running` | 10s | 5s | 5 |
| MinIO | `mc ready local` | 10s | 5s | 5 |

## Definition of Done

- [ ] Docker Compose file is created and valid
- [ ] Helper script is created and executable
- [ ] `./scripts/dev-infra.sh start` launches all services
- [ ] All services pass health checks
- [ ] PostgreSQL is accessible on port 5432
- [ ] Redis is accessible on port 6379
- [ ] RabbitMQ Management UI is accessible at http://localhost:15672
- [ ] MinIO Console is accessible at http://localhost:9001
- [ ] `zyenta-media` bucket is created automatically
- [ ] `./scripts/dev-infra.sh clean` removes all data
- [ ] Code review completed

## Testing

1. Run `./scripts/dev-infra.sh start`
2. Wait for all services to be healthy
3. Connect to PostgreSQL: `psql postgresql://zyenta:zyenta_dev@localhost:5432/zyenta`
4. Connect to Redis: `redis-cli -h localhost ping`
5. Open RabbitMQ UI: http://localhost:15672 (zyenta/zyenta_dev)
6. Open MinIO Console: http://localhost:9001 (minioadmin/minioadmin)
7. Verify `zyenta-media` bucket exists
8. Run `./scripts/dev-infra.sh stop`
9. Run `./scripts/dev-infra.sh clean`

## Notes

- All credentials are for local development only
- Production will use managed services (RDS, ElastiCache, Amazon MQ, S3)
- MinIO provides S3-compatible API for local testing

## Related Stories

- Depends on: STORY-001 (Monorepo Setup - directory structure)
- Required by: STORY-002 (Database Package - for testing migrations)
