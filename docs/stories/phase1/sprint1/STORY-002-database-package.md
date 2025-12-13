# Story 002: Database Package

## Story Information

| Field | Value |
|-------|-------|
| **Story ID** | STORY-002 |
| **Sprint** | Phase 1 - Sprint 1 |
| **Title** | Database Package with Prisma Schema |
| **Priority** | Critical |
| **Story Points** | 8 |
| **Status** | Ready for Development |
| **Dependencies** | STORY-001 |

## User Story

**As a** developer
**I want** a database package with complete Prisma schema and client configuration
**So that** all applications and services can share a consistent database layer with typed queries

## Acceptance Criteria

### AC1: Package Configuration
- [ ] `packages/database/package.json` is created with correct exports
- [ ] Package name is `@zyenta/database`
- [ ] Proper build, generate, migrate, and studio scripts are defined
- [ ] Dependencies include `@prisma/client` and dev dependencies include `prisma`

### AC2: TypeScript Configuration
- [ ] `packages/database/tsconfig.json` extends base config
- [ ] Output directory is `./dist`
- [ ] Declaration files are generated

### AC3: Prisma Schema - User & Authentication
- [ ] `User` model with id, email, name, passwordHash, emailVerified, image
- [ ] Stripe Connect fields (stripeAccountId, stripeOnboarded)
- [ ] `Account` model for OAuth providers
- [ ] `Session` model for session management
- [ ] `VerificationToken` model for email verification
- [ ] Proper relations and indexes defined

### AC4: Prisma Schema - Supplier Accounts
- [ ] `SupplierAccount` model with encrypted API credentials
- [ ] `SupplierProvider` enum (ALIEXPRESS, CJ_DROPSHIPPING, SPOCKET)
- [ ] Unique constraint on userId + provider combination
- [ ] Proper cascade delete from User

### AC5: Prisma Schema - Stores
- [ ] `Store` model with all required fields
- [ ] `StoreStatus` enum (CREATING, PENDING_REVIEW, ACTIVE, PAUSED, ARCHIVED)
- [ ] Brand identity stored as JSON
- [ ] Custom domain and subdomain fields with unique constraints
- [ ] Relations to products, pages, media, and jobs

### AC6: Prisma Schema - Products
- [ ] `Product` model with source info, details, pricing, inventory, SEO
- [ ] `ProductStatus` enum (DRAFT, ACTIVE, OUT_OF_STOCK, ARCHIVED)
- [ ] `ProductImage` model with original and processed URLs
- [ ] `ProductVariant` model with SKU, pricing, stock, and options
- [ ] Proper indexes and unique constraints

### AC7: Prisma Schema - Content
- [ ] `Page` model for store content pages
- [ ] `PageType` enum (PRIVACY_POLICY, TERMS_OF_SERVICE, REFUND_POLICY, ABOUT, CONTACT, FAQ, CUSTOM)
- [ ] `MediaAsset` model for store media files
- [ ] `MediaType` enum (IMAGE, VIDEO, LOGO, BACKGROUND)

### AC8: Prisma Schema - Agent Jobs
- [ ] `AgentJob` model with type, status, input/output, timing, progress
- [ ] `AgentType` enum (BRAND_GENERATOR, PRODUCT_SCOUT, COPYWRITER, IMAGE_PROCESSOR, VIDEO_GENERATOR)
- [ ] `JobStatus` enum (PENDING, PROCESSING, COMPLETED, FAILED, CANCELLED)
- [ ] Proper indexes for querying jobs

### AC9: Database Client Export
- [ ] `packages/database/src/index.ts` exports Prisma client and types
- [ ] `packages/database/src/client.ts` implements singleton pattern
- [ ] Development logging configured
- [ ] Global prisma instance handling for hot reload

### AC10: Seed Script
- [ ] `packages/database/prisma/seed.ts` creates test data
- [ ] Test user with email `test@zyenta.com`
- [ ] Test store with cyberpunk theme
- [ ] Proper error handling and cleanup

## Technical Details

### Schema Models Summary

| Model | Description | Key Fields |
|-------|-------------|------------|
| User | User account | email, stripeAccountId, stripeOnboarded |
| Account | OAuth accounts | provider, providerAccountId, access_token |
| Session | User sessions | sessionToken, expires |
| VerificationToken | Email verification | token, expires |
| SupplierAccount | Supplier API keys | provider, apiKey, apiSecret |
| Store | User stores | name, slug, niche, status, brandIdentity |
| Product | Store products | title, description, pricing, inventory |
| ProductImage | Product images | originalUrl, processedUrl, altText |
| ProductVariant | Product variants | sku, title, price, options |
| Page | Store pages | type, title, slug, content |
| MediaAsset | Media files | type, filename, url, metadata |
| AgentJob | AI agent jobs | type, status, input, output, progress |

### Files to Create

1. **`packages/database/package.json`**
2. **`packages/database/tsconfig.json`**
3. **`packages/database/prisma/schema.prisma`**
4. **`packages/database/src/index.ts`**
5. **`packages/database/src/client.ts`**
6. **`packages/database/prisma/seed.ts`**

### Dependencies

```json
{
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

## Definition of Done

- [ ] All files are created as specified
- [ ] `pnpm db:generate` creates Prisma client without errors
- [ ] `pnpm db:push` syncs schema to database (with Docker running)
- [ ] `pnpm db:studio` opens Prisma Studio successfully
- [ ] Seed script runs without errors
- [ ] TypeScript types are properly exported
- [ ] Code review completed

## Testing

1. Start Docker infrastructure
2. Run `pnpm db:generate`
3. Run `pnpm db:push`
4. Run `pnpm db:seed`
5. Open Prisma Studio and verify data

## Notes

- All prices use Decimal(10,2) for precision
- API keys should be encrypted at application layer (noted in schema)
- JSON fields store complex nested data (brandIdentity, settings, options)

## Related Stories

- Depends on: STORY-001 (Monorepo Setup)
- Related to: STORY-003 (Docker Compose - needs database running)
