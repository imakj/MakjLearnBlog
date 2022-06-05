# Kubernetes 对象

参考文档：https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/kubernetes-objects/

在 Kubernetes 系统中，*Kubernetes 对象* 是持久化的实体。 Kubernetes 使用这些实体去表示整个集群的状态。特别地，它们描述了如下信息：

* 哪些容器化应用在运行（以及在哪些节点上）
* 可以被应用使用的资源
* 关于应用运行时表现的策略，比如重启策略、升级策略，以及容错策略

查看一个yaml文件,通常情况下一个yaml会被分成5个内容

1. apiVersion：创建该对象所使用的 Kubernetes API 的版本
2. kind：想要创建的对象的类别除了比如：Deployment
3. metadata：帮助唯一性标识对象的一些数据，包括一个 `name` 字符串、UID 和可选的 `namespace`
4. spec：定义的资源期望的目标的样子
5. status：当前容器的一个状态样子。和spec进行匹配，如果和spec不一样，会进行匹配协调达到一致

```
[root@k8snode1 k8s]# kubectl get deployments.apps nginxdemo1 -o yaml
apiVersion: apps/v1 
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "4"
  creationTimestamp: "2021-04-18T15:42:17Z"
  generation: 5
  labels:
    app: nginxdemo1
  name: nginxdemo1
  namespace: default
  resourceVersion: "349108"
  uid: bb080a17-ec85-4eb7-81cd-f9c9997b3614
spec:
  progressDeadlineSeconds: 600
  replicas: 4
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nginxdemo1
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginxdemo1
    spec:
      containers:
      - image: nginx:1.7.9
        imagePullPolicy: IfNotPresent
        name: nginx
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
  availableReplicas: 4
  conditions:
  - lastTransitionTime: "2021-04-19T06:39:09Z"
    lastUpdateTime: "2021-04-19T06:39:09Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: "2021-04-18T15:42:17Z"
    lastUpdateTime: "2021-04-19T07:25:24Z"
    message: ReplicaSet "nginxdemo1-6995675b4b" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  observedGeneration: 5
  readyReplicas: 4
  replicas: 4
  updatedReplicas: 4

```

# k8s对象管理

参考：https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/object-management/

* Imperative commands：命令行的交互。适用于在线开发，可以快速定义一个对象，开发建议
* Imperative object configuration：资源定义清单，就是yaml定义方式，生产建议
* Declarative object configuration：声明式对象配置，一个资源对象包含多个资源对象的时候，可以讲多个资源对象放在一个目录里，进行打包



| Management technique             | Operates on          | Recommended environment | Supported writers | Learning curve |
| -------------------------------- | -------------------- | ----------------------- | ----------------- | -------------- |
| Imperative commands              | Live objects         | Development projects    | 1+                | Lowest         |
| Imperative object configuration  | Individual files     | Production projects     | 1                 | Moderate       |
| Declarative object configuration | Directories of files | Production projects     | 1+                | Highest        |

## 

# 创建YAML资源文件

* 使用--dry-run模式。有三种选项
  * server
  * client：尝试提交，但不会真正提交
  * none

1. 创建一个yaml文件，然后就可以根据这个来修改

   ```
   [root@k8snode1 ~]# kubectl create deployment yamltest --image=nginx:1.7.9 --dry-run -o yaml
   W0419 22:37:05.454466   10155 helpers.go:557] --dry-run is deprecated and can be replaced with --dry-run=client.
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     creationTimestamp: null
     labels:
       app: yamltest
     name: yamltest
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: yamltest
     strategy: {}
     template:
       metadata:
         creationTimestamp: null
         labels:
           app: yamltest
       spec:
         containers:
         - image: nginx:1.7.9
           name: nginx
           resources: {}
   status: {}
   
   #可以转存到文件中
   [root@k8snode1 k8s]# kubectl create deployment yamltest --image=nginx:1.7.9 --dry-run -o yaml > yamltest.yaml
   W0419 22:38:26.664209   12019 helpers.go:557] --dry-run is deprecated and can be replaced with --dry-run=client.
   [root@k8snode1 k8s]# cat yamltest.yaml 
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     creationTimestamp: null
     labels:
       app: yamltest
     name: yamltest
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: yamltest
     strategy: {}
     template:
       metadata:
         creationTimestamp: null
         labels:
           app: yamltest
       spec:
         containers:
         - image: nginx:1.7.9
           name: nginx
           resources: {}
   status: {}
   
   #直接提交这个文件就可以创建一个容器
   [root@k8snode1 k8s]# kubectl apply -f yamltest.yaml 
   deployment.apps/yamltest created
   [root@k8snode1 k8s]# kubectl get deployments.apps 
   NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
   nginx-dashboard-demo   3/3     3            3           5h6m
   nginxdemo1             4/4     4            4           22h
   yamltest               1/1     1            1           41s
   
   ```

2. 删除容器资源

   ```
   [root@k8snode1 k8s]# kubectl delete -f yamltest.yaml 
   deployment.apps "yamltest" deleted
   [root@k8snode1 k8s]# kubectl get deployments.apps 
   NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
   nginx-dashboard-demo   3/3     3            3           5h7m
   nginxdemo1             4/4     4            4           22h
   
   ```

