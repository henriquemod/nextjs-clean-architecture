# Code Examples

Complete examples for implementing new features in Clean Architecture.

Replace `{Entity}`, `{entity}`, `{Domain}`, `{domain}` with your actual entity and domain names (e.g., `Product`, `product`, `Catalog`, `catalog`).

---

## Example: Adding a Complete CRUD Feature

This example walks through adding a new entity with create, read, update, and delete operations.

### 1. Entity Model

```typescript
// src/entities/models/{entity}.ts
import { z } from 'zod';

export const select{Entity}Schema = z.object({
  id: z.number(),
  name: z.string().min(1),
  description: z.string().optional(),
  status: z.enum(['active', 'inactive', 'archived']),
  userId: z.string(),
  createdAt: z.date(),
  updatedAt: z.date(),
});
export type {Entity} = z.infer<typeof select{Entity}Schema>;

export const insert{Entity}Schema = select{Entity}Schema.omit({
  id: true,
  createdAt: true,
  updatedAt: true,
});
export type {Entity}Insert = z.infer<typeof insert{Entity}Schema>;

export const update{Entity}Schema = insert{Entity}Schema.partial();
export type {Entity}Update = z.infer<typeof update{Entity}Schema>;
```

### 2. Custom Errors (if needed)

```typescript
// src/entities/errors/{domain}.ts
export class {Entity}NotFoundError extends Error {
  constructor(id: number) {
    super(`{Entity} with id ${id} not found`);
  }
}

export class {Entity}LimitExceededError extends Error {
  constructor(limit: number) {
    super(`Maximum of ${limit} {entity}s allowed`);
  }
}
```

### 3. Repository Interface

```typescript
// src/application/repositories/{entity}s.repository.interface.ts
import type { {Entity}, {Entity}Insert, {Entity}Update } from '@/src/entities/models/{entity}';

export interface I{Entity}sRepository {
  create(data: {Entity}Insert, tx?: any): Promise<{Entity}>;
  getById(id: number): Promise<{Entity} | undefined>;
  getByUserId(userId: string): Promise<{Entity}[]>;
  update(id: number, data: {Entity}Update, tx?: any): Promise<{Entity}>;
  delete(id: number, tx?: any): Promise<void>;
  count(userId: string): Promise<number>;
}
```

### 4. Use Cases

```typescript
// src/application/use-cases/{domain}/create-{entity}.use-case.ts
import { {Entity}LimitExceededError } from '@/src/entities/errors/{domain}';
import type { {Entity} } from '@/src/entities/models/{entity}';
import type { IInstrumentationService } from '@/src/application/services/instrumentation.service.interface';
import type { I{Entity}sRepository } from '@/src/application/repositories/{entity}s.repository.interface';

const MAX_ENTITIES_PER_USER = 100;

export type ICreate{Entity}UseCase = ReturnType<typeof create{Entity}UseCase>;

export const create{Entity}UseCase =
  (
    instrumentationService: IInstrumentationService,
    repository: I{Entity}sRepository
  ) =>
  async (
    input: { name: string; description?: string },
    userId: string,
    tx?: any
  ): Promise<{Entity}> => {
    return instrumentationService.startSpan(
      { name: 'create{Entity} Use Case', op: 'function' },
      async () => {
        // Authorization: check user quota
        const count = await repository.count(userId);
        if (count >= MAX_ENTITIES_PER_USER) {
          throw new {Entity}LimitExceededError(MAX_ENTITIES_PER_USER);
        }

        return await repository.create(
          {
            name: input.name,
            description: input.description,
            status: 'active',
            userId,
          },
          tx
        );
      }
    );
  };

// src/application/use-cases/{domain}/get-{entity}.use-case.ts
export type IGet{Entity}UseCase = ReturnType<typeof get{Entity}UseCase>;

export const get{Entity}UseCase =
  (
    instrumentationService: IInstrumentationService,
    repository: I{Entity}sRepository
  ) =>
  async (id: number, userId: string): Promise<{Entity}> => {
    return instrumentationService.startSpan(
      { name: 'get{Entity} Use Case', op: 'function' },
      async () => {
        const entity = await repository.getById(id);

        if (!entity || entity.userId !== userId) {
          throw new {Entity}NotFoundError(id);
        }

        return entity;
      }
    );
  };

// src/application/use-cases/{domain}/update-{entity}.use-case.ts
export type IUpdate{Entity}UseCase = ReturnType<typeof update{Entity}UseCase>;

export const update{Entity}UseCase =
  (
    instrumentationService: IInstrumentationService,
    repository: I{Entity}sRepository
  ) =>
  async (
    id: number,
    input: { name?: string; description?: string; status?: string },
    userId: string,
    tx?: any
  ): Promise<{Entity}> => {
    return instrumentationService.startSpan(
      { name: 'update{Entity} Use Case', op: 'function' },
      async () => {
        // Authorization: verify ownership
        const existing = await repository.getById(id);
        if (!existing || existing.userId !== userId) {
          throw new {Entity}NotFoundError(id);
        }

        return await repository.update(id, input, tx);
      }
    );
  };

// src/application/use-cases/{domain}/delete-{entity}.use-case.ts
export type IDelete{Entity}UseCase = ReturnType<typeof delete{Entity}UseCase>;

export const delete{Entity}UseCase =
  (
    instrumentationService: IInstrumentationService,
    repository: I{Entity}sRepository
  ) =>
  async (id: number, userId: string, tx?: any): Promise<void> => {
    return instrumentationService.startSpan(
      { name: 'delete{Entity} Use Case', op: 'function' },
      async () => {
        // Authorization: verify ownership
        const existing = await repository.getById(id);
        if (!existing || existing.userId !== userId) {
          throw new {Entity}NotFoundError(id);
        }

        await repository.delete(id, tx);
      }
    );
  };
```

