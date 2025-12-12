# Sprint 6: Integration & Polish - Action Plan

## Overview

This document provides a comprehensive implementation guide for Sprint 6 of Phase 1, focusing on integrating all components, end-to-end testing, error handling improvements, performance optimization, security audit, and beta launch preparation.

### Sprint Objectives
- [ ] Complete end-to-end flow testing
- [ ] Implement comprehensive error handling
- [ ] Optimize performance across all services
- [ ] Create user and developer documentation
- [ ] Conduct security audit and fixes
- [ ] Prepare for beta launch

### Prerequisites
- Sprint 1-5 completed
- All services deployed to staging environment
- Test data seeded in database

---

## 1. End-to-End Flow Testing

### 1.1 Test Scenarios Definition

```typescript
// packages/e2e-tests/src/scenarios/index.ts

export const E2E_SCENARIOS = {
  // Complete store creation flow
  STORE_CREATION: {
    name: 'Complete Store Creation',
    steps: [
      'User registers and logs in',
      'User connects Stripe account',
      'User connects supplier account (AliExpress)',
      'User enters niche and starts Genesis',
      'Genesis Engine creates brand identity',
      'Product Scout finds and imports products',
      'Media Studio processes product images',
      'Copywriter generates all content',
      'Store is deployed and accessible',
    ],
    expectedDuration: '< 15 minutes',
  },

  // Customer purchase flow
  CUSTOMER_PURCHASE: {
    name: 'Customer Purchase Flow',
    steps: [
      'Customer visits storefront',
      'Customer browses products',
      'Customer adds items to cart',
      'Customer proceeds to checkout',
      'Stripe payment processes successfully',
      'Order is created in database',
      'Webhook triggers fulfillment',
      'Order confirmation email sent',
    ],
    expectedDuration: '< 2 minutes',
  },

  // Dashboard management flow
  DASHBOARD_MANAGEMENT: {
    name: 'Dashboard Store Management',
    steps: [
      'User logs into dashboard',
      'User views store overview',
      'User edits product details',
      'User regenerates product images',
      'User updates store settings',
      'Changes reflect on storefront',
    ],
    expectedDuration: '< 5 minutes',
  },
};
```

### 1.2 Playwright E2E Test Setup

```typescript
// packages/e2e-tests/playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [
    ['html', { outputFolder: 'reports/html' }],
    ['json', { outputFile: 'reports/results.json' }],
    ['junit', { outputFile: 'reports/junit.xml' }],
  ],
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
    {
      name: 'mobile-chrome',
      use: { ...devices['Pixel 5'] },
    },
    {
      name: 'mobile-safari',
      use: { ...devices['iPhone 12'] },
    },
  ],
  webServer: [
    {
      command: 'pnpm --filter @zyenta/dashboard dev',
      url: 'http://localhost:3000',
      reuseExistingServer: !process.env.CI,
    },
    {
      command: 'pnpm --filter @zyenta/storefront dev',
      url: 'http://localhost:3001',
      reuseExistingServer: !process.env.CI,
    },
  ],
});
```

### 1.3 Store Creation E2E Test

```typescript
// packages/e2e-tests/tests/store-creation.spec.ts
import { test, expect } from '@playwright/test';
import { generateTestUser, generateTestNiche } from '../utils/generators';

test.describe('Store Creation Flow', () => {
  let testUser: ReturnType<typeof generateTestUser>;

  test.beforeAll(async () => {
    testUser = generateTestUser();
  });

  test('complete store creation from registration to live store', async ({ page }) => {
    // Step 1: Register
    await page.goto('/register');
    await page.fill('[name="email"]', testUser.email);
    await page.fill('[name="password"]', testUser.password);
    await page.fill('[name="name"]', testUser.name);
    await page.click('button[type="submit"]');

    await expect(page).toHaveURL('/dashboard');

    // Step 2: Start store creation wizard
    await page.click('text=Create New Store');
    await expect(page).toHaveURL('/stores/new');

    // Step 3: Enter niche
    const niche = generateTestNiche();
    await page.fill('[name="niche"]', niche);
    await page.click('button:has-text("Continue")');

    // Step 4: Skip supplier connection for test (use mock)
    await page.click('button:has-text("Skip for now")');

    // Step 5: Skip Stripe connection for test
    await page.click('button:has-text("Setup Later")');

    // Step 6: Launch Genesis
    await page.click('button:has-text("Create My Store")');

    // Step 7: Wait for progress page
    await expect(page).toHaveURL(/\/stores\/new\/progress/);

    // Step 8: Monitor progress (with timeout)
    await expect(page.locator('[data-testid="progress-brand"]')).toHaveText(
      /completed/i,
      { timeout: 120000 }
    );
    await expect(page.locator('[data-testid="progress-products"]')).toHaveText(
      /completed/i,
      { timeout: 300000 }
    );
    await expect(page.locator('[data-testid="progress-content"]')).toHaveText(
      /completed/i,
      { timeout: 180000 }
    );

    // Step 9: Verify store is accessible
    const storeUrl = await page.locator('[data-testid="store-url"]').textContent();
    expect(storeUrl).toMatch(/\.zyenta\.com$/);

    // Step 10: Visit storefront
    await page.goto(storeUrl!);
    await expect(page.locator('h1')).toContainText(/.+/); // Store has a name
    await expect(page.locator('[data-testid="product-grid"]')).toBeVisible();
  });
});
```

### 1.4 Purchase Flow E2E Test

