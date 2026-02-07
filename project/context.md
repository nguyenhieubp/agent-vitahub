# Project Context: InvoiceFlow (Middleware System)

## 1. System Overview

**InvoiceFlow** is a mission-critical middleware that acts as the "Central Nervous System" for enterprise operations. It aggregates data from POS (Zappy), E-commerce, and Loyalty systems, processes complex business logic, and synchronizes financial data to the ERP (FAST API).

## 2. Infrastructure & Configuration

- **Runtime**: Node.js with NestJS Framework.
- **Database Architecture**: Multi-Database / Multi-Tenant (PostgreSQL).
  - **Primary**: Main application data (103.145.79.165).
  - **Secondary**: Additional data source (103.145.79.165).
  - **Third**: Legacy/External data source (103.145.79.37).
- **Network**:
  - **CORS**: Engineered to support `invoiceflow.vn` and `localhost:3000`.
  - **Timeout Strategy**: Aggressive 5-minute (300,000ms) timeout for heavy `export-orders` operations.

## 3. Module Registry (Backend Map)

### A. Core Business Domains

| Module          | Description                                                                           | Key Services                                       |
| :-------------- | :------------------------------------------------------------------------------------ | :------------------------------------------------- |
| **Sales**       | The heart of order processing. Orchestrates the flow from valid order to ERP invoice. | `SalesInvoiceService`, `FastApiInvoiceFlowService` |
| **Payment**     | Manages cash handling, bank reconciliation, and payment method mapping.               | `PaymentService`                                   |
| **Invoices**    | Generic invoice management (likely internal representation).                          | `InvoicesService`                                  |
| **PlatformFee** | Calculates and tracks fees from E-commerce platforms.                                 | `PlatformFeeService`                               |
| **OrderFee**    | Manages additional order-related fees/surcharges.                                     | `OrderFeeService`                                  |

### B. Integration & Sync Domains

| Module              | Description                                                                         | Key Services                     |
| :------------------ | :---------------------------------------------------------------------------------- | :------------------------------- |
| **Sync**            | Ingestion engine pulling data from Zappy (Stock, Sales, Cashio).                    | `SyncService`, `ZappyApiService` |
| **FastApiInvoices** | Specialized integration for pushing invoices to FAST ERP and tracking their status. | `FastApiInvoiceService`          |
| **Categories**      | Master data management (Products, Promotions, Customers).                           | `CategoriesService`              |

### C. Infrastructure Domains

| Module      | Description                                               | Key Services     |
| :---------- | :-------------------------------------------------------- | :--------------- |
| **MultiDb** | Dynamic database connection management for multi-tenancy. | `MultiDbService` |

## 4. Key Workflows

### 4.1. The "Sales Invoice" Pipeline

1.  **Ingestion**: `SyncService` pulls raw sales data from Zappy.
2.  **Routing**: `SalesInvoiceService` uses `sales.utils.ts` to classify orders (Retail/Service/Return).
3.  **Enrichment**: Adds info from `LoyaltyService` (Customers) and `Promotion` (Discounts).
4.  **Execution**: `FastApiInvoiceFlowService` pushes to FAST ERP.
5.  **Audit**: Failures are captured in `FastApiInvoice` table with full JSON payloads.

### 4.2. Inventory Synchronization

- **Stock Transfer**: Multi-part sync (Part 1, 2, 3) to handle large datasets.
- **Warehouse Mapping**: Resolves Zappy Store Codes to FAST Warehouse Codes based on Prefix Rules ('L' for Service, 'B' for Retail).

## 5. Development Philosophy

- **Robustness First**: API integrations must never crash the main thread; they must catch, log, and return failure objects.
- **Centralized Logic**: Business rules (e.g., "What is a VIP Order?") live in `utils`, not strewn across services.
- **Auditability**: Every external interaction is a transactional event that must be logged.
