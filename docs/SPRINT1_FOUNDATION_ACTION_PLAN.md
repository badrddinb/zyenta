# Sprint 1: Foundation - Actionable Implementation Plan

## Overview

This document provides step-by-step implementation instructions for Sprint 1 of Zyenta's Phase 1 MVP. Sprint 1 establishes the project foundation that all subsequent development depends on.

**Objective:** Create a fully functional monorepo with database, infrastructure, and shared packages ready for feature development.

---

## Prerequisites

Before starting, ensure you have:
- [ ] Node.js 20+ installed
- [ ] pnpm 8+ installed (`npm install -g pnpm`)
- [ ] Python 3.11+ installed
- [ ] Docker & Docker Compose installed
- [ ] Git configured with SSH keys
- [ ] Code editor with TypeScript/Python support

---

## Task 1: Monorepo Setup

### 1.1 Initialize pnpm Workspace

**File: `package.json`**
```bash
# Create root package.json
```

```json
{
  "name": "zyenta",
  "version": "0.1.0",
  "private": true,
  "description": "The World's First Zero-Effort Autonomous AI-Commerce Engine",
  "keywords": ["ai", "ecommerce", "dropshipping", "automation"],
  "license": "MIT",
  "author": "Zyenta Team",
  "scripts": {
    "build": "turbo build",
    "dev": "turbo dev",
    "lint": "turbo lint",
    "lint:fix": "turbo lint:fix",
    "test": "turbo test",
    "clean": "turbo clean && rm -rf node_modules",
    "format": "prettier --write \"**/*.{ts,tsx,js,jsx,json,md}\"",
    "format:check": "prettier --check \"**/*.{ts,tsx,js,jsx,json,md}\"",
    "db:generate": "pnpm --filter @zyenta/database generate",
    "db:migrate": "pnpm --filter @zyenta/database migrate:dev",
    "db:push": "pnpm --filter @zyenta/database push",
    "db:studio": "pnpm --filter @zyenta/database studio",
    "prepare": "husky"
  },
  "devDependencies": {
    "@types/node": "^20.10.0",
    "husky": "^9.0.0",
    "lint-staged": "^15.2.0",
    "prettier": "^3.2.0",
    "turbo": "^2.0.0",
    "typescript": "^5.3.0"
  },
  "packageManager": "pnpm@8.15.0",
  "engines": {
    "node": ">=20.0.0",
    "pnpm": ">=8.0.0"
  }
}
```

**File: `pnpm-workspace.yaml`**
```yaml
packages:
  - "apps/*"
  - "packages/*"
  - "services/*"
```

### 1.2 Setup Turborepo

**File: `turbo.json`**
```json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": [".env*"],
  "globalEnv": [
    "NODE_ENV",
    "DATABASE_URL",
    "REDIS_URL",
    "NEXTAUTH_SECRET",
    "NEXTAUTH_URL"
  ],
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "!.next/cache/**", "dist/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "lint": {
      "dependsOn": ["^build"],
      "outputs": []
    },
    "lint:fix": {
      "outputs": []
    },
    "test": {
      "dependsOn": ["^build"],
      "outputs": ["coverage/**"]
    },
    "clean": {
      "cache": false
    },
    "generate": {
      "cache": false
    }
  }
}
```

### 1.3 Configure TypeScript Base

**File: `tsconfig.base.json`**
```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "display": "Base",
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "allowJs": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationMap": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true
  },
  "exclude": ["node_modules", "dist", ".next", "coverage"]
}
```

### 1.4 Configure ESLint

**File: `.eslintrc.js`**
```javascript
module.exports = {
  root: true,
  extends: ["eslint:recommended"],
  parserOptions: {
    ecmaVersion: "latest",
    sourceType: "module",
  },
  env: {
    node: true,
    es2022: true,
  },
  ignorePatterns: [
    "node_modules/",
    "dist/",
    ".next/",
    "coverage/",
    "*.config.js",
    "*.config.mjs",
  ],
  overrides: [
    {
      files: ["**/*.ts", "**/*.tsx"],
      parser: "@typescript-eslint/parser",
      extends: [
        "eslint:recommended",
        "plugin:@typescript-eslint/recommended",
        "prettier",
      ],
      plugins: ["@typescript-eslint"],
      rules: {
        "@typescript-eslint/no-unused-vars": [
          "error",
          { argsIgnorePattern: "^_" },
        ],
        "@typescript-eslint/no-explicit-any": "warn",
        "@typescript-eslint/consistent-type-imports": [
          "error",
          { prefer: "type-imports" },
        ],
      },
    },
    {
      files: ["**/*.tsx"],
      extends: ["plugin:react/recommended", "plugin:react-hooks/recommended"],
      plugins: ["react", "react-hooks"],
      settings: {
        react: { version: "detect" },
      },
      rules: {
        "react/react-in-jsx-scope": "off",
        "react/prop-types": "off",
      },
    },
  ],
};
```

