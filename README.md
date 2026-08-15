# Sales Analytics — Medallion Lakehouse on Microsoft Fabric

An end-to-end data engineering solution on Microsoft Fabric: an orchestrated pipeline ingests raw sales data, refines it through a three-layer Medallion architecture (Bronze → Silver → Gold), lands a star schema in a Fabric Data Warehouse, and serves an executive sales dashboard over DirectLake.

![Sales Analytics Overview](screenshots/report-sales-overview.png)

---

## What This Project Demonstrates

Most portfolio dashboards start from a clean CSV. This one starts from raw, unvalidated order data and builds the plumbing that makes a dashboard trustworthy — layered refinement, dimensional modelling, and a parameterised pipeline that can be re-run against a different source file without touching a single notebook.

The reporting layer is the last 20% of the work. The other 80% is below.

---

## Architecture

```
sales_data.csv (parameterised source)
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│  Pipeline: Pl_ingest_sales                              │
│                                                         │
│  Copy data  →  Notebook  →  Notebook  →  Notebook  →  Copy data
│  CopyRaw       Copydata     Transform     SilverTo      transformsilver
│  SalesCSV      todelta      BronzeTo      Gold          towarehouse
│                             Silver                      │
└─────────────────────────────────────────────────────────┘
        │                                          │
        ▼                                          ▼
  Lakehouse (lh_sales)                    Warehouse (wh_sales)
  ├── bronze_sales                        └── analytics schema
  ├── silver_sales                            ├── fact_sales
  ├── gold_sales_summary                      ├── dim_customer
  ├── fact_sales                              ├── dim_date
  ├── dim_customer                            ├── dim_product
  ├── dim_date                                └── dim_region
  ├── dim_product                                 + Views, Functions
  └── dim_region
                                                   │
                                              DirectLake
                                                   │
                                                   ▼
                                    Power BI — Sales Analytics Overview
```

---

## The Medallion Layers

| Layer | Table | Purpose |
|---|---|---|
| **Bronze** | `bronze_sales` | Raw landing zone. Source CSV written to Delta as-is — no cleansing, no type coercion. Preserves an auditable copy of exactly what arrived. |
| **Silver** | `silver_sales` | Cleansed and conformed. Type enforcement, date parsing, handling of zero-revenue and null rows, standardised region and category values. |
| **Gold** | `gold_sales_summary`, `fact_sales`, `dim_*` | Business-ready. Dimensional model built out into one fact table and four conformed dimensions, plus a pre-aggregated summary table for fast report reads. |

Keeping Bronze immutable is the point of the pattern: when a transformation rule changes, Silver and Gold get rebuilt from Bronze rather than requiring a fresh extract from the source system.

---

## Pipeline Orchestration

![Pipeline](screenshots/pipeline-orchestration.png)

`Pl_ingest_sales` chains five activities with success-dependency links, so a failure at any stage halts the run rather than propagating bad data downstream:

1. **CopyRawSalesCSV** *(Copy data)* — pulls the source file into the lakehouse Files area
2. **Copydatatodelta** *(Notebook)* — writes raw CSV into the `bronze_sales` Delta table
3. **TransformBronzeToSilver** *(Notebook)* — PySpark cleansing and conforming
4. **SilverToGold** *(Notebook)* — builds the fact and dimension tables
5. **transformsilvertowarehouse** *(Copy data)* — loads the star schema into the `analytics` schema of `wh_sales`

**Parameterised source:** the pipeline exposes a `Sourcefilename` parameter (String, default `sales_data.csv`), so the same pipeline can process a different file — a new month's extract, a backfill, a test dataset — without any code change.

---

## Data Model

![Warehouse star schema](screenshots/warehouse-star-schema.png)

A classic star schema in the `analytics` schema of `wh_sales`:

**Fact:** `fact_sales` — order grain, with revenue and quantity measures

**Dimensions:** `dim_customer`, `dim_date`, `dim_product`, `dim_region`

Star schema was chosen over a flat wide table deliberately: it keeps the fact narrow, makes filter propagation predictable in the semantic model, and lets dimensions be reused if a second fact (returns, targets) is added later.

