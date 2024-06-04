---
title: "Override Policy"
date: 2024-05-30
weight: 6
keywords: ["override policy", "cluster override policy"]
description: "OverridePolicy and ClusterOverridePolicy are used to manage the differential configuration of resources when propagated to different clusters."
---

## Introduction {#introduction}

OverridePolicy and ClusterOverridePolicy are used to define differentiated configurations when the federated resource is propagated to different clusters. OverridePolicy can only act on namespaced resources, and ClusterOverridePolicy can act on cluster scoped and namespaced resources. The overrides are generally configured using `JSONPatch`. And in addition, overwriting syntax encapsulated for specified objects (including: `Image`, `Command`, `Args`, `Labels`, `Annotations`, etc.) is provided. Common usage scenarios are as follows:

- Configure customized features of different cloud service providers through annotations. For example, for the ingress and service resources of different cloud service providers, differentiated strategies can be used to enable LB of different specifications and corresponding load balancing policy configurations through annotations.
- Independently adjust the number of replicas of an application in different clusters. For example: the number of replicas declared by the my-nginx application is 3. You can use the OverridePolicy to force the specified resources to be propagated to the cluster: the number of replicas of Cluster A is 1, the number of replicas of Cluster B is 5, and the number of replicas of Cluster C is 7.
- Independently adjust container images applied in different clusters. For example: when an application is distributed to a private cluster and a public cloud cluster, OverridePolicy can be used to independently configure the address to be pulled by the container image.
- Adjust some configurations of the cluster in the application. For example: before the application is applied to cluster Cluster A, a OverridePolicy can be used to inject a sidecar container.
- Configure cluster information for resource instances distributed to a cluster, for example: `apps.my.company/running-in: cluster-01`.
- Publish changes to specified cluster resources. For example: when encountering situations such as major promotions, sudden traffic, emergency expansion, etc., and you need to make changes to the application, you can gradually release your changes to the designated clusters to reduce the risk scope; you can also delete the OverridePolicy or disassociate the OverridePolicy from the resources to roll back to the state before the change.

## About OverridePolicy and ClusterOverridePolicy  {#about-overridepolicy-and-clusteroverridepolicy}

Except for the difference in kind, the structures of OverridePolicy and ClusterOverridePolicy are exactly the same. A resource supports associating to a maximum of 1 OverridePolicy and 1 ClusterOverridePolicy, which are specified through the labels `kubeadmiral.io/override-policy-name` and `kubeadmiral.io/cluster-override-policy-name` respectively. If a namespace scoped resource has associated to both OverridePolicy and ClusterOverridePolicy, ClusterOverridePolicy and OverridePolicy will take effect at the same time and the order of effect is first ClusterOverridePolicy and then OverridePolicy; and if a cluster scoped resource  has associated to both OverridePolicy and ClusterOverridePolicy, only ClusterOverridePolicy will take effect.

The way to use them are as follows:

```YAML {lineNos=inline}
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: my-dep
    kubeadmiral.io/cluster-override-policy-name: my-cop   # Overwrite this Deployment via ClusterOverridePolicy.
    kubeadmiral.io/override-policy-name: my-op            # Overwrite this Deployment via OverridePolicy.
    kubeadmiral.io/propagation-policy-name: my-pp         # Propagate this Deployment via PropagationPolicy.
  name: my-dep
  namespace: default
...
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubeadmiral.io/cluster-override-policy-name: my-cop     # Overwrite this ClusterRole via ClusterOverridePolicy.
    kubeadmiral.io/cluster-propagation-policy-name: my-cpp  # Propagate this ClusterRole via ClusterPropagationPolicy.
  name: pod-reader
...
```

## Writing OverridePolicy {#writing-override-policy}

The OverridePolicy supports configuring multiple override rules within one policy. And it supports multiple semantics within one rule to help users select one or more target clusters, including: `clusters`, `clusterSelector`, and `clusterAffinity`. And within one rule, also supports configuring multiple override operations.

A typical OverridePolicy looks like this:

```YAML {lineNos=inline}
apiVersion: core.kubeadmiral.io/v1alpha1
kind: OverridePolicy
metadata:
  name: mypolicy
  namespace: default
spec:
  overrideRules:
    - targetClusters:
        clusters:
          - Cluster-01 # Modify the selected cluster to propagate the resource.
          - Cluster-02 # Modify the selected cluster to propagate the resource.
        #clusterSelector:
          #region: beijing
          #az: zone1
        #clusterAffinity:
          #- matchExpressions:
            #- key: region
              #operator: In
              #values:
              #- beijing
            #- key: provider
              #operator: In
              #values:
              #- my-provider
      overriders:
        jsonpatch:
          - path: /spec/template/spec/containers/0/image
            operator: replace
            value: nginx:test
          - path: /metadata/labels/hello
            operator: add
            value: world
          - path: /metadata/labels/foo
            operator: remove
```

