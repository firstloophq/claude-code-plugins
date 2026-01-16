---
name: manage-crud-architecture
description: Add or modify CRUD entities following the layered architecture pattern. Use when adding new database models, creating API endpoints, or implementing data access layers.
---

# CRUD Architecture

This skill guides implementing CRUD operations following the layered architecture pattern.

## Architecture Overview

```
1. Prisma Schema     → apps/server/src/db/prisma/schema.prisma
2. Zod Schemas       → packages/common/src/types/<entity>/  (shared types)
3. Repository Layer  → apps/server/src/repositories/
4. Controller Layer  → apps/server/src/controllers/
5. tRPC Router       → apps/server/src/routers/
6. DI Container      → apps/server/src/di/container.ts
```

## Checklist for Adding New CRUD Entity

- [ ] Add Prisma model in `apps/server/src/db/prisma/schema.prisma`
- [ ] Run migration: `cd apps/server && bun run db:migrate`
- [ ] Regenerate client: `cd apps/server && bun run db:generate`
- [ ] Create Zod schemas in `packages/common/src/types/<entity>/`
- [ ] Export from `packages/common/src/index.ts`
- [ ] Create Repository in `apps/server/src/repositories/`
- [ ] Create Controller in `apps/server/src/controllers/`
- [ ] Create tRPC Router in `apps/server/src/routers/`
- [ ] Register router in `apps/server/src/routers/_app.ts`
- [ ] Add to DI container in `apps/server/src/di/container.ts`
- [ ] Run build to verify: `cd apps/server && bun run build`

## Critical Gotchas

### 1. Clerk Integration
- **Organization IDs are NOT UUIDs** - Clerk uses `org_xxxxx` format. Use `z.string().min(1)` not `z.string().uuid()`
- **Don't pass organizationId in inputs** - Get it from `ctx.auth.orgId` in routers
- **Webhooks may not have synced** (optional consideration) - Use `getOrCreate` patterns for users/orgs when webhooks aren't guaranteed

### 2. Foreign Key Constraints (optional)
- If entity has FK to `users` (e.g., `created_by`), user must exist first
- Use `ensureUser` and `ensureOrganization` utilities before creating records
- See `apps/server/src/utils/ensure-user.ts` and `ensure-organization.ts`

### 3. tRPC Procedures
- Use `authenticatedProcedure` for routes requiring login
- Use `organizationProcedure` for routes requiring org context (auto-checks `ctx.auth.orgId`)
- Don't use `publicProcedure` with manual auth checks
- Procedures defined in `apps/server/src/trpc.ts`

For detailed implementation patterns, see [reference.md](reference.md).
