# Variant-Based Inventory Management System - Production Architecture

## System Overview

This document outlines a production-ready ERP system for managing variant-level inventory with strict financial integrity through:
- **Stock Ledger**: Transactional audit trail of all inventory movements
- **General Ledger**: Double-entry accounting for financial reconciliation
- **Product/Variant Hierarchy**: Parent products with multiple SKU variants

### Core Principles
1. **No Blind Overwrites**: Every inventory change is transactional and auditable
2. **Financial Integrity**: Every stock movement has corresponding GL entries
3. **Variant Specificity**: Operations target specific variants, never product-level
4. **Real-time Validation**: Quantity checks before any sale execution
5. **Cost Tracking**: Moving Average Cost (MAC) maintained per variant

---

## Module Overview

### Module 1: Stock Adjustment
- Corrects physical vs. book inventory discrepancies
- Generates Stock Ledger entries (delta calculation)
- Creates GL entries for variance gains/losses
- Maintains audit trail

### Module 2: Direct Purchase (Vendor Intake)
- Records vendor purchases with invoice metadata
- Updates variant stock quantities
- Recalculates Moving Average Cost
- Creates payables GL entries

### Module 3: Sales/Invoice (Point of Sale)
- Real-time variant-level stock validation
- Prevents sales if specific variant stock insufficient
- Records revenue and COGS
- Manages receivables GL entries

---

## Key Database Constraints

| Constraint | Purpose |
|-----------|---------|
| `FK_variants_products` | Ensures variants link to valid products |
| `FK_stock_ledger_variants` | Maintains referential integrity for ledger |
| `UNIQUE(sku_code, barcode)` | Prevents duplicate variant identifiers |
| `DEFAULT GETDATE()` | Automatic transaction timestamping |
| `CHECK(quantity >= 0)` | Prevents negative inventory states |

---

## Transaction Flow Diagrams

### Stock Adjustment Flow
```
User Posts Adjustment
    ↓
Validate Header/Details
    ↓
Calculate Delta (Physical - Book)
    ↓
BEGIN TRANSACTION
    ├─ Update variant quantities
    ├─ Insert Stock Ledger rows
    ├─ Insert GL Journal Entries
    └─ COMMIT/ROLLBACK
    ↓
Post Adjustment Header Status
```

### Purchase Order Flow
```
Vendor Invoice Received
    ↓
Create PO Header + Details
    ↓
BEGIN TRANSACTION
    ├─ Insert Stock Ledger (Receipt)
    ├─ Recalculate Moving Average Cost
    ├─ Update variant quantities
    ├─ Create GL Entries (AP + Inventory Asset)
    └─ COMMIT/ROLLBACK
    ↓
Mark PO as Posted
```

### Sales/Invoice Flow
```
Customer Creates Sale Order
    ↓
FOR EACH line item:
    ├─ CHECK variant stock (Real-time)
    ├─ IF stock < qty THEN HALT with error
    └─ IF stock >= qty THEN CONTINUE
    ↓
BEGIN TRANSACTION
    ├─ Deduct from variant stock
    ├─ Insert Stock Ledger (Sale)
    ├─ Insert GL Revenue Entry
    ├─ Insert GL COGS Entry (using variant cost basis)
    └─ COMMIT/ROLLBACK
    ↓
Mark Sale as Posted
```

---

## Accounting Rules

### Chart of Accounts (Essential)
```
Asset Accounts:
  1010 - Cash
  1020 - Accounts Receivable
  1110 - Inventory Asset (Variants tracked here)
  
Liability Accounts:
  2010 - Accounts Payable
  
Revenue Accounts:
  4010 - Sales Revenue
  
Expense Accounts:
  5010 - Cost of Goods Sold (COGS)
  5020 - Stock Variance Expense (Shortages)
  5021 - Stock Variance Income (Surpluses)
```

### Double-Entry Rules

**Stock Adjustment - Shortage (Physical < Book):**
```
DR: Stock Variance Expense     (5020)
  CR: Inventory Asset          (1110)
```

**Stock Adjustment - Surplus (Physical > Book):**
```
DR: Inventory Asset            (1110)
  CR: Stock Variance Income    (5021)
```

**Direct Purchase - Receipt:**
```
DR: Inventory Asset            (1110)
  CR: Accounts Payable         (2010)
```

**Sale - Revenue Recognition:**
```
DR: Cash/AR                    (1010/1020)
  CR: Sales Revenue            (4010)
```

**Sale - COGS Realization:**
```
DR: COGS                       (5010)
  CR: Inventory Asset          (1110)
```

---

## Key Performance Metrics

- **Stock Accuracy**: Variance between physical and system counts
- **Inventory Turnover**: Sales value / Average inventory cost
- **COGS Accuracy**: Proper costing at sale time
- **GL Reconciliation**: Asset account = Sum(variant quantities × cost)
