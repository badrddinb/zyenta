# Phase 2: "Ops" Foundation - Production Execution Plan

## Overview

This document outlines the complete execution plan for building Zyenta's Phase 2: **The Operations Foundation**. This phase establishes the autonomous 24/7 operations layer that fulfills the "Zero-Effort" promise.

### Phase 2 Objectives
- [ ] Implement 24/7 Inventory & Price Sync Agent
- [ ] Build Automated Fulfillment Orchestrator
- [ ] Add Video Generation to Media Studio
- [ ] Launch "Autopilot" tier

### Prerequisites
- Phase 1 fully completed and deployed
- Genesis Engine operational
- Media Studio (Images) operational
- Stripe Connect integration working
- At least 10 beta stores running

---

## 1. Operations Engine Service Architecture

### 1.1 Service Structure

```
services/ops-engine/
├── src/
│   ├── main.ts                    # NestJS app entry
│   ├── app.module.ts              # Root module
│   │
│   ├── config/
│   │   ├── configuration.ts       # Environment config
│   │   └── validation.schema.ts   # Config validation
│   │
│   ├── modules/
│   │   ├── inventory/
│   │   │   ├── inventory.module.ts
│   │   │   ├── inventory.service.ts
│   │   │   ├── inventory.controller.ts
│   │   │   ├── sync/
│   │   │   │   ├── sync.service.ts
│   │   │   │   ├── sync.scheduler.ts
│   │   │   │   └── sync.processor.ts
│   │   │   ├── price/
│   │   │   │   ├── price.service.ts
│   │   │   │   ├── price.calculator.ts
│   │   │   │   └── margin.strategy.ts
│   │   │   └── alerts/
│   │   │       ├── alert.service.ts
│   │   │       └── notification.service.ts
│   │   │
│   │   ├── fulfillment/
│   │   │   ├── fulfillment.module.ts
│   │   │   ├── fulfillment.service.ts
│   │   │   ├── fulfillment.controller.ts
│   │   │   ├── orchestrator/
│   │   │   │   ├── orchestrator.service.ts
│   │   │   │   ├── workflow.engine.ts
│   │   │   │   └── state.machine.ts
│   │   │   ├── suppliers/
│   │   │   │   ├── supplier.factory.ts
│   │   │   │   ├── aliexpress.adapter.ts
│   │   │   │   ├── cj-dropshipping.adapter.ts
│   │   │   │   └── spocket.adapter.ts
│   │   │   ├── tracking/
│   │   │   │   ├── tracking.service.ts
│   │   │   │   └── carrier.adapters/
│   │   │   └── notifications/
│   │   │       ├── email.service.ts
│   │   │       └── templates/
│   │   │
│   │   ├── webhooks/
│   │   │   ├── webhooks.module.ts
│   │   │   ├── webhooks.controller.ts
│   │   │   ├── stripe.handler.ts
│   │   │   ├── supplier.handler.ts
│   │   │   └── verification.guard.ts
│   │   │
│   │   └── health/
│   │       ├── health.module.ts
│   │       └── health.controller.ts
│   │
│   ├── shared/
│   │   ├── database/
│   │   │   └── prisma.service.ts
│   │   ├── queue/
│   │   │   ├── queue.module.ts
│   │   │   └── processors/
│   │   ├── cache/
│   │   │   └── redis.service.ts
│   │   └── events/
│   │       ├── event-bus.service.ts
│   │       └── event.types.ts
│   │
│   └── utils/
│       ├── retry.util.ts
│       ├── encryption.util.ts
│       └── currency.util.ts
│
├── test/
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
├── package.json
├── tsconfig.json
├── nest-cli.json
├── Dockerfile
└── docker-compose.yml
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
    "start:prod": "node dist/main",
    "lint": "eslint \"{src,test}/**/*.ts\"",
    "test": "jest",
    "test:e2e": "jest --config ./test/jest-e2e.json"
  },
  "dependencies": {
    "@nestjs/common": "^10.3.0",
    "@nestjs/core": "^10.3.0",
    "@nestjs/config": "^3.1.1",
    "@nestjs/schedule": "^4.0.0",
    "@nestjs/bull": "^10.0.1",
    "@nestjs/event-emitter": "^2.0.3",
    "@nestjs/terminus": "^10.2.0",
    "@zyenta/database": "workspace:*",
    "@zyenta/shared-types": "workspace:*",
    "@zyenta/queue": "workspace:*",
    "bull": "^4.12.0",
    "ioredis": "^5.3.2",
    "xstate": "^5.5.0",
    "stripe": "^14.10.0",
    "nodemailer": "^6.9.7",
    "handlebars": "^4.7.8",
    "axios": "^1.6.2",
    "crypto-js": "^4.2.0",
    "class-validator": "^0.14.0",
    "class-transformer": "^0.5.1"
  },
  "devDependencies": {
    "@nestjs/cli": "^10.2.1",
    "@nestjs/testing": "^10.3.0",
    "@types/node": "^20.10.5",
    "@types/jest": "^29.5.11",
    "jest": "^29.7.0",
    "typescript": "^5.3.3"
  }
}
```

---

## 2. Inventory & Price Sync Agent

### 2.1 Sync Scheduler

```typescript
// services/ops-engine/src/modules/inventory/sync/sync.scheduler.ts
import { Injectable, Logger } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';
import { InjectQueue } from '@nestjs/bull';
import { Queue } from 'bull';
import { PrismaService } from '@/shared/database/prisma.service';

@Injectable()
export class SyncScheduler {
  private readonly logger = new Logger(SyncScheduler.name);

  constructor(
    private readonly prisma: PrismaService,
    @InjectQueue('inventory-sync') private readonly syncQueue: Queue,
  ) {}

  // Run every 15 minutes for active stores
  @Cron(CronExpression.EVERY_15_MINUTES)
  async scheduleHighFrequencySync() {
    this.logger.log('Starting high-frequency inventory sync...');

    const activeStores = await this.prisma.store.findMany({
      where: {
        status: 'ACTIVE',
        settings: {
          path: ['syncFrequency'],
          equals: 'HIGH',
        },
      },
      select: {
        id: true,
        userId: true,
      },
    });

    for (const store of activeStores) {
      await this.syncQueue.add(
        'sync-store-inventory',
        {
          storeId: store.id,
          userId: store.userId,
          priority: 'high',
        },
        {
          priority: 1,
          attempts: 3,
          backoff: {
            type: 'exponential',
            delay: 5000,
          },
        },
      );
    }

    this.logger.log(`Queued ${activeStores.length} high-frequency syncs`);
  }

  // Run every hour for standard stores
  @Cron(CronExpression.EVERY_HOUR)
  async scheduleStandardSync() {
    this.logger.log('Starting standard inventory sync...');

    const stores = await this.prisma.store.findMany({
      where: {
        status: 'ACTIVE',
        OR: [
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
        ],
      },
      select: {
        id: true,
        userId: true,
      },
    });

    for (const store of stores) {
      await this.syncQueue.add(
        'sync-store-inventory',
        {
          storeId: store.id,
          userId: store.userId,
          priority: 'standard',
        },
        {
          priority: 5,
          attempts: 3,
          backoff: {
            type: 'exponential',
            delay: 10000,
          },
        },
      );
    }

    this.logger.log(`Queued ${stores.length} standard syncs`);
  }

  // Daily full sync at 3 AM UTC
  @Cron('0 3 * * *')
  async scheduleFullSync() {
    this.logger.log('Starting daily full inventory sync...');

    const allStores = await this.prisma.store.findMany({
      where: { status: 'ACTIVE' },
      select: { id: true, userId: true },
    });

    for (const store of allStores) {
      await this.syncQueue.add(
        'full-sync-store-inventory',
        {
          storeId: store.id,
          userId: store.userId,
          fullSync: true,
        },
        {
          priority: 10,
          attempts: 3,
        },
      );
    }

    this.logger.log(`Queued ${allStores.length} full syncs`);
  }
}
```

