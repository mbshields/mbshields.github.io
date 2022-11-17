# Sveltos - A Feature Manager for K8s Clusters

<img src="../assets/images/logo.png" width="200">

## What is Sveltos?

[Sveltos](https://github.com/projectsveltos/sveltos-manager) is a tool for policy driven management of features and resources in ClusterAPI powered Kubernetes clusters. Sveltos extends the functionality of ClusterAPI (CAPI) to add management of Kubernetes resources and Helm charts. Sveltos is a lightweight, freely available open source project that can be installed on a Kubernetes cluster in minutes.

## How does Sveltos help?

Sveltos provides declarative APIs for provisioning features such as [Helm](https://helm.sh) charts, ingress controllers, CNIs, storage classes, and other resources in a set of Kubernetes clusters. 

The Sveltos project follows the Kubernetes [Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) and uses [Controllers](https://kubernetes.io/docs/concepts/architecture/controller/), providing a reconcile function responsible for synchronizing resources until the desired state is reached on the cluster. Changes can be modeled, committed, and rolled back if necessary.

## What does Sveltos require?

Sveltos requires an existing cluster with [ClusterAPI](https://github.com/kubernetes-sigs/cluster-api) installed, which we refer to as a CAPI cluster. ClusterAPI is a Kubernetes sub-project that provides declarative APIs and tooling to simplify provisioning, upgrading, and operating multiple Kubernetes clusters.

## How does Sveltos work?

Sveltos policies and operation are defined in a YAML file as a profile that can be applied to a CAPI cluster object. The Sveltos profile directs Kubernetes to perform the following operations:

1. Identify any [CAPI clusters](https://github.com/kubernetes-sigs/cluster-api/blob/main/api/v1beta1/cluster_types.go) having a specific Kubernetes [label selector](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors).
2. Deploy Sveltos-specified features ([helm releases](https://helm.sh) or Kubernetes resources) on the selected cluster or group of clusters.

## Here's a quick example:

In this example, Sveltos specifies that any CAPI cluster with the label _env: prod_ will have the following features deployed:
*  a Kyverno helm chart (version v2.5.0)
*  a kubernetes resource(s) contained in the referenced Secret: _default/storage-class_
*  a kubernetes resource(s) contained in the referenced ConfigMap: _default/contour_

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

When a CAPI cluster is found to match the `clusterSelector` _env=prod_, all of the specified features are automatically deployed in the cluster.

For a simple demonstration, watch the video [Sveltos Overview](https://www.youtube.com/watch?v=Ai5Mr9haWKM) on YouTube.

## Getting started with a sandbox

If you want to test it out, just execute, `make create-cluster` and it will:
1. create a [KIND](https://sigs.k8s.io/kind) cluster;
2. install ClusterAPI;
3. create a CAPI Cluster with Docker as infrastructure provider;
4. install CRD and the Deployment from this project;
5. create a ClusterProfile instance;
6. modify CAPI Cluster labels so to match ClusterProfile selector.


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

