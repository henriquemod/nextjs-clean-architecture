# Code Examples

Complete examples for implementing new features in Clean Architecture.

## Example: Adding a "Delete Todo" Feature

### 1. Entity Model (already exists)

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
```

### 2. Repository Interface (add method)

```typescript
// src/application/repositories/todos.repository.interface.ts
import type { Todo, TodoInsert } from '@/src/entities/models/todo';

export interface ITodosRepository {
  // ... existing methods
  deleteTodo(id: number, tx?: any): Promise<void>;
}
```

### 3. Use Case

```typescript
// src/application/use-cases/todos/delete-todo.use-case.ts
import { NotFoundError } from '@/src/entities/errors/common';
import type { IInstrumentationService } from '@/src/application/services/instrumentation.service.interface';
import type { ITodosRepository } from '@/src/application/repositories/todos.repository.interface';

export type IDeleteTodoUseCase = ReturnType<typeof deleteTodoUseCase>;

export const deleteTodoUseCase =
  (
    instrumentationService: IInstrumentationService,
    todosRepository: ITodosRepository
  ) =>
  async (todoId: number, userId: string, tx?: any): Promise<void> => {
    return instrumentationService.startSpan(
      { name: 'deleteTodo Use Case', op: 'function' },
      async () => {
        // Authorization: verify todo belongs to user
        const todo = await todosRepository.getTodo(todoId);

        if (!todo) {
          throw new NotFoundError('Todo not found');
        }

        if (todo.userId !== userId) {
          throw new NotFoundError('Todo not found'); // Don't leak existence
        }

        await todosRepository.deleteTodo(todoId, tx);
      }
    );
  };
```

### 4. Repository Implementation

```typescript
// src/infrastructure/repositories/todos.repository.ts
import { eq } from 'drizzle-orm';
import { db, Transaction } from '@/drizzle';
import { todos } from '@/drizzle/schema';
import { ITodosRepository } from '@/src/application/repositories/todos.repository.interface';

export class TodosRepository implements ITodosRepository {
  // ... constructor and other methods

  async deleteTodo(id: number, tx?: Transaction): Promise<void> {
    const invoker = tx ?? db;

    await this.instrumentationService.startSpan(
      { name: 'TodosRepository > deleteTodo' },
      async () => {
        try {
          await invoker.delete(todos).where(eq(todos.id, id));
        } catch (err) {
          this.crashReporterService.report(err);
          throw err;
        }
      }
    );
  }
}
```

### 5. Controller

```typescript
// src/interface-adapters/controllers/todos/delete-todo.controller.ts
import { z } from 'zod';
import { IDeleteTodoUseCase } from '@/src/application/use-cases/todos/delete-todo.use-case';
import { IAuthenticationService } from '@/src/application/services/authentication.service.interface';
import { IInstrumentationService } from '@/src/application/services/instrumentation.service.interface';
import { UnauthenticatedError } from '@/src/entities/errors/auth';
import { InputParseError } from '@/src/entities/errors/common';

const inputSchema = z.object({ todoId: z.number() });

export type IDeleteTodoController = ReturnType<typeof deleteTodoController>;

export const deleteTodoController =
  (
    instrumentationService: IInstrumentationService,
    authenticationService: IAuthenticationService,
    deleteTodoUseCase: IDeleteTodoUseCase
  ) =>
  async (
    input: Partial<z.infer<typeof inputSchema>>,
    sessionId: string | undefined
  ): Promise<void> => {
    return await instrumentationService.startSpan(
      { name: 'deleteTodo Controller' },
      async () => {
        // 1. Authentication
        if (!sessionId) {
          throw new UnauthenticatedError('Must be logged in');
        }
        const { user } = await authenticationService.validateSession(sessionId);

        // 2. Input Validation
        const { data, error } = inputSchema.safeParse(input);
        if (error) {
          throw new InputParseError('Invalid data', { cause: error });
        }

        // 3. Execute Use Case
        await deleteTodoUseCase(data.todoId, user.id);
      }
    );
  };