### 1.5 Configure Prettier

**File: `.prettierrc`**
```json
{
  "semi": true,
  "singleQuote": false,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 80,
  "useTabs": false,
  "bracketSpacing": true,
  "bracketSameLine": false,
  "arrowParens": "always",
  "endOfLine": "lf"
}
```

**File: `.prettierignore`**
```
node_modules
dist
.next
coverage
pnpm-lock.yaml
*.prisma
```

### 1.6 Setup Husky & Lint-Staged

**File: `.husky/pre-commit`**
```bash
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

pnpm lint-staged
```

**File: `.lintstagedrc.js`**
```javascript
module.exports = {
  "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
  "*.{js,jsx}": ["eslint --fix", "prettier --write"],
  "*.{json,md,yml,yaml}": ["prettier --write"],
  "*.py": ["black", "ruff check --fix"],
};
```

### 1.7 Create Directory Structure

```bash
# Execute these commands to create the directory structure
mkdir -p apps/dashboard/src/{app,components,lib,hooks,styles}
mkdir -p apps/storefront/src/{app,components,lib}
mkdir -p services/genesis-engine/app/{agents,api,core,models,services,workers}
mkdir -p services/genesis-engine/tests
mkdir -p services/media-studio/app/{processors,api,services,workers}
mkdir -p services/media-studio/tests
mkdir -p packages/database/prisma
mkdir -p packages/queue/src
mkdir -p packages/shared-types/src
mkdir -p packages/ui/src
mkdir -p infra/docker
mkdir -p infra/k8s/{base,apps,monitoring,ingress}
mkdir -p infra/terraform
mkdir -p .github/workflows
```

### 1.8 Create .gitignore

**File: `.gitignore`**
```gitignore
# Dependencies
node_modules/
.pnpm-store/

# Build outputs
dist/
.next/
out/
build/

# Cache
.turbo/
.eslintcache
*.tsbuildinfo

# Environment
.env
.env.local
.env.*.local
!.env.example

# IDE
.idea/
.vscode/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Testing
coverage/
.nyc_output/

# Python
__pycache__/
*.py[cod]
*$py.class
.venv/
venv/
.pytest_cache/
.ruff_cache/
*.egg-info/

# Logs
*.log
logs/

# Database
*.db
*.sqlite

# Prisma
packages/database/prisma/migrations/**/migration_lock.toml
```

### 1.9 Create .env.example

**File: `.env.example`**
```bash
# ===========================================
# ZYENTA ENVIRONMENT CONFIGURATION
# ===========================================
# Copy this file to .env and fill in the values

# Environment
NODE_ENV=development

# ===========================================
# DATABASE
# ===========================================
DATABASE_URL="postgresql://zyenta:zyenta_dev@localhost:5432/zyenta"

# ===========================================
# REDIS
# ===========================================
REDIS_URL="redis://localhost:6379"

# ===========================================
# RABBITMQ
# ===========================================
RABBITMQ_URL="amqp://zyenta:zyenta_dev@localhost:5672"

# ===========================================
# S3 / MINIO (Local Development)
# ===========================================
S3_ENDPOINT="http://localhost:9000"
S3_ACCESS_KEY="minioadmin"
S3_SECRET_KEY="minioadmin"
S3_BUCKET="zyenta-media"
S3_REGION="us-east-1"

# ===========================================
# LLM APIs
# ===========================================
OPENAI_API_KEY="sk-..."
ANTHROPIC_API_KEY="sk-ant-..."

# ===========================================
# STRIPE
# ===========================================
STRIPE_SECRET_KEY="sk_test_..."
STRIPE_PUBLISHABLE_KEY="pk_test_..."
STRIPE_WEBHOOK_SECRET="whsec_..."
STRIPE_CONNECT_CLIENT_ID="ca_..."

# ===========================================
# AUTHENTICATION (NextAuth.js)
# ===========================================
NEXTAUTH_SECRET="generate-a-secure-secret-here"
NEXTAUTH_URL="http://localhost:3000"

# OAuth Providers (optional)
GOOGLE_CLIENT_ID=""
GOOGLE_CLIENT_SECRET=""
GITHUB_CLIENT_ID=""
GITHUB_CLIENT_SECRET=""

# ===========================================
# SUPPLIER APIS
# ===========================================
ALIEXPRESS_APP_KEY=""
ALIEXPRESS_APP_SECRET=""
CJ_DROPSHIPPING_API_KEY=""
SPOCKET_API_KEY=""

# ===========================================
# FEATURE FLAGS
# ===========================================
ENABLE_VIDEO_GENERATION="false"
ENABLE_PAID_ADS_AGENT="false"

# ===========================================
# SERVICES (Internal)
# ===========================================
GENESIS_ENGINE_URL="http://localhost:8001"
MEDIA_STUDIO_URL="http://localhost:8002"
```

---

## Task 2: Database Package

### 2.1 Initialize Package

