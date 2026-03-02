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
- **Organizations use Clerk IDs** — `@id` without `@default(uuid())` for org table

**After editing schema:**
```bash
cd apps/server && bun run db:migrate
cd apps/server && bun run db:generate
```

---

## 2. Zod Types

**Location**: `packages/common/src/contracts/v1/<entity>/types.ts`

All types for an entity live in a single `types.ts` file, shared between frontend and backend.

```typescript
import { z } from "zod";

// Entity schema
export const projectSchema = z.object({
  id: z.string().uuid(),
  organizationId: z.string().min(1),
  name: z.string().min(1),
  createdAt: z.date(),
  updatedAt: z.date(),
});

export type Project = z.infer<typeof projectSchema>;

// Create input
export const createProjectInputSchema = z.object({
  name: z.string().min(1),
});

export type CreateProjectInput = z.infer<typeof createProjectInputSchema>;

// Update input
export const updateProjectInputSchema = z.object({
  name: z.string().min(1).optional(),
});

export type UpdateProjectInput = z.infer<typeof updateProjectInputSchema>;
```

**Key Points:**
- Use camelCase for TypeScript types (maps from snake_case in Prisma)
- Export both Zod schema and inferred TypeScript type
- Update input fields should be optional
- Don't include `organizationId` in create/update inputs — it comes from auth context

---

## 3. oRPC Contract

**File**: `packages/common/src/contracts/v1/<entity>/router.ts`

Contracts define the API shape (method, path, input/output schemas) shared between client and server.

```typescript
import { z } from "zod";
import { appErrors } from "../../errors";
import { projectSchema, createProjectInputSchema, updateProjectInputSchema } from "./types";

export const projectsContract = {
    listProjects: appErrors
        .route({ method: 'GET', path: '/v1/projects' })
        .output(z.object({ message: z.string(), data: z.array(projectSchema) })),
    createProject: appErrors
        .route({ method: 'POST', path: '/v1/projects' })
        .input(createProjectInputSchema)
        .output(z.object({ message: z.string(), data: projectSchema })),
    getProject: appErrors
        .route({ method: 'GET', path: '/v1/projects/{id}' })
        .input(z.object({ id: z.string().uuid() }))
        .output(z.object({ message: z.string(), data: projectSchema })),
    updateProject: appErrors
        .route({ method: 'PUT', path: '/v1/projects/{id}' })
        .input(updateProjectInputSchema.extend({ id: z.string().uuid() }))
        .output(z.object({ message: z.string(), data: projectSchema })),
    deleteProject: appErrors
        .route({ method: 'DELETE', path: '/v1/projects/{id}' })
        .input(z.object({ id: z.string().uuid() }))
        .output(z.object({ message: z.string() })),
};
```

**Key Points:**
- `appErrors` comes from `packages/common/src/contracts/errors.ts` — attaches shared error codes via `oc.errors()`
- `.route()` defines HTTP method and path
- Path params use `{param}` syntax
- All outputs follow `{ message: string, data: T }` format
- `.input()` is omitted when there's no request body (e.g., list endpoints)

**Register in app contract** (`packages/common/src/contracts/app-contract.ts`):
```typescript
import { projectsContract } from './v1/projects/router';

export const appContract = {
    ...usersContract,
    ...todosContract,
    ...projectsContract,
};
```

**Export from** `packages/common/src/contracts/index.ts`:
```typescript
export * from './v1/projects/types';
export * from './v1/projects/router';
```

---

## 4. Repository Layer

**File**: `apps/server/src/repositories/ProjectsRepository.ts`

```typescript
import { injectable, inject } from "inversify";
import { PrismaClient } from "@server/generated/prisma";
import { Result, success, failure } from "common";
import { Project } from "common";
import { CreateProjectInput } from "common";
import { UpdateProjectInput } from "common";

@injectable()
export class ProjectsRepository {
  constructor(@inject(PrismaClient) private prisma: PrismaClient) {}

  async create({ data }: { data: CreateProjectInput & { organizationId: string } }): Promise<Result<Project>> {
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
- `@injectable()` decorator on the class
- `@inject(PrismaClient)` on the constructor parameter
- **Always create a mapping function** (`mapPrisma[Entity]To[Entity]`)
- All methods return `Result<T>` for consistent error handling
- Use `success(data)` and `failure(message, error)` from `common` package
- All methods use single parameter objects: `{ id }`, `{ data }`, `{ id, data }`
- Repository only handles data access — no business logic
- For soft deletes, filter with `deleted_at: null` in queries

---

## 5. Controller Layer

**File**: `apps/server/src/controllers/ProjectsController.ts`

```typescript
import { injectable, inject } from "inversify";
import { Result, failure } from "common";
import { ProjectsRepository } from "@server/repositories/ProjectsRepository";
import { Project } from "common";
import { CreateProjectInput } from "common";
import { UpdateProjectInput } from "common";
import { logger } from "@server/common/utils/logger";

