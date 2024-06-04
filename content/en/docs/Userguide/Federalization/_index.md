---
title: "Resource Federalization"
linkTitle: "Resource Federalization"
weight: 1
date: 2024-05-30
description: KubeAdmiral offers the option to federate existing resources within a member cluster, making it convenient and efficient to take control of these resources. This article will guide you on how to perform resource federalization.

---
## What is Resource Federalization
Assume there is a member cluster already associated with a host cluster, and it has deployed resources (such as Deployments) that are not managed by KubeAdmiral. In such cases, we can refer to the [How to Perform Resource Federalization](#how-to-perform-resource-federalization) section to directly hand over the management of those resources to KubeAdmiral without causing a restart of pods belonging to workload-type resources. This capability is provided by resource federalization.
## How to perform Resource Federalization {#how-to-perform-resource-federalization}
### Before you begin
Refer to the [Quickstart](../../getting-started/quick-start/) section for a quick launch of KubeAdmiral.
### Create some resources in the member cluster
1. Select the member cluster **kubeadmiral-member-1**. <br>
   ```Bash
   $ export KUBECONFIG=$HOME/.kube/kubeadmiral/member-1.config
   ```
2. Create the resource Deployment **my-nginx**. <br>
   ```Bash
   $ kubectl apply -f ./my-nginx.yaml
   ```
    ```yaml
    # ./my-nginx.yaml 
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: my-nginx
    spec:
      selector:
        matchLabels:
          run: my-nginx
      replicas: 2
      template:
        metadata:
          labels:
            run: my-nginx
        spec:
          containers:
            - name: my-nginx
              image: nginx
              ports:
                - containerPort: 80
    ```
3. Create the resource Service **my-nginx**. <br>
   ```Bash
   $ kubectl apply -f ./my-nginx-svc.yaml
   ```
    ```yaml 
    # ./my-nginx-svc.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: my-nginx
      labels:
        run: my-nginx
    spec:
      ports:
        - port: 80
          protocol: TCP
      selector:
        run: my-nginx
    ```
4. View the created resources. <br>
    ```console
    $ kubectl get pod,deploy,svc                                               
    NAME                            READY   STATUS    RESTARTS   AGE
    pod/my-nginx-5b56ccd65f-l7dm5   1/1     Running   0          29s
    pod/my-nginx-5b56ccd65f-ldfp4   1/1     Running   0          29s

    NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/my-nginx   2/2     2            2           29s

    NAME                 TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
    service/my-nginx     ClusterIP   10.96.72.40   <none>        80/TCP    25s
    ```
### Create a PropagationPolicy for resource binding in the host cluster
1. Select the host cluster. <br>
   ```Bash
   $ export KUBECONFIG=$HOME/.kube/kubeadmiral/kubeadmiral.config
   ```

2. Create the PropagationPolicy **nginx-pp**. <br>
   ```Bash
   $ kubectl apply -f ./propagationPolicy.yaml
   ```
    ```yaml 
    # ./propagationPolicy.yaml
    apiVersion: core.kubeadmiral.io/v1alpha1
    kind: PropagationPolicy
    metadata:
      name: nginx-pp
      namespace: default
    spec:
      placement:
      - cluster: kubeadmiral-member-1 #The member clusters participating in resource federalization are referred to as federated clusters. 
        preferences:
          weight: 1
      replicaRescheduling:
        avoidDisruption: true
      reschedulePolicy:
        replicaRescheduling:
          avoidDisruption: true
        rescheduleWhen:
          clusterAPIResourcesChanged: false
          clusterJoined: false
          clusterLabelsChanged: false
          policyContentChanged: true
      schedulingMode: Duplicate
      schedulingProfile: ""
      stickyCluster: false
    ```
### Create the same resource in the host cluster and associate it with the PropagationPolicy
1. Select the member cluster **kubeadmiral-member-1** and perform operations on it. <br>
   ```Bash
   $ export KUBECONFIG=$HOME/.kube/kubeadmiral/member-1.config
   ```
2. Retrieve and save the YAML for Deployment resources in the member cluster. <br>
    ```console
    $ kubectl get deploy my-nginx -oyaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      annotations:
        deployment.kubernetes.io/revision: "1"
        kubectl.kubernetes.io/last-applied-configuration: |
          {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"name":"my-nginx","namespace":"default"},"spec":{"replicas":2,"selector":{"matchLabels":{"run":"my-nginx"}},"template":{"metadata":{"labels":{"run":"my-nginx"}},"spec":{"containers":[{"image":"nginx","name":"my-nginx","ports":[{"containerPort":80}]}]}}}}
      creationTimestamp: "2023-08-30T02:26:57Z"
      generation: 1
      name: my-nginx
      namespace: default
      resourceVersion: "898"
      uid: 5b64f73b-ce6d-4ada-998e-db6f682155f6
    spec:
      progressDeadlineSeconds: 600
      replicas: 2
      revisionHistoryLimit: 10
      selector:
        matchLabels:
          run: my-nginx
      strategy:
        rollingUpdate:
          maxSurge: 25%
          maxUnavailable: 25%
        type: RollingUpdate
      template:
        metadata:
          creationTimestamp: null
          labels:
            run: my-nginx
        spec:
          containers:
          - image: nginx
            imagePullPolicy: Always
            name: my-nginx
            ports:
            - containerPort: 80
              protocol: TCP
            resources: {}
            terminationMessagePath: /dev/termination-log
            terminationMessagePolicy: File
          dnsPolicy: ClusterFirst
          restartPolicy: Always
          schedulerName: default-scheduler
          securityContext: {}
          terminationGracePeriodSeconds: 30
    status:
      availableReplicas: 2
      conditions:
      - lastTransitionTime: "2023-08-30T02:27:21Z"
        lastUpdateTime: "2023-08-30T02:27:21Z"
        message: Deployment has minimum availability.
        reason: MinimumReplicasAvailable
        status: "True"
        type: Available
      - lastTransitionTime: "2023-08-30T02:26:57Z"
        lastUpdateTime: "2023-08-30T02:27:21Z"
        message: ReplicaSet "my-nginx-5b56ccd65f" has successfully progressed.
        reason: NewReplicaSetAvailable
        status: "True"
        type: Progressing
      observedGeneration: 1
      readyReplicas: 2
      replicas: 2
      updatedReplicas: 2
    ```
3. Retrieve and save the YAML for Service resources in the member cluster. <br>
    ```console
    $ kubectl get svc my-nginx -oyaml
    apiVersion: v1
    kind: Service
    metadata:
      annotations:
        kubectl.kubernetes.io/last-applied-configuration: |
          {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"run":"my-nginx"},"name":"my-nginx","namespace":"default"},"spec":{"ports":[{"port":80,"protocol":"TCP"}],"selector":{"run":"my-nginx"}}}
      creationTimestamp: "2023-08-30T02:27:01Z"
      labels:
        run: my-nginx
      name: my-nginx
      namespace: default
      resourceVersion: "855"
      uid: cc06cd52-1a80-4d3c-8fcf-e416d8c3027d
    spec:
      clusterIP: 10.96.72.40
      clusterIPs:
      - 10.96.72.40
      ipFamilies:
      - IPv4
      ipFamilyPolicy: SingleStack
      ports:
      - port: 80
        protocol: TCP
        targetPort: 80
      selector:
        run: my-nginx
      sessionAffinity: None
      type: ClusterIP
    status:
      loadBalancer: {}
    ```
4. Merge the resource YAML and perform pre-processing for federation.(You can refer to the comments in resources.yaml.) <br>
   a. Remove the resourceVersion field from the resources. <br>
   b. For Service resources, remove the clusterIP and clusterIPs fields. <br>
   c. Add a label for the PropagationPolicy. <br>
   d. Add an annotation for resource takeover. <br>
    ```yaml
    # ./resources.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        kubeadmiral.io/propagation-policy-name: nginx-pp #Add a label for the PropagationPolicy.
      annotations:
        kubeadmiral.io/conflict-resolution: adopt #Add an annotation for resource takeove.
        deployment.kubernetes.io/revision: "1"
        kubectl.kubernetes.io/last-applied-configuration: |
          {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"name":"my-nginx","namespace":"default"},"spec":{"replicas":2,"selector":{"matchLabels":{"run":"my-nginx"}},"template":{"metadata":{"labels":{"run":"my-nginx"}},"spec":{"containers":[{"image":"nginx","name":"my-nginx","ports":[{"containerPort":80}]}]}}}}
      creationTimestamp: "2023-08-30T02:26:57Z"
      generation: 1
      name: my-nginx
      namespace: default
      #resourceVersion: "898" remove
      uid: 5b64f73b-ce6d-4ada-998e-db6f682155f6
    spec:
      progressDeadlineSeconds: 600
      replicas: 2
      revisionHistoryLimit: 10
      selector:
        matchLabels:
          run: my-nginx
      strategy:
        rollingUpdate:
          maxSurge: 25%
          maxUnavailable: 25%
        type: RollingUpdate
      template:
        metadata:
          creationTimestamp: null
          labels:
            run: my-nginx
        spec:
          containers:
            - image: nginx
              imagePullPolicy: Always
              name: my-nginx
              ports:
                - containerPort: 80
                  protocol: TCP
              resources: {}
              terminationMessagePath: /dev/termination-log
              terminationMessagePolicy: File
          dnsPolicy: ClusterFirst
          restartPolicy: Always
          schedulerName: default-scheduler
          securityContext: {}
          terminationGracePeriodSeconds: 30
    status:
      availableReplicas: 2
      conditions:
        - lastTransitionTime: "2023-08-30T02:27:21Z"
          lastUpdateTime: "2023-08-30T02:27:21Z"
          message: Deployment has minimum availability.
          reason: MinimumReplicasAvailable
          status: "True"
          type: Available
        - lastTransitionTime: "2023-08-30T02:26:57Z"
          lastUpdateTime: "2023-08-30T02:27:21Z"
          message: ReplicaSet "my-nginx-5b56ccd65f" has successfully progressed.
          reason: NewReplicaSetAvailable
          status: "True"
          type: Progressing
      observedGeneration: 1
      readyReplicas: 2
      replicas: 2
      updatedReplicas: 2
    ---
    apiVersion: v1
    kind: Service
    metadata:
      annotations:
        kubeadmiral.io/conflict-resolution: adopt #Add an annotation for resource takeove.
        kubectl.kubernetes.io/last-applied-configuration: |
          {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"run":"my-nginx"},"name":"my-nginx","namespace":"default"},"spec":{"ports":[{"port":80,"protocol":"TCP"}],"selector":{"run":"my-nginx"}}}
      creationTimestamp: "2023-08-30T02:27:01Z"
      labels:
        run: my-nginx
        kubeadmiral.io/propagation-policy-name: nginx-pp #Add a label for the PropagationPolicy.
      name: my-nginx
      namespace: default
      #resourceVersion: "855" remove
      uid: cc06cd52-1a80-4d3c-8fcf-e416d8c3027d
    spec:
      # Remove the clusterIP address, the network segment of the host cluster may conflict with that of the cluster.
      # clusterIP: 10.96.72.40
      # clusterIPs:
      # - 10.96.72.40
      ipFamilies:
        - IPv4
      ipFamilyPolicy: SingleStack
      ports:
        - port: 80
          protocol: TCP
          targetPort: 80
      selector:
        run: my-nginx
      sessionAffinity: None
      type: ClusterIP
    status:
      loadBalancer: {}
    ```
5. Select the host cluster. <br>
   ```Bash
   $ export KUBECONFIG=/Users/bytedance/.kube/kubeadmiral/kubeadmiral.config
   ```

6. Create resources in the host cluster. <br>
    ```console
    $ kubectl apply -f ./resources.yaml                                                                        
    deployment.apps/my-nginx created
    service/my-nginx created
    ```
### View the results of Resource Federalization
1. Check the distribution status of host cluster resources, successfully distributed to the member cluster. <br>
    ```console
    $ kubectl get federatedobjects.core.kubeadmiral.io -oyaml
    apiVersion: v1
    items:
      - apiVersion: core.kubeadmiral.io/v1alpha1
        kind: FederatedObject
        metadata:
          annotations:
            federate.controller.kubeadmiral.io/observed-annotations: kubeadmiral.io/conflict-resolution|deployment.kubernetes.io/revision,kubeadmiral.io/latest-replicaset-digests,kubectl.kubernetes.io/last-applied-configuration
            federate.controller.kubeadmiral.io/observed-labels: kubeadmiral.io/propagation-policy-name|
            federate.controller.kubeadmiral.io/template-generator-merge-patch: '{"metadata":{"annotations":{"kubeadmiral.io/conflict-resolution":null,"kubeadmiral.io/latest-replicaset-digests":null},"creationTimestamp":null,"finalizers":null,"labels":{"kubeadmiral.io/propagation-policy-name":null},"managedFields":null,"resourceVersion":null,"uid":null},"status":null}'
            internal.kubeadmiral.io/enable-follower-scheduling: "true"
            kubeadmiral.io/conflict-resolution: adopt
            kubeadmiral.io/pending-controllers: '[]'
            kubeadmiral.io/scheduling-triggers: '{"schedulingAnnotationsHash":"1450640401","replicaCount":2,"resourceRequest":{"millicpu":0,"memory":0,"ephemeralStorage":0,"scalarResources":null},"policyName":"nginx-pp","policyContentHash":"638791993","clusters":["kubeadmiral-member-2","kubeadmiral-member-3","kubeadmiral-member-1"],"clusterLabelsHashes":{"kubeadmiral-member-1":"2342744735","kubeadmiral-member-2":"3001383825","kubeadmiral-member-3":"2901236891"},"clusterTaintsHashes":{"kubeadmiral-member-1":"913756753","kubeadmiral-member-2":"913756753","kubeadmiral-member-3":"913756753"},"clusterAPIResourceTypesHashes":{"kubeadmiral-member-1":"2027866002","kubeadmiral-member-2":"2027866002","kubeadmiral-member-3":"2027866002"}}'
          creationTimestamp: "2023-08-30T06:48:42Z"
          finalizers:
            - kubeadmiral.io/sync-controller
          generation: 2
          labels:
            apps/v1: Deployment
            kubeadmiral.io/propagation-policy-name: nginx-pp
          name: my-nginx-deployments.apps
          namespace: default
          ownerReferences:
            - apiVersion: apps/v1
              blockOwnerDeletion: true
              controller: true
              kind: Deployment
              name: my-nginx
              uid: 8dd32323-b023-479a-8e60-b69a7dc1be28
          resourceVersion: "8045"
          uid: 444c83ec-2a3c-4366-b334-36ee9178df94
        spec:
          placements:
            - controller: kubeadmiral.io/global-scheduler
              placement:
                - cluster: kubeadmiral-member-1
          template:
            apiVersion: apps/v1
            kind: Deployment
            metadata:
              annotations:
                deployment.kubernetes.io/revision: "1"
                kubectl.kubernetes.io/last-applied-configuration: |
                  {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{"deployment.kubernetes.io/revision":"1","kubeadmiral.io/conflict-resolution":"adopt"},"creationTimestamp":"2023-08-30T02:26:57Z","generation":1,"labels":{"kubeadmiral.io/propagation-policy-name":"nginx-pp"},"name":"my-nginx","namespace":"default","uid":"5b64f73b-ce6d-4ada-998e-db6f682155f6"},"spec":{"progressDeadlineSeconds":600,"replicas":2,"revisionHistoryLimit":10,"selector":{"matchLabels":{"run":"my-nginx"}},"strategy":{"rollingUpdate":{"maxSurge":"25%","maxUnavailable":"25%"},"type":"RollingUpdate"},"template":{"metadata":{"creationTimestamp":null,"labels":{"run":"my-nginx"}},"spec":{"containers":[{"image":"nginx","imagePullPolicy":"Always","name":"my-nginx","ports":[{"containerPort":80,"protocol":"TCP"}],"resources":{},"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File"}],"dnsPolicy":"ClusterFirst","restartPolicy":"Always","schedulerName":"default-scheduler","securityContext":{},"terminationGracePeriodSeconds":30}}},"status":{"availableReplicas":2,"conditions":[{"lastTransitionTime":"2023-08-30T02:27:21Z","lastUpdateTime":"2023-08-30T02:27:21Z","message":"Deployment has minimum availability.","reason":"MinimumReplicasAvailable","status":"True","type":"Available"},{"lastTransitionTime":"2023-08-30T02:26:57Z","lastUpdateTime":"2023-08-30T02:27:21Z","message":"ReplicaSet \"my-nginx-5b56ccd65f\" has successfully progressed.","reason":"NewReplicaSetAvailable","status":"True","type":"Progressing"}],"observedGeneration":1,"readyReplicas":2,"replicas":2,"updatedReplicas":2}}
              generation: 1
              labels: {}
              name: my-nginx
              namespace: default
            spec:
              progressDeadlineSeconds: 600
              replicas: 2
              revisionHistoryLimit: 10
              selector:
                matchLabels:
                  run: my-nginx
              strategy:
                rollingUpdate:
                  maxSurge: 25%
                  maxUnavailable: 25%
                type: RollingUpdate
              template:
                metadata:
                  creationTimestamp: null
                  labels:
                    run: my-nginx
                spec:
                  containers:
                    - image: nginx
                      imagePullPolicy: Always
                      name: my-nginx
                      ports:
                        - containerPort: 80
                          protocol: TCP
                      resources: {}
                      terminationMessagePath: /dev/termination-log
                      terminationMessagePolicy: File
                  dnsPolicy: ClusterFirst
                  restartPolicy: Always
                  schedulerName: default-scheduler
                  securityContext: {}
                  terminationGracePeriodSeconds: 30
        status:
          clusters:
            - cluster: kubeadmiral-member-1
              lastObservedGeneration: 2
              status: OK
          conditions:
            - lastTransitionTime: "2023-08-30T06:48:42Z"
              lastUpdateTime: "2023-08-30T06:48:42Z"
              status: "True"
              type: Propagated
          syncedGeneration: 2
      - apiVersion: core.kubeadmiral.io/v1alpha1
        kind: FederatedObject
        metadata:
          annotations:
            federate.controller.kubeadmiral.io/observed-annotations: kubeadmiral.io/conflict-resolution|kubectl.kubernetes.io/last-applied-configuration
            federate.controller.kubeadmiral.io/observed-labels: kubeadmiral.io/propagation-policy-name|run
            federate.controller.kubeadmiral.io/template-generator-merge-patch: '{"metadata":{"annotations":{"kubeadmiral.io/conflict-resolution":null},"creationTimestamp":null,"finalizers":null,"labels":{"kubeadmiral.io/propagation-policy-name":null},"managedFields":null,"resourceVersion":null,"uid":null},"status":null}'
            internal.kubeadmiral.io/enable-follower-scheduling: "true"
            kubeadmiral.io/conflict-resolution: adopt
            kubeadmiral.io/pending-controllers: '[]'
            kubeadmiral.io/scheduling-triggers: '{"schedulingAnnotationsHash":"1450640401","replicaCount":0,"resourceRequest":{"millicpu":0,"memory":0,"ephemeralStorage":0,"scalarResources":null},"policyName":"nginx-pp","policyContentHash":"638791993","clusters":["kubeadmiral-member-1","kubeadmiral-member-2","kubeadmiral-member-3"],"clusterLabelsHashes":{"kubeadmiral-member-1":"2342744735","kubeadmiral-member-2":"3001383825","kubeadmiral-member-3":"2901236891"},"clusterTaintsHashes":{"kubeadmiral-member-1":"913756753","kubeadmiral-member-2":"913756753","kubeadmiral-member-3":"913756753"},"clusterAPIResourceTypesHashes":{"kubeadmiral-member-1":"2027866002","kubeadmiral-member-2":"2027866002","kubeadmiral-member-3":"2027866002"}}'
          creationTimestamp: "2023-08-30T06:48:42Z"
          finalizers:
            - kubeadmiral.io/sync-controller
          generation: 2
          labels:
            kubeadmiral.io/propagation-policy-name: nginx-pp
            v1: Service
          name: my-nginx-services
          namespace: default
          ownerReferences:
            - apiVersion: v1
              blockOwnerDeletion: true
              controller: true
              kind: Service
              name: my-nginx
              uid: 6a2a63a2-be82-464b-86b6-0ac4e6c3b69f
          resourceVersion: "8031"
          uid: 7c077821-3c7d-4e3b-8523-5b6f2b166e68
        spec:
          placements:
            - controller: kubeadmiral.io/global-scheduler
              placement:
                - cluster: kubeadmiral-member-1
          template:
            apiVersion: v1
            kind: Service
            metadata:
              annotations:
                kubectl.kubernetes.io/last-applied-configuration: |
                  {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{"kubeadmiral.io/conflict-resolution":"adopt"},"creationTimestamp":"2023-08-30T02:27:01Z","labels":{"kubeadmiral.io/propagation-policy-name":"nginx-pp","run":"my-nginx"},"name":"my-nginx","namespace":"default","uid":"cc06cd52-1a80-4d3c-8fcf-e416d8c3027d"},"spec":{"ipFamilies":["IPv4"],"ipFamilyPolicy":"SingleStack","ports":[{"port":80,"protocol":"TCP","targetPort":80}],"selector":{"run":"my-nginx"},"sessionAffinity":"None","type":"ClusterIP"},"status":{"loadBalancer":{}}}
              labels:
                run: my-nginx
              name: my-nginx
              namespace: default
            spec:
              clusterIP: 10.106.114.20
              clusterIPs:
                - 10.106.114.20
              ports:
                - port: 80
                  protocol: TCP
                  targetPort: 80
              selector:
                run: my-nginx
              sessionAffinity: None
              type: ClusterIP
        status:
          clusters:
            - cluster: kubeadmiral-member-1
              status: OK
          conditions:
            - lastTransitionTime: "2023-08-30T06:48:42Z"
              lastUpdateTime: "2023-08-30T06:48:42Z"
              status: "True"
              type: Propagated
          syncedGeneration: 2
    kind: List
    metadata:
      resourceVersion: "" 
    ```
3. Select the member cluster **kubeadmiral-member-1**. <br>
   ```Bash
   $ export KUBECONFIG=$HOME/.kube/kubeadmiral/member-1.config
   ```

4. View the status of pod resources in the member cluster, the restart has not been performed.<br>
    ```console
    $ kubectl get po
    NAME                        READY   STATUS    RESTARTS   AGE
    my-nginx-5b56ccd65f-l7dm5   1/1     Running   0          4h49m
    my-nginx-5b56ccd65f-ldfp4   1/1     Running   0          4h49m 
    ```