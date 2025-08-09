# 🥫 Order Tracking - Event Driven Data Ingestion

## 📌 Project Overview
This project implements an **event-driven data ingestion pipeline** in **Databricks** for processing order tracking files.  
It automatically triggers workflows upon **file arrival**, stages the incoming data, and performs **SCD Type 1 (upsert) merges** into a Delta Lake target table.

The workflow consists of two jobs:
1. **Orders Stage Load** - Reads raw files from Databricks Volumes and loads them into a staging Delta table.
2. **Orders Target Merge** - Performs an incremental upsert (SCD1) from the staging table into the target Delta table.

---

## 🛠 Tech Stack
- **Google Cloud Storage (GCS)** – Raw file storage
- **Databricks** – Data engineering & orchestration
- **PySpark** – Distributed data processing
- **Delta Lake** – ACID-compliant storage format
- **Databricks Workflows** – Job scheduling & triggers
- **GitHub** – Version control for notebooks

---

## 📂 Project Structure
```plaintext
.
├── notebooks/
│   ├── orders_stage.py       # Stage job: Load raw files → staging Delta table
│   ├── orders_target.py      # Target job: Merge from staging → target Delta table
├── README.md
