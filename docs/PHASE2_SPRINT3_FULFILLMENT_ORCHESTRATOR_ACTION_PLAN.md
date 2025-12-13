# Phase 2 Sprint 3: Fulfillment Orchestrator - Action Plan

## Overview

This document provides a comprehensive implementation guide for Phase 2 Sprint 3, focusing on building an XState-based fulfillment workflow engine, supplier order placement adapters, tracking integration, and customer notification system.

### Sprint Objectives
- [ ] Implement XState-based workflow engine for order fulfillment
- [ ] Build supplier order placement adapters (AliExpress, CJ Dropshipping, Spocket)
- [ ] Implement tracking number integration and sync
- [ ] Build customer notification system (order status updates)
- [ ] Create fulfillment dashboard UI components

### Prerequisites
- Phase 2 Sprint 1 & 2 completed
- Inventory sync operational
- Alert system functional
- Supplier API credentials configured
- Email service configured

---

## 1. XState Workflow Engine

### 1.1 Fulfillment Module Definition

```typescript
// services/ops-engine/src/modules/fulfillment/fulfillment.module.ts
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bull';
import { FulfillmentService } from './fulfillment.service';
import { FulfillmentController } from './fulfillment.controller';
import { OrderStateMachine } from './state-machine/order.machine';
import { FulfillmentProcessor } from './fulfillment.processor';
import { SupplierOrderModule } from './suppliers/supplier-order.module';
import { TrackingModule } from './tracking/tracking.module';
import { CustomerNotificationModule } from './notifications/customer-notification.module';

@Module({
  imports: [
    BullModule.registerQueue({
      name: 'fulfillment',
      defaultJobOptions: {
        removeOnComplete: 100,
        removeOnFail: 200,
        attempts: 3,
        backoff: {
          type: 'exponential',
          delay: 5000,
        },
      },
    }),
    SupplierOrderModule,
    TrackingModule,
    CustomerNotificationModule,
  ],
  controllers: [FulfillmentController],
  providers: [
    FulfillmentService,
    OrderStateMachine,
    FulfillmentProcessor,
  ],
  exports: [FulfillmentService],
})
export class FulfillmentModule {}
```

### 1.2 Order State Machine Definition

```typescript
// services/ops-engine/src/modules/fulfillment/state-machine/order.machine.ts
import { Injectable, Logger } from '@nestjs/common';
import { createMachine, interpret, assign, StateMachine, Interpreter } from 'xstate';

// Order States
export enum OrderState {
  PENDING = 'pending',
  PAYMENT_CONFIRMED = 'payment_confirmed',
  PROCESSING = 'processing',
  PLACED_WITH_SUPPLIER = 'placed_with_supplier',
  SUPPLIER_CONFIRMED = 'supplier_confirmed',
  SHIPPED = 'shipped',
  IN_TRANSIT = 'in_transit',
  OUT_FOR_DELIVERY = 'out_for_delivery',
  DELIVERED = 'delivered',
  CANCELLED = 'cancelled',
  REFUND_REQUESTED = 'refund_requested',
  REFUNDED = 'refunded',
  FAILED = 'failed',
}

// Order Events
export type OrderEvent =
  | { type: 'PAYMENT_RECEIVED' }
  | { type: 'START_PROCESSING' }
  | { type: 'PLACE_ORDER'; supplierId?: string }
  | { type: 'SUPPLIER_CONFIRMED'; supplierOrderId: string }
  | { type: 'SUPPLIER_REJECTED'; reason: string }
  | { type: 'TRACKING_RECEIVED'; trackingNumber: string; carrier: string }
  | { type: 'IN_TRANSIT_UPDATE'; location?: string }
  | { type: 'OUT_FOR_DELIVERY_UPDATE' }
  | { type: 'DELIVERED'; deliveredAt: Date }
  | { type: 'CANCEL'; reason: string }
  | { type: 'REQUEST_REFUND'; reason: string }
  | { type: 'REFUND_PROCESSED'; refundId: string }
  | { type: 'RETRY' }
  | { type: 'FAIL'; error: string };

// Order Context
export interface OrderContext {
  orderId: string;
  storeId: string;
  customerId: string;
  items: OrderItem[];
  totalAmount: number;
  currency: string;
  supplierOrderId?: string;
  trackingNumber?: string;
  trackingCarrier?: string;
  trackingUrl?: string;
  shippingAddress: ShippingAddress;
  error?: string;
  retryCount: number;
  history: OrderHistoryEntry[];
}

interface OrderItem {
  productId: string;
  variantId?: string;
  sourceId: string;
  sourceProvider: string;
  quantity: number;
  unitPrice: number;
  costPrice: number;
}

interface ShippingAddress {
  name: string;
  line1: string;
  line2?: string;
  city: string;
  state: string;
  postalCode: string;
  country: string;
  phone?: string;
}

interface OrderHistoryEntry {
  state: string;
  timestamp: Date;
  metadata?: Record<string, any>;
}

@Injectable()
export class OrderStateMachine {
  private readonly logger = new Logger(OrderStateMachine.name);

  createMachine(initialContext: Partial<OrderContext>) {
    return createMachine<OrderContext, OrderEvent>(
      {
        id: 'order',
        initial: OrderState.PENDING,
        context: {
          orderId: '',
          storeId: '',
          customerId: '',
          items: [],
          totalAmount: 0,
          currency: 'USD',
          shippingAddress: {} as ShippingAddress,
          retryCount: 0,
          history: [],
          ...initialContext,
        },
        states: {
          [OrderState.PENDING]: {
            on: {
              PAYMENT_RECEIVED: {
                target: OrderState.PAYMENT_CONFIRMED,
                actions: ['recordHistory', 'notifyPaymentReceived'],
              },
              CANCEL: {
                target: OrderState.CANCELLED,
                actions: ['recordHistory', 'notifyCancellation'],
              },
            },
          },

          [OrderState.PAYMENT_CONFIRMED]: {
            on: {
              START_PROCESSING: {
                target: OrderState.PROCESSING,
                actions: ['recordHistory'],
              },
            },
            after: {
              // Auto-transition after 1 minute
              60000: {
                target: OrderState.PROCESSING,
                actions: ['recordHistory'],
              },
            },
          },

          [OrderState.PROCESSING]: {
            entry: ['startSupplierOrder'],
            on: {
              PLACE_ORDER: {
                target: OrderState.PLACED_WITH_SUPPLIER,
                actions: ['recordHistory', 'notifyOrderPlaced'],
              },
              FAIL: {
                target: OrderState.FAILED,
                actions: ['setError', 'recordHistory', 'notifyFailure'],
              },
              CANCEL: {
                target: OrderState.CANCELLED,
                actions: ['recordHistory', 'notifyCancellation'],
              },
            },
          },

          [OrderState.PLACED_WITH_SUPPLIER]: {
            on: {
              SUPPLIER_CONFIRMED: {
                target: OrderState.SUPPLIER_CONFIRMED,
                actions: ['setSupplierOrderId', 'recordHistory', 'notifySupplierConfirmed'],
              },
              SUPPLIER_REJECTED: {
                target: OrderState.FAILED,
                actions: ['setError', 'recordHistory', 'notifySupplierRejected'],
              },
              RETRY: {
                target: OrderState.PROCESSING,
                actions: ['incrementRetry', 'recordHistory'],
                cond: 'canRetry',
              },
            },
            after: {
              // Timeout after 24 hours
              86400000: {
                target: OrderState.FAILED,
                actions: ['setTimeoutError', 'recordHistory', 'notifyTimeout'],
              },
            },
          },

          [OrderState.SUPPLIER_CONFIRMED]: {
            on: {
              TRACKING_RECEIVED: {
                target: OrderState.SHIPPED,
                actions: ['setTrackingInfo', 'recordHistory', 'notifyShipped'],
              },
              CANCEL: {
                target: OrderState.REFUND_REQUESTED,
                actions: ['recordHistory', 'initiateRefund'],
              },
            },
          },

          [OrderState.SHIPPED]: {
            entry: ['startTrackingSync'],
            on: {
              IN_TRANSIT_UPDATE: {
                target: OrderState.IN_TRANSIT,
                actions: ['recordHistory', 'notifyInTransit'],
              },
            },
          },

          [OrderState.IN_TRANSIT]: {
            on: {
              IN_TRANSIT_UPDATE: {
                actions: ['updateTrackingLocation', 'recordHistory'],
              },
              OUT_FOR_DELIVERY_UPDATE: {
                target: OrderState.OUT_FOR_DELIVERY,
                actions: ['recordHistory', 'notifyOutForDelivery'],
              },
              DELIVERED: {
                target: OrderState.DELIVERED,
                actions: ['recordHistory', 'notifyDelivered'],
              },
            },
          },

          [OrderState.OUT_FOR_DELIVERY]: {
            on: {
              DELIVERED: {
                target: OrderState.DELIVERED,
                actions: ['recordHistory', 'notifyDelivered'],
              },
            },
          },

          [OrderState.DELIVERED]: {
            type: 'final',
            entry: ['completeOrder'],
            on: {
              REQUEST_REFUND: {
                target: OrderState.REFUND_REQUESTED,
                actions: ['recordHistory', 'initiateRefund'],
              },
            },
          },

          [OrderState.CANCELLED]: {
            type: 'final',
            entry: ['cancelOrder'],
          },

          [OrderState.REFUND_REQUESTED]: {
            on: {
              REFUND_PROCESSED: {
                target: OrderState.REFUNDED,
                actions: ['recordHistory', 'notifyRefunded'],
              },
            },
          },

          [OrderState.REFUNDED]: {
            type: 'final',
          },

          [OrderState.FAILED]: {
            on: {
              RETRY: {
                target: OrderState.PROCESSING,
                actions: ['incrementRetry', 'recordHistory'],
                cond: 'canRetry',
              },
              CANCEL: {
                target: OrderState.CANCELLED,
                actions: ['recordHistory'],
              },
            },
          },
        },
      },
      {
        actions: {
          recordHistory: assign({
            history: (context, event) => [
              ...context.history,
              {
                state: event.type,
                timestamp: new Date(),
                metadata: event,
              },
            ],
          }),

          setError: assign({
            error: (_, event: any) => event.error || event.reason,
          }),

          setTimeoutError: assign({
            error: () => 'Order placement timed out',
          }),

          setSupplierOrderId: assign({
            supplierOrderId: (_, event: any) => event.supplierOrderId,
          }),

          setTrackingInfo: assign({
            trackingNumber: (_, event: any) => event.trackingNumber,
            trackingCarrier: (_, event: any) => event.carrier,
          }),

          incrementRetry: assign({
            retryCount: (context) => context.retryCount + 1,
          }),

          updateTrackingLocation: assign({
            // Could store location history
          }),

          // These actions will be implemented by the service
          notifyPaymentReceived: () => {},
          notifyOrderPlaced: () => {},
          notifySupplierConfirmed: () => {},
          notifySupplierRejected: () => {},
          notifyShipped: () => {},
          notifyInTransit: () => {},
          notifyOutForDelivery: () => {},
          notifyDelivered: () => {},
          notifyCancellation: () => {},
          notifyFailure: () => {},
          notifyTimeout: () => {},
          notifyRefunded: () => {},
          startSupplierOrder: () => {},
          startTrackingSync: () => {},
          initiateRefund: () => {},
          completeOrder: () => {},
          cancelOrder: () => {},
        },

        guards: {
          canRetry: (context) => context.retryCount < 3,
        },
      },
    );
  }

  createInterpreter(
    initialContext: Partial<OrderContext>,
    actions: Record<string, (context: OrderContext, event: any) => void>,
  ): Interpreter<OrderContext, any, OrderEvent> {
    const machine = this.createMachine(initialContext);

    const service = interpret(
      machine.withConfig({
        actions: {
          ...machine.options.actions,
          ...actions,
        },
      }),
    );

    return service;
  }
}
```

