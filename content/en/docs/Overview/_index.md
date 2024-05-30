---
title: "Overview"
linkTitle: "Overview"
weight: 1
keywords: ["KubeAdmiral", "multi-cloud", "Kubernetes", "cloud native", "Architecture Design"]
description: "This document provides a brief description of what is KubeAdmiral, including its key features and explanations of core concepts."
---

## **What is KubeAdmiral**

KubeAdmiral is a powerful multi-Kubernetes management engine that is compatible with Kubernetes native APIs, provides rich scheduling policies and an extensible scheduling framework that can help you easily deploy and manage cloud native applications in multi-cloud/hybrid cloud/edge environments without any modification to the application.

## **Key features**

- **Unified Management of Multi-Clusters**
  - Support for managing Kubernetes clusters from public cloud providers such as Volcano Engine, Alibaba Cloud, Huawei Cloud, Tencent Cloud, and others.
  - Support for managing Kubernetes clusters from private cloud provider.
  - Support for managing user-built Kubernetes clusters.

- **Multi-Cluster Application Distribution**
  - Compatibility with various types of applications
    - Native Kubernetes resources such as Deployments, StatefulSets, ConfigMaps, etc.
    - Custom Resource Definitions (CRDs) with support for collecting custom status fields and enabling replica mode scheduling.
    - Helm Charts.
  - Cross-cluster scheduling modes
    - Duplicate mode for multi-cluster application distribution.
    - Static weight mode for replica distribution.
    - Dynamic weight mode for replica distribution.
  - Cluster selection strategies
    - Specifying specific member clusters.
    - All member clusters.
    - Cluster selection based on labels.
  - Automatic propagation of dependencies with follower scheduling
    - Built-in follow resources, such as workloads referencing ConfigMaps, Secrets, etc.
    - Specifying follow resources, where workloads can specify follow resources in labels, such as Services, Ingress, etc.
  - Rescheduling strategy configuration
    - Support for enabling/disabling rescheduling.
    - Support configuring rescheduling trigger conditions.
  - Seamless takeover of existing single-cluster resources
  - Overriding member cluster resource configurations with override policy
- **Failover**
  - Manually evict applications in case of cluster failures
  - Automatic migration of applications across clusters in case of application failures
    - Automatic migration of application replicas when they cannot be scheduled.
    - Automatic migration of faulty replicas when they recover.

## **Architecture**
The overall architecture of KubeAdmiral is shown as below:

![Architecture](../General/architecture.png)

The KubeAdmiral control plane runs in the host cluster and consists of the following components:
- Fed ETCD: Stores the KubeAdmiral API objects and federated Kubernetes objects managed by KubeAdmiral.
- Fed Kube Apiserver: A native Kubernetes API Server, the operation entry for federated Kubernetes objects.
- Fed Kube Controller Manager: A native Kubernetes controller, selectively enables specific controllers as needed, such as the namespace controller and garbage collector (GC) controller.
- KubeAdmiral Controller: Provides the core control logic of the entire system. For example, member cluster management, resource scheduling and distribution, fault migration, status aggregation, etc.

![Kube Admiral Controller](../General/kubeadmiral-controller.png)

The KubeAdmiral Controller consists of a scheduler and various controllers that perform core functionalities. Below are several key components:
- Federated Cluster Controller: Watches the FederatedCluster object and manages the lifecycle of member clusters. Including adding, removing, and collecting status of member clusters.
- Federate Controller: Watches the Kubernetes resources and creates FederatedObject objects for each individual resource.
- Scheduler：Responsible for scheduling resources to member clusters. In the replica scheduling scenario, it will calculate the replicas that each cluster deserves based on various factors.
- Sync Controller: Watches the FederatedObject object and is responsible for distributing federated resources to each member cluster.
- Status Controller: Responsible for collecting the status of resources deployed in member clusters. It retrieves and aggregates the status information of federated resources from each member cluster.

## **Concepts**

### **Cluster concepts**

#### Federation Control Plane(Federation/KubeAdmiral Cluster)

The Federation Control Plane (KubeAdmiral control plane) consists of core components of KubeAdmiral, providing a consistent Kubernetes API to the outside world. Users can use the Federation Control Plane for multi-cluster management, application scheduling and distribution, and other operations.

#### Member Clusters

Kubernetes clusters that have been joined to the Federation Control Plane. Different types of Kubernetes clusters can be joined to the Federation Control Plane to become member clusters, supporting the response to the propagation policy of the Federation Control Plane and completing application propagation.

### **Core concepts**

#### Federated Namespace

The namespace in the Federation Control Plane refers to the Federated namespace. When a namespace is created in the Federation Control Plane, the system automatically creates the same Kubernetes namespace in all member clusters. In addition, the Federated Namespace has the same meaning and function as the namespace in Kubernetes, and can be used to achieve logical isolation of user resources and fine-grained user resource management.

#### PropogationPolicy/ClusterPropagationPolicy

KubeAdmiral defines the policy of multi-cluster resources distribution through PropogationPolicy/ClusterPropagationPolicy. According to the policies, multiple replicas of application instances can be distributed and deployed to specified member clusters. When a single cluster fails, application replicas can be flexibly scheduled to other clusters to ensure high availability of the business. Currently supported scheduling modes include replica duplicated or divided scheduling.

#### OverridePolicy/ClusterOverridePolicy

The OverridePolicy/ClusterOverridePolicy is used to define different configurations for the distribution of the same resource in different member clusters. It uses the `JsonPatch` override syntax for configuration and supports three override operations: `add`, `remove`, and `replace`.

#### Resource Propagation

Within a federation cluster, various native Kubernetes resources are distributed and scheduled in multiple clusters, which is the core capability of KubeAdmiral:

- It provides replica scheduling modes such as duplicated propagation, dynamic weight propagation, and static weight propagation.
- It supports strategies such as automatically deploying associated resources of applications to corresponding member clusters, automatically migrating applications to other clusters after failures, and automatically taking over conflicting resources of member clusters which flexibly meet the resource scheduling requirements in multi-cluster scenarios.


