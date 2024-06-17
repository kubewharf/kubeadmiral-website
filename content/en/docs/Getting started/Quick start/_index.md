---
title: "Quick start"
linkTitle: "Quick start"
weight: 2
---
## Prerequisites

- Kubectl version v0.20.15+
- KubeAdmiral cluster

## Propagating deployment resources with KubeAdmiral

The most common use case for KubeAdmiral is to manage Kubernetes resources across multiple clusters with a single unified API. This section shows you how to propagate Deployments to multiple member clusters and view their respective statuses using KubeAdmiral.

1.Make sure you are currently using the kubeconfig for the KubeAdmiral control plane.

```Bash
$ export KUBECONFIG=$HOME/.kube/kubeadmiral/kubeadmiral.config
```

2.Create a deployment object in KubeAdmiral

```YAML
$ kubectl create -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-server
  labels:
    app: echo-server
spec:
  replicas: 6
  selector:
    matchLabels:
      app: echo-server
  template:
    metadata:
      labels:
        app: echo-server
    spec:
      containers:
      - name: server
        image: ealen/echo-server:latest
        ports:
        - containerPort: 8080
          protocol: TCP
          name: echo-server
EOF
```

3.Create a new PropagationPolicy in KubeAdmiral

The following propagation policy will default to propagating bound resources to all clusters:

```YAML
$ kubectl create -f - <<EOF
apiVersion: core.kubeadmiral.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: policy-all-clusters
spec:
  schedulingMode: Divide
  clusterSelector: {}
EOF
```

4.Bind the created deployment object to the PropagationPolicy

Bind the specific PropagationPolicy by labeling the deployment object.

```Shell
$ kubectl label deployment echo-server kubeadmiral.io/propagation-policy-name=policy-all-clusters
```

5.Wait for the deployment object to be propagated to all member clusters

If the KubeAdmiral control plane is working properly, the deployment object will be quickly propagated to all member clusters. We can observe the brief propagation situation by looking at the number of ready replicas of the control plane deployment:

```Bash
$ kubectl get deploy echo-server

NAME          READY   UP-TO-DATE   AVAILABLE   AGE
echo-server   6/6     6            6           10m
```

Meanwhile, we can also view the specific status of the propagated resources in each member cluster through the CollectedStatus object:

```Bash
$ kubectl get collectedstatuses echo-server-deployments.apps -oyaml

apiVersion: core.kubeadmiral.io/v1alpha1
kind: CollectedStatus
metadata:
  name: echo-server-deployments.apps
  namespace: default
clusterStatus:
- clusterName: kubeadmiral-member-1
  collectedFields:
    metadata:
      creationTimestamp: "2023-03-14T08:02:05Z"
    spec:
      replicas: 2
    status:
      availableReplicas: 2
      conditions:
        - lastTransitionTime: "2023-03-14T08:02:10Z"
          lastUpdateTime: "2023-03-14T08:02:10Z"
          message: Deployment has minimum availability.
          reason: MinimumReplicasAvailable
          status: "True"
          type: Available
        - lastTransitionTime: "2023-03-14T08:02:05Z"
          lastUpdateTime: "2023-03-14T08:02:10Z"
          message: ReplicaSet "echo-server-65dcc57996" has successfully progressed.
          reason: NewReplicaSetAvailable
          status: "True"
          type: Progressing
      observedGeneration: 1
      readyReplicas: 2
      replicas: 2
      updatedReplicas: 2
- clusterName: kubeadmiral-member-2
  collectedFields:
    metadata:
      creationTimestamp: "2023-03-14T08:02:05Z"
    spec:
      replicas: 2
    status:
      availableReplicas: 2
      conditions:
        - lastTransitionTime: "2023-03-14T08:02:09Z"
          lastUpdateTime: "2023-03-14T08:02:09Z"
          message: Deployment has minimum availability.
          reason: MinimumReplicasAvailable
          status: "True"
          type: Available
        - lastTransitionTime: "2023-03-14T08:02:05Z"
          lastUpdateTime: "2023-03-14T08:02:09Z"
          message: ReplicaSet "echo-server-65dcc57996" has successfully progressed.
          reason: NewReplicaSetAvailable
          status: "True"
          type: Progressing
      observedGeneration: 1
      readyReplicas: 2
      replicas: 2
      updatedReplicas: 2
- clusterName: kubeadmiral-member-3
  collectedFields:
    metadata:
      creationTimestamp: "2023-03-14T08:02:05Z"
    spec:
      replicas: 2
    status:
      availableReplicas: 2
      conditions:
        - lastTransitionTime: "2023-03-14T08:02:13Z"
          lastUpdateTime: "2023-03-14T08:02:13Z"
          message: Deployment has minimum availability.
          reason: MinimumReplicasAvailable
          status: "True"
          type: Available
        - lastTransitionTime: "2023-03-14T08:02:05Z"
          lastUpdateTime: "2023-03-14T08:02:13Z"
          message: ReplicaSet "echo-server-65dcc57996" has successfully progressed.
          reason: NewReplicaSetAvailable
          status: "True"
          type: Progressing
      observedGeneration: 1
      readyReplicas: 2
      replicas: 2
      updatedReplicas: 2
```
