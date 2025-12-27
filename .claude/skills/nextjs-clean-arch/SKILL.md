---
name: nextjs-clean-arch
description: Clean Architecture implementation for Next.js projects. Use when building or modifying Next.js apps with layered architecture, dependency injection, and separation of concerns.
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# Next.js Clean Architecture

Implements Clean Architecture in Next.js applications with proper layer separation, dependency injection via ioctopus, and ESLint-enforced boundaries.

## Core Principles

1. **Dependency Rule**: Layers depend only on layers BELOW them, never above
2. **Interface Segregation**: Infrastructure implements interfaces from Application layer
3. **Dependency Injection**: Use ioctopus container to wire dependencies
4. **Single Responsibility**: Each component has ONE job (controller orchestrates, use case executes logic, repository handles data)

## Layer Hierarchy (Outer to Inner)

```
app/              → Frameworks & Drivers (Next.js pages, actions, components)
    ↓ uses
interface-adapters/  → Controllers (orchestration, auth, validation, presenters)
    ↓ uses
application/      → Use Cases + Repository/Service Interfaces
    ↓ uses
entities/         → Models (Zod schemas) + Custom Errors
    ↑ implements
infrastructure/   → Repository/Service Implementations
    ↑ wires
di/               → Dependency Injection Container
```

## Workflow

### Step 1: Identify the Layer

| Task | Layer | Location |
| ---- | ----- | -------- |
| Add page/action | Frameworks & Drivers | `app/` |
| Add controller | Interface Adapters | `src/interface-adapters/controllers/` |
| Add business logic | Application | `src/application/use-cases/` |
| Add data model | Entities | `src/entities/models/` |
| Add custom error | Entities | `src/entities/errors/` |
| Add DB operation | Infrastructure | `src/infrastructure/repositories/` |
| Add external service | Infrastructure | `src/infrastructure/services/` |

### Step 2: Follow Layer Patterns

**Controllers** (Interface Adapters):
- Factory function returning async handler
- Receive pre-injected dependencies
- Perform authentication + input validation
- Orchestrate use cases
- Use presenters for response formatting

**Use Cases** (Application):
- Factory function returning operation function
- Receive repository/service interfaces
- Contain authorization logic
- One operation per use case (no use case calling another)

**Repositories/Services** (Infrastructure):
- Classes implementing interfaces from Application layer
- Handle database/external service operations
- Catch library errors, throw custom Entities errors

### Step 3: Wire Dependencies

1. Add interface to `src/application/{repositories,services}/`
2. Add implementation to `src/infrastructure/{repositories,services}/`
3. Add symbol to `di/types.ts`
4. Bind in appropriate module in `di/modules/`
5. Use via `getInjection('ISymbolName')` in app layer

## Quick Reference

### Creating a New Feature

```
1. Model      → src/entities/models/feature.ts
2. Errors     → src/entities/errors/feature.ts (if needed)
3. Interface  → src/application/repositories/feature.repository.interface.ts
4. Use Case   → src/application/use-cases/feature/action.use-case.ts
5. Repository → src/infrastructure/repositories/feature.repository.ts
6. Controller → src/interface-adapters/controllers/feature/action.controller.ts
7. DI Types   → di/types.ts (add symbols)
8. DI Module  → di/modules/feature.module.ts
9. Container  → di/container.ts (load module)
10. Action    → app/actions.ts or app/feature/actions.ts
```

### Import Rules (Enforced by ESLint)

| From | Can Import |
| ---- | ---------- |
| `app/` | `entities`, `di` only |
| `controllers` | `entities`, `service-interfaces`, `repository-interfaces`, `use-cases` |
| `use-cases` | `entities`, `service-interfaces`, `repository-interfaces` |
| `infrastructure` | `service-interfaces`, `repository-interfaces`, `entities` |
| `entities` | `entities` only |

## References

See [LAYERS.md](LAYERS.md) for detailed layer documentation with code patterns.
See [DI.md](DI.md) for dependency injection setup and wiring guide.
See [EXAMPLES.md](EXAMPLES.md) for complete code examples.