### 1.3 Fulfillment Service

```typescript
// services/ops-engine/src/modules/fulfillment/fulfillment.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectQueue } from '@nestjs/bull';
import { Queue } from 'bull';
import { PrismaService } from '@/shared/database/prisma.service';
import { RedisService } from '@/shared/cache/redis.service';
import { OrderStateMachine, OrderContext, OrderEvent, OrderState } from './state-machine/order.machine';
import { SupplierOrderService } from './suppliers/supplier-order.service';
import { TrackingService } from './tracking/tracking.service';
import { CustomerNotificationService } from './notifications/customer-notification.service';

@Injectable()
export class FulfillmentService {
  private readonly logger = new Logger(FulfillmentService.name);

  constructor(
    private readonly prisma: PrismaService,
    private readonly redis: RedisService,
    private readonly stateMachine: OrderStateMachine,
    private readonly supplierOrderService: SupplierOrderService,
    private readonly trackingService: TrackingService,
    private readonly customerNotification: CustomerNotificationService,
    @InjectQueue('fulfillment') private readonly fulfillmentQueue: Queue,
  ) {}

  async processOrder(orderId: string): Promise<void> {
    const order = await this.getOrderWithDetails(orderId);

    if (!order) {
      throw new Error(`Order not found: ${orderId}`);
    }

    const context: Partial<OrderContext> = {
      orderId: order.id,
      storeId: order.storeId,
      customerId: order.customerId,
      items: order.items.map((item) => ({
        productId: item.productId,
        variantId: item.variantId,
        sourceId: item.product.sourceId,
        sourceProvider: item.product.sourceProvider,
        quantity: item.quantity,
        unitPrice: Number(item.unitPrice),
        costPrice: Number(item.product.costPrice),
      })),
      totalAmount: Number(order.totalAmount),
      currency: order.currency,
      shippingAddress: order.shippingAddress as any,
    };

    // Create interpreter with action implementations
    const interpreter = this.stateMachine.createInterpreter(context, {
      startSupplierOrder: async (ctx) => {
        await this.queueSupplierOrder(ctx.orderId);
      },
      startTrackingSync: async (ctx) => {
        await this.queueTrackingSync(ctx.orderId);
      },
      notifyPaymentReceived: async (ctx) => {
        await this.customerNotification.sendOrderConfirmation(ctx.orderId);
      },
      notifyShipped: async (ctx) => {
        await this.customerNotification.sendShippingNotification(ctx.orderId);
      },
      notifyDelivered: async (ctx) => {
        await this.customerNotification.sendDeliveryNotification(ctx.orderId);
      },
      notifyFailure: async (ctx) => {
        await this.customerNotification.sendFailureNotification(ctx.orderId, ctx.error);
      },
      completeOrder: async (ctx) => {
        await this.markOrderComplete(ctx.orderId);
      },
      cancelOrder: async (ctx) => {
        await this.markOrderCancelled(ctx.orderId);
      },
    });

    // Start the interpreter
    interpreter.start();

    // Store current state
    await this.saveOrderState(orderId, interpreter.getSnapshot());

    // Send payment received event to start the flow
    if (order.paymentStatus === 'PAID') {
      interpreter.send({ type: 'PAYMENT_RECEIVED' });
    }

    this.logger.log(`Started fulfillment for order: ${orderId}`);
  }

  async sendEvent(orderId: string, event: OrderEvent): Promise<void> {
    const savedState = await this.getOrderState(orderId);

    if (!savedState) {
      throw new Error(`No state found for order: ${orderId}`);
    }

    const order = await this.getOrderWithDetails(orderId);
    const context: Partial<OrderContext> = {
      ...savedState.context,
      orderId: order.id,
    };

    const interpreter = this.stateMachine.createInterpreter(context, {
      // ... same actions as above
    });

    // Restore previous state
    interpreter.start(savedState);

    // Send the event
    interpreter.send(event);

    // Save new state
    await this.saveOrderState(orderId, interpreter.getSnapshot());

    // Update order in database
    await this.updateOrderStatus(orderId, interpreter.getSnapshot().value as string);
  }

  async placeSupplierOrder(orderId: string): Promise<void> {
    const order = await this.getOrderWithDetails(orderId);

    if (!order) {
      throw new Error(`Order not found: ${orderId}`);
    }

    try {
      // Group items by supplier
      const itemsBySupplier = this.groupItemsBySupplier(order.items);

      for (const [provider, items] of Object.entries(itemsBySupplier)) {
        const result = await this.supplierOrderService.placeOrder({
          provider,
          items,
          shippingAddress: order.shippingAddress as any,
          orderId: order.id,
        });

        if (result.success) {
          await this.sendEvent(orderId, {
            type: 'SUPPLIER_CONFIRMED',
            supplierOrderId: result.supplierOrderId,
          });
        } else {
          await this.sendEvent(orderId, {
            type: 'SUPPLIER_REJECTED',
            reason: result.error || 'Unknown error',
          });
        }
      }
    } catch (error) {
      this.logger.error(`Failed to place supplier order: ${orderId}`, error);
      await this.sendEvent(orderId, {
        type: 'FAIL',
        error: error.message,
      });
    }
  }

  async syncTracking(orderId: string): Promise<void> {
    const order = await this.prisma.order.findUnique({
      where: { id: orderId },
    });

    if (!order || !order.trackingNumber) {
      return;
    }

    const trackingInfo = await this.trackingService.getTrackingInfo(
      order.trackingNumber,
      order.trackingCarrier,
    );

    if (trackingInfo.status === 'in_transit') {
      await this.sendEvent(orderId, {
        type: 'IN_TRANSIT_UPDATE',
        location: trackingInfo.currentLocation,
      });
    } else if (trackingInfo.status === 'out_for_delivery') {
      await this.sendEvent(orderId, { type: 'OUT_FOR_DELIVERY_UPDATE' });
    } else if (trackingInfo.status === 'delivered') {
      await this.sendEvent(orderId, {
        type: 'DELIVERED',
        deliveredAt: new Date(trackingInfo.deliveredAt),
      });
    }
  }

  async getOrderFulfillmentStatus(orderId: string) {
    const [order, state] = await Promise.all([
      this.getOrderWithDetails(orderId),
      this.getOrderState(orderId),
    ]);

    if (!order) {
      throw new Error(`Order not found: ${orderId}`);
    }

    return {
      orderId: order.id,
      status: order.status,
      fulfillmentState: state?.value || OrderState.PENDING,
      supplierOrderId: order.supplierOrderId,
      trackingNumber: order.trackingNumber,
      trackingCarrier: order.trackingCarrier,
      trackingUrl: order.trackingUrl,
      history: state?.context?.history || [],
      estimatedDelivery: order.estimatedDelivery,
    };
  }

  async cancelOrder(orderId: string, reason: string): Promise<void> {
    await this.sendEvent(orderId, { type: 'CANCEL', reason });

    // Attempt to cancel with supplier if already placed
    const order = await this.prisma.order.findUnique({
      where: { id: orderId },
    });

    if (order?.supplierOrderId) {
      await this.supplierOrderService.cancelOrder(
        order.supplierOrderId,
        order.items[0]?.product?.sourceProvider,
      );
    }
  }

  private async queueSupplierOrder(orderId: string): Promise<void> {
    await this.fulfillmentQueue.add('place-supplier-order', { orderId }, {
      delay: 1000, // Small delay to ensure state is saved
      priority: 1,
    });
  }

  private async queueTrackingSync(orderId: string): Promise<void> {
    await this.fulfillmentQueue.add('sync-tracking', { orderId }, {
      delay: 3600000, // Start tracking sync after 1 hour
      repeat: {
        every: 21600000, // Repeat every 6 hours
      },
    });
  }

  private async saveOrderState(orderId: string, state: any): Promise<void> {
    await this.redis.set(
      `order:state:${orderId}`,
      JSON.stringify(state),
      86400 * 30, // 30 days
    );
  }

  private async getOrderState(orderId: string): Promise<any> {
    const state = await this.redis.get<string>(`order:state:${orderId}`);
    return state ? JSON.parse(state) : null;
  }

  private async getOrderWithDetails(orderId: string) {
    return this.prisma.order.findUnique({
      where: { id: orderId },
      include: {
        items: {
          include: {
            product: true,
          },
        },
        store: true,
        customer: true,
      },
    });
  }

  private async updateOrderStatus(orderId: string, status: string): Promise<void> {
    await this.prisma.order.update({
      where: { id: orderId },
      data: {
        status: this.mapStateToStatus(status),
        updatedAt: new Date(),
      },
    });
  }

  private async markOrderComplete(orderId: string): Promise<void> {
    await this.prisma.order.update({
      where: { id: orderId },
      data: {
        status: 'DELIVERED',
        deliveredAt: new Date(),
      },
    });
  }

  private async markOrderCancelled(orderId: string): Promise<void> {
    await this.prisma.order.update({
      where: { id: orderId },
      data: {
        status: 'CANCELLED',
        cancelledAt: new Date(),
      },
    });
  }

  private groupItemsBySupplier(items: any[]): Record<string, any[]> {
    return items.reduce((acc, item) => {
      const provider = item.product.sourceProvider;
      if (!acc[provider]) {
        acc[provider] = [];
      }
      acc[provider].push(item);
      return acc;
    }, {} as Record<string, any[]>);
  }

  private mapStateToStatus(state: string): string {
    const mapping: Record<string, string> = {
      pending: 'PENDING',
      payment_confirmed: 'CONFIRMED',
      processing: 'PROCESSING',
      placed_with_supplier: 'PROCESSING',
      supplier_confirmed: 'PROCESSING',
      shipped: 'SHIPPED',
      in_transit: 'SHIPPED',
      out_for_delivery: 'OUT_FOR_DELIVERY',
      delivered: 'DELIVERED',
      cancelled: 'CANCELLED',
      refunded: 'REFUNDED',
      failed: 'FAILED',
    };

    return mapping[state] || 'PENDING';
  }
}
```

