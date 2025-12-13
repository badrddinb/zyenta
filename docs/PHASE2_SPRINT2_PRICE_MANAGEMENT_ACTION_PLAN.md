# Phase 2 Sprint 2: Price Management & Alerts - Action Plan

## Overview

This document provides a comprehensive implementation guide for Phase 2 Sprint 2, focusing on building the price management system with dynamic pricing strategies, alert notifications, and monitoring capabilities.

### Sprint Objectives
- [ ] Implement price calculator with multiple pricing strategies
- [ ] Build alert system for stock and price changes
- [ ] Create notification service (email, dashboard, webhooks)
- [ ] Implement email templates for alerts
- [ ] Build dashboard alert UI components
- [ ] Add sync logging and monitoring

### Prerequisites
- Phase 2 Sprint 1 completed (Ops Engine with Inventory Sync)
- BullMQ job processing operational
- Redis instance configured
- SMTP service configured for email notifications

---

## 1. Price Calculator Module

### 1.1 Module Definition

```typescript
// services/ops-engine/src/modules/pricing/pricing.module.ts
import { Module } from '@nestjs/common';
import { PricingService } from './pricing.service';
import { PricingController } from './pricing.controller';
import { StrategyFactory } from './strategies/strategy.factory';
import { FixedMarkupStrategy } from './strategies/fixed-markup.strategy';
import { PercentageMarkupStrategy } from './strategies/percentage-markup.strategy';
import { CompetitorBasedStrategy } from './strategies/competitor-based.strategy';
import { DynamicPricingStrategy } from './strategies/dynamic-pricing.strategy';

@Module({
  controllers: [PricingController],
  providers: [
    PricingService,
    StrategyFactory,
    FixedMarkupStrategy,
    PercentageMarkupStrategy,
    CompetitorBasedStrategy,
    DynamicPricingStrategy,
  ],
  exports: [PricingService],
})
export class PricingModule {}
```

### 1.2 Pricing Strategy Interface

```typescript
// services/ops-engine/src/modules/pricing/strategies/base.strategy.ts
export interface PricingContext {
  costPrice: number;
  currentSellingPrice: number;
  previousCostPrice?: number;
  competitorPrices?: number[];
  demandScore?: number;  // 0-100
  stockLevel?: number;
  category?: string;
  storeSettings: PricingSettings;
}

export interface PricingResult {
  suggestedPrice: number;
  minimumPrice: number;
  maximumPrice: number;
  margin: number;
  marginPercentage: number;
  strategy: string;
  reasoning: string;
}

export interface PricingSettings {
  strategy: 'FIXED_MARKUP' | 'PERCENTAGE_MARKUP' | 'COMPETITOR_BASED' | 'DYNAMIC';
  fixedMarkup?: number;
  percentageMarkup?: number;
  minimumMarginPercentage: number;
  maximumMarginPercentage: number;
  roundToNearest?: number;  // e.g., 0.99, 0.95
  competitorPriceAdjustment?: number;  // percentage below/above competitor
}

export abstract class BasePricingStrategy {
  abstract name: string;
  abstract calculate(context: PricingContext): PricingResult;

  protected roundPrice(price: number, roundTo?: number): number {
    if (!roundTo) return Math.round(price * 100) / 100;

    const base = Math.floor(price);
    return base + roundTo;
  }

  protected calculateMargin(sellingPrice: number, costPrice: number): number {
    return sellingPrice - costPrice;
  }

  protected calculateMarginPercentage(sellingPrice: number, costPrice: number): number {
    if (costPrice === 0) return 100;
    return ((sellingPrice - costPrice) / costPrice) * 100;
  }

  protected enforceMarginLimits(
    price: number,
    costPrice: number,
    minMarginPct: number,
    maxMarginPct: number,
  ): number {
    const minPrice = costPrice * (1 + minMarginPct / 100);
    const maxPrice = costPrice * (1 + maxMarginPct / 100);

    return Math.min(Math.max(price, minPrice), maxPrice);
  }
}
```

### 1.3 Fixed Markup Strategy

```typescript
// services/ops-engine/src/modules/pricing/strategies/fixed-markup.strategy.ts
import { Injectable } from '@nestjs/common';
import { BasePricingStrategy, PricingContext, PricingResult } from './base.strategy';

@Injectable()
export class FixedMarkupStrategy extends BasePricingStrategy {
  name = 'FIXED_MARKUP';

  calculate(context: PricingContext): PricingResult {
    const { costPrice, storeSettings } = context;
    const markup = storeSettings.fixedMarkup || 10;

    let suggestedPrice = costPrice + markup;

    suggestedPrice = this.enforceMarginLimits(
      suggestedPrice,
      costPrice,
      storeSettings.minimumMarginPercentage,
      storeSettings.maximumMarginPercentage,
    );

    suggestedPrice = this.roundPrice(suggestedPrice, storeSettings.roundToNearest);

    const margin = this.calculateMargin(suggestedPrice, costPrice);
    const marginPercentage = this.calculateMarginPercentage(suggestedPrice, costPrice);

    return {
      suggestedPrice,
      minimumPrice: costPrice * (1 + storeSettings.minimumMarginPercentage / 100),
      maximumPrice: costPrice * (1 + storeSettings.maximumMarginPercentage / 100),
      margin,
      marginPercentage,
      strategy: this.name,
      reasoning: `Fixed markup of $${markup} applied to cost price of $${costPrice}`,
    };
  }
}
```

### 1.4 Percentage Markup Strategy