### 2.2 Sync Processor

```typescript
// services/ops-engine/src/modules/inventory/sync/sync.processor.ts
import { Process, Processor } from '@nestjs/bull';
import { Logger } from '@nestjs/common';
import { Job } from 'bull';
import { PrismaService } from '@/shared/database/prisma.service';
import { SupplierFactory } from '@/modules/fulfillment/suppliers/supplier.factory';
import { PriceCalculator } from '../price/price.calculator';
import { AlertService } from '../alerts/alert.service';
import { CacheService } from '@/shared/cache/redis.service';

interface SyncJobData {
  storeId: string;
  userId: string;
  priority: 'high' | 'standard';
  fullSync?: boolean;
}

interface ProductSyncResult {
  productId: string;
  sourceId: string;
  previousStock: number;
  currentStock: number;
  previousPrice: number;
  currentPrice: number;
  changes: string[];
}

@Processor('inventory-sync')
export class SyncProcessor {
  private readonly logger = new Logger(SyncProcessor.name);

  constructor(
    private readonly prisma: PrismaService,
    private readonly supplierFactory: SupplierFactory,
    private readonly priceCalculator: PriceCalculator,
    private readonly alertService: AlertService,
    private readonly cacheService: CacheService,
  ) {}

  @Process('sync-store-inventory')
  async handleInventorySync(job: Job<SyncJobData>) {
    const { storeId, userId, priority } = job.data;
    this.logger.log(`Processing inventory sync for store: ${storeId}`);

    const startTime = Date.now();
    const results: ProductSyncResult[] = [];

    try {
      // Get store products with supplier info
      const products = await this.prisma.product.findMany({
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

      // Group products by supplier
      const productsBySupplier = this.groupBySupplier(products);

      // Sync each supplier's products
      for (const [provider, supplierProducts] of Object.entries(productsBySupplier)) {
        const supplierAccount = products[0].store.user.supplierAccounts.find(
          (acc) => acc.provider === provider,
        );

        if (!supplierAccount) {
          this.logger.warn(`No supplier account found for ${provider}`);
          continue;
        }

        const supplier = this.supplierFactory.create(provider, supplierAccount);
        const sourceIds = supplierProducts.map((p) => p.sourceId);

        // Batch fetch from supplier API
        const supplierData = await supplier.getProductsBatch(sourceIds);

        // Process each product
        for (const product of supplierProducts) {
          const sourceData = supplierData.get(product.sourceId);
          if (!sourceData) continue;

          const result = await this.syncProduct(product, sourceData);
          results.push(result);

          // Update progress
          job.progress(
            Math.round((results.length / products.length) * 100),
          );
        }
      }

      // Handle out-of-stock alerts
      const stockAlerts = results.filter(
        (r) => r.currentStock === 0 && r.previousStock > 0,
      );
      if (stockAlerts.length > 0) {
        await this.alertService.sendStockAlert(storeId, stockAlerts);
      }

      // Handle significant price changes
      const priceAlerts = results.filter((r) => {
        const priceChange = Math.abs(r.currentPrice - r.previousPrice) / r.previousPrice;
        return priceChange > 0.1; // > 10% change
      });
      if (priceAlerts.length > 0) {
        await this.alertService.sendPriceAlert(storeId, priceAlerts);
      }

      // Record sync completion
      await this.recordSyncCompletion(storeId, results, startTime);

      this.logger.log(
        `Completed sync for store ${storeId}: ${results.length} products processed`,
      );

      return { success: true, processed: results.length, results };
    } catch (error) {
      this.logger.error(`Sync failed for store ${storeId}:`, error);
      await this.alertService.sendSyncFailureAlert(storeId, error);
      throw error;
    }
  }

  @Process('full-sync-store-inventory')
  async handleFullSync(job: Job<SyncJobData>) {
    // Similar to above but includes all products and deeper validation
    return this.handleInventorySync(job);
  }

  private async syncProduct(
    product: any,
    sourceData: any,
  ): Promise<ProductSyncResult> {
    const changes: string[] = [];
    const previousStock = product.stockQuantity;
    const previousPrice = Number(product.costPrice);

    // Check stock changes
    if (sourceData.stock !== product.stockQuantity) {
      changes.push(`Stock: ${product.stockQuantity} → ${sourceData.stock}`);
    }

    // Check cost price changes
    if (sourceData.price !== Number(product.costPrice)) {
      changes.push(
        `Cost: ${product.costPrice} → ${sourceData.price}`,
      );
    }

    // Calculate new selling price if cost changed
    let newSellingPrice = Number(product.sellingPrice);
    if (sourceData.price !== Number(product.costPrice)) {
      newSellingPrice = await this.priceCalculator.calculateSellingPrice(
        sourceData.price,
        product.storeId,
      );
    }

    // Determine new status
    let newStatus = product.status;
    if (sourceData.stock === 0 && product.status === 'ACTIVE') {
      newStatus = 'OUT_OF_STOCK';
      changes.push('Status: ACTIVE → OUT_OF_STOCK');
    } else if (sourceData.stock > 0 && product.status === 'OUT_OF_STOCK') {
      newStatus = 'ACTIVE';
      changes.push('Status: OUT_OF_STOCK → ACTIVE');
    }

    // Update product if changes detected
    if (changes.length > 0) {
      await this.prisma.product.update({
        where: { id: product.id },
        data: {
          stockQuantity: sourceData.stock,
          costPrice: sourceData.price,
          sellingPrice: newSellingPrice,
          status: newStatus,
          updatedAt: new Date(),
        },
      });

      // Invalidate cache
      await this.cacheService.invalidate(`product:${product.id}`);
      await this.cacheService.invalidate(`store:${product.storeId}:products`);
    }

    return {
      productId: product.id,
      sourceId: product.sourceId,
      previousStock,
      currentStock: sourceData.stock,
      previousPrice,
      currentPrice: sourceData.price,
      changes,
    };
  }

  private groupBySupplier(products: any[]): Record<string, any[]> {
    return products.reduce((acc, product) => {
      const provider = product.sourceProvider;
      if (!acc[provider]) acc[provider] = [];
      acc[provider].push(product);
      return acc;
    }, {});
  }

  private async recordSyncCompletion(
    storeId: string,
    results: ProductSyncResult[],
    startTime: number,
  ) {
    const duration = Date.now() - startTime;
    const changedProducts = results.filter((r) => r.changes.length > 0);

    await this.prisma.syncLog.create({
      data: {
        storeId,
        type: 'INVENTORY',
        status: 'COMPLETED',
        productsProcessed: results.length,
        productsChanged: changedProducts.length,
        duration,
        metadata: {
          stockChanges: results.filter(
            (r) => r.previousStock !== r.currentStock,
          ).length,
          priceChanges: results.filter(
            (r) => r.previousPrice !== r.currentPrice,
          ).length,
        },
      },
    });
  }
}
```

### 2.3 Price Calculator

