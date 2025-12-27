# Clean Architecture Layers

Detailed documentation for each layer in the Clean Architecture implementation.

## Entities Layer (`src/entities/`)

The innermost layer containing domain models and errors. Has NO dependencies on other layers.

### Models (`src/entities/models/`)

Define data shapes using Zod schemas with validation rules.

```typescript
// src/entities/models/{entity}.ts
import { z } from 'zod';

export const select{Entity}Schema = z.object({
  id: z.number(),
  name: z.string().min(1),
  status: z.enum(['active', 'inactive']),
  userId: z.string(),
  createdAt: z.date(),
});
export type {Entity} = z.infer<typeof select{Entity}Schema>;

export const insert{Entity}Schema = select{Entity}Schema.pick({
  name: true,
  userId: true,
  status: true,
});
export type {Entity}Insert = z.infer<typeof insert{Entity}Schema>;
```

**Rules:**
- Use Zod for schema definition and validation
- Export both schema and inferred type
- Create separate schemas for select/insert/update operations
- Keep validation rules here (min length, format, etc.)

### Errors (`src/entities/errors/`)

Custom error classes for domain-specific errors.

```typescript
// src/entities/errors/common.ts
export class DatabaseOperationError extends Error {
  constructor(message: string, options?: ErrorOptions) {
    super(message, options);
  }
}

export class NotFoundError extends Error {
  constructor(message: string, options?: ErrorOptions) {
    super(message, options);
  }
}

export class InputParseError extends Error {
  constructor(message: string, options?: ErrorOptions) {
    super(message, options);
  }
}

// src/entities/errors/auth.ts
export class UnauthenticatedError extends Error {
  constructor(message: string, options?: ErrorOptions) {
    super(message, options);
  }
}

export class UnauthorizedError extends Error {
  constructor(message: string, options?: ErrorOptions) {
    super(message, options);
  }
}
```

**Rules:**
- Extend base `Error` class
- Include `ErrorOptions` for cause chaining
- Use descriptive names matching the error type
- Never throw library-specific errors outside infrastructure

---

## Application Layer (`src/application/`)

Contains business logic (use cases) and defines interfaces for repositories/services.

### Use Cases (`src/application/use-cases/`)

Each use case represents a single business operation.

```typescript
// src/application/use-cases/{domain}/create-{entity}.use-case.ts
import { InputParseError } from '@/src/entities/errors/common';
import type { {Entity} } from '@/src/entities/models/{entity}';
import type { IInstrumentationService } from '@/src/application/services/instrumentation.service.interface';
import type { I{Entity}sRepository } from '@/src/application/repositories/{entity}s.repository.interface';

export type ICreate{Entity}UseCase = ReturnType<typeof create{Entity}UseCase>;

export const create{Entity}UseCase =
  (
    instrumentationService: IInstrumentationService,
    {entity}sRepository: I{Entity}sRepository
  ) =>
  (
    input: { name: string },
    userId: string,
    tx?: any
  ): Promise<{Entity}> => {
    return instrumentationService.startSpan(
      { name: 'create{Entity} Use Case', op: 'function' },
      async () => {
        // Authorization checks go here
        // Example: check if user has permission, quota limits, etc.

        if (input.name.length < 3) {
          throw new InputParseError('Name must be at least 3 characters');
        }

        const newEntity = await {entity}sRepository.create{Entity}(
          { name: input.name, userId, status: 'active' },
          tx
        );

        return newEntity;
      }
    );
  };
```

**Rules:**
- Factory function pattern returning the operation
- Export the return type as `I{Name}UseCase`
- Receive dependencies via factory parameters
- Handle authorization (not authentication)
- Use repository/service interfaces, never implementations
- Never call other use cases

### Repository Interfaces (`src/application/repositories/`)

Define contracts for data access.

```typescript
// src/application/repositories/{entity}s.repository.interface.ts
import type { {Entity}, {Entity}Insert } from '@/src/entities/models/{entity}';

export interface I{Entity}sRepository {
  create{Entity}(data: {Entity}Insert, tx?: any): Promise<{Entity}>;
  get{Entity}(id: number): Promise<{Entity} | undefined>;
  get{Entity}sForUser(userId: string): Promise<{Entity}[]>;
  update{Entity}(id: number, input: Partial<{Entity}Insert>, tx?: any): Promise<{Entity}>;
  delete{Entity}(id: number, tx?: any): Promise<void>;
}
```

