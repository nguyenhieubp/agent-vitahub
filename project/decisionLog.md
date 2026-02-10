# Decision Log

## [2026-02-10] Replace stock_transfers Date Filter with sale.docDate

### Context

The `findAllOrders` method filtered orders by `stock_transfers.transDate` using either an INNER JOIN (~13s) or a pre-filter step (`SELECT DISTINCT soCode FROM stock_transfers WHERE transDate BETWEEN ...`, ~7-15s). The `stock_transfers` table has no index on `transDate`, making any query against it a full table scan.

Multiple approaches were tried:
- **INNER JOIN on fullQuery**: ~13s
- **Two-step pre-filter (materialized soCodes)**: 7s + 1s = 8s
- **SQL IN subquery**: 23s (PG materialized the inner query)
- **EXISTS subquery**: 27s (PG still scanned full stock_transfers per row)

### Decision

**Replace date filtering with `sale.docDate` WHERE clause** — no JOIN, no stock_transfers scan required.

### Rationale

- **Performance**: Eliminates 7-15s bottleneck entirely (sale.docDate filter is <100ms on indexed sales table)
- **Acceptable tradeoff**: `sale.docDate` and `stock_transfer.transDate` are nearly identical (order date ≈ export date). Edge cases (1-day difference) are negligible for pagination purposes.
- **Data accuracy preserved**: The explosion/enrichment step still uses actual stock_transfer data, so the detailed view is always accurate.

---

## [2026-02-10] Parallel COUNT for Exact Pagination Total

### Context

Previously used `LIMIT + 1` trick to detect "has more pages" without running a COUNT query. User requested exact total count.

### Decision

**Run `COUNT(DISTINCT docCode)` in parallel** with the pagination query via `Promise.all`. Both queries use the same filters.

### Rationale

- **Exact total**: Users see precise page count instead of "Next →" guessing
- **Zero latency cost**: COUNT runs alongside pagination — total time = MAX(pagination, count), not SUM
- **Same filter**: Both queries apply identical brand/date/status filters for consistency

---

## [2026-02-10] Eliminate Redundant enrichOrdersWithCashio & enrichSalesWithMaVtRef

### Context

`enrichOrdersWithCashio` was called twice: once before formatting (to explode sales lines by stock transfers) and once after (to re-enrich with cashio data post-explosion). Each call independently fetched `DailyCashio`, `loyaltyProductMap`, `warehouseCodeMap`, and ran `enrichSalesWithMaVtRef`. The pre-explosion call to `enrichSalesWithMaVtRef` was also redundant since the post-explosion call covers all lines.

### Decision

1. **Pre-fetch shared data** (`cashioRecords`, `loyaltyProductMap`, `warehouseCodeMap`) in the parallel batch fetch
2. **Pass pre-fetched data** to the second `enrichOrdersWithCashio` call via optional `prefetchedData` parameter
3. **Skip `enrichSalesWithMaVtRef`** inside `enrichOrdersWithCashio` via `skipMaVtRef` flag (handled separately after)
4. **Remove first redundant call** entirely

### Rationale

- **Avoids 3 redundant DB/API calls** per request
- **Backward compatible**: Optional parameter — callers without pre-fetched data still work as before

---

## [2026-02-10] Parallelize 8 Independent Data Fetches

### Context

`findAllOrders` fetched 8 independent datasets sequentially: stockTransfers, departments, warehouseCodes, svcCodes, orderFees, employeeStatus, cardData, dailyCashio. Each took 100-500ms, totaling ~2-4s sequentially.

### Decision

**Wrap all 8 fetches in a single `Promise.all` call.**

### Rationale

- All 8 are independent (no data dependencies between them)
- Parallel execution reduces total time from SUM to MAX (~500ms instead of ~3s)
- Simple refactor with no logic changes

---

## [2026-02-09] Clear Voucher Fields for Point Exchange Orders

### Context

Point Exchange orders ("03. Đổi điểm") were displaying `ma_ck05` (Voucher Code) and `ck05_nt` (Voucher Amount) fields in the frontend, which is incorrect per business rules.

### Decision

Add final clearing step in `SalesQueryService` after payment override logic to ensure Point Exchange orders never have voucher discount fields, regardless of payment method data.

### Rationale

- **Business Rule Compliance**: Point Exchange orders should not show voucher discounts.
- **Override Safety**: Payment method logic may set these fields based on `cashioData`, but order type takes precedence.

---

## [2026-02-09] Centralize Discount Logic (ma_ck11)

### Context

Fast API payload was missing `ma_ck11` while Frontend API had it (hardcoded). Logic was duplicated and inconsistent.

### Decision

