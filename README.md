# An-Open-Source-Data-Lakehouse
## Overview

This project implements a complete open‑source data lakehouse using Apache Spark, Delta Lake, Hadoop HDFS, Docker, and Tableau. It ingests data from three heterogeneous sources (SQL Server, MongoDB, and a Parquet file), processes it through a medallion architecture (Bronze, Silver, Gold) with Delta Lake as an open table format and finally visualizes key business metrics in a Tableau dashboard.


---

## Architecture Diagram

<img width="2527" height="971" alt="architecture_diagram" src="https://github.com/user-attachments/assets/fce55084-a8de-45b5-a71c-b3bcaaea773e" />


---
## Tools & Technologies

| Component          | Technology                          |
|--------------------|-------------------------------------|
| Containerization   | Docker              |
| Distributed Storage| Hadoop HDFS                         |
| Processing Engine  | Apache Spark               |
| Lake Format        | Delta Lake                |
| Metastore          | Hive Metastore (PostgreSQL backend) |
| Data Sources       | SQL Server, MongoDB, Parquet file   |
| Visualization      | Tableau Desktop                     |

---

## Bronze Layer

Three raw datasets are ingested as Delta tables on HDFS. Each pipeline adds an `ingestion_timestamp`, writes in `overwrite` mode, and registers the table in Hive.

| Source     | Dataset                               | Bronze Table                 | HDFS Path                      |
|------------|---------------------------------------|------------------------------|--------------------------------|
| SQL Server | `customer_db.dbo.customer_data`  | `bronze.bronze_customer`  | `/delta/bronze/customer`       |
| MongoDB    | `products_1.products_1`     | `bronze.bronze_products`  | `/delta/bronze/products`       |
| Parquet    | `transaction.snappy.parquet`    | `bronze.bronze_transactions` | `/delta/bronze/transactions` |

---

## Silver Layer

Reads incrementally from Bronze via `ingestion_timestamp`, applies quality rules and derives new columns, then upserts via Delta Lake merge. All tables stored under `/delta/silver/…` and registered in Hive.

| Table | Quality / Filters | Derived Columns |
|-------|-------------------|-----------------|
| `silver.silver_customers` | Age 18–100, non-null email, `total_purchases ≥ 0` | `customer_segment` (High/Medium/Low Value), `days_since_registration`, `last_updated` |
| `silver.silver_products` | `price`, `stock_quantity ≥ 0`; `rating` in [0,5]; non-null `name` & `category` | `price_category` (Premium/Standard/Budget), `stock_status` (Out of/Low/Moderate/Sufficient Stock) |
| `silver.silver_orders` | `quantity`, `total_amount ≥ 0`; non-null `transaction_date`, `customer_id`, `product_id` | `order_status` (Cancelled if qty or amount = 0, else Completed) |

---

## Gold Layer

Two aggregate tables built from Silver for BI tools.

| Table | Logic | Dashboard Use |
|-------|-------|---------------|
| `gold.gold_category_sales` | Join orders + products → group by `category`, sum `total_amount` | Revenue per category |
| `gold.gold_daily_sales` | Group orders by `transaction_date`, sum `total_amount` | Sales trend over time |

---
## BI Layer

<img width="1583" height="807" alt="tableau dashboard" src="https://github.com/user-attachments/assets/a2aac2f8-f2de-4eaa-9b1e-61e43939fc95" />


The tableau dashboard visualises the business metrics and reveals the following key business insights:

- Total revenue reached 4.66M across 9.7K orders, with an average order value of 480
- Sales show a generally stable trend with slight growth over time
- Clothing, Sports, and Toys are the top-performing categories
- Revenue is globally distributed, with Brazil, Germany, and Australia leading
- Premium customers contribute the highest share of revenue
---

## Access Ports (Local Deployment)

After running `docker compose up -d`, the following services are exposed on `localhost`:

| Service                  | Port / URL                                        |
|--------------------------|---------------------------------------------------|
| Hadoop NameNode UI       | http://localhost:9870                             |
| Hadoop ResourceManager UI| http://localhost:8088                             |
| Hadoop HistoryServer UI  | http://localhost:19888                            |
| DataNodes                | 9864 (worker1), 9865 (worker2), 9866 (worker3)   |
| NodeManagers             | 8042 (worker1), 8043 (worker2), 8044 (worker3)   |
| Spark Master UI          | http://localhost:8080                             |
| Spark Workers UI         | 8081 (worker1), 8082 (worker2), 8083 (worker3)   |
| Spark History Server UI  | http://localhost:18080                            |
| Spark Master URL         | `spark://172.28.1.2:7077`                         |
| Jupyter Notebook         | http://localhost:8888                             |
| Hive Metastore (Thrift)  | `thrift://master:9083`                            |
| HiveServer2 (JDBC)       | `jdbc:hive2://master:10000`                       |
| PostgreSQL (Metastore DB)| `localhost:5432` (user: `postgres`, password: `jupyter`) |

> **Note**: All internal IPs are fixed via a custom Docker network (`sparknet` subnet `172.28.0.0/16`). Use `localhost` for browser access, but Spark applications inside the cluster must use the service names (`master`, `worker1`, etc.).

---

