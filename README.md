# Flux TMC Multi-tenant

This repo serves as a starting point for managing multi-tenant clusters using Flux and TMC.In this case what we mean by multi-tenant is that different teams can share a K8s cluster and have access to self service namespaces in that cluster but not be able to touch other teams namespaces in the same cluster. This does not get into network policy for locking down namepsace networking, that is out of scope.  



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

# How it works

TMC has the concept of a workspace which can group multiple namespaces across clusters under a single space. Using this concept we can create a tenancy model around namespaces in a cluster. This workspace concept is core to the multitenancy in this repo. The way this tenancy model works is this:

*  tenants get their own workspace and using TMC permissions we can ensure they are not able to access namespaces outside of their workspace as a logged in user
*  A flux bootstrap namespace and service account is created for each tenant, this namespace is where the initial kustomization for flux is deployed by the platform admin.
*  The bootstrap namespace service account is given permissions in TMC to the workspace. *** This is a key piece in that TMC handles propogating the permissions on the service account to access the tenants other namespaces. without TMC doing this another tool is needed to handle this, it's not native to k8s. ***
*  A TMC custom OPA policy is used to enforce that the tenants kustomizations and other flux reosurces use a service account. this prevents the flux controller from operating as cluster admin. This si cirtical in limiting the tenants access.
*  Enforcing namespace creation through flux allows for the platform admins to overwrite the critical fields to make sure namespaces are created in the right workspace.
  




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

### Setup secrets management

