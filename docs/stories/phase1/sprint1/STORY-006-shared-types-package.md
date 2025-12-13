# Story 006: Shared Types Package

## Story Information

| Field | Value |
|-------|-------|
| **Story ID** | STORY-006 |
| **Sprint** | Phase 1 - Sprint 1 |
| **Title** | Shared Types Package with Zod Schemas |
| **Priority** | High |
| **Story Points** | 5 |
| **Status** | Ready for Development |
| **Dependencies** | STORY-001 |

## User Story

**As a** developer
**I want** a shared types package with TypeScript types and Zod validation schemas
**So that** all applications and services use consistent type definitions with runtime validation

## Acceptance Criteria

### AC1: Package Configuration
- [ ] `packages/shared-types/package.json` is created
- [ ] Package name is `@zyenta/shared-types`
- [ ] Proper exports configuration
- [ ] Dependencies: zod
- [ ] Build and dev scripts defined

### AC2: TypeScript Configuration
- [ ] `packages/shared-types/tsconfig.json` extends base config
- [ ] Declaration files are generated
- [ ] Proper output directory configuration

### AC3: API Types
- [ ] `ApiResponseSchema` generic function for response wrapper
- [ ] `ApiResponse<T>` type with success, data, error, meta
- [ ] Error object includes code, message, optional details
- [ ] Meta object includes pagination info
- [ ] `PaginationSchema` with page and perPage
- [ ] `Pagination` type inferred from schema

### AC4: Store Types
- [ ] `StoreStatusSchema` enum (CREATING, PENDING_REVIEW, ACTIVE, PAUSED, ARCHIVED)
- [ ] `StoreStatus` type
- [ ] `BrandIdentitySchema` with name, logo, colors, fonts, tone, tagline
- [ ] `BrandIdentity` type
- [ ] `CreateStoreSchema` with niche and optional preferences
- [ ] `CreateStoreInput` type
- [ ] `StoreSchema` with all store fields
- [ ] `Store` type

### AC5: Product Types
- [ ] `SupplierProviderSchema` enum (ALIEXPRESS, CJ_DROPSHIPPING, SPOCKET)
- [ ] `SupplierProvider` type
- [ ] `ProductStatusSchema` enum (DRAFT, ACTIVE, OUT_OF_STOCK, ARCHIVED)
- [ ] `ProductStatus` type
- [ ] `ProductImageSchema` with id, urls, altText, position
- [ ] `ProductImage` type
- [ ] `ProductVariantSchema` with sku, title, price, stock, options
- [ ] `ProductVariant` type
- [ ] `ProductSchema` with all product fields
- [ ] `Product` type

### AC6: Job Types
- [ ] `AgentTypeSchema` enum (BRAND_GENERATOR, PRODUCT_SCOUT, COPYWRITER, IMAGE_PROCESSOR, VIDEO_GENERATOR)
- [ ] `AgentType` type
- [ ] `JobStatusSchema` enum (PENDING, PROCESSING, COMPLETED, FAILED, CANCELLED)
- [ ] `JobStatus` type
- [ ] `AgentJobSchema` with all job fields including progress
- [ ] `AgentJob` type

### AC7: Package Exports
- [ ] `index.ts` exports all types and schemas
- [ ] Exports from api, store, product, job modules
- [ ] Clean barrel file pattern

## Technical Details

### Type Structure

```
@zyenta/shared-types
├── api.ts
│   ├── ApiResponseSchema<T>
│   ├── ApiResponse<T>
│   ├── PaginationSchema
│   └── Pagination
│
├── store.ts
│   ├── StoreStatusSchema / StoreStatus
│   ├── BrandIdentitySchema / BrandIdentity
│   ├── CreateStoreSchema / CreateStoreInput
│   └── StoreSchema / Store
│
├── product.ts
│   ├── SupplierProviderSchema / SupplierProvider
│   ├── ProductStatusSchema / ProductStatus
│   ├── ProductImageSchema / ProductImage
│   ├── ProductVariantSchema / ProductVariant
│   └── ProductSchema / Product
│
└── job.ts
    ├── AgentTypeSchema / AgentType
    ├── JobStatusSchema / JobStatus
    └── AgentJobSchema / AgentJob
```

### Files to Create

1. **`packages/shared-types/package.json`**
   - Package configuration

2. **`packages/shared-types/tsconfig.json`**
   - TypeScript configuration

3. **`packages/shared-types/src/index.ts`**
   - Barrel export file

4. **`packages/shared-types/src/api.ts`**
   - API response wrapper schema
   - Pagination schema

5. **`packages/shared-types/src/store.ts`**
   - Store-related types and schemas

6. **`packages/shared-types/src/product.ts`**
   - Product-related types and schemas

7. **`packages/shared-types/src/job.ts`**
   - Agent job types and schemas

### Dependencies

```json
{
  "dependencies": {
    "zod": "^3.22.0"
  },
  "devDependencies": {
    "@types/node": "^20.10.0",
    "typescript": "^5.3.0"
  }
}
```

### Color Schema Structure

```typescript
colors: z.object({
  primary: z.string(),
  secondary: z.string(),
  accent: z.string(),
  background: z.string().optional(),
  text: z.string().optional(),
})
```

## Definition of Done

- [ ] All files are created as specified
- [ ] Package builds without errors (`pnpm build`)
- [ ] All Zod schemas validate correctly
- [ ] TypeScript types are properly inferred
- [ ] Can be imported by other packages
- [ ] Schemas match database models
- [ ] Code review completed

## Testing

1. Run `pnpm build` from packages/shared-types
2. Create test file that:
   - Imports schemas and types
   - Validates sample data with schemas
   - Checks type inference works correctly
   - Tests edge cases (missing fields, invalid data)

## Example Usage

```typescript
import { StoreSchema, CreateStoreSchema, ApiResponse, Store } from "@zyenta/shared-types";

// Validate input
const input = CreateStoreSchema.parse({
  niche: "Cyberpunk home decor",
  preferences: { style: "futuristic" }
});

// Type-safe response
const response: ApiResponse<Store> = {
  success: true,
  data: StoreSchema.parse(storeData)
};
```

## Notes

- Zod provides both runtime validation and TypeScript type inference
- Schemas should match Prisma models closely
- Use `z.coerce` for fields that may come as strings (dates, numbers)
- All nullable fields use `z.string().nullable()` pattern

## Related Stories

- Depends on: STORY-001 (Monorepo Setup)
- Related to: STORY-002 (Database Package - schemas should match)
