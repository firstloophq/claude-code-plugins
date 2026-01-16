# CRUD Architecture Reference

Detailed implementation patterns for each layer.

## 1. Prisma Schema

**File**: `apps/server/src/db/prisma/schema.prisma`

```prisma
model projects {
  id              String   @id @default(uuid())
  organization_id String
  name            String
  created_at      DateTime @default(now()) @db.Timestamptz(6)
  updated_at      DateTime @updatedAt @db.Timestamptz(6)

  organization  organizations   @relation(fields: [organization_id], references: [id], onDelete: Cascade, onUpdate: Cascade)
  content_items content_items[]

  @@index([organization_id], map: "idx_projects_organization_id")
  @@index([name], map: "idx_projects_name")
}
```

**Key Points:**
- Use snake_case for database fields
- Include `created_at` and `updated_at` timestamps
- Use `@updatedAt` for automatic update tracking
- Add `deleted_at` for soft deletes when needed
- Use cascading deletes/updates on foreign keys
- Add indexes for commonly queried fields
- **Organizations use Clerk IDs** - `@id` without `@default(uuid())` for org table

**After editing schema:**
```bash
cd apps/server && bun run db:migrate
cd apps/server && bun run db:generate
```

---

## 2. Zod Schemas

**Location**: `packages/common/src/types/<entity>/` (shared between frontend and backend)

Create three files per entity:

### Main Entity Schema
**File**: `apps/server/src/types/project/project.ts`
```typescript
import { z } from "zod";

export const projectSchema = z.object({
  id: z.string().uuid(),
  organizationId: z.string().uuid(),
  name: z.string().min(1),
  createdAt: z.date(),
  updatedAt: z.date(),
});

export type Project = z.infer<typeof projectSchema>;
```

### Create Input Schema
**File**: `apps/server/src/types/project/create-project-input.ts`
```typescript
import { z } from "zod";

export const createProjectInputSchema = z.object({
  organizationId: z.string().uuid(),
  name: z.string().min(1),
});

export type CreateProjectInput = z.infer<typeof createProjectInputSchema>;
```

### Update Input Schema
**File**: `apps/server/src/types/project/update-project-input.ts`
```typescript
import { z } from "zod";

export const updateProjectInputSchema = z.object({
  name: z.string().min(1).optional(),
});

export type UpdateProjectInput = z.infer<typeof updateProjectInputSchema>;
```

### Index Export
**File**: `apps/server/src/types/project/index.ts`
```typescript
export * from "./project";
export * from "./create-project-input";
export * from "./update-project-input";
```

**Key Points:**
- Use camelCase for TypeScript types (maps from snake_case in Prisma)
- Export both Zod schema and inferred TypeScript type
- Create separate schemas for: entity, create input, update input
- Update input fields should be optional

---

## 3. Repository Layer

**File**: `apps/server/src/repositories/ProjectsRepository.ts`

