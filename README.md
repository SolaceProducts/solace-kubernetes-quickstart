[![Build Status](https://travis-ci.org/SolaceProducts/solace-kubernetes-quickstart.svg?branch=master)](https://travis-ci.org/SolaceProducts/solace-kubernetes-quickstart)

# Install a Solace PubSub+ Event Broker: Software onto a Kubernetes cluster

The [Solace PubSub+ Platform](https://solace.com/products/platform/)'s [PubSub+ Event Broker: Software](https://solace.com/products/event-broker/software/) efficiently streams event-driven information between applications, IoT devices and user interfaces running in cloud, on-premises, and hybrid environments using open APIs and protocols like AMQP, JMS, MQTT, REST and WebSocket. It can be installed into a variety of public and private clouds, PaaS, and on-premises environments, and brokers in multiple locations can be linked together in an [event mesh](https://solace.com/what-is-an-event-mesh/) to dynamically share events across the distributed enterprise.

## Overview

This document provides a quick getting started guide to install a Solace PubSub+ Event Broker: Software (PubSub+ EBS) in various configurations onto a [Kubernetes](https://kubernetes.io/docs/home/) cluster. The recommended PubSub+ EBS version is 9.4 or later.

Detailed documentation is provided in the [Solace PubSub+ Event Broker: Software on Kubernetes Documentation](docs/PubSubPlusK8SDeployment.md).

This quick start is intended mainly for development and demo purposes. Consult the [Deployment Considerations](https://github.com/SolaceDev/solace-kubernetes-quickstart/blob/HelmReorg/docs/PubSubPlusK8SDeployment.md#pubsub-event-broker-deployment-considerations) section of the Documentation when planning your deployment.

This document is applicable to any platform supporting Kubernetes, with specific hints on how to set up a simple MiniKube deployment on a Linux-based machine. To view examples of other Kubernetes platforms see:

- [Deploying a Solace PubSub+ Event Broker: Software HA group onto a Google Kubernetes Engine](//github.com/SolaceProducts/solace-gke-quickstart )
- [Deploying a Solace PubSub+ Event Broker: Software HA Group onto an OpenShift 3.11 platform](//github.com/SolaceProducts/solace-openshift-quickstart )
- Deploying a Solace PubSub+ Event Broker: Software HA Group onto Amazon EKS (Amazon Elastic Container Service for Kubernetes): follow the [AWS documentation](//docs.aws.amazon.com/eks/latest/userguide/getting-started.html ) to set up EKS then this guide to deploy.
- [Install a Solace PubSub+ Event Broker: Software onto a Pivotal Container Service (PKS) cluster](//github.com/SolaceProducts/solace-pks )
- Deploying a Solace PubSub+ Event Broker: Software HA Group onto Azure Kubernetes Service (AKS): follow the [Azure documentation](//docs.microsoft.com/en-us/azure/aks/ ) to deploy an AKS cluster then this guide to deploy.

## How to deploy the Solace PubSub+ EBS onto Kubernetes

Solace PubSub+ EBS can be deployed in either a three-node High-Availability (HA) group, or as a single-node Standalone deployment. For simple test environments that need only to validate application functionality, a single instance will suffice. Note that in production, or any environment where message loss cannot be tolerated, an HA deployment is required.

We recommend using the Helm tool for convenience. An [alternative method](docs/PubSubPlusK8SDeployment.md#alternative-deployment-with-generating-templates-for-the-kubernetes-kubectl-tool) using generated templates is also provided in the detailed Documentation.

In this quick start we go through the steps to set up a PubSub+ EBS using [Solace PubSub+ Helm charts](//hub.helm.sh/charts/solace).

There are three Helm chart variants available with default small-size configurations:
1.	`pubsubplus-dev` - minimum footprint PubSub+ EBS for Developers (Standalone)
2.	`pubsubplus` - PubSub+ EBS Standalone, supporting 100 connections
3.	`pubsubplus-ha` - PubSub+ EBS HA, supporting 100 connections

For other PubSub+ EBS configurations or sizes, refer to the [PubSub+ Event Broker: Software Helm Chart Reference](/pubsubplus/README.md).

### 1. Get a Kubernetes environment

Follow your Kubernetes provider's instructions ([other options available here](https://kubernetes.io/docs/setup/)). Ensure you meet [minimum CPU, Memory and Storage requirements](docs/PubSubPlusK8SDeployment.md#cpu-and-memory-requirements) for the targeted PubSub+ EBS configuration size.
> Note: If using [MiniKube](https://kubernetes.io/docs/setup/learning-environment/minikube/), `minikube start` will setup Kubernetes by default on a VM with 2 CPUs and 2 GB memory allocated, which may leave the bare minimum resources for the PubSub+ EBS deployment. For more granular control, use the `--cpus` and `--memory` options.

Also have the `kubectl` tool [installed](https://kubernetes.io/docs/tasks/tools/install-kubectl/) locally.

Check to ensure your Kubernetes environment is ready:
```bash
# This shall return worker nodes listed and ready
kubectl get nodes
```

### 2. Install and configure Helm

Follow the [Helm Installation notes of your target release](https://github.com/helm/helm/releases) for your platform.
Note that Helm is transitioning from v2 to v3. Many deployments still use v2. The PubSub+ EBS can be deployed using either version, however concurrent use of v2 and v3 from the same command-line environment is not supported.

On Linux a simple option to set up the latest stable release is to run:

<details open=true><summary><b>Instructions for Helm v2 setup</b></summary>
<p>

```bash
curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get | bash
```

Deploy Tiller, Helm's in-cluster operator:
```bash
# This enables getting started on most platforms by granting Tiller cluster-admin privileges
kubectl -n kube-system create serviceaccount tiller
kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
helm init --wait --service-account=tiller --upgrade # this may take some time
```
Warning: [more restricted Tiller privileges](/docs/PubSubPlusK8SDeployment.md#install-and-setup-the-helm-package-manager) are recommended in a production environment.
</p>
</details>

<details><summary><b>Instructions for Helm v3 setup</b></summary>
<p>

```bash
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```
</p>
</details>


Helm is configured properly if the command `helm version` returns no error.


### 3. Install the Solace PubSub+ EBS with default configuration

- Add the Solace Helm charts to your local Helm repo:
```bash
  helm repo add solacecharts https://solacedev.github.io/solace-kubernetes-quickstart/helm-charts
```

- By default the publicly available [latest Docker image of PubSub+ EBS Standard Edition](https://hub.Docker.com/r/solace/solace-pubsub-standard/tags/) will be used. Specify a different image or [use a Docker image from a private registry](/docs/PubSubPlusK8SDeployment.md#using-private-registries) if required. If using a different image, add the `image.repository=<your-image-location>,image.tag=<your-image-tag>` values to the `--set` commands below, comma-separated.

- Use one of the following chart variants to create a deployment. For configuration options and delete instructions, consult the [PubSub+ EBS Helm Chart Reference](https://github.com/SolaceDev/solace-kubernetes-quickstart/tree/HelmReorg/pubsubplus).

<details open=true><summary><b>Instructions using Helm v2</b></summary>
<p>

a) Create a Solace PubSub+ EBS minimum deployment for development purposes using `pubsubplus-dev`. It requires a minimum of 1 CPU and 2 GB of memory be available to the PubSub+ EBS pod.
```bash
# Deploy PubSub+ EBS Standard edition, minimum footprint developer version
helm install --name my-release solacecharts/pubsubplus-dev
```

b) Create a Solace PubSub+ Standalone deployment, supporting 100 connections scaling using `pubsubplus`. A minimum of 2 CPUs and 4 GB of memory must be available to the PubSub+ EBS pod.
```bash
# Deploy PubSub+ EBS Standard edition, Standalone
helm install --name my-release solacecharts/pubsubplus
```

c) Create a Solace PubSub+ HA deployment, supporting 100 connections scaling using `pubsubplus-ha`. The minimum resource requirements are 2 CPU and 4 GB of memory available to each of the three PubSub+ EBS pods.
```bash
# Deploy PubSub+ EBS Standard edition, HA
helm install --name my-release solacecharts/pubsubplus-ha
```
</p>
</details>

<details><summary><b>Instructions using Helm v3</b></summary>
<p>

a) Create a Solace PubSub+ EBS minimum deployment for development purposes using `pubsubplus-dev`. It requires a minimum of 1 CPU and 2 GB of memory available to the PubSub+ EBS pod.
```bash
# Deploy PubSub+ EBS Standard edition, minimum footprint developer version
helm install my-release solacecharts/pubsubplus-dev
```

b) Create a Solace PubSub+ Standalone deployment, supporting 100 connections scaling using `pubsubplus`. A minimum of 2 CPUs and 4 GB of memory must be available to the PubSub+ EBS pod.
```bash
# Deploy PubSub+ EBS Standard edition, Standalone
helm install my-release solacecharts/pubsubplus
```

c) Create a Solace PubSub+ HA deployment, supporting 100 connections scaling using `pubsubplus-ha`. The minimum resource requirements are 2 CPU and 4 GB of memory available to each of the three event broker pods.
```bash
# Deploy PubSub+ EBS Standard edition, HA
helm install my-release solacecharts/pubsubplus-ha
```
</p>
</details>

The above options will start the deployment and write related information and notes to the screen.

> Note: When using MiniKube, there is no integrated Load Balancer, which is the default service type. For a workaround, execute `minikube service my-release-pubsubplus-dev` to expose the services. Services will be accessible directly using the NodePort instead of direct Port access, for which the mapping can be obtained from `kubectl describe service my-release-pubsubplus-dev`.

Wait for the deployment to complete following the information printed on the console.

Refer to the detailed PubSub+ EBS Kubernetes documentation for:
* [Validating the deployment](//github.com/SolaceDev/solace-kubernetes-quickstart/blob/HelmReorg/docs/PubSubPlusK8SDeployment.md#validating-the-deployment); or
* [Troubleshooting](//github.com/SolaceDev/solace-kubernetes-quickstart/blob/HelmReorg/docs/PubSubPlusK8SDeployment.md#troubleshooting)
* [Modifying or Upgrading](//github.com/SolaceDev/solace-kubernetes-quickstart/blob/HelmReorg/docs/PubSubPlusK8SDeployment.md#modifying-or-upgrading-a-deployment)
* [Deleting the deployment](//github.com/SolaceDev/solace-kubernetes-quickstart/blob/HelmReorg/docs/PubSubPlusK8SDeployment.md#deleting-a-deployment)

## Contributing

Please read [CONTRIBUTING.md](CONTRIBUTING.md) for details on our code of conduct, and the process for submitting pull requests to us.

## Authors

See the list of [contributors](//github.com/SolaceProducts/solace-kubernetes-quickstart/graphs/contributors) who participated in this project.

## License

This project is licensed under the Apache License, Version 2.0. - See the [LICENSE](LICENSE) file for details.

## Resources

For more information about Solace technology in general please visit these resources:

- The Solace Developer Portal website at: [solace.dev](//solace.dev/)
- Understanding [Solace technology](//solace.com/products/platform/)
- Ask the [Solace community](//dev.solace.com/community/).
