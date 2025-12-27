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
    └── {domain}.module.ts
```

## Setting Up DI

### Step 1: Define Symbols (`di/types.ts`)

Add symbol for each injectable component.

```typescript
import { IAuthenticationService } from '@/src/application/services/authentication.service.interface';
import { I{Entity}sRepository } from '@/src/application/repositories/{entity}s.repository.interface';
import { ICreate{Entity}UseCase } from '@/src/application/use-cases/{domain}/create-{entity}.use-case';
import { ICreate{Entity}Controller } from '@/src/interface-adapters/controllers/{domain}/create-{entity}.controller';

export const DI_SYMBOLS = {
  // Services
  IAuthenticationService: Symbol.for('IAuthenticationService'),
  IInstrumentationService: Symbol.for('IInstrumentationService'),
  ICrashReporterService: Symbol.for('ICrashReporterService'),
  ITransactionManagerService: Symbol.for('ITransactionManagerService'),

  // Repositories
  I{Entity}sRepository: Symbol.for('I{Entity}sRepository'),
  IUsersRepository: Symbol.for('IUsersRepository'),

  // Use Cases
  ICreate{Entity}UseCase: Symbol.for('ICreate{Entity}UseCase'),
  IUpdate{Entity}UseCase: Symbol.for('IUpdate{Entity}UseCase'),
  IDelete{Entity}UseCase: Symbol.for('IDelete{Entity}UseCase'),

  // Controllers
  ICreate{Entity}Controller: Symbol.for('ICreate{Entity}Controller'),
  IUpdate{Entity}Controller: Symbol.for('IUpdate{Entity}Controller'),
};

export interface DI_RETURN_TYPES {
  // Services
  IAuthenticationService: IAuthenticationService;
  IInstrumentationService: IInstrumentationService;
  ICrashReporterService: ICrashReporterService;
  ITransactionManagerService: ITransactionManagerService;

  // Repositories
  I{Entity}sRepository: I{Entity}sRepository;
  IUsersRepository: IUsersRepository;

  // Use Cases
  ICreate{Entity}UseCase: ICreate{Entity}UseCase;
  IUpdate{Entity}UseCase: IUpdate{Entity}UseCase;
  IDelete{Entity}UseCase: IDelete{Entity}UseCase;

  // Controllers
  ICreate{Entity}Controller: ICreate{Entity}Controller;
  IUpdate{Entity}Controller: IUpdate{Entity}Controller;
}
```

**Naming Convention:**
- Symbol key: `I{ComponentName}`
- Symbol value: `Symbol.for('I{ComponentName}')`
- Type mapping: Same key pointing to the interface type

### Step 2: Create Module (`di/modules/{domain}.module.ts`)

Group related bindings in a module.

```typescript
import { createModule } from '@evyweb/ioctopus';
import { DI_SYMBOLS } from '@/di/types';

// Infrastructure implementations
import { {Entity}sRepository } from '@/src/infrastructure/repositories/{entity}s.repository';
import { Mock{Entity}sRepository } from '@/src/infrastructure/repositories/{entity}s.repository.mock';

// Use cases
import { create{Entity}UseCase } from '@/src/application/use-cases/{domain}/create-{entity}.use-case';
import { update{Entity}UseCase } from '@/src/application/use-cases/{domain}/update-{entity}.use-case';

// Controllers
import { create{Entity}Controller } from '@/src/interface-adapters/controllers/{domain}/create-{entity}.controller';