**File: `packages/database/package.json`**
```json
{
  "name": "@zyenta/database",
  "version": "0.1.0",
  "private": true,
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "default": "./dist/index.js"
    },
    "./client": {
      "types": "./dist/client.d.ts",
      "default": "./dist/client.js"
    }
  },
  "scripts": {
    "build": "tsc",
    "dev": "tsc --watch",
    "clean": "rm -rf dist",
    "generate": "prisma generate",
    "migrate:dev": "prisma migrate dev",
    "migrate:deploy": "prisma migrate deploy",
    "push": "prisma db push",
    "studio": "prisma studio",
    "seed": "tsx prisma/seed.ts",
    "lint": "eslint src --ext .ts",
    "lint:fix": "eslint src --ext .ts --fix"
  },
  "dependencies": {
    "@prisma/client": "^5.10.0"
  },
  "devDependencies": {
    "@types/node": "^20.10.0",
    "prisma": "^5.10.0",
    "tsx": "^4.7.0",
    "typescript": "^5.3.0"
  }
}
```

**File: `packages/database/tsconfig.json`**
```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,
    "declarationMap": true,
    "noEmit": false
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "prisma"]
}
```

### 2.2 Create Prisma Schema

**File: `packages/database/prisma/schema.prisma`**
```prisma
// ===========================================
// ZYENTA DATABASE SCHEMA
// ===========================================

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ===========================================
// USER & AUTHENTICATION
// ===========================================

model User {
  id            String    @id @default(cuid())
  email         String    @unique
  name          String?
  passwordHash  String?
  emailVerified DateTime?
  image         String?

  // Stripe Connect
  stripeAccountId String?  @unique
  stripeOnboarded Boolean  @default(false)

  // Relations
  stores           Store[]
  supplierAccounts SupplierAccount[]
  sessions         Session[]
  accounts         Account[]

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([email])
  @@map("users")
}

model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String? @db.Text
  access_token      String? @db.Text
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String? @db.Text
  session_state     String?

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerAccountId])
  @@index([userId])
  @@map("accounts")
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@map("sessions")
}

model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime

  @@unique([identifier, token])
  @@map("verification_tokens")
}

// ===========================================
// SUPPLIER ACCOUNTS
// ===========================================

model SupplierAccount {
  id        String           @id @default(cuid())
  userId    String
  user      User             @relation(fields: [userId], references: [id], onDelete: Cascade)
  provider  SupplierProvider
  apiKey    String           // Encrypted at application layer
  apiSecret String?          // Encrypted at application layer
  accountId String?
  isActive  Boolean          @default(true)

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@unique([userId, provider])
  @@index([userId])
  @@map("supplier_accounts")
}

enum SupplierProvider {
  ALIEXPRESS
  CJ_DROPSHIPPING
  SPOCKET
}

// ===========================================
// STORES
// ===========================================

model Store {
  id          String      @id @default(cuid())
  userId      String
  user        User        @relation(fields: [userId], references: [id], onDelete: Cascade)
  name        String
  slug        String      @unique
  description String?
  niche       String
  status      StoreStatus @default(CREATING)

  // Brand Identity (JSON: logo, colors, fonts, tone)
  brandIdentity Json?

  // Domain Configuration
  customDomain String? @unique
  subdomain    String  @unique

  // Store Settings (JSON)
  settings Json?

  // Relations
  products Product[]
  pages    Page[]
  media    MediaAsset[]
  jobs     AgentJob[]

  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
  publishedAt DateTime?

  @@index([userId])
  @@index([slug])
  @@index([status])
  @@map("stores")
}

enum StoreStatus {
  CREATING
  PENDING_REVIEW
  ACTIVE
  PAUSED
  ARCHIVED
}

// ===========================================
// PRODUCTS
// ===========================================

model Product {
  id             String           @id @default(cuid())
  storeId        String
  store          Store            @relation(fields: [storeId], references: [id], onDelete: Cascade)
  sourceProvider SupplierProvider
  sourceId       String
  sourceUrl      String?

  // Product Details
  title         String
  originalTitle String
  description   String  @db.Text
  slug          String

  // Pricing
  costPrice      Decimal  @db.Decimal(10, 2)
  sellingPrice   Decimal  @db.Decimal(10, 2)
  compareAtPrice Decimal? @db.Decimal(10, 2)
  currency       String   @default("USD")

  // Inventory
  stockQuantity  Int     @default(0)
  trackInventory Boolean @default(true)

  // SEO
  metaTitle       String?
  metaDescription String?

  // Relations
  images   ProductImage[]
  variants ProductVariant[]

  status ProductStatus @default(DRAFT)

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@unique([storeId, slug])
  @@unique([storeId, sourceProvider, sourceId])
  @@index([storeId])
  @@index([status])
  @@map("products")
}

enum ProductStatus {
  DRAFT
  ACTIVE
  OUT_OF_STOCK
  ARCHIVED
}

model ProductImage {
  id           String  @id @default(cuid())
  productId    String
  product      Product @relation(fields: [productId], references: [id], onDelete: Cascade)
  originalUrl  String
  processedUrl String?
  altText      String?
  position     Int     @default(0)

  createdAt DateTime @default(now())

  @@index([productId])
  @@map("product_images")
}

model ProductVariant {
  id        String  @id @default(cuid())
  productId String
  product   Product @relation(fields: [productId], references: [id], onDelete: Cascade)
  sku       String?
  title     String
  price     Decimal @db.Decimal(10, 2)
  costPrice Decimal @db.Decimal(10, 2)
  stock     Int     @default(0)
  options   Json    // { color: "Red", size: "Large" }

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@unique([productId, sku])
  @@index([productId])
  @@map("product_variants")
}

// ===========================================
// CONTENT PAGES
// ===========================================

model Page {
  id      String   @id @default(cuid())
  storeId String
  store   Store    @relation(fields: [storeId], references: [id], onDelete: Cascade)
  type    PageType
  title   String
  slug    String
  content String   @db.Text

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@unique([storeId, slug])
  @@index([storeId])
  @@map("pages")
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

// ===========================================
// MEDIA ASSETS
// ===========================================

model MediaAsset {
  id       String    @id @default(cuid())
  storeId  String
  store    Store     @relation(fields: [storeId], references: [id], onDelete: Cascade)
  type     MediaType
  filename String
  mimeType String
  size     Int       // bytes
  url      String

  // Processing metadata
  originalUrl String?
  metadata    Json?   // dimensions, format, etc.

  createdAt DateTime @default(now())

  @@index([storeId])
  @@index([type])
  @@map("media_assets")
}

enum MediaType {
  IMAGE
  VIDEO
  LOGO
  BACKGROUND
}

// ===========================================
// AGENT JOBS
// ===========================================

model AgentJob {
  id      String  @id @default(cuid())
  storeId String?
  store   Store?  @relation(fields: [storeId], references: [id], onDelete: SetNull)

  type     AgentType
  status   JobStatus @default(PENDING)
  priority Int       @default(0)

  // Input/Output
  input  Json
  output Json?
  error  String?

  // Timing
  startedAt   DateTime?
  completedAt DateTime?

  // Progress
  progress    Int     @default(0) // 0-100
  currentStep String?

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([storeId])
  @@index([status])
  @@index([type])
  @@map("agent_jobs")
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

### 2.3 Create Database Client Export

**File: `packages/database/src/index.ts`**
```typescript
export * from "@prisma/client";
export { prisma } from "./client";
```

**File: `packages/database/src/client.ts`**
```typescript
import { PrismaClient } from "@prisma/client";

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined;
};

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log:
      process.env.NODE_ENV === "development"
        ? ["query", "error", "warn"]
        : ["error"],
  });

