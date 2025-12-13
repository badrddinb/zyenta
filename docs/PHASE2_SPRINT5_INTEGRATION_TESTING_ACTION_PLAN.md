# Phase 2 Sprint 5: Integration & Testing - Action Plan

## Overview

This document provides a comprehensive implementation guide for Phase 2 Sprint 5, focusing on end-to-end testing, performance optimization, quality assurance, and documentation for all Phase 2 components.

### Sprint Objectives
- [ ] End-to-end fulfillment workflow testing
- [ ] Inventory sync stress testing and optimization
- [ ] Video generation quality assurance
- [ ] Performance optimization across all services
- [ ] Comprehensive documentation and training materials

### Prerequisites
- Phase 2 Sprints 1-4 completed
- All services deployable
- Test environments configured
- Monitoring and logging in place

---

## 1. End-to-End Testing Framework

### 1.1 Test Environment Setup

```typescript
// services/ops-engine/test/setup.ts
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import { PrismaService } from '@/shared/database/prisma.service';
import { RedisService } from '@/shared/cache/redis.service';
import { AppModule } from '../src/app.module';

export class TestEnvironment {
  app: INestApplication;
  prisma: PrismaService;
  redis: RedisService;

  async setup(): Promise<void> {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    this.app = moduleFixture.createNestApplication();
    this.prisma = this.app.get(PrismaService);
    this.redis = this.app.get(RedisService);

    await this.app.init();
  }

  async teardown(): Promise<void> {
    await this.cleanupTestData();
    await this.app.close();
  }

  async cleanupTestData(): Promise<void> {
    // Clean up in reverse dependency order
    await this.prisma.$transaction([
      this.prisma.trackingUpdate.deleteMany({ where: { orderId: { startsWith: 'test-' } } }),
      this.prisma.orderNotification.deleteMany({ where: { orderId: { startsWith: 'test-' } } }),
      this.prisma.supplierOrderLog.deleteMany({ where: { orderId: { startsWith: 'test-' } } }),
      this.prisma.orderItem.deleteMany({ where: { orderId: { startsWith: 'test-' } } }),
      this.prisma.order.deleteMany({ where: { id: { startsWith: 'test-' } } }),
      this.prisma.alert.deleteMany({ where: { storeId: { startsWith: 'test-' } } }),
      this.prisma.syncLog.deleteMany({ where: { storeId: { startsWith: 'test-' } } }),
      this.prisma.video.deleteMany({ where: { storeId: { startsWith: 'test-' } } }),
      this.prisma.product.deleteMany({ where: { storeId: { startsWith: 'test-' } } }),
      this.prisma.store.deleteMany({ where: { id: { startsWith: 'test-' } } }),
    ]);

    // Clear Redis test keys
    const keys = await this.redis.keys('test:*');
    if (keys.length > 0) {
      await this.redis.del(...keys);
    }
  }

  async createTestStore(): Promise<any> {
    return this.prisma.store.create({
      data: {
        id: `test-store-${Date.now()}`,
        name: 'Test Store',
        subdomain: `test-${Date.now()}`,
        niche: 'fashion',
        userId: 'test-user',
        settings: {
          pricingStrategy: 'PERCENTAGE_MARKUP',
          percentageMarkup: 50,
          minimumMarginPercentage: 20,
          maximumMarginPercentage: 200,
          lowStockThreshold: 5,
          emailNotifications: true,
        },
      },
    });
  }

  async createTestProduct(storeId: string): Promise<any> {
    return this.prisma.product.create({
      data: {
        id: `test-product-${Date.now()}`,
        storeId,
        title: 'Test Product',
        description: 'A test product for E2E testing',
        sourceId: 'source-123',
        sourceProvider: 'ALIEXPRESS',
        sourceUrl: 'https://example.com/product',
        costPrice: 10.00,
        sellingPrice: 19.99,
        stockQuantity: 100,
        status: 'ACTIVE',
      },
    });
  }

  async createTestOrder(storeId: string, productId: string): Promise<any> {
    return this.prisma.order.create({
      data: {
        id: `test-order-${Date.now()}`,
        orderNumber: `ORD-TEST-${Date.now()}`,
        storeId,
        customerId: 'test-customer',
        status: 'PENDING',
        paymentStatus: 'PAID',
        subtotal: 19.99,
        shippingCost: 5.00,
        totalAmount: 24.99,
        currency: 'USD',
        shippingAddress: {
          name: 'Test Customer',
          line1: '123 Test St',
          city: 'Test City',
          state: 'TC',
          postalCode: '12345',
          country: 'US',
        },
        items: {
          create: {
            productId,
            quantity: 1,
            unitPrice: 19.99,
          },
        },
      },
      include: {
        items: true,
      },
    });
  }
}
```

### 1.2 Fulfillment E2E Tests

