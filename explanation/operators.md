# Operators in Kubernetes

This is not an in depth explanation of operators, but a quick overview as it relates to Strimzi Operator.

## CRDs and Operators

An Operator is a pod (or set of pods) that encapsulates the intelligence that human operators would use to do standard tasks. The concept is not to be a replacement for human operators, but to be automation and leverage for safer operation and more powerful abstractions.

More often than not, an Operator is installed with one or more Custom Resource Definitions, which are usually abbreviated as CRDs. These are specs or schemas of JSON/YAML objects that can be created in Kubernetes. If I have a blueprint as a CRD, I can create one or many Custom Resources that reference that CRD and conform to its schema.

Operators than use Kubernetes APIs to watch these objects as they are created, modified, and deleted. Based on what's changing, they may take action such as updating other kubernetes resource, configurations, or communicate with external resources like updating a DNS provider in the case of `external-dns`

## Strimzi

Strimzi is probably the most common operator used outside of Confluent's own platform. Different audiences may prefer one or the other, but it will probably be related to the support desired from Confluent itself.

Strimzi is a set of 3 operators. The core Cluster Operator which controls Kafka clusters and Zookeeper clusters is the most common. The Entity Operator is just two operators, the User and Topic operators. It's worth noting that the Cluster Operator controls the Entity Operator, and the Entity Operator controls the User and Topic operators. This hierarchy isn't important most of the time, but if you want to manage Users and Topics in Strimzi, you need to install the Entity Operator when you install the Cluster Operator.

User and Topic operators unsurprisingly control Users and Topics. Users are importantly where ACLs are kept, not on Topics.

## Configuration in Operators

One configures operators through YAML applied to the Kubernetes cluster. This is no different in Strimzi. The operator itself has configuration, probably through a helm chart installation that you provide values for. The cluster and other Strimzi objects are configured through CRs created.

The flow of this can be anything. Kubernetes and indeed operators do not care how the configuration got applied to the Kubernetes API. Separation of concerns means they are only concerned with the state desired (the CR definition) and the current state it has to reconcile (the pods and other resources that it may or may not have created already so achieve the desired state).

We recommend using a [gitops](./gitops.md) workflow in every case possible.
