---
title: "Automatic propagation of associated resources"
weight: 5
---

## What are associated resources

The workloads (such as Deployments, StatefulSets, etc.) in Kubernetes usually rely on many other resources, such as ConfigMaps, Secrets, PVCs, etc.

Therefore,  it is necessary to ensure that when the workload is propagated to the target member cluster, the associated resources are synchronously distributed to the same member cluster.


The ways of associating workloads and other resources mainly include the following two types:

- **Built-in follower resources**: Refer to resources that are associated with each other in the Yaml configuration file. For example, workloads (Deployment, StatefulSet, etc.) and ConfigMap, Secret, PVC, etc. When resources are distributed, if the workload and associated resources are not distributed to the same cluster, it will cause the workload to fail to deploy due to resource missing.

- **Specified follower resources**: Mainly refer to **Service** and **Ingress**. The absence of specified follower resources will not cause failure in workload deployment, but will affect usage. For example, when service and ingress are not distributed to the same member cluster as the workload, the workload will not be able to provide services externally.


## Automatic propagation of associated resources

KubeAdmiral supports the automatic propagation of associated resources: when a workload is associated with related resources, KubeAdmiral will ensure that the workload and associated resources are scheduled to the same member cluster.

We name the automatic propagation of associated resources as follower scheduling.

## Supported resource types for follower scheduling
**Built-in follower resources**: Directly configure the associated resources when using YAML to configure workloads.

| Association Type             | Workloads     | Associated Resources  |
|------------------------------|---------------|-----------------------|
| Built-in follower resources  | Deployment    | ConfigMap             |\
|                              | StatefulSet   | Secret                |\
|                              | DaemonSet     | PersistentVolumeClaim |\
|                              | Job           | ServiceAccount        |\ 
|                              | CronJob       |                       |\                      
|                              | Pod           |                       |\

**Specified follower resources**: Using Annotations to declare the resources that need to be associated when creating workloads. 

| Association Type             | Workloads     | Associated Resources  |
|------------------------------|---------------|-----------------------|
| Specified follower resources | Deployment    | ConfigMap             |\
|                              | StatefulSet   | Secret                |\
|                              | DaemonSet     | PersistentVolumeClaim |\
|                              | Job           | ServiceAccount        |\
|                              | CronJob       | Service               |\
|                              | Pod           | Ingress               |


## How to configure follower scheduling

### Built-in follower resources

KubeAdmiral will propagate the build-in follower resources automatically which does not require users to add additional configurations.

For examples:

1. The Deployment A mounts the ConfigMap N, and the Deployment A is specified to be propagated to Cluster1 and Cluster2.

2. The ConfigMap N does not specify a propagation policy, but will follow Deployment A to be propagated to Cluster1 and Cluster2.


### Specified follower resources

When creating a workload, users can declare one or more associated resources using Annotations, which will be propagated to the target member clusters automatically along with the workload.

The format for specifying associated resources using Annotations is as follows:

- Annotation Key: `kubeadmiral.io/followers:`
- Each associated resource contains 3 fields: `group`, `kind`, and `name`. They are wrapped in `{}`. 
- When there are multiple associated resources, they are separated by `,`, and all resources are wrapped in `[]`.


Different associated resources have different field configurations in the Annotation, as follows:

| kind | group | name          | Anonotation 配置举例 |
| --- | --- |---------------| --- |
| ConfigMap | "" | resource name | kubeadmiral.io/followers: '\[{"group": "", "kind": "ConfigMap", "name": "configmap-name"}\]' |
| Secret | "" | resource name        | kubeadmiral.io/followers: '\[{"group": "", "kind": "Secret", "name": "secret-name"}\]' |
| Service | "" | resource name        | kubeadmiral.io/followers: '\[{"group": "", "kind": "Service", "name": "service-name"}\]' |
| PersistentVolumeClaim | "" | resource name        | kubeadmiral.io/followers: '\[{"group": "", "kind": "PersistentVolumeClaim, "name": "pvc-name"}\]' |
| ServiceAcount | "" | resource name        | kubeadmiral.io/followers: '\[{"group": "", "kind": "ServiceAcount, "name": "serviceacount-name"}\]' |
| Ingress | networking.k8s.io | resource name        | kubeadmiral.io/followers: '\[{"group": "networking.k8s.io", "kind": "Ingress, "name": "ingress-name"}\]' |

In this example, the Deployment is associated with two resources, namely Secret and Ingress.

```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    kubeadmiral.io/followers: '[{"group": "", "kind": "Secret", "name": "serect-demo"}, {"group": "networking.k8s.io",  "kind": "Ingress", "name": "ingress-demo"}]'
  name: deployment-demo
spec:
  replicas: 2 
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - image: demo-repo:v1
        name: demo
        ports: 
        - containerPort: 80
```

### Disable follower scheduling for workloads

Follower scheduling is enabled by default in KubeAdmiral. 
If users want to disable follower scheduling, they need to modify the `PropagationPolicy` by setting the `disableFollowerScheduling` field to `true`. Here is an example:

```YAML
apiVersion: core.kubeadmiral.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: follow-demo
  namespace: default
spec:
  disableFollowerScheduling: true
```

### Disable follower scheduling for associated resources

To prevent some associated resources from follower scheduling, users add the following declaration on the Annotation of the associated resources: `kubeadmiral.io/disable-following: "true"`

For example:
1. The Deployment A is mounted with ConfigMap N and Secret N, and the workload is specified to be propagated to Cluster1 and Cluster2.
2. If the user does not want Secret N to follow the scheduling, by adding the Annotation `kubeadmiral.io/disable-following: "true"` to Secret N, Secret N will not automatically be propagated to Cluster1 and Cluster2.
3. ConfigMap N will still follow Deployment A to be distributed to Cluster1 and Cluster2.

The YAML is as follows:

```YAML
apiVersion: v1
kind: Secret
metadata:
  annotations:
    kubeadmiral.io/disable-following: "true"
  name: follow-demo
  namespace: default
data: {}
```
