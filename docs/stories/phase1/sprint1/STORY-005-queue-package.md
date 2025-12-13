# Story 005: Queue Package

## Story Information

| Field | Value |
|-------|-------|
| **Story ID** | STORY-005 |
| **Sprint** | Phase 1 - Sprint 1 |
| **Title** | Queue Package with BullMQ |
| **Priority** | High |
| **Story Points** | 5 |
| **Status** | Ready for Development |
| **Dependencies** | STORY-001, STORY-003 |

## User Story

**As a** developer
**I want** a shared queue package with BullMQ configuration
**So that** all services can use a consistent queue system for background job processing

## Acceptance Criteria

### AC1: Package Configuration
- [ ] `packages/queue/package.json` is created
- [ ] Package name is `@zyenta/queue`
- [ ] Proper exports configuration
- [ ] Dependencies: bullmq, ioredis
- [ ] Build and dev scripts defined

### AC2: TypeScript Configuration
- [ ] `packages/queue/tsconfig.json` extends base config
- [ ] Declaration files are generated
- [ ] Proper output directory configuration

### AC3: Redis Configuration
- [ ] `config.ts` provides Redis connection factory
- [ ] Reads `REDIS_URL` from environment
- [ ] Falls back to localhost:6379 for development
- [ ] `QueueConfig` interface is exported
- [ ] Connection uses `maxRetriesPerRequest: null` for BullMQ

### AC4: Queue Definitions
- [ ] `QUEUE_NAMES` constant defines: GENESIS, MEDIA, NOTIFICATIONS
- [ ] `genesisQueue` is created with proper defaults
- [ ] `mediaQueue` is created with proper defaults
- [ ] `notificationsQueue` is created with proper defaults
- [ ] Each queue has appropriate retry and backoff settings

### AC5: Worker Factory
- [ ] `createWorker` function is exported
- [ ] Accepts queue name, processor function, and options
- [ ] Returns configured Worker instance
- [ ] Uses shared Redis connection

### AC6: Job Types
- [ ] `JobType` union type defined
- [ ] `JobData` interface with type, storeId, userId, payload
- [ ] `BrandGenerationPayload` interface
- [ ] `ProductScoutingPayload` interface
- [ ] `ImageProcessingPayload` interface

### AC7: Package Exports
- [ ] `index.ts` exports all public APIs
- [ ] Exports config, queues, and types
- [ ] Clean barrel file pattern

## Technical Details

### Queue Configuration

| Queue | Name | Attempts | Backoff | removeOnComplete |
|-------|------|----------|---------|------------------|
| Genesis | genesis | 3 | exponential 1000ms | 100 |
| Media | media | 3 | exponential 2000ms | 100 |
| Notifications | notifications | 5 | default | true |

### Files to Create

1. **`packages/queue/package.json`**
   - Package configuration with exports

2. **`packages/queue/tsconfig.json`**
   - TypeScript configuration

3. **`packages/queue/src/index.ts`**
   - Barrel export file

4. **`packages/queue/src/config.ts`**
   - Redis connection configuration
   - `createRedisConnection()` function
   - `getQueueConfig()` function

5. **`packages/queue/src/queues.ts`**
   - Queue name constants
   - Queue instances
   - Worker factory function

6. **`packages/queue/src/types.ts`**
   - JobType union
   - JobData interface
   - Payload interfaces

### Dependencies

```json
{
  "dependencies": {
    "bullmq": "^5.1.0",
    "ioredis": "^5.3.0"
  },
  "devDependencies": {
    "@types/node": "^20.10.0",
    "typescript": "^5.3.0"
  }
}
```

### Job Type Definitions

```typescript
type JobType =
  | "BRAND_GENERATION"
  | "PRODUCT_SCOUTING"
  | "COPYWRITING"
  | "IMAGE_PROCESSING"
  | "SEND_EMAIL"
  | "SEND_WEBHOOK";
```

## Definition of Done

- [ ] All files are created as specified
- [ ] Package builds without errors (`pnpm build`)
- [ ] TypeScript types are properly exported
- [ ] Can be imported by other packages
- [ ] Queue connects to Redis (with Docker running)
- [ ] Worker factory creates functional workers
- [ ] Code review completed

## Testing

1. Start Docker infrastructure (for Redis)
2. Run `pnpm build` from packages/queue
3. Create a test script that:
   - Imports the package
   - Adds a job to a queue
   - Creates a worker to process it
   - Verifies job completion

## Notes

- BullMQ requires `maxRetriesPerRequest: null` on Redis connection
- All queues share the same Redis connection
- Job data should be serializable (no functions or complex objects)
- Consider adding job progress and logging in future iterations

## Related Stories

- Depends on: STORY-001 (Monorepo Setup), STORY-003 (Docker Compose - Redis)
- Used by: Genesis Engine and Media Studio services (future sprints)
