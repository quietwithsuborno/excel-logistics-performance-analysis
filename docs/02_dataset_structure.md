# Dataset Structure Design
### Shohoz Express Ltd. — Logistics & Delivery Performance Analysis

This project uses three linked tables, designed to simulate a realistic operational dataset for a logistics/delivery network. The raw version of this dataset is intentionally messy (see `02_data_cleaning_log.md` for details) to demonstrate a complete real-world data cleaning workflow.

---

## Table 1: Shipments (Fact Table)

The core transactional table — one row per shipment.

| Column | Description |
|---|---|
| Shipment_ID | Unique identifier for each shipment |
| Order_Date | Date the order was placed |
| Promised_Delivery_Date | Committed delivery date given to the customer |
| Actual_Delivery_Date | Date the shipment was actually delivered |
| Carrier_ID | Foreign key → Carriers table |
| Route_ID | Foreign key → Routes table |
| Distance_KM | Distance covered for this shipment |
| Shipment_Cost | Total cost of the shipment |
| Weight_KG | Weight of the shipped goods |
| Status | Delivered / Returned / In Transit |
| Delay_Reason | Root cause of delay (Traffic / Weather / Vehicle Breakdown / Wrong Address / Warehouse Delay / None) |

## Table 2: Carriers (Dimension Table)

| Column | Description |
|---|---|
| Carrier_ID | Unique identifier for each carrier |
| Carrier_Name | Name of the third-party carrier |
| Vehicle_Type | Truck / Van / Bike |
| Base_Rate_per_KM | Base shipping rate per kilometer |
| Contract_Start_Date | Date the carrier contract began |

## Table 3: Routes (Dimension Table)

| Column | Description |
|---|---|
| Route_ID | Unique identifier for each route |
| Route_Name | Specific route name (e.g., "Dhaka Central", "Dhaka Suburb", "Chittagong Port Area") |
| Region_Name | Parent region (Dhaka / Chittagong / Sylhet / Khulna / Rajshahi) |
| Zone | Urban / Rural classification |
| Avg_Route_Distance_KM | Average distance for this route |

---

## Relationships

- `Shipments.Carrier_ID` → `Carriers.Carrier_ID` (many-to-one)
- `Shipments.Route_ID` → `Routes.Route_ID` (many-to-one)

This structure supports two levels of geographic analysis — **Region-level** (via `Routes.Region_Name`) and **Route-level** (via `Routes.Route_ID`) — and enables root-cause analysis of delays via the `Delay_Reason` field, in addition to standard carrier and cost analysis.