```typescript
// packages/e2e-tests/tests/purchase-flow.spec.ts
import { test, expect } from '@playwright/test';
import { seedTestStore } from '../utils/seed';

test.describe('Customer Purchase Flow', () => {
  let testStore: Awaited<ReturnType<typeof seedTestStore>>;

  test.beforeAll(async () => {
    testStore = await seedTestStore();
  });

  test('complete purchase from browse to confirmation', async ({ page }) => {
    // Visit storefront
    await page.goto(`https://${testStore.subdomain}.zyenta.com`);

    // Browse products
    await page.click('text=Shop All');
    await expect(page.locator('[data-testid="product-card"]')).toHaveCount(
      testStore.productCount
    );

    // Select first product
    await page.click('[data-testid="product-card"]:first-child');
    await expect(page).toHaveURL(/\/products\/.+/);

    // Add to cart
    await page.click('button:has-text("Add to Cart")');
    await expect(page.locator('[data-testid="cart-count"]')).toHaveText('1');

    // Open cart
    await page.click('[data-testid="cart-button"]');
    await expect(page.locator('[data-testid="cart-drawer"]')).toBeVisible();

    // Go to checkout
    await page.click('text=Checkout');
    await expect(page).toHaveURL('/checkout');

    // In test mode, Stripe checkout is mocked
    await page.click('button:has-text("Proceed to Payment")');

    // Mock Stripe redirect and success
    await page.goto('/checkout/success?session_id=test_session');

    // Verify success page
    await expect(page.locator('h1')).toContainText('Thank You');
  });
});
```

### 1.5 API Integration Tests

```typescript
// packages/e2e-tests/tests/api-integration.spec.ts
import { test, expect } from '@playwright/test';

const GENESIS_API = process.env.GENESIS_API_URL || 'http://localhost:8000';
const MEDIA_API = process.env.MEDIA_API_URL || 'http://localhost:8001';

test.describe('API Integration Tests', () => {
  test('Genesis Engine health check', async ({ request }) => {
    const response = await request.get(`${GENESIS_API}/health`);
    expect(response.ok()).toBeTruthy();
    const data = await response.json();
    expect(data.status).toBe('healthy');
    expect(data.services.database).toBe('connected');
    expect(data.services.redis).toBe('connected');
  });

  test('Media Studio health check', async ({ request }) => {
    const response = await request.get(`${MEDIA_API}/health`);
    expect(response.ok()).toBeTruthy();
    const data = await response.json();
    expect(data.status).toBe('healthy');
    expect(data.services.storage).toBe('connected');
  });

  test('Genesis to Media Studio integration', async ({ request }) => {
    // Create a test job that requires image processing
    const jobResponse = await request.post(`${GENESIS_API}/api/v1/test/integration`, {
      data: {
        type: 'IMAGE_PROCESSING',
        imageUrl: 'https://example.com/test-image.jpg',
      },
    });

    expect(jobResponse.ok()).toBeTruthy();
    const job = await jobResponse.json();

    // Poll for completion
    let status = 'PENDING';
    let attempts = 0;
    while (status !== 'COMPLETED' && attempts < 30) {
      await new Promise((r) => setTimeout(r, 2000));
      const statusResponse = await request.get(
        `${GENESIS_API}/api/v1/jobs/${job.id}`
      );
      const statusData = await statusResponse.json();
      status = statusData.status;
      attempts++;
    }

    expect(status).toBe('COMPLETED');
  });
});
```

---

## 2. Error Handling Improvements

### 2.1 Global Error Handler (Dashboard)

```typescript
// apps/dashboard/src/components/error-boundary.tsx
'use client';

import { Component, ReactNode } from 'react';
import { AlertTriangle, RefreshCw, Home } from 'lucide-react';
import { Button } from '@/components/ui/button';

interface Props {
  children: ReactNode;
  fallback?: ReactNode;
}

interface State {
  hasError: boolean;
  error: Error | null;
  errorInfo: React.ErrorInfo | null;
}

export class ErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { hasError: false, error: null, errorInfo: null };
  }

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error, errorInfo: null };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    // Log to error tracking service
    this.logError(error, errorInfo);
    this.setState({ errorInfo });
  }

  private logError(error: Error, errorInfo: React.ErrorInfo) {
    // Send to error tracking service (e.g., Sentry)
    console.error('Error caught by boundary:', error, errorInfo);

    if (typeof window !== 'undefined' && (window as any).Sentry) {
      (window as any).Sentry.captureException(error, {
        extra: { componentStack: errorInfo.componentStack },
      });
    }
  }

  private handleReset = () => {
    this.setState({ hasError: false, error: null, errorInfo: null });
  };

  render() {
    if (this.state.hasError) {
      if (this.props.fallback) {
        return this.props.fallback;
      }

      return (
        <div className="min-h-[400px] flex items-center justify-center p-8">
          <div className="text-center max-w-md">
            <div className="w-16 h-16 bg-red-100 rounded-full flex items-center justify-center mx-auto mb-6">
              <AlertTriangle className="w-8 h-8 text-red-500" />
            </div>
            <h2 className="text-xl font-bold mb-2">Something went wrong</h2>
            <p className="text-muted-foreground mb-6">
              We're sorry, but something unexpected happened. Our team has been
              notified and is working on a fix.
            </p>
            {process.env.NODE_ENV === 'development' && this.state.error && (
              <pre className="text-left text-xs bg-muted p-4 rounded-lg mb-6 overflow-auto max-h-40">
                {this.state.error.message}
                {this.state.errorInfo?.componentStack}
              </pre>
            )}
            <div className="flex gap-4 justify-center">
              <Button variant="outline" onClick={this.handleReset}>
                <RefreshCw className="w-4 h-4 mr-2" />
                Try Again
              </Button>
              <Button onClick={() => (window.location.href = '/dashboard')}>
                <Home className="w-4 h-4 mr-2" />
                Go Home
              </Button>
            </div>
          </div>
        </div>
      );
    }

    return this.props.children;
  }
}
```

### 2.2 API Error Handler (Genesis Engine)

```python
# services/genesis-engine/app/core/error_handlers.py
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from pydantic import ValidationError
import traceback
import logging
from typing import Union
import sentry_sdk

logger = logging.getLogger(__name__)

class APIError(Exception):
    """Base API Error"""
    def __init__(
        self,
        message: str,
        status_code: int = 500,
        error_code: str = "INTERNAL_ERROR",
        details: dict = None
    ):
        self.message = message
        self.status_code = status_code
        self.error_code = error_code
        self.details = details or {}
        super().__init__(message)


