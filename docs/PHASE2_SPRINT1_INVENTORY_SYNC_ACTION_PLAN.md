# Phase 2 Sprint 1: Inventory Sync Foundation - Action Plan

## Overview

This document provides a comprehensive implementation guide for Phase 2 Sprint 1, focusing on establishing the Ops Engine service with inventory synchronization capabilities.

### Sprint Objectives
- [ ] Setup NestJS Ops Engine project structure
- [ ] Implement sync scheduler (cron jobs)
- [ ] Build sync processor with batch handling
- [ ] Implement supplier adapters for inventory fetching
- [ ] Setup BullMQ for job processing

### Prerequisites
- Phase 1 completed and deployed
- Database schema with Product and Store models
- Redis instance available
- Supplier API credentials configured

---

## 1. NestJS Ops Engine Project Setup

### 1.1 Initialize Project

```bash
# From monorepo root
cd services
mkdir ops-engine && cd ops-engine

# Initialize NestJS project
pnpm dlx @nestjs/cli new . --package-manager pnpm --skip-git
```

### 1.2 Package Configuration

```json
// services/ops-engine/package.json
{
  "name": "@zyenta/ops-engine",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "build": "nest build",
    "start": "nest start",
    "start:dev": "nest start --watch",
    "start:debug": "nest start --debug --watch",
    "start:prod": "node dist/main",
    "lint": "eslint \"{src,test}/**/*.ts\" --fix",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:cov": "jest --coverage",
    "test:e2e": "jest --config ./test/jest-e2e.json"
  },
  "dependencies": {
    "@nestjs/common": "^10.3.0",
    "@nestjs/core": "^10.3.0",
    "@nestjs/config": "^3.1.1",
    "@nestjs/platform-express": "^10.3.0",
    "@nestjs/schedule": "^4.0.0",
    "@nestjs/bull": "^10.0.1",
    "@nestjs/terminus": "^10.2.0",
    "@zyenta/database": "workspace:*",
    "@zyenta/shared-types": "workspace:*",
    "@zyenta/queue": "workspace:*",
    "bull": "^4.12.0",
    "ioredis": "^5.3.2",
    "axios": "^1.6.2",
    "class-validator": "^0.14.0",
    "class-transformer": "^0.5.1",
    "crypto-js": "^4.2.0",
    "reflect-metadata": "^0.1.13",
    "rxjs": "^7.8.1"
  },
  "devDependencies": {
    "@nestjs/cli": "^10.2.1",
    "@nestjs/schematics": "^10.0.3",
    "@nestjs/testing": "^10.3.0",
    "@types/express": "^4.17.21",
    "@types/jest": "^29.5.11",
    "@types/node": "^20.10.5",
    "@typescript-eslint/eslint-plugin": "^6.15.0",
    "@typescript-eslint/parser": "^6.15.0",
    "eslint": "^8.56.0",
    "eslint-config-prettier": "^9.1.0",
    "eslint-plugin-prettier": "^5.1.2",
    "jest": "^29.7.0",
    "prettier": "^3.1.1",
    "source-map-support": "^0.5.21",
    "ts-jest": "^29.1.1",
    "ts-loader": "^9.5.1",
    "ts-node": "^10.9.2",
    "tsconfig-paths": "^4.2.0",
    "typescript": "^5.3.3"
  }
}
```

### 1.3 TypeScript Configuration

```json
// services/ops-engine/tsconfig.json
{
  "compilerOptions": {
    "module": "commonjs",
    "declaration": true,
    "removeComments": true,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "allowSyntheticDefaultImports": true,
    "target": "ES2021",
    "sourceMap": true,
    "outDir": "./dist",
    "baseUrl": "./",
    "incremental": true,
    "skipLibCheck": true,
    "strictNullChecks": true,
    "noImplicitAny": true,
    "strictBindCallApply": true,
    "forceConsistentCasingInFileNames": true,
    "noFallthroughCasesInSwitch": true,
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

### 1.4 NestJS Configuration

```typescript
// services/ops-engine/nest-cli.json
{
  "$schema": "https://json.schemastore.org/nest-cli",
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "deleteOutDir": true
  }
}
```

### 1.5 Main Application Entry

```typescript
// services/ops-engine/src/main.ts
import { NestFactory } from '@nestjs/core';
import { ValidationPipe, Logger } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { AppModule } from './app.module';

async function bootstrap() {
  const logger = new Logger('Bootstrap');
  const app = await NestFactory.create(AppModule);

  // Global validation pipe
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      transform: true,
      forbidNonWhitelisted: true,
    }),
  );

  // CORS
  app.enableCors({
    origin: process.env.CORS_ORIGINS?.split(',') || ['http://localhost:3000'],
    credentials: true,
  });

  // Global prefix
  app.setGlobalPrefix('api/v1');

  const configService = app.get(ConfigService);
  const port = configService.get('PORT', 8002);

  await app.listen(port);
  logger.log(`Ops Engine running on port ${port}`);
}

bootstrap();
```

### 1.6 Root Application Module

```typescript
// services/ops-engine/src/app.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { ScheduleModule } from '@nestjs/schedule';
import { BullModule } from '@nestjs/bull';
import { TerminusModule } from '@nestjs/terminus';

import { DatabaseModule } from './shared/database/database.module';
import { CacheModule } from './shared/cache/cache.module';
import { InventoryModule } from './modules/inventory/inventory.module';
import { HealthModule } from './modules/health/health.module';

import configuration from './config/configuration';

