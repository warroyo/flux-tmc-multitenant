# 2.0.0

This release contains updates to the tenant management as well as a few minor fixes.

* Fixed issue with external secrets CRDs erroring out during app kustomization reconcile. This issue was due to using the `v1aplpa1` version of the api on the resources instead of the `v1beta1`. related commits:
  * https://github.com/warroyo/flux-tmc-multitenant/commit/d2f3aa394e1d581341b5c20748ef7034fd1e7a4c
  * https://github.com/warroyo/flux-tmc-multitenant/commit/7448a7b088dbf496ea0c99c01b8895f4d0c2a452
  * https://github.com/warroyo/flux-tmc-multitenant/commit/43b858e576419dccc9eea25c015ebf299eeb97d5
* passed `cluster_name` variable through to downstream kustoization. this reduces the need to manually enter cluster name in other locations. related commits:
  * https://github.com/warroyo/flux-tmc-multitenant/commit/8aaefa1451f78d584c515a6d03c2fe644e0cf5a9
* added helm chart to manage tenants on clusters. This reduces the code duplication needed to add tenants on clusters.
  * handles both infra and workload cluster tenants
  * see docs [here](https://github.com/warroyo/flux-tmc-multitenant/tree/main/tenant-generator) 
* refactored tenants to use new helm chart
* updated documentation for tenants to reflect new helm chart

# 1.0.0

initial release