class NotFoundError(APIError):
    def __init__(self, resource: str, identifier: str):
        super().__init__(
            message=f"{resource} not found: {identifier}",
            status_code=404,
            error_code="NOT_FOUND",
            details={"resource": resource, "identifier": identifier}
        )


class ValidationError(APIError):
    def __init__(self, message: str, details: dict = None):
        super().__init__(
            message=message,
            status_code=400,
            error_code="VALIDATION_ERROR",
            details=details
        )


class ServiceUnavailableError(APIError):
    def __init__(self, service: str, message: str = None):
        super().__init__(
            message=message or f"{service} is currently unavailable",
            status_code=503,
            error_code="SERVICE_UNAVAILABLE",
            details={"service": service}
        )


class RateLimitError(APIError):
    def __init__(self, retry_after: int = 60):
        super().__init__(
            message="Rate limit exceeded",
            status_code=429,
            error_code="RATE_LIMIT_EXCEEDED",
            details={"retry_after": retry_after}
        )


def setup_error_handlers(app: FastAPI):
    """Configure error handlers for the FastAPI application"""

    @app.exception_handler(APIError)
    async def api_error_handler(request: Request, exc: APIError):
        logger.error(
            f"API Error: {exc.error_code} - {exc.message}",
            extra={
                "path": request.url.path,
                "method": request.method,
                "error_code": exc.error_code,
                "details": exc.details
            }
        )
        return JSONResponse(
            status_code=exc.status_code,
            content={
                "error": {
                    "code": exc.error_code,
                    "message": exc.message,
                    "details": exc.details
                }
            }
        )

    @app.exception_handler(RequestValidationError)
    async def validation_exception_handler(
        request: Request,
        exc: RequestValidationError
    ):
        errors = []
        for error in exc.errors():
            errors.append({
                "field": ".".join(str(loc) for loc in error["loc"]),
                "message": error["msg"],
                "type": error["type"]
            })

        return JSONResponse(
            status_code=422,
            content={
                "error": {
                    "code": "VALIDATION_ERROR",
                    "message": "Request validation failed",
                    "details": {"errors": errors}
                }
            }
        )

    @app.exception_handler(HTTPException)
    async def http_exception_handler(request: Request, exc: HTTPException):
        return JSONResponse(
            status_code=exc.status_code,
            content={
                "error": {
                    "code": "HTTP_ERROR",
                    "message": exc.detail,
                    "details": {}
                }
            }
        )

    @app.exception_handler(Exception)
    async def unhandled_exception_handler(request: Request, exc: Exception):
        # Log full traceback
        logger.exception(
            "Unhandled exception",
            extra={
                "path": request.url.path,
                "method": request.method,
            }
        )

        # Report to Sentry
        sentry_sdk.capture_exception(exc)

        # Don't expose internal errors in production
        message = "An unexpected error occurred"
        if app.debug:
            message = str(exc)

        return JSONResponse(
            status_code=500,
            content={
                "error": {
                    "code": "INTERNAL_ERROR",
                    "message": message,
                    "details": {}
                }
            }
        )
```

### 2.3 Retry Logic for External Services

```typescript
// packages/shared-utils/src/retry.ts

export interface RetryOptions {
  maxAttempts: number;
  initialDelayMs: number;
  maxDelayMs: number;
  backoffMultiplier: number;
  retryableErrors?: string[];
  onRetry?: (error: Error, attempt: number) => void;
}

const DEFAULT_OPTIONS: RetryOptions = {
  maxAttempts: 3,
  initialDelayMs: 1000,
  maxDelayMs: 30000,
  backoffMultiplier: 2,
};

export async function withRetry<T>(
  fn: () => Promise<T>,
  options: Partial<RetryOptions> = {}
): Promise<T> {
  const opts = { ...DEFAULT_OPTIONS, ...options };
  let lastError: Error;
  let delay = opts.initialDelayMs;

  for (let attempt = 1; attempt <= opts.maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;

      // Check if error is retryable
      if (opts.retryableErrors && opts.retryableErrors.length > 0) {
        const isRetryable = opts.retryableErrors.some(
          (code) =>
            lastError.message.includes(code) ||
            (lastError as any).code === code
        );
        if (!isRetryable) {
          throw lastError;
        }
      }

      // Don't retry on last attempt
      if (attempt === opts.maxAttempts) {
        break;
      }

      // Notify retry callback
      if (opts.onRetry) {
        opts.onRetry(lastError, attempt);
      }

      // Wait before retrying
      await sleep(delay);

      // Increase delay for next attempt
      delay = Math.min(delay * opts.backoffMultiplier, opts.maxDelayMs);
    }
  }

  throw lastError!;
}