# Namespaces

可以隔离不通的资源，比如隔离测试和开发环境,当资源属于集群的时候，不会出现在Namespaces下

1. 查看所有的namespaces

   ```
   [root@k8snode1 k8s]# kubectl get namespaces 
   NAME                   STATUS   AGE
   default                Active   47h
   kube-node-lease        Active   47h
   kube-public            Active   47h
   kube-system            Active   47h
   kubernetes-dashboard   Active   5h41m
   tigera-operator        Active   47h
   
   ```

2. 再创建资源的时候 就可以指定一个命名空间

   ```
   [root@k8snode1 k8s]# kubectl apply -f yamltest.yaml --namespace=kube-system
   ```

3. 当资源属于集群的时候，不会出现在Namespaces下，可以通过api-resource查看

   ```
   [root@k8snode1 k8s]# kubectl api-resources --namespaced=true
   NAME                        SHORTNAMES   APIVERSION                     NAMESPACED   KIND
   bindings                                 v1                             true         Binding
   configmaps                  cm           v1                             true         ConfigMap
   endpoints                   ep           v1                             true         Endpoints
   events                      ev           v1                             true         Event
   limitranges                 limits       v1                             true         LimitRange
   persistentvolumeclaims      pvc          v1                             true         PersistentVolumeClaim
   pods                        po           v1                             true         Pod
   podtemplates                             v1                             true         PodTemplate
   replicationcontrollers      rc           v1                             true         ReplicationController
   resourcequotas              quota        v1                             true         ResourceQuota
   secrets                                  v1                             true         Secret
   serviceaccounts             sa           v1                             true         ServiceAccount
   services                    svc          v1                             true         Service
   controllerrevisions                      apps/v1                        true         ControllerRevision
   daemonsets                  ds           apps/v1                        true         DaemonSet
   deployments                 deploy       apps/v1                        true         Deployment
   replicasets                 rs           apps/v1                        true         ReplicaSet
   statefulsets                sts          apps/v1                        true         StatefulSet
   localsubjectaccessreviews                authorization.k8s.io/v1        true         LocalSubjectAccessReview
   horizontalpodautoscalers    hpa          autoscaling/v1                 true         HorizontalPodAutoscaler
   cronjobs                    cj           batch/v1                       true         CronJob
   jobs                                     batch/v1                       true         Job
   leases                                   coordination.k8s.io/v1         true         Lease
   networkpolicies                          crd.projectcalico.org/v1       true         NetworkPolicy
   networksets                              crd.projectcalico.org/v1       true         NetworkSet
   endpointslices                           discovery.k8s.io/v1            true         EndpointSlice
   events                      ev           events.k8s.io/v1               true         Event
   ingresses                   ing          extensions/v1beta1             true         Ingress
   ingresses                   ing          networking.k8s.io/v1           true         Ingress
   networkpolicies             netpol       networking.k8s.io/v1           true         NetworkPolicy
   poddisruptionbudgets        pdb          policy/v1                      true         PodDisruptionBudget
   rolebindings                             rbac.authorization.k8s.io/v1   true         RoleBinding
   roles                                    rbac.authorization.k8s.io/v1   true         Role
   csistoragecapacities                     storage.k8s.io/v1beta1         true         CSIStorageCapacity
   [root@k8snode1 k8s]# kubectl api-resources --namespaced=false
   NAME                              SHORTNAMES   APIVERSION                             NAMESPACED   KIND
   componentstatuses                 cs           v1                                     false        ComponentStatus
   namespaces                        ns           v1                                     false        Namespace
   nodes                             no           v1                                     false        Node
   persistentvolumes                 pv           v1                                     false        PersistentVolume
   mutatingwebhookconfigurations                  admissionregistration.k8s.io/v1        false        MutatingWebhookConfiguration
   validatingwebhookconfigurations                admissionregistration.k8s.io/v1        false        ValidatingWebhookConfiguration
   customresourcedefinitions         crd,crds     apiextensions.k8s.io/v1                false        CustomResourceDefinition
   apiservices                                    apiregistration.k8s.io/v1              false        APIService
   tokenreviews                                   authentication.k8s.io/v1               false        TokenReview
   selfsubjectaccessreviews                       authorization.k8s.io/v1                false        SelfSubjectAccessReview
   selfsubjectrulesreviews                        authorization.k8s.io/v1                false        SelfSubjectRulesReview
   subjectaccessreviews                           authorization.k8s.io/v1                false        SubjectAccessReview
   certificatesigningrequests        csr          certificates.k8s.io/v1                 false        CertificateSigningRequest
   bgpconfigurations                              crd.projectcalico.org/v1               false        BGPConfiguration
   bgppeers                                       crd.projectcalico.org/v1               false        BGPPeer
   blockaffinities                                crd.projectcalico.org/v1               false        BlockAffinity
   clusterinformations                            crd.projectcalico.org/v1               false        ClusterInformation
   felixconfigurations                            crd.projectcalico.org/v1               false        FelixConfiguration
   globalnetworkpolicies                          crd.projectcalico.org/v1               false        GlobalNetworkPolicy
   globalnetworksets                              crd.projectcalico.org/v1               false        GlobalNetworkSet
   hostendpoints                                  crd.projectcalico.org/v1               false        HostEndpoint
   ipamblocks                                     crd.projectcalico.org/v1               false        IPAMBlock
   ipamconfigs                                    crd.projectcalico.org/v1               false        IPAMConfig
   ipamhandles                                    crd.projectcalico.org/v1               false        IPAMHandle
   ippools                                        crd.projectcalico.org/v1               false        IPPool
   kubecontrollersconfigurations                  crd.projectcalico.org/v1               false        KubeControllersConfiguration
   flowschemas                                    flowcontrol.apiserver.k8s.io/v1beta1   false        FlowSchema
   prioritylevelconfigurations                    flowcontrol.apiserver.k8s.io/v1beta1   false        PriorityLevelConfiguration
   ingressclasses                                 networking.k8s.io/v1                   false        IngressClass
   runtimeclasses                                 node.k8s.io/v1                         false        RuntimeClass
   imagesets                                      operator.tigera.io/v1                  false        ImageSet
   installations                                  operator.tigera.io/v1                  false        Installation
   tigerastatuses                                 operator.tigera.io/v1                  false        TigeraStatus
   podsecuritypolicies               psp          policy/v1beta1                         false        PodSecurityPolicy
   clusterrolebindings                            rbac.authorization.k8s.io/v1           false        ClusterRoleBinding
   clusterroles                                   rbac.authorization.k8s.io/v1           false        ClusterRole
   priorityclasses                   pc           scheduling.k8s.io/v1                   false        PriorityClass
   csidrivers                                     storage.k8s.io/v1                      false        CSIDriver
   csinodes                                       storage.k8s.io/v1                      false        CSINode
   storageclasses                    sc           storage.k8s.io/v1                      false        StorageClass
   volumeattachments                              storage.k8s.io/v1                      false        VolumeAttachment
   
   ```