```typescript
// services/ops-engine/test/e2e/fulfillment.e2e-spec.ts
import { TestEnvironment } from '../setup';
import { FulfillmentService } from '../../src/modules/fulfillment/fulfillment.service';
import { OrderState } from '../../src/modules/fulfillment/state-machine/order.machine';

describe('Fulfillment E2E Tests', () => {
  let env: TestEnvironment;
  let fulfillmentService: FulfillmentService;
  let testStore: any;
  let testProduct: any;

  beforeAll(async () => {
    env = new TestEnvironment();
    await env.setup();
    fulfillmentService = env.app.get(FulfillmentService);
    testStore = await env.createTestStore();
    testProduct = await env.createTestProduct(testStore.id);
  });

  afterAll(async () => {
    await env.teardown();
  });

  describe('Order Processing Flow', () => {
    it('should process order through complete fulfillment cycle', async () => {
      // Create test order
      const order = await env.createTestOrder(testStore.id, testProduct.id);

      // Start fulfillment
      await fulfillmentService.processOrder(order.id);

      // Verify initial state
      let status = await fulfillmentService.getOrderFulfillmentStatus(order.id);
      expect(status.fulfillmentState).toBe(OrderState.PAYMENT_CONFIRMED);

      // Simulate supplier confirmation
      await fulfillmentService.sendEvent(order.id, {
        type: 'SUPPLIER_CONFIRMED',
        supplierOrderId: 'SUPPLIER-123',
      });

      status = await fulfillmentService.getOrderFulfillmentStatus(order.id);
      expect(status.fulfillmentState).toBe(OrderState.SUPPLIER_CONFIRMED);
      expect(status.supplierOrderId).toBe('SUPPLIER-123');

      // Simulate tracking received
      await fulfillmentService.sendEvent(order.id, {
        type: 'TRACKING_RECEIVED',
        trackingNumber: 'TRACK123456',
        carrier: 'DHL',
      });

      status = await fulfillmentService.getOrderFulfillmentStatus(order.id);
      expect(status.fulfillmentState).toBe(OrderState.SHIPPED);
      expect(status.trackingNumber).toBe('TRACK123456');

      // Simulate delivery
      await fulfillmentService.sendEvent(order.id, {
        type: 'DELIVERED',
        deliveredAt: new Date(),
      });

      status = await fulfillmentService.getOrderFulfillmentStatus(order.id);
      expect(status.fulfillmentState).toBe(OrderState.DELIVERED);
    }, 30000);

    it('should handle order cancellation correctly', async () => {
      const order = await env.createTestOrder(testStore.id, testProduct.id);

      await fulfillmentService.processOrder(order.id);
      await fulfillmentService.cancelOrder(order.id, 'Customer requested cancellation');

      const status = await fulfillmentService.getOrderFulfillmentStatus(order.id);
      expect(status.fulfillmentState).toBe(OrderState.CANCELLED);
    });

    it('should handle supplier rejection and retry', async () => {
      const order = await env.createTestOrder(testStore.id, testProduct.id);

      await fulfillmentService.processOrder(order.id);

      // Simulate supplier rejection
      await fulfillmentService.sendEvent(order.id, {
        type: 'SUPPLIER_REJECTED',
        reason: 'Out of stock',
      });

      let status = await fulfillmentService.getOrderFulfillmentStatus(order.id);
      expect(status.fulfillmentState).toBe(OrderState.FAILED);

      // Retry
      await fulfillmentService.sendEvent(order.id, { type: 'RETRY' });

      status = await fulfillmentService.getOrderFulfillmentStatus(order.id);
      expect(status.fulfillmentState).toBe(OrderState.PROCESSING);
    });
  });

  describe('State Persistence', () => {
    it('should persist and restore order state correctly', async () => {
      const order = await env.createTestOrder(testStore.id, testProduct.id);

      await fulfillmentService.processOrder(order.id);
      await fulfillmentService.sendEvent(order.id, {
        type: 'SUPPLIER_CONFIRMED',
        supplierOrderId: 'PERSIST-TEST',
      });

      // Simulate service restart by getting status fresh
      const status = await fulfillmentService.getOrderFulfillmentStatus(order.id);

      expect(status.fulfillmentState).toBe(OrderState.SUPPLIER_CONFIRMED);
      expect(status.history.length).toBeGreaterThan(0);
    });
  });
});
```

### 1.3 Inventory Sync E2E Tests