function sleep(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

// Usage example for API calls
export async function fetchWithRetry(
  url: string,
  options: RequestInit = {}
): Promise<Response> {
  return withRetry(
    async () => {
      const response = await fetch(url, options);

      // Retry on 5xx errors
      if (response.status >= 500) {
        throw new Error(`Server error: ${response.status}`);
      }

      // Retry on rate limits (with longer delay)
      if (response.status === 429) {
        const retryAfter = response.headers.get('Retry-After');
        if (retryAfter) {
          await sleep(parseInt(retryAfter) * 1000);
        }
        throw new Error('Rate limited');
      }

      return response;
    },
    {
      maxAttempts: 3,
      initialDelayMs: 1000,
      retryableErrors: ['ECONNRESET', 'ETIMEDOUT', 'Server error', 'Rate limited'],
    }
  );
}
```

### 2.4 User-Friendly Error Messages

```typescript
// packages/shared-utils/src/error-messages.ts

export const ERROR_MESSAGES: Record<string, string> = {
  // Authentication
  AUTH_INVALID_CREDENTIALS: 'Invalid email or password. Please try again.',
  AUTH_EMAIL_EXISTS: 'An account with this email already exists.',
  AUTH_SESSION_EXPIRED: 'Your session has expired. Please log in again.',
  AUTH_UNAUTHORIZED: 'You do not have permission to perform this action.',

  // Store
  STORE_NOT_FOUND: 'Store not found. It may have been deleted or does not exist.',
  STORE_CREATION_FAILED: 'Failed to create store. Please try again later.',
  STORE_LIMIT_REACHED: 'You have reached the maximum number of stores for your plan.',

  // Products
  PRODUCT_NOT_FOUND: 'Product not found.',
  PRODUCT_OUT_OF_STOCK: 'This product is currently out of stock.',
  PRODUCT_IMPORT_FAILED: 'Failed to import product. The supplier may be unavailable.',

  // Payments
  PAYMENT_FAILED: 'Payment failed. Please check your card details and try again.',
  PAYMENT_STRIPE_NOT_CONNECTED: 'Please connect your Stripe account to accept payments.',
  PAYMENT_INSUFFICIENT_FUNDS: 'Payment declined due to insufficient funds.',

  // Genesis
  GENESIS_FAILED: 'Store generation failed. Our team has been notified.',
  GENESIS_TIMEOUT: 'Store generation is taking longer than expected. We will email you when it is ready.',

  // Media
  MEDIA_PROCESSING_FAILED: 'Failed to process image. Please try with a different image.',
  MEDIA_UNSUPPORTED_FORMAT: 'Unsupported image format. Please use JPG, PNG, or WebP.',
  MEDIA_FILE_TOO_LARGE: 'File is too large. Maximum size is 10MB.',

  // Network
  NETWORK_ERROR: 'Unable to connect to the server. Please check your internet connection.',
  SERVER_ERROR: 'Something went wrong on our end. Please try again later.',
  TIMEOUT_ERROR: 'The request timed out. Please try again.',

  // Rate Limiting
  RATE_LIMIT_EXCEEDED: 'Too many requests. Please wait a moment and try again.',

  // Default
  UNKNOWN_ERROR: 'An unexpected error occurred. Please try again.',
};

export function getUserFriendlyMessage(errorCode: string): string {
  return ERROR_MESSAGES[errorCode] || ERROR_MESSAGES.UNKNOWN_ERROR;
}

export function formatAPIError(error: any): string {
  if (error?.response?.data?.error?.code) {
    return getUserFriendlyMessage(error.response.data.error.code);
  }

  if (error?.code === 'ECONNREFUSED' || error?.code === 'ENOTFOUND') {
    return ERROR_MESSAGES.NETWORK_ERROR;
  }

  if (error?.message?.includes('timeout')) {
    return ERROR_MESSAGES.TIMEOUT_ERROR;
  }

  return ERROR_MESSAGES.UNKNOWN_ERROR;
}
```

---

## 3. Performance Optimization

### 3.1 Database Query Optimization

```typescript
// packages/database/src/optimization/queries.ts
import { Prisma } from '@prisma/client';

/**
 * Optimized query for fetching store with essential relations
 */
export const getStoreWithProductsQuery = (storeId: string) => ({
  where: { id: storeId },
  include: {
    products: {
      where: { status: 'ACTIVE' },
      take: 20,
      orderBy: { createdAt: 'desc' as const },
      select: {
        id: true,
        title: true,
        slug: true,
        sellingPrice: true,
        compareAtPrice: true,
        stockQuantity: true,
        images: {
          take: 1,
          orderBy: { position: 'asc' as const },
          select: {
            processedUrl: true,
            altText: true,
          },
        },
      },
    },
    _count: {
      select: { products: true },
    },
  },
});

/**
 * Raw query for dashboard analytics (optimized with indexes)
 */
export const getDashboardAnalyticsQuery = (storeId: string, days: number = 30) =>
  Prisma.sql`
    SELECT
      DATE(created_at) as date,
      COUNT(*) as order_count,
      SUM(total_amount) as revenue,
      AVG(total_amount) as avg_order_value
    FROM orders
    WHERE store_id = ${storeId}
      AND created_at >= NOW() - INTERVAL '${days} days'
    GROUP BY DATE(created_at)
    ORDER BY date DESC
  `;

/**
 * Batch upsert for product sync
 */
export const batchUpsertProducts = async (
  prisma: any,
  products: Array<{
    storeId: string;
    sourceId: string;
    data: any;
  }>
) => {
  // Use transaction for atomicity
  return prisma.$transaction(
    products.map((product) =>
      prisma.product.upsert({
        where: {
          storeId_sourceId: {
            storeId: product.storeId,
            sourceId: product.sourceId,
          },
        },
        update: product.data,
        create: {
          storeId: product.storeId,
          sourceId: product.sourceId,
          ...product.data,
        },
      })
    )
  );
};
```

### 3.2 Redis Caching Layer

```typescript
// packages/cache/src/index.ts
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL!);

interface CacheOptions {
  ttl?: number; // Time to live in seconds
  tags?: string[]; // Cache tags for invalidation
}

const DEFAULT_TTL = 3600; // 1 hour

export async function getFromCache<T>(key: string): Promise<T | null> {
  const cached = await redis.get(key);
  if (!cached) return null;

  try {
    return JSON.parse(cached) as T;
  } catch {
    return null;
  }
}

export async function setInCache<T>(
  key: string,
  value: T,
  options: CacheOptions = {}
): Promise<void> {
  const { ttl = DEFAULT_TTL, tags = [] } = options;

  await redis.setex(key, ttl, JSON.stringify(value));

  // Store cache tags for invalidation
  if (tags.length > 0) {
    const tagPipeline = redis.pipeline();
    for (const tag of tags) {
      tagPipeline.sadd(`cache:tag:${tag}`, key);
      tagPipeline.expire(`cache:tag:${tag}`, ttl);
    }
    await tagPipeline.exec();
  }
}

export async function invalidateByTag(tag: string): Promise<void> {
  const keys = await redis.smembers(`cache:tag:${tag}`);
  if (keys.length > 0) {
    await redis.del(...keys, `cache:tag:${tag}`);
  }
}

export async function invalidateByPattern(pattern: string): Promise<void> {
  const keys = await redis.keys(pattern);
  if (keys.length > 0) {
    await redis.del(...keys);
  }
}

