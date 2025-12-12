# Phase 1: "Creator" MVP - Production Execution Plan

## Overview

This document outlines the complete execution plan for building Zyenta's Phase 1 MVP: **The Creator**. This phase establishes the foundation for the autonomous AI-commerce engine.

### Phase 1 Objectives
- [ ] Build Genesis Engine (Scout, Brand, Copy Agents)
- [ ] Build Basic Media Studio (Images only)
- [ ] Implement MoR Payment Architecture (Client Stripe Connect)
- [ ] Launch "Store Builder" tier

---

## 1. Project Foundation Setup

### 1.1 Monorepo Configuration

**Tooling:** pnpm workspaces + Turborepo

```bash
# Root package.json structure
{
  "name": "zyenta",
  "private": true,
  "workspaces": [
    "apps/*",
    "packages/*",
    "services/*"
  ]
}
```

**Action Items:**
- [ ] Initialize pnpm workspace configuration
- [ ] Setup Turborepo for build orchestration
- [ ] Configure shared TypeScript/Python configs
- [ ] Setup ESLint + Prettier (JS/TS) and Black + Ruff (Python)
- [ ] Configure Husky pre-commit hooks

### 1.2 Directory Structure Implementation

```
zyenta/
├── apps/
│   ├── dashboard/              # Next.js 14 Client Dashboard
│   │   ├── src/
│   │   │   ├── app/            # App Router
│   │   │   ├── components/     # React components
│   │   │   ├── lib/            # Utilities
│   │   │   └── hooks/          # Custom hooks
│   │   ├── package.json
│   │   └── next.config.js
│   │
│   └── storefront/             # Headless Storefront Template
│       ├── src/
│       ├── package.json
│       └── next.config.js
│
├── services/
│   ├── genesis-engine/         # Python FastAPI
│   │   ├── app/
│   │   │   ├── agents/         # Agent implementations
│   │   │   │   ├── brand/
│   │   │   │   ├── scout/
│   │   │   │   └── copywriter/
│   │   │   ├── api/            # API routes
│   │   │   ├── core/           # Config, deps, security
│   │   │   ├── models/         # Pydantic models
│   │   │   └── services/       # Business logic
│   │   ├── requirements.txt
│   │   ├── pyproject.toml
│   │   └── Dockerfile
│   │
│   └── media-studio/           # Python FastAPI
│       ├── app/
│       │   ├── processors/     # Image processing
│       │   ├── api/
│       │   └── services/
│       ├── requirements.txt
│       └── Dockerfile
│
├── packages/
│   ├── database/               # Prisma Schema + Migrations
│   │   ├── prisma/
│   │   │   └── schema.prisma
│   │   └── package.json
│   │
│   ├── queue/                  # Redis/BullMQ Configuration
│   │   ├── src/
│   │   └── package.json
│   │
│   ├── shared-types/           # Shared TypeScript types
│   │   ├── src/
│   │   └── package.json
│   │
│   └── ui/                     # Shared React Components (Shadcn)
│       ├── src/
│       └── package.json
│
├── infra/
│   ├── docker/
│   │   ├── docker-compose.yml
│   │   └── docker-compose.dev.yml
│   ├── k8s/
│   └── terraform/
│
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
│
├── turbo.json
├── pnpm-workspace.yaml
├── package.json
└── README.md
```

---

## 2. Database Schema Design

### 2.1 Core Entities (Prisma Schema)

