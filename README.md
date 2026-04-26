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
```
+----+-------------+
|year|Total_Revenue|
+----+-------------+
|2016|     46653.74|
|2017|   6921535.24|
|2018|   8451584.77|
+----+-------------+
```
**Q2 — Which product categories drive the most sales?**
```
+--------------------+-------------+
|            category|Total_Revenue|
+--------------------+-------------+
|        beleza_saude|   1412089.53|
|  relogios_presentes|   1264333.12|
|     cama_mesa_banho|   1225209.26|
|       esporte_lazer|   1118256.91|
|informatica_acess...|   1032723.77|
|    moveis_decoracao|    880329.92|
|utilidades_domest...|    758392.25|
|          cool_stuff|    691680.89|
|          automotivo|    669454.75|
|  ferramentas_jardim|    567145.68|
|          brinquedos|    547061.06|
|               bebes|    466727.65|
|          perfumaria|    443171.63|
|           telefonia|    379202.62|
|   moveis_escritorio|    335211.36|
|           papelaria|    269575.05|
|            pet_shop|    250614.20|
|                 pcs|    228349.76|
|instrumentos_musi...|    202187.12|
|         eletronicos|    200723.09|
+--------------------+-------------+
```

---

### 👥 Customer Behaviour

**Q3 — Who are the top 10 customers by total spend?**
```
+--------------------+-------------+
|         customer_id|Total_Revenue|
+--------------------+-------------+
|1617b1357756262bf...|     13664.08|
|ec5b2ba62e5743423...|      7274.88|
|c6e2731c5b391845f...|      6929.31|
|f48d464a0baaea338...|      6922.21|
|3fd6777bbce08a352...|      6726.66|
|05455dfa7cd02f13d...|      6081.54|
|df55c14d1476a9a34...|      4950.34|
|24bbf5fd2f2e1b359...|      4764.34|
|3d979689f636322c6...|      4681.78|
|1afc82cd60e303ef0...|      4513.32|
+--------------------+-------------+
```

---

### 📦 Order Operations

**Q4 — What is the order status breakdown?**
```
+------------+------------+------------+
|order_status|total_orders|pct_of_total|
+------------+------------+------------+
|   delivered|       96478|       97.02|
|     shipped|        1107|        1.11|
|    canceled|         625|        0.63|
| unavailable|         609|        0.61|
|    invoiced|         314|        0.32|
|  processing|         301|        0.30|
|     created|           5|        0.01|
|    approved|           2|        0.00|
+------------+------------+------------+
```

**Q5 — What is the average delivery time?**
```
+------------------+
|  avg_delivry_days|
+------------------+
|12.497336125046644|
+------------------+
```

---

### ⭐ Customer Satisfaction

**Q6 — What is the average review score by product category? (top 10)**
```
+---------------------+----------------+
|product_category_name|avg_review_score|
+---------------------+----------------+
|    cds_dvds_musicais|            4.64|
| fashion_roupa_inf...|             4.5|
| livros_interesse_...|            4.45|
| construcao_ferram...|            4.44|
|               flores|            4.42|
|    livros_importados|             4.4|
|      livros_tecnicos|            4.36|
|     malas_acessorios|            4.31|
|    alimentos_bebidas|            4.31|
| portateis_casa_fo...|             4.3|
+---------------------+----------------+
```


## How to Run


```bash
# 1. Load CSVs into MariaDB on ecommerce schema
mysql --local-infile=1 -u your_user -p ecommerce < csv_to_mysql/load_csv_to_mariadb.sql

# 2. Ingest into HDFS (Bronze)
chmod +x sqoop/sqoop_import.sh
./sqoop/sqoop_import.sh

# 3. Run Spark cleaning notebook (Silver)
jupyter notebook silver_layer/silver_cleaning.ipynb

# 4. Run Spark SQL aggregation notebook (Gold)
jupyter notebook gold_layer/gold_agg.ipynb
```
