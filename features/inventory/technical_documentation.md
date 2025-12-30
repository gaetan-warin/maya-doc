# Inventory Management
## Technical Design Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-TSD-INV-001 |
| **Version** | 1.0 |
| **Last Updated** | December 30, 2025 |
| **Status** | Approved |
| **Owner** | Engineering Team |
| **Classification** | Internal - Technical |

---

## 1. System Architecture

### 1.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Frontend (Vue 3)                          │
├─────────────────────────────────────────────────────────────────┤
│  Views               │  Components              │  Stores       │
│  ├─ stock.vue        │  ├─ StockKPI.vue         │  stockmgmt.js │
│  └─ purchase.vue     │  ├─ StockHistory.vue     │  inventory/   │
│                      │  ├─ PurchaseIndex.vue    │  └─ deliveries│
│                      │  ├─ PurchaseRequest.vue  │               │
│                      │  └─ Invoice.vue          │               │
├─────────────────────────────────────────────────────────────────┤
│                          API Layer                               │
│                    apiClient (Axios V2)                          │
├─────────────────────────────────────────────────────────────────┤
│                     Backend (Node.js)                            │
│         Stock Service + Purchase Service + Delivery Service      │
├─────────────────────────────────────────────────────────────────┤
│                      Database (MySQL)                            │
│  product | stock | purchase_request | purchase_order | delivery │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Technology Stack

| Layer | Technology | Version |
|:---|:---|:---|
| Frontend Framework | Vue.js | 3.x |
| State Management | Pinia | 2.x |
| Data Table | MDataTable (Custom) | Internal |
| Backend | Node.js / Express | 18.x |
| Database | MySQL | 8.x |

---

## 2. Component Architecture

### 2.1 View Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `stock.vue` | `web/src/views/stock.vue` | Stock levels, tabs, actions (710 lines) |
| `purchase.vue` | `web/src/views/purchase.vue` | Purchase workflow tabs |

### 2.2 Stock Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `StockKPI.vue` | `components/stock/` | Dashboard metrics |
| `StockHistory.vue` | `components/stock/` | Transaction history |
| `StockAndHistoryTab.vue` | `components/stock/` | Tab navigation |
| `StockUpdateForm.vue` | `components/stock/` | Edit stock item |
| `StockTakeForm.vue` | `components/stock/` | Physical count form |
| `TransferStocks.vue` | `components/stock/` | Location transfer modal |
| `InventoryTab.vue` | `components/stock/` | Category tabs |

### 2.3 Purchase Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `PurchaseIndex.vue` | `components/purchase/` | PO list and management |
| `PurchaseRequest.vue` | `components/purchase/` | Request list and form |
| `PurchaseForm.vue` | `components/purchase/` | PO creation modal |
| `PurchaseTable.vue` | `components/purchase/` | Order data table |
| `PurchasePrint.vue` | `components/purchase/` | Print-friendly PO |
| `Invoice.vue` | `components/purchase/` | Invoice matching |
| `ExpectedDelivery.vue` | `components/purchase/` | Delivery tracking |

---

## 3. State Management

### 3.1 Stock Management Store

**File**: `web/src/store/stockmanagement.js`

```javascript
// State Structure (partial)
{
  api: piniaEndPoint(),
  tenantId: null,
  stockTypes: [],          // Stock type categories
  vats: [],                // VAT rates
  taxes: [],               // Tax rates
  productDocuments: [],    // Product documents
  machineryPartCategory: [],
  selectedProduct: {},
  isLoading: false
}
```

### 3.2 Key Actions (58+ total)