```typescript
// services/ops-engine/src/modules/inventory/price/price.calculator.ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '@/shared/database/prisma.service';

export interface PricingStrategy {
  type: 'FIXED_MARGIN' | 'PERCENTAGE_MARGIN' | 'COMPETITIVE' | 'DYNAMIC';
  value: number; // margin value or target
  minMargin?: number;
  maxMargin?: number;
  roundTo?: number; // e.g., 0.99 for psychological pricing
}

@Injectable()
export class PriceCalculator {
  constructor(private readonly prisma: PrismaService) {}

  async calculateSellingPrice(
    costPrice: number,
    storeId: string,
  ): Promise<number> {
    // Get store pricing strategy
    const store = await this.prisma.store.findUnique({
      where: { id: storeId },
      select: {
        settings: true,
      },
    });

    const strategy: PricingStrategy = (store?.settings as any)?.pricingStrategy || {
      type: 'PERCENTAGE_MARGIN',
      value: 50, // 50% margin by default
      roundTo: 0.99,
    };

    let sellingPrice: number;

    switch (strategy.type) {
      case 'FIXED_MARGIN':
        sellingPrice = costPrice + strategy.value;
        break;

      case 'PERCENTAGE_MARGIN':
        sellingPrice = costPrice * (1 + strategy.value / 100);
        break;

      case 'COMPETITIVE':
        // Would integrate with market price data
        sellingPrice = costPrice * 1.4; // 40% margin as fallback
        break;

      case 'DYNAMIC':
        // Would use ML model for price optimization
        sellingPrice = await this.calculateDynamicPrice(costPrice, storeId);
        break;

      default:
        sellingPrice = costPrice * 1.5; // 50% margin fallback
    }

    // Apply min/max margin constraints
    if (strategy.minMargin !== undefined) {
      const minPrice = costPrice * (1 + strategy.minMargin / 100);
      sellingPrice = Math.max(sellingPrice, minPrice);
    }

    if (strategy.maxMargin !== undefined) {
      const maxPrice = costPrice * (1 + strategy.maxMargin / 100);
      sellingPrice = Math.min(sellingPrice, maxPrice);
    }

    // Apply psychological pricing (e.g., $19.99 instead of $20)
    if (strategy.roundTo) {
      sellingPrice = this.applyPsychologicalPricing(sellingPrice, strategy.roundTo);
    }

    return Math.round(sellingPrice * 100) / 100; // Round to 2 decimal places
  }

  private applyPsychologicalPricing(price: number, roundTo: number): number {
    const decimal = roundTo % 1;
    const base = Math.floor(price);

    if (price - base > 0.5) {
      return base + 1 + decimal - 1; // e.g., 19.99
    }
    return base + decimal - 1; // e.g., 18.99
  }

  private async calculateDynamicPrice(
    costPrice: number,
    storeId: string,
  ): Promise<number> {
    // Placeholder for ML-based dynamic pricing
    // Would consider: competitor prices, demand, inventory levels, etc.
    return costPrice * 1.45;
  }

  async calculateBulkPrices(
    products: Array<{ id: string; costPrice: number }>,
    storeId: string,
  ): Promise<Map<string, number>> {
    const prices = new Map<string, number>();

    for (const product of products) {
      const sellingPrice = await this.calculateSellingPrice(
        product.costPrice,
        storeId,
      );
      prices.set(product.id, sellingPrice);
    }

    return prices;
  }
}
```

### 2.4 Alert Service

```typescript
// services/ops-engine/src/modules/inventory/alerts/alert.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { PrismaService } from '@/shared/database/prisma.service';
import { NotificationService } from './notification.service';

@Injectable()
export class AlertService {
  private readonly logger = new Logger(AlertService.name);

  constructor(
    private readonly prisma: PrismaService,
    private readonly notificationService: NotificationService,
  ) {}

  async sendStockAlert(
    storeId: string,
    outOfStockProducts: Array<{
      productId: string;
      sourceId: string;
      previousStock: number;
    }>,
  ) {
    const store = await this.prisma.store.findUnique({
      where: { id: storeId },
      include: {
        user: { select: { email: true, name: true } },
        products: {
          where: {
            id: { in: outOfStockProducts.map((p) => p.productId) },
          },
          select: { id: true, title: true },
        },
      },
    });

    if (!store) return;

    // Create alert record
    await this.prisma.alert.create({
      data: {
        storeId,
        type: 'STOCK_OUT',
        severity: 'HIGH',
        title: `${outOfStockProducts.length} product(s) out of stock`,
        message: `The following products are now out of stock and have been hidden from your storefront.`,
        metadata: {
          products: outOfStockProducts.map((p) => ({
            id: p.productId,
            title: store.products.find((sp) => sp.id === p.productId)?.title,
            previousStock: p.previousStock,
          })),
        },
      },
    });

    // Send email notification
    await this.notificationService.sendEmail({
      to: store.user.email,
      subject: `[${store.name}] Stock Alert: ${outOfStockProducts.length} product(s) out of stock`,
      template: 'stock-alert',
      data: {
        storeName: store.name,
        userName: store.user.name,
        products: store.products,
        dashboardUrl: `${process.env.DASHBOARD_URL}/stores/${storeId}/products`,
      },
    });

    this.logger.log(`Stock alert sent for store ${storeId}`);
  }

  async sendPriceAlert(
    storeId: string,
    priceChangedProducts: Array<{
      productId: string;
      previousPrice: number;
      currentPrice: number;
    }>,
  ) {
    const store = await this.prisma.store.findUnique({
      where: { id: storeId },
      include: {
        user: { select: { email: true, name: true } },
        products: {
          where: {
            id: { in: priceChangedProducts.map((p) => p.productId) },
          },
          select: { id: true, title: true, sellingPrice: true },
        },
      },
    });

    if (!store) return;

    const significantChanges = priceChangedProducts.map((p) => {
      const product = store.products.find((sp) => sp.id === p.productId);
      const changePercent = Math.round(
        ((p.currentPrice - p.previousPrice) / p.previousPrice) * 100,
      );
      return {
        id: p.productId,
        title: product?.title,
        previousCost: p.previousPrice,
        newCost: p.currentPrice,
        newSellingPrice: product?.sellingPrice,
        changePercent,
      };
    });

    await this.prisma.alert.create({
      data: {
        storeId,
        type: 'PRICE_CHANGE',
        severity: 'MEDIUM',
        title: `Significant price changes detected`,
        message: `${priceChangedProducts.length} product(s) had supplier price changes > 10%. Your selling prices have been automatically adjusted.`,
        metadata: { products: significantChanges },
      },
    });

    await this.notificationService.sendEmail({
      to: store.user.email,
      subject: `[${store.name}] Price Alert: Supplier prices changed`,
      template: 'price-alert',
      data: {
        storeName: store.name,
        userName: store.user.name,
        products: significantChanges,
        dashboardUrl: `${process.env.DASHBOARD_URL}/stores/${storeId}/products`,
      },
    });
  }

  async sendSyncFailureAlert(storeId: string, error: Error) {
    const store = await this.prisma.store.findUnique({
      where: { id: storeId },
      include: {
        user: { select: { email: true, name: true } },
      },
    });

    if (!store) return;

    await this.prisma.alert.create({
      data: {
        storeId,
        type: 'SYNC_FAILURE',
        severity: 'CRITICAL',
        title: 'Inventory sync failed',
        message: `The automated inventory sync encountered an error. Our team has been notified.`,
        metadata: {
          error: error.message,
          stack: error.stack,
        },
      },
    });

    // Notify internal team for critical failures
    await this.notificationService.sendInternalAlert({
      severity: 'CRITICAL',
      service: 'ops-engine',
      message: `Inventory sync failed for store ${storeId}`,
      error: error.message,
    });
  }
}
```

---

## 3. Fulfillment Orchestrator

### 3.1 State Machine Definition