---

## 2. Supplier Order Adapters

### 2.1 Supplier Order Module

```typescript
// services/ops-engine/src/modules/fulfillment/suppliers/supplier-order.module.ts
import { Module } from '@nestjs/common';
import { SupplierOrderService } from './supplier-order.service';
import { AliExpressOrderAdapter } from './aliexpress-order.adapter';
import { CJDropshippingOrderAdapter } from './cj-order.adapter';
import { SpocketOrderAdapter } from './spocket-order.adapter';

@Module({
  providers: [
    SupplierOrderService,
    AliExpressOrderAdapter,
    CJDropshippingOrderAdapter,
    SpocketOrderAdapter,
  ],
  exports: [SupplierOrderService],
})
export class SupplierOrderModule {}
```

### 2.2 Supplier Order Service

```typescript
// services/ops-engine/src/modules/fulfillment/suppliers/supplier-order.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { PrismaService } from '@/shared/database/prisma.service';
import { AliExpressOrderAdapter } from './aliexpress-order.adapter';
import { CJDropshippingOrderAdapter } from './cj-order.adapter';
import { SpocketOrderAdapter } from './spocket-order.adapter';
import { BaseOrderAdapter, OrderPlacementRequest, OrderPlacementResult } from './base-order.adapter';

@Injectable()
export class SupplierOrderService {
  private readonly logger = new Logger(SupplierOrderService.name);

  constructor(
    private readonly prisma: PrismaService,
    private readonly aliexpress: AliExpressOrderAdapter,
    private readonly cjDropshipping: CJDropshippingOrderAdapter,
    private readonly spocket: SpocketOrderAdapter,
  ) {}

  async placeOrder(request: OrderPlacementRequest): Promise<OrderPlacementResult> {
    const adapter = this.getAdapter(request.provider);

    if (!adapter) {
      return {
        success: false,
        error: `Unsupported supplier: ${request.provider}`,
      };
    }

    // Get supplier credentials
    const credentials = await this.getSupplierCredentials(
      request.orderId,
      request.provider,
    );

    if (!credentials) {
      return {
        success: false,
        error: `No supplier credentials found for: ${request.provider}`,
      };
    }

    try {
      const result = await adapter.placeOrder(request, credentials);

      // Log the order placement
      await this.logOrderPlacement(request, result);

      return result;
    } catch (error) {
      this.logger.error(`Failed to place order with ${request.provider}:`, error);

      return {
        success: false,
        error: error.message,
      };
    }
  }

  async getOrderStatus(
    supplierOrderId: string,
    provider: string,
  ): Promise<{
    status: string;
    trackingNumber?: string;
    trackingCarrier?: string;
  }> {
    const adapter = this.getAdapter(provider);

    if (!adapter) {
      throw new Error(`Unsupported supplier: ${provider}`);
    }

    const credentials = await this.getSupplierCredentialsByProvider(provider);
    return adapter.getOrderStatus(supplierOrderId, credentials);
  }

  async cancelOrder(
    supplierOrderId: string,
    provider: string,
  ): Promise<{ success: boolean; error?: string }> {
    const adapter = this.getAdapter(provider);

    if (!adapter) {
      return { success: false, error: `Unsupported supplier: ${provider}` };
    }

    const credentials = await this.getSupplierCredentialsByProvider(provider);
    return adapter.cancelOrder(supplierOrderId, credentials);
  }

  private getAdapter(provider: string): BaseOrderAdapter | null {
    switch (provider) {
      case 'ALIEXPRESS':
        return this.aliexpress;
      case 'CJ_DROPSHIPPING':
        return this.cjDropshipping;
      case 'SPOCKET':
        return this.spocket;
      default:
        return null;
    }
  }

  private async getSupplierCredentials(orderId: string, provider: string) {
    const order = await this.prisma.order.findUnique({
      where: { id: orderId },
      include: {
        store: {
          include: {
            user: {
              include: {
                supplierAccounts: {
                  where: { provider },
                },
              },
            },
          },
        },
      },
    });

    return order?.store?.user?.supplierAccounts?.[0];
  }

  private async getSupplierCredentialsByProvider(provider: string) {
    // Get from environment or database
    return {
      apiKey: process.env[`${provider}_API_KEY`],
      apiSecret: process.env[`${provider}_API_SECRET`],
    };
  }

  private async logOrderPlacement(
    request: OrderPlacementRequest,
    result: OrderPlacementResult,
  ) {
    await this.prisma.supplierOrderLog.create({
      data: {
        orderId: request.orderId,
        provider: request.provider,
        supplierOrderId: result.supplierOrderId,
        success: result.success,
        error: result.error,
        requestPayload: request as any,
        responsePayload: result as any,
      },
    });
  }
}
```