@Module({
  imports: [
    // Configuration
    ConfigModule.forRoot({
      isGlobal: true,
      load: [configuration],
    }),

    // Scheduling for cron jobs
    ScheduleModule.forRoot(),

    // BullMQ for job processing
    BullModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: (configService: ConfigService) => ({
        redis: {
          host: configService.get('REDIS_HOST', 'localhost'),
          port: configService.get('REDIS_PORT', 6379),
          password: configService.get('REDIS_PASSWORD'),
        },
        defaultJobOptions: {
          removeOnComplete: 100,
          removeOnFail: 500,
          attempts: 3,
          backoff: {
            type: 'exponential',
            delay: 5000,
          },
        },
      }),
      inject: [ConfigService],
    }),

    // Health checks
    TerminusModule,

    // Shared modules
    DatabaseModule,
    CacheModule,

    // Feature modules
    InventoryModule,
    HealthModule,
  ],
})
export class AppModule {}
```

### 1.7 Configuration

```typescript
// services/ops-engine/src/config/configuration.ts
export default () => ({
  port: parseInt(process.env.PORT, 10) || 8002,
  environment: process.env.NODE_ENV || 'development',

  database: {
    url: process.env.DATABASE_URL,
  },

  redis: {
    host: process.env.REDIS_HOST || 'localhost',
    port: parseInt(process.env.REDIS_PORT, 10) || 6379,
    password: process.env.REDIS_PASSWORD,
  },

  sync: {
    highFrequencyInterval: process.env.SYNC_HIGH_FREQ_INTERVAL || '*/15 * * * *', // Every 15 min
    standardInterval: process.env.SYNC_STANDARD_INTERVAL || '0 * * * *', // Every hour
    fullSyncTime: process.env.SYNC_FULL_TIME || '0 3 * * *', // 3 AM daily
    batchSize: parseInt(process.env.SYNC_BATCH_SIZE, 10) || 50,
  },

  suppliers: {
    aliexpress: {
      appKey: process.env.ALIEXPRESS_APP_KEY,
      appSecret: process.env.ALIEXPRESS_APP_SECRET,
    },
    cjDropshipping: {
      apiKey: process.env.CJ_API_KEY,
    },
    spocket: {
      apiKey: process.env.SPOCKET_API_KEY,
    },
  },
});
```

---

## 2. Database Module

### 2.1 Prisma Service

```typescript
// services/ops-engine/src/shared/database/prisma.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy, Logger } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  private readonly logger = new Logger(PrismaService.name);

  constructor() {
    super({
      log: [
        { emit: 'event', level: 'query' },
        { emit: 'stdout', level: 'info' },
        { emit: 'stdout', level: 'warn' },
        { emit: 'stdout', level: 'error' },
      ],
    });
  }

  async onModuleInit() {
    await this.$connect();
    this.logger.log('Database connected');

    // Log slow queries in development
    if (process.env.NODE_ENV === 'development') {
      this.$on('query' as never, (e: any) => {
        if (e.duration > 100) {
          this.logger.warn(`Slow query (${e.duration}ms): ${e.query}`);
        }
      });
    }
  }

  async onModuleDestroy() {
    await this.$disconnect();
    this.logger.log('Database disconnected');
  }

  async cleanDatabase() {
    if (process.env.NODE_ENV !== 'test') {
      throw new Error('cleanDatabase can only be used in test environment');
    }
    // Only for testing
  }
}
```

### 2.2 Database Module

```typescript
// services/ops-engine/src/shared/database/database.module.ts
import { Global, Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class DatabaseModule {}
```

---

## 3. Cache Module

### 3.1 Redis Service

```typescript
// services/ops-engine/src/shared/cache/redis.service.ts
import { Injectable, OnModuleDestroy, Logger } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import Redis from 'ioredis';

@Injectable()
export class RedisService implements OnModuleDestroy {
  private readonly logger = new Logger(RedisService.name);
  private readonly client: Redis;

  constructor(private readonly configService: ConfigService) {
    this.client = new Redis({
      host: this.configService.get('redis.host'),
      port: this.configService.get('redis.port'),
      password: this.configService.get('redis.password'),
      retryStrategy: (times) => {
        const delay = Math.min(times * 50, 2000);
        return delay;
      },
    });

    this.client.on('connect', () => {
      this.logger.log('Redis connected');
    });

    this.client.on('error', (err) => {
      this.logger.error('Redis error:', err);
    });
  }

  async onModuleDestroy() {
    await this.client.quit();
  }

  async get<T>(key: string): Promise<T | null> {
    const data = await this.client.get(key);
    if (!data) return null;
    try {
      return JSON.parse(data) as T;
    } catch {
      return data as unknown as T;
    }
  }

  async set(key: string, value: any, ttlSeconds?: number): Promise<void> {
    const serialized = typeof value === 'string' ? value : JSON.stringify(value);
    if (ttlSeconds) {
      await this.client.setex(key, ttlSeconds, serialized);
    } else {
      await this.client.set(key, serialized);
    }
  }

  async del(key: string): Promise<void> {
    await this.client.del(key);
  }

  async invalidatePattern(pattern: string): Promise<void> {
    const keys = await this.client.keys(pattern);
    if (keys.length > 0) {
      await this.client.del(...keys);
    }
  }

  async exists(key: string): Promise<boolean> {
    const result = await this.client.exists(key);
    return result === 1;
  }

  async incr(key: string): Promise<number> {
    return this.client.incr(key);
  }

  async expire(key: string, seconds: number): Promise<void> {
    await this.client.expire(key, seconds);
  }

  getClient(): Redis {
    return this.client;
  }
}
```

### 3.2 Cache Module

```typescript
// services/ops-engine/src/shared/cache/cache.module.ts
import { Global, Module } from '@nestjs/common';
import { RedisService } from './redis.service';

@Global()
@Module({
  providers: [RedisService],
  exports: [RedisService],
})
export class CacheModule {}
```

---

## 4. Inventory Module

### 4.1 Module Definition

```typescript
// services/ops-engine/src/modules/inventory/inventory.module.ts
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bull';
import { InventoryController } from './inventory.controller';
import { InventoryService } from './inventory.service';
import { SyncScheduler } from './sync/sync.scheduler';
import { SyncProcessor } from './sync/sync.processor';
import { SupplierModule } from './suppliers/supplier.module';

@Module({
  imports: [
    BullModule.registerQueue({
      name: 'inventory-sync',
      defaultJobOptions: {
        removeOnComplete: 50,
        removeOnFail: 100,
      },
    }),
    SupplierModule,
  ],
  controllers: [InventoryController],
  providers: [
    InventoryService,
    SyncScheduler,
    SyncProcessor,
  ],
  exports: [InventoryService],
})
export class InventoryModule {}
```

### 4.2 Inventory Service

```typescript
// services/ops-engine/src/modules/inventory/inventory.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { PrismaService } from '@/shared/database/prisma.service';
import { RedisService } from '@/shared/cache/redis.service';

export interface SyncResult {
  storeId: string;
  productsProcessed: number;
  productsUpdated: number;
  errors: string[];
  duration: number;
}

@Injectable()
export class InventoryService {
  private readonly logger = new Logger(InventoryService.name);

  constructor(
    private readonly prisma: PrismaService,
    private readonly redis: RedisService,
  ) {}

  async getStoresForSync(syncType: 'HIGH' | 'STANDARD' | 'ALL'): Promise<string[]> {
    const whereClause: any = {
      status: 'ACTIVE',
    };

    if (syncType === 'HIGH') {
      whereClause.settings = {
        path: ['syncFrequency'],
        equals: 'HIGH',
      };
    } else if (syncType === 'STANDARD') {
      whereClause.OR = [
        {
          settings: {
            path: ['syncFrequency'],
            equals: 'STANDARD',
          },
        },
        {
          settings: {
            path: ['syncFrequency'],
            equals: null,
          },
        },
      ];
    }

    const stores = await this.prisma.store.findMany({
      where: whereClause,
      select: { id: true },
    });

    return stores.map((s) => s.id);
  }

  async getStoreProducts(storeId: string) {
    return this.prisma.product.findMany({
      where: {
        storeId,
        status: { in: ['ACTIVE', 'OUT_OF_STOCK'] },
      },
      include: {
        variants: true,
        store: {
          include: {
            user: {
              include: {
                supplierAccounts: true,
              },
            },
          },
        },
      },
    });
  }

  async updateProductInventory(
    productId: string,
    data: {
      stockQuantity?: number;
      costPrice?: number;
      sellingPrice?: number;
      status?: string;
    },
  ) {
    const updated = await this.prisma.product.update({
      where: { id: productId },
      data: {
        ...data,
        updatedAt: new Date(),
      },
    });

    // Invalidate cache
    await this.redis.del(`product:${productId}`);
    await this.redis.invalidatePattern(`store:${updated.storeId}:products*`);

    return updated;
  }

  async recordSyncLog(data: {
    storeId: string;
    type: 'INVENTORY' | 'PRICE' | 'FULL';
    status: 'STARTED' | 'COMPLETED' | 'FAILED';
    productsProcessed?: number;
    productsChanged?: number;
    duration?: number;
    error?: string;
    metadata?: any;
  }) {
    return this.prisma.syncLog.create({
      data: {
        storeId: data.storeId,
        type: data.type,
        status: data.status,
        productsProcessed: data.productsProcessed || 0,
        productsChanged: data.productsChanged || 0,
        duration: data.duration || 0,
        error: data.error,
        metadata: data.metadata,
      },
    });
  }

  async getLastSyncTime(storeId: string): Promise<Date | null> {
    const lastSync = await this.prisma.syncLog.findFirst({
      where: {
        storeId,
        status: 'COMPLETED',
      },
      orderBy: { createdAt: 'desc' },
      select: { createdAt: true },
    });

    return lastSync?.createdAt || null;
  }
}
```

### 4.3 Inventory Controller

```typescript
// services/ops-engine/src/modules/inventory/inventory.controller.ts
import {
  Controller,
  Get,
  Post,
  Param,
  Query,
  HttpCode,
  HttpStatus,
} from '@nestjs/common';
import { InjectQueue } from '@nestjs/bull';
import { Queue } from 'bull';
import { InventoryService } from './inventory.service';

@Controller('inventory')
export class InventoryController {
  constructor(
    private readonly inventoryService: InventoryService,
    @InjectQueue('inventory-sync') private readonly syncQueue: Queue,
  ) {}

  @Get('status/:storeId')
  async getSyncStatus(@Param('storeId') storeId: string) {
    const lastSync = await this.inventoryService.getLastSyncTime(storeId);
    const pendingJobs = await this.syncQueue.getJobs(['waiting', 'active']);
    const storePendingJob = pendingJobs.find(
      (job) => job.data.storeId === storeId,
    );

    return {
      storeId,
      lastSyncAt: lastSync,
      syncInProgress: !!storePendingJob,
      jobId: storePendingJob?.id,
    };
  }

  @Post('trigger/:storeId')
  @HttpCode(HttpStatus.ACCEPTED)
  async triggerSync(
    @Param('storeId') storeId: string,
    @Query('priority') priority: string = 'standard',
  ) {
    const job = await this.syncQueue.add(
      'sync-store-inventory',
      {
        storeId,
        priority: priority as 'high' | 'standard',
        triggeredManually: true,
      },
      {
        priority: priority === 'high' ? 1 : 5,
      },
    );

    return {
      message: 'Sync job queued',
      jobId: job.id,
      storeId,
    };
  }

  @Get('logs/:storeId')
  async getSyncLogs(
    @Param('storeId') storeId: string,
    @Query('limit') limit: string = '20',
  ) {
    // Would implement via prisma query
    return {
      storeId,
      logs: [],
    };
  }

  @Get('queue/status')
  async getQueueStatus() {
    const [waiting, active, completed, failed] = await Promise.all([
      this.syncQueue.getWaitingCount(),
      this.syncQueue.getActiveCount(),
      this.syncQueue.getCompletedCount(),
      this.syncQueue.getFailedCount(),
    ]);

    return {
      waiting,
      active,
      completed,
      failed,
    };
  }
}
```

---

## 5. Sync Scheduler

### 5.1 Scheduler Implementation

```typescript
// services/ops-engine/src/modules/inventory/sync/sync.scheduler.ts
import { Injectable, Logger } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';
import { InjectQueue } from '@nestjs/bull';
import { Queue } from 'bull';
import { ConfigService } from '@nestjs/config';
import { InventoryService } from '../inventory.service';

@Injectable()
export class SyncScheduler {
  private readonly logger = new Logger(SyncScheduler.name);

  constructor(
    private readonly inventoryService: InventoryService,
    private readonly configService: ConfigService,
    @InjectQueue('inventory-sync') private readonly syncQueue: Queue,
  ) {}

  /**
   * High-frequency sync for premium stores
   * Runs every 15 minutes
   */
  @Cron(CronExpression.EVERY_15_MINUTES)
  async scheduleHighFrequencySync() {
    this.logger.log('Starting high-frequency inventory sync scheduling...');

    try {
      const storeIds = await this.inventoryService.getStoresForSync('HIGH');

      for (const storeId of storeIds) {
        await this.syncQueue.add(
          'sync-store-inventory',
          {
            storeId,
            priority: 'high',
            syncType: 'incremental',
          },
          {
            priority: 1,
            attempts: 3,
            backoff: {
              type: 'exponential',
              delay: 5000,
            },
            jobId: `high-sync-${storeId}-${Date.now()}`,
          },
        );
      }

      this.logger.log(`Queued ${storeIds.length} high-frequency sync jobs`);
    } catch (error) {
      this.logger.error('Failed to schedule high-frequency sync:', error);
    }
  }

  /**
   * Standard sync for regular stores
   * Runs every hour
   */
  @Cron(CronExpression.EVERY_HOUR)
  async scheduleStandardSync() {
    this.logger.log('Starting standard inventory sync scheduling...');

    try {
      const storeIds = await this.inventoryService.getStoresForSync('STANDARD');

      for (const storeId of storeIds) {
        await this.syncQueue.add(
          'sync-store-inventory',
          {
            storeId,
            priority: 'standard',
            syncType: 'incremental',
          },
          {
            priority: 5,
            attempts: 3,
            backoff: {
              type: 'exponential',
              delay: 10000,
            },
            jobId: `standard-sync-${storeId}-${Date.now()}`,
          },
        );
      }

      this.logger.log(`Queued ${storeIds.length} standard sync jobs`);
    } catch (error) {
      this.logger.error('Failed to schedule standard sync:', error);
    }
  }

  /**
   * Full sync for all stores
   * Runs daily at 3 AM UTC
   */
  @Cron('0 3 * * *')
  async scheduleFullSync() {
    this.logger.log('Starting daily full inventory sync scheduling...');

    try {
      const storeIds = await this.inventoryService.getStoresForSync('ALL');

      for (const storeId of storeIds) {
        await this.syncQueue.add(
          'sync-store-inventory',
          {
            storeId,
            priority: 'standard',
            syncType: 'full',
            fullSync: true,
          },
          {
            priority: 10,
            attempts: 3,
            jobId: `full-sync-${storeId}-${Date.now()}`,
          },
        );
      }

      this.logger.log(`Queued ${storeIds.length} full sync jobs`);
    } catch (error) {
      this.logger.error('Failed to schedule full sync:', error);
    }
  }

  /**
   * Check for stale syncs and retry
   * Runs every 30 minutes
   */
  @Cron('*/30 * * * *')
  async checkStaleSyncs() {
    this.logger.log('Checking for stale sync jobs...');

    try {
      const staleJobs = await this.syncQueue.getJobs(['active']);
      const now = Date.now();
      const staleThreshold = 30 * 60 * 1000; // 30 minutes

      for (const job of staleJobs) {
        const jobAge = now - job.timestamp;
        if (jobAge > staleThreshold) {
          this.logger.warn(`Found stale job: ${job.id}, moving to failed`);
          await job.moveToFailed({ message: 'Job timed out' }, true);
        }
      }
    } catch (error) {
      this.logger.error('Failed to check stale syncs:', error);
    }
  }
}
```

---

## 6. Sync Processor

### 6.1 Processor Implementation

```typescript
// services/ops-engine/src/modules/inventory/sync/sync.processor.ts
import { Process, Processor, OnQueueError, OnQueueFailed } from '@nestjs/bull';
import { Logger } from '@nestjs/common';
import { Job } from 'bull';
import { PrismaService } from '@/shared/database/prisma.service';
import { RedisService } from '@/shared/cache/redis.service';
import { InventoryService } from '../inventory.service';
import { SupplierFactory } from '../suppliers/supplier.factory';

interface SyncJobData {
  storeId: string;
  priority: 'high' | 'standard';
  syncType: 'incremental' | 'full';
  triggeredManually?: boolean;
}

interface ProductSyncResult {
  productId: string;
  sourceId: string;
  previousStock: number;
  currentStock: number;
  previousPrice: number;
  currentPrice: number;
  changes: string[];
  error?: string;
}

@Processor('inventory-sync')
export class SyncProcessor {
  private readonly logger = new Logger(SyncProcessor.name);

  constructor(
    private readonly prisma: PrismaService,
    private readonly redis: RedisService,
    private readonly inventoryService: InventoryService,
    private readonly supplierFactory: SupplierFactory,
  ) {}

  @Process('sync-store-inventory')
  async handleInventorySync(job: Job<SyncJobData>) {
    const { storeId, priority, syncType } = job.data;
    const startTime = Date.now();

    this.logger.log(
      `Processing ${syncType} inventory sync for store: ${storeId} (priority: ${priority})`,
    );

    // Record sync start
    await this.inventoryService.recordSyncLog({
      storeId,
      type: syncType === 'full' ? 'FULL' : 'INVENTORY',
      status: 'STARTED',
    });

    const results: ProductSyncResult[] = [];
    let productsChanged = 0;

    try {
      // Get store products with supplier info
      const products = await this.inventoryService.getStoreProducts(storeId);

      if (products.length === 0) {
        this.logger.log(`No products to sync for store: ${storeId}`);
        return { success: true, processed: 0, changed: 0 };
      }

      // Group products by supplier
      const productsBySupplier = this.groupBySupplier(products);

      // Process each supplier's products
      for (const [provider, supplierProducts] of Object.entries(productsBySupplier)) {
        const supplierAccount = products[0].store.user.supplierAccounts.find(
          (acc) => acc.provider === provider,
        );

        if (!supplierAccount) {
          this.logger.warn(`No supplier account found for ${provider}`);
          continue;
        }

        try {
          const supplier = this.supplierFactory.create(provider, {
            apiKey: supplierAccount.apiKey,
            apiSecret: supplierAccount.apiSecret || undefined,
            accessToken: supplierAccount.accountId || undefined,
          });

          // Get product IDs for batch fetch
          const sourceIds = supplierProducts.map((p) => p.sourceId);

          // Batch fetch from supplier API
          const supplierData = await supplier.getProductsBatch(sourceIds);

          // Process each product
          for (const product of supplierProducts) {
            const result = await this.syncProduct(product, supplierData.get(product.sourceId));
            results.push(result);

            if (result.changes.length > 0) {
              productsChanged++;
            }

            // Update job progress
            const progress = Math.round((results.length / products.length) * 100);
            await job.progress(progress);
          }
        } catch (supplierError) {
          this.logger.error(`Error syncing ${provider} products:`, supplierError);
          // Continue with other suppliers
        }
      }

      const duration = Date.now() - startTime;

      // Record successful sync
      await this.inventoryService.recordSyncLog({
        storeId,
        type: syncType === 'full' ? 'FULL' : 'INVENTORY',
        status: 'COMPLETED',
        productsProcessed: results.length,
        productsChanged,
        duration,
        metadata: {
          stockChanges: results.filter((r) => r.previousStock !== r.currentStock).length,
          priceChanges: results.filter((r) => r.previousPrice !== r.currentPrice).length,
        },
      });

      this.logger.log(
        `Completed sync for store ${storeId}: ${results.length} processed, ${productsChanged} changed (${duration}ms)`,
      );

      return {
        success: true,
        processed: results.length,
        changed: productsChanged,
        duration,
      };
    } catch (error) {
      const duration = Date.now() - startTime;

      // Record failed sync
      await this.inventoryService.recordSyncLog({
        storeId,
        type: syncType === 'full' ? 'FULL' : 'INVENTORY',
        status: 'FAILED',
        productsProcessed: results.length,
        productsChanged,
        duration,
        error: error.message,
      });

      this.logger.error(`Sync failed for store ${storeId}:`, error);
      throw error;
    }
  }

  private async syncProduct(
    product: any,
    sourceData: any | undefined,
  ): Promise<ProductSyncResult> {
    const result: ProductSyncResult = {
      productId: product.id,
      sourceId: product.sourceId,
      previousStock: product.stockQuantity,
      currentStock: product.stockQuantity,
      previousPrice: Number(product.costPrice),
      currentPrice: Number(product.costPrice),
      changes: [],
    };

    if (!sourceData) {
      result.error = 'Product not found at supplier';
      return result;
    }

    try {
      const updates: any = {};

      // Check stock changes
      if (sourceData.stock !== product.stockQuantity) {
        result.currentStock = sourceData.stock;
        result.changes.push(`Stock: ${product.stockQuantity} → ${sourceData.stock}`);
        updates.stockQuantity = sourceData.stock;
      }

      // Check cost price changes
      if (sourceData.price !== Number(product.costPrice)) {
        result.currentPrice = sourceData.price;
        result.changes.push(`Cost: ${product.costPrice} → ${sourceData.price}`);
        updates.costPrice = sourceData.price;
      }

      // Determine new status
      if (sourceData.stock === 0 && product.status === 'ACTIVE') {
        updates.status = 'OUT_OF_STOCK';
        result.changes.push('Status: ACTIVE → OUT_OF_STOCK');
      } else if (sourceData.stock > 0 && product.status === 'OUT_OF_STOCK') {
        updates.status = 'ACTIVE';
        result.changes.push('Status: OUT_OF_STOCK → ACTIVE');
      }

      // Apply updates if any changes detected
      if (Object.keys(updates).length > 0) {
        await this.inventoryService.updateProductInventory(product.id, updates);
      }
    } catch (error) {
      result.error = error.message;
      this.logger.error(`Error syncing product ${product.id}:`, error);
    }

    return result;
  }

  private groupBySupplier(products: any[]): Record<string, any[]> {
    return products.reduce((acc, product) => {
      const provider = product.sourceProvider;
      if (!acc[provider]) {
        acc[provider] = [];
      }
      acc[provider].push(product);
      return acc;
    }, {} as Record<string, any[]>);
  }

  @OnQueueError()
  onError(error: Error) {
    this.logger.error('Queue error:', error);
  }

  @OnQueueFailed()
  onFailed(job: Job, error: Error) {
    this.logger.error(`Job ${job.id} failed:`, error);
  }
}
```

---

## 7. Supplier Module

### 7.1 Module Definition

```typescript
// services/ops-engine/src/modules/inventory/suppliers/supplier.module.ts
import { Module } from '@nestjs/common';
import { SupplierFactory } from './supplier.factory';
import { AliExpressAdapter } from './aliexpress.adapter';
import { CJDropshippingAdapter } from './cj-dropshipping.adapter';
import { SpocketAdapter } from './spocket.adapter';

@Module({
  providers: [
    SupplierFactory,
    AliExpressAdapter,
    CJDropshippingAdapter,
    SpocketAdapter,
  ],
  exports: [SupplierFactory],
})
export class SupplierModule {}
```

### 7.2 Supplier Factory

```typescript
// services/ops-engine/src/modules/inventory/suppliers/supplier.factory.ts
import { Injectable } from '@nestjs/common';
import { BaseSupplierAdapter } from './base.adapter';
import { AliExpressAdapter } from './aliexpress.adapter';
import { CJDropshippingAdapter } from './cj-dropshipping.adapter';
import { SpocketAdapter } from './spocket.adapter';

export interface SupplierCredentials {
  apiKey: string;
  apiSecret?: string;
  accessToken?: string;
}

@Injectable()
export class SupplierFactory {
  create(provider: string, credentials: SupplierCredentials): BaseSupplierAdapter {
    switch (provider) {
      case 'ALIEXPRESS':
        return new AliExpressAdapter(credentials);
      case 'CJ_DROPSHIPPING':
        return new CJDropshippingAdapter(credentials);
      case 'SPOCKET':
        return new SpocketAdapter(credentials);
      default:
        throw new Error(`Unknown supplier provider: ${provider}`);
    }
  }
}
```

### 7.3 Base Adapter Interface

```typescript
// services/ops-engine/src/modules/inventory/suppliers/base.adapter.ts
export interface ProductData {
  sourceId: string;
  title: string;
  price: number;
  stock: number;
  images: string[];
  variants?: VariantData[];
}

export interface VariantData {
  id: string;
  price: number;
  stock: number;
  attributes: Record<string, string>;
}

export abstract class BaseSupplierAdapter {
  abstract getProduct(sourceId: string): Promise<ProductData | null>;
  abstract getProductsBatch(sourceIds: string[]): Promise<Map<string, ProductData>>;
}
```

### 7.4 AliExpress Adapter

```typescript
// services/ops-engine/src/modules/inventory/suppliers/aliexpress.adapter.ts
import { Logger } from '@nestjs/common';
import axios, { AxiosInstance } from 'axios';
import * as crypto from 'crypto';
import { BaseSupplierAdapter, ProductData } from './base.adapter';

export class AliExpressAdapter extends BaseSupplierAdapter {
  private readonly logger = new Logger(AliExpressAdapter.name);
  private readonly client: AxiosInstance;
  private readonly appKey: string;
  private readonly appSecret: string;
  private readonly accessToken: string;

  constructor(credentials: { apiKey: string; apiSecret?: string; accessToken?: string }) {
    super();
    this.appKey = credentials.apiKey;
    this.appSecret = credentials.apiSecret || '';
    this.accessToken = credentials.accessToken || '';

    this.client = axios.create({
      baseURL: 'https://api-sg.aliexpress.com/sync',
      timeout: 30000,
    });
  }

  private generateSignature(params: Record<string, string>): string {
    const sortedKeys = Object.keys(params).sort();
    let signString = this.appSecret;

    for (const key of sortedKeys) {
      signString += key + params[key];
    }

    signString += this.appSecret;

    return crypto
      .createHash('md5')
      .update(signString)
      .digest('hex')
      .toUpperCase();
  }

  async getProduct(sourceId: string): Promise<ProductData | null> {
    try {
      const params: Record<string, string> = {
        app_key: this.appKey,
        access_token: this.accessToken,
        method: 'aliexpress.ds.product.get',
        sign_method: 'md5',
        timestamp: new Date().toISOString(),
        product_id: sourceId,
      };

      params.sign = this.generateSignature(params);

      const response = await this.client.post('', null, { params });
      const result = response.data?.aliexpress_ds_product_get_response?.result;

      if (!result) {
        return null;
      }

      const skus = result.ae_item_sku_info_dtos || [];
      const firstSku = skus[0] || {};

      return {
        sourceId,
        title: result.ae_item_base_info_dto?.subject || '',
        price: parseFloat(firstSku.sku_price || '0'),
        stock: parseInt(firstSku.sku_stock || '0', 10),
        images: (result.ae_multimedia_info_dto?.image_urls || '').split(';').filter(Boolean),
        variants: skus.map((sku: any) => ({
          id: sku.sku_id,
          price: parseFloat(sku.sku_price || '0'),
          stock: parseInt(sku.sku_stock || '0', 10),
          attributes: this.parseSkuAttributes(sku.sku_attr),
        })),
      };
    } catch (error) {
      this.logger.error(`Error fetching AliExpress product ${sourceId}:`, error);
      return null;
    }
  }

  async getProductsBatch(sourceIds: string[]): Promise<Map<string, ProductData>> {
    const results = new Map<string, ProductData>();

    // AliExpress doesn't support batch, so we parallelize with rate limiting
    const batchSize = 5;
    for (let i = 0; i < sourceIds.length; i += batchSize) {
      const batch = sourceIds.slice(i, i + batchSize);
      const batchResults = await Promise.all(
        batch.map((id) => this.getProduct(id)),
      );

      batchResults.forEach((result, index) => {
        if (result) {
          results.set(batch[index], result);
        }
      });

      // Rate limiting delay between batches
      if (i + batchSize < sourceIds.length) {
        await this.delay(1000);
      }
    }

    return results;
  }

  private parseSkuAttributes(skuAttr: string): Record<string, string> {
    if (!skuAttr) return {};

    const attributes: Record<string, string> = {};
    // Parse AliExpress SKU attribute format
    const parts = skuAttr.split(';');
    for (const part of parts) {
      const [key, value] = part.split(':');
      if (key && value) {
        attributes[key] = value;
      }
    }
    return attributes;
  }

  private delay(ms: number): Promise<void> {
    return new Promise((resolve) => setTimeout(resolve, ms));
  }
}
```

### 7.5 CJ Dropshipping Adapter

```typescript
// services/ops-engine/src/modules/inventory/suppliers/cj-dropshipping.adapter.ts
import { Logger } from '@nestjs/common';
import axios, { AxiosInstance } from 'axios';
import { BaseSupplierAdapter, ProductData } from './base.adapter';

export class CJDropshippingAdapter extends BaseSupplierAdapter {
  private readonly logger = new Logger(CJDropshippingAdapter.name);
  private readonly client: AxiosInstance;
  private readonly apiKey: string;

  constructor(credentials: { apiKey: string }) {
    super();
    this.apiKey = credentials.apiKey;

    this.client = axios.create({
      baseURL: 'https://developers.cjdropshipping.com/api2.0/v1',
      timeout: 30000,
      headers: {
        'CJ-Access-Token': this.apiKey,
        'Content-Type': 'application/json',
      },
    });
  }

  async getProduct(sourceId: string): Promise<ProductData | null> {
    try {
      const response = await this.client.get(`/product/query`, {
        params: { pid: sourceId },
      });

      const product = response.data?.data;
      if (!product) {
        return null;
      }

      return {
        sourceId,
        title: product.productNameEn || '',
        price: parseFloat(product.sellPrice || '0'),
        stock: parseInt(product.stock || '0', 10),
        images: product.productImage || [],
        variants: (product.variants || []).map((v: any) => ({
          id: v.vid,
          price: parseFloat(v.sellPrice || '0'),
          stock: parseInt(v.stock || '0', 10),
          attributes: v.variantKey || {},
        })),
      };
    } catch (error) {
      this.logger.error(`Error fetching CJ product ${sourceId}:`, error);
      return null;
    }
  }

  async getProductsBatch(sourceIds: string[]): Promise<Map<string, ProductData>> {
    const results = new Map<string, ProductData>();

    // CJ supports batch queries
    try {
      const response = await this.client.post('/product/list/query', {
        pids: sourceIds.join(','),
      });

      const products = response.data?.data?.list || [];
      for (const product of products) {
        results.set(product.pid, {
          sourceId: product.pid,
          title: product.productNameEn || '',
          price: parseFloat(product.sellPrice || '0'),
          stock: parseInt(product.stock || '0', 10),
          images: product.productImage || [],
        });
      }
    } catch (error) {
      this.logger.error('Error fetching CJ products batch:', error);
      // Fallback to individual queries
      for (const sourceId of sourceIds) {
        const product = await this.getProduct(sourceId);
        if (product) {
          results.set(sourceId, product);
        }
      }
    }

    return results;
  }
}
```

### 7.6 Spocket Adapter

```typescript
// services/ops-engine/src/modules/inventory/suppliers/spocket.adapter.ts
import { Logger } from '@nestjs/common';
import axios, { AxiosInstance } from 'axios';
import { BaseSupplierAdapter, ProductData } from './base.adapter';

export class SpocketAdapter extends BaseSupplierAdapter {
  private readonly logger = new Logger(SpocketAdapter.name);
  private readonly client: AxiosInstance;

  constructor(credentials: { apiKey: string }) {
    super();

    this.client = axios.create({
      baseURL: 'https://api.spocket.co/v1',
      timeout: 30000,
      headers: {
        Authorization: `Bearer ${credentials.apiKey}`,
        'Content-Type': 'application/json',
      },
    });
  }

  async getProduct(sourceId: string): Promise<ProductData | null> {
    try {
      const response = await this.client.get(`/products/${sourceId}`);
      const product = response.data?.product;

      if (!product) {
        return null;
      }

      return {
        sourceId,
        title: product.title || '',
        price: parseFloat(product.wholesale_price || '0'),
        stock: product.inventory_quantity || 0,
        images: (product.images || []).map((img: any) => img.src),
        variants: (product.variants || []).map((v: any) => ({
          id: v.id,
          price: parseFloat(v.price || '0'),
          stock: v.inventory_quantity || 0,
          attributes: {
            size: v.size,
            color: v.color,
          },
        })),
      };
    } catch (error) {
      this.logger.error(`Error fetching Spocket product ${sourceId}:`, error);
      return null;
    }
  }

  async getProductsBatch(sourceIds: string[]): Promise<Map<string, ProductData>> {
    const results = new Map<string, ProductData>();

    // Spocket doesn't support batch, parallelize with rate limiting
    const batchSize = 10;
    for (let i = 0; i < sourceIds.length; i += batchSize) {
      const batch = sourceIds.slice(i, i + batchSize);
      const batchResults = await Promise.all(
        batch.map((id) => this.getProduct(id)),
      );

      batchResults.forEach((result, index) => {
        if (result) {
          results.set(batch[index], result);
        }
      });

      if (i + batchSize < sourceIds.length) {
        await new Promise((resolve) => setTimeout(resolve, 500));
      }
    }

    return results;
  }
}
```

---

## 8. Health Module

### 8.1 Health Controller

```typescript
// services/ops-engine/src/modules/health/health.controller.ts
import { Controller, Get } from '@nestjs/common';
import {
  HealthCheck,
  HealthCheckService,
  PrismaHealthIndicator,
} from '@nestjs/terminus';
import { PrismaService } from '@/shared/database/prisma.service';
import { RedisService } from '@/shared/cache/redis.service';

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private prismaHealth: PrismaHealthIndicator,
    private prisma: PrismaService,
    private redis: RedisService,
  ) {}

  @Get()
  @HealthCheck()
  async check() {
    return this.health.check([
      () => this.prismaHealth.pingCheck('database', this.prisma),
      () => this.checkRedis(),
    ]);
  }

  private async checkRedis() {
    try {
      const client = this.redis.getClient();
      await client.ping();
      return {
        redis: {
          status: 'up',
        },
      };
    } catch (error) {
      return {
        redis: {
          status: 'down',
          message: error.message,
        },
      };
    }
  }
}
```

### 8.2 Health Module

```typescript
// services/ops-engine/src/modules/health/health.module.ts
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';
import { PrismaHealthIndicator } from '@nestjs/terminus';
import { HealthController } from './health.controller';

