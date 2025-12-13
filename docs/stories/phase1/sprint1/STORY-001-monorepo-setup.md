# Story 001: Monorepo Setup

## Story Information

| Field | Value |
|-------|-------|
| **Story ID** | STORY-001 |
| **Sprint** | Phase 1 - Sprint 1 |
| **Title** | Monorepo Setup with pnpm and Turborepo |
| **Priority** | Critical |
| **Story Points** | 8 |
| **Status** | Ready for Development |

## User Story

**As a** developer
**I want** a properly configured monorepo with pnpm workspaces and Turborepo
**So that** I can efficiently manage multiple packages and services with shared dependencies and build orchestration

## Acceptance Criteria

### AC1: pnpm Workspace Configuration
- [ ] Root `package.json` is created with workspace configuration
- [ ] `pnpm-workspace.yaml` defines `apps/*`, `packages/*`, and `services/*` workspaces
- [ ] `packageManager` field specifies pnpm 8.15.0+
- [ ] Engine requirements specify Node.js 20+

### AC2: Turborepo Configuration
- [ ] `turbo.json` is created with proper task definitions
- [ ] `build` task has correct `dependsOn` and `outputs` configuration
- [ ] `dev` task is configured as persistent with no cache
- [ ] `lint`, `test`, and `clean` tasks are properly defined
- [ ] Global environment variables are correctly specified

### AC3: TypeScript Base Configuration
- [ ] `tsconfig.base.json` is created with strict TypeScript settings
- [ ] Target is ES2022 with ESNext module resolution
- [ ] Strict mode is enabled with proper compiler options
- [ ] Proper exclude patterns for node_modules, dist, .next, coverage

### AC4: ESLint Configuration
- [ ] `.eslintrc.js` is created with proper rules
- [ ] TypeScript parser and plugins are configured
- [ ] React rules are configured for TSX files
- [ ] Proper ignore patterns are set

### AC5: Prettier Configuration
- [ ] `.prettierrc` is created with consistent formatting rules
- [ ] `.prettierignore` excludes appropriate files
- [ ] Semi-colons, double quotes, 2-space tabs configured

### AC6: Husky & Lint-Staged Setup
- [ ] Husky pre-commit hook is configured
- [ ] lint-staged runs ESLint and Prettier on staged files
- [ ] Python files run black and ruff

### AC7: Directory Structure
- [ ] `apps/dashboard/src/{app,components,lib,hooks,styles}` directories exist
- [ ] `apps/storefront/src/{app,components,lib}` directories exist
- [ ] `services/genesis-engine/app/{agents,api,core,models,services,workers}` directories exist
- [ ] `services/genesis-engine/tests` directory exists
- [ ] `services/media-studio/app/{processors,api,services,workers}` directories exist
- [ ] `services/media-studio/tests` directory exists
- [ ] `packages/database/prisma` directory exists
- [ ] `packages/queue/src` directory exists
- [ ] `packages/shared-types/src` directory exists
- [ ] `packages/ui/src` directory exists
- [ ] `infra/docker` directory exists
- [ ] `infra/k8s/{base,apps,monitoring,ingress}` directories exist
- [ ] `infra/terraform` directory exists
- [ ] `.github/workflows` directory exists

### AC8: Git Configuration
- [ ] `.gitignore` includes all necessary patterns (node_modules, dist, .env, etc.)
- [ ] Python-specific ignores are included
- [ ] IDE and OS-specific ignores are included

### AC9: Environment Template
- [ ] `.env.example` is created with all required environment variables
- [ ] Variables are properly categorized and documented
- [ ] Sensible development defaults are provided

## Technical Details

### Files to Create

1. **`package.json`** - Root package configuration
   - Scripts: build, dev, lint, test, clean, format, db:*, prepare
   - DevDependencies: turbo, typescript, prettier, husky, lint-staged

2. **`pnpm-workspace.yaml`** - Workspace definition

3. **`turbo.json`** - Turborepo pipeline configuration

4. **`tsconfig.base.json`** - Shared TypeScript configuration

5. **`.eslintrc.js`** - ESLint rules and overrides

6. **`.prettierrc`** - Prettier formatting rules

7. **`.prettierignore`** - Files to skip formatting

8. **`.husky/pre-commit`** - Git pre-commit hook

9. **`.lintstagedrc.js`** - Lint-staged configuration

10. **`.gitignore`** - Git ignore patterns

11. **`.env.example`** - Environment variable template

### Dependencies

```json
{
  "devDependencies": {
    "@types/node": "^20.10.0",
    "husky": "^9.0.0",
    "lint-staged": "^15.2.0",
    "prettier": "^3.2.0",
    "turbo": "^2.0.0",
    "typescript": "^5.3.0"
  }
}
```

## Definition of Done

- [ ] All files are created as specified
- [ ] `pnpm install` completes without errors
- [ ] `pnpm build` runs successfully (may warn about missing packages)
- [ ] `pnpm lint` runs without configuration errors
- [ ] Husky pre-commit hook is triggered on commit attempt
- [ ] Directory structure matches specification
- [ ] Code review completed
- [ ] Documentation updated if needed

## Notes

- This is the foundation story - all other sprint 1 stories depend on this
- Ensure all configurations follow the exact specifications in the action plan
- Test the complete workflow before marking as done

## Related Stories

- Blocks: All other Sprint 1 stories
- Related to: STORY-002 (Database Package), STORY-004 (CI/CD Pipeline)