### 5. Repository Implementation

```typescript
// src/infrastructure/repositories/{entity}s.repository.ts
import { eq, and, count as drizzleCount } from 'drizzle-orm';
import { db, Transaction } from '@/drizzle';
import { {entity}s } from '@/drizzle/schema';
import { I{Entity}sRepository } from '@/src/application/repositories/{entity}s.repository.interface';
import { DatabaseOperationError } from '@/src/entities/errors/common';
import type { {Entity}, {Entity}Insert, {Entity}Update } from '@/src/entities/models/{entity}';
import type { IInstrumentationService } from '@/src/application/services/instrumentation.service.interface';
import type { ICrashReporterService } from '@/src/application/services/crash-reporter.service.interface';

export class {Entity}sRepository implements I{Entity}sRepository {
  constructor(
    private readonly instrumentationService: IInstrumentationService,
    private readonly crashReporterService: ICrashReporterService
  ) {}

  async create(data: {Entity}Insert, tx?: Transaction): Promise<{Entity}> {
    const invoker = tx ?? db;
    return this.instrumentationService.startSpan(
      { name: '{Entity}sRepository > create' },
      async () => {
        try {
          const [created] = await invoker
            .insert({entity}s)
            .values(data)
            .returning();
          if (!created) throw new DatabaseOperationError('Failed to create {entity}');
          return created;
        } catch (err) {
          this.crashReporterService.report(err);
          throw err;
        }
      }
    );
  }

  async getById(id: number): Promise<{Entity} | undefined> {
    return this.instrumentationService.startSpan(
      { name: '{Entity}sRepository > getById' },
      async () => {
        try {
          return await db.query.{entity}s.findFirst({
            where: eq({entity}s.id, id),
          });
        } catch (err) {
          this.crashReporterService.report(err);
          throw err;
        }
      }
    );
  }

  async getByUserId(userId: string): Promise<{Entity}[]> {
    return this.instrumentationService.startSpan(
      { name: '{Entity}sRepository > getByUserId' },
      async () => {
        try {
          return await db.query.{entity}s.findMany({
            where: eq({entity}s.userId, userId),
          });
        } catch (err) {
          this.crashReporterService.report(err);
          throw err;
        }
      }
    );
  }

  async update(id: number, data: {Entity}Update, tx?: Transaction): Promise<{Entity}> {
    const invoker = tx ?? db;
    return this.instrumentationService.startSpan(
      { name: '{Entity}sRepository > update' },
      async () => {
        try {
          const [updated] = await invoker
            .update({entity}s)
            .set({ ...data, updatedAt: new Date() })
            .where(eq({entity}s.id, id))
            .returning();
          return updated;
        } catch (err) {
          this.crashReporterService.report(err);
          throw err;
        }
      }
    );
  }

  async delete(id: number, tx?: Transaction): Promise<void> {
    const invoker = tx ?? db;
    await this.instrumentationService.startSpan(
      { name: '{Entity}sRepository > delete' },
      async () => {
        try {
          await invoker.delete({entity}s).where(eq({entity}s.id, id));
        } catch (err) {
          this.crashReporterService.report(err);
          throw err;
        }
      }
    );
  }

  async count(userId: string): Promise<number> {
    return this.instrumentationService.startSpan(
      { name: '{Entity}sRepository > count' },
      async () => {
        try {
          const [result] = await db
            .select({ count: drizzleCount() })
            .from({entity}s)
            .where(eq({entity}s.userId, userId));
          return result?.count ?? 0;
        } catch (err) {
          this.crashReporterService.report(err);
          throw err;
        }
      }
    );
  }
}
```

