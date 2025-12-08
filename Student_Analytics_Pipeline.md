# Student Analytics Pipeline — Guide 🚀📊

## 🌟 Project Overview
This project automates the processing of student CSV files using Azure Data Factory (ADF). When a new or updated file arrives in the `raw/` container, ADF triggers a pipeline that:

- Validates file existence
- Copies the file to `processed/` with a timestamp
- Copies a log version to `logs/`
- Records file size
- Handles missing files cleanly

Perfect for team demos, learning paths, and real-world data engineering practice!

---

## 📁 Folder Structure (ADLS Gen2)

```
praccontainer/
 ├── raw/
 │     └── studentX.csv  
 ├── processed/
 │     └── studentX_20250101_102030.csv
 └── logs/
       └── log_studentX_20250101_102030.csv
```

---

## 🛠️ Step 1 — Create Required Azure Resources

### 1️⃣ Storage Account  
- Create ADLS Gen2-enabled storage  
- Create container `praccontainer`  
- Create folders:
  - `raw/`
  - `processed/`
  - `logs/`

### 2️⃣ Key Vault (Optional)  
- Store secrets securely  
- Connect via Managed Identity in ADF

---

## 🧩 Step 2 — Create Linked Services in ADF

### ✔ Linked Service for ADLS Gen2  
Name: `LS_practice2sa`  
Auth: System-assigned Managed Identity  

### ✔ Optional: Linked Service for Key Vault  
Name: `LS_KeyVault`

---

## 🗂️ Step 3 — Create Datasets

### 📌 `ds_student_raw` (full file path)
- Parameter: `FilePath` (String)
- Connection:
  - File path: `@dataset().FilePath`

### 📌 `ds_student_processed`
- Parameter: `TargetFile` (String)
- Connection: `processed/@dataset().TargetFile`

### 📌 `ds_student_logs`
- Parameter: `LogFile` (String)
- Connection: `logs/@dataset().LogFile`

---

## 🏗️ Step 4 — Build the Pipeline  
**Pipeline Name: `PL_StudentData_Processing`**

### Parameters:
| Name | Type | Description |
|------|--------|-------------|
| SourceFile | string | full raw path e.g. `raw/student.csv` |
| ProcessedFileNamePrefix | string | example: `student_marks` |

### Variables:
- `FileSize`
- `Status`

---

## 🔍 Step 5 — Activities Overview

### 1️⃣ Get Metadata  
Fields:
- exists  
- size  
- lastModified  
Dataset: `ds_student_raw`  
Mapping:
```
FilePath = @pipeline().parameters.SourceFile
```

### 2️⃣ If Condition  
Expression:
```
@activity('GetMeta_CheckFile').output.exists
```

### TRUE branch → Process file  
- **Copy_To_Processed**  
  - TargetFile:
    ```
    @concat(
        pipeline().parameters.ProcessedFileNamePrefix,
        '_',
        formatDateTime(utcNow(),'yyyyMMdd_HHmmss'),
        '.csv'
    )
    ```
- **Copy_To_Logs**
  - LogFile:
    ```
    @concat(
        'log_',
        last(split(pipeline().parameters.SourceFile,'/')),
        '_',
        formatDateTime(utcNow(),'yyyyMMdd_HHmmss'),
        '.csv'
    )
    ```

- **Set FileSize Variable**
  ```
  @string(activity('GetMeta_CheckFile').output.size)
  ```

### FALSE branch → Missing file  
- Set variable:
  ```
  "File Missing"
  ```

---

## ⚡ Step 6 — Event Trigger (New or Updated Files)

### Trigger Type:
✔ Storage events  
✔ Container: `praccontainer`  
✔ Path Begins With: `raw/`  
✔ Events:
- Blob Created  
- Blob Modified  

### Trigger Parameter Mapping:
| Pipeline Param | Expression |
|----------------|------------|
| SourceFile | `@triggerBody().fileName` |
| ProcessedFileNamePrefix | `student_marks` |

---

## 🧪 Step 7 — Testing

### ✔ Test 1: New file
Upload:  
`raw/new_students_01.csv`

Expected:
- Pipeline triggers
- Processed file appears
- Log file appears

### ✔ Test 2: Update file
Upload same file again → overwrite  
Expected:
- Pipeline triggers again

---

## 🎉 Final Outcome  
You now have a **fully automated**, **event-driven**, **incremental**, and **production-style** ADF pipeline.

Perfect for:
- Team learning  
- Demo presentations  
- Resume + portfolio  
- Real-world data engineering workflow  

---

## 🙌 Need More?  
I can create:
- PPT slides  
- Architecture diagrams  
- GitHub-ready README  
- Next-level upgrades (SQL, dedupe, watermarking)

Just ask! 😎🔥