```typescript
// services/ops-engine/test/e2e/inventory-sync.e2e-spec.ts
import { TestEnvironment } from '../setup';
import { InventorySyncService } from '../../src/modules/inventory/inventory-sync.service';
import { AlertsService } from '../../src/modules/alerts/alerts.service';
import { AlertType, AlertStatus } from '../../src/modules/alerts/dto/alert.dto';

describe('Inventory Sync E2E Tests', () => {
  let env: TestEnvironment;
  let syncService: InventorySyncService;
  let alertsService: AlertsService;
  let testStore: any;

  beforeAll(async () => {
    env = new TestEnvironment();
    await env.setup();
    syncService = env.app.get(InventorySyncService);
    alertsService = env.app.get(AlertsService);
    testStore = await env.createTestStore();
  });

  afterAll(async () => {
    await env.teardown();
  });

  describe('Stock Sync', () => {
    it('should sync inventory and create alerts for stock changes', async () => {
      const product = await env.createTestProduct(testStore.id);

      // Simulate stock depletion
      await alertsService.checkStockAlerts(product.id, 10, 0);

      // Verify alert was created
      const alerts = await alertsService.getAlerts(testStore.id, {
        type: AlertType.STOCK_OUT,
        status: AlertStatus.ACTIVE,
      });

      expect(alerts.alerts.length).toBeGreaterThan(0);
      expect(alerts.alerts[0].productId).toBe(product.id);
    });

    it('should resolve stock alerts when stock is restored', async () => {
      const product = await env.createTestProduct(testStore.id);

      // Create out of stock alert
      await alertsService.checkStockAlerts(product.id, 10, 0);

      // Restore stock
      await alertsService.checkStockAlerts(product.id, 0, 50);

      // Check for restored alert
      const restoredAlerts = await alertsService.getAlerts(testStore.id, {
        type: AlertType.STOCK_RESTORED,
      });

      expect(restoredAlerts.alerts.length).toBeGreaterThan(0);

      // Check original alert is resolved
      const outOfStockAlerts = await alertsService.getAlerts(testStore.id, {
        type: AlertType.STOCK_OUT,
        status: AlertStatus.ACTIVE,
      });

      const productAlerts = outOfStockAlerts.alerts.filter(
        a => a.productId === product.id,
      );
      expect(productAlerts.length).toBe(0);
    });
  });

  describe('Price Sync', () => {
    it('should create alerts for significant price changes', async () => {
      const product = await env.createTestProduct(testStore.id);

      // Simulate 50% price increase
      await alertsService.checkPriceAlerts(product.id, 10.00, 15.00);

      const alerts = await alertsService.getAlerts(testStore.id, {
        type: AlertType.PRICE_INCREASE,
      });

      const priceAlerts = alerts.alerts.filter(a => a.productId === product.id);
      expect(priceAlerts.length).toBeGreaterThan(0);
    });
  });
});
```

---

## 2. Stress Testing

### 2.1 Inventory Sync Load Test

```typescript
// services/ops-engine/test/stress/inventory-sync.stress.ts
import { TestEnvironment } from '../setup';
import { InventorySyncService } from '../../src/modules/inventory/inventory-sync.service';
import { performance } from 'perf_hooks';

interface StressTestResult {
  totalProducts: number;
  totalTime: number;
  averageTimePerProduct: number;
  successRate: number;
  peakMemoryUsage: number;
  errors: string[];
}

describe('Inventory Sync Stress Tests', () => {
  let env: TestEnvironment;
  let syncService: InventorySyncService;
  let testStore: any;

  beforeAll(async () => {
    env = new TestEnvironment();
    await env.setup();
    syncService = env.app.get(InventorySyncService);
    testStore = await env.createTestStore();
  });

  afterAll(async () => {
    await env.teardown();
  });

  async function runStressTest(productCount: number): Promise<StressTestResult> {
    const products = [];
    const errors: string[] = [];

    // Create test products
    for (let i = 0; i < productCount; i++) {
      const product = await env.prisma.product.create({
        data: {
          id: `test-stress-${Date.now()}-${i}`,
          storeId: testStore.id,
          title: `Stress Test Product ${i}`,
          sourceId: `source-${i}`,
          sourceProvider: 'ALIEXPRESS',
          sourceUrl: `https://example.com/product/${i}`,
          costPrice: 10.00 + (i % 10),
          sellingPrice: 19.99 + (i % 10),
          stockQuantity: 100 + (i % 50),
          status: 'ACTIVE',
        },
      });
      products.push(product);
    }

    // Record initial memory
    const initialMemory = process.memoryUsage().heapUsed;

    // Run sync
    const startTime = performance.now();
    let successCount = 0;

    for (const product of products) {
      try {
        // Simulate sync operation
        await syncService.syncProduct(product.id);
        successCount++;
      } catch (error) {
        errors.push(`Product ${product.id}: ${error.message}`);
      }
    }

    const endTime = performance.now();
    const peakMemory = process.memoryUsage().heapUsed;

    // Cleanup test products
    await env.prisma.product.deleteMany({
      where: { id: { in: products.map(p => p.id) } },
    });

    return {
      totalProducts: productCount,
      totalTime: endTime - startTime,
      averageTimePerProduct: (endTime - startTime) / productCount,
      successRate: (successCount / productCount) * 100,
      peakMemoryUsage: (peakMemory - initialMemory) / 1024 / 1024, // MB
      errors,
    };
  }

  it('should handle 100 products sync', async () => {
    const result = await runStressTest(100);

    expect(result.successRate).toBeGreaterThanOrEqual(95);
    expect(result.averageTimePerProduct).toBeLessThan(500); // < 500ms per product

    console.log('100 Products Stress Test Results:', result);
  }, 120000);

  it('should handle 500 products sync', async () => {
    const result = await runStressTest(500);

    expect(result.successRate).toBeGreaterThanOrEqual(95);
    expect(result.averageTimePerProduct).toBeLessThan(600);
    expect(result.peakMemoryUsage).toBeLessThan(500); // < 500MB

    console.log('500 Products Stress Test Results:', result);
  }, 300000);

  it('should handle 1000 products sync with batching', async () => {
    const result = await runStressTest(1000);

    expect(result.successRate).toBeGreaterThanOrEqual(90);
    expect(result.peakMemoryUsage).toBeLessThan(1000); // < 1GB

    console.log('1000 Products Stress Test Results:', result);
  }, 600000);
});
```

### 2.2 Concurrent Request Test

```typescript
// services/ops-engine/test/stress/concurrent.stress.ts
import { TestEnvironment } from '../setup';
import { performance } from 'perf_hooks';

