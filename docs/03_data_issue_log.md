# Data Quality Issue Log
### Shohoz Express Ltd. — Pre-Cleaning Audit

This log documents every data quality issue identified through manual inspection of the three raw datasets (`Shipments_raw.xlsx`, `Carriers_raw.xlsx`, `Routes_raw.xlsx`) **before** any cleaning was performed. Each issue is logged with its location, scope, and the cleaning approach planned for Phase 2.

---

## Table: Shipments (5,100 rows, 11 columns)

| # | Issue | Affected Column(s) | Scope | Planned Fix |
|---|---|---|---|---|
| 1 | Inconsistent date formats — 5 different formats mixed in the same column (`DD/MM/YYYY`, `MM-DD-YYYY`, `YYYY.MM.DD`, `DD-Mon-YYYY`, `YYYY/MM/DD`) | Order_Date, Promised_Delivery_Date, Actual_Delivery_Date | ~5,100 rows, roughly evenly split across all 5 formats | Use Power Query's "Date.FromText" with conditional format detection, or split-and-rebuild logic to standardize to a single ISO format (YYYY-MM-DD) |
| 2 | Missing values | Actual_Delivery_Date (525 blank), Distance_KM (205 blank), Weight_KG (203 blank) | ~4% of rows per column | Investigate cause per column: Actual_Delivery_Date blanks where Status = "In Transit" are *expected* (not yet delivered); true missing values in Distance_KM/Weight_KG will be flagged or imputed using route-average distance |
| 3 | Missing values (logical vs. true) | Delay_Reason (3,180 blank/NaN) | ~62% of rows | Most blanks correspond to on-time deliveries (no delay occurred) — these are not "missing," they should be explicitly labeled "None" rather than blank. A smaller subset may represent true missing data and will be reviewed separately |
| 4 | Duplicate rows | Entire row (all 11 columns identical) | 100 exact duplicate rows (Shipment_ID also duplicated) | Remove duplicates using Power Query's "Remove Duplicates" on Shipment_ID after confirming full-row match |
| 5 | Inconsistent text casing | Status column: "Delivered" / "DELIVERED" / "delivered" (same for In Transit, Returned) | All 5,100 rows affected by casing variation | Standardize using Power Query's "Text.Proper" or fixed capitalization transformation |
| 6 | Extra whitespace (leading/trailing/both) | Delay_Reason | ~5% of non-blank Delay_Reason values | Trim using Power Query's "Text.Trim" |
| 7 | Typos in categorical values | Delay_Reason: "Trafic" (Traffic), "Wether"/"Weathr" (Weather), "Warehose Delay"/"Warehouse Dealy" (Warehouse Delay), "Vehical Breakdown"/"Vehicle Breakdwn" (Vehicle Breakdown), "Wrong Adress"/"Wrong Addres" (Wrong Address) | ~25% of non-blank Delay_Reason values | Standardize via Power Query conditional replace / fuzzy matching against the 5 valid category values |
| 8 | Wrong data type — numeric value stored as text with unit suffix | Distance_KM: some values stored as "18 km" instead of 18 | ~15% of non-blank Distance_KM values | Extract numeric portion using Power Query "Text.BeforeDelimiter" / remove non-numeric characters, then convert column type to Decimal Number |
| 9 | Outliers / logically impossible values | Shipment_Cost: 21 rows with negative cost (impossible) | 21 rows (~0.4%) | Flag and correct — likely a data entry sign error; convert to absolute value and verify against expected cost range per route/carrier |
| 10 | Inconsistent ID formats | Carrier_ID: three different formats used for the same set of carriers — "C-0X" (1,734 rows), "CX" (1,691 rows), and plain "X" (1,675 rows) | All 5,100 rows | Standardize all Carrier_ID values to a single format ("C-0X") using Power Query text transformation before joining to the Carriers table |
| 11 | Mixed currency/number notation | Shipment_Cost: plain numbers, "BDT 1,500" style prefixed values, and comma-separated values mixed in the same column | Mixed across all rows | Strip "BDT" prefix and thousands separators, convert to a clean Decimal Number type |

**Status value distribution (casing issue, Issue #5):**

| Value | Count |
|---|---|
| Delivered | 2,258 |
| DELIVERED | 1,131 |
| delivered | 1,129 |
| In Transit | 182 |
| in transit | 91 |
| IN TRANSIT | 73 |
| Returned | 124 |
| RETURNED | 70 |
| returned | 42 |

**Order_Date format distribution (Issue #1):**

| Format | Count |
|---|---|
| MM-DD-YYYY | 1,064 |
| YYYY/MM/DD | 1,030 |
| YYYY.MM.DD | 1,021 |
| DD-Mon-YYYY | 1,002 |
| DD/MM/YYYY | 983 |

**Carrier_ID format distribution (Issue #10):**

| Format | Count |
|---|---|
| C-0X (e.g., C-01) | 1,734 |
| CX (e.g., C1) | 1,691 |
| X (e.g., 1) | 1,675 |

---

## Table: Carriers (7 rows, 5 columns)

| # | Issue | Affected Column(s) | Scope | Planned Fix |
|---|---|---|---|---|
| 12 | Duplicate row | All columns (Carrier_ID "C1" duplicates "C-01" — same carrier, "Sundarban Logistics," listed twice under two different ID formats) | 1 duplicate carrier | Standardize Carrier_ID format first, then remove the resulting duplicate |
| 13 | Inconsistent ID formats | Carrier_ID: mix of "C-0X", "CX", and plain "X" formats | 7 rows | Standardize to single format, consistent with the Shipments table fix |
| 14 | Inconsistent text casing | Vehicle_Type: "Truck" / "TRUCK" / "truck" / "Van" / "VAN" / "Bike" | 7 rows | Standardize using Text.Proper |
| 15 | Extra whitespace | Carrier_Name | Present in multiple rows | Trim using Text.Trim |
| 16 | Missing value | Base_Rate_per_KM (Karatoa Express has a blank rate) | 1 row | Investigate and impute (likely using the average rate for the same Vehicle_Type) |
| 17 | Inconsistent date formats | Contract_Start_Date: same 5-format mix as Shipments dates | 7 rows | Standardize to ISO format, consistent with the Shipments table fix |

---

## Table: Routes (10 rows, 5 columns)

| # | Issue | Affected Column(s) | Scope | Planned Fix |
|---|---|---|---|---|
| 18 | Inconsistent text casing | Region_Name: "Dhaka" / "dhaka"; Zone: "Urban" / "URBAN" / "urban" | 10 rows | Standardize using Text.Proper |
| 19 | Extra whitespace | Route_Name | Present in multiple rows | Trim using Text.Trim |
| 20 | Missing value | Zone (Chittagong Hill Tracts has a blank zone) | 1 row | Investigate and impute (likely "Rural," based on route name/distance) |
| 21 | Wrong data type — numeric value stored as text with unit suffix | Avg_Route_Distance_KM: some values stored as "11 km" / "30 km" instead of plain numbers | 2 rows | Extract numeric portion, convert column type to Decimal Number, consistent with the Shipments table fix |

---

## Summary

| Table | Total Issues Logged | Rows Affected (approx.) |
|---|---|---|
| Shipments | 11 | All 5,100 rows affected by at least one issue |
| Carriers | 6 | All 7 rows affected by at least one issue |
| Routes | 4 | All 10 rows affected by at least one issue |

**Next step (Phase 2, Day 4 onward):** Begin systematic cleaning in Power Query, table by table, starting with Shipments. Each fix will be documented in `03_data_cleaning_log.md` with before/after examples and the specific Power Query steps used.
