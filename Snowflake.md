# Snowflake

## 1. Built for the cloud

- Built from scratch
- Optimized for cloud
- Storage & compute is decoupled

## 2. Software as a Service

No software, infrastructure or upgrades to manage

## 3. Pay only for used compute & storage

Storage & compute charged independently, only for us

## 4. Scalable

Virtual warehouse enable compute scaling

## Pricing Overview
- Pricing is based on the amount of data stored and the amount of compute used. Since compute and storage is decoupled, they can be billed separately
- Storage is charged monthly, and is based on the average data stored
- Compute is calculated via credits 
- No cost for data transfer into Snowflake, but it costs to transfer out of Snowflake or between regions 

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

