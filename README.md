# 🔄 Alteryx Transaction Reconciliation Workflow

> Reconcile customer orders with return data across multiple sources and produce a unified **Master Transaction Report**.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Workflow Architecture](#workflow-architecture)
- [Data Sources](#data-sources)
- [Tool Inventory](#tool-inventory)
- [Pipeline Logic](#pipeline-logic)
- [Outputs](#outputs)
- [Getting Started](#getting-started)
- [File Structure](#file-structure)
- [Data Dictionary](#data-dictionary)
- [Known Issues & Edge Cases](#known-issues--edge-cases)
- [Contributing](#contributing)

---

## Overview

This Alteryx Designer workflow (`alteryxproject.yxmd`) consolidates transactional data from **four heterogeneous sources** — an orders table, a completed-returns log, an in-store transactions CSV, and a customers master — into two clean output datasets:

| Output | Description |
|--------|-------------|
| `Master_Transaction_Report.yxdb` | All transactions with customer details, return flags, and deduplication applied |
| `Reconciled_Customer_Returns.yxdb` | Return records enriched with customer metadata, ready for reconciliation reporting |

**Key problems this workflow solves:**

- Merges online orders (`Orders Table`) with in-store transactions (`STORE_TRANSACTIONS_ALL_AUG_2021`)
- Reconciles the `RETURNED` flag against the official August 2021 returns log
- Handles **duplicate `CUSTOMER_ID`** values via a Find & Replace lookup table
- Enriches every transaction with full customer profile data from the Customers master
- Produces a single canonical transaction set by Union-ing all branches

---

## Workflow Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        DATA INGESTION LAYER                                 │
│                                                                             │
│  [1] Orders Table.xlsx     [3] Returns Aug-2021.xlsx   [4] Store Txns.csv  │
│         │                          │                         │              │
│         ▼                          ▼                         ▼              │
│  [5] Select (type fix)    [6] Select (passthrough)   [7] Select (type fix) │
│  CUSTOMER_ID→Int64         RETURNED passthrough       QUANTITY/DATE/etc     │
│         │                          │                         │              │
└─────────┼──────────────────────────┼─────────────────────────┼─────────────┘
          │                          │                         │
┌─────────┼──────────────────────────┼─────────────────────────┼─────────────┐
│                       CUSTOMER ENRICHMENT LAYER                             │
│                                                                             │
│  [2] Customers.xlsx                                                         │
│         │                                                                   │
│  [18] Select (ID→Int64)                                                     │
│         │                                                                   │
│  [25] Join (Orders ⋈ Customers on CUSTOMER_ID=ID)                          │
│    ├── Left (unmatched) ──►  [26] Union                                     │
│    └── Join (matched)   ──►  [26] Union ─────────────────────────────────► │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
          │
┌─────────┼───────────────────────────────────────────────────────────────────┐
│                       RETURNS RECONCILIATION LAYER                          │
│                                                                             │
│  [8] Join (Orders branch ⋈ Returns on ORDER_ID)                            │
│    ├── Left (no return match) → [29] Formula: RETURNED=0 → [19] Union      │
│    └── Join (return matched)  → [20] Select    → [19] Union                │
│                                        │                                    │
│  [11] Join (Store Txns ⋈ Returns on ORDER_ID)                              │
│    ├── Left (no return match) → [24] Formula: RETURNED=0 → [23] Union      │
│    └── Join (return matched)  → [23] Union                                 │
│                                        │                                    │
│  [21] Union (Orders returns + Store Txns returns)                           │
│         │                                                                   │
│  [22] OUTPUT: Reconciled_Customer_Returns.yxdb                              │
└─────────────────────────────────────────────────────────────────────────────┘
          │
┌─────────┼───────────────────────────────────────────────────────────────────┐
│                    DEDUPLICATION & MASTER REPORT LAYER                      │
│                                                                             │
│  [10] Select (Store Txns, type fixes)                                       │
│    └──► [12] Unique (deduplicate on ORDER_ID)                               │
│           ├── Unique records    ──────────────────────────► [27] Union      │
│           └── Duplicates  → [13] FindReplace (CUSTOMER_ID) → [27] Union    │
│                                       ▲                                     │
│                            [14] TextInput (ID mapping: 1468427→1468427)    │
│                            [30] Select (Old_ID/New_ID→Int64)               │
│                                                                             │
│  [27] Union → [15] Join (Txns ⋈ Customers on CUSTOMER_ID=ID)              │
│                  ├── Right (unmatched customers) ─► [16] Union             │
│                  └── Join (matched)              ─► [16] Union             │
│                                                         │                  │
│  [17] OUTPUT: Master_Transaction_Report.yxdb ◄──────────┘                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Data Sources

| Tool ID | File | Format | Description |
|---------|------|--------|-------------|
| 1 | `Orders Table (2).xlsx` — `Sheet1$` | Excel | Online order records with product & return fields |
| 2 | `CUSTOMERS (1).xlsx` — `Sheet1$` | Excel | Customer master: name, email, address, phone |
| 3 | `Completed Returns - August 2021 (1).xlsx` — `Sheet1$` | Excel | Official return confirmations for Aug 2021 |
| 4 | `STORE_TRANSACTIONS_ALL_AUG_2021 (3).csv` | CSV (Latin-1 / ISO-8859-1) | All in-store POS transactions for August 2021 |

> ⚠️ **Local paths are hardcoded** to `C:\Users\LENOVO\Downloads\`. Update these paths before running on a new machine (see [Getting Started](#getting-started)).

---

## Tool Inventory

| Tool ID | Plugin | Purpose |
|---------|--------|---------|
| 1–4 | `DbFileInput` | Read all four source files |
| 5, 6, 7, 9, 10, 18, 20, 30 | `AlteryxSelect` | Type casting (Double→Int64, String→DateTime, etc.) and field cleanup |
| 8, 11, 15, 25 | `Join` | Inner joins across Orders ↔ Returns ↔ Customers ↔ Store Txns |
| 12 | `Unique` | Deduplicate store transactions by `ORDER_ID` |
| 13 | `FindReplace` | Fix duplicate `CUSTOMER_ID` values using a lookup table |
| 14 | `TextInput` | Hardcoded ID correction lookup (`Old_ID` → `New_ID`) |
| 16, 19, 21, 23, 26, 27 | `Union` | Merge matched and unmatched branches back together (by name) |
| 17, 22 | `DbFileOutput` | Write `.yxdb` output files |
| 24, 29 | `Formula` | Set `RETURNED = 0` (False) for records with no return match |
| 28 | `TextBox` | Workflow description annotation (non-processing) |

---

## Pipeline Logic

### Branch 1 — Reconciled Customer Returns (`→ Tool 22`)

1. Orders are type-cast and joined against the Customers master (**Tool 25**) — unmatched orders are kept via the Left output.
2. The resulting set joins against the Returns log (**Tool 8**) on `ORDER_ID`:
   - **Matched** → return flag carried through
   - **Unmatched** → `RETURNED` set to `0` via Formula (**Tool 29**)
3. Store transactions repeat the same Returns join (**Tool 11**) with the same fallback (**Tool 24**).
4. Both branches are Union-ed (**Tools 19, 21**) → written to `Reconciled_Customer_Returns.yxdb`.

### Branch 2 — Master Transaction Report (`→ Tool 17`)

1. Store transactions are type-cast (**Tool 7 → 10**) and de-duplicated on `ORDER_ID` (**Tool 12**).
2. Duplicate `CUSTOMER_ID` records are corrected using a Find & Replace lookup (**Tools 14 → 30 → 13**).
3. Deduplicated + corrected records are Union-ed (**Tool 27**) then joined with the Customers master (**Tool 15**):
   - All records (matched and unmatched) are retained via Union (**Tool 16**).
4. Written to `Master_Transaction_Report.yxdb`.

---

## Outputs

### `Master_Transaction_Report.yxdb`

Full schema (all fields from Store Transactions enriched with Customer data):

| Field | Type | Source |
|-------|------|--------|
| ORDER_ID | V_String | Store Txns / Orders |
| PRODUCT_SKU | V_String | Store Txns / Orders |
| QUANTITY | Double | Store Txns / Orders |
| ORDER_DATE | DateTime | Store Txns / Orders |
| CUSTOMER_ID | Int64 | Store Txns / Orders (deduped) |
| PRODUCT_NAME | V_String | Store Txns / Orders |
| PRODUCT_DESC | V_String | Store Txns / Orders |
| PRODUCT_PRICE | Double | Store Txns / Orders |
| PRODUCT_CATEGORY | V_String | Store Txns / Orders |
| FirstName | V_String | Customers |
| LastName | V_String | Customers |
| UserName | V_String | Customers |
| Email | V_String | Customers |
| State | V_String | Customers |
| City | V_String | Customers |
| Street | V_String | Customers |
| PostCode | V_String | Customers |
| Phone | V_String | Customers |

### `Reconciled_Customer_Returns.yxdb`

Contains the same schema as above, with an additional reconciled `RETURNED` (Bool) field reflecting the August 2021 returns log.

---

## Getting Started

### Prerequisites

- **Alteryx Designer** 2026.1 or compatible version
- Input files placed in an accessible directory

### Setup

1. **Clone this repository**
   ```bash
   git clone https://github.com/your-org/alteryx-transaction-reconciliation.git
   cd alteryx-transaction-reconciliation
   ```

2. **Place your data files** in the `data/` folder:
   ```
   data/
   ├── Orders Table (2).xlsx
   ├── CUSTOMERS (1).xlsx
   ├── Completed Returns - August 2021 (1).xlsx
   └── STORE_TRANSACTIONS_ALL_AUG_2021 (3).csv
   ```

3. **Update file paths** in the workflow:
   Open `alteryxproject.yxmd` in Alteryx Designer and update the four Input Data tools (Tool IDs 1–4) to point to your local `data/` folder.

   Alternatively, use the provided helper script:
   ```bash
   python scripts/update_paths.py --data-dir "C:/path/to/your/data"
   ```

4. **Run the workflow** in Alteryx Designer (Ctrl+R) or via the Alteryx Engine:
   ```bash
   AlteryxEngineCmd.exe alteryxproject.yxmd
   ```

5. **Outputs** will be written to:
   - `data/output/Master_Transaction_Report.yxdb`
   - `data/output/Reconciled_Customer_Returns.yxdb`

---

## File Structure

```
alteryx-transaction-reconciliation/
│
├── alteryxproject.yxmd              # Main Alteryx workflow
│
├── data/
│   ├── sample/                      # Anonymised sample data for testing
│   │   ├── orders_sample.xlsx
│   │   ├── customers_sample.xlsx
│   │   ├── returns_sample.xlsx
│   │   └── store_transactions_sample.csv
│   └── output/                      # (gitignored) runtime outputs
│
├── docs/
│   ├── workflow_diagram.png         # Visual flow diagram
│   ├── data_dictionary.md           # Full field-level documentation
│   └── design_decisions.md         # Notes on join strategy, deduplication logic
│
├── scripts/
│   ├── update_paths.py              # CLI tool to repoint file paths in the .yxmd
│   └── validate_inputs.py          # Pre-run data quality checks
│
├── tests/
│   └── test_output_schema.py        # Validate output field counts and types
│
├── .github/
│   └── workflows/
│       └── validate.yml             # CI: schema validation on PR
│
├── .gitignore
├── CHANGELOG.md
└── README.md                        # This file
```

---

## Data Dictionary

### Orders Table (`ORDER_ID`, `PRODUCT_SKU`, `QUANTITY`, `ORDER_DATE`, `CUSTOMER_ID`, `RETURNED`, `PRODUCT_NAME`, `PRODUCT_DESC`, `PRODUCT_PRICE`, `PRODUCT_CATEGORY`)

| Field | Type | Notes |
|-------|------|-------|
| ORDER_ID | V_String (255) | Unique order identifier |
| PRODUCT_SKU | V_String (255) | Stock-keeping unit code |
| QUANTITY | Double → cast to Int via pipeline | Units ordered |
| ORDER_DATE | DateTime (19) | ISO format `YYYY-MM-DD HH:MM:SS` |
| CUSTOMER_ID | Double → **Int64** after Select Tool 5 | Foreign key to Customers |
| RETURNED | Bool | Pre-join flag; overwritten by Returns reconciliation |
| PRODUCT_NAME | V_String (255) | Display product name |
| PRODUCT_DESC | V_String (255) | Product description |
| PRODUCT_PRICE | Double | Unit price at time of order |
| PRODUCT_CATEGORY | V_String (255) | Top-level product category |

### Customers (`ID`, `FirstName`, `LastName`, `UserName`, `Email`, `State`, `City`, `Street`, `PostCode`, `Phone`)

| Field | Type | Notes |
|-------|------|-------|
| ID | Double → **Int64** after Select Tool 18 | Primary key |
| FirstName | V_String (255) | |
| LastName | V_String (255) | |
| UserName | V_String (255) | Login handle |
| Email | V_String (255) | |
| State | V_String (255) | US state abbreviation or full name |
| City | V_String (255) | |
| Street | V_String (255) | Street address line |
| PostCode | V_String (255) | Stored as string to preserve leading zeros |
| Phone | V_String (255) | |

### Returns Log (`ORDER_ID`, `RETURNED`)

| Field | Type | Notes |
|-------|------|-------|
| ORDER_ID | V_String (255) | Matches Orders.ORDER_ID |
| RETURNED | Bool | Always `True` in this file (it's a completed returns log) |

### Store Transactions CSV

Same schema as Orders Table but all fields arrive as `V_String (254)` and require explicit type casting (handled by Select Tool 7).

---

## Known Issues & Edge Cases

| Issue | Handling |
|-------|----------|
| **Duplicate CUSTOMER_ID** (e.g., `1468427`) | Detected by Unique Tool 12; corrected by FindReplace Tool 13 using the TextInput lookup (Tool 14). To add more corrections, extend the TextInput table. |
| **Unmatched orders** (no customer record) | Kept via the Left output of Join tools 25 and 15 — they appear in outputs with null customer fields. |
| **CSV encoding** | Store transactions file uses ISO-8859-1 (Latin-1, code page 28591). If you see garbled characters, verify the file encoding before ingestion. |
| **DateTime parsing on CSV** | `ORDER_DATE` arrives as V_String from the CSV; Select Tool 7 casts it to DateTime. Ensure source dates match `YYYY-MM-DD HH:MM:SS` format. |
| **Hardcoded file paths** | All four input tools reference `C:\Users\LENOVO\Downloads\`. Use `scripts/update_paths.py` to relocate. |
| **Returns flag default** | Records with no match in the Returns log default to `RETURNED = 0 (False)` via Formula Tools 24 and 29. |

---

## Contributing

1. Fork the repository and create a feature branch: `git checkout -b feature/my-change`
2. Make your changes to the workflow or supporting scripts
3. Run schema validation: `python tests/test_output_schema.py`
4. Commit with a descriptive message and open a Pull Request

Please document any new data sources or logic changes in `docs/design_decisions.md`.

---

## License

This project is for internal use. See `LICENSE` for details.
