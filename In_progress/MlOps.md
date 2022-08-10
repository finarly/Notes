# MlOps

## Infrastructure needs to address scale
### Scale by Service
Deploy and use services better suited to the task at hand
### Scale by Task Parallelisation
Split tasks and run in parallel, combining their outputs on completion of execution. Does thing quicker
### Scale by Code Efficiency
Refactoring code (pandas is resource hungry, bad when iterating through them). Maybe consider vectorisation.
### Scale by Processor
Bigger CPU (GCP: GPUs and TPUs)
### Scale by Memory
Throw more RAM at things

## ML Tooling
### Foundation: 
1. DS Notebooks (in this case Jupyter)
2. Accessing cloud resources
3. Containerisation

### Jupyter Tips
Use bqclient to access data in GCP 

### Pipelines, Kubernetes & Containers
Containers is really important for ML. ML 