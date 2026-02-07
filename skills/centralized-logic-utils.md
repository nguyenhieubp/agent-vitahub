# Centralized Domain Utilities

## Context

Business rules (like "Is this a Service Order?", "Is this customer VIP?") often duplicate across the codebase (Controllers, Services, Frontend). Hardcoding strings (e.g., `ordertype === '02. Làm dịch vụ'`) properties in multiple places leads to bugs when requirements change.

## The Skill

**Extract pure business logic into dedicated Utility files (`*.utils.ts`).**

### 1. Pure Functions

Keep utils stateless. They should accept data (Entity/DTO) and return a decision (Boolean/String/Object). They should NOT depend on database repositories or services.

### 2. Implementation Pattern

Create a `utils` directory (e.g., `backend/src/utils/sales.utils.ts`) and export named functions.

- **Normalization**: Handle trimming, casing, and aliases in one place.
- **Classification**: Encapsulate complex `if/else` logic for types.
- **Formatting**: Standardize how IDs or Codes are generated.

### 3. Example (From `sales.utils.ts`)

**BAD (Scattered Logic):**

```typescript
// In Service A
if (order.type === '02. Làm dịch vụ' || order.type === 'LAM_DV') { ... }

// In Service B
if (order.type === '02. Làm dịch vụ') { ... } // Forgot LAM_DV!
```

**GOOD (Centralized Utility):**

```typescript
// sales.utils.ts
export function isServiceOrder(type: string): boolean {
  const normalized = type?.trim().toUpperCase();
  return [
    '02. LÀM DỊCH VỤ',
    'LAM_DV',
    'DOI_DV'
  ].includes(normalized);
}

// In Service A & B
if (SalesUtils.isServiceOrder(order.type)) { ... }
```

## Benefits

- **Consistency**: The definition of "Service Order" is the same everywhere.
- **Refactoring Safety**: Changing a rule requires editing only one file.
- **Testability**: Pure utility functions are extremely easy to unit test (no mocks needed).
