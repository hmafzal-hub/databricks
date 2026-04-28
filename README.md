# 🚀 Retail Lakehouse Data Pipeline on Databricks

## 📌 Overview

This project demonstrates a **production-grade Retail Lakehouse architecture** built using **Databricks, Delta Lake, and Medallion Architecture (Bronze → Silver → Gold)**.

It simulates a real-world **ERP (NetSuite-style) data platform**, implementing:

* Incremental data ingestion
* Data quality & validation framework
* Slowly Changing Dimensions (SCD Type 2)
* Fact table modeling
* Audit & reject logging
* Dashboard-ready analytics layer

---

## 🧠 Business Problem

A retail organization needs:

* Reliable and scalable analytics platform
* Historical tracking of customer and product changes
* Incremental data processing (no full reloads)
* Data quality monitoring
* Dashboard-ready datasets

---

## 🏗️ Architecture

```
Parquet Files (ERP Simulation)
        ↓
Bronze Layer (Raw - Append Only)
        ↓
Silver Layer (Cleaned & Deduplicated - MERGE)
        ↓
Gold Layer
   ├── Dimensions (SCD Type 2)
   └── Fact Table (Incremental)
        ↓
Power BI / Databricks SQL Dashboards
```

---

## 🧱 Tech Stack

| Category      | Tools Used               |
| ------------- | ------------------------ |
| Data Platform | Databricks               |
| Storage       | Delta Lake               |
| Processing    | PySpark                  |
| Orchestration | Databricks Workflows     |
| Governance    | Unity Catalog            |
| Visualization | Power BI, Databricks SQL |
| Language      | Python, SQL              |

---

## 📂 Data Sources

* Simulated ERP data (NetSuite-style schema)
* Generated via Python and stored as **Parquet files**
* Stored in:

  ```
  /Volumes/retail/default/raw_data/
  ```

---

## 🥉 Bronze Layer (Raw Ingestion)

### Features:

* Append-only ingestion
* All columns stored as STRING
* File-level incremental loading
* Metadata tracking

### Tables:

* ns_brands
* ns_customers
* ns_items
* ns_sales_orders
* ns_sales_order_lines
* ns_shipments

### Metadata Columns:

* etl_run_id
* etl_loaded_at
* ingestion_date
* source_system
* record_hash

---

## 🥈 Silver Layer (Clean & Trusted)

### Transformations:

* Data type casting (safe casting)
* Deduplication using record_hash
* Data validation & reject handling
* Incremental MERGE logic

### Features:

* Only processes new Bronze runs
* Rejects invalid records (stored in audit table)
* Maintains clean, reliable datasets

---

## 🥇 Gold Layer (Business Layer)

### 🔹 Dimensions (SCD Type 2)

* dim_brand
* dim_customer
* dim_item
* dim_date

### SCD2 Columns:

* surrogate_key
* effective_start_date
* effective_end_date
* is_current
* record_hash

### 🔹 Fact Table

* fact_sales (order-line grain)

### Fact Features:

* Incremental MERGE
* Foreign key validation
* Joins with dimensions
* Revenue metrics

---

## 🔄 Incremental Processing

| Layer  | Strategy                |
| ------ | ----------------------- |
| Bronze | Append new files only   |
| Silver | MERGE using record_hash |
| Gold   | SCD Type 2 + MERGE      |

---

## 🧪 Data Quality Framework

### Implemented Checks:

* Null checks
* Duplicate checks
* Data type validation
* Revenue reconciliation
* Schema validation

---

## 📊 Audit & Monitoring

### Tables:

#### 1. ETL Run Log

`retail.audit.etl_run_log`

Tracks:

* start_time / end_time
* status
* rows_read / rows_written
* duration
* errors

---

#### 2. Reject Log

`retail.audit.etl_reject_log`

Captures:

* invalid records
* reason & column
* raw payload
* error messages

---

#### 3. File Tracking

`retail.audit.ingested_files`

Ensures:

* Idempotent ingestion
* No duplicate file processing

---

## 📈 Key Business Metrics

* Total Revenue
* Average Order Value
* Top Customers (LTV)
* Product Performance
* Monthly Sales Trends
* Shipment Performance

---

## ⚙️ Pipeline Orchestration

* Built using **Databricks Workflows**
* Layer dependencies:

  * Bronze → Silver → Gold → Audit
* Failure handling & logging included

---

## 🔐 Governance

* Unity Catalog integration
* Table-level access control
* Data lineage tracking
* Secure storage with Volumes

---

## 📊 Dashboard Layer

* Databricks SQL dashboards
* Power BI dashboards
* Business KPIs & insights

---

## 🧪 Validation & Reconciliation

* Source vs Bronze validation
* Bronze vs Silver reconciliation
* Silver vs Gold validation
* Revenue consistency checks

---

## 🚀 Project Highlights (Interview Ready)

* Built **end-to-end Lakehouse pipeline**
* Implemented **SCD Type 2**
* Designed **incremental pipelines (no full reloads)**
* Developed **data quality + audit framework**
* Achieved **high data reliability**
* Designed for **real business reporting use cases**

---

## 📌 Future Enhancements

* Auto Loader (Streaming ingestion)
* Delta optimization (Z-ORDER, partitioning)
* Advanced SCD optimization
* Real-time dashboards
* Data observability tools integration

---

## 🙋 Author

**Muhammad Afzal**
Senior Data Engineer
Expertise: Databricks | BigQuery | ETL | Data Modeling | Analytics

---