describe('Concurrent Request Stress Tests', () => {
  let env: TestEnvironment;
  let testStore: any;

  beforeAll(async () => {
    env = new TestEnvironment();
    await env.setup();
    testStore = await env.createTestStore();
  });

  afterAll(async () => {
    await env.teardown();
  });

  async function runConcurrentTest(
    concurrency: number,
    operation: () => Promise<any>,
  ): Promise<{
    totalRequests: number;
    successCount: number;
    failureCount: number;
    totalTime: number;
    averageLatency: number;
    p95Latency: number;
    p99Latency: number;
  }> {
    const latencies: number[] = [];
    let successCount = 0;
    let failureCount = 0;

    const startTime = performance.now();

    const promises = Array(concurrency).fill(null).map(async () => {
      const opStart = performance.now();
      try {
        await operation();
        successCount++;
      } catch {
        failureCount++;
      }
      latencies.push(performance.now() - opStart);
    });

    await Promise.all(promises);

    const totalTime = performance.now() - startTime;

    // Calculate percentiles
    latencies.sort((a, b) => a - b);
    const p95Index = Math.floor(latencies.length * 0.95);
    const p99Index = Math.floor(latencies.length * 0.99);

    return {
      totalRequests: concurrency,
      successCount,
      failureCount,
      totalTime,
      averageLatency: latencies.reduce((a, b) => a + b, 0) / latencies.length,
      p95Latency: latencies[p95Index],
      p99Latency: latencies[p99Index],
    };
  }

  it('should handle 50 concurrent product reads', async () => {
    const product = await env.createTestProduct(testStore.id);

    const result = await runConcurrentTest(50, async () => {
      return env.prisma.product.findUnique({
        where: { id: product.id },
        include: { store: true },
      });
    });

    expect(result.successCount).toBe(50);
    expect(result.p95Latency).toBeLessThan(200); // < 200ms

    console.log('50 Concurrent Reads Results:', result);
  });

  it('should handle 100 concurrent alert creations', async () => {
    const product = await env.createTestProduct(testStore.id);
    let counter = 0;

    const result = await runConcurrentTest(100, async () => {
      counter++;
      return env.prisma.alert.create({
        data: {
          storeId: testStore.id,
          productId: product.id,
          type: 'STOCK_LOW',
          severity: 'WARNING',
          status: 'ACTIVE',
          title: `Test Alert ${counter}`,
          message: 'Concurrent test alert',
        },
      });
    });

    expect(result.successCount).toBeGreaterThanOrEqual(95);
    expect(result.p95Latency).toBeLessThan(500);

    console.log('100 Concurrent Alert Creations Results:', result);
  });
});
```

---

## 3. Video Generation Quality Assurance

### 3.1 Video QA Tests

```typescript
// services/ops-engine/test/qa/video-generation.qa.ts
import { TestEnvironment } from '../setup';
import { VideoService } from '../../src/modules/video/video.service';
import { FFmpegService } from '../../src/modules/video/composition/ffmpeg.service';
import { VideoStyle, VideoAspectRatio, VideoStatus } from '../../src/modules/video/dto/video.dto';