if (process.env.NODE_ENV !== "production") {
  globalForPrisma.prisma = prisma;
}
```

### 2.4 Create Seed Script

**File: `packages/database/prisma/seed.ts`**
```typescript
import { PrismaClient } from "@prisma/client";

const prisma = new PrismaClient();

async function main() {
  console.log("Seeding database...");

  // Create test user
  const user = await prisma.user.upsert({
    where: { email: "test@zyenta.com" },
    update: {},
    create: {
      email: "test@zyenta.com",
      name: "Test User",
      emailVerified: new Date(),
    },
  });

  console.log("Created user:", user.email);

  // Create test store
  const store = await prisma.store.upsert({
    where: { slug: "test-store" },
    update: {},
    create: {
      userId: user.id,
      name: "Test Cyberpunk Store",
      slug: "test-store",
      subdomain: "test-store",
      niche: "Cyberpunk home decor",
      status: "ACTIVE",
      brandIdentity: {
        name: "NeonNest",
        colors: {
          primary: "#FF00FF",
          secondary: "#00FFFF",
          accent: "#FFFF00",
        },
        fonts: {
          heading: "Orbitron",
          body: "Inter",
        },
        tone: "futuristic",
      },
    },
  });

  console.log("Created store:", store.name);

  console.log("Seeding completed.");
}

main()
  .catch((e) => {
    console.error(e);
    process.exit(1);
  })
  .finally(async () => {
    await prisma.$disconnect();
  });
```

---

## Task 3: Docker Compose for Local Development

### 3.1 Development Docker Compose

**File: `infra/docker/docker-compose.dev.yml`**
```yaml
version: "3.8"