```typescript
// services/ops-engine/src/modules/pricing/strategies/percentage-markup.strategy.ts
import { Injectable } from '@nestjs/common';
import { BasePricingStrategy, PricingContext, PricingResult } from './base.strategy';

@Injectable()
export class PercentageMarkupStrategy extends BasePricingStrategy {
  name = 'PERCENTAGE_MARKUP';

  calculate(context: PricingContext): PricingResult {
    const { costPrice, storeSettings } = context;
    const markupPercentage = storeSettings.percentageMarkup || 50;

    let suggestedPrice = costPrice * (1 + markupPercentage / 100);

    suggestedPrice = this.enforceMarginLimits(
      suggestedPrice,
      costPrice,
      storeSettings.minimumMarginPercentage,
      storeSettings.maximumMarginPercentage,
    );

    suggestedPrice = this.roundPrice(suggestedPrice, storeSettings.roundToNearest);

    const margin = this.calculateMargin(suggestedPrice, costPrice);
    const marginPercentage = this.calculateMarginPercentage(suggestedPrice, costPrice);

    return {
      suggestedPrice,
      minimumPrice: costPrice * (1 + storeSettings.minimumMarginPercentage / 100),
      maximumPrice: costPrice * (1 + storeSettings.maximumMarginPercentage / 100),
      margin,
      marginPercentage,
      strategy: this.name,
      reasoning: `${markupPercentage}% markup applied to cost price of $${costPrice}`,
    };
  }
}
```

### 1.5 Dynamic Pricing Strategy

```typescript
// services/ops-engine/src/modules/pricing/strategies/dynamic-pricing.strategy.ts
import { Injectable } from '@nestjs/common';
import { BasePricingStrategy, PricingContext, PricingResult } from './base.strategy';

@Injectable()
export class DynamicPricingStrategy extends BasePricingStrategy {
  name = 'DYNAMIC';

  calculate(context: PricingContext): PricingResult {
    const {
      costPrice,
      demandScore = 50,
      stockLevel = 100,
      competitorPrices = [],
      storeSettings,
    } = context;

    // Base markup starts at minimum margin
    let baseMarkupPct = storeSettings.minimumMarginPercentage;

    // Demand adjustment: high demand = higher price
    // demandScore 0-100, we add 0-30% based on demand
    const demandAdjustment = (demandScore / 100) * 30;

    // Stock adjustment: low stock = higher price
    // stockLevel 0-100, we add 0-20% for low stock
    const stockAdjustment = ((100 - Math.min(stockLevel, 100)) / 100) * 20;

    // Competitor adjustment
    let competitorAdjustment = 0;
    if (competitorPrices.length > 0) {
      const avgCompetitorPrice = competitorPrices.reduce((a, b) => a + b, 0) / competitorPrices.length;
      const targetPrice = avgCompetitorPrice * (1 + (storeSettings.competitorPriceAdjustment || 0) / 100);
      const targetMarkup = ((targetPrice - costPrice) / costPrice) * 100;
      competitorAdjustment = Math.max(0, targetMarkup - baseMarkupPct);
    }

    // Calculate final markup
    const totalMarkupPct = baseMarkupPct + demandAdjustment + stockAdjustment + competitorAdjustment;

    let suggestedPrice = costPrice * (1 + totalMarkupPct / 100);

    suggestedPrice = this.enforceMarginLimits(
      suggestedPrice,
      costPrice,
      storeSettings.minimumMarginPercentage,
      storeSettings.maximumMarginPercentage,
    );

    suggestedPrice = this.roundPrice(suggestedPrice, storeSettings.roundToNearest);

    const margin = this.calculateMargin(suggestedPrice, costPrice);
    const marginPercentage = this.calculateMarginPercentage(suggestedPrice, costPrice);

    return {
      suggestedPrice,
      minimumPrice: costPrice * (1 + storeSettings.minimumMarginPercentage / 100),
      maximumPrice: costPrice * (1 + storeSettings.maximumMarginPercentage / 100),
      margin,
      marginPercentage,
      strategy: this.name,
      reasoning: `Dynamic pricing: base ${baseMarkupPct.toFixed(1)}% + demand ${demandAdjustment.toFixed(1)}% + stock ${stockAdjustment.toFixed(1)}% + competitor ${competitorAdjustment.toFixed(1)}%`,
    };
  }
}
```

### 1.6 Strategy Factory

```typescript
// services/ops-engine/src/modules/pricing/strategies/strategy.factory.ts
import { Injectable } from '@nestjs/common';
import { BasePricingStrategy, PricingSettings } from './base.strategy';
import { FixedMarkupStrategy } from './fixed-markup.strategy';
import { PercentageMarkupStrategy } from './percentage-markup.strategy';
import { CompetitorBasedStrategy } from './competitor-based.strategy';
import { DynamicPricingStrategy } from './dynamic-pricing.strategy';

@Injectable()
export class StrategyFactory {
  constructor(
    private readonly fixedMarkup: FixedMarkupStrategy,
    private readonly percentageMarkup: PercentageMarkupStrategy,
    private readonly competitorBased: CompetitorBasedStrategy,
    private readonly dynamic: DynamicPricingStrategy,
  ) {}

  getStrategy(settings: PricingSettings): BasePricingStrategy {
    switch (settings.strategy) {
      case 'FIXED_MARKUP':
        return this.fixedMarkup;
      case 'PERCENTAGE_MARKUP':
        return this.percentageMarkup;
      case 'COMPETITOR_BASED':
        return this.competitorBased;
      case 'DYNAMIC':
        return this.dynamic;
      default:
        return this.percentageMarkup;
    }
  }
}
```

### 1.7 Pricing Service

