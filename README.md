# Formula 1 Data Engineering Project

## Project Overview

This project implements an end-to-end **Formula 1 Data Lakehouse using Azure Databricks**.

The solution ingests Formula 1 racing data from multiple source formats, processes it through a **Landing → Bronze → Silver → Gold** architecture, and builds analytics-ready datasets for analyzing driver and constructor performance across seasons.

The pipeline was initially implemented using full-refresh processing and later enhanced into a **batch-based incremental pipeline** that processes only newly arrived race batches.

---

## Solution Architecture

```text
Formula 1 Source Data
        |
        v
Azure Data Lake Storage Gen2
        |
        v
Landing Layer
Batch-based Raw Files
        |
        v
Bronze Layer
Delta Tables
Append + Audit Metadata
        |
        v
Silver Layer
Clean + Standardize
Data Quality + MERGE
        |
        v
Gold Layer
Dimensional Model
Incremental MERGE
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

Six primary datasets are processed:

| Dataset | Description | Format |
|---|---|---|
| Circuits | Circuit and geographical information | CSV |
| Races | Race details by season and round | CSV |
| Constructors | Formula 1 teams | JSON |
| Drivers | Driver information including nested attributes | JSON |
| Results | Driver race results, positions, laps and points | Multi-file JSON |
| Sprints | Sprint race results | Multi-line, multi-file JSON |

This provides ingestion scenarios involving CSV, JSON, nested JSON, multi-line JSON and multi-file datasets.

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

## Lakehouse Layers

### Landing Layer

Raw Formula 1 source files are stored in **Azure Data Lake Storage Gen2 (ADLS Gen2)**.

The Landing location is exposed to Databricks using a **Unity Catalog External Volume**.

For incremental processing, data is organized into batch folders such as:

```text
landing/
├── 2025-01/
├── 2025-02/
├── 2025-03/
└── ...
```

Each `batch_id` represents a Formula 1 season and race round.

---

### Bronze Layer

The Bronze layer preserves source data with minimal transformation while maintaining traceability.

Key processing includes:

- Explicit schemas and appropriate data types
- Source file name
- Ingestion timestamp
- Batch ID
- Delta table storage
- Append-based processing to preserve batch history

Bronze acts as the historical record of data received from the source.

---

### Silver Layer

The Silver layer produces clean and trusted datasets.

Processing includes:

- Standardized column names and data types
- Nested JSON flattening
- Removal of unnecessary attributes
- Null business-key validation
- Duplicate removal
- Preservation of business keys and relationships
- Incremental `MERGE` for inserts and updates

Silver contains:

```text
circuits
races
constructors
drivers
results
sprints
```

---

## Gold Layer

The Gold layer is designed using **dimensional modeling** based on analytical requirements.

Instead of exposing the Silver tables directly to reporting users, the Gold layer provides a Star Schema:

```text
                         dim_races
                             |
                             v
dim_drivers ------ fact_session_results ------ dim_constructors
```

### Dimensions

- `dim_drivers`
- `dim_constructors`
- `dim_races`

### Fact Table

`fact_session_results` contains both Race and Sprint results.

Race and Sprint datasets have compatible schemas and the same analytical grain, so they are combined into a single fact table.

A `session_type` column identifies whether a record represents:

```text
Race
Sprint
```

The two datasets are combined using `unionByName()`.

The fact table also contains derived analytical fields:

```text
is_win      -> final_position = 1
is_podium   -> final_position between 1 and 3
has_points  -> points > 0
```

These simplify common analytical queries such as wins, podiums and points analysis.

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

## Incremental Processing

The first version of the pipeline used a **full refresh**, where all historical data was reprocessed during every execution.

The pipeline was later enhanced to use **batch-based incremental processing**.

### Batch Design

Each race batch is stored in a separate Landing folder:

```text
2025-01
2025-02
2025-03
...
```

`2025-01` acts as the initial cutover batch and contains historical data up to that point. Subsequent folders contain data for the newly completed race.

### Source Data Types

The datasets follow two ingestion patterns:

**Snapshot-style data**
- Circuits
- Races
- Constructors
- Drivers

**Change data**
- Results
- Sprints

Bronze appends all incoming batches to preserve history.

Silver and Gold use Delta `MERGE` for both snapshot and change datasets so the pipeline can safely handle:

- New records
- Updated records
- Duplicate prevention
- Batch reruns
- Out-of-order batch processing

---

## Batch Control & Orchestration

A Delta control table tracks the processing status of each batch.

```text
batch_control
-----------------------------------------
batch_id
status
created_timestamp
updated_timestamp
```

Example:

```text
batch_id | status
---------|----------
2025-01  | completed
2025-02  | completed
2025-03  | in_progress
```

The orchestration flow is:

```text
Landing Batch Folders
        |
        v
