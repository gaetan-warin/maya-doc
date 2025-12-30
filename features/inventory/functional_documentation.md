# Inventory Management
## Functional Requirements Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-FRS-INV-001 |
| **Version** | 1.0 |
| **Last Updated** | December 30, 2025 |
| **Status** | Approved |
| **Owner** | Product Team |
| **Classification** | Internal |

---

## Executive Summary

The **Inventory Management** module provides comprehensive control over all consumable products, machinery parts, and materials used in turf and facility management. It encompasses four integrated sub-modules: **Stock** (quantity tracking), **Stock Items** (product catalog), **Purchase Orders** (procurement workflow), and **Deliveries** (receiving goods).

This document defines the functional requirements for inventory tracking, procurement workflows, supplier management, and cost allocation. It integrates with Fleet Management for parts consumption and with Financial Reporting for budget analysis.

**Key Business Outcomes:**
- **Visibility**: Real-time stock levels prevent operational delays from stockouts.
- **Cost Control**: Track spending by cost center with supplier price comparisons.
- **Compliance**: Complete audit trail of purchases and consumption.
- **Efficiency**: Streamlined procurement from request to invoice.

---

## 1. Scope & Objectives

### 1.1 In Scope
- Stock level tracking by product and location
- Product catalog management with category hierarchies
- Supplier relationship and pricing management
- Purchase requisition and approval workflow
- Purchase order creation and tracking
- Delivery receiving and stock adjustment
- Invoice reconciliation
- Stock take (physical inventory count)
- Stock transfers between locations

### 1.2 Out of Scope
- Automated reorder point triggers (future enhancement)
- Barcode/RFID scanning integration
- Contract management with suppliers

### 1.3 Target Users
| Role | Primary Use Case |
|:---|:---|
| **Warehouse Manager** | Stock level monitoring, receiving deliveries |
| **Purchasing Agent** | Creating and managing purchase orders |
| **Grounds Staff** | Submitting purchase requests |
| **Finance Manager** | Invoice reconciliation, cost analysis |

---

## 2. Stock Module

### 2.1 Overview

The Stock module provides real-time visibility of all product quantities across the organization.

### 2.2 Stock Dashboard

| Component | Description |
|:---|:---|
| **KPI Summary** | Total stock value, low stock alerts, out of stock count |
| **Tab Navigation** | All / Agrochemical / Fertilizer / Seed / Parts / History |
| **Spending Overview** | Link to financial analysis |

### 2.3 Stock Table Columns

| Column | Description |
|:---|:---|
| Name | Product name |
| Stock Level | Current quantity (red if ≤ 0) |
| Low in Stock | Boolean threshold indicator |
| Reference Price | Unit cost |
| Reference Unit | Measurement unit (L, kg, etc.) |
| Category | Product classification |

### 2.4 Stock Actions

| Action | Description |
|:---|:---|
| **Adjust** | Manual stock level correction |
| **Update** | Edit product details |
| **Transfer** | Move stock between locations |
| **Stock Take** | Record physical inventory count |

### 2.5 Stock History Tab

View all stock movements with:
- Date and time of transaction
- Transaction type (adjustment, delivery, consumption)
- Quantity change
- User who performed action
- Related document reference

---

## 3. Stock Items Module

### 3.1 Overview

Stock Items is the master product catalog containing all products and parts that can be stocked and purchased.

### 3.2 Tab Structure

| Tab | Purpose |
|:---|:---|
| **Detail** | Core product information |
| **Supplier** | Linked suppliers for this product |
| **Price** | Pricing by supplier |
| **Cost Center** | Budget allocation settings |
| **Category** | Category hierarchy management |

### 3.3 Product Detail Fields

| Field | Type | Description |
|:---|:---|:---|
| Name | Text | Product name |
| Stock Type | Dropdown | Agrochemical, Fertilizer, Seed, etc. |
| Category | Dropdown | Sub-classification |
| Reference Price | Decimal | Standard unit cost |
| Reference Unit | Dropdown | L, kg, unit, etc. |
| Low Stock Threshold | Integer | Alert trigger level |
| Status | Toggle | Active / Inactive |

### 3.4 Supplier Tab

Link multiple suppliers to a single product:
- Supplier name and contact
- Supplier's product code/SKU
- Lead time expectation
- Preferred supplier flag

### 3.5 Price Tab

Define pricing per supplier:
| Field | Description |
|:---|:---|
| Supplier | Source company |
| Unit Price | Cost per unit |
| Currency | Pricing currency |
| Valid From/To | Price validity period |

### 3.6 Cost Center Tab

Allocate product costs to budget centers:
- Default cost center assignment
- Allocation percentage (for split costs)

### 3.7 Category Hierarchy