### 2.3 Base Order Adapter

```typescript
// services/ops-engine/src/modules/fulfillment/suppliers/base-order.adapter.ts
export interface OrderPlacementRequest {
  orderId: string;
  provider: string;
  items: OrderItem[];
  shippingAddress: ShippingAddress;
}

export interface OrderItem {
  productId: string;
  sourceId: string;
  quantity: number;
  unitPrice: number;
  variantId?: string;
}

export interface ShippingAddress {
  name: string;
  line1: string;
  line2?: string;
  city: string;
  state: string;
  postalCode: string;
  country: string;
  phone?: string;
}

export interface OrderPlacementResult {
  success: boolean;
  supplierOrderId?: string;
  estimatedDeliveryDate?: Date;
  error?: string;
}

export interface SupplierCredentials {
  apiKey: string;
  apiSecret?: string;
  accessToken?: string;
}

export abstract class BaseOrderAdapter {
  abstract placeOrder(
    request: OrderPlacementRequest,
    credentials: SupplierCredentials,
  ): Promise<OrderPlacementResult>;

  abstract getOrderStatus(
    supplierOrderId: string,
    credentials: SupplierCredentials,
  ): Promise<{
    status: string;
    trackingNumber?: string;
    trackingCarrier?: string;
  }>;

  abstract cancelOrder(
    supplierOrderId: string,
    credentials: SupplierCredentials,
  ): Promise<{ success: boolean; error?: string }>;
}
```

### 2.4 AliExpress Order Adapter

```typescript
// services/ops-engine/src/modules/fulfillment/suppliers/aliexpress-order.adapter.ts
import { Injectable, Logger } from '@nestjs/common';
import axios from 'axios';
import * as crypto from 'crypto';
import {
  BaseOrderAdapter,
  OrderPlacementRequest,
  OrderPlacementResult,
  SupplierCredentials,
} from './base-order.adapter';

@Injectable()
export class AliExpressOrderAdapter extends BaseOrderAdapter {
  private readonly logger = new Logger(AliExpressOrderAdapter.name);
  private readonly baseUrl = 'https://api-sg.aliexpress.com/sync';

  async placeOrder(
    request: OrderPlacementRequest,
    credentials: SupplierCredentials,
  ): Promise<OrderPlacementResult> {
    try {
      const params = this.buildOrderParams(request, credentials);
      params.sign = this.generateSignature(params, credentials.apiSecret);

      const response = await axios.post(this.baseUrl, null, { params });
      const result = response.data?.aliexpress_ds_order_create_response;

      if (result?.result?.is_success) {
        return {
          success: true,
          supplierOrderId: result.result.order_id,
          estimatedDeliveryDate: this.calculateEstimatedDelivery(14), // 14 days default
        };
      }

      return {
        success: false,
        error: result?.result?.error_msg || 'Order placement failed',
      };
    } catch (error) {
      this.logger.error('AliExpress order placement failed:', error);
      return {
        success: false,
        error: error.message,
      };
    }
  }

  async getOrderStatus(
    supplierOrderId: string,
    credentials: SupplierCredentials,
  ): Promise<{
    status: string;
    trackingNumber?: string;
    trackingCarrier?: string;
  }> {
    try {
      const params: Record<string, string> = {
        app_key: credentials.apiKey,
        access_token: credentials.accessToken,
        method: 'aliexpress.ds.order.get',
        sign_method: 'md5',
        timestamp: new Date().toISOString(),
        order_id: supplierOrderId,
      };

      params.sign = this.generateSignature(params, credentials.apiSecret);

      const response = await axios.post(this.baseUrl, null, { params });
      const result = response.data?.aliexpress_ds_order_get_response?.result;

      return {
        status: this.mapOrderStatus(result?.order_status),
        trackingNumber: result?.logistics_info_list?.[0]?.logistics_no,
        trackingCarrier: result?.logistics_info_list?.[0]?.logistics_service,
      };
    } catch (error) {
      this.logger.error('Failed to get AliExpress order status:', error);
      throw error;
    }
  }

  async cancelOrder(
    supplierOrderId: string,
    credentials: SupplierCredentials,
  ): Promise<{ success: boolean; error?: string }> {
    try {
      const params: Record<string, string> = {
        app_key: credentials.apiKey,
        access_token: credentials.accessToken,
        method: 'aliexpress.ds.order.cancel',
        sign_method: 'md5',
        timestamp: new Date().toISOString(),
        order_id: supplierOrderId,
      };

      params.sign = this.generateSignature(params, credentials.apiSecret);

      const response = await axios.post(this.baseUrl, null, { params });
      const result = response.data?.aliexpress_ds_order_cancel_response;

      return {
        success: result?.result?.is_success || false,
        error: result?.result?.error_msg,
      };
    } catch (error) {
      return {
        success: false,
        error: error.message,
      };
    }
  }

  private buildOrderParams(
    request: OrderPlacementRequest,
    credentials: SupplierCredentials,
  ): Record<string, string> {
    const address = request.shippingAddress;

    return {
      app_key: credentials.apiKey,
      access_token: credentials.accessToken,
      method: 'aliexpress.ds.order.create',
      sign_method: 'md5',
      timestamp: new Date().toISOString(),
      product_items: JSON.stringify(
        request.items.map((item) => ({
          product_id: item.sourceId,
          sku_attr: item.variantId || '',
          quantity: item.quantity,
        })),
      ),
      logistics_address: JSON.stringify({
        contact_person: address.name,
        full_name: address.name,
        address: address.line1,
        address2: address.line2 || '',
        city: address.city,
        province: address.state,
        zip: address.postalCode,
        country: address.country,
        mobile_no: address.phone || '',
      }),
    };
  }

  private generateSignature(params: Record<string, string>, secret: string): string {
    const sortedKeys = Object.keys(params).sort();
    let signString = secret;

    for (const key of sortedKeys) {
      signString += key + params[key];
    }

    signString += secret;

    return crypto.createHash('md5').update(signString).digest('hex').toUpperCase();
  }

  private mapOrderStatus(status: string): string {
    const mapping: Record<string, string> = {
      PLACE_ORDER_SUCCESS: 'confirmed',
      IN_CANCEL: 'cancelling',
      WAIT_SELLER_SEND_GOODS: 'processing',
      SELLER_PART_SEND_GOODS: 'partially_shipped',
      WAIT_BUYER_ACCEPT_GOODS: 'shipped',
      FUND_PROCESSING: 'processing',
      FINISH: 'delivered',
      IN_FROZEN: 'on_hold',
      NO_PAYMENT: 'pending_payment',
    };

    return mapping[status] || 'unknown';
  }

  private calculateEstimatedDelivery(days: number): Date {
    const date = new Date();
    date.setDate(date.getDate() + days);
    return date;
  }
}
```

