# Flux TMC Multi-tenant

This repo serves as a starting point for managing multi-tenant clusters using Flux and TMC.In this case what we mean by multi-tenant is that different teams can share a K8s cluster and have access to self service namespaces in that cluster but not be able to interfere with other teams namespaces in the same cluster. This does not get into network policy for locking down namespace networking, that is out of scope.  The overall goal of this repo is to give an base template an example of a real world flux multiteant use case with TMC. The structure of this repo accounts for multiple environments and clusters and is meant to be able to represent an enterprise environment.

## Associated repos

Here are the repos for the tenants that are mentioned throughout this doc.

[iris-green bootstrap repo](https://github.com/warroyo/iris-green-gitops)

[iris-green app repo](https://github.com/warroyo/iris-green-apps)

[iris-blue bootstrap repo](https://github.com/warroyo/iris-blue-gitops)

[iris-blue app repo](https://github.com/warroyo/iris-blue-apps)

[iris-red bootstrap repo](https://github.com/warroyo/iris-red-gitops)

[iris-red app repo](https://github.com/warroyo/iris-red-apps)



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

## How it works

TMC has the concept of a workspace which can group multiple namespaces across clusters under a single space. Using this concept we can create a tenancy model around namespaces in a cluster. This workspace concept is core to the multitenancy in this repo. The way this tenancy model works is this:

*  tenants get their own workspace and using TMC permissions we can ensure they are not able to access namespaces outside of their workspace as a logged in user
*  A flux bootstrap namespace and service account is created for each tenant, this namespace is where the initial kustomization for flux is deployed by the platform admin.
*  The bootstrap namespace service account is given permissions in TMC to the workspace. *** This is a key piece in that TMC handles propogating the permissions on the service account to access the tenants other namespaces. without TMC doing this another tool is needed to handle this, it's not native to k8s. ***
*  A TMC custom OPA policy is used to enforce that the tenants kustomizations and other flux reosurces use a service account. this prevents the flux controller from operating as cluster admin. This si cirtical in limiting the tenants access.
*  Enforcing namespace creation through flux allows for the platform admins to overwrite the critical fields to make sure namespaces are created in the right workspace.

### Clustergroup bootstrapping

One key element to this setup is how every cluster gets bootstrapped with the right flux config and paths. The approach used here is to only enable the TMC CD components at the clustergroup level. The typical challenge with this approach is that setting a `kustomization` at the clustergroup level means you cannot override the path per cluster and means everything needs to be the same in each cluster. This is not the case in most situations so in order for this to work we need every cluster to be able to override clustergroup configuration. This is done by using a combination of carvel tooling, tmc provided information and flux variables injection. Here is a breakdown of the process.

1. when a clustergroup kustomization is created it will point to the folder `clustergroups/<groupname>`
2. from there two other kustomizations are created `clustergroup-gitops` and `group-apps`
3. `group-apps` points to `apps/clustergroups/<groupname>` which defines any group level apps that need to be installed. most importantly it installs the `apps/base/cluster-name-secret` 
4. the `cluster-name-secret` app in this case is using [carvel secretgen](https://github.com/carvel-dev/secretgen-controller) to create a secret containing the cluster name from the tmc provided configmap `stack-config` and then using a `secretExport` and `secretImport` to copy that secret into the `tanzu-continous-delivery-resources` namespace.
5. `clustergroup-gitops` depends on `group-apps` and points to `clustergroups/common/per-cluster` where it creates a `kustomization` called `cluster-gitops` also it uses the flux `substituteFrom` capability to inject the cluster name from the previosuly created secret as a variable for use.
6. `cluster-gitops` creates a path dynamically using the cluster name to point to `clusters/<clustername>` which allows for cluster specific flux configuration.

Using this structure we can now set a single kustomization at the clustergroup level and have it dynamically create cluster specific `kustomizations` 

## Multi Tenant Architecture

This architecture is simplified and does not show all of the details but it highlights a few key components which are the tenancy with workspaces, the infra-ops cluster, and the use of different repos and flux.

![](images/2023-07-19-15-03-50.png)
## Platform admin repo structure

The platform admin repo has the following directories
### clustergroups 
 this contains directories with Flux config for each group. This is where the base bootstrap config lives. This is used when initially configuring a cluster group's `Kustomization` in TMC. Everything is initiated from here. 

```
clustergroups
│   ├── common
│   │   └── per-cluster
│   │       └── base.yaml
│   └── <clustergroup-name>
│       ├── base.yml
│       └── kustomization.yaml
```

subdirectories:
* `common` - used for any common config between all cluster groups.
* `common/per-cluster` -  this holds the `kustomization` that is dynamically created for each cluster that is mentioned in the clustergroup boostrapping section.
* `clustergroup-name` - folder per cluster group with configs for that clustergroup.

### clusters

This contains directories with Flux config for each cluster. This is what will hold each clusters base configuration that gets boostrapped from the cluster group.

```
clusters
│   └── <cluster-name>
│       ├── apps.yml
│       ├── infrastructure.yml
│       └── tenants
│           └── <tenant-name>.yml
```

subdirectories/files:

`cluster-name>` -  each cluster will have it's own directory that contains any cluster specific configuration. 
`apps.yml` -  sets up the cluster specific kustomization pointing to the clusters directory in the `apps` folder. 
`infrastructure.yml` - sets up the cluster specifc kustomization ponting to the clusters directory `infrastructure` directory. 
`tenants` -  contains a yml file for each tenant. this yaml file sets up the tenants bootstrap namespace in the cluster as well as the `kustomization` and `gitrepo` that point to the tenants bootstrap git repo. 

### infrastructure

Contains config to install common infra tools per environment. This idea here is that `infrastructure` is used for installing tools that are more infra focused (ex. external secrets operator) per environment and the `apps`  directory is used for more general apps/config per cluster.

```
infrastructure
│   ├── base
│   │   ├── <some-app>
│   │   │   ├── kustomization.yml
│   │   │   └── install.yml  
│   │   └── kustomization.yml
│   ├── env
│   │   ├── <environment>
│   │   │   ├── <some-app>
│   │   │   │   └── install.yml
│   │   │   └── kustomization.yml

```

subdirectories/files:

* `base` - contains folders for each infra app to be installed in the cluster.
* `env` - contains folders for every environment. 
* `env/<environment>` - holds the configuration for which apps to install in that environment.
* `env/<environment>/<some-app>` - holds any environment specific overrides for that app.

### apps

Contains all of the app configs that will be installed per cluster or clustergroup. Also allows for environment specific overrides if necessary.

```
apps
│   ├── base
│   │   ├── <some-app>
│   │   │   ├── install.yml
│   │   │   └── kustomization.yaml
│   ├── clustergroups
│   │   ├── <clustergroup-name>
│   │   │   └── kustomization.yaml
│   ├── clusters
│   │   └── <cluster-name>
│   │       └── kustomization.yaml
│   └── env
│       └── <environment>
│           └── <some-app>
│               ├── kustomization.yaml
│               └── override.yml
```

subdirectories/files:
* `base` - contains folders for each app to be installed in the clusters.
* `clustergroups` - contains folders for each clustergroup. This allows for adding config to install certain apps and overrides per cluster group.
* `clusters` - contains folders for each cluster. This allows for adding config to install certain apps and overrides per cluster.
* `env` - contains folders for every environment. 
* `env/<environment>/<some-app>` - holds any environment specific overrides for that app.

examples:

some good examples of different ways to override settings can be found in the following directories.

* `apps/clusters/iris-dev` - this shows how to use an env override with the `components` directive for external-secrets. It also shows how to do an override specific to the cluster using the inline patches to add an override values file for contour.
* 

### bootstrap

this directory holds the template for the azure-sp credential to setup secrets managment.

```
 bootstrap
 └── azure-secret.yaml
```
### tmc

Contains any yaml needed for creating TMC objects with the CLI. 


## Tenant repo structure

The tenant repo has the following directories

### clusters
this has a directory for each cluster. this will be the reference point for the tenants configured in the platform admin repo. This is used as a bootstrap for the tenant to configure their own kustomizations. 


```
clusters
    └── <cluster-name>
        ├── kustomization.yml
        ├── namespaces
        │   ├── kustomization.yml
        │   └── namespaces.yml
```

subdirectories/files:

* `<cluster-name>` - one for each cluster to deploy on
* `<cluster-name>/namespaces` - hold files(s) that define the `TMCNamespace` objects for namespace self service. also contains the `kustomization.yml` that adds the unique suffic to the `TMCNamespace` objects

## Initial Setup
These steps walk through a somewhat opinionated way of organzing things. This is not the only way of doing it and is just an example. Also for any infrastructure created it's advised that this be automated rather than being done by hand, in this example we will use the cli for everything we can. If you already have existing infra, it's not necessary to create new clusters etc. just use those. Also for the purpose of this setup we will assume that we have a platform team and 3 tenants. Those tenants are all within the same product group but are different app teams. our product group name is Iris, so we will have a setup where each team gets a workspace in TMC and k8s clusters will be grouped by environment into cluster groups, each product group in this case will have a cluster(s) per environment. Our three dev teams will be called iris-green, iris-red, iris-blue.

### Explaining the Infra-Ops cluster

In the next few sections it will refer to an "infra-ops" cluster. This cluster is not something that tenants will deploy onto. This cluster is used by platform admins to run thier workloads for mgmt purposes. Specifically in this case it is used to run the [tmc-controller](https://github.com/warroyo/metacontrollers/tree/main/tmc-controller) this is a metacontroller that is used for creating TMC namespaces. This allows for self service management of namespaces in TMC with gitops. 

 

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

### Create initial workspaces for each team

Each team gets their own workspace in which all of their namespaces will be created


```bash
tanzu tmc workspace create -f tmc/workspaces/iris-green.yaml
tanzu tmc workspace create -f tmc/workspaces/iris-blue.yaml
```


### Setup secrets management

Some type of secrets management will be needed when using gitops. There are a few different approaches, but in this exmaple we will be using Azure Key Vault. This will allow us to use one bootstrap secret that can then be used by [external secrets](https://github.com/external-secrets/external-secrets) that will handle all other secrets. 


This will not go thorugh the entire process of setting up AKV but it will include installing [external secrets with AKV](https://external-secrets.io/v0.4.1/provider-azure-key-vault/).

Additionally when setting this up we will use a bootstrap credential in the clusters, however if you have workload identity setup to work with azure this could be done using a role instead of a SP.


Other options:

[SOPS](https://fluxcd.io/flux/guides/mozilla-sops/)
[Sealed Secrets](https://fluxcd.io/flux/guides/sealed-secrets/)

#### Secret tenancy

Since this is a multitenant setup we need multitenant secrets.

**Option 1** 

This option is the we have implemented for this repo. 

There will be one `ClusterSecretStore` that is managed by the platform admins. We will create one AKV per environment. The secrets in AKV will be based on a naming convention, this will allow for policy based access to secrets. The naming will be something along the lines of `$tenant-$secretname` this would then mean that per workspace(tenant) a policy will enforce what they can access. You can read more about the inspiration for this in [Dodd Pfeffer's](https://github.com/doddatpivotal) detailed blog post [here](https://dodd-pfeffer.medium.com/deliver-secure-access-to-azure-key-vault-from-aks-powered-by-tanzu-dfdfc98138c5)


This approach still leaves the option open for developers to create their own namespace bound `secretStores` but it is up to them to botstrap the credentials for those. If desired, the cluster secret store could even be used as a way to inject the namespace based `secretstore` boostrap creds into the clusters. 

Pros:
* optional self service for tenants
* centralized vault so tenants don't need to worry about managing vaults
* works well in a managed platform environment
Cons:
* bootstrap secrets required
* some custom automation may be required to enable secret creation.


As a Platform admin:

1. Create centralized keyvault in azure per environment. for this setup create the following
   1. `ss-env`  - this is the shared service environment vault(mostly used for platform stuff)
   2. `dev-env` - dev env vault
   3. `test-env` - test env vault
2. Create a Azure SP that has read access to the keyvault 
3. bootstrap the cluster with the credentials created above. this needs to be done per cluster and should be automated with your cluster creation process. Modify the file `bootstrap/azure-secret.yaml` to have your credentials and apply it into the clusters.

```bash
kubectl apply -f  bootstrap/azure-secret.yaml
```

4. create a policy in TMC using gatekeeper to prevent tenants from accessing eachothers secrets and apply it to the two cluster groups. This policy will make sure that the naming convention matches the tenant workspace label on the namepsace. becuase of the access given to tenants they will not be able to modify this label so it is a good identifier to match on. The policy will match any namespace with the `tmc.cloud.vmware.com/workspace` label, this can be changed to meet your needs. For example you could have a list of workspaces to apply this to using the same label but with a list of values to match.

```bash
tanzu tmc policy policy-template create --object-file tmc/policy/enforce-es-naming-template.yaml --data-inventory "/v1/Namespace"
tanzu tmc policy create -f tmc/policy/enforce-es-naming-dev.yaml -s clustergroup
tanzu tmc policy create -f tmc/policy/enforce-es-naming-test.yaml -s clustergroup
```

5. Update the `ClusterSecretStore` details in the git repo. Under `apps/env/<env-name>/external-secrets-crd` there is an override yaml. update this with your tenant id and vault url for each environment. 

6. create secrets in the key vault for the tenants. This document does not cover the automation, but this could be done with a process that allows tenants to manage their own creential entries.

As a tenant:

1. create an external secret object that references the secret you want in your namespace. An example of this can be found [here](https://github.com/warroyo/iris-green-apps/blob/main/config/k8s/base/secret.yml), notice the naming convention being used `iris-green-example-secret`

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

We need to create an Gatekeeper constraint template that will allow us to enforce service account usage on the flux CRDs. This prevents tenants from being able to use admin level creds with flux.

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


The next role is the one that will allow the flux tenant service account in the infra-ops cluster the ability to create TMC namespaces. this enables namespace self service.

```bash
tanzu tmc iam role create -f tmc/iam/tmcnamespaces-iam.yaml
```


### Bind the roles to the service account with TMC IAM policy

This role binding is going to bind the tenants service account to the cluster admin equivalent role in the tenants workspace. This will mean that anytime a new namespace is created the service account flux is using will immediately have the correct permissions on the namespace. This is very useful, without this workspace level binding we would need to use some type of controller to handle this dynamically for us.


```bash
tanzu tmc iam update-policy -s workspace -n iris-blue -f tmc/iam/sa-rb-workspace-blue.yaml
tanzu tmc iam update-policy -s workspace -n iris-green -f tmc/iam/sa-rb-workspace-green.yaml
```


### Create the base Gitrepos for each cluster group

Each cluster group will need a gitrepo added as the base git repo to bootstrap the cluster. In this case that repo is this one. This could be any git repo though as long as the structure is setup properly. 

```bash
tanzu tmc continuousdelivery gitrepository create -f tmc/continousdelivery/test-gitrepo.yaml -s clustergroup
tanzu tmc continuousdelivery gitrepository create -f tmc/continousdelivery/dev-gitrepo.yaml -s clustergroup
tanzu tmc continuousdelivery gitrepository create -f tmc/continousdelivery/infra-ops-gitrepo.yaml -s clustergroup
```



### Setup the Infra-Ops Cluster

The infra ops cluster should be a one time setup since there is not one per tenant. This cluster will host our TMC Controller that enables the self service of namespaces.

#### Create the TMC credential

The infra-ops cluster needs to be able to communicate with the TMC API. Create a credential in the `ss-env` AKV for the TMC API token.

1. create a TMC API token
2. create the AKV entries. 

```bash
az keyvault secret set --vault-name "ss-env" --name "tmc-csp-token" --value "<tmc-api-token>"
az keyvault secret set --vault-name "ss-env" --name "tmc-host" --value "<tmc-hostname>"
```

#### Setup the bootstrap Kustomization

By enabling this bootstrap kustomization it will start the process of installing all of the required tools on the infra-ops clusters. 

1. rename the folder `clusters/eks.eks-warroyo2.us-west-2.infra-ops` to match your TMC cluster name. 
2. enable the kustomization

```bash
tanzu tmc continuousdelivery kustomization create -f tmc/continousdelivery/infra-ops.yaml -s clustergroup
```

Here is a breakdown on what gets installed and how. This does not cover the initial bootstrap process in detail which is covered in detail in the [above section](#clustergroup-bootstrapping)

1. The kustomization created above points to this path `clustergroups/infra-ops` in this repo. 
2. From that folder two more kustomizations are created `group-apps` and `clustergroup-gitops`
3. `group-apps` points at `/apps/clustergroups/infra-ops`. This installs the metacontroller and the tmc controller from the `apps/base` directory. these are kustomizations that point at other git repos. 
4. `clustergroup-gitops` bootstraps the cluster specific `kustomization` called `cluster-gitops` 
5. `cluster-gitops` points at `/clusters/eks.eks-warroyo2.us-west-2.infra-ops` which creates a few more kustomizations `infrastructure`, `apps`, `infra-pre-reqs` and the tenant specific kustomizations. Read more about the standard repo stucture [above](#platform-admin-repo-structure) for details on what each kustomization's purpose is.
6. `infrastructure` sets up external secrets operator and cert-manager.
7. `apps` - sets up the `clusterSecretStore` pointing at our ss-env AKV using the boostrap credential.
8. `infra-pre-reqs` - installs any pre-reqs for infra apps. this can be used to install any dependencies since `infrastructure` kustomization depends on it.
9. tenants specific kustomizations,  this sets up a tenant namespace and service account, as well as the kustomization that points at the tenants bootstrap repo and path to namespaces. It also creates an initial `TMCNamespace` object for the bootstrap ns in the downstream cluster(s). This is what configures namespace self service. the kustomization also overrides fields in the `TMCNamespace` objects to ensure they are not creating things outside of their workspace.




### Summary of initial setup

At this point all of the one time setup has been complete. This means that we can now onboard tenants and/or new clusters. 


Here is a summary of what was created:

* clusters for each env
* infra-ops cluster with TMC controller
* workspaces for each team
* cluster groups for each environment
* policies for enforcing tenancy
* IAM policy for enforcing tenancy
* enabling base flux components at the cluster group level
* secrets management setup with AKV


## Add a tenant to a cluster

For this step we will walk through adding a tenant to a clustergroup. This will use pre-existing configuration from this repo. In another section we will cover creating a tenant from scratch.

### Setup the bootstrap Kustomizations

Each cluster group will have an initial kustomization that points to a specific path in the git repo to bootstrap. This will create other Kustomizations that will be specific to each cluster.

1. rename the folder `clusters/eks.eks-warroyo2.us-west-2.iris-dev` and `clusters/eks.eks-warroyo2.us-west-2.iris-dev`  to match your TMC cluster names. 
2. this creates the `kustomizations` for the dev and test cluster groups.  

```bash
tanzu tmc continuousdelivery kustomization create -f tmc/continousdelivery/dev.yaml -s clustergroup
tanzu tmc continuousdelivery kustomization create -f tmc/continousdelivery/test.yaml -s clustergroup
```


Here is a breakdown of what is happening after creating these. This breakdown is for the dev clustergroup and iris-dev cluster but this same process happens for any cluster added.

1. The kustomization created above points to this path `clustergroups/dev` in this repo. 
2. From that folder two more kustomizations are created `group-apps` and `clustergroup-gitops`
3. `group-apps` points at `/apps/clustergroups/dev`. currently the only thing installed here is the secret imports and exports to make the next step work.
4. `clustergroup-gitops` bootstraps the cluster specific `kustomization` called `cluster-gitops` 
5. `cluster-gitops` points at `/clusters/eks.eks-warroyo2.us-west-2.iris-dev` which creates a few more kustomizations `infrastructure`, `apps`, `infra-pre-reqs` and the tenant specific kustomizations. Read more about the standard repo stucture [above](#platform-admin-repo-structure) for details on what each kustomization's purpose is.
6. `infrastructure` sets up external secrets operator and cert-manager.
7. `apps` - sets up the `clusterSecretStore` pointing at our dev-env AKV using the boostrap credential. Also installs the contour package.
8. `infra-pre-reqs` - installs any pre-reqs for infra apps. this can be used to install any dependencies since `infrastructure` kustomization depends on it.
9. tenants specific kustomizations, this sets up a service account in the tenant bootstrap ns that was created through the ns automation in the infra-ops cluster. As well as the kustomization that points at the tenants bootstrap repo and path to the cluster name. Permissions for the service account are inherited by the access policy on the workspace. This is what allows the tenant to now create their own `kustomizations` and `gitrepos` to install their apps. 


After the reconcile completes you should see a number of kustomizations in the cluster and everything should be deployed from the tenant bootstrap repo. It will look like this:


```
NAMESPACE↑                                   NAME                            READY          STATUS                                                                    AGE              │
│ iris-blue                                    iris-blue                       True           Applied revision: main/7f1800ddfe91e8546e8ee895cb463cea76ca799f           28h              │
│ iris-blue                                    iris-blue-apps                  True           Applied revision: main/0a0022ebc2735d8defaa373d511499ac64949474           12m              │
│ iris-green                                   iris-green                      True           Applied revision: main/2402dfc51ea88adab1535306741925bc6ecaeb4f           28h              │
│ iris-green                                   iris-green-apps                 True           Applied revision: main/b1d1541217c8565f37e8659391c3f152227692ed           25m              │
│ iris-red                                     iris-red                        True           Applied revision: main/a79bd9de6fbd1e3fd6f41b864873e00aa63f03ee           25h              │
│ iris-red                                     iris-red-apps                   True           Applied revision: main/3215ab378aed49d155427c1efe1c70b62bdd55ea           16s              │
│ tanzu-continuousdelivery-resources           apps                            True           Applied revision: main/177933088ded390537795fc8b1bb94a230af1239           46h              │
│ tanzu-continuousdelivery-resources           cg-base                         True           Applied revision: main/177933088ded390537795fc8b1bb94a230af1239           2d22h            │
│ tanzu-continuousdelivery-resources           cluster-gitops                  True           Applied revision: main/177933088ded390537795fc8b1bb94a230af1239           46h              │
│ tanzu-continuousdelivery-resources           clustergroup-gitops             True           Applied revision: main/177933088ded390537795fc8b1bb94a230af1239           46h              │
│ tanzu-continuousdelivery-resources           external-secrets-crds           True           Applied revision: main/59bf53e7a3cdd38dd0d742f0950b731d89cc50e6           46h              │
│ tanzu-continuousdelivery-resources           group-apps                      True           Applied revision: main/177933088ded390537795fc8b1bb94a230af1239           46h              │
│ tanzu-continuousdelivery-resources           infra-pre-reqs                  True           Applied revision: main/177933088ded390537795fc8b1bb94a230af1239           46h              │
│ tanzu-continuousdelivery-resources           infrastructure                  True           Applied revision: main/177933088ded390537795fc8b1bb94a230af1239           46h 
```


## Adding a new cluster

This will outline adding a new cluster to an existing environment, the process would be similar for adding an environment but you would just need to add more folders for the environment specific pieces.

### Platform admin Tasks

1. create the cluster and add it to the appropriate cluster group
2. if using secret management be sure to create the bootstrap credential in the newly created cluster
4. create a new folder in the `clusters` directory with the name of the cluster from TMC. 
5. add the neccessary files, examples of what are in the files can be foudn in this directory and are explained in the repo stucture.
   1. `tenants/<tenant-name>.yml`
   2. `apps.yml`
   3. `infrastructure.yml`
6. create a new folder in the `apps/clusters` directory with the name of the cluster. this must match the path given in the `apps.yml`
7.  create a `kustomization.yml` in that directory with the references to the apps you want installed in that cluster.


### Tenant tasks

These steps would only be done if the tenant was added to the new cluster. The steps are the same as the steps below for adding a new Tenant.



## Adding a new tenant

Adding a new Tenant has a few steps that could be automated. Some ideas around automating these are listed below. These steps should be completed any time a new team is wanting to be onboarded. These steps are outlined with commands referencing this repo's setup but these could be adpated to be done generically.

using `iris-red` as the new tenant.

### Platform admin tasks

1. create a new workspace in TMC for the team. 

```bash
tanzu tmc workspace create -f tmc/workspaces/iris-red.yaml
```

2. bind the service account to the IAM role using a tmc access policy. this gives the service account in the bootstrap namespace permissions to any namespace in the workspace. 

```bash
tanzu tmc iam update-policy -s workspace -n iris-red -f tmc/iam/sa-rb-workspace-red.yaml
```

3. create a tenant file in the infra-ops cluster folder. Replace the tenant name in all locations in the file.

```
cp clusters/eks.eks-warroyo2.us-west-2.infra-ops/tenants/iris-blue.yml clusters/eks.eks-warroyo2.us-west-2.infra-ops/tenants/iris-red.yml 

##replace the tenant name in the file
```

4. create a tenant gitops repo, this could also be handled by the tenant initially and passed to the admins. In this case our git repo name is `iris-red-gitops`


5. create a tenant file in the clusters that you would like that tenant to exist in. for this example we will assume dev only. This file contains the `gitrepo` setup for the tenant so make sure it matches the git repo from step 4.

```
cp clusters/eks.eks-warroyo2.us-west-2.iris-dev/tenants/iris-blue.yml clusters/eks.eks-warroyo2.us-west-2.iris-dev/tenants/iris-red.yml 

##replace the tenant name in the file
```

6. commit these files into the git repo and wait for flux to reconcile. here is what is created
   1. tenant workspace
   2. tenant git repo
   3. tenant ns in the cluster(s)
   4. IAM rules for tenant
   5. ability to do NS self service
   6. intial flux config so they can start installing apps.

### Tenant tasks

The tenant tasks will vary based on what the tenant wants to do but here is the minimum required to get things working.

1. Create folder for clusters in the tenant git repo

```bash
mkdir -p clusters/iris-dev
```

2. Create a folder for namespaces per cluster

```bash
mkdir -p clusters/iris-dev/namespaces
```

3. create the `kustomization` files in each directory to let flux know which files to reconcile. This will depend on how the tenant wants to structure thier repo. an example can be found [here](https://github.com/warroyo/iris-red-gitops). 



### Automation ideas

In the steps above we just copied existing files and did search and replace. However in an automation scenario what you would most liklely do is template out the files and use variables to generate them. 

* TAP acelerators for tenant repos
* ADO pipelines + YTT templated files
* Simple bash scripts to create the files. 
* CLI scaffolding generator tooling

## Self Service a namespace

The setup in the repo allows for tenants to self service namespaces. This is done using TMC workspaces, without a TMC workspaces this problem is someone hard to solve and requires other 3rd party tools. Additionally we want self service through gitops, to solve this problem the [TMC controller](https://github.com/warroyo/metacontrollers/tree/main/tmc-controller) was created. So what is enabled is the ability for a tenant to put TMC namespace configs into a directory in thier tenant git repo and it will create them and give them permissions to those namespaces automatically.

1. In the tenant git repo make sure you have the `clusters/<clustername/namespaces` folder created. this is what flux watches.
2. create a file(s) in the directory that has the namespace config. An example can be found [here](https://github.com/warroyo/iris-green-gitops/blob/main/clusters/iris-dev/namespaces/namespaces.yml)
3. make sure that the `kustomization` includes that file, example [here](https://github.com/warroyo/iris-green-gitops/blob/main/clusters/iris-dev/namespaces/kustomization.yml). Make sure that the `kustomization` also includes a unique `nameSuffix` this will make sure that the `TMCNamespace` objects do not have collisions in the infra-ops cluster.