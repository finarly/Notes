# DataBricks Learning

## Overview

Databricks is a multi-cloud lake-house platform based on Apache Spark. It aims to achieve the best of both worlds between data warehouses and data lake.
- Data lakes are known to be more open, flexible, and support ML better (as it benefits from raw data).
- Data warehouses are known to be more reliable, has strong governance, and performance very well when queried.

Because it is built off Spark, the compute is done in the memory of multiple nodes in a cluster. it also supports **Batch** and **Stream** Processing, and can work with **Structured**, **Semi-Structured**, and **Unstructured** data.

It supports all languages supported by Spark:
- Scala
- Python
- SQL
- R
- Java

### Layout

Databricks has 3 layers: 

1. Cloud Service: AWS, Azure, GCP etc.
2. Runtime: Apache Spark and Delta Lake
3. Workspace: databricks GUI

### Data resource deployment view

- Control plane: Web UI, Cluster management, Workflows, Notebooks
    - The control plane lives in the Databrick's account,  

- Data plane: cluster of VMs, Storage (DBFS)
    - Compute and storage will always be in the customer's account. DB will provide tool to use and control infrastructure.
    - The data plane lives in the customer's account.
    - Since Apache Spark processes data in distributed manner, DBricks has a native support of a distributed file system. It is just an abstraction layer, in actuality the data is just stored in Cloud Storage (e.g. S3)

## Delta Lake

Delta lake is an open-source software that provides the foundation for storing data and tables in the Databricks. It extends parquet data files with a file-based *transaction log* for ACID transactions and scalable metadata handling. All tables on Databricks are Delta tables unless otherwise specified. 

It is storage framework that helps data lakes become lake house.

The data held in delta tables are stored in one or more files in parquet format, alongside a **Transaction Log**.

### Transaction log

This is a JSON file which has ordered record of every transaction performed on the table. This is the single source of truth. 

### Advanced Delta Lake features

#### Time Travel

In Databricks, every iteration of the table is versioned, this allows you to look back into time of a table.

To see the history of a table:
> DESCRIBE HISTORY

To read the table at a particular point:

- Using timestamp:
    > SELECT * FROM my_table **TIMESTAMP AS OF** "2019-01-01"

- Using version number:
    > SELECT * FROM my_table **VERSION AS OF** 36
    > SELECT * FROM my_table@36


To rollback:

- RESTORE TABLE:
    >**RESTORE TABLE** my_table **TO TIMESTAMP AS OF** "2019-01-01"
    >**RESTORE TABLE** my_table **TO VERSION AS OF** 36

#### Compaction

You can improve read query speeds by compacting small files into larger ones. 

> OPTIMIZE
    OPTIMIZE my_table
    > ZORDER BY column_name


#### Vacuum

This is a command to help clean up unused files such as:
- uncommitted files
- files that are no longer in the latest table state

- VACUUM (default period 7 days): 
    > VACUUM table_name [*retention period*]

Note: if you vacuumed, then you cannot perform time travel. 

### Setting up Delta Tables

```
CREATE TABLE new_table
    - **COMMENT** "Contains PII"
    - **PARTITIONED BY** (city, birth_date)
    - **LOCATION** '/some/path'
AS SELECT id, name, email, birth_date, city FROM users
```

> CREATE TABLE table_1 AS SELECT _ FROM table_2


## Relational entities

### Databases

In Databricks, a database is a schema in Hive meta store, therefore:

**CREATE DATABASE** db_name = **CREATE SCHEMA** db_name

Hive metastore is a repository of metadata, which holds metadata about your table and data. 

The default database location is in the default hive directory: *dbfs:/user/hive/warehouse* 

You can create databases outside of this using **LOCATION** command.

> **LOCATION** 'dbfs:/custom/path/db_y.db

The 'db' suffix is what lets us know that it is a database.

### Tables

There are 2 types of tables:

- Managed tables:
    - Created under the database directory
    > **CREATE TABLE** table_name
    - The underlying data files will be deleted when dropping the table.

- External tables:
    - Created outside the database directory
    > **CREATE TABLE** table_name **LOCATION** 'path'
    - The underlying data files will not be deleted when dropping the table.