// Decorator for caching function results
export function cached<T>(options: CacheOptions & { keyPrefix: string }) {
  return function (
    target: any,
    propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    const originalMethod = descriptor.value;

    descriptor.value = async function (...args: any[]) {
      const cacheKey = `${options.keyPrefix}:${JSON.stringify(args)}`;

      const cached = await getFromCache<T>(cacheKey);
      if (cached !== null) {
        return cached;
      }

      const result = await originalMethod.apply(this, args);
      await setInCache(cacheKey, result, options);
      return result;
    };

    return descriptor;
  };
}

// Usage in services
export class StoreService {
  @cached({ keyPrefix: 'store', ttl: 300, tags: ['stores'] })
  async getStoreBySlug(slug: string) {
    // Database query
  }
}
```

### 3.3 Image Optimization Pipeline

```python
# services/media-studio/app/optimization/images.py
from PIL import Image
import io
from typing import Tuple, Optional
import aiohttp
import asyncio
from dataclasses import dataclass


@dataclass
class OptimizedImage:
    data: bytes
    format: str
    width: int
    height: int
    original_size: int
    optimized_size: int


class ImageOptimizer:
    """High-performance image optimization"""

    QUALITY_SETTINGS = {
        "thumbnail": {"max_size": (150, 150), "quality": 70},
        "small": {"max_size": (400, 400), "quality": 80},
        "medium": {"max_size": (800, 800), "quality": 85},
        "large": {"max_size": (1200, 1200), "quality": 90},
        "original": {"max_size": (2000, 2000), "quality": 95},
    }

    def __init__(self):
        self.session: Optional[aiohttp.ClientSession] = None

    async def get_session(self) -> aiohttp.ClientSession:
        if self.session is None or self.session.closed:
            self.session = aiohttp.ClientSession()
        return self.session

    async def download_image(self, url: str) -> bytes:
        """Download image with timeout and retry"""
        session = await self.get_session()
        async with session.get(url, timeout=30) as response:
            response.raise_for_status()
            return await response.read()

    def optimize(
        self,
        image_data: bytes,
        preset: str = "medium",
        output_format: str = "webp"
    ) -> OptimizedImage:
        """Optimize image using preset settings"""
        settings = self.QUALITY_SETTINGS.get(preset, self.QUALITY_SETTINGS["medium"])

        original_size = len(image_data)

        # Open image
        img = Image.open(io.BytesIO(image_data))

        # Convert to RGB if necessary (for WebP support)
        if img.mode in ("RGBA", "P"):
            # Create white background for transparency
            background = Image.new("RGB", img.size, (255, 255, 255))
            if img.mode == "RGBA":
                background.paste(img, mask=img.split()[3])
            else:
                background.paste(img)
            img = background
        elif img.mode != "RGB":
            img = img.convert("RGB")

        # Resize if larger than max size
        max_size = settings["max_size"]
        if img.width > max_size[0] or img.height > max_size[1]:
            img.thumbnail(max_size, Image.LANCZOS)

        # Optimize and save
        output = io.BytesIO()
        save_kwargs = {
            "quality": settings["quality"],
            "optimize": True,
        }

        if output_format == "webp":
            save_kwargs["method"] = 6  # Slower but better compression
            img.save(output, "WebP", **save_kwargs)
        elif output_format == "jpeg":
            save_kwargs["progressive"] = True
            img.save(output, "JPEG", **save_kwargs)
        else:
            img.save(output, output_format.upper(), **save_kwargs)

        optimized_data = output.getvalue()

        return OptimizedImage(
            data=optimized_data,
            format=output_format,
            width=img.width,
            height=img.height,
            original_size=original_size,
            optimized_size=len(optimized_data),
        )

    async def process_batch(
        self,
        urls: list[str],
        preset: str = "medium",
        output_format: str = "webp"
    ) -> list[OptimizedImage]:
        """Process multiple images concurrently"""
        async def process_single(url: str) -> OptimizedImage:
            image_data = await self.download_image(url)
            return self.optimize(image_data, preset, output_format)

        # Process up to 5 images concurrently
        semaphore = asyncio.Semaphore(5)

        async def bounded_process(url: str) -> OptimizedImage:
            async with semaphore:
                return await process_single(url)

        results = await asyncio.gather(
            *[bounded_process(url) for url in urls],
            return_exceptions=True
        )

        return [r for r in results if isinstance(r, OptimizedImage)]
```

### 3.4 Frontend Performance

```typescript
// apps/dashboard/next.config.js
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
});

/** @type {import('next').NextConfig} */
const nextConfig = {
  // Enable React strict mode
  reactStrictMode: true,

  // Optimize images
  images: {
    formats: ['image/avif', 'image/webp'],
    deviceSizes: [640, 750, 828, 1080, 1200, 1920],
    imageSizes: [16, 32, 48, 64, 96, 128, 256],
  },

  // Enable SWC minifier
  swcMinify: true,

  // Optimize chunks
  experimental: {
    optimizePackageImports: [
      'lucide-react',
      '@radix-ui/react-icons',
      'date-fns',
    ],
  },

  // Configure webpack
  webpack: (config, { dev, isServer }) => {
    // Reduce bundle size in production
    if (!dev && !isServer) {
      config.optimization.splitChunks = {
        chunks: 'all',
        minSize: 20000,
        maxSize: 244000,
        cacheGroups: {
          vendor: {
            test: /[\\/]node_modules[\\/]/,
            name: (module) => {
              const packageName = module.context.match(
                /[\\/]node_modules[\\/](.*?)([\\/]|$)/
              )[1];
              return `vendor.${packageName.replace('@', '')}`;
            },
          },
        },
      };
    }
    return config;
  },

  // Headers for caching
  async headers() {
    return [
      {
        source: '/static/:path*',
        headers: [
          {
            key: 'Cache-Control',
            value: 'public, max-age=31536000, immutable',
          },
        ],
      },
      {
        source: '/:path*.svg',
        headers: [
          {
            key: 'Cache-Control',
            value: 'public, max-age=31536000, immutable',
          },
        ],
      },
    ];
  },
};