4. 创建一个namespace

   ```
   #同样 根据实例看看
   [root@k8snode1 k8s]# kubectl create namespace ndemo --dry-run -o yaml
   W0419 22:50:23.954189   23515 helpers.go:557] --dry-run is deprecated and can be replaced with --dry-run=client.
   apiVersion: v1
   kind: Namespace
   metadata:
     creationTimestamp: null
     name: ndemo
   spec: {}
   status: {}
   
   apiVersion: v1
   kind: Namespace
   metadata:
     creationTimestamp: null
     name: ndemo  #namespace的名字
   spec: {}
   status: {}
   
   #提交一个命名空间
   [root@k8snode1 k8s]# kubectl apply -f namespace-demo.yaml 
   namespace/ndemo created
   
   [root@k8snode1 k8s]# kubectl get namespaces 
   NAME                   STATUS   AGE
   default                Active   2d
   kube-node-lease        Active   2d
   kube-public            Active   2d
   kube-system            Active   2d
   kubernetes-dashboard   Active   5h50m
   ndemo                  Active   13s
   tigera-operator        Active   47h
   
   #查看这个命名空间 会自动加上一些需要的字段
   [root@k8snode1 k8s]# kubectl get namespaces ndemo -o yaml
   apiVersion: v1
   kind: Namespace
   metadata:
     annotations:
       kubectl.kubernetes.io/last-applied-configuration: |
         {"apiVersion":"v1","kind":"Namespace","metadata":{"annotations":{},"creationTimestamp":null,"name":"ndemo"},"spec":{},"status":{}}
     creationTimestamp: "2021-04-19T14:54:08Z"
     labels:
       kubernetes.io/metadata.name: ndemo
     name: ndemo
     resourceVersion: "401026"
     uid: 01c62197-fd04-4d28-b7db-bcc34bf3c410
   spec:
     finalizers:
     - kubernetes
   status:
     phase: Active
   
   ```

5. 创建一个新的资源指定新的命名空间

   ```
   [root@k8snode1 k8s]# kubectl apply -f yamltest.yaml --namespace=ndemo
   deployment.apps/yamltest created
   
   #可以在这个命名空间下看到这个资源
   [root@k8snode1 k8s]# kubectl get deployments.apps -n ndemo
   NAME       READY   UP-TO-DATE   AVAILABLE   AGE
   yamltest   1/1     1            1           20s
   
   #删除
   [root@k8snode1 k8s]# kubectl delete -f yamltest.yaml --namespace=ndemo
   deployment.apps "yamltest" deleted
   [root@k8snode1 k8s]# kubectl get deployments.apps -n ndemo
   No resources found in ndemo namespace.
   ```

6. 创建一个新的资源指定新的命名空间，在资源中明确指定命名空间

   ```
   [root@k8snode1 k8s]# vim yamltestnamespace.yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     creationTimestamp: null
     labels:  #这个标签是Deployment的标签
       app: yamltest
     name: yamltest
     namespace: ndemo #指定命名空间
   spec:
     replicas: 1
     selector:  #pod的标签选择器
       matchLabels:
         app: yamltest
     strategy: {}
     template:
       metadata:
         creationTimestamp: null
         labels:  #这个标签是pod的标签
           app: yamltest
       spec:
         containers:
         - image: nginx:1.7.9
           name: nginx
           resources: {}
   status: {}
   
   [root@k8snode1 k8s]# kubectl apply -f yamltestnamespace.yaml 
   deployment.apps/yamltest created
   [root@k8snode1 k8s]# kubectl get deployments.apps -n ndemo
   NAME       READY   UP-TO-DATE   AVAILABLE   AGE
   yamltest   1/1     1            1           5s
   ```

