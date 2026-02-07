# Service Decomposition Pattern

## Context

In large NestJS applications, "God Services" like the original `SalesService` often emerge. They handle too many responsibilities: querying, multiple external integrations, transformations, and complex business logic. This violates the Single Responsibility Principle and makes testing/maintenance difficult.

## The Skill

**Decompose large services into specialized, domain-focused services.**

### 1. Identify Responsibilities

Break down the monolith by grouping methods into logical domains:

- **Integration Layer**: Services that talk to external APIs (e.g., `FastApiInvoiceFlowService`, `FastApiClientService`).
- **Business Logic Layer**: Services that enforce rules and coordinate workflows (e.g., `SalesInvoiceService`).
- **Query/Data Layer**: Services aimed at complex data retrieval (e.g., `SalesQueryService`).

### 2. Implementation Pattern

- **Create new specialized service files**.
- **Use Dependency Injection**: Inject the new services into the main controller or facade.
- **Handle Circular Dependencies**: Use `@Inject(forwardRef(() => ServiceName))` if two services strictly need each other (though try to avoid this by design).

### 3. Example (From Project)

Instead of putting everything in `SalesService`:

```typescript
// BAD: Monolithic SalesService
class SalesService {
  async findSales() { ... }
  async createInvoice() { ... }
  async syncToFastApi() { ... }
}

// GOOD: Decomposed Services
class SalesQueryService {
  async findSales() { ... } // Optimized queries
}

class FastApiInvoiceFlowService {
  async createInvoice() { ... } // Dumb pipe to API
}

class SalesInvoiceService {
  constructor(
    private queryService: SalesQueryService,
    private apiService: FastApiInvoiceFlowService
  ) {}

  async processInvoiceFlow() {
    // 1. Get Data
    const sale = await this.queryService.findSales(...);
    // 2. Business Logic
    if (!this.validate(sale)) return;
    // 3. Call External API
    await this.apiService.createInvoice(sale);
  }
}
```

## Benefits

- **Testability**: Easier to mock `FastApiInvoiceFlowService` when testing `SalesInvoiceService`.
- **Readability**: Smaller files are easier to understand.
- **Maintainability**: Changes in API integration don't risk breaking query logic.
