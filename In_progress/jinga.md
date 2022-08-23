# Jinja

## What is Jinja?

dbt is powered by SQL (for data transformations), yaml (configuring, documentating, and testing data transformation), and Jinja (templating and lineage).

Jinja is a python based templating language that boosts up your SQL functionality. It uses two types of syntaxes. 

```
{% for i range (10) %}

    My favourite number is {{ i }}

{% endfor %}}
```

"{% ... %}" means some operation is happening in the Jinja context 
"{{ ... }}" means we're pulling something out of the Jinja context

An example is what emails use to fill names of recipients. 

This is seen in the ref function:

> {{ ref('stg_orders') }}

## Jinja basics

### Setting variables

We can use basic python data objects: dictionary, list, simple variables

#### Simple variable

This will set the variable
```
{% set example_string = 'wowza' %}

{% set example_number = 100 %}
```

This will print the variable
```
{{ example_string }} 
> wowza

And then I said {{ example_string }}
> And then I said wowza
```

#### List

```
{% set my_animals = ['tiger', 'bear', 'wolf'] %}

{ my_animals[0] }
```

> tiger 

#### Dictionary

(% set websters_dict = {
    'word': 'data',
    'type': 'noun',
    'definition': 'abc'
} %)

### Comment

> {# ... #}

### For loop

```
{% for i range (10) %}

    My favourite number is {{ i }}

{% endfor %}}
```

### If..else...then statement

```
{% set temperature = 75 %}
{% if temperature <65 %}
    Time for a cappucino!
{% else %}
    Time for a cold brew!
{% endif %}
```

### Remove whitespace

To remove whitespace as our code gets translated to whitespace in the output, add a '-' sign next to the '%' in the Jinja blocks.

e.g.

```
{%- set temperature = 75 -%}
```

# Macros

## What are macros?

Macros allow you to write generic logic which can then be reuseable. It's like modularising dbt models or creating functions in a file and calling it in another file. 

1. Create new file under macro folder

2. Example

``` 
{% macro cents_to_dollars(column_name, decimal_places) %}
round( 1.0* {{ column_name }}/100, {{ decimal_places }})
{% endmacro %}
```

3. In SQL code, call the function as

```
{$ cents_to_dollars('amount', 4) %}
```

4. The above example is probably unnecessary as it makes it harder than the original code which was 

```
amount/100
```

## DRY vs readability

Macros allows DRY code (Don't Repeat Yourself). Essentially everything is going to be tucked away in different files and its going to be hard to understand the logic times. Therefore you have to find a good middle point. 

# Packages 

When you're doing something, you might think "Someone's definitely done this or figured this out before, right?" that's when packages come in. Projects are ways to bring in **models and macros** into your project.

Macros packages would be functions that are very helpful for your tables. 

Model packages would be models that are built on top of popular data sets.

## Installing packages 

Create a file called "packages.yml" and include packages in there. There are some different ways to specify packages in yml file:

1. Use dbt Hub
2. Straight from GitHub
3. From your local

```
packages:
- packages: fivetran/github
  version: 0.1.1
- git: (GitHub URL)
  revision: master
- local: sub-project
```

This will install your packages: 
> dbt deps

# Advanced Jinja + Macros

You can create Macro to grant privileges in dbt.