# Refactoring SQL for Modularity

## 1. Migrating legacy code

Create a sub-folder under **Models** called **Legacy**. Create sql file under **Legacy**.

Try running code. Note you might need to change your script due to there being differnt flavours of SQL. 

## 2. Implmenting sources

Create **_source.yml** files for each of your models and then update sql script by updating hardcoded table refernces with source functions. 

## 3. Choosing a refactoring strategy

Either create a new branch in git and refactor that. Or create a copy and put it in your desired location and then edit the code there. 

## 4. CTE groupings and cosmetic cleanups

### Cosmetic

- Add white spacing between **SELECT** and **FROM** in sql script.
- Break up long lines so you dont have to scroll right.
- Consistent case keywords (can do automatically in dbt)

### CTE Groupings

Example structure:

```
-- with statement

    with

-- import CTEs (for each source table that the query uses)

    customers as (

        select * from {{ source('jaffle_shop', 'customers') }}

    ),

    orders as (

        select * from {{ source('jaffle_shop', 'orders') }}

    ),

    payments as (

        select * from {{ source('stripe', 'payment') }}

    ),


-- logical CTEs (move subqueries into their own CTE here)

    customers as (

    select 

        first_name || ' ' || last_name as name, 
        * 

    from customers

),

    a as (

        select 

            row_number() over (
                partition by user_id 
                order by order_date, id
            ) as user_order_seq,
            *

        from orders

    ),

    b as ( 

        select 

            first_name || ' ' || last_name as name, 
            * 

        from customers

    ),

    customer_order_history as (

        select 

            b.id as customer_id,
            b.name as full_name,
            b.last_name as surname,
            b.first_name as givenname,

            min(order_date) as first_order_date,

            min(case 
                when a.status not in ('returned','return_pending') 
                then order_date 
            end) as first_non_returned_order_date,

            max(case 
                when a.status not in ('returned','return_pending') 
                then order_date 
            end) as most_recent_non_returned_order_date,

            coalesce(max(user_order_seq),0) as order_count,

            coalesce(count(case 
                when a.status != 'returned' 
                then 1 end),
                0
            ) as non_returned_order_count,

            sum(case 
                when a.status not in ('returned','return_pending') 
                then round(c.amount/100.0,2) 
                else 0 
            end) as total_lifetime_value,

            sum(case 
                when a.status not in ('returned','return_pending') 
                then round(c.amount/100.0,2) 
                else 0 
            end)
            / nullif(count(case 
                when a.status not in ('returned','return_pending') 
                then 1 end),
                0
            ) as avg_non_returned_order_value,

            array_agg(distinct a.id) as order_ids

        from a

        join b
        on a.user_id = b.id

        left outer join payments as c
        on a.id = c.orderid

        where a.status not in ('pending') and c.status != 'fail'

        group by b.id, b.name, b.last_name, b.first_name

    ),

-- final CTE

    final as (

        select 

            orders.id as order_id,
            orders.user_id as customer_id,
            last_name as surname,
            first_name as givenname,
            first_order_date,
            order_count,
            total_lifetime_value,
            round(amount/100.0,2) as order_value_dollars,
            orders.status as order_status,
            payments.status as payment_status

        from orders

        join customers
        on orders.user_id = customers.id

        join customer_order_history
        on orders.user_id = customer_order_history.customer_id

        left outer join payments
        on orders.id = payments.orderid

        where payments.status != 'fail'

    )

-- simple select statement

    select * from final

```

## 5. Centralising transformations & splitting up models

For more detailed breakdown [Refactoring SQL]()

1. Staging models

Look at sources, ignoring the import CTEs, staging CTE aim to transform just the source without any joins. 

2. Marts model

These CTEs conduct joins.

3. Remove redundant CTEs

Remove any redudant CTEs that have already been covered in the previous 2 steps. Also replace any references to removed CTEs

4. Moving transformation from Marts to Staging

If the transformation can be done with done data set (pre-join), and if it is done on a field whose value is not due to a join, move the transformation to the approviate CTE under --staging. 

Make sure that when you move these, you are:
- Removing redundant transformations
- Re-referencing the CTE and field names correctly
- Giving good names to fields that dont have a good name estabished

5. Focusing on CTEs and moving logic to intermediate models

6. Final Model

## 6. Audting 

Making sure that the data that we get now that the code is refactored is the same as the results before. 

You can use packages to help audit. 

To install package, include package in package.yml file and run

> dbt deps

to install the package.