```

### 6. DI Types

```typescript
// di/types.ts
import { IDeleteTodoUseCase } from '@/src/application/use-cases/todos/delete-todo.use-case';
import { IDeleteTodoController } from '@/src/interface-adapters/controllers/todos/delete-todo.controller';

export const DI_SYMBOLS = {
  // ... existing symbols
  IDeleteTodoUseCase: Symbol.for('IDeleteTodoUseCase'),
  IDeleteTodoController: Symbol.for('IDeleteTodoController'),
};

export interface DI_RETURN_TYPES {
  // ... existing types
  IDeleteTodoUseCase: IDeleteTodoUseCase;
  IDeleteTodoController: IDeleteTodoController;
}
```

### 7. DI Module Bindings

```typescript
// di/modules/todos.module.ts
import { deleteTodoUseCase } from '@/src/application/use-cases/todos/delete-todo.use-case';
import { deleteTodoController } from '@/src/interface-adapters/controllers/todos/delete-todo.controller';

export function createTodosModule() {
  const todosModule = createModule();

  // ... existing bindings

  todosModule
    .bind(DI_SYMBOLS.IDeleteTodoUseCase)
    .toHigherOrderFunction(deleteTodoUseCase, [
      DI_SYMBOLS.IInstrumentationService,
      DI_SYMBOLS.ITodosRepository,
    ]);

  todosModule
    .bind(DI_SYMBOLS.IDeleteTodoController)
    .toHigherOrderFunction(deleteTodoController, [
      DI_SYMBOLS.IInstrumentationService,
      DI_SYMBOLS.IAuthenticationService,
      DI_SYMBOLS.IDeleteTodoUseCase,
    ]);

  return todosModule;
}
```

### 8. Server Action

```typescript
// app/actions.ts
'use server';
import { revalidatePath } from 'next/cache';
import { cookies } from 'next/headers';
import { SESSION_COOKIE } from '@/config';
import { getInjection } from '@/di/container';
import { UnauthenticatedError } from '@/src/entities/errors/auth';
import { NotFoundError, InputParseError } from '@/src/entities/errors/common';

export async function deleteTodo(todoId: number) {
  const instrumentationService = getInjection('IInstrumentationService');

  return await instrumentationService.instrumentServerAction(
    'deleteTodo',
    { recordResponse: true },
    async () => {
      try {
        const sessionId = cookies().get(SESSION_COOKIE)?.value;
        const deleteTodoController = getInjection('IDeleteTodoController');
        await deleteTodoController({ todoId }, sessionId);
      } catch (err) {
        if (err instanceof InputParseError) {
          return { error: err.message };
        }
        if (err instanceof UnauthenticatedError) {
          return { error: 'Must be logged in to delete a todo' };
        }
        if (err instanceof NotFoundError) {
          return { error: 'Todo not found' };
        }

        getInjection('ICrashReporterService').report(err);
        return { error: 'An error occurred. Please try again.' };
      }

      revalidatePath('/');
      return { success: true };
    }
  );
}
```

---

## Example: Adding a New Service

### 1. Service Interface

```typescript
// src/application/services/email.service.interface.ts
export interface IEmailService {
  sendWelcomeEmail(to: string, username: string): Promise<void>;
  sendPasswordResetEmail(to: string, resetToken: string): Promise<void>;
}
```

### 2. Service Implementation

```typescript
// src/infrastructure/services/email.service.ts
import { Resend } from 'resend';
import { IEmailService } from '@/src/application/services/email.service.interface';
import type { IInstrumentationService } from '@/src/application/services/instrumentation.service.interface';

export class EmailService implements IEmailService {
  private resend: Resend;

  constructor(
    private readonly instrumentationService: IInstrumentationService
  ) {
    this.resend = new Resend(process.env.RESEND_API_KEY);
  }