export function create{Domain}Module() {
  const module = createModule();

  // Repository - use mock in test environment
  if (process.env.NODE_ENV === 'test') {
    module.bind(DI_SYMBOLS.I{Entity}sRepository).toClass(Mock{Entity}sRepository);
  } else {
    module
      .bind(DI_SYMBOLS.I{Entity}sRepository)
      .toClass({Entity}sRepository, [
        DI_SYMBOLS.IInstrumentationService,
        DI_SYMBOLS.ICrashReporterService,
      ]);
  }

  // Use Case - factory function
  module
    .bind(DI_SYMBOLS.ICreate{Entity}UseCase)
    .toHigherOrderFunction(create{Entity}UseCase, [
      DI_SYMBOLS.IInstrumentationService,
      DI_SYMBOLS.I{Entity}sRepository,
    ]);

  // Controller - factory function
  module
    .bind(DI_SYMBOLS.ICreate{Entity}Controller)
    .toHigherOrderFunction(create{Entity}Controller, [
      DI_SYMBOLS.IInstrumentationService,
      DI_SYMBOLS.IAuthenticationService,
      DI_SYMBOLS.ITransactionManagerService,
      DI_SYMBOLS.ICreate{Entity}UseCase,
    ]);

  return module;
}
```

### Step 3: Load Module (`di/container.ts`)

Register the module with the container.

```typescript
import { createContainer } from '@evyweb/ioctopus';
import { DI_RETURN_TYPES, DI_SYMBOLS } from '@/di/types';
import { create{Domain}Module } from '@/di/modules/{domain}.module';

const ApplicationContainer = createContainer();

// Load modules in dependency order
ApplicationContainer.load(Symbol('MonitoringModule'), createMonitoringModule());
ApplicationContainer.load(Symbol('DatabaseModule'), createTransactionManagerModule());
ApplicationContainer.load(Symbol('AuthenticationModule'), createAuthenticationModule());
ApplicationContainer.load(Symbol('UsersModule'), createUsersModule());
ApplicationContainer.load(Symbol('{Domain}Module'), create{Domain}Module());

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
module.bind(DI_SYMBOLS.I{Entity}sRepository).toClass({Entity}sRepository, [
  DI_SYMBOLS.IInstrumentationService,  // First constructor param
  DI_SYMBOLS.ICrashReporterService,    // Second constructor param
]);
```

### Factory Functions (Use Cases, Controllers)

Use `toHigherOrderFunction` for factory pattern.

```typescript
// Factory function signature:
// (dep1, dep2) => (input) => result

module.bind(DI_SYMBOLS.ICreate{Entity}UseCase).toHigherOrderFunction(
  create{Entity}UseCase,  // The factory function
  [
    DI_SYMBOLS.IInstrumentationService,  // First factory param
    DI_SYMBOLS.I{Entity}sRepository,     // Second factory param
  ]
);
```

## Usage

### In Server Actions (Frameworks Layer)

```typescript
'use server';
import { getInjection } from '@/di/container';

export async function create{Entity}(formData: FormData) {
  const controller = getInjection('ICreate{Entity}Controller');
  const sessionId = cookies().get(SESSION_COOKIE)?.value;

  await controller(Object.fromEntries(formData.entries()), sessionId);
}
```

### In Pages (Server Components)

```typescript
import { getInjection } from '@/di/container';

export default async function {Entity}sPage() {
  const getController = getInjection('IGet{Entity}sForUserController');
  const sessionId = cookies().get(SESSION_COOKIE)?.value;

  const items = await getController(sessionId);
  return <{Entity}List items={items} />;
}
```

## Testing

### Mock Implementations

Create mock classes in `src/infrastructure/{repositories,services}/*.mock.ts`.

```typescript
// src/infrastructure/repositories/{entity}s.repository.mock.ts
import { I{Entity}sRepository } from '@/src/application/repositories/{entity}s.repository.interface';

export class Mock{Entity}sRepository implements I{Entity}sRepository {
  private items: {Entity}[] = [];

  async create{Entity}(data: {Entity}Insert): Promise<{Entity}> {
    const newItem = { ...data, id: this.items.length + 1 };
    this.items.push(newItem);
    return newItem;
  }

  async get{Entity}(id: number): Promise<{Entity} | undefined> {
    return this.items.find(item => item.id === id);
  }
  // ... other mock methods
}
```

### Environment-Based Binding

```typescript
if (process.env.NODE_ENV === 'test') {
  module.bind(DI_SYMBOLS.I{Entity}sRepository).toClass(Mock{Entity}sRepository);
} else {
  module.bind(DI_SYMBOLS.I{Entity}sRepository).toClass({Entity}sRepository, [...]);
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
ApplicationContainer.load(Symbol('{Domain}Module'), create{Domain}Module());
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