**Rules:**
- Interface only, no implementation
- Use entity models for parameters and returns
- Include transaction parameter where needed
- One interface per aggregate/entity

### Service Interfaces (`src/application/services/`)

Define contracts for external services.

```typescript
// src/application/services/authentication.service.interface.ts
import { Cookie } from '@/src/entities/models/cookie';
import { Session } from '@/src/entities/models/session';
import { User } from '@/src/entities/models/user';

export interface IAuthenticationService {
  generateUserId(): string;
  validateSession(sessionId: Session['id']): Promise<{ user: User; session: Session }>;
  validatePasswords(inputPassword: string, usersHashedPassword: string): Promise<boolean>;
  createSession(user: User): Promise<{ session: Session; cookie: Cookie }>;
  invalidateSession(sessionId: Session['id']): Promise<{ blankCookie: Cookie }>;
}

// src/application/services/email.service.interface.ts
export interface IEmailService {
  sendEmail(to: string, subject: string, body: string): Promise<void>;
  sendTemplatedEmail(to: string, template: string, data: Record<string, any>): Promise<void>;
}

// src/application/services/payment.service.interface.ts
export interface IPaymentService {
  createPaymentIntent(amount: number, currency: string): Promise<{ clientSecret: string }>;
  confirmPayment(paymentIntentId: string): Promise<boolean>;
}
```

---

## Infrastructure Layer (`src/infrastructure/`)

Implements interfaces defined in Application layer using actual technologies.

### Repositories (`src/infrastructure/repositories/`)

Implement repository interfaces with database operations.

```typescript
// src/infrastructure/repositories/{entity}s.repository.ts
import { eq } from 'drizzle-orm';
import { db, Transaction } from '@/drizzle';
import { {entity}s } from '@/drizzle/schema';
import { I{Entity}sRepository } from '@/src/application/repositories/{entity}s.repository.interface';
import { DatabaseOperationError } from '@/src/entities/errors/common';
import { {Entity}Insert, {Entity} } from '@/src/entities/models/{entity}';

export class {Entity}sRepository implements I{Entity}sRepository {
  constructor(
    private readonly instrumentationService: IInstrumentationService,
    private readonly crashReporterService: ICrashReporterService
  ) {}

  async create{Entity}(data: {Entity}Insert, tx?: Transaction): Promise<{Entity}> {
    const invoker = tx ?? db;
    try {
      const [created] = await invoker.insert({entity}s).values(data).returning();
      if (created) return created;
      throw new DatabaseOperationError('Cannot create {entity}');
    } catch (err) {
      this.crashReporterService.report(err);
      throw err;
    }
  }

  async get{Entity}(id: number): Promise<{Entity} | undefined> {
    try {
      return await db.query.{entity}s.findFirst({
        where: eq({entity}s.id, id),
      });
    } catch (err) {
      this.crashReporterService.report(err);
      throw err;
    }
  }

  // ... other methods follow same pattern
}
```

**Rules:**
- Class implementing the interface
- Receive dependencies via constructor
- Use database library (Drizzle, Prisma, etc.) here only
- Catch library errors, throw custom errors
- Support transaction parameter

### Services (`src/infrastructure/services/`)

Implement service interfaces with external libraries.

```typescript
// src/infrastructure/services/authentication.service.ts
import { Lucia } from 'lucia';
import { IAuthenticationService } from '@/src/application/services/authentication.service.interface';

export class AuthenticationService implements IAuthenticationService {
  private _lucia: Lucia;

  constructor(
    private readonly _usersRepository: IUsersRepository,
    private readonly _instrumentationService: IInstrumentationService
  ) {
    this._lucia = new Lucia(luciaAdapter, { /* config */ });
  }

  async validateSession(sessionId: string): Promise<{ user: User; session: Session }> {
    const result = await this._lucia.validateSession(sessionId);
    if (!result.user || !result.session) {
      throw new UnauthenticatedError('Unauthenticated');
    }
    const user = await this._usersRepository.getUser(result.user.id);
    if (!user) {
      throw new UnauthenticatedError("User doesn't exist");
    }
    return { user, session: result.session };
  }
  // ... other methods
}

// src/infrastructure/services/email.service.ts
import { Resend } from 'resend';
import { IEmailService } from '@/src/application/services/email.service.interface';

export class EmailService implements IEmailService {
  private resend: Resend;

  constructor(private readonly instrumentationService: IInstrumentationService) {
    this.resend = new Resend(process.env.RESEND_API_KEY);
  }

  async sendEmail(to: string, subject: string, body: string): Promise<void> {
    await this.resend.emails.send({ from: 'noreply@app.com', to, subject, html: body });
  }
  // ... other methods
}
```

