# 💥 Mixed File Format to Parquet Pipeline – Compressed Guide

## ⚡ Overview
This project processes **CSV, JSON, and Parquet files** from a `raw` folder and converts everything into **Parquet** format inside an `output` folder using **Azure Data Factory (ADF)**.

---

## ✔ 1. Create Required Datasets

### ✅ 1.1 DS_CCSV (DelimitedText)
- Format: **DelimitedText**
- Parameters:
  - `folderPath`
  - `fileName`
- Connection:
  - Folder path → `@dataset().folderPath`
  - File → `@dataset().fileName`
- Header: **True**

### ✅ 1.2 DS_JSON (JSON)
- Format: **JSON**
- Parameters:
  - `folderPath`
  - `fileName`
- Connection:
  - Folder path → `@dataset().folderPath`
  - File → `@dataset().fileName`

### ✅ 1.3 DS_PARQUET (Parquet)
- Format: **Parquet**
- Parameters:
  - `folderPath`
  - `fileName`

### ✅ 1.4 DS_PARQUET_OUT (Parquet Output)
- Format: **Parquet**
- Parameters:
  - `folderPath`
  - `fileName`

---

## ✔ 2. Create Pipeline

### ✅ 2.1 Add Get Metadata Activity
Name: **Get_File_List**

- Dataset: **DS_CSV**
- Parameters:
  - `folderPath = "raw"`
  - `fileName = ""`
- Field list: `childItems`

This retrieves all file names inside the `raw` folder.

---

## ✔ 3. Add ForEach Activity

### Settings
Items:
```
@activity('Get_File_List').output.childItems
```

---

## ✔ 4. Add Switch Activity
Expression:
```
@toLower(last(split(item().name, '.')))
```

---

## ✔ 5. Add Cases

### CSV Case
Input:
```
folderPath = "raw"
fileName = @item().name
```

Output:
```
folderPath = "output"
fileName = @concat(split(item().name,'.')[0], '.parquet')
```

### JSON Case
Same inputs, JSON reader.

### Parquet Case
Input + Output file name = @item().name

---

## 6. Validate JSON Files
Replace:
- NaN → null
- Infinity → null

---

## 7. Folder Layout
```
raw/
output/
```

---

## 8. Summary
- Parameterized datasets
- Dynamic file routing
- Multi-format processing
- Parquet conversion