# lablels，标签和标签选择器

参考文档：https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/labels/

*标签（Labels）* 是附加到 Kubernetes 对象（比如 Pods）上的键值对。 标签旨在用于指定对用户有意义且相关的对象的标识属性，但不直接对核心系统有语义含义。 标签可以用于组织和选择对象的子集。标签可以在创建时附加到对象，随后可以随时添加和修改。 每个对象都可以定义一组键/值标签。每个键对于给定对象必须是唯一的。

主要是为了解决多个资源为一个集群的标识，可以指定是哪个控制器创建的

1. 获取所有同一标签下的资源

   ```
   [root@k8snode1 k8s]# kubectl get pods -n kube-system -l k8s-app=calico-node
   NAME                READY   STATUS    RESTARTS   AGE
   calico-node-249tw   1/1     Running   0          47h
   calico-node-8rxs5   1/1     Running   1          47h
   calico-node-zs4hx   1/1     Running   1          47h
   
   ```

2. 在services里面就是通过标签来选择符合条件的pod的，services有个标签选择器，就是选择符合条件的pod的

   ```
   [root@k8snode1 k8s]# kubectl get services nginxdemo1 -o yaml
   apiVersion: v1
   kind: Service
   metadata:
     creationTimestamp: "2021-04-19T06:47:17Z"
     labels:
       app: nginxdemo1
     name: nginxdemo1
     namespace: default
     resourceVersion: "344933"
     uid: 998cda3e-0c0f-44b3-b12d-508f9a8038b9
   spec:
     clusterIP: 10.105.52.111
     clusterIPs:
     - 10.105.52.111
     externalTrafficPolicy: Cluster
     ipFamilies:
     - IPv4
     ipFamilyPolicy: SingleStack
     ports:
     - nodePort: 31204
       port: 80
       protocol: TCP
       targetPort: 80 
     selector:   #这个就是标签选择器
       app: nginxdemo1
     sessionAffinity: None
     type: NodePort
   status:
     loadBalancer: {}
   
   #通过标签选择器把pod筛选出来后就可以构建endpoints
   [root@k8snode1 k8s]# kubectl get endpoints
   NAME         ENDPOINTS                                                           AGE
   kubernetes   192.168.10.185:6443                                                 2d
   nginxdemo1   172.16.185.208:80,172.16.185.209:80,172.16.185.211:80 + 1 more...   8h
   
   ```

3. 筛选这个标签下所有的pod资源

   ```
   [root@k8snode1 k8s]# kubectl get pods -l app=nginxdemo1
   NAME                          READY   STATUS        RESTARTS   AGE
   nginxdemo1-6995675b4b-gtbhh   1/1     Running       0          37m
   nginxdemo1-6995675b4b-jbfrt   1/1     Running       1          7h49m
   nginxdemo1-6995675b4b-pnjnp   1/1     Running       1          7h49m
   nginxdemo1-6995675b4b-rsc4s   1/1     Terminating   0          7h49m
   nginxdemo1-6995675b4b-z7q72   1/1     Running       1          7h49m
   ```

4. 定义标签的时候尽可能定义一个有标识的名称，还可以支持基于集合需求的资源

5. api使用

   ```
   [root@k8snode1 k8s]# vim yamltestnamespace.yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     creationTimestamp: null
     labels:  #这个标签是Deployment的标签
       app: yamltest
     name: yamltest
     namespace: ndemo #指定命名空间
   spec:
     replicas: 1
     selector:  #pod的标签选择器
       matchLabels:
         app: yamltest
     strategy: {}
     template:
       metadata:
         creationTimestamp: null
         labels:  #这个标签是pod的标签
           app: yamltest
       spec:
         containers:
         - image: nginx:1.7.9
           name: nginx
           resources: {}
   status: {}
   
   
   [root@k8snode1 k8s]# vim yamltestnamespace.yaml 
   
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     creationTimestamp: null
     labels:
       app: yamltest
       encironment: product  #额外添加两个标签
       version: 1.7.9  #额外添加两个标签
     name: yamltest
     namespace: ndemo
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: yamltest
         encironment: product
         version: 1.7.9
     strategy: {}
     template:
       metadata:
         creationTimestamp: null
         labels:
           app: yamltest
           encironment: product
           version: 1.7.9
       spec:
         containers:
         - image: nginx:1.7.9
           name: nginx
           resources: {}
   status: {}
   
   #先删除以前的
   [root@k8snode1 k8s]# kubectl get deployments.apps -n ndemo
   NAME       READY   UP-TO-DATE   AVAILABLE   AGE
   yamltest   1/1     1            1           22m
   [root@k8snode1 k8s]# kubectl delete deployments.apps yamltest -n ndemo
   deployment.apps "yamltest" deleted
   
   #提交这个资源
   [root@k8snode1 k8s]# kubectl apply -f yamltestnamespace.yaml 
   deployment.apps/yamltest created
   
   #查看这个资源的标签
   [root@k8snode1 k8s]# kubectl get deployments.apps --show-labels -n ndemo
   NAME       READY   UP-TO-DATE   AVAILABLE   AGE   LABELS
   yamltest   1/1     1            1           67s   app=yamltest,encironment=product,version=1.7.9
   
   [root@k8snode1 k8s]# kubectl get pods --show-labels -n ndemo
   NAME                        READY   STATUS    RESTARTS   AGE   LABELS
   yamltest-5cdd97b5bd-tg4m7   1/1     Running   0          89s   app=yamltest,encironment=product,pod-template-hash=5cdd97b5bd,version=1.7.9
   
   #这样就可以通过条件来过滤了
   [root@k8snode1 k8s]# kubectl get pods --show-labels -n ndemo -l version=1.7.9 --show-labels 
   NAME                        READY   STATUS    RESTARTS   AGE     LABELS
   yamltest-5cdd97b5bd-tg4m7   1/1     Running   0          2m38s   app=yamltest,encironment=product,pod-template-hash=5cdd97b5bd,version=1.7.9
   
   ```