1.  **Unified Helper**: Create `InvoiceLogicUtils.resolveMaCk11` as Single Source of Truth.
2.  **Refactor**: Update both `SalesQueryService` and `SalesPayloadService` to use this helper.

### Rationale

- **Consistency**: Ensure consistency between UI and Backend Payload.
- **Maintainability**: Single place to update brand mappings (e.g., adding `F3`, `CHANDO`).

---

## [2026-02-04] Fallback Lookup for Sale Return Orders

### Context

When processing Return/Exchange orders (suffixed with `_X`), the system attempts to find the "Original Order" by stripping the `_X` suffix. However, in some cases (e.g., legacy data or mismatched codes), the order without `_X` does not exist in the database. The `SalesInvoiceService` throws a generic `NotFoundException` in this case, crashing the flow.

### Decision

**Implement "Try-Catch-Fallback" pattern:**

1.  Try to find order using `docCodeWithoutX`.
2.  Catch `NotFoundException`.
3.  Fallback to finding order using the original `docCode` (with `_X`).

### Rationale

- **Resilience**: Prevents a single missing order reference from crashing the entire processing queue.
- **Flexibility**: Handles cases where the `_X` order might be a standalone entity or valid on its own.

---

## [2026-02-04] Warehouse Delete uses Transaction Date

### Context

The "Warehouse Statistics" page displays data filtered by `transDate` (Date of the Stock Transfer). However, the "Delete" and "Retry" batch actions were filtering by `processedDate` (Date the system processed the record). This caused a disconnect: users would select a date range of transactions to delete, but the system would try (and fail) to delete based on when they were processed suitable for system cleanup but not for user-driven management.

### Decision

**Switch Delete/Retry actions to filter by `transDate`.**

### Rationale

- **WYSIWYG**: The action matches the user's visual context. If they see records for "Feb 1st", deleting "Feb 1st" should remove them, regardless of when the system technically processed them.

---

## [2026-01-18] Split Stock Transfer Sync into Two Phases

### Context

The `syncStockTransferRange` function processed each day sequentially (Sync -> Process Warehouse). Alternatively, if the warehouse processing step (calling Fast API) hung or was slow, it blocked the sync for subsequent days.

### Decision

Refactor `syncStockTransferRange` to execute in two phases:

1.  **Phase 1**: Sync data from Zappy for the _entire_ date range. Skip warehouse processing.
2.  **Phase 2**: Iterate through the date range again and process warehouse transfers for all synced data.

### Rationale

- **Reliability**: Ensures we capture all raw data from Zappy even if the downstream Fast API is unstable or slow.
- **Visibility**: Users can see "Phase 1" complete and know data is safe in the DB.
- **Performance**: Decouples the fetching speed from the processing speed.

- **Performance**: Decouples the fetching speed from the processing speed.

---

## [2026-01-18] Split Sales Sync into Two Phases

### Context

Users observed that Sales Sync and Fast API invoice creation seemed to run in parallel or get blocked, causing confusion and potential data issues. The process needed to be clearer and more robust.

### Decision

Implement a **Two-Phase Sync** workflow for Sales (similar to Stock Transfer):

1.  **Phase 1 (Sync)**: Fetch all sales data from Zappy for the date range -> Save to DB.
2.  **Phase 2 (Process)**: Query the DB for orders in that date range -> Create Invoices in Fast API.

### Rationale

- **Sequential Integrity**: Ensures all data is safely in the local DB before attempting any external API pushes.
- **Error Isolation**: A failure in Phase 2 (Fast API) does not prevent Phase 1 (Data Fetching) from completing.
- **User Experience**: Provides clear "Syncing..." then "Processing..." feedback.

---

## [2026-01-18] Remove Aggregation for Sales Payload

### Context

The Fast API requires an "exploded" view where 1 sale item with multiple stock transfers results in multiple detail items. The previous logic attempted to aggregate items, which was incorrect for this requirement.

### Decision

- **Disable Aggregation**: Removed all grouping/merging logic in `sales-payload.service.ts`.
- **Enforce Explosion**: Ensure `sales-query.service.ts` explodes sale items based on Stock Transfers (splitting by serial/batch) _before_ passing them to the payload service.
- **Handler Responsibility**: Update all Order Handlers (`normal`, `special`, `return`) to explicitly call `enrichOrdersWithCashio` to trigger this explosion.

### Rationale

To strictly comply with the Fast API requirement of 1 detail line per stock transfer movement.

---

## [2026-01-17] Optimize Sales Sync Performance

### Context

The sales sync process suffered from N+1 query issues, particularly when fetching order details and products.

### Decision

Refactor `sales-sync.service.ts` to implement batch fetching for:

- Orders by DocCode.
- Product types and missing items.

### Rationale