### TargetClusters

`targetClusters` is used to help users select the target cluster for overwriting. It includes three optional cluster selection methods:

- `clusters`: This is a cluster list. This value explicitly enumerates the list of clusters in which this override rule should take effect. That is, only resources scheduled to member clusters in this list will take effect in this override rule.
- `clusterSelector`: Match clusters by labels in the form of key-value pairs. If a resource is scheduled to a member cluster whose `clusterSelector` matches the label, this override rule will take effect in this member cluster.
- `clusterAffinity`: Match clusters by affinity configurations of cluster labels. If a resource is scheduled to a member cluster that matches `clusterAffinity`, this override rule will take effect in this member cluster. It likes node affinity of Pod, you can see more detail from here: [Affinity and anti-affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity).

The three selectors above ​​are all optional. If multiple selectors ​​are configured at the same time, the effective rules are as follows:

- If `clusterSelector` is used, the target member cluster must match all labels.
- If `clusterAffinity` are used, member clusters only need to satisfy any one of the `matchExpressions`, but each label selectors in `matchExpressions` must be matched at the same time.
- If any two or three selectors are used at the same time, the target member cluster needs to **meet every selector at the same time** for the overwrite rule to take effect.
- If none of the three selection methods of targetClusters is selected, that is: clusters is empty or has a length of 0, and the contents of clusterSelector and clusterAffinity are both empty, it means matching all clusters.

### Overriders

`overriders` indicates the overriding rules to be applied to the selected target cluster. Currently, it supports `JSONPatch` and encapsulated overwriting syntax for specified objects(including: `Image`, `Command`, `Args`, `Labels`, `Annotations`, etc.).

#### JSONPatch

The value of `JSONPatch` is a list of patches, specifies overriders in a syntax similar to RFC6902 JSON Patch. Each patch needs to contain:

- `path`: Indicates the path of the target overwritten field.
- `operator`: Indicates supported operations, including: add, remove, replace.
  - `add`: Append or insert one or more elements to the resource.
  - `remove`: Remove one or more elements from the resource.
  - `replace`: Replace one or more elements in a resource.
- `value`: Indicates the value of the target overwrite field. It is required when the operator is add or replace. It does not need to be filled in when the operator is remove.

**Note:**

