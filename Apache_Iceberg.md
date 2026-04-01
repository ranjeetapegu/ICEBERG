# Apache Iceberg Tables — Complete Guide

---

## 1. What is Apache Iceberg?

Apache Iceberg is an **open table format** for large analytic datasets stored in cloud object storage (S3, GCS, Azure Data Lake Storage).

> Think of it as a "smart layer" that sits between your data files and your query engines.

It is **not** a database or query engine. It is a specification that defines how:
- Table **metadata** (schema, partitions, snapshots) is stored
- Data **files** (Parquet, ORC, Avro) are tracked and organized

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Metadata layer** | JSON manifests tracking schema, partitions, and snapshots |
| **Data files** | Actual data stored as Parquet/ORC in object storage |
| **ACID transactions** | Snapshot isolation — readers see consistent state during writes |
| **Time travel** | Query data as it existed at any previous snapshot |
| **Schema evolution** | Add, drop, rename columns without rewriting data |
| **Hidden partitioning** | Partition by transforms (year, month, bucket) invisible to queries |

### How Iceberg Works — Layered Architecture

```
Your Query Engine (Snowflake / Spark / Athena)
        ↓
  Iceberg Catalog  ← stores metadata (table name → metadata location)
        ↓
  Metadata Files   ← JSON files describing snapshots, partitions, schema
        ↓
  Data Files       ← Parquet/ORC files in S3 / GCS / ADLS
```

---

## 2. Why Was Iceberg Created?

Before Iceberg, data lakes had serious problems:

| Problem | Description |
|---------|-------------|
| No ACID support | Two jobs writing at the same time could corrupt data |
| Schema changes broke things | Adding a column required rewriting the entire dataset |
| Slow partition listing | Query engines had to scan all S3 paths to find data |
| No time travel | Once data was overwritten, it was gone |
| Multi-engine inconsistency | Hive Metastore was unreliable across different engines |

Iceberg solves all of these.

---

## 3. Iceberg in Snowflake

Snowflake supports Iceberg in two modes:

### Mode 1 — Snowflake-Managed Iceberg Tables
- Snowflake owns the catalog and writes data to **your** external volume (S3/GCS/ADLS)
- Full Snowflake feature set (clustering, search optimization, etc.)
- Data files are in open Parquet format — other engines can read them

### Mode 2 — Externally Managed Iceberg Tables
- Table is owned by an external catalog (Glue, Unity Catalog, Polaris)
- Snowflake reads (and optionally writes) it as an Iceberg table
- You must configure a **Catalog Integration** and **External Volume**

### Key Snowflake Components

| Component | Purpose |
|-----------|---------|
| **External Volume** | Credentials + path for Snowflake to access your S3/GCS/ADLS bucket |
| **Catalog Integration** | Connection to external catalog (Glue, Unity, Polaris) |
| **Catalog-Linked Database (CLD)** | Auto-discovers and syncs all Iceberg tables from an external catalog |
| **Auto-refresh** | Snowflake polls the catalog for new snapshots and updates metadata |
| **ALLOW_WRITES** | Lets Snowflake write back to externally managed Iceberg tables |

### Supported External Catalogs in Snowflake

