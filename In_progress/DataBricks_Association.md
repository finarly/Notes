l# DataBricks Learning

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



## Databricks Lakehouse Platform

Delta lake is an open-source software that provides the foundation for storing data and tables in the Databricks. It extends parquet data files with a file-based *transaction log* for ACID transactions and scalable metadata handling. All tables on Databricks are Delta tables unless otherwise specified. 

It is a storage framework that helps data lakes become lake house.

The data held in delta tables are stored in one or more files in parquet format, alongside a **Transaction Log**.


### Delta Lake

### Transaction log

This is a JSON file which has ordered record of every transaction performed on the table. This is the single source of truth. 


### Advanced Delta Lake features

#### Time Travel

In Databricks, every iteration of the table is versioned, this allows you to look back into time of a table.

To see the history of a table:
> DESCRIBE HISTORY

To read the table at a particular point:

- Using timestamp:
    - ```SELECT * FROM my_table TIMESTAMP AS OF "2019-01-01"```

- Using version number:
    - ```SELECT * FROM my_table VERSION AS OF 36``` or
    - ```SELECT * FROM my_table@36```


To rollback:

- RESTORE TABLE:
    - ```RESTORE TABLE my_table TO TIMESTAMP AS OF "2019-01-01"``` or
    - ```RESTORE TABLE my_table TO VERSION AS OF 36```

#### Compaction

You can improve read query speeds by compacting small files into larger ones. 

```
OPTIMIZE my_table
ZORDER BY column_name
```

#### Vacuum

This is a command to help clean up unused files such as:
- Uncommitted files
- Files that are no longer in the latest table state

- VACUUM (default period 7 days):
    - ```VACUUM table_name [*retention period*]```

Note: if you vacuumed, then you cannot perform time travel. 


### Setting up Delta Tables

#### Create Table AS (CTAS)

```
CREATE TABLE new_table
    - **COMMENT** "Contains PII"
    - **PARTITIONED BY** (city, birth_date - should be used in huge files only)
    - **LOCATION** '/some/path'
AS SELECT id, name, email, birth_date, city FROM users
```

#### Table Constraints

- NOT NULL constraints
- CHECK constraints

General format:
```ALTER TABLE table_name ADD CONSTRAINT constraint_name constraint_details```

Example:
```ALTER TABLE orders ADD CONSTRAINT validate CHECK (date > '2020-01-01')```


#### Cloning Delta Lake Tables

NB: Either cloning methods will not affect the source tables.

- Deep Clone:
    - Fully copies date + metadata from a source table to a target, which means that this will take a while. Executing this again will sync the changes.
```
CREATE TABLE table_clone
DEEP CLONE source_table
 ```

- Shadow Clone:
    - Quickly creates a copy of a table by copying over the Delta transaction logs, which means there's no data moving.
```
CREATE TABLE table_clone
SHALLOW CLONE source_table
```

### Setting up Views

Same as views in other databases. 

Types of views: 
1. (Stored) Views: persisted like a table in the database.
    - Dropped only by **DROP VIEW**
    - ```CREATE VIEW view_name AS query```

2. Temporary views: tied to a spark session. It gets dropped when the session ends. 
    - Spark session is created when:
        - Opening a new notebook
        - Detaching and reattaching to a cluster
        - Installing a python package
        - Restarting a cluster
    - ```CREATE TEMP VIEW view_name AS query```

3. Global Temporary views: tied to a cluster. Dropped when a cluster is restarted.
```
CREATE GLOBAL TEMP VIEW view_name AS query
SELECT * FROM global_temp.view_name
```

### Relational entities

#### Databases

In Databricks, a database is a schema in Hive meta-store, therefore:

```
CREATE DATABASE db_name = CREATE SCHEMA db_name
```

Hive meta-store is a repository of metadata, which holds metadata about your table and data. 

The default database location is in the default hive directory: *dbfs:/user/hive/warehouse* 

You can create databases outside of this using **LOCATION** command.

```
 **LOCATION** 'dbfs:/custom/path/db_y.db
```

The 'db' suffix is what lets us know that it is a database.

#### Tables

There are 2 types of tables:

- Managed tables:
    - Created under the database directory
        - ```CREATE TABLE table_name```
    - The underlying data files will be deleted when dropping the table.

- External tables:
    - Created outside the database directory
        -```CREATE TABLE table_name LOCATION 'path'```
    - The underlying data files will not be deleted when dropping the table.


## ELT with Spark SQL and Python

### Querying files

![query diagram](./databricks/Screenshot%202023-11-01%20000422.png)


Extract as raw strings - when working with text based files (e.g. JSON, CSV, TSV, and TXT)
- SELECT * FROM text.`` `/path/to/file` ``
- Example JSON:
    - SELECT * FROM json.`` `/path/file_name.json` ``

Extract as raw bytes - when working with images or unstructured data:
- SELECT * FROM **binaryFile**.`` `/path/to/file` ``

To load data from files to tables:

```CREATE TABLE table_name AS SELECT * FROM file_format.`` `/path/to/file` `` ```

The table will automatically infer schema information from query results. CTAS are useful for external data ingestion with well-defined schema, but is very limited with options.

That is why we use the normal table creation statement.

```
CREATE TABLE table_name 
(col_name1 col_type1,...)
USING data_source
OPTIONS (key1 = val1,key2=val2,...)
LOCATION = path
```

With these commands we are always create an external table, therefore we are just pointing to files. The tables created are **non-delta tables**. The limitation here is that since it is not a delta table, there is no reliability guarantee.

The solution to this problem is:
1. Create a temporary view using create table method
2. Create a table using CTAS using the temporary view in step 1.

### Writing to tables

When writing to table, there are multiple benefits in overwriting the data, rather than dropping and creating a new table.
Those are:
- The old version of the table still exist, so it can be time travelled to
- It is faster because it does not need to go to the directory recursively or delete any files
- It is an atomic operation, concurrent queries can still read the table as you are overwriting it
- If overwriting table fails, it will be in its table state


#### Overwrite methods

1. 
```
CREATE OR REPLACE TABLE table_name AS
SELECT * FROM date_source.`path`
```

2. Can only overwrite an existing table, it is does not have the risk of change the table schema
```
INSERT OVERWRITE table_name AS
SELECT * FROM data_source.`path`
```

#### Append records to table

1. No built in guarantees for de-duplicating records
```
INSERT INTO table_name
SELECT * FROM data_source.`path`
``` 

2. Guarantee for de-duplication using **MERGE**, example: 
```
MERGE INTO customers c
USING customers_updates u 
ON c.customer_id = u.customer_id
WHEN MATCHED AND c.email IS NULL AND u.email IS NOT NULL THEN
    UPDATE SET email = u.email, updated = u.updated
WHEN NOT MATCHED THEN INSERT *

```

### Advanced transformations 

Spark SQL allows you to traverse/parse JSON file formats and get to the nested values.

e.g.
```
profile
{"first_name":"Thomas","last_name":"Lane","gender":"Male","address":{"street":"06 Boulevard Victor Hugo","city":"Paris","country":"France"}}


SELECT customer_id, profile:first_name,profile:address:country
FROM customers
```