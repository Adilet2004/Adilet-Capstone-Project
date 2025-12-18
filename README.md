# Capstone Project  
## Adilet Abdulazisov 
** Used tools AWS Glue · Amazon S3 · Athena · Cloud9**

## 1. Project Goal

The goal of this project is to design and implement a cloud-based data engineering pipeline on AWS that ingests raw CSV datasets, transforms them into an optimized columnar format (Apache Parquet), catalogs the data using AWS Glue, and enables analytical querying using Amazon Athena.

The project focuses on analyzing global fishing activity data from the *Sea Around Us* dataset, with a particular emphasis on:
- Open seas (high seas) fishing
- Exclusive Economic Zones (EEZs)
- Country-level and species-level analysis (case study: Fiji)

---

## 2. Dataset Description

The dataset is provided by **Sea Around Us**, containing historical global fisheries data from **1950 to 2018**.

### Data Files Used

| File | Description |
|-----|------------|
| `SAU-GLOBAL-1-v48-0.csv` | All global high seas fishing data |
| `SAU-HighSeas-71-v48-0.csv` | Pacific, Western Central high seas area |
| `SAU-EEZ-242-v48-0.csv` | Exclusive Economic Zone (EEZ) of Fiji |

The data includes:
- Year of fishing activity
- Fishing country (`fishing_entity`)
- Fish species (`common_name`)
- Catch weight (`tonnes`)
- Economic value (`landed_value`, 2010 USD)

---

## 3. Architecture Overview

**Final Architecture:**

- **AWS Cloud9** – Development environment
- **Amazon S3** – Data lake (raw & processed Parquet files)
- **AWS Glue Crawler** – Schema inference and metadata catalog
- **AWS Glue Data Catalog** – Centralized metadata store
- **Amazon Athena** – Serverless SQL analytics
- **Athena Views** – Reusable analytical queries

---

## 4. Environment Setup

### AWS Resources

- **Cloud9 IDE**: `CapstoneIDE`
- **S3 Buckets**:
  - `data-source-0808` – Parquet data storage
  - `query-results-0808` – Athena query outputs
- **IAM Role**:
  - `CapstoneGlueRole`

---

## 5. Data Transformation

### CSV → Parquet Conversion

All CSV files were converted to Apache Parquet using **pandas** for improved query performance.

```python
import pandas as pd

df = pd.read_csv('SAU-GLOBAL-1-v48-0.csv')
df.to_parquet('SAU-GLOBAL-1-v48-0.parquet')
```
## 6. Schema Alignment (EEZ Dataset)

The EEZ dataset contained column names that were inconsistent with the other datasets.  
Although the data semantics were identical, schema alignment was required before ingestion.

### Column Mapping

| Original Column | Standard Column |
|-----------------|-----------------|
| `fish_name` | `common_name` |
| `country` | `fishing_entity` |

### Transformation Code

```python
import pandas as pd

df = pd.read_csv('SAU-EEZ-242-v48-0-old.csv')

df.rename(
    columns={
        "fish_name": "common_name",
        "country": "fishing_entity"
    },
    inplace=True
)

df.to_csv('SAU-EEZ-242-v48-0.csv', index=False)
df.to_parquet('SAU-EEZ-242-v48-0.parquet')
```
## 6. Schema Alignment (EEZ Dataset)

The EEZ dataset contained column names that were inconsistent with the other datasets.  
Although the data semantics were identical, schema alignment was required before ingestion.

### Column Mapping

| Original Column | Standard Column |
|-----------------|-----------------|
| `fish_name` | `common_name` |
| `country` | `fishing_entity` |

### Transformation Code

```python
import pandas as pd

df = pd.read_csv('SAU-EEZ-242-v48-0-old.csv')

df.rename(
    columns={
        "fish_name": "common_name",
        "country": "fishing_entity"
    },
    inplace=True
)

df.to_csv('SAU-EEZ-242-v48-0.csv', index=False)
df.to_parquet('SAU-EEZ-242-v48-0.parquet')
```