```typescript
// services/ops-engine/src/modules/fulfillment/orchestrator/state.machine.ts
import { createMachine, assign } from 'xstate';

export interface OrderContext {
  orderId: string;
  storeId: string;
  customerId: string;
  items: Array<{
    productId: string;
    variantId?: string;
    quantity: number;
    sourceId: string;
    supplier: string;
  }>;
  supplierOrders: Array<{
    supplier: string;
    orderId: string;
    status: string;
    trackingNumber?: string;
    trackingUrl?: string;
  }>;
  shippingAddress: any;
  totalAmount: number;
  error?: string;
  retryCount: number;
}

type OrderEvent =
  | { type: 'PAYMENT_CONFIRMED' }
  | { type: 'SUPPLIER_ORDER_PLACED'; data: any }
  | { type: 'SUPPLIER_ORDER_FAILED'; error: string }
  | { type: 'TRACKING_RECEIVED'; data: any }
  | { type: 'ORDER_SHIPPED' }
  | { type: 'ORDER_DELIVERED' }
  | { type: 'REFUND_REQUESTED' }
  | { type: 'RETRY' };

export const orderFulfillmentMachine = createMachine<OrderContext, OrderEvent>(
  {
    id: 'orderFulfillment',
    initial: 'pending',
    context: {
      orderId: '',
      storeId: '',
      customerId: '',
      items: [],
      supplierOrders: [],
      shippingAddress: null,
      totalAmount: 0,
      retryCount: 0,
    },
    states: {
      pending: {
        on: {
          PAYMENT_CONFIRMED: 'placingSupplierOrders',
        },
      },

      placingSupplierOrders: {
        invoke: {
          id: 'placeSupplierOrders',
          src: 'placeSupplierOrders',
          onDone: {
            target: 'awaitingTracking',
            actions: assign({
              supplierOrders: (_, event) => event.data.supplierOrders,
            }),
          },
          onError: {
            target: 'supplierOrderFailed',
            actions: assign({
              error: (_, event) => event.data.message,
            }),
          },
        },
      },

      supplierOrderFailed: {
        entry: 'logFailure',
        always: [
          {
            target: 'placingSupplierOrders',
            cond: 'canRetry',
            actions: assign({
              retryCount: (ctx) => ctx.retryCount + 1,
            }),
          },
          { target: 'requiresManualIntervention' },
        ],
      },

      awaitingTracking: {
        invoke: {
          id: 'pollTracking',
          src: 'pollForTracking',
        },
        on: {
          TRACKING_RECEIVED: {
            target: 'shipped',
            actions: assign({
              supplierOrders: (ctx, event) =>
                ctx.supplierOrders.map((order) =>
                  order.supplier === event.data.supplier
                    ? {
                        ...order,
                        trackingNumber: event.data.trackingNumber,
                        trackingUrl: event.data.trackingUrl,
                        status: 'shipped',
                      }
                    : order,
                ),
            }),
          },
        },
        after: {
          // Timeout after 7 days
          604800000: 'requiresManualIntervention',
        },
      },

      shipped: {
        entry: ['notifyCustomerShipped', 'syncTrackingToStorefront'],
        invoke: {
          id: 'trackDelivery',
          src: 'trackDelivery',
        },
        on: {
          ORDER_DELIVERED: 'delivered',
        },
      },

      delivered: {
        entry: 'notifyCustomerDelivered',
        type: 'final',
      },

      requiresManualIntervention: {
        entry: 'notifyAdminIntervention',
        on: {
          RETRY: {
            target: 'placingSupplierOrders',
            actions: assign({
              retryCount: 0,
              error: undefined,
            }),
          },
        },
      },

      refundRequested: {
        // Handle refund flow
        type: 'final',
      },
    },
  },
  {
    guards: {
      canRetry: (ctx) => ctx.retryCount < 3,
    },
  },
);
```

### 3.2 Orchestrator Service

```typescript
// services/ops-engine/src/modules/fulfillment/orchestrator/orchestrator.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { interpret, State } from 'xstate';
import { PrismaService } from '@/shared/database/prisma.service';
import { SupplierFactory } from '../suppliers/supplier.factory';
import { TrackingService } from '../tracking/tracking.service';
import { EmailService } from '../notifications/email.service';
import { orderFulfillmentMachine, OrderContext } from './state.machine';

@Injectable()
export class FulfillmentOrchestratorService {
  private readonly logger = new Logger(FulfillmentOrchestratorService.name);
  private readonly activeOrders = new Map<string, any>();

  constructor(
    private readonly prisma: PrismaService,
    private readonly supplierFactory: SupplierFactory,
    private readonly trackingService: TrackingService,
    private readonly emailService: EmailService,
  ) {}

  async startFulfillment(orderId: string) {
    this.logger.log(`Starting fulfillment for order: ${orderId}`);

    // Load order data
    const order = await this.prisma.order.findUnique({
      where: { id: orderId },
      include: {
        items: {
          include: {
            product: true,
            variant: true,
          },
        },
        store: {
          include: {
            user: {
              include: {
                supplierAccounts: true,
              },
            },
          },
        },
        customer: true,
      },
    });

    if (!order) {
      throw new Error(`Order not found: ${orderId}`);
    }

    // Create machine context
    const context: OrderContext = {
      orderId: order.id,
      storeId: order.storeId,
      customerId: order.customerId,
      items: order.items.map((item) => ({
        productId: item.productId,
        variantId: item.variantId || undefined,
        quantity: item.quantity,
        sourceId: item.product.sourceId,
        supplier: item.product.sourceProvider,
      })),
      supplierOrders: [],
      shippingAddress: order.shippingAddress,
      totalAmount: Number(order.totalAmount),
      retryCount: 0,
    };

    // Create and start the state machine
    const service = interpret(
      orderFulfillmentMachine.withContext(context).withConfig({
        services: {
          placeSupplierOrders: async (ctx) =>
            this.placeSupplierOrders(ctx, order.store.user.supplierAccounts),
          pollForTracking: (ctx) => this.pollForTracking(ctx),
          trackDelivery: (ctx) => this.trackDelivery(ctx),
        },
        actions: {
          logFailure: (ctx) => this.logFailure(ctx),
          notifyCustomerShipped: (ctx) => this.notifyCustomerShipped(ctx, order),
          notifyCustomerDelivered: (ctx) =>
            this.notifyCustomerDelivered(ctx, order),
          notifyAdminIntervention: (ctx) =>
            this.notifyAdminIntervention(ctx, order),
          syncTrackingToStorefront: (ctx) =>
            this.syncTrackingToStorefront(ctx, order),
        },
      }),
    );

    // Subscribe to state changes
    service.onTransition((state) => {
      this.handleStateTransition(orderId, state);
    });

    // Start the machine
    service.start();
    this.activeOrders.set(orderId, service);

    // Send initial event
    service.send('PAYMENT_CONFIRMED');

    return { orderId, status: 'fulfillment_started' };
  }

  private async placeSupplierOrders(
    ctx: OrderContext,
    supplierAccounts: any[],
  ) {
    this.logger.log(`Placing supplier orders for order: ${ctx.orderId}`);

    const supplierOrders = [];

    // Group items by supplier
    const itemsBySupplier = ctx.items.reduce((acc, item) => {
      if (!acc[item.supplier]) acc[item.supplier] = [];
      acc[item.supplier].push(item);
      return acc;
    }, {} as Record<string, typeof ctx.items>);

    for (const [supplierName, items] of Object.entries(itemsBySupplier)) {
      const account = supplierAccounts.find(
        (acc) => acc.provider === supplierName,
      );

      if (!account) {
        throw new Error(`No supplier account for ${supplierName}`);
      }

      const supplier = this.supplierFactory.create(supplierName, account);

      // Place order with supplier
      const supplierOrder = await supplier.placeOrder({
        items: items.map((item) => ({
          sourceId: item.sourceId,
          quantity: item.quantity,
          variantId: item.variantId,
        })),
        shippingAddress: ctx.shippingAddress,
      });

      supplierOrders.push({
        supplier: supplierName,
        orderId: supplierOrder.orderId,
        status: 'placed',
      });

      // Record in database
      await this.prisma.supplierOrder.create({
        data: {
          orderId: ctx.orderId,
          supplier: supplierName,
          supplierOrderId: supplierOrder.orderId,
          status: 'PLACED',
          itemsJson: JSON.stringify(items),
        },
      });
    }

    return { supplierOrders };
  }

  private pollForTracking(ctx: OrderContext) {
    return (callback: any) => {
      const checkTracking = async () => {
        for (const supplierOrder of ctx.supplierOrders) {
          if (supplierOrder.trackingNumber) continue;

          const supplier = this.supplierFactory.create(
            supplierOrder.supplier,
            null, // Will use cached credentials
          );

          const tracking = await supplier.getOrderTracking(
            supplierOrder.orderId,
          );

          if (tracking?.trackingNumber) {
            callback({
              type: 'TRACKING_RECEIVED',
              data: {
                supplier: supplierOrder.supplier,
                trackingNumber: tracking.trackingNumber,
                trackingUrl: tracking.trackingUrl,
              },
            });
          }
        }
      };

      // Check every 6 hours
      const interval = setInterval(checkTracking, 6 * 60 * 60 * 1000);
      checkTracking(); // Initial check

      return () => clearInterval(interval);
    };
  }

  private trackDelivery(ctx: OrderContext) {
    return (callback: any) => {
      const checkDelivery = async () => {
        const allDelivered = await Promise.all(
          ctx.supplierOrders.map(async (order) => {
            if (!order.trackingNumber) return false;

            const status = await this.trackingService.getDeliveryStatus(
              order.trackingNumber,
            );

            return status === 'DELIVERED';
          }),
        );

        if (allDelivered.every(Boolean)) {
          callback({ type: 'ORDER_DELIVERED' });
        }
      };

      // Check daily
      const interval = setInterval(checkDelivery, 24 * 60 * 60 * 1000);
      checkDelivery();

      return () => clearInterval(interval);
    };
  }

  private async handleStateTransition(orderId: string, state: State<OrderContext>) {
    this.logger.log(`Order ${orderId} transitioned to: ${state.value}`);

    // Update order status in database
    const statusMap: Record<string, string> = {
      pending: 'PENDING',
      placingSupplierOrders: 'PROCESSING',
      awaitingTracking: 'AWAITING_SHIPMENT',
      shipped: 'SHIPPED',
      delivered: 'DELIVERED',
      requiresManualIntervention: 'INTERVENTION_REQUIRED',
    };

    const status = statusMap[state.value as string];
    if (status) {
      await this.prisma.order.update({
        where: { id: orderId },
        data: {
          fulfillmentStatus: status,
          updatedAt: new Date(),
        },
      });
    }

    // Persist state for recovery
    await this.prisma.orderState.upsert({
      where: { orderId },
      update: {
        state: JSON.stringify(state),
        updatedAt: new Date(),
      },
      create: {
        orderId,
        state: JSON.stringify(state),
      },
    });
  }

  private logFailure(ctx: OrderContext) {
    this.logger.error(`Fulfillment failed for order ${ctx.orderId}: ${ctx.error}`);
  }

  private async notifyCustomerShipped(ctx: OrderContext, order: any) {
    await this.emailService.send({
      to: order.customer.email,
      subject: `Your order has shipped! - ${order.store.name}`,
      template: 'order-shipped',
      data: {
        customerName: order.customer.name,
        storeName: order.store.name,
        orderId: ctx.orderId,
        trackingNumbers: ctx.supplierOrders.map((o) => ({
          number: o.trackingNumber,
          url: o.trackingUrl,
        })),
      },
    });
  }

  private async notifyCustomerDelivered(ctx: OrderContext, order: any) {
    await this.emailService.send({
      to: order.customer.email,
      subject: `Your order has been delivered! - ${order.store.name}`,
      template: 'order-delivered',
      data: {
        customerName: order.customer.name,
        storeName: order.store.name,
        orderId: ctx.orderId,
        reviewUrl: `${order.store.customDomain || `${order.store.subdomain}.zyenta.com`}/review/${ctx.orderId}`,
      },
    });
  }

  private async notifyAdminIntervention(ctx: OrderContext, order: any) {
    await this.emailService.sendInternal({
      subject: `[URGENT] Order requires manual intervention: ${ctx.orderId}`,
      template: 'admin-intervention',
      data: {
        orderId: ctx.orderId,
        storeName: order.store.name,
        error: ctx.error,
        retryCount: ctx.retryCount,
        dashboardUrl: `${process.env.ADMIN_DASHBOARD_URL}/orders/${ctx.orderId}`,
      },
    });
  }

  private async syncTrackingToStorefront(ctx: OrderContext, order: any) {
    // Update order with tracking info visible to customer
    await this.prisma.order.update({
      where: { id: ctx.orderId },
      data: {
        trackingInfo: ctx.supplierOrders.map((o) => ({
          trackingNumber: o.trackingNumber,
          trackingUrl: o.trackingUrl,
          carrier: this.trackingService.detectCarrier(o.trackingNumber || ''),
        })),
      },
    });
  }

  // Recovery: Resume interrupted fulfillment processes on startup
  async recoverPendingOrders() {
    const pendingStates = await this.prisma.orderState.findMany({
      where: {
        order: {
          fulfillmentStatus: {
            notIn: ['DELIVERED', 'CANCELLED', 'REFUNDED'],
          },
        },
      },
      include: { order: true },
    });

    for (const orderState of pendingStates) {
      this.logger.log(`Recovering order: ${orderState.orderId}`);
      // Restart the state machine from persisted state
      await this.startFulfillment(orderState.orderId);
    }
  }
}
```

