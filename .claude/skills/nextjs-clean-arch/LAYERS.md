# Clean Architecture Layers

Detailed documentation for each layer in the Clean Architecture implementation.

## Entities Layer (`src/entities/`)

The innermost layer containing domain models and errors. Has NO dependencies on other layers.

### Models (`src/entities/models/`)

Define data shapes using Zod schemas with validation rules.

```typescript
// src/entities/models/todo.ts
import { z } from 'zod';

export const selectTodoSchema = z.object({
  id: z.number(),
  todo: z.string(),
  completed: z.boolean(),
  userId: z.string(),
});
export type Todo = z.infer<typeof selectTodoSchema>;

export const insertTodoSchema = selectTodoSchema.pick({
  todo: true,
  userId: true,
  completed: true,
});
export type TodoInsert = z.infer<typeof insertTodoSchema>;
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
// src/application/use-cases/todos/create-todo.use-case.ts
import { InputParseError } from '@/src/entities/errors/common';
import type { Todo } from '@/src/entities/models/todo';
import type { IInstrumentationService } from '@/src/application/services/instrumentation.service.interface';
import type { ITodosRepository } from '@/src/application/repositories/todos.repository.interface';

export type ICreateTodoUseCase = ReturnType<typeof createTodoUseCase>;

export const createTodoUseCase =
  (
    instrumentationService: IInstrumentationService,
    todosRepository: ITodosRepository
  ) =>
  (
    input: { todo: string },
    userId: string,
    tx?: any
  ): Promise<Todo> => {
    return instrumentationService.startSpan(
      { name: 'createTodo Use Case', op: 'function' },
      async () => {
        // Authorization checks go here
        if (input.todo.length < 4) {
          throw new InputParseError('Todo must be at least 4 chars');
        }

        const newTodo = await todosRepository.createTodo(
          { todo: input.todo, userId, completed: false },
          tx
        );

        return newTodo;
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
// src/application/repositories/todos.repository.interface.ts
import type { Todo, TodoInsert } from '@/src/entities/models/todo';

export interface ITodosRepository {
  createTodo(todo: TodoInsert, tx?: any): Promise<Todo>;
  getTodo(id: number): Promise<Todo | undefined>;
  getTodosForUser(userId: string): Promise<Todo[]>;
  updateTodo(id: number, input: Partial<TodoInsert>, tx?: any): Promise<Todo>;
  deleteTodo(id: number, tx?: any): Promise<void>;
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
```

---

## Infrastructure Layer (`src/infrastructure/`)

Implements interfaces defined in Application layer using actual technologies.

### Repositories (`src/infrastructure/repositories/`)

Implement repository interfaces with database operations.

```typescript
// src/infrastructure/repositories/todos.repository.ts
import { eq } from 'drizzle-orm';
import { db, Transaction } from '@/drizzle';
import { todos } from '@/drizzle/schema';
import { ITodosRepository } from '@/src/application/repositories/todos.repository.interface';
import { DatabaseOperationError } from '@/src/entities/errors/common';
import { TodoInsert, Todo } from '@/src/entities/models/todo';

export class TodosRepository implements ITodosRepository {
  constructor(
    private readonly instrumentationService: IInstrumentationService,
    private readonly crashReporterService: ICrashReporterService
  ) {}

  async createTodo(todo: TodoInsert, tx?: Transaction): Promise<Todo> {
    const invoker = tx ?? db;
    try {
      const [created] = await invoker.insert(todos).values(todo).returning();
      if (created) return created;
      throw new DatabaseOperationError('Cannot create todo');
    } catch (err) {
      this.crashReporterService.report(err);
      throw err;
    }
  }
  // ... other methods
}
```

**Rules:**
- Class implementing the interface
- Receive dependencies via constructor
- Use database library (Drizzle) here only
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
    // ... rest of implementation
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
// src/interface-adapters/controllers/todos/create-todo.controller.ts
import { z } from 'zod';
import { ICreateTodoUseCase } from '@/src/application/use-cases/todos/create-todo.use-case';
import { UnauthenticatedError } from '@/src/entities/errors/auth';
import { InputParseError } from '@/src/entities/errors/common';

function presenter(todos: Todo[], instrumentationService: IInstrumentationService) {
  return instrumentationService.startSpan({ name: 'createTodo Presenter' }, () =>
    todos.map((todo) => ({
      id: todo.id,
      todo: todo.todo,
      userId: todo.userId,
      completed: todo.completed,
    }))
  );
}

const inputSchema = z.object({ todo: z.string().min(1) });

export type ICreateTodoController = ReturnType<typeof createTodoController>;

export const createTodoController =
  (
    instrumentationService: IInstrumentationService,
    authenticationService: IAuthenticationService,
    transactionManagerService: ITransactionManagerService,
    createTodoUseCase: ICreateTodoUseCase
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

    // 3. Orchestrate Use Cases
    const todos = await transactionManagerService.startTransaction(async (tx) => {
      return await Promise.all(
        data.todo.split(',').map((t) => createTodoUseCase({ todo: t.trim() }, user.id, tx))
      );
    });

    // 4. Present Response
    return presenter(todos ?? [], instrumentationService);
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
import { cookies } from 'next/headers';
import { SESSION_COOKIE } from '@/config';
import { getInjection } from '@/di/container';
import { UnauthenticatedError } from '@/src/entities/errors/auth';
import { InputParseError } from '@/src/entities/errors/common';

export async function createTodo(formData: FormData) {
  const instrumentationService = getInjection('IInstrumentationService');

  return await instrumentationService.instrumentServerAction('createTodo', {}, async () => {
    try {
      const data = Object.fromEntries(formData.entries());
      const sessionId = cookies().get(SESSION_COOKIE)?.value;
      const createTodoController = getInjection('ICreateTodoController');
      await createTodoController(data, sessionId);
    } catch (err) {
      if (err instanceof InputParseError) return { error: err.message };
      if (err instanceof UnauthenticatedError) return { error: 'Must be logged in' };

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
