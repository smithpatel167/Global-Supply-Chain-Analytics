# Global Supply Chain Analytics

An end-to-end supply chain analytics project built on a live **DirectQuery** connection to a **Microsoft Fabric Warehouse** - procurement, warehousing, logistics, and inventory data, modeled as a proper star schema and visualized across a 5-page Power BI report.

## 🔗 Live Interactive Demo
[View Interactive Demo on Power BI Service](https://app.powerbi.com/view?r=eyJrIjoiNTllODIyNTMtNWU5NS00YTA4LThlNDQtOGJlNjU0MWRjNmYwIiwidCI6ImMxNjQxMjEyLWJkODItNDg1Ny1hMTE1LWQxYjdiY2I4NWFjNSJ9)

---

## **[Download Project Files from Google Drive](https://drive.google.com/drive/folders/1uuoNchqSnH4vpvh96aLFGpmcNY4TPFml?usp=sharing)**

## Why this project

Most portfolio Power BI dashboards import a static CSV once and call it done. This one doesn't: every visual queries a live Fabric Warehouse SQL endpoint in real time via DirectQuery - there's no cached snapshot, no scheduled refresh, the report reflects whatever's in the warehouse at the moment you open it. The `.pbix` file itself is under 700KB despite sitting on top of **1.25M+ rows** of fact data, because DirectQuery models don't embed the data - they embed the query.

Beyond the technical setup, the underlying dataset was engineered (not just randomly generated) to carry a real analytical story: a 2022 global logistics crisis and a 2024 Red Sea/Suez shipping crisis are baked into the delay patterns, so on-time-rate trends actually dip where you'd expect real-world disruptions to show up - something to point at and explain in an interview, not just a chart.

## Architecture

```
Python (pandas/numpy) synthetic data generator
        │
        ▼
CSV exports (6 dimension tables + 3 fact tables, 1.25M+ fact rows)
        │
        ▼
Microsoft Fabric Warehouse (T-SQL, star schema, "supplychain" schema)
        │  SQL analytics endpoint
        ▼
Power BI Desktop - DirectQuery connection (no Import, no cached data)
        │
        ▼
Power BI Service - published report + dashboard, live against the warehouse
```

## Dataset

Synthetic but deliberately realistic - generated with engineered seasonality, supplier-risk correlation, and two disruption windows rather than pure random noise.

| Table | Grain | Rows |
|---|---|---|
| `Dim_Date` | one row per day, 2022–2026 | 1,642 |
| `Dim_Supplier` | one row per supplier | 150 |
| `Dim_Product` | one row per SKU | 600 |
| `Dim_Warehouse` | one row per distribution center | 30 |
| `Dim_Carrier` | one row per logistics carrier | 40 |
| `Dim_DistributionCenter` | one row per downstream DC/customer | 120 |
| `Fact_PurchaseOrders` | one row per PO line (procurement stage) | 350,000 |
| `Fact_Shipments` | one row per shipment line (outbound logistics stage) | 550,000 |
| `Fact_InventorySnapshot` | one row per warehouse/product/week | 351,000 |

Engineered patterns worth knowing about if you're exploring the data:
- **2022 global logistics crisis** (Jan–Sep) - elevated delays across all modes.
- **2024 Red Sea/Suez crisis** (Jan–Aug) - Ocean freight out of APAC/MEA specifically hit harder than other modes.
- Supplier `RiskScore` is correlated with actual delay likelihood, not random.
- `Fact_InventorySnapshot` only covers the most recent 78 weeks (roughly Jan 2025–Jun 2026), not the full date range - a deliberate choice to mirror how inventory systems in practice retain shorter rolling windows than order history.

## Data model

Star schema with three fact tables at genuinely different grains, sharing dimensions:

```
                         Dim_Date
                       /     |     \
                      /      |      \
     Fact_PurchaseOrders  Fact_Shipments  Fact_InventorySnapshot
       |        |             |    |             |        |
  Dim_Supplier  |         Dim_Warehouse  Dim_Carrier    Dim_Product
                |              |
           Dim_Product    Dim_DistributionCenter
                |
          Dim_Warehouse
```

`Dim_Date` connects to three different date roles (`OrderDateKey`, `ShipDateKey`, `SnapshotDateKey`) across the three fact tables - Power BI only allows one active relationship per table pair, so this required deliberately managing active/inactive relationships rather than relying on auto-detection.

Measures live in a dedicated `_Measures` table (a small DAX calculated table, `{BLANK()}`) so the model stays a clean composite of DirectQuery fact/dimension tables plus one lightweight measure container - kept separate from the physical tables for organization.

## Report pages

**Executive Overview**
5-card KPI row (Total PO Spend, Total Landed Cost, PO On-Time Rate, Shipment On-Time Rate, Stockout Rate) · monthly PO-vs-Shipment on-time trend line · total PO spend by category · warehouse map (ArcGIS Maps for Power BI, colored by shipment on-time rate) · Year and Warehouse Region slicers.

**Supplier Performance**
Supplier performance detail table (risk score, on-time rate, quality reject rate, spend) with conditional formatting · average delay days by supplier country · PO spend by supplier tier (donut).

**Logistics & Carriers**
On-time rate by transport mode · monthly on-time rate trend split by mode · carrier performance detail table · freight cost per kg by mode.

**Inventory & Warehouse**
Average days of supply over time · stockout rate by warehouse · warehouse × category inventory matrix · 3-card inventory KPI row · Category and Warehouse Region slicers.

**Cost & Fulfillment**
Total landed cost by region (stacked column: PO spend + freight cost) · cost breakdown by product category · YoY PO spend % quarterly trend · On-Time In-Full (OTIF) rate.

All 5 pages share a consistent title/subtitle banner, custom page icons, and bookmark-driven navigation buttons to jump between pages.

## A few DAX measures worth highlighting

```dax
PO On-Time Rate =
DIVIDE (
    CALCULATE ( COUNTROWS ( Fact_PurchaseOrders ), Fact_PurchaseOrders[DelayDays] <= 0 ),
    CALCULATE ( COUNTROWS ( Fact_PurchaseOrders ), NOT ISBLANK ( Fact_PurchaseOrders[DelayDays] ) )
)
```

```dax
YoY PO Spend % =
VAR CurrentSpend = [Total PO Spend]
VAR PriorSpend =
    CALCULATE ( [Total PO Spend], SAMEPERIODLASTYEAR ( Dim_Date[Date] ) )
RETURN
    DIVIDE ( CurrentSpend - PriorSpend, PriorSpend )
```

```dax
On-Time In-Full (OTIF) Rate =
-- Blended proxy: PO fill rate × shipment on-time rate.
-- A stricter OTIF would need both stages joined at the order-line grain.
[PO On-Time Rate] * [Fill Rate (PO)]
```

Full measure library (25+ measures across procurement, logistics, inventory, and executive/cross-process categories) is in [`docs/02_Data_Model_and_DAX.md`](./docs/02_Data_Model_and_DAX.md).

## Tech stack

- **Microsoft Fabric** - Warehouse (SQL analytics endpoint), OneLake
- **T-SQL** - schema DDL, `COPY INTO` bulk loading
- **Power BI Desktop / Service** - DirectQuery semantic model, DAX, report design
- **Python** (pandas, numpy) - synthetic dataset generation with engineered realism
- **DAX** - time intelligence, multi-fact-table relationship management, composite model

## Reproducing this project

Full step-by-step guides are in [`/docs`](./docs):

1. [`01_Fabric_Setup_Guide.md`](./docs/01_Fabric_Setup_Guide.md) - create the Fabric workspace/Warehouse, load the data, connect Power BI Desktop via DirectQuery.
2. [`02_Data_Model_and_DAX.md`](./docs/02_Data_Model_and_DAX.md) - table renaming, relationship setup, full DAX measure library.
3. [`03_Report_and_Dashboard_Design.md`](./docs/03_Report_and_Dashboard_Design.md) - field-by-field build instructions for every visual on all 5 pages, plus the Service dashboard.
4. [`04_Troubleshooting_and_Publish_Checklist.md`](./docs/04_Troubleshooting_and_Publish_Checklist.md) - common DirectQuery/model issues and a pre-publish polish checklist.

Data and schema files:
- [`/data`](./data) - the 9 CSVs (dimension + fact tables)
- [`/sql`](./sql) - `01_create_tables.sql` (DDL) and `02_load_data_options.sql` (loading options + troubleshooting)
- [`/theme`](./theme) - custom Power BI report theme (`SupplyChain_Slate_Amber_Theme.json`)

You'll need your own Microsoft Fabric capacity (a trial capacity works) - this isn't a "just open the .pbix" project by design, since the whole point is a live DirectQuery connection rather than an imported snapshot.

## Known limitations

- Data is synthetic, generated for this project - not real company data. Realism (seasonality, disruption windows, risk correlation) was deliberately engineered, but absolute figures don't represent any actual company's supply chain.
- OTIF is a blended proxy measure (`PO On-Time Rate × Fill Rate`), not a strict order-line-level OTIF calculation - noted directly in the DAX comment.
- Requires a Fabric capacity (trial or paid) to actually run live - there's no offline/Import fallback mode.

## License / attribution

Dataset and all DAX/report design work in this repo are original, built for portfolio purposes. Feel free to reuse the schema design or DAX patterns - attribution appreciated but not required.

## 👤 About

Built by **Smith Patel**

[LinkedIn](https://www.linkedin.com/in/smithpatel167/) • [GitHub](https://github.com/smithpatel167)