```typescript
// services/ops-engine/src/modules/pricing/pricing.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { PrismaService } from '@/shared/database/prisma.service';
import { RedisService } from '@/shared/cache/redis.service';
import { StrategyFactory } from './strategies/strategy.factory';
import { PricingContext, PricingResult, PricingSettings } from './strategies/base.strategy';

@Injectable()
export class PricingService {
  private readonly logger = new Logger(PricingService.name);

  constructor(
    private readonly prisma: PrismaService,
    private readonly redis: RedisService,
    private readonly strategyFactory: StrategyFactory,
  ) {}

  async calculatePrice(
    productId: string,
    newCostPrice?: number,
  ): Promise<PricingResult> {
    const product = await this.prisma.product.findUnique({
      where: { id: productId },
      include: {
        store: true,
      },
    });

    if (!product) {
      throw new Error(`Product not found: ${productId}`);
    }

    const storeSettings = this.getStoreSettings(product.store);
    const strategy = this.strategyFactory.getStrategy(storeSettings);

    const context: PricingContext = {
      costPrice: newCostPrice || Number(product.costPrice),
      currentSellingPrice: Number(product.sellingPrice),
      previousCostPrice: Number(product.costPrice),
      storeSettings,
    };

    return strategy.calculate(context);
  }

  async updateProductPrice(
    productId: string,
    options: {
      autoApply?: boolean;
      newCostPrice?: number;
    } = {},
  ): Promise<{
    product: any;
    pricing: PricingResult;
    applied: boolean;
  }> {
    const pricing = await this.calculatePrice(productId, options.newCostPrice);

    let applied = false;
    let product = await this.prisma.product.findUnique({
      where: { id: productId },
    });

    if (options.autoApply) {
      product = await this.prisma.product.update({
        where: { id: productId },
        data: {
          sellingPrice: pricing.suggestedPrice,
          costPrice: options.newCostPrice || product.costPrice,
          updatedAt: new Date(),
        },
      });

      // Invalidate cache
      await this.redis.del(`product:${productId}`);
      await this.redis.invalidatePattern(`store:${product.storeId}:products*`);

      applied = true;

      this.logger.log(
        `Updated product ${productId} price: $${pricing.suggestedPrice}`,
      );
    }

    return { product, pricing, applied };
  }

  async bulkCalculatePrices(
    storeId: string,
    options: {
      onlyOutdated?: boolean;
      limit?: number;
    } = {},
  ): Promise<Array<{ productId: string; pricing: PricingResult }>> {
    const products = await this.prisma.product.findMany({
      where: {
        storeId,
        status: { in: ['ACTIVE', 'OUT_OF_STOCK'] },
      },
      take: options.limit || 100,
    });

    const results = [];

    for (const product of products) {
      try {
        const pricing = await this.calculatePrice(product.id);
        results.push({
          productId: product.id,
          pricing,
        });
      } catch (error) {
        this.logger.error(`Failed to calculate price for ${product.id}:`, error);
      }
    }

    return results;
  }

  private getStoreSettings(store: any): PricingSettings {
    const settings = store.settings || {};

    return {
      strategy: settings.pricingStrategy || 'PERCENTAGE_MARKUP',
      fixedMarkup: settings.fixedMarkup || 10,
      percentageMarkup: settings.percentageMarkup || 50,
      minimumMarginPercentage: settings.minimumMarginPercentage || 20,
      maximumMarginPercentage: settings.maximumMarginPercentage || 200,
      roundToNearest: settings.roundToNearest || 0.99,
      competitorPriceAdjustment: settings.competitorPriceAdjustment || -5,
    };
  }
}
```

---

## 2. Alert System

### 2.1 Alert Module Definition

```typescript
// services/ops-engine/src/modules/alerts/alerts.module.ts
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bull';
import { AlertsService } from './alerts.service';
import { AlertsController } from './alerts.controller';
import { AlertProcessor } from './alert.processor';
import { NotificationModule } from '../notifications/notification.module';

@Module({
  imports: [
    BullModule.registerQueue({
      name: 'alerts',
      defaultJobOptions: {
        removeOnComplete: 100,
        removeOnFail: 200,
      },
    }),
    NotificationModule,
  ],
  controllers: [AlertsController],
  providers: [AlertsService, AlertProcessor],
  exports: [AlertsService],
})
export class AlertsModule {}
```

### 2.2 Alert Types and DTOs

```typescript
// services/ops-engine/src/modules/alerts/dto/alert.dto.ts
import { IsEnum, IsString, IsOptional, IsObject, IsNumber } from 'class-validator';

export enum AlertType {
  STOCK_LOW = 'STOCK_LOW',
  STOCK_OUT = 'STOCK_OUT',
  STOCK_RESTORED = 'STOCK_RESTORED',
  PRICE_INCREASE = 'PRICE_INCREASE',
  PRICE_DECREASE = 'PRICE_DECREASE',
  PRICE_THRESHOLD = 'PRICE_THRESHOLD',
  SYNC_FAILED = 'SYNC_FAILED',
  SUPPLIER_UNAVAILABLE = 'SUPPLIER_UNAVAILABLE',
  MARGIN_WARNING = 'MARGIN_WARNING',
}

export enum AlertSeverity {
  INFO = 'INFO',
  WARNING = 'WARNING',
  CRITICAL = 'CRITICAL',
}

export enum AlertStatus {
  ACTIVE = 'ACTIVE',
  ACKNOWLEDGED = 'ACKNOWLEDGED',
  RESOLVED = 'RESOLVED',
  DISMISSED = 'DISMISSED',
}

export class CreateAlertDto {
  @IsEnum(AlertType)
  type: AlertType;

  @IsEnum(AlertSeverity)
  severity: AlertSeverity;

  @IsString()
  storeId: string;

  @IsString()
  @IsOptional()
  productId?: string;

  @IsString()
  title: string;

  @IsString()
  message: string;

  @IsObject()
  @IsOptional()
  metadata?: Record<string, any>;
}

export class AlertSettingsDto {
  @IsNumber()
  lowStockThreshold: number;

  @IsNumber()
  priceChangeThresholdPercent: number;

  @IsNumber()
  minimumMarginWarningPercent: number;

  emailNotifications: boolean;
  dashboardNotifications: boolean;
  webhookUrl?: string;
}
```

### 2.3 Alerts Service