services:
  # ===========================================
  # PostgreSQL Database
  # ===========================================
  postgres:
    image: postgres:15-alpine
    container_name: zyenta-postgres
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: zyenta
      POSTGRES_PASSWORD: zyenta_dev
      POSTGRES_DB: zyenta
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U zyenta -d zyenta"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - zyenta-network

  # ===========================================
  # Redis (Queue & Cache)
  # ===========================================
  redis:
    image: redis:7-alpine
    container_name: zyenta-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - zyenta-network

  # ===========================================
  # RabbitMQ (Message Bus)
  # ===========================================
  rabbitmq:
    image: rabbitmq:3-management-alpine
    container_name: zyenta-rabbitmq
    ports:
      - "5672:5672"   # AMQP
      - "15672:15672" # Management UI
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
    networks:
      - zyenta-network

  # ===========================================
  # MinIO (S3-Compatible Storage)
  # ===========================================
  minio:
    image: minio/minio:latest
    container_name: zyenta-minio
    ports:
      - "9000:9000" # API
      - "9001:9001" # Console
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    volumes:
      - minio_data:/data
    command: server /data --console-address ":9001"
    healthcheck:
      test: ["CMD", "mc", "ready", "local"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - zyenta-network

  # ===========================================
  # MinIO Setup (Create default bucket)
  # ===========================================
  minio-setup:
    image: minio/mc:latest
    container_name: zyenta-minio-setup
    depends_on:
      minio:
        condition: service_healthy
    entrypoint: >
      /bin/sh -c "
      mc alias set myminio http://minio:9000 minioadmin minioadmin;
      mc mb myminio/zyenta-media --ignore-existing;
      mc anonymous set public myminio/zyenta-media;
      exit 0;
      "
    networks:
      - zyenta-network

volumes:
  postgres_data:
  redis_data:
  rabbitmq_data:
  minio_data:

networks:
  zyenta-network:
    driver: bridge
```

### 3.2 Create Helper Script

**File: `scripts/dev-infra.sh`**
```bash
#!/bin/bash

# ===========================================
# Zyenta Development Infrastructure Manager
# ===========================================

set -e

DOCKER_COMPOSE_FILE="infra/docker/docker-compose.dev.yml"

case "$1" in
  start)
    echo "Starting development infrastructure..."
    docker compose -f $DOCKER_COMPOSE_FILE up -d
    echo ""
    echo "Services started:"
    echo "  - PostgreSQL: localhost:5432"
    echo "  - Redis: localhost:6379"
    echo "  - RabbitMQ: localhost:5672 (Management: http://localhost:15672)"
    echo "  - MinIO: localhost:9000 (Console: http://localhost:9001)"
    echo ""
    echo "Run 'pnpm db:migrate' to set up the database schema."
    ;;
  stop)
    echo "Stopping development infrastructure..."
    docker compose -f $DOCKER_COMPOSE_FILE down
    ;;
  restart)
    echo "Restarting development infrastructure..."
    docker compose -f $DOCKER_COMPOSE_FILE down
    docker compose -f $DOCKER_COMPOSE_FILE up -d
    ;;
  logs)
    docker compose -f $DOCKER_COMPOSE_FILE logs -f "${@:2}"
    ;;
  status)
    docker compose -f $DOCKER_COMPOSE_FILE ps
    ;;
  clean)
    echo "Stopping and removing all data..."
    docker compose -f $DOCKER_COMPOSE_FILE down -v
    echo "All data has been removed."
    ;;
  *)
    echo "Usage: $0 {start|stop|restart|logs|status|clean}"
    echo ""
    echo "Commands:"
    echo "  start   - Start all development services"
    echo "  stop    - Stop all services"
    echo "  restart - Restart all services"
    echo "  logs    - View logs (optionally specify service name)"
    echo "  status  - Show status of all services"
    echo "  clean   - Stop services and remove all data"
    exit 1
    ;;
esac
```

Make it executable:
```bash
chmod +x scripts/dev-infra.sh
```

---

## Task 4: CI/CD Pipeline

### 4.1 GitHub Actions CI Workflow

**File: `.github/workflows/ci.yml`**
```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  NODE_VERSION: "20"
  PNPM_VERSION: "8"
  PYTHON_VERSION: "3.11"

jobs:
  # ===========================================
  # Lint & Type Check
  # ===========================================
  lint:
    name: Lint & Type Check
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup pnpm
        uses: pnpm/action-setup@v2
        with:
          version: ${{ env.PNPM_VERSION }}

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "pnpm"

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Generate Prisma Client
        run: pnpm db:generate

      - name: Lint
        run: pnpm lint

      - name: Type Check
        run: pnpm build

      - name: Format Check
        run: pnpm format:check

  # ===========================================
  # Test (Node.js packages)
  # ===========================================
  test-node:
    name: Test (Node.js)
    runs-on: ubuntu-latest
    needs: lint
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup pnpm
        uses: pnpm/action-setup@v2
        with:
          version: ${{ env.PNPM_VERSION }}

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "pnpm"

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Generate Prisma Client
        run: pnpm db:generate

      - name: Run migrations
        run: pnpm db:push
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/test

      - name: Run tests
        run: pnpm test
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/test
          REDIS_URL: redis://localhost:6379

  # ===========================================
  # Test (Python services)
  # ===========================================
  test-python:
    name: Test (Python)
    runs-on: ubuntu-latest
    needs: lint
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
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}

      - name: Install Python dependencies (Genesis Engine)
        working-directory: services/genesis-engine
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt -r requirements-dev.txt
        continue-on-error: true

      - name: Run Python tests (Genesis Engine)
        working-directory: services/genesis-engine
        run: pytest --cov=app tests/
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/test
          REDIS_URL: redis://localhost:6379
        continue-on-error: true

  # ===========================================
  # Build
  # ===========================================
  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [test-node]
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup pnpm
        uses: pnpm/action-setup@v2
        with:
          version: ${{ env.PNPM_VERSION }}

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "pnpm"

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Generate Prisma Client
        run: pnpm db:generate

      - name: Build
        run: pnpm build
