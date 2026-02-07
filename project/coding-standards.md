# Coding Standards & Conventions

## 1. General Principles

- **Explicit over Implicit**: Code should be self-documenting. Use descriptive variable names (`customerAddress` vs `addr`).
- **Strict Typing**: Avoid `any`. Define Interfaces or DTOs for all data structures, especially external API responses.
- **Fail Fast, Log Loudly**: Catch errors at the lowest logical level, log them with context (`Logger`), and decide whether to propagate or sanitize.

## 2. Naming Conventions

### Files & Directories

- **Kebab-case** for all files and folders:
  - `sales-invoice.service.ts`
  - `customer-profile.entity.ts`
  - `billing/`

### Classes & Interfaces

- **PascalCase** for Classes, Interfaces, Enums:
  - `SalesService`
  - `CreateInvoiceDto`
  - `OrderStatus`

### Methods & Variables

- **CamelCase** for methods, properties, variables:
  - `calculateTotal()`
  - `isActive`

### Database

- **Snake_case** for column names in TypeORM entities (to match DB):
  - `@Column({ name: 'customer_code' }) customerCode: string;`

## 3. Architecture Layering

### Controllers (`*.controller.ts`)

- **Responsibility**: Receive HTTP requests, validate DTOs, call Services, return standard responses.
- **Rule**: NO business logic in controllers.

### Services (`*.service.ts`)

- **Responsibility**: Business logic, rules, orchestration.
- **Rule**: Should not access HTTP context directly.

### Repositories/Database

- **Responsibility**: Data persistence.
- **Rule**: Use TypeORM `Repository` pattern.

### Utilities (`*.utils.ts`)

- **Responsibility**: Pure functions (formatting, calculation, classification).
- **Rule**: Stateless, no DI (Dependency Injection), no side effects.

## 4. Error Handling

- Use `try/catch` blocks in Service methods that interact with external systems.
- **Integration Methods**: Return a `Result Object` (`{ status, message, data }`) instead of throwing, to ensure auditability.
- **Validation**: Use `class-validator` DTOs for input validation.

## 5. Comments & Documentation

- **JSDoc** for public methods explains _Why_ and _What_, not just _How_.
- **Inline comments** for complex regex or business rule exceptions.
