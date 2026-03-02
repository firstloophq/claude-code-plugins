---
name: integration-tests
description: Add and manage integration tests for repositories and API endpoints. Use when adding integration tests, testing a repository, testing an API endpoint, or setting up integration test infrastructure.
---

# Integration Tests

Two test categories: **Repository tests** (direct DB via Prisma) and **API tests** (HTTP via supertest).

## Test Isolation

- `truncateAllTables()` in `beforeEach` — wipes all data between tests
- `resetSeq()` in `beforeEach` — resets the factory sequence counter for predictable IDs
- Testcontainers provides a throwaway Postgres instance per test run

## Repository Test Pattern

```typescript
import { PrismaClient } from "@server/generated/prisma";
import { SomeRepository } from "@server/repositories/SomeRepository";
import { createTestPrismaClient, truncateAllTables } from "../utils/db/prisma-test-client";
import { resetSeq } from "../utils/db/test-factories";

let prisma: PrismaClient;
let repo: SomeRepository;

beforeAll(() => {
    prisma = createTestPrismaClient();
    repo = new SomeRepository(prisma);
});

beforeEach(async () => {
    await truncateAllTables(prisma);
    resetSeq();
});

afterAll(async () => {
    await prisma.$disconnect();
});
```

Repository tests instantiate repos directly with the test PrismaClient — they bypass DI.

## API Test Pattern

```typescript
import request from "supertest";
import type { Express } from "express";
import { PrismaClient } from "@server/generated/prisma";
import { resetMockAuth, setMockUserId } from "../utils/api/auth-helper";
import createApp from "@server/server";
import { createTestPrismaClient, truncateAllTables } from "../utils/db/prisma-test-client";
import { resetSeq } from "../utils/db/test-factories";

let app: Express;
let prisma: PrismaClient;

beforeAll(async () => {
    app = createApp();
    prisma = createTestPrismaClient();
});

beforeEach(async () => {
    await truncateAllTables(prisma);
    resetSeq();
    resetMockAuth();
});

afterAll(async () => {
    await prisma.$disconnect();
});
```

API tests use the real Express app with mocked auth via DI container rebinding. Use `setMockUserId()` to set the authenticated user and `resetMockAuth()` to clear it.

## Factory Pattern

```typescript
export async function createEntity(
    prisma: PrismaClient,
    overrides: Partial<{ field1: string; field2: string }> = {},
) {
    const n = seq();
    return prisma.entities.create({
        data: {
            field1: overrides.field1 ?? `Default ${n}`,
            field2: overrides.field2 ?? `value-${n}`,
        },
    });
}
```

- `seq()` returns an auto-incrementing number for unique defaults
- `overrides` partial lets tests specify only the fields they care about

## Checklist for Adding Tests for a New Entity

1. Add factory function(s) to `tests/integration/utils/db/test-factories.ts`
2. Add the new table to `truncateAllTables()` in FK-safe order in `tests/integration/utils/db/prisma-test-client.ts`
3. Create `tests/integration/repositories/<Entity>Repository.integration.test.ts`
4. Create `tests/integration/api/<entity>.integration.test.ts`
5. Test file naming: `*.integration.test.ts`

## What to Test

- Happy path for each operation
- Auth rejection (API tests — request without `setMockUserId()`)
- Soft-delete exclusion (if entity uses soft deletes)
- Cross-user / cross-org isolation

## First-Time Setup

If integration test infrastructure doesn't exist yet, see [setup.md](setup.md) for the one-time setup guide.