@Module({
  imports: [TerminusModule],
  controllers: [HealthController],
  providers: [PrismaHealthIndicator],
})
export class HealthModule {}
```

---

## 9. Docker Configuration

### 9.1 Dockerfile

```dockerfile
# services/ops-engine/Dockerfile
FROM node:20-alpine AS base

# Install pnpm
RUN corepack enable && corepack prepare pnpm@latest --activate

WORKDIR /app

# Dependencies stage
FROM base AS deps

COPY package.json pnpm-lock.yaml* ./
RUN pnpm install --frozen-lockfile

# Builder stage
FROM base AS builder

COPY --from=deps /app/node_modules ./node_modules
COPY . .

RUN pnpm build

# Production stage
FROM base AS runner

ENV NODE_ENV=production

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nestjs

COPY --from=builder --chown=nestjs:nodejs /app/dist ./dist
COPY --from=builder --chown=nestjs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nestjs:nodejs /app/package.json ./

USER nestjs

EXPOSE 8002

CMD ["node", "dist/main"]
```

### 9.2 Docker Compose for Development

```yaml
# services/ops-engine/docker-compose.yml
version: '3.8'

services:
  ops-engine:
    build:
      context: .
      dockerfile: Dockerfile
      target: deps
    command: pnpm start:dev
    volumes:
      - .:/app
      - /app/node_modules
    ports:
      - "8002:8002"
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://zyenta:zyenta_dev@postgres:5432/zyenta
      - REDIS_HOST=redis
      - REDIS_PORT=6379
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: zyenta
      POSTGRES_PASSWORD: zyenta_dev
      POSTGRES_DB: zyenta
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

