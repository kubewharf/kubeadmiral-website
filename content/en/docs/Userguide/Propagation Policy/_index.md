---
title: "Propagation Policy"
linkTitle: "Propagation Policy"
weight: 1
date: 2024-05-30
---

## Overview

Kubeadmiral defines the propagation policy of multi-cluster applications in the federated cluster through PropagationPolicy/ClusterPropagationPolicy.
Multiple replicas of the application can be deployed to the specified member clusters according to the propagation policy.
When a member cluster fails, the replicas can be flexibly scheduled to other clusters to ensure the high availability of the business.

Currently supported scheduling modes include Duplicated and Divide.
Among them, schedulingMode Divide can be divided into Dynamic Weight and Static Weight.

## Policy type

Propagation policies can be divided into two categories according to the effective scope.
* Namespace-scope(**PropagationPolicy**): It indicates that the policy takes effect within the specified namespace.
* Cluster-scope(**ClusterPropagationPolicy**): It indicates that the policy takes effect in all namespaces within the cluster.

## Target cluster selection

PropagationPolicy provides multiple semantics to help users select the appropriate target cluster, including placement, clusterSelector, clusterAffinity, tolerations, and maxClusters.

### Placement

Users can configure Placement to make the propagation policy only take effect in the specified member cluster, and resources are only scheduled in the specified member cluster.

A typical scenario is to select multiple member clusters as deployment clusters to meet the requirements of high availability.
At the same time, Placement provides the Preference parameter to allow the configuration of the cluster, weight, and number of replicas for resource propagation,
which is suitable for the scenario of multi-cluster propagation.

* **cluster**: The cluster specified in the resource propagation, selected from the existing member clusters.
* **weight**: The relative weight in static weight propagation, with a value range of 1 to 100. The larger the number, the higher the relative weight,
  and the actual relative weight takes effect according to the member cluster configuration.
  For example, if the weights of the two selected deployment clusters are 1 (or 100), the static weights are each 50%.
* **minReplicas**: The minimum number of replicas of the current cluster.
* **maxReplicas**: The maximum number of replicas of the current cluster.

```yaml
placement:
  - cluster: member1
  - cluster: member2
```

The propagation policy configured above will propagate resources to the two clusters, member1 and member2.
The advanced usage of placement will be detailed in the sections in the following.

### ClusterSelector

Users can use cluster labels to match clusters.
Propagation policies take effect in member clusters that match clusterSelector labels, and resources are scheduled in member clusters that match clusterSelector labels.

If multiple labels are configured at the same time, the effective rules are as follows:

* If using clusterAffinity type cluster labels, member clusters only need to meet any one of the following "conditions", and all labels in each "condition" must match at the same time.
* If both clusterSelector and clusterAffinity cluster labels are used at the same time, the results between the two labels will take the intersection.
* Both clusterSelector and clusterAffinity are empty, indicating that all clusters are matched.

```yaml
clusterSelector:
  region: beijing
  az: zone1
```

The propagation policy configured above will propagate resources to the clusters with the two labels of "region: beijing" and "az: zone1".

### ClusterAffinity

The selector label configured in the mandatory scheduling condition is used to match the cluster.
The propagation policy takes effect in the member clusters that match the clusterAffinity label, and resources are only scheduled in the member clusters that match the clusterAffinity label.

If multiple labels are configured at the same time, the effective rules are as follows:

* If using clusterAffinity type cluster labels, member clusters only need to meet any one of the following "conditions", and all labels in each "condition" must match at the same time.
* If both clusterSelector and clusterAffinity cluster labels are used at the same time, the results between the two labels will take the intersection.
* Both clusterSelector and clusterAffinity are empty, indicating that all clusters are matched.

```yaml
clusterAffinity:
  matchExpressions:
    - key: region
      operator: In
      values:
        - beijing
    - key: provider
      operator: In
      values:
        - volcengine
```

The propagation policy configured above will propagate resources to the clusters with the two labels of "region: beijing" and "provider: volcengine".

### Tolerations

The cluster taint scheduling can be configured to configure taint tolerance as needed, and perform cluster scheduling according to the selected multiple taints.

```yaml
tolerations:
- effect: NoSchedule
  key: dedicated
  operator: Equal
  value: groupName
- effect: NoExecute
  key: special
  operator: Exists
```

Generally speaking, the scheduler will default to filter out clusters with the taints of NoSchedule and NoExecute,
while the propagation policy configured above can tolerate specific taints on the cluster.

### MaxClusters

The maximum number of clusters for replica scheduling can be used to configure the upper limit of the replica number of member clusters to which resources can be scheduled.
The value range is a positive integer. In a single cluster propagation scenario, the maximum number of clusters can be configured to 1.
For example: for task scheduling, if the maximum number of clusters is set to 1, the task will select a cluster with the best resources from multiple optional member clusters for scheduling and execution.

## Duplicated scheduling mode