```typescript
import { PrismaClient } from "@server/generated/prisma";
import { Result, success, failure } from "common";
import { Project } from "@server/types/project";
import { CreateProjectInput } from "@server/types/project/create-project-input";
import { UpdateProjectInput } from "@server/types/project/update-project-input";

export class ProjectsRepository {
  private prisma: PrismaClient;

  constructor({ prisma }: { prisma: PrismaClient }) {
    this.prisma = prisma;
  }

  async create({ data }: { data: CreateProjectInput }): Promise<Result<Project>> {
    try {
      const result = await this.prisma.projects.create({
        data: {
          organization_id: data.organizationId,
          name: data.name,
        },
      });
      return success(mapPrismaProjectToProject(result));
    } catch (error) {
      return failure("Failed to create project", error);
    }
  }

  async getById({ id }: { id: string }): Promise<Result<Project>> {
    try {
      const result = await this.prisma.projects.findUnique({
        where: { id },
      });
      if (!result) {
        return failure("Project not found");
      }
      return success(mapPrismaProjectToProject(result));
    } catch (error) {
      return failure("Failed to get project", error);
    }
  }

  async update({ id, data }: { id: string; data: UpdateProjectInput }): Promise<Result<Project>> {
    try {
      const result = await this.prisma.projects.update({
        where: { id },
        data: {
          ...(data.name !== undefined && { name: data.name }),
        },
      });
      return success(mapPrismaProjectToProject(result));
    } catch (error) {
      return failure("Failed to update project", error);
    }
  }

  async delete({ id }: { id: string }): Promise<Result<Project>> {
    try {
      const result = await this.prisma.projects.delete({
        where: { id },
      });
      return success(mapPrismaProjectToProject(result));
    } catch (error) {
      return failure("Failed to delete project", error);
    }
  }

  async listByOrganization({ organizationId }: { organizationId: string }): Promise<Result<Project[]>> {
    try {
      const results = await this.prisma.projects.findMany({
        where: { organization_id: organizationId },
        orderBy: { created_at: "desc" },
      });
      return success(results.map(mapPrismaProjectToProject));
    } catch (error) {
      return failure("Failed to list projects", error);
    }
  }
}

// Mapping function: Prisma → Domain Type
function mapPrismaProjectToProject(prisma: {
  id: string;
  organization_id: string;
  name: string;
  created_at: Date;
  updated_at: Date;
}): Project {
  return {
    id: prisma.id,
    organizationId: prisma.organization_id,
    name: prisma.name,
    createdAt: prisma.created_at,
    updatedAt: prisma.updated_at,
  };
}
```

**Key Points:**
- **Always create a mapping function** (`mapPrisma[Entity]To[Entity]`)
- All methods return `Result<T>` for consistent error handling
- Use `success(data)` and `failure(message, error)` from `common` package
- Constructor uses single parameter object: `{ prisma }`
- All methods use single parameter objects: `{ id }`, `{ data }`, `{ id, data }`
- Repository only handles data access - no business logic
- For soft deletes, filter with `deleted_at: null` in queries

---

## 4. Controller Layer

**File**: `apps/server/src/controllers/ProjectsController.ts`

```typescript
import { Result, failure } from "common";
import { ProjectsRepository } from "@server/repositories/ProjectsRepository";
import { Project } from "@server/types/project";
import { CreateProjectInput } from "@server/types/project/create-project-input";
import { UpdateProjectInput } from "@server/types/project/update-project-input";
import { logger } from "@server/common/utils/logger";

export class ProjectsController {
  private projectsRepository: ProjectsRepository;

  constructor({ projectsRepository }: { projectsRepository: ProjectsRepository }) {
    this.projectsRepository = projectsRepository;
  }

  async create({ data }: { data: CreateProjectInput }): Promise<Result<Project>> {
    try {
      logger.info("[ProjectsController] Creating project", { name: data.name });
      return await this.projectsRepository.create({ data });
    } catch (error) {
      logger.error("[ProjectsController] Error creating project", { error });
      return failure("Error creating project", error);
    }
  }

  async getById({ id }: { id: string }): Promise<Result<Project>> {
    try {
      return await this.projectsRepository.getById({ id });
    } catch (error) {
      logger.error("[ProjectsController] Error getting project", { error });
      return failure("Error getting project", error);
    }
  }

  async update({ id, data }: { id: string; data: UpdateProjectInput }): Promise<Result<Project>> {
    try {
      logger.info("[ProjectsController] Updating project", { id });
      // Business logic goes here (validation, authorization, etc.)
      return await this.projectsRepository.update({ id, data });
    } catch (error) {
      logger.error("[ProjectsController] Error updating project", { error });
      return failure("Error updating project", error);
    }
  }

  async delete({ id }: { id: string }): Promise<Result<Project>> {
    try {
      logger.info("[ProjectsController] Deleting project", { id });
      return await this.projectsRepository.delete({ id });
    } catch (error) {
      logger.error("[ProjectsController] Error deleting project", { error });
      return failure("Error deleting project", error);
    }
  }

  async listByOrganization({ organizationId }: { organizationId: string }): Promise<Result<Project[]>> {
    try {
      return await this.projectsRepository.listByOrganization({ organizationId });
    } catch (error) {
      logger.error("[ProjectsController] Error listing projects", { error });
      return failure("Error listing projects", error);
    }
  }
}
```