### 3.3 Supplier Adapter (AliExpress Example)

```typescript
// services/ops-engine/src/modules/fulfillment/suppliers/aliexpress.adapter.ts
import { Injectable } from '@nestjs/common';
import { BaseSupplierAdapter, SupplierOrder, OrderInput, ProductData } from './base.adapter';
import axios, { AxiosInstance } from 'axios';
import * as crypto from 'crypto';

@Injectable()
export class AliExpressAdapter extends BaseSupplierAdapter {
  private client: AxiosInstance;
  private appKey: string;
  private appSecret: string;
  private accessToken: string;

  constructor(credentials: {
    apiKey: string;
    apiSecret: string;
    accessToken: string;
  }) {
    super();
    this.appKey = credentials.apiKey;
    this.appSecret = credentials.apiSecret;
    this.accessToken = credentials.accessToken;

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
      .createHmac('md5', this.appSecret)
      .update(signString)
      .digest('hex')
      .toUpperCase();
  }

  async getProduct(sourceId: string): Promise<ProductData> {
    const params = {
      app_key: this.appKey,
      access_token: this.accessToken,
      method: 'aliexpress.ds.product.get',
      sign_method: 'md5',
      timestamp: new Date().toISOString(),
      product_id: sourceId,
    };

    params['sign'] = this.generateSignature(params);

    const response = await this.client.post('', null, { params });
    const product = response.data.aliexpress_ds_product_get_response.result;

    return {
      sourceId,
      title: product.ae_item_base_info_dto.subject,
      price: parseFloat(product.ae_item_sku_info_dtos[0]?.sku_price || '0'),
      stock: product.ae_item_sku_info_dtos[0]?.sku_stock || 0,
      images: product.ae_multimedia_info_dto.image_urls.split(';'),
      variants: product.ae_item_sku_info_dtos.map((sku: any) => ({
        id: sku.sku_id,
        price: parseFloat(sku.sku_price),
        stock: sku.sku_stock,
        attributes: sku.sku_attr,
      })),
    };
  }

  async getProductsBatch(sourceIds: string[]): Promise<Map<string, ProductData>> {
    const results = new Map<string, ProductData>();

    // AliExpress API doesn't support batch, so we parallelize
    const batchSize = 10;
    for (let i = 0; i < sourceIds.length; i += batchSize) {
      const batch = sourceIds.slice(i, i + batchSize);
      const products = await Promise.all(
        batch.map((id) => this.getProduct(id).catch(() => null)),
      );

      products.forEach((product, index) => {
        if (product) {
          results.set(batch[index], product);
        }
      });

      // Rate limiting delay
      if (i + batchSize < sourceIds.length) {
        await new Promise((resolve) => setTimeout(resolve, 1000));
      }
    }

    return results;
  }

  async placeOrder(input: OrderInput): Promise<SupplierOrder> {
    const params = {
      app_key: this.appKey,
      access_token: this.accessToken,
      method: 'aliexpress.ds.order.create',
      sign_method: 'md5',
      timestamp: new Date().toISOString(),
      param_place_order_request4_open_api_d_t_o: JSON.stringify({
        product_items: input.items.map((item) => ({
          product_id: item.sourceId,
          sku_id: item.variantId,
          product_count: item.quantity,
        })),
        logistics_address: {
          address: input.shippingAddress.address1,
          address2: input.shippingAddress.address2,
          city: input.shippingAddress.city,
          country: input.shippingAddress.country,
          full_name: input.shippingAddress.name,
          phone_country: input.shippingAddress.phoneCountry,
          mobile_no: input.shippingAddress.phone,
          province: input.shippingAddress.state,
          zip: input.shippingAddress.postalCode,
        },
      }),
    };

    params['sign'] = this.generateSignature(params);

    const response = await this.client.post('', null, { params });
    const result = response.data.aliexpress_ds_order_create_response.result;

    if (!result.is_success) {
      throw new Error(`AliExpress order failed: ${result.error_msg}`);
    }

    return {
      orderId: result.order_list[0].order_id.toString(),
      status: 'placed',
      estimatedDelivery: this.calculateEstimatedDelivery(),
    };
  }

  async getOrderTracking(orderId: string): Promise<{
    trackingNumber?: string;
    trackingUrl?: string;
    status: string;
  }> {
    const params = {
      app_key: this.appKey,
      access_token: this.accessToken,
      method: 'aliexpress.ds.order.tracking.get',
      sign_method: 'md5',
      timestamp: new Date().toISOString(),
      order_id: orderId,
    };

    params['sign'] = this.generateSignature(params);

    const response = await this.client.post('', null, { params });
    const result = response.data.aliexpress_ds_order_tracking_get_response;

    if (result.result.tracking_number) {
      return {
        trackingNumber: result.result.tracking_number,
        trackingUrl: `https://global.cainiao.com/detail.htm?mailNoList=${result.result.tracking_number}`,
        status: 'shipped',
      };
    }

    return { status: 'awaiting_shipment' };
  }

  private calculateEstimatedDelivery(): Date {
    // AliExpress typically 15-45 days
    const deliveryDays = 30;
    const date = new Date();
    date.setDate(date.getDate() + deliveryDays);
    return date;
  }
}
```

---

## 4. Video Generation (Media Studio Enhancement)

### 4.1 Video Generator Service

```python
# services/media-studio/app/processors/video/generator.py
from dataclasses import dataclass
from typing import List, Optional
import asyncio
import aiohttp
from enum import Enum
import json