module.exports = withBundleAnalyzer(nextConfig);
```

---

## 4. Documentation

### 4.1 API Documentation (OpenAPI)

```python
# services/genesis-engine/app/main.py
from fastapi import FastAPI
from fastapi.openapi.utils import get_openapi

app = FastAPI(
    title="Zyenta Genesis Engine API",
    description="""
## Genesis Engine API

The Genesis Engine is responsible for creating new stores autonomously.

### Features
- **Brand Generation**: Creates brand identity from a niche
- **Product Scouting**: Discovers winning products from suppliers
- **Content Generation**: Writes SEO-optimized copy

### Authentication
All endpoints require a valid Bearer token in the Authorization header.
    """,
    version="1.0.0",
    contact={
        "name": "Zyenta Support",
        "email": "support@zyenta.com",
    },
    license_info={
        "name": "MIT",
        "url": "https://opensource.org/licenses/MIT",
    },
)


def custom_openapi():
    if app.openapi_schema:
        return app.openapi_schema

    openapi_schema = get_openapi(
        title=app.title,
        version=app.version,
        description=app.description,
        routes=app.routes,
    )

    # Add security schemes
    openapi_schema["components"]["securitySchemes"] = {
        "bearerAuth": {
            "type": "http",
            "scheme": "bearer",
            "bearerFormat": "JWT",
        }
    }

    # Apply security globally
    openapi_schema["security"] = [{"bearerAuth": []}]

    app.openapi_schema = openapi_schema
    return app.openapi_schema


app.openapi = custom_openapi
```

### 4.2 User Documentation Structure

```markdown
<!-- docs/user-guide/README.md -->

# Zyenta User Guide

Welcome to Zyenta! This guide will help you get started with building your
autonomous e-commerce store.

## Table of Contents

1. [Getting Started](./getting-started.md)
   - Account Setup
   - Connecting Payment Providers
   - Connecting Suppliers

2. [Creating Your First Store](./creating-store.md)
   - Choosing Your Niche
   - The Genesis Process
   - Understanding Your Dashboard

3. [Managing Products](./managing-products.md)
   - Product Overview
   - Editing Products
   - Regenerating Images
   - Pricing Strategies

4. [Store Customization](./customization.md)
   - Brand Settings
   - Theme Options
   - Custom Domains

5. [Payments & Billing](./payments.md)
   - Stripe Connect Setup
   - Understanding Fees
   - Payouts

6. [Marketing Integration](./marketing.md)
   - Social Media Connections
   - Ad Platform Setup
   - Content Calendar

7. [Analytics & Reports](./analytics.md)
   - Dashboard Overview
   - Sales Reports
   - Traffic Analytics

8. [Troubleshooting](./troubleshooting.md)
   - Common Issues
   - Getting Support

9. [FAQ](./faq.md)
```

### 4.3 Developer Documentation

```markdown
<!-- docs/developer-guide/README.md -->

# Zyenta Developer Guide

Technical documentation for developers working on Zyenta.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         Load Balancer                           │
└─────────────────────┬───────────────────────────────────────────┘
                      │
         ┌────────────┼────────────┐
         │            │            │
         ▼            ▼            ▼
    ┌─────────┐  ┌─────────┐  ┌─────────┐
    │Dashboard│  │Storefront│  │   API   │
    │(Next.js)│  │(Next.js) │  │(FastAPI)│
    └────┬────┘  └────┬────┘  └────┬────┘
         │            │            │
         └────────────┼────────────┘
                      │
         ┌────────────┼────────────┐
         │            │            │
         ▼            ▼            ▼
    ┌─────────┐  ┌─────────┐  ┌─────────┐
    │ Genesis │  │  Media  │  │   Ops   │
    │ Engine  │  │ Studio  │  │ Engine  │
    └────┬────┘  └────┬────┘  └────┬────┘
         │            │            │
         └────────────┼────────────┘
                      │
    ┌─────────────────┼─────────────────┐
    │                 │                 │
    ▼                 ▼                 ▼
┌───────┐       ┌─────────┐       ┌─────────┐
│Postgres│       │  Redis  │       │ RabbitMQ│
└───────┘       └─────────┘       └─────────┘
```

## Local Development Setup

### Prerequisites
- Node.js 20+
- Python 3.11+
- Docker & Docker Compose
- pnpm 8+

### Quick Start

```bash
# Clone the repository
git clone https://github.com/zyenta/zyenta.git
cd zyenta

# Install dependencies
pnpm install

# Start infrastructure
docker-compose -f infra/docker/docker-compose.dev.yml up -d

# Run database migrations
pnpm db:migrate

# Seed development data
pnpm db:seed

# Start all services
pnpm dev
```

## Service Documentation

- [Genesis Engine](./services/genesis-engine.md)
- [Media Studio](./services/media-studio.md)
- [Ops Engine](./services/ops-engine.md)
- [Growth Engine](./services/growth-engine.md)

## API Reference

- [REST API](./api/rest.md)
- [WebSocket Events](./api/websocket.md)
- [Webhooks](./api/webhooks.md)

## Contributing

See [CONTRIBUTING.md](../CONTRIBUTING.md) for guidelines.
```

---

## 5. Security Audit

### 5.1 Security Checklist