---

## Interface Adapters Layer (`src/interface-adapters/`)

Contains controllers that bridge the outer framework layer with the application core.

### Controllers (`src/interface-adapters/controllers/`)

Orchestrate use cases and handle cross-cutting concerns.

```typescript
// src/interface-adapters/controllers/{domain}/create-{entity}.controller.ts
import { z } from 'zod';
import { ICreate{Entity}UseCase } from '@/src/application/use-cases/{domain}/create-{entity}.use-case';
import { UnauthenticatedError } from '@/src/entities/errors/auth';
import { InputParseError } from '@/src/entities/errors/common';

// Presenter: transforms domain model to API response
function presenter(entities: {Entity}[], instrumentationService: IInstrumentationService) {
  return instrumentationService.startSpan({ name: 'create{Entity} Presenter' }, () =>
    entities.map((entity) => ({
      id: entity.id,
      name: entity.name,
      status: entity.status,
      // Exclude sensitive fields like internal IDs, timestamps, etc.
    }))
  );
}

const inputSchema = z.object({ name: z.string().min(1) });

export type ICreate{Entity}Controller = ReturnType<typeof create{Entity}Controller>;

export const create{Entity}Controller =
  (
    instrumentationService: IInstrumentationService,
    authenticationService: IAuthenticationService,
    transactionManagerService: ITransactionManagerService,
    create{Entity}UseCase: ICreate{Entity}UseCase
  ) =>
  async (
    input: Partial<z.infer<typeof inputSchema>>,
    sessionId: string | undefined
  ): Promise<ReturnType<typeof presenter>> => {
    // 1. Authentication
    if (!sessionId) throw new UnauthenticatedError('Must be logged in');
    const { user } = await authenticationService.validateSession(sessionId);

    // 2. Input Validation
    const { data, error } = inputSchema.safeParse(input);
    if (error) throw new InputParseError('Invalid data', { cause: error });

    // 3. Orchestrate Use Cases (with transaction if needed)
    const entities = await transactionManagerService.startTransaction(async (tx) => {
      return await create{Entity}UseCase(data, user.id, tx);
    });

    // 4. Present Response
    return presenter([entities], instrumentationService);
  };
```

**Controller Responsibilities:**
1. **Authentication** - Validate session exists
2. **Input Validation** - Parse and validate input with Zod
3. **Orchestration** - Call use cases (potentially in transactions)
4. **Presentation** - Format response for the consumer

---

## Frameworks & Drivers Layer (`app/`)

Next.js pages, server actions, and components. Only interacts with the system via DI container.

### Server Actions (`app/actions.ts`)

Entry points that call controllers from DI container.

```typescript
'use server';
import { revalidatePath } from 'next/cache';
import { cookies } from 'next/headers';
import { SESSION_COOKIE } from '@/config';
import { getInjection } from '@/di/container';
import { UnauthenticatedError } from '@/src/entities/errors/auth';
import { InputParseError, NotFoundError } from '@/src/entities/errors/common';

export async function create{Entity}(formData: FormData) {
  const instrumentationService = getInjection('IInstrumentationService');

  return await instrumentationService.instrumentServerAction('create{Entity}', {}, async () => {
    try {
      const data = Object.fromEntries(formData.entries());
      const sessionId = cookies().get(SESSION_COOKIE)?.value;
      const controller = getInjection('ICreate{Entity}Controller');
      await controller(data, sessionId);
    } catch (err) {
      if (err instanceof InputParseError) return { error: err.message };
      if (err instanceof UnauthenticatedError) return { error: 'Must be logged in' };
      if (err instanceof NotFoundError) return { error: 'Not found' };

      getInjection('ICrashReporterService').report(err);
      return { error: 'An error occurred. Please try again.' };
    }

    revalidatePath('/');
    return { success: true };
  });
}
```

**Rules:**
- Use `'use server'` directive
- Get controllers via `getInjection()`
- Handle errors from entities layer
- Never import from infrastructure or use-cases directly