```

### 4.2 Deploy Workflow (Placeholder)

**File: `.github/workflows/deploy.yml`**
```yaml
name: Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    name: Deploy to Production
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      # Placeholder for deployment steps
      - name: Deploy Notice
        run: |
          echo "Deployment pipeline configured."
          echo "Add deployment steps when infrastructure is ready."
```

---

## Task 5: Shared Packages

### 5.1 Queue Package

**File: `packages/queue/package.json`**
```json
{
  "name": "@zyenta/queue",
  "version": "0.1.0",
  "private": true,
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "default": "./dist/index.js"
    }
  },
  "scripts": {
    "build": "tsc",
    "dev": "tsc --watch",
    "clean": "rm -rf dist",
    "lint": "eslint src --ext .ts",
    "lint:fix": "eslint src --ext .ts --fix"
  },
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

**File: `packages/queue/tsconfig.json`**
```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,
    "declarationMap": true,
    "noEmit": false
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

**File: `packages/queue/src/index.ts`**
```typescript
export * from "./config";
export * from "./queues";
export * from "./types";
```

**File: `packages/queue/src/config.ts`**
```typescript
import { Redis } from "ioredis";

export interface QueueConfig {
  host: string;
  port: number;
  password?: string;
  maxRetriesPerRequest: null;
}

export function createRedisConnection(): Redis {
  const redisUrl = process.env.REDIS_URL || "redis://localhost:6379";
  return new Redis(redisUrl, {
    maxRetriesPerRequest: null,
  });
}

export function getQueueConfig(): QueueConfig {
  const redisUrl = new URL(process.env.REDIS_URL || "redis://localhost:6379");

  return {
    host: redisUrl.hostname,
    port: parseInt(redisUrl.port || "6379", 10),
    password: redisUrl.password || undefined,
    maxRetriesPerRequest: null,
  };
}
```

**File: `packages/queue/src/queues.ts`**
```typescript
import { Queue, Worker, type Job, type WorkerOptions } from "bullmq";
import { getQueueConfig } from "./config";
import type { JobData, JobType } from "./types";

const connection = getQueueConfig();

// Queue names
export const QUEUE_NAMES = {
  GENESIS: "genesis",
  MEDIA: "media",
  NOTIFICATIONS: "notifications",
} as const;

// Create queues
export const genesisQueue = new Queue<JobData>(QUEUE_NAMES.GENESIS, {
  connection,
  defaultJobOptions: {
    attempts: 3,
    backoff: {
      type: "exponential",
      delay: 1000,
    },
    removeOnComplete: 100,
    removeOnFail: 50,
  },
});

export const mediaQueue = new Queue<JobData>(QUEUE_NAMES.MEDIA, {
  connection,
  defaultJobOptions: {
    attempts: 3,
    backoff: {
      type: "exponential",
      delay: 2000,
    },
    removeOnComplete: 100,
    removeOnFail: 50,
  },
});

export const notificationsQueue = new Queue<JobData>(
  QUEUE_NAMES.NOTIFICATIONS,
  {
    connection,
    defaultJobOptions: {
      attempts: 5,
      removeOnComplete: true,
    },
  }
);

// Worker factory
export function createWorker<T = JobData>(
  queueName: string,
  processor: (job: Job<T>) => Promise<void>,
  options?: Partial<WorkerOptions>
): Worker<T> {
  return new Worker<T>(queueName, processor, {
    connection,
    ...options,
  });
}
```

**File: `packages/queue/src/types.ts`**
```typescript
export type JobType =
  | "BRAND_GENERATION"
  | "PRODUCT_SCOUTING"
  | "COPYWRITING"
  | "IMAGE_PROCESSING"
  | "SEND_EMAIL"
  | "SEND_WEBHOOK";

export interface JobData {
  type: JobType;
  storeId?: string;
  userId?: string;
  payload: Record<string, unknown>;
}

export interface BrandGenerationPayload {
  niche: string;
  preferences?: {
    style?: string;
    tone?: string;
  };
}

export interface ProductScoutingPayload {
  niche: string;
  supplierAccounts: Array<{
    provider: string;
    accountId: string;
  }>;
  maxProducts?: number;
}