##7. AWS Glue Crawler Configuration

An AWS Glue crawler was configured to automatically infer schemas and populate the Glue Data Catalog.
Configuration:
* Crawler Name: `fishcrawler`
* Database: `fishdb`
* S3 Source: `s3://data-source-0808/`
* IAM Role: `CapstoneGlueRole`
* Schedule: `On demand`
* The crawler created a unified table:
  * `fishdb.data_source_0808`

## 8. Data Validation with Amazon Athena

To validate that all datasets were successfully integrated, the following query was executed:
```SQL
SELECT DISTINCT area_name
FROM fishdb.data_source_0808;
```
### Results:
`Pacific, Western Central`
`Fiji`
`NULL (global high seas)`

These results confirm that data from all sources was correctly ingested and categorized.

## 9. Analytical Queries (Fiji Case Study)
### Open Seas Catch
```SQL
SELECT year, fishing_entity AS Country,
CAST(CAST(SUM(landed_value) AS DOUBLE) AS DECIMAL(38,2)) AS ValueOpenSeasCatch
FROM fishdb.data_source_0808
WHERE area_name IS NULL
  AND fishing_entity = 'Fiji'
  AND year > 2000
GROUP BY year, fishing_entity
ORDER BY year;
```
### Fiji EEZ Catch
```SQL
SELECT year, fishing_entity AS Country,
CAST(CAST(SUM(landed_value) AS DOUBLE) AS DECIMAL(38,2)) AS ValueEEZCatch
FROM fishdb.data_source_0808
WHERE area_name LIKE '%Fiji%'
  AND fishing_entity = 'Fiji'
  AND year > 2000
GROUP BY year, fishing_entity
ORDER BY year;
```
### Combined EEZ and Open Seas Catch
```SQL
SELECT year, fishing_entity AS Country,
CAST(CAST(SUM(landed_value) AS DOUBLE) AS DECIMAL(38,2)) AS ValueEEZAndOpenSeasCatch
FROM fishdb.data_source_0808
WHERE (area_name LIKE '%Fiji%' OR area_name IS NULL)
  AND fishing_entity = 'Fiji'
  AND year > 2000
GROUP BY year, fishing_entity
ORDER BY year;
```
The combined values correctly equal the sum of EEZ and Open Seas catches, validating data consistency.

## 10. Athena View Creation

### A reusable Athena view was created to analyze mackerel catches.
```SQL
CREATE OR REPLACE VIEW MackerelsCatch AS
SELECT year,
       area_name AS WhereCaught,
       fishing_entity AS Country,
       SUM(tonnes) AS TotalWeight
FROM fishdb.data_source_0808
WHERE common_name LIKE '%Mackerels%'
  AND year > 2014
GROUP BY year, area_name, fishing_entity, tonnes
ORDER BY tonnes DESC;
```
## 11. View-Based Analysis
### Countries with Highest Mackerel Catch by Year
```SQL
SELECT year, Country, MAX(TotalWeight) AS Weight
FROM fishdb.mackerelscatch
GROUP BY year, Country
ORDER BY year, Weight DESC;
```
### Example: China
```SQL
SELECT *
FROM fishdb.mackerelscatch
WHERE country = 'China';
```
## 12. Key Outcomes
- Successfully built an end-to-end AWS-based data engineering pipeline
- Unified heterogeneous datasets into a single analytical schema
- Enabled serverless analytics using Amazon Athena
- Validated EEZ versus Open Seas data consistency
- Created reusable analytical views

## 13. Conclusion

This project demonstrates a complete cloud-based data engineering workflow, from raw data ingestion to analytical insights.
By leveraging AWS managed services, the solution achieves scalability, reliability, and efficient analytics while maintaining schema consistency across multiple heterogeneous datasets.
