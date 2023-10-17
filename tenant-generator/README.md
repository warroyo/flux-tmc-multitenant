# Tenant Generator Helm Chart

This chart will create tenants in clusters. 


## Usage

there are two types of tenants that this can generate.

1. workload tenants -  this is for use with the clusters that tenants will deploy on. this will be most commonly used. 
2. infra tenants - this is for use with [infra ops clusters](https://github.com/warroyo/flux-tmc-multitenant#explaining-the-infra-ops-cluster)

to set the cluster type use the top level variable `cluster_type` this accepts either `workload` or `infra` as a value.

### Workload Clusters

This handles the creation of the service account, kustomization and gitrepo for each tenant. There are only a few variables that need to be set. This should be used for clusters where tenants will be deploying workloads. 
`workload_cluster.cluster_name` -  the name of the cluster , this will be used to set the path to the tenant git repo folder for the kustomization
`workload_cluster.tenants` - this is an array of tenants to create
`workload_cluster.tenants.namespace` - the namespace for this tenant
`workload_cluster.tenants.name` - the name of the tenant, this is used for the names of the objects created
`workload_cluster.tenants.git_repo_url` -  the url of the git repo for the tenant


### Infra Clusters

This handles the creation of the service account, git repo, rolebinding, kustomizations and intial TMCns per cluster. This should be used for infra ops clusters

`infra_cluster.tenants` - array of tenants for the cluster
`infra_cluster.tenants.namespace` - the namespace for this tenant
`infra_cluster.tenants.name` - the name of the tenant, this is used for the names of the objects created
`infra_cluster.tenants.git_repo_url` -  the url of the git repo for the tenant
`infra_cluster.tenants.workspace` -  the workspace that this tenant is a part of
`infra_cluster.tenants.clusters` -  array of clusters to create the tenant in.
`infra_cluster.tenants.clusters.name` -  the cluster name
`infra_cluster.tenants.clusters.mgmt_cluster` -  the name of the TMC mgmt cluster for the cluster
`infra_cluster.tenants.clusters.provisoner` - the name of the TMC provisioner for the cluster