```typescript
// services/ops-engine/src/modules/alerts/alerts.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectQueue } from '@nestjs/bull';
import { Queue } from 'bull';
import { PrismaService } from '@/shared/database/prisma.service';
import { RedisService } from '@/shared/cache/redis.service';
import {
  CreateAlertDto,
  AlertType,
  AlertSeverity,
  AlertStatus,
  AlertSettingsDto,
} from './dto/alert.dto';

@Injectable()
export class AlertsService {
  private readonly logger = new Logger(AlertsService.name);

  constructor(
    private readonly prisma: PrismaService,
    private readonly redis: RedisService,
    @InjectQueue('alerts') private readonly alertQueue: Queue,
  ) {}

  async createAlert(dto: CreateAlertDto): Promise<any> {
    // Check for duplicate active alerts
    const existingAlert = await this.prisma.alert.findFirst({
      where: {
        storeId: dto.storeId,
        productId: dto.productId,
        type: dto.type,
        status: AlertStatus.ACTIVE,
      },
    });

    if (existingAlert) {
      // Update existing alert count
      return this.prisma.alert.update({
        where: { id: existingAlert.id },
        data: {
          occurrenceCount: { increment: 1 },
          lastOccurredAt: new Date(),
          metadata: dto.metadata,
        },
      });
    }

    // Create new alert
    const alert = await this.prisma.alert.create({
      data: {
        type: dto.type,
        severity: dto.severity,
        storeId: dto.storeId,
        productId: dto.productId,
        title: dto.title,
        message: dto.message,
        metadata: dto.metadata,
        status: AlertStatus.ACTIVE,
        occurrenceCount: 1,
        lastOccurredAt: new Date(),
      },
    });

    // Queue notification job
    await this.alertQueue.add('send-notification', {
      alertId: alert.id,
      storeId: dto.storeId,
    });

    this.logger.log(`Created alert: ${dto.type} for store ${dto.storeId}`);

    return alert;
  }

  async checkStockAlerts(
    productId: string,
    previousStock: number,
    currentStock: number,
  ): Promise<void> {
    const product = await this.prisma.product.findUnique({
      where: { id: productId },
      include: { store: true },
    });

    if (!product) return;

    const settings = this.getAlertSettings(product.store);
    const lowStockThreshold = settings.lowStockThreshold || 5;

    // Stock depleted
    if (currentStock === 0 && previousStock > 0) {
      await this.createAlert({
        type: AlertType.STOCK_OUT,
        severity: AlertSeverity.CRITICAL,
        storeId: product.storeId,
        productId: product.id,
        title: 'Product Out of Stock',
        message: `${product.title} is now out of stock. Previous stock: ${previousStock}`,
        metadata: {
          previousStock,
          currentStock,
          productTitle: product.title,
        },
      });
    }
    // Stock restored
    else if (currentStock > 0 && previousStock === 0) {
      await this.createAlert({
        type: AlertType.STOCK_RESTORED,
        severity: AlertSeverity.INFO,
        storeId: product.storeId,
        productId: product.id,
        title: 'Stock Restored',
        message: `${product.title} is back in stock. Current stock: ${currentStock}`,
        metadata: {
          previousStock,
          currentStock,
          productTitle: product.title,
        },
      });

      // Resolve previous out of stock alert
      await this.resolveAlerts(product.storeId, product.id, AlertType.STOCK_OUT);
    }
    // Low stock warning
    else if (currentStock <= lowStockThreshold && currentStock > 0) {
      await this.createAlert({
        type: AlertType.STOCK_LOW,
        severity: AlertSeverity.WARNING,
        storeId: product.storeId,
        productId: product.id,
        title: 'Low Stock Warning',
        message: `${product.title} has low stock (${currentStock} remaining)`,
        metadata: {
          previousStock,
          currentStock,
          threshold: lowStockThreshold,
          productTitle: product.title,
        },
      });
    }
  }

  async checkPriceAlerts(
    productId: string,
    previousPrice: number,
    currentPrice: number,
  ): Promise<void> {
    const product = await this.prisma.product.findUnique({
      where: { id: productId },
      include: { store: true },
    });

    if (!product) return;

    const settings = this.getAlertSettings(product.store);
    const threshold = settings.priceChangeThresholdPercent || 10;

    const changePercent = ((currentPrice - previousPrice) / previousPrice) * 100;
    const absChangePercent = Math.abs(changePercent);

    if (absChangePercent >= threshold) {
      const alertType = changePercent > 0 ? AlertType.PRICE_INCREASE : AlertType.PRICE_DECREASE;
      const severity = absChangePercent >= 25 ? AlertSeverity.CRITICAL : AlertSeverity.WARNING;

      await this.createAlert({
        type: alertType,
        severity,
        storeId: product.storeId,
        productId: product.id,
        title: `Significant Price ${changePercent > 0 ? 'Increase' : 'Decrease'}`,
        message: `${product.title} cost changed by ${changePercent.toFixed(1)}% ($${previousPrice} → $${currentPrice})`,
        metadata: {
          previousPrice,
          currentPrice,
          changePercent,
          productTitle: product.title,
        },
      });
    }
  }

  async checkMarginAlerts(
    productId: string,
    costPrice: number,
    sellingPrice: number,
  ): Promise<void> {
    const product = await this.prisma.product.findUnique({
      where: { id: productId },
      include: { store: true },
    });

    if (!product) return;

    const settings = this.getAlertSettings(product.store);
    const minMargin = settings.minimumMarginWarningPercent || 15;

    const marginPercent = ((sellingPrice - costPrice) / costPrice) * 100;

    if (marginPercent < minMargin) {
      await this.createAlert({
        type: AlertType.MARGIN_WARNING,
        severity: marginPercent < 5 ? AlertSeverity.CRITICAL : AlertSeverity.WARNING,
        storeId: product.storeId,
        productId: product.id,
        title: 'Low Margin Warning',
        message: `${product.title} margin is ${marginPercent.toFixed(1)}% (below ${minMargin}% threshold)`,
        metadata: {
          costPrice,
          sellingPrice,
          marginPercent,
          threshold: minMargin,
          productTitle: product.title,
        },
      });
    }
  }

  async getAlerts(
    storeId: string,
    options: {
      status?: AlertStatus;
      type?: AlertType;
      severity?: AlertSeverity;
      limit?: number;
      offset?: number;
    } = {},
  ) {
    const where: any = { storeId };

    if (options.status) where.status = options.status;
    if (options.type) where.type = options.type;
    if (options.severity) where.severity = options.severity;

    const [alerts, total] = await Promise.all([
      this.prisma.alert.findMany({
        where,
        orderBy: { createdAt: 'desc' },
        take: options.limit || 50,
        skip: options.offset || 0,
        include: {
          product: {
            select: {
              id: true,
              title: true,
              images: true,
            },
          },
        },
      }),
      this.prisma.alert.count({ where }),
    ]);

    return {
      alerts,
      total,
      hasMore: (options.offset || 0) + alerts.length < total,
    };
  }

  async acknowledgeAlert(alertId: string, userId: string): Promise<any> {
    return this.prisma.alert.update({
      where: { id: alertId },
      data: {
        status: AlertStatus.ACKNOWLEDGED,
        acknowledgedAt: new Date(),
        acknowledgedBy: userId,
      },
    });
  }

  async resolveAlerts(
    storeId: string,
    productId: string,
    type: AlertType,
  ): Promise<void> {
    await this.prisma.alert.updateMany({
      where: {
        storeId,
        productId,
        type,
        status: { in: [AlertStatus.ACTIVE, AlertStatus.ACKNOWLEDGED] },
      },
      data: {
        status: AlertStatus.RESOLVED,
        resolvedAt: new Date(),
      },
    });
  }

  async dismissAlert(alertId: string): Promise<any> {
    return this.prisma.alert.update({
      where: { id: alertId },
      data: {
        status: AlertStatus.DISMISSED,
      },
    });
  }

  async getAlertSummary(storeId: string) {
    const [critical, warning, info, total] = await Promise.all([
      this.prisma.alert.count({
        where: { storeId, status: AlertStatus.ACTIVE, severity: AlertSeverity.CRITICAL },
      }),
      this.prisma.alert.count({
        where: { storeId, status: AlertStatus.ACTIVE, severity: AlertSeverity.WARNING },
      }),
      this.prisma.alert.count({
        where: { storeId, status: AlertStatus.ACTIVE, severity: AlertSeverity.INFO },
      }),
      this.prisma.alert.count({
        where: { storeId, status: AlertStatus.ACTIVE },
      }),
    ]);

    return {
      total,
      critical,
      warning,
      info,
    };
  }

  private getAlertSettings(store: any): AlertSettingsDto {
    const settings = store.settings || {};

    return {
      lowStockThreshold: settings.lowStockThreshold || 5,
      priceChangeThresholdPercent: settings.priceChangeThresholdPercent || 10,
      minimumMarginWarningPercent: settings.minimumMarginWarningPercent || 15,
      emailNotifications: settings.emailNotifications !== false,
      dashboardNotifications: settings.dashboardNotifications !== false,
      webhookUrl: settings.alertWebhookUrl,
    };
  }
}
```

