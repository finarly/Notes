# dbt Training

## Data Platform

dbt is designed to do 'transformation' in 'ELT' process for data platforms (snowflake, databricks, bigquery, redshift etc.)

dbt creates connection to DP and runs SQL code against the warehouse to transform data.

## Version Control

dbt alllows developers to use version controlling to manage their code base.

## What is Modeling in data analytics

It is shaping of the data from raw data through to your final transformed data. **Models in dbt are just SQL select statements.** Each represents one modular piece of logic that will take raw to final data. These are SQL files under model folder. Each model maps 1 to 1 to a table or view in the data warehouse.

- Add .sql extension to files under Models folder
- materialize="table" in config makes dbt buid table rather than view

### Traditional Modelling

- Star schema
- Kimball
- Data vault

These were created when storage was expensive, and they use the idea of normalised modelling, where the number of times of data is limited in duplication. 


### Modularity

Building a final product piece by piece and combining them rather than all at once. So after you have transformed the data into a "model" you can use it again later on.
 
### Model Naming Conventions

#### 1. Sources

Arent actually models, its a way of referencing raw tables that are actuall loaded. 

#### 2. Staging

To clean and standardise the data, should be build 1 to 1 with source tables

#### 3. Intermediate

Exist between staging and final models, and be built on staging models (not source). You might want to hide these from final users. 

#### 4. Fact

Skinny table that represent things that have occured or are occuring. Things that will keep happening over time e.g. clicks, events 

#### 5. Dimension

Things are exist, things that are e.g. customer, user, products 

