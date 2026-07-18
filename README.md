# Earthquake Data Engineering Pipeline (Azure Databricks & Power BI)

An end-to-end data engineering pipeline that ingests daily global earthquake data from the USGS API, processes it using a structured Medallion Architecture in Azure Databricks, and exposes the final dataset to Power BI for visualization and monitoring.

---

## 🚀 Architecture Overview

The data pipeline utilizes the **Medallion Architecture** (Bronze ➡️ Silver ➡️ Gold) within an Azure Data Lake Storage (ADLS Gen2) ecosystem, orchestrated using Azure Data Factory (ADF):

```
[ USGS API ] 
     │
     ▼ (ADF Orchestration / Requests)
┌──────────┐      ┌──────────┐      ┌──────────┐
│  Bronze  │ ───> │  Silver  │ ───> │   Gold   │ ───> [ Power BI Dashboard ]
│  (JSON)  │      │ (Parquet)│      │ (Parquet)│
└──────────┘      └──────────┘      └──────────┘
```
<img src="Docs/bsg%20block%20diagram.png" alt="Architecture Diagram" width="900"/>

1. **Bronze Layer**: Raw GeoJSON data requested dynamically from the USGS earthquake API for specific date ranges and saved directly to the data lake as raw `.json` objects.
2. **Silver Layer**: The nested JSON properties are parsed and flattened into schemas, Unix epoch timestamps are converted to native Spark timestamps, and standard null-handling protocols are applied. Output is saved as efficient `.parquet` files.
3. **Gold Layer**: Enhanced business logic layer where reverse geocoding (`reverse_geocoder`) maps geographical coordinates to actual country codes and a customizable earthquake significance classifier is calculated.

---

## 📂 Notebooks Structure

The repository contains three primary PySpark notebooks designed to handle incremental processing via parameters configured in Azure Data Factory widgets:

### 1. Bronze Notebook
* **Purpose:** External API Ingestion.
* **Key Functions:** 
  * Accepts dynamic `start_date` and `end_date` pipeline parameters.
  * Fetches live feature records from the [USGS Earthquake API](https://earthquake.usgs.gov/).
  * Persists records directly into the ADLS Gen2 Bronze container as raw JSON.

### 2. Silver Notebook
* **Purpose:** Schema Enforcement, Data Type Corrections, and Cleaning.
* **Key Functions:**
  * Extracts nested structural data elements (`geometry.coordinates`, `properties.mag`, `properties.time`, etc.).
  * Implements explicit null-handling structures (e.g., fallback defaults like `Unknown Location`).
  * Converts epoch millisecond properties into standardized Spark `TimestampType` fields.
  * Appends structured records to the Silver parquet container.

### 3. Gold Notebook
* **Purpose:** Data Enrichment and Analytics Preparation.
* **Key Functions:**
  * Uses a Python UDF paired with the `reverse_geocoder` library to derive `country_code` flags from the underlying latitude/longitude coordinates.
  * Injects custom business categorizations based on event significance score (`sig` metric divided into *Low*, *Moderate*, and *High* classifications).
  * Generates clean, ready-for-BI tables inside the Gold directory.

---

## 🛠️ Prerequisites & Tech Stack

* **Cloud Infrastructure:** Azure Databricks, Azure Data Lake Storage Gen2 (ADLS)
* **Orchestration:** Azure Data Factory (or Databricks Workflows)
* **Languages & Core Frameworks:** Python, PySpark, Spark SQL
* **Reporting Layer:** Power BI Desktop / Service
* **External Dependencies:** `reverse_geocoder` 

> ⚠️ **Note:** To run the Gold transformation notebook successfully, the `reverse_geocoder` package must be pre-installed on your active Databricks Cluster or configured via initialization scripts.

---

## ⚙️ Deployment & Parameterization

The notebooks utilize `dbutils.widgets` to dynamically receive execution metadata directly from your orchestration pipeline (e.g., Azure Data Factory). 

Ensure you pass the parameters into your Databricks activity block using valid stringified JSON configurations:

```json
// Example parameter string for bronze_params
{
  "start_date": "2026-07-06",
  "end_date": "2026-07-07",
  "bronze_adls": "abfss://bronze@yourdatalake.dfs.core.windows.net/",
  "silver_adls": "abfss://silver@yourdatalake.dfs.core.windows.net/",
  "gold_adls": "abfss://gold@yourdatalake.dfs.core.windows.net/"
}
```

---

## 📊 Business Intelligence (Power BI)

The processed files residing in the **Gold Container** (`earthquake_events_gold/`) are consumed downstream via Power BI. Because the files are stored as enriched parquet tables with predefined taxonomy classifications (`sig_class`, `country_code`), they can easily be hooked up to native mapping components, line charts tracking temporal earthquake frequency trends, and volume KPIs.