```prisma
// packages/database/prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ==================== USER & AUTH ====================

model User {
  id              String    @id @default(cuid())
  email           String    @unique
  name            String?
  passwordHash    String?
  emailVerified   DateTime?
  image           String?

  // Stripe Connect
  stripeAccountId String?   @unique
  stripeOnboarded Boolean   @default(false)

  // Relations
  stores          Store[]
  supplierAccounts SupplierAccount[]

  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  @@index([email])
}

model SupplierAccount {
  id          String   @id @default(cuid())
  userId      String
  user        User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  provider    SupplierProvider
  apiKey      String   // Encrypted
  apiSecret   String?  // Encrypted
  accountId   String?
  isActive    Boolean  @default(true)

  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  @@unique([userId, provider])
}

enum SupplierProvider {
  ALIEXPRESS
  CJ_DROPSHIPPING
  SPOCKET
}

// ==================== STORE ====================

model Store {
  id              String    @id @default(cuid())
  userId          String
  user            User      @relation(fields: [userId], references: [id], onDelete: Cascade)

  // Basic Info
  name            String
  slug            String    @unique
  description     String?
  niche           String
  status          StoreStatus @default(CREATING)

  // Brand Identity (Generated)
  brandIdentity   Json?     // { logo, colors, fonts, tone }

  // Domain
  customDomain    String?   @unique
  subdomain       String    @unique

  // Products & Content
  products        Product[]
  pages           Page[]
  media           MediaAsset[]

  // Configuration
  settings        Json?     // Store settings

  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt
  publishedAt     DateTime?

  @@index([userId])
  @@index([slug])
}

enum StoreStatus {
  CREATING        // Genesis in progress
  PENDING_REVIEW  // Awaiting user approval
  ACTIVE          // Live
  PAUSED          // Temporarily offline
  ARCHIVED        // Soft deleted
}

// ==================== PRODUCTS ====================

model Product {
  id              String    @id @default(cuid())
  storeId         String
  store           Store     @relation(fields: [storeId], references: [id], onDelete: Cascade)

  // Source Info
  sourceProvider  SupplierProvider
  sourceId        String    // External product ID
  sourceUrl       String?

  // Product Details
  title           String    // AI-generated
  originalTitle   String    // From supplier
  description     String    @db.Text // AI-generated
  slug            String

  // Pricing
  costPrice       Decimal   @db.Decimal(10, 2)
  sellingPrice    Decimal   @db.Decimal(10, 2)
  compareAtPrice  Decimal?  @db.Decimal(10, 2)
  currency        String    @default("USD")

  // Inventory
  stockQuantity   Int       @default(0)
  trackInventory  Boolean   @default(true)

  // SEO
  metaTitle       String?
  metaDescription String?

  // Media
  images          ProductImage[]

  // Variants
  variants        ProductVariant[]

  // Status
  status          ProductStatus @default(DRAFT)

  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  @@unique([storeId, slug])
  @@index([storeId])
}

enum ProductStatus {
  DRAFT
  ACTIVE
  OUT_OF_STOCK
  ARCHIVED
}

model ProductImage {
  id          String   @id @default(cuid())
  productId   String
  product     Product  @relation(fields: [productId], references: [id], onDelete: Cascade)

  originalUrl String   // Supplier URL
  processedUrl String? // Our CDN URL (processed)
  altText     String?
  position    Int      @default(0)

  createdAt   DateTime @default(now())
}

model ProductVariant {
  id          String   @id @default(cuid())
  productId   String
  product     Product  @relation(fields: [productId], references: [id], onDelete: Cascade)

  sku         String?
  title       String   // e.g., "Red / Large"
  price       Decimal  @db.Decimal(10, 2)
  costPrice   Decimal  @db.Decimal(10, 2)
  stock       Int      @default(0)

  options     Json     // { color: "Red", size: "Large" }

  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  @@unique([productId, sku])
}

// ==================== CONTENT ====================

model Page {
  id          String   @id @default(cuid())
  storeId     String
  store       Store    @relation(fields: [storeId], references: [id], onDelete: Cascade)

  type        PageType
  title       String
  slug        String
  content     String   @db.Text // AI-generated

  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  @@unique([storeId, slug])
}

enum PageType {
  PRIVACY_POLICY
  TERMS_OF_SERVICE
  REFUND_POLICY
  ABOUT
  CONTACT
  FAQ
  CUSTOM
}

model MediaAsset {
  id          String    @id @default(cuid())
  storeId     String
  store       Store     @relation(fields: [storeId], references: [id], onDelete: Cascade)

  type        MediaType
  filename    String
  mimeType    String
  size        Int       // bytes
  url         String

  // Processing metadata
  originalUrl String?   // If processed from external source
  metadata    Json?     // dimensions, format, etc.

  createdAt   DateTime  @default(now())
}

enum MediaType {
  IMAGE
  VIDEO
  LOGO
  BACKGROUND
}

// ==================== AGENT JOBS ====================

model AgentJob {
  id          String     @id @default(cuid())
  storeId     String?

  type        AgentType
  status      JobStatus  @default(PENDING)
  priority    Int        @default(0)

  // Input/Output
  input       Json       // Agent-specific input
  output      Json?      // Agent-specific output
  error       String?

  // Timing
  startedAt   DateTime?
  completedAt DateTime?

  // Progress tracking
  progress    Int        @default(0) // 0-100
  currentStep String?

  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt

  @@index([storeId])
  @@index([status])
}

enum AgentType {
  BRAND_GENERATOR
  PRODUCT_SCOUT
  COPYWRITER
  IMAGE_PROCESSOR
  VIDEO_GENERATOR
}

enum JobStatus {
  PENDING
  PROCESSING
  COMPLETED
  FAILED
  CANCELLED
}
```