# Annotations 

注解：你可以使用 Kubernetes 注解为对象附加任意的非标识的元数据。客户端程序（例如工具和库）能够获取这些元数据信息。

为对象附加元数据：你可以使用标签或注解将元数据附加到 Kubernetes 对象。 标签可以用来选择对象和查找满足某些条件的对象集合。 相反，注解不用于标识和选择对象。 注解中的元数据，可以很小，也可以很大，可以是结构化的，也可以是非结构化的，能够包含标签不允许的字符。

可以放一些临时数据，在k8s中没有定义的字段，可以放到这里,

*注解（Annotations）* 存储的形式是键/值对。有效的注解键分为两部分： 可选的前缀和名称，以斜杠（`/`）分隔。 名称段是必需项，并且必须在63个字符以内，以字母数字字符（`[a-z0-9A-Z]`）开头和结尾， 并允许使用破折号（`-`），下划线（`_`），点（`.`）和字母数字。 前缀是可选的。如果指定，则前缀必须是DNS子域：一系列由点（`.`）分隔的DNS标签， 总计不超过253个字符，后跟斜杠（`/`）。 如果省略前缀，则假定注解键对用户是私有的。 由系统组件添加的注解 （例如，`kube-scheduler`，`kube-controller-manager`，`kube-apiserver`，`kubectl` 或其他第三方组件），必须为终端用户添加注解前缀。

`kubernetes.io/` 和 `k8s.io/` 前缀是为Kubernetes核心组件保留的。

例如，我们查看node1节点的注解