### 2.4 Alert Processor

```typescript
// services/ops-engine/src/modules/alerts/alert.processor.ts
import { Process, Processor, OnQueueFailed } from '@nestjs/bull';
import { Logger } from '@nestjs/common';
import { Job } from 'bull';
import { PrismaService } from '@/shared/database/prisma.service';
import { NotificationService } from '../notifications/notification.service';

@Processor('alerts')
export class AlertProcessor {
  private readonly logger = new Logger(AlertProcessor.name);

  constructor(
    private readonly prisma: PrismaService,
    private readonly notificationService: NotificationService,
  ) {}

  @Process('send-notification')
  async handleSendNotification(job: Job<{ alertId: string; storeId: string }>) {
    const { alertId, storeId } = job.data;

    const alert = await this.prisma.alert.findUnique({
      where: { id: alertId },
      include: {
        store: {
          include: {
            user: true,
          },
        },
        product: true,
      },
    });

    if (!alert) {
      this.logger.warn(`Alert not found: ${alertId}`);
      return;
    }

    const settings = alert.store.settings as any || {};

    // Send email notification
    if (settings.emailNotifications !== false) {
      await this.notificationService.sendEmail({
        to: alert.store.user.email,
        template: 'alert',
        data: {
          alertType: alert.type,
          severity: alert.severity,
          title: alert.title,
          message: alert.message,
          storeName: alert.store.name,
          productTitle: alert.product?.title,
          metadata: alert.metadata,
          dashboardUrl: `${process.env.DASHBOARD_URL}/stores/${storeId}/alerts`,
        },
      });
    }

    // Send webhook notification
    if (settings.alertWebhookUrl) {
      await this.notificationService.sendWebhook(settings.alertWebhookUrl, {
        event: 'alert.created',
        alert: {
          id: alert.id,
          type: alert.type,
          severity: alert.severity,
          title: alert.title,
          message: alert.message,
          metadata: alert.metadata,
        },
      });
    }

    this.logger.log(`Sent notifications for alert: ${alertId}`);
  }

  @OnQueueFailed()
  onFailed(job: Job, error: Error) {
    this.logger.error(`Alert notification job ${job.id} failed:`, error);
  }
}
```

---

## 3. Notification Service

### 3.1 Notification Module

```typescript
// services/ops-engine/src/modules/notifications/notification.module.ts
import { Module } from '@nestjs/common';
import { NotificationService } from './notification.service';
import { EmailService } from './email.service';
import { WebhookService } from './webhook.service';

@Module({
  providers: [NotificationService, EmailService, WebhookService],
  exports: [NotificationService],
})
export class NotificationModule {}
```

### 3.2 Email Service

