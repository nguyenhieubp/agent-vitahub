# Sales Module Logic Documentation

This document describes the core logic of the Sales Module in the InvoiceFlow backend, specifically focusing on how sales data is processed, transformed, and synchronized with the FAST API.

## 1. Overview

The Sales Module is responsible for:

1.  **Querying**: Fetching sales orders from the local database (`Sale` entity) and enriching them with related data (Customer, Payment/Cashio, Stock Transfers).
2.  **Orchestration**: Determining the correct processing flow for each order based on its type (Normal, Service, Return, etc.).
3.  **Transformation**: converting internal data models into the specific JSON format required by the FAST API.
4.  **Synchronization**: Sending data to FAST API (Customer -> Order -> Invoice -> Payment) and tracking status.

## 2. Core Components

### 2.1 SalesQueryService (`sales-query.service.ts`)

- **Purpose**: Handles data retrieval and pre-processing.
- **Key Logic**:
  - **`findAllOrders`**: Primary method to get orders.
    - Retrieves `Sale` records.
    - Groups `Sale` lines into logical Orders by `docCode`.
    - **Formatting Delegation**: Delegates enrichment and formatting to `SalesFormattingService`.
  - **`enrichOrdersWithCashio`**: (Still present for legacy flow support but primarily handled in formatting service for new V2 endpoints).
    - Explodes sales lines based on `StockTransfer` data.

### 2.2 InvoiceFlowOrchestratorService (`invoice-flow-orchestrator.service.ts`)

- **Purpose**: The central decision maker for processing a specific order (`docCode`).
- **Logic Flow**:
  1.  **Validation**: Checks if the order has sales and determines `ma_dvcs` (Unit Code) from `branchCode`.
  2.  **Routing**: Inspects `ordertype` to decide the strategy:
      - **Sale Return**: -> `SaleReturnHandlerService`
      - **Service**: -> `SpecialOrderHandlerService` (Service Flow)
      - **Special Types** (Exchange, Birthday, etc.): -> `SpecialOrderHandlerService`
      - **Normal**: -> `NormalOrderHandlerService`
  3.  **Persistence**: Saves the result (Success/Fail) to `FastApiInvoice` entity for tracking.

### 2.3 NormalOrderHandlerService (`normal-order-handler.service.ts`)

- **Purpose**: Handles standard retail/wholesale orders.
- **Steps**:
  1.  **Sync Customer**: `createOrUpdateCustomer`.
  2.  **Enrichment**:
      - Explodes Sales Lines based on Stock Transfers.
      - Fetches Card Data from N8n (for "Tách Thẻ").
  3.  **Sales Order (Unified)**:
      - Creates **ONE** Sales Order for the entire order using original `docDate`.
      - This ensures the order is recorded as a single entity in Fast API.
  4.  **Sales Invoice (Split by Date)**:
      - Groups sales lines by **Stock Transfer Date** (`transDate`).
      - Creates **MULTIPLE** Sales Invoices if dates differ:
        - `docCode` suffix: `-1`, `-2`, etc. (if split).
        - `ngay_ct`, `ngay_lct`: Set to the specific `transDate` of that group.
      - This ensures accounting records match the actual warehouse export dates.
  5.  **Payment Sync**:
      - Links all payments (Cashio & Stock) to the **FIRST** successfully created Sales Invoice `docCode`.

### 2.4 SalesPayloadService (`sales-payload.service.ts`)