### 2.2 Action Items
- [ ] Create Prisma schema with all entities
- [ ] Setup database migrations workflow
- [ ] Create seed data for development
- [ ] Configure connection pooling (PgBouncer)
- [ ] Setup read replicas for production

---

## 3. Genesis Engine Service (Python FastAPI)

### 3.1 Service Architecture

```
services/genesis-engine/
├── app/
│   ├── __init__.py
│   ├── main.py                 # FastAPI app entry
│   ├── config.py               # Settings management
│   │
│   ├── api/
│   │   ├── __init__.py
│   │   ├── deps.py             # Dependencies
│   │   ├── v1/
│   │   │   ├── __init__.py
│   │   │   ├── router.py       # API router
│   │   │   ├── stores.py       # Store endpoints
│   │   │   ├── products.py     # Product endpoints
│   │   │   └── jobs.py         # Job status endpoints
│   │   └── health.py
│   │
│   ├── agents/
│   │   ├── __init__.py
│   │   ├── base.py             # Base agent class
│   │   │
│   │   ├── brand/
│   │   │   ├── __init__.py
│   │   │   ├── agent.py        # Brand agent
│   │   │   ├── prompts.py      # LLM prompts
│   │   │   └── schemas.py      # Input/Output models
│   │   │
│   │   ├── scout/
│   │   │   ├── __init__.py
│   │   │   ├── agent.py        # Product scout agent
│   │   │   ├── suppliers/      # Supplier integrations
│   │   │   │   ├── __init__.py
│   │   │   │   ├── base.py
│   │   │   │   ├── aliexpress.py
│   │   │   │   ├── cj_dropshipping.py
│   │   │   │   └── spocket.py
│   │   │   └── ranking.py      # Product ranking logic
│   │   │
│   │   └── copywriter/
│   │       ├── __init__.py
│   │       ├── agent.py        # Copywriter agent
│   │       ├── prompts.py      # Writing prompts
│   │       └── templates.py    # Output templates
│   │
│   ├── models/
│   │   ├── __init__.py
│   │   ├── store.py            # Store models
│   │   ├── product.py          # Product models
│   │   └── job.py              # Job models
│   │
│   ├── services/
│   │   ├── __init__.py
│   │   ├── llm.py              # LLM integration (OpenAI/Anthropic)
│   │   ├── database.py         # Database operations
│   │   ├── queue.py            # Redis queue operations
│   │   └── storage.py          # File storage (S3)
│   │
│   └── workers/
│       ├── __init__.py
│       ├── celery_app.py       # Celery configuration
│       └── tasks.py            # Background tasks
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── test_agents/
│   └── test_api/
│
├── requirements.txt
├── requirements-dev.txt
├── pyproject.toml
├── Dockerfile
└── docker-compose.yml
```

### 3.2 Brand Agent Implementation

**Responsibilities:**
- Generate brand name from niche
- Create logo concept/prompt
- Define color palette (primary, secondary, accent)
- Select typography (heading, body fonts)
- Establish brand voice/tone

**Flow:**
```
Input: { niche: "Cyberpunk home decor", preferences?: {...} }
                    ↓
        [1. Brand Name Generation]
        - Generate 5 name options via LLM
        - Check domain availability
        - Select best available option
                    ↓
        [2. Visual Identity Generation]
        - Generate logo concept/prompt
        - Create color palette (6 colors)
        - Select font pairing
                    ↓
        [3. Brand Voice Definition]
        - Define tone (professional, playful, etc.)
        - Create tagline
        - Generate brand story snippet
                    ↓
Output: BrandIdentity { name, logo, colors, fonts, voice }
```

**Key Dependencies:**
- `openai` or `anthropic` - LLM for generation
- Domain availability API (GoDaddy/Namecheap API)
- Color theory library for palette generation