  async sendWelcomeEmail(to: string, username: string): Promise<void> {
    return this.instrumentationService.startSpan(
      { name: 'EmailService > sendWelcomeEmail' },
      async () => {
        await this.resend.emails.send({
          from: 'noreply@example.com',
          to,
          subject: 'Welcome!',
          html: `<p>Welcome, ${username}!</p>`,
        });
      }
    );
  }

  async sendPasswordResetEmail(to: string, resetToken: string): Promise<void> {
    // Implementation...
  }
}
```

### 3. Mock Implementation

```typescript
// src/infrastructure/services/email.service.mock.ts
import { IEmailService } from '@/src/application/services/email.service.interface';

export class MockEmailService implements IEmailService {
  public sentEmails: Array<{ to: string; type: string }> = [];

  async sendWelcomeEmail(to: string, username: string): Promise<void> {
    this.sentEmails.push({ to, type: 'welcome' });
  }

  async sendPasswordResetEmail(to: string, resetToken: string): Promise<void> {
    this.sentEmails.push({ to, type: 'password-reset' });
  }
}
```

### 4. DI Setup

```typescript
// di/types.ts
import { IEmailService } from '@/src/application/services/email.service.interface';

export const DI_SYMBOLS = {
  // ...
  IEmailService: Symbol.for('IEmailService'),
};

export interface DI_RETURN_TYPES {
  // ...
  IEmailService: IEmailService;
}
```

```typescript
// di/modules/email.module.ts
import { createModule } from '@evyweb/ioctopus';
import { DI_SYMBOLS } from '@/di/types';
import { EmailService } from '@/src/infrastructure/services/email.service';
import { MockEmailService } from '@/src/infrastructure/services/email.service.mock';

export function createEmailModule() {
  const emailModule = createModule();

  if (process.env.NODE_ENV === 'test') {
    emailModule.bind(DI_SYMBOLS.IEmailService).toClass(MockEmailService);
  } else {
    emailModule
      .bind(DI_SYMBOLS.IEmailService)
      .toClass(EmailService, [DI_SYMBOLS.IInstrumentationService]);
  }

  return emailModule;
}
```

---

## File Naming Conventions

| Component Type | File Pattern | Example |
| -------------- | ------------ | ------- |
| Model | `{name}.ts` | `todo.ts` |
| Error | `{domain}.ts` | `auth.ts`, `common.ts` |
| Repository Interface | `{name}.repository.interface.ts` | `todos.repository.interface.ts` |
| Service Interface | `{name}.service.interface.ts` | `email.service.interface.ts` |
| Repository Impl | `{name}.repository.ts` | `todos.repository.ts` |
| Repository Mock | `{name}.repository.mock.ts` | `todos.repository.mock.ts` |
| Service Impl | `{name}.service.ts` | `email.service.ts` |
| Service Mock | `{name}.service.mock.ts` | `email.service.mock.ts` |
| Use Case | `{action}.use-case.ts` | `create-todo.use-case.ts` |
| Controller | `{action}.controller.ts` | `create-todo.controller.ts` |
| DI Module | `{domain}.module.ts` | `todos.module.ts` |

## Directory Structure Template

```
src/
├── application/
│   ├── repositories/
│   │   └── {entity}.repository.interface.ts
│   ├── services/
│   │   └── {service}.service.interface.ts
│   └── use-cases/
│       └── {domain}/
│           └── {action}.use-case.ts
├── entities/
│   ├── errors/
│   │   └── {domain}.ts
│   └── models/
│       └── {entity}.ts
├── infrastructure/
│   ├── repositories/
│   │   ├── {entity}.repository.ts
│   │   └── {entity}.repository.mock.ts
│   └── services/
│       ├── {service}.service.ts
│       └── {service}.service.mock.ts
└── interface-adapters/
    └── controllers/
        └── {domain}/
            └── {action}.controller.ts

di/
├── container.ts
├── types.ts
└── modules/
    └── {domain}.module.ts

app/
├── actions.ts
├── {feature}/
│   ├── page.tsx
│   └── actions.ts
└── _components/
    └── {component}.tsx
```
