---
title: "Member clusters management"
linkTitle: "Member clusters management"
weight: 1
date: 2024-05-30
---

## Prerequisites

- Kubectl v0.20.15+
- KubeAdmiral cluster
- Client cert, client key and CA information of the new member cluster

## Associate member clusters

1.Get the kubeconfig of the member cluster

Replace `KUBEADMIRAL_CLUSTER_KUBECONFIG` with the kubeconfig path to connect to the KubeAdmiral cluster.

```Bash
$ export KUBECONFIG=KUBEADMIRAL_CLUSTER_KUBECONFIG
```

2.Encode the client certificate, client key, and CA for the member cluster

Replace `NEW_CLUSTER_CA`, `NEW_CLUSTER_CERT`, and `NEW_CLUSTER_KEY` with the CA(certificate authority), client certificate, and client key of the new member cluster, respectively (you can obtain these from the kubeconfig of the member cluster).

```Bash
$ export ca_data=$(base64 NEW_CLUSTER_CA)
$ export cert_data=$(base64 NEW_CLUSTER_CERT)
$ export key_data=$(base64 NEW_CLUSTER_KEY)
```

3.Create `Secret` in KubeAdmiral for storing connection information for the member cluster

Replace `CLUSTER_NAME` with the name of the new member cluster:

```Bash
$ kubectl apply -f - << EOF
apiVersion: v1
kind: Secret
metadata:
  name: CLUSTER_NAME
  namespace: kube-admiral-system
data:
  certificate-authority-data: $ca_data
  client-certificate-data: $cert_data
  client-key-data: $key_data
EOF
```

4.Create a `FederatedCluster` object for the member cluster in KubeAdmiral

Replace `CLUSTER_NAME` and `CLUSTER_ENDPOINT` with the name and address of the member cluster:

```Bash
$ kubectl apply -f - << EOF
apiVersion: core.kubeadmiral.io/v1alpha1
kind: FederatedCluster
metadata:
  name: $CLUSTER_NAME
spec:
  apiEndpoint: $CLUSTER_ENDPOINT
  secretRef:
    name: $CLUSTER_NAME
  useServiceAccount: true
EOF
```

5.View member cluster status

If both the KubeAdmiral control plane and the member cluster are working properly, the association process should complete quickly and you should see the following:

```Bash
$ kubectl get federatedclusters

NAME                   READY   JOINED   AGE
...
CLUSTER_NAME           True    True     1m
...
```

Note: The status of successfully associated member clusters should be `READY` and `JOINED`.

## Dissociate member clusters

6.Delete the `FederatedCluster` object corresponding to the member cluster in KubeAdmiral

Replace `CLUSTER_NAME` with the name of the member cluster to be deleted.

```Bash
$ kubectl delete federatedcluster CLUSTER_NAME
```

