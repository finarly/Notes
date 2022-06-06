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

1. JSON data
    - Stage the data: Make JSON available in SF stage
2. Load as raw into temp table
    - Load the JSON data as raw string into a temporary table with one column of variant data type (can contain any type of data)
3. Analyse and prepare
    - Using SQL, analyse the JSON and prepare for flattening the structure. Use flatten function by SF
4. Flatten & Load
    - Flatten & load into the target table
5. Regular updates
    - Regular updates to the data may require delta detection (Delta detection is a task that compares new data from a source system against last versions of data in the data warehouse to find out whether a new version has to be created)


## Snowpipe
- It it used by SF to load high frequency or streaming data
- Used usually when there's data continuously arriving, like transactions or events that needs to be made available to business immediately
- Serverless so you dont need a virtual warehouse and is separately billed too

### How does it work?
- Snowpipe definitions contain a COPY statement which is used by SF to load the data so it knows what data to load and where to load it to
- it can be automatically or manually triggered to upload data.

### Steps
1. Stage the data: S3
2. Test copy command: Create target table & validate your copy into command
3. Create pipe: Create pipe and provide copy command
4. Configure cloud event: 
    - Use cloud native event triggers to trigger snow pipe (preferable | has no GCP option currently). OR
    - Use REST API endpoints (GCP friendly)

## Performance optimisation
SF was designed for simplicity so provides limited options in performance optimisation. Options on SF for optimisation:

1. Splitting files to appropriate sizes
2. Assigning app data types
3. Optimising virtual warehouses
4. Introduce cluster keys to large tables

### Consideration

- Running SF for large number of users: Use dedicated virtual warehouses for users with similar workloads
    1. Identify and categorise workloads executing on your SF e.g. ETL, DataS, BI, AdHoc
        - Dont go too fine grain with categorisation, creating many vw can speed up your processes, but having lots of vw increases risks of under utilisation which will drive up costs
        - Should be a regular activity to keep these groups updated
    2. Identify your users and put them into those groups
    3. Create dedicated vw for these classification 
- Scaling up virtual warehouses: use for peak processing period
    - Increasing size of vw as response to change in workloads, this could be on an adhoc basis or could be based on a pattern
    - Sizing up is usually used where query complexity has increased. If number of users increase, these queries increases exponentially so scale out may be better option
- Scaling out for unknown & unexpected workloads: Use auto spawn virtual warehouses based on workloads
    - Also known as multi cluster warehouses, spawning new vw and directing user to new vw and shutting down when demand decreasese is all automated
    - Requires setting minimum or maximum number of vw
    - Advantage over adding new vw on demand, since if you opt for manually add new vw, you need to redirect users to new vw. 
- Design to maximise cache usage: cache is automatic, but can be maximised. Group users that use similar data a lot and make them use same virtual warehouse
    - Automatic in SF and turned on by default
    - Results are cached for 24hours and if underlying data has changed SF can detect and will therefore re-execute query
    - To maximise cache usage, ensure similar queries go to the same virtual warehouse
- Use cluster keys to partition large tables: For tables that have very large data use cluster keys to improve query performance


