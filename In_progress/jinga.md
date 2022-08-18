# Jinga, Macros, and Packages

## What is Jinga?

dbt is powered by SQL (for data transformations), yaml (configuring, documentating, and testing data transformation), and jinga (templating and lineage).

Jinga is a python based templating language that boosts up your SQL functionality. It uses two types of syntaxes. 

```
{% for i range (10) %}

    select {{ i }}...
```

"{%" means some operation is happening in the jinga context 
"{{" means we're pulling something out of the jinga context

An example is what emails use to fill names of recipients. 

This is seen in the ref function:

> {{ ref('stg_orders') }}

### Setting variables

We can use basic python data objects: dictionary, list, simple variables

#### Simple variable

This will set the variable
> {% set example_string = 'wowza' %}

> {% set example_number = 100 %}

This will print the variable
```
{{ example_string }}
And then I said {{ example_string }}
```

#### List

{% set my_animals = ['tiger', 'bear', 'wolf'] %}

{ my_animals[0] }

> tiger 
### Comment

> {# ... #}