### 2.5 CJ Dropshipping Order Adapter

```typescript
// services/ops-engine/src/modules/fulfillment/suppliers/cj-order.adapter.ts
import { Injectable, Logger } from '@nestjs/common';
import axios, { AxiosInstance } from 'axios';
import {
  BaseOrderAdapter,
  OrderPlacementRequest,
  OrderPlacementResult,
  SupplierCredentials,
} from './base-order.adapter';

@Injectable()
export class CJDropshippingOrderAdapter extends BaseOrderAdapter {
  private readonly logger = new Logger(CJDropshippingOrderAdapter.name);
  private readonly baseUrl = 'https://developers.cjdropshipping.com/api2.0/v1';

  private createClient(credentials: SupplierCredentials): AxiosInstance {
    return axios.create({
      baseURL: this.baseUrl,
      headers: {
        'CJ-Access-Token': credentials.apiKey,
        'Content-Type': 'application/json',
      },
      timeout: 30000,
    });
  }

  async placeOrder(
    request: OrderPlacementRequest,
    credentials: SupplierCredentials,
  ): Promise<OrderPlacementResult> {
    const client = this.createClient(credentials);

    try {
      const orderData = this.buildOrderPayload(request);
      const response = await client.post('/shopping/order/createOrder', orderData);

      if (response.data?.result && response.data?.data?.orderId) {
        return {
          success: true,
          supplierOrderId: response.data.data.orderId,
          estimatedDeliveryDate: this.calculateEstimatedDelivery(
            response.data.data.logisticsInfo?.shipDays || 12,
          ),
        };
      }

      return {
        success: false,
        error: response.data?.message || 'Order placement failed',
      };
    } catch (error) {
      this.logger.error('CJ Dropshipping order placement failed:', error);
      return {
        success: false,
        error: error.response?.data?.message || error.message,
      };
    }
  }

  async getOrderStatus(
    supplierOrderId: string,
    credentials: SupplierCredentials,
  ): Promise<{
    status: string;
    trackingNumber?: string;
    trackingCarrier?: string;
  }> {
    const client = this.createClient(credentials);

    try {
      const response = await client.get('/shopping/order/getOrderDetail', {
        params: { orderId: supplierOrderId },
      });

      const order = response.data?.data;

      return {
        status: this.mapOrderStatus(order?.orderStatus),
        trackingNumber: order?.trackNumber,
        trackingCarrier: order?.logisticsName,
      };
    } catch (error) {
      this.logger.error('Failed to get CJ order status:', error);
      throw error;
    }
  }

  async cancelOrder(
    supplierOrderId: string,
    credentials: SupplierCredentials,
  ): Promise<{ success: boolean; error?: string }> {
    const client = this.createClient(credentials);

    try {
      const response = await client.post('/shopping/order/deleteOrder', {
        orderId: supplierOrderId,
      });

      return {
        success: response.data?.result || false,
        error: response.data?.message,
      };
    } catch (error) {
      return {
        success: false,
        error: error.message,
      };
    }
  }

  private buildOrderPayload(request: OrderPlacementRequest): any {
    const address = request.shippingAddress;

    return {
      orderNumber: request.orderId,
      shippingZip: address.postalCode,
      shippingCountry: address.country,
      shippingProvince: address.state,
      shippingCity: address.city,
      shippingAddress: address.line1 + (address.line2 ? `, ${address.line2}` : ''),
      shippingCustomerName: address.name,
      shippingPhone: address.phone || '',
      products: request.items.map((item) => ({
        vid: item.sourceId,
        quantity: item.quantity,
      })),
    };
  }

  private mapOrderStatus(status: string): string {
    const mapping: Record<string, string> = {
      CREATED: 'pending',
      IN_CART: 'pending',
      UNPAID: 'pending_payment',
      UNSHIPPED: 'processing',
      SHIPPED: 'shipped',
      DELIVERED: 'delivered',
      CANCELLED: 'cancelled',
    };

    return mapping[status] || 'unknown';
  }

  private calculateEstimatedDelivery(days: number): Date {
    const date = new Date();
    date.setDate(date.getDate() + days);
    return date;
  }
}
```

---

## 3. Tracking Integration

### 3.1 Tracking Module

```typescript
// services/ops-engine/src/modules/fulfillment/tracking/tracking.module.ts
import { Module } from '@nestjs/common';
import { TrackingService } from './tracking.service';
import { TrackingController } from './tracking.controller';
import { Track17Adapter } from './adapters/track17.adapter';
import { AfterShipAdapter } from './adapters/aftership.adapter';

@Module({
  controllers: [TrackingController],
  providers: [TrackingService, Track17Adapter, AfterShipAdapter],
  exports: [TrackingService],
})
export class TrackingModule {}
```

### 3.2 Tracking Service

