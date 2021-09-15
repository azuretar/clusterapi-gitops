This folder will hold all YAML files of Workload Cluster's created. YAML files, like azuretar-reactor-1.yaml will be moved from Init folder after cluster creation. All day-2 operations of the clusters will be managed by GitOps configured on the AKS Management Cluster, once yaml file is moved on this folder.

You can find examples of these .YAML files at ./cluster-examples folder.

Cluster API templates can be found here: https://github.com/kubernetes-sigs/cluster-api-provider-azure/tree/main/templates