- **Purpose**: "The Translator" - Complexity center for payload mapping.
- **Key Mapping Rules**:
  - **`ma_dvcs` (Unit Code)**:
    - Critical field for multi-branch systems.
    - Resolved via `LoyaltyService.fetchMaDvcs(branchCode)` (which queries the Loyalty API's Department endpoint).
  - **`ma_kho` (Warehouse Code)**:
    - Priority: `StockTransfer.stockCode` > `Sale.maKho` > Default based on Branch.
  - **`ma_vt` (Material Code)**:
    - Mapped from `svc_code` (Service Code) using `LoyaltyService.getMaterialCodeBySvcCode` when applicable.
  - **Financials**:
    - Calculates `tienHang`, `tienThue`, discounts (`ck01`...`ck22`).
    - **Platform Orders**: Maps `voucher_from_seller` (~`paid_by_voucher...`) to `ck15` ("VC CTKM SÀN") and clears `ck05`.
    - Distributes values if a sale line was split (exploded) by stock transfers.

- **Purpose**: "The Messenger" - Direct interaction with `FastApiClientService`.
- **Key Methods**:
  - `executeFullInvoiceFlow`: Wraps Customer -> Order -> Invoice creation.
  - `processCashioPayment`: Converts `DailyCashio` records into FAST Payment payloads (`/Fast/paymentMethod`).
  - `createSalesOrder`: Creates Sales Order with Promotion handling:
    - Filters unique promotions from `detail` items (`ma_ck01`, `ma_ctkm_th`).
    - Calls `callPromotion` API for each unique promotion with DVCS mapping.
    - Validates promotion API responses (status must be 1).
    - Submits final Sales Order payload to FAST API.

### 2.6 SalesInvoiceService (`sales-invoice.service.ts`)

- **Purpose**: Entry point for invoice creation, handles routing and special cases.
- **Key Methods**:
  - `createInvoiceViaFastApi`: Main entry point with optional `onlySalesOrder` flag.
    - **Full Flow** (default): Delegates to `InvoiceFlowOrchestratorService`.
    - **Sales Order Only**: Transforms data using `SalesPayloadService` and calls `createSalesOrder`.

### 2.7 SalesFormattingService (`sales-formatting.service.ts`)

- **Purpose**: Unified enrichment and formatting for Frontend API V2.
- **Key Logic**:
  - **`formatSingleSale`**:
    - Resolves Batch/Serial from ST or legacy fields.
    - Calls `InvoiceLogicUtils.calculateSaleFields` (Single Source of Truth).
    - Checks for Employee Discounts (`isEmployee`) and Platform Orders.
    - Maps `maKho` consistently with Payload service.
    - Returns a fully enriched object for the Mapper.

## 3. Critical Data Flows

### A. The "Explosion" Pattern (Stock Transfer Integration)

1.  **Query**: `SalesQueryService` fetches raw `Sale` lines.
2.  **Match**: It fetches `StockTransfer` records for the same `docCode`.
3.  **Explode**: If a `Sale` line (qty: 5) maps to 2 `StockTransfers` (qty: 2 from Batch A, qty: 3 from Batch B), the system "explodes" this into 2 lines in the FAST payload.
    - _Why?_ To accurately track Batch/Serial numbers and precise Warehouse (`ma_kho`) locations for every item.

### B. `ma_dvcs` Resolution (The tricky bit)

1.  **Frontend/Order**: Provides `branchCode` (e.g., "HCM01").
2.  **Orchestrator**: Passes this to Payload Service.
3.  **Payload Service**: Calls `LoyaltyService.fetchMaDvcs("HCM01")`.
4.  **Loyalty Service**: Calls external API `/departments?branchcode=HCM01` to get the accounting unit code (`ma_dvcs`).
    - _Correction Note_: Previously there was confusion between using `ma_bp` vs `branchcode` in the API query. Logic now uses `branchcode` to reliably fetch `ma_dvcs`.

## 4. Helper Services

- **`LoyaltyService`**: Bridges gap between local data and external product/department definitions.
- **`CategoriesService`**: Manages category data with in-memory caching.
- **`N8nService`**: Handles external webhooks (e.g., card data).
- **`SalesUtils` / `InvoiceLogicUtils`**: Shared pure functions. `InvoiceLogicUtils` is now the **sole authority** for core business math (prices, accounts, promotion logic).
