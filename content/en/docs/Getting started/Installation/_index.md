---
title: "Installation"
linkTitle: "Installation"
weight: 2
---
## Prerequisites

Make sure the following tools are installed in the environment before installing KubeAdmiral:

- [Go](https://go.dev/) version v1.19+
- [Kind](https://kind.sigs.k8s.io/) version v0.14.0+
- [Kubectl](https://github.com/kubernetes/kubectl) version v0.20.15+

## Local installation

If you want to understand how KubeAdmiral works, you can easily start a cluster with KubeAdmiral control plane on your local computer.

1.Clone the KubeAdmiral repository to your local environment：

```Bash
$ git clone https://github.com/kubewharf/kubeadmiral
```

2.Switch to the KubeAdmiral directory:

```Bash
$ cd kubeadmiral
```

3.Install and start KubeAdmiral:

```Bash
$ make local-up
```

The command performs the following tasks mainly:

- Use the Kind tool to start a meta-cluster;
- Install the KubeAdmiral control plane components on the meta-cluster;
- Use the Kind tool to start 3 member clusters;
- Bootstrap the joining of the 3 member clusters to the federated control plane;
- Export the cluster kubeconfigs to the $HOME/.kube/kubeadmiral directory.

If all the previous steps went successfully, we would see the following message:

```Bash
Your local KubeAdmiral has been deployed successfully!

To start using your KubeAdmiral, run:
  export KUBECONFIG=$HOME/.kube/kubeadmiral/kubeadmiral.config
  
To observe the status of KubeAdmiral control-plane components, run:
  export KUBECONFIG=$HOME/.kube/kubeadmiral/meta.config

To inspect your member clusters, run one of the following:
  export KUBECONFIG=$HOME/.kube/kubeadmiral/member-1.config
  export KUBECONFIG=$HOME/.kube/kubeadmiral/member-2.config
  export KUBECONFIG=$HOME/.kube/kubeadmiral/member-3.config
```

4.Wait for all member clusters to join KubeAdmiral

After the member clusters have successfully joined KubeAdmiral, we can observe the detailed status of the member clusters through the control plane: three member clusters have joined KubeAdmiral, and their statuses are all READY and JOINED.

```Bash
$ kubectl get fcluster

NAME                   READY   JOINED   AGE
kubeadmiral-member-1   True    True     1m
kubeadmiral-member-2   True    True     1m
kubeadmiral-member-3   True    True     1m
```