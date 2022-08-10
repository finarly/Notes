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

## Traditional Modelling

- Star schema
- Kimball
- Data vault

These were created when storage was expensive, and they use the idea of normalised modelling, where the number of times of data is limited in duplication. 


## Modularity

Modularity is the degree to which a system's components may be separated and recombined.

Building a final product piece by piece and combining them rather than all at once. So after you have transformed the data into a "model" you can use it again later on.
 
## Model Naming Conventions

### 1. Sources

Arent actually models, its a way of referencing raw tables that are actually loaded. 

Raw tables can be referenced like *raw.jaffle_shop.customers*

But better practice would be configuring it in a yaml file 

e.g.

```
version: 2

sources:
  - name: jaffle_shop
    database: raw
    schema: jaffle_shop
    tables:
      - name: orders
        loaded_at_field: _etl_loaded_at
        freshness:
          warn_after: {count: 12, period: hour}
          error_after: {count: 24, period: hour}
```
### 2. Staging

To clean and standardise the data, should be build 1 to 1 with source tables

### 3. Intermediate

Exist between staging and final models, and be built on staging models (not source). You might want to hide these from final users. 

### 4. Fact

Skinny table that represent things that have occured or are occuring. Things that will keep happening over time e.g. clicks, events 

### 5. Dimension

Things are exist, things that are e.g. customer, user, products 

## Testing

dbt allows scaling of tests accross the project so you get coverage and you can find out when things break.

Tests are assertions you have about your data. It is critical that these assertions are true.

Testing at scale is difficult, but dbt can do it with: 

'''
dbt test runs all tests in the dbt project
dbt test --select test_type:generic
dbt test --select test_type:singular
dbt test --select one_specific_model
'''

You can write a test which will test against models in development, and as the model gets pushed to production, the test can continuously scheduled to run against the model. 

e.g.

> models/staging/jaffle_shop/stg_jaffle_shop.yml

```
version: 2

models:
  - name: stg_customers
    columns: 
      - name: customer_id
        tests:
          - unique
          - not_null

  - name: stg_orders
    columns:
      - name: order_id
        tests:
          - unique
          - not_null
      - name: status
        tests:
          - accepted_values:
              values:
                - completed
                - shipped
                - returned
                - return_pending
                - placed
```

### Singular test

Very specific, apply to probably 1 or 2 models. 

### Generic test

Very simple and scalable tests. You can either use packages, write your own, or use the ones that dbt innately have:

- unique
- not_null
- accepted_values: value in column is one of the provided values
- relationships: tests to ensure that every value in a column exists in a column in another model


## Documentation

dbt tries to put the code and the documentation as close as possible, in the yml in the dbt repo. You can use **description** as one of the levels in the yml file. This can be just a string, or it can reference a markdown file with more detailed description (Doc Block). Description can be added on a database & schema, table, or column level.

```
version: 2

models:
  - name: stg_customers
    description: Staged customer data from our jaffle shop app.
    columns: 
      - name: customer_id
        description: The primary key for customers.
        tests:
          - unique
          - not_null

  - name: stg_orders
    description: Staged order data from our jaffle shop app.
    columns: 
      - name: order_id
        description: Primary key for orders.
        tests:
          - unique
          - not_null
      - name: status
        description: "{{ doc('order_status') }}"
        tests:
          - accepted_values:
              values:
                - completed
                - shipped
                - returned
                - placed
                - return_pending
      - name: customer_id
        description: Foreign key to stg_customers.customer_id.
        tests:
          - relationships:
              to: ref('stg_customers')
              field: customer_id
```

To update documentation:

> dbt docs generate

Doc block:

```
{% docs order_status %}
	
One of the following values: 

| status         | definition                                       |
|----------------|--------------------------------------------------|
| placed         | Order placed, not yet shipped                    |
| shipped        | Order has been shipped, not yet been delivered   |
| completed      | Order has been received by customers             |
| return pending | Customer indicated they want to return this item |
| returned       | Item has been returned                           |

{% enddocs %}
```

DAG (directed acyclic graph) is automatically generated (through ref function) to show flow of data from source to final models.

## Deployment

Running a set of commands on a schedule that powers your business. There is usually a dedicated production branch (master or main), a dedicated production schema (e.g. dbt_production). 

