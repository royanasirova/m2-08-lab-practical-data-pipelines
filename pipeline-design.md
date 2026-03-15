## Task 1: Pipeline Architecture Diagram

```
                              Roya Nasirova Data Pipeline Design
                                    ======================

┌─────────────────┐                    ┌─────────────────┐
│   HISTORICAL    │                    │  LIVE STREAM    │
│   BATCH FILE    │                    │  TRANSACTIONS   │
│ (Excel sheet    │                    │ (New orders     │
│  with all old   │                    │  one by one)    │
│  transactions)  │                    │                 │
└────────┬────────┘                    └────────┬────────┘
         │                                       │
         └───────────────────┬───────────────────┘
                             ▼
                    ┌─────────────────┐
                    │  INGESTION      │
                    │  LAYER          │
                    │  - Load batch   │
                    │  - Catch stream │
                    │  - Check format │
                    └────────┬────────┘
                             ▼
                    ┌─────────────────┐
                    │  LANDING ZONE   │
                    │  RAW STORAGE    │
                    │  - Save as is   │
                    │  - Parquet files│
                    │  - By date      │
                    └────────┬────────┘
                             ▼
                    ┌─────────────────┐
                    │  VALIDATION     │
                    │  LAYER          │
                    │  - Check schema │
                    │  - Check ranges │
                    │  - Check rules  │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              │              ▼
    ┌─────────────────┐      │    ┌─────────────────┐
    │  DEAD LETTER    │      │    │  TRANSFORMATION │
    │  QUEUE          │      │    │  LAYER          │
    │  - Failed       │      │    │  - Clean data   │
    │    records      │      │    │  - Add columns  │
    │  - Error logs   │      │    │  - Summarize    │
    └─────────────────┘      │    └────────┬────────┘
                             │             │
                             ▼             ▼
                    ┌─────────────────────────┐
                    │     STORAGE LAYERS      │
                    │  ┌───────────────────┐  │
                    │  │ CLEAN LAYER       │  │
                    │  │ - Validated data  │  │
                    │  │ - Ready for use   │  │
                    │  └───────────────────┘  │
                    │           │              │
                    │  ┌───────────────────┐  │
                    │  │ FEATURE STORE     │  │
                    │  │ - Customer data   │  │
                    │  │ - For ML model    │  │
                    │  └───────────────────┘  │
                    └────────────┬────────────┘
                                 │
                 ┌───────────────┼───────────────┐
                 ▼               ▼               ▼
        ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
        │  BI DASHBOARD   │ │  DAILY REPORTS  │ │  ML MODEL       │
        │  - Sales charts │ │  - Email to     │ │  - Training     │
        │  - Top products │ │    managers     │ │  - Predictions  │
        │  - By country   │ │  - PDF exports  │ │                 │
        └─────────────────┘ └─────────────────┘ └─────────────────┘
                                 ▲
                                 │
                    ┌─────────────────────────┐
                    │    MONITORING           │
                    │    - Freshness checks   │
                    │    - Volume tracking    │
                    │    - Quality metrics    │
                    │    - Alerts dashboard   │
                    └─────────────────────────┘
```

---

# 1.2 Explanation of the Pipeline

The dataset contains transaction data from an online retail store. It includes information such as invoice number, product description, quantity, price, invoice date, customer ID, and country.
In this pipeline there would be two possible data sources. One is the historical dataset which contains all past transactions. This dataset would be loaded as a batch job. The second source represents new transactions that arrive when customers place orders.
Both sources would go into the ingestion layer. This layer would collect the data and check that the basic structure looks correct before storing it.
After that the data would go to the landing zone. This would be the raw storage area where the data is saved exactly as it arrives. Nothing would be changed here so the original data is always preserved.
Next the data would pass through a validation layer. This layer would check if the records follow expected rules such as correct data types, valid price ranges, and logical values. If a record fails validation it would be sent to the dead letter queue so it can be inspected later.
The transformation layer would be responsible for cleaning the data and creating new columns. For example revenue would be calculated using quantity multiplied by unit price. In the earlier tasks I also created customer level features such as total revenue, number of orders, number of products, first purchase date, and last purchase date.
After transformation the data would be stored in different layers. The clean layer would contain processed transaction data that can be used for analysis. The feature store would contain customer level features that are useful for machine learning models.
Finally the data could be used by dashboards, reports, or machine learning models. For example the features created earlier could be used to train a model that predicts whether a customer is high value or whether they will make a purchase in the future.
Monitoring would track whether the pipeline is running correctly and detect potential data quality issues.

---

# Task 2: Validation and Error Handling Design

## 2.1 Validation Rules

Since the dataset contains transaction records, I would define several validation rules.
First I would check the schema. Important fields such as invoice number, quantity, invoice date, and unit price should exist and have the correct data type. Customer ID could sometimes be missing because some purchases may not be linked to a registered customer.
Next I would check value ranges. Quantity would normally be positive, but negative values could appear for returns. Unit price should be greater than zero and within a reasonable range.
I would also check some basic business rules. For example if an invoice number starts with "C" it usually indicates a cancelled order, so the quantity should be negative. Another rule is that all rows with the same invoice number should belong to the same customer.
These checks would help ensure that the data is consistent before being used for analysis or machine learning.

---

## 2.2 Error Handling

If a record fails validation it would not be deleted. Instead it would be stored in a dead letter queue so it can be reviewed later.
For example if a record contains missing fields or invalid values it would be saved together with an error message explaining the problem.
The pipeline would also log these errors so they can be monitored. If many records suddenly start failing validation it could indicate a problem with the data source.
Once the issue is identified and fixed the rejected records could be reprocessed.

---

# Task 3: Transformation and Storage Design

## 3.1 Transformations

In the transformation step the data would be cleaned and additional features would be created.
First duplicates would be removed and formatting issues would be fixed. For example extra spaces in text fields could be removed and country names could be standardized.
Next derived columns would be created. For example revenue would be calculated by multiplying quantity and unit price. Time related features such as year or month could also be extracted from the invoice date.
In the previous tasks I also created customer level features using groupby operations. For each customer I calculated total revenue, number of orders, number of unique products, first purchase date, and last purchase date.
These features could then be used for machine learning models that predict customer behavior.

---

## 3.2 Storage Layers

The pipeline would store data in three main layers.
The raw layer would contain the original data exactly as it arrived. This would be useful for debugging or reprocessing.
The clean layer would contain validated and transformed transaction data that analysts or dashboards could use.
The feature layer would store aggregated customer features used by machine learning models. This is similar to the dataset created earlier when predicting high value customers.

---

## 3.3 Incremental Updates

The pipeline should not process the entire dataset every time new data arrives. Instead it would process only new records.
To do this the system would keep track of the last processed timestamp or message offset. When the pipeline runs again it would process only records that arrived after that point.
Sometimes records may arrive late. If the delay is small they could simply be added to the clean layer. If the delay is larger the system might need to recompute some customer features.
Customer features could also be refreshed periodically, for example once per day. During this refresh the system would update aggregates like total revenue or number of orders for customers who had new transactions.
This would ensure that the feature store always contains up to date information for machine learning models.