class VideoStyle(str, Enum):
    PRODUCT_SHOWCASE = "product_showcase"
    LIFESTYLE = "lifestyle"
    SOCIAL_MEDIA = "social_media"
    AD_CREATIVE = "ad_creative"
    UNBOXING = "unboxing"


@dataclass
class VideoConfig:
    style: VideoStyle
    duration: int = 15  # seconds
    aspect_ratio: str = "9:16"  # vertical for social
    music: Optional[str] = None
    voice_over: Optional[str] = None
    transitions: str = "smooth"
    text_overlays: List[dict] = None


@dataclass
class VideoGenerationInput:
    product_images: List[str]
    product_title: str
    product_description: str
    brand_colors: dict
    config: VideoConfig


@dataclass
class GeneratedVideo:
    video_url: str
    thumbnail_url: str
    duration: int
    resolution: str
    format: str
    metadata: dict


class VideoGeneratorService:
    """
    Video generation service using:
    - Runway ML for AI video generation
    - FFmpeg for video processing
    - ElevenLabs for voice-over
    """

    def __init__(self):
        self.runway_api_key = os.environ.get("RUNWAY_API_KEY")
        self.elevenlabs_api_key = os.environ.get("ELEVENLABS_API_KEY")
        self.session: Optional[aiohttp.ClientSession] = None

    async def get_session(self) -> aiohttp.ClientSession:
        if self.session is None or self.session.closed:
            self.session = aiohttp.ClientSession()
        return self.session

    async def generate_video(
        self,
        input_data: VideoGenerationInput,
    ) -> GeneratedVideo:
        """Main video generation pipeline"""

        # Step 1: Prepare assets
        prepared_images = await self._prepare_images(input_data.product_images)

        # Step 2: Generate video scenes using Runway
        scenes = await self._generate_scenes(
            prepared_images,
            input_data.config.style,
            input_data.config.duration,
        )

        # Step 3: Generate voice-over if requested
        audio_url = None
        if input_data.config.voice_over:
            audio_url = await self._generate_voice_over(
                input_data.config.voice_over,
                input_data.config.duration,
            )

        # Step 4: Compose final video
        final_video = await self._compose_video(
            scenes=scenes,
            audio_url=audio_url,
            config=input_data.config,
            brand_colors=input_data.brand_colors,
            text_overlays=input_data.config.text_overlays,
        )

        # Step 5: Generate thumbnail
        thumbnail_url = await self._generate_thumbnail(final_video)

        return GeneratedVideo(
            video_url=final_video.url,
            thumbnail_url=thumbnail_url,
            duration=input_data.config.duration,
            resolution="1080x1920" if input_data.config.aspect_ratio == "9:16" else "1920x1080",
            format="mp4",
            metadata={
                "style": input_data.config.style,
                "has_voice_over": audio_url is not None,
            },
        )

    async def _generate_scenes(
        self,
        images: List[str],
        style: VideoStyle,
        total_duration: int,
    ) -> List[dict]:
        """Generate video scenes using Runway ML Gen-3"""

        session = await self.get_session()
        scenes = []
        scene_duration = total_duration // len(images)

        for i, image_url in enumerate(images):
            # Create motion prompt based on style
            motion_prompt = self._get_motion_prompt(style, i, len(images))

            # Call Runway API
            async with session.post(
                "https://api.runwayml.com/v1/generations",
                headers={
                    "Authorization": f"Bearer {self.runway_api_key}",
                    "Content-Type": "application/json",
                },
                json={
                    "model": "gen3",
                    "image_url": image_url,
                    "prompt": motion_prompt,
                    "duration": scene_duration,
                    "aspect_ratio": "9:16",
                },
            ) as response:
                result = await response.json()

                if response.status != 200:
                    raise Exception(f"Runway API error: {result}")

                scenes.append({
                    "video_url": result["output_url"],
                    "duration": scene_duration,
                    "order": i,
                })

        return scenes

    def _get_motion_prompt(
        self,
        style: VideoStyle,
        scene_index: int,
        total_scenes: int,
    ) -> str:
        """Generate appropriate motion prompt for each style"""

        prompts = {
            VideoStyle.PRODUCT_SHOWCASE: [
                "Slow smooth rotation revealing product details",
                "Gentle zoom in highlighting features",
                "Elegant pan across product surface",
            ],
            VideoStyle.LIFESTYLE: [
                "Product in natural setting with soft lighting",
                "Lifestyle scene with subtle movement",
                "Ambient environment with product focus",
            ],
            VideoStyle.SOCIAL_MEDIA: [
                "Dynamic zoom with trendy movements",
                "Quick snap focus transitions",
                "Energetic product reveal",
            ],
            VideoStyle.AD_CREATIVE: [
                "Professional cinematic movement",
                "Dramatic lighting shift",
                "Premium product presentation",
            ],
        }

        style_prompts = prompts.get(style, prompts[VideoStyle.PRODUCT_SHOWCASE])
        return style_prompts[scene_index % len(style_prompts)]

    async def _generate_voice_over(
        self,
        script: str,
        duration: int,
    ) -> str:
        """Generate voice-over using ElevenLabs"""

        session = await self.get_session()

        async with session.post(
            "https://api.elevenlabs.io/v1/text-to-speech/21m00Tcm4TlvDq8ikWAM",
            headers={
                "xi-api-key": self.elevenlabs_api_key,
                "Content-Type": "application/json",
            },
            json={
                "text": script,
                "model_id": "eleven_multilingual_v2",
                "voice_settings": {
                    "stability": 0.5,
                    "similarity_boost": 0.75,
                },
            },
        ) as response:
            if response.status != 200:
                raise Exception("ElevenLabs API error")

            audio_data = await response.read()

            # Upload to S3 and return URL
            return await self._upload_audio(audio_data)

    async def _compose_video(
        self,
        scenes: List[dict],
        audio_url: Optional[str],
        config: VideoConfig,
        brand_colors: dict,
        text_overlays: List[dict],
    ) -> dict:
        """Compose final video using FFmpeg"""

        import subprocess
        import tempfile
        import os

        # Download all scene videos
        scene_files = []
        for scene in scenes:
            scene_path = await self._download_file(scene["video_url"])
            scene_files.append(scene_path)

        # Create FFmpeg filter complex
        filter_complex = self._build_filter_complex(
            len(scenes),
            config,
            brand_colors,
            text_overlays,
        )

        # Create concat file
        with tempfile.NamedTemporaryFile(mode='w', suffix='.txt', delete=False) as f:
            for scene_file in scene_files:
                f.write(f"file '{scene_file}'\n")
            concat_file = f.name

        output_path = tempfile.mktemp(suffix='.mp4')

        # Build FFmpeg command
        cmd = [
            'ffmpeg', '-y',
            '-f', 'concat', '-safe', '0', '-i', concat_file,
        ]

        if audio_url:
            audio_path = await self._download_file(audio_url)
            cmd.extend(['-i', audio_path])
            cmd.extend(['-map', '0:v', '-map', '1:a'])

        cmd.extend([
            '-filter_complex', filter_complex,
            '-c:v', 'libx264', '-preset', 'medium',
            '-c:a', 'aac', '-b:a', '128k',
            output_path,
        ])

        subprocess.run(cmd, check=True, capture_output=True)

        # Upload final video
        video_url = await self._upload_video(output_path)

        # Cleanup
        os.unlink(concat_file)
        for f in scene_files:
            os.unlink(f)
        os.unlink(output_path)

        return {"url": video_url}

    def _build_filter_complex(
        self,
        num_scenes: int,
        config: VideoConfig,
        brand_colors: dict,
        text_overlays: List[dict],
    ) -> str:
        """Build FFmpeg filter complex for transitions and overlays"""

        filters = []

        # Add transition between scenes
        if config.transitions == "smooth":
            filters.append("xfade=transition=fade:duration=0.5")
        elif config.transitions == "zoom":
            filters.append("xfade=transition=zoomin:duration=0.5")

        # Add text overlays
        if text_overlays:
            for overlay in text_overlays:
                filters.append(
                    f"drawtext=text='{overlay['text']}':"
                    f"fontcolor={brand_colors.get('primary', 'white')}:"
                    f"fontsize=48:x=(w-text_w)/2:y=h-100:"
                    f"enable='between(t,{overlay.get('start', 0)},{overlay.get('end', 5)})'"
                )

        return ",".join(filters) if filters else "null"

    async def _generate_thumbnail(self, video: dict) -> str:
        """Extract best frame as thumbnail"""

        import subprocess
        import tempfile

        video_path = await self._download_file(video["url"])
        output_path = tempfile.mktemp(suffix='.jpg')

        # Extract frame at 2 seconds
        subprocess.run([
            'ffmpeg', '-y',
            '-i', video_path,
            '-ss', '2',
            '-vframes', '1',
            '-q:v', '2',
            output_path,
        ], check=True, capture_output=True)

        thumbnail_url = await self._upload_image(output_path)

        os.unlink(video_path)
        os.unlink(output_path)

        return thumbnail_url

    async def _download_file(self, url: str) -> str:
        """Download file to temporary location"""
        import tempfile

        session = await self.get_session()
        async with session.get(url) as response:
            data = await response.read()
            suffix = '.mp4' if 'video' in response.content_type else '.wav'
            with tempfile.NamedTemporaryFile(suffix=suffix, delete=False) as f:
                f.write(data)
                return f.name

    async def _upload_video(self, path: str) -> str:
        """Upload video to S3"""
        from app.services.storage import StorageService
        storage = StorageService()
        return await storage.upload_file(path, 'videos/')

    async def _upload_audio(self, data: bytes) -> str:
        """Upload audio to S3"""
        from app.services.storage import StorageService
        storage = StorageService()
        return await storage.upload_bytes(data, 'audio/', 'mp3')

    async def _upload_image(self, path: str) -> str:
        """Upload image to S3"""
        from app.services.storage import StorageService
        storage = StorageService()
        return await storage.upload_file(path, 'thumbnails/')

    async def _prepare_images(self, image_urls: List[str]) -> List[str]:
        """Prepare images for video generation (enhance, resize)"""
        from app.optimization.images import ImageOptimizer

        optimizer = ImageOptimizer()
        prepared = []

        for url in image_urls:
            image_data = await optimizer.download_image(url)
            optimized = optimizer.optimize(
                image_data,
                preset="large",
                output_format="jpeg",
            )
            upload_url = await self._upload_image_bytes(optimized.data)
            prepared.append(upload_url)

        return prepared

    async def _upload_image_bytes(self, data: bytes) -> str:
        """Upload image bytes to S3"""
        from app.services.storage import StorageService
        storage = StorageService()
        return await storage.upload_bytes(data, 'video-assets/', 'jpg')
