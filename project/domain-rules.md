# Domain Business Rules

## 1. Warehouse & Inventory (Kho)

### A. Stock Transfer Sync

- **Mechanism**: Pulls from Zappy via `SyncService` using a multi-part strategy (Parts 1, 2, 3) to prevent API timeouts.
- **Entity**: `StockTransfer`.
- **Deduplication**: Records are identified by a composite key (implied, usually `guid` or `docCode + line`).

### B. Warehouse Mapping

- **Routing**: The system maps Zappy `Store Code` to ISO System `Warehouse Code` via the `WarehouseCodeMapping` entity.
- **Logic**:
  - **Orders starting with "L"** (e.g., `LAM_DV`) -> Routed to **Service Warehouse (Kho Làm)**.
  - **Orders starting with "B"** (e.g., `NORMAL`) -> Routed to **Retail Warehouse (Kho Bán)**.

### C. Display & Visibility (New)

- **Stock Transfer as Root**: In Sales Order views (List & Detail), data is "exploded" based on individual Stock Transfer records.
  - **Rule**: 1 Sales Item in Zappy -> N Frontend Rows if there are N Warehouse Transactions (Serials/Batches).
  - **Mapping**: 1-to-1 mapping ensures full visibility of warehouse operations.
  - **Financials**: `Revenue`, `Discount`, and `Qty` are recalculated proportionally for each exploded line.

## 2. Promotions (Khuyến Mãi)

### A. Classification

- **Header**: `Promotion` entity.
- **Lines**: `PromotionLine` entity.
- **Sources**: Synced from Zappy or created manually via Excel Import.

### B. Validation Rules

- **Fast API Integration**: Before creating an invoice, promotion codes (`ma_ck01`, `ma_ctkm_th`) MUST be validated against the **Loyalty API**.
- **Formatting**:
  - Codes like `VCHB`, `VCKM` are normalized to `VC HB`, `VC KM` before validation.
  - Suffixes (after `-`) are stripped for validation (e.g., `PRMN.01-ABC` -> check `PRMN.01`).

## 3. Vouchers

### A. VIP Classification

Determines if a product is a Voucher or Special Item for VIP calculation (`sales.utils.ts`):

- **Rule 1 (Service)**: `productType == 'DIVU'` -> **VIP DV MAT**.
- **Rule 2 (Voucher)**: `productType == 'VOUC'` -> **VIP VC MP**.
- **Rule 3 (Material)**: `materialCode` starts with `E.` or contains `VC` -> **VIP VC MP**.
- **Default**: **VIP MP**.

## 4. Cashio (Payment & Banking)

### A. Sync Strategy

- **Entity**: `DailyCashio`.
- **Identifier**: `api_id` is the strict unique key from Zappy.
- **Conflict Resolution**: If `api_id` exists, UPDATE the record; otherwise INSERT.

### B. Payment Method Mapping

- **Entity**: `PaymentMethod`.
- **One-to-Many**: One POS payment method (e.g., "Thẻ") can map to multiple Bank Accounts based on the **Branch (DVCS)**.
- **Auto-Process**: The `PaymentService` automatically matches payment records to invoices based on time windows and amounts.