```
Stock Type (Level 1)
└── Product Category (Level 2)
    └── Sub-Category (Level 3 - optional)
```

Example:
```
Agrochemicals
├── Fungicides
│   ├── Systemic
│   └── Contact
├── Herbicides
└── Insecticides
```

---

## 4. Purchase Orders Module

### 4.1 Overview

The Purchase Orders module manages the complete procurement lifecycle from request to invoice.

### 4.2 Tab Structure

| Tab | Purpose | Counts |
|:---|:---|:---|
| **Purchase Request** | Initial requisitions | # of "Requested" status |
| **Purchase Order** | Approved orders | # of "Draft" status |
| **Invoice** | Financial documents | # awaiting action |

### 4.3 Purchase Request Workflow

```
┌──────────┐    ┌──────────┐    ┌───────────┐    ┌──────────┐
│ Requested│───▶│ Approved │───▶│ Ordered   │───▶│ Completed│
└──────────┘    └──────────┘    └───────────┘    └──────────┘
      │
      ▼
┌──────────┐
│ Rejected │
└──────────┘
```

### 4.4 Purchase Request Fields

| Field | Description |
|:---|:---|
| Request Date | Submission date |
| Requested By | User submitting |
| Products | Line items with quantities |
| Expected Date | When needed |
| Notes | Justification or details |
| Status | Current workflow state |

### 4.5 Purchase Order Creation

When a request is approved:
1. Select supplier for each line item
2. Set unit prices
3. Apply VAT/Tax rates
4. Set expected delivery date
5. Generate PO number
6. Send to supplier (email/print)

### 4.6 Purchase Order Fields

| Field | Description |
|:---|:---|
| PO Number | Unique identifier |
| Supplier | Vendor |
| Order Date | Date issued |
| Expected Delivery | Target date |
| Line Items | Products, quantities, prices |
| Subtotal | Pre-tax total |
| VAT | Tax amount |
| Total | Final amount |
| Status | Draft / Sent / Partial / Complete |

### 4.7 Invoice Tab

Match invoices to purchase orders:
- Invoice number entry
- Invoice date
- Amount verification
- Discrepancy flagging
- Payment status tracking

---

## 5. Deliveries Module

### 5.1 Overview

The Deliveries module tracks incoming shipments and automatically updates stock levels when goods are received.

### 5.2 Delivery Process

1. **Expected Delivery**: Auto-generated from PO expected dates
2. **Receive Delivery**: Confirm actual quantities received
3. **Validate**: Stock levels automatically updated
4. **Discrepancy**: Flag any quantity differences

### 5.3 Delivery Record Fields

| Field | Description |
|:---|:---|
| Delivery Date | Date received |
| PO Reference | Linked purchase order |
| Supplier | Vendor name |
| Products | Received line items |
| Expected Qty | Ordered quantity |
| Received Qty | Actual quantity |
| Status | Pending / Validated / Discrepancy |

### 5.4 Automatic Stock Update

Upon delivery validation:
- Stock levels increase by received quantity
- Stock history entry created
- PO status updated (Partial or Complete)

---

## 6. Stock Take (Physical Inventory)

### 6.1 Purpose

Reconcile physical inventory counts with system records.

### 6.2 Stock Take Process

1. **Initiate**: Create stock take session
2. **Count**: Enter actual quantities per product
3. **Compare**: System shows variance
4. **Adjust**: Create adjustment entries for discrepancies
5. **Complete**: Finalize and lock session

### 6.3 Stock Take Record

| Field | Description |
|:---|:---|
| Date | Count date |
| Products | Items counted |
| System Qty | Expected quantity |
| Actual Qty | Counted quantity |
| Variance | Difference |
| Adjustment | Auto-generated correction |

---

## 7. Transfer Stocks

### 7.1 Purpose

Move inventory between locations or courses within the organization.

### 7.2 Transfer Fields

| Field | Description |
|:---|:---|
| From Location | Source |
| To Location | Destination |
| Products | Items to transfer |
| Quantity | Amount to move |
| Transfer Date | When executed |

---

## 8. Integration Points

| System | Integration Type | Data Flow |
|:---|:---|:---|
| **Fleet Management** | Transactional | Parts consumption from maintenance |
| **Task Module** | Transactional | Product consumption from tasks |
| **Cost Centers** | Reference | Budget allocation |
| **Reporting** | Consumer | Spending analysis |

---

## 9. Acceptance Criteria

1. Products can be created with all required fields and appear in stock list.
2. Purchase requests can be submitted and flow through approval workflow.
3. Approved requests can be converted to purchase orders with supplier selection.
4. Deliveries update stock levels automatically upon validation.
5. Stock take sessions record variances and create adjustment entries.
6. Transfers decrease source location and increase destination location.
7. Invoice matching flags discrepancies from PO amounts.

---

**Document End**