Duplicate Scheduling mode, which means that exactly the same number of replicas are propagated in multiple member clusters.

```yaml
schedulingMode: Duplicate
placement:
  - cluster: member1
  - cluster: member2
    preferences:
      minReplicas: 3
      maxReplicas: 3
  - cluster: member3
    preferences:
      minReplicas: 1
      maxReplicas: 3
```

The propagation policy configured above will deploy resources in the clusters of member1, member2, and member3.
Member1 will use the replica number defined in the resource template, member2 will deploy 3 replicas, and member3 will deploy 1-3 replicas depending on the situation.

## Divided scheduling mode - dynamic weight

Dynamic weight scheduling strategy means that when scheduling resources, the controller will dynamically calculate the current available resources of each member cluster according to the preset dynamic weight scheduling algorithm, and dynamically propagate the replicas to multiple member clusters according to the expected total number, so as to achieve the purpose of automatically balancing resources between member clusters.

For example: if users need to propagate resources (target 5 replicas) by weight to the member clusters Cluster A and Cluster B. At this time, kubeadmiral will propagate different numbers of replicas to the member clusters according to the weight of the cluster.
If the dynamic cluster weight is selected, it will be propagated according to the weight calculated by the system, and the actual number of replicas propagated to each member cluster depends on the total amount of cluster resources and the remaining resources.

```yaml
schedulingMode: Divide
placement:
  - cluster: member1
  - cluster: member2
```

The propagation policy configured above will propagate the replicas to the two clusters, member1 and member2, according to the dynamic weight scheduling strategy.

## Divided scheduling mode - static weight

Static weight scheduling strategy refers to the situation where the controller propagates replicas to multiple member clusters based on the weights manually configured by the user during resource scheduling.
The range of static cluster weights is 1-100, and the larger the number, the higher the relative weight. The actual relative weight configured by the effective member cluster takes effect.

For example, users need to propagate resources (target 5 replicas) to member clusters Cluster A and Cluster B according to weight. At this time, kubeadmiral will propagate different number of replicas to member clusters according to the weight of the cluster.
If static weight scheduling is selected and the weight is configured as Cluster A (30%): Cluster B (20%) = 3:2, then Cluster A will be propagated to 3 replicas and Cluster B will be propagated to 2 replicas.

```yaml
schedulingMode: Divide
placement:
  - cluster: member1
    preferences:
      weight: 40
  - cluster: member2
    preferences:
      weight: 60
```

The propagation policy of the above configuration will propagate replicas to two clusters, member1 and member2, according to the static weight (40:60) configured by the user.
For example, if the number of replicas of the resource object is 10, member1 will be propagated to 4 replicas, and member2 will be propagated to 6 replicas.

## Rescheduling

Kubeadmiral allows users to configure rescheduling behavior by configuring propagation policies.
The options related to rescheduling are as follows:
* **DisableRescheduling**: The overall switch of the rescheduling. If turned on, after resources are propagated to member clusters, it will trigger replica rescheduling according to the configured rescheduling conditions.
  If turned off, resource rescheduling will not be triggered due to resource modifications, policy changes, and other reasons after resources are propagated to member clusters.

* **RescheduleWhen**: Under the rescheduling mechanism, users can specify the conditions that trigger rescheduling. When the condition occurs, it will automatically trigger the rescheduling of resources according to the latest policy configuration and cluster environment.
  Resource template changes are the default configuration of the system and cannot be cancelled. Only changes in the request and replica fields will trigger rescheduling, and changes in other fields will only synchronize and update the configuration to the replicas in the propagated member cluster.
  In addition to resource template changes, Kubeadmiral provides the following optional trigger conditions.
    * **policyContentChanged**: When the propagation policy scheduling semantics change, the scheduler will trigger rescheduling. The policy scheduling semantics do not include the label, annotation, and autoMigration options. This trigger condition is enabled by default.
    * **clusterJoined**: When a new member cluster is added, the scheduler will trigger rescheduling. This trigger condition is disabled by default.
    * **clusterLabelsChanged**: When the member cluster label is changed, the scheduler will trigger rescheduling. This trigger condition is off by default.
    * **clusterAPIResourcesChanged**: When the member cluster API Resource changes, the scheduler will trigger rescheduling. This trigger condition is off by default.

* **ReplicaRescheduling**: The behavior of replicas propagation during rescheduling. Currently, only one option, avoidDisruption, is provided, which is enabled by default.
  When replicas are reallocated due to rescheduling, it will not affect the currently scheduled replicas.

When users do not explicitly configure the rescheduling option, the default behavior is as follows:

```yaml
reschedulePolicy:
  disableRescheduling: false
  rescheduleWhen:
    policyContentChanged: true
    clusterJoined: false
    clusterLabelsChanged: false
    clusterAPIResourcesChanged: false
  replicaRescheduling:
    avoidDisruption: true
```