```markdown
<!-- docs/security/AUDIT_CHECKLIST.md -->

# Security Audit Checklist

## Authentication & Authorization

- [ ] JWT tokens have appropriate expiration times
- [ ] Refresh token rotation is implemented
- [ ] Password requirements meet security standards
- [ ] Rate limiting on authentication endpoints
- [ ] Account lockout after failed attempts
- [ ] Session invalidation on password change
- [ ] Multi-factor authentication option available

## Data Protection

- [ ] All sensitive data encrypted at rest
- [ ] TLS 1.3 enforced for all connections
- [ ] API keys stored encrypted in database
- [ ] No sensitive data in logs
- [ ] PII data properly anonymized in analytics
- [ ] Proper key rotation procedures

## API Security

- [ ] Input validation on all endpoints
- [ ] SQL injection protection (Prisma ORM)
- [ ] XSS prevention headers set
- [ ] CORS properly configured
- [ ] Rate limiting implemented
- [ ] Request size limits enforced
- [ ] File upload validation

## Infrastructure

- [ ] Firewall rules properly configured
- [ ] Database not publicly accessible
- [ ] Redis not publicly accessible
- [ ] Secrets managed via environment/vault
- [ ] Container images scanned for vulnerabilities
- [ ] Kubernetes RBAC configured

## Compliance

- [ ] GDPR data deletion capability
- [ ] Privacy policy up to date
- [ ] Cookie consent implemented
- [ ] Data export capability
- [ ] Audit logs for sensitive actions

## Third-Party

- [ ] Stripe webhook signature verification
- [ ] OAuth state parameter validation
- [ ] Supplier API keys properly scoped
- [ ] Dependencies regularly updated
```

### 5.2 Security Headers Configuration

```typescript
// apps/dashboard/src/middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const response = NextResponse.next();

  // Security headers
  const headers = response.headers;

  // Prevent clickjacking
  headers.set('X-Frame-Options', 'DENY');

  // Prevent MIME sniffing
  headers.set('X-Content-Type-Options', 'nosniff');

  // Enable XSS filter
  headers.set('X-XSS-Protection', '1; mode=block');

  // Referrer policy
  headers.set('Referrer-Policy', 'strict-origin-when-cross-origin');

  // Permissions policy
  headers.set(
    'Permissions-Policy',
    'camera=(), microphone=(), geolocation=(), interest-cohort=()'
  );

  // Content Security Policy
  const csp = [
    "default-src 'self'",
    "script-src 'self' 'unsafe-inline' 'unsafe-eval' https://js.stripe.com",
    "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com",
    "img-src 'self' blob: data: https://cdn.zyenta.com https://*.stripe.com",
    "font-src 'self' https://fonts.gstatic.com",
    "connect-src 'self' https://api.zyenta.com https://api.stripe.com",
    "frame-src https://js.stripe.com https://hooks.stripe.com",
    "object-src 'none'",
    "base-uri 'self'",
    "form-action 'self'",
    "frame-ancestors 'none'",
    "upgrade-insecure-requests",
  ].join('; ');

  headers.set('Content-Security-Policy', csp);

  // Strict Transport Security
  headers.set(
    'Strict-Transport-Security',
    'max-age=31536000; includeSubDomains; preload'
  );

  return response;
}

export const config = {
  matcher: [
    '/((?!api|_next/static|_next/image|favicon.ico).*)',
  ],
};
```

### 5.3 Vulnerability Scanning Configuration

```yaml
# .github/workflows/security.yml
name: Security Scan

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1' # Weekly on Monday

jobs:
  dependency-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run npm audit
        run: pnpm audit --audit-level=high

      - name: Run Snyk scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high

  docker-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build images
        run: |
          docker build -t zyenta/genesis:scan ./services/genesis-engine
          docker build -t zyenta/media:scan ./services/media-studio

      - name: Run Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'zyenta/genesis:scan'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'

      - name: Upload scan results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

  code-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v2
        with:
          languages: javascript, python

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v2

  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Gitleaks scan
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## 6. Beta Launch Preparation

### 6.1 Launch Checklist

```markdown
<!-- docs/launch/BETA_CHECKLIST.md -->

# Beta Launch Checklist

## Pre-Launch (2 weeks before)

### Infrastructure
- [ ] Production environment fully deployed
- [ ] Database backups configured and tested
- [ ] Monitoring and alerting set up
- [ ] Load testing completed
- [ ] CDN configured and caching optimized
- [ ] SSL certificates installed
- [ ] DNS configured properly

### Application
- [ ] All critical bugs fixed
- [ ] Performance benchmarks met
- [ ] Error tracking (Sentry) configured
- [ ] Analytics (Posthog/Mixpanel) set up
- [ ] Feature flags for gradual rollout
- [ ] Maintenance mode page ready

### Security
- [ ] Security audit completed
- [ ] Penetration testing done
- [ ] All vulnerabilities addressed
- [ ] Rate limiting configured
- [ ] DDoS protection enabled

### Legal & Compliance
- [ ] Terms of Service finalized
- [ ] Privacy Policy finalized
- [ ] Cookie consent implemented
- [ ] GDPR compliance verified
- [ ] Payment processing compliance (PCI)

### Documentation
- [ ] User documentation complete
- [ ] API documentation complete
- [ ] FAQ prepared
- [ ] Support articles written

### Support
- [ ] Support email configured
- [ ] Help desk system ready
- [ ] Knowledge base populated
- [ ] Support team trained
- [ ] Escalation procedures defined

## Launch Day

### Communication
- [ ] Beta users notified
- [ ] Launch announcement prepared
- [ ] Social media posts scheduled
- [ ] Press release ready (if applicable)

### Monitoring
- [ ] All team members on standby
- [ ] Communication channel (Slack/Discord) active
- [ ] Monitoring dashboards visible
- [ ] Incident response plan ready

### Rollout
- [ ] Feature flags configured for gradual rollout
- [ ] First batch of users enabled (10%)
- [ ] Monitor for 2 hours
- [ ] Expand to 50% if stable
- [ ] Full rollout after 24 hours if stable

## Post-Launch

### Week 1
- [ ] Daily check-ins with support team
- [ ] Bug triage and prioritization
- [ ] User feedback collection
- [ ] Performance monitoring
- [ ] Cost monitoring

### Week 2-4
- [ ] Weekly retrospective meetings
- [ ] Feature prioritization based on feedback
- [ ] Documentation updates
- [ ] Support process refinement
```

### 6.2 Feature Flags Configuration

```typescript
// packages/feature-flags/src/index.ts
import { createClient } from '@vercel/edge-config';

const edgeConfig = createClient(process.env.EDGE_CONFIG);

export interface FeatureFlags {
  // Beta features
  ENABLE_VIDEO_GENERATION: boolean;
  ENABLE_SOCIAL_AGENT: boolean;
  ENABLE_ADS_AGENT: boolean;

