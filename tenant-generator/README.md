# Tenant Generator Helm Chart

This chart will create tenants in clusters. This handles the creation of the service account, kustomization and gitrepo for each tenant. 


## Usage

There are only a few variables that need to be set.

`cluster_name` -  the name of the cluster , this will be used to set the path to the tenant git repo folder for the kustomization
`tenants` - this is an array of tenants to create
`tenants.namespace` - the namespace for this tenant
`tenants.name` - the name of the tenant, this is used for the names of the objects created
`tenants.git_repo_url` -  the url of the git repo for the tenant