![Lakehouse tables](screenshots/lakehouse-medallion.png)

---

## Report

The Power BI report reads the Gold layer over **DirectLake**, so no import refresh runs and no data is duplicated — the report queries Delta files in OneLake directly.

### Headline metrics

| KPI | Value |
|---|---|
| Total Revenue | ₹13.6M |
| Total Revenue (USD) | $164.4K |
| Total Orders | 1,268 |
| Total Quantity Sold | 10,043 |
| Avg Revenue per Order | ₹10.8K |

A dual-currency measure exposes revenue in both INR and USD off the same fact table, so local and head-office audiences read the same report without a second model.

### Visuals

- Monthly revenue and orders trend (dual-axis, revenue as columns against an orders line)
- Revenue by region and orders by region
- Revenue by product category and orders by product category
- Top 10 customers by revenue, ranked
- Slicers on date, month, quarter, and year (2023–2024)

### What the data says

**Revenue and volume tell opposite stories.** Orders split almost evenly across the three categories — Electronics 36.0%, Furniture 32.3%, Office Supplies 31.6% — but revenue does not: Electronics ₹9.0M against Office Supplies ₹0.8M. Office Supplies drives a third of order volume for 6% of revenue.

**Quantity confirms it.** Office Supplies accounts for 70.0% of units sold (7,028 of 10,043) — high-frequency, low-value consumables. The operational cost of servicing those orders is worth examining against their margin contribution.

**Regional demand is flat; regional value isn't.** Order counts sit within 18 of each other across all four regions (North 328, East 315, South 315, West 310), so the revenue spread between regions reflects basket composition rather than market penetration.

**Revenue is not concentrated.** The top 10 customers contribute ₹4.69M of ₹13.6M — roughly a third. Healthy diversification, but the top accounts still warrant retention attention.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Platform | Microsoft Fabric, OneLake |
| Orchestration | Fabric Data Pipelines (parameterised, dependency-chained) |
| Storage | Lakehouse (`lh_sales`), Delta Lake, Medallion architecture |
| Transformation | PySpark, Spark notebooks |
| Warehouse | Fabric Data Warehouse (`wh_sales`), T-SQL, star schema |
| Semantic layer | DirectLake semantic model, DAX |
| Reporting | Power BI |

---

## Repository Structure

```
├── README.md
├── notebooks/
│   ├── copy_data_to_delta.ipynb       # CSV → bronze_sales
│   ├── transform_bronze_to_silver.ipynb
│   └── silver_to_gold.ipynb           # fact + dimension build
├── warehouse/
│   └── schema.sql                     # analytics schema DDL
├── semantic-model/
│   └── measures.dax
└── screenshots/
    ├── report-sales-overview.png
    ├── pipeline-orchestration.png
    ├── lakehouse-medallion.png
    └── warehouse-star-schema.png
```

---

## Reproducing This Project

1. Create a Fabric workspace with capacity or trial enabled.
2. Create a Lakehouse (`lh_sales`) and a Warehouse (`wh_sales`).
3. Upload your sales CSV to the lakehouse Files area.
4. Import the notebooks from `notebooks/` and repoint them at your lakehouse.
5. Build a pipeline chaining the five activities described above, adding a `Sourcefilename` string parameter.
6. Run the pipeline and confirm bronze, silver, and gold tables populate in order.
7. Create a DirectLake semantic model over the Gold tables, add the measures from `semantic-model/`, and build the report.

---

## What I Took Away From This

The Medallion pattern only earns its complexity once something goes wrong. Partway through, a transformation rule for zero-revenue rows turned out to be wrong — and because Bronze held an untouched copy, fixing it meant re-running two notebooks rather than re-extracting the source. That's the argument for the layer, and it's not obvious until you need it.

The other lesson was about the Lakehouse-to-Warehouse split. Both can hold a star schema, and it's tempting to pick one. Keeping the modelling in the lakehouse and pushing a copy to the warehouse gives Spark-based transformation and T-SQL querying over the same model — worth the extra pipeline activity when downstream consumers are SQL-native.
