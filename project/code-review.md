# Senior Code Review: Sales Module

## Executive Summary

The Sales module (`backend/src/modules/sales`) has evolved significantly to handle complex business logic (Normal Orders, Special Orders, Stock Transfers, Vouchers, E-commerce Integration). While functional and recently refactored to address performance issues (N+1 queries), it exhibits signs of "Organic Growth" where business rules are scattered across Service layers.

Recent efforts to unify logic (e.g. `N8nService.getInvoiceCardSerialMap`) are a strong step in the right direction. However, `SalesQueryService` remains a bottleneck of complexity.

## Key Strengths

1.  **Performance Optimization**: Recent work to implement Batch Fetching (Loyalty, N8N) and remove N+1 queries is excellent. The move to "Explosion" logic (handling 1 Sale -> N Deliveries) is architecturally sound.
2.  **Logic Unification**: The recent extraction of `getInvoiceCardSerialMap` into `N8nService` sets a good precedent for "Single Source of Truth".
3.  **Separation of Flows**: The distinction between `NormalOrderHandler`, `SpecialOrderHandler`, and `SalesOrderHandler` helps isolate flow-specific logic.

## Critical Areas for Improvement

### 1. `SalesQueryService` Complexity ("God Class")

- **Observation**: `SalesQueryService.findByOrderCode` is doing too much: Fetching, Enrichment, Matching, Calculation, and Formatting. It is ~900 lines long even after refactoring.
- **Risk**: High cognitive load. Modifying one part (e.g., matching logic) risks breaking another (e.g., formatting).
- **Recommendation**:
  - Extract **Matching Logic** (Stock Transfer <-> Sale) into a dedicated `SalesMatchingService`.
  - Extract **Enrichment Logic** (Loyalty, Vouchers, N8N) into a `SalesEnrichmentOrchestrator`.
  - Keep `SalesQueryService` strictly for Repository interaction.

### 2. Error Handling & Observability

- **Observation**: Multiple instances of `try-catch` blocks that simply "Ignore error" (e.g., fetching card data, svc_code lookup).
- **Risk**: Silent failures. If N8N or Loyalty API goes down, the system degrades silently, leading to "Empty Data" issues like the one just debugging.
- **Recommendation**: Implement a `SoftFailure` pattern. Log warnings with distinct codes/tags. Return "Partial Success" or explicit "Missing Data" flags to Frontend so UI can warn the user (e.g., "⚠️ Missing Card Data").

### 3. Business Logic Leaks in Payload Service

- **Observation**: `SalesPayloadService` contains business rule detection logic (e.g., Detecting `isPlatformOrder` by querying `OrderFee` repository directly inside the payload builder).
- **Risk**: Payload service should be "Dumb" (Pure Transformation). Logic about _what_ an order is should be resolved upstream (in Handlers).
- **Recommendation**: Move `isPlatformOrder` detection and `OrderFee` loading to the **Handlers** (`NormalOrderHandlerService`). Pass the fully enriched Context object to `SalesPayloadService`.

### 4. Hardcoded Constants & Magic Strings

- **Observation**: Strings like `'2505MN.CK521'`, `'MENARD'`, `'TSG'` are hardcoded in Utility files.
- **Risk**: Changing these requires code deploys. Hard to manage across environments (Test vs Prod).
- **Recommendation**: Move these to a `BusinessConfigurationService` or Database-backed Configuration Config.

## Immediate Action Items

1.  **Refactor**: Move `OrderFee` loading out of `SalesPayloadService` and into the Handler/Explosion layer.
2.  **Safety**: Add proper logging to the "Ignore Error" catch blocks.
3.  **Cleanup**: Verify if `SalesExplosionService` can take over the "Matching" logic currently remaining in `SalesQueryService.findByOrderCode`.

## Conclusion

The module is in a **Solid/Functional** state but requires architectural discipline to prevent further degradation into a "Monolith Service". The recent "Unified Logic" refactor is the correct pattern to follow for all future changes.
