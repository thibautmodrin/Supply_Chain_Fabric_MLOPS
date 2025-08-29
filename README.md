# Fabric Medallion Pipeline (Demo)

This repository contains a **demo data pipeline** built on **Microsoft Fabric**, following the **Medallion Architecture** (Bronze → Silver → Gold).  
It ingests structured (CSV), semi-structured (GeoJSON), and tabular data into **OneLake**, transforms them in a **Lakehouse**, and publishes curated analytics into a **Warehouse** for **Power BI**.

---

## 🚀 Architecture

```mermaid
flowchart TD
    A["Sources (CSV, GeoJSON, Excel, SQL)"] --> B["Bronze (Lakehouse - Raw)"]
    B --> C["Silver (Lakehouse - Conformed)"]
    C --> D["Gold (Warehouse - Star Schema)"]
    D --> E["Power BI (Semantic Model, Dashboards)"]
