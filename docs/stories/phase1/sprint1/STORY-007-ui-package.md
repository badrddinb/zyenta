# Story 007: UI Package Foundation

## Story Information

| Field | Value |
|-------|-------|
| **Story ID** | STORY-007 |
| **Sprint** | Phase 1 - Sprint 1 |
| **Title** | UI Package Foundation with Utility Functions |
| **Priority** | Medium |
| **Story Points** | 3 |
| **Status** | Ready for Development |
| **Dependencies** | STORY-001 |

## User Story

**As a** developer
**I want** a shared UI package with utility functions and component infrastructure
**So that** the dashboard and storefront applications can share common styling utilities and components

## Acceptance Criteria

### AC1: Package Configuration
- [ ] `packages/ui/package.json` is created
- [ ] Package name is `@zyenta/ui`
- [ ] Proper exports for JS and CSS
- [ ] Dependencies: class-variance-authority, clsx, tailwind-merge
- [ ] Peer dependencies: react, react-dom
- [ ] Build and dev scripts defined

### AC2: TypeScript Configuration
- [ ] `packages/ui/tsconfig.json` extends base config
- [ ] JSX support is enabled (react-jsx)
- [ ] Declaration files are generated
- [ ] Proper output directory configuration

### AC3: Utility Functions
- [ ] `cn()` function is implemented
- [ ] Combines clsx and tailwind-merge
- [ ] Properly handles class merging for Tailwind CSS
- [ ] TypeScript types are correct

### AC4: Package Exports
- [ ] `index.ts` exports utility functions
- [ ] Comment placeholder for future components
- [ ] Clean barrel file pattern

## Technical Details

### Package Structure

```
packages/ui/
├── package.json
├── tsconfig.json
└── src/
    ├── index.ts      # Barrel exports
    └── utils.ts      # Utility functions (cn)
```

### Files to Create

1. **`packages/ui/package.json`**
   - Package configuration with peer dependencies

2. **`packages/ui/tsconfig.json`**
   - TypeScript configuration with JSX

3. **`packages/ui/src/index.ts`**
   - Export utilities
   - Placeholder for components

4. **`packages/ui/src/utils.ts`**
   - `cn()` utility function

### Dependencies

```json
{
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

### cn() Function Implementation

```typescript
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

### Example Usage

```typescript
import { cn } from "@zyenta/ui";

// Merges classes intelligently
const className = cn(
  "px-4 py-2 rounded",
  "bg-blue-500",
  isActive && "bg-blue-700",
  className // allows override
);
// Result: "px-4 py-2 rounded bg-blue-700" (bg-blue-500 is merged with bg-blue-700)
```

## Definition of Done

- [ ] All files are created as specified
- [ ] Package builds without errors (`pnpm build`)
- [ ] TypeScript types are properly exported
- [ ] `cn()` function works correctly
- [ ] Can be imported by dashboard and storefront apps
- [ ] Tailwind class merging works as expected
- [ ] Code review completed

## Testing

1. Run `pnpm build` from packages/ui
2. Create test that:
   - Imports `cn` from the package
   - Tests class merging scenarios
   - Verifies Tailwind conflict resolution

### Test Cases for cn()

```typescript
// Test 1: Basic class combination
cn("foo", "bar") // => "foo bar"

// Test 2: Conditional classes
cn("base", true && "active", false && "hidden") // => "base active"

// Test 3: Tailwind conflict resolution
cn("px-2", "px-4") // => "px-4" (later wins)

// Test 4: Complex Tailwind merging
cn("bg-red-500", "bg-blue-500") // => "bg-blue-500"

// Test 5: Array and object syntax
cn(["foo", "bar"], { active: true, disabled: false }) // => "foo bar active"
```

## Notes

- This is a foundation package - components will be added incrementally
- The `cn()` function is essential for Tailwind CSS + dynamic classes
- Shadcn UI components will be added in later sprints
- Consider adding common theme tokens in future iterations

## Future Additions (Not This Story)

- Shadcn UI component library
- Theme configuration
- Icon library (lucide-react)
- Animation utilities (framer-motion)
- Common layout components

## Related Stories

- Depends on: STORY-001 (Monorepo Setup)
- Related to: Dashboard and Storefront applications (future sprints)