Some type of secrets management will be needed when using gitops. There are a few different approaches, but in this exmaple we will be using Azure Key Vault. This will allow us to use one bootstrap secret that can then be used by [external secrets](https://github.com/external-secrets/external-secrets) that will handle all other secrets. 


This will not go thorugh the entire process of setting up AKV but it will include installing [external secrets with AKV](https://external-secrets.io/v0.4.1/provider-azure-key-vault/).

Additionally when setting this up we will use a bootstrap credential in the clusters, however if you have pod identity setup to work with azure this could be done using a role instead of a SP.


Other options:

[SOPS](https://fluxcd.io/flux/guides/mozilla-sops/)
[Sealed Secrets](https://fluxcd.io/flux/guides/sealed-secrets/)

#### Secret tenancy

Since this is a multitenant setup we need multitenant secrets.

**Option 1** 

This option is the we have implemented for this repo. 

There will be one `ClusterSecretStore` this is managed by the platform admins. we will create one AKV per environment. The secrets in AKV will be based on a naming convention, this will allow for policy based access to secrets. The naming will be something along the lines of `$tenant-$secretname` this would then mean that per workspace(tenant) a policy will enforce what they can access. You can read more about the inspiration for this in [Dodd Pfeffer's](https://github.com/doddatpivotal) detailed blog post [here](https://dodd-pfeffer.medium.com/deliver-secure-access-to-azure-key-vault-from-aks-powered-by-tanzu-dfdfc98138c5)


This approach still leaves the option open for developers to create their own namespace bound `secretStores` but it is up to them to botstrap the credentials for those. If desired, the cluster secret store could even be used as a way to key the other boostrap creds into the clusters. 

Pros:
* optional partial self service for tenants
* centralized vault so tenants don't need to worry about managing vaults
* works well in a managed platform environment
Cons:
* bootstrap secrets required
* some custom automation may be required to enable secret creation.


As a Platform admin:

1. Create centralized keyvault in azure
2. Create a Azure SP that has read access to the keyvault 
3. bootstrap the cluster with the credentials created above. this needs to be done per cluster and should be automated with your cluster creation process. Modify the file `bootstrap/azure-secret.yaml` to have your credentials and apply it into the clusters.

```bash
kubectl apply -f  bootstrap/azure-secret.yaml
```

4. create a policy in TMC using gatekeeper to prevent tenants from accessing eachothers secrets and apply it to the two cluster groups. This policy will make sure that the naming convention matches the tenant workspace label on the namepsace. becuase of the access given to tenants they will not be able to modify this label so it is a good identifier to match on. The policy will match any namespace with the `tmc.cloud.vmware.com/workspace` label, this can be changed to meet your needs. for example you could have a list of workspaces to apply this to using the same label but with a list of values to match.

```bash
tanzu tmc policy policy-template create --object-file tmc/policy/enforce-es-naming-template.yaml --data-inventory "/v1/Namespace"
tanzu tmc policy create -f tmc/policy/enforce-es-naming-dev.yaml -s clustergroup
tanzu tmc policy create -f tmc/policy/enforce-es-naming-test.yaml -s clustergroup
```

5. create secrets in the key vault for the tenants. This document does not cover the automation, but this could be done with a process that allows tenants to manage their own creential entries.

As a tenant:

1. create an external secret object that references the secret you want in your namespace.

**Option 2** 

This option works well for teams that manage their own AKV instances and also if your clusters are federated with Azure(AKS or using [workload-identity](https://azure.github.io/azure-workload-identity/docs/introduction.html)) and can use workload identities. In this setup tenants will manage their own `secretStores` in their namespaces. This does not require any setup from the platform admin team outside of installing the operator which this repo already handles.

Pros:
* self service for tenants
* no secrets invovled to bootstrap
* works well in an environment with more advanced tenants
Cons:
* no centralized control of vault so more effort for tenants(less platform like)
* advanced setup of identities outside of AKS

This repo does not cover the scope of setting up workload identity for Azure, but the [quickstart](https://azure.github.io/azure-workload-identity/docs/quick-start.html) here does  a nice job of explaining. Once this is complete the below process can be used by tenants or admins to access their vaults.

As a tenant:

1. create a k8s service account in my namespace and assign it a workload identity that has permissions to the vault. docs [here](https://azure.github.io/azure-workload-identity/docs/topics/azwi/serviceaccount-create.html).
2. create a `secretStore` in my namespace and set the service account ref field to use the previosuly created account.  docs [here](https://external-secrets.io/v0.8.5/provider/azure-key-vault/#referenced-service-account).
3. start using `externalSecrets` to consume secrets.



#### Install the operator

Since we are using gitops, this will be installed automatically though this repo. there is nothing else to do here. In the following steps the `kustomizations` will be created that deploy this.


### Create initial clusters

This will not walk through creating the clusters needed since it's out of scope, but create the following cluster's on your favorite flavor of k8s and put them in the respective cluster groups.

clusters:

* infra-ops
* iris-dev
* iris-test

### Create initial workspaces for each team

Each team gets their own workspace in which all of their namespaces will be created


```bash
tanzu tmc workspace create -f tmc/workspaces/iris-green.yaml
tanzu tmc workspace create -f tmc/workspaces/iris-blue.yaml
tanzu tmc workspace create -f tmc/workspaces/iris-red.yaml
```

<!-- ### Create the workspace for the tenant namepsaces in the infra-ops cluster

We will group all tenant namespaces in the infra-ops cluster into the same workspace to make it easier to manage permissions on them. Tenants will not have direct access to these namespaces and can only interact through gitops with very limited permissions.

```bash
tanzu tmc workspace create -f tmc/workspaces/tenants-infra-ops.yaml
``` -->

### Enable CD components on the cluster groups for TMC

This will enable the flux kustomize and helm controllers in all of the clusters that are added to the cluster groups.


```bash
tanzu tmc continuousdelivery enable -g infra-ops -s clustergroup
tanzu tmc continuousdelivery enable -g dev -s clustergroup
tanzu tmc continuousdelivery enable -g test -s clustergroup

tanzu tmc helm enable -g infra-ops -s clustergroup
tanzu tmc helm enable -g dev -s clustergroup
tanzu tmc helm enable -g test -s clustergroup

```

### Create the custom TMC policy to enforce service account usage

We need to create an Gatekeeper constraint template that will allow us to enforce service account usage on the flux CRDs. Thsi prevents tenants from being able to use admin level creds with flux.

```bash
tanzu tmc policy policy-template create --object-file tmc/policy/enforce-sa-template.yaml
```

Apply that template to the two cluster groups that tenants clusters will be in. We will also exclude the namespaces that we don't want it enforce on. 

```bash
tanzu tmc policy create -f tmc/policy/enforce-sa-dev.yaml -s clustergroup
tanzu tmc policy create -f tmc/policy/enforce-sa-test.yaml -s clustergroup
```


### Create the TMC IAM roles needed

The first role needed is the role that will be equivalent to cluster admin and bound to the bootsrap namespace's service account called `tenant-flux-reconciler`. this is required becuase currently the built in TMC roles at the workspace level do not have enough permisisons for CRDs. 


```bash
tanzu tmc iam role create -f tmc/iam/cluster-admin-equiv-iam.yaml
```


The next role is the one that will allow the flux tenant service account in the infra-ops cluster the ability to create TMC namespaces. this enable namespace self service.

```bash
tanzu tmc iam role create -f tmc/iam/tmcnamespaces-iam.yaml
```


### Bind the roles to the service account with TMC IAM policy

This first role binding is going to bind the tenants service account to the cluster admin equivalent role in the tenants workspace. This will mean that anytime a new namespace is created the service account flux is using will immediately have the correct permissions on the namespace. This is very useful, without this workspace level binding we would need to use some type of controller to handle this dynamically for us.


```bash
tanzu tmc iam update-policy -s workspace -n iris-blue -f tmc/iam/sa-rb-workspace-blue.yaml
tanzu tmc iam update-policy -s workspace -n iris-green -f tmc/iam/sa-rb-workspace-green.yaml
tanzu tmc iam update-policy -s workspace -n iris-red -f tmc/iam/sa-rb-workspace-red.yaml
```
<!-- 
The next role binding to create is for the allowing the tenant namespace service accounts in the infra-ops cluster the ability to create TMCNamespace objects. This enables self service.  -->


### Create the base Gitrepos for each cluster group

Each cluster group will need a gitrepo added as the base git repo to bootstrap the cluster. In this case that repo is this one. This could be any git repo though as long as the structure is setup properly. 

```bash
tanzu tmc continuousdelivery gitrepository create -f tmc/continousdelivery/test-gitrepo.yaml -s clustergroup
tanzu tmc continuousdelivery gitrepository create -f tmc/continousdelivery/dev-gitrepo.yaml -s clustergroup
tanzu tmc continuousdelivery gitrepository create -f tmc/continousdelivery/infra-ops-gitrepo.yaml -s clustergroup
```

### Create the base Kustomizations for each cluster group

Each cluster group will have an initial kustomization that points to a specific path in the git repo to bootstrap. This will create other Kustomizations that will be specific to each cluster.


```bash
tanzu tmc continuousdelivery kustomization create -f tmc/continousdelivery/dev.yaml -s clustergroup
tanzu tmc continuousdelivery kustomization create -f tmc/continousdelivery/test.yaml -s clustergroup
tanzu tmc continuousdelivery kustomization create -f tmc/continousdelivery/infra-ops.yaml -s clustergroup
```
This will start reconciling all of the components that we have configured to be installed using this git repo structure. Inspect each cluster and make sure Kustomizations are being created and are healthy

```bash
kubectl get kustomization -A
```




## Adding a new cluster
TODO

## adding a new tenant
TODO