```

### 4.2 Video Generation API

```python
# services/media-studio/app/api/v1/videos.py
from fastapi import APIRouter, HTTPException, BackgroundTasks
from pydantic import BaseModel
from typing import List, Optional
from app.processors.video.generator import (
    VideoGeneratorService,
    VideoGenerationInput,
    VideoConfig,
    VideoStyle,
)
from app.services.queue import QueueService

router = APIRouter(prefix="/videos", tags=["videos"])

video_generator = VideoGeneratorService()
queue_service = QueueService()


class VideoGenerationRequest(BaseModel):
    product_images: List[str]
    product_title: str
    product_description: str
    brand_colors: dict
    style: VideoStyle = VideoStyle.PRODUCT_SHOWCASE
    duration: int = 15
    aspect_ratio: str = "9:16"
    voice_over_script: Optional[str] = None
    text_overlays: Optional[List[dict]] = None


class VideoGenerationResponse(BaseModel):
    job_id: str
    status: str
    estimated_time: int  # seconds


class VideoJobStatus(BaseModel):
    job_id: str
    status: str
    progress: int
    video_url: Optional[str] = None
    thumbnail_url: Optional[str] = None
    error: Optional[str] = None


@router.post("/generate", response_model=VideoGenerationResponse)
async def generate_video(
    request: VideoGenerationRequest,
    background_tasks: BackgroundTasks,
):
    """
    Start video generation job.
    Videos are generated asynchronously due to processing time.
    """

    # Create job
    job_id = await queue_service.create_job(
        "video-generation",
        {
            "product_images": request.product_images,
            "product_title": request.product_title,
            "product_description": request.product_description,
            "brand_colors": request.brand_colors,
            "config": {
                "style": request.style,
                "duration": request.duration,
                "aspect_ratio": request.aspect_ratio,
                "voice_over": request.voice_over_script,
                "text_overlays": request.text_overlays,
            },
        },
    )

    # Estimate time based on duration
    estimated_time = request.duration * 10  # ~10s processing per second of video

    return VideoGenerationResponse(
        job_id=job_id,
        status="queued",
        estimated_time=estimated_time,
    )


@router.get("/jobs/{job_id}", response_model=VideoJobStatus)
async def get_video_job_status(job_id: str):
    """Get status of video generation job"""

    job = await queue_service.get_job("video-generation", job_id)

    if not job:
        raise HTTPException(status_code=404, detail="Job not found")

    return VideoJobStatus(
        job_id=job_id,
        status=job.status,
        progress=job.progress,
        video_url=job.result.get("video_url") if job.result else None,
        thumbnail_url=job.result.get("thumbnail_url") if job.result else None,
        error=job.error,
    )


@router.post("/batch")
async def generate_batch_videos(
    products: List[dict],
    style: VideoStyle = VideoStyle.PRODUCT_SHOWCASE,
):
    """Generate videos for multiple products"""

    job_ids = []

    for product in products:
        job_id = await queue_service.create_job(
            "video-generation",
            {
                "product_images": product["images"],
                "product_title": product["title"],
                "product_description": product["description"],
                "brand_colors": product.get("brand_colors", {}),
                "config": {
                    "style": style,
                    "duration": 15,
                    "aspect_ratio": "9:16",
                },
            },
        )
        job_ids.append(job_id)

    return {
        "job_ids": job_ids,
        "total": len(job_ids),
        "status": "queued",
    }