### 3.3 Product Scout Agent Implementation

**Responsibilities:**
- Connect to supplier APIs
- Search products by niche keywords
- Analyze product potential (reviews, sales, margins)
- Rank and select top 10-20 products
- Extract product data for import

**Flow:**
```
Input: { niche: "Cyberpunk home decor", supplierAccounts: [...] }
                    ↓
        [1. Keyword Generation]
        - Generate 20-30 search keywords from niche
        - Include variations and synonyms
                    ↓
        [2. Product Discovery]
        - Query each connected supplier API
        - Fetch 100+ candidate products per supplier
                    ↓
        [3. Product Analysis & Ranking]
        - Calculate "winning score" per product:
          * Review count & rating
          * Order count (if available)
          * Price point analysis
          * Shipping time
          * Image quality score
          * Margin potential (cost vs. market price)
                    ↓
        [4. Selection & Deduplication]
        - Remove duplicates across suppliers
        - Select top 15-20 products
        - Prefer variety in product types
                    ↓
Output: ScoutResult { products: ProductData[], metadata: {...} }
```

**Key Dependencies:**
- Supplier API integrations
- `numpy`/`pandas` for scoring algorithms
- Image analysis for quality scoring

### 3.4 Copywriter Agent Implementation

**Responsibilities:**
- Generate SEO-optimized product titles
- Write persuasive product descriptions
- Create meta tags (title, description)
- Generate legal pages (Privacy, Terms, Refund)
- Write about page content

**Flow:**
```
Input: { store: StoreData, products: ProductData[], brandVoice: {...} }
                    ↓
        [1. Product Copy Generation]
        For each product:
        - Rewrite title (SEO + brand voice)
        - Generate description (features, benefits, CTA)
        - Create meta title + description
                    ↓
        [2. Store Page Generation]
        - Privacy Policy (template + customization)
        - Terms of Service
        - Refund Policy
        - About Us page
        - FAQ page
                    ↓
        [3. SEO Optimization]
        - Keyword density check
        - Readability scoring
        - Meta tag optimization
                    ↓
Output: CopywriterResult { products: [...], pages: [...] }
```

**Key Dependencies:**
- LLM for content generation
- SEO analysis tools
- Legal template library

### 3.5 Action Items
- [ ] Setup FastAPI project structure
- [ ] Implement base agent class with common functionality
- [ ] Build Brand Agent with LLM integration
- [ ] Build Product Scout Agent with supplier integrations
- [ ] Build Copywriter Agent
- [ ] Setup Celery workers for background processing
- [ ] Implement job status tracking and webhooks
- [ ] Write comprehensive tests for each agent

---

## 4. Media Studio Service (Python FastAPI)

### 4.1 Service Architecture

```
services/media-studio/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── config.py
│   │
│   ├── api/
│   │   ├── __init__.py
│   │   └── v1/
│   │       ├── __init__.py
│   │       ├── images.py       # Image processing endpoints
│   │       └── jobs.py         # Job status
│   │
│   ├── processors/
│   │   ├── __init__.py
│   │   ├── base.py             # Base processor
│   │   ├── watermark.py        # Watermark detection/removal
│   │   ├── background.py       # Background removal/replacement
│   │   ├── enhancement.py      # Quality enhancement
│   │   └── lifestyle.py        # Lifestyle image generation
│   │
│   ├── services/
│   │   ├── __init__.py
│   │   ├── image_ai.py         # AI image services
│   │   ├── storage.py          # S3/CDN upload
│   │   └── queue.py            # Redis queue
│   │
│   └── workers/
│       ├── __init__.py
│       └── tasks.py            # Background tasks
│
├── tests/
├── requirements.txt
├── Dockerfile
└── docker-compose.yml
```

### 4.2 Image Processing Pipeline

**Phase 1 Capabilities (Images Only):**

1. **Watermark Detection & Removal**
   - Detect common supplier watermarks
   - Use inpainting to remove watermarks
   - Quality: Must be undetectable

2. **Background Removal**
   - Segment product from background
   - Create transparent PNG
   - Tools: `rembg`, SAM (Segment Anything Model)

3. **Lifestyle Background Generation**
   - Generate contextual backgrounds
   - Categories: "marble desk", "living room", "outdoor", etc.
   - Composite product onto background
   - Tools: Stable Diffusion, DALL-E