### 6. Controller (Create example)

```typescript
// src/interface-adapters/controllers/{domain}/create-{entity}.controller.ts
import { z } from 'zod';
import { ICreate{Entity}UseCase } from '@/src/application/use-cases/{domain}/create-{entity}.use-case';
import { IAuthenticationService } from '@/src/application/services/authentication.service.interface';
import { IInstrumentationService } from '@/src/application/services/instrumentation.service.interface';
import { UnauthenticatedError } from '@/src/entities/errors/auth';
import { InputParseError } from '@/src/entities/errors/common';
import type { {Entity} } from '@/src/entities/models/{entity}';

function presenter(entity: {Entity}) {
  return {
    id: entity.id,
    name: entity.name,
    description: entity.description,
    status: entity.status,
    createdAt: entity.createdAt.toISOString(),
  };
}

const inputSchema = z.object({
  name: z.string().min(1).max(100),
  description: z.string().max(500).optional(),
});

export type ICreate{Entity}Controller = ReturnType<typeof create{Entity}Controller>;

export const create{Entity}Controller =
  (
    instrumentationService: IInstrumentationService,
    authenticationService: IAuthenticationService,
    create{Entity}UseCase: ICreate{Entity}UseCase
  ) =>
  async (
    input: unknown,
    sessionId: string | undefined
  ): Promise<ReturnType<typeof presenter>> => {
    return instrumentationService.startSpan(
      { name: 'create{Entity} Controller' },
      async () => {
        // 1. Authentication
        if (!sessionId) {
          throw new UnauthenticatedError('Must be logged in');
        }
        const { user } = await authenticationService.validateSession(sessionId);

        // 2. Input Validation
        const { data, error } = inputSchema.safeParse(input);
        if (error) {
          throw new InputParseError('Invalid input', { cause: error });
        }

        // 3. Execute Use Case
        const entity = await create{Entity}UseCase(data, user.id);

        // 4. Present Response
        return presenter(entity);
      }
    );
  };
```

### 7. DI Setup

```typescript
// di/types.ts - Add these entries
export const DI_SYMBOLS = {
  // ... existing
  I{Entity}sRepository: Symbol.for('I{Entity}sRepository'),
  ICreate{Entity}UseCase: Symbol.for('ICreate{Entity}UseCase'),
  IGet{Entity}UseCase: Symbol.for('IGet{Entity}UseCase'),
  IUpdate{Entity}UseCase: Symbol.for('IUpdate{Entity}UseCase'),
  IDelete{Entity}UseCase: Symbol.for('IDelete{Entity}UseCase'),
  ICreate{Entity}Controller: Symbol.for('ICreate{Entity}Controller'),
  // ... add other controllers
};

// di/modules/{domain}.module.ts
import { createModule } from '@evyweb/ioctopus';
import { DI_SYMBOLS } from '@/di/types';
import { {Entity}sRepository } from '@/src/infrastructure/repositories/{entity}s.repository';
import { create{Entity}UseCase } from '@/src/application/use-cases/{domain}/create-{entity}.use-case';
import { create{Entity}Controller } from '@/src/interface-adapters/controllers/{domain}/create-{entity}.controller';

export function create{Domain}Module() {
  const module = createModule();

  module.bind(DI_SYMBOLS.I{Entity}sRepository).toClass({Entity}sRepository, [
    DI_SYMBOLS.IInstrumentationService,
    DI_SYMBOLS.ICrashReporterService,
  ]);

  module.bind(DI_SYMBOLS.ICreate{Entity}UseCase).toHigherOrderFunction(
    create{Entity}UseCase,
    [DI_SYMBOLS.IInstrumentationService, DI_SYMBOLS.I{Entity}sRepository]
  );

  module.bind(DI_SYMBOLS.ICreate{Entity}Controller).toHigherOrderFunction(
    create{Entity}Controller,
    [
      DI_SYMBOLS.IInstrumentationService,
      DI_SYMBOLS.IAuthenticationService,
      DI_SYMBOLS.ICreate{Entity}UseCase,
    ]
  );

  // Add other use cases and controllers...

  return module;
}
```

