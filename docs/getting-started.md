# Getting Started

<img src="./images/logo.png" width="200">

- [Getting Started](#getting-started)
  - [Getting started with a sandbox](#getting-started-with-a-sandbox)
  - [Getting started on any Kubernetes cluster](#getting-started-on-any-kubernetes-cluster)
    - [Deploy YAML](#deploy-yaml)
    - [Install CRD and Deployment](#install-crd-and-deployment)
    - [Uninstall CRDs](#uninstall-crds)
    - [Undeploy controller](#undeploy-controller)
  - [Contributing](#contributing)
  - [License](#license)

[Sveltos](https://github.com/projectsveltos/sveltos-manager) is a tool for policy driven management of features and resources in ClusterAPI powered Kubernetes 

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