```typescript
// services/ops-engine/src/modules/notifications/email.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import * as nodemailer from 'nodemailer';
import * as Handlebars from 'handlebars';
import * as fs from 'fs';
import * as path from 'path';

export interface EmailOptions {
  to: string;
  subject?: string;
  template: string;
  data: Record<string, any>;
}

@Injectable()
export class EmailService {
  private readonly logger = new Logger(EmailService.name);
  private transporter: nodemailer.Transporter;
  private templates: Map<string, Handlebars.TemplateDelegate> = new Map();

  constructor(private readonly configService: ConfigService) {
    this.transporter = nodemailer.createTransport({
      host: this.configService.get('SMTP_HOST'),
      port: this.configService.get('SMTP_PORT', 587),
      secure: this.configService.get('SMTP_SECURE', false),
      auth: {
        user: this.configService.get('SMTP_USER'),
        pass: this.configService.get('SMTP_PASS'),
      },
    });

    this.loadTemplates();
  }

  private loadTemplates() {
    const templatesDir = path.join(__dirname, 'templates');

    if (fs.existsSync(templatesDir)) {
      const files = fs.readdirSync(templatesDir);

      for (const file of files) {
        if (file.endsWith('.hbs')) {
          const templateName = file.replace('.hbs', '');
          const templateContent = fs.readFileSync(
            path.join(templatesDir, file),
            'utf-8',
          );
          this.templates.set(templateName, Handlebars.compile(templateContent));
        }
      }
    }

    // Register partials
    Handlebars.registerPartial('header', this.getHeaderPartial());
    Handlebars.registerPartial('footer', this.getFooterPartial());
  }

  async send(options: EmailOptions): Promise<void> {
    const template = this.templates.get(options.template);

    if (!template) {
      throw new Error(`Email template not found: ${options.template}`);
    }

    const html = template(options.data);
    const subject = options.subject || this.getSubjectForTemplate(options.template, options.data);

    try {
      await this.transporter.sendMail({
        from: this.configService.get('SMTP_FROM', 'noreply@zyenta.com'),
        to: options.to,
        subject,
        html,
      });

      this.logger.log(`Email sent to ${options.to}: ${options.template}`);
    } catch (error) {
      this.logger.error(`Failed to send email to ${options.to}:`, error);
      throw error;
    }
  }

  private getSubjectForTemplate(template: string, data: any): string {
    const subjects: Record<string, string> = {
      alert: `[${data.severity}] ${data.title} - ${data.storeName}`,
      'stock-out': `Action Required: ${data.productTitle} is out of stock`,
      'price-change': `Price Alert: ${data.productTitle}`,
      'daily-summary': `Daily Summary for ${data.storeName}`,
      'sync-failed': `Sync Failed: ${data.storeName}`,
    };

    return subjects[template] || 'Zyenta Notification';
  }

  private getHeaderPartial(): string {
    return `
      <div style="background-color: #f8f9fa; padding: 20px; text-align: center;">
        <h1 style="color: #333; margin: 0;">Zyenta</h1>
      </div>
    `;
  }

  private getFooterPartial(): string {
    return `
      <div style="background-color: #f8f9fa; padding: 20px; text-align: center; font-size: 12px; color: #666;">
        <p>This is an automated message from Zyenta.</p>
        <p><a href="{{dashboardUrl}}">View in Dashboard</a> | <a href="{{unsubscribeUrl}}">Unsubscribe</a></p>
      </div>
    `;
  }
}
```

### 3.3 Email Templates

```handlebars
<!-- services/ops-engine/src/modules/notifications/templates/alert.hbs -->
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{{title}}</title>
  <style>
    body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; margin: 0; padding: 0; background-color: #f5f5f5; }
    .container { max-width: 600px; margin: 0 auto; background-color: #ffffff; }
    .header { padding: 24px; background-color: #1a1a1a; text-align: center; }
    .header h1 { color: #ffffff; margin: 0; font-size: 24px; }
    .content { padding: 32px 24px; }
    .alert-badge { display: inline-block; padding: 4px 12px; border-radius: 4px; font-size: 12px; font-weight: 600; text-transform: uppercase; }
    .badge-critical { background-color: #fee2e2; color: #dc2626; }
    .badge-warning { background-color: #fef3c7; color: #d97706; }
    .badge-info { background-color: #dbeafe; color: #2563eb; }
    .alert-title { font-size: 20px; font-weight: 600; margin: 16px 0 8px 0; color: #1a1a1a; }
    .alert-message { color: #4b5563; line-height: 1.6; margin-bottom: 24px; }
    .details-box { background-color: #f9fafb; border-radius: 8px; padding: 16px; margin-bottom: 24px; }
    .details-row { display: flex; justify-content: space-between; padding: 8px 0; border-bottom: 1px solid #e5e7eb; }
    .details-row:last-child { border-bottom: none; }
    .details-label { color: #6b7280; font-size: 14px; }
    .details-value { color: #1a1a1a; font-weight: 500; font-size: 14px; }
    .cta-button { display: inline-block; background-color: #1a1a1a; color: #ffffff; text-decoration: none; padding: 12px 24px; border-radius: 6px; font-weight: 500; }
    .footer { padding: 24px; background-color: #f9fafb; text-align: center; font-size: 12px; color: #6b7280; }
  </style>
</head>
<body>
  <div class="container">
    <div class="header">
      <h1>Zyenta</h1>
    </div>

    <div class="content">
      <span class="alert-badge badge-{{lowercase severity}}">{{severity}}</span>

      <h2 class="alert-title">{{title}}</h2>
      <p class="alert-message">{{message}}</p>

      {{#if productTitle}}
      <div class="details-box">
        <div class="details-row">
          <span class="details-label">Product</span>
          <span class="details-value">{{productTitle}}</span>
        </div>
        {{#if metadata.previousStock}}
        <div class="details-row">
          <span class="details-label">Previous Stock</span>
          <span class="details-value">{{metadata.previousStock}}</span>
        </div>
        <div class="details-row">
          <span class="details-label">Current Stock</span>
          <span class="details-value">{{metadata.currentStock}}</span>
        </div>
        {{/if}}
        {{#if metadata.previousPrice}}
        <div class="details-row">
          <span class="details-label">Previous Price</span>
          <span class="details-value">${{metadata.previousPrice}}</span>
        </div>
        <div class="details-row">
          <span class="details-label">Current Price</span>
          <span class="details-value">${{metadata.currentPrice}}</span>
        </div>
        <div class="details-row">
          <span class="details-label">Change</span>
          <span class="details-value">{{metadata.changePercent}}%</span>
        </div>
        {{/if}}
      </div>
      {{/if}}

      <a href="{{dashboardUrl}}" class="cta-button">View in Dashboard</a>
    </div>

    <div class="footer">
      <p>This is an automated message from Zyenta for {{storeName}}.</p>
      <p>You're receiving this because you have alerts enabled for your store.</p>
    </div>
  </div>
</body>
</html>
```

### 3.4 Webhook Service

