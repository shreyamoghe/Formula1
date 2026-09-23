# Formula 1 Data Engineering Project

## Project Overview

This project implements an end-to-end **Formula 1 Data Lakehouse** on Azure Databricks for processing historical and recent Formula 1 racing data.

The solution ingests raw Formula 1 datasets in multiple file formats, processes them through a **Landing → Bronze → Silver → Gold** architecture, and produces analytics-ready datasets for analyzing driver and constructor performance across seasons.

The pipeline is initially implemented as a full load and is designed to evolve into an incremental processing solution.

---

## Architecture

The project follows Medallion Architecture principles with an additional Landing layer:

**Landing → Bronze → Silver → Gold**

### Landing
- Raw source files stored in **Azure Data Lake Storage Gen2 (ADLS)**
- Exposed to Databricks through a **Unity Catalog external volume**
- Acts as the controlled entry point for incoming source data

### Bronze
- Raw source data ingested into **Delta tables**
- Explicit schemas applied for predictable data types
- Audit metadata added:
  - Ingestion timestamp
  - Source file name
- Source structure retained for traceability

### Silver
- Data cleaned and standardized
- Consistent column naming applied
- Nested JSON structures flattened
- Null business keys and duplicate records handled
- Produces trusted datasets for downstream modeling

### Gold
- Analytics-ready **dimensional data model**
- Driver, constructor, and race dimensions
- Race and sprint result facts
- Aggregated datasets for:
  - Driver standings
  - Constructor standings
  - Historical performance analysis

---

## Data Source

The project uses Formula 1 data based on the open-source **Jolpica F1 dataset/API**, following the relational-style Ergast data model.

Six primary datasets are processed:

| Dataset | Description | Source Format |
|---|---|---|
| Circuits | Circuit and location information | CSV |
| Races | Race information by season and round | CSV |
| Constructors | Formula 1 teams | JSON |
| Drivers | Driver information including nested attributes | JSON |
| Results | Driver race results and points | Multi-file JSON |
| Sprints | Sprint race results | Multi-line, multi-file JSON |

The different source structures provide ingestion scenarios covering CSV, JSON, nested JSON, multi-line JSON, and multi-file datasets.

---

## Technology Stack

- **Azure Databricks**
- **Apache Spark / PySpark**
- **Azure Data Lake Storage Gen2**
- **Delta Lake**
- **Unity Catalog**
- **Databricks Jobs**
- **SQL**

---

## Data Governance & Storage

Unity Catalog is used to organize and govern the project data.

```text
formula1
│
├── landing
│   └── files (External Volume)
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
