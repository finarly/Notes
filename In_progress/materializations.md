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

#### What if:
Data showed up in the data warehouse late? Cut off time might mean we miss these data.
- We can widen the window to the last three days.
```
{% if is_incremental() %}
where collector_tstamp >= (select dateadd('day', -3,max(max_collector_tstamp)) from {{this}})
{% endif %}
```
- however...we'll get duplicate records. To fix this we change the configuration:
```
{[ config(
    materializied='incremental',
    unique_key='page_view_id'
)]}
```

#### Why 3 days?
The goal of incremental tables is it approximates the true table with a fraction of the runtime. "Close enough & performant". To decide on how many days you'll need to:
- Perform an analysis on the arrival time of data
- Figure out your organisation's tolerance for correctness
- Set the cutoff based on these two inputs
- Once a week (or however many days), perform a --full-refresh run to get the 'true' table 

#### How do things for apart
Sometimes the cut off doesnt cover all the data even with the 3 days, so we may end up under counting, which may not be a big problem since we can do a full refresh. 

What if we want to calculate window functions? This would be a problem since we are only calculating using 3 days of data. To fix this we can:

Always correct & slow:
- If a user has a new event, recalculate all page views for that user

Too clever by half:
- When a user has a new session, pull the user's mopst recent session only and preform relative calculations

### What about truly massive datasets?

Bigger datasets are a different cost-optimisation problems
- Always rebuild past 3 days. Fully ignore late arrivals: maybe you've got too much data and its too big of a problem to try and fix it.
- Always replace data at the partition level
- No unique keys: Merge is way more expensive to run than a straight up insert. 
- Don't bother with smart ways to scan data 

### Should we use incremental model?

#### Good candidates
- Immutable events streams: tall + skinny table, append-only, no updates
- If there are any updates, a reliable **updated_at** field

#### Not so good
- You have small-ish data
- Your data changes constantly: new columns, rename columns etc
- Your data is updated in unpredicatable ways
- Your transformation performs comparisons or calculations that require other_rows (table + view)

#### Incremental models introduce tradeoffs:
- Most incremental models are "approximately correct"
- They introduce a level of code complexity (i.e. how easy it is for someone to understand your code)
- Prioritising correctness can negate performance gains from incrementality. 

*Think of incremental models as an upgrate, not a starting point. Start with views, then tables, then if it takes too long to build, use incremental models.*

#### Tips
Keep the inputs and transformations of your incremental models as singlar, simple, and immutable as posible.
- Slowly changing dimensions, like a product_name that the company reglarly rebards? Join from a dim table in a downstream model rather than have to go back and reprocess. 
- Window functions for quality of life counters? Fix it in post - i.e. a separate downstream model as well

## Snapshots

In dbt, snapshots are select statements, defined within a snapshot block in a sql file, typically sitting in the **snapshots** folder. 

```
{% snapshot orders_snapshot %}

{{
    config(
      target_database='analytics',
      target_schema='snapshots',
      unique_key='id',

      strategy='timestamp',
      updated_at='updated_at',
    )
}}

select * from {{ source('jaffle_shop', 'orders') }}

{% endsnapshot %}
```

Snapshots allow you to look at the changes done in a particular row. To create a snapshot run:

> dbt snapshots

- On the first run, dbt will create the initial snapshot table and this will result your select statement, with additional columns including **dbt_valid_from** and **dbt_valid_to**. All recrods will have a **dbt_valid_to=null**.
- In subsequent runs, dbt will check records that have changed or if any new records have been created and the dbt_valid_to column will be updated for any existing records that have been changed, and updated records and any new records will be inserted into the snapshot table. These new records will now have **dbt_valid_to=null**

These tables can be reference by using the ref function. 

### Tips

- Use the **timestamp** strategy where possible, it handles column additions and deletions better than the **check_cols** strategy.
- Ensure your unique key is truly unique as that is what dbt uses to match up rows
- Use a target schema that is separate to your analytics schema because snapshots cannot be rebuilt, so its better to put it in a separate schema for end users to know that they are special. 
- Snapshot source data 
- Keep query as possible 
