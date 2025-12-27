# Dependency Injection Guide

Setup and wiring guide for the ioctopus dependency injection container.

## Overview

The project uses [ioctopus](https://github.com/Evyweb/ioctopus) for dependency injection, which:
- Works on all runtimes (Edge, Node, Serverless)
- Doesn't require `reflect-metadata`
- Uses symbols for type-safe resolution

## Structure

```
di/
├── container.ts   # Main container and getInjection helper
├── types.ts       # Symbols and return type mappings
└── modules/       # Feature-based modules
    ├── authentication.module.ts
    ├── database.module.ts
    ├── monitoring.module.ts
    ├── todos.module.ts
    └── users.module.ts
```

## Setting Up DI

### Step 1: Define Symbols (`di/types.ts`)

Add symbol for each injectable component.

```typescript
import { IAuthenticationService } from '@/src/application/services/authentication.service.interface';
import { ICreateTodoUseCase } from '@/src/application/use-cases/todos/create-todo.use-case';
import { ICreateTodoController } from '@/src/interface-adapters/controllers/todos/create-todo.controller';

export const DI_SYMBOLS = {
  // Services
  IAuthenticationService: Symbol.for('IAuthenticationService'),

  // Repositories
  ITodosRepository: Symbol.for('ITodosRepository'),

  // Use Cases
  ICreateTodoUseCase: Symbol.for('ICreateTodoUseCase'),

  // Controllers
  ICreateTodoController: Symbol.for('ICreateTodoController'),
};

export interface DI_RETURN_TYPES {
  IAuthenticationService: IAuthenticationService;
  ITodosRepository: ITodosRepository;
  ICreateTodoUseCase: ICreateTodoUseCase;
  ICreateTodoController: ICreateTodoController;
}
```

**Naming Convention:**
- Symbol key: `I{ComponentName}`
- Symbol value: `Symbol.for('I{ComponentName}')`
- Type mapping: Same key pointing to the interface type

### Step 2: Create Module (`di/modules/{feature}.module.ts`)

Group related bindings in a module.

```typescript
import { createModule } from '@evyweb/ioctopus';
import { DI_SYMBOLS } from '@/di/types';

// Infrastructure implementations
import { TodosRepository } from '@/src/infrastructure/repositories/todos.repository';
import { MockTodosRepository } from '@/src/infrastructure/repositories/todos.repository.mock';

// Use cases
import { createTodoUseCase } from '@/src/application/use-cases/todos/create-todo.use-case';

// Controllers
import { createTodoController } from '@/src/interface-adapters/controllers/todos/create-todo.controller';

export function createTodosModule() {
  const todosModule = createModule();

  // Repository - use mock in test environment
  if (process.env.NODE_ENV === 'test') {
    todosModule.bind(DI_SYMBOLS.ITodosRepository).toClass(MockTodosRepository);
  } else {
    todosModule
      .bind(DI_SYMBOLS.ITodosRepository)
      .toClass(TodosRepository, [
        DI_SYMBOLS.IInstrumentationService,
        DI_SYMBOLS.ICrashReporterService,
      ]);
  }

  // Use Case - factory function
  todosModule
    .bind(DI_SYMBOLS.ICreateTodoUseCase)
    .toHigherOrderFunction(createTodoUseCase, [
      DI_SYMBOLS.IInstrumentationService,
      DI_SYMBOLS.ITodosRepository,
    ]);

  // Controller - factory function
  todosModule
    .bind(DI_SYMBOLS.ICreateTodoController)
    .toHigherOrderFunction(createTodoController, [
      DI_SYMBOLS.IInstrumentationService,
      DI_SYMBOLS.IAuthenticationService,
      DI_SYMBOLS.ITransactionManagerService,
      DI_SYMBOLS.ICreateTodoUseCase,
    ]);

  return todosModule;
}
```

### Step 3: Load Module (`di/container.ts`)

Register the module with the container.

```typescript
import { createContainer } from '@evyweb/ioctopus';
import { DI_RETURN_TYPES, DI_SYMBOLS } from '@/di/types';
import { createTodosModule } from '@/di/modules/todos.module';

const ApplicationContainer = createContainer();

// Load modules in dependency order
ApplicationContainer.load(Symbol('MonitoringModule'), createMonitoringModule());
ApplicationContainer.load(Symbol('DatabaseModule'), createTransactionManagerModule());
ApplicationContainer.load(Symbol('AuthenticationModule'), createAuthenticationModule());
ApplicationContainer.load(Symbol('UsersModule'), createUsersModule());
ApplicationContainer.load(Symbol('TodosModule'), createTodosModule());

export function getInjection<K extends keyof typeof DI_SYMBOLS>(
  symbol: K
): DI_RETURN_TYPES[K] {
  return ApplicationContainer.get(DI_SYMBOLS[symbol]);
}
```

## Binding Types

### Classes (Repositories, Services)

Use `toClass` for class-based implementations.

```typescript
// No dependencies
module.bind(DI_SYMBOLS.ISimpleService).toClass(SimpleService);

// With dependencies (constructor injection)
module.bind(DI_SYMBOLS.ITodosRepository).toClass(TodosRepository, [
  DI_SYMBOLS.IInstrumentationService,  // First constructor param
  DI_SYMBOLS.ICrashReporterService,    // Second constructor param
]);
```

### Factory Functions (Use Cases, Controllers)

Use `toHigherOrderFunction` for factory pattern.

```typescript
// Factory function signature:
// (dep1, dep2) => (input) => result

module.bind(DI_SYMBOLS.ICreateTodoUseCase).toHigherOrderFunction(
  createTodoUseCase,  // The factory function
  [
    DI_SYMBOLS.IInstrumentationService,  // First factory param
    DI_SYMBOLS.ITodosRepository,          // Second factory param
  ]
);
```

## Usage

### In Server Actions (Frameworks Layer)

```typescript
'use server';
import { getInjection } from '@/di/container';

export async function createTodo(formData: FormData) {
  const controller = getInjection('ICreateTodoController');
  const sessionId = cookies().get(SESSION_COOKIE)?.value;

  await controller(Object.fromEntries(formData.entries()), sessionId);
}
```

### In Pages (Server Components)

```typescript
import { getInjection } from '@/di/container';

export default async function TodosPage() {
  const getTodosController = getInjection('IGetTodosForUserController');
  const sessionId = cookies().get(SESSION_COOKIE)?.value;

  const todos = await getTodosController(sessionId);
  return <TodoList todos={todos} />;
}
```

## Testing

### Mock Implementations

Create mock classes in `src/infrastructure/{repositories,services}/*.mock.ts`.

```typescript
// src/infrastructure/repositories/todos.repository.mock.ts
import { ITodosRepository } from '@/src/application/repositories/todos.repository.interface';

export class MockTodosRepository implements ITodosRepository {
  private todos: Todo[] = [];

  async createTodo(todo: TodoInsert): Promise<Todo> {
    const newTodo = { ...todo, id: this.todos.length + 1 };
    this.todos.push(newTodo);
    return newTodo;
  }
  // ... other mock methods
}
```

### Environment-Based Binding

```typescript
if (process.env.NODE_ENV === 'test') {
  module.bind(DI_SYMBOLS.ITodosRepository).toClass(MockTodosRepository);
} else {
  module.bind(DI_SYMBOLS.ITodosRepository).toClass(TodosRepository, [...]);
}
```

## Module Loading Order

Load modules in dependency order (dependencies first):

```typescript
// 1. Core services (no dependencies)
ApplicationContainer.load(Symbol('MonitoringModule'), createMonitoringModule());

// 2. Database services
ApplicationContainer.load(Symbol('DatabaseModule'), createTransactionManagerModule());

// 3. Auth (depends on monitoring)
ApplicationContainer.load(Symbol('AuthenticationModule'), createAuthenticationModule());

// 4. Domain repositories (depend on monitoring, crash reporter)
ApplicationContainer.load(Symbol('UsersModule'), createUsersModule());

// 5. Feature modules (depend on repositories, services)
ApplicationContainer.load(Symbol('TodosModule'), createTodosModule());
```

## Checklist

When adding a new injectable component:

- [ ] Create interface in `src/application/{repositories,services}/`
- [ ] Create implementation in `src/infrastructure/{repositories,services}/`
- [ ] Create mock implementation for testing
- [ ] Add symbol to `DI_SYMBOLS` in `di/types.ts`
- [ ] Add type mapping to `DI_RETURN_TYPES`
- [ ] Bind in appropriate module in `di/modules/`
- [ ] Ensure module is loaded in `di/container.ts`