4. **Image Enhancement**
   - Upscale low-resolution images
   - Color correction
   - Sharpening
   - Tools: Real-ESRGAN, PIL

**Processing Flow:**
```
Input: { imageUrl: string, operations: Operation[] }
                    ↓
        [1. Download & Validate]
        - Fetch image from URL
        - Validate format & dimensions
        - Create working copy
                    ↓
        [2. Processing Pipeline]
        - Apply operations in sequence:
          1. Watermark removal (if detected)
          2. Background removal (if requested)
          3. Background replacement (if requested)
          4. Enhancement (upscale, color correct)
                    ↓
        [3. Output & Storage]
        - Export in multiple sizes (thumbnail, medium, large)
        - Upload to CDN (S3 + CloudFront)
        - Return URLs
                    ↓
Output: { urls: { thumbnail, medium, large }, metadata: {...} }
```

### 4.3 Action Items
- [ ] Setup FastAPI project structure
- [ ] Implement image download and validation
- [ ] Build watermark detection/removal processor
- [ ] Build background removal processor (rembg)
- [ ] Build lifestyle background generator (Stable Diffusion)
- [ ] Build image enhancement processor
- [ ] Setup S3 storage and CDN
- [ ] Implement Celery workers for batch processing
- [ ] Write tests with sample images

---

## 5. Dashboard Application (Next.js 14)

### 5.1 Application Structure

```
apps/dashboard/
├── src/
│   ├── app/
│   │   ├── (auth)/
│   │   │   ├── login/
│   │   │   ├── register/
│   │   │   └── layout.tsx
│   │   │
│   │   ├── (dashboard)/
│   │   │   ├── layout.tsx      # Dashboard shell
│   │   │   ├── page.tsx        # Overview
│   │   │   │
│   │   │   ├── stores/
│   │   │   │   ├── page.tsx    # Store list
│   │   │   │   ├── new/
│   │   │   │   │   └── page.tsx # Create store wizard
│   │   │   │   └── [storeId]/
│   │   │   │       ├── page.tsx
│   │   │   │       ├── products/
│   │   │   │       ├── media/
│   │   │   │       ├── pages/
│   │   │   │       └── settings/
│   │   │   │
│   │   │   ├── settings/
│   │   │   │   ├── page.tsx
│   │   │   │   ├── profile/
│   │   │   │   ├── billing/
│   │   │   │   └── integrations/
│   │   │   │       ├── stripe/
│   │   │   │       └── suppliers/
│   │   │   │
│   │   │   └── jobs/
│   │   │       └── page.tsx    # Agent job monitor
│   │   │
│   │   ├── api/
│   │   │   ├── auth/
│   │   │   │   └── [...nextauth]/
│   │   │   └── webhooks/
│   │   │       └── stripe/
│   │   │
│   │   ├── layout.tsx
│   │   └── page.tsx            # Landing/redirect
│   │
│   ├── components/
│   │   ├── ui/                 # Shadcn components
│   │   ├── dashboard/          # Dashboard-specific
│   │   ├── stores/             # Store management
│   │   ├── products/           # Product components
│   │   └── jobs/               # Job tracking
│   │
│   ├── lib/
│   │   ├── api.ts              # API client
│   │   ├── auth.ts             # Auth utilities
│   │   └── utils.ts            # Helpers
│   │
│   ├── hooks/
│   │   ├── use-store.ts
│   │   ├── use-products.ts
│   │   └── use-jobs.ts
│   │
│   └── styles/
│       └── globals.css
│
├── public/
├── package.json
├── next.config.js
├── tailwind.config.ts
└── tsconfig.json
```

### 5.2 Key Features for Phase 1

**Authentication:**
- Email/password registration
- OAuth (Google, GitHub)
- Email verification
- Password reset

**Store Creation Wizard:**
```
Step 1: Niche Selection
├── Text input: "What do you want to sell?"
├── Niche suggestions/autocomplete
└── Examples: "Cyberpunk home decor"

Step 2: Supplier Connection
├── Connect AliExpress account
├── Connect CJ Dropshipping
└── Connect Spocket

Step 3: Payment Setup (Stripe Connect)
├── Redirect to Stripe onboarding
├── Or "Setup Later" option
└── Stripe account verification status

Step 4: Launch Genesis
├── Review summary
├── Estimated generation time
└── "Create My Store" button

Step 5: Progress Monitor
├── Real-time job status updates
├── Brand generation progress
├── Product scouting progress
├── Content generation progress
├── Estimated completion time
```

