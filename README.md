%md

# Formula 1 Data Lakehouse

## Project Overview

This project implements an end-to-end **Formula 1 Data Lakehouse on Azure Databricks**.

The solution ingests Formula 1 source data stored in **Azure Data Lake Storage Gen2**, processes it through **Landing, Bronze, Silver and Gold layers**, and builds analytics-ready datasets for analyzing driver and constructor performance across seasons.

The final Gold model is exposed through analytics views and an interactive **Databricks Dashboard**.

---

## Architecture

```text
Formula 1 Source Data
        |
        v
Azure Data Lake Storage Gen2
        |
        v
Landing
Raw CSV / JSON Files
Unity Catalog External Volume
        |
        v
Bronze
Delta Tables
Schema + Audit Metadata
        |
        v
Silver
Clean + Standardized + DQ
        |
        v
Gold
Dimensional Model
        |
        v
Analytics Views
        |
        v
Databricks Dashboard
```

---

## Data Source

The project uses Formula 1 data based on the open-source **Jolpica F1 dataset/API**, following the relational-style Ergast data model.

| Dataset | Description | Format |
|---|---|---|
| Circuits | Circuit and geographical information | CSV |
| Races | Race details by season and round | CSV |
| Constructors | Formula 1 teams | JSON |
| Drivers | Driver information | Nested JSON |
| Results | Race results, positions, laps and points | Multi-file JSON |
| Sprints | Sprint race results | Multi-line, multi-file JSON |

The project handles multiple ingestion patterns including CSV, JSON, nested JSON, multi-line JSON and multi-file datasets.

---

## Technology Stack

- Azure Databricks
- Apache Spark / PySpark
- Spark SQL
- Azure Data Lake Storage Gen2
- Delta Lake
- Unity Catalog
- Databricks Jobs
- Databricks Dashboards

---

## Lakehouse Design

### Landing

Raw Formula 1 files are stored in **ADLS Gen2** and exposed to Databricks through a **Unity Catalog External Volume**.

No business transformation is applied at this stage.

### Bronze

The Bronze layer stores a structured and traceable representation of the source data.

Key processing includes:

- Explicit schema and data type handling
- Addition of `ingestion_timestamp`
- Addition of `source_file`
- Minimal transformation
- Storage as Delta tables

### Silver

The Silver layer creates clean and trusted datasets.

Transformations include:

- Standardizing column names and data types
- Flattening nested JSON
- Removing unnecessary attributes
- Null business-key validation
- Duplicate removal
- Preserving business keys and relationships

Silver contains:

```text
circuits
races
constructors
drivers
results
sprints
```

### Gold

The Gold layer is designed using **dimensional modeling** to support analytical and reporting requirements.

```text
                         dim_races
                             |
                             v
dim_drivers ------ fact_session_results ------ dim_constructors
```

The model contains:

- `dim_drivers`
- `dim_constructors`
- `dim_races`
- `fact_session_results`

---

## Fact Table Design

`fact_session_results` represents:

> One driver's result for a particular Formula 1 Race or Sprint session.

Race results and Sprint results have compatible schemas and grain, so they are combined into a single fact table.

```text
results -----\
              \
               ---> fact_session_results
              /
sprints -----/
```

A `session_type` column identifies whether the record belongs to:

```text
Race
Sprint
```

The two datasets are combined using `unionByName()`.

The fact also contains derived analytical flags:

```text
is_win      -> final_position = 1

is_podium   -> final_position between 1 and 3

has_points  -> points > 0
```

These make common analytical queries easier and more consistent.

---

## Silver to Gold Mapping

```text
Silver                        Gold
------------------------------------------------

drivers -------------------> dim_drivers

constructors --------------> dim_constructors

races + circuit context ---> dim_races

results --------\
                 \
                  ---> fact_session_results
                 /
sprints --------/
```

---

## Analytics

Analytics views are created on top of the Gold model to support:

- Driver standings by season
- Constructor standings by season
- Driver performance across seasons
- Constructor performance across seasons
- Dominant drivers over time
- Dominant constructors over time

Example:

```text
fact_session_results
        |
        v
Aggregate Points
        |
        v
Rank within Season
        |
        v
Driver / Constructor Standings
```

These views are used by the **Databricks Dashboard** for reporting and visualization.

---

## Delta Lake

Delta Lake is used from the Bronze layer onward to provide reliable table operations.

Key capabilities used by the solution include:

- ACID transactions
- Consistent reads
- Version history
- Time travel
- Updates and deletes
- MERGE support
- Data correction and rollback

Delta stores the actual data in Parquet files and tracks committed table state through the `_delta_log`.

---

## Unity Catalog

Unity Catalog is used to organize and govern the project.

```text
formula1
|
+-- landing
|   +-- files
|       External Volume
|
+-- bronze
|   Delta Tables
|
+-- silver
|   Delta Tables
|
+-- gold
    Dimensions
    Fact Table
    Analytics Views
```

ADLS Gen2 provides the physical storage, while Unity Catalog provides the governed namespace for schemas, tables, views and volumes.

---

## Pipeline Orchestration

The end-to-end workflow is orchestrated using **Databricks Jobs**.

```text
Bronze Ingestion
       |
       v
Silver Transformation
       |
       v
Gold Dimensional Model
       |
       v
Analytics Views
       |
       v
Dashboard
```

The pipeline supports:

- Task dependencies
- Scheduled execution
- Retry handling
- Monitoring
- Failure alerts
- Graceful completion when no new race data is available

The target schedule is **every Sunday at 10 PM**.

---

## Processing Strategy

The project initially uses full-load processing and is designed to evolve toward incremental processing using Delta Lake capabilities such as:

```text
MERGE
UPDATE
DELETE
```

---

## Key Engineering Highlights

- End-to-end Azure Databricks Lakehouse pipeline
- Multi-format CSV and JSON ingestion
- Explicit schema enforcement
- Audit metadata and source traceability
- Delta Lake storage
- Data cleansing and standardization
- Nested JSON flattening
- Null-key validation and deduplication
- Unity Catalog-based governance
- Requirement-driven dimensional modeling
- Star Schema design
- Unified Race and Sprint fact table
- Derived analytical indicators
- Driver and constructor standings
- Historical performance analysis
- Analytics views and dashboard
- Databricks Jobs orchestration

---

## Final Data Flow

```text
Source Data
    |
    v
Landing
    |
    v
Bronze
    |
    v
Silver
    |
    v
Gold
    |
    v
Analytics Views
    |
    v
Databricks Dashboard
```

---

## Project Objective

The project demonstrates the complete data engineering lifecycle using Azure Databricks:

```text
Ingestion
   ->
Transformation
   ->
Data Quality
   ->
Dimensional Modeling
   ->
Analytics
   ->
Visualization
```