# Formula 1 Data Engineering Project

## Project Overview

This project implements an end-to-end **Formula 1 Data Lakehouse** using Azure Databricks.

The solution ingests Formula 1 racing data from multiple source formats, processes it through a **Landing → Bronze → Silver → Gold** architecture, and builds analytics-ready datasets for analyzing driver and constructor performance across seasons.

The pipeline is initially implemented using full-load processing and is designed to support incremental data processing as the solution evolves.

---

## Solution Architecture

The project follows Medallion Architecture principles with an additional Landing layer.

```text
Formula 1 Source Data
        │
        ▼
┌─────────────────────┐
│      ADLS Gen2      │
│    Landing Layer    │
│  Raw CSV/JSON Files │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│       Bronze        │
│    Delta Tables     │
│  Schema Enforcement │
│   Audit Metadata    │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│       Silver        │
│ Clean & Standardize │
│ Flatten / DQ Checks │
│    Deduplication    │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│        Gold         │
│ Dimensional Model   │
│ Facts / Dimensions  │
│    Aggregations     │
└──────────┬──────────┘
           │
           ▼
 Driver & Constructor
       Analytics
```

### Landing Layer

Raw Formula 1 source files are stored in **Azure Data Lake Storage Gen2 (ADLS)**.

The landing location is exposed to Databricks using a **Unity Catalog external volume**, providing governed access to the source files before processing.

### Bronze Layer

The Bronze layer preserves the source data with minimal transformation while providing traceability and reliability.

- Explicit schemas and appropriate data types
- Source file name for traceability
- Ingestion timestamp for auditing
- Data stored as Delta tables

### Silver Layer

The Silver layer produces clean and consistent datasets for downstream processing.

- Standardized column naming
- Nested JSON structures flattened
- Unnecessary attributes removed
- Null business keys handled
- Duplicate records removed
- Business keys preserved across transformations

### Gold Layer

The Gold layer provides analytics-ready datasets using **dimensional modeling**.

It contains dimensions for entities such as drivers, constructors, and races, along with fact tables for race and sprint results.

Aggregated datasets are built for driver and constructor standings and historical performance analysis.

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
| Results | Driver results, positions, laps and points | Multi-file JSON |
| Sprints | Sprint race results | Multi-line, multi-file JSON |

This provides ingestion scenarios involving CSV, JSON, nested JSON, multi-line JSON, and datasets distributed across multiple files.

---

## Technology Stack

- Azure Databricks
- Apache Spark / PySpark
- Azure Data Lake Storage Gen2
- Delta Lake
- Unity Catalog
- Databricks Jobs
- SQL

---

## Unity Catalog & Data Organization

Unity Catalog is used for centralized organization and governance of the project data.

```text
formula1
│
├── landing
│   └── files              # External Volume
│
├── bronze
│   └── Delta Tables
│
├── silver
│   └── Delta Tables
│
└── gold
    ├── Dimensions
    ├── Facts
    └── Analytical Datasets
```

The Landing schema exposes the ADLS source files through an external volume, while Bronze, Silver, and Gold contain the managed tables created during processing.

---

## Data Model

The source data is organized around the following key Formula 1 entities:

- **Circuit** — identified by `circuitId`
- **Race** — identified by `season` and `round`
- **Driver** — identified by `driverId`
- **Constructor** — identified by `constructorId`
- **Race Result** — associated with season, round, driver, and constructor
- **Sprint Result** — associated with season, round, driver, and constructor

The Gold layer transforms these datasets into a dimensional model suitable for analytical workloads.

---

## Data Ingestion

PySpark DataFrame APIs are used to implement the ingestion pipeline.

The ingestion process:

1. Reads source files from the Unity Catalog landing volume.
2. Applies explicit schemas and appropriate data types.
3. Adds ingestion timestamp and source file metadata.
4. Writes the processed data into Bronze Delta tables.

A common ingestion pattern is used across the six datasets to minimize hard-coded and repetitive processing logic.

---

## Data Quality & Transformation

The Silver transformation layer applies data quality and standardization rules including:

- Null-key validation
- Duplicate removal
- Consistent naming conventions
- Nested JSON flattening
- Data type standardization
- Preservation of business keys

This produces trusted datasets for dimensional modeling and analytical processing.

---

## Analytics

The Gold layer supports analysis including:

- Driver standings by season
- Constructor standings by season
- Driver performance across historical and recent seasons
- Constructor performance across seasons
- Identification of dominant drivers and constructors over time

---

## Pipeline Orchestration

The end-to-end pipeline is orchestrated using **Databricks Jobs**.

```text
Bronze Ingestion
       ↓
Silver Transformation
       ↓
Gold Dimensional Modeling
       ↓
Gold Aggregations
```

Databricks Jobs provides:

- Task dependencies
- Scheduled execution
- Retry handling
- Pipeline monitoring
- Failure alerting

The pipeline is designed to run on a scheduled basis and complete successfully even when no new race data is available.

---

## Delta Lake Capabilities

Delta Lake is used throughout the processing layers to provide reliable table operations.

The solution is designed to leverage capabilities such as:

- ACID transactions
- `MERGE` for incremental processing
- Updates and deletes
- Table version history
- Time travel
- Data correction and rollback

The initial implementation processes the complete dataset, with incremental processing introduced as the pipeline evolves.

---

## Project Objective

The project demonstrates how a modern Azure Databricks Lakehouse can be used to build a governed and reliable data engineering pipeline that handles heterogeneous source data and transforms it into analytics-ready datasets for downstream reporting and analysis.