**Store Dashboard:**
- Overview with key metrics
- Product management (view, edit, toggle)
- Media gallery (view, regenerate)
- Content pages (view, edit)
- Settings (domain, SEO, branding)

**Settings:**
- Profile management
- Stripe Connect status
- Supplier account management
- Billing and subscription

### 5.3 Action Items
- [ ] Initialize Next.js 14 project with App Router
- [ ] Setup Tailwind CSS + Shadcn UI
- [ ] Implement authentication (NextAuth.js)
- [ ] Build dashboard layout/shell
- [ ] Create store creation wizard
- [ ] Build real-time job progress component
- [ ] Implement store management pages
- [ ] Build product listing and editing
- [ ] Create settings pages
- [ ] Implement Stripe Connect integration
- [ ] Add supplier account connection UI

---

## 6. Storefront Application (Next.js 14)

### 6.1 Application Structure

```
apps/storefront/
├── src/
│   ├── app/
│   │   ├── layout.tsx          # Root layout with theme
│   │   ├── page.tsx            # Homepage
│   │   │
│   │   ├── products/
│   │   │   ├── page.tsx        # Product listing
│   │   │   └── [slug]/
│   │   │       └── page.tsx    # Product detail
│   │   │
│   │   ├── cart/
│   │   │   └── page.tsx        # Shopping cart
│   │   │
│   │   ├── checkout/
│   │   │   ├── page.tsx        # Checkout flow
│   │   │   └── success/
│   │   │       └── page.tsx    # Order confirmation
│   │   │
│   │   ├── pages/
│   │   │   └── [slug]/
│   │   │       └── page.tsx    # Dynamic pages (privacy, etc.)
│   │   │
│   │   └── api/
│   │       └── checkout/
│   │           └── route.ts    # Stripe checkout session
│   │
│   ├── components/
│   │   ├── layout/             # Header, Footer, Nav
│   │   ├── product/            # Product card, gallery
│   │   ├── cart/               # Cart drawer, items
│   │   └── checkout/           # Checkout form
│   │
│   └── lib/
│       ├── store-context.tsx   # Store data context
│       └── cart.ts             # Cart utilities
│
├── package.json
├── next.config.js
└── tailwind.config.ts
```

### 6.2 Key Features