```
[root@k8snode1 k8s]# kubectl get nodes k8snode1 -o yaml
apiVersion: v1
kind: Node
metadata:
  annotations: #注解k8s就可以自己调用这些
    kubeadm.alpha.kubernetes.io/cri-socket: /var/run/dockershim.sock
    node.alpha.kubernetes.io/ttl: "0"
    projectcalico.org/IPv4Address: 192.168.10.185/24
    projectcalico.org/IPv4IPIPTunnelAddr: 172.16.249.0
    volumes.kubernetes.io/controller-managed-attach-detach: "true"
  creationTimestamp: "2021-04-17T14:50:23Z"
  labels:
    beta.kubernetes.io/arch: amd64
    beta.kubernetes.io/os: linux
    kubernetes.io/arch: amd64
    kubernetes.io/hostname: k8snode1
    kubernetes.io/os: linux
    node-role.kubernetes.io/control-plane: ""
    node-role.kubernetes.io/master: ""
    node.kubernetes.io/exclude-from-external-load-balancers: ""
  name: k8snode1
  resourceVersion: "403589"
  uid: b8ec4ebc-8d33-48b3-9121-1c4afb504c10
spec:
  podCIDR: 172.16.0.0/24
  podCIDRs:
  - 172.16.0.0/24
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
status:
  addresses:
  - address: 192.168.10.185
    type: InternalIP
  - address: k8snode1
    type: Hostname
  allocatable:
    cpu: "4"
    ephemeral-storage: "72928906325"
    hugepages-1Gi: "0"
    hugepages-2Mi: "0"
    memory: 3777200Ki
    pods: "110"
  capacity:
    cpu: "4"
    ephemeral-storage: 79132928Ki
    hugepages-1Gi: "0"
    hugepages-2Mi: "0"
    memory: 3879600Ki
    pods: "110"
  conditions:
  - lastHeartbeatTime: "2021-04-19T14:31:30Z"
    lastTransitionTime: "2021-04-19T14:31:30Z"
    message: Calico is running on this node
    reason: CalicoIsUp
    status: "False"
    type: NetworkUnavailable
  - lastHeartbeatTime: "2021-04-19T15:25:39Z"
    lastTransitionTime: "2021-04-17T14:50:22Z"
    message: kubelet has sufficient memory available
    reason: KubeletHasSufficientMemory
    status: "False"
    type: MemoryPressure
  - lastHeartbeatTime: "2021-04-19T15:25:39Z"
    lastTransitionTime: "2021-04-17T14:50:22Z"
    message: kubelet has no disk pressure
    reason: KubeletHasNoDiskPressure
    status: "False"
    type: DiskPressure
  - lastHeartbeatTime: "2021-04-19T15:25:39Z"
    lastTransitionTime: "2021-04-17T14:50:22Z"
    message: kubelet has sufficient PID available
    reason: KubeletHasSufficientPID
    status: "False"
    type: PIDPressure
  - lastHeartbeatTime: "2021-04-19T15:25:39Z"
    lastTransitionTime: "2021-04-18T10:23:11Z"
    message: kubelet is posting ready status
    reason: KubeletReady
    status: "True"
    type: Ready
  daemonEndpoints:
    kubeletEndpoint:
      Port: 10250
  images:
  - names:
    - k8s.gcr.io/etcd:3.4.13-0
    sizeBytes: 253392289
  - names:
    - kubernetesui/dashboard@sha256:06868692fb9a7f2ede1a06de1b7b32afabc40ec739c1181d83b5ed3eb147ec6e
    - kubernetesui/dashboard:v2.0.0
    sizeBytes: 221895031
  - names:
    - calico/node@sha256:dea9d024b9df20e1d4aa0624ff3a89b7f5e04116b7dafd9556c059ac3353282b
    - calico/node:v3.18.1
    sizeBytes: 172054747
  - names:
    - calico/cni@sha256:ed3ee0da61036fd0984442624a85ede293c71792d26072b4e813ed5460f7bbaf
    - calico/cni:v3.18.1
    sizeBytes: 131137949
  - names:
    - k8s.gcr.io/kube-apiserver:v1.21.0
    sizeBytes: 125591921
  - names:
    - k8s.gcr.io/kube-proxy:v1.21.0
    sizeBytes: 122238196
  - names:
    - k8s.gcr.io/kube-controller-manager:v1.21.0
    sizeBytes: 119808868
  - names:
    - calico/kube-controllers@sha256:bd19b85801762a1634e6b5c569e7c8fd528be352b31777ae047b99ed98b7da7f
    - calico/kube-controllers:v3.18.1
    sizeBytes: 53391108
  - names:
    - k8s.gcr.io/kube-scheduler:v1.21.0
    sizeBytes: 50631537
  - names:
    - k8s.gcr.io/coredns/coredns:v1.8.0
    sizeBytes: 42454755
  - names:
    - kubernetesui/metrics-scraper@sha256:555981a24f184420f3be0c79d4efb6c948a85cfce84034f85a563f4151a81cbf
    - kubernetesui/metrics-scraper:v1.0.4
    sizeBytes: 36937728
  - names:
    - calico/pod2daemon-flexvol@sha256:78022f886d60004084284c99d294f42f6bc7574fc1a7a2c7eafa8936c3a0bb7b
    - calico/pod2daemon-flexvol:v3.18.1
    sizeBytes: 21666928
  - names:
    - k8s.gcr.io/pause:3.4.1
    sizeBytes: 682696
  nodeInfo:
    architecture: amd64
    bootID: cb27a47e-d5cb-413e-abb5-cb4b3527bdcf
    containerRuntimeVersion: docker://19.3.9
    kernelVersion: 3.10.0-1160.24.1.el7.x86_64
    kubeProxyVersion: v1.21.0
    kubeletVersion: v1.21.0
    machineID: 7de771b3710f4ef69422414077898d01
    operatingSystem: linux
    osImage: CentOS Linux 7 (Core)
    systemUUID: 74084D56-DF1F-C9F9-3E4F-3112E1B5B56B


[root@k8snode1 k8s]# kubectl get deployments.apps -n kube-system coredns -o yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"  #可以看到这个资源的滚动更新的版本是1
  creationTimestamp: "2021-04-17T14:50:25Z"
  generation: 1
  labels:
    k8s-app: kube-dns
  name: coredns
  namespace: kube-system
  resourceVersion: "399660"
  uid: fc3a38b4-85a6-4a0f-b746-ec218264d4bf
spec:
  progressDeadlineSeconds: 600
  replicas: 2
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kube-dns
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        k8s-app: kube-dns
    spec:
      containers:
      - args:
        - -conf
        - /etc/coredns/Corefile
        image: k8s.gcr.io/coredns/coredns:v1.8.0  
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 5
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 5
        name: coredns
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /ready
            port: 8181
            scheme: HTTP
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          limits:
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - all
          readOnlyRootFilesystem: true
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /etc/coredns
          name: config-volume
          readOnly: true
      dnsPolicy: Default
      nodeSelector:
        kubernetes.io/os: linux
      priorityClassName: system-cluster-critical
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      serviceAccount: coredns
      serviceAccountName: coredns
      terminationGracePeriodSeconds: 30
      tolerations:
      - key: CriticalAddonsOnly
        operator: Exists
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
      - effect: NoSchedule
        key: node-role.kubernetes.io/control-plane
      volumes:
      - configMap:
          defaultMode: 420
          items:
          - key: Corefile
            path: Corefile
          name: coredns
        name: config-volume
status:
  availableReplicas: 2
  conditions:
  - lastTransitionTime: "2021-04-17T15:48:03Z"
    lastUpdateTime: "2021-04-17T15:48:03Z"
    message: ReplicaSet "coredns-558bd4d5db" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  - lastTransitionTime: "2021-04-19T14:37:18Z"
    lastUpdateTime: "2021-04-19T14:37:18Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  observedGeneration: 1
  readyReplicas: 2
  replicas: 2
  updatedReplicas: 2

```

