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
## Analysis A: Fiji Open Seas Catch (Excluding EEZ)

This analysis focuses exclusively on fish catches made by Fiji in **open seas areas**, explicitly excluding any catches within the Fiji Exclusive Economic Zone (EEZ).

Open seas records are identified by the absence of an `area_name` value (`area_name IS NULL`), which corresponds to fishing activities conducted outside national EEZ boundaries.

### SQL Query

```sql
SELECT year,
       fishing_entity AS Country,
       CAST(CAST(SUM(landed_value) AS DOUBLE) AS DECIMAL(38,2)) AS ValueOpenSeasCatch
FROM fishdb.data_source_0808
WHERE area_name IS NULL
  AND fishing_entity = 'Fiji'
  AND year > 2000
GROUP BY year, fishing_entity
ORDER BY year;
```
| Year | Country | Open Seas Catch Value (USD) |
| ---: | ------- | --------------------------: |
| 2001 | Fiji    |                6,549,257.47 |
| 2002 | Fiji    |                6,519,235.99 |
| 2003 | Fiji    |                6,268,209.46 |
| 2004 | Fiji    |                7,326,897.74 |
| 2005 | Fiji    |                7,051,547.92 |
| 2006 | Fiji    |                7,080,650.49 |
| 2007 | Fiji    |                6,745,242.63 |
| 2008 | Fiji    |                6,776,453.14 |
| 2009 | Fiji    |                6,956,364.81 |

## Analysis B: Fiji EEZ Catch (Including Exclusive Economic Zone)

This analysis examines fish catches made by Fiji **within its Exclusive Economic Zone (EEZ)**, covering fishing activities that occurred inside national maritime boundaries.

EEZ records are identified using the `area_name` field, where values contain the string `Fiji`.

### SQL Query

```sql
SELECT year,
       fishing_entity AS Country,
       CAST(CAST(SUM(landed_value) AS DOUBLE) AS DECIMAL(38,2)) AS ValueEEZCatch
FROM fishdb.data_source_0808
WHERE area_name LIKE '%Fiji%'
  AND fishing_entity = 'Fiji'
  AND year > 2000
GROUP BY year, fishing_entity
ORDER BY year;
```
| Year | Country | EEZ Catch Value (USD) |
| ---: | ------- | --------------------: |
| 2001 | Fiji    |          6,248,871.36 |
| 2002 | Fiji    |          5,370,834.33 |
| 2003 | Fiji    |          6,908,355.08 |
| 2004 | Fiji    |          7,033,705.33 |
| 2005 | Fiji    |          6,984,709.97 |
| 2006 | Fiji    |          7,301,642.06 |
| 2007 | Fiji    |          8,680,240.18 |
| 2008 | Fiji    |          7,419,271.49 |
| 2009 | Fiji    |          6,861,470.47 |
| 2010 | Fiji    |          6,060,812.43 |

## Analysis C: Fiji Combined Catch (EEZ + Open Seas)

This analysis aggregates fish catch values for Fiji from **both the Exclusive Economic Zone (EEZ) and open seas areas**, providing a complete picture of national fishing activity.

The combined dataset includes:
- EEZ records (`area_name LIKE '%Fiji%'`)
- Open seas records (`area_name IS NULL`)

### SQL Query

```sql
SELECT year,
       fishing_entity AS Country,
       CAST(CAST(SUM(landed_value) AS DOUBLE) AS DECIMAL(38,2)) AS ValueEEZAndOpenSeasCatch
FROM fishdb.data_source_0808
WHERE (area_name LIKE '%Fiji%' OR area_name IS NULL)
  AND fishing_entity = 'Fiji'
  AND year > 2000
GROUP BY year, fishing_entity
ORDER BY year;
```
| Year | Country | EEZ + Open Seas Catch Value (USD) |
| ---: | ------- | --------------------------------: |
| 2001 | Fiji    |                     12,797,997.82 |
| 2002 | Fiji    |                     11,889,193.92 |
| 2003 | Fiji    |                     13,177,045.54 |
| 2004 | Fiji    |                     14,360,602.07 |
| 2005 | Fiji    |                     14,036,167.90 |
| 2006 | Fiji    |                     14,382,592.55 |
| 2007 | Fiji    |                     15,425,482.81 |
| 2008 | Fiji    |                     13,918,572.63 |
| 2009 | Fiji    |                     13,817,836.28 |
| 2010 | Fiji    |                     12,377,461.42 |

## Key Analytical Insights

### Analysis A: Open Seas (Excluding EEZ)
Fiji’s open seas fishing activity demonstrates a stable and predictable economic contribution, averaging approximately 6–7 million USD annually.  
This consistency indicates sustained engagement in international waters, largely independent of domestic regulatory or environmental constraints.

### Analysis B: EEZ (Domestic Waters)
Fishing within Fiji’s Exclusive Economic Zone shows higher variability and stronger peaks compared to open seas catches.  
This suggests that domestic-zone fishing is more sensitive to policy, ecological conditions, and operational intensity, while also offering greater revenue potential in peak years.

### Analysis C: Combined EEZ and Open Seas
The combined analysis confirms that Fiji’s fisheries economy relies on a balanced interaction between domestic and international fishing activities.  
Data consistency across all three analyses validates the reliability of the data pipeline and confirms that total catch values accurately represent the sum of their components.

## Summary of analysis 

This project delivers an end-to-end data engineering pipeline for analyzing global fisheries data using AWS managed services.  
Three complementary analyses were conducted to evaluate Fiji’s fishing activity: open seas catches, EEZ catches, and a combined aggregation.  
Results show that open seas fishing provides a stable annual baseline, while EEZ fishing contributes higher variability and peak values.  
The combined analysis confirms data consistency and demonstrates that Fiji’s fisheries economy depends on a balanced contribution from both domestic and international waters.  
Overall, the solution validates scalable ingestion, schema harmonization, and reliable analytics using AWS Glue and Amazon Athena.




## 12. Key Outcomes
- Successfully built an end-to-end AWS-based data engineering pipeline
- Unified heterogeneous datasets into a single analytical schema
- Enabled serverless analytics using Amazon Athena
- Validated EEZ versus Open Seas data consistency
- Created reusable analytical views

## 13. Conclusion

This project demonstrates a complete cloud-based data engineering workflow, from raw data ingestion to analytical insights.
By leveraging AWS managed services, the solution achieves scalability, reliability, and efficient analytics while maintaining schema consistency across multiple heterogeneous datasets.