- AWS Glue Data Catalog
- Databricks Unity Catalog
- Polaris / OpenCatalog (Snowflake's own open-source catalog)

### Typical Snowflake + Iceberg Flow

```
Spark writes data to S3  →  Glue Catalog updated
        ↓
Snowflake Catalog Integration polls Glue
        ↓
Snowflake auto-refreshes Iceberg table metadata
        ↓
Analysts query in Snowflake with full SQL
```

---

## 4. Iceberg in AWS

AWS has native, deep support for Iceberg across its analytics stack.

### AWS Services and Their Iceberg Role

| Service | Role |
|---------|------|
| **Amazon S3** | Stores Parquet data files + Iceberg metadata JSON files |
| **AWS Glue Data Catalog** | Acts as the Iceberg catalog — stores table definitions, partitions, snapshots |
| **Amazon Athena** | Serverless SQL engine to query Iceberg tables directly |
| **Amazon EMR** | Run Spark / Flink jobs that read and write Iceberg at scale |
| **AWS Lake Formation** | Governs access control to Glue-cataloged Iceberg tables |
| **Amazon Redshift** | Can query Iceberg via Redshift Spectrum (read-only from Glue) |

### Typical AWS Iceberg Architecture

```
┌─────────────────────────────────────────┐
│              AWS Glue Catalog            │  ← Table metadata, schema, partitions
└────────────────────┬────────────────────┘
                     │
       ┌─────────────┼──────────────┐
       ↓             ↓              ↓
  Amazon Athena   Amazon EMR    Snowflake
  (SQL queries)  (Spark jobs)  (BI / Analytics)
       │             │
       └─────────────┘
                ↓
          Amazon S3
    (Parquet files + Iceberg metadata)
```

### AWS Iceberg Key Features
- **Glue Crawler**: Automatically discovers Iceberg tables in S3 and registers them in Glue
- **Lake Formation**: Fine-grained row/column access control on Iceberg tables
- **S3 Versioning**: Works alongside Iceberg snapshots for data recovery
- **Athena ACID**: Athena supports full INSERT, UPDATE, DELETE on Iceberg tables

---

## 5. When to USE Iceberg

Use Iceberg when any of these apply:

| Scenario | Why Iceberg Fits |
|----------|-----------------|
| **Multi-engine access** | Spark, Athena, Snowflake, and Trino all need to read the same data |
| **Avoid vendor lock-in** | You want your data in open format, not Snowflake-proprietary storage |
| **Large-scale CDC pipelines** | Change Data Capture with upserts and deletes at scale |
| **Long-term data retention** | Store petabytes cheaply on S3 with full query capability |
| **Regulatory / data ownership** | You must own and control the raw data files |
| **Frequent schema changes** | Add/drop/rename columns on massive datasets without rewrites |
| **Audit requirements** | Time travel lets you query historical states of the data |

---

## 6. When NOT to Use Iceberg

Avoid Iceberg when:

| Scenario | Why Iceberg is Overkill or Wrong |
|----------|----------------------------------|
| **Pure Snowflake shop** | Native Snowflake tables are faster, cheaper, and simpler — no IAM/catalog complexity |
| **High-frequency small writes** | Iceberg commits are not optimized for many tiny transactions per second |
| **Real-time / sub-second latency** | Auto-refresh has polling lag; not suitable for true streaming analytics |
| **Small teams / simple use cases** | Catalog integrations, IAM roles, external volumes = significant operational overhead |
| **Snowflake Marketplace data** | Marketplace shared tables cannot be Iceberg |
| **Streaming inserts** | Native Snowflake tables handle streaming better; Iceberg needs batch commits |
| **Heavy Snowflake-specific features** | Search optimization, automatic clustering work best on native tables |

---

## 7. Iceberg vs Competitors (Table Formats)

| Format | Maintained By | Strengths | Weaknesses |
|--------|--------------|-----------|------------|
| **Apache Iceberg** | Apache (Netflix/Apple origins) | Most vendor-neutral, widest adoption, strong AWS/Snowflake support | More complex catalog setup |
| **Delta Lake** | Linux Foundation (Databricks origins) | Dominant in Databricks/Spark, mature tooling, simple setup | Less neutral — Databricks-centric |
| **Apache Hudi** | Apache (Uber origins) | Best for CDC/upsert-heavy workloads | Smaller ecosystem, less adoption |

### Market Trend
> Iceberg is winning the open format war. AWS, Snowflake, Google, Azure, and Databricks all support it natively. Delta Lake remains dominant inside Databricks environments.

---

## 8. Iceberg Catalog Competitors

The catalog is who stores the table metadata. Different vendors offer different catalogs:

| Catalog | Vendor | Best For |
|---------|--------|---------|
| **AWS Glue Data Catalog** | Amazon | AWS-native stacks |
| **Databricks Unity Catalog** | Databricks | Spark/Databricks-heavy shops |
| **Polaris / OpenCatalog** | Snowflake | Snowflake-centric or neutral setups |
| **Project Nessie** | Dremio | Git-like branching for data versioning |
| **Hive Metastore** | Apache | Legacy Hadoop/Spark environments |
| **Tabular** | Tabular (startup) | Managed Iceberg catalog service |

---

## 9. Query Engines That Support Iceberg

Any of these can read and write Iceberg tables:

- **Cloud Warehouses**: Snowflake, BigQuery (via manifest), Redshift Spectrum
- **Open Source**: Apache Spark, Apache Flink, Trino, Presto, DuckDB
- **Managed Services**: Amazon Athena, Amazon EMR, Databricks, Dremio, Starburst

---

## 10. Quick Reference Summary

| Question | Answer |
|----------|--------|
| What is it? | Open table format for cloud data lakehouses |
| What problem does it solve? | ACID, schema evolution, multi-engine access on object storage |
| Snowflake role | Read/write Iceberg tables; integrates with Glue, Unity, Polaris catalogs |
| AWS role | Glue catalog + S3 storage + Athena/EMR as query engines |
| Use when | Multi-engine, no lock-in, CDC, large-scale analytics, data ownership |
| Avoid when | Pure Snowflake, real-time, simple/small use cases, high-frequency writes |
| Main format competitors | Delta Lake (Databricks), Apache Hudi |
| Main catalog competitors | Glue, Unity Catalog, Polaris, Nessie |

---

## 11. Key Terms Glossary

| Term | Definition |
|------|-----------|
| **Table format** | Specification for how data files and metadata are organized |
| **Snapshot** | Immutable point-in-time state of a table |
| **Manifest file** | JSON file listing data files in a snapshot |
| **External volume** | Snowflake object holding credentials to access cloud storage |
| **Catalog integration** | Snowflake connection to an external Iceberg catalog |
| **CLD** | Catalog-Linked Database — auto-syncs all tables from an external catalog |
| **Auto-refresh** | Automatic sync of Iceberg metadata from catalog to Snowflake |
| **Parquet** | Columnar file format used to store Iceberg table data |
| **Hidden partitioning** | Iceberg feature that partitions data without users needing to know about it |
| **Time travel** | Ability to query data as it was at a previous snapshot |

---

*Reference: [Snowflake Iceberg Docs](https://docs.snowflake.com/user-guide/tables-iceberg) | [Apache Iceberg Spec](https://iceberg.apache.org/spec/)*
