# Apache Iceberg: A Comprehensive Guide to the Data Lakehouse

> **Based on a video series featuring Alex Merced (Developer Advocate at Dremio), hosted by Mustafa**
> This guide synthesizes both the introductory presentation and the hands-on demo into a single, complete reference.

---

## Table of Contents

1. [Introduction & Background](#1-introduction--background)
2. [What is a Table Format?](#2-what-is-a-table-format)
3. [How Traditional Table Formats Work (Hive)](#3-how-traditional-table-formats-work-hive)
4. [What is Apache Iceberg?](#4-what-is-apache-iceberg)
5. [How Apache Iceberg Works (Architecture Deep Dive)](#5-how-apache-iceberg-works-architecture-deep-dive)
   - [The Catalog](#51-the-catalog)
   - [The Metadata File](#52-the-metadata-file)
   - [The Manifest List](#53-the-manifest-list)
   - [The Manifest Files](#54-the-manifest-files)
   - [The Read Path](#55-the-read-path)
   - [The Write Path](#56-the-write-path)
   - [Snapshots Explained](#57-snapshots-explained)
6. [Benefits of Apache Iceberg](#6-benefits-of-apache-iceberg)
   - [Efficient Small Updates](#61-efficient-small-updates)
   - [Snapshot Isolation & Time Travel](#62-snapshot-isolation--time-travel)
   - [Table & Partition Evolution](#63-table--partition-evolution)
   - [Hidden Partitioning](#64-hidden-partitioning)
   - [Compatibility Mode & File Distribution](#65-compatibility-mode--file-distribution)
   - [Performance Improvements](#66-performance-improvements)
   - [Schema Evolution](#67-schema-evolution)
   - [Cost Savings](#68-cost-savings)
   - [Wide Ecosystem](#69-wide-ecosystem)
7. [Iceberg Catalog Options](#7-iceberg-catalog-options)
   - [Project Nessie (Git-like Catalog)](#71-project-nessie-git-like-catalog)
   - [Hive Metastore](#72-hive-metastore)
   - [AWS Glue](#73-aws-glue)
   - [Hadoop / File System Catalog](#74-hadoop--file-system-catalog)
   - [Other Catalogs](#75-other-catalogs)
   - [Choosing the Right Catalog](#76-choosing-the-right-catalog)
   - [Migrating Between Catalogs](#77-migrating-between-catalogs)
8. [Apache Iceberg File Layout](#8-apache-iceberg-file-layout)
9. [Dremio & Apache Iceberg](#9-dremio--apache-iceberg)
   - [What is Dremio?](#91-what-is-dremio)
   - [Key SQL Commands in Dremio](#92-key-sql-commands-in-dremio)
   - [Reflections (Query Acceleration)](#93-reflections-query-acceleration)
   - [Caching](#94-caching)
   - [Dremio Arctic Catalogs](#95-dremio-arctic-catalogs)
   - [Data Lakehouse Architecture with Dremio](#96-data-lakehouse-architecture-with-dremio)
10. [Hands-On Demo: Spinning Up a Data Lakehouse on Your Laptop](#10-hands-on-demo-spinning-up-a-data-lakehouse-on-your-laptop)
    - [Prerequisites](#101-prerequisites)
    - [The Docker Compose Stack](#102-the-docker-compose-stack)
    - [Step 1: Start the Environment](#103-step-1-start-the-environment)
    - [Step 2: Configure MinIO (Object Storage)](#104-step-2-configure-minio-object-storage)
    - [Step 3: Set Up Dremio](#105-step-3-set-up-dremio)
    - [Step 4: Connect the Nessie Catalog](#106-step-4-connect-the-nessie-catalog)
    - [Step 5: Create an Iceberg Table](#107-step-5-create-an-iceberg-table)
    - [Step 6: Insert Data & Observe Snapshots](#108-step-6-insert-data--observe-snapshots)
    - [Step 7: Inspect the Metadata in MinIO](#109-step-7-inspect-the-metadata-in-minio)
    - [Step 8: Ingest Data from External Sources](#1010-step-8-ingest-data-from-external-sources)
    - [Step 9: Organize Data with the Wiki & Spaces](#1011-step-9-organize-data-with-the-wiki--spaces)
11. [Table Optimization & Maintenance](#11-table-optimization--maintenance)
    - [Compaction (OPTIMIZE)](#111-compaction-optimize)
    - [Snapshot Cleanup (VACUUM)](#112-snapshot-cleanup-vacuum)
12. [On-Premises & Cloud-Agnostic Use Cases](#12-on-premises--cloud-agnostic-use-cases)
13. [Iceberg vs. Proprietary Data Warehouses](#13-iceberg-vs-proprietary-data-warehouses)
14. [When to Use Apache Iceberg](#14-when-to-use-apache-iceberg)
15. [Connecting BI Tools (Power BI, Tableau)](#15-connecting-bi-tools-power-bi-tableau)
16. [Additional Resources](#16-additional-resources)

---

## 1. Introduction & Background

The data world has a fundamental problem: **data is moved too many times**.

In a traditional architecture, data flows like this:

```
Source Systems → Data Lake (raw) → Data Warehouse (processed) → BI / AI / ML
```

Each hop has a cost:
- **Time** — moving data delays insights to dashboards, reports, and models.
- **Storage** — every copy of the data costs money.
- **Compute** — every transformation pipeline costs compute.

The dream is to **narrow the gap** — to do the work of a data warehouse *directly on the data lake*, without making multiple copies. This concept is called the **Data Lakehouse**.

**Apache Iceberg** is the open standard that makes the data lakehouse possible. It enables data lake storage (like S3, HDFS, or MinIO) to behave like a high-performance data warehouse — with ACID transactions, time travel, schema evolution, and fast analytical queries.

---

## 2. What is a Table Format?

To understand Apache Iceberg, you first need to understand what a **table format** is and why it exists.

A data lake is essentially a collection of files (typically Parquet files) stored on distributed object storage (like Amazon S3, Azure Data Lake Storage, or MinIO). Raw files, however, have no concept of a "table." You can't run `SELECT * FROM my_data` on a folder of Parquet files without some layer of organization.

A **table format** creates a **metadata layer** that:

- Groups individual data files into logical tables.
- Tracks the schema of those tables.
- Tracks partitioning information.
- Enables ACID transactions (Atomicity, Consistency, Isolation, Durability).
- Enables **time travel** (querying historical versions of the data).
- Gives query engines the information they need to **skip files** they don't need to read.

Without a table format, the data lake is just a dump of files. With a table format, it becomes a **structured, queryable, ACID-compliant data store**.

The key components you need for a data lakehouse are:

| Component | Purpose | Example Technologies |
|---|---|---|
| Object Storage | Store the actual data files | S3, ADLS, MinIO, HDFS |
| File Format | Optimized columnar format for analytics | Apache Parquet |
| Table Format | Metadata layer that makes files look like tables | Apache Iceberg |
| Catalog | Directory of all tables and where they live | Nessie, Hive Metastore, AWS Glue |
| Query Engine | Executes SQL queries on the tables | Dremio, Apache Spark, Trino, Presto, Flink |

---

## 3. How Traditional Table Formats Work (Hive)

Before modern table formats like Iceberg, the dominant solution was the **Apache Hive** table format.

Hive was created to make analytical queries on Hadoop easier. Instead of writing complex Java MapReduce jobs, Hive let users write SQL. But to write SQL, you need tables. Hive's answer was a **folder-based approach**:

- A folder called `table_a/` contains all the data for Table A.
- A folder called `table_b/` contains all the data for Table B.
- Within each table folder, **partition folders** divide the data — e.g., `table_a/month=2024-01/`, `table_a/month=2024-02/`, etc.
- A **Hive Metastore** (a relational database, typically MySQL or PostgreSQL) tracked which folders belonged to which tables.

### Problems with the Hive Approach

1. **Expensive updates**: To update even a single record in a partition, Hive had to rewrite the *entire partition folder*. This meant creating a new folder, rewriting all the data files, and swapping the folder reference in the metastore.

2. **Costly partition column management**: If you wanted to partition by month, you had to create a *separate `month` column* in your data and populate it yourself. Analysts then had to remember to filter by this extra column to avoid full table scans.

3. **No file-level statistics**: Since Hive tracks folders, not individual files, it can't store metadata about the contents of each individual file. It can't skip files that don't contain relevant data.

4. **Consistency risks with multiple writers**: Without strong file-level transaction semantics, concurrent writes could lead to data corruption or inconsistency.

Modern table formats like Apache Iceberg solve all of these problems by switching to a **file-list approach** — tracking individual files, not folders.

---

## 4. What is Apache Iceberg?

Apache Iceberg is an **open table format standard** for large analytic datasets. Its primary goal is to define *how metadata is written and organized* so that any tool that follows the standard can read any Iceberg table written by any other tool.

**Key clarifications about what Iceberg is and isn't:**

| It IS... | It IS NOT... |
|---|---|
| A metadata standard / specification | A storage engine (data lives in S3, HDFS, etc.) |
| A set of Java/Python/Go/Rust libraries | An execution engine (queries run in Spark, Dremio, Trino, etc.) |
| A format for tracking table history | A service you need to deploy and run |
| A standard enabling tool interoperability | A database |

Because Iceberg is just a *standard*, you don't need to "run Iceberg." You configure your query engine (Spark, Dremio, Trino) to read and write Iceberg metadata, and it handles the rest.

```
You write SQL in Dremio / Spark / Trino
         ↓
Query engine reads/writes Iceberg metadata
         ↓
Metadata + data files live in your object storage (S3, MinIO, etc.)
```

The power of this model is **portability**: the data is always yours. If you stop using one query engine, you simply configure a different engine to point at the same catalog, and all your tables are immediately available — no migration, no data rewriting.

---

## 5. How Apache Iceberg Works (Architecture Deep Dive)

The Apache Iceberg architecture is a layered hierarchy of metadata files. Understanding this hierarchy is key to understanding Iceberg's power.

```
Catalog
   └── Metadata File (.json)
            └── Manifest List (.avro)  [one per snapshot]
                     └── Manifest File (.avro)  [one per group of data files]
                              └── Data Files (.parquet)
```

### 5.1 The Catalog

The **catalog** is the top-level directory of all Iceberg tables. Think of it like a phone book: it doesn't contain the data itself, but it knows the *name* of every table and the *location* of its current metadata file.

When a query engine like Dremio or Spark wants to query a table, it first goes to the catalog and asks: *"Where is the current metadata file for `table_one`?"* The catalog responds with a path in the file system (e.g., an S3 URI).

Popular catalog implementations include:
- **Project Nessie** (git-like versioning)
- **Hive Metastore** (existing on-prem infrastructure)
- **AWS Glue** (managed AWS service)
- **REST Catalog** (open standard REST API)
- **JDBC Catalog** (any relational database)

### 5.2 The Metadata File

The **metadata file** (a JSON file) is the high-level definition of a table at a specific point in time. It contains:

- **Current schema** — all column names and types.
- **Historical schemas** — every past version of the schema (for schema evolution).
- **Current partitioning** — how the table is currently partitioned.
- **Historical partitioning** — all past partitioning schemes (for partition evolution).
- **Current snapshot** — a pointer to the most recent state of the data.
- **All historical snapshots** — pointers to every past state of the data.

Crucially, the metadata file contains *everything you need to reconstruct the table at any point in its history*. Every time the table changes (insert, update, delete, schema change), a **new metadata file is created**. The old one is never overwritten — it's kept for time travel.

After every transaction, the catalog is updated to point to the newest metadata file.

### 5.3 The Manifest List

Each **snapshot** in the metadata file points to a **manifest list** (an Apache Avro file). The manifest list represents the state of the table at one specific moment in time — one complete snapshot.

The manifest list contains a list of all the **manifest files** that make up that snapshot. Importantly, it also contains **partition-level statistics** for each manifest, such as:
- Which partition values this manifest covers (min/max values).
- How many records this manifest contains.
- How much data (in bytes) this manifest represents.

These partition-level statistics allow the query engine to **prune entire manifests** without reading their contents — a major performance optimization.

### 5.4 The Manifest Files

Each **manifest file** (also an Apache Avro file) tracks a group of individual **data files** (Parquet files). For each data file, the manifest stores:

- The file's location (path in object storage).
- The file's partition values.
- **Column-level statistics**: min value, max value, null count, and record count for every column.

These column-level statistics allow the query engine to **prune individual data files** — skipping files that definitely don't contain data matching the query's filter predicates.

### 5.5 The Read Path

Here is how a query engine reads an Iceberg table, step by step:

```
1. Query Engine → Catalog
   "Where is the metadata file for table_one?"
   Catalog returns: s3://warehouse/table_one/metadata/v3.metadata.json

2. Query Engine → Metadata File
   Reads the JSON. Identifies the current snapshot.
   The current snapshot points to: s3://warehouse/table_one/metadata/snap-3.avro

3. Query Engine → Manifest List (snap-3.avro)
   Reads the list of manifests. Uses partition statistics to prune manifests
   that cannot contain data matching the query's WHERE clause.
   Remaining manifests to read: [manifest-A.avro, manifest-C.avro]

4. Query Engine → Manifest Files (manifest-A.avro, manifest-C.avro)
   Reads the list of data files. Uses column-level statistics to prune
   individual files that cannot contain matching data.
   Files to actually read: [file-001.parquet, file-007.parquet]

5. Query Engine → Data Files
   Reads only the necessary Parquet files. Query is answered.
```

The result: a table might have **1,000 data files**, but because of Iceberg's metadata pruning, the engine reads only **20**. This is how Iceberg delivers massive query performance improvements on large datasets.

### 5.6 The Write Path

Writing is the reverse of reading — metadata is built from the bottom up:

```
1. Write data files (.parquet) → to object storage
2. Create or update manifest files (.avro) → list the new data files + their column stats
3. Create a new manifest list (.avro) → the new snapshot
4. Create a new metadata file (.json) → points to the new snapshot
5. Update the catalog → points to the new metadata file
```

**Manifest Reuse**: If a manifest file's contents haven't changed (none of its data files were added or removed), the new snapshot can *reuse* the existing manifest. Only manifests that changed need to be rewritten. This is efficient — a small insert only rewrites one or two manifests, not the entire table.

**Immutability**: Iceberg **never deletes or overwrites existing files** during normal operations. Every write creates new files. Old files are preserved for time travel. Cleanup of old files is a separate, explicit operation (VACUUM).

### 5.7 Snapshots Explained

A **snapshot** is a complete, consistent view of the table at one point in time. Every write operation creates a new snapshot:

| Operation | Result |
|---|---|
| `CREATE TABLE` | Snapshot 1 (empty) |
| `INSERT INTO table VALUES (1, 'Alice')` | Snapshot 2 (1 row) |
| `INSERT INTO table VALUES (2, 'Bob')` | Snapshot 3 (2 rows) |
| `UPDATE table SET name='Bobby' WHERE id=2` | Snapshot 4 (2 rows, updated) |
| `DELETE FROM table WHERE id=1` | Snapshot 5 (1 row) |

Each snapshot is independent. You can query any of them with time travel:

```sql
-- Query the table as it was at snapshot 2
SELECT * FROM table_one AS OF VERSION 2;

-- Query the table as it was at a specific timestamp
SELECT * FROM table_one AS OF TIMESTAMP '2024-01-15 10:00:00';
```

---

## 6. Benefits of Apache Iceberg

### 6.1 Efficient Small Updates

In Hive, updating one record in a 10GB partition required rewriting the entire 10GB partition. With Iceberg, only the specific Parquet file containing that record needs to be rewritten. This makes row-level updates and deletes dramatically more efficient.

### 6.2 Snapshot Isolation & Time Travel

Every transaction is isolated in its own snapshot. This means:
- **Concurrent readers** always see a consistent view of the data (the last completed transaction).
- **Time travel** allows you to query any historical version of the table.
- You can **roll back** to a previous snapshot if a bad write is detected.

```sql
-- Roll back the table to a previous snapshot
CALL catalog.system.rollback_to_snapshot('table_one', 12345678);
```

### 6.3 Table & Partition Evolution

You can **change the table's schema or partitioning scheme without rewriting the data**:

```sql
-- Add a column
ALTER TABLE table_one ADD COLUMN email STRING;

-- Change partitioning from monthly to daily (future data uses new scheme)
ALTER TABLE table_one SET PARTITION BY DAY(event_timestamp);
```

Old data retains its old partitioning; new data uses the new partitioning. The query engine handles both transparently. No full table rewrite required.

### 6.4 Hidden Partitioning

In Hive, partitioning by month required creating a separate `month` column. In Iceberg, partitioning is **hidden** — it's a function applied to an existing column, tracked only in the metadata.

```sql
-- Partition by the month of event_timestamp — no extra column needed
CREATE TABLE events (
  id BIGINT,
  event_timestamp TIMESTAMP,
  name STRING
)
PARTITIONED BY (MONTH(event_timestamp));
```

Benefits:
- **No extra column** in the table schema — data is cleaner.
- **Analysts don't need to know** about the partitioning. They filter by `event_timestamp`, and the engine automatically takes advantage of partitioning.
- **Fewer accidental full table scans** — analysts don't need to remember to filter by a partition column.

Hidden partition transforms available in Iceberg:

| Transform | Description | Example |
|---|---|---|
| `YEAR(col)` | Partition by year | `YEAR(order_date)` |
| `MONTH(col)` | Partition by month | `MONTH(order_date)` |
| `DAY(col)` | Partition by day | `DAY(order_date)` |
| `HOUR(col)` | Partition by hour | `HOUR(event_timestamp)` |
| `BUCKET(n, col)` | Hash into n buckets | `BUCKET(16, user_id)` |
| `TRUNCATE(n, col)` | Truncate string/int to n chars/digits | `TRUNCATE(3, zip_code)` |
| `VOID(col)` | No partitioning | — |

### 6.5 Compatibility Mode & File Distribution

Because Iceberg tracks files by their paths in the manifest (not by their folder location), data files can be stored **anywhere** in object storage — they don't need to be organized into strict partition folder hierarchies.

This is extremely valuable in object storage (like S3) because storing too many files in the same folder (prefix) can cause **request throttling**. Dremio, for example, automatically writes Iceberg data files to hashed subdirectories to distribute load across prefixes, avoiding throttling without requiring any configuration.

### 6.6 Performance Improvements

Iceberg improves query performance through:

1. **Partition pruning** — skip entire manifests based on partition statistics in the manifest list.
2. **File pruning** — skip individual data files based on column-level min/max statistics in manifests.
3. **Pre-computed statistics** — column statistics are written at ingest time, so no separate `ANALYZE TABLE` command is needed. Statistics are always current.

The result: instead of scanning 1,000 files for a filtered query, the engine might scan only 20.

### 6.7 Schema Evolution

Iceberg handles schema evolution gracefully. Since the schema is tracked in the metadata, even if old data files were written with an old schema, the query engine can apply the new schema when reading — handling:
- **Added columns** (returns `NULL` for old records that don't have the column)
- **Dropped columns** (simply not returned)
- **Renamed columns** (tracked by column ID, not by name)
- **Type promotions** (e.g., `INT` → `BIGINT`)

### 6.8 Cost Savings

- Object storage (S3, MinIO) is **much cheaper** than managed warehouse storage (Snowflake, Redshift).
- File-level updates mean you write **less data** per operation.
- Metadata pruning means you **read less data** per query, reducing S3 GET costs and compute time.
- You control your own data, so you control your own costs.

### 6.9 Wide Ecosystem

Apache Iceberg has the **broadest ecosystem** of any table format. Tools that can read and/or write Iceberg tables include:

| Category | Tools |
|---|---|
| Query Engines | Dremio, Apache Spark, Trino, Presto, Apache Flink |
| Streaming | Apache Kafka (via Flink/Spark), Apache Flink |
| BI / Serving | Dremio (direct), Tableau, Power BI (via Dremio/Trino) |
| Lakehouse Platforms | Dremio, Starburst (Trino), Cloudera |
| Cloud Managed | AWS Athena, AWS EMR, Google BigQuery (Iceberg tables), Azure HDInsight |
| Single-Node / Analytics | DuckDB, Polars |
| Data Quality / Catalog | Apache Atlas, Marquez, OpenMetadata |
| Orchestration | Apache Airflow, Prefect, dbt |

---

## 7. Iceberg Catalog Options

The catalog is one of the most important architectural decisions when adopting Apache Iceberg. The catalog tracks your tables and must be compatible with all the query engines you plan to use.

### 7.1 Project Nessie (Git-like Catalog)

[Project Nessie](https://projectnessie.org/) is an open-source catalog that brings **git-like versioning** to your entire data catalog.

Key features:
- **Branching & Merging**: Create a branch of your entire catalog, make changes across multiple tables, validate the results, then merge back to `main`. All changes are invisible to end users until merged.
- **Catalog-level time travel**: Tag the entire catalog state (e.g., at the end of each month) and reproduce any historical state of *all* tables simultaneously.
- **Multi-table transactions**: Make atomic changes across multiple tables using branches — without requiring a `BEGIN TRANSACTION` / `COMMIT` pattern.
- **Rollback**: If a bad ingest poisons multiple tables, roll back the entire branch — not each table individually.

```
main branch:  [Table A v1] [Table B v1] [Table C v1]
                    |
ingestion-branch: [Table A v2] [Table B v2] [Table C v2]  ← data engineers work here
                    |
      (validate, test, QA)
                    |
main branch (after merge): [Table A v2] [Table B v2] [Table C v2]
```

**Nessie-compatible engines**: Dremio, Apache Spark, Apache Flink, Trino (via REST adapter)

**Deployment options**:
- Self-hosted open-source Nessie server (Docker image available)
- **Dremio Arctic** — cloud-managed Nessie service (AWS, Azure, GCP agnostic)

### 7.2 Hive Metastore

If you already run a Hive Metastore (common in on-prem Hadoop environments), you can use it as an Iceberg catalog immediately — no migration required.

**Pros**: Low friction if you're already using it; broad compatibility.
**Cons**: No git-like features; requires running a separate database (MySQL/PostgreSQL) and Hive Metastore service.

### 7.3 AWS Glue

AWS Glue Data Catalog is the natural choice if you're fully within the AWS ecosystem.

**Pros**: Fully managed; integrates natively with Athena, EMR, Glue ETL.
**Cons**: Tightly coupled to AWS; may not work well with tools outside AWS.

### 7.4 Hadoop / File System Catalog

The Hadoop catalog (sometimes called the "file system catalog") writes catalog state directly to the file system (HDFS, S3, etc.) — no separate catalog service required.

**Pros**: Zero external dependencies; simplest to get started.
**Cons**: Consistency risks with multiple concurrent writers; no git-like features.

### 7.5 Other Catalogs

- **REST Catalog** — standardized REST API, broadly supported and the direction the ecosystem is moving.
- **JDBC Catalog** — use any relational database as the catalog store.
- **DynamoDB Catalog** — AWS DynamoDB as the catalog backend.
- **Snowflake Catalog** — Snowflake as the catalog (for Iceberg tables stored externally).
- **LakeFS** — file-level git-like versioning (different from Nessie's catalog-level versioning).

### 7.6 Choosing the Right Catalog

When selecting a catalog, ask two questions:

1. **Which query engines will I use?** The catalog must be supported by all of them.
2. **Where am I running?** On AWS, GCP, Azure, or on-prem?

General guidance:

| Situation | Recommended Catalog |
|---|---|
| Fully on AWS | AWS Glue |
| On-prem with existing Hadoop | Hive Metastore |
| Multi-cloud or cloud-agnostic | Project Nessie / Dremio Arctic |
| Simplest possible start (single-writer) | Hadoop / File System Catalog |
| Want git-like branching | Project Nessie / Dremio Arctic |

### 7.7 Migrating Between Catalogs

You can migrate tables from one catalog to another **without moving or rewriting data**. Since Iceberg tables are just metadata pointing to files, migration is a matter of registering the existing metadata in the new catalog:

```sql
-- Register an existing Iceberg table in a new catalog
CALL new_catalog.system.register_table(
  table => 'my_database.my_table',
  metadata_file => 's3://warehouse/my_table/metadata/v3.metadata.json'
);
```

Dremio and other platforms also offer catalog migration tools that automate this process for all tables at once.

---

## 8. Apache Iceberg File Layout

When Iceberg tables are written with a standard file-system layout, here is the directory structure:

```
warehouse/
└── my_database/
    └── table_one/
        ├── data/
        │   ├── part-00000-abc123.parquet    ← Actual data files
        │   ├── part-00001-def456.parquet
        │   └── part-00002-ghi789.parquet
        └── metadata/
            ├── v1.metadata.json             ← Metadata after CREATE TABLE
            ├── v2.metadata.json             ← Metadata after first INSERT
            ├── v3.metadata.json             ← Metadata after second INSERT (current)
            ├── snap-001.avro               ← Manifest list for snapshot 1
            ├── snap-002.avro               ← Manifest list for snapshot 2
            ├── snap-003.avro               ← Manifest list for snapshot 3
            ├── manifest-a.avro              ← Manifest file (list of data files)
            ├── manifest-b.avro
            └── manifest-c.avro
```

File types and their formats:

| File Type | Format | Purpose |
|---|---|---|
| Metadata file | JSON | Table schema, snapshots, partitioning history |
| Manifest list | Apache Avro | List of manifests for one snapshot, with partition stats |
| Manifest file | Apache Avro | List of data files with column-level statistics |
| Data file | Apache Parquet | Actual data, columnar, highly compressed |

**Why Avro for manifests and JSON for metadata?**
- JSON is human-readable and good for the top-level, relatively small metadata file.
- Avro is row-oriented, which is efficient for the manifest files where each row represents one data file or one manifest — and you often need to scan all rows.
- Parquet is column-oriented, which is efficient for the actual data files where queries typically access a few columns across many rows.

**Note on Dremio's compatibility mode**: When Dremio writes Iceberg tables, it automatically distributes data files across hashed subdirectories (rather than strict partition folders). This avoids S3 prefix throttling without any configuration:

```
warehouse/
└── table_one/
    ├── data/
    │   ├── ab/cd/ef/   ← hashed prefix directories
    │   │   └── part-00001.parquet
    │   └── 12/34/56/
    │       └── part-00002.parquet
    └── metadata/
        └── ...
```

---

## 9. Dremio & Apache Iceberg

### 9.1 What is Dremio?

Dremio is a **Data Lakehouse Platform** — a query engine and data platform specifically designed to work natively with Apache Iceberg and open data lake storage.

At its core, Dremio provides:
- **A fast SQL query engine** built on Apache Arrow (the same columnar in-memory format used by Pandas, Spark, etc.)
- **Multi-source connectivity** — query Iceberg tables alongside PostgreSQL, MySQL, SQL Server, Snowflake, MongoDB, S3 CSV/Parquet files, and more, all from one SQL interface.
- **A semantic layer** — curate views and virtual datasets on top of raw data without making copies.
- **A self-service UI** — analysts can build their own views and calculated fields without needing SQL.
- **Native BI integration** — one-click connections to Power BI and Tableau.

> **Interesting fact**: Apache Arrow — the open standard in-memory columnar format used by virtually every analytics tool — originally started at Dremio before being open-sourced.

Dremio is particularly popular for:
- **BI dashboards on the data lake** — replacing expensive data warehouse extracts with direct lake queries.
- **Hadoop modernization** — adding a modern SQL layer on top of existing HDFS/Hive clusters.
- **On-premises data lakehouses** — for organizations that can't use cloud services.

### 9.2 Key SQL Commands in Dremio

Dremio provides SQL commands for the full lifecycle of Iceberg table management:

**DDL (Data Definition Language)**
```sql
-- Create an Iceberg table
CREATE TABLE nessie.table_one (
  id INT,
  name VARCHAR
);

-- Alter the schema
ALTER TABLE nessie.table_one ADD COLUMN email VARCHAR;

-- Change partitioning
ALTER TABLE nessie.table_one SET PARTITION BY DAY(created_at);
```

**DML (Data Manipulation Language)**
```sql
-- Simple insert
INSERT INTO nessie.table_one VALUES (1, 'Alex'), (2, 'Mustafa');

-- COPY INTO — ingest files from object storage
COPY INTO nessie.table_one
FROM 's3://my-bucket/incoming/'
FILE_FORMAT = 'csv';

-- MERGE INTO — upsert (insert new, update existing)
MERGE INTO nessie.table_one AS target
USING source_table AS source
ON target.id = source.id
WHEN MATCHED THEN UPDATE SET name = source.name
WHEN NOT MATCHED THEN INSERT (id, name) VALUES (source.id, source.name);
```

**Table Maintenance**
```sql
-- Compaction: merge small files into larger ones
OPTIMIZE TABLE nessie.table_one;

-- Cleanup old snapshots
VACUUM TABLE nessie.table_one EXPIRE SNAPSHOTS OLDER_THAN = '2024-01-01';
```

**Time Travel**
```sql
-- Query a specific snapshot
SELECT * FROM nessie.table_one AS OF VERSION 3;

-- Query the table as of a point in time
SELECT * FROM nessie.table_one AS OF TIMESTAMP '2024-03-01 00:00:00';
```

**CREATE TABLE AS SELECT — Convert existing data to Iceberg**
```sql
-- Convert a CSV or other table to an Iceberg table
CREATE TABLE nessie.weather AS
SELECT * FROM my_s3_source.weather_data.weather_csv;
```

### 9.3 Reflections (Query Acceleration)

Dremio's **Reflections** feature automates query acceleration. Instead of manually creating materialized views or maintaining separate aggregation tables, you tell Dremio *what kind of queries* are coming and it handles the rest transparently.

There are two types of Reflections:

**Raw Reflections** — cache the raw data of a view or table as a hidden Iceberg table, so repeated queries don't re-read from source.

```
Use case: A frequently joined 3-table view. Without Reflections, every query
re-executes the 3-way join. With a Raw Reflection, the joined result is
cached as a hidden Iceberg table and served from there.
```

**Aggregation Reflections** — pre-aggregate data for BI dashboards, specifying which dimensions and measures to pre-compute.

```
Use case: A dashboard that always groups by country, product_category, and month
and sums revenue. An Aggregation Reflection pre-computes these groupings so
dashboard queries are near-instant.
```

Key properties of Reflections:
- **Transparent to end users**: Users always query the same view or table. Dremio automatically decides whether to use a Reflection to answer the query faster.
- **Multiple Reflections per table**: If different query patterns exist (e.g., some queries filter by region, others by date), you can create one Reflection per pattern. Dremio picks the best one for each query.
- **Powered by Iceberg**: Behind the scenes, every Reflection is stored as an Apache Iceberg table.
- **Configurable columns**: You can choose exactly which columns to include in a Reflection, avoiding unnecessary data storage.
- **Configurable sorting and partitioning**: Optimize the Reflection for the specific query patterns you're serving.

### 9.4 Caching

Dremio also has a **Cloud Cache** (also called Column Cache) that caches frequently accessed portions of Parquet files in memory on the query engine nodes. This:
- Eliminates repeated network calls to S3 for popular data.
- Reduces S3 GET request costs.
- Speeds up queries by serving data from local memory.

The cache is automatic — no configuration required for basic usage.

### 9.5 Dremio Arctic Catalogs

**Dremio Arctic** is Dremio's cloud-managed version of Project Nessie, available as part of the Dremio Cloud product.

Arctic catalogs provide:
- All git-like features of Project Nessie (branching, merging, tagging, rollback).
- Automatic table optimization (OPTIMIZE runs on a schedule you configure).
- Automatic snapshot cleanup (VACUUM runs on a schedule you configure).
- Fine-grained access control on branches, tables, and namespaces.
- Cloud-agnostic (works with AWS S3, Azure ADLS, Google Cloud Storage).

With Arctic, your Iceberg tables are maintained automatically. You get the feel of a managed data warehouse with the cost profile and flexibility of an open data lake.

### 9.6 Data Lakehouse Architecture with Dremio

Here is a complete reference architecture for a production data lakehouse using Dremio and Apache Iceberg:

```
┌─────────────────────────────────────────────────────────┐
│                    DATA SOURCES                          │
│  Databases (Postgres, MySQL)  ·  SaaS APIs  ·  Streams  │
└───────────────────┬─────────────────────────────────────┘
                    │
                    ▼ ETL / ELT
┌─────────────────────────────────────────────────────────┐
│                 INGESTION LAYER                          │
│  Apache Spark  ·  Apache Flink  ·  Dremio COPY INTO     │
│  DBT  ·  Airbyte  ·  Custom Pipelines                   │
└───────────────────┬─────────────────────────────────────┘
                    │  Raw Iceberg tables
                    ▼
┌─────────────────────────────────────────────────────────┐
│              STORAGE (Object Storage)                    │
│         S3  ·  Azure ADLS  ·  GCS  ·  MinIO             │
│                 (Parquet data files)                     │
└───────────────────┬─────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│               CATALOG (Nessie / Dremio Arctic)          │
│    Tracks all Iceberg tables  ·  Git-like versioning    │
│    Automated OPTIMIZE + VACUUM via Arctic               │
└───────────────────┬─────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│               DREMIO (Query & Semantic Layer)           │
│                                                         │
│  Bronze Views ──▶ Silver Views ──▶ Gold Views           │
│  (raw)             (cleaned)        (business-ready)    │
│                                                         │
│  Reflections (auto-acceleration)                        │
│  Access Control  ·  Data Catalog / Wiki                 │
│  Multi-source joins (Iceberg + PG + Snowflake + ...)    │
└───────────────────┬─────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│                  CONSUMPTION                            │
│  Power BI  ·  Tableau  ·  Jupyter Notebooks             │
│  REST API  ·  JDBC/ODBC  ·  Apache Arrow Flight        │
│  Ad-hoc SQL  ·  AI / ML  ·  Custom Applications        │
└─────────────────────────────────────────────────────────┘
```

This architecture follows an **ELT** (Extract, Load, Transform) pattern:
1. **Land raw data** as Iceberg tables in the data lake.
2. **Transform virtually** using Dremio views (Bronze → Silver → Gold) — no data copies.
3. **Accelerate** automatically with Reflections.
4. **Serve** all consumers from a single Dremio connection.

---

## 10. Hands-On Demo: Spinning Up a Data Lakehouse on Your Laptop

This section walks through the complete demo from the video — spinning up a full data lakehouse locally using Docker.

### 10.1 Prerequisites

- **Docker Desktop** — Install from [docker.com](https://www.docker.com/get-started/)
- **A terminal** — Any command prompt or shell
- Basic SQL familiarity

### 10.2 The Docker Compose Stack

The local lakehouse uses three containers:

| Container Name | Technology | Role | Port |
|---|---|---|---|
| `catalog` | Project Nessie | Iceberg catalog with git-like versioning | 19120 |
| `storage` | MinIO | S3-compatible object storage | 9001 |
| `dremio` | Dremio | SQL query engine & data platform | 9047 |

Create a file called `docker-compose.yml` with the following content (available at Alex's GitHub / linked in the video description):

```yaml
version: "3"

services:
  # Nessie Catalog Server
  catalog:
    image: projectnessie/nessie:latest
    container_name: catalog
    networks:
      - iceberg-net
    ports:
      - "19120:19120"

  # MinIO - S3-Compatible Object Storage
  storage:
    image: minio/minio:latest
    container_name: storage
    networks:
      - iceberg-net
    environment:
      - MINIO_ROOT_USER=admin
      - MINIO_ROOT_PASSWORD=password
    ports:
      - "9001:9001"
      - "9000:9000"
    command: ["server", "/data", "--console-address", ":9001"]

  # Dremio
  dremio:
    platform: linux/x86_64
    image: dremio/dremio-oss:latest
    container_name: dremio
    networks:
      - iceberg-net
    ports:
      - "9047:9047"
      - "31010:31010"
      - "32010:32010"

networks:
  iceberg-net:
```

> **Important note on Docker networking**: When using Docker Compose, containers can reach each other by their `container_name`. That's why you'll use `catalog:19120` and `storage:9000` as endpoints within the Dremio configuration — you're using the container names as hostnames.

### 10.3 Step 1: Start the Environment

Open a terminal in the directory containing your `docker-compose.yml` file and run:

```bash
docker compose up
```

Docker will pull the required images and start all three containers. The first run may take several minutes. You'll see log output from all three services. Wait until you see Dremio's startup message before proceeding.

### 10.4 Step 2: Configure MinIO (Object Storage)

MinIO is your local S3-compatible object storage. You need to create a **bucket** where your Iceberg data files will be stored.

1. Open a browser and go to: `http://localhost:9001`
2. Log in with:
   - **Username**: `admin`
   - **Password**: `password`
3. Go to **Buckets** → **Create Bucket**
4. Name the bucket: `warehouse`
5. Click **Create Bucket**

You now have an empty storage location for your data lake.

### 10.5 Step 3: Set Up Dremio

1. Open a browser and go to: `http://localhost:9047`
2. On first launch, you'll be prompted to create an admin account.
3. Fill in your details (name, email, password) and click **Create Account**.
4. You're now logged into your local Dremio instance.

### 10.6 Step 4: Connect the Nessie Catalog

Now connect Dremio to your Nessie catalog *and* your MinIO storage.

1. In Dremio, click **Add Source** (bottom left).
2. Select **Nessie**.
3. Fill in the **General** tab:
   - **Name**: `nessie`
   - **Nessie Endpoint URL**: `http://catalog:19120/api/v2`
   - **Authentication**: None

4. Click the **Storage** tab and fill in:
   - **AWS Access Key**: `admin`
   - **AWS Access Secret**: `password`
   - **Root Path**: `/warehouse`
   
5. Under **Connection Properties**, add:
   - `fs.s3a.path.style.access` → `true`
   - `dremio.s3.compat` → `true`

6. Scroll down to **Encrypt Connection** and turn it **OFF** (no SSL on localhost).

7. Under **Endpoint**, change the endpoint from `s3.amazonaws.com` to:
   `storage:9000` *(the MinIO container)*

8. Click **Save**.

Your Nessie source will now appear in the Dremio sources panel. You're connected!

### 10.7 Step 5: Create an Iceberg Table

Click the **SQL Runner** icon (or press `⌘/Ctrl + K`). In the SQL editor, run:

```sql
CREATE TABLE nessie.table_one (
  id INT,
  name VARCHAR
);
```

Click **Run**. Dremio will:
1. Write a metadata JSON file to MinIO.
2. Register the table in Nessie.

You've just created your first Apache Iceberg table!

### 10.8 Step 6: Insert Data & Observe Snapshots

Run two separate INSERT statements:

```sql
INSERT INTO nessie.table_one VALUES (1, 'Alex'), (2, 'Mustafa');
```

```sql
INSERT INTO nessie.table_one VALUES (3, 'Becky');
```

After both inserts, your table has gone through **3 transactions** (create + 2 inserts) = **3 snapshots** = **3 metadata files**.

Verify the data:
```sql
SELECT * FROM nessie.table_one;
```

Expected result:
```
id | name
---|-------
1  | Alex
2  | Mustafa
3  | Becky
```

### 10.9 Step 7: Inspect the Metadata in MinIO

Go back to MinIO at `http://localhost:9001` and navigate to:
`warehouse` → `table_one` → `metadata/`

You should see:
- **3 `metadata.json` files** — one per transaction (create, insert 1, insert 2)
- **3 manifest list files** (`.avro` files with "snap" in the name) — one per snapshot
- **Manifest files** (`.avro` files without "snap") — listing the actual data files

Also check `warehouse` → `table_one` → data directory to see the actual Parquet files.

> **Notice**: Dremio writes data files to hashed subdirectories (e.g., `ab/cd/ef/`) rather than partition folders, distributing load across S3 prefixes automatically.

### 10.10 Step 8: Ingest Data from External Sources

You can also create Iceberg tables by selecting data from other connected sources:

```sql
-- If you've connected an S3 source with CSV files:
CREATE TABLE nessie.weather AS
SELECT * FROM my_s3_source."weather_data.csv";
```

Or ingest CSV files directly:
```sql
COPY INTO nessie.weather
FROM 's3://my-source-bucket/weather/'
FILE_FORMAT = 'csv'
(TRIM_SPACE 'true', RECORD_DELIMITER '\n', FIELD_DELIMITER ',', SKIP_HEADER 1);
```

After running, go back to MinIO and check for your new `weather` table. You'll see a Parquet data file has been created — all the CSV data is now stored as a high-performance Iceberg table.

### 10.11 Step 9: Organize Data with the Wiki & Spaces

Dremio lets you organize your data into **Spaces** (equivalent to databases or schemas) and document everything in a built-in **Wiki**.

Create a Bronze/Silver/Gold hierarchy:

```sql
-- In the "Analytics" space, create a Bronze view
CREATE VIEW Analytics.Bronze."weather_raw" AS
SELECT * FROM nessie.weather AT BRANCH "main";

-- Silver — cleaned and cast
CREATE VIEW Analytics.Silver."weather_cleaned" AS
SELECT
  CAST(date AS DATE) AS date,
  CAST(temp_max AS DOUBLE) AS temp_max,
  CAST(temp_min AS DOUBLE) AS temp_min,
  weather
FROM Analytics.Bronze."weather_raw"
WHERE date IS NOT NULL;

-- Gold — business-ready aggregation
CREATE VIEW Analytics.Gold."avg_monthly_temp" AS
SELECT
  DATE_TRUNC('month', date) AS month,
  weather,
  AVG(temp_max) AS avg_high,
  AVG(temp_min) AS avg_low
FROM Analytics.Silver."weather_cleaned"
GROUP BY 1, 2;
```

You can also grant specific users or groups access to specific spaces, so different teams see only the data they need.

---

## 11. Table Optimization & Maintenance

### 11.1 Compaction (OPTIMIZE)

Over time, many small Parquet files accumulate in your Iceberg table (especially with frequent small inserts or streaming ingestion). Reading many small files is slower than reading a few large files.

**Compaction** (called OPTIMIZE in Dremio) merges small files into larger ones:

```sql
-- Compact all small files in the table
OPTIMIZE TABLE nessie.table_one;

-- Compact with custom file size targets
OPTIMIZE TABLE nessie.table_one
FOR PARTITIONS (MONTH(event_date) = '2024-01')
OPTIONS (TARGET_FILE_SIZE_MB = 256, MIN_FILE_SIZE_MB = 100, MAX_FILE_SIZE_MB = 512);
```

**How it works**:
1. Dremio scans each partition and looks at file sizes.
2. Files that are too small (below `MIN_FILE_SIZE_MB`) are grouped for rewriting.
3. Files are re-read and written into new, larger Parquet files.
4. A new snapshot is created pointing to the new files.
5. Old files are preserved for time travel but are no longer part of the current snapshot.

**When to run**: After streaming ingestion, after large bulk deletes/updates, or on a regular schedule.

### 11.2 Snapshot Cleanup (VACUUM)

Every transaction creates a new snapshot and writes new files — old files are never deleted automatically. Over time, this accumulates storage costs.

**VACUUM** removes old snapshots and their associated data files that are no longer part of any active snapshot:

```sql
-- Remove all snapshots older than 7 days (keep at least the last 2)
VACUUM TABLE nessie.table_one
EXPIRE SNAPSHOTS OLDER_THAN = (CURRENT_TIMESTAMP - INTERVAL '7' DAY)
RETAIN LAST 2;
```

> **Warning**: After running VACUUM, you can no longer time travel to the removed snapshots. Always consider your time travel requirements before setting the retention period.

**Automating maintenance with Dremio Arctic**: In Dremio Arctic, you can enable automatic OPTIMIZE and VACUUM on a configurable schedule — no manual maintenance required.

---

## 12. On-Premises & Cloud-Agnostic Use Cases

One of Apache Iceberg's most important properties for many organizations is that it works **everywhere** — cloud, on-prem, or hybrid.

For organizations in regions without major cloud providers (common in parts of the Middle East, Africa, and other regions), or for organizations with strict data residency and security requirements, Iceberg + Dremio offers a complete on-prem data lakehouse:

| Component | On-Premises Solution |
|---|---|
| Object Storage | **MinIO** (open source, S3-compatible, production-grade) |
| Table Format | **Apache Iceberg** (open standard) |
| Catalog | **Project Nessie** (self-hosted, open source) |
| Query Engine | **Dremio** (supports on-prem deployment) |
| ETL/ELT | **Apache Spark** or **Apache Flink** (open source) |

Dremio is also popular for **Hadoop modernization**: organizations running legacy Hadoop/Hive clusters can add Dremio on top of their existing HDFS + Hive infrastructure, getting modern SQL performance without migrating data. Later, if cloud migration becomes possible, the same Dremio setup simply points to S3/ADLS instead of HDFS — end users see no change.

---

## 13. Iceberg vs. Proprietary Data Warehouses

| Property | Apache Iceberg (Open) | Snowflake / Redshift / BigQuery (Proprietary) |
|---|---|---|
| Data format | Open (Parquet) | Proprietary internal format |
| Portability | Any engine can read/write | Locked to the vendor |
| Multi-engine | Spark, Flink, Trino, Dremio, etc. | Single vendor's engine |
| Storage cost | Object storage rates (cheap) | Proprietary storage (expensive) |
| Migration cost | Low (metadata only, no data move) | High (full data export/import) |
| ACID transactions | Yes (via snapshot isolation) | Yes |
| Time travel | Yes (configurable retention) | Yes (usually 90 days, billed) |
| Schema evolution | Yes | Yes |
| Hidden partitioning | Yes | Automatic (but hidden) |
| Community governance | Open (Apache Foundation) | Vendor controlled |
| On-prem deployment | Yes | No (cloud-only) |

The core advantage of Iceberg is **freedom**. Your data is always yours, in an open format, readable by any tool. New tools will emerge over the next decade — with Iceberg, you can adopt them without migrating petabytes of data.

---

## 14. When to Use Apache Iceberg

**Use Apache Iceberg when:**
- Your dataset has **multiple Parquet files** and you want fast query pruning.
- You need **ACID transactions** on your data lake.
- You want **time travel** and the ability to roll back writes.
- You need **schema evolution** without rewriting data.
- You want **multiple engines** (Spark, Dremio, Trino, Flink) to share the same data.
- You need **partition evolution** — changing how data is partitioned over time.
- You're building a **data lakehouse** and want data warehouse features at data lake cost.
- You care about **data portability** and don't want vendor lock-in.

**Even without UPDATE/DELETE**, Iceberg adds value for:
- **Portability**: Any tool that reads Iceberg can see your tables immediately.
- **Time travel**: Audit historical states of your data.
- **Statistics**: Always-current column statistics improve query planning.
- **Schema governance**: Formal schema tracking and evolution.

**When a single Parquet file might be sufficient:**
- Tiny datasets (single file, no partitioning needed).
- Truly append-only data with only one reader that never changes.
- Quick one-off data analysis that doesn't need to be productionized.

Even then, starting with Iceberg from the beginning is generally the right call — it adds little overhead and saves migration work later.

---

## 15. Connecting BI Tools (Power BI, Tableau)

Apache Iceberg is not a BI server — you need a **query engine in the middle** to serve BI dashboards. The architecture is:

```
Power BI / Tableau
        ↕ JDBC / ODBC / Arrow Flight
   Query Engine (Dremio, Trino, Athena)
        ↕ Iceberg reads
   Data Lake (S3, MinIO, HDFS)
```

**Recommended options:**

| Query Engine | Catalog Support | BI Integration |
|---|---|---|
| **Dremio** | Nessie, Hive, Glue, REST | Native Power BI & Tableau buttons in UI |
| **Trino / Starburst** | Nessie, Hive, Glue, REST | JDBC/ODBC to Power BI & Tableau |
| **AWS Athena** | AWS Glue | Native AWS integration |
| **Presto** | Hive, Glue | JDBC/ODBC |

In Dremio specifically, once you have a dataset or view:
1. Click the **Power BI** or **Tableau** button on the dataset.
2. Dremio generates the connection string and opens the BI tool.
3. The BI tool connects directly to Dremio via the native connector.

Enable **Aggregation Reflections** on the datasets your BI dashboards query to ensure sub-second dashboard load times even on large Iceberg tables.

---

## 16. Additional Resources

### Books
- 📖 **[Apache Iceberg: The Definitive Guide](https://learning.oreilly.com/library/view/apache-iceberg-the/9781098148614/)** — Co-authored by Alex Merced (Dremio). Free early release copy via QR code in the original video.

### Video Playlists
- 📺 **Alex Merced's YouTube Channel**: [youtube.com/alexmerceddata](https://youtube.com/alexmerceddata)
  - Playlist: **Apache Iceberg Lakehouse Engineering** — hands-on videos using Dremio, Spark, Polars, and more.

### Blog Posts (Dremio Blog)
- [Dremio Blog](https://www.dremio.com/blog) — Search for:
  - "Apache Iceberg 101"
  - "Apache Iceberg with Flink"
  - "Configuring Spark with Apache Iceberg"
  - "Spin up a Data Lakehouse on Your Laptop"

### Official Documentation
- 📄 [Apache Iceberg Documentation](https://iceberg.apache.org/docs/latest/) — The official spec and format docs.
- 📄 [Project Nessie Documentation](https://projectnessie.org/docs/) — Catalog with git-like versioning.
- 📄 [Dremio Documentation](https://docs.dremio.com) — Dremio SQL reference, setup guides, and more.

### Community
- 💬 **Apache Iceberg Slack**: `#iceberg` on the Apache Slack workspace.
- 📧 **Apache Iceberg Mailing List**: Public discussions on all format changes.
- 📅 **Apache Iceberg Public Meetings**: Bi-weekly open community meetings (link in official docs).

### Connect with Alex Merced
- LinkedIn: Search "Alex Merced Dremio"
- Dremio offers **free consultations** to discuss your specific data challenges — reach out via LinkedIn.

---

## Glossary

| Term | Definition |
|---|---|
| **Catalog** | A registry of all Iceberg tables and their current metadata file locations. |
| **Compaction** | Merging many small Parquet files into fewer, larger files for better query performance. |
| **Data Lake** | A storage system (S3, HDFS, MinIO) containing raw data files. |
| **Data Lakehouse** | A data lake that also provides data warehouse capabilities (ACID, time travel, fast analytics). |
| **Data Warehouse** | A managed system optimized for analytics, usually with a proprietary format. |
| **Manifest File** | An Avro file listing a group of data files with their column-level statistics. |
| **Manifest List** | An Avro file listing all manifest files that make up one snapshot. |
| **Metadata File** | A JSON file containing the table schema, partitioning, and snapshot history. |
| **MinIO** | An open-source, S3-compatible object storage system. Good for on-prem and local dev. |
| **Nessie** | An open-source Iceberg catalog providing git-like branching and versioning. |
| **Parquet** | An open-source, columnar file format optimized for analytical workloads. |
| **Partition** | A subdivision of a table's data based on a column value (e.g., all records from January). |
| **Partition Evolution** | The ability to change a table's partitioning scheme without rewriting existing data. |
| **Partition Pruning** | Skipping entire partitions (and their manifests) that don't satisfy a query's WHERE clause. |
| **Reflection** | A Dremio feature that automatically creates and maintains hidden acceleration tables (raw or aggregated) for faster queries. |
| **Schema Evolution** | The ability to add, rename, or remove columns without rewriting existing data. |
| **Snapshot** | A complete, consistent view of an Iceberg table at one point in time. |
| **Time Travel** | The ability to query an Iceberg table as it existed at a previous snapshot or timestamp. |
| **VACUUM** | A cleanup operation that removes old snapshots and their associated data files. |

---

*This guide was compiled from a two-part YouTube video series featuring Alex Merced (Developer Advocate at Dremio) and Mustafa. All credit for the original content goes to the presenters.*

*Last updated: 2026*