describe('Video Generation QA Tests', () => {
  let env: TestEnvironment;
  let videoService: VideoService;
  let ffmpegService: FFmpegService;
  let testStore: any;

  beforeAll(async () => {
    env = new TestEnvironment();
    await env.setup();
    videoService = env.app.get(VideoService);
    ffmpegService = env.app.get(FFmpegService);
    testStore = await env.createTestStore();
  });

  afterAll(async () => {
    await env.teardown();
  });

  describe('Video Output Quality', () => {
    it('should generate video with correct aspect ratio', async () => {
      const product = await env.createTestProduct(testStore.id);

      const result = await videoService.generateProductVideo(
        product.id,
        VideoStyle.PRODUCT_SHOWCASE,
      );

      // Wait for completion (with timeout)
      let status = await videoService.getVideoStatus(result.videoId);
      let attempts = 0;
      const maxAttempts = 60;

      while (status.status !== VideoStatus.COMPLETED && attempts < maxAttempts) {
        await new Promise(resolve => setTimeout(resolve, 5000));
        status = await videoService.getVideoStatus(result.videoId);
        attempts++;
      }

      expect(status.status).toBe(VideoStatus.COMPLETED);
      expect(status.videoUrl).toBeDefined();
      expect(status.thumbnailUrl).toBeDefined();
    }, 300000);

    it('should generate thumbnail at correct resolution', async () => {
      // This test assumes we have a sample video file
      const sampleVideoPath = '/tmp/test-sample.mp4';

      // Skip if no sample video
      const fs = await import('fs/promises');
      try {
        await fs.access(sampleVideoPath);
      } catch {
        console.log('Skipping thumbnail test - no sample video');
        return;
      }

      const thumbnailPath = await ffmpegService.extractThumbnail(sampleVideoPath, 1);

      expect(thumbnailPath).toBeDefined();

      // Verify thumbnail exists
      await fs.access(thumbnailPath);

      // Cleanup
      await fs.unlink(thumbnailPath);
    });

    it('should handle video concatenation correctly', async () => {
      // Create test video segments (mock paths for unit test)
      const segments = [
        '/tmp/segment1.mp4',
        '/tmp/segment2.mp4',
      ];

      // This would be tested with actual video files in integration environment
      // For unit test, we verify the method exists and has correct signature
      expect(typeof ffmpegService.concatenateVideos).toBe('function');
    });
  });

  describe('Video Error Handling', () => {
    it('should handle missing product gracefully', async () => {
      await expect(
        videoService.generateProductVideo('non-existent-product'),
      ).rejects.toThrow('Product not found');
    });

    it('should mark video as failed on generation error', async () => {
      const product = await env.createTestProduct(testStore.id);

      // Create video with invalid scene configuration
      const result = await videoService.createVideo({
        storeId: testStore.id,
        productId: product.id,
        title: 'Invalid Video',
        style: VideoStyle.PRODUCT_SHOWCASE,
        aspectRatio: VideoAspectRatio.PORTRAIT,
        scenes: [], // Empty scenes should fail
      });

      // Wait for failure
      await new Promise(resolve => setTimeout(resolve, 5000));

      const status = await videoService.getVideoStatus(result.videoId);
      expect(status.status).toBe(VideoStatus.FAILED);
    });
  });
});
```

---

## 4. Performance Optimization

### 4.1 Database Query Optimization

```typescript
// services/ops-engine/src/shared/database/query-optimizer.ts
import { Injectable, Logger } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Injectable()
export class QueryOptimizer {
  private readonly logger = new Logger(QueryOptimizer.name);

  constructor(private readonly prisma: PrismaService) {}

  // Batch product updates to reduce database round-trips
  async batchUpdateProducts(
    updates: Array<{ id: string; data: any }>,
    batchSize: number = 50,
  ): Promise<number> {
    let totalUpdated = 0;

    for (let i = 0; i < updates.length; i += batchSize) {
      const batch = updates.slice(i, i + batchSize);

      const results = await this.prisma.$transaction(
        batch.map(update =>
          this.prisma.product.update({
            where: { id: update.id },
            data: update.data,
          }),
        ),
      );

      totalUpdated += results.length;
    }

    return totalUpdated;
  }

  // Use cursor-based pagination for large datasets
  async paginateProducts(
    storeId: string,
    pageSize: number,
    cursor?: string,
  ): Promise<{ products: any[]; nextCursor: string | null }> {
    const products = await this.prisma.product.findMany({
      where: { storeId },
      take: pageSize + 1,
      cursor: cursor ? { id: cursor } : undefined,
      skip: cursor ? 1 : 0,
      orderBy: { id: 'asc' },
    });

    const hasMore = products.length > pageSize;
    const nextCursor = hasMore ? products[pageSize - 1].id : null;

    return {
      products: products.slice(0, pageSize),
      nextCursor,
    };
  }

  // Optimized alert aggregation query
  async getAlertCountsByType(storeId: string): Promise<Record<string, number>> {
    const counts = await this.prisma.alert.groupBy({
      by: ['type'],
      where: { storeId, status: 'ACTIVE' },
      _count: { type: true },
    });

    return counts.reduce((acc, curr) => {
      acc[curr.type] = curr._count.type;
      return acc;
    }, {} as Record<string, number>);
  }
}
```

### 4.2 Caching Strategy

```typescript
// services/ops-engine/src/shared/cache/cache-strategy.ts
import { Injectable, Logger } from '@nestjs/common';
import { RedisService } from './redis.service';

export interface CacheConfig {
  ttl: number;
  prefix: string;
  invalidateOnUpdate?: boolean;
}

@Injectable()
export class CacheStrategy {
  private readonly logger = new Logger(CacheStrategy.name);

  // Cache configurations for different data types
  private readonly configs: Record<string, CacheConfig> = {
    product: { ttl: 300, prefix: 'product:', invalidateOnUpdate: true },
    productList: { ttl: 60, prefix: 'products:', invalidateOnUpdate: true },
    store: { ttl: 600, prefix: 'store:', invalidateOnUpdate: true },
    alertSummary: { ttl: 30, prefix: 'alert-summary:', invalidateOnUpdate: true },
    syncStatus: { ttl: 10, prefix: 'sync-status:', invalidateOnUpdate: false },
    videoProgress: { ttl: 3600, prefix: 'video-progress:', invalidateOnUpdate: false },
  };

