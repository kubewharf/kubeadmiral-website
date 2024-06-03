---
title: "差异化策略"
date: 2024-05-30
weight: 6
keywords: ["差异化策略", "集群级差异化策略"]
description: "差异化策略用于管理资源在不同集群分发时的差异化配置。"
---

## 介绍 {#introduction}


差异化策略用于定义同一资源在不同集群中分发时的差异化配置，分为 OverridePolicy 和 ClusterOverridePolicy 两种类型。OverridePolicy 只能作用于 namespace 级别的资源， ClusterOverridePolicy 可以作用于 cluster 级别和 namespace 级别的资源。差异化策略采用 `JSONPatch` 覆写语法进行配置，除此外还提供了针对指定对象（包括：`Image`、`Command`、`Args`、`Labels`、`Annotations` 等）封装的覆写语法。常见使用场景如下：

- 通过 annotations 配置不同云服务商的定制特性。例如：针对不同云服务商 ingress、service 资源，可使用差异化策略，通过 annotations 开启不同规格的 LB 及相应的负载均衡策略配置。
- 独立调整应用在不同集群中的副本数。例如：my-nginx 应用声明的副本数为 3，可使用差异化策略将当前资源分发到集群 Cluster A、集群 Cluster B、集群 Cluster C 上的副本数目指定为 3，5，7。
- 独立调整应用在不同集群中的容器镜像。例如：应用分发到私有化集群和公有云集群时，可使用差异化策略独立配置容器镜像拉取的地址。
- 调整集群在应用中的某些配置。例如：应用分发到集群 Cluster A 之前，可使用差异化策略注入一个 Sidecar 容器。
- 为分发到某个集群上的资源实例配置集群信息，例如：`apps.my.company/running-in: cluster-01`。
- 针对指定集群资源进行变更发布。例如：当遇到如大促、突发流量、紧急扩容等情况，要对应用进行变更时，可以针对指定集群资源进行变更发布，减小风险范围；亦可将差异化策略删除或与资源解除关联，直接回滚到变更前的状态。

## 关于 OverridePolicy 和 ClusterOverridePolicy {#about-overridepolicy-and-clusteroverridepolicy}

OverridePolicy 和 ClusterOverridePolicy 除了 kind 不同之外，两者的结构完全一致。一个资源最多支持绑定 1 个 OverridePolicy 和 1 个 ClusterOverridePolicy， 分别通过标签 `kubeadmiral.io/override-policy-name` 和 `kubeadmiral.io/cluster-override-policy-name` 来指定。 如果一个资源同时绑定了 OverridePolicy 和 ClusterOverridePolicy，且这个资源是 namespace 级别的资源，则 ClusterOverridePolicy 和 OverridePolicy 会同时生效且生效的顺序为先 ClusterOverridePolicy 后 OverridePolicy；而如果这个资源是 cluster 级别的资源，则只有 ClusterOverridePolicy 会生效。

使用的方式如下所示：

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

## 编写差异化策略 {#writing-override-policy}

差异化策略支持在一个策略内配置多条覆写规则；在一条规则内支持多种语意来帮助用户选择一个或多个目标集群，包括：`clusters`、`clusterSelector`，以及 `clusterAffinity`；并且在一条规则内，同样支持配置多个覆写的操作。

一个典型的 OverridePolicy 如下所示：

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

### targetClusters

`targetClusters` 用于帮助用户选择覆写的目标集群，其包含了三种可选的集群选择方式：

- `clusers`：为一个集群列表，该值显性地枚举了该条覆写规则应该生效的集群列表，即：只有被调度到这个列表内的成员集群的资源才会生效该条覆写规则。
- `clusterSelector`：通过键值对形式的标签来匹配集群。如果有资源被调度到了 `clusterSelector` 匹配标签的成员集群中，该条覆写规则就会在这个成员集群内生效。
- `clusterAffinity`：通过亲和性条件中配置的选择器标签来匹配集群。如果有资源被调度到了符合 `clusterAffinity` 的成员集群中，该条覆写规则就会在这个成员集群内生效。

上述三个值为可选项，若同时配置多个，生效规则如下：

- 若使用 `clusterSelector` 类型的选择器，则目标成员集群必须匹配所有标签。
- 若使用 `clusterAffinity` 类型的集群标签，成员集群只需满足任意一个 `matchExpressions`，每个 `matchExpressions` 中的所有标签必须同时匹配。
- 若同时使用任意两种或三种，则目标成员集群需要 **同时满足** 所配置的几种条件才能被覆写规则生效。
- `targetClusters` 的三种选择方式如果都没有选择，即：`clusers` 为空或者长度为 0，且 `clusterSelector` 和 `clusterAffinity` 内容均为空时，表示匹配所有集群。

### overriders

`overriders` 是表明了选中的目标集群所要应用的覆写规则，目前支持 `JSONPatch` 的模式和针对指定对象（包括：`Image`、`Command`、`Args`、`Labels`、`Annotations` 等）封装的模式配置覆写规则。