**Key Points:**
- Controllers contain business logic only
- Inject dependencies via constructor using single parameter object
- Return `Result<T>` for all operations
- Add logging for debugging
- Keep controllers thin - complex logic should be in services

---

## 5. tRPC Router

**File**: `apps/server/src/routers/projects-router.ts`

```typescript
import { z } from "zod";
import { TRPCError } from "@trpc/server";
import { publicProcedure, router } from "@server/trpc";
import { Container } from "@server/di/container";
import { projectSchema } from "@server/types/project";
import { createProjectInputSchema } from "@server/types/project/create-project-input";
import { updateProjectInputSchema } from "@server/types/project/update-project-input";

export function createProjectsRouter(container: Container) {
  const { projectsController } = container;

  return router({
    create: publicProcedure
      .input(createProjectInputSchema)
      .output(z.object({ message: z.string(), data: projectSchema }))
      .mutation(async ({ input, ctx }) => {
        if (!ctx.auth.userId) {
          throw new TRPCError({ code: "UNAUTHORIZED", message: "Authentication required" });
        }

        const result = await projectsController.create({ data: input });

        if (!result.success) {
          throw new TRPCError({ code: "INTERNAL_SERVER_ERROR", message: result.message });
        }

        return { message: "Project created", data: result.data };
      }),

    getById: publicProcedure
      .input(z.object({ id: z.string().uuid() }))
      .output(z.object({ message: z.string(), data: projectSchema }))
      .query(async ({ input, ctx }) => {
        if (!ctx.auth.userId) {
          throw new TRPCError({ code: "UNAUTHORIZED", message: "Authentication required" });
        }

        const result = await projectsController.getById({ id: input.id });

        if (!result.success) {
          throw new TRPCError({ code: "NOT_FOUND", message: result.message });
        }

        return { message: "Project retrieved", data: result.data };
      }),

    update: publicProcedure
      .input(z.object({ id: z.string().uuid(), data: updateProjectInputSchema }))
      .output(z.object({ message: z.string(), data: projectSchema }))
      .mutation(async ({ input, ctx }) => {
        if (!ctx.auth.userId) {
          throw new TRPCError({ code: "UNAUTHORIZED", message: "Authentication required" });
        }

        const result = await projectsController.update({ id: input.id, data: input.data });

        if (!result.success) {
          throw new TRPCError({ code: "NOT_FOUND", message: result.message });
        }

        return { message: "Project updated", data: result.data };
      }),

    delete: publicProcedure
      .input(z.object({ id: z.string().uuid() }))
      .output(z.object({ message: z.string(), data: projectSchema }))
      .mutation(async ({ input, ctx }) => {
        if (!ctx.auth.userId) {
          throw new TRPCError({ code: "UNAUTHORIZED", message: "Authentication required" });
        }

        const result = await projectsController.delete({ id: input.id });

        if (!result.success) {
          throw new TRPCError({ code: "NOT_FOUND", message: result.message });
        }

        return { message: "Project deleted", data: result.data };
      }),

    listByOrganization: publicProcedure
      .input(z.object({ organizationId: z.string().uuid() }))
      .output(z.object({ message: z.string(), data: z.array(projectSchema) }))
      .query(async ({ input, ctx }) => {
        if (!ctx.auth.userId) {
          throw new TRPCError({ code: "UNAUTHORIZED", message: "Authentication required" });
        }

        const result = await projectsController.listByOrganization({ organizationId: input.organizationId });

        if (!result.success) {
          throw new TRPCError({ code: "INTERNAL_SERVER_ERROR", message: result.message });
        }

        return { message: "Projects retrieved", data: result.data };
      }),
  });
}
```

