# Microsoft Fabric Medallion Pipeline (Demo)

This repository contains a **demo project** that implements a **data pipeline on Microsoft Fabric**, following the **Medallion Architecture** (Bronze → Silver → Gold).  
It demonstrates how to ingest, transform, and analyze structured and semi-structured data using Fabric components:

- **Data Factory** for ingestion & orchestration  
- **Lakehouse** for Bronze/Silver data layers  
- **Warehouse** for Gold / star schema modeling  
- **Power BI** for visualization and KPI dashboards  

---

## 🚀 Architecture Overview

```mermaid
flowchart TD
    A["Sources (CSV, GeoJSON, Excel, SQL extracts)"] --> B["Bronze (Lakehouse - Raw)"]
    B --> C["Silver (Lakehouse - Conformed)"]
    C --> D["Gold (Warehouse - Star Schema & KPIs)"]
    D --> E["Power BI (Semantic Model & Dashboards)"]
````

* **Bronze** → raw ingestion (CSV, JSON, GeoJSON, SQL extracts, Excel).
* **Silver** → cleaned, standardized, conformed data (dimensions & facts ready for modeling).
* **Gold** → star schema with KPI views for analytics and dashboards.
* **BI Layer** → Power BI reports consuming the Warehouse directly (Direct Lake/DirectQuery).

---

## 📂 Project Structure

```
fabric-medallion-pipeline/
│
├── data/                  # Example raw sources
│   ├── farms.geojson
│   ├── protected_areas.geojson
│   ├── lots.csv
│   └── certifications.csv
│
├── notebooks/             # PySpark Notebooks for Silver transformations
│   └── bronze_to_silver.ipynb
│
├── sql/                   # SQL scripts for Gold layer
│   ├── create_dim_tables.sql
│   ├── create_fact_tables.sql
│   └── create_kpi_views.sql
│
├── diagrams/              # Architecture diagrams (Mermaid/PNG)
│   └── pipeline_mermaid.md
│
└── README.md
```

---

## 📊 Example Data (Synthetic)

### farms.geojson

```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "geometry": { "type": "Polygon", "coordinates": [[[ -5.8, 6.5 ], [ -5.75, 6.5 ], [ -5.75, 6.55 ], [ -5.8, 6.55 ], [ -5.8, 6.5 ]]] },
      "properties": { "farm_id": "FARM_001", "coop_id": "COOP_A", "country": "CI", "region": "Sassandra-Marahoué" }
    }
  ]
}
```

### lots.csv

```csv
lot_id,farm_id,date_collecte,date_embarquement,volume,destination
LOT_0001,FARM_001,2025-06-10,2025-06-20,800,EU
LOT_0002,FARM_002,2025-06-12,2025-06-25,1200,EU
```

### certifications.csv

```csv
certif_id,farm_id,type,start_date,end_date
CERT_001,FARM_001,RA,2024-01-01,2026-12-31
CERT_002,FARM_002,FT,2023-01-01,2025-12-31
```

---

## 🔧 Transformations (Notebook PySpark)

Example PySpark code for **Bronze → Silver**:

```python
from pyspark.sql import functions as F

# Bronze paths (Lakehouse Files)
farms_p  = "Files/bronze/farms_geojson/"
lots_p   = "Files/bronze/lots/"
cert_p   = "Files/bronze/certifications/"

# Farms: flatten GeoJSON
df_farms_raw = spark.read.json(farms_p)
df_farms = (df_farms_raw
  .select(F.explode("features").alias("f"))
  .select(F.col("f.properties.*"), F.col("f.geometry").alias("geom_json"))
  .withColumn("centroid_lon", F.expr("aggregate(geom_json.coordinates[0], 0D, (acc, p) -> acc + p[0]) / size(geom_json.coordinates[0])"))
  .withColumn("centroid_lat", F.expr("aggregate(geom_json.coordinates[0], 0D, (acc, p) -> acc + p[1]) / size(geom_json.coordinates[0])"))
)
df_farms.write.mode("overwrite").format("delta").saveAsTable("farms_silver")

# Lots
df_lots = (spark.read.option("header",True).csv(lots_p)
  .withColumn("date_collecte", F.to_date("date_collecte"))
  .withColumn("date_embarquement", F.to_date("date_embarquement"))
  .withColumn("volume", F.col("volume").cast("double"))
)
df_lots.write.mode("overwrite").format("delta").saveAsTable("lots_silver")

# Certifications
df_cert = spark.read.option("header",True).csv(cert_p)
df_cert.write.mode("overwrite").format("delta").saveAsTable("certifications_silver")
```

---

## 🗄️ Gold Layer (SQL Warehouse)

Example SQL scripts for **Gold (Warehouse)**:

```sql
-- Dimensions
CREATE OR ALTER VIEW dim_farm AS
  SELECT farm_id, coop_id, country, region, centroid_lon, centroid_lat
  FROM farms_silver;

CREATE OR ALTER VIEW dim_certif AS
  SELECT certif_id, farm_id, type, start_date, end_date
  FROM certifications_silver;

-- Facts
CREATE OR ALTER VIEW fact_lot AS
  SELECT lot_id, farm_id, date_collecte, date_embarquement, volume,
         CASE WHEN destination='EU' THEN 1 ELSE 0 END AS destination_eu
  FROM lots_silver;

-- KPI Views
CREATE OR ALTER VIEW vw_kpi_traceability AS
  SELECT 100.0*SUM(CASE WHEN df.centroid_lon IS NOT NULL THEN 1 ELSE 0 END)/COUNT(*) AS pct_traceable_lots
  FROM fact_lot fl JOIN dim_farm df ON fl.farm_id=df.farm_id;
```

---

## ✅ Example KPIs

* **% Farms Geocoded** → farms with valid centroid/lat-lon.
* **% Lots Traceable** → lots linked to geocoded farms.
* **Certified Lots Ratio** → proportion of certified lots.
* **Overlap with Protected Areas** → farms intersecting conservation polygons.

---

## 🛡️ Governance & Monitoring

* **Security**: Entra ID (RBAC roles), row-level security for sensitive data.
* **Governance**: Purview (lineage, sensitivity labels, catalog).
* **Data Quality**: Great Expectations + Fabric Data Quality Rules.
* **CI/CD**: Git integration + Fabric Deployment Pipelines (Dev → Test → Prod).
* **Monitoring**: Capacity Metrics App + Audit Logs (optional Log Analytics export).

---

## ⚙️ Tech Stack

* **Microsoft Fabric** (Lakehouse, Warehouse, Data Factory, Notebooks)
* **PySpark** for transformations
* **T-SQL** for Warehouse modeling
* **Power BI** for dashboards
* **Delta Lake** (OneLake storage format)

---

## ⚠️ Disclaimer

This repository uses **synthetic demo data**.
It is intended for **educational and demonstration purposes only**.