- If you need to refer to a key with `~` or `/` in its name, you must escape the characters with `~0` and `~1` respectively. For example, to get "baz" from `{ "foo/bar~": "baz" }` you’d use the pointer `/foo~1bar~0`.
- If you need to refer to the end of an array you can use - instead of an index. For example, to refer to the end of the array of biscuits above you would use /biscuits/-. This is useful when you need to insert a value at the end of an array.
- For more detail about JSONPatch: [https://jsonpatch.com](https://jsonpatch.com) 。

#### Image

`image` means overwriting various fields of container image. The container image address consists of: `[Registry '/'] Repository [ ":" Tag ] [ "@" Digest ]`. The overwriting syntax parameters involved are as follows:

- **containerNames**: `containerNames` are ignored when `imagePath` is set. If empty, the image override rule applies to all containers. Otherwise, this override targets the specified container(s) or init container(s) in the pod template.
- **imagePath**: `imagePath` represents the image path of the target. For example: `/spec/template/spec/containers/0/image`. If empty, the system will automatically resolve the image path when the resource type is Pod, CronJob, Deployment, StatefulSet, DaemonSet or Job.
- **operations**: Indicates the operation method to be performed on the target.
  - imageComponent: required, indicating which component of the image address to be operated on. Optional values ​​are as follows.
    - Registry: The address of the registry where the image is located.
    - Repository: Image name.
    - Tag: Image version number.
    - Digest: Image identifier.
  - operator: operator specifies the operation, optional values ​​are as follows: `addIfAbsent`, `overwrite`, `delete`. If empty, the default behavior is `overwrite`.
  - value: The value required for the operation. For `addIfAbsent`, `overwrite` value cannot be empty.

Example:

```YAML {lineNos=inline}
apiVersion: core.kubeadmiral.io/v1alpha1
kind: ClusterOverridePolicy
metadata:
  name: mypolicy
spec:
  overrideRules:
    - targetClusters:
        clusters:
          - kubeadmiral-member-1
      overriders:
        image:
          - containerNames: 
              - "server-1"
              - "server-2"
            operations: 
              - imageComponent: Registry
                operator: addIfAbsent
                value: cluster.io
    - targetClusters:
        clusters:
          - kubeadmiral-member-2
      overriders:
        image:
          - imagePath: "/spec/templates/0/container/image"
            operations: 
            - imageComponent: Registry
              operator: addIfAbsent
              value: cluster.io
            - imageComponent: Repository
              operator: overwrite
              value: "over/echo-server"
            - imageComponent: Tag
              operator: delete
            - imageComponent: Digest
              operator: addIfAbsent
              value: "sha256:aaaaf56b44807c64d294e6c8059b479f35350b454492398225034174808d1726"
```

#### Command and Args {#command-and-args}

`command` and `args` represent overwriting the command and args fields of the pod template. The overwriting syntax parameters involved are as follows:

- **containerName**: Required, declares that this override will target the specified container or init container in the pod template.
- **operator**: operator specifies the operation, optional values ​​are as follows: `append`, `overwrite`, `delete`. If empty, the default behavior is `overwrite`.
- **value**: String array of command/args that will be applied to containerName.
  - If operator is `append`, the items in value (empty is not allowed) are appended to command / args.
  - If operator is `overwrite`, containerName's current command / args will be completely replaced by value.
  - If operator is `delete`, items in value that match command / args will be deleted.

Examples:

```YAML {lineNos=inline}
apiVersion: core.kubeadmiral.io/v1alpha1
kind: ClusterOverridePolicy
metadata:
  name: mypolicy
spec:
  overrideRules:
    - targetClusters:
        clusters:
          - kubeadmiral-member-1
      overriders:
        command:
          - containerName: "server-1"
            operator: append
            value: 
              - "/bin/sh"
              - "-c"
              - "sleep 10s"
          - containerName: "server-2"
            operator: overwrite
            value: 
              - "/bin/sh"
              - "-c"
              - "sleep 10s"
          - containerName: "server-3"
            operator: delete
            value:  
              - "sleep 10s"
    - targetClusters:
        clusters:
          - kubeadmiral-member-2
      overriders:
        args:
          - containerName: "server-1"
            operator: append
            value:
              - "-v=4"
              - "--enable-profiling" 
```

#### Labels and Annotations {#labels-and-annotations}

`labels` and `annotations` represent overwriting the labels and annotations fields of Kubernetes resources. The overwriting syntax parameters involved are as follows:
- **operator**: operator specifies the operation, optional values ​​are as follows: `addIfAbsent`, `overwrite`, `delete`. If empty, the default behavior is `overwrite`.
- **value**: the map that will be applied to resource labels / annotations.
  - If operator is `addIfAbsent`, the items in value (empty is not allowed) will be added to labels / annotations.
    - For the `addIfAbsent` operator, keys in value cannot conflict with labels / annotations.
  - If operator is `overwrite`, items in value that match labels / annotations will be replaced.
  - If operator is `delete`, items in value that match labels / annotations will be deleted.

Examples:

```YAML {lineNos=inline}
apiVersion: core.kubeadmiral.io/v1alpha1
kind: ClusterOverridePolicy
metadata:
  name: mypolicy
spec:
  overrideRules:
    - targetClusters:
        clusters:
          - kubeadmiral-member-1
      overriders:
        labels:
          - operator: addIfAbsent
            value: 
              app: "chat"
          - operator: overwrite
            value: 
              version: "v1.1.0"
          - operator: delete
            value: 
              action: ""
    - targetClusters:
        clusters:
          - kubeadmiral-member-2
      overriders:
        annotations:
          - operator: addIfAbsent
            value: 
              app: "chat"
          - operator: overwrite
            value: 
              version: "v1.1.0"
          - operator: delete
            value: 
              action: ""
```

#### Order of effect  {#order-of-effect}

Multiple `overrideRules` are overridden in the order of declaration, and the later override rules have higher priority.

Override rules within the same overrideRules are executed in the following order:

1. Image
2. Command
3. Args
4. Annotations
5. Labels
6. JSONPatch

So, JSONPatch has the highest overwriting priority.

Multiple operators within the same overrider are executed in the order of declaration, and the later operators have higher priority.