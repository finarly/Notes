# Airflow

Airflow is a orchestrator. It is currently not a streaming solution and is not a data processing framework in a sense you shouldn't process anything in Airflow, you just use it to trigger other tools which will do so.


## Core Components

### Web Server

Flask Python web server that's a User Interface.

### Scheduler

Handles triggering tasks and pipelines. Important this works or else nothing works. 

### Metastore

A database that is compatible with SQL, e.g. Postgres, MySQL, Oracle etc. 

This database will have metadata related to your data, pipelines, tasks, airflow users etc.

### Triggerer

Allows the running of specific type of tasks...

TODO

### Executor

How and on which support tasks are executed, e.g. Kubernetes cluster -> Kubernetes Executor,
Celery cluster -> Celery Executor. 

The Executor doesn't execute any tasks. 

### Queue

Tasks will be pushed into the `queue` so they can be executed in the right order.

### Worker

The `worker` is where the tasks are executed. 


## Core Concepts

When all concepts below are combined together, you have a `workflow`.

### DAG

`Directed Acyclic Graph` there's nodes and edges and no cycles. 

### Operator

Pre-defined task where you can string together quickly to build most parts of your DAG.

#### Action Operator

An `Action Operator` executes something, e.g. Python Operator executes Python function, Bash Operator executes Bash command, Email Operator sends an email.  

#### Transfer Operator

Allows transferral of data from point A to point B, e.g MySQL to Redshift.

#### Sensor Operator

Special type of operator that is designed for one thing - to wait for something to occur. It can be time-based, or waiting for a file, or an external event, but they all wait for something to happen, and then *succeeds* so their downstream tasks can run. 

#### Task/Task Instance

An `operator` is a `task`, and when you run a `task`, you get a `task instance`.

## Architectures

### Single-node Architecture

Where you have a machine or node where the `web-server`, `meta-database/metastore`, as well as `scheduler` and `executor` are running.

![single_node](../Images/single_node.png)

Everything communicates with the `metadatabase` 

### Multi-node Architecture

To run Airflow in prod, you don't want a single-node Architecture as that might fail and ruin your whole pipeline. You want a highly available architecture which multi-node provides. 

![multi_node](../Images/multi_node.png)

- Node 1:
    - `Web-Server`
    - `Scheduler` + `Executor`

- Node 2:
    - `Metastore`
    - `Queue`

- Worker Node 1/2/3:
    - Airflow Worker: pulls work from Queue. 

With this architecture, if you need more execute resources, just add more Worker Nodes on a new machine. You should have at least 2 `Schedulers`, 2 `Web-Servers`, maybe a Load Balancer in from of your `Web-Servers` to deal with requests to Airflow UI, as well as a `PGBouncer` as a database proxy to deal with the number of connections that will be made to your `metastore`. 