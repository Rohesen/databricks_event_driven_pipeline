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

````

---

## ⚙️ Setup Instructions

### 1️⃣ Clone the repository

```bash
git clone https://github.com/<your-username>/<repo-name>.git
cd <repo-name>
```

### 2️⃣ Connect GitHub with Databricks

1. Go to **User Settings → Git Integration** in Databricks.
2. Connect with your GitHub account and configure the repository.

### 3️⃣ Create Databricks Volumes

```sql
CREATE CATALOG IF NOT EXISTS incremental_load;
CREATE SCHEMA IF NOT EXISTS incremental_load.default;
```

### 4️⃣ Upload Sample Files

Place daily order tracking CSV files into:

```
/Volumes/incremental_load/default/orders_data/
```

---

## 📜 Step-by-Step Job Execution Flow

### **1. File Arrival**

* A new CSV file arrives in `/Volumes/incremental_load/default/orders_data/`
* Databricks Workflow **triggers automatically**.

### **2. Orders Stage Load Job**

* Reads CSV from source directory.
* Writes to **staging Delta table**: `incremental_load.default.orders_stage`.
* Moves processed files to archive directory.

### **3. Orders Target Merge Job**

* Checks if `orders_target` Delta table exists.

  * If **not exists** → Creates target table from staging data.
  * If **exists** → Performs **SCD1 upsert**:

    * **Update** if `tracking_num` matches.
    * **Insert** if no match.
* Target table: `incremental_load.default.orders_target`.

---

## 🖼 Architecture Diagram

```mermaid
flowchart LR
    A[📁 CSV Files in GCS/Databricks Volume] -->|File Arrival Trigger| B[📦 Orders Stage Job]
    B -->|Write Staging Delta Table| C[(🗄 orders_stage Delta Table)]
    C --> D[⚡ Orders Target Job]
    D -->|Merge (SCD1)| E[(🗄 orders_target Delta Table)]
    B -->|Move Processed Files| F[📂 Archive Directory]
```

---

## 📜 Sample Merge Code (SCD1)

```python
from delta.tables import DeltaTable
from pyspark.sql.utils import AnalysisException

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

---

## ⚡ Event-Driven Trigger

* The Databricks Workflow is configured to **trigger automatically when a file is uploaded** to the source volume.
* Job sequence:

  1. **Orders Stage Load**
  2. **Orders Target Merge**

---

## 🧪 Testing

Upload a sample CSV file:

```csv
tracking_num,order_id,status
12345,1,Delivered
67890,2,Shipped
```

Then run:

```sql
SELECT * FROM incremental_load.default.orders_target;
```

---

## 📌 Notes

* Ensure **Unity Catalog write access** to the target schema.
* For shared clusters, use a **single-user cluster** to avoid JVM attribute restrictions.
* Handle schema evolution if CSV structure changes.

---

## 📜 License

This project is licensed under the MIT License – see the LICENSE file for details.


