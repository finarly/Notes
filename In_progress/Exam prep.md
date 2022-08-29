# Exam prep

## Viewpoint

### Analytics is collaborative

- Version control: code needs version control
- Quality assurance: code needs to be reviewed
- Documentation: there needs to be definitions
- Modularity

### Analytic code is an asset

We believe a mature analytics organization’s workflow should have the following characteristics so as to protect and grow that investment:

- Environments: analytics require multiple environments
- Service level guarantees: 
- Design for maintainability: future changes to the schema and data should be anticipated and code should be written in a way to minimise the corresponding impact. 

## Sources

### Source properties

Source properties are declared in .yml files sitting in models/ directory. These files can be named whatever 

### Source configuration

Sources only support one configuration, enabled. They can be configured via a **config:** block within their .yml definitions, or from the dbt_project.yml file under the **sources:** key. 

When a resource is disabled, dbt will not consider it as a part of your project, though this may cause compilation errors. IF you want to exclude sometha model in a particuaring 

This could be useful when you disable a model in a package to use your own, or to disable source freshness checks running on source tables from packages. 

### database 

The database your source is stored in.

```
version: 2

sources:
  - name: <source_name>
    database: <database_name>
    tables:
      - name: <table_name>
      - ...
```

### external

A dictionary of metadata specific to sources that point to external tables. These are optional built in properties.

### freshness

The acceptable amount of time between the most recent record, and now, for a table to be considered "fresh"

It has:
- **warn_after**
- **error_after**
If neither are provided then dbt won't calculate freshness.

Addtionally, the **loaded_at_field** is requred to calculate freshness for a table, if it isnt provided then freshness wont be caluclated. 

Freshness blocks are applied hierarchically:
- a freshness and loaded_at_field property added to a source will be applied to all all tables defined in that source
- a freshness and loaded_at_field property added to a source table will override any properties applied to the source.

models/<filename>.yml
```
version: 2

sources:
  - name: jaffle_shop
    database: raw

    freshness: # default freshness
      warn_after: {count: 12, period: hour}
      error_after: {count: 24, period: hour}

    loaded_at_field: _etl_loaded_at

    tables:
      - name: customers # this will use the freshness defined above

      - name: orders
        freshness: # make this a little more strict
          warn_after: {count: 6, period: hour}
          error_after: {count: 12, period: hour}
          # Apply a where clause in the freshness query
          filter: datediff('day', _etl_loaded_at, current_timestamp) < 2
```

### identifier

The table name as stored in the database. By default dbt will use the table's **name:** but the identifier parameter is useful if you want to use a source table name that differs from the table name in the database. 

models/<filename>.yml
```
version: 2

sources:
  - name: jaffle_shop
    tables:
      - name: orders
        identifier: api_orders
```

In a downstream model:
```
select * from {{ source('jaffle_shop', 'orders') }}
```

Will get compiled to:
```
select * from jaffle_shop.api_orders
```

### loader (documentation only)

Describe the tool that loads this source into your warehouse, this property is for documentation purposes only. It is not used in any meaningful way. 

### quoting

Optionally condigure whether dbt should quote databases, schemas, and identifiers when resolving a {{ source () }} function to a direct relation reference. 

models/<filename>.yml
```
version: 2

sources:
  - name: jaffle_shop
    database: raw
    quoting:
      database: true
      schema: true
      identifier: true

    tables:
      - name: orders
      - name: customers
        # This overrides the `jaffle_shop` quoting config
        quoting:
          identifier: false
```

In a downstream model:
```
select
  ...

-- this should be quoted
from {{ source('jaffle_shop', 'orders') }}

-- here, the identifier should be unquoted
left join {{ source('jaffle_shop', 'customers') }} using (order_id)
```
This will get compiled to:
```
select
  ...

-- this should be quoted
from "raw"."jaffle_shop"."orders"

-- here, the identifier should be unquoted
left join "raw"."jaffle_shop".customers using (order_id)
``` 

### schema

The schema name as stored in the database. THis parameter is useful if you want to use a source name that differes from the schema name. by defualt dbt will use the source's **name:** parameter as the schema name.

models/<filename>.yml
```
version: 2

sources:
  - name: jaffle_shop
    schema: postgres_backend_public_schema
    tables:
      - name: orders
```

In a downstream model:
```
select * from {{ source('jaffle_shop', 'orders') }}
```
Will get compiled to:
```
select * from postgres_backend_public_schema.orders
```

### overrides

Override a source defined in an included package. The properties defined in the overriding source will be applied on top of the base properties of the overridden sources.

The following source properties can be overridden:
- description
- meta
- database
- schema
- loader
- quoting
- freshness
- loaded_at_field
- tags

## Syntax overview

### Arguments

#### '--selector'

You can write resources selectors in YAML, save them and reference them using --selector. 