**Key Points:**
- Router is a factory function receiving the DI container
- Use `.input()` and `.output()` with Zod schemas
- Use `.query()` for GET operations, `.mutation()` for POST/PUT/DELETE
- Check `ctx.auth.userId` for authorization
- Check `result.success` and throw `TRPCError` on failure
- Return consistent format: `{ message: string, data: T }`

---

## 6. Register in App Router

**File**: `apps/server/src/routers/_app.ts`

```typescript
import { router } from "@server/trpc";
import { Container } from "@server/di/container";
import { createUserRouter } from "./user-router";
import { createProjectsRouter } from "./projects-router";

export function createAppRouter(container: Container) {
  return router({
    user: createUserRouter(container),
    projects: createProjectsRouter(container),  // Add new router
  });
}

export type AppRouter = ReturnType<typeof createAppRouter>;
```

---

## 7. Update DI Container

**File**: `apps/server/src/di/container.ts`

```typescript
import { PrismaClient } from "@server/generated/prisma";
import { UsersRepository } from "@server/repositories/UsersRepository";
import { UsersController } from "@server/controllers/UsersController";
import { ProjectsRepository } from "@server/repositories/ProjectsRepository";
import { ProjectsController } from "@server/controllers/ProjectsController";

export const createContainer = () => {
  const prisma = new PrismaClient();

  // Users
  const usersRepository = new UsersRepository({ prisma });
  const usersController = new UsersController({ usersRepository });

  // Projects
  const projectsRepository = new ProjectsRepository({ prisma });
  const projectsController = new ProjectsController({ projectsRepository });

  return {
    prisma,
    usersRepository,
    usersController,
    projectsRepository,
    projectsController,
  };
};

export type Container = ReturnType<typeof createContainer>;
```

**Dependency Graph:**
1. Prisma client created first
2. Repositories receive Prisma instance
3. Controllers receive repositories
4. Container exported with all dependencies

---

## Important Patterns

### Single Parameter Pattern

All functions accept a single object parameter:

```typescript
// Good
async update({ id, data }: { id: string; data: UpdateProjectInput }) {}

// Bad
async update(id: string, data: UpdateProjectInput) {}
```

### Result Type Pattern

Import from `common` package:

```typescript
import { Result, success, failure } from "common";

async getById({ id }: { id: string }): Promise<Result<Project>> {
  try {
    const data = await this.prisma.projects.findUnique({ where: { id } });
    if (!data) return failure("Not found");
    return success(mapPrismaProjectToProject(data));
  } catch (error) {
    return failure("Database error", error);
  }
}
```

### Mapping Function Pattern

Always create explicit mapping functions:

```typescript
// Good - explicit mapping function
function mapPrismaProjectToProject(prisma: PrismaProject): Project {
  return {
    id: prisma.id,
    organizationId: prisma.organization_id,
    name: prisma.name,
    createdAt: prisma.created_at,
    updatedAt: prisma.updated_at,
  };
}

// Bad - inline mapping repeated in every method
return success({
  id: result.id,
  organizationId: result.organization_id,
  // ...
});
```

### Soft Delete Pattern

For entities with soft deletes:

```typescript
// Repository methods filter by deleted_at
async getById({ id }: { id: string }): Promise<Result<Project>> {
  const result = await this.prisma.projects.findUnique({
    where: { id, deleted_at: null },
  });
  // ...
}

// Soft delete sets deleted_at instead of deleting
async softDelete({ id }: { id: string }): Promise<Result<Project>> {
  const result = await this.prisma.projects.update({
    where: { id },
    data: { deleted_at: new Date() },
  });
  // ...
}
```
