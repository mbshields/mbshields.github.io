# Sveltos - Configuration and Operation

<img src="./images/logo.png" width="200">

- [Sveltos - Configuration and Operation](#sveltos---configuration-and-operation)
  - [Cluster Selection](#cluster-selection)
  - [Sync Modes](#sync-modes)
  - [Dry Run Mode](#dry-run-mode)
  - [Snapshots and Rollback](#snapshots-and-rollback)
    - [Rollback](#rollback)
  - [Managed Features](#managed-features)
    - [ConfigMaps and Secrets](#configmaps-and-secrets)
    - [Helm charts](#helm-charts)
    - [List helm charts and resources deployed in a CAPI Cluster.](#list-helm-charts-and-resources-deployed-in-a-capi-cluster)
  - [Detecting conflicts](#detecting-conflicts)
  - [Getting started on any Kubernetes cluster](#getting-started-on-any-kubernetes-cluster)
    - [Deploy YAML](#deploy-yaml)
    - [Install CRD and Deployment](#install-crd-and-deployment)
    - [Uninstall CRDs](#uninstall-crds)
    - [Undeploy controller](#undeploy-controller)
  - [Contributing](#contributing)
  - [License](#license)

The main features of Sveltos include:

* [Flexible cluster selection](#markdown-header-cluster-selection)
* [Sync Modes](#markdown-header-sync-modes): One Time or Continuous 
* [Dry Run](#markdown-header-dry-run)
* [Snapshots and Rollback](#markdown-header-snapshotting)
* [Detecting Conflicts](#markdown-detecting-conflicts)
* [Declarative API and CLI]

## Cluster Selection


The `clusterSelector` field is a Kubernetes [label selector](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#resources-that-support-set-based-requirements) that matches against labels on CAPI clusters.

>Example:  `clusterSelector: env=prod`

## Sync Modes 

The `syncMode` field has three possible options. `Continuous` and `OneTime` are explained below. `DryRun` is explained in a separate section.

>Example:  `syncMode: Continuous`

* OneTime

  Upon deployment of a `ClusterProfile` with a `syncMode` configuration of `OneTime`, all CAPI clusters are checked for a `clusterSelector` match, and all matching clusters will have the current `ClusterProfile`-specified features installed at that time.

  Any subsequent changes to the `ClusterProfile` features will not be deployed into the already matching CAPI clusters.

* Continuous

  When the `syncMode` configuration is `Continuous`, any new changes made to the `ClusterProfile`-specified features are immediately reconciled into the matching CAPI clusters.

  Reconciliation consists of one of these actions on matching clusters: 
  - deploy a feature -- whenever a feature is added to the ClusterProfile or when a CAPI cluster newly matches the ClusterProfile
  - update a feature -- whenever the ClusterProfile configuration changes or any referenced ConfigMap/Secret changes
  - remove a feature -- whenever a Helm release or a ConfigMap/Secret is deleted from the ClusterProfile

* DryRun

  See [Dry Run Mode](#markdown-header-dry-run-mode).

## Dry Run Mode

Before adding, modifying, or deleting a ClusterProfile, it is often useful to see what changes will result. When you deploy a `ClusterProfile` with a `syncMode` configuration of `DryRun`, a workflow is launched that will simulate all of the operations that would be executed in an actual run. No actual code is executed in the dry run workflow, and there are no side effects. The dry run workflow generates a list of potential changes for each matching CAPI cluster, allowing you to inspect and validate these changes before deploying the new `ClusterProfile` configuration.

You can see the change list by viewing a generated Custom Resource Definition (CRD) named _ClusterReport_, but it is much easier to view the list using a  [sveltosctl](https://github.com/projectsveltos/sveltosctl) CLI command, as shown in the following example:

```
./bin/sveltosctl show dryrun
+-------------------------------------+--------------------------+-----------+----------------+-----------+--------------------------------+------------------+
|               CLUSTER               |      RESOURCE TYPE       | NAMESPACE |      NAME      |  ACTION   |            MESSAGE             | CLUSTER FEATURES |
+-------------------------------------+--------------------------+-----------+----------------+-----------+--------------------------------+------------------+
| default/sveltos-management-workload | helm release             | kyverno   | kyverno-latest | Install   |                                | dryrun           |
| default/sveltos-management-workload | helm release             | nginx     | nginx-latest   | Install   |                                | dryrun           |
| default/sveltos-management-workload | :Pod                     | default   | nginx          | No Action | Object already deployed.       | dryrun           |
|                                     |                          |           |                |           | And policy referenced by       |                  |
|                                     |                          |           |                |           | ClusterProfile has not changed |                  |
|                                     |                          |           |                |           | since last deployment.         |                  |
| default/sveltos-management-workload | kyverno.io:ClusterPolicy |           | no-gateway     | Create    |                                | dryrun           |
+-------------------------------------+--------------------------+-----------+----------------+-----------+--------------------------------+------------------+
```

For a demonstration of dry run mode, watch the video [Sveltos, introduction to DryRun mode](https://www.youtube.com/watch?v=gfWN_QJAL6k&t=4s) on YouTube.

## Snapshots and Rollback

The snapshot feature allows you to capture a complete policy configuration at an instant in time. Using snapshots from different times, you can see what configuration changes occurred between two timestamps, and you can roll back and forward policy configurations to any saved configuration snapshot.

Operations using snapshots, such as capture, diff, and rollback, are performed with the Sveltos command line interface, `sveltosctl`.

For a demonstration of snapshots, watch the video [Sveltos, introduction to Snapshots](https://www.youtube.com/watch?v=ALcp1_Nj9r4) on YouTube.

### Rollback

Rollback is when a previous configuration snapshot is used to replace the current configuration deployed by ClusterProfiles. Rollback can be executed with the following granularities:
1. namespace: Rolls back only ConfigMaps/Secrets and Cluster labels in this namespace. If no namespace is specified, all namespaces are updated.
2. cluster: Rolls back only labels for a cluster with this name. If no cluster name is specified, labels for all clusters are updated.
3. clusterprofile: Rolls back only ClusterProfiles with this name. If no ClusterProfile name is specified, all ClusterProfiles are updated.

When all of the configuration files for a particular version are used to replace the current configuration, this is referred to as a full rollback.

For a demonstration of rollback, watch the video [Sveltos, introduction to Rollback mode](https://www.youtube.com/watch?v=sTo6RcWP1BQ) on YouTube.

## Managed Features

The purpose of Sveltos is to deploy features selectively to specified CAPI clusters. The features to be deployed can be Helm charts, Kubernetes Secrets, or Kubernetes ConfigMaps. In the example below, one of each type of feature is deployed. 

In this example, we want to deploy specific features into a subset of CAPI clusters: those that are labeled with _env: prod_. We first create a `ClusterProfile` instance, such as the YAML file shown below. In the instance, we specify an appropriate `clusterSelector` to match with the intended CAPI clusters. We then specify the desired features to be deployed into the selected CAPI clusters.

Using the example `ClusterProfile`, Sveltos will direct Kubernetes to ensure that any CAPI cluster with the label _env: prod_ will have the following features deployed:
*  a Helm chart (version v2.5.0) for installing Kyverno
*  a Kubernetes resource(s) contained in the referenced Secret: _default/storage-class_
*  a Kubernetes resource(s) contained in the referenced ConfigMap: _default/contour_

```
apiVersion: config.projectsveltos.io/v1alpha1
kind: ClusterProfile
metadata:
  name: demo
spec:
  clusterSelector: env=prod
  syncMode: Continuous
  helmCharts:
  - repositoryURL: https://kyverno.github.io/kyverno/
    repositoryName: kyverno
    chartName: kyverno/kyverno
    chartVersion: v2.5.0
    releaseName: kyverno-latest
    releaseNamespace: kyverno
    helmChartAction: Install
 policyRefs:
  - name: storage-class
    namespace: default
    kind: Secret
  - name: contour-gateway
    namespace: default
    kind: ConfigMap
```
### ConfigMaps and Secrets

ConfigMaps and Secrets contain Kubernetes resources that can be deployed in CAPI clusters. 

- A Kubernetes [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/) stores non-confidential data in key-value pairs.
- A Kubernetes [Secret](https://kubernetes.io/docs/concepts/configuration/secret/) is similar to a ConfigMap, but is used to store sensitive data.

ConfigMaps and Secrets are listed and identified by name, namespace, and kind in the `policyRefs` section of the `ClusterProfile`. 

The data field of a ConfigMap or Secret can be a list of key-value pairs. Any key is acceptable, and the value can be multiple objects, described in YAML or JSON format. 

The following YAML file is an example of the _contour-gateway_ ConfigMap, which  is referenced by our example `ClusterProfile`. When Sveltos deploys this ConfigMap as part of our `ClusterProfile`, a GatewayClass and Gateway instance are automatically deployed in any matching CAPI cluster.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: contour-gateway
  namespace: default
data:
  gatewayclass.yaml: |
    kind: GatewayClass
    apiVersion: gateway.networking.k8s.io/v1beta1
    metadata:
      name: contour
    spec:
      controllerName: projectcontour.io/projectcontour/contour
  gateway.yaml: |
    kind: Namespace
    apiVersion: v1
    metadata:
      name: projectcontour
    ---
    kind: Gateway
    apiVersion: gateway.networking.k8s.io/v1beta1
    metadata:
     name: contour
     namespace: projectcontour
    spec:
      gatewayClassName: contour
      listeners:
        - name: http
          protocol: HTTP
          port: 80
          allowedRoutes:
            namespaces:
              from: All
```

### Helm charts

[Helm](https://helm.sh) is a CNCF graduated project that serves as a package manager widely used in the Kubernetes community. Helm uses "charts" to orchestrate the deployment of a set of Kubernetes resources. 

Helm charts to be deployed by Sveltos are listed in the `helmCharts` section of the `ClusterProfile`. Sveltos uses the [Helm golang SDK](helm.sh/helm/v3/pkg) to deploy all Helm charts listed in a `ClusterProfile` instance.

### List helm charts and resources deployed in a CAPI Cluster.

There is many-to-many mapping between Clusters and ClusterProfile: 
- Multiple ClusterProfiles can match with a CAPI cluster; 
- Multiple CAPI clusters can match with a single ClusterProfile.

As for DryRun, [sveltosctl](https://github.com/projectsveltos/sveltosctl) can be used to properly list all deployed features per CAPI cluster:

```
./bin/sveltosctl show features
+-------------------------------------+--------------------------+-----------+----------------+---------+-------------------------------+------------------+
|               CLUSTER               |      RESOURCE TYPE       | NAMESPACE |      NAME      | VERSION |             TIME              | CLUSTER PROFILES |
+-------------------------------------+--------------------------+-----------+----------------+---------+-------------------------------+------------------+
| default/sveltos-management-workload | helm chart               | kyverno   | kyverno-latest | v2.5.0  | 2022-10-11 20:59:18 -0700 PDT | mgianluc         |
| default/sveltos-management-workload | helm chart               | nginx     | nginx-latest   | 0.14.0  | 2022-10-11 20:59:25 -0700 PDT | mgianluc         |
| default/sveltos-management-workload | helm chart               | mysql     | mysql          | 9.3.3   | 2022-10-11 20:43:41 -0700 PDT | mgianluc         |
| default/sveltos-management-workload | :Pod                     | default   | nginx          | N/A     | 2022-10-12 09:33:25 -0700 PDT | mgianluc         |
| default/sveltos-management-workload | kyverno.io:ClusterPolicy |           | no-gateway     | N/A     | 2022-10-12 09:33:25 -0700 PDT | mgianluc         |
+-------------------------------------+--------------------------+-----------+----------------+---------+-------------------------------+------------------+
```

Otherwise, a new CRD is introduced to easily summarize which features (either helm charts or kubernetes resources) are deployed in a given CAPI cluster because of one or more ClusterProfiles.
Such CRD is called *ClusterConfiguration*.

There is exactly only one ClusterConfiguration for each CAPI Cluster.

Following example shows us that because of ClusterProfile *demo* three helm charts and one Kyverno ClusterPolicy were deployed.


```
apiVersion: v1
items:
- apiVersion: config.projectsveltos.io/v1alpha1
  kind: ClusterConfiguration
  metadata:
    creationTimestamp: "2022-09-19T21:14:15Z"
    generation: 1
    name: sveltos-management-workload
    namespace: default
    ownerReferences:
    - apiVersion: config.projectsveltos.io/v1alpha1
      kind: ClusterProfile
      name: demo2
      uid: f0f93440-95ae-4663-aff7-3b27c1135dfc
    resourceVersion: "45488"
    uid: efa326cb-a7e0-401c-918f-0a9252033856
  status:
    clusterProfileResources:
    - Features:
      - charts:
        - appVersion: v1.7.0
          chartName: kyverno-latest
          chartVersion: v2.5.0
          lastAppliedTime: "2022-09-19T21:14:16Z"
          namespace: kyverno
          repoURL: https://kyverno.github.io/kyverno/
        - appVersion: 2.3.0
          chartName: nginx-latest
          chartVersion: 0.14.0
          lastAppliedTime: "2022-09-19T21:14:23Z"
          namespace: nginx
          repoURL: https://helm.nginx.com/stable
        - appVersion: 1.22.1
          chartName: contour
          chartVersion: 9.1.2
          lastAppliedTime: "2022-09-19T21:17:31Z"
          namespace: projectcontour
          repoURL: https://charts.bitnami.com/bitnami
        featureID: Helm
      - featureID: Resources
        resources:
        - group: kyverno.io
          kind: ClusterPolicy
          lastAppliedTime: "2022-09-19T21:20:18Z"
          name: no-gateway
          owner:
            kind: ConfigMap
            name: kyverno-disallow-gateway
            namespace: default
      clusterProfileName: demo
```


## Detecting conflicts
Multiple ClusterProfiles can match same CAPI cluster. Because of that misconfiguration can happen and need to be detected.

For instance:
1. ClusterProfile A references ConfigMap A containing a Kyverno ClusterPolicy called *no-gateway"
2. ClusterProfile B references ConfigMap B containing a Kyverno ClusterPolicy called *no-gateway"

In such case, ClusterProfile will be allowed to deploy the Kyverno ClusterPolicy, while ClusterProfile B will report a conflict.

Note that in following example there is no conflict since both ClusterProfiles are referencing same ConfigMap:
1. ClusterProfile A references ConfigMap A containing a Kyverno ClusterPolicy called *no-gateway"
2. ClusterProfile B references ConfigMap A containing a Kyverno ClusterPolicy called *no-gateway"

Another example of misconfiguration is when two different ClusterProfiles match same CAPI Cluster(s) and both want to deploy same Helm chart in the same namespace.

In such a case, only one ClusterProfile will be elected and given permission to manage a specific helm release in a given CAPI cluster. Other ClusterProfiles will report such misconfiguration.

## Getting started on any Kubernetes cluster
You can use [KIND](https://sigs.k8s.io/kind) to get a local cluster for testing, or run against a remote cluster.

First you need to install ClusterAPI in such cluster. [ClusterAPI instruction](https://cluster-api.sigs.k8s.io/user/quick-start.html) can be followed.

Second you need to install the CRD and Deployment for the project in the management cluster:

### Deploy YAML
You can post the YAML to the management cluster

```
kubectl create -f  https://raw.githubusercontent.com/projectsveltos/cluster-api-feature-manager/master/manifest/manifest.yaml
```

### Install CRD and Deployment
1. . Deploy the controller to the cluster with the image specified by `IMG`:

```sh
make deploy IMG=<some-registry>/cluster-api-feature-manager:tag
```

### Uninstall CRDs
To delete the CRDs from the cluster:

```sh
make uninstall
```

### Undeploy controller
UnDeploy the controller to the cluster:

```sh
make undeploy
```

## Contributing
If you have questions, noticed any bug or want to get the latest project news, you can connect with us in the following ways:
1. Open a bug/feature enhancement on github;

## License

Copyright 2022.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