@injectable()
export class ProjectsController {
  constructor(@inject(ProjectsRepository) private projectsRepository: ProjectsRepository) {}

  async create({ organizationId, data }: { organizationId: string; data: CreateProjectInput }): Promise<Result<Project>> {
    try {
      logger.info("[ProjectsController] Creating project", { name: data.name });
      return await this.projectsRepository.create({ data: { ...data, organizationId } });
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
- `@injectable()` decorator on the class
- `@inject(ProjectsRepository)` on the constructor parameter — import the class (not `type`-only)
- Controllers contain business logic only
- Return `Result<T>` for all operations
- Add logging for debugging
- Keep controllers thin — complex logic should be in services

---

## 6. oRPC Router Implementation

**File**: `apps/server/src/routers/v1/<entity>/router.ts`

The router implementation wires the oRPC contract to the controller via DI.

```typescript
import { container } from "@server/di/container";
import { ProjectsController } from "@server/controllers/ProjectsController";
import { unwrapResult } from "common";
import { authed } from "../../orpc-middleware";

const projectsController = container.get(ProjectsController);

export const listProjects = authed.listProjects
    .handler(async ({ context }) => {
        const result = await projectsController.listByOrganization({ organizationId: context.orgId });
        return { message: "Projects retrieved", data: unwrapResult(result) };
    });

export const createProject = authed.createProject
    .handler(async ({ context, input }) => {
        const result = await projectsController.create({ organizationId: context.orgId, data: input });
        return { message: "Project created", data: unwrapResult(result) };
    });

export const getProject = authed.getProject
    .handler(async ({ input }) => {
        const result = await projectsController.getById({ id: input.id });
        return { message: "Project retrieved", data: unwrapResult(result) };
    });

export const updateProject = authed.updateProject
    .handler(async ({ input }) => {
        const { id, ...data } = input;
        const result = await projectsController.update({ id, data });
        return { message: "Project updated", data: unwrapResult(result) };
    });

export const deleteProject = authed.deleteProject
    .handler(async ({ input }) => {
        const result = await projectsController.delete({ id: input.id });
        unwrapResult(result);
        return { message: "Project deleted" };
    });
```

**Key Points:**
- Resolve controller from DI container at module level: `container.get(ProjectsController)`
- Use `authed.<handlerName>` from `orpc-middleware.ts` — handles auth automatically
- `context.orgId` and `context.userId` are available from the `authed` middleware
- `unwrapResult(result)` converts `Result<T>` — returns data on success, throws `ORPCError` on failure
- Each handler is a named export matching the contract key

**Register handlers in** `apps/server/src/routers/orpc.ts`:
```typescript
import { listProjects, createProject, getProject, updateProject, deleteProject } from './v1/projects/router';

const router = os.router({
    // ... existing handlers
    listProjects,
    createProject,
    getProject,
    updateProject,
    deleteProject,
});
```

---

## 7. DI Container Registration

**File**: `apps/server/src/di/container.ts`

```typescript
import { ProjectsRepository } from "@server/repositories/ProjectsRepository";
import { ProjectsController } from "@server/controllers/ProjectsController";

// Repositories
container.bind(ProjectsRepository).toSelf().inSingletonScope();

// Controllers
container.bind(ProjectsController).toSelf().inSingletonScope();
```

Add bindings under the appropriate section (Repositories, Controllers). See the `monotemplate:di` skill for full DI documentation.

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

### unwrapResult Pattern

Used in oRPC router handlers to convert `Result<T>` into thrown errors:

```typescript
import { unwrapResult } from "common";

// If result is success → returns result.data
// If result is failure → throws ORPCError with the failure message
const data = unwrapResult(result);
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
