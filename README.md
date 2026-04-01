# About 
In this project repo, we will learn about Apache -Iceberg and how can we use it?

## What is Apache Iceberg?
Apache Iceberg is an open-source, high-performance table format designed for massive analytical datasets stored in data lakes (like S3, GCS, or Azure). Unlike traditional file-based formats that track data by folders, Iceberg tracks data at the file level using a sophisticated metadata layer. This allows it to bring the reliability and simplicity of SQL tables to the "wild west" of big data, enabling features 
like ACID transactions, time travel (querying historical versions of data), and schema evolution (adding or renaming columns) without 
the risk of breaking your downstream pipelines.

## Why is it so popular in 2026?
Iceberg has become the industry standard primarily due to its engine-agnostic nature. 
While other formats are often tied to specific tools, Iceberg is natively supported by almost every major data engine, 
including Spark, Trino, Snowflake, BigQuery, and DuckDB. 
Its unique ability to perform partition evolution—changing how data is organized (e.g., from daily to hourly)
without rewriting the entire dataset—makes it a lifesaver for growing companies that need to scale their data architecture without downtime