#### JSONPatch

`JSONPatch` 的值是一个 patch 的列表，每一项 patch 需要包含：
- `path`：表示目标覆写字段的路径。
- `operator`：表示支持的操作，包括：add、remove、replace。
  - add: 向资源追加一个或多个元素。
  - remove: 从资源中删除一个或多个元素。
  - replace: 替换资源中的一个或多个元素。
- `value`：表示目标覆写字段的值，在 operator 为 add 或 replace 时必填，operator 为 remove 时不需要填写。

**注意：**

- 如果需要在 path 中使用 `~` 和 `/`，需要将其转义为 `~0` 和 `~1`。例如：原本的 json 为 { "foo/bar~": "baz" }，如果需要修改它的 key，那么你应该使用 `/foo~1bar~0` 作为 JSONPatch 的 path。
- 如果你需要在 path 中引用数组的末尾，可以使用 `-` 而不是索引。例如：json 为 { "foos": [ "bar1", "bar2" ]}，如果需要修改 foos 的最后一个值，可以使用 `foos/-1` 作为 JSONPatch 的 path; 如果要添加新的值到 foos 的最后，可以使用 `foos/-` 作为 JSONPatch 的 path。
- 关于 JSONPatch 的详情可参考 [https://jsonpatch.com](https://jsonpatch.com) 。

#### Image

`image` 表示覆写 Kubernetes 原生资源镜像仓库的各个字段。镜像仓库地址组成为：`[Registry '/'] Repository [ ":" Tag ][ "@" Digest ]`，针对镜像地址资源，涉及到的覆写语法参数如下：

- **containerNames**：设置ImagePath时，ContainerNames 将被忽略。如果为空，则镜像覆盖规则适用于所有容器。否则，此覆盖将针对 pod 模板中的指定容器或 init 容器。
- **imagePath**：ImagePath 表示目标的镜像地址所在的路径。例如：`/spec/template/spec/containers/0/image` 。如果为空，当资源类型为 Pod、CronJob、Deployment、StatefulSet、DaemonSet 或 Job 时，系统将自动解析镜像路径。
- **operations**：表示要对目标的操作方式。
  - **imageComponent**：必填项，表示要操作的镜像仓库地址的哪个组成部分，可选值如下。
    - Registry：镜像所在仓库地址。
    - Repository：镜像名称。
    - Tag：镜像版本号。
    - Digest：镜像标识符。
  - **operator**：操作符，可选值如下：`addIfAbsent`、`overwrite`、`delete`。 如果为空，默认行为是 overwrite
  - **value**：操作所需的值。对于 `addIfAbsent`、`overwrite` Value 不能为空。

例子如下：

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

#### Command 和 Args {#command-and-args}

`command` 和 `args` 表示覆写 Kubernetes 原生 pod 的 `command` 和 `args` 字段，涉及到的覆写语法参数如下：

- **containerName**：必填项，声明此覆盖将针对 pod 模板中的指定容器或 init 容器。
- **operator**：操作符，可选值如下：`append`、`overwrite`、`delete`。 如果为空，默认行为是 overwrite
- **value**：将应用于 containerName 的 `command` / `args` 的字符串数组。
  - 如果 operator 为 `append`，则 value 中的项目（不允许为空）将附加到 `command` / `args`。
  - 如果 operator 为 `overwrite`，则 containerName 的当前 `command` / `args` 将完全被 value 替换。
  - 如果 operator 为 `delete`，则 value 中与 `command` / `args` 匹配的项目将被删除。

例子如下：

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

#### Labels 和 Annotations {#labels-and-annotations}

`labels` 和 `annotations` 表示覆写 Kubernetes 原生资源的 `labels` 和 `annotations` 字段，涉及到的覆写语法参数如下：

- **operator**：操作符，可选值如下：`addIfAbsent`、`overwrite`、`delete`。 如果为空，默认行为是 overwrite
- **value**: 将应用于资源 `labels` / `annotations` 的 map。
  - 如果 operator 为 `addIfAbsent`，则 value 中的项目（不允许为空）将添加到 `labels` / `annotations` 中。
    - 对于 `addIfAbsent` 操作符，value 中的键不能与 `labels` / `annotations` 冲突。
  - 如果 operator 为 `overwrite`，则 value 中与 `labels` / `annotations` 相匹配的项目将被替换。
  - 如果 operator 为 `delete`，则 value 中与 `labels` / `annotations` 相匹配的项目将被删除。

例子如下：

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

#### 生效的顺序 {#order-of-effect}

多个 `overrideRules` 之间按照声明的顺序进行复写，越后面的覆写规则优先级越高。

同一个 overrideRules 内的覆写规则按照如下顺序执行：

1. Image
2. Command
3. Args
4. Annotations
5. Labels
6. JSONPatch

即 JSONPatch 覆写优先级最高。

而同一个 overrider 内的多个 operator 按照声明的顺序执行，越后面的 operator 优先级越高。