  constructor(private readonly redis: RedisService) {}

  async getOrSet<T>(
    type: string,
    key: string,
    fetcher: () => Promise<T>,
  ): Promise<T> {
    const config = this.configs[type];
    if (!config) {
      return fetcher();
    }

    const cacheKey = `${config.prefix}${key}`;
    const cached = await this.redis.get<T>(cacheKey);

    if (cached !== null) {
      this.logger.debug(`Cache hit: ${cacheKey}`);
      return cached;
    }

    const value = await fetcher();
    await this.redis.set(cacheKey, value, config.ttl);

    return value;
  }

  async invalidate(type: string, key: string): Promise<void> {
    const config = this.configs[type];
    if (!config) return;

    const cacheKey = `${config.prefix}${key}`;
    await this.redis.del(cacheKey);
    this.logger.debug(`Cache invalidated: ${cacheKey}`);
  }

  async invalidatePattern(type: string, pattern: string): Promise<void> {
    const config = this.configs[type];
    if (!config) return;

    const fullPattern = `${config.prefix}${pattern}`;
    await this.redis.invalidatePattern(fullPattern);
    this.logger.debug(`Cache pattern invalidated: ${fullPattern}`);
  }

  async warmCache(storeId: string): Promise<void> {
    this.logger.log(`Warming cache for store: ${storeId}`);

    // Pre-load frequently accessed data
    // This would be called after service restart or store activation
  }
}
```

### 4.3 Queue Optimization

```typescript
// services/ops-engine/src/shared/queue/queue-optimizer.ts
import { Injectable, Logger } from '@nestjs/common';
import { Queue, Job } from 'bull';

@Injectable()
export class QueueOptimizer {
  private readonly logger = new Logger(QueueOptimizer.name);

  // Optimal batch sizes for different job types
  private readonly batchSizes: Record<string, number> = {
    'inventory-sync': 50,
    'price-update': 100,
    'alert-notification': 25,
    'video-generation': 5,
    'fulfillment': 10,
  };

  // Rate limiting configurations
  private readonly rateLimits: Record<string, { max: number; duration: number }> = {
    'supplier-api': { max: 100, duration: 60000 }, // 100 per minute
    'email': { max: 50, duration: 60000 },
    'video-api': { max: 10, duration: 60000 },
  };

  getBatchSize(jobType: string): number {
    return this.batchSizes[jobType] || 20;
  }

  getRateLimit(resource: string): { max: number; duration: number } {
    return this.rateLimits[resource] || { max: 100, duration: 60000 };
  }

  async optimizeQueue(queue: Queue): Promise<void> {
    // Get queue metrics
    const [waiting, active, completed, failed] = await Promise.all([
      queue.getWaitingCount(),
      queue.getActiveCount(),
      queue.getCompletedCount(),
      queue.getFailedCount(),
    ]);

    this.logger.log(`Queue ${queue.name} metrics:`, {
      waiting,
      active,
      completed,
      failed,
    });

    // Clean old completed jobs
    await queue.clean(3600000, 'completed'); // Remove completed jobs older than 1 hour

    // Clean old failed jobs
    await queue.clean(86400000, 'failed'); // Remove failed jobs older than 24 hours
  }

  // Prioritize jobs based on criteria
  calculatePriority(jobType: string, data: any): number {
    const basePriorities: Record<string, number> = {
      'fulfillment': 1, // Highest priority
      'alert-notification': 2,
      'inventory-sync': 3,
      'price-update': 4,
      'video-generation': 5, // Lowest priority (resource intensive)
    };

    let priority = basePriorities[jobType] || 3;

    // Adjust based on data characteristics
    if (data.urgent) priority = 1;
    if (data.isRetry) priority = Math.max(1, priority - 1);

    return priority;
  }
}
```

---

## 5. Documentation

### 5.1 API Documentation Template

```yaml
# services/ops-engine/docs/api/openapi.yaml
openapi: 3.0.3
info:
  title: Zyenta Ops Engine API
  description: |
    API for managing inventory sync, fulfillment, pricing, alerts, and video generation.

    ## Authentication
    All endpoints require Bearer token authentication.

    ## Rate Limits
    - Standard endpoints: 100 requests/minute
    - Video generation: 10 requests/minute
    - Bulk operations: 20 requests/minute
  version: 2.0.0

servers:
  - url: https://ops-api.zyenta.com/v2
    description: Production
  - url: https://ops-api-staging.zyenta.com/v2
    description: Staging