---

## 10. Testing

### 10.1 Jest Configuration

```javascript
// services/ops-engine/jest.config.js
module.exports = {
  moduleFileExtensions: ['js', 'json', 'ts'],
  rootDir: 'src',
  testRegex: '.*\\.spec\\.ts$',
  transform: {
    '^.+\\.(t|j)s$': 'ts-jest',
  },
  collectCoverageFrom: ['**/*.(t|j)s'],
  coverageDirectory: '../coverage',
  testEnvironment: 'node',
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/$1',
  },
};
```

### 10.2 Sync Processor Tests

```typescript
// services/ops-engine/src/modules/inventory/sync/__tests__/sync.processor.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { getQueueToken } from '@nestjs/bull';
import { SyncProcessor } from '../sync.processor';
import { PrismaService } from '@/shared/database/prisma.service';
import { RedisService } from '@/shared/cache/redis.service';
import { InventoryService } from '../../inventory.service';
import { SupplierFactory } from '../../suppliers/supplier.factory';

describe('SyncProcessor', () => {
  let processor: SyncProcessor;
  let inventoryService: InventoryService;
  let supplierFactory: SupplierFactory;

  const mockPrismaService = {
    product: {
      findMany: jest.fn(),
      update: jest.fn(),
    },
    syncLog: {
      create: jest.fn(),
    },
  };

  const mockRedisService = {
    del: jest.fn(),
    invalidatePattern: jest.fn(),
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        SyncProcessor,
        InventoryService,
        SupplierFactory,
        { provide: PrismaService, useValue: mockPrismaService },
        { provide: RedisService, useValue: mockRedisService },
      ],
    }).compile();

    processor = module.get<SyncProcessor>(SyncProcessor);
    inventoryService = module.get<InventoryService>(InventoryService);
    supplierFactory = module.get<SupplierFactory>(SupplierFactory);
  });

  it('should be defined', () => {
    expect(processor).toBeDefined();
  });

  describe('handleInventorySync', () => {
    it('should process inventory sync for a store', async () => {
      // Test implementation
    });

    it('should handle empty product list', async () => {
      // Test implementation
    });

    it('should update stock when changed', async () => {
      // Test implementation
    });
  });
});
```