  // Rollout percentages
  GENESIS_ROLLOUT_PERCENT: number;
  MEDIA_STUDIO_ROLLOUT_PERCENT: number;

  // Limits
  MAX_STORES_PER_USER: number;
  MAX_PRODUCTS_PER_STORE: number;

  // Maintenance
  MAINTENANCE_MODE: boolean;
  MAINTENANCE_MESSAGE: string;
}

const DEFAULT_FLAGS: FeatureFlags = {
  ENABLE_VIDEO_GENERATION: false,
  ENABLE_SOCIAL_AGENT: false,
  ENABLE_ADS_AGENT: false,
  GENESIS_ROLLOUT_PERCENT: 100,
  MEDIA_STUDIO_ROLLOUT_PERCENT: 100,
  MAX_STORES_PER_USER: 3,
  MAX_PRODUCTS_PER_STORE: 50,
  MAINTENANCE_MODE: false,
  MAINTENANCE_MESSAGE: '',
};

export async function getFeatureFlags(): Promise<FeatureFlags> {
  try {
    const flags = await edgeConfig.getAll<FeatureFlags>();
    return { ...DEFAULT_FLAGS, ...flags };
  } catch {
    return DEFAULT_FLAGS;
  }
}

export async function isFeatureEnabled(
  feature: keyof FeatureFlags,
  userId?: string
): Promise<boolean> {
  const flags = await getFeatureFlags();
  const value = flags[feature];

  if (typeof value === 'boolean') {
    return value;
  }

  // For percentage-based rollout
  if (typeof value === 'number' && userId) {
    const hash = hashUserId(userId);
    return hash < value;
  }

  return false;
}

function hashUserId(userId: string): number {
  let hash = 0;
  for (let i = 0; i < userId.length; i++) {
    hash = ((hash << 5) - hash) + userId.charCodeAt(i);
    hash = hash & hash; // Convert to 32-bit integer
  }
  return Math.abs(hash) % 100;
}

// Usage in API routes
export function withFeatureFlag(feature: keyof FeatureFlags) {
  return async (req: Request, ctx: any) => {
    const enabled = await isFeatureEnabled(feature, ctx.userId);
    if (!enabled) {
      return new Response(
        JSON.stringify({ error: 'Feature not available' }),
        { status: 403 }
      );
    }
    return ctx.next();
  };
}
```

### 6.3 Monitoring Dashboard Configuration

```yaml
# infra/monitoring/grafana/dashboards/beta-launch.json
{
  "dashboard": {
    "title": "Beta Launch Dashboard",
    "panels": [
      {
        "title": "Active Users",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(increase(http_requests_total{status=~\"2..\"}[5m]))"
          }
        ]
      },
      {
        "title": "Store Creations",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(genesis_stores_created_total[5m]))"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "gauge",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m])) * 100"
          }
        ],
        "thresholds": {
          "mode": "absolute",
          "steps": [
            { "color": "green", "value": null },
            { "color": "yellow", "value": 1 },
            { "color": "red", "value": 5 }
          ]
        }
      },
      {
        "title": "Response Time (p95)",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))"
          }
        ]
      },
      {
        "title": "Genesis Engine Queue",
        "type": "stat",
        "targets": [
          {
            "expr": "genesis_queue_length"
          }
        ]
      },
      {
        "title": "Media Studio Queue",
        "type": "stat",
        "targets": [
          {
            "expr": "media_studio_queue_length"
          }
        ]
      },
      {
        "title": "Payment Success Rate",
        "type": "gauge",
        "targets": [
          {
            "expr": "sum(rate(stripe_payments_successful_total[5m])) / sum(rate(stripe_payments_total[5m])) * 100"
          }
        ]
      },
      {
        "title": "Database Connections",
        "type": "graph",
        "targets": [
          {
            "expr": "pg_stat_activity_count"
          }
        ]
      }
    ]
  }
}
```

---

## Verification Checklist

### End-to-End Testing
- [ ] Store creation flow works completely
- [ ] Customer purchase flow works completely
- [ ] Dashboard management flow works completely
- [ ] All E2E tests pass in CI

### Error Handling
- [ ] Error boundary catches React errors
- [ ] API errors return user-friendly messages
- [ ] Retry logic handles transient failures
- [ ] All error states have proper UI

### Performance
- [ ] Page load time < 3 seconds
- [ ] API response time < 200ms (p95)
- [ ] Database queries optimized
- [ ] Caching reduces load

### Documentation
- [ ] API documentation complete
- [ ] User guide written
- [ ] Developer guide written
- [ ] Support articles prepared

### Security
- [ ] Security audit completed
- [ ] No critical vulnerabilities
- [ ] Headers properly configured
- [ ] Secrets management in place

### Launch Readiness
- [ ] All checklist items completed
- [ ] Team trained and ready
- [ ] Monitoring active
- [ ] Rollback plan documented

---

## Rollback Procedures

### Database Rollback
```bash
# List available backups
aws rds describe-db-snapshots --db-instance-identifier zyenta-prod

# Restore from snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier zyenta-prod-restored \
  --db-snapshot-identifier zyenta-prod-snapshot-YYYYMMDD

# Update connection string and deploy
```

### Application Rollback
```bash
# Using Kubernetes
kubectl rollout undo deployment/genesis-engine -n production
kubectl rollout undo deployment/media-studio -n production
kubectl rollout undo deployment/dashboard -n production

# Verify rollback
kubectl rollout status deployment/genesis-engine -n production
```

### Feature Flag Rollback
```bash
# Disable problematic feature
curl -X PATCH https://api.vercel.com/v1/edge-config \
  -H "Authorization: Bearer $VERCEL_TOKEN" \
  -d '{"items": [{"key": "ENABLE_VIDEO_GENERATION", "value": false}]}'
```

---

*Sprint 6 completes Phase 1 by integrating all components, ensuring quality through comprehensive testing, optimizing performance, conducting security audits, and preparing for beta launch.*