```typescript
// services/ops-engine/src/modules/fulfillment/tracking/tracking.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { PrismaService } from '@/shared/database/prisma.service';
import { Track17Adapter } from './adapters/track17.adapter';
import { AfterShipAdapter } from './adapters/aftership.adapter';

export interface TrackingInfo {
  trackingNumber: string;
  carrier: string;
  status: 'pending' | 'in_transit' | 'out_for_delivery' | 'delivered' | 'exception';
  currentLocation?: string;
  deliveredAt?: string;
  estimatedDelivery?: string;
  events: TrackingEvent[];
}

export interface TrackingEvent {
  timestamp: Date;
  status: string;
  description: string;
  location?: string;
}

@Injectable()
export class TrackingService {
  private readonly logger = new Logger(TrackingService.name);

  constructor(
    private readonly prisma: PrismaService,
    private readonly configService: ConfigService,
    private readonly track17: Track17Adapter,
    private readonly aftership: AfterShipAdapter,
  ) {}

  async getTrackingInfo(
    trackingNumber: string,
    carrier?: string,
  ): Promise<TrackingInfo> {
    // Try 17Track first (free tier available)
    try {
      return await this.track17.getTracking(trackingNumber, carrier);
    } catch (error) {
      this.logger.warn('17Track failed, trying AfterShip:', error.message);
    }

    // Fallback to AfterShip
    return this.aftership.getTracking(trackingNumber, carrier);
  }

  async syncOrderTracking(orderId: string): Promise<void> {
    const order = await this.prisma.order.findUnique({
      where: { id: orderId },
    });

    if (!order?.trackingNumber) {
      return;
    }

    try {
      const trackingInfo = await this.getTrackingInfo(
        order.trackingNumber,
        order.trackingCarrier,
      );

      // Update order with latest tracking info
      await this.prisma.order.update({
        where: { id: orderId },
        data: {
          trackingStatus: trackingInfo.status,
          trackingLastUpdate: new Date(),
          trackingHistory: trackingInfo.events as any,
          estimatedDelivery: trackingInfo.estimatedDelivery
            ? new Date(trackingInfo.estimatedDelivery)
            : undefined,
        },
      });

      // Create tracking update record
      await this.prisma.trackingUpdate.create({
        data: {
          orderId,
          status: trackingInfo.status,
          location: trackingInfo.currentLocation,
          events: trackingInfo.events as any,
        },
      });

      this.logger.log(`Synced tracking for order ${orderId}: ${trackingInfo.status}`);
    } catch (error) {
      this.logger.error(`Failed to sync tracking for order ${orderId}:`, error);
    }
  }

  async getTrackingUrl(
    trackingNumber: string,
    carrier: string,
  ): Promise<string> {
    const carrierUrls: Record<string, string> = {
      'yanwen': `https://track.yw56.com.cn/en/querydel?nums=${trackingNumber}`,
      'china_post': `https://www.17track.net/en/track?nums=${trackingNumber}`,
      'cainiao': `https://global.cainiao.com/detail.htm?mailNoList=${trackingNumber}`,
      'dhl': `https://www.dhl.com/en/express/tracking.html?AWB=${trackingNumber}`,
      'usps': `https://tools.usps.com/go/TrackConfirmAction?tLabels=${trackingNumber}`,
      'fedex': `https://www.fedex.com/fedextrack/?trknbr=${trackingNumber}`,
      'ups': `https://www.ups.com/track?tracknum=${trackingNumber}`,
      'default': `https://www.17track.net/en/track?nums=${trackingNumber}`,
    };

    const normalizedCarrier = carrier.toLowerCase().replace(/[^a-z]/g, '_');
    return carrierUrls[normalizedCarrier] || carrierUrls['default'];
  }

  async detectCarrier(trackingNumber: string): Promise<string | null> {
    // Common carrier detection patterns
    const patterns: Array<{ pattern: RegExp; carrier: string }> = [
      { pattern: /^[A-Z]{2}\d{9}[A-Z]{2}$/, carrier: 'china_post' },
      { pattern: /^LP\d{14}$/, carrier: 'yanwen' },
      { pattern: /^YT\d{16}$/, carrier: 'yto' },
      { pattern: /^\d{10,22}$/, carrier: 'dhl' },
      { pattern: /^1Z[A-Z0-9]{16}$/, carrier: 'ups' },
      { pattern: /^\d{12,14}$/, carrier: 'fedex' },
    ];

    for (const { pattern, carrier } of patterns) {
      if (pattern.test(trackingNumber)) {
        return carrier;
      }
    }

    return null;
  }
}
```

### 3.3 17Track Adapter

```typescript
// services/ops-engine/src/modules/fulfillment/tracking/adapters/track17.adapter.ts
import { Injectable, Logger } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import axios from 'axios';
import { TrackingInfo, TrackingEvent } from '../tracking.service';

@Injectable()
export class Track17Adapter {
  private readonly logger = new Logger(Track17Adapter.name);
  private readonly apiKey: string;
  private readonly baseUrl = 'https://api.17track.net/track/v2';

  constructor(private readonly configService: ConfigService) {
    this.apiKey = this.configService.get('TRACK17_API_KEY');
  }

  async getTracking(trackingNumber: string, carrier?: string): Promise<TrackingInfo> {
    try {
      const response = await axios.post(
        `${this.baseUrl}/gettrackinfo`,
        [{ number: trackingNumber, carrier: carrier ? parseInt(carrier) : undefined }],
        {
          headers: {
            '17token': this.apiKey,
            'Content-Type': 'application/json',
          },
        },
      );

      const data = response.data?.data?.accepted?.[0];

      if (!data) {
        throw new Error('No tracking data found');
      }

      const track = data.track;
      const events = this.parseEvents(track?.z1 || track?.z2 || []);

      return {
        trackingNumber,
        carrier: track?.w1 || carrier || 'unknown',
        status: this.mapStatus(track?.e),
        currentLocation: events[0]?.location,
        deliveredAt: track?.e === 40 ? events[0]?.timestamp.toISOString() : undefined,
        estimatedDelivery: track?.ymd,
        events,
      };
    } catch (error) {
      this.logger.error('17Track API error:', error);
      throw error;
    }
  }

  private parseEvents(rawEvents: any[]): TrackingEvent[] {
    return rawEvents.map((event) => ({
      timestamp: new Date(event.a || event.d),
      status: event.z || '',
      description: event.c || event.z || '',
      location: event.d || '',
    }));
  }

  private mapStatus(code: number): TrackingInfo['status'] {
    const mapping: Record<number, TrackingInfo['status']> = {
      0: 'pending',
      10: 'in_transit',
      20: 'in_transit',
      30: 'out_for_delivery',
      40: 'delivered',
      50: 'exception',
    };

    return mapping[code] || 'pending';
  }
}
```

---

## 4. Customer Notification System

### 4.1 Customer Notification Module

```typescript
// services/ops-engine/src/modules/fulfillment/notifications/customer-notification.module.ts
import { Module } from '@nestjs/common';
import { CustomerNotificationService } from './customer-notification.service';
import { NotificationModule } from '../../notifications/notification.module';

@Module({
  imports: [NotificationModule],
  providers: [CustomerNotificationService],
  exports: [CustomerNotificationService],
})
export class CustomerNotificationModule {}
```

### 4.2 Customer Notification Service

```typescript
// services/ops-engine/src/modules/fulfillment/notifications/customer-notification.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { PrismaService } from '@/shared/database/prisma.service';
import { EmailService } from '../../notifications/email.service';

@Injectable()
export class CustomerNotificationService {
  private readonly logger = new Logger(CustomerNotificationService.name);

  constructor(
    private readonly prisma: PrismaService,
    private readonly emailService: EmailService,
  ) {}

  async sendOrderConfirmation(orderId: string): Promise<void> {
    const order = await this.getOrderWithDetails(orderId);

    if (!order) return;

    await this.emailService.send({
      to: order.customer.email,
      template: 'order-confirmation',
      data: {
        customerName: order.customer.name,
        orderNumber: order.orderNumber,
        orderDate: order.createdAt,
        items: order.items.map((item) => ({
          name: item.product.title,
          quantity: item.quantity,
          price: Number(item.unitPrice),
          image: item.product.images?.[0]?.url,
        })),
        subtotal: Number(order.subtotal),
        shipping: Number(order.shippingCost),
        total: Number(order.totalAmount),
        shippingAddress: order.shippingAddress,
        storeName: order.store.name,
        storeUrl: this.getStoreUrl(order.store),
      },
    });

    await this.logNotification(orderId, 'ORDER_CONFIRMATION', 'email');
  }

