# Data Cleaning Log
### Shohoz Express Ltd. — Logistics & Delivery Performance Analysis

This document provides a complete record of every data quality issue identified and resolved during the cleaning phase. All cleaning was performed in **Microsoft Excel** using built-in functions and formulas. Each issue is documented with its root cause, the logic behind the chosen fix, the exact formula or method used, and the final outcome.

**Tables cleaned:** Shipments (5,100 rows → 5,000 rows), Carriers (7 rows → 6 rows), Routes (10 rows)
**Total issues resolved:** 11 (Shipments) + 6 (Carriers) + 4 (Routes) = 21 issues

---

## Table: Shipments

### Issue 1 — Inconsistent Date Formats

**Affected columns:** `Order_Date`, `Promised_Delivery_Date`, `Actual_Delivery_Date`
**Scope:** All ~5,100 rows, split roughly evenly across 5 different formats

**Problem:**
The same date columns contained five different date format styles mixed together — `DD/MM/YYYY`, `MM-DD-YYYY`, `DD-MMM-YYYY` (e.g., 01-Apr-2026), `YYYY.MM.DD`, and `YYYY/MM/DD`. This made it impossible to sort, filter, or calculate date differences reliably.

**Fix — Format-specific formulas with Filter Wildcards:**
Excel's Filter search box was used with wildcard patterns (e.g., `??/??/????`, `??-???-????`) to isolate each format group. A temporary column was then used with a format-specific formula for each group before pasting values back.

| Format | Wildcard Used | Formula Applied |
|---|---|---|
| `DD/MM/YYYY` | `??/??/????` | `=TEXT(DATE(RIGHT(A2,4), MID(A2,4,2), LEFT(A2,2)), "yyyy-mm-dd")` |
| `MM-DD-YYYY` | `??-??-????` | `=TEXT(DATE(RIGHT(A2,4), LEFT(A2,2), MID(A2,4,2)), "yyyy-mm-dd")` |
| `DD-MMM-YYYY` | `??-???-????` | `=TEXT(DATEVALUE(A2), "yyyy-mm-dd")` |
| `YYYY.MM.DD` | `????.??.??` | `=TEXT(DATE(LEFT(A2,4), MID(A2,6,2), RIGHT(A2,2)), "yyyy-mm-dd")` |
| `YYYY/MM/DD` | `????/??/??` | `=TEXT(DATE(LEFT(A2,4), MID(A2,6,2), RIGHT(A2,2)), "yyyy-mm-dd")` |

> **Note on DD-MMM-YYYY:** Excel natively recognizes abbreviated month names (e.g., "Apr", "Aug"), so `DATEVALUE()` converts the string directly to a date serial without manual parsing. The other four formats required positional text extraction using `LEFT`, `MID`, and `RIGHT`.

**Outcome:** All date values standardized to `YYYY-MM-DD` format across all three date columns.

---

### Issue 2 — Missing Values in `Actual_Delivery_Date`

**Scope:** 525 blank cells identified

**Problem:**
Initial screening revealed that most blank `Actual_Delivery_Date` values corresponded to rows where `Status = "In Transit"` — these are logically expected (the shipment has not yet been delivered). However, a deeper check found **174 rows** where `Status = "Delivered"` but `Actual_Delivery_Date` was still blank. This is a direct data integrity/consistency error.

**Fix — Data Flagging:**
Rather than guessing or deleting these rows, the 174 problematic blank cells were explicitly flagged with the text `"Missing Date"`. This preserves the original row structure and makes these anomalies easy to filter and investigate separately during analysis.

The remaining blanks (Status = "In Transit") were left blank, as the absence of a delivery date is factually correct for in-transit shipments.

**Outcome:** 174 rows flagged as `"Missing Date"`. Remaining 351 blanks retained as expected nulls for in-transit shipments.

---

### Issue 3 — Missing Values in `Distance_KM`

**Scope:** 205 blank cells

**Problem:**
Distance values were missing for approximately 4% of shipments. Filling these with an arbitrary global average would have been statistically inappropriate, as route distances vary significantly across the 10 routes.

**Fix — Route-Average Imputation using `AVERAGEIF`:**
A temporary column was created beside `Distance_KM`. The following formula calculated the average distance for the specific route of each row and filled only blank cells:

```excel
=IF(ISBLANK(G2), AVERAGEIF(F:F, F2, G:G), G2)
```
*(F = Route_ID column, G = Distance_KM column)*

After the formula populated all values, the temporary column was copied and **Paste Special → Values** was applied back to the original `Distance_KM` column. The temporary column was then deleted.

**Outcome:** All 205 missing distance values filled with their respective route-level average. No global average was used.

---

### Issue 4 — Missing Values in `Weight_KG`

**Scope:** 203 blank cells