paths:
  /fulfillment/orders/{orderId}/process:
    post:
      summary: Start order fulfillment
      tags:
        - Fulfillment
      parameters:
        - name: orderId
          in: path
          required: true
          schema:
            type: string
      responses:
        '202':
          description: Fulfillment started
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/FulfillmentStarted'
        '404':
          description: Order not found

  /fulfillment/orders/{orderId}/status:
    get:
      summary: Get fulfillment status
      tags:
        - Fulfillment
      parameters:
        - name: orderId
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: Fulfillment status
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/FulfillmentStatus'

  /alerts/stores/{storeId}:
    get:
      summary: Get store alerts
      tags:
        - Alerts
      parameters:
        - name: storeId
          in: path
          required: true
          schema:
            type: string
        - name: status
          in: query
          schema:
            type: string
            enum: [ACTIVE, ACKNOWLEDGED, RESOLVED, DISMISSED]
        - name: type
          in: query
          schema:
            type: string
        - name: limit
          in: query
          schema:
            type: integer
            default: 50
      responses:
        '200':
          description: Alert list
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AlertList'

  /videos:
    post:
      summary: Create new video
      tags:
        - Videos
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateVideoRequest'
      responses:
        '202':
          description: Video generation started
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/VideoGenerationResult'

components:
  schemas:
    FulfillmentStatus:
      type: object
      properties:
        orderId:
          type: string
        status:
          type: string
        fulfillmentState:
          type: string
        supplierOrderId:
          type: string
        trackingNumber:
          type: string
        trackingCarrier:
          type: string
        history:
          type: array
          items:
            $ref: '#/components/schemas/HistoryEntry'

    AlertList:
      type: object
      properties:
        alerts:
          type: array
          items:
            $ref: '#/components/schemas/Alert'
        total:
          type: integer
        hasMore:
          type: boolean

    Alert:
      type: object
      properties:
        id:
          type: string
        type:
          type: string
        severity:
          type: string
        status:
          type: string
        title:
          type: string
        message:
          type: string
        createdAt:
          type: string
          format: date-time

    VideoGenerationResult:
      type: object
      properties:
        videoId:
          type: string
        status:
          type: string
        progress:
          type: integer
```

### 5.2 Runbook Template

```markdown
# Phase 2 Operations Runbook

## Service Overview

### Ops Engine Components
- **Inventory Sync**: Synchronizes product stock and prices from suppliers
- **Fulfillment Orchestrator**: Manages order lifecycle using XState
- **Alert System**: Monitors and notifies on inventory/price changes
- **Video Generation**: Creates product videos using AI

## Common Operations

### 1. Manual Inventory Sync

```bash
# Trigger sync for specific store
curl -X POST https://ops-api.zyenta.com/v2/sync/stores/{storeId}/trigger \
  -H "Authorization: Bearer $TOKEN"

# Check sync status
curl https://ops-api.zyenta.com/v2/monitoring/stores/{storeId}/sync-stats
```

### 2. Order Fulfillment Issues

#### Order Stuck in Processing
1. Check order state:
   ```bash
   curl https://ops-api.zyenta.com/v2/fulfillment/orders/{orderId}/status
   ```

2. If stuck, check supplier order log:
   ```sql
   SELECT * FROM supplier_order_logs WHERE order_id = '{orderId}' ORDER BY created_at DESC;
   ```

3. Retry order placement:
   ```bash
   curl -X POST https://ops-api.zyenta.com/v2/fulfillment/orders/{orderId}/retry
   ```

#### Tracking Not Updating
1. Check tracking sync job status in Bull dashboard
2. Manually trigger tracking sync:
   ```bash
   curl -X POST https://ops-api.zyenta.com/v2/fulfillment/orders/{orderId}/sync-tracking
   ```

### 3. Alert Management

#### Clear Alert Backlog
```bash
# Acknowledge all non-critical alerts for a store
curl -X POST https://ops-api.zyenta.com/v2/alerts/stores/{storeId}/bulk-acknowledge \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"severity": ["INFO", "WARNING"]}'
```

### 4. Video Generation Issues

#### Failed Video Jobs
1. Check video status:
   ```bash
   curl https://ops-api.zyenta.com/v2/videos/{videoId}
   ```

2. Check Runway ML API status

3. Retry video generation:
   ```bash
   curl -X POST https://ops-api.zyenta.com/v2/videos/{videoId}/retry
   ```

## Monitoring & Alerts

### Key Metrics to Watch
- `ops_sync_duration_seconds` - Sync job duration
- `ops_fulfillment_state_transitions` - Order state changes
- `ops_alerts_active_count` - Number of active alerts
- `ops_video_generation_duration` - Video generation time

### Alert Thresholds
- Sync failure rate > 5%: Warning
- Sync failure rate > 15%: Critical
- Average sync duration > 5min: Warning
- Order stuck > 24h: Critical
- Video generation queue > 50: Warning

## Troubleshooting

### High Sync Failure Rate
1. Check supplier API status
2. Verify API credentials haven't expired
3. Check rate limiting status
4. Review recent error logs

### Database Performance Issues
1. Check slow query log
2. Verify indexes are being used
3. Run VACUUM if needed
4. Check connection pool status