  async sendShippingNotification(orderId: string): Promise<void> {
    const order = await this.getOrderWithDetails(orderId);

    if (!order) return;

    const trackingUrl = order.trackingUrl || this.buildTrackingUrl(
      order.trackingNumber,
      order.trackingCarrier,
    );

    await this.emailService.send({
      to: order.customer.email,
      template: 'order-shipped',
      data: {
        customerName: order.customer.name,
        orderNumber: order.orderNumber,
        trackingNumber: order.trackingNumber,
        trackingCarrier: order.trackingCarrier,
        trackingUrl,
        estimatedDelivery: order.estimatedDelivery,
        items: order.items.map((item) => ({
          name: item.product.title,
          quantity: item.quantity,
          image: item.product.images?.[0]?.url,
        })),
        shippingAddress: order.shippingAddress,
        storeName: order.store.name,
      },
    });

    await this.logNotification(orderId, 'SHIPPING_NOTIFICATION', 'email');
  }

  async sendDeliveryNotification(orderId: string): Promise<void> {
    const order = await this.getOrderWithDetails(orderId);

    if (!order) return;

    await this.emailService.send({
      to: order.customer.email,
      template: 'order-delivered',
      data: {
        customerName: order.customer.name,
        orderNumber: order.orderNumber,
        deliveredAt: order.deliveredAt,
        items: order.items.map((item) => ({
          name: item.product.title,
          quantity: item.quantity,
          image: item.product.images?.[0]?.url,
        })),
        reviewUrl: `${this.getStoreUrl(order.store)}/orders/${order.id}/review`,
        storeName: order.store.name,
      },
    });

    await this.logNotification(orderId, 'DELIVERY_NOTIFICATION', 'email');
  }

  async sendFailureNotification(orderId: string, error?: string): Promise<void> {
    const order = await this.getOrderWithDetails(orderId);

    if (!order) return;

    await this.emailService.send({
      to: order.customer.email,
      template: 'order-issue',
      data: {
        customerName: order.customer.name,
        orderNumber: order.orderNumber,
        message: 'We encountered an issue processing your order and our team is working on it.',
        supportEmail: order.store.supportEmail || 'support@zyenta.com',
        storeName: order.store.name,
      },
    });

    await this.logNotification(orderId, 'FAILURE_NOTIFICATION', 'email');
  }

  async sendStatusUpdate(orderId: string, status: string, message: string): Promise<void> {
    const order = await this.getOrderWithDetails(orderId);

    if (!order) return;

    await this.emailService.send({
      to: order.customer.email,
      template: 'order-status-update',
      data: {
        customerName: order.customer.name,
        orderNumber: order.orderNumber,
        status,
        message,
        orderUrl: `${this.getStoreUrl(order.store)}/orders/${order.id}`,
        storeName: order.store.name,
      },
    });

    await this.logNotification(orderId, 'STATUS_UPDATE', 'email');
  }

  private async getOrderWithDetails(orderId: string) {
    return this.prisma.order.findUnique({
      where: { id: orderId },
      include: {
        items: {
          include: {
            product: true,
          },
        },
        store: true,
        customer: true,
      },
    });
  }

  private async logNotification(
    orderId: string,
    type: string,
    channel: string,
  ): Promise<void> {
    await this.prisma.orderNotification.create({
      data: {
        orderId,
        type,
        channel,
        sentAt: new Date(),
      },
    });
  }

  private getStoreUrl(store: any): string {
    if (store.customDomain) {
      return `https://${store.customDomain}`;
    }
    return `https://${store.subdomain}.zyenta.com`;
  }