**Problem:**
Weight values were missing for approximately 4% of shipments. Unlike `Distance_KM`, the dataset does not contain a product category or product ID column that could be used for group-level imputation.

**Fix — Global Mean Imputation using `AVERAGE`:**
Given the absence of a meaningful grouping variable, the statistically standard approach of filling with the overall column average was applied using a temporary column:

```excel
=IF(ISBLANK(I2), AVERAGE(I:I), I2)
```
*(I = Weight_KG column)*

The result was pasted back as values and the temporary column was removed.

**Outcome:** All 203 missing weight values filled with the global average weight across the dataset.

---

### Issue 5 — Logical vs. True Missing Values in `Delay_Reason`

**Scope:** 3,180 blank/NaN cells (~62% of rows)

**Problem:**
The `Delay_Reason` column had a very high blank rate. A naive approach would treat all 3,180 blanks as "missing data." However, on-time shipments have no delay reason by definition — their blanks are **logically expected**, not errors. The blanks needed to be classified into three distinct categories before any analysis.

**Fix — Nested IF formula with date comparison:**
Since no explicit on-time flag column existed, on-time vs. late delivery was determined by comparing `Promised_Delivery_Date` (column C) and `Actual_Delivery_Date` (column D). A temporary column was used with the following nested IF formula:

```excel
=IF(ISBLANK(K2), IF(D2="Missing Date", "Review Required", IF(D2<=C2, "None", "Missing Reason")), K2)
```
*(K = Delay_Reason column)*

**Classification logic:**

| Category | Condition | Meaning |
|---|---|---|
| `None` | `Actual_Delivery_Date <= Promised_Delivery_Date` | On-time or early delivery — no delay reason expected |
| `Missing Reason` | Late delivery but `Delay_Reason` was blank | True missing data — delay occurred but cause was not recorded |
| `Review Required` | `Actual_Delivery_Date = "Missing Date"` | Delivery date itself is missing — cannot determine if delayed |

**Outcome:** All 3,180 blanks reclassified: majority set to `"None"` (on-time deliveries), 76 flagged as `"Missing Reason"`, and 9 flagged as `"Review Required"`.

---

### Issue 6 — Duplicate Rows

**Scope:** 100 exact full-row duplicates

**Problem:**
100 rows were found where all 11 columns were identical to another row — a clear data entry or system export error.

**Fix — Excel's built-in Remove Duplicates tool:**
Rather than using Power Query, Excel's native **Data → Remove Duplicates** was used:
- **My data has headers** was checked to protect the header row
- **Select All** (all 11 columns) was checked to ensure only rows that are identical across every field were removed (not just matching Shipment_IDs with different values in other columns)

**Outcome:** 100 duplicate rows removed. Dataset reduced from 5,100 to 5,000 rows.

---

### Issue 7 — Inconsistent Text Casing in `Status`

**Scope:** All 5,100 rows

**Problem:**
The `Status` column contained the same three values written in inconsistent cases — `"Delivered"`, `"DELIVERED"`, `"delivered"`, `"In Transit"`, `"IN TRANSIT"`, `"in transit"`, etc. This would cause incorrect grouping in any PivotTable or formula.

**Fix — `PROPER()` function via temporary column:**
```excel
=PROPER(E2)
```
*(E = Status column)*

The formula results were copied and **Paste Special → Values** was applied back to the original column. The temporary column was then deleted.

**Outcome:** All Status values standardized to Proper Case: `"Delivered"`, `"In Transit"`, `"Returned"`.

---

### Issue 8 — Extra Whitespace in `Delay_Reason`

**Scope:** ~5% of non-blank Delay_Reason values had leading, trailing, or double internal spaces

**Problem:**
Extra whitespace around text values causes silent failures in text matching and filtering — `" Traffic"` and `"Traffic"` look the same visually but are treated as different values by Excel.

**Fix — `TRIM()` function via temporary column:**
```excel
=TRIM(K2)
```
*(K = Delay_Reason column)*

The formula results were pasted back as values and the temporary column was removed.

**Outcome:** All leading, trailing, and excess internal whitespace removed from `Delay_Reason`.

---

### Issue 9 — Typos in `Delay_Reason` Categorical Values

**Scope:** ~25% of non-blank Delay_Reason values

**Problem:**
Manual data entry introduced spelling errors across all five delay reason categories. These typos would cause incorrect grouping — `"Trafic"` and `"Traffic"` would appear as separate categories in a PivotTable.

**Fix — Find and Replace (Ctrl + H):**
Instead of complex Power Query fuzzy matching, Excel's **Find and Replace** was used to correct each typo directly:

| Incorrect Value | Corrected To |
|---|---|
| `Trafic` | `Traffic` |
| `Wether`, `Weathr` | `Weather` |
| `Warehose Delay`, `Warehouse Dealy` | `Warehouse Delay` |
| `Vehical Breakdown`, `Vehicle Breakdwn` | `Vehicle Breakdown` |
| `Wrong Adress`, `Wrong Addres` | `Wrong Address` |