1. 自定义注解

   ```
   [root@k8snode1 k8s]# vim yamltestnamespace.yaml
   
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     creationTimestamp: null
     labels:
       app: yamltest
       encironment: product
       version: 1.7.9
     annotations:
       imageregistry: 192.168.10.185:80  #自定义注解 键值对自定义
       nginx.version: 1.7.9 #自定义注解 键值对自定义
     name: yamltest
     namespace: ndemo
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: yamltest
         encironment: product
         version: 1.7.9
     strategy: {}
     template:
       metadata:
         creationTimestamp: null
         labels:
           app: yamltest
           encironment: product
           version: 1.7.9
       spec:
         containers:
         - image: nginx:1.7.9
           name: nginx
           resources: {}
   status: {}
   
   #先删除以前的
   [root@k8snode1 k8s]# kubectl get deployments.apps -n ndemo
   NAME       READY   UP-TO-DATE   AVAILABLE   AGE
   yamltest   1/1     1            1           22m
   [root@k8snode1 k8s]# kubectl delete deployments.apps yamltest -n ndemo
   deployment.apps "yamltest" deleted
   
   #提交这个资源
   [root@k8snode1 k8s]# kubectl apply -f yamltestnamespace.yaml 
   deployment.apps/yamltest created
   
   [root@k8snode1 k8s]# kubectl get deployments.apps -n ndemo -o yaml
   apiVersion: v1
   items:
   - apiVersion: apps/v1
     kind: Deployment
     metadata:
       annotations:
         deployment.kubernetes.io/revision: "1"
         imageregistry: 192.168.10.185:80   #可以看到自定义注解
         kubectl.kubernetes.io/last-applied-configuration: |
           {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{"imageregistry":"192.168.10.185:80","nginx.version":"1.7.9"},"creationTimestamp":null,"labels":{"app":"yamltest","encironment":"product","version":"1.7.9"},"name":"yamltest","namespace":"ndemo"},"spec":{"replicas":1,"selector":{"matchLabels":{"app":"yamltest","encironment":"product","version":"1.7.9"}},"strategy":{},"template":{"metadata":{"creationTimestamp":null,"labels":{"app":"yamltest","encironment":"product","version":"1.7.9"}},"spec":{"containers":[{"image":"nginx:1.7.9","name":"nginx","resources":{}}]}}},"status":{}}
         nginx.version: 1.7.9
       creationTimestamp: "2021-04-19T15:36:12Z"
       generation: 1
       labels:
         app: yamltest
         encironment: product
         version: 1.7.9
       name: yamltest
       namespace: ndemo
       resourceVersion: "404451"
       uid: 0c5556df-e538-454d-850c-64c8ab8becd6
     spec:
       progressDeadlineSeconds: 600
       replicas: 1
       revisionHistoryLimit: 10
       selector:
         matchLabels:
           app: yamltest
           encironment: product
           version: 1.7.9
       strategy:
         rollingUpdate:
           maxSurge: 25%
           maxUnavailable: 25%
         type: RollingUpdate
       template:
         metadata:
           creationTimestamp: null
           labels:
             app: yamltest
             encironment: product
             version: 1.7.9
         spec:
           containers:
           - image: nginx:1.7.9
             imagePullPolicy: IfNotPresent
             name: nginx
             resources: {}
             terminationMessagePath: /dev/termination-log
             terminationMessagePolicy: File
           dnsPolicy: ClusterFirst
           restartPolicy: Always
           schedulerName: default-scheduler
           securityContext: {}
           terminationGracePeriodSeconds: 30
     status:
       availableReplicas: 1
       conditions:
       - lastTransitionTime: "2021-04-19T15:36:14Z"
         lastUpdateTime: "2021-04-19T15:36:14Z"
         message: Deployment has minimum availability.
         reason: MinimumReplicasAvailable
         status: "True"
         type: Available
       - lastTransitionTime: "2021-04-19T15:36:12Z"
         lastUpdateTime: "2021-04-19T15:36:14Z"
         message: ReplicaSet "yamltest-5cdd97b5bd" has successfully progressed.
         reason: NewReplicaSetAvailable
         status: "True"
         type: Progressing
       observedGeneration: 1
       readyReplicas: 1
       replicas: 1
       updatedReplicas: 1
   kind: List
   metadata:
     resourceVersion: ""
     selfLink: ""
   
   ```

# *Field selectors*

*字段选择器（Field selectors*）允许你根据一个或多个资源字段的值 [筛选 Kubernetes 资源](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/kubernetes-objects)。 下面是一些使用字段选择器查询的例子：

- `metadata.name=my-service`
- `metadata.namespace!=default`
- `status.phase=Pending`