---

## Verification Checklist

### Project Setup
- [ ] NestJS project initialized
- [ ] Dependencies installed
- [ ] TypeScript configured
- [ ] Module structure created

### Database Integration
- [ ] Prisma service working
- [ ] Database connection established
- [ ] Queries executing correctly

### Redis/BullMQ
- [ ] Redis connection working
- [ ] BullMQ queues registered
- [ ] Jobs can be added to queue
- [ ] Processor handles jobs

### Sync Scheduler
- [ ] High-frequency cron running
- [ ] Standard cron running
- [ ] Full sync cron running
- [ ] Jobs queued correctly

### Supplier Adapters
- [ ] AliExpress adapter implemented
- [ ] CJ Dropshipping adapter implemented
- [ ] Spocket adapter implemented
- [ ] Batch fetching working

### Testing
- [ ] Unit tests passing
- [ ] Integration tests passing
- [ ] API endpoints responding

---

## Next Steps

After completing Sprint 1, proceed to:

1. **Sprint 2: Price Management & Alerts**
   - Price calculator with strategies
   - Alert system
   - Notification service
   - Dashboard alert UI

---

*Phase 2 Sprint 1 establishes the foundational Ops Engine service with inventory synchronization capabilities, enabling automatic stock and availability updates from suppliers.*