### 8. Server Action

```typescript
// app/{domain}/actions.ts
'use server';
import { revalidatePath } from 'next/cache';
import { cookies } from 'next/headers';
import { SESSION_COOKIE } from '@/config';
import { getInjection } from '@/di/container';
import { UnauthenticatedError } from '@/src/entities/errors/auth';
import { InputParseError, NotFoundError } from '@/src/entities/errors/common';
import { {Entity}LimitExceededError } from '@/src/entities/errors/{domain}';

export async function create{Entity}(formData: FormData) {
  const instrumentationService = getInjection('IInstrumentationService');

  return await instrumentationService.instrumentServerAction(
    'create{Entity}',
    { recordResponse: true },
    async () => {
      try {
        const data = Object.fromEntries(formData.entries());
        const sessionId = cookies().get(SESSION_COOKIE)?.value;
        const controller = getInjection('ICreate{Entity}Controller');
        const result = await controller(data, sessionId);
        revalidatePath('/{domain}');
        return { success: true, data: result };
      } catch (err) {
        if (err instanceof InputParseError) {
          return { error: err.message };
        }
        if (err instanceof UnauthenticatedError) {
          return { error: 'Must be logged in' };
        }
        if (err instanceof {Entity}LimitExceededError) {
          return { error: err.message };
        }

        getInjection('ICrashReporterService').report(err);
        return { error: 'An unexpected error occurred' };
      }
    }
  );
}
```

---

## File Naming Conventions

| Component Type | File Pattern | Example |
| -------------- | ------------ | ------- |
| Model | `{entity}.ts` | `product.ts` |
| Error | `{domain}.ts` | `catalog.ts` |
| Repository Interface | `{entity}s.repository.interface.ts` | `products.repository.interface.ts` |
| Service Interface | `{name}.service.interface.ts` | `payment.service.interface.ts` |
| Repository Impl | `{entity}s.repository.ts` | `products.repository.ts` |
| Repository Mock | `{entity}s.repository.mock.ts` | `products.repository.mock.ts` |
| Service Impl | `{name}.service.ts` | `payment.service.ts` |
| Service Mock | `{name}.service.mock.ts` | `payment.service.mock.ts` |
| Use Case | `{action}-{entity}.use-case.ts` | `create-product.use-case.ts` |
| Controller | `{action}-{entity}.controller.ts` | `create-product.controller.ts` |
| DI Module | `{domain}.module.ts` | `catalog.module.ts` |

---

## Directory Structure Template

```
src/
├── application/
│   ├── repositories/
│   │   └── {entity}s.repository.interface.ts
│   ├── services/
│   │   └── {service}.service.interface.ts
│   └── use-cases/
│       └── {domain}/
│           ├── create-{entity}.use-case.ts
│           ├── get-{entity}.use-case.ts
│           ├── update-{entity}.use-case.ts
│           └── delete-{entity}.use-case.ts
├── entities/
│   ├── errors/
│   │   ├── common.ts
│   │   ├── auth.ts
│   │   └── {domain}.ts
│   └── models/
│       └── {entity}.ts
├── infrastructure/
│   ├── repositories/
│   │   ├── {entity}s.repository.ts
│   │   └── {entity}s.repository.mock.ts
│   └── services/
│       ├── {service}.service.ts
│       └── {service}.service.mock.ts
└── interface-adapters/
    └── controllers/
        └── {domain}/
            ├── create-{entity}.controller.ts
            ├── get-{entity}.controller.ts
            ├── update-{entity}.controller.ts
            └── delete-{entity}.controller.ts

di/
├── container.ts
├── types.ts
└── modules/
    └── {domain}.module.ts

app/
├── {domain}/
│   ├── page.tsx
│   └── actions.ts
└── _components/
    └── {domain}/
        └── {component}.tsx
```