By recording selectors in a top-level **selectors.yml** file:

- Legibility: complex selection criteria are composed of dictionaries and arrays
- Version control: selector definitions are stored in the same git repository as the dbt project
- Reusability: selectors can be referenced in multiple job definitions, and their definitions are extensible (via YAML anchors)

Selectors live in a top-level file named selectors.yml. Each must have a name and a definition, and can optionally define a description and default flag.

selector.yml
```
selectors:
  - name: nodes_to_joy
    definition: ...
  - name: nodes_to_a_grecian_urn
    description: Attic shape with a fair attitude
    default: true
    definition: ...
```

The definition comprises one or more arguments, which can be:
- CLI-style: strings, representing CLI style arguments. This simple syntax supports use of the +, @, and * operators. Does not support exclude.
```
definition:
    'tag:nightly'
```

- Key-value: pairs in the form 'method: value'. This simple syntax does not support any operators or exclude.
```
definition:
  tag: nightly
```

- Full YAML: fully specified dictionaries with items for method, value, operator-equivalent keywords, and support for exclude.
```
definition:
  method: tag
  value: nightly

  # Optional keywords map to the `+` and `@` operators:

  children: true | false
  parents: true | false

  children_depth: 1    # if children: true, degrees to include
  parents_depth: 1     # if parents: true, degrees to include

  childrens_parents: true | false     # @ operator
  
  indirect_selection: eager | cautious  # include all tests selected indirectly? eager by default
```

#### '--select' or '-s'

By default dbt run/seed/snapshot will execute all nodes. Is used to specify a subset of nodes to execute (a particular model/seed/snapshot). 

How does selection work?
1. dbt gathers all the resources that are matched by one or more of the --select criteria, in the order of selection methods (e.g. tag:), then graph operators (e.g. +), then finally set operators (unions, intersections, exclusions).

2. The selected resources may be models, sources, seeds, snapshots, tests. (Tests can also be selected "indirectly" via their parents; see test selection examples for details.)

3. dbt now has a list of still-selected resources of varying types. As a final step, it tosses away any resource that does not match the resource type of the current task. (Only seeds are kept for dbt seed, only models for dbt run, only tests for dbt test, and so on.)

### run

e.g.
> dbt run --full-refresh

Arguments:
- '--select'
- '--exclude'
- '--selector'
- '--defer'

It executes compiled sql model files against the current target database. dbt connects to the target database and runs the releveant SQL required to materialise all data models using the specified materialisation strategies. Models are run in the order defined by the dependency graph generated during compilation. Intelligent multi-threading (things running synchronously) is used to minimize execution time without violating dependencies.

You can provide '--full-refresh' argument to dbt run, which will treat incremental models as table models, and it is useful when
1. The schema of an incremental model changes and you need to recreated it
2. You want to reprocess the entirety of the incremental model because of new logic in the model code.

### test

e.g.
> dbt test --select one_specific_model,test_type:singular

Arguments:
- '--select'
- '--exclude'
- '--selector'
- '--defer'

It runs tests defined on models, sources,snapshots, and seeds. It expects that you have already created those resources through teh appropriate commands. You can use --select to run particular tests, or run tests for particular models, or run particular tests for particular models.

### seed

e.g.
> dbt seed --select country_codes

The seed command will load csv files located in the seed-paths directory of your dbt project into your data warehouse. 

### snapshot

The snapshot command executes the snapshots defined in your project. In dbt, snapshots are select statements, defined within a snapshot block in a .sql file (typically in your snapshots directory).

