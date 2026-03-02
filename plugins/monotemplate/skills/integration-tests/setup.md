# Integration Test Setup

One-time setup guide for adding integration test infrastructure to a monotemplate project.

**Prerequisite:** Docker must be running locally. GitHub Actions `ubuntu-latest` has Docker by default.

## Directory Structure

```
apps/server/
├── vitest.integration.config.mts
└── tests/
    └── integration/
        ├── global-setup.ts
        ├── utils/
        │   ├── test-env.ts
        │   ├── db/
        │   │   ├── prisma-test-client.ts
        │   │   └── test-factories.ts
        │   └── api/
        │       └── auth-helper.ts
        ├── repositories/
        │   └── *.integration.test.ts
        └── api/
            └── *.integration.test.ts
```

## Dependencies

```bash
cd apps/server && bun add -d @testcontainers/postgresql testcontainers
```

vitest and supertest should already be in the template.

## Files to Create

### `apps/server/vitest.integration.config.mts`

Vitest config with:
- `fileParallelism: false` — tests run sequentially (shared DB)
- `globalSetup` pointing to `global-setup.ts`
- Dummy Clerk env vars so the server boots without real credentials
- `@server` path aliases matching `tsconfig.json`

### `apps/server/tests/integration/global-setup.ts`

- Starts a Postgres container via `@testcontainers/postgresql`
- Runs `prisma migrate deploy` against the container
- Sets `DATABASE_URL` env var for the test process
- Tears down the container in the `teardown` export

### `apps/server/tests/integration/utils/test-env.ts`

- Validates that `DATABASE_URL` is set (fails fast if global setup didn't run)

### `apps/server/tests/integration/utils/db/prisma-test-client.ts`

- `createTestPrismaClient()` — creates a PrismaClient connected to the test DB
- `truncateAllTables(prisma)` — truncates all tables in FK-safe order (cascade)

### `apps/server/tests/integration/utils/db/test-factories.ts`

- `seq()` — returns an auto-incrementing number, resets via `resetSeq()`
- Factory functions for each entity (e.g., `createUser()`, `createOrganization()`)

### `apps/server/tests/integration/utils/api/auth-helper.ts`

- Mocks `AuthContextFactory` via `container.rebindSync()`
- `setMockUserId(userId, orgId)` — sets the mock auth context
- `resetMockAuth()` — clears mock auth (requests will be unauthenticated)

## Package Scripts

**`apps/server/package.json`:**
```json
{
  "scripts": {
    "test:integration": "vitest run --config vitest.integration.config.mts"
  }
}
```

**Root `package.json`:**
```json
{
  "scripts": {
    "test:integration": "bun --filter server test:integration"
  }
}
```

## Config Updates

### `knip.json`

- Add `testcontainers` to `ignoreDependencies`
- Add `vitest.integration.config.mts` to the vitest config array
- Add `tests/**/*.ts` to entry and project arrays

### `tsconfig.json`

- Add `vitest.integration.config.mts` to `include`

## CI Integration

**`.github/workflows/pr-gate.yml`** — add an `integration-tests` job:

```yaml
integration-tests:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - uses: oven-sh/setup-bun@v2
    - run: bun install --frozen-lockfile
    - run: bun run test:integration
```

Docker is available on `ubuntu-latest` by default — no extra setup needed.