```

---

## 5. Database Schema Extensions

### 5.1 New Prisma Models

```prisma
// packages/database/prisma/schema.prisma (additions)

// ==================== OPERATIONS ====================

model SyncLog {
  id                String    @id @default(cuid())
  storeId           String
  store             Store     @relation(fields: [storeId], references: [id], onDelete: Cascade)

  type              SyncType
  status            SyncStatus
  productsProcessed Int       @default(0)
  productsChanged   Int       @default(0)
  duration          Int       // milliseconds
  metadata          Json?
  error             String?

  createdAt         DateTime  @default(now())

  @@index([storeId])
  @@index([createdAt])
}

enum SyncType {
  INVENTORY
  PRICE
  FULL
}

enum SyncStatus {
  STARTED
  COMPLETED
  FAILED
}

model Alert {
  id          String      @id @default(cuid())
  storeId     String
  store       Store       @relation(fields: [storeId], references: [id], onDelete: Cascade)

  type        AlertType
  severity    AlertSeverity
  title       String
  message     String
  metadata    Json?
  read        Boolean     @default(false)
  readAt      DateTime?

  createdAt   DateTime    @default(now())

  @@index([storeId])
  @@index([type])
}

enum AlertType {
  STOCK_OUT
  STOCK_LOW
  PRICE_CHANGE
  SYNC_FAILURE
  ORDER_ISSUE
  PAYMENT_ISSUE
}

enum AlertSeverity {
  LOW
  MEDIUM
  HIGH
  CRITICAL
}

// ==================== FULFILLMENT ====================

model Order {
  id                String        @id @default(cuid())
  storeId           String
  store             Store         @relation(fields: [storeId], references: [id])
  customerId        String
  customer          Customer      @relation(fields: [customerId], references: [id])

  // Order details
  orderNumber       String        @unique
  status            OrderStatus   @default(PENDING)
  fulfillmentStatus FulfillmentStatus @default(PENDING)

  // Amounts
  subtotal          Decimal       @db.Decimal(10, 2)
  shippingAmount    Decimal       @db.Decimal(10, 2) @default(0)
  taxAmount         Decimal       @db.Decimal(10, 2) @default(0)
  totalAmount       Decimal       @db.Decimal(10, 2)
  currency          String        @default("USD")

  // Addresses
  shippingAddress   Json
  billingAddress    Json?

  // Payment
  stripePaymentId   String?
  paidAt            DateTime?

  // Tracking
  trackingInfo      Json?         // Array of tracking numbers/urls

  // Relations
  items             OrderItem[]
  supplierOrders    SupplierOrder[]
  state             OrderState?

  createdAt         DateTime      @default(now())
  updatedAt         DateTime      @updatedAt

  @@index([storeId])
  @@index([customerId])
  @@index([status])
}

enum OrderStatus {
  PENDING
  PAID
  PROCESSING
  SHIPPED
  DELIVERED
  CANCELLED
  REFUNDED
}

enum FulfillmentStatus {
  PENDING
  PROCESSING
  AWAITING_SHIPMENT
  SHIPPED
  DELIVERED
  INTERVENTION_REQUIRED
  CANCELLED
}

model OrderItem {
  id          String    @id @default(cuid())
  orderId     String
  order       Order     @relation(fields: [orderId], references: [id], onDelete: Cascade)
  productId   String
  product     Product   @relation(fields: [productId], references: [id])
  variantId   String?
  variant     ProductVariant? @relation(fields: [variantId], references: [id])

  quantity    Int
  unitPrice   Decimal   @db.Decimal(10, 2)
  totalPrice  Decimal   @db.Decimal(10, 2)

  createdAt   DateTime  @default(now())
}

model SupplierOrder {
  id              String    @id @default(cuid())
  orderId         String
  order           Order     @relation(fields: [orderId], references: [id], onDelete: Cascade)

  supplier        String
  supplierOrderId String
  status          String
  itemsJson       String    // JSON of items in this supplier order
  trackingNumber  String?
  trackingUrl     String?

  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  @@index([orderId])
  @@index([supplierOrderId])
}

model OrderState {
  id          String    @id @default(cuid())
  orderId     String    @unique
  order       Order     @relation(fields: [orderId], references: [id], onDelete: Cascade)

  state       String    // Serialized XState state
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
}

model Customer {
  id          String    @id @default(cuid())
  storeId     String
  store       Store     @relation(fields: [storeId], references: [id], onDelete: Cascade)

  email       String
  name        String
  phone       String?

  orders      Order[]

  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt

  @@unique([storeId, email])
  @@index([storeId])
}

// ==================== VIDEO GENERATION ====================

model VideoAsset {
  id          String      @id @default(cuid())
  storeId     String
  store       Store       @relation(fields: [storeId], references: [id], onDelete: Cascade)
  productId   String?
  product     Product?    @relation(fields: [productId], references: [id])

  type        VideoType
  style       String      // VideoStyle enum value
  url         String
  thumbnailUrl String
  duration    Int         // seconds
  resolution  String
  format      String

  metadata    Json?

  createdAt   DateTime    @default(now())

  @@index([storeId])
  @@index([productId])
}

enum VideoType {
  PRODUCT_VIDEO
  AD_CREATIVE
  SOCIAL_CONTENT
  LIFESTYLE
}
```

---

## 6. Implementation Order

### Sprint 1: Inventory Sync Foundation (Weeks 1-2)
1. Setup NestJS Ops Engine project structure
2. Implement sync scheduler (cron jobs)
3. Build sync processor with batch handling
4. Implement supplier adapters for inventory fetching
5. Setup BullMQ for job processing

### Sprint 2: Price Management & Alerts (Weeks 3-4)
1. Implement price calculator with strategies
2. Build alert system and notification service
3. Create email templates for alerts
4. Implement dashboard alert UI
5. Add sync logging and monitoring

### Sprint 3: Fulfillment Orchestrator (Weeks 5-6)
1. Implement XState-based workflow engine
2. Build supplier order placement adapters
3. Implement tracking integration
4. Build customer notification system
5. Create fulfillment dashboard UI

### Sprint 4: Video Generation (Weeks 7-8)
1. Integrate Runway ML API
2. Implement FFmpeg video composition
3. Add ElevenLabs voice-over
4. Build video generation queue
5. Create video management UI in dashboard

### Sprint 5: Integration & Testing (Weeks 9-10)
1. End-to-end fulfillment testing
2. Inventory sync stress testing
3. Video generation quality assurance
4. Performance optimization
5. Documentation and training

---

## 7. Success Metrics

### Technical Metrics
- [ ] Inventory sync completes in < 5 minutes per store
- [ ] Price updates reflected within 15 minutes
- [ ] Fulfillment order placed within 5 minutes of payment
- [ ] Tracking sync updates every 6 hours
- [ ] Video generation completes in < 5 minutes

### Business Metrics
- [ ] Zero manual fulfillment intervention (< 1% exception rate)
- [ ] Customer shipping notification within 24 hours
- [ ] Stock-out alerts prevent 95%+ oversells
- [ ] Video generation increases product page conversion by 20%

---

## 8. Risk Mitigation

| Risk | Impact | Mitigation |
|------|--------|------------|
| Supplier API rate limits | High | Request pooling, caching, multiple account rotation |
| Fulfillment delays | High | Clear SLA communication, proactive notifications |
| Video generation costs | Medium | Usage limits, efficient batch processing |
| State machine complexity | Medium | Comprehensive testing, state visualization tools |
| Tracking integration failures | Medium | Multiple carrier fallbacks, manual entry option |

---

*Phase 2 establishes the autonomous operations foundation that enables truly "zero-effort" e-commerce by handling inventory, pricing, fulfillment, and video generation automatically.*