dbt will look for Snapshtos in the snapshot-path defined in your dbt_project.yml file. By default, the snapshot-paths path is **snapshot/**. That are you can put the config block into the your snapshot .sql file. 

### ls (list)

Arguments:

- '--resource-type': This flag limits the "resource types" that dbt will return in the dbt ls command. By default, the following resources are included in the results of dbt ls: models, snapshots, seeds, tests, and sources.
- '--select': This flag specifies one or more selection-type arguments used to filter the nodes returned by the dbt ls command
- '--models': Like the --select flag, this flag is used to select nodes. It implies --resource-type=model, and will only return models in the results of the dbt ls command. Supported for backwards compatibility only.
- '--exclude': Specify selectors that should be excluded from the list of returned nodes.
- '--selector': This flag specifies one or more named selectors, defined in a selectors.yml file.
- '--output': This flag controls the format of output from the dbt ls command.
- '--output-keys': If --output json, this flag controls which node properties are included in the output.

This command list resources you have in your dbt project. It accepts selector arguments that are similar to those in dbt run. 

### compile

Arguments:
- '--select'
- '--exclude'
- '--selector'

Generates executable SQL form source model, test, and analysus files. You can find these compiled SQL files in the **target/** directory of your dbt project. 

The compile command is usefil for:
1. Visually inspecting the compiled output of model files. This is useful for validating complex jinja logic or macro usage. 
2. Manually running compiled SQL. While debigging a model ro schema test, it's often useful to execute the underlying select statement to find the source of the bug. 
3. Compiling analysrs files. 

It is not a pre-requisite of dbt run.

### freshness 

e.g. Snapshot freshness for all Snowplow tables:
> dbt source freshness --select source:snowplow

Snapshot freshness for a particular source table:
> dbt source freshness --select source:snowplow.event

To check freshness of source files. When it finishes running, a JSON file containing info on freshness will be created at **target/sources.json**. To override the location of output use -o or --output

>  dbt source freshness --output target/source_freshness.json

### build

This command will:
- run models
- test tests
- snapshott snapshots
- seed seeds 

Artifacts: build will create a single manifest and a single run results artifact. It will tell you all about models, tests, seeds, and snapshots that were selected to build, combined into one file. 

Skipping on failures: Tests on upstream resources will block downstream resources from running, and a test failure will cause those downstream resources to skip entirely. E.g. if model_b depends on model_a, and a unique test on model_a fails, then model_b will SKIP. 
  
  - If you dont want a test to cause skipping then adjust severity or thresholds to *warn* instead of *error*.
  -  In case of a test with multiple parents, where one parent depends on the other (e.g. a relationship test between model_a + model_b), that thest will block and skip children of the most downsteram parent only (model_b)

Selecting resources: the build task supports standard selection syntax (--select, --exclude, --selector), as well as a --resource-type flag thbat offers a final filter (just like *list*). WHichever resources are selected, those are the ones that build execute. 

  - Tests support indirect selection, so dbt build -s model_a will both run and test model_a. What does thatt mean? Any tests that directly depend on model_a will be included, so long as those tests dont also depend on other unselected parents. 

Flags: the build task supports all the same flags as run, test, snapshot, and seed. For flags that are shared between mutiple tasks (e.g. --full-refresh ), build will use the same value for all selected resource types (e.g. both models and seeds will be full refreshed)

## General properties

### columns

e.g. models
```
version: 2

models:
  - name: <model_name>
    columns:
      - name: <column_name>
        data_type: <string>
        [description](description): <markdown_string>
        [quote](quote): true | false
        [tests](resource-properties/tests): ...
        [tags](resource-configs/tags): ...
        [meta](resource-configs/meta): ...
      - name: <another_column>
        ...
```

They are not resources in and of themselves, and instead they are child properties of another resource type. They can define sub-properties that aresimilar to properties defined at the resource level:
- tags
- meta 
- tests 
- description

As columns are not resources, *tags* and *meta* properties are not true configurations. They do not ingerit the *tags* or *meta* values of their parent resources. 

However, you can select a generic test, defined on a column, using tags applied to its column or top-level resources. 

```
version: 2

models:
  - name: orders
    columns:
      - name: order_id
        tests:
        tags: [my_column_tag]
          - unique
```

> $ dbt test --select tag:my_column_tag

Columns may also optionally define *data_type*. This is for metadata purposes only, such as to use alongside the *external* property of sources. 

### config

e.g.
```
version: 2

models:
  - name: <model_name>
    config:
      [<model_config>](model-configs): <config_value>
      ...
```

The config property allows your to configure resources at the same time you're defining properties in yaml files. 

### description

A user defined description can be used to document:
- a model, and model columns
- sources, source tables, and source columns
- seeds, and seed columns
- snapshots, and snapshot columns
- analyses, and analysis columns
- macros, and macro arguments

These description are used by the documentation website rendered by dbt. It can be a simple description, a multi-line description, or a doc block:

e.g. doc-block

models/schema.yml
```
version: 2

models:
  - name: fct_orders
    description: This table has basic information about orders, as well as some derived facts based on payments

    columns:
      - name: status
        description: '{{ doc("orders_status") }}'
```

models/docs.md
```
{% docs orders_status %}

Orders can be one of the following statuses:

| status         | description                                                               |
|----------------|---------------------------------------------------------------------------|
| placed         | The order has been placed but has not yet left the warehouse              |
| shipped        | The order has ben shipped to the customer and is currently in transit     |
| completed      | The order has been received by the customer                               |
| returned       | The order has been returned by the customer and received at the warehouse |


{% enddocs %}
```
You link another model to the description or include links to images. 

### quote

The *quote* field can be used to enable or disable quoting for column names. Default is false. 

Quoting may be relevant to using SnowFlake, when:
- a source table has a column that needs to be quoted to be selected, for example, to preserve column casting. 
- 

## Miscellaneous

### Flags

Flags variables contains values of flags provided on the command line. 

```
{% if flags.FULL_REFRESH %}
drop table ...
{% else %}
-- no-op
{% endif %}
```
