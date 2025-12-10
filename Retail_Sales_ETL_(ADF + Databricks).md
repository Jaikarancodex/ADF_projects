# 🚀Retail Sales ETL (ADF + Databricks) — Step-by-step Guide

> Purpose: end-to-end beginner-friendly guide to build a Bronze → Silver → Gold ETL using Azure Data Factory and Azure Databricks. Includes exact navigation, code snippets, and troubleshooting tips.

---

## ⭐Prerequisites

* Azure subscription (or free/Student subscription)
* Access to Azure Portal ([https://portal.azure.com](https://portal.azure.com))
* A Databricks workspace (we will create it)
* Basic familiarity with Portal UI

---

## 📁Folder & File naming conventions used in this guide

* **Resource group:** `RGnew` (example)
* **Storage account:** `karansa2s` (example)
* **Container:** `data`
* **Folders inside container:** `raw/`, `bronze/`, `silver/`, `gold/`, `reject/`
* **Sample file name(s):** `sales_customers_YYYY-MM-DD.csv`

---

## ✔️ 1. Create Resource Group

**Portal**: Portal → Resource Groups → + Create

* Name: `RGnew`
* Region: choose nearest

CLI (optional):

```bash
az group create -n RGnew -l eastus
```

---

## ✔️ 2. Create ADLS Gen2 Storage Account (with HNS)

**Portal**: Storage accounts → + Create

* Resource group: `RGnew`
* Name: e.g. `karansa2s` (must be globally unique)
* Account kind: `StorageV2`
* Enable **Hierarchical namespace** (this makes it ADLS Gen2)

CLI optional:

```bash
az storage account create --name karansa2s --resource-group RGnew --location eastus --sku Standard_LRS --kind StorageV2 --hierarchical-namespace true
```

---

## ✔️ 3. Create Container & Folder Structure

**Portal**: Storage account → Containers → + Container → name: `data`
Inside the container create folders (you can use the portal UI or upload with paths):

* `raw/`
* `bronze/`
* `silver/`
* `gold/`
* `reject/`

Upload a test CSV to `data/raw/` (sample CSV is provided in this guide). File name example: `sales_customers_2025-12-01.csv`.

---

## ✔️ 4. Sample CSV (copy & upload to `data/raw/`)

```
SaleId,CustomerName,Email,Phone,SaleDate,Amount,Qty,Item,Store
1001,Arun Kumar,arun.kumar@example.com,9876543210,2025-12-01,250,1,Mouse,A
1002,Priya Shah,priya.shah@example.com,9123456780,2025-12-01,-50,1,Mouse,B
1003,Ravi M,ravi.m@example.com,9988776655,,300,1,Keyboard,C
1004,Neha Singh,neha.singh@example.com,9090909090,2025-12-02,200,1,Tablet,A
1005,Amit Doshi,amit.doshi@example.com,9876501234,2025-12-02,0,2,Laptop,B
1006,Anjali Rao,anjali.rao@example.com,9876598765,2025-12-03,1200,1,Monitor,A
```

---

## ✔️ 5. Create Databricks Workspace + Cluster

**Portal**: Marketplace → Azure Databricks → Create

* Resource group: `RGnew`
* Workspace name: e.g. `db-karan-demo`
* Pricing tier: Standard (or trial)

After creation → Launch Workspace → Compute → Create Cluster

* Name: `cluster-demo`
* Mode: Single Node (or Standard)
* Small VM size (to save cost)
* Auto-terminate: 20/30 min

---

## ✔️ 6. Give Databricks access to Storage (quick beginner option)

**Portal**: Storage account → Access keys → copy Key 1
In Databricks notebook we set the spark config using this key (for learning only). In production use Managed Identity or Service Principal + Key Vault.

Example snippet (run in notebook before file operations):

```python
storage_account = "karansa2s"
storage_key = "<PASTE_STORAGE_KEY>"
spark.conf.set(
  f"fs.azure.account.key.{storage_account}.dfs.core.windows.net",
  storage_key
)
```

---

## ✔️ 7. Create `BronzeToSilver` Notebook (Python)

**Workspace** → Create → Notebook → Name: `BronzeToSilver` → Language: Python → Attach to `cluster-demo`.

### 🔸 7.1 Read RAW CSV (top cells)

```python
dbutils.widgets.text("FileName", "")
file_name = dbutils.widgets.get("FileName")

storage_account = "karansa2s"
# paste your storage key for learning only
storage_key = "<PASTE_STORAGE_KEY_HERE>"

spark.conf.set(f"fs.azure.account.key.{storage_account}.dfs.core.windows.net", storage_key)

raw_path = f"abfss://data@{storage_account}.dfs.core.windows.net/raw/{file_name}"

df = spark.read.option("header","true").option("inferSchema","true").csv(raw_path)
display(df)
```

### 🔸 7.2 Save Bronze (parquet)

```python
bronze_path = f"abfss://data@{storage_account}.dfs.core.windows.net/bronze/{file_name.replace('.csv','')}"
df.write.mode('overwrite').parquet(bronze_path)
```

### 🔸 7.3 Cleaning rules & reject

```python
from pyspark.sql.functions import col
# keep only rows where SaleDate is not null and Amount > 0

df_clean = df.filter((col('SaleDate').isNotNull()) & (col('Amount') > 0))
df_reject = df.filter((col('SaleDate').isNull()) | (col('Amount') <= 0))
```

### 🔸 7.4 Use YOUR UDF-style Email mask (gmail only)

```python
from pyspark.sql.functions import udf
from pyspark.sql.types import StringType

@udf(StringType())
def Memail(x):
    if x is None:
        return None

    if "@gmail.com" not in x.lower():
        return x

    f = x[0]
    count = 0
    for i in x:
        if i == "@":
            break
        else:
            count += 1
    index = x[count:]
    for _ in range(count - 1):
        f += "*"
    return f + index

# ✔️🔐apply masking
from pyspark.sql.functions import col

df_masked = df_clean.withColumn('EmailMasked', Memail(col('Email')))

# optional: mask phone and name similarly (implement if desired)
```

### 🔸 7.5 Write Silver & Reject

```python
silver_path = f"abfss://data@{storage_account}.dfs.core.windows.net/silver/{file_name.replace('.csv','')}"
df_masked.write.mode('overwrite').parquet(silver_path)

reject_path = f"abfss://data@{storage_account}.dfs.core.windows.net/reject/{file_name.replace('.csv','')}_reject"
df_reject.write.mode('overwrite').csv(reject_path, header=True)
```

---

## ✔️ 8. Create `SilverToGold` Notebook (Python)🏅

**Workspace** → New Notebook → Name: `SilverToGold` → Attach to `cluster-demo`.

### 🔸 8.1 Read Silver (use FileName widget)

```python
dbutils.widgets.text("FileName", "")
file_name = dbutils.widgets.get("FileName")

storage_account = "karansa2s"
# same spark.conf setup as needed

silver_folder = file_name.replace('.csv','')
silver_path = f"abfss://data@{storage_account}.dfs.core.windows.net/silver/{silver_folder}/"

df_silver = spark.read.parquet(silver_path)
display(df_silver)
```

### 🔸 8.2 Aggregations (Gold tables)

```python
from pyspark.sql.functions import col, sum as _sum, count, desc

# 🎯Daily sales

daily_sales = df_silver.groupBy('SaleDate').agg(
  _sum('Amount').alias('TotalRevenue'),
  _sum('Qty').alias('TotalQty'),
  count('*').alias('TransactionCount')
)

# 🎯 Sales by store
sales_by_store = df_silver.groupBy('Store').agg(
  _sum('Amount').alias('TotalRevenue'),
  _sum('Qty').alias('TotalQty'),
  count('*').alias('TransactionCount')
)

# 🎯 Top items
top_items = df_silver.groupBy('Item').agg(_sum('Qty').alias('TotalQtySold')).orderBy(desc('TotalQtySold'))
```

### 🔸 8.3 Write Gold

```python
daily_sales.write.mode('overwrite').parquet(f"abfss://data@{storage_account}.dfs.core.windows.net/gold/daily_sales")
sales_by_store.write.mode('overwrite').parquet(f"abfss://data@{storage_account}.dfs.core.windows.net/gold/sales_by_store")
top_items.write.mode('overwrite').parquet(f"abfss://data@{storage_account}.dfs.core.windows.net/gold/top_items")
```

---

## ✔️ 9. Azure Data Factory — Linked Services

**Author** → Manage → Linked services → + New

1. **ADLS Gen2** (LS_ADLS_karansa2s)

   * Type: Azure Data Lake Storage Gen2
   * Authentication: Account Key
   * Storage account: `karansa2s`
   * Test connection → Create

2. **Azure Databricks** (LS_Databricks)

   * Type: Azure Databricks
   * Authentication: Access Token
   * Workspace URL: your workspace URL (from Databricks)
   * Access Token: generate in Databricks (User Settings → Tokens)
   * Test → Create

---

## ✔️ 10. ADF Pipeline (PL_Sales_ETL) — simple orchestration

**Author → Pipelines → New pipeline**

### 🔸 10.1 Create Activities

* Drag **Get Metadata** → configure dataset pointing at `data/raw` folder (leave filename blank). In field list add `childItems`.
* Drag **ForEach** → set Items to: `@activity('GetRawFile').output.childItems`
* Inside ForEach activities (in the ForEach inner canvas):

  * Databricks Notebook activity → NB_BronzeToSilver

    * Linked service: `LS_Databricks`
    * Notebook path: `/Workspace/.../BronzeToSilver`
    * Base parameters: `FileName = @item().name`
  * Databricks Notebook activity → NB_SilverToGold (connect after NB_BronzeToSilver)

    * Base parameters: `FileName = @item().name`

> NOTE: Do NOT create a top-level pipeline parameter for `FileName`. Use @item().name inside ForEach.

### 🔸 10.2 Validate & Debug

* Click **Debug** to run one iteration. Confirm both notebooks run and outputs are created in storage.

---

## ✔️ 11. Create Storage Event Trigger (Auto-run on new file)

**Author → Manage → Triggers → + New → Storage events**

Trigger fields (CRITICAL):

* Name: `TRG_NewFile_AutoRun`
* Account selection: From Azure subscription
* Storage account name: `karansa2s`
* Container name: `data`
* Blob path begins with: `raw/`  ← **very important**
* Blob path ends with: `.csv`
* Event: Blob created
* Ignore empty blobs: Yes
* Start trigger on creation: Checked (if you want it active immediately)

Click **Continue → OK**

**After creating trigger**: go to your pipeline canvas → Add Trigger → Choose `TRG_NewFile_AutoRun` → OK → Publish All.

> If ADF forces a pipeline parameter value at publish time: delete any top-level pipeline parameter named `FileName`, ensure `FileName` is passed inside ForEach as `@item().name`, then publish again.

---

## ✔️ 12. Final test (upload a NEW file)

Upload a new file to `data/raw/` (use a new filename every time, e.g. `sales_customers_2025-12-03_test.csv`).

Wait 5–20 seconds and check: ADF → Monitor → Pipeline runs. You should see the pipeline triggered by the storage event and the two Databricks notebooks running in order.

---

## ✔️ 13. Troubleshooting checklist🧪

* Trigger not firing: confirm trigger status is Enabled (ADF → Manage → Triggers)
* Event Subscription prefix: Storage account → Events → ensure prefix shows `raw/`
* If publish blocks on parameter: delete pipeline-level `FileName` parameter
* If notebook cannot read storage: verify `spark.conf` was set with the correct storage key or use secret scopes/service principal
* If event subscription shows delivery errors: open Event Subscription details and check Dead-letter/Errors

---

## ✅ 14. Good-to-have improvements (next steps)

* Use Service Principal + Key Vault + Databricks Secret Scope instead of account key
* Use Delta Lake (.write.format("delta")) instead of parquet for Gold and Silver
* Implement SCD Type-2 in Gold if customer history is required
* Add notifications/email on pipeline failures
* Add idempotency: check if silver folder exists to avoid reprocessing the same file

---

## ✅ 15. Appendix — Quick reference commands & snippets

* Set spark config:

```python
spark.conf.set(f"fs.azure.account.key.{storage_account}.dfs.core.windows.net", storage_key)
```

* Read CSV:

```python
spark.read.option('header','true').option('inferSchema','true').csv(raw_path)
```

* Write parquet:

```python
df.write.mode('overwrite').parquet(bronze_path)
```

---

**Done.**

> 🧪Open the file in the canvas on the right side and use Download to save the markdown locally.