  private buildTrackingUrl(trackingNumber: string, carrier: string): string {
    return `https://www.17track.net/en/track?nums=${trackingNumber}`;
  }
}
```

### 4.3 Email Templates

```handlebars
<!-- services/ops-engine/src/modules/notifications/templates/order-confirmation.hbs -->
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Order Confirmation</title>
</head>
<body style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; background-color: #f5f5f5; margin: 0; padding: 20px;">
  <div style="max-width: 600px; margin: 0 auto; background-color: #ffffff; border-radius: 8px; overflow: hidden;">
    <div style="background-color: #1a1a1a; padding: 24px; text-align: center;">
      <h1 style="color: #ffffff; margin: 0; font-size: 24px;">{{storeName}}</h1>
    </div>

    <div style="padding: 32px 24px;">
      <h2 style="margin: 0 0 16px 0; color: #1a1a1a;">Thanks for your order!</h2>
      <p style="color: #4b5563; line-height: 1.6;">
        Hi {{customerName}}, we've received your order and are getting it ready.
      </p>

      <div style="background-color: #f9fafb; border-radius: 8px; padding: 16px; margin: 24px 0;">
        <p style="margin: 0; color: #6b7280; font-size: 14px;">Order Number</p>
        <p style="margin: 4px 0 0 0; color: #1a1a1a; font-size: 18px; font-weight: 600;">{{orderNumber}}</p>
      </div>

      <h3 style="margin: 24px 0 16px 0; color: #1a1a1a;">Order Summary</h3>

      {{#each items}}
      <div style="display: flex; padding: 16px 0; border-bottom: 1px solid #e5e7eb;">
        {{#if image}}
        <img src="{{image}}" alt="{{name}}" style="width: 80px; height: 80px; object-fit: cover; border-radius: 4px;">
        {{/if}}
        <div style="margin-left: 16px; flex: 1;">
          <p style="margin: 0; color: #1a1a1a; font-weight: 500;">{{name}}</p>
          <p style="margin: 4px 0 0 0; color: #6b7280; font-size: 14px;">Qty: {{quantity}}</p>
        </div>
        <p style="margin: 0; color: #1a1a1a; font-weight: 500;">${{price}}</p>
      </div>
      {{/each}}

      <div style="padding: 16px 0; border-bottom: 1px solid #e5e7eb;">
        <div style="display: flex; justify-content: space-between;">
          <span style="color: #6b7280;">Subtotal</span>
          <span style="color: #1a1a1a;">${{subtotal}}</span>
        </div>
        <div style="display: flex; justify-content: space-between; margin-top: 8px;">
          <span style="color: #6b7280;">Shipping</span>
          <span style="color: #1a1a1a;">${{shipping}}</span>
        </div>
      </div>

      <div style="display: flex; justify-content: space-between; padding: 16px 0;">
        <span style="color: #1a1a1a; font-weight: 600; font-size: 18px;">Total</span>
        <span style="color: #1a1a1a; font-weight: 600; font-size: 18px;">${{total}}</span>
      </div>

      <h3 style="margin: 24px 0 16px 0; color: #1a1a1a;">Shipping Address</h3>
      <p style="color: #4b5563; line-height: 1.6; margin: 0;">
        {{shippingAddress.name}}<br>
        {{shippingAddress.line1}}<br>
        {{#if shippingAddress.line2}}{{shippingAddress.line2}}<br>{{/if}}
        {{shippingAddress.city}}, {{shippingAddress.state}} {{shippingAddress.postalCode}}<br>
        {{shippingAddress.country}}
      </p>
    </div>

    <div style="background-color: #f9fafb; padding: 24px; text-align: center;">
      <p style="margin: 0; color: #6b7280; font-size: 14px;">
        Questions? Contact us at support@{{storeName}}.com
      </p>
    </div>
  </div>
</body>
</html>
```

```handlebars
<!-- services/ops-engine/src/modules/notifications/templates/order-shipped.hbs -->
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Your Order Has Shipped</title>
</head>
<body style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; background-color: #f5f5f5; margin: 0; padding: 20px;">
  <div style="max-width: 600px; margin: 0 auto; background-color: #ffffff; border-radius: 8px; overflow: hidden;">
    <div style="background-color: #1a1a1a; padding: 24px; text-align: center;">
      <h1 style="color: #ffffff; margin: 0; font-size: 24px;">{{storeName}}</h1>
    </div>

    <div style="padding: 32px 24px;">
      <div style="text-align: center; margin-bottom: 24px;">
        <div style="width: 64px; height: 64px; background-color: #10b981; border-radius: 50%; margin: 0 auto 16px; display: flex; align-items: center; justify-content: center;">
          <span style="color: white; font-size: 32px;">📦</span>
        </div>
        <h2 style="margin: 0; color: #1a1a1a;">Your order is on its way!</h2>
      </div>

      <p style="color: #4b5563; line-height: 1.6; text-align: center;">
        Hi {{customerName}}, your order has shipped and is on its way to you.
      </p>

      <div style="background-color: #f9fafb; border-radius: 8px; padding: 16px; margin: 24px 0; text-align: center;">
        <p style="margin: 0; color: #6b7280; font-size: 14px;">Tracking Number</p>
        <p style="margin: 8px 0; color: #1a1a1a; font-size: 18px; font-weight: 600;">{{trackingNumber}}</p>
        <p style="margin: 0; color: #6b7280; font-size: 14px;">via {{trackingCarrier}}</p>
      </div>

      <div style="text-align: center; margin: 24px 0;">
        <a href="{{trackingUrl}}" style="display: inline-block; background-color: #1a1a1a; color: #ffffff; text-decoration: none; padding: 12px 24px; border-radius: 6px; font-weight: 500;">
          Track Your Package
        </a>
      </div>

      {{#if estimatedDelivery}}
      <div style="text-align: center; margin: 24px 0;">
        <p style="margin: 0; color: #6b7280; font-size: 14px;">Estimated Delivery</p>
        <p style="margin: 4px 0 0 0; color: #1a1a1a; font-size: 16px; font-weight: 500;">{{estimatedDelivery}}</p>
      </div>
      {{/if}}

      <h3 style="margin: 24px 0 16px 0; color: #1a1a1a;">Items in This Shipment</h3>

      {{#each items}}
      <div style="display: flex; padding: 12px 0; border-bottom: 1px solid #e5e7eb;">
        {{#if image}}
        <img src="{{image}}" alt="{{name}}" style="width: 60px; height: 60px; object-fit: cover; border-radius: 4px;">
        {{/if}}
        <div style="margin-left: 12px;">
          <p style="margin: 0; color: #1a1a1a;">{{name}}</p>
          <p style="margin: 4px 0 0 0; color: #6b7280; font-size: 14px;">Qty: {{quantity}}</p>
        </div>
      </div>
      {{/each}}
    </div>

    <div style="background-color: #f9fafb; padding: 24px; text-align: center;">
      <p style="margin: 0; color: #6b7280; font-size: 14px;">
        Questions about your shipment? Contact us at support@{{storeName}}.com
      </p>
    </div>
  </div>
</body>
</html>
```

---

## 5. Fulfillment Controller

```typescript
// services/ops-engine/src/modules/fulfillment/fulfillment.controller.ts
import {
  Controller,
  Get,
  Post,
  Param,
  Body,
  Query,
  HttpCode,
  HttpStatus,
} from '@nestjs/common';
import { FulfillmentService } from './fulfillment.service';
import { TrackingService } from './tracking/tracking.service';

@Controller('fulfillment')
export class FulfillmentController {
  constructor(
    private readonly fulfillmentService: FulfillmentService,
    private readonly trackingService: TrackingService,
  ) {}

  @Get('orders/:orderId/status')
  async getOrderStatus(@Param('orderId') orderId: string) {
    return this.fulfillmentService.getOrderFulfillmentStatus(orderId);
  }

  @Post('orders/:orderId/process')
  @HttpCode(HttpStatus.ACCEPTED)
  async processOrder(@Param('orderId') orderId: string) {
    await this.fulfillmentService.processOrder(orderId);
    return { message: 'Order processing started', orderId };
  }

  @Post('orders/:orderId/cancel')
  async cancelOrder(
    @Param('orderId') orderId: string,
    @Body('reason') reason: string,
  ) {
    await this.fulfillmentService.cancelOrder(orderId, reason);
    return { message: 'Order cancellation initiated', orderId };
  }

  @Get('orders/:orderId/tracking')
  async getTracking(@Param('orderId') orderId: string) {
    const order = await this.fulfillmentService.getOrderFulfillmentStatus(orderId);

    if (!order.trackingNumber) {
      return { status: 'no_tracking', message: 'Tracking not available yet' };
    }

    return this.trackingService.getTrackingInfo(
      order.trackingNumber,
      order.trackingCarrier,
    );
  }

  @Post('orders/:orderId/sync-tracking')
  @HttpCode(HttpStatus.ACCEPTED)
  async syncTracking(@Param('orderId') orderId: string) {
    await this.fulfillmentService.syncTracking(orderId);
    return { message: 'Tracking sync initiated', orderId };
  }

  @Get('tracking/:trackingNumber')
  async getTrackingByNumber(
    @Param('trackingNumber') trackingNumber: string,
    @Query('carrier') carrier?: string,
  ) {
    return this.trackingService.getTrackingInfo(trackingNumber, carrier);
  }
}
```

---

## 6. Database Schema Updates

```prisma
// packages/database/prisma/schema.prisma (additions)

model SupplierOrderLog {
  id              String    @id @default(cuid())
  orderId         String
  order           Order     @relation(fields: [orderId], references: [id], onDelete: Cascade)

  provider        String
  supplierOrderId String?
  success         Boolean
  error           String?   @db.Text

  requestPayload  Json?
  responsePayload Json?

  createdAt       DateTime  @default(now())

  @@index([orderId])
}

model TrackingUpdate {
  id        String    @id @default(cuid())
  orderId   String
  order     Order     @relation(fields: [orderId], references: [id], onDelete: Cascade)

  status    String
  location  String?
  events    Json?

  createdAt DateTime  @default(now())

  @@index([orderId])
}

model OrderNotification {
  id        String    @id @default(cuid())
  orderId   String
  order     Order     @relation(fields: [orderId], references: [id], onDelete: Cascade)

  type      String    // ORDER_CONFIRMATION, SHIPPING_NOTIFICATION, etc.
  channel   String    // email, sms, push
  sentAt    DateTime

  @@index([orderId])
}
```

---

## Verification Checklist

### State Machine
- [ ] Order state machine transitions correctly
- [ ] Events trigger appropriate state changes
- [ ] Actions execute on transitions
- [ ] State is persisted and recoverable

### Supplier Orders
- [ ] AliExpress orders place successfully
- [ ] CJ Dropshipping orders place successfully
- [ ] Order status can be retrieved
- [ ] Order cancellation works

### Tracking
- [ ] 17Track integration works
- [ ] Tracking status updates correctly
- [ ] Tracking events are recorded
- [ ] Carrier detection works

### Notifications
- [ ] Order confirmation emails send
- [ ] Shipping notifications send
- [ ] Delivery notifications send
- [ ] Email templates render correctly

---

## Next Steps

After completing Sprint 3, proceed to:

**Sprint 4: Video Generation**
- Runway ML API integration
- FFmpeg video composition
- ElevenLabs voice-over
- Video generation queue
- Video management UI

---

*Phase 2 Sprint 3 establishes the fulfillment orchestrator with XState-based workflow management, supplier order integration, tracking synchronization, and customer notification system.*