1. 过滤podip为指定ip的资源,同样支持运算操作，比如取反 集合等

   ```
   [root@k8snode1 k8s]# kubectl get pods -n ndemo -o yaml
   apiVersion: v1
   items:
   - apiVersion: v1
     kind: Pod
     metadata:
       annotations:
         cni.projectcalico.org/podIP: 172.16.185.219/32
         cni.projectcalico.org/podIPs: 172.16.185.219/32
       creationTimestamp: "2021-04-19T15:36:12Z"
       generateName: yamltest-5cdd97b5bd-
       labels:
         app: yamltest
         encironment: product
         pod-template-hash: 5cdd97b5bd
         version: 1.7.9
       name: yamltest-5cdd97b5bd-ndqtz
       namespace: ndemo
       ownerReferences:
       - apiVersion: apps/v1
         blockOwnerDeletion: true
         controller: true
         kind: ReplicaSet
         name: yamltest-5cdd97b5bd
         uid: b43f0c05-9672-4c45-ab18-ec9e357f8821
       resourceVersion: "404449"
       uid: 73592950-3e32-4ba7-a478-9875f4c683b4
     spec:
       containers:
       - image: nginx:1.7.9
         imagePullPolicy: IfNotPresent
         name: nginx
         resources: {}
         terminationMessagePath: /dev/termination-log
         terminationMessagePolicy: File
         volumeMounts:
         - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
           name: kube-api-access-kp7gl
           readOnly: true
       dnsPolicy: ClusterFirst
       enableServiceLinks: true
       nodeName: k8snode2
       preemptionPolicy: PreemptLowerPriority
       priority: 0
       restartPolicy: Always
       schedulerName: default-scheduler
       securityContext: {}
       serviceAccount: default
       serviceAccountName: default
       terminationGracePeriodSeconds: 30
       tolerations:
       - effect: NoExecute
         key: node.kubernetes.io/not-ready
         operator: Exists
         tolerationSeconds: 300
       - effect: NoExecute
         key: node.kubernetes.io/unreachable
         operator: Exists
         tolerationSeconds: 300
       volumes:
       - name: kube-api-access-kp7gl
         projected:
           defaultMode: 420
           sources:
           - serviceAccountToken:
               expirationSeconds: 3607
               path: token
           - configMap:
               items:
               - key: ca.crt
                 path: ca.crt
               name: kube-root-ca.crt
           - downwardAPI:
               items:
               - fieldRef:
                   apiVersion: v1
                   fieldPath: metadata.namespace
                 path: namespace
     status:
       conditions:
       - lastProbeTime: null
         lastTransitionTime: "2021-04-19T15:36:12Z"
         status: "True"
         type: Initialized
       - lastProbeTime: null
         lastTransitionTime: "2021-04-19T15:36:14Z"
         status: "True"
         type: Ready
       - lastProbeTime: null
         lastTransitionTime: "2021-04-19T15:36:14Z"
         status: "True"
         type: ContainersReady
       - lastProbeTime: null
         lastTransitionTime: "2021-04-19T15:36:12Z"
         status: "True"
         type: PodScheduled
       containerStatuses:
       - containerID: docker://604e774a8b2cf18e49ca05b6b38eb407fd1a1fc0805a31cdac311dc45459a1b0
         image: nginx:1.7.9
         imageID: docker-pullable://nginx@sha256:e3456c851a152494c3e4ff5fcc26f240206abac0c9d794affb40e0714846c451
         lastState: {}
         name: nginx
         ready: true
         restartCount: 0
         started: true
         state:
           running:
             startedAt: "2021-04-19T15:36:13Z"
       hostIP: 192.168.10.186
       phase: Running
       podIP: 172.16.185.219
       podIPs:
       - ip: 172.16.185.219
       qosClass: BestEffort
       startTime: "2021-04-19T15:36:12Z"
   kind: List
   metadata:
     resourceVersion: ""
     selfLink: ""
   
   [root@k8snode1 k8s]# kubectl get pods --field-selector status.podIP=172.16.185.219 -n ndemo
   NAME                        READY   STATUS    RESTARTS   AGE
   yamltest-5cdd97b5bd-ndqtz   1/1     Running   0          7m12s
   
   [root@k8snode1 k8s]# kubectl get pods --field-selector status.phase=Running -n ndemo
   NAME                        READY   STATUS    RESTARTS   AGE
   yamltest-5cdd97b5bd-ndqtz   1/1     Running   0          7m58s
   
   ```

# 推荐使用的标签

参考资料：https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/common-labels/

为了充分利用这些标签，应该在每个资源对象上都使用它们。

| 键                             | 描述                                               | 示例           | 类型   |
| ------------------------------ | -------------------------------------------------- | -------------- | ------ |
| `app.kubernetes.io/name`       | 应用程序的名称                                     | `mysql`        | 字符串 |
| `app.kubernetes.io/instance`   | 用于唯一确定应用实例的名称                         | `mysql-abcxzy` | 字符串 |
| `app.kubernetes.io/version`    | 应用程序的当前版本（例如，语义版本，修订版哈希等） | `5.7.21`       | 字符串 |
| `app.kubernetes.io/component`  | 架构中的组件                                       | `database`     | 字符串 |
| `app.kubernetes.io/part-of`    | 此级别的更高级别应用程序的名称                     | `wordpress`    | 字符串 |
| `app.kubernetes.io/managed-by` | 用于管理应用程序的工具                             | `helm`         | 字符串 |