**Outcome:** All Delay_Reason values corrected to one of the 5 standard categories.

---

### Issue 10 — Wrong Data Type in `Distance_KM` (Numeric as Text)

**Affected column:** `Distance_KM`
**Scope:** ~15% of rows had values like `"18 km"` instead of `18`

**Problem:**
The text suffix `" km"` caused the entire column to be treated as text by Excel, making SUM, AVERAGE, and any numerical calculation impossible.

**Fix — Find and Replace to strip the unit suffix:**
1. The `Distance_KM` column was fully selected
2. **Find and Replace (Ctrl + H)** was opened
3. `Find what`: ` km` (with space) → `Replace with`: (left empty) → **Replace All**
4. After the suffix was removed, the column format was changed to **General/Number** to convert all remaining text strings to proper numeric values

**Outcome:** All 15% of affected cells converted to clean numeric values. Column is now fully numeric and ready for calculations.

---

### Issue 11 — Negative / Impossible Values in `Shipment_Cost`

**Scope:** 21 rows with negative cost values (~0.4% of dataset)

**Problem:**
Shipment cost cannot logically be negative. These values were identified as data entry sign errors — the magnitude of the cost is correct but the sign was accidentally entered as negative.

**Fix — `ABS()` function via temporary column:**
```excel
=ABS(H2)
```
*(H = Shipment_Cost column)*

This converts negative values to their positive equivalent while leaving already-positive values unchanged. The result was pasted back as values and the temporary column was removed.

**Outcome:** All 21 negative cost values converted to positive. No cost values were deleted or estimated.

---

### Issue 12 — Mixed Currency Notation in `Shipment_Cost`

**Scope:** Mixed across all rows — plain numbers, `"BDT 1,500"` prefixed values, and comma-separated values

**Problem:**
Inconsistent notation (e.g., `1500`, `BDT 1500`, `1,500`) caused the column to be read as text in some rows, breaking numerical operations.

**Fix — Find and Replace to strip prefix and separators:**
- `Find: BDT ` → `Replace: ` (empty)
- `Find: ,` → `Replace: ` (empty)
- Column format changed to **Number** after cleanup

**Outcome:** All Shipment_Cost values standardized to plain numeric format.

---

## Table: Carriers

| # | Issue | Fix Applied |
|---|---|---|
| 13 | Duplicate row (Sundarban Logistics appeared twice under two ID formats) | Standardized Carrier_ID first, then removed resulting duplicate via **Remove Duplicates** |
| 14 | Inconsistent Carrier_ID formats (`C-01`, `C1`, `1`) | Applied `="C-0"&RIGHT(B2,1)` to extract the numeric part and rebuild in standard format |
| 15 | Inconsistent Vehicle_Type casing (`TRUCK`, `truck`, `Truck`) | Applied `=PROPER()` via temporary column, pasted back as values |
| 16 | Extra whitespace in Carrier_Name | Applied `=TRIM()` via temporary column, pasted back as values |
| 17 | Missing Base_Rate_per_KM for Karatoa Express | Imputed using the average rate of the same Vehicle_Type (Truck: average of Sundarban Logistics and Jamuna Cargo Movers) → **18.75** |
| 18 | Inconsistent Contract_Start_Date formats | Same 5-format standardization approach as Shipments date columns |

---

## Table: Routes

| # | Issue | Fix Applied |
|---|---|---|
| 19 | Inconsistent Region_Name casing (`dhaka`, `DHAKA`, `Dhaka`) | Applied `=PROPER()` via temporary column |
| 20 | Inconsistent Zone casing (`urban`, `URBAN`, `Urban`) | Applied `=PROPER()` via temporary column |
| 21 | Extra whitespace in Route_Name | Applied `=TRIM()` via temporary column |
| 22 | Missing Zone value for Chittagong Hill Tracts | Imputed as `"Rural"` based on route name and the high average route distance (45 km), consistent with all other Rural-zone routes in the dataset |
| 23 | Wrong data type in Avg_Route_Distance_KM (`"11 km"` instead of `11`) | Applied Find and Replace to remove `" km"` suffix, then changed column format to Number |

---

## Summary

| Table | Issues Resolved | Key Methods Used |
|---|---|---|
| Shipments | 12 | TEXT/DATE formulas, AVERAGEIF, PROPER, TRIM, ABS, Find & Replace, Remove Duplicates, Nested IF |
| Carriers | 6 | RIGHT/concatenation formula, PROPER, TRIM, Remove Duplicates, manual imputation |
| Routes | 5 | PROPER, TRIM, Find & Replace, logical imputation |

**All three tables are now clean and ready for Data Modeling (Phase 3).**