**Multi-Tenant Architecture:**
- Subdomain-based routing (store-slug.zyenta.com)
- Custom domain support (store's own domain)
- Dynamic theming based on brand identity

**Pages:**
- Homepage with hero and featured products
- Product listing with filtering
- Product detail with gallery
- Shopping cart
- Checkout (Stripe integration)
- Order confirmation
- Legal pages (Privacy, Terms, Refund)

**Checkout Flow (Client as MoR):**
```
Cart → Checkout Page → Stripe Checkout Session → Success Page
         │
         └── Uses Client's Stripe API Keys (stored encrypted)
         └── Client receives funds directly
         └── Zyenta gets platform fee via Stripe Connect
```

### 6.3 Action Items
- [ ] Initialize Next.js 14 project
- [ ] Setup multi-tenant routing
- [ ] Build dynamic theming system
- [ ] Create homepage template
- [ ] Build product listing page
- [ ] Build product detail page
- [ ] Implement shopping cart
- [ ] Build checkout with Stripe
- [ ] Create legal page templates
- [ ] Setup custom domain handling

---

## 7. Shared Packages

### 7.1 Database Package (`packages/database`)

**Contents:**
- Prisma schema
- Database client generation
- Migration scripts
- Seed data

**Action Items:**
- [ ] Setup Prisma project
- [ ] Create complete schema
- [ ] Write migration scripts
- [ ] Create development seed data

### 7.2 Queue Package (`packages/queue`)

**Contents:**
- BullMQ configuration
- Queue definitions
- Job type definitions
- Worker utilities

**Action Items:**
- [ ] Setup BullMQ with Redis
- [ ] Define job types and queues
- [ ] Create worker utilities
- [ ] Implement retry logic

### 7.3 UI Package (`packages/ui`)

**Contents:**
- Shadcn UI components
- Custom shared components
- Theme configuration
- Icon library setup

**Action Items:**
- [ ] Initialize Shadcn UI
- [ ] Create base components
- [ ] Setup theme tokens
- [ ] Export shared components

### 7.4 Shared Types Package (`packages/shared-types`)

**Contents:**
- TypeScript type definitions
- API response types
- Shared enums
- Zod schemas

**Action Items:**
- [ ] Define core types
- [ ] Create API schemas
- [ ] Export Zod validators

---

## 8. Infrastructure Setup

### 8.1 Local Development (Docker Compose)

```yaml
# infra/docker/docker-compose.dev.yml
version: '3.8'

services:
  # PostgreSQL Database
  postgres:
    image: postgres:15-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: zyenta
      POSTGRES_PASSWORD: zyenta_dev
      POSTGRES_DB: zyenta
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U zyenta"]
      interval: 5s
      timeout: 5s
      retries: 5

  # Redis for Queue & Cache
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

  # RabbitMQ for Message Bus
  rabbitmq:
    image: rabbitmq:3-management-alpine
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: zyenta
      RABBITMQ_DEFAULT_PASS: zyenta_dev
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_running"]
      interval: 10s
      timeout: 5s
      retries: 5

  # MinIO for S3-compatible storage
  minio:
    image: minio/minio:latest
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    volumes:
      - minio_data:/data
    command: server /data --console-address ":9001"

volumes:
  postgres_data:
  redis_data:
  rabbitmq_data:
  minio_data:
```

### 8.2 Production Infrastructure (Kubernetes)

**Core Components:**
- PostgreSQL (RDS or managed)
- Redis (ElastiCache or managed)
- RabbitMQ (Amazon MQ or self-managed)
- S3 + CloudFront for media
- EKS/GKE for container orchestration

**Kubernetes Manifests Structure:**
```
infra/k8s/
├── base/
│   ├── namespace.yaml
│   ├── configmaps/
│   ├── secrets/
│   └── services/
├── apps/
│   ├── dashboard/
│   ├── storefront/
│   └── services/
├── monitoring/
│   ├── prometheus/
│   └── grafana/
└── ingress/
    └── nginx-ingress.yaml
```

### 8.3 CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'pnpm'
      - run: pnpm install
      - run: pnpm lint

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test
        ports:
          - 5432:5432
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v4
      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - run: pnpm install
      - run: pnpm test

  build:
    runs-on: ubuntu-latest
    needs: [lint, test]
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v4
      - run: pnpm install
      - run: pnpm build
```

### 8.4 Action Items
- [ ] Create Docker Compose for local development
- [ ] Setup Kubernetes manifests structure
- [ ] Create GitHub Actions CI workflow
- [ ] Configure Terraform for cloud resources
- [ ] Setup monitoring (Prometheus + Grafana)
- [ ] Configure logging (ELK or CloudWatch)

---

## 9. Security & Compliance

### 9.1 Security Requirements

**Data Encryption:**
- [ ] Encrypt all API keys and secrets at rest (AES-256)
- [ ] Use TLS 1.3 for all communications
- [ ] Implement proper key rotation

**Authentication & Authorization:**
- [ ] Implement JWT with refresh tokens
- [ ] Setup RBAC for multi-user stores (future)
- [ ] Rate limiting on all endpoints
- [ ] CSRF protection

**Payment Security:**
- [ ] Never store full credit card data
- [ ] Use Stripe Connect for compliant handling
- [ ] PCI DSS compliance via Stripe
- [ ] Secure webhook verification

**API Security:**
- [ ] Input validation on all endpoints
- [ ] SQL injection prevention (Prisma handles)
- [ ] XSS prevention (React handles)
- [ ] CORS configuration
- [ ] API key authentication for service-to-service

### 9.2 Action Items
- [ ] Implement secret encryption service
- [ ] Setup rate limiting middleware
- [ ] Configure CORS properly
- [ ] Implement webhook signature verification
- [ ] Create security audit checklist

---

## 10. Implementation Order

### Sprint 1: Foundation (Weeks 1-2)
1. Monorepo setup (pnpm, Turborepo, configs)
2. Database package with Prisma schema
3. Docker Compose for local dev
4. CI/CD pipeline
5. Shared packages (queue, types, ui base)

### Sprint 2: Genesis Engine Core (Weeks 3-4)
1. FastAPI project setup
2. Base agent class implementation
3. Brand Agent (LLM integration)
4. Product Scout Agent (1 supplier: AliExpress)
5. Copywriter Agent

### Sprint 3: Media Studio (Weeks 5-6)
1. FastAPI project setup
2. Image download and validation
3. Background removal processor
4. Watermark removal processor
5. S3 storage integration
6. Celery workers setup

### Sprint 4: Dashboard MVP (Weeks 7-8)
1. Next.js project setup
2. Authentication (NextAuth.js)
3. Dashboard layout
4. Store creation wizard
5. Job progress tracking
6. Basic store management

### Sprint 5: Storefront & Payments (Weeks 9-10)
1. Storefront Next.js setup
2. Multi-tenant routing
3. Product pages
4. Cart and checkout
5. Stripe Connect integration
6. Custom domain setup

### Sprint 6: Integration & Polish (Weeks 11-12)
1. End-to-end flow testing
2. Error handling improvements
3. Performance optimization
4. Documentation
5. Security audit
6. Beta launch preparation

---

## 11. Success Metrics

### Technical Metrics
- [ ] Store generation time < 10 minutes
- [ ] Image processing time < 30 seconds per image
- [ ] API response time < 200ms (p95)
- [ ] 99.9% uptime SLA
- [ ] Zero critical security vulnerabilities

### Business Metrics (Post-Launch)
- [ ] Successful store generations
- [ ] Payment processing success rate
- [ ] User retention rate
- [ ] Time to first sale per store

---

## 12. Risk Mitigation

| Risk | Impact | Mitigation |
|------|--------|------------|
| Supplier API rate limits | High | Implement caching, request pooling |
| LLM API costs | Medium | Batch processing, caching, model selection |
| Image processing quality | High | Multiple fallback processors, human review option |
| Stripe Connect complexity | High | Thorough testing, staging environment |
| Multi-tenancy security | Critical | Proper isolation, audit logging |

---

## Appendix A: Environment Variables

```bash
# .env.example

# Database
DATABASE_URL="postgresql://zyenta:zyenta_dev@localhost:5432/zyenta"

# Redis
REDIS_URL="redis://localhost:6379"

# RabbitMQ
RABBITMQ_URL="amqp://zyenta:zyenta_dev@localhost:5672"

# S3 / MinIO
S3_ENDPOINT="http://localhost:9000"
S3_ACCESS_KEY="minioadmin"
S3_SECRET_KEY="minioadmin"
S3_BUCKET="zyenta-media"

# LLM APIs
OPENAI_API_KEY="sk-..."
ANTHROPIC_API_KEY="sk-ant-..."

# Stripe
STRIPE_SECRET_KEY="sk_test_..."
STRIPE_WEBHOOK_SECRET="whsec_..."
STRIPE_CONNECT_CLIENT_ID="ca_..."

# Auth
NEXTAUTH_SECRET="..."
NEXTAUTH_URL="http://localhost:3000"

# Supplier APIs (encrypted in production)
ALIEXPRESS_APP_KEY="..."
ALIEXPRESS_APP_SECRET="..."

# Feature Flags
ENABLE_VIDEO_GENERATION="false"
```

---

## Appendix B: API Endpoints Reference

### Genesis Engine API

```
POST   /api/v1/stores                   # Create new store (starts Genesis)
GET    /api/v1/stores/{id}              # Get store details
GET    /api/v1/stores/{id}/products     # List store products
POST   /api/v1/stores/{id}/products     # Add product manually
PATCH  /api/v1/stores/{id}/products/{pid} # Update product
DELETE /api/v1/stores/{id}/products/{pid} # Delete product

GET    /api/v1/jobs/{id}                # Get job status
GET    /api/v1/jobs/{id}/logs           # Get job logs

POST   /api/v1/agents/brand/generate    # Trigger brand generation
POST   /api/v1/agents/scout/search      # Trigger product scouting
POST   /api/v1/agents/copywriter/generate # Trigger copy generation
```

### Media Studio API

```
POST   /api/v1/images/process           # Process single image
POST   /api/v1/images/batch             # Process multiple images
GET    /api/v1/images/jobs/{id}         # Get processing job status
POST   /api/v1/images/backgrounds       # Generate lifestyle background
```

---

*This execution plan provides a comprehensive roadmap for building Zyenta's Phase 1 MVP. Each section contains actionable items that can be tracked and completed incrementally.*