| Action | Description | API Call |
|:---|:---|:---|
| `getAllStockTypes()` | Load stock categories | `GET /stock-types` |
| `getAllStockTypesV2()` | Load with caching | `GET /stock-types/v2` |
| `getProduct(id)` | Load single product | `GET /products/{id}` |
| `getInventory()` | Load stock levels | `GET /inventory` |
| `addStockType(data)` | Create category | `POST /stock-types` |
| `editStockType(id, data)` | Update category | `PUT /stock-types/{id}` |
| `deleteStockType(id)` | Remove category | `DELETE /stock-types/{id}` |
| `stockTake(data)` | Submit count | `POST /stock-take` |
| `stockTakeLog(data)` | Log adjustment | `POST /stock-take-log` |
| `shareInventory(data)` | Share with location | `POST /share-inventory` |
| `getAllVats()` | Load VAT rates | `GET /vats` |
| `getAllTaxes()` | Load tax rates | `GET /taxes` |
| `addVat(data)` | Create VAT rate | `POST /vats` |
| `addTax(data)` | Create tax rate | `POST /taxes` |
| `getCostEstimate(qty)` | Calculate cost | `GET /cost-estimate` |
| `getSubstances()` | Load active substances | `GET /substances` |
| `updateProductStatus(id, status)` | Toggle product | `PATCH /products/{id}/status` |
| `createStockType(data)` | Create type V2 | `POST /stock-types/v2` |
| `changeStockTypeStatus(id, data)` | Toggle type | `PATCH /stock-types/{id}/status` |

### 3.3 Deliveries Store

**File**: `web/src/store/inventory/deliveries.js`

Handles delivery-specific operations and expected shipment tracking.

---

## 4. Data Models

### 4.1 Product Entity

**Table**: `product`

| Column | Type | Constraints | Description |
|:---|:---|:---|:---|
| `idproduct` | CHAR(36) | PK | UUID identifier |
| `idtenant` | CHAR(36) | FK, NOT NULL | Tenant reference |
| `idstock_type` | CHAR(36) | FK | Stock type reference |
| `idproduct_category` | CHAR(36) | FK | Category reference |
| `name` | VARCHAR(255) | NOT NULL | Product name |
| `reference_price` | DECIMAL(12,4) | | Unit cost |
| `reference_unit` | VARCHAR(20) | | Measurement unit |
| `low_stock_threshold` | INT | | Alert level |
| `status` | TINYINT | | 1=Active, 0=Inactive |
| `created_at` | TIMESTAMP | | Creation timestamp |

### 4.2 Stock Entity

**Table**: `stock`

| Column | Type | Description |
|:---|:---|:---|
| `idstock` | CHAR(36) | UUID identifier |
| `idproduct` | CHAR(36) | Product reference |
| `idtenant` | CHAR(36) | Tenant reference |
| `idsite` | CHAR(36) | Location reference |
| `stock_level` | DECIMAL(12,4) | Current quantity |
| `last_updated` | TIMESTAMP | Last change |

### 4.3 Stock Type Entity

**Table**: `stock_type`

| Column | Type | Description |
|:---|:---|:---|
| `idstock_type` | CHAR(36) | UUID identifier |
| `idtenant` | CHAR(36) | Tenant reference |
| `name` | VARCHAR(100) | Type name |
| `status` | TINYINT | Active flag |

### 4.4 Purchase Request Entity

**Table**: `purchase_request`

| Column | Type | Constraints | Description |
|:---|:---|:---|:---|
| `id` | CHAR(36) | PK | UUID identifier |
| `idtenant` | CHAR(36) | FK | Tenant reference |
| `pr_number` | VARCHAR(50) | | Request number |
| `requested_by` | CHAR(36) | FK | User reference |
| `request_date` | DATE | | Submission date |
| `expected_date` | DATE | | When needed |
| `pr_status` | ENUM | | 'requested', 'approved', 'rejected', 'ordered' |
| `notes` | TEXT | | Comments |
| `created_at` | TIMESTAMP | | Creation timestamp |

### 4.5 Purchase Request Item Entity

**Table**: `purchase_request_item`

| Column | Type | Description |
|:---|:---|:---|
| `id` | CHAR(36) | UUID identifier |
| `purchase_request_id` | CHAR(36) | Parent request |
| `idproduct` | CHAR(36) | Product reference |
| `quantity` | DECIMAL(12,4) | Requested amount |
| `unit` | VARCHAR(20) | Measurement unit |

### 4.6 Purchase Order Entity

**Table**: `purchase_order`

| Column | Type | Constraints | Description |
|:---|:---|:---|:---|
| `id` | CHAR(36) | PK | UUID identifier |
| `idtenant` | CHAR(36) | FK | Tenant reference |
| `po_number` | VARCHAR(50) | UNIQUE | Order number |
| `idsupplier` | CHAR(36) | FK | Supplier reference |
| `order_date` | DATE | | Order date |
| `expected_delivery` | DATE | | Target date |
| `subtotal` | DECIMAL(12,2) | | Pre-tax |
| `vat_amount` | DECIMAL(12,2) | | Tax |
| `total` | DECIMAL(12,2) | | Final amount |
| `po_status` | ENUM | | 'draft', 'sent', 'partial', 'complete' |

