# Snowflake

## What is Snowflake?

### 1. Built for the cloud

- Built from scratch
- Optimized for cloud
- Storage & compute is decoupled

### 2. Software as a Service

No software, infrastructure or upgrades to manage

### 3. Pay only for used compute & storage

Storage & compute charged independently, only for us

### 4. Scalable

Virtual warehouse enable compute scaling

## Pricing Overview
- Pricing is based on the amount of data stored and the amount of compute used. Since compute and storage is decoupled, they can be billed separately
- No cost for data transfer into Snowflake, but it costs to transfer out of Snowflake or between regions 
### Storage
- Storage is charged monthly, and is based on the average data stored per month
- Storage cost is calculated after compression
- Storage can be either on demand or prepaid/upfront (Sydney: On demand: $46 per TB, prepaid: $25 per TB)
### Compute
- Compute is calculated via credits 
- Credits are only used when virtual warehouses are in compute state
- Rate of consumption is dependent on the size of the virtual warehouse

## What is the purpose of warehouse?
An analytical DB that collects all data in single location through ETL or ELT process. Within DB data is organised into several different layers to make it easier to maintain. 

- Examples:
    - raw data layer
    - integrated/unified data model: transform raw data so its easier to query. Also will transform data to create relationships between data from different data sources. 
    - summary data
    - access layer

When data enters the data warehouse, it usually doesnt go straight to the raw layer, but enters a staging area first (a transient and temp storage before it is loaded). A staging area can be in the DW (internal stage) or outside of the DW (external stage).

## Snowflake architecture - hybrid
- Shared disk: Similar to shared disk SF uses a central data repository (uses single copy of data)
- MMP compute clusters: Similar to shared nothing, SF processes queries using MPP (massively parallel processing) compute clusters where each node in the cluster stores a portion of the entire data set locally. 
- Summary: SF decouples storage from compute 
- Storage used is S3 (if AWS): data is automatically compressed
- Compute in SF consist of one or more clusters, consist of compute nodes that are used to load and query data. Nodes are called virtual warehouse. Compute nodes do computing on S3 data, they do not store any data in their compute nodes. 
- Virtual warehouses can be set up for different purposes e.g. reporting (dashboards), analytics (R, python), ETL/ELT
- Compute nodes can be paused when no usage, or can be resumed and even scaled

## Snowflake - scalability & virtual warehouses
- Storage is based on cloud so it is infinitely scalable
- However for virtual warehouses (compute clusters), it is a little different
    - You can provision additional computer clusters (maybe request a bigger cluster)
    - Multi cluster virtual warehouse (enterprise customer only): SF will automatically scale in and out a particular node that has MCluster enabled

## Getting Data into SnowFlake
- Bulk/batch Load:
    - Most popular method
    - Loaded at regular intervals
    - SF uses COPY command for batch loading, which uses virtual warehouse
    - Batch load allows basic transformation of the data like reordering columns, exclude columns, data types, truncating strings
- Continuous Load:
    - Snowpipe is used
    - Uses serverless approach
    - Doesnt use virtual warehouse compute resource, snowpipe is separate
- External tables:
    - Good for small subsets of data you want to query
    - Cost can be optimised by creating materialised views

## Staging data
### Why do we need a stage

The Stage is an area external to the db, the db in our case SnowFlake, but that area is accessible to the db. Its a area used for ETL or ELT that sits between source and target (snowflake). It is most commonly somewhere like in an S3 bucket, Azure blob, or even somewhere in your local file system. 

- COPY INTO command is common mechanism for loading data into Snowflake in batch mode. There are several options: 
    - Specify path to load
    - Specify the file names to be loaded 
    - Use pattern matching to load only files matching the pattern

- JSON data
    - Stage the data: Make JSON available in SF stage
- Load as raw into temp table
    - Load the JSON data as raw string into a temporary table with a column of variant data type (can contain any type of data)
- Analyse and prepare
    - Using SQL, analyse the JSON and prepare for flattening the structure. Use flatten function by SF
- Flatten & Load
    - Flatten & load into the target table
- Regular updates
    - Regular updates to the data may require delta detection
    * Delta detection is a task that compares new data from a source system against last versions of data in the data warehouse to find out whether a new version has to be created  