# Advanced Materialisations

## What is materialisations

Ways that dbt is going to build the models you write (a model is just a select statements).

## Tables

This actual table will be built in the database. It is create via a **create table as** statement.

### Pros: 
- Tables are fast to query
### Cons: 
- Tables can take a long time to rebuild, especially for complex transformations
- New records in underlying source data are not automatically added to the table
### Advice: 
- Use the table materialization for any models being queried by BI tools, to give your end user a faster experience
- Also use the table materialization for any slower transformations that are used by many downstream models

## Views

The query itself is what gets stored in the database. It is create via a **create view as** statement.

### Pros: 
- No additional data is stored, views on top of source data will always have the latest records in them.
### Cons: 
- Views that perform significant transformation, or are stacked on top of other views, are slow to query.
### Advice: 
- Generally start with views for your models, and only change to another materialization when you're noticing performance problems.
- Views are best suited for models that do not do significant transformation, e.g. renaming, recasting columns.

## Ephemeral

Don't exist in the database, it is like a reusable code snippet. dbt will interpolate the code from this model into dependent models as a common table expression.

### Pros: 
- You can still write reusable logic
- Ephemeral models can help keep your data warehouse clean by reducing clutter (also consider splitting your models across multiple schemas by using custom schemas).
### Cons: 
- You cannot select directly from this model.
- Operations (e.g. macros called via dbt run-operation cannot ref() ephemeral nodes)
- Overuse of the ephemeral materialization can also make queries harder to debug.
### Advice: Use the ephemeral materialization for:
- Very light-weight transformations that are early on in your DAG
- are only used in one or two downstream models, and
- do not need to be queried directly

## Incremental

Keeping the old table, just adding the new records. 

It costs time and money to transform data, and historical data doesnt generally change so you shouldn't re-transform historical data. 

Think about incremental models as an upgrade path:
- Start with view 
- If it tables too long to query, switch to using a table. It takes longer to build, but would be faster to query. 
- When that takes too long to build, switch to using incremental model

### How to create incremental table:

1. We have to tell dbt to add new rows instead of recreating the table.
- We need to use a different materialisation **materialized='incremental'**
2. How do we identify new rows on "subsequent" runs only (not the first time table is built)?
- We compare the source datta to the already-transformed data
> select  max(max_collector_tstamp) from {{ ref('page_views') }}
- To do so we can add an if cause using Jinja, so it 
```
{% if is_incremental() %}
where collector_tstamp >= (select max(max_collector_tstamp) from {{this}})
{% endif %}
```
- {{ this }} is the current existing database object mapped to this model. We use this because we don't want to ref and model to itself
- is_incremental(): Checks 4 conditions, if the answers to below conditions are **Yes Yes Yes No** then this is an incremental run.
    - Does this model already exist ias an object in the database?
    - Is that database obkect a table?
    - Is this model configured with **materialized='incremental'**?
    - Was the --full-refresh flag passed to this dbt run?

### Utility of full-refresh run

What if:
1. Data showerd up in the data warehouse late? Cut off time might mean we miss these data.
- We can widen the window to the last three days.
```
{% if is_incremental() %}
where collector_tstamp >= (select dateadd('day', -3,max(max_collector_tstamp)) from {{this}})
{% endif %}
```
- however...we'll get duplicate records. To fix this we change the configuration:
'''
{[ config(
    materializied='incremental',
    unique_key='page_view_id'
)]}

## Snapshots