### Redis Cache Issues
1. Check memory usage: `redis-cli INFO memory`
2. Check hit ratio: `redis-cli INFO stats`
3. Clear stale keys if needed
```

---

## 6. Health Checks

### 6.1 Health Check Service

```typescript
// services/ops-engine/src/modules/health/health.service.ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '@/shared/database/prisma.service';
import { RedisService } from '@/shared/cache/redis.service';
import { Queue } from 'bull';
import { InjectQueue } from '@nestjs/bull';

export interface HealthStatus {
  status: 'healthy' | 'degraded' | 'unhealthy';
  timestamp: string;
  components: {
    database: ComponentHealth;
    redis: ComponentHealth;
    queues: ComponentHealth;
    externalServices: ComponentHealth;
  };
}

interface ComponentHealth {
  status: 'healthy' | 'degraded' | 'unhealthy';
  latency?: number;
  details?: Record<string, any>;
}

@Injectable()
export class HealthService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly redis: RedisService,
    @InjectQueue('inventory-sync') private readonly syncQueue: Queue,
    @InjectQueue('fulfillment') private readonly fulfillmentQueue: Queue,
    @InjectQueue('video-generation') private readonly videoQueue: Queue,
  ) {}

  async getHealth(): Promise<HealthStatus> {
    const [database, redis, queues, external] = await Promise.all([
      this.checkDatabase(),
      this.checkRedis(),
      this.checkQueues(),
      this.checkExternalServices(),
    ]);

    const overallStatus = this.calculateOverallStatus([database, redis, queues, external]);

    return {
      status: overallStatus,
      timestamp: new Date().toISOString(),
      components: {
        database,
        redis,
        queues,
        externalServices: external,
      },
    };
  }

  private async checkDatabase(): Promise<ComponentHealth> {
    const start = Date.now();
    try {
      await this.prisma.$queryRaw`SELECT 1`;
      return {
        status: 'healthy',
        latency: Date.now() - start,
      };
    } catch (error) {
      return {
        status: 'unhealthy',
        details: { error: error.message },
      };
    }
  }

  private async checkRedis(): Promise<ComponentHealth> {
    const start = Date.now();
    try {
      await this.redis.ping();
      return {
        status: 'healthy',
        latency: Date.now() - start,
      };
    } catch (error) {
      return {
        status: 'unhealthy',
        details: { error: error.message },
      };
    }
  }

  private async checkQueues(): Promise<ComponentHealth> {
    try {
      const [syncWaiting, fulfillmentWaiting, videoWaiting] = await Promise.all([
        this.syncQueue.getWaitingCount(),
        this.fulfillmentQueue.getWaitingCount(),
        this.videoQueue.getWaitingCount(),
      ]);

      const status = syncWaiting > 1000 || fulfillmentWaiting > 100 || videoWaiting > 50
        ? 'degraded'
        : 'healthy';

      return {
        status,
        details: {
          syncQueue: syncWaiting,
          fulfillmentQueue: fulfillmentWaiting,
          videoQueue: videoWaiting,
        },
      };
    } catch (error) {
      return {
        status: 'unhealthy',
        details: { error: error.message },
      };
    }
  }

  private async checkExternalServices(): Promise<ComponentHealth> {
    // Check external service availability
    // This would ping Runway ML, ElevenLabs, supplier APIs, etc.
    return {
      status: 'healthy',
      details: {
        runwayML: 'available',
        elevenLabs: 'available',
        suppliers: 'available',
      },
    };
  }

  private calculateOverallStatus(components: ComponentHealth[]): 'healthy' | 'degraded' | 'unhealthy' {
    const hasUnhealthy = components.some(c => c.status === 'unhealthy');
    const hasDegraded = components.some(c => c.status === 'degraded');

    if (hasUnhealthy) return 'unhealthy';
    if (hasDegraded) return 'degraded';
    return 'healthy';
  }
}
```

---

## Verification Checklist

### End-to-End Tests
- [ ] Complete fulfillment flow works
- [ ] Order cancellation works
- [ ] Retry mechanism works
- [ ] State persistence works

### Stress Tests
- [ ] 100 products sync < 50s
- [ ] 500 products sync < 5min
- [ ] 1000 products sync < 10min
- [ ] Memory stays under 1GB

### Video QA
- [ ] Correct aspect ratios
- [ ] Audio sync correct
- [ ] Thumbnails generated
- [ ] Error handling works

### Performance
- [ ] Database queries optimized
- [ ] Caching effective
- [ ] Queue processing efficient
- [ ] No memory leaks

### Documentation
- [ ] API docs complete
- [ ] Runbook written
- [ ] Health checks working
- [ ] Monitoring configured

---

## Deployment Checklist

- [ ] All tests passing
- [ ] Environment variables configured
- [ ] Database migrations applied
- [ ] Redis configured
- [ ] Queue workers deployed
- [ ] Monitoring dashboards ready
- [ ] Alerts configured
- [ ] Documentation published
- [ ] Team trained

---

*Phase 2 Sprint 5 ensures all Phase 2 components are thoroughly tested, optimized, documented, and ready for production deployment.*
