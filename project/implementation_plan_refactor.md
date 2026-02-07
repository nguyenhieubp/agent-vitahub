# Implementation Plan - Sales Module Refactor (Strategy Pattern)

## Goal

Decouple `SalesQueryService` ("Read Path") and prevent logic leaks in `SalesPayloadService` ("Write Path") by separating logic into distinct files based on Order Type (Regular, Service, Wholesale, etc.).

## 1. Separate "Read" Logic (Sales Query)

Refactor `SalesQueryService.findByOrderCode` into a Strategy Pattern.

### New Architecture

- **Folder**: `backend/src/modules/sales/strategies/`
- **Interface**: `SalesEnrichmentStrategy`
  ```typescript
  interface SalesEnrichmentStrategy {
    supports(orderType: string): boolean;
    enrich(sales: Sale[], context: EnrichmentContext): Promise<EnrichedSale[]>;
  }
  ```
- **Strategies**:
  1.  `RegularSalesStrategy.ts` ("01. Thường", "06. Bán buôn"):
      - Standard Flow: Stock Transfer Match -> Employee Check -> Format.
  2.  `ServiceSalesStrategy.ts` ("02. Làm dịch vụ"):
      - Special Flow: Filter for 'S' items -> Skip heavy inventory checks -> Format.
  3.  `SplitCardSalesStrategy.ts` ("08. Tách thẻ"):
      - Enrichment Flow: Standard + N8N Card Data Fetch (with shared `getInvoiceCardSerialMap` logic).

### Changes

- **`SalesQueryService.ts`**:
  - Remove monolithic `findByOrderCode`.
  - Inject Strategies.
  - Implement `EnrichmentContext` builder (fetching Loyalty, Dept, etc. once).
  - Method `findByOrderCode` becomes a Dispatcher: `strategy.enrich(...)`.

## 2. Fix Logic Leaks (Write Path)

Move business logic out of `SalesPayloadService`.

### Changes

- **`SalesPayloadService.ts`**:
  - Remove `OrderFee` repository usage.
  - Remove `isPlatformOrder` detection logic.
  - Accept `isPlatformOrder` and `platformBrand` as arguments in `buildFastApiInvoiceData`.
- **`NormalOrderHandlerService.ts`**:
  - Perform `OrderFee` lookup here.
  - Pass flags to `SalesPayloadService`.

## 3. Improve Error Handling

Address "Silent Failures".

### Changes

- **`N8nService.ts`** & **`LoyaltyService.ts`**:
  - Replace `// Ignore error` with `Logger.warn` containing `docCode` and exact error.
  - Ensure critical failures (like Loyalty down) don't crash the flow but are visible.

## Execution Order

1.  **Refactor Payload Service (Easy)**: Move `OrderFee` logic up to Handler.
2.  **Create Enum/Constants**: Ensure Order Types are consistent.
3.  **Implement Strategies** for Query Service.
4.  **Wire up** Dispatcher in Query Service.
5.  **Audit** Error Handling.