export interface ImageProcessingPayload {
  imageUrl: string;
  operations: Array<{
    type: "REMOVE_BACKGROUND" | "REMOVE_WATERMARK" | "ENHANCE" | "RESIZE";
    params?: Record<string, unknown>;
  }>;
}
```

### 5.2 Shared Types Package

**File: `packages/shared-types/package.json`**
```json
{
  "name": "@zyenta/shared-types",
  "version": "0.1.0",
  "private": true,
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "default": "./dist/index.js"
    }
  },
  "scripts": {
    "build": "tsc",
    "dev": "tsc --watch",
    "clean": "rm -rf dist",
    "lint": "eslint src --ext .ts",
    "lint:fix": "eslint src --ext .ts --fix"
  },
  "dependencies": {
    "zod": "^3.22.0"
  },
  "devDependencies": {
    "@types/node": "^20.10.0",
    "typescript": "^5.3.0"
  }
}
```

**File: `packages/shared-types/tsconfig.json`**
```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,
    "declarationMap": true,
    "noEmit": false
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

**File: `packages/shared-types/src/index.ts`**
```typescript
export * from "./api";
export * from "./store";
export * from "./product";
export * from "./job";
```

**File: `packages/shared-types/src/api.ts`**
```typescript
import { z } from "zod";

// API Response wrapper
export const ApiResponseSchema = <T extends z.ZodTypeAny>(dataSchema: T) =>
  z.object({
    success: z.boolean(),
    data: dataSchema.optional(),
    error: z
      .object({
        code: z.string(),
        message: z.string(),
        details: z.record(z.unknown()).optional(),
      })
      .optional(),
    meta: z
      .object({
        page: z.number().optional(),
        perPage: z.number().optional(),
        total: z.number().optional(),
      })
      .optional(),
  });

export type ApiResponse<T> = {
  success: boolean;
  data?: T;
  error?: {
    code: string;
    message: string;
    details?: Record<string, unknown>;
  };
  meta?: {
    page?: number;
    perPage?: number;
    total?: number;
  };
};

// Pagination
export const PaginationSchema = z.object({
  page: z.coerce.number().min(1).default(1),
  perPage: z.coerce.number().min(1).max(100).default(20),
});

export type Pagination = z.infer<typeof PaginationSchema>;
```

**File: `packages/shared-types/src/store.ts`**
```typescript
import { z } from "zod";

export const StoreStatusSchema = z.enum([
  "CREATING",
  "PENDING_REVIEW",
  "ACTIVE",
  "PAUSED",
  "ARCHIVED",
]);

export type StoreStatus = z.infer<typeof StoreStatusSchema>;

export const BrandIdentitySchema = z.object({
  name: z.string(),
  logo: z.string().url().optional(),
  colors: z.object({
    primary: z.string(),
    secondary: z.string(),
    accent: z.string(),
    background: z.string().optional(),
    text: z.string().optional(),
  }),
  fonts: z.object({
    heading: z.string(),
    body: z.string(),
  }),
  tone: z.string(),
  tagline: z.string().optional(),
});

export type BrandIdentity = z.infer<typeof BrandIdentitySchema>;

export const CreateStoreSchema = z.object({
  niche: z.string().min(3).max(100),
  preferences: z
    .object({
      style: z.string().optional(),
      tone: z.string().optional(),
    })
    .optional(),
});

export type CreateStoreInput = z.infer<typeof CreateStoreSchema>;

export const StoreSchema = z.object({
  id: z.string(),
  userId: z.string(),
  name: z.string(),
  slug: z.string(),
  description: z.string().nullable(),
  niche: z.string(),
  status: StoreStatusSchema,
  brandIdentity: BrandIdentitySchema.nullable(),
  customDomain: z.string().nullable(),
  subdomain: z.string(),
  createdAt: z.coerce.date(),
  updatedAt: z.coerce.date(),
  publishedAt: z.coerce.date().nullable(),
});

export type Store = z.infer<typeof StoreSchema>;
```