```typescript
// services/ops-engine/src/modules/notifications/webhook.service.ts
import { Injectable, Logger } from '@nestjs/common';
import axios from 'axios';
import * as crypto from 'crypto';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class WebhookService {
  private readonly logger = new Logger(WebhookService.name);

  constructor(private readonly configService: ConfigService) {}

  async send(
    url: string,
    payload: Record<string, any>,
    options: {
      secret?: string;
      retries?: number;
    } = {},
  ): Promise<void> {
    const { secret, retries = 3 } = options;

    const body = JSON.stringify(payload);
    const headers: Record<string, string> = {
      'Content-Type': 'application/json',
      'X-Zyenta-Event': payload.event,
      'X-Zyenta-Timestamp': Date.now().toString(),
    };

    // Add signature if secret is provided
    if (secret) {
      const signature = crypto
        .createHmac('sha256', secret)
        .update(body)
        .digest('hex');
      headers['X-Zyenta-Signature'] = signature;
    }

    let lastError: Error | null = null;

    for (let attempt = 1; attempt <= retries; attempt++) {
      try {
        await axios.post(url, payload, {
          headers,
          timeout: 10000,
        });

        this.logger.log(`Webhook sent successfully to ${url}`);
        return;
      } catch (error) {
        lastError = error;
        this.logger.warn(
          `Webhook attempt ${attempt}/${retries} failed for ${url}:`,
          error.message,
        );

        if (attempt < retries) {
          await this.delay(Math.pow(2, attempt) * 1000);
        }
      }
    }

    this.logger.error(`Webhook failed after ${retries} attempts: ${url}`);
    throw lastError;
  }

  private delay(ms: number): Promise<void> {
    return new Promise((resolve) => setTimeout(resolve, ms));
  }
}
```

### 3.5 Notification Service

```typescript
// services/ops-engine/src/modules/notifications/notification.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { EmailService, EmailOptions } from './email.service';
import { WebhookService } from './webhook.service';

@Injectable()
export class NotificationService {
  private readonly logger = new Logger(NotificationService.name);

  constructor(
    private readonly emailService: EmailService,
    private readonly webhookService: WebhookService,
  ) {}

  async sendEmail(options: EmailOptions): Promise<void> {
    try {
      await this.emailService.send(options);
    } catch (error) {
      this.logger.error('Failed to send email:', error);
    }
  }

  async sendWebhook(url: string, payload: Record<string, any>): Promise<void> {
    try {
      await this.webhookService.send(url, payload);
    } catch (error) {
      this.logger.error('Failed to send webhook:', error);
    }
  }
}
```

---

## 4. Sync Logging & Monitoring

### 4.1 Sync Log Service

```typescript
// services/ops-engine/src/modules/monitoring/sync-log.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { PrismaService } from '@/shared/database/prisma.service';

export interface SyncLogData {
  storeId: string;
  type: 'INVENTORY' | 'PRICE' | 'FULL';
  status: 'STARTED' | 'COMPLETED' | 'FAILED';
  productsProcessed?: number;
  productsChanged?: number;
  duration?: number;
  error?: string;
  metadata?: Record<string, any>;
}

@Injectable()
export class SyncLogService {
  private readonly logger = new Logger(SyncLogService.name);

  constructor(private readonly prisma: PrismaService) {}

  async createLog(data: SyncLogData) {
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

  async getSyncHistory(
    storeId: string,
    options: {
      type?: string;
      days?: number;
      limit?: number;
    } = {},
  ) {
    const where: any = { storeId };

    if (options.type) {
      where.type = options.type;
    }

    if (options.days) {
      where.createdAt = {
        gte: new Date(Date.now() - options.days * 24 * 60 * 60 * 1000),
      };
    }

    return this.prisma.syncLog.findMany({
      where,
      orderBy: { createdAt: 'desc' },
      take: options.limit || 50,
    });
  }

  async getSyncStats(storeId: string, days: number = 7) {
    const startDate = new Date(Date.now() - days * 24 * 60 * 60 * 1000);

    const logs = await this.prisma.syncLog.findMany({
      where: {
        storeId,
        createdAt: { gte: startDate },
      },
    });

    const totalSyncs = logs.length;
    const successfulSyncs = logs.filter(l => l.status === 'COMPLETED').length;
    const failedSyncs = logs.filter(l => l.status === 'FAILED').length;
    const totalProductsProcessed = logs.reduce((sum, l) => sum + l.productsProcessed, 0);
    const totalProductsChanged = logs.reduce((sum, l) => sum + l.productsChanged, 0);
    const averageDuration = logs.length > 0
      ? logs.reduce((sum, l) => sum + l.duration, 0) / logs.length
      : 0;

    return {
      period: `${days} days`,
      totalSyncs,
      successfulSyncs,
      failedSyncs,
      successRate: totalSyncs > 0 ? (successfulSyncs / totalSyncs * 100).toFixed(1) : '0',
      totalProductsProcessed,
      totalProductsChanged,
      averageDuration: Math.round(averageDuration),
    };
  }

  async getLastSync(storeId: string, type?: string) {
    return this.prisma.syncLog.findFirst({
      where: {
        storeId,
        ...(type && { type }),
        status: 'COMPLETED',
      },
      orderBy: { createdAt: 'desc' },
    });
  }
}
```

### 4.2 Monitoring Controller

```typescript
// services/ops-engine/src/modules/monitoring/monitoring.controller.ts
import { Controller, Get, Param, Query } from '@nestjs/common';
import { SyncLogService } from './sync-log.service';
import { AlertsService } from '../alerts/alerts.service';

@Controller('monitoring')
export class MonitoringController {
  constructor(
    private readonly syncLogService: SyncLogService,
    private readonly alertsService: AlertsService,
  ) {}

  @Get('stores/:storeId/sync-history')
  async getSyncHistory(
    @Param('storeId') storeId: string,
    @Query('type') type?: string,
    @Query('days') days?: string,
    @Query('limit') limit?: string,
  ) {
    return this.syncLogService.getSyncHistory(storeId, {
      type,
      days: days ? parseInt(days) : undefined,
      limit: limit ? parseInt(limit) : undefined,
    });
  }

  @Get('stores/:storeId/sync-stats')
  async getSyncStats(
    @Param('storeId') storeId: string,
    @Query('days') days?: string,
  ) {
    return this.syncLogService.getSyncStats(
      storeId,
      days ? parseInt(days) : 7,
    );
  }

  @Get('stores/:storeId/dashboard')
  async getDashboard(@Param('storeId') storeId: string) {
    const [syncStats, alertSummary, lastSync] = await Promise.all([
      this.syncLogService.getSyncStats(storeId, 7),
      this.alertsService.getAlertSummary(storeId),
      this.syncLogService.getLastSync(storeId),
    ]);

    return {
      sync: {
        ...syncStats,
        lastSyncAt: lastSync?.createdAt,
      },
      alerts: alertSummary,
    };
  }
}
```

---

## 5. Database Schema Updates

