# dbt Training

## Data Platform

dbt is designed to do 'transformation' in 'ELT' process for data platforms (snowflake, databricks, bigquery, redshift etc.)

dbt creates connection to DP and runs SQL code against the warehouse to transform data.

## Version Control

dbt alllows developers to use version controlling to manage their code base.

## What is Modeling in data analytics

It is shaping of the data from raw data through to your final transformed data. Models in dbt are just SQL select statements. Each represents one modular piece of logic that will take raw to final data. These are SQL files under model folder. Each model maps 1 to 1 to a table or view in the data warehouse.

- Add .sql extension to files under Models folder
- materialize="table" in config makes dbt buid table rather than view