**File: `packages/shared-types/src/product.ts`**
```typescript
import { z } from "zod";

export const SupplierProviderSchema = z.enum([
  "ALIEXPRESS",
  "CJ_DROPSHIPPING",
  "SPOCKET",
]);

export type SupplierProvider = z.infer<typeof SupplierProviderSchema>;

export const ProductStatusSchema = z.enum([
  "DRAFT",
  "ACTIVE",
  "OUT_OF_STOCK",
  "ARCHIVED",
]);

export type ProductStatus = z.infer<typeof ProductStatusSchema>;

export const ProductImageSchema = z.object({
  id: z.string(),
  originalUrl: z.string().url(),
  processedUrl: z.string().url().nullable(),
  altText: z.string().nullable(),
  position: z.number(),
});

export type ProductImage = z.infer<typeof ProductImageSchema>;

export const ProductVariantSchema = z.object({
  id: z.string(),
  sku: z.string().nullable(),
  title: z.string(),
  price: z.number(),
  costPrice: z.number(),
  stock: z.number(),
  options: z.record(z.string()),
});

export type ProductVariant = z.infer<typeof ProductVariantSchema>;

export const ProductSchema = z.object({
  id: z.string(),
  storeId: z.string(),
  sourceProvider: SupplierProviderSchema,
  sourceId: z.string(),
  sourceUrl: z.string().url().nullable(),
  title: z.string(),
  originalTitle: z.string(),
  description: z.string(),
  slug: z.string(),
  costPrice: z.number(),
  sellingPrice: z.number(),
  compareAtPrice: z.number().nullable(),
  currency: z.string(),
  stockQuantity: z.number(),
  trackInventory: z.boolean(),
  metaTitle: z.string().nullable(),
  metaDescription: z.string().nullable(),
  images: z.array(ProductImageSchema),
  variants: z.array(ProductVariantSchema),
  status: ProductStatusSchema,
  createdAt: z.coerce.date(),
  updatedAt: z.coerce.date(),
});

export type Product = z.infer<typeof ProductSchema>;
```

**File: `packages/shared-types/src/job.ts`**
```typescript
import { z } from "zod";

export const AgentTypeSchema = z.enum([
  "BRAND_GENERATOR",
  "PRODUCT_SCOUT",
  "COPYWRITER",
  "IMAGE_PROCESSOR",
  "VIDEO_GENERATOR",
]);

export type AgentType = z.infer<typeof AgentTypeSchema>;

export const JobStatusSchema = z.enum([
  "PENDING",
  "PROCESSING",
  "COMPLETED",
  "FAILED",
  "CANCELLED",
]);

export type JobStatus = z.infer<typeof JobStatusSchema>;

export const AgentJobSchema = z.object({
  id: z.string(),
  storeId: z.string().nullable(),
  type: AgentTypeSchema,
  status: JobStatusSchema,
  priority: z.number(),
  input: z.record(z.unknown()),
  output: z.record(z.unknown()).nullable(),
  error: z.string().nullable(),
  progress: z.number().min(0).max(100),
  currentStep: z.string().nullable(),
  startedAt: z.coerce.date().nullable(),
  completedAt: z.coerce.date().nullable(),
  createdAt: z.coerce.date(),
  updatedAt: z.coerce.date(),
});

export type AgentJob = z.infer<typeof AgentJobSchema>;
```

### 5.3 UI Package (Placeholder)

**File: `packages/ui/package.json`**
```json
{
  "name": "@zyenta/ui",
  "version": "0.1.0",
  "private": true,
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "default": "./dist/index.js"
    },
    "./styles.css": "./dist/styles.css"
  },
  "scripts": {
    "build": "tsc",
    "dev": "tsc --watch",
    "clean": "rm -rf dist",
    "lint": "eslint src --ext .ts,.tsx",
    "lint:fix": "eslint src --ext .ts,.tsx --fix"
  },
  "dependencies": {
    "class-variance-authority": "^0.7.0",
    "clsx": "^2.1.0",
    "tailwind-merge": "^2.2.0"
  },
  "peerDependencies": {
    "react": "^18.0.0",
    "react-dom": "^18.0.0"
  },
  "devDependencies": {
    "@types/react": "^18.2.0",
    "@types/react-dom": "^18.2.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "typescript": "^5.3.0"
  }
}
```

**File: `packages/ui/tsconfig.json`**
```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,
    "declarationMap": true,
    "noEmit": false,
    "jsx": "react-jsx"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

**File: `packages/ui/src/index.ts`**
```typescript
export * from "./utils";
// Components will be added as needed
```

**File: `packages/ui/src/utils.ts`**
```typescript
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

---

## Verification Checklist

After completing all tasks, verify the setup:

### Monorepo
- [ ] `pnpm install` completes without errors
- [ ] `pnpm build` builds all packages
- [ ] `pnpm lint` runs without errors
- [ ] Husky pre-commit hook is triggered on commit

### Database
- [ ] `pnpm db:generate` creates Prisma client
- [ ] `pnpm db:push` syncs schema to database
- [ ] `pnpm db:studio` opens Prisma Studio

### Infrastructure
- [ ] `./scripts/dev-infra.sh start` launches all services
- [ ] PostgreSQL is accessible on port 5432
- [ ] Redis is accessible on port 6379
- [ ] RabbitMQ management UI is accessible on port 15672
- [ ] MinIO console is accessible on port 9001

### CI/CD
- [ ] GitHub Actions workflow runs on push
- [ ] All CI jobs pass

---

## Next Steps

After completing Sprint 1, proceed to:

1. **Sprint 2:** Genesis Engine Core
   - FastAPI project setup
   - Base agent class
   - Brand Agent implementation
   - Product Scout Agent implementation
   - Copywriter Agent implementation

Refer to `PHASE1_EXECUTION_PLAN.md` for detailed specifications.
