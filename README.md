# Flux TMC Multi-tenant

This repo serves as a starting point for managing multi-tenant clusters using Flux and TMC.



## Roles

**Platform Admin**

* Has admin access to TMC
* Has admin access to all k8s clusters
* Manages the "infra-ops" cluster
* Manages cluster wide resources
* Onboards the tenant’s main `GitRepository` and `Kustomization`


**Tenant**

* Has self service namespace creation
* Has admin access to the namespaces they create 
* Manages `GitRepositories`, `Kustomizations`, `HelmRepositories` and `HelmReleases` in their namespaces
* has editor access into thier TMC workspace


## Repo structure

The platform admin repo has the following directories

* clustergroups - this contains directories with Flux config for each group. This is where the base bootstrap config lives. This is used when initially configuring a cluster group's `Kustomization` in TMC. Everything is initiated from here.
* clusters - this contains directories with Flux config for each cluster. This is what will hold each clusters base configuration that gets boostrapped from the cluster group.
* infrastructure - contains Flux config to install common infra tools used by multiple clusters.
* apps - 


The tenant repo has the following directories

* clusters -  this has a directory for each cluster. this will be the reference point for the tenants configured in the platform admin repo. This is used as a bootstrap for the tenant to configure their own kustomizations.
* clusters/<clustername>/namespaces - this nested directory is where tenants place thier namespace yaml for namespace self service.




## Initial Setup
These steps walk through a somewhat opinionated way of organzing things. This is not the only way of doing it and is just an example. Also for any infrastructure created it's advised that this be automated rather than being done by hand, n this example we will use the cli for everything we can. If you already have existing infra, it's not necessary to create new clusters etc. just use those. Also for the purpose of this setup we will assume that we have a platform team and 3 tenants. Those tenants are all within the same product group but are different app teams. our product group name is Iris, so we will have a setup where each team gets a workspace in TMC and k8s clusters will be grouped by environment into cluster groups, each product group in this case will have a cluster(s) per environment. Our three dev teams will be called iris-green, iris-red, iris-blue.

### Create initial cluster groups

For this setup we will have the following cluster groups. Since we are implementing multitenancy we will also assume that there will be multiple dev teams using a single cluster separated by namespace. 

* infra-ops -  used by platform admins
* dev -  tenants will exist here to deploy dev code
* test -  tenants will exist here to deploy tes code


Create these cluster groups in TMC:

```bash
tanzu tmc clustergroup create -f tmc/clustergroups/infra-ops.yaml
tanzu tmc clustergroup create -f tmc/clustergroups/dev.yaml
tanzu tmc clustergroup create -f tmc/clustergroups/test.yaml
```

### Create initial clusters

This will not walk through creating the clusters needed since it's out of scope, but create the following cluster's on your favorite flavor of k8s and put them in the respective cluster groups.

clusters:

* infra-ops
* iris-dev
* iris-test




### Enable CD components on the cluster groups for TMC

This


```bash
tanzu tmc continuousdelivery enable -g infra-ops -s clustergroup
tanzu tmc continuousdelivery enable -g dev -s clustergroup
tanzu tmc continuousdelivery enable -g test -s clustergroup

tanzu tmc helm enable -g infra-ops -s clustergroup
tanzu tmc helm enable -g dev -s clustergroup
tanzu tmc helm enable -g test -s clustergroup

```