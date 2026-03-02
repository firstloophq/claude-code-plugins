---
name: crud
description: Add or modify CRUD entities following the layered architecture pattern with oRPC contracts and Inversify DI. Use when adding new database models, creating API endpoints, or implementing data access layers.
---

# CRUD Architecture

This skill guides implementing CRUD operations following the layered architecture pattern.

## Architecture Overview

```
1. Prisma Schema       → apps/server/src/db/prisma/schema.prisma
2. Zod Types           → packages/common/src/contracts/v1/<entity>/types.ts
3. oRPC Contract       → packages/common/src/contracts/v1/<entity>/router.ts
4. Repository Layer    → apps/server/src/repositories/
5. Controller Layer    → apps/server/src/controllers/
6. oRPC Router (impl)  → apps/server/src/routers/v1/<entity>/router.ts
7. DI Container        → apps/server/src/di/container.ts
```

## Checklist for Adding New CRUD Entity

- [ ] Add Prisma model in `apps/server/src/db/prisma/schema.prisma`
- [ ] Run migration: `cd apps/server && bun run db:migrate`
- [ ] Regenerate client: `cd apps/server && bun run db:generate`
- [ ] Create Zod types in `packages/common/src/contracts/v1/<entity>/types.ts`
- [ ] Create oRPC contract in `packages/common/src/contracts/v1/<entity>/router.ts`
- [ ] Export from `packages/common/src/contracts/index.ts`
- [ ] Merge into `packages/common/src/contracts/app-contract.ts`
- [ ] Create Repository in `apps/server/src/repositories/` (with `@injectable()`)
- [ ] Create Controller in `apps/server/src/controllers/` (with `@injectable()`)
- [ ] Create oRPC router implementation in `apps/server/src/routers/v1/<entity>/router.ts`
- [ ] Register handlers in `apps/server/src/routers/orpc.ts`
- [ ] Register Repository and Controller bindings in DI container
- [ ] Run build to verify: `cd apps/server && bun run build`

## Critical Gotchas

### 1. Clerk Integration
- **Organization IDs are NOT UUIDs** — Clerk uses `org_xxxxx` format. Use `z.string().min(1)` not `z.string().uuid()`
- **Don't pass organizationId in inputs** — Get it from `context.orgId` in router handlers
- **Webhooks may not have synced** (optional consideration) — Use `getOrCreate` patterns for users/orgs when webhooks aren't guaranteed

### 2. Foreign Key Constraints (optional)
- If entity has FK to `users` (e.g., `created_by`), user must exist first
- Use `ensureUser` and `ensureOrganization` utilities before creating records
- See `apps/server/src/utils/ensure-user.ts` and `ensure-organization.ts`

### 3. oRPC Patterns
- Use `authed` middleware from `orpc-middleware.ts` for authenticated routes — no manual auth checks
- Use `unwrapResult(result)` to convert `Result<T>` into a thrown `ORPCError` on failure
- Contracts define HTTP method and path (e.g., `method: 'GET', path: '/v1/entities'`)
- Error codes are defined in `packages/common/src/contracts/errors.ts` via `oc.errors()`

For detailed implementation patterns, see [reference.md](reference.md).
