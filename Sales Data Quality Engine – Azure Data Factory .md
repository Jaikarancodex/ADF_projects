# 🚀 Sales Data Quality Engine – Azure Data Factory (End-to-End Project)

A complete walkthrough to build a Data Quality (DQ) engine using Azure Data Factory that:

- Validates a Sales CSV file  
- Detects missing / negative / corrupted values  
- Enriches bad rows with `ReasonForFailure` and `ErrorValue`  
- Splits **valid** vs **invalid** rows  
- Writes outputs to curated & error zones in ADLS  

---

## 1️⃣ Create Required Azure Resources

### Resource Group
- Name: `RGnew2`

### Storage Account
- Name: `projectfindx`
- Enable: **Hierarchical Namespace (ADLS Gen2)**

### Container Structure
Create container:  
```
data
```

Inside it, create folders:
```
raw/sales
polished/sales
error/sales
```

Upload:
```
data/raw/sales/sales_data_demox.csv
```

---

## 2️⃣ Build Data Factory Structure

| Object | Name |
|--------|------|
| Linked Service | `LS_ADLS_gen2` |
| Dataset | `Sales_data_demox` |
| Data Flow | `DF_Sales_DataQuality` |
| Pipeline | `PL_Sales_DQ_Processor` |

---

## 3️⃣ Dataset Setup (Sales_data_demox)

### Dataset Parameter
```
fileName (String)
```

### Path
```
data/raw/sales/@dataset().fileName
```

---

## 4️⃣ Pipeline: PL_Sales_DQ_Processor

### Pipeline Parameter
```
fileName : sales_data_demox.csv
```

---

## 5️⃣ Data Flow: DF_Sales_DataQuality

### 5.1 Source
Dataset: `Sales_data_demox`  
Param: `fileName`

---

### 5.2 Derived Column – Casting
```
Amount_str    = iif(isNull(Amount), '', toString(Amount))
Quantity_str  = iif(isNull(Quantity), '', toString(Quantity))
```

---

### 5.3 Derived Column – Error Enrichment

#### ReasonForFailure
```
iif(isNull(SaleID) || SaleID == '', 'SaleID missing; ', '') +
iif(isNull(Branch) || Branch == '', 'Branch missing; ', '') +
iif(isNull(SaleDate) || SaleDate == '', 'SaleDate missing; ', '') +
iif(Amount_str == '', 'Amount missing; ', '') +
iif(Quantity_str == '', 'Quantity missing; ', '') +
iif(isNull(Product) || Product == '', 'Product missing; ', '') +
iif(Amount_str != '' && toInteger(Amount_str) < 0, 'Amount negative; ', '') +
iif(Quantity_str != '' && toInteger(Quantity_str) < 0, 'Quantity negative; ', '')
```

#### ErrorValue
```
iif(Amount_str == '' || toInteger(Amount_str) < 0, concat('Amount=', Amount_str, '; '), '') +
iif(Quantity_str == '' || toInteger(Quantity_str) < 0, concat('Quantity=', Quantity_str, '; '), '') +
iif(isNull(SaleDate) || SaleDate == '', 'SaleDate=NULL; ', '') +
iif(isNull(Branch) || Branch == '', 'Branch=NULL; ', '') +
iif(isNull(Product) || Product == '', 'Product=NULL; ', '')
```

---

### 5.4 Conditional Split

#### valid
```
SaleID != '' &&
Branch != '' &&
SaleDate != '' &&
Amount_str != '' &&
Quantity_str != '' &&
Product != '' &&
toInteger(Amount_str) >= 0 &&
toInteger(Quantity_str) >= 0
```

#### invalid
```
true()
```

---

### 5.5 Sink – Valid Rows
```
data/polished/sales/
```

### 5.6 Sink – Invalid Rows
```
data/error/sales/
```

Both with:
- Inline DelimitedText  
- Schema drift ON  
- Auto-map ON  

---

## 6️⃣ Add Data Flow to Pipeline

Drag **Data Flow activity** into canvas, then:

```
Data flow = DF_Sales_DataQuality
fileName  = @pipeline().parameters.fileName
```

---

## 7️⃣ Run the Pipeline
- Click **Debug**
- Start Data Flow debug session
- Pipeline processes file and writes outputs

---

## 8️⃣ Validate Output

### Valid rows:
```
data/polished/sales/
```

### Invalid rows:
```
data/error/sales/
```

Includes:
- ReasonForFailure  
- ErrorValue  

---

# ✔️ Completed Data Quality Engine  
Your ADF pipeline now validates, enriches, and routes data cleanly for downstream consumption.