### 4.7 Delivery Entity

**Table**: `delivery`

| Column | Type | Description |
|:---|:---|:---|
| `id` | CHAR(36) | UUID identifier |
| `purchase_order_id` | CHAR(36) | PO reference |
| `delivery_date` | DATE | Receipt date |
| `status` | ENUM | 'pending', 'validated', 'discrepancy' |

---

## 5. API Specification

### 5.1 Get Stock Levels

```
GET /api/v2/gtStock?idtenant={tenant}
Authorization: Bearer {token}

Response: 200 OK
[
  {
    "idproduct": "product-uuid",
    "name": "Fungicide X",
    "stock_level": 25.5,
    "reference_price": 45.00,
    "reference_unit": "L",
    "category_name": "Fungicides"
  }
]
```

### 5.2 Create Purchase Request

```
POST /api/v2/pr?idtenant={tenant}
Content-Type: application/json
Authorization: Bearer {token}

Request Body:
{
  "expected_date": "2025-12-30",
  "notes": "Urgent restock needed",
  "items": [
    { "idproduct": "product-uuid", "quantity": 10, "unit": "L" }
  ]
}

Response: 201 Created
{
  "id": "pr-uuid",
  "pr_number": "PR-2025-001"
}
```

### 5.3 Stock Take

```
POST /api/v2/stock-take?idtenant={tenant}
Content-Type: application/json
Authorization: Bearer {token}

Request Body:
{
  "items": [
    {
      "idproduct": "product-uuid",
      "system_qty": 25.5,
      "actual_qty": 24.0,
      "variance": -1.5
    }
  ],
  "adjustment_reason": "Breakage"
}

Response: 200 OK
{
  "adjustments_created": 1,
  "message": "Stock take completed"
}
```

### 5.4 Validate Delivery

```
POST /api/v2/delivery/{id}/validate?idtenant={tenant}
Content-Type: application/json
Authorization: Bearer {token}

Request Body:
{
  "items": [
    { "idproduct": "product-uuid", "received_qty": 10 }
  ]
}

Response: 200 OK
{
  "stock_updated": true,
  "po_status": "complete"
}
```

---

## 6. Sequence Diagrams

### 6.1 Purchase Workflow

```
┌──────┐    ┌──────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│ User │    │   PR     │    │Approver │    │   PO    │    │Delivery │
└──┬───┘    └────┬─────┘    └────┬────┘    └────┬────┘    └────┬────┘
   │  Submit PR  │               │              │               │
   │────────────>│               │              │               │
   │             │  Notify       │              │               │
   │             │──────────────>│              │               │
   │             │  Approve      │              │               │
   │             │<──────────────│              │               │
   │             │  Create PO    │              │               │
   │             │──────────────────────────────>              │
   │             │               │              │  Send to Supp │
   │             │               │              │──────────────>│
   │             │               │              │  Goods Arrive │
   │             │               │              │<──────────────│
   │             │               │              │  Validate     │
   │             │               │              │  Stock Update │
```

---

## 7. VAT & Tax Configuration

### 7.1 VAT Store Actions

```javascript
// Add new VAT rate
addVat(data) → POST /vat
editVat(id, data) → PUT /vat/{id}
deleteVat(id) → DELETE /vat/{id}

// Add new Tax rate
addTax(data) → POST /tax
editTax(id, data) → PUT /tax/{id}
deleteTax(id) → DELETE /tax/{id}
```

---

## 8. Integration Points

| System | Type | Data Flow |
|:---|:---|:---|
| Fleet Management | Transactional | Parts consumption |
| Task Module | Transactional | Product consumption |
| Cost Centers | Reference | Budget allocation |
| Reporting | Consumer | Spending analysis |
| Suppliers | Reference | Pricing and lead times |

---

## 9. Performance Requirements

| Metric | Requirement |
|:---|:---|
| Stock list load | < 3 seconds |
| Stock take submission | < 2 seconds |
| PO creation | < 2 seconds |
| Delivery validation | < 1 second |

---

**Document End**
