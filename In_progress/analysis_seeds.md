# Analyses

Snippets of SQL or Jinja-fied SQL that you can test or compile without materialising them in your warehouse.SQL files that sit in Analyses folder that support Jinja, these are sql statements that don't fit the mold of a dbt model. It's like a scratch. They could be:
 
- One off queries 
- Scripts to create a new user
- Training queries (for practice)
- Auditing or refactoring of a project

# Seeds

Are .csv files that live in your dbt folder. The purpose is to take a small amount of data in a csv to manifest into your table in your data warehouse. To build these tables use

> dbt seed

Examples:
- Country codes (US for America, AUS for Australia)
- Employee id/emails

Seeds are not designed for large or frequently changing data, you probably want to use an API for that.  Seeds are for files that don't change that often. 

To add test or description its similar to models, just create a yaml file **schema.yml**

```
version: 2

seeds:
    - name: employees
      description: a manual map of employees to customers
      columns:
        - name: employee_id
          tests:
            - unique
            - not null
```

