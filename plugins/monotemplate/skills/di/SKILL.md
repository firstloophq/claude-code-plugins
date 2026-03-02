---
name: di
description: Wire classes into the Inversify DI container correctly. Use when adding new repositories, controllers, services, or middleware to the server. Use when wiring dependencies, configuring the DI container, or understanding how classes are resolved.
---

# Inversify Dependency Injection

The monotemplate uses **Inversify v7** with `reflect-metadata` for decorator-based dependency injection. All DI configuration lives in `apps/server/src/di/container.ts`, which exports a singleton `container` instance.

## Making a Class Injectable

Every Repository, Controller, and Service class must be decorated with `@injectable()`. Constructor dependencies use `@inject(ClassName)` parameter decorators. Import the class itself (not `type`-only) so the class token is available at runtime.

```typescript
import { injectable, inject } from "inversify";
import { SomeRepository } from "@server/repositories/SomeRepository";

@injectable()
export class SomeController {
  constructor(@inject(SomeRepository) private someRepository: SomeRepository) {}
}
```

## Registering Bindings in `container.ts`

| Pattern | Use case | Example |
|---------|----------|---------|
| `toSelf().inSingletonScope()` | Classes that auto-resolve their own deps | Repositories, Controllers |
| `toConstantValue(value)` | External/constant values | PrismaClient, env-gated services |
| `to(ConcreteClass).inSingletonScope()` | Abstract to concrete | Auth factories |
| `bind(SYMBOL).toConstantValue(value)` (repeated) | Multi-bindings | Express middleware |

### Binding Order

Follow this order in `container.ts`:

```typescript
// Database
container.bind(PrismaClient).toConstantValue(prisma);

// Middleware (multi-binding via Symbol token)
container.bind(EXPRESS_MIDDLEWARE).toConstantValue(helmet());
container.bind(EXPRESS_MIDDLEWARE).toConstantValue(express.json());

// Auth (abstract → concrete, with simulated mode gating)
if (isSimulated()) {
    container.bind(AuthContextFactory).to(SimulatedAuthContextFactory).inSingletonScope();
} else {
    container.bind(AuthContextFactory).to(ClerkAuthContextFactory).inSingletonScope();
}

// Repositories
container.bind(FooRepository).toSelf().inSingletonScope();

// Controllers
container.bind(FooController).toSelf().inSingletonScope();
```

## Resolving Dependencies

- In routers/handlers: `container.get(ClassName)` — import both `container` and the class
- In `server.ts`: `container.get(AuthContextFactory)`, `container.getAll<T>(EXPRESS_MIDDLEWARE)`
- Never destructure from a container object; always use `container.get()`

```typescript
import { container } from "@server/di/container";
import { UsersController } from "@server/controllers/UsersController";

const result = await container.get(UsersController).getUserById(id);
```

## Symbol Tokens (`di/tokens.ts`)

Used for multi-bindings where multiple values share one key (e.g., `EXPRESS_MIDDLEWARE`).

```typescript
export const TOKEN_NAME = Symbol.for("TokenName");
```

## Simulated / E2E Auth Gating

- `__dev__/` directory code is dynamically imported inside `isSimulated()` checks (Bun macro)
- Never use static imports for `__dev__/` code in production paths

```typescript
if (isSimulated()) {
    const { SimulatedAuthContextFactory } = await import("@server/__dev__/SimulatedAuthContextFactory");
    container.bind(AuthContextFactory).to(SimulatedAuthContextFactory).inSingletonScope();
}
```

## Nullable / Env-Gated Services

When a service depends on an env variable that may not be set, bind it as `ServiceClass | null`:

```typescript
container.bind<StorageService | null>(StorageService).toConstantValue(
    env.SOME_KEY ? new StorageService({ ... }) : null,
);

// Resolve with explicit type annotation:
const svc = container.get<StorageService | null>(StorageService);
```

## tsconfig Requirements

- `experimentalDecorators: true` and `emitDecoratorMetadata: true` must be set
- `import "reflect-metadata"` must be the first import in the entry point (`index.ts`)

## New Entity Checklist

When adding a new Repository, Controller, or Service:

1. Add `@injectable()` to the class
2. Add `@inject(Dep)` to each constructor parameter
3. Import classes (not `type`-only) for injected dependencies
4. Register binding in `container.ts` under the appropriate section
5. In routers, resolve via `container.get(ClassName)`
