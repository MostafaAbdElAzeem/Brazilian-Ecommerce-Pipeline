# 🛒 E-Commerce Data Pipeline
### Olist Brazilian E-Commerce — End-to-End Analytics Project

![Pipeline](https://img.shields.io/badge/Pipeline-Medallion_Architecture-blue)
![MariaDB](https://img.shields.io/badge/Database-MariaDB-4479A1?logo=mysql&logoColor=white)
![Sqoop](https://img.shields.io/badge/Ingestion-Apache_Sqoop-F7A80D)
![Spark](https://img.shields.io/badge/Processing-Apache_Spark-E25A1C?logo=apachespark&logoColor=white)
![Hive](https://img.shields.io/badge/Storage-Apache_Hive-FDEE21)
![HDFS](https://img.shields.io/badge/HDFS-Hadoop-66CCFF)

---

## 📋 Table of Contents
- [Overview](#overview)
- [Tech Stack](#tech-stack)
- [Dataset](#dataset)
- [Project Structure](#project-structure)
- [Pipeline](#pipeline)
  - [1. CSV → MariaDB](#1-csv--mariadb)
  - [2. Sqoop → HDFS (Bronze)](#2-sqoop--hdfs-bronze-layer)
  - [3. Spark Cleaning (Silver)](#3-spark-cleaning-silver-layer)
  - [4. Gold Layer](#4-gold-layer)
- [Business Questions](#business-questions)
- [Key Findings](#key-findings)
- [How to Run](#how-to-run)

---

## Overview

This project builds a full end-to-end data pipeline on the **Olist Brazilian E-Commerce** dataset, following the **Medallion Architecture**:
```
┌──────────────┐     ┌───────────────┐     ┌───────────────┐     ┌───────────────┐     ┌───────────────┐     ┌───────────────┐
│   CSV Files  │────▶│    MariaDB    │────▶│  Apache Sqoop │──▶│  HDFS Bronze  │────▶│ Spark Silver  |───▶│   Hive Gold   │
│  (Olist DB)  │     │  ecommerce DB │     │               │     │  Raw Parquet  │     │  Clean/Parq.  │     │  Aggregated   │
│  5 tables    │     │  (5 tables)   │     │               │     │ /staging_zone │     │  (PySpark)    │     │  (Spark SQL)  │
└──────────────┘     └───────────────┘     └───────────────┘     └───────────────┘     └───────────────┘     └───────────────┘
                                                                                                                      │
                                                                                                                      ▼
                                                                                                           ┌───────────────────┐
                                                                                                           │    Analytics      │
                                                                                                           │  Business Answers │
                                                                                                           │                   │
                                                                                                           └───────────────────┘
```
---

## Tech Stack

| Layer | Tool |
|---|---|
| Source Database | MariaDB |
| Ingestion | Apache Sqoop |
| Distributed Storage | HDFS (Hadoop) |
| Processing & Cleaning | Apache Spark (PySpark) — Jupyter Notebooks |
| Data Warehouse | Apache Hive |
| File Format |  Parquet 
| Query Language | SQL / HiveQL |

---

## Dataset

**Source:** [Olist Brazilian E-Commerce — Kaggle](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)
> 100,000+ orders placed on Olist marketplace between 2016–2018 across multiple Brazilian states.

| Table | Rows | Description |
|---|---|---|
| `customers` | 99,441 | Customer ID, city, state, zip code |
| `products` | 32,951 | Category, dimensions, weight |
| `orders` | 99,441 | Order lifecycle — status and all timestamps |
| `order_items` | 112,650 | Items per order — price, freight, seller |
| `order_reviews` | 99,224 | Review score and comment per order |

**Schema relationships:**

```
customers ◄── orders ◄── order_items ──► products
                  └────► order_reviews
```

---

## Project Structure

```
ecommerce-pipeline/
│
├── dataset/                          # Raw CSV source files
│   ├── olist_customers_dataset.csv
│   ├── olist_products_dataset.csv
│   ├── olist_orders_dataset.csv
│   ├── olist_order_items_dataset.csv
│   └── olist_order_reviews_dataset.csv
│
├── csv_to_mysql/                     # Step 1 — MariaDB schema + loader
│   └── load_csv_to_mariadb.sql
│
├── sqoop/                            # Step 2 — Bronze layer ingestion
│   └── sqoop_import.sh
│
├── silver_layer/                     # Step 3 — PySpark cleaning notebooks
│   └── silver_cleaning.ipynb
│
├── gold_layer/                       # Step 4 — Aggregation SQL scripts
│   └── gold_agg.ipynb
│   
└── README.md
```

---

## Pipeline

### 1. CSV → MariaDB

Raw CSV files are loaded into a MariaDB database (`ecommerce`) using a single SQL script that creates all 5 tables and populates them with `LOAD DATA INFILE`.

```bash
mysql --local-infile=1 -u your_user -p ecommerce < load_csv_to_mariadb.sql
```

> **Note:** `SET foreign_key_checks = 0` is used during load to allow flexible table order. Empty datetime fields are handled via `NULLIF()` to store `NULL` instead of `0000-00-00`.

---

### 2. Sqoop → HDFS (Bronze Layer)

All 5 tables are ingested from MariaDB into HDFS using Apache Sqoop with a **full load** strategy.
Data stored as parquet files.
```bash
./sqoop_import.sh
```

**HDFS output:**
```
hdfs://namenode:8020/staging_zone/
├── customers/      
├── products/       
├── orders/         
├── order_items/    
└── order_reviews/  
```


### 3. Spark Cleaning (Silver Layer)


Cleaning is performed in **`silver_cleaning.ipynb`** using PySpark. Each table has its own section in the notebook.

#### Cleaning Summary

| Operation | Detail |
|---|---|
| ✅ Null handling | Optional datetime nulls kept as `NULL` + boolean flag columns added |
| ✅ Duplicate check | Verified no duplicates on primary keys |
| 🕐 Unix ms → Timestamp |  applied to all datetime columns |
| ✂️ Type standardization | Columns cast to correct types — `INT`, `DECIMAL`, `TIMESTAMP`, `STRING` |
| ➕ Derived columns | `delivery_days`, `is_late`, `has_message` |
| 🗑️ Dropped columns | Unnecessary or redundant raw columns removed after derivation |
| 🔤 String normalization | `trim()`, `lower()` on cities/status, `upper()` on state codes |

#### Silver Output

Tables are written as **external Parquet tables** in Hive .

```python
df_silver.write \
    .mode("overwrite") \
    .format("parquet") \
    .option("path", "hdfs://.../silver/table_name") \
    .saveAsTable("silver.table_name")
```


### 4. Gold Layer

Aggregations are performed in gold_agg.ipynb using Spark SQL, querying the Hive silver tables and writing results back to Hive as a gold database.

Each business question has its own notebook cell with a named output table in the `gold` database.

## Business Questions

### 💰 Sales & Revenue

**Q1 — What is the total revenue per year?**
Joins `orders` + `order_items` to get revenue per year. 

**Q2 — Which product categories drive the most sales?**
Returns revenue per category with .

---

### 👥 Customer Behaviour

**Q3 — Who are the top 10 customers by total spend?**

---

### 📦 Order Operations

**Q4 — What is the order status breakdown?**

**Q5 — What is the average delivery time?**

---

### ⭐ Customer Satisfaction

**Q6 — What is the average review score by product category?**


## How to Run


```bash
# 1. Load CSVs into MariaDB
mysql --local-infile=1 -u your_user -p ecommerce < csv_to_mysql/load_csv_to_mariadb.sql

# 2. Ingest into HDFS (Bronze)
chmod +x sqoop/sqoop_import.sh
./sqoop/sqoop_import.sh

# 3. Run Spark cleaning notebook (Silver)
jupyter notebook silver_layer/silver_cleaning.ipynb

# 4. Run Spark SQL aggregation notebook (Gold)
jupyter notebook gold_layer/gold_agg.ipynb
```