To significantly reduce database load and improve sync speed.

---

## [2026-01-17] Extract Invoice Data Enrichment

### Context

Logic for enriching invoice data was tightly coupled in `SalesInvoiceService`.

### Decision

Extract `findByOrderCode` and related logic into a dedicated `InvoiceDataEnrichmentService`.

### Rationale

## To improve separation of concerns and testability.

## [2026-01-29] Consolidate Sales Formatting Logic & Mathematics

### Context

Logic for price, discount, and accounting account calculations was duplicated and slightly inconsistent between the Frontend API (`SalesQueryService`) and the Fast API Integration (`SalesPayloadService`). Duplicated Batch/Serial display was also reported in the Frontend.

### Decision

1.  **Unified Utility**: Established `InvoiceLogicUtils.calculateSaleFields` as the **Single Source of Truth** for all core business math.
2.  **Unified Service**: Created `SalesFormattingService` to handle Frontend-specific enrichment, replacing multiple legacy services and utilities.
3.  **Metadata-Driven Display**: Included `trackBatch`, `trackSerial`, and `trackStocktake` flags in the API response metadata instead of trying to perfectly guess the display column in the backend.
4.  **Mapper Refinement**: Refined the response mapper to exclusively populate `maLo` or `maSerial` based on product metadata to prevent UI duplication.

### Rationale

- **Consistency**: Ensures users see the same numbers and accounting data on the web UI that are sent to the ERP.
- **Maintainability**: Reduces the number of services and ensures a single point of failure/update for business rules.

---

## [2026-02-02] Enforce Mutual Exclusivity for Batch/Serial in Export

### Context

Users reported that Excel exports frequently showed values in both "Mã lô" and "Số serial" columns for the same line item, causing confusion about whether the item was tracked by Batch or Serial.

### Decision

**Enforce specific priority in `SalesFormattingService`:**

1.  If `trackBatch` is TRUE (from Product metadata):
    - Populate `maLo` (Batch).
    - Explicitly **CLEAR** `maSerial`, `serial`, and `soSerial` (set to null/empty).
2.  If `trackBatch` is FALSE and `trackSerial` is TRUE:
    - Populate `soSerial` / `maSerial`.
    - `maLo` remains empty.

### Rationale

- **Data Integrity**: Eliminates ambiguity. An item is physically tracked either by Lot or by Serial (or neither), rarely both in a way that requires dual reporting in this specific export context.

---

## [2026-02-02] Implement "Best Fit" Matching for Stock Transfers

### Context

When a Sales Order contains multiple lines with the **same** `itemCode` (e.g., Line 1: Qty 5, Line 2: Qty 1 [Gift]), the previous "First Unused" matching logic often assigned Stock Transfers incorrectly (e.g., ST with Qty 5 matching to Line with Qty 1).

### Decision

**Switch matching algorithm from "First Unused" to "Best Fit":**

1.  **Priority 1 (Exact Match)**: Find an unused Sales Line where `abs(SalesLine.Qty) === abs(StockTransfer.Qty)`.
2.  **Priority 2 (Fallback)**: If no exact match found, fall back to the first available unused Sales Line.

### Rationale

- **Accuracy**: Ensures that specific inventory movements (e.g., a specific batch for a gift item) are correctly linked to their corresponding sales line, preserving financial attributes (price 0 vs price X).
- **Robustness**: Handles the common edge case of "Buy X Get 1 Free" of the same product without requiring complex manual mapping.

---

## [2026-02-03] Separate Update Logic for Error Orders & Enforce Query Limits

### Context

1.  **Update Logic**: The existing `StockTransfer` update logic was being used for `Error Orders`, but Error Orders are fundamentally `Sale` entities (often missing `itemCode` or `branchCode`). Using the wrong API caused 404 errors due to ID mismatches and incorrect entity targeting.
2.  **Performance**: The `getStatusAsys` query allowed retrieving huge datasets without date limits, causing full-table scans and performance degradation.

### Decision

1.  **Dedicated Endpoint**: Created `updateErrorOrder` specifically for `Sale` entities. It resolves `materialCode` -> `itemCode` via Loyalty API check before saving.
2.  **Strict Query Limits**: Enforced mandatory `dateFrom` and `dateTo` parameters (max 31 days range) in `getStatusAsys`. Removed the arbitrary `SCAN_LIMIT` to rely on the date index instead.

### Rationale

- **Correctness**: Ensures we are updating the correct table (`sales` vs `stock_transfers`) using the correct ID.
- **Stability**: Prevents users from accidentally triggering massive database queries that could hang the server.
- **Data Integrity**: Automatically validating `materialCode` against Loyalty API ensures we don't save invalid codes into the system.