Identify Next Batch
        |
        v
Mark Batch as in_progress
        |
        v
Process Bronze
        |
        v
Process Silver
        |
        v
Process Gold
        |
        v
Mark Batch as completed
```

The pipeline compares batch folders available in Landing with the batches already tracked in `batch_control`.

The earliest unprocessed batch is selected and passed to downstream Databricks Job tasks using:

```text
p_batch_id
has_batch
```

If no new batch is available, `has_batch` is set to `false`, allowing the pipeline to complete without unnecessarily executing downstream processing.

---

## Pipeline Orchestration

The end-to-end workflow is orchestrated using **Databricks Jobs**.

```text
Identify Next Batch
        |
        v
Create New Batch
        |
        v
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
Complete Batch
```

Databricks Jobs provides:

- Task dependencies
- Parameter passing between tasks
- Scheduled execution
- Retry handling
- Pipeline monitoring
- Failure alerts

The target schedule is **every Sunday at 10 PM**.

---

## Analytics

Analytics views are created on top of the Gold dimensional model to support:

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

The analytics views are consumed by an interactive **Databricks Dashboard**.

---

## Delta Lake Capabilities

Delta Lake is used throughout the processing layers to provide reliable table operations.

Key capabilities used include:

- ACID transactions
- Incremental `MERGE`
- Updates and deletes
- Consistent reads
- Table version history
- Time travel
- Data correction and rollback

Delta stores the underlying data in Parquet files and tracks committed table state using the `_delta_log`.

---

## Unity Catalog & Data Organization

Unity Catalog is used for centralized organization and governance.

For the incremental pipeline, the project uses:

```text
formula1_incr
|
├── landing
│   └── files              # External Volume
│
├── bronze
│   └── Delta Tables
│
├── silver
│   └── Delta Tables
│
├── gold
│   ├── Dimensions
│   ├── fact_session_results
│   └── Analytics Views
│
└── control
    └── batch_control
```

ADLS Gen2 stores the physical data, while Unity Catalog provides the governed namespace for catalogs, schemas, tables, views and volumes.

---

## Key Engineering Highlights

- End-to-end Azure Databricks Lakehouse pipeline
- Landing, Bronze, Silver and Gold architecture
- Multi-format CSV and JSON ingestion
- Explicit schema enforcement
- Audit metadata and source traceability
- Batch-based incremental processing
- Batch control framework
- Parameterized notebook execution
- Bronze append-based history
- Silver and Gold Delta `MERGE`
- Snapshot and change-data handling
- Null-key validation and deduplication
- Nested JSON flattening
- Unity Catalog-based governance
- Requirement-driven dimensional modeling
- Unified Race and Sprint fact table
- Derived analytical indicators
- Driver and constructor standings
- Analytics views and Databricks dashboard
- Databricks Jobs orchestration
- Delta ACID transactions and time travel

---

## Project Objective

The project demonstrates how a modern **Azure Databricks Lakehouse** can be used to build a governed, reliable and scalable data engineering pipeline covering the complete data lifecycle:

```text
Ingestion
   ->
Incremental Processing
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
