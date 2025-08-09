# 🥫 Incremental Data Load into Delta Table – Event Driven (Databricks + GCS)

## 📌 Overview

This project implements an **incremental data loading pipeline** from **Google Cloud Storage (GCS)** into **Delta Tables** on Databricks, using an **event-driven file arrival trigger**.

The workflow automatically:

1. **Detects new files** uploaded by clients into the GCS `source` folder.
2. **Loads** them into a **stage Delta table**.
3. **Merges (UPSERT)** them into the target Delta table.
4. **Archives** processed files.

---

## 📊 Workflow Diagram

![Event_data_ingestion_architecture_diagram3](/Event_data_ingestion_architecture_diagram3.jpg)

---

## 🗂 Google Cloud Storage Structure

* **Bucket Name:** `incremental_load_dataaa`
* **Folders:**

  * `source/` → New client-uploaded CSV files.
  * `archive/` → Processed files moved here after ingestion.

Example:

```
incremental_load_dataaa/
│
├── source/
│   ├── file1.csv
│   ├── file2.csv
│
└── archive/
    ├── file1.csv
    ├── file2.csv
```

---

## ⚙ Databricks Setup

1. **External Location**

   ```
   for_incremental_load → gs://incremental_load_dataaa/
   ```

2. **Unity Catalog & Volume**

   * **Catalog:** `incremental_load`
   * **Volume Path:** `/Volumes/incremental_load/default/orders_data/`

     * `/source/` → mapped to GCS `source/`
     * `/archive/` → mapped to GCS `archive/`

3. **Tables**

   * **Stage Table:** `incremental_load.default.orders_stage`
   * **Target Table:** `incremental_load.default.orders_target`

---

## 🔄 Workflow & Version Control

### **Pipeline Name:**

`order_tracking_incremental_load`

* **Trigger:** **File Arrival** in GCS `source/` folder.
* **Tasks:**

  1. **order\_stage\_load** (Notebook in Git repo) – Reads from `source/`, loads stage table, moves files to `archive/`.
  2. **order\_target\_load** (Notebook in Git repo) – Performs Delta Lake **UPSERT** into target table.

Both notebooks are stored in a **Git-connected Databricks folder** for version control with GitHub.

---

## 📝 Task 1 – Stage Load (`order_stage_load`)

```python
source_dir = "/Volumes/incremental_load/default/orders_data/source/"
target_dir = "/Volumes/incremental_load/default/orders_data/archive/"
stage_table = "incremental_load.default.orders_stage"

# Read CSV files from source
df = spark.read.csv(source_dir, header=True, inferSchema=True)

# Overwrite into stage table
df.write.format("delta").mode("overwrite").saveAsTable(stage_table)

# Move processed files to archive
files = dbutils.fs.ls(source_dir)
for file in files:
    src_path = file.path
    target_path = target_dir + src_path.split("/")[-1]
    dbutils.fs.mv(src_path, target_path)
```

---

## 📝 Task 2 – Merge into Target Table (`order_target_load`)

```python
from delta.tables import DeltaTable
from pyspark.sql.utils import AnalysisException

stage_df = spark.read.table("incremental_load.default.orders_stage")
target_table_name = "incremental_load.default.orders_target"

# Check if target table exists
try:
    target_table = DeltaTable.forName(spark, target_table_name)
    table_exists = True
except AnalysisException:
    table_exists = False

if not table_exists:
    stage_df.write.format("delta").saveAsTable(target_table_name)
else:
    target_table.alias("target") \
        .merge(
            stage_df.alias("stage"),
            "stage.tracking_num = target.tracking_num"
        ) \
        .whenMatchedUpdateAll() \
        .whenNotMatchedInsertAll() \
        .execute()
```



## 🚀 How It Works

1. Client uploads a file (e.g., `file1.csv`) to **GCS** → `source/`.
2. **Databricks File Arrival Trigger** detects the new file.
3. **Workflow `order_tracking_incremental_load`** runs automatically:

   * **Task 1**: Loads CSV into stage table, moves file to `archive/`.
   * **Task 2**: Merges staged data into target table (UPSERT).
4. Data is now available in `orders_target` table for downstream analytics.

---

## ✅ Benefits

* Fully **event-driven** – no manual execution needed.
* **Version-controlled notebooks** for reproducibility.
* **Scalable incremental loading** with Delta Lake merge.
* **Automated archival** of processed files.

---