### 5.1 Alert Model

```prisma
// packages/database/prisma/schema.prisma (additions)

model Alert {
  id              String       @id @default(cuid())
  storeId         String
  store           Store        @relation(fields: [storeId], references: [id], onDelete: Cascade)
  productId       String?
  product         Product?     @relation(fields: [productId], references: [id], onDelete: SetNull)

  type            String       // STOCK_LOW, STOCK_OUT, PRICE_INCREASE, etc.
  severity        String       // INFO, WARNING, CRITICAL
  status          String       @default("ACTIVE") // ACTIVE, ACKNOWLEDGED, RESOLVED, DISMISSED

  title           String
  message         String       @db.Text
  metadata        Json?

  occurrenceCount Int          @default(1)
  lastOccurredAt  DateTime     @default(now())

  acknowledgedAt  DateTime?
  acknowledgedBy  String?
  resolvedAt      DateTime?

  createdAt       DateTime     @default(now())
  updatedAt       DateTime     @updatedAt

  @@index([storeId])
  @@index([status])
  @@index([type])
  @@index([severity])
}

model SyncLog {
  id                String    @id @default(cuid())
  storeId           String
  store             Store     @relation(fields: [storeId], references: [id], onDelete: Cascade)

  type              String    // INVENTORY, PRICE, FULL
  status            String    // STARTED, COMPLETED, FAILED

  productsProcessed Int       @default(0)
  productsChanged   Int       @default(0)
  duration          Int       @default(0) // milliseconds

  error             String?   @db.Text
  metadata          Json?

  createdAt         DateTime  @default(now())

  @@index([storeId])
  @@index([type])
  @@index([status])
  @@index([createdAt])
}
```

---

## 6. Testing

### 6.1 Pricing Strategy Tests

```typescript
// services/ops-engine/src/modules/pricing/__tests__/pricing.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { PricingService } from '../pricing.service';
import { StrategyFactory } from '../strategies/strategy.factory';
import { PercentageMarkupStrategy } from '../strategies/percentage-markup.strategy';

describe('PricingService', () => {
  let service: PricingService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        PricingService,
        StrategyFactory,
        PercentageMarkupStrategy,
        // ... other strategies
      ],
    }).compile();

    service = module.get<PricingService>(PricingService);
  });

  describe('PercentageMarkupStrategy', () => {
    it('should calculate 50% markup correctly', () => {
      const strategy = new PercentageMarkupStrategy();
      const result = strategy.calculate({
        costPrice: 10,
        currentSellingPrice: 15,
        storeSettings: {
          strategy: 'PERCENTAGE_MARKUP',
          percentageMarkup: 50,
          minimumMarginPercentage: 20,
          maximumMarginPercentage: 200,
        },
      });

      expect(result.suggestedPrice).toBe(15);
      expect(result.marginPercentage).toBe(50);
    });

    it('should enforce minimum margin', () => {
      const strategy = new PercentageMarkupStrategy();
      const result = strategy.calculate({
        costPrice: 10,
        currentSellingPrice: 11,
        storeSettings: {
          strategy: 'PERCENTAGE_MARKUP',
          percentageMarkup: 5,
          minimumMarginPercentage: 20,
          maximumMarginPercentage: 200,
        },
      });

      expect(result.suggestedPrice).toBeGreaterThanOrEqual(12);
    });
  });
});
```

### 6.2 Alert Service Tests

```typescript
// services/ops-engine/src/modules/alerts/__tests__/alerts.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { AlertsService } from '../alerts.service';
import { AlertType, AlertSeverity } from '../dto/alert.dto';

describe('AlertsService', () => {
  let service: AlertsService;

  describe('checkStockAlerts', () => {
    it('should create STOCK_OUT alert when stock depletes', async () => {
      const createAlertSpy = jest.spyOn(service, 'createAlert');

      await service.checkStockAlerts('product-1', 10, 0);

      expect(createAlertSpy).toHaveBeenCalledWith(
        expect.objectContaining({
          type: AlertType.STOCK_OUT,
          severity: AlertSeverity.CRITICAL,
        }),
      );
    });

    it('should create STOCK_LOW alert when below threshold', async () => {
      const createAlertSpy = jest.spyOn(service, 'createAlert');

      await service.checkStockAlerts('product-1', 10, 3);

      expect(createAlertSpy).toHaveBeenCalledWith(
        expect.objectContaining({
          type: AlertType.STOCK_LOW,
          severity: AlertSeverity.WARNING,
        }),
      );
    });
  });

  describe('checkPriceAlerts', () => {
    it('should create PRICE_INCREASE alert for significant increase', async () => {
      const createAlertSpy = jest.spyOn(service, 'createAlert');

      await service.checkPriceAlerts('product-1', 10, 15); // 50% increase

      expect(createAlertSpy).toHaveBeenCalledWith(
        expect.objectContaining({
          type: AlertType.PRICE_INCREASE,
        }),
      );
    });
  });
});
```

---

## Verification Checklist

### Price Calculator
- [ ] Fixed markup strategy calculates correctly
- [ ] Percentage markup strategy calculates correctly
- [ ] Dynamic pricing considers demand and stock
- [ ] Margin limits are enforced
- [ ] Price rounding works correctly

### Alert System
- [ ] Stock alerts trigger appropriately
- [ ] Price alerts trigger on significant changes
- [ ] Margin warnings trigger below threshold
- [ ] Duplicate alerts are merged
- [ ] Alerts can be acknowledged and resolved

### Notifications
- [ ] Email templates render correctly
- [ ] Emails are sent successfully
- [ ] Webhooks deliver with retries
- [ ] Webhook signatures are valid

### Monitoring
- [ ] Sync logs are recorded
- [ ] Sync statistics are accurate
- [ ] Dashboard endpoint returns all data
- [ ] Historical data is queryable

---

## Next Steps

After completing Sprint 2, proceed to:

**Sprint 3: Fulfillment Orchestrator**
- XState-based workflow engine
- Supplier order placement adapters
- Tracking integration
- Customer notification system
- Fulfillment dashboard UI

---

*Phase 2 Sprint 2 establishes the price management system with dynamic pricing strategies and a comprehensive alert system for monitoring inventory and price changes.*
