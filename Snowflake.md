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

When data enters the data warehouse, it usually doesn't go straight to the raw layer, but enters a staging area first (a transient and temp storage before it is loaded). A staging area can be in the DW (internal stage) or outside of the DW (external stage).

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
    SF maintains micro partitions for a table automatically and it most cases that is enough, but over time as table grows very large, and new data comes in this partitioning may not remain most optimal. Normally you would use those columns in cluster keys which are frequently used in WHERE clauses or are frequently used in JOINS or ORDER BY
    - Clustering is not for all tables, generally large multi terabyte size will benefit
    - Clustering can be used on:
        - Single/multiple column in your clustering statement
        - Expressions
            - e.g. If your table has a transaction date but most of your queries uses MONTH


## Time travel

Access historical data by traveling back to a point in time up to 90 days. You can time travel for databases, schemas, and tables. 

## Failsafe

A continuous data protection that is separate from time travel feature, that ensures data protection in event of failure. It provides a 7 day period during which historical data is recoverable by SF. The failsafe starts immediately after time travel ends, and the data can only be recovered by SF support.

Table types:
  - Permanent table: Default table type
    - Time travel: up to 90
      Failsafe: 7 days
  - Temporary table: Pure temporary tables and only exist for lifetime of a session. They are not visible to other sessions and are removed immediately once session ends. 
    - Time travel: up to 1
      Failsafe: 1 day
  - Transient table: Sort of temp tables but they persist between sessions. They are designed to hold temp data that needs to be accessed across sessions e.g. ETL jobs (one ETL job writing to a table, and another job reading from that)
    - Time travel: up to 1
      Failsafe: 1 day

## Zero Copy Cloning & Time Travel

In most data warehouses, in order to copy you'll need to create the structure and then insert the data in it. SF has ZCC which creates a copy of db, schema, or table, but it doesn't replicate the data, it is a metadata operation where a snapshot of the original data is made available to the cloned project. The cloned object is independent and can be independently modified. 

A cloned object shares the original object's data until the data starts getting modified in either object.

You can also clone a database, schema, or table while time traveling. 

## Sharing data

Historically sharing data has been troublesome. SF has a structure which enables sharing with minimal effort, it can be shared ***without extracting data***. Shared data can by used by the consumers by using their own compute resources (any updates to the data will be reflected to the consumer because the data is shared in place), or if that are a non SF user, they can get shared data through reader accounts (non SF users use your organisations compute resources).

Sharing is:
- Instantaneous
- Up to date
- Secure & Governed

### Sharing with SF users

> CREATE SHARE ***SHARE_OBJECT***
> GRANT USAGE ON DATABASE ***DB*** TO SAHRE ***SHARE_OBJECT***
> GRANT USAGE ON SCHEMA ***SCHEMA*** TO SHARE ***SHARE_OBJECT***
> ALTER SHARE ***SHARE_OBJECT*** ADD ***ACCOUNT=***
> GRANT SELECT ON TABLE ***TABLE*** TO SHARE ***SHARE_OBJECT***

example:
> CREATE DATABASE crm_data FROM SHARE <producer_account_number>.share_customer

### Sharing with non SF users

- Create new reader account 
- Share data with reader account
- Create users in user account 
- Create database from share
- Configure VW for reader account (user small Vw)

> CREATE SHARE ***SHARE_OBJECT***
> GRANT USAGE ON DATABASE ***DB*** TO SAHRE ***SHARE_OBJECT***
> GRANT USAGE ON SCHEMA ***SCHEMA*** TO SHARE ***SHARE_OBJECT***
> ALTER SHARE ***SHARE_OBJECT*** ADD ***ACCOUNT=***
> CREATE MANAGED ACCOUNT ***ACCOUNT***
> ADMIN_NAME = ***NAME*** , ADMIN_PASSWORD = ***PASSWORD***
> TYPE = READER

> ALTER SHARE ***TABLE*** ADD ***ACCOUNT=***

### Sharing complete DBs and Schema

> CREATE SHARE ***SHARE_OBJECT***

- grant usage on the database in which our table is contained
>GRANT USAGE ON DATABASE ***DB*** TO SHARE ***SHARE_OBJECT***

- grant usage on the schema in which our table is contained
>GRANT USAGE ON SCHEMA ***SCHEMA*** TO SHARE ***SHARE_OBJECT***

- grant select on all tables in this database, if a table is added at a later stage, that will not be shared
>GRANT SELECT ON ALL TABLES in schema ***SCHEMA*** TO SHARE ***SHARE_OBJECT***

> ALTER SHARE ***SHARE_OBJECT*** ADD ***ACCOUNT=***

> CREATE DATABASE ***DB*** FORM SHARE ***SHARE_OBJECT***

### Sharing view 

> CREATE SHARE ***SHARE_OBJECT***

- grant usage on the database in which our table is contained
>GRANT USAGE ON DATABASE ***DB*** TO SHARE ***SHARE_OBJECT***

- grant usage on the schema in which our table is contained
>GRANT USAGE ON SCHEMA ***SCHEMA*** TO SHARE ***SHARE_OBJECT***

- View requires this grant 
> GRANT REFERENCE_USAGE ON DATABASE ***DB*** TO SHARE ***SHARE_OBJECT***

> GRANT SELECT ON VIEW ***VIEW*** TO SHARE ***SHARE_OBJECT***

## Snowflake Access Management

Access control makes sure that only appropriate people have access. 

- Discretionary Access Control (DAC)
    - Each object has an owner and can grant access to that object
- Role Base Access Control (RBAC)
    - Access is through roles. Role are granted privileges and then the roles are granted to users & other roles

e.g.

> CREATE USER ***USER*** PASSWORD = "***PASSWORD***" DEFAULT_ROLE ***ACCOUNTADMIN*** MUST_CHANGE_PASSWORD = ***TRUE***
> GRANT ROLE ***ACCOUNTADMIN*** TO USER ***USER***

- Key Concept
    - User: human or machine
    - Role: An entity which privileges are granted
    - Privilege: level of access to an object
    - Securable Object: objects on which access can be granted e.g. tables, views etc

- SF provided roles:
    - SECURITY ADMIN:
        - Role that is used to create, monitor, and manage users and roles
    - SYS ADMIN:
        - Role which has rights to create DB, tables, warehouses and other objects
    - PUBLIC:
        - Default role
    - CUSTOM ROLES:
        - new roles can be defined by SECURITY ADMIN and is cannot be automatically granted and must be granted explicitly

### ACCOUNT ADMIN

Most powerful role, assignment of role should be very limited but assign at least 2 people in the organisation and this role should use 2FA. It encapsulates SECURITY ADMIN & SYS ADMIN

It can:
- View and manage all objects
- View billing and credit data
- Stop any SQL statements
- Perform other account level activities like creating reader accounts 

Don't use this role to create objects unless you ***HAVE*** to.

### SECURITY ADMIN

Used mainly for creating and managing users, roles, and the creation management of privileges. 

e.g. 
> CREATE ROLE ***ROLE***
> GRANT ROLE ***ROLE*** TO ROLE ***HIGHER_ROLE***
> GRANT ROLE ***HIGHER_ROLE*** TO ROLE SYSADMIN

### SYS ADMIN

Used for the creation and management of objects e.g. warehouses, DBS, tables & other objects. 











