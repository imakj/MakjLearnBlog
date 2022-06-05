# k8s架构分析

![image-20210414223611566](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/k8s%E6%9E%B6%E6%9E%84%E8%A7%A3%E6%9E%90.png?x-oss-process=style/makjwatermarks)

* kubernetes包含两种⻆⾊：master节点和node节点，master节点是集群的控制管理节点，作
  为整个k8s集群的调度负责
  * 负责集群所有接⼊请求(kube-apiserver)，在整个集群的⼊⼝；
  * 集群资源调度(kube-controller-scheduler)，通过watch监视pod的创建，负责将pod调度到合适的node节点；
  * 集群状态的⼀致性(kube-controller-manager)，通过多种控制器确保集群的⼀致性，包含有Node Controller，Replication Controller，Endpoints Controller等；
  * 元数据信息存储(etcd)，数据持久存储化，存储集群中包括node，pod，rc，service等数
    据；
* node节点是实际的⼯作节点，负责集群负载的实际运⾏，即pod运⾏的载体，其通常包含三个组件：Container Runtime，kubelet和kube-proxy
  * Container Runtime是容器运⾏时，负责实现container⽣命周期管理，如docker，containerd，rktlet；
  * kubelet负责镜像和pod的管理，比如创建容器
  * kube-proxy是service服务实现的抽象，负责维护和转发pod的路由，实现集群内部和外部⽹络的访问，就是负责容器的网络监控，一个反向代理的角色，通过iptables或者lvs进行负载均衡
* 其他组件
  * cloud-controller-manager，⽤于公有云的接⼊实现，提供节点管理(node)，路由管理，服务管理(LoadBalancer和Ingress)，存储管理(Volume，如云盘，NAS接⼊)，需要由公有云⼚商实现具体的细节，kubernetes提供实现接⼝的接⼊，如腾讯云⽬前提供CVM的node管理，节点的弹性伸缩(AS),负载均衡的接⼊(CLB),存储的管理(CBS和CFS)等产品
    的集成；
  * DNS组件由kube-dns或coredns实现集群内的名称解析；
  * kubernetes-dashboard⽤于图形界⾯管理；
  * kubectl命令⾏⼯具进⾏API交互；
  * 服务外部接⼊，通过ingress实现七层接⼊，由多种controller控制器组成
    * traefik
    * nginx ingress controller
    * haproxy ingress controller
    * 公有云⼚商ingress controller
  * 监控系统⽤于采集node和pod的监控数据
    * metric-server 核⼼指标监控
    * prometheus ⾃定义指标监控，提供丰富功能
    * heapster+influxdb+grafana 旧核⼼指标监控⽅案，现已废弃
  * ⽇志采集系统，⽤于收集容器的业务数据,实现⽇志的采集，存储和展示，由EFK实现
    * Fluentd ⽇志采集
    * ElasticSearch ⽇志存储+检索
    * Kiabana 数据展示

# 基础概念

1. kubernetes是⼀个开源的容器引擎管理平台，实现容器化应⽤的⾃动化部署，任务调度，弹性
   伸缩，负载均衡等功能，cluster是由master和node两种⻆⾊组成
   * master负责管理集群，master包含kube-apiserver，kube-controller-manager，kubescheduler，etcd组件
   * node节点运⾏容器应⽤，由Container Runtime，kubelet和kube-proxy组成，其中Container Runtime可能是Docker，rke，containerd，node节点可由物理机或者虚拟机组成。

```
# 查看node节点
[root@k8snode1 ~]# kubectl get nodes
NAME       STATUS   ROLES                  AGE   VERSION
k8snode1   Ready    control-plane,master   24h   v1.21.0
k8snode2   Ready    <none>                 24h   v1.21.0
k8snode3   Ready    <none>                 24h   v1.21.0

#查看node节点详情
[root@k8snode1 ~]# kubectl describe node k8snode2
Name:               k8snode2
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=k8snode2
                    kubernetes.io/os=linux
Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: /var/run/dockershim.sock
                    node.alpha.kubernetes.io/ttl: 0
                    projectcalico.org/IPv4Address: 192.168.10.186/24
                    projectcalico.org/IPv4IPIPTunnelAddr: 172.16.185.192
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Sat, 17 Apr 2021 23:00:20 +0800
Taints:             <none>
Unschedulable:      false #是否禁⽤调度，cordon命令控制的标识位。
Lease:
  HolderIdentity:  k8snode2
  AcquireTime:     <unset>
  RenewTime:       Sun, 18 Apr 2021 23:30:01 +0800
Conditions:	#资源调度能⼒，MemoryPressure内存是否有压⼒（即内存不⾜）
#DiskPressure磁盘压⼒
#PIDPressure磁盘压⼒
#Ready，是否就绪，表明节点是否处于正常⼯作状态，表示资源充⾜+相关进
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Sat, 17 Apr 2021 23:50:46 +0800   Sat, 17 Apr 2021 23:50:46 +0800   CalicoIsUp                   Calico is running
  MemoryPressure       False   Sun, 18 Apr 2021 23:29:11 +0800   Sat, 17 Apr 2021 23:00:19 +0800   KubeletHasSufficientMemory   kubelet has suffi
  DiskPressure         False   Sun, 18 Apr 2021 23:29:11 +0800   Sat, 17 Apr 2021 23:00:19 +0800   KubeletHasNoDiskPressure     kubelet has no di
  PIDPressure          False   Sun, 18 Apr 2021 23:29:11 +0800   Sat, 17 Apr 2021 23:00:19 +0800   KubeletHasSufficientPID      kubelet has suffi
  Ready                True    Sun, 18 Apr 2021 23:29:11 +0800   Sat, 17 Apr 2021 23:48:58 +0800   KubeletReady                 kubelet is postin
Addresses:  #地址和主机名
  InternalIP:  192.168.10.186
  Hostname:    k8snode2
Capacity:  #容器的资源容量
  cpu:                4
  ephemeral-storage:  79132928Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3879608Ki
  pods:               110
Allocatable:  #已分配资源情况
  cpu:                4
  ephemeral-storage:  72928906325
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3777208Ki
  pods:               110
System Info:  #系统信息，如内核版本，操作系统版本，cpu架构，node节点软件版本
  Machine ID:                 7de771b3710f4ef69422414077898d01
  System UUID:                45A14D56-32A8-5E58-3250-BB736358C9BE
  Boot ID:                    478d2b91-53ed-4748-9f20-45041dff76ae
  Kernel Version:             3.10.0-1160.15.2.el7.x86_64
  OS Image:                   CentOS Linux 7 (Core)
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  docker://19.3.9
  Kubelet Version:            v1.21.0
  Kube-Proxy Version:         v1.21.0
PodCIDR:                      172.16.1.0/24
PodCIDRs:                     172.16.1.0/24
Non-terminated Pods:          (2 in total)
  Namespace                   Name                 CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                 ------------  ----------  ---------------  -------------  ---
  kube-system                 calico-node-8rxs5    250m (6%)     0 (0%)      0 (0%)           0 (0%)         23h
  kube-system                 kube-proxy-r4xpx     0 (0%)        0 (0%)      0 (0%)           0 (0%)         24h
Allocated resources:  #已分配资源情况
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests   Limits
  --------           --------   ------
  cpu                250m (6%)  0 (0%)
  memory             0 (0%)     0 (0%)
  ephemeral-storage  0 (0%)     0 (0%)
  hugepages-1Gi      0 (0%)     0 (0%)
  hugepages-2Mi      0 (0%)     0 (0%)
Events:              <none>

```

2. kubernetes是容器编排引擎，其负责容器的调度，管理和容器的运⾏，但kubernetes调度最⼩单位并⾮是container，⽽是pod，pod中可包含多个container，通常集群中不会直接运⾏pod，⽽是通过各种⼯作负载的控制器如Deployments，ReplicaSets，DaemonSets的⽅式运⾏，为啥？因为控制器能够保证pod状态的⼀致性，正如官⽅所描述的⼀样“make sure thecurrent state match to the desire state”，确保当前状态和预期的⼀致，简单来说就是pod异常了，控制器会在其他节点重建，确保集群当前运⾏的pod和预期设定的⼀致。

   * pod是kubernetes中运⾏的最⼩单元

   * pod中包含⼀个容器或者多个容器，如果是多个容器，他们会共享一个网络命名空间或者存储命名空间，容器是封装在pod里面的

   * pod不会单独使⽤，需要有⼯作负载来控制，如Deployments，StatefulSets，DaemonSets，CronJobs等

   * 参考连接：https://kubernetes.io/docs/tutorials/kubernetes-basics/scale/scale-intro/

     ```
     # 查看所有pod
     [root@k8snode1 ~]# kubectl get pods -n kube-system
     NAME                                       READY   STATUS    RESTARTS   AGE
     calico-kube-controllers-6d8ccdbf46-s6qdx   1/1     Running   0          23h
     calico-node-249tw                          1/1     Running   0          23h
     calico-node-8rxs5                          1/1     Running   0          23h
     calico-node-zs4hx                          1/1     Running   0          23h
     coredns-558bd4d5db-56pr7                   1/1     Running   0          24h
     coredns-558bd4d5db-wbgr6                   1/1     Running   0          24h
     etcd-k8snode1                              1/1     Running   0          24h
     kube-apiserver-k8snode1                    1/1     Running   0          24h
     kube-controller-manager-k8snode1           1/1     Running   0          5h15m
     kube-proxy-9tvkf                           1/1     Running   0          24h
     kube-proxy-c4m82                           1/1     Running   0          24h
     kube-proxy-r4xpx                           1/1     Running   0          24h
     kube-scheduler-k8snode1                    1/1     Running   1          5h12m
     
     #查看某个容器的详细pod
     [root@k8snode1 ~]# kubectl describe pods calico-kube-controllers-6d8ccdbf46-s6qdx -n kube-system
     Name:                 calico-kube-controllers-6d8ccdbf46-s6qdx
     Namespace:            kube-system
     Priority:             2000000000
     Priority Class Name:  system-cluster-critical
     Node:                 k8snode3/192.168.10.187
     Start Time:           Sat, 17 Apr 2021 23:46:47 +0800
     Labels:               k8s-app=calico-kube-controllers
                           pod-template-hash=6d8ccdbf46
     Annotations:          cni.projectcalico.org/podIP: 172.16.98.194/32
                           cni.projectcalico.org/podIPs: 172.16.98.194/32
     Status:               Running
     IP:                   172.16.98.194
     IPs:
       IP:           172.16.98.194
     Controlled By:  ReplicaSet/calico-kube-controllers-6d8ccdbf46
     Containers:
       calico-kube-controllers:
         Container ID:   docker://93ec5f12c143e8bdf3c474c2346a1cffc7f8a3d0fe1fc8ccc1d4307c03e00f0c
         Image:          docker.io/calico/kube-controllers:v3.18.1
         Image ID:       docker-pullable://calico/kube-controllers@sha256:bd19b85801762a1634e6b5c569e7c8fd528be352b31777ae047b99ed98b7da7f
         Port:           <none>
         Host Port:      <none>
         State:          Running
           Started:      Sat, 17 Apr 2021 23:48:35 +0800
         Ready:          True
         Restart Count:  0
         Readiness:      exec [/usr/bin/check-status -r] delay=0s timeout=1s period=10s #success=1 #failure=3
         Environment:
           ENABLED_CONTROLLERS:  node
           DATASTORE_TYPE:       kubernetes
         Mounts:
           /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-fg9dw (ro)
     Conditions:
       Type              Status
       Initialized       True 
       Ready             True 
       ContainersReady   True 
       PodScheduled      True 
     Volumes:
       kube-api-access-fg9dw:
         Type:                    Projected (a volume that contains injected data from multiple sources)
         TokenExpirationSeconds:  3607
         ConfigMapName:           kube-root-ca.crt
         ConfigMapOptional:       <nil>
         DownwardAPI:             true
     QoS Class:                   BestEffort
     Node-Selectors:              kubernetes.io/os=linux
     Tolerations:                 CriticalAddonsOnly op=Exists
                                  node-role.kubernetes.io/master:NoSchedule
                                  node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                                  node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
     Events:                      <none>
     
     ```

     

   * Container，容器是⼀种轻量化的虚拟化技术，通过将应⽤封装在镜像中，实现便捷部署，应⽤分发。

     ```
     [root@k8snode1 ~]# docker container list
     CONTAINER ID        IMAGE                    COMMAND                  CREATED             STATUS              PORTS               NAMES
     02be69aa0683        62ad3129eca8             "kube-scheduler --au…"   5 hours ago         Up 5 hours                              k8s_kube-scheduler_kube-scheduler-k8snode1_kube-system_3e66fad3dc9a0828ca0c99e59ab39f10_1
     eadb24902e34        k8s.gcr.io/pause:3.4.1   "/pause"                 5 hours ago         Up 5 hours                              k8s_POD_kube-scheduler-k8snode1_kube-system_3e66fad3dc9a0828ca0c99e59ab39f10_1
     85086d6bf7d6        09708983cc37             "kube-controller-man…"   5 hours ago         Up 5 hours                              k8s_kube-controller-manager_kube-controller-manager-k8snode1_kube-system_b7ffb44249d2e29621abd5e1c402371c_0
     9f7c87693222        k8s.gcr.io/pause:3.4.1   "/pause"                 5 hours ago         Up 5 hours                              k8s_POD_kube-controller-manager-k8snode1_kube-system_b7ffb44249d2e29621abd5e1c402371c_0
     de1713f03617        calico/node              "start_runit"            24 hours ago        Up 24 hours                             k8s_calico-node_calico-node-zs4hx_kube-system_0b193931-f23d-4627-916f-ccf0d311c03d_0
     e16bd1e7bdf6        k8s.gcr.io/pause:3.4.1   "/pause"                 24 hours ago        Up 24 hours                             k8s_POD_calico-node-zs4hx_kube-system_0b193931-f23d-4627-916f-ccf0d311c03d_0
     dfeb2c5173bb        38ddd85fe90e             "/usr/local/bin/kube…"   25 hours ago        Up 25 hours                             k8s_kube-proxy_kube-proxy-c4m82_kube-system_1428a2f0-4f90-4619-82b2-484ad5b4a7c2_0
     38af914df1ae        k8s.gcr.io/pause:3.4.1   "/pause"                 25 hours ago        Up 25 hours                             k8s_POD_kube-proxy-c4m82_kube-system_1428a2f0-4f90-4619-82b2-484ad5b4a7c2_0
     69444092c2ee        4d217480042e             "kube-apiserver --ad…"   25 hours ago        Up 25 hours                             k8s_kube-apiserver_kube-apiserver-k8snode1_kube-system_45b9c9571b561573a462cbbb349ad36e_0
     797899d7a2eb        k8s.gcr.io/pause:3.4.1   "/pause"                 25 hours ago        Up 25 hours                             k8s_POD_kube-apiserver-k8snode1_kube-system_45b9c9571b561573a462cbbb349ad36e_0
     98f6a023eee9        0369cf4303ff             "etcd --advertise-cl…"   25 hours ago        Up 25 hours                             k8s_etcd_etcd-k8snode1_kube-system_5d52c7d516d5438ca27bbb218374c99c_0
     f280af565b21        k8s.gcr.io/pause:3.4.1   "/pause"                 25 hours ago        Up 25 hours                             k8s_POD_etcd-k8snode1_kube-system_5d52c7d516d5438ca27bbb218374c99c_0
     
     ```

     

   * Pod，kubernetes中最⼩的调度单位，封装容器，包含⼀个pause容器和应⽤容器，容器之间共享相同的命名空间，⽹络，存储，共享进程

   * Deployments，部署组也称应⽤，也就是自己的应用，严格上来说是⽆状态化⼯作负载，另外⼀种由状态化⼯组负载是StatefulSets，Deployments是⼀种控制器，可以控制⼯作负载的副本数replicas，通过kube-controller-manager中的Deployments Controller实现副本数状态的控制。最终应用会落到某个node节点上

     ```
     # 创建一个应用 老版本
     [root@k8snode1 ~]# kubectl run nginxdemo --image=nginx:1.7.9 --port=80 
     pod/nginxdemo created
     
     --image：镜像名称
     --port：端口
     
     # 创建一个应用 新版本
     [root@k8snode1 ~]# kubectl create deployment nginxdemo1 --image=nginx:1.7.9 --port=80 
     kubectl create deployment：创建应用
     
     #查看所有deployments应用
     [root@k8snode1 ~]# kubectl get deployments.apps 
     NAME         READY   UP-TO-DATE   AVAILABLE   AGE
     nginxdemo1   1/1     1            1           35s
     
     #查看deployments详情
     [root@k8snode1 ~]# kubectl describe deployments.apps nginxdemo1
     Name:                   nginxdemo1
     Namespace:              default
     CreationTimestamp:      Sun, 18 Apr 2021 23:42:17 +0800
     Labels:                 app=nginxdemo1
     Annotations:            deployment.kubernetes.io/revision: 1
     Selector:               app=nginxdemo1
     Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
     StrategyType:           RollingUpdate
     MinReadySeconds:        0
     RollingUpdateStrategy:  25% max unavailable, 25% max surge
     Pod Template:
       Labels:  app=nginxdemo1
       Containers:
        nginx:
         Image:        nginx:1.7.9  #使用的哪个镜像
         Port:         80/TCP  #用的哪个端口
         Host Port:    0/TCP
         Environment:  <none>
         Mounts:       <none>
       Volumes:        <none>
     Conditions:
       Type           Status  Reason
       ----           ------  ------
       Available      True    MinimumReplicasAvailable
       Progressing    True    NewReplicaSetAvailable
     OldReplicaSets:  <none>
     NewReplicaSet:   nginxdemo1-6995675b4b (1/1 replicas created) #副本集，用的那几个副本集合，是通过ReplicaSet:-conntroller创建的
     Events:          <none>
     
     
     #下面查看这个副本集 这个replicasets就是来管理副本的复制的
     [root@k8snode1 ~]# kubectl get replicasets.apps 
     NAME                    DESIRED   CURRENT   READY   AGE
     nginxdemo1-6995675b4b   1         1         1       14h
     
     #查看详细
     [root@k8snode1 ~]# kubectl describe replicasets.apps nginxdemo1-6995675b4b
     Name:           nginxdemo1-6995675b4b
     Namespace:      default
     Selector:       app=nginxdemo1,pod-template-hash=6995675b4b
     Labels:         app=nginxdemo1
                     pod-template-hash=6995675b4b
     Annotations:    deployment.kubernetes.io/desired-replicas: 1
                     deployment.kubernetes.io/max-replicas: 2
                     deployment.kubernetes.io/revision: 1
     Controlled By:  Deployment/nginxdemo1
     Replicas:       1 current / 1 desired
     Pods Status:    1 Running / 0 Waiting / 0 Succeeded / 0 Failed
     Pod Template:
       Labels:  app=nginxdemo1
                pod-template-hash=6995675b4b
       Containers:
        nginx:
         Image:        nginx:1.7.9
         Port:         80/TCP
         Host Port:    0/TCP
         Environment:  <none>
         Mounts:       <none>
       Volumes:        <none>
     Events:           <none>
     
     #查看所有容器和详情
     [root@k8snode1 ~]# kubectl get pod
     NAME                          READY   STATUS    RESTARTS   AGE
     nginxdemo                     1/1     Running   0          14h
     nginxdemo1-6995675b4b-9kqbb   1/1     Running   0          14h
     [root@k8snode1 ~]# kubectl describe pods nginxdemo1-6995675b4b-9kqbb
     Name:         nginxdemo1-6995675b4b-9kqbb
     Namespace:    default
     Priority:     0
     Node:         k8snode2/192.168.10.186
     Start Time:   Sun, 18 Apr 2021 23:42:17 +0800
     Labels:       app=nginxdemo1
                   pod-template-hash=6995675b4b
     Annotations:  cni.projectcalico.org/podIP: 172.16.185.194/32
                   cni.projectcalico.org/podIPs: 172.16.185.194/32
     Status:       Running
     IP:           172.16.185.194
     IPs:
       IP:           172.16.185.194
     Controlled By:  ReplicaSet/nginxdemo1-6995675b4b
     Containers:
       nginx:
         Container ID:   docker://963c091a56d9b3734232a0ab5762c95846095d238c4841a6b3bfa06fcf1d4102
         Image:          nginx:1.7.9
         Image ID:       docker-pullable://nginx@sha256:e3456c851a152494c3e4ff5fcc26f240206abac0c9d794affb40e0714846c451
         Port:           80/TCP
         Host Port:      0/TCP
         State:          Running
           Started:      Sun, 18 Apr 2021 23:42:18 +0800
         Ready:          True
         Restart Count:  0
         Environment:    <none>
         Mounts:
           /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-lg2d9 (ro)
     Conditions:
       Type              Status
       Initialized       True 
       Ready             True 
       ContainersReady   True 
       PodScheduled      True 
     Volumes:
       kube-api-access-lg2d9:
         Type:                    Projected (a volume that contains injected data from multiple sources)
         TokenExpirationSeconds:  3607
         ConfigMapName:           kube-root-ca.crt
         ConfigMapOptional:       <nil>
         DownwardAPI:             true
     QoS Class:                   BestEffort
     Node-Selectors:              <none>
     Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                                  node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
     Events:                      <none>
     
     
     #查看所有容器
     [root@k8snode1 ~]# kubectl get pods -o wide
     NAME                          READY   STATUS    RESTARTS   AGE   IP               NODE       NOMINATED NODE   READINESS GATES
     nginxdemo                     1/1     Running   0          14h   172.16.185.193   k8snode2   <none>           <none>
     nginxdemo1-6995675b4b-9kqbb   1/1     Running 
     
     #访问下容器
     [root@k8snode1 ~]# curl 172.16.185.194
     
     #查看容器日志
     [root@k8snode1 ~]# kubectl logs nginxdemo1-6995675b4b-9kqbb
     172.16.249.0 - - [19/Apr/2021:06:33:42 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"
     
     #进入到容器
     [root@k8snode1 ~]# kubectl exec -it nginxdemo1-6995675b4b-9kqbb /bin/bash
     kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
     root@nginxdemo1-6995675b4b-9kqbb:/# uname -r
     3.10.0-1160.15.2.el7.x86_64
     root@nginxdemo1-6995675b4b-9kqbb:/# 
     
     
     #扩展Deployments容器
     [root@k8snode1 ~]# kubectl get deployments.apps 
     NAME         READY   UP-TO-DATE   AVAILABLE   AGE
     nginxdemo1   1/1     1            1           14h
     [root@k8snode1 ~]# kubectl scale deployment --replicas=4 nginxdemo1
     deployment.apps/nginxdemo1 scaled
     #查看是否扩容成功
     [root@k8snode1 ~]# kubectl get deployments.apps 
     NAME         READY   UP-TO-DATE   AVAILABLE   AGE
     nginxdemo1   4/4     4            4            14h
     
     
     [root@k8snode1 ~]# kubectl get pods
     NAME                          READY   STATUS    RESTARTS   AGE
     nginxdemo                     1/1     Running   0          14h
     nginxdemo1-6995675b4b-6jhdt   1/1     Running   0          43s
     nginxdemo1-6995675b4b-9kqbb   1/1     Running   0          14h
     nginxdemo1-6995675b4b-bz22l   1/1     Running   0          43s
     nginxdemo1-6995675b4b-m9snz   1/1     Running   0          43s
     
     
     ```

   * services服务暴露，相当于一个负载均衡器,讲容器的服务器暴露给外部提供访问，在所有的节点内部ip前面加一个vip进行负载均衡，因为节点ip是会变化的，vip不会变，就实现了负载均衡

     每个pod都会有个标签。负载均衡器会根据标签来选择标签

     ```
     [root@k8snode1 ~]# kubectl get pods -o wide
     NAME                          READY   STATUS    RESTARTS   AGE   IP               NODE       NOMINATED NODE   READINESS GATES
     nginxdemo                     1/1     Running   0          14h   172.16.185.193   k8snode2   <none>           <none>
     nginxdemo1-6995675b4b-6jhdt   1/1     Running   0          98s   172.16.185.195   k8snode2   <none>           <none>
     nginxdemo1-6995675b4b-9kqbb   1/1     Running   0          14h   172.16.185.194   k8snode2   <none>           <none>
     nginxdemo1-6995675b4b-bz22l   1/1     Running   0          98s   172.16.185.196   k8snode2   <none>           <none>
     nginxdemo1-6995675b4b-m9snz   1/1     Running   0          98s   172.16.98.196    k8snode3   <none>           <none>
     
     #查看每个容器的标签 容器标签是不会变化的 通过LABELS讲满足条件的标签添加到一个负载均衡器
     [root@k8snode1 ~]# kubectl get pods --show-labels
     NAME                          READY   STATUS    RESTARTS   AGE     LABELS
     nginxdemo                     1/1     Running   0          15h     run=nginxdemo
     nginxdemo1-6995675b4b-6jhdt   1/1     Running   0          5m38s   app=nginxdemo1,pod-template-hash=6995675b4b
     nginxdemo1-6995675b4b-9kqbb   1/1     Running   0          15h     app=nginxdemo1,pod-template-hash=6995675b4b
     nginxdemo1-6995675b4b-bz22l   1/1     Running   0          5m38s   app=nginxdemo1,pod-template-hash=6995675b4b
     nginxdemo1-6995675b4b-m9snz   1/1     Running   0          5m38s   app=nginxdemo1,pod-template-hash=6995675b4b
     
     #创建service并且导出nginxdemo1的服务器对外为TCP协议，端口为80 ip为内部集群ip ，ClusterIP：只能在k8s内部集群访问
     [root@k8snode1 ~]# kubectl expose deployment nginxdemo1 --port=80 --protocol=TCP --type=ClusterIP
     service/nginxdemo1 exposed
     
     #查看service
     [root@k8snode1 ~]# kubectl get service
     NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
     kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   39h
     nginxdemo1   ClusterIP   10.105.52.111   <none>        80/TCP    28s
     
     #查看service详情
     [root@k8snode1 ~]# kubectl describe services nginxdemo1
     Name:              nginxdemo1
     Namespace:         default
     Labels:            app=nginxdemo1
     Annotations:       <none>
     Selector:          app=nginxdemo1
     Type:              ClusterIP
     IP Family Policy:  SingleStack
     IP Families:       IPv4
     IP:                
     IPs:               10.105.52.111
     Port:              <unset>  80/TCP
     TargetPort:        80/TCP
     #可以看到下面所有的nginx的pod的podip被锁定了
     Endpoints:         172.16.185.194:80,172.16.185.195:80,172.16.185.196:80 + 1 more...
     Session Affinity:  None
     Events:            <none>
     
     #直接访问集群ip 就访问到了应用
     [root@k8snode1 ~]# curl http://10.105.52.111
     <!DOCTYPE html>
     <html>
     <head>
     <title>Welcome to nginx!</title>
     <style>
         body {
             width: 35em;
             margin: 0 auto;
             font-family: Tahoma, Verdana, Arial, sans-serif;
         }
     </style>
     </head>
     <body>
     <h1>Welcome to nginx!</h1>
     <p>If you see this page, the nginx web server is successfully installed and
     working. Further configuration is required.</p>
     
     <p>For online documentation and support please refer to
     <a href="http://nginx.org/">nginx.org</a>.<br/>
     Commercial support is available at
     <a href="http://nginx.com/">nginx.com</a>.</p>
     
     <p><em>Thank you for using nginx.</em></p>
     </body>
     </html>
     
     #设置services对pod进行轮训的负载均衡，默认对所有的pod进行轮训的负载均衡的
     #查看所有endpoints
     [root@k8snode1 ~]# kubectl get endpoints
     NAME         ENDPOINTS                                                           AGE
     kubernetes   192.168.10.185:6443                                                 40h
     nginxdemo1   172.16.185.194:80,172.16.185.195:80,172.16.185.196:80 + 1 more...   3m14s
     
     #查看endpoints详情
     [root@k8snode1 ~]# kubectl get endpoints
     NAME         ENDPOINTS                                                           AGE
     kubernetes   192.168.10.185:6443                                                 40h
     nginxdemo1   172.16.185.194:80,172.16.185.195:80,172.16.185.196:80 + 1 more...   3m14s
     [root@k8snode1 ~]# kubectl describe endpoints nginxdemo1
     Name:         nginxdemo1
     Namespace:    default
     Labels:       app=nginxdemo1
     Annotations:  endpoints.kubernetes.io/last-change-trigger-time: 2021-04-19T06:47:17Z
     Subsets:
       Addresses:          172.16.185.194,172.16.185.195,172.16.185.196,172.16.98.196
       NotReadyAddresses:  <none>
       Ports:
         Name     Port  Protocol
         ----     ----  --------
         <unset>  80    TCP
     
     Events:  <none>
     
     #service服务对外访问，修改type的ClusterIP为NodePort
     [root@k8snode1 ~]# kubectl get services
     NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
     kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   40h
     nginxdemo1   ClusterIP   10.105.52.111   <none>        80/TCP    11m
     
     [root@k8snode1 ~]# kubectl edit service nginxdemo1
     # Please edit the object below. Lines beginning with a '#' will be ignored,
     # and an empty file will abort the edit. If an error occurs while saving this file will be
     # reopened with the relevant failures.
     #
     apiVersion: v1
     kind: Service
     metadata:
       creationTimestamp: "2021-04-19T06:47:17Z"
       labels:
         app: nginxdemo1
       name: nginxdemo1
       namespace: default
       resourceVersion: "343162"
       uid: 998cda3e-0c0f-44b3-b12d-508f9a8038b9
     spec:
       clusterIP: 10.105.52.111
       clusterIPs:
       - 10.105.52.111
       ipFamilies:
       - IPv4
       ipFamilyPolicy: SingleStack
       ports:
       - port: 80
         protocol: TCP
         targetPort: 80
       selector:
         app: nginxdemo1
       sessionAffinity: None
       type: NodePort   #修改ClusterIP为NodePort
     status:
       loadBalancer: {}
     
     #修改完成 会发现在所有节点上会暴露一个31204的端口 映射到k8s内网的80上，由kube-proxy 转发
     [root@k8snode1 ~]# kubectl get services
     NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
     kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        40h
     nginxdemo1   NodePort    10.105.52.111   <none>        80:31204/TCP   12m
     
     [root@k8snode1 ~]# netstat -anp| grep 31204
     tcp        0      0 0.0.0.0:31204           0.0.0.0:*               LISTEN      29854/kube-proxy 
     
     [root@k8snode1 ~]# curl http://k8snode1:31204
     [root@k8snode1 ~]# curl http://k8snode2:31204
     [root@k8snode1 ~]# curl http://k8snode3:31204
     
     [root@k8snode1 ~]# kubectl describe services nginxdemo1
     Name:                     nginxdemo1
     Namespace:                default
     Labels:                   app=nginxdemo1
     Annotations:              <none>
     Selector:                 app=nginxdemo1
     Type:                     NodePort  #变成了nodeport
     IP Family Policy:         SingleStack
     IP Families:              IPv4
     IP:                       10.105.52.111
     IPs:                      10.105.52.111
     Port:                     <unset>  80/TCP
     TargetPort:               80/TCP
     NodePort:                 <unset>  31204/TCP  #nodeport的对外端口
     Endpoints:                172.16.185.194:80,172.16.185.195:80,172.16.185.196:80 + 1 more...
     Session Affinity:         None
     External Traffic Policy:  Cluster
     Events:                   <none>
     
     #注意，除了NodePort类型还有这么多--type='': Type for this service: ClusterIP, NodePort, LoadBalancer, or ExternalName. Default is 'ClusterIP'
     通过kubectl expose -h查看
     
     services的负载均衡和转发是基于iptables实现的，可以查看节点的iptables规则的一条条转发，也可以看到负载均衡的时候转发到每条链路的权重
     [root@k8snode1 ~]# iptables -t nat -L -n | less
     KUBE-MARK-MASQ  tcp  --  0.0.0.0/0            0.0.0.0/0            /* default/nginxdemo1 */ tcp dpt:31204
     KUBE-SVC-JUHXCDZEOSJBBI3Q  tcp  --  0.0.0.0/0            0.0.0.0/0            /* default/nginxdemo1 */ tcp dpt:31204
     
     ```

   * k8s的滚动更新。在pod容器进行更新的时候，会逐渐的进行更新，先构建一个新的pod，讲流量负载到新的pod上，成功后在删除旧的。依次这么操作可以做到零宕机

     ```
     #新开一个窗口，打开实时监控
     [root@k8snode1 ~]# kubectl get pods -w
     NAME                          READY   STATUS    RESTARTS   AGE
     nginxdemo                     1/1     Running   0          15h
     nginxdemo1-6995675b4b-6jhdt   1/1     Running   0          36m
     nginxdemo1-6995675b4b-9kqbb   1/1     Running   0          15h
     nginxdemo1-6995675b4b-bz22l   1/1     Running   0          36m
     nginxdemo1-6995675b4b-m9snz   1/1     Running   0          36m
     
     
     #查看要升级的镜像是哪个Containers
     [root@k8snode1 ~]# kubectl describe deployments.apps nginxdemo1
     Name:                   nginxdemo1
     Namespace:              default
     CreationTimestamp:      Sun, 18 Apr 2021 23:42:17 +0800
     Labels:                 app=nginxdemo1
     Annotations:            deployment.kubernetes.io/revision: 4
     Selector:               app=nginxdemo1
     Replicas:               4 desired | 4 updated | 4 total | 4 available | 0 unavailable
     StrategyType:           RollingUpdate
     MinReadySeconds:        0
     RollingUpdateStrategy:  25% max unavailable, 25% max surge
     Pod Template:
       Labels:  app=nginxdemo1
       Containers:
        nginx:  #可以看到Containers名字是nginx
         Image:        nginx:1.7.9
         Port:         80/TCP
         Host Port:    0/TCP
         Environment:  <none>
         Mounts:       <none>
       Volumes:        <none>
     Conditions:
       Type           Status  Reason
       ----           ------  ------
       Available      True    MinimumReplicasAvailable
       Progressing    True    NewReplicaSetAvailable
     OldReplicaSets:  <none>
     NewReplicaSet:   nginxdemo1-6995675b4b (4/4 replicas created)
     Events:
       Type    Reason             Age                    From                   Message
       ----    ------             ----                   ----                   -------
       Normal  ScalingReplicaSet  48m                    deployment-controller  Scaled up replica set nginxdemo1-6995675b4b to 4
       Normal  ScalingReplicaSet  10m                    deployment-controller  Scaled up replica set nginxdemo1-855bd4cd9f to 1
       Normal  ScalingReplicaSet  10m                    deployment-controller  Scaled down replica set nginxdemo1-6995675b4b to 3
       Normal  ScalingReplicaSet  10m                    deployment-controller  Scaled up replica set nginxdemo1-855bd4cd9f to 2
       Normal  ScalingReplicaSet  9m46s                  deployment-controller  Scaled down replica set nginxdemo1-6995675b4b to 2
       Normal  ScalingReplicaSet  9m46s                  deployment-controller  Scaled up replica set nginxdemo1-855bd4cd9f to 3
       Normal  ScalingReplicaSet  9m45s                  deployment-controller  Scaled down replica set nginxdemo1-6995675b4b to 1
       Normal  ScalingReplicaSet  9m45s                  deployment-controller  Scaled up replica set nginxdemo1-855bd4cd9f to 4
       Normal  ScalingReplicaSet  9m41s                  deployment-controller  Scaled down replica set nginxdemo1-6995675b4b to 0
       Normal  ScalingReplicaSet  5m26s                  deployment-controller  Scaled up replica set nginxdemo1-5bb6dd6f49 to 1
       Normal  ScalingReplicaSet  117s (x15 over 5m26s)  deployment-controller  (combined from similar events): Scaled down replica set nginxdemo1-5bb6dd6f49 to 0
     
     
     #更新nginx为1.8.1
     [root@k8snode1 ~]# kubectl set image deployments.apps nginxdemo1 nginx=nginx:1.8.1
     deployment.apps/nginxdemo1 image updated
     
     #可以看到先创建了一个容器，然后启动 在杀掉之前的容器
     [root@k8snode1 ~]# kubectl get pods -w
     NAME                          READY   STATUS    RESTARTS   AGE
     nginxdemo                     1/1     Running   0          15h
     nginxdemo1-6995675b4b-6jhdt   1/1     Running   0          36m
     nginxdemo1-6995675b4b-9kqbb   1/1     Running   0          15h
     nginxdemo1-6995675b4b-bz22l   1/1     Running   0          36m
     nginxdemo1-6995675b4b-m9snz   1/1     Running   0          36m
     nginxdemo1-855bd4cd9f-99dj7   0/1     Pending   0          0s   #1.先创建了一个容器
     nginxdemo1-6995675b4b-6jhdt   1/1     Terminating   0          38m  #2.在杀掉以前的其中一个容器
     nginxdemo1-855bd4cd9f-99dj7   0/1     Pending       0          0s
     nginxdemo1-855bd4cd9f-99dj7   0/1     ContainerCreating   0          0s  #再把之前创建的容器启动起来
     nginxdemo1-855bd4cd9f-574rx   0/1     Pending             0          0s
     nginxdemo1-855bd4cd9f-574rx   0/1     Pending             0          0s
     nginxdemo1-855bd4cd9f-574rx   0/1     ContainerCreating   0          0s
     nginxdemo1-6995675b4b-6jhdt   1/1     Terminating         0          38m
     nginxdemo1-855bd4cd9f-574rx   0/1     ContainerCreating   0          0s
     nginxdemo1-855bd4cd9f-99dj7   0/1     ContainerCreating   0          0s
     nginxdemo1-6995675b4b-6jhdt   0/1     Terminating         0          38m
     nginxdemo1-6995675b4b-6jhdt   0/1     Terminating         0          38m
     nginxdemo1-6995675b4b-6jhdt   0/1     Terminating         0          38m
     nginxdemo1-855bd4cd9f-574rx   1/1     Running             0          19s
     nginxdemo1-6995675b4b-bz22l   1/1     Terminating         0          38m
     nginxdemo1-855bd4cd9f-h5f5g   0/1     Pending             0          0s
     nginxdemo1-855bd4cd9f-h5f5g   0/1     Pending             0          0s
     nginxdemo1-855bd4cd9f-h5f5g   0/1     ContainerCreating   0          0s
     nginxdemo1-6995675b4b-bz22l   1/1     Terminating         0          38m
     nginxdemo1-855bd4cd9f-h5f5g   0/1     ContainerCreating   0          1s
     nginxdemo1-6995675b4b-bz22l   0/1     Terminating         0          38m
     nginxdemo1-855bd4cd9f-h5f5g   1/1     Running             0          1s
     nginxdemo1-6995675b4b-9kqbb   1/1     Terminating         0          15h
     nginxdemo1-855bd4cd9f-w8xmq   0/1     Pending             0          0s
     nginxdemo1-855bd4cd9f-w8xmq   0/1     Pending             0          0s
     nginxdemo1-855bd4cd9f-w8xmq   0/1     ContainerCreating   0          0s
     nginxdemo1-6995675b4b-9kqbb   1/1     Terminating         0          15h
     nginxdemo1-6995675b4b-9kqbb   0/1     Terminating         0          15h
     nginxdemo1-855bd4cd9f-w8xmq   0/1     ContainerCreating   0          1s
     nginxdemo1-6995675b4b-bz22l   0/1     Terminating         0          38m
     nginxdemo1-855bd4cd9f-99dj7   1/1     Running             0          24s  #3.新的容器启动成功 之后在重复 直到所有的容器更新完成
     nginxdemo1-6995675b4b-bz22l   0/1     Terminating         0          38m
     nginxdemo1-6995675b4b-m9snz   1/1     Terminating         0          38m
     nginxdemo1-6995675b4b-m9snz   1/1     Terminating         0          38m
     nginxdemo1-6995675b4b-m9snz   0/1     Terminating         0          38m
     nginxdemo1-855bd4cd9f-w8xmq   1/1     Running             0          7s
     nginxdemo1-6995675b4b-m9snz   0/1     Terminating         0          38m
     nginxdemo1-6995675b4b-m9snz   0/1     Terminating         0          38m
     nginxdemo1-6995675b4b-9kqbb   0/1     Terminating         0          15h
     nginxdemo1-6995675b4b-9kqbb   0/1     Terminating         0          15h   #直到这里，所有的老的容器全部杀掉成功
     
     #查看新的全部容器 已经更新完成和上面的镜像全部对应
     [root@k8snode1 ~]# kubectl get pods
     NAME                          READY   STATUS    RESTARTS   AGE
     nginxdemo                     1/1     Running   0          15h
     nginxdemo1-855bd4cd9f-574rx   1/1     Running   0          3m2s
     nginxdemo1-855bd4cd9f-99dj7   1/1     Running   0          3m2s
     nginxdemo1-855bd4cd9f-h5f5g   1/1     Running   0          2m43s
     nginxdemo1-855bd4cd9f-w8xmq   1/1     Running   0          2m42s
     
     #中间如果停止了，可以重启下
     [root@k8snode1 ~]# kubectl rollout restart deployment 
     deployment.apps/nginxdemo1 restarted
     
     #查看已经成功了
     [root@k8snode1 ~]# kubectl rollout status deployment nginxdemo1
     deployment "nginxdemo1" successfully rolled out
     
     #查看版本已经更新成功
     [root@k8snode1 ~]# curl -I http://k8snode1:31204
     HTTP/1.1 200 OK
     Server: nginx/1.8.1
     Date: Mon, 19 Apr 2021 07:23:10 GMT
     Content-Type: text/html
     Content-Length: 612
     Last-Modified: Tue, 26 Jan 2016 15:24:47 GMT
     Connection: keep-alive
     ETag: "56a78fbf-264"
     Accept-Ranges: bytes
     
     
     #版本回退
     #查看历史版本
     [root@k8snode1 ~]# kubectl rollout history deployment nginxdemo1
     deployment.apps/nginxdemo1 
     REVISION  CHANGE-CAUSE
     1         <none>
     2         <none>
     3         <none>
     
     #回退到某个版本。比如1
     [root@k8snode1 ~]# kubectl rollout undo -h
     [root@k8snode1 ~]# kubectl rollout undo deployment nginxdemo1 --to-revision=1
     deployment.apps/nginxdemo1 rolled back
     
     #查看是否成功
     [root@k8snode1 ~]# kubectl rollout status deployment nginxdemo1
     deployment "nginxdemo1" successfully rolled out
     [root@k8snode1 ~]# curl -I http://k8snode1:31204
     HTTP/1.1 200 OK
     Server: nginx/1.7.9
     Date: Mon, 19 Apr 2021 07:26:03 GMT
     Content-Type: text/html
     Content-Length: 612
     Last-Modified: Tue, 23 Dec 2014 16:25:09 GMT
     Connection: keep-alive
     ETag: "54999765-264"
     Accept-Ranges: bytes
     
     ```

   * 参考文档

     * 基础概念：https://kubernetes.io/docs/tutorials/kubernetes-basics/
     * 部署应⽤：https://kubernetes.io/docs/tutorials/kubernetes-basics/deploy-app/deploy-intro/
     * 访问应⽤：https://kubernetes.io/docs/tutorials/kubernetes-basics/explore/explore-intro/
     * 外部访问：https://kubernetes.io/docs/tutorials/kubernetes-basics/expose/expose-intro/
     * 访问应⽤：https://kubernetes.io/docs/tutorials/kubernetes-basics/scale/scale-intro/
     * 滚动升级：https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/

# ApiService

ApiService主要与认证和授权

1. k8s提供客户端证书认证(TLS双向认证)，客户端在访问ApiService的时候，双方验证证书是否合法，k8s用的自己的一个CA证书机制

2. BearerToken认证，通过一个复杂的密码进行认证

3. ServuceAccount认证，容器内的pod需要访问ApiService时候的认证方式，包括namespace、token、CA

4. 鉴权分为ABAC  WebHook  RBAC

5. RBAC是最新的鉴权方式，在1.6版本引入的。全称为Role Based Access Control 基于角色的访问控制

   基于User->Role->Authority方式

   * User分为user用户和serviceAccount ，user就是普通的用户访问 serviceAccount 是集群内部访问
   * Role：包括角色名字name 角色权限resources 和verbs
   * Authority是资源权限访问维度，分为Resource和Verbs(就是CURD的方式)
   * RoleBinding：用来标识角色和用户的关系

6. 角色是在namespace下的，这个角色权限只能在这个namespace下生效

7. ClusterRole：集群角色。用于操作集群的角色定义，用来确定某个用户和多个namespace的权限绑定。相应也会有个ClusterRoleBinding

8. AdminisionContrlol：相当于过滤器，在一个请求进来的时候，会根据配置依次执行所有的过滤器

# kubelet 

在每个work节点都存在。负责维护当前节点的pod的生命周期，网络，资源等管理，最终调用当前节点的容器来启动一个个pod

# Pods

参考：https://kubernetes.io/zh/docs/concepts/workloads/pods/

一个pod里可以有多个容器 pod里有个pause容器，这个容器的作用就是把这个 pod里的其他容器相关联

是k8s的最基础最小的一个执行单元，pod里封装了container，也可以封装多个container，如果是多个container，那么多个container共享网络和磁盘的命名空间并且调度到同一个node上

通常你不需要直接创建 Pod，甚至单实例 Pod。 相反，你会使用诸如 [Deployment](https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/) 或 [Job](https://kubernetes.io/zh/docs/concepts/workloads/controllers/job/) 这类工作负载资源 来创建 Pod。如果 Pod 需要跟踪状态， 可以考虑 [StatefulSet](https://kubernetes.io/zh/docs/concepts/workloads/controllers/statefulset/) 资源。

Kubernetes 集群中的 Pod 主要有两种用法：

- **运行单个容器的 Pod**。"每个 Pod 一个容器"模型是最常见的 Kubernetes 用例； 在这种情况下，可以将 Pod 看作单个容器的包装器，并且 Kubernetes 直接管理 Pod，而不是容器。
- **运行多个协同工作的容器的 Pod**。 Pod 可能封装由多个紧密耦合且需要共享资源的共处容器组成的应用程序。 这些位于同一位置的容器可能形成单个内聚的服务单元 —— 一个容器将文件从共享卷提供给公众， 而另一个单独的“挂斗”（sidecar）容器则刷新或更新这些文件。 Pod 将这些容器和存储资源打包为一个可管理的实体。

pod管理多个容器

​	Pod 被设计成支持形成内聚服务单元的多个协作过程（形式为容器）。 Pod 中的容器被自动安排到集群中的同一物理机或虚拟机上，并可以一起进行调度。 容器之间可以共享资源和依赖、彼此通信、协调何时以及何种方式终止自身。

例如，你可能有一个容器，为共享卷中的文件提供 Web 服务器支持，以及一个单独的 “sidecar（挂斗）”容器负责从远端更新这些文件，

pause容器 在pod里的第一个容器

pod可以单独定义，但是没有自我修复的能力，一般通过控制器来控制pod，防止一个pod出问题了 其他的pod顶上来

1. 查看pod语法

   ```
   [root@k8snode1 ~]# kubectl explain pods
   KIND:     Pod
   VERSION:  v1
   
   DESCRIPTION:
        Pod is a collection of containers that can run on a host. This resource is
        created by clients and scheduled onto hosts.
   
   FIELDS:
      apiVersion	<string>  #版本
        APIVersion defines the versioned schema of this representation of an
        object. Servers should convert recognized schemas to the latest internal
        value, and may reject unrecognized values. More info:
        https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources
   
      kind	<string>  #资源类型
        Kind is a string value representing the REST resource this object
        represents. Servers may infer this from the endpoint the client submits
        requests to. Cannot be updated. In CamelCase. More info:
        https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds
   
      metadata	<Object>  #元数据
        Standard object's metadata. More info:
        https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
   
      spec	<Object>  #
        Specification of the desired behavior of the pod. More info:
        https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
   
      status	<Object>  #
        Most recently observed status of the pod. This data may not be up to date.
        Populated by the system. Read-only. More info:
        https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
   
   #查看详情命令
   [root@k8snode1 ~]# kubectl explain pods.metadata
   KIND:     Pod
   VERSION:  v1
   
   RESOURCE: metadata <Object>
   
   DESCRIPTION:
        Standard object's metadata. More info:
        https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
   
        ObjectMeta is metadata that all persisted resources must have, which
        includes all objects users must create.
   
   FIELDS:
      annotations	<map[string]string>
        Annotations is an unstructured key value map stored with a resource that
        may be set by external tools to store and retrieve arbitrary metadata. They
        are not queryable and should be preserved when modifying objects. More
        info: http://kubernetes.io/docs/user-guide/annotations
   
      clusterName	<string>
        The name of the cluster which the object belongs to. This is used to
        distinguish resources with same name and namespace in different clusters.
        This field is not set anywhere right now and apiserver is going to ignore
        it if set in create or update request.
   
      creationTimestamp	<string>
        CreationTimestamp is a timestamp representing the server time when this
        object was created. It is not guaranteed to be set in happens-before order
        across separate operations. Clients may not set this value. It is
        represented in RFC3339 form and is in UTC.
   
        Populated by the system. Read-only. Null for lists. More info:
        https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
   
      deletionGracePeriodSeconds	<integer>
        Number of seconds allowed for this object to gracefully terminate before it
        will be removed from the system. Only set when deletionTimestamp is also
        set. May only be shortened. Read-only.
   
      deletionTimestamp	<string>
        DeletionTimestamp is RFC 3339 date and time at which this resource will be
        deleted. This field is set by the server when a graceful deletion is
        requested by the user, and is not directly settable by a client. The
        resource is expected to be deleted (no longer visible from resource lists,
        and not reachable by name) after the time in this field, once the
        finalizers list is empty. As long as the finalizers list contains items,
        deletion is blocked. Once the deletionTimestamp is set, this value may not
        be unset or be set further into the future, although it may be shortened or
        the resource may be deleted prior to this time. For example, a user may
        request that a pod is deleted in 30 seconds. The Kubelet will react by
        sending a graceful termination signal to the containers in the pod. After
        that 30 seconds, the Kubelet will send a hard termination signal (SIGKILL)
        to the container and after cleanup, remove the pod from the API. In the
        presence of network partitions, this object may still exist after this
        timestamp, until an administrator or automated process can determine the
        resource is fully terminated. If not set, graceful deletion of the object
        has not been requested.
   
        Populated by the system when a graceful deletion is requested. Read-only.
        More info:
        https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
   
      finalizers	<[]string>
        Must be empty before the object is deleted from the registry. Each entry is
        an identifier for the responsible component that will remove the entry from
        the list. If the deletionTimestamp of the object is non-nil, entries in
        this list can only be removed. Finalizers may be processed and removed in
        any order. Order is NOT enforced because it introduces significant risk of
        stuck finalizers. finalizers is a shared field, any actor with permission
        can reorder it. If the finalizer list is processed in order, then this can
        lead to a situation in which the component responsible for the first
        finalizer in the list is waiting for a signal (field value, external
        system, or other) produced by a component responsible for a finalizer later
        in the list, resulting in a deadlock. Without enforced ordering finalizers
        are free to order amongst themselves and are not vulnerable to ordering
        changes in the list.
   
      generateName	<string>
        GenerateName is an optional prefix, used by the server, to generate a
        unique name ONLY IF the Name field has not been provided. If this field is
        used, the name returned to the client will be different than the name
        passed. This value will also be combined with a unique suffix. The provided
        value has the same validation rules as the Name field, and may be truncated
        by the length of the suffix required to make the value unique on the
        server.
   
        If this field is specified and the generated name exists, the server will
        NOT return a 409 - instead, it will either return 201 Created or 500 with
        Reason ServerTimeout indicating a unique name could not be found in the
        time allotted, and the client should retry (optionally after the time
        indicated in the Retry-After header).
   
        Applied only if Name is not specified. More info:
        https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#idempotency
   
      generation	<integer>
        A sequence number representing a specific generation of the desired state.
        Populated by the system. Read-only.
   
      labels	<map[string]string>
        Map of string keys and values that can be used to organize and categorize
        (scope and select) objects. May match selectors of replication controllers
        and services. More info: http://kubernetes.io/docs/user-guide/labels
   
      managedFields	<[]Object>
        ManagedFields maps workflow-id and version to the set of fields that are
        managed by that workflow. This is mostly for internal housekeeping, and
        users typically shouldn't need to set or understand this field. A workflow
        can be the user's name, a controller's name, or the name of a specific
        apply path like "ci-cd". The set of fields is always in the version that
        the workflow used when modifying the object.
   
      name	<string>
        Name must be unique within a namespace. Is required when creating
        resources, although some resources may allow a client to request the
        generation of an appropriate name automatically. Name is primarily intended
        for creation idempotence and configuration definition. Cannot be updated.
        More info: http://kubernetes.io/docs/user-guide/identifiers#names
   
      namespace	<string>
        Namespace defines the space within which each name must be unique. An empty
        namespace is equivalent to the "default" namespace, but "default" is the
        canonical representation. Not all objects are required to be scoped to a
        namespace - the value of this field for those objects will be empty.
   
        Must be a DNS_LABEL. Cannot be updated. More info:
        http://kubernetes.io/docs/user-guide/namespaces
   
      ownerReferences	<[]Object>
        List of objects depended by this object. If ALL objects in the list have
        been deleted, this object will be garbage collected. If this object is
        managed by a controller, then an entry in this list will point to this
        controller, with the controller field set to true. There cannot be more
        than one managing controller.
   
      resourceVersion	<string>
        An opaque value that represents the internal version of this object that
        can be used by clients to determine when objects have changed. May be used
        for optimistic concurrency, change detection, and the watch operation on a
        resource or set of resources. Clients must treat these values as opaque and
        passed unmodified back to the server. They may only be valid for a
        particular resource or set of resources.
   
        Populated by the system. Read-only. Value must be treated as opaque by
        clients and . More info:
        https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#concurrency-control-and-consistency
   
      selfLink	<string>
        SelfLink is a URL representing this object. Populated by the system.
        Read-only.
   
        DEPRECATED Kubernetes will stop propagating this field in 1.20 release and
        the field is planned to be removed in 1.21 release.
   
      uid	<string>
        UID is the unique in time and space value for this object. It is typically
        generated by the server on successful creation of a resource and is not
        allowed to change on PUT operations.
   
        Populated by the system. Read-only. More info:
        http://kubernetes.io/docs/user-guide/identifiers#uids
   
   
   ```

2. pod定义单个容器

   ```
   [root@k8snode1 k8s]# vim poddemo.yaml
   
   apiVersion: v1
   kind: Pod
   metadata:
     name: nginxpodsdemo
     annotations:
       poduse: "pod demo the pod"
     labels: 
       app: nginxpodsdemo
       version: 1.7.9
   spec:
     containers:
     - name: nginxpodsdemo
       image: nginx:1.7.9
       ports:
       - name: nginx-port-80
         protocol: TCP
         containerPort: 80
   ```

   * apiVersion api使⽤的版本,kubectl api-versions可查看到当前系统能⽀持的版本列表
   * kind 指定资源类型，表示为Pod的资源类型
   * metadata 指定Pod的元数据，metadata.name指定名称，metadata.labels指定Pod的所属的标签
   * spec 指定Pod的模版属性，spec.containers配置容器的信息，spec.containers.name指定名字，spec.containers.image指定容器镜像的名称，spec.containers.imagePullPolicy是镜像的下载⽅式，IfNotPresent表示当镜像不存在时下载spec.containers.ports.name指定port的名称，spec.containers.ports.protocol协议类型为TCP，
     spec.containers.ports.containerPort为容器端⼝。

3. 提交pod应用

   ```
   [root@k8snode1 k8s]# kubectl apply -f poddemo.yaml 
   pod/nginxpodsdemo created
   [root@k8snode1 k8s]# kubectl get pod
   poddisruptionbudgets.policy  pods                         podsecuritypolicies.policy   podtemplates
   [root@k8snode1 k8s]# kubectl get pods
   NAME                                    READY   STATUS        RESTARTS   AGE
   nginx-dashboard-demo-5cd59c4f6d-bpq6h   1/1     Running       2          24h
   nginx-dashboard-demo-5cd59c4f6d-dbc2d   1/1     Terminating   0          24h
   nginx-dashboard-demo-5cd59c4f6d-j2c7l   1/1     Running       1          19h
   nginx-dashboard-demo-5cd59c4f6d-jbzb8   1/1     Terminating   0          24h
   nginx-dashboard-demo-5cd59c4f6d-w2n6d   1/1     Running       1          19h
   nginxdemo                               1/1     Running       2          42h
   nginxdemo1-6995675b4b-gtbhh             1/1     Running       1          19h
   nginxdemo1-6995675b4b-jbfrt             1/1     Running       2          27h
   nginxdemo1-6995675b4b-pnjnp             1/1     Running       2          27h
   nginxdemo1-6995675b4b-rsc4s             1/1     Terminating   0          27h
   nginxdemo1-6995675b4b-z7q72             1/1     Running       2          27h
   nginxpodsdemo                           1/1     Running       0          8s
   ```

4. pod的生命周期

   | 取值                | 描述                                                         |
   | :------------------ | :----------------------------------------------------------- |
   | `Pending`（悬决）   | Pod 已被 Kubernetes 系统接受，但有一个或者多个容器尚未创建亦未运行。此阶段包括等待 Pod 被调度的时间和通过网络下载镜像的时间， |
   | `Running`（运行中） | Pod 已经绑定到了某个节点，Pod 中所有的容器都已被创建。至少有一个容器仍在运行，或者正处于启动或重启状态。 |
   | `Succeeded`（成功） | Pod 中的所有容器都已成功终止，并且不会再重启。               |
   | `Failed`（失败）    | Pod 中的所有容器都已终止，并且至少有一个容器是因为失败终止。也就是说，容器以非 0 状态退出或者被系统终止。 |
   | `Unknown`（未知）   | 因为某些原因无法取得 Pod 的状态。这种情况通常是因为与 Pod 所在主机通信失败。 |

5. 查看pod详细信息

   ```
   [root@k8snode1 k8s]# kubectl describe pods nginxpodsdemo
   Name:         nginxpodsdemo
   Namespace:    default
   Priority:     0
   Node:         k8snode2/192.168.10.186
   Start Time:   Tue, 20 Apr 2021 18:32:03 +0800
   Labels:       app=nginxpodsdemo
                 version=1.7.9
   Annotations:  cni.projectcalico.org/podIP: 172.16.185.229/32
                 cni.projectcalico.org/podIPs: 172.16.185.229/32
                 poduse: pod demo the pod
   Status:       Running
   IP:           172.16.185.229
   IPs:
     IP:  172.16.185.229
   Containers:
     nginxpodsdemo:
       Container ID:   docker://5bae15e0469811c95e54e85c1f88aaddf8d1d7d7f81a3ecfc70c08ce87031a40
       Image:          nginx:1.7.9
       Image ID:       docker-pullable://nginx@sha256:e3456c851a152494c3e4ff5fcc26f240206abac0c9d794affb40e0714846c451
       Port:           80/TCP
       Host Port:      0/TCP
       State:          Running
         Started:      Tue, 20 Apr 2021 18:32:04 +0800
       Ready:          True
       Restart Count:  0
       Environment:    <none>
       Mounts:
         /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-x55fx (ro)
   Conditions:
     Type              Status
     Initialized       True 
     Ready             True 
     ContainersReady   True 
     PodScheduled      True 
   Volumes:
     kube-api-access-x55fx:
       Type:                    Projected (a volume that contains injected data from multiple sources)
       TokenExpirationSeconds:  3607
       ConfigMapName:           kube-root-ca.crt
       ConfigMapOptional:       <nil>
       DownwardAPI:             true
   QoS Class:                   BestEffort
   Node-Selectors:              <none>
   Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                                node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
   Events:
     Type    Reason     Age    From               Message
     ----    ------     ----   ----               -------
     Normal  Scheduled  2m59s  default-scheduler  Successfully assigned default/nginxpodsdemo to k8snode2
     Normal  Pulled     3m     kubelet            Container image "nginx:1.7.9" already present on machine
     Normal  Created    2m59s  kubelet            Created container nginxpodsdemo
     Normal  Started    2m59s  kubelet            Started container nginxpodsdemo
   
   
   [root@k8snode1 k8s]# kubectl get pods nginxpodsdemo -o yaml
   apiVersion: v1
   kind: Pod
   metadata:
     annotations:
       cni.projectcalico.org/podIP: 172.16.185.229/32
       cni.projectcalico.org/podIPs: 172.16.185.229/32
       kubectl.kubernetes.io/last-applied-configuration: |
         {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{"poduse":"pod demo the pod"},"labels":{"app":"nginxpodsdemo","version":"1.7.9"},"name":"nginxpodsdemo","namespace":"default"},"spec":{"containers":[{"image":"nginx:1.7.9","name":"nginxpodsdemo","ports":[{"containerPort":80,"name":"nginx-port-80","protocol":"TCP"}]}]}}
       poduse: pod demo the pod
     creationTimestamp: "2021-04-20T10:32:03Z"
     labels:
       app: nginxpodsdemo
       version: 1.7.9
     name: nginxpodsdemo
     namespace: default
     resourceVersion: "472119"
     uid: 6f4bcc27-a908-471e-87e9-52080531d157
   spec:
     containers:
     - image: nginx:1.7.9
       imagePullPolicy: IfNotPresent
       name: nginxpodsdemo
       ports:
       - containerPort: 80
         name: nginx-port-80
         protocol: TCP
       resources: {}
       terminationMessagePath: /dev/termination-log
       terminationMessagePolicy: File
       volumeMounts:
       - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
         name: kube-api-access-x55fx
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
     - name: kube-api-access-x55fx
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
     	#下面是容器在运行过程中的中间状态
     - lastProbeTime: null
       lastTransitionTime: "2021-04-20T10:32:03Z"
       status: "True"
       type: Initialized  #初始化
     - lastProbeTime: null
       lastTransitionTime: "2021-04-20T10:32:04Z"
       status: "True"
       type: Ready  #完成 中间还有个containersready状态
     - lastProbeTime: null
       lastTransitionTime: "2021-04-20T10:32:04Z"
       status: "True"
       type: ContainersReady
     - lastProbeTime: null
       lastTransitionTime: "2021-04-20T10:32:03Z"
       status: "True"
       type: PodScheduled
     containerStatuses:
     - containerID: docker://5bae15e0469811c95e54e85c1f88aaddf8d1d7d7f81a3ecfc70c08ce87031a40
       image: nginx:1.7.9
       imageID: docker-pullable://nginx@sha256:e3456c851a152494c3e4ff5fcc26f240206abac0c9d794affb40e0714846c451
       lastState: {}
       name: nginxpodsdemo
       ready: true
       restartCount: 0
       started: true
       state:
         running:
           startedAt: "2021-04-20T10:32:04Z"
     hostIP: 192.168.10.186
     phase: Running
     podIP: 172.16.185.229
     podIPs:
     - ip: 172.16.185.229
     qosClass: BestEffort
     startTime: "2021-04-20T10:32:03Z"
   
   ```

6. 会有个容器探针来对pod进行诊断会有三种结果

   每次探测都将获得以下三种结果之一：

   - `Success`（成功）：容器通过了诊断。
   - `Failure`（失败）：容器未通过诊断。
   - `Unknown`（未知）：诊断失败，因此不会采取任何行动。

   针对运行中的容器，`kubelet` 可以选择是否执行以下三种探针，以及如何针对探测结果作出反应：

   - `livenessProbe`：指示容器是否正在运行。如果存活态探测失败，则 kubelet 会杀死容器， 并且容器将根据其[重启策略](https://kubernetes.io/zh/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy)决定未来。如果容器不提供存活探针， 则默认状态为 `Success`。
   - `readinessProbe`：指示容器是否准备好为请求提供服务。如果就绪态探测失败， 端点控制器将从与 Pod 匹配的所有服务的端点列表中删除该 Pod 的 IP 地址。 初始延迟之前的就绪态的状态值默认为 `Failure`。 如果容器不提供就绪态探针，则默认状态为 `Success`。
   - `startupProbe`: 指示容器中的应用是否已经启动。如果提供了启动探针，则所有其他探针都会被 禁用，直到此探针成功为止。如果启动探测失败，`kubelet` 将杀死容器，而容器依其 [重启策略](https://kubernetes.io/zh/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy)进行重启。 如果容器没有提供启动探测，则默认状态为 `Success`。

7. 镜像拉去策略

   参考文档：https://kubernetes.io/zh/docs/concepts/containers/images/

   容器镜像拉取策略包含有两种：

   * IfNotPresent，判断本地是否存在，如果不存在责则拉取镜像，默认策略。
   * Always，总是拉取镜像，不论本地是否存在都会拉取镜像

   镜像拉去策略通过imagePullPolicy来标记

   默认的镜像拉取策略是 `IfNotPresent`：在镜像已经存在的情况下， [kubelet](https://kubernetes.io/docs/reference/generated/kubelet) 将不再去拉取镜像。 如果希望强制总是拉取镜像，你可以执行以下操作之一：

   - 设置容器的 `imagePullPolicy` 为 `Always`。
   - 省略 `imagePullPolicy`，并使用 `:latest` 作为要使用的镜像的标签。
   - 省略 `imagePullPolicy` 和要使用的镜像标签。
   - 启用 [AlwaysPullImages](https://kubernetes.io/zh/docs/reference/access-authn-authz/admission-controllers/#alwayspullimages) 准入控制器（Admission Controller）。

   如果 `imagePullPolicy` 未被定义为特定的值，也会被设置为 `Always`

8. 容器启动命令和参数

   参考文档：https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/

   Docker启动的时候会有⼀个默认的启动进程，启动⽅式在dockerfile中包含有两种：

   * CMD
   * EntryPoint

   EntryPoint通常定义的是启动的命令，CMD定义的是启动参数，当然CMD也可以定义启 如果EntryPoint为空那么久读取cmd
   动的命令+参数

   在kubernetes中command对应的是EntryPoint，args对应的是CMD。

   * 未指定command和args，则使⽤容器默认的EntryPoint和CMD
   * 使⽤command未指定args，则将command替换EntryPoint
   * 使⽤args未使⽤command，则args替换CMD参数
   * 使⽤command和args，则使⽤command替换EntryPoint，args替换CMD参数

   如果要覆盖默认的 Entrypoint 与 Cmd，需要遵循如下规则：

   - 如果在容器配置中没有设置 `command` 或者 `args`，那么将使用 Docker 镜像自带的命令及其参数。
   - 如果在容器配置中只设置了 `command` 但是没有设置 `args`，那么容器启动时只会执行该命令， Docker 镜像中自带的命令及其参数会被忽略。
   - 如果在容器配置中只设置了 `args`，那么 Docker 镜像中自带的命令会使用该新参数作为其执行时的参数。
   - 如果在容器配置中同时设置了 `command` 与 `args`，那么 Docker 镜像中自带的命令及其参数会被忽略。 容器启动时只会执行配置中设置的命令，并使用配置中设置的参数作为命令的参数。

   ```
   #查看启动的镜像 可以看到EntryPoint是空的
   [root@k8snode2 ~]# docker image inspect nginx:1.7.9
   [
       {
           "Id": "sha256:84581e99d807a703c9c03bd1a31cd9621815155ac72a7365fd02311264512656",
           "RepoTags": [
               "nginx:1.7.9"
           ],
           "RepoDigests": [
               "nginx@sha256:e3456c851a152494c3e4ff5fcc26f240206abac0c9d794affb40e0714846c451"
           ],
           "Parent": "",
           "Comment": "",
           "Created": "2015-01-27T18:43:46.874407465Z",
           "Container": "a90675014650c505222e83897ffabcc42d43edd9465b4702dd4dc170a8713585",
           "ContainerConfig": {
               "Hostname": "dc534047acbb",
               "Domainname": "",
               "User": "",
               "AttachStdin": false,
               "AttachStdout": false,
               "AttachStderr": false,
               "ExposedPorts": {
                   "443/tcp": {},
                   "80/tcp": {}
               },
               "Tty": false,
               "OpenStdin": false,
               "StdinOnce": false,
               "Env": [
                   "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                   "NGINX_VERSION=1.7.9-1~wheezy"
               ],
               "Cmd": [
                   "/bin/sh",
                   "-c",
                   "#(nop) CMD [nginx -g daemon off;]"
               ],
               "Image": "e90c322c3a1c8416eb76e6eec8ad2aac7ae2c37b9e6fe6d62cce8224f90e3001",
               "Volumes": {
                   "/var/cache/nginx": {}
               },
               "WorkingDir": "",
               "Entrypoint": null,
               "OnBuild": [],
               "Labels": null
           },
           "DockerVersion": "1.4.1",
           "Author": "NGINX Docker Maintainers \"docker-maint@nginx.com\"",
           "Config": {
               "Hostname": "dc534047acbb",
               "Domainname": "",
               "User": "",
               "AttachStdin": false,
               "AttachStdout": false,
               "AttachStderr": false,
               "ExposedPorts": {
                   "443/tcp": {},
                   "80/tcp": {}
               },
               "Tty": false,
               "OpenStdin": false,
               "StdinOnce": false,
               "Env": [
                   "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                   "NGINX_VERSION=1.7.9-1~wheezy"
               ],
               "Cmd": [
                   "nginx",
                   "-g",
                   "daemon off;"
               ],
               "Image": "e90c322c3a1c8416eb76e6eec8ad2aac7ae2c37b9e6fe6d62cce8224f90e3001",
               "Volumes": {
                   "/var/cache/nginx": {}
               },
               "WorkingDir": "",
               "Entrypoint": null,
               "OnBuild": [],
               "Labels": null
           },
           "Architecture": "amd64",
           "Os": "linux",
           "Size": 91664166,
           "VirtualSize": 91664166,
           "GraphDriver": {
               "Data": {
                   "LowerDir": "/var/lib/docker/overlay2/c5f40ecbb11380abf21b2c03e55de28dfbfca5162e0d84a0645ce2f72ee5c7a9/diff:/var/lib/docker/overlay2/9be332ff962daae0ac84d88443428a03bd62a306c374bb0b8394ecce040e2cf7/diff:/var/lib/docker/overlay2/42c47924b2560d0f5159b229b6128d7293d6afafeeba070fd245ab466152cdc1/diff:/var/lib/docker/overlay2/b48caba0ab00be6cdf2e27972d264d6e00f47665e748d59dd54f1d623d35df84/diff:/var/lib/docker/overlay2/27ba7bcdaf00da4bc03e358563f2b06e19fbb0f160e52364ad2c105f2d6f4002/diff:/var/lib/docker/overlay2/770b770b94dc1b7253b121c13ca6d1b12dc2ecbca30406e9a70213873b0ccce5/diff:/var/lib/docker/overlay2/4cdb54e0108d9a20936a32988f3822f6e6640943c16139f895cbd2c828046723/diff:/var/lib/docker/overlay2/8ac8a20c64bbce06c11883858d39ddb9b297f70212e7f46d109c571fd7f61ddb/diff:/var/lib/docker/overlay2/6eeb475ab7893f79112d665b0a56f05281cae67ddf68b4effcbc220c5665d18c/diff:/var/lib/docker/overlay2/f4a0aef8d764f67aaf6d26ea3996049d2c3c7f6894cc3ea322f1c460ccd32b23/diff:/var/lib/docker/overlay2/73e9aa1fd089d6762665dcd05f5c15dc347c728d44467597b7e95e6c9dd09d2a/diff:/var/lib/docker/overlay2/02715ac4c06a385507983a94ed2fddbb4edd3cbde6302e48defc139d128fb683/diff",
                   "MergedDir": "/var/lib/docker/overlay2/e9919586c7449b294f7b17a2483ceb7c9c616581ceae2698788363cb21d50bef/merged",
                   "UpperDir": "/var/lib/docker/overlay2/e9919586c7449b294f7b17a2483ceb7c9c616581ceae2698788363cb21d50bef/diff",
                   "WorkDir": "/var/lib/docker/overlay2/e9919586c7449b294f7b17a2483ceb7c9c616581ceae2698788363cb21d50bef/work"
               },
               "Name": "overlay2"
           },
           "RootFS": {
               "Type": "layers",
               "Layers": [
                   "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef",
                   "sha256:dea2e4984e295e43392ea576c8159168eb877c117714940abf3af9b604e874c2",
                   "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef",
                   "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef",
                   "sha256:e02dce553481aaff0458344505cb6adc66c2714755af96a4234463c7374be46a",
                   "sha256:63bf84221cce5ef5920911ded704f3bf363f6024929d3b8c9aa31a4b0c7fb46a",
                   "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef",
                   "sha256:e387107e2065a133384512ef47c36a2c1a31961310468b3c3eae00f4da47f2c1",
                   "sha256:ccb1d68e3fb76bc27e1745f004ba389489b56d00f350c6e93f7cc5b8baa03309",
                   "sha256:4b26ab29a475a42c35ee78b4153a67eaa9ef33e7c9e1c7d1f22c113a71b71604",
                   "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef",
                   "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef",
                   "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef"
               ]
           },
           "Metadata": {
               "LastTagTime": "0001-01-01T00:00:00Z"
           }
       }
   ]
   ```

9. 自定义启动参数

   ```
   [root@k8snode1 k8s]# vim poddemo1.yaml 
   apiVersion: v1
   kind: Pod
   metadata:
     name: nginxpodsdemo
     annotations:
       poduse: "pod demo the pod"
     labels:
       app: nginxpodsdemo
       version: 1.7.9
   spec:
     containers:
     - name: nginxpodsdemo
       image: nginx:1.7.9
       imagePullPolicy: IfNotPresent
       command: ["nginx"]
       args: ["-g", "daemon off;"]
       ports:
       - name: nginx-port-80
         protocol: TCP
         containerPort: 80    
   
   [root@k8snode1 k8s]# kubectl apply -f poddemo1.yaml 
   pod/nginxpodsdemo1 created
   
   
   # 也可以这么写
   [root@k8snode1 k8s]# vim poddemo2.yaml 
   
   apiVersion: v1
   kind: Pod
   metadata:
     name: nginxpodsdemo2
     annotations:
       poduse: "pod demo the pod"
     labels:
       app: nginxpodsdemo2
       version: 1.7.9
   spec:
     containers:
     - name: nginxpodsdemo2
       image: nginx:1.7.9
       imagePullPolicy: IfNotPresent
       command: ["nginx","-g", "daemon off;"]
       ports:
       - name: nginx-port-80
         protocol: TCP
         containerPort: 80
   
   [root@k8snode1 k8s]# kubectl apply -f poddemo2.yaml 
   pod/nginxpodsdemo2 created
   
   [root@k8snode1 k8s]# kubectl get pods
   NAME                                    READY   STATUS        RESTARTS   AGE
   nginx-dashboard-demo-5cd59c4f6d-bpq6h   1/1     Running       2          28h
   nginx-dashboard-demo-5cd59c4f6d-dbc2d   1/1     Terminating   0          28h
   nginx-dashboard-demo-5cd59c4f6d-j2c7l   1/1     Running       1          23h
   nginx-dashboard-demo-5cd59c4f6d-jbzb8   1/1     Terminating   0          28h
   nginx-dashboard-demo-5cd59c4f6d-w2n6d   1/1     Running       1          23h
   nginxdemo                               1/1     Running       2          46h
   nginxdemo1-6995675b4b-gtbhh             1/1     Running       1          23h
   nginxdemo1-6995675b4b-jbfrt             1/1     Running       2          30h
   nginxdemo1-6995675b4b-pnjnp             1/1     Running       2          30h
   nginxdemo1-6995675b4b-rsc4s             1/1     Terminating   0          30h
   nginxdemo1-6995675b4b-z7q72             1/1     Running       2          30h
   nginxpodsdemo                           1/1     Running       0          3h30m
   nginxpodsdemo1                          1/1     Running       0          68s
   nginxpodsdemo2                          1/1     Running       0          21s
   
   [root@k8snode1 k8s]# kubectl describe pods nginxpodsdemo2
   Name:         nginxpodsdemo2
   Namespace:    default
   Priority:     0
   Node:         k8snode2/192.168.10.186
   Start Time:   Tue, 20 Apr 2021 22:02:07 +0800
   Labels:       app=nginxpodsdemo2
                 version=1.7.9
   Annotations:  cni.projectcalico.org/podIP: 172.16.185.231/32
                 cni.projectcalico.org/podIPs: 172.16.185.231/32
                 poduse: pod demo the pod
   Status:       Running
   IP:           172.16.185.231
   IPs:
     IP:  172.16.185.231
   Containers:
     nginxpodsdemo2:
       Container ID:  docker://d2929ea10b16cc6b2c0637f08f5aecc77f92a7df629b5df5bec88d23a9a815a6
       Image:         nginx:1.7.9
       Image ID:      docker-pullable://nginx@sha256:e3456c851a152494c3e4ff5fcc26f240206abac0c9d794affb40e0714846c451
       Port:          80/TCP
       Host Port:     0/TCP
       Command:
         nginx
         -g
         daemon off;
       State:          Running
         Started:      Tue, 20 Apr 2021 22:02:08 +0800
       Ready:          True
       Restart Count:  0
       Environment:    <none>
       Mounts:
         /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-mrbv8 (ro)
   Conditions:
     Type              Status
     Initialized       True 
     Ready             True 
     ContainersReady   True 
     PodScheduled      True 
   Volumes:
     kube-api-access-mrbv8:
       Type:                    Projected (a volume that contains injected data from multiple sources)
       TokenExpirationSeconds:  3607
       ConfigMapName:           kube-root-ca.crt
       ConfigMapOptional:       <nil>
       DownwardAPI:             true
   QoS Class:                   BestEffort
   Node-Selectors:              <none>
   Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                                node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
   Events:
     Type    Reason     Age   From               Message
     ----    ------     ----  ----               -------
     Normal  Scheduled  46s   default-scheduler  Successfully assigned default/nginxpodsdemo2 to k8snode2
     Normal  Pulled     46s   kubelet            Container image "nginx:1.7.9" already present on machine
     Normal  Created    46s   kubelet            Created container nginxpodsdemo2
     Normal  Started    46s   kubelet            Started container nginxpodsdemo2
   
   ```

pod的生命周期：

1. 还未开始调度状态：pendding

2. 创建容器状态 :containerCreating

3. 运行中状态:running
4. 运行成功退出后状态：successed
5. 运行失败退出后状态：failed
6. 健康检查成功后的状态：Ready
7. 健康检查运行失败后的状态：CrashLoopBackOff
8. pod和apiservice通讯失败的状态：Unknown

多pod的关系例如：

1. pod-volume.yaml   一个pod里的多个容器里的host的文件是一致的 由pod管理的

   ```
   apiVersion: v1
   kind: Pod
   metadata:
     name: pod-volume
   spec:
     hostNetwork: true #使用宿主机的host网络
     hostPID: true   #使用宿主机的进程空间
     hostAliases:  #可以绑定host 也就是容器里的host文件
     - ip: "10.155.20.120"
       hostname:
       - "www.hostname.com"
     containers:
     - name: web
       image: hub.mooc.com/kubernetes/web:v1
       ports:
       - containerPort: 8080
       volumeMounts: #声明一个volumes
       - name: shared-volume # 名字就是下面定义的volumes
         mountPath: /shared-web  #挂载到自己容器的shared-web
     - name: dubbo
       env:
       - name: DUBBO_PORT
         value: "20881"
       image: hub.mooc.com/kubernetes/dubbo:v1
       ports:
       - containerPort: 20881
         hostPort: 20881
         protocol: TCP
       lifecycle:   #容器的生命周期 在启动 关闭之前执行一些命令
         postStart:
           exec:  #容器在执行了启动命令后 也会同时执行这个exec命令 是一个并行的过程
             command: ["/bin/sh","-c","echo web start... >> /var/log/message"]
         postStop:
           exec: #容器在关闭之前 先执行这个exec命令
             command: ["/bin/sh","-c","echo web start... >> /var/log/message"]
       volumeMounts:
       - name: shared-volume
         mountPath: /shared-dubbo
     volumes:  #设置volumes 后 所有的容器都可以这么用
     - name: shared-volume
       hostPath:
         path: /shared-volume-data
   
   ```

2. ProjectedVolume 主要有Secret ConfigMap DownLoadAPI

   * Secret：用来存储加密的数据 主要存储在etcd里 用户和apiservice交互的认证 

     ```
     #查看
     [root@k8shardway1 ~]# kubectl get secret
     NAME                  TYPE                                  DATA   AGE
     default-token-d4vmc   kubernetes.io/service-account-token   3      4d23h
     
     [root@k8shardway1 ~]# kubectl get secret default-token-d4vmc -o yaml
     apiVersion: v1
     data:
       ca.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUR4RENDQXF5Z0F3SUJBZ0lVZFVrbk1OR2gxcDhCaVNIdllhYTFMUzdhbVNjd0RRWUpLb1pJaHZjTkFRRUwKQlFBd2FERUxNQWtHQTFVRUJoTUNWVk14RHpBTkJnTlZCQWdUQms5eVpXZHZiakVSTUE4R0ExVUVCeE1JVUc5eQpkR3hoYm1ReEV6QVJCZ05WQkFvVENrdDFZbVZ5Ym1WMFpYTXhDekFKQmdOVkJBc1RBa05CTVJNd0VRWURWUVFECkV3cExkV0psY201bGRHVnpNQjRYRFRJeE1EVXdPREUwTVRnd01Gb1hEVEkyTURVd056RTBNVGd3TUZvd2FERUwKTUFrR0ExVUVCaE1DVlZNeER6QU5CZ05WQkFnVEJrOXlaV2R2YmpFUk1BOEdBMVVFQnhNSVVHOXlkR3hoYm1ReApFekFSQmdOVkJBb1RDa3QxWW1WeWJtVjBaWE14Q3pBSkJnTlZCQXNUQWtOQk1STXdFUVlEVlFRREV3cExkV0psCmNtNWxkR1Z6TUlJQklqQU5CZ2txaGtpRzl3MEJBUUVGQUFPQ0FROEFNSUlCQ2dLQ0FRRUFvL3hYSjcxQktjWlQKTkkvY0hXc0R3TUZOdkM1L1RQdm01Z1V4UUpWSG1FUXJaQ3krNDA3N3dWQ2xMaXhnaWx3U2M1NURwVmZVN3JuQgo3UDFweStSL1hhTFJLUHpDR0ZBK0FUa0dUMG0zVytQc1hUVDY2Ui9MMndVV3E0OVhpSzhWTVdqWmZETm1aTnBmCkRKWUNHdEFwZlBOTklIVFBBaVVRUlcwSE9NVS8zeTRMSGhMQ3JvdlZDcDZ3MjM1YjN0djJqWHlTb1RCMFNwZW4KVjhVOXlvWmF0Kyt0WXFIdFFjT05KeHV1MU1mS2tSTjVuYmJ4T1c0QkZOVUUwcHB3TFFaNUhDQ1JSbEpweFB5bwp1bzU0WDRmbVEvRHY2OWxra094SWkzZVd5cnZkYTJyQ3EvWHE4Z2tBSVpBZTg1Q2RxMStsQmVIL0NQUitLNEMxCjZOeDJiNEFqRXdJREFRQUJvMll3WkRBT0JnTlZIUThCQWY4RUJBTUNBUVl3RWdZRFZSMFRBUUgvQkFnd0JnRUIKL3dJQkFqQWRCZ05WSFE0RUZnUVVjWkZXdHJXZ2hnL2l5SS9UK0svaFVYUVJRVmt3SHdZRFZSMGpCQmd3Rm9BVQpjWkZXdHJXZ2hnL2l5SS9UK0svaFVYUVJRVmt3RFFZSktvWklodmNOQVFFTEJRQURnZ0VCQUFNZ1l5ZWtKaENICkxHMS81d3BjRHpoRlV4VUJDQk5DZVFMbmNDbndzWnA3M1lybEJJNW52aUl4QnZMQkJ5NkRVWWs0Zk1LZi9id1MKY0wwQnNrQzB2UG9IVm56cVFEbkUvUDJ5bGVRWjhTdTlEdXdFaHJubUxsY3N4LzlMM2s0UnNYeU8wNzFpVFl4dwpmMnpxZllTYWNzTitZTG5jM2RSalQ2b3l5SWhGc1IzWUs1Uk9KclMyOTFhOS81VkJZYkMyaUtCaGZDQ0FHQmZmCnNGa0xJMGJzWmxyWE5STU0zUUYwZlBkbUtTM2ptZGhTMkFuSEFnVFk3U0dsZjdTOUxDUjNBRlVYZVUvYlJaN1AKK3hyYmVCMlVKQkFqakJTVHVxWWkrK2IwMmNldlk0V3ljdjFLOW4vc2x6MWc1TUJXRkE1UFRlTGYxWmZTTmtUbgpuZnpFYkdHa1lpND0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
       namespace: ZGVmYXVsdA==
       token: ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNklsVk5UR2xKVnpWdWRFMHlaMUJwTVhaeE1VZDJSMlpOVjJKNFQydHRlV3h5ZFRaa2FVcEpjbTR5Ukc4aWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUprWldaaGRXeDBJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5elpXTnlaWFF1Ym1GdFpTSTZJbVJsWm1GMWJIUXRkRzlyWlc0dFpEUjJiV01pTENKcmRXSmxjbTVsZEdWekxtbHZMM05sY25acFkyVmhZMk52ZFc1MEwzTmxjblpwWTJVdFlXTmpiM1Z1ZEM1dVlXMWxJam9pWkdWbVlYVnNkQ0lzSW10MVltVnlibVYwWlhNdWFXOHZjMlZ5ZG1salpXRmpZMjkxYm5RdmMyVnlkbWxqWlMxaFkyTnZkVzUwTG5WcFpDSTZJakkyWVRSaFptTmxMVGRrWkdZdE5HTm1aaTFoT0RZM0xXRTJORGsxTkRrNFpUUmtOQ0lzSW5OMVlpSTZJbk41YzNSbGJUcHpaWEoyYVdObFlXTmpiM1Z1ZERwa1pXWmhkV3gwT21SbFptRjFiSFFpZlEuZy15REJZYkpjVWstSHNXTE9UOE41VTFYcWVORHlqc1BQeUs0cm9falpYUWNFTUU0bmlTQWxOcmpldE9mTlJxMmhFUUhOQ2FtdkV3aU8tbXR0S3htQkU5QXhGVUdpcGZYc0dEZ2ktRlBiSEZCRVNPU3V1VENwX1FuQ0cxMUF1UXczekhiWmxDSldQXzM3LWhCa1FvWS1WRVNHaTh2eW5HZ2tnM3NRS0ZzMjNaaTRkWUJtNnlTcndtR2RSbjg3ZjBnZFpFX1EyY2R3MzB6NUExZXA3akM3NExESUNaS2RfbFdJV3AyajRpYmpacjJKOW80ekdydmJuNV9CYmg1UVlMcHR2dTl6bXRENF9Ga0ljR0FsemQ3bHptdTNuemNFb0JMeXdUVjZ3QXc4RkNjWUlpOHhLY3dYaWV4bDFOZ2VrWUV4X2xxNF9JQnJvTmt2RlVVNzdLQ3Jn
     kind: Secret
     metadata:
       annotations:
         kubernetes.io/service-account.name: default
         kubernetes.io/service-account.uid: 26a4afce-7ddf-4cff-a867-a6495498e4d4
       creationTimestamp: "2021-05-08T15:36:51Z"
       managedFields:
       - apiVersion: v1
         fieldsType: FieldsV1
         fieldsV1:
           f:data:
             .: {}
             f:ca.crt: {}
             f:namespace: {}
             f:token: {}
           f:metadata:
             f:annotations:
               .: {}
               f:kubernetes.io/service-account.name: {}
               f:kubernetes.io/service-account.uid: {}
           f:type: {}
         manager: kube-controller-manager
         operation: Update
         time: "2021-05-08T15:36:51Z"
       name: default-token-d4vmc
       namespace: default
       resourceVersion: "314"
       uid: 2b8340fa-f5eb-489b-b7bf-7cbd85bfbf0f
     #查看这个会把一个命名空间的service-account-token自动加到每个容器里
     type: kubernetes.io/service-account-token
     
     [root@k8shardway1 ~]# kubectl get pod nginx-ds-99jwv -o yaml
     apiVersion: v1
     kind: Pod
     metadata:
       annotations:
         cni.projectcalico.org/podIP: 10.200.43.79/32
         cni.projectcalico.org/podIPs: 10.200.43.79/32
       creationTimestamp: "2021-05-09T07:48:55Z"
       generateName: nginx-ds-
       labels:
         app: nginx-ds
         controller-revision-hash: 7d58bbc569
         pod-template-generation: "1"
       managedFields:
       - apiVersion: v1
         fieldsType: FieldsV1
         fieldsV1:
           f:metadata:
             f:generateName: {}
             f:labels:
               .: {}
               f:app: {}
               f:controller-revision-hash: {}
               f:pod-template-generation: {}
             f:ownerReferences:
               .: {}
               k:{"uid":"28947b5f-b74a-429d-8110-a074ca103ce4"}:
                 .: {}
                 f:apiVersion: {}
                 f:blockOwnerDeletion: {}
                 f:controller: {}
                 f:kind: {}
                 f:name: {}
                 f:uid: {}
           f:spec:
             f:affinity:
               .: {}
               f:nodeAffinity:
                 .: {}
                 f:requiredDuringSchedulingIgnoredDuringExecution:
                   .: {}
                   f:nodeSelectorTerms: {}
             f:containers:
               k:{"name":"my-nginx"}:
                 .: {}
                 f:image: {}
                 f:imagePullPolicy: {}
                 f:name: {}
                 f:ports:
                   .: {}
                   k:{"containerPort":80,"protocol":"TCP"}:
                     .: {}
                     f:containerPort: {}
                     f:protocol: {}
                 f:resources: {}
                 f:terminationMessagePath: {}
                 f:terminationMessagePolicy: {}
             f:dnsPolicy: {}
             f:enableServiceLinks: {}
             f:restartPolicy: {}
             f:schedulerName: {}
             f:securityContext: {}
             f:terminationGracePeriodSeconds: {}
             f:tolerations: {}
         manager: kube-controller-manager
         operation: Update
         time: "2021-05-09T07:48:55Z"
       - apiVersion: v1
         fieldsType: FieldsV1
         fieldsV1:
           f:metadata:
             f:annotations:
               .: {}
               f:cni.projectcalico.org/podIP: {}
               f:cni.projectcalico.org/podIPs: {}
         manager: calico
         operation: Update
         time: "2021-05-09T07:48:59Z"
       - apiVersion: v1
         fieldsType: FieldsV1
         fieldsV1:
           f:status:
             f:conditions:
               k:{"type":"ContainersReady"}:
                 .: {}
                 f:lastProbeTime: {}
                 f:lastTransitionTime: {}
                 f:status: {}
                 f:type: {}
               k:{"type":"Initialized"}:
                 .: {}
                 f:lastProbeTime: {}
                 f:lastTransitionTime: {}
                 f:status: {}
                 f:type: {}
               k:{"type":"Ready"}:
                 .: {}
                 f:lastProbeTime: {}
                 f:lastTransitionTime: {}
                 f:status: {}
                 f:type: {}
             f:containerStatuses: {}
             f:hostIP: {}
             f:phase: {}
             f:podIP: {}
             f:podIPs:
               .: {}
               k:{"ip":"10.200.43.79"}:
                 .: {}
                 f:ip: {}
             f:startTime: {}
         manager: kubelet
         operation: Update
         time: "2021-05-13T15:00:14Z"
       name: nginx-ds-99jwv
       namespace: default
       ownerReferences:
       - apiVersion: apps/v1
         blockOwnerDeletion: true
         controller: true
         kind: DaemonSet
         name: nginx-ds
         uid: 28947b5f-b74a-429d-8110-a074ca103ce4
       resourceVersion: "514839"
       uid: 97037b25-f9c4-40b1-b1d6-a29e9cd755da
     spec:
       affinity:
         nodeAffinity:
           requiredDuringSchedulingIgnoredDuringExecution:
             nodeSelectorTerms:
             - matchFields:
               - key: metadata.name
                 operator: In
                 values:
                 - k8shardway2
       containers:
       - image: nginx:1.19
         imagePullPolicy: IfNotPresent
         name: my-nginx
         ports:
         - containerPort: 80
           protocol: TCP
         resources: {}
         terminationMessagePath: /dev/termination-log
         terminationMessagePolicy: File 
         volumeMounts:  #自动挂载
         - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
           name: default-token-d4vmc
           readOnly: true
       dnsPolicy: ClusterFirst
       enableServiceLinks: true
       nodeName: k8shardway2
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
       - effect: NoExecute
         key: node.kubernetes.io/unreachable
         operator: Exists
       - effect: NoSchedule
         key: node.kubernetes.io/disk-pressure
         operator: Exists
       - effect: NoSchedule
         key: node.kubernetes.io/memory-pressure
         operator: Exists
       - effect: NoSchedule
         key: node.kubernetes.io/pid-pressure
         operator: Exists
       - effect: NoSchedule
         key: node.kubernetes.io/unschedulable
         operator: Exists
       volumes:
       - name: default-token-d4vmc
         secret:
           defaultMode: 420
           secretName: default-token-d4vmc
     status:
       conditions:
       - lastProbeTime: null
         lastTransitionTime: "2021-05-09T07:48:59Z"
         status: "True"
         type: Initialized
       - lastProbeTime: null
         lastTransitionTime: "2021-05-13T15:00:09Z"
         status: "True"
         type: Ready
       - lastProbeTime: null
         lastTransitionTime: "2021-05-13T15:00:09Z"
         status: "True"
         type: ContainersReady
       - lastProbeTime: null
         lastTransitionTime: "2021-05-09T07:48:55Z"
         status: "True"
         type: PodScheduled
       containerStatuses:
       - containerID: containerd://f81cb6b4bfc6447c28d9ad7486ec6d335ddbb7b83061c00092919f23c6296b76
         image: docker.io/library/nginx:1.19
         imageID: docker.io/library/nginx@sha256:75a55d33ecc73c2a242450a9f1cc858499d468f077ea942867e662c247b5e412
         lastState:
           terminated:
             containerID: containerd://278fabbd2d56bf66aef6555c67a5a2aad23ce82c51f167d11243bdfc55a7a888
             exitCode: 255
             finishedAt: "2021-05-13T14:59:55Z"
             reason: Unknown
             startedAt: "2021-05-10T07:47:07Z"
         name: my-nginx
         ready: true
         restartCount: 2
         started: true
         state:
           running:
             startedAt: "2021-05-13T15:00:09Z"
       hostIP: 192.168.10.181
       phase: Running
       podIP: 10.200.43.79
       podIPs:
       - ip: 10.200.43.79
       qosClass: BestEffort
       startTime: "2021-05-09T07:48:59Z"
       
     #自定义secret 
     [root@k8shardway1 ~]# vim secret.yaml
     
     apiVersion: v1
     kind: Secret
     metadata:
       name: dbpass
     type: Opaque
     data:
       username: aW1vb2M= #用户名 imooc 
       passwd:  aW1vb2MxMjM= #密码 imooc123
     
     [root@k8shardway1 ~]# kubectl create -f secret.yaml 
     secret/dbpass created
     
     
     #创建一个带secret的pod
     [root@k8shardway1 ~]# vim pod-secret.yaml
     
     apiVersion: v1
     kind: Pod
     metadata:
       name: pod-secret
     spec:
       containers:
       - name: springboot-web
         image: nginx:1.7.9
         ports:
         - containerPort: 8080
         volumeMounts:  #挂载volumeMounts 
         - name: db-secret #名字就是下面随便起的那个
           mountPath: /db-secret  #就会挂载到容器里的这个目录下
           readOnly: true
       volumes:
       - name: db-secret  #这个名字就是随便起的
         projected:
           sources:
           - secret:
               name: dbpass  #使用上面的secret创建的secret
     
     ```

   * ConfigMap 和上面一样  主要用来存储不用加密的属性

     ```
     #创建一个配置文件
     [root@k8shardway1 ~]# vim game.properties
     
     enemies=aliens
     lives=3
     enemies.cheat=true
     enemies.cheat.level=noGoodRotten
     secret.code.passphrase=UUDDLRLRBABAS
     secret.code.allowed=true
     secret.code.lives=30
     
     #创建configmap
     [root@k8shardway1 ~]# kubectl create configmap web-game --from-file game.properties 
     configmap/web-game created
     
     #查看详情
     [root@k8shardway1 ~]# kubectl get cm web-game -o yaml
     apiVersion: v1
     data:
      #这个下面就是配置文件
       game.properties: |+
         enemies=aliens
         lives=3
         enemies.cheat=true
         enemies.cheat.level=noGoodRotten
         secret.code.passphrase=UUDDLRLRBABAS
         secret.code.allowed=true
         secret.code.lives=30
     
     kind: ConfigMap
     metadata:
       creationTimestamp: "2021-05-13T15:41:42Z"
       managedFields:
       - apiVersion: v1
         fieldsType: FieldsV1
         fieldsV1:
           f:data:
             .: {}
             f:game.properties: {}
         manager: kubectl-create
         operation: Update
         time: "2021-05-13T15:41:42Z"
       name: web-game
       namespace: default
       resourceVersion: "518827"
       uid: 0f68359f-b393-4ccc-b0b5-3dd0fcfb855c
     
     #在容器里使用
     [root@k8shardway1 ~]# vim pod-game.yaml
     
     apiVersion: v1
     kind: Pod
     metadata:
       name: pod-game
     spec:
       containers:
       - name: web
         image: hub.mooc.com/kubernetes/springboot-web:v1
         ports:
         - containerPort: 8080
         volumeMounts:
         - name: game #名字要和下面一样
           mountPath: /etc/config/game  #挂载到了这个目录下
           readOnly: true
       volumes:  #定义一个game
       - name: game #同样随便起
         configMap:
           name: web-game  #使用了上面的web-game
     
     
     
     #另外一种使用方式
     #创建一个configmap文件
     [root@k8shardway1 ~]# vim configmap.yaml
     
     apiVersion: v1
     kind: ConfigMap
     metadata:
       name: configs  #configmap的名字叫configs
     data:
       JAVA_OPTS: -Xms1024m
       LOG_LEVEL: DEBUG
     
     
     [root@k8shardway1 ~]# kubectl create -f configmap.yaml 
     configmap/configs created
     
     #通过环境变量的方式使用
     [root@k8shardway1 ~]# vim pod-env.yaml
     
     apiVersion: v1
     kind: Pod
     metadata:
       name: pod-env
     spec:
       containers:
       - name: web
         image: hub.mooc.com/kubernetes/springboot-web:v1
         ports:
         - containerPort: 8080
         env:  #pod的环境变量 这个值就是当前系统的环境变量 在java程序启动的时候也可以用
           - name: LOG_LEVEL_CONFIG
             valueFrom:
               configMapKeyRef: #值就是从configMapKeyRef来的 
                 name: configs   #configmap的名字叫configs
                 key: LOG_LEVEL  #取其中的一个key值 LOG_LEVEL
                 
                 
     #通过环境变量的方式使用
     [root@k8shardway1 ~]# vim pod-cmd.yaml
     
     apiVersion: v1
     kind: Pod
     metadata:
       name: pod-cmd
     spec:
       containers:
       - name: web
         image: hub.mooc.com/kubernetes/springboot-web:v1
         command: ["/bin/sh", "-c", "java -jar /springboot-web.jar -DJAVA_OPTS=$(JAVA_OPTS)"]
         ports:
         - containerPort: 8080
         env:
           - name: JAVA_OPTS  env:  #pod的环境变量 这个值就是当前系统的环境变量 在java程序启动的时候也可以用
             valueFrom:
               configMapKeyRef:
                 name: configs
                 key: JAVA_OPTS
     
     
     ```

   * downwardapi: 在程序中可以拿到pod容器本身的信息

     ```
     [root@k8shardway1 ~]# vim pod-downwardapi.yaml
     
     apiVersion: v1
     kind: Pod
     metadata:
       name: pod-downwardapi
       labels:
         app: downwardapi
         type: webapp
     spec:
       containers:
       - name: web
         image: hub.mooc.com/kubernetes/springboot-web:v1
         ports:
         - containerPort: 8080
         volumeMounts:
           - name: podinfo
             mountPath: /etc/podinfo  #挂载到这个下面
       volumes:  #定义了一个volumes 名字是 podinfo 来源是downwardAPI
         - name: podinfo
           projected:
             sources:
             - downwardAPI:
                 items:  #其中的项目是
                   - path: "labels" #目录 
                     fieldRef:
                       fieldPath: metadata.labels  #来源从哪里来
                   - path: "name"
                     fieldRef:
                       fieldPath: metadata.name
                   - path: "namespace"
                     fieldRef:
                       fieldPath: metadata.namespace 
                   - path: "memory-request"  #还可以拿到pod的资源信息 
                     resourceFieldRef:
                       containerName: web
                       resource: limits.memory
     
     
     #还可以拿到系统信息
     kubernetes 自从1.7开始，可以在pod 的container 内获取pod的spec,metadata 等信息。
     
     具体方法可以通过env获取：
     
           env:
             - name: MY_NODE_NAME
               valueFrom:
                 fieldRef:
                   fieldPath: spec.nodeName
             - name: MY_POD_NAME
               valueFrom:
                 fieldRef:
                   fieldPath: metadata.name
             - name: MY_POD_NAMESPACE
               valueFrom:
                 fieldRef:
                   fieldPath: metadata.namespace
             - name: MY_POD_IP
               valueFrom:
                 fieldRef:
                   fieldPath: status.podIP
             - name: MY_POD_SERVICE_ACCOUNT
               valueFrom:
                 fieldRef:
                   fieldPath: spec.serviceAccountName
     spec.nodeName ： pod所在节点的IP、宿主主机IP
     
     status.podIP ：pod IP
     
     metadata.namespace : pod 所在的namespace
     更多参数：https://kubernetes.io/docs/tasks/inject-data-application/environment-variable-expose-pod-information/
     
     
     ```
     
     

# ReplicaSet(RS)

副本集 位于pod的上层，用来管理pod的副本的扩容与收缩 相当于Deployment的一个小弟

在deployment下层 管理deployment的pod版本 在升级的时候 会新建一个rs，逐步改变新的rs的副本集 较少旧的 完成平滑升级 如果升级过程中新的版本出现问题，则会卡主不升级

主要靠rs的副本数来控制Deployment对应的版本的升级 也是操作rs的副本数

# Deployment

位于ReplicaSet的上层，用于管理rs和pod的更新和回滚。是滚动部署的

# NameSpace

主要是资源隔离，比如service之间隔离 

namespace是基于名字的隔离，不是基于ip的

```
# 查看namespace
[root@k8shardway1 ~]# kubectl get namespace

#查看namespace下所有的资源
[root@k8shardway1 ~]# kubectl get all -n nginx-ingress
NAME                                 READY   STATUS    RESTARTS   AGE
pod/nginx-ingress-747cd8c4db-d5thp   1/1     Running   0          46h
pod/nginx-ingress-747cd8c4db-pchjw   1/1     Running   0          47h

NAME                    TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
service/nginx-ingress   NodePort   10.233.115.7   <none>        80:30238/TCP,443:30151/TCP   46h

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-ingress   2/2     2            2           47h

NAME                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-ingress-747cd8c4db   2         2         2       47h

#查看容器的 dns解析的范围
nginx@nginx-ingress-747cd8c4db-d5thp:/$ cat /etc/resolv.conf 
search nginx-ingress.svc.cluster.local svc.cluster.local cluster.local
nameserver 169.254.25.10
options ndots:5

# 不通namespace之间 pod基于ip是可以访问的，但是services不能访问
```

# 容器重启策略

Pod中可以定义容器失败重启策略，默认是Always，通常包含三种策略：

* Always，总是重启，不管容器内部正常退出还是异常退出，⼀旦检测到容器进程运⾏异常，就会重启Pod中的所有容器；
* OnFailure，当检测到容器进程失败时，重启容器；
* Never，不重启，不管什么场景，都不重启，适⽤于批处理任务Jobs和CronJobs；

当容器内部进程出现异常的时候 对容器执行的一个策略

```
[root@k8snode1 k8s]# vim restartPolicy.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-restart-demo
  labels:
    run: pod-restart-demo
spec:
  containers:
  - name: pod-restart-demo
    image: busybox:latest
    imagePullPolicy: IfNotPresent
    command: ["cat"]
    args: ["/etc/passwd"]
  restartPolicy: OnFailure  #重启策略

[root@k8snode1 k8s]# kubectl apply -f restartPolicy.yaml 
pod/pod-restart-demo created

#查看pod运行完成
[root@k8snode1 k8s]# kubectl get pods
NAME                                    READY   STATUS        RESTARTS   AGE
nginx-dashboard-demo-5cd59c4f6d-bpq6h   1/1     Running       2          28h
nginx-dashboard-demo-5cd59c4f6d-dbc2d   1/1     Terminating   0          28h
nginx-dashboard-demo-5cd59c4f6d-j2c7l   1/1     Running       1          23h
nginx-dashboard-demo-5cd59c4f6d-jbzb8   1/1     Terminating   0          28h
nginx-dashboard-demo-5cd59c4f6d-w2n6d   1/1     Running       1          23h
nginxdemo                               1/1     Running       2          46h
nginxdemo1-6995675b4b-gtbhh             1/1     Running       1          23h
nginxdemo1-6995675b4b-jbfrt             1/1     Running       2          30h
nginxdemo1-6995675b4b-pnjnp             1/1     Running       2          30h
nginxdemo1-6995675b4b-rsc4s             1/1     Terminating   0          30h
nginxdemo1-6995675b4b-z7q72             1/1     Running       2          30h
nginxpodsdemo                           1/1     Running       0          3h46m
nginxpodsdemo1                          1/1     Running       0          16m
nginxpodsdemo2                          1/1     Running       0          15m
pod-restart-demo                        0/1     Completed     0          56s  #OnFailure 只有在报错的时候才会重启现在没有报错不会重启


#定义一个故意写错的容器策略
[root@k8snode1 k8s]# vim restartPolicy2.yaml 

apiVersion: v1
kind: Pod
metadata:
  name: pod-restart-demo2
  labels:
    run: pod-restart-demo2
spec:
  containers:
  - name: pod-restart-demo2
    image: busybox:latest
    imagePullPolicy: IfNotPresent
    command: ["cat"]
    args: ["/etc/passwdq"]  #故意写错命令
  restartPolicy: OnFailure

[root@k8snode1 k8s]# kubectl apply -f restartPolicy2.yaml 
pod/pod-restart-demo2 created

[root@k8snode1 k8s]# kubectl apply -f restartPolicy2.yaml 
pod/pod-restart-demo2 created
[root@k8snode1 k8s]# kubectl get pods
NAME                                    READY   STATUS             RESTARTS   AGE
nginx-dashboard-demo-5cd59c4f6d-bpq6h   1/1     Running            2          28h
nginx-dashboard-demo-5cd59c4f6d-dbc2d   1/1     Terminating        0          28h
nginx-dashboard-demo-5cd59c4f6d-j2c7l   1/1     Running            1          23h
nginx-dashboard-demo-5cd59c4f6d-jbzb8   1/1     Terminating        0          28h
nginx-dashboard-demo-5cd59c4f6d-w2n6d   1/1     Running            1          23h
nginxdemo                               1/1     Running            2          46h
nginxdemo1-6995675b4b-gtbhh             1/1     Running            1          23h
nginxdemo1-6995675b4b-jbfrt             1/1     Running            2          30h
nginxdemo1-6995675b4b-pnjnp             1/1     Running            2          30h
nginxdemo1-6995675b4b-rsc4s             1/1     Terminating        0          30h
nginxdemo1-6995675b4b-z7q72             1/1     Running            2          30h
nginxpodsdemo                           1/1     Running            0          3h51m
nginxpodsdemo1                          1/1     Running            0          22m
nginxpodsdemo2                          1/1     Running            0          21m
pod-restart-demo                        0/1     Completed          0          6m45s
pod-restart-demo1                       0/1     CrashLoopBackOff   5          3m46s
pod-restart-demo2                       0/1     CrashLoopBackOff   1          10s  #可以发现报错在一直重启

#定义一个不管怎么样都重启的
[root@k8snode1 k8s]# vim restartPolicy1.yaml 

apiVersion: v1
kind: Pod
metadata:
  name: pod-restart-demo1
  labels:
    run: pod-restart-demo1
spec:
  containers:
  - name: pod-restart-demo
    image: busybox:latest
    imagePullPolicy: IfNotPresent
    command: ["cat"]
    args: ["/etc/passwd"]
  restartPolicy: Always

[root@k8snode1 k8s]# kubectl apply -f restartPolicy1.yaml 
pod/pod-restart-demo1 created
[root@k8snode1 k8s]# kubectl get pods
NAME                                    READY   STATUS        RESTARTS   AGE
nginx-dashboard-demo-5cd59c4f6d-bpq6h   1/1     Running       2          28h
nginx-dashboard-demo-5cd59c4f6d-dbc2d   1/1     Terminating   0          28h
nginx-dashboard-demo-5cd59c4f6d-j2c7l   1/1     Running       1          23h
nginx-dashboard-demo-5cd59c4f6d-jbzb8   1/1     Terminating   0          28h
nginx-dashboard-demo-5cd59c4f6d-w2n6d   1/1     Running       1          23h
nginxdemo                               1/1     Running       2          46h
nginxdemo1-6995675b4b-gtbhh             1/1     Running       1          23h
nginxdemo1-6995675b4b-jbfrt             1/1     Running       2          30h
nginxdemo1-6995675b4b-pnjnp             1/1     Running       2          30h
nginxdemo1-6995675b4b-rsc4s             1/1     Terminating   0          30h
nginxdemo1-6995675b4b-z7q72             1/1     Running       2          30h
nginxpodsdemo                           1/1     Running       0          3h48m
nginxpodsdemo1                          1/1     Running       0          18m
nginxpodsdemo2                          1/1     Running       0          18m
pod-restart-demo                        0/1     Completed     0          3m3s
pod-restart-demo1                       0/1     Completed     2          17s   #可以看到一直在重启

[root@k8snode1 k8s]# kubectl describe pods pod-restart-demo1
Name:         pod-restart-demo1
Namespace:    default
Priority:     0
Node:         k8snode2/192.168.10.186
Start Time:   Tue, 20 Apr 2021 22:20:09 +0800
Labels:       run=pod-restart-demo1
Annotations:  cni.projectcalico.org/podIP: 172.16.185.233/32
              cni.projectcalico.org/podIPs: 172.16.185.233/32
Status:       Running
IP:           172.16.185.233
IPs:
  IP:  172.16.185.233
Containers:
  pod-restart-demo:
    Container ID:  docker://91baba893cfbd42c959b82cd652fc83dfcfcaac30e3e53ac4f51a92be733497d
    Image:         busybox:latest
    Image ID:      docker-pullable://busybox@sha256:ae39a6f5c07297d7ab64dbd4f82c77c874cc6a94cea29fdec309d0992574b4f7
    Port:          <none>
    Host Port:     <none>
    Command:
      cat
    Args:
      /etc/passwd
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Tue, 20 Apr 2021 22:20:26 +0800
      Finished:     Tue, 20 Apr 2021 22:20:26 +0800
    Ready:          False
    Restart Count:  2
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-9szd7 (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             False 
  ContainersReady   False 
  PodScheduled      True 
Volumes:
  kube-api-access-9szd7:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  40s                default-scheduler  Successfully assigned default/pod-restart-demo1 to k8snode2
  Normal   Pulled     24s (x3 over 40s)  kubelet            Container image "busybox:latest" already present on machine
  Normal   Created    24s (x3 over 40s)  kubelet            Created container pod-restart-demo
  Normal   Started    24s (x3 over 40s)  kubelet            Started container pod-restart-demo
  #查看log 有重启的策略 因为Always发现容器停止了，就会过一段时间就重启下
  Warning  BackOff    8s (x4 over 38s)   kubelet            Back-off restarting failed container


#最后删除俩失败的
[root@k8snode1 k8s]# kubectl delete pods pod-restart-demo1
pod "pod-restart-demo1" deleted

[root@k8snode1 k8s]# kubectl delete pods pod-restart-demo2
pod "pod-restart-demo2" deleted

```

# 运行多个容器

Pod中可以定义运⾏多个容器，不同容器之间使⽤列表的⽅式分割，多个容器内部共享相同的⽹络命名空间，存储命名空间。适⽤于容器之间具有相互配合的业务场景。

```
#定义俩容器的pod
[root@k8snode1 k8s]# vim pod-muti-container.yaml

apiVersion: v1
kind: Pod
metadata:
  name: pod-muti-container
  labels:
    run: pod-muti-container
spec:
  containers:
  - name: nginx
    image: nginx:1.7.9
    imagePullPolicy: IfNotPresent
    ports:
    - name: http-80
      containerPort: 80
      protocol: TCP
  - name: busybox
    image: busybox:latest
    imagePullPolicy: IfNotPresent
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello; sleep 1;done"]
   
   
[root@k8snode1 k8s]# kubectl apply -f pod-muti-container.yaml 
pod/pod-muti-container created
[root@k8snode1 k8s]# kubectl get pods
NAME                                    READY   STATUS        RESTARTS   AGE
nginx-dashboard-demo-5cd59c4f6d-bpq6h   1/1     Running       2          28h
nginx-dashboard-demo-5cd59c4f6d-dbc2d   1/1     Terminating   0          28h
nginx-dashboard-demo-5cd59c4f6d-j2c7l   1/1     Running       1          23h
nginx-dashboard-demo-5cd59c4f6d-jbzb8   1/1     Terminating   0          28h
nginx-dashboard-demo-5cd59c4f6d-w2n6d   1/1     Running       1          23h
nginxdemo                               1/1     Running       2          46h
nginxdemo1-6995675b4b-gtbhh             1/1     Running       1          23h
nginxdemo1-6995675b4b-jbfrt             1/1     Running       2          31h
nginxdemo1-6995675b4b-pnjnp             1/1     Running       2          31h
nginxdemo1-6995675b4b-rsc4s             1/1     Terminating   0          31h
nginxdemo1-6995675b4b-z7q72             1/1     Running       2          31h
nginxpodsdemo                           1/1     Running       0          3h56m
nginxpodsdemo1                          1/1     Running       0          27m
nginxpodsdemo2                          1/1     Running       0          26m
pod-muti-container                      2/2     Running       0          5s
pod-restart-demo                        0/1     Completed     0          11m

#多个容器查看日志 kubectl logs 资源名 容器名
[root@k8snode1 k8s]# kubectl logs pod-muti-container nginx
[root@k8snode1 k8s]# kubectl logs -f pod-muti-container busybox #也可以这样 相当于tail -f

[root@k8snode1 k8s]# kubectl describe pods pod-muti-container 
Name:         pod-muti-container
Namespace:    default
Priority:     0
Node:         k8snode2/192.168.10.186
Start Time:   Tue, 20 Apr 2021 22:28:15 +0800
Labels:       run=pod-muti-container
Annotations:  cni.projectcalico.org/podIP: 172.16.185.235/32
              cni.projectcalico.org/podIPs: 172.16.185.235/32
Status:       Running
IP:           172.16.185.235
IPs:
  IP:  172.16.185.235
Containers:
  nginx:
    Container ID:   docker://39ba2ad66c825211a31785eed83f2c4b218a97a07b1e7749728d6cded7e4a326
    Image:          nginx:1.7.9
    Image ID:       docker-pullable://nginx@sha256:e3456c851a152494c3e4ff5fcc26f240206abac0c9d794affb40e0714846c451
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Tue, 20 Apr 2021 22:28:16 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-d8whq (ro)
  busybox:
    Container ID:  docker://2d701f2f2ea5f682cc2dce8b46ceeeee73afb9cb533916617c3055470f1a3246
    Image:         busybox:latest
    Image ID:      docker-pullable://busybox@sha256:ae39a6f5c07297d7ab64dbd4f82c77c874cc6a94cea29fdec309d0992574b4f7
    Port:          <none>
    Host Port:     <none>
    Command:
      /bin/sh
    Args:
      -c
      while true; do echo hello; sleep 1;done
    State:          Running
      Started:      Tue, 20 Apr 2021 22:28:16 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-d8whq (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  kube-api-access-d8whq:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  2m15s  default-scheduler  Successfully assigned default/pod-muti-container to k8snode2
  Normal  Pulled     2m15s  kubelet            Container image "nginx:1.7.9" already present on machine
  Normal  Created    2m15s  kubelet            Created container nginx
  Normal  Started    2m15s  kubelet            Started container nginx
  Normal  Pulled     2m15s  kubelet            Container image "busybox:latest" already present on machine
  Normal  Created    2m15s  kubelet            Created container busybox
  Normal  Started    2m15s  kubelet            Started container busybox
 #可以查看有两个容器启动
```

# resources-Pod资源管理

参考文档：https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/

用来管理每个pod使用多少cpu 内存 磁盘等 本身使用的就是docker的隔离 机制

apiserver会收集每个node的资源，用来分配pod的时候寻找合适的node资源

Resources核心设计为：requests和limits

* requests ：表示容器需要的资源量
* limits：容器的最大的资源量 超过这个就会杀掉容器

当requests ==limits的时候 资源优先级最高

两个都不设置的话，最低优先级

limits > requests 的时候 正常资源 根据优先级来区分

通过资源管理实现容器之间的资源隔离，资源争抢和分配新的资源的时候可以落到可用的node上

docker的cpushares 是在发生资源竞争的时候，怎么分配这个资源

容器运⾏过程中需要分配所需的资源，如何与cggroup联动配合呢？答案是通过定义resource来实现资源的分配，资源的分配单位主要是cpu和memory，资源的定义分两种：requests和limits，requests表示请求资源，主要⽤于初始kubernetes调度pod时的依据，表示必须满⾜的分配资源;limits表示资源的限制，即pod不能超过limits定义的限制⼤⼩，超过则通过cggroup限制，pod中定义资源可以通过下⾯四个字段定义：

* spec.container[].resources.requests.cpu 请求cpu资源的⼤⼩，如0.1个cpu和100m表示分配1/10个cpu；
* spec.container[].resources.requests.memory 请求内存⼤⼩，单位可⽤M，Mi，G，Gi表示；
* spec.container[].resources.limits.cpu 限制cpu的⼤⼩，不能超过阀值，cggroup中限制的值；
* spec.container[].resources.limits.memory 限制内存的⼤⼩，不能超过阀值，超过会发⽣OOM；

1. 查看一下语法

   ```
   [root@k8snode1 ~]# kubectl explain pods.spec.containers.resources
   KIND:     Pod
   VERSION:  v1
   
   RESOURCE: resources <Object>
   
   DESCRIPTION:
        Compute Resources required by this container. Cannot be updated. More info:
        https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
   
        ResourceRequirements describes the compute resource requirements.
   
   FIELDS:
      limits	<map[string]string>
        Limits describes the maximum amount of compute resources allowed. More
        info:
        https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
   
      requests	<map[string]string>
        Requests describes the minimum amount of compute resources required. If
        Requests is omitted for a container, it defaults to Limits if that is
        explicitly specified, otherwise to an implementation-defined value. More
        info:
        https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
   
   ```

2. 定义一个资源限制的

   ```
   [root@k8snode1 k8s]# vim resourcedemo.yaml
   
   apiVersion: v1
   kind: Pod
   metadata:
     name: nginx-demo
     labels:
       name: nginx-demo
   spec:
     containers:
     - name: nginx-demo
       image: nginx:1.7.9
       imagePullPolicy: IfNotPresent
       ports:
       - name: nginx-port-80
         protocol: TCP
         containerPort: 80
       resources:
         requests:
           cpu: 0.25
           memory: 128Mi
         limits:
           cpu: 500m  #1核心=1000m 500m就是0.5核
           memory: 256Mi  #256MB的内存
   
   [root@k8snode1 k8s]# kubectl apply -f resourcedemo.yaml 
   pod/nginx-demo created
   
   [root@k8snode1 k8s]# kubectl describe pods nginx-demo
   Name:         nginx-demo
   Namespace:    default
   Priority:     0
   Node:         k8snode2/192.168.10.186
   Start Time:   Tue, 20 Apr 2021 22:59:02 +0800
   Labels:       name=nginx-demo
   Annotations:  cni.projectcalico.org/podIP: 172.16.185.236/32
                 cni.projectcalico.org/podIPs: 172.16.185.236/32
   Status:       Running
   IP:           172.16.185.236
   IPs:
     IP:  172.16.185.236
   Containers:
     nginx-demo:
       Container ID:   docker://42e05af5b0976f0e84cf7bac3b5747090a3df296fe6bce999de65a24b6677504
       Image:          nginx:1.7.9
       Image ID:       docker-pullable://nginx@sha256:e3456c851a152494c3e4ff5fcc26f240206abac0c9d794affb40e0714846c451
       Port:           80/TCP
       Host Port:      0/TCP
       State:          Running
         Started:      Tue, 20 Apr 2021 22:59:03 +0800
       Ready:          True
       Restart Count:  0
       #可以看到 已经被定义出来
       Limits:
         cpu:     500m
         memory:  256Mi
       Requests:
         cpu:        250m
         memory:     128Mi
       Environment:  <none>
       Mounts:
         /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-mbhp4 (ro)
   Conditions:
     Type              Status
     Initialized       True 
     Ready             True 
     ContainersReady   True 
     PodScheduled      True 
   Volumes:
     kube-api-access-mbhp4:
       Type:                    Projected (a volume that contains injected data from multiple sources)
       TokenExpirationSeconds:  3607
       ConfigMapName:           kube-root-ca.crt
       ConfigMapOptional:       <nil>
       DownwardAPI:             true
   QoS Class:                   Burstable
   Node-Selectors:              <none>
   Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                                node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
   Events:
     Type    Reason     Age   From               Message
     ----    ------     ----  ----               -------
     Normal  Scheduled  62s   default-scheduler  Successfully assigned default/nginx-demo to k8snode2
     Normal  Pulled     62s   kubelet            Container image "nginx:1.7.9" already present on machine
     Normal  Created    62s   kubelet            Created container nginx-demo
     Normal  Started    62s   kubelet            Started container nginx-demo
   
   ```

3. 资源分配原理

   Pod的资源如何分配呢？毫⽆疑问是从node上分配的，当我们创建⼀个pod的时候如果设置了requests，kubernetes的调度器kube-scheduler会执⾏两个调度过程：filter过滤和weight称重，kube-scheduler会根据请求的资源过滤，把符合条件的node筛选出来，然后再进⾏排序，把最满⾜运⾏pod的node筛选出来，然后再特定的node上运⾏pod。调度算法和细节可以参考下kubernetes调度算法介绍。如下是node-3节点资源的分配详情

   带资源约束的 Pod 如何运行

   当 kubelet 启动 Pod 中的 Container 时，它会将 CPU 和内存约束信息传递给容器运行时。

   当使用 Docker 时：

   - `spec.containers[].resources.requests.cpu` 先被转换为可能是小数的基础值，再乘以 1024。 这个数值和 2 的较大者用作 `docker run` 命令中的 [`--cpu-shares`](https://docs.docker.com/engine/reference/run/#/cpu-share-constraint) 标志的值。

   - `spec.containers[].resources.limits.cpu` 先被转换为 millicore 值，再乘以 100。 其结果就是每 100 毫秒内容器可以使用的 CPU 时间总量。在此期间（100ms），容器所使用的 CPU 时间不会超过它被分配的时间。

     > **说明：** 默认的配额（Quota）周期为 100 毫秒。CPU 配额的最小精度为 1 毫秒。

   - `spec.containers[].resources.limits.memory` 被转换为整数值，作为 `docker run` 命令中的 [`--memory`](https://docs.docker.com/engine/reference/run/#/user-memory-constraints) 参数值。

   如果 Container 超过其内存限制，则可能会被终止。如果容器可重新启动，则与所有其他类型的 运行时失效一样，kubelet 将重新启动容器。

   如果一个 Container 内存用量超过其内存请求值，那么当节点内存不足时，容器所处的 Pod 可能被逐出。

   每个 Container 可能被允许也可能不被允许使用超过其 CPU 约束的处理时间。 但是，容器不会由于 CPU 使用率过高而被杀死。

   要确定 Container 是否会由于资源约束而无法调度或被杀死，请参阅[疑难解答](https://kubernetes.io/zh/docs/concepts/configuration/manage-resources-containers/#troubleshooting) 部分。

   ```
   #查看pod落到哪个node
   [root@k8snode1 k8s]# kubectl get pods -o wide
   NAME                                    READY   STATUS        RESTARTS   AGE     IP               NODE       NOMINATED NODE   READINESS GATES
   nginx-dashboard-demo-5cd59c4f6d-bpq6h   1/1     Running       2          29h     172.16.185.220   k8snode2   <none>           <none>
   nginx-dashboard-demo-5cd59c4f6d-dbc2d   1/1     Terminating   0          29h     172.16.98.205    k8snode3   <none>           <none>
   nginx-dashboard-demo-5cd59c4f6d-j2c7l   1/1     Running       1          24h     172.16.185.228   k8snode2   <none>           <none>
   nginx-dashboard-demo-5cd59c4f6d-jbzb8   1/1     Terminating   0          29h     172.16.98.206    k8snode3   <none>           <none>
   nginx-dashboard-demo-5cd59c4f6d-w2n6d   1/1     Running       1          24h     172.16.185.227   k8snode2   <none>           <none>
   nginx-demo                              1/1     Running       0          5m11s   172.16.185.236   k8snode2   <none>           <none>
   nginxdemo                               1/1     Running       2          47h     172.16.185.221   k8snode2   <none>           <none>
   nginxdemo1-6995675b4b-gtbhh             1/1     Running       1          24h     172.16.185.225   k8snode2   <none>           <none>
   nginxdemo1-6995675b4b-jbfrt             1/1     Running       2          31h     172.16.185.223   k8snode2   <none>           <none>
   nginxdemo1-6995675b4b-pnjnp             1/1     Running       2          31h     172.16.185.224   k8snode2   <none>           <none>
   nginxdemo1-6995675b4b-rsc4s             1/1     Terminating   0          31h     172.16.98.201    k8snode3   <none>           <none>
   nginxdemo1-6995675b4b-z7q72             1/1     Running       2          31h     172.16.185.222   k8snode2   <none>           <none>
   nginxpodsdemo                           1/1     Running       0          4h32m   172.16.185.229   k8snode2   <none>           <none>
   nginxpodsdemo1                          1/1     Running       0          62m     172.16.185.230   k8snode2   <none>           <none>
   nginxpodsdemo2                          1/1     Running       0          62m     172.16.185.231   k8snode2   <none>           <none>
   pod-muti-container                      2/2     Running       0          35m     172.16.185.235   k8snode2   <none>           <none>
   pod-restart-demo                        0/1     Completed     0          47m     172.16.185.232   k8snode2   <none>           <none>
   
   #过滤出容器
   [root@k8snode2 ~]# docker container list| grep "nginx-demo"
   42e05af5b097        84581e99d807             "nginx -g 'daemon of…"   6 minutes ago       Up 6 minutes                            k8s_nginx-demo_nginx-demo_default_f2f50a94-9987-484c-8168-96e31bbef521_0
   35bf3241180d        k8s.gcr.io/pause:3.4.1   "/pause"                 6 minutes ago       Up 6 minutes                            k8s_POD_nginx-demo_default_f2f50a94-9987-484c-8168-96e31bbef521_0
   
   #查看容器详情
   [root@k8snode2 ~]# docker container inspect 42e05af5b097
   [
       {
           "Id": "42e05af5b0976f0e84cf7bac3b5747090a3df296fe6bce999de65a24b6677504",
           "Created": "2021-04-20T14:59:03.347954899Z",
           "Path": "nginx",
           "Args": [
               "-g",
               "daemon off;"
           ],
           "State": {
               "Status": "running",
               "Running": true,
               "Paused": false,
               "Restarting": false,
               "OOMKilled": false,
               "Dead": false,
               "Pid": 7209,
               "ExitCode": 0,
               "Error": "",
               "StartedAt": "2021-04-20T14:59:03.492987431Z",
               "FinishedAt": "0001-01-01T00:00:00Z"
           },
           "Image": "sha256:84581e99d807a703c9c03bd1a31cd9621815155ac72a7365fd02311264512656",
           "ResolvConfPath": "/var/lib/docker/containers/35bf3241180d1a678e50b1fb4ead67ac9bc8b5b269de84d34a2824db53d63ac4/resolv.conf",
           "HostnamePath": "/var/lib/docker/containers/35bf3241180d1a678e50b1fb4ead67ac9bc8b5b269de84d34a2824db53d63ac4/hostname",
           "HostsPath": "/var/lib/kubelet/pods/f2f50a94-9987-484c-8168-96e31bbef521/etc-hosts",
           "LogPath": "/var/lib/docker/containers/42e05af5b0976f0e84cf7bac3b5747090a3df296fe6bce999de65a24b6677504/42e05af5b0976f0e84cf7bac3b5747090a3df296fe6bce999de65a24b6677504-json.log",
           "Name": "/k8s_nginx-demo_nginx-demo_default_f2f50a94-9987-484c-8168-96e31bbef521_0",
           "RestartCount": 0,
           "Driver": "overlay2",
           "Platform": "linux",
           "MountLabel": "",
           "ProcessLabel": "",
           "AppArmorProfile": "",
           "ExecIDs": null,
           "HostConfig": {
               "Binds": [
                   "/var/lib/kubelet/pods/f2f50a94-9987-484c-8168-96e31bbef521/volumes/kubernetes.io~projected/kube-api-access-mbhp4:/var/run/secrets/kubernetes.io/serviceaccount:ro",
                   "/var/lib/kubelet/pods/f2f50a94-9987-484c-8168-96e31bbef521/etc-hosts:/etc/hosts",
                   "/var/lib/kubelet/pods/f2f50a94-9987-484c-8168-96e31bbef521/containers/nginx-demo/e004f8ac:/dev/termination-log"
               ],
               "ContainerIDFile": "",
               "LogConfig": {
                   "Type": "json-file",
                   "Config": {
                       "max-size": "100m"
                   }
               },
               "NetworkMode": "container:35bf3241180d1a678e50b1fb4ead67ac9bc8b5b269de84d34a2824db53d63ac4",
               "PortBindings": null,
               "RestartPolicy": {
                   "Name": "no",
                   "MaximumRetryCount": 0
               },
               "AutoRemove": false,
               "VolumeDriver": "",
               "VolumesFrom": null,
               "CapAdd": null,
               "CapDrop": null,
               "Capabilities": null,
               "Dns": null,
               "DnsOptions": null,
               "DnsSearch": null,
               "ExtraHosts": null,
               "GroupAdd": null,
               "IpcMode": "container:35bf3241180d1a678e50b1fb4ead67ac9bc8b5b269de84d34a2824db53d63ac4",
               "Cgroup": "",
               "Links": null,
               "OomScoreAdj": 967,
               "PidMode": "",
               "Privileged": false,
               "PublishAllPorts": false,
               "ReadonlyRootfs": false,
               "SecurityOpt": [
                   "seccomp=unconfined"
               ],
               "UTSMode": "",
               "UsernsMode": "",
               "ShmSize": 67108864,
               "Runtime": "runc",
               "ConsoleSize": [
                   0,
                   0
               ],
               "Isolation": "",
               "CpuShares": 256,
               "Memory": 268435456,  #可以看到内存限制
               "NanoCpus": 0,
               "CgroupParent": "kubepods-burstable-podf2f50a94_9987_484c_8168_96e31bbef521.slice",
               "BlkioWeight": 0,
               "BlkioWeightDevice": null,
               "BlkioDeviceReadBps": null,
               "BlkioDeviceWriteBps": null,
               "BlkioDeviceReadIOps": null,
               "BlkioDeviceWriteIOps": null,
               "CpuPeriod": 100000,
               "CpuQuota": 50000,
               "CpuRealtimePeriod": 0,
               "CpuRealtimeRuntime": 0,
               "CpusetCpus": "",
               "CpusetMems": "",
               "Devices": [],
               "DeviceCgroupRules": null,
               "DeviceRequests": null,
               "KernelMemory": 0,
               "KernelMemoryTCP": 0,
               "MemoryReservation": 0,
               "MemorySwap": 268435456,
               "MemorySwappiness": null,
               "OomKillDisable": false,
               "PidsLimit": null,
               "Ulimits": null,
               "CpuCount": 0,
               "CpuPercent": 0,
               "IOMaximumIOps": 0,
               "IOMaximumBandwidth": 0,
               "MaskedPaths": [
                   "/proc/acpi",
                   "/proc/kcore",
                   "/proc/keys",
                   "/proc/latency_stats",
                   "/proc/timer_list",
                   "/proc/timer_stats",
                   "/proc/sched_debug",
                   "/proc/scsi",
                   "/sys/firmware"
               ],
               "ReadonlyPaths": [
                   "/proc/asound",
                   "/proc/bus",
                   "/proc/fs",
                   "/proc/irq",
                   "/proc/sys",
                   "/proc/sysrq-trigger"
               ]
           },
           "GraphDriver": {
               "Data": {
                   "LowerDir": "/var/lib/docker/overlay2/6da8a0a17c7cd07463b7113d27dec526d9cf487c26e7cb40dc5a05f365c7bd4d-init/diff:/var/lib/docker/overlay2/e9919586c7449b294f7b17a2483ceb7c9c616581ceae2698788363cb21d50bef/diff:/var/lib/docker/overlay2/c5f40ecbb11380abf21b2c03e55de28dfbfca5162e0d84a0645ce2f72ee5c7a9/diff:/var/lib/docker/overlay2/9be332ff962daae0ac84d88443428a03bd62a306c374bb0b8394ecce040e2cf7/diff:/var/lib/docker/overlay2/42c47924b2560d0f5159b229b6128d7293d6afafeeba070fd245ab466152cdc1/diff:/var/lib/docker/overlay2/b48caba0ab00be6cdf2e27972d264d6e00f47665e748d59dd54f1d623d35df84/diff:/var/lib/docker/overlay2/27ba7bcdaf00da4bc03e358563f2b06e19fbb0f160e52364ad2c105f2d6f4002/diff:/var/lib/docker/overlay2/770b770b94dc1b7253b121c13ca6d1b12dc2ecbca30406e9a70213873b0ccce5/diff:/var/lib/docker/overlay2/4cdb54e0108d9a20936a32988f3822f6e6640943c16139f895cbd2c828046723/diff:/var/lib/docker/overlay2/8ac8a20c64bbce06c11883858d39ddb9b297f70212e7f46d109c571fd7f61ddb/diff:/var/lib/docker/overlay2/6eeb475ab7893f79112d665b0a56f05281cae67ddf68b4effcbc220c5665d18c/diff:/var/lib/docker/overlay2/f4a0aef8d764f67aaf6d26ea3996049d2c3c7f6894cc3ea322f1c460ccd32b23/diff:/var/lib/docker/overlay2/73e9aa1fd089d6762665dcd05f5c15dc347c728d44467597b7e95e6c9dd09d2a/diff:/var/lib/docker/overlay2/02715ac4c06a385507983a94ed2fddbb4edd3cbde6302e48defc139d128fb683/diff",
                   "MergedDir": "/var/lib/docker/overlay2/6da8a0a17c7cd07463b7113d27dec526d9cf487c26e7cb40dc5a05f365c7bd4d/merged",
                   "UpperDir": "/var/lib/docker/overlay2/6da8a0a17c7cd07463b7113d27dec526d9cf487c26e7cb40dc5a05f365c7bd4d/diff",
                   "WorkDir": "/var/lib/docker/overlay2/6da8a0a17c7cd07463b7113d27dec526d9cf487c26e7cb40dc5a05f365c7bd4d/work"
               },
               "Name": "overlay2"
           },
           "Mounts": [
               {
                   "Type": "bind",
                   "Source": "/var/lib/kubelet/pods/f2f50a94-9987-484c-8168-96e31bbef521/containers/nginx-demo/e004f8ac",
                   "Destination": "/dev/termination-log",
                   "Mode": "",
                   "RW": true,
                   "Propagation": "rprivate"
               },
               {
                   "Type": "volume",
                   "Name": "66d98c87b747a4e31c4aadf9195b18419bae8dab1cf2b07b5a3dfc75d7c9e4c9",
                   "Source": "/var/lib/docker/volumes/66d98c87b747a4e31c4aadf9195b18419bae8dab1cf2b07b5a3dfc75d7c9e4c9/_data",
                   "Destination": "/var/cache/nginx",
                   "Driver": "local",
                   "Mode": "",
                   "RW": true,
                   "Propagation": ""
               },
               {
                   "Type": "bind",
                   "Source": "/var/lib/kubelet/pods/f2f50a94-9987-484c-8168-96e31bbef521/volumes/kubernetes.io~projected/kube-api-access-mbhp4",
                   "Destination": "/var/run/secrets/kubernetes.io/serviceaccount",
                   "Mode": "ro",
                   "RW": false,
                   "Propagation": "rprivate"
               },
               {
                   "Type": "bind",
                   "Source": "/var/lib/kubelet/pods/f2f50a94-9987-484c-8168-96e31bbef521/etc-hosts",
                   "Destination": "/etc/hosts",
                   "Mode": "",
                   "RW": true,
                   "Propagation": "rprivate"
               }
           ],
           "Config": {
               "Hostname": "nginx-demo",
               "Domainname": "",
               "User": "0",
               "AttachStdin": false,
               "AttachStdout": false,
               "AttachStderr": false,
               "ExposedPorts": {
                   "443/tcp": {},
                   "80/tcp": {}
               },
               "Tty": false,
               "OpenStdin": false,
               "StdinOnce": false,
               "Env": [
                   "KUBERNETES_SERVICE_PORT=443",
                   "KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1",
                   "NGINXDEMO1_SERVICE_PORT=80",
                   "KUBERNETES_SERVICE_PORT_HTTPS=443",
                   "KUBERNETES_PORT=tcp://10.96.0.1:443",
                   "KUBERNETES_PORT_443_TCP_PROTO=tcp",
                   "NGINXDEMO1_PORT_80_TCP_PROTO=tcp",
                   "NGINXDEMO1_PORT_80_TCP_PORT=80",
                   "KUBERNETES_SERVICE_HOST=10.96.0.1",
                   "NGINXDEMO1_SERVICE_HOST=10.105.52.111",
                   "NGINXDEMO1_PORT=tcp://10.105.52.111:80",
                   "NGINXDEMO1_PORT_80_TCP=tcp://10.105.52.111:80",
                   "NGINXDEMO1_PORT_80_TCP_ADDR=10.105.52.111",
                   "KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443",
                   "KUBERNETES_PORT_443_TCP_PORT=443",
                   "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                   "NGINX_VERSION=1.7.9-1~wheezy"
               ],
               "Cmd": [
                   "nginx",
                   "-g",
                   "daemon off;"
               ],
               "Healthcheck": {
                   "Test": [
                       "NONE"
                   ]
               },
               "Image": "sha256:84581e99d807a703c9c03bd1a31cd9621815155ac72a7365fd02311264512656",
               "Volumes": {
                   "/var/cache/nginx": {}
               },
               "WorkingDir": "",
               "Entrypoint": null,
               "OnBuild": null,
               "Labels": {
                   "annotation.io.kubernetes.container.hash": "3eba6278",
                   "annotation.io.kubernetes.container.ports": "[{\"name\":\"nginx-port-80\",\"containerPort\":80,\"protocol\":\"TCP\"}]",
                   "annotation.io.kubernetes.container.restartCount": "0",
                   "annotation.io.kubernetes.container.terminationMessagePath": "/dev/termination-log",
                   "annotation.io.kubernetes.container.terminationMessagePolicy": "File",
                   "annotation.io.kubernetes.pod.terminationGracePeriod": "30",
                   "io.kubernetes.container.logpath": "/var/log/pods/default_nginx-demo_f2f50a94-9987-484c-8168-96e31bbef521/nginx-demo/0.log",
                   "io.kubernetes.container.name": "nginx-demo",
                   "io.kubernetes.docker.type": "container",
                   "io.kubernetes.pod.name": "nginx-demo",
                   "io.kubernetes.pod.namespace": "default",
                   "io.kubernetes.pod.uid": "f2f50a94-9987-484c-8168-96e31bbef521",
                   "io.kubernetes.sandbox.id": "35bf3241180d1a678e50b1fb4ead67ac9bc8b5b269de84d34a2824db53d63ac4"
               }
           },
           "NetworkSettings": {
               "Bridge": "",
               "SandboxID": "",
               "HairpinMode": false,
               "LinkLocalIPv6Address": "",
               "LinkLocalIPv6PrefixLen": 0,
               "Ports": {},
               "SandboxKey": "",
               "SecondaryIPAddresses": null,
               "SecondaryIPv6Addresses": null,
               "EndpointID": "",
               "Gateway": "",
               "GlobalIPv6Address": "",
               "GlobalIPv6PrefixLen": 0,
               "IPAddress": "",
               "IPPrefixLen": 0,
               "IPv6Gateway": "",
               "MacAddress": "",
               "Networks": {}
           }
       }
   ]
   
   
   #查看node资源
   [root@k8snode1 ~]# kubectl describe node k8snode3
   Name:               k8snode3
   Roles:              <none>
   Labels:             beta.kubernetes.io/arch=amd64
                       beta.kubernetes.io/os=linux
                       kubernetes.io/arch=amd64
                       kubernetes.io/hostname=k8snode3
                       kubernetes.io/os=linux
   Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: /var/run/dockershim.sock
                       node.alpha.kubernetes.io/ttl: 0
                       projectcalico.org/IPv4Address: 192.168.10.187/24
                       projectcalico.org/IPv4IPIPTunnelAddr: 172.16.98.192
                       volumes.kubernetes.io/controller-managed-attach-detach: true
   CreationTimestamp:  Sat, 17 Apr 2021 23:01:06 +0800
   Taints:             node.kubernetes.io/unreachable:NoExecute
                       node.kubernetes.io/unreachable:NoSchedule
   Unschedulable:      false
   Lease:
     HolderIdentity:  k8snode3
     AcquireTime:     <unset>
     RenewTime:       Mon, 19 Apr 2021 21:09:28 +0800
   Conditions:
     Type                 Status    LastHeartbeatTime                 LastTransitionTime                Reason              Message
     ----                 ------    -----------------                 ------------------                ------              -------
     NetworkUnavailable   False     Sat, 17 Apr 2021 23:55:36 +0800   Sat, 17 Apr 2021 23:55:36 +0800   CalicoIsUp          Calico is running on this node
     MemoryPressure       Unknown   Mon, 19 Apr 2021 21:06:24 +0800   Mon, 19 Apr 2021 22:32:11 +0800   NodeStatusUnknown   Kubelet stopped posting node status.
     DiskPressure         Unknown   Mon, 19 Apr 2021 21:06:24 +0800   Mon, 19 Apr 2021 22:32:11 +0800   NodeStatusUnknown   Kubelet stopped posting node status.
     PIDPressure          Unknown   Mon, 19 Apr 2021 21:06:24 +0800   Mon, 19 Apr 2021 22:32:11 +0800   NodeStatusUnknown   Kubelet stopped posting node status.
     Ready                Unknown   Mon, 19 Apr 2021 21:06:24 +0800   Mon, 19 Apr 2021 22:32:11 +0800   NodeStatusUnknown   Kubelet stopped posting node status.
   Addresses:
     InternalIP:  192.168.10.187
     Hostname:    k8snode3
   Capacity:
     cpu:                4
     ephemeral-storage:  79132928Ki
     hugepages-1Gi:      0
     hugepages-2Mi:      0
     memory:             3879604Ki
     pods:               110
   Allocatable:
     cpu:                4
     ephemeral-storage:  72928906325
     hugepages-1Gi:      0
     hugepages-2Mi:      0
     memory:             3777204Ki
     pods:               110
   System Info:
     Machine ID:                 7de771b3710f4ef69422414077898d01
     System UUID:                01644D56-7B25-BBE3-EB16-14796E87DF02
     Boot ID:                    503ccc79-ca31-4616-a52f-a57b6750452d
     Kernel Version:             3.10.0-1160.15.2.el7.x86_64
     OS Image:                   CentOS Linux 7 (Core)
     Operating System:           linux
     Architecture:               amd64
     Container Runtime Version:  docker://19.3.9
     Kubelet Version:            v1.21.0
     Kube-Proxy Version:         v1.21.0
   PodCIDR:                      172.16.2.0/24
   PodCIDRs:                     172.16.2.0/24
   Non-terminated Pods:          (11 in total)
     Namespace                   Name                                          CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
     ---------                   ----                                          ------------  ----------  ---------------  -------------  ---
     default                     cpu-demo                                      0 (0%)        0 (0%)      100Mi (2%)       200Mi (5%)     4m9s
     default                     nginx-dashboard-demo-5cd59c4f6d-dbc2d         0 (0%)        0 (0%)      0 (0%)           0 (0%)         30h
     default                     nginx-dashboard-demo-5cd59c4f6d-jbzb8         0 (0%)        0 (0%)      0 (0%)           0 (0%)         30h
     default                     nginxdemo1-6995675b4b-rsc4s                   0 (0%)        0 (0%)      0 (0%)           0 (0%)         32h
     kube-system                 calico-kube-controllers-6d8ccdbf46-s6qdx      0 (0%)        0 (0%)      0 (0%)           0 (0%)         2d23h
     kube-system                 calico-node-249tw                             250m (6%)     0 (0%)      0 (0%)           0 (0%)         2d23h
     kube-system                 coredns-558bd4d5db-56pr7                      100m (2%)     0 (0%)      70Mi (1%)        170Mi (4%)     3d
     kube-system                 coredns-558bd4d5db-wbgr6                      100m (2%)     0 (0%)      70Mi (1%)        170Mi (4%)     3d
     kube-system                 kube-proxy-9tvkf                              0 (0%)        0 (0%)      0 (0%)           0 (0%)         3d
     kubernetes-dashboard        dashboard-metrics-scraper-5594697f48-xj59f    0 (0%)        0 (0%)      0 (0%)           0 (0%)         30h
     tigera-operator             tigera-operator-675ccbb69c-ptdmm              0 (0%)        0 (0%)      0 (0%)           0 (0%)         3d
   Allocated resources:
     (Total limits may be over 100 percent, i.e., overcommitted.)
     Resource           Requests    Limits
     --------           --------    ------
     cpu                450m (11%)  0 (0%)
     memory             240Mi (6%)  540Mi (14%)
     ephemeral-storage  0 (0%)      0 (0%)
     hugepages-1Gi      0 (0%)      0 (0%)
     hugepages-2Mi      0 (0%)      0 (0%)
   Events:              <none>
   
   
   ```

   pod资源压力测试参考：https://kubernetes.io/zh/docs/tasks/configure-pod-container/assign-cpu-resource/

   ```
   # 创建一个压测镜像
   [root@k8snode1 k8s]# vim cpu-request-limit.yaml
   
   apiVersion: v1
   kind: Pod
   metadata:
     name: cpu-demo
   spec:
     containers:
     - name: cpu-demo-ctr
       image: vish/stress
       resources:
         limits:
           cpu: "1"
         requests:
           cpu: "0.5"
       args:
       - -cpus
       - "2"
   
   [root@k8snode1 k8s]# kubectl apply -f cpu-request-limit.yaml 
   pod/cpu-demo created
   
   [root@k8snode1 k8s]# kubectl get pods -o wide
   NAME                                    READY   STATUS              RESTARTS   AGE     IP               NODE       NOMINATED NODE   READINESS GATES
   cpu-demo                                0/1     ContainerCreating   0          21s     <none>           k8snode2   <none>           <none>
   nginx-dashboard-demo-5cd59c4f6d-bpq6h   1/1     Running             2          29h     172.16.185.220   k8snode2   <none>           <none>
   nginx-dashboard-demo-5cd59c4f6d-dbc2d   1/1     Terminating         0          29h     172.16.98.205    k8snode3   <none>           <none>
   nginx-dashboard-demo-5cd59c4f6d-j2c7l   1/1     Running             1          24h     172.16.185.228   k8snode2   <none>           <none>
   nginx-dashboard-demo-5cd59c4f6d-jbzb8   1/1     Terminating         0          29h     172.16.98.206    k8snode3   <none>           <none>
   nginx-dashboard-demo-5cd59c4f6d-w2n6d   1/1     Running             1          24h     172.16.185.227   k8snode2   <none>           <none>
   nginx-demo                              1/1     Running             0          14m     172.16.185.236   k8snode2   <none>           <none>
   nginxdemo                               1/1     Running             2          47h     172.16.185.221   k8snode2   <none>           <none>
   nginxdemo1-6995675b4b-gtbhh             1/1     Running             1          24h     172.16.185.225   k8snode2   <none>           <none>
   nginxdemo1-6995675b4b-jbfrt             1/1     Running             2          31h     172.16.185.223   k8snode2   <none>           <none>
   nginxdemo1-6995675b4b-pnjnp             1/1     Running             2          31h     172.16.185.224   k8snode2   <none>           <none>
   nginxdemo1-6995675b4b-rsc4s             1/1     Terminating         0          31h     172.16.98.201    k8snode3   <none>           <none>
   nginxdemo1-6995675b4b-z7q72             1/1     Running             2          31h     172.16.185.222   k8snode2   <none>           <none>
   nginxpodsdemo                           1/1     Running             0          4h41m   172.16.185.229   k8snode2   <none>           <none>
   nginxpodsdemo1                          1/1     Running             0          72m     172.16.185.230   k8snode2   <none>           <none>
   nginxpodsdemo2                          1/1     Running             0          71m     172.16.185.231   k8snode2   <none>           <none>
   pod-muti-container                      2/2     Running             0          45m     172.16.185.235   k8snode2   <none>           <none>
   pod-restart-demo                        0/1     Completed           0          56m     172.16.185.232   k8snode2   <none>           <none>
   
   #node2 用top查看 cpu确实占用了100% 如果不限制cpu，会占用node节点剩余的所有cpu资源
   
   
   #内存压力测试 如果超过内存 就会发送OOMKill 通过查看日志可以看到
   [root@k8snode1 k8s]# vim menory-request-limit.yaml 
   
   apiVersion: v1
   kind: Pod
   metadata:
     name: cpu-demo
   spec:
     containers:
     - name: cpu-demo-ctr
       image: vish/stress
       resources:
         limits:
           memory: "200Mi"
         requests:
           memory: "100Mi"
       command: ["stress"]
       args: ["--vm", "1", "--vm-bytes", "256M", "--vm-hang", "1"]
     nodeName: k8snode3
   
   
   [root@k8snode1 k8s]# kubectl apply -f menory-request-limit.yaml 
   pod/cpu-demo created
   
   
   
   ```

# limitrang

限制容器资源的比值，防止任意区分

```
[root@k8shardway1 kubernetes]# vim linits-test.yaml

apiVersion: v1
kind: LimitRange
metadata:
  name: test-limits
spec:
  limits: 
  - max:  #限制最大cpu和内存
      cpu: 4000m
      memory: 2Gi
    min:  #最小cpu和内存
      cpu: 100m
      memory: 100Mi
    maxLimitRequestRatio: #Request和Limit的最大比值不能超过多少
      cpu: 3  #cpu比值为3 就是cpu的Limit最大可以大于Request3倍
      memory: 2
    type: Pod  #对类型是 pod进行限制
  - default:   #容器的限制多了个默认值，因为一个pod可能有多个容器当服务没有配置的时候 会默认使用则
      cpu: 300m
      memory: 200Mi
    defaultRequest:
      cpu: 200m
      memory: 100Mi
    max:
      cpu: 2000m
      memory: 1Gi
    min:
      cpu: 100m
      memory: 100Mi
    maxLimitRequestRatio:
      cpu: 5
      memory: 4
    type: Container   #容器的限制 同上

#创建
[root@k8shardway1 kubernetes]# kubectl create -f linits-test.yaml 
limitrange/test-limits created

#查看创建的limits详情 可以看到pod的cpu最大和最小的限制 Container的cpu和内存最大最小的限制  
[root@k8shardway1 kubernetes]# kubectl describe limits
Name:       test-limits
Namespace:  default
Type        Resource  Min    Max  Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---    ---  ---------------  -------------  -----------------------
Pod         cpu       100m   4    -                -              3
Pod         memory    100Mi  2Gi  -                -              2
Container   cpu       100m   2    200m             300m           5
Container   memory    100Mi  1Gi  100Mi            200Mi          4


#后面创建的资源就是基于这个模板的 创建一个试试
[root@k8shardway1 kubernetes]# vim nginx-text.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-demo
  namespace: default
spec:
  selector:
    matchLabels:
      app: web-demo
  replicas: 1
  template:
    metadata:
      labels:
        app: web-demo
    spec:
      containers:
      - name: web-demo
        image: nginx:1.7.9
        ports:
        - containerPort: 8080

[root@k8shardway1 kubernetes]# kubectl apply -f nginx-text.yaml 
deployment.apps/web-demo created

#查看deployment详情 发现resources为空
[root@k8shardway1 kubernetes]# kubectl get deployment web-demo -o yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"name":"web-demo","namespace":"default"},"spec":{"replicas":1,"selector":{"matchLabels":{"app":"web-demo"}},"template":{"metadata":{"labels":{"app":"web-demo"}},"spec":{"containers":[{"image":"nginx:1.7.9","name":"web-demo","ports":[{"containerPort":8080}]}]}}}}
  creationTimestamp: "2021-05-12T14:27:26Z"
  generation: 1
  managedFields:
  - apiVersion: apps/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:kubectl.kubernetes.io/last-applied-configuration: {}
      f:spec:
        f:progressDeadlineSeconds: {}
        f:replicas: {}
        f:revisionHistoryLimit: {}
        f:selector: {}
        f:strategy:
          f:rollingUpdate:
            .: {}
            f:maxSurge: {}
            f:maxUnavailable: {}
          f:type: {}
        f:template:
          f:metadata:
            f:labels:
              .: {}
              f:app: {}
          f:spec:
            f:containers:
              k:{"name":"web-demo"}:
                .: {}
                f:image: {}
                f:imagePullPolicy: {}
                f:name: {}
                f:ports:
                  .: {}
                  k:{"containerPort":8080,"protocol":"TCP"}:
                    .: {}
                    f:containerPort: {}
                    f:protocol: {}
                f:resources: {}
                f:terminationMessagePath: {}
                f:terminationMessagePolicy: {}
            f:dnsPolicy: {}
            f:restartPolicy: {}
            f:schedulerName: {}
            f:securityContext: {}
            f:terminationGracePeriodSeconds: {}
    manager: kubectl-client-side-apply
    operation: Update
    time: "2021-05-12T14:27:26Z"
  - apiVersion: apps/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          f:deployment.kubernetes.io/revision: {}
      f:status:
        f:availableReplicas: {}
        f:conditions:
          .: {}
          k:{"type":"Available"}:
            .: {}
            f:lastTransitionTime: {}
            f:lastUpdateTime: {}
            f:message: {}
            f:reason: {}
            f:status: {}
            f:type: {}
          k:{"type":"Progressing"}:
            .: {}
            f:lastTransitionTime: {}
            f:lastUpdateTime: {}
            f:message: {}
            f:reason: {}
            f:status: {}
            f:type: {}
        f:observedGeneration: {}
        f:readyReplicas: {}
        f:replicas: {}
        f:updatedReplicas: {}
    manager: kube-controller-manager
    operation: Update
    time: "2021-05-12T14:27:28Z"
  name: web-demo
  namespace: default
  resourceVersion: "380619"
  uid: 28e96c67-bdd2-40a7-9c84-e46e7db89bd7
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: web-demo
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: web-demo
    spec:
      containers:
      - image: nginx:1.7.9
        imagePullPolicy: IfNotPresent
        name: web-demo
        ports:
        - containerPort: 8080
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
  availableReplicas: 1
  conditions:
  - lastTransitionTime: "2021-05-12T14:27:28Z"
    lastUpdateTime: "2021-05-12T14:27:28Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: "2021-05-12T14:27:27Z"
    lastUpdateTime: "2021-05-12T14:27:28Z"
    message: ReplicaSet "web-demo-664db7f84" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  observedGeneration: 1
  readyReplicas: 1
  replicas: 1
  updatedReplicas: 1


#查看对应的pod 发现resources限制是附加到pod上的
[root@k8shardway1 kubernetes]# kubectl get pods web-demo-664db7f84-ftlct -o yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    cni.projectcalico.org/podIP: 10.200.43.75/32
    cni.projectcalico.org/podIPs: 10.200.43.75/32
    kubernetes.io/limit-ranger: 'LimitRanger plugin set: cpu, memory request for container
      web-demo; cpu, memory limit for container web-demo'
  creationTimestamp: "2021-05-12T14:27:27Z"
  generateName: web-demo-664db7f84-
  labels:
    app: web-demo
    pod-template-hash: 664db7f84
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          f:cni.projectcalico.org/podIP: {}
          f:cni.projectcalico.org/podIPs: {}
    manager: calico
    operation: Update
    time: "2021-05-12T14:27:27Z"
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:generateName: {}
        f:labels:
          .: {}
          f:app: {}
          f:pod-template-hash: {}
        f:ownerReferences:
          .: {}
          k:{"uid":"033f1c81-ce4b-4b24-9801-4811618d21b6"}:
            .: {}
            f:apiVersion: {}
            f:blockOwnerDeletion: {}
            f:controller: {}
            f:kind: {}
            f:name: {}
            f:uid: {}
      f:spec:
        f:containers:
          k:{"name":"web-demo"}:
            .: {}
            f:image: {}
            f:imagePullPolicy: {}
            f:name: {}
            f:ports:
              .: {}
              k:{"containerPort":8080,"protocol":"TCP"}:
                .: {}
                f:containerPort: {}
                f:protocol: {}
            f:resources:
              .: {}
              f:limits:
                f:cpu: {}
                f:memory: {}
              f:requests:
                f:cpu: {}
                f:memory: {}
            f:terminationMessagePath: {}
            f:terminationMessagePolicy: {}
        f:dnsPolicy: {}
        f:enableServiceLinks: {}
        f:restartPolicy: {}
        f:schedulerName: {}
        f:securityContext: {}
        f:terminationGracePeriodSeconds: {}
    manager: kube-controller-manager
    operation: Update
    time: "2021-05-12T14:27:27Z"
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:status:
        f:conditions:
          k:{"type":"ContainersReady"}:
            .: {}
            f:lastProbeTime: {}
            f:lastTransitionTime: {}
            f:status: {}
            f:type: {}
          k:{"type":"Initialized"}:
            .: {}
            f:lastProbeTime: {}
            f:lastTransitionTime: {}
            f:status: {}
            f:type: {}
          k:{"type":"Ready"}:
            .: {}
            f:lastProbeTime: {}
            f:lastTransitionTime: {}
            f:status: {}
            f:type: {}
        f:containerStatuses: {}
        f:hostIP: {}
        f:phase: {}
        f:podIP: {}
        f:podIPs:
          .: {}
          k:{"ip":"10.200.43.75"}:
            .: {}
            f:ip: {}
        f:startTime: {}
    manager: kubelet
    operation: Update
    time: "2021-05-12T14:27:28Z"
  name: web-demo-664db7f84-ftlct
  namespace: default
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: web-demo-664db7f84
    uid: 033f1c81-ce4b-4b24-9801-4811618d21b6
  resourceVersion: "380617"
  uid: ac6c19d3-9d98-4502-bf1d-d88ad21fa70f
spec:
  containers:
  - image: nginx:1.7.9
    imagePullPolicy: IfNotPresent
    name: web-demo
    ports:
    - containerPort: 8080
      protocol: TCP
    resources:
      limits:
        cpu: 300m
        memory: 200Mi
      requests:
        cpu: 200m
        memory: 100Mi
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-d4vmc
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: k8shardway2
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
  - name: default-token-d4vmc
    secret:
      defaultMode: 420
      secretName: default-token-d4vmc
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2021-05-12T14:27:27Z"
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2021-05-12T14:27:28Z"
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2021-05-12T14:27:28Z"
    status: "True"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2021-05-12T14:27:26Z"
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: containerd://77bb1b4391b25308a0cccd34b6ed69af33d1110a0d92aafff1700af4f07e5726
    image: docker.io/library/nginx:1.7.9
    imageID: sha256:35d28df486f6150fa3174367499d1eb01f22f5a410afe4b9581ac0e0e58b3eaf
    lastState: {}
    name: web-demo
    ready: true
    restartCount: 0
    started: true
    state:
      running:
        startedAt: "2021-05-12T14:27:28Z"
  hostIP: 192.168.10.181
  phase: Running
  podIP: 10.200.43.75
  podIPs:
  - ip: 10.200.43.75
  qosClass: Burstable
  startTime: "2021-05-12T14:27:27Z"

#如果改成不合比例的 则pod是没法调度的 最终导致会创建失败 可以通过查看详情查看错误信息
[root@k8shardway1 kubernetes]# kubectl get deployment web-demo -o yaml
```

# ResourceQuota

对namespace的资源的限制

```
[root@k8shardway1 kubernetes]# vim compute-resource.yaml

apiVersion: v1
kind: ResourceQuota
metadata:
  name: resource-quota
spec:
  hard:
    pods: 4   # pod最多4个
    requests.cpu: 2000m  #cpu最小
    requests.memory: 4Gi  #内存最小
    limits.cpu: 4000m  #cpu最大
    limits.memory: 8Gi #内存最大

[root@k8shardway1 kubernetes]# kubectl apply -f compute-resource.yaml 
resourcequota/resource-quota created

#还有其他资源限制
[root@k8shardway1 kubernetes]# vim object-count.yaml

apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-counts
spec:
  hard:
    configmaps: 10  #最大的configmaps
    persistentvolumeclaims: 4  #最大4个pvc
    replicationcontrollers: 20  #最大4个replicationcontrollers
    secrets: 10  #最大secrets
    services: 10 #最大services

[root@k8shardway1 kubernetes]# kubectl apply -f object-count.yaml 
resourcequota/object-counts created


#查看详情
[root@k8shardway1 kubernetes]# kubectl get quota 
NAME             AGE   REQUEST                                                                                                      LIMIT
object-counts    36s   configmaps: 1/10, persistentvolumeclaims: 0/4, replicationcontrollers: 0/20, secrets: 1/10, services: 3/10   
resource-quota   52s   pods: 6/4, requests.cpu: 200m/2, requests.memory: 100Mi/4Gi                                                  limits.cpu: 300m/4, limits.memory: 200Mi/8Gi

#查看当前使用详情
[root@k8shardway1 kubernetes]# kubectl describe quota resource-quota
Name:            resource-quota
Namespace:       default
Resource         Used   Hard
--------         ----   ----
limits.cpu       300m   4
limits.memory    200Mi  8Gi
pods             6      4
requests.cpu     200m   2
requests.memory  100Mi  4Gi
```



# QOS服务质量

参考：https://kubernetes.io/zh/docs/tasks/configure-pod-container/quality-service-pod/

就是将资源进行优先级分类

 使用 QoS 类来决定 Pod 的调度和驱逐策略(当没有定义resources，一个容器导致当前node资源不够的时候，就会将当前node其他pod迁移到其他的node，如果定义了resources，当所有的pod的resources资源加起来超过当前宿主机的资源的情况下，就会把一些pod迁移到其他容器或者出发OOM)

驱逐的时候就是根据QOS来定义的。

服务质量QOS（Quality of Service）主要⽤于pod调度和驱逐时参考的重要因素，不同的QOS，其服务质量不同，对应不同的优先级，主要分为三种类型的Qos：

* BestEffort 尽最⼤努⼒分配资源，默认没有指定resource分配的Qos，优先级最低；Pod中没有定义resource，默认的Qos策略为BestEffort,优先级别最低，当资源⽐较进展是需要驱逐evice时，优先驱逐BestEffort定义的Pod

  * 对于 QoS 类为 BestEffort 的 Pod，Pod 中的容器必须没有设置内存和 CPU 限制或请求。

    例如：

    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: qos-demo-3
      namespace: qos-example
    spec:
      containers:
      - name: qos-demo-3-ctr
        image: nginx
        
    [root@k8snode1 k8s]# kubectl get pods nginxpodsdemo1 -o yaml
    apiVersion: v1
    kind: Pod
    metadata:
      annotations:
        cni.projectcalico.org/podIP: 172.16.185.230/32
        cni.projectcalico.org/podIPs: 172.16.185.230/32
        kubectl.kubernetes.io/last-applied-configuration: |
          {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{"poduse":"pod demo the pod"},"labels":{"app":"nginxpodsdemo1","version":"1.7.9"},"name":"nginxpodsdemo1","namespace":"default"},"spec":{"containers":[{"args":["-g","daemon off;"],"command":["nginx"],"image":"nginx:1.7.9","imagePullPolicy":"IfNotPresent","name":"nginxpodsdemo1","ports":[{"containerPort":80,"name":"nginx-port-80","protocol":"TCP"}]}]}}
        poduse: pod demo the pod
      creationTimestamp: "2021-04-20T14:01:21Z"
      labels:
        app: nginxpodsdemo1
        version: 1.7.9
      name: nginxpodsdemo1
      namespace: default
      resourceVersion: "488467"
      uid: 0a68381a-a491-47a3-867c-75ed7e09c815
    spec:
      containers:
      - args:
        - -g
        - daemon off;
        command:
        - nginx
        image: nginx:1.7.9
        imagePullPolicy: IfNotPresent
        name: nginxpodsdemo1
        ports:
        - containerPort: 80
          name: nginx-port-80
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
          name: kube-api-access-spt72
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
      - name: kube-api-access-spt72
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
        lastTransitionTime: "2021-04-20T14:01:20Z"
        status: "True"
        type: Initialized
      - lastProbeTime: null
        lastTransitionTime: "2021-04-20T14:01:22Z"
        status: "True"
        type: Ready
      - lastProbeTime: null
        lastTransitionTime: "2021-04-20T14:01:22Z"
        status: "True"
        type: ContainersReady
      - lastProbeTime: null
        lastTransitionTime: "2021-04-20T14:01:21Z"
        status: "True"
        type: PodScheduled
      containerStatuses:
      - containerID: docker://3aef02a948f83e6ba1f22b10461b1e6eb9a3a7b890342a34b363133e01b313e1
        image: nginx:1.7.9
        imageID: docker-pullable://nginx@sha256:e3456c851a152494c3e4ff5fcc26f240206abac0c9d794affb40e0714846c451
        lastState: {}
        name: nginxpodsdemo1
        ready: true
        restartCount: 0
        started: true
        state:
          running:
            startedAt: "2021-04-20T14:01:21Z"
      hostIP: 192.168.10.186
      phase: Running
      podIP: 172.16.185.230
      podIPs:
      - ip: 172.16.185.230
      qosClass: BestEffort
      startTime: "2021-04-20T14:01:20Z"
    
    ```

    

* Burstable 可波动的资源，⾄少需要分配到requests中的资源，常⻅的QOS；

  如果满足下面条件，将会指定 Pod 的 QoS 类为 Burstable：

  - Pod 不符合 Guaranteed QoS 类的标准。

  - Pod 中至少一个容器具有内存或 CPU 请求。

    例如：

    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: qos-demo-2
      namespace: qos-example
    spec:
      containers:
      - name: qos-demo-2-ctr
        image: nginx
        resources:
          limits:
            memory: "200Mi"
          requests:
            memory: "100Mi"
    ```

    通过命令查看

    ```
    [root@k8snode1 k8s]# kubectl get pods cpu-demo -o yaml
    apiVersion: v1
    kind: Pod
    metadata:
      annotations:
        kubectl.kubernetes.io/last-applied-configuration: |
          {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"name":"cpu-demo","namespace":"default"},"spec":{"containers":[{"args":["--vm","1","--vm-bytes","256M","--vm-hang","1"],"command":["stress"],"image":"vish/stress","name":"cpu-demo-ctr","resources":{"limits":{"memory":"200Mi"},"requests":{"memory":"100Mi"}}}],"nodeName":"k8snode3"}}
      creationTimestamp: "2021-04-20T15:40:18Z"
      deletionGracePeriodSeconds: 30
      deletionTimestamp: "2021-04-20T15:42:01Z"
      name: cpu-demo
      namespace: default
      resourceVersion: "496480"
      uid: 42c90214-3a37-4fe1-bffa-9e5cf2f19dab
    spec:
      containers:
      - args:
        - --vm
        - "1"
        - --vm-bytes
        - 256M
        - --vm-hang
        - "1"
        command:
        - stress
        image: vish/stress
        imagePullPolicy: Always
        name: cpu-demo-ctr
        resources:
          limits:
            memory: 200Mi
          requests:
            memory: 100Mi
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
          name: kube-api-access-gdxkx
          readOnly: true
      dnsPolicy: ClusterFirst
      enableServiceLinks: true
      nodeName: k8snode3
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
      - name: kube-api-access-gdxkx
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
      phase: Pending
      qosClass: Burstable  #qosClass的状态
    
    ```

    

* Guaranteed 完全可保障资源，requests和limits定义的资源相同，优先级最⾼。

  * Pod 中的每个容器，包含初始化容器，必须指定内存请求和内存限制，并且两者要相等。

  * Pod 中的每个容器，包含初始化容器，必须指定 CPU 请求和 CPU 限制，并且两者要相等。

  * 也就是分配的资源limits=requests

    例如：

    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: qos-demo
      namespace: qos-example
    spec:
      containers:
      - name: qos-demo-ctr
        image: nginx
        resources:
          limits:
            memory: "200Mi"
            cpu: "700m"
          requests:
            memory: "200Mi"
            cpu: "700m"
    ```

* 如果有多个容器，以优先满足条件为准

* 通过介绍resource资源的分配和服务质量Qos，关于resource有节点使⽤建议：

  * requests和limits资源定义推荐不超过1:2，避免分配过多资源⽽出现资源争抢，发⽣OOM；

  * pod中默认没有定义resource，推荐给namespace定义⼀个limitrange，确保pod能分到资源；

  * 防⽌node上资源过度⽽出现机器hang住或者OOM，建议node上设置保留和驱逐资源，如保留资源--system-reserved=cpu=200m,memory=1G，驱逐条件--eviction-hard=memory.available<500Mi。

    修改：

    ```
    [root@k8snode3 ~]# vim /etc/sysconfig/kubelet 
    
    KUBELET_EXTRA_ARGS="--kube-reserved=cpu=200m,memory=500Mi"
    
    [root@k8snode3 ~]# systemctl restart kubelet
    
    [root@k8snode1 k8s]# kubectl describe node k8snode3
    Name:               k8snode3
    Roles:              <none>
    Labels:             beta.kubernetes.io/arch=amd64
                        beta.kubernetes.io/os=linux
                        kubernetes.io/arch=amd64
                        kubernetes.io/hostname=k8snode3
                        kubernetes.io/os=linux
    Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: /var/run/dockershim.sock
                        node.alpha.kubernetes.io/ttl: 0
                        projectcalico.org/IPv4Address: 192.168.10.187/24
                        projectcalico.org/IPv4IPIPTunnelAddr: 172.16.98.192
                        volumes.kubernetes.io/controller-managed-attach-detach: true
    CreationTimestamp:  Sat, 17 Apr 2021 23:01:06 +0800
    Taints:             <none>
    Unschedulable:      false
    Lease:
      HolderIdentity:  k8snode3
      AcquireTime:     <unset>
      RenewTime:       Wed, 21 Apr 2021 00:21:21 +0800
    Conditions:
      Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
      ----                 ------  -----------------                 ------------------                ------                       -------
      NetworkUnavailable   False   Sat, 17 Apr 2021 23:55:36 +0800   Sat, 17 Apr 2021 23:55:36 +0800   CalicoIsUp                   Calico is running on this node
      MemoryPressure       False   Wed, 21 Apr 2021 00:21:11 +0800   Wed, 21 Apr 2021 00:21:11 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
      DiskPressure         False   Wed, 21 Apr 2021 00:21:11 +0800   Wed, 21 Apr 2021 00:21:11 +0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
      PIDPressure          False   Wed, 21 Apr 2021 00:21:11 +0800   Wed, 21 Apr 2021 00:21:11 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
      Ready                True    Wed, 21 Apr 2021 00:21:11 +0800   Wed, 21 Apr 2021 00:21:11 +0800   KubeletReady                 kubelet is posting ready status
    Addresses:
      InternalIP:  192.168.10.187
      Hostname:    k8snode3
    Capacity:
      cpu:                4
      ephemeral-storage:  79132928Ki
      hugepages-1Gi:      0
      hugepages-2Mi:      0
      memory:             3879604Ki
      pods:               110
    Allocatable:
      cpu:                3800m
      ephemeral-storage:  72928906325
      hugepages-1Gi:      0
      hugepages-2Mi:      0
      memory:             3265204Ki
      pods:               110
    System Info:
      Machine ID:                 7de771b3710f4ef69422414077898d01
      System UUID:                01644D56-7B25-BBE3-EB16-14796E87DF02
      Boot ID:                    255e4253-843e-44f2-a583-ad283d249ed2
      Kernel Version:             3.10.0-1160.15.2.el7.x86_64
      OS Image:                   CentOS Linux 7 (Core)
      Operating System:           linux
      Architecture:               amd64
      Container Runtime Version:  docker://19.3.9
      Kubelet Version:            v1.21.0
      Kube-Proxy Version:         v1.21.0
    PodCIDR:                      172.16.2.0/24
    PodCIDRs:                     172.16.2.0/24
    Non-terminated Pods:          (10 in total)
      Namespace                   Name                                          CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
      ---------                   ----                                          ------------  ----------  ---------------  -------------  ---
      default                     nginx-dashboard-demo-5cd59c4f6d-dbc2d         0 (0%)        0 (0%)      0 (0%)           0 (0%)         30h
      default                     nginx-dashboard-demo-5cd59c4f6d-jbzb8         0 (0%)        0 (0%)      0 (0%)           0 (0%)         30h
      default                     nginxdemo1-6995675b4b-rsc4s                   0 (0%)        0 (0%)      0 (0%)           0 (0%)         32h
      kube-system                 calico-kube-controllers-6d8ccdbf46-s6qdx      0 (0%)        0 (0%)      0 (0%)           0 (0%)         3d
      kube-system                 calico-node-249tw                             250m (6%)     0 (0%)      0 (0%)           0 (0%)         3d
      kube-system                 coredns-558bd4d5db-56pr7                      100m (2%)     0 (0%)      70Mi (2%)        170Mi (5%)     3d1h
      kube-system                 coredns-558bd4d5db-wbgr6                      100m (2%)     0 (0%)      70Mi (2%)        170Mi (5%)     3d1h
      kube-system                 kube-proxy-9tvkf                              0 (0%)        0 (0%)      0 (0%)           0 (0%)         3d1h
      kubernetes-dashboard        dashboard-metrics-scraper-5594697f48-xj59f    0 (0%)        0 (0%)      0 (0%)           0 (0%)         31h
      tigera-operator             tigera-operator-675ccbb69c-ptdmm              0 (0%)        0 (0%)      0 (0%)           0 (0%)         3d1h
    Allocated resources:
      (Total limits may be over 100 percent, i.e., overcommitted.)
      Resource           Requests    Limits
      --------           --------    ------
      cpu                450m (11%)  0 (0%)
      memory             140Mi (4%)  340Mi (10%)
      ephemeral-storage  0 (0%)      0 (0%)
      hugepages-1Gi      0 (0%)      0 (0%)
      hugepages-2Mi      0 (0%)      0 (0%)
    Events:
      Type     Reason                   Age                From        Message
      ----     ------                   ----               ----        -------
      Normal   Starting                 10s                kubelet     Starting kubelet.
      #查看已经设置了资源
      Normal   NodeAllocatableEnforced  10s                kubelet     Updated Node Allocatable limit across pods
      Normal   NodeHasSufficientMemory  10s (x2 over 10s)  kubelet     Node k8snode3 status is now: NodeHasSufficientMemory
      Normal   NodeHasNoDiskPressure    10s (x2 over 10s)  kubelet     Node k8snode3 status is now: NodeHasNoDiskPressure
      Normal   NodeHasSufficientPID     10s (x2 over 10s)  kubelet     Node k8snode3 status is now: NodeHasSufficientPID
      Warning  Rebooted                 10s                kubelet     Node k8snode3 has been rebooted, boot id: 255e4253-843e-44f2-a583-ad283d249ed2
      Normal   NodeReady                10s                kubelet     Node k8snode3 status is now: NodeReady
      Normal   Starting                 5s                 kube-proxy  Starting kube-proxy.
    
    
    #也可以设置驱逐的条件，就是当可用内存就剩下1G的时候就进行驱逐调整
    [root@k8snode3 ~]# vim /etc/sysconfig/kubelet 
    
    KUBELET_EXTRA_ARGS="--kube-reserved=cpu=200m,memory=500Mi --eviction-hard=memory.available<500Mi"
    [root@k8snode3 ~]# systemctl restart kubelet
    ```

常见的驱逐策略

1. --eviction-soft=memory.available < 1.5Gi  
2. --eviction-soft-grace-period=memory.available=1m30s  当上面1结合使用 当节点内存少于1.5G并且持续1分30秒的时候 开始驱逐
3. --eviction-hard=memory.available<500Mi,nodefs.available < 1Gi,nodefs.inodesFree < 5% 当内存小于500M 磁盘小于1G 剩余inodesFree  立刻开始驱逐

磁盘紧缺的时候怎么处理

1. 删除撕掉的pod和容器
2. 删除没有用的镜像
3. 最后还紧缺 就开始按照优先级 资源占用情况驱逐pod

内存紧缺

1. 先驱逐不可靠的pod
2. 驱逐基本不可靠的pod
3. 最后没办法了 在驱逐可靠的pod

# 健康检查

在不设置健康检查的，会根据入口程序是否存在而判断当前pod是否正常是否需要重启

主要分为存活和就绪

就绪如果不成功 就会重启pod

存活如果不成功 就会从service提出去

应⽤在运⾏过程中难免会出现错误，如程序异常，软件异常，硬件故障，⽹络故障等，kubernetes提供Health Check健康检查机制，当发现应⽤异常时会⾃动重启容器，将应⽤从service服务中剔除，保障应⽤的⾼可⽤性。k8s定义了三种探针Probe：

* readiness probes 准备就绪检查，通过readiness是否准备接受流量，准备完毕加⼊到endpoint，否则剔除
* liveness probes 在线检查机制，检查应⽤是否可⽤，如死锁，⽆法响应，异常时会⾃动重启容器
* startup probes 启动检查机制，应⽤⼀些启动缓慢的业务，避免业务⻓时间启动⽽被前⾯的探针kill掉

每种探测机制⽀持三种健康检查⽅法，分别是命令⾏exec，httpGet和tcpSocket，其中exec通⽤性最强，适⽤与⼤部分场景，tcpSocket适⽤于TCP业务，httpGet适⽤于web业务。

* exec 提供命令或shell的检测，在容器中执⾏命令检查，返回码为0健康，⾮0异常
* httpGet http协议探测，在容器中发送http请求，根据http返回码判断业务健康情况
* tcpSocket tcp协议探测，向容器发送tcp建⽴连接，能建⽴则说明正常

1. exec的健康检查，

   ```
   [root@k8snode1 k8s]# kubectl explain pods.spec.containers.livenessProbe
   
   #定义一个key
   [root@k8snode1 k8s]# vim pods-probe-exec-liveness-yaml
   
   apiVersion: v1
   kind: Pod
   metadata:
     labels:
       test: liveness
     name: liveness-exec
   spec:
     containers:
     - name: liveness
       image: busybox
       args:  #定义一个脚本过会删除他
       - /bin/sh
       - -c
       - 任务
       livenessProbe:   
         exec:  #exec健康检测
           command: # 检测命令 看看这个文件存在不
           - cat
           - /tmp/healthy
         initialDelaySeconds: 5  #容器启动的时间需要多久
         periodSeconds: 5   #多久执行一次 5秒一次
         faulureThreashold: 3 #健康检查失败的次数 当失败3次的时候 就任务彻底失败了
         timeoutSeconds: 1   #每次执行超时时间
         successThreshold: 1  #从faulureThreashold到正确执行多少次算成功
   
   [root@k8snode1 k8s]# kubectl apply -f pods-probe-exec-liveness-yaml 
   pod/liveness-exec created
   
   [root@k8snode1 k8s]# kubectl describe pod liveness-exec
   Name:         liveness-exec
   Namespace:    default
   Priority:     0
   Node:         k8snode3/192.168.10.187
   Start Time:   Wed, 21 Apr 2021 17:58:49 +0800
   Labels:       test=liveness
   Annotations:  cni.projectcalico.org/podIP: 172.16.98.208/32
                 cni.projectcalico.org/podIPs: 172.16.98.208/32
   Status:       Running
   IP:           172.16.98.208
   IPs:
     IP:  172.16.98.208
   Containers:
     liveness:
       Container ID:  docker://02bceaddb44f525a789878c02fe9c5e8aeaf79f39005bdf612217ebb87864891
       Image:         busybox
       Image ID:      docker-pullable://busybox@sha256:ae39a6f5c07297d7ab64dbd4f82c77c874cc6a94cea29fdec309d0992574b4f7
       Port:          <none>
       Host Port:     <none>
       Args:
         /bin/sh
         -c
         touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
       State:          Running
         Started:      Wed, 21 Apr 2021 17:58:58 +0800
       Ready:          True
       Restart Count:  0 
       Liveness:       exec [cat /tmp/healthy] delay=5s timeout=1s period=5s #success=1 #failure=3   #这个就是执行的健康检查的探针
       Environment:    <none>
       Mounts:
         /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-9jvfq (ro)
   Conditions:
     Type              Status
     Initialized       True 
     Ready             True 
     ContainersReady   True 
     PodScheduled      True 
   Volumes:
     kube-api-access-9jvfq:
       Type:                    Projected (a volume that contains injected data from multiple sources)
       TokenExpirationSeconds:  3607
       ConfigMapName:           kube-root-ca.crt
       ConfigMapOptional:       <nil>
       DownwardAPI:             true
   QoS Class:                   BestEffort
   Node-Selectors:              <none>
   Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                                node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
   Events:
     Type    Reason     Age        From               Message
     ----    ------     ----       ----               -------
     Normal  Scheduled  14s        default-scheduler  Successfully assigned default/liveness-exec to k8snode3
     Normal  Pulling    0s         kubelet            Pulling image "busybox"
     Normal  Pulled     <invalid>  kubelet            Successfully pulled image "busybox" in 7.203328413s
     Normal  Created    <invalid>  kubelet            Created container liveness
     Normal  Started    <invalid>  kubelet            Started container liveness
     
   [root@k8snode1 k8s]# kubectl describe pod liveness-exec
   Name:         liveness-exec
   Namespace:    default
   Priority:     0
   Node:         k8snode3/192.168.10.187
   Start Time:   Wed, 21 Apr 2021 17:58:49 +0800
   Labels:       test=liveness
   Annotations:  cni.projectcalico.org/podIP: 172.16.98.208/32
                 cni.projectcalico.org/podIPs: 172.16.98.208/32
   Status:       Running
   IP:           172.16.98.208
   IPs:
     IP:  172.16.98.208
   Containers:
     liveness:
       Container ID:  docker://75ec9d7bbce69cc61c99462824e380e8661aa40cf00332646b443e53b2531665
       Image:         busybox
       Image ID:      docker-pullable://busybox@sha256:ae39a6f5c07297d7ab64dbd4f82c77c874cc6a94cea29fdec309d0992574b4f7
       Port:          <none>
       Host Port:     <none>
       Args:
         /bin/sh
         -c
         touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
       State:          Running
         Started:      Wed, 21 Apr 2021 18:00:14 +0800
       Last State:     Terminated
         Reason:       Error
         Exit Code:    137
         Started:      Wed, 21 Apr 2021 17:58:58 +0800
         Finished:     Wed, 21 Apr 2021 18:00:09 +0800
       Ready:          True
       Restart Count:  1
       Liveness:       exec [cat /tmp/healthy] delay=5s timeout=1s period=5s #success=1 #failure=3
       Environment:    <none>
       Mounts:
         /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-9jvfq (ro)
   Conditions:
     Type              Status
     Initialized       True 
     Ready             True 
     ContainersReady   True 
     PodScheduled      True 
   Volumes:
     kube-api-access-9jvfq:
       Type:                    Projected (a volume that contains injected data from multiple sources)
       TokenExpirationSeconds:  3607
       ConfigMapName:           kube-root-ca.crt
       ConfigMapOptional:       <nil>
       DownwardAPI:             true
   QoS Class:                   BestEffort
   Node-Selectors:              <none>
   Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                                node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
   Events:
     Type     Reason     Age                From               Message
     ----     ------     ----               ----               -------
     Normal   Scheduled  103s               default-scheduler  Successfully assigned default/liveness-exec to k8snode3
     Normal   Pulled     81s                kubelet            Successfully pulled image "busybox" in 7.203328413s
     #过了30秒后，发现文件不能存在，报错了 
     Warning  Unhealthy  39s (x3 over 49s)  kubelet            Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
     # kill掉这个容器
     Normal   Killing    39s                kubelet            Container liveness failed liveness probe, will be restarted
     Normal   Pulling    8s (x2 over 88s)   kubelet            Pulling image "busybox"
     #又创建一个 在启动 状态有正常了
     Normal   Created    4s (x2 over 81s)   kubelet            Created container liveness
     Normal   Started    4s (x2 over 80s)   kubelet            Started container liveness
     Normal   Pulled     4s                 kubelet            Successfully pulled image "busybox" in 4.247772722s
   
   ```

2. http的健康检查，发送一个http的get请求，返回是200就成功 非200就是容器异常

   ```
   [root@k8snode1 k8s]# vim nginx-httpGet-liveness-readiness.yaml 
   
   apiVersion: v1
   kind: Pod
   metadata:
     name: nginx-httpget-livesss
     annotations:
       kubernetes.io/description: "nginx-httpGet-livess-readiness-probe"
   spec:
     containers:
     - name: nginx-httpget-livess-readiness-probe
       image: nginx:latest
       ports:
       - name: http-80-port
         protocol: TCP
         containerPort: 80
       livenessProbe: #健康检查机制，通过httpGet实现实现检查
         httpGet:
           port: 80
           scheme: HTTP
           path: /index.html
         initialDelaySeconds: 3
         periodSeconds: 10
         timeoutSeconds: 3
   
   [root@k8snode1 k8s]# kubectl apply -f nginx-httpGet-liveness-readiness.yaml 
   pod/nginx-httpget-livesss created
   
   [root@k8snode1 k8s]# kubectl describe pod  nginx-httpget-livesss
   Name:         nginx-httpget-livesss
   Namespace:    default
   Priority:     0
   Node:         k8snode3/192.168.10.187
   Start Time:   Wed, 21 Apr 2021 18:14:01 +0800
   Labels:       <none>
   Annotations:  cni.projectcalico.org/podIP: 172.16.98.209/32
                 cni.projectcalico.org/podIPs: 172.16.98.209/32
                 kubernetes.io/description: nginx-httpGet-livess-readiness-probe
   Status:       Running
   IP:           172.16.98.209
   IPs:
     IP:  172.16.98.209
   Containers:
     nginx-httpget-livess-readiness-probe:
       Container ID:   docker://a8c30b65986d0690beff8ff8bf980438875868aa52a78cd36dc707fecd5c81c3
       Image:          nginx:latest
       Image ID:       docker-pullable://nginx@sha256:75a55d33ecc73c2a242450a9f1cc858499d468f077ea942867e662c247b5e412
       Port:           80/TCP
       Host Port:      0/TCP
       State:          Running
         Started:      Wed, 21 Apr 2021 18:14:19 +0800
       Ready:          True
       Restart Count:  0
       Liveness:       http-get http://:80/index.html delay=3s timeout=3s period=10s #success=1 #failure=3  #http的探针
       Environment:    <none>
       Mounts:
         /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-k6d44 (ro)
   Conditions:
     Type              Status
     Initialized       True 
     Ready             True 
     ContainersReady   True 
     PodScheduled      True 
   Volumes:
     kube-api-access-k6d44:
       Type:                    Projected (a volume that contains injected data from multiple sources)
       TokenExpirationSeconds:  3607
       ConfigMapName:           kube-root-ca.crt
       ConfigMapOptional:       <nil>
       DownwardAPI:             true
   QoS Class:                   BestEffort
   Node-Selectors:              <none>
   Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                                node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
   Events:
     Type    Reason     Age   From               Message
     ----    ------     ----  ----               -------
     Normal  Scheduled  48s   default-scheduler  Successfully assigned default/nginx-httpget-livesss to k8snode3
     Normal  Pulling    34s   kubelet            Pulling image "nginx:latest"
     Normal  Pulled     18s   kubelet            Successfully pulled image "nginx:latest" in 15.665010088s
     Normal  Created    17s   kubelet            Created container nginx-httpget-livess-readiness-probe
     Normal  Started    17s   kubelet            Started container nginx-httpget-livess-readiness-probe
   
   #进入到容器里 把index.html文件删掉
   [root@k8snode1 k8s]# kubectl exec -it nginx-httpget-livesss /bin/bash
   kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
   root@nginx-httpget-livesss:/# cd /usr/local/  
   bin/     etc/     games/   include/ lib/     man/     sbin/    share/   src/     
   root@nginx-httpget-livesss:/# cd /usr/share/nginx/html
   root@nginx-httpget-livesss:/usr/share/nginx/html# rm -rf index.html 
   
   #再次进入容器发现被删掉了又重启了一次
   [root@k8snode1 k8s]# kubectl describe pod  nginx-httpget-livesss
   Name:         nginx-httpget-livesss
   Namespace:    default
   Priority:     0
   Node:         k8snode3/192.168.10.187
   Start Time:   Wed, 21 Apr 2021 18:14:01 +0800
   Labels:       <none>
   Annotations:  cni.projectcalico.org/podIP: 172.16.98.209/32
                 cni.projectcalico.org/podIPs: 172.16.98.209/32
                 kubernetes.io/description: nginx-httpGet-livess-readiness-probe
   Status:       Running
   IP:           172.16.98.209
   IPs:
     IP:  172.16.98.209
   Containers:
     nginx-httpget-livess-readiness-probe:
       Container ID:   docker://a8c30b65986d0690beff8ff8bf980438875868aa52a78cd36dc707fecd5c81c3
       Image:          nginx:latest
       Image ID:       docker-pullable://nginx@sha256:75a55d33ecc73c2a242450a9f1cc858499d468f077ea942867e662c247b5e412
       Port:           80/TCP
       Host Port:      0/TCP
       State:          Running
         Started:      Wed, 21 Apr 2021 18:14:19 +0800
       Ready:          True
       Restart Count:  0
       Liveness:       http-get http://:80/index.html delay=3s timeout=3s period=10s #success=1 #failure=3
       Environment:    <none>
       Mounts:
         /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-k6d44 (ro)
   Conditions:
     Type              Status
     Initialized       True 
     Ready             True 
     ContainersReady   True 
     PodScheduled      True 
   Volumes:
     kube-api-access-k6d44:
       Type:                    Projected (a volume that contains injected data from multiple sources)
       TokenExpirationSeconds:  3607
       ConfigMapName:           kube-root-ca.crt
       ConfigMapOptional:       <nil>
       DownwardAPI:             true
   QoS Class:                   BestEffort
   Node-Selectors:              <none>
   Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                                node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
   Events:
     Type     Reason     Age                        From               Message
     ----     ------     ----                       ----               -------
     Normal   Scheduled  3m2s                       default-scheduler  Successfully assigned default/nginx-httpget-livesss to k8snode3
     Normal   Pulled     2m32s                      kubelet            Successfully pulled image "nginx:latest" in 15.665010088s
     Normal   Created    2m31s                      kubelet            Created container nginx-httpget-livess-readiness-probe
     Normal   Started    2m31s                      kubelet            Started container nginx-httpget-livess-readiness-probe
     #再次进入容器发现被删掉了又重启了一次
     Warning  Unhealthy  <invalid> (x3 over 9s)     kubelet            Liveness probe failed: HTTP probe failed with statuscode: 404
     Normal   Killing    <invalid>                  kubelet            Container nginx-httpget-livess-readiness-probe failed liveness probe, will be restarted
     Normal   Pulling    <invalid> (x2 over 2m48s)  kubelet            Pulling image "nginx:latest"
   
   ```

3. tcp的健康检查，根据端口建立一个连接，如果连接正常，则正常 如果连接建立失败 则认为容器异常

   ```
   
   [root@k8snode1 k8s]# vim nginx-tcp-liveness.yaml
   
   apiVersion: v1
   kind: Pod
   metadata:
     name: nginx-tcp-liveness-probe
     annotations:
       kubernetes.io/description: "nginx-tcp-liveness-probe"
   spec:
     containers:
     - name: nginx-tcp-liveness-probe
       image: nginx:latest
       ports:
       - name: http-80-port
         protocol: TCP
         containerPort: 80
       livenessProbe: #健康检查为tcpSocket，探测TCP 80端⼝
         tcpSocket:
           port: 80
         initialDelaySeconds: 3
         periodSeconds: 10
         timeoutSeconds: 3
   
   
   [root@k8snode1 k8s]# kubectl apply -f nginx-tcp-liveness.yaml 
   pod/nginx-tcp-liveness-probe created
   [root@k8snode1 k8s]# kubectl describe pod nginx-tcp-liveness-probe
   Name:         nginx-tcp-liveness-probe
   Namespace:    default
   Priority:     0
   Node:         k8snode3/192.168.10.187
   Start Time:   Wed, 21 Apr 2021 21:27:31 +0800
   Labels:       <none>
   Annotations:  cni.projectcalico.org/podIP: 172.16.98.210/32
                 cni.projectcalico.org/podIPs: 172.16.98.210/32
                 kubernetes.io/description: nginx-tcp-liveness-probe
   Status:       Running
   IP:           172.16.98.210
   IPs:
     IP:  172.16.98.210
   Containers:
     nginx-tcp-liveness-probe:
       Container ID:   docker://e15cd2acb1dc75c0bfa11802bf92747d489534c6b22d550b661f574e7297963f
       Image:          nginx:latest
       Image ID:       docker-pullable://nginx@sha256:75a55d33ecc73c2a242450a9f1cc858499d468f077ea942867e662c247b5e412
       Port:           80/TCP
       Host Port:      0/TCP
       State:          Running
         Started:      Wed, 21 Apr 2021 21:27:38 +0800
       Ready:          True
       Restart Count:  0
       Liveness:       tcp-socket :80 delay=3s timeout=3s period=10s #success=1 #failure=3
       Environment:    <none>
       Mounts:
         /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-wbx5v (ro)
   Conditions:
     Type              Status
     Initialized       True 
     Ready             True 
     ContainersReady   True 
     PodScheduled      True 
   Volumes:
     kube-api-access-wbx5v:
       Type:                    Projected (a volume that contains injected data from multiple sources)
       TokenExpirationSeconds:  3607
       ConfigMapName:           kube-root-ca.crt
       ConfigMapOptional:       <nil>
       DownwardAPI:             true
   QoS Class:                   BestEffort
   Node-Selectors:              <none>
   Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                                node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
   Events:
     Type    Reason     Age   From               Message
     ----    ------     ----  ----               -------
     Normal  Scheduled  50s   default-scheduler  Successfully assigned default/nginx-tcp-liveness-probe to k8snode3
     Normal  Pulling    35s   kubelet            Pulling image "nginx:latest"
     Normal  Pulled     29s   kubelet            Successfully pulled image "nginx:latest" in 5.614647146s
     Normal  Created    29s   kubelet            Created container nginx-tcp-liveness-probe
     Normal  Started    29s   kubelet            Started container nginx-tcp-liveness-probe
   
   
   [root@k8snode1 k8s]# kubectl exec -it nginx-tcp-liveness-probe /bin/bash
   kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
   #root@nginx-tcp-liveness-probe:/# apt-get update 就可以安装工具了
   
   #运⾏htop查看进程，容器进程通常为1，kill掉进程观察容器状态，观察RESTART次数重启次数增加
   root@nginx-tcp-liveness-probe:/# kill 1
   root@nginx-tcp-liveness-probe:/# command terminated with exit code 137
   
   [root@k8snode1 k8s]#  kubectl describe pod nginx-tcp-liveness-probe
   Name:         nginx-tcp-liveness-probe
   Namespace:    default
   Priority:     0
   Node:         k8snode3/192.168.10.187
   Start Time:   Wed, 21 Apr 2021 21:27:31 +0800
   Labels:       <none>
   Annotations:  cni.projectcalico.org/podIP: 172.16.98.210/32
                 cni.projectcalico.org/podIPs: 172.16.98.210/32
                 kubernetes.io/description: nginx-tcp-liveness-probe
   Status:       Running
   IP:           172.16.98.210
   IPs:
     IP:  172.16.98.210
   Containers:
     nginx-tcp-liveness-probe:
       Container ID:   docker://e106fc2339b755212d6df1dfc6e6ac25fb5df63efc7c5f1db55217412c170eca
       Image:          nginx:latest
       Image ID:       docker-pullable://nginx@sha256:75a55d33ecc73c2a242450a9f1cc858499d468f077ea942867e662c247b5e412
       Port:           80/TCP
       Host Port:      0/TCP
       State:          Running
         Started:      Wed, 21 Apr 2021 21:34:14 +0800
       Last State:     Terminated
         Reason:       Completed
         Exit Code:    0
         Started:      Wed, 21 Apr 2021 21:31:39 +0800
         Finished:     Wed, 21 Apr 2021 21:33:50 +0800
       Ready:          True
       Restart Count:  2
       Liveness:       tcp-socket :80 delay=3s timeout=3s period=10s #success=1 #failure=3
       Environment:    <none>
       Mounts:
         /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-wbx5v (ro)
   Conditions:
     Type              Status
     Initialized       True 
     Ready             True 
     ContainersReady   True 
     PodScheduled      True 
   Volumes:
     kube-api-access-wbx5v:
       Type:                    Projected (a volume that contains injected data from multiple sources)
       TokenExpirationSeconds:  3607
       ConfigMapName:           kube-root-ca.crt
       ConfigMapOptional:       <nil>
       DownwardAPI:             true
   QoS Class:                   BestEffort
   Node-Selectors:              <none>
   Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                                node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
   Events:
     Type     Reason     Age                        From               Message
     ----     ------     ----                       ----               -------
     Normal   Scheduled  6m53s                      default-scheduler  Successfully assigned default/nginx-tcp-liveness-probe to k8snode3
     Normal   Pulled     6m31s                      kubelet            Successfully pulled image "nginx:latest" in 5.614647146s
     Normal   Pulled     2m30s                      kubelet            Successfully pulled image "nginx:latest" in 14.363691414s
    # kill掉进程后就可以看到 有在重新pulling一个新的image 然后启动
    Warning  BackOff    17s (x2 over 18s)          kubelet            Back-off restarting failed container
     Normal   Pulling    6s (x3 over 6m37s)         kubelet            Pulling image "nginx:latest"
     Normal   Created    <invalid> (x3 over 6m31s)  kubelet            Created container nginx-tcp-liveness-probe
     Normal   Started    <invalid> (x3 over 6m31s)  kubelet            Started container nginx-tcp-liveness-probe
     Normal   Pulled     <invalid>                  kubelet            Successfully pulled image "nginx:latest" in 10.660390743s
   
   ```

4. readiness健康就绪探针

   就绪检查⽤于应⽤接⼊到service的场景，⽤于判断应⽤是否已经就绪完毕，即是否可以接受外部转发的流量，健康检查正常则将pod加⼊到service的endpoints中，健康检查异常则从service的endpoints中删除，避免影响业务的访问。也就是判断是否可以从外部接收请求了。

5. readiness⽤httpGet的健康检查机制，参考文档：https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

   ```
   [root@k8snode1 k8s]# vim httpget-readiness-liveness-readiness-probe.yaml
   
   apiVersion: v1
   kind: Pod
   metadata:
     name: nginx-tcp-readiness-liveness-probe
     annotations:
       kubernetes.io/description: "nginx-tcp-liveness-probe"
     labels:
       app: nginx-readiness
   spec:
     containers:
     - name: nginx-tcp-readiness-liveness-probe
       image: nginx:1.7.9
       ports:
       - name: http-80-port
         protocol: TCP
         containerPort: 80
       livenessProbe:
         httpGet:
           port: 80
           path: /index.html
           scheme: HTTP
         initialDelaySeconds: 3
         periodSeconds: 10
         timeoutSeconds: 3
       readinessProbe:
         httpGet:
           port: 80
           path: /test.html  #会探测失败，就会加不到服务里
           scheme: HTTP
         initialDelaySeconds: 3
         periodSeconds: 10
         timeoutSeconds: 3
   
   [root@k8snode1 k8s]# kubectl apply -f httpget-readiness-liveness-readiness-probe.yaml 
   pod/nginx-tcp-readiness-liveness-probe created
   
   
   [root@k8snode1 k8s]# vim httpget-liveness-readiness-probe-server.yaml
   
   apiVersion: v1
   kind: Service
   metadata:
     labels:
       app: nginx-readiness
     name: nginx-service
   spec:
     ports:
     - name: http
       port: 80
       protocol: TCP
       targetPort: 80
     selector:
       app: nginx
     type: ClusterIP
   
   [root@k8snode1 k8s]# kubectl apply -f httpget-readiness-liveness-readiness-probe-server.yaml 
   service/nginx-service created
   
   #因为探测失败，所以会访问失败
   [root@k8snode1 k8s]# curl http://10.98.148.23:80
   curl: (7) Failed connect to 10.98.148.23:80; Connection refused
   
   
   [root@k8snode1 k8s]# kubectl get pods
   NAME                                    READY   STATUS      RESTARTS   AGE
   nginx-dashboard-demo-5cd59c4f6d-bpq6h   1/1     Running     2          2d4h
   nginx-dashboard-demo-5cd59c4f6d-j2c7l   1/1     Running     1          47h
   nginx-dashboard-demo-5cd59c4f6d-w2n6d   1/1     Running     1          47h
   nginx-demo                              1/1     Running     0          23h
   nginx-httpget-livesss                   1/1     Running     1          3h52m
   nginx-tcp-liveness-probe                1/1     Running     2          38m
   nginx-tcp-readiness-liveness-probe      0/1     Running     0          5m55s # 服务没有read
   nginxdemo                               1/1     Running     2          2d22h
   nginxdemo1-6995675b4b-gtbhh             1/1     Running     1          47h
   nginxdemo1-6995675b4b-jbfrt             1/1     Running     2          2d6h
   nginxdemo1-6995675b4b-pnjnp             1/1     Running     2          2d6h
   nginxdemo1-6995675b4b-z7q72             1/1     Running     2          2d6h
   nginxpodsdemo                           1/1     Running     0          27h
   nginxpodsdemo1                          1/1     Running     0          24h
   nginxpodsdemo2                          1/1     Running     0          24h
   pod-muti-container                      2/2     Running     0          23h
   pod-restart-demo                        0/1     Completed   0          23h
   
   [root@k8snode1 k8s]# kubectl describe pod nginx-tcp-readiness-liveness-probe
   Name:         nginx-tcp-readiness-liveness-probe
   Namespace:    default
   Priority:     0
   Node:         k8snode3/192.168.10.187
   Start Time:   Wed, 21 Apr 2021 22:00:34 +0800
   Labels:       app=nginx-readiness
   Annotations:  cni.projectcalico.org/podIP: 172.16.98.211/32
                 cni.projectcalico.org/podIPs: 172.16.98.211/32
                 kubernetes.io/description: nginx-tcp-liveness-probe
   Status:       Running
   IP:           172.16.98.211
   IPs:
     IP:  172.16.98.211
   Containers:
     nginx-tcp-readiness-liveness-probe:
       Container ID:   docker://ba7d6dff71d3d8c35a24b9d3c93eb620ee481fcad3aa194497f26656eb82222d
       Image:          nginx:1.7.9
       Image ID:       docker-pullable://nginx@sha256:e3456c851a152494c3e4ff5fcc26f240206abac0c9d794affb40e0714846c451
       Port:           80/TCP
       Host Port:      0/TCP
       State:          Running
         Started:      Wed, 21 Apr 2021 22:00:35 +0800
       Ready:          False
       Restart Count:  0
       Liveness:       http-get http://:80/index.html delay=3s timeout=3s period=10s #success=1 #failure=3
       Readiness:      http-get http://:80/test.html delay=3s timeout=3s period=10s #success=1 #failure=3
       Environment:    <none>
       Mounts:
         /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-7hf9c (ro)
   Conditions:
     Type              Status
     Initialized       True 
     Ready             False 
     ContainersReady   False 
     PodScheduled      True 
   Volumes:
     kube-api-access-7hf9c:
       Type:                    Projected (a volume that contains injected data from multiple sources)
       TokenExpirationSeconds:  3607
       ConfigMapName:           kube-root-ca.crt
       ConfigMapOptional:       <nil>
       DownwardAPI:             true
   QoS Class:                   BestEffort
   Node-Selectors:              <none>
   Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                                node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
   Events:
     Type     Reason     Age                   From               Message
     ----     ------     ----                  ----               -------
     Normal   Scheduled  5m8s                  default-scheduler  Successfully assigned default/nginx-tcp-readiness-liveness-probe to k8snode3
     Normal   Pulled     4m53s                 kubelet            Container image "nginx:1.7.9" already present on machine
     Normal   Created    4m53s                 kubelet            Created container nginx-tcp-readiness-liveness-probe
     Normal   Started    4m53s                 kubelet            Started container nginx-tcp-readiness-liveness-probe
     #发现探测服务是否是就绪 就是失败，发现并没有
     Warning  Unhealthy  74s (x22 over 4m44s)  kubelet            Readiness probe failed: HTTP probe failed with statuscode: 404
   
   #也并没有加入到service的 endpoints里面
   [root@k8snode1 k8s]# kubectl describe service nginx-service
   Name:              nginx-service
   Namespace:         default
   Labels:            app=nginx-readiness
   Annotations:       <none>
   Selector:          app=nginx
   Type:              ClusterIP
   IP Family Policy:  SingleStack
   IP Families:       IPv4
   IP:                10.98.148.23
   IPs:               10.98.148.23
   Port:              http  80/TCP
   TargetPort:        80/TCP
   Endpoints:         <none>  #也并没有加入到service的 endpoints里面
   Session Affinity:  None
   Events:            <none>
   
   
   #删除pod
   [root@k8snode1 k8s]# kubectl delete -f httpget-readiness-liveness-readiness-probe.yaml 
   pod "nginx-tcp-readiness-liveness-probe" deleted
   
   
   
   #也可以直接expose导出
   [root@k8snode1 k8s]# kubectl expose pod  nginx-readiness
   
   #已经导出service的vip
   [root@k8snode1 k8s]# kubectl get service
   NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
   kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP        3d23h
   nginx-service   ClusterIP   10.98.148.23    <none>        80/TCP         48s
   nginxdemo1      NodePort    10.105.52.111   <none>        80:31204/TCP   2d7h
   
   [root@k8snode1 k8s]# vim httpget-readiness-liveness-readiness-probe.yaml
   
   apiVersion: v1
   kind: Pod
   metadata:
     name: nginx-tcp-readiness-liveness-probe
     annotations:
       kubernetes.io/description: "nginx-tcp-liveness-probe"
     labels:  #需要定义labels，后⾯定义的service需要调⽤
       app: nginx-readiness
   spec:
     containers:
     - name: nginx-tcp-readiness-liveness-probe
       image: nginx:1.7.9
       ports:
       - name: http-80-port
         protocol: TCP
         containerPort: 80
       livenessProbe:  #存活检查探针
         httpGet:
           port: 80
           path: /index.html
           scheme: HTTP
         initialDelaySeconds: 3
         periodSeconds: 10
         timeoutSeconds: 3
       readinessProbe:  #就绪检查探针
         httpGet:
           port: 80
           path: /index.html  #修改探测服务为存在的页面
           scheme: HTTP
         initialDelaySeconds: 3
         periodSeconds: 10
         timeoutSeconds: 3
   
   #暴露服务
   [root@k8snode1 k8s]# kubectl expose pod nginx-tcp-readiness-liveness-probe
   service/nginx-tcp-readiness-liveness-probe exposed
   
   #健康检测成功
   [root@k8snode1 k8s]# kubectl describe services nginx-tcp-readiness-liveness-probe
   Name:              nginx-tcp-readiness-liveness-probe
   Namespace:         default
   Labels:            app=nginx-readiness
   Annotations:       <none>
   Selector:          app=nginx-readiness
   Type:              ClusterIP
   IP Family Policy:  SingleStack
   IP Families:       IPv4
   IP:                10.104.193.231
   IPs:               10.104.193.231
   Port:              <unset>  80/TCP
   TargetPort:        80/TCP
   Endpoints:         172.16.98.212:80  #ip也进来了
   Session Affinity:  None
   Events:            <none>
   
   #服务也可以访问
   [root@k8snode1 k8s]# kubectl get service
   NAME                                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
   kubernetes                           ClusterIP   10.96.0.1        <none>        443/TCP        3d23h
   nginx-tcp-readiness-liveness-probe   ClusterIP   10.104.193.231   <none>        80/TCP         35s
   nginxdemo1                           NodePort    10.105.52.111    <none>        80:31204/TCP   2d7h
   [root@k8snode1 k8s]# curl 10.104.193.231
   <!DOCTYPE html>
   <html>
   <head>
   <title>Welcome to nginx!</title>
   <style>
       body {
           width: 35em;
           margin: 0 auto;
           font-family: Tahoma, Verdana, Arial, sans-serif;
       }
   </style>
   </head>
   <body>
   <h1>Welcome to nginx!</h1>
   <p>If you see this page, the nginx web server is successfully installed and
   working. Further configuration is required.</p>
   
   <p>For online documentation and support please refer to
   <a href="http://nginx.org/">nginx.org</a>.<br/>
   Commercial support is available at
   <a href="http://nginx.com/">nginx.com</a>.</p>
   
   <p><em>Thank you for using nginx.</em></p>
   </body>
   </html>
   
   如果中间再次探测失败的时候，就会把这个ip从service里摘除 负载均衡就会感知到 请求就不会转发到这里
   ```

6. startup启动探针 ，主要为了在程序初始化的时候，需要加载数据很久的时候，startup探针在前面探针之前 会连续探测，在探测多久没成功后，就会把程序进行重启

   文档：https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

   ```
   [root@k8snode1 k8s]# vim httpget-startup-liveness-readiness-probe.yaml
   
   apiVersion: v1
   kind: Pod
   metadata:
     name: nginx-tcp-startup-liveness-probe
     annotations:
       kubernetes.io/description: "nginx-tcp-liveness-probe"
     labels:
       app: nginx-startup
   spec:
     containers:
     - name: nginx-tcp-startup-liveness-probe
       image: nginx:1.7.9
       ports:
       - name: http-80-port
         protocol: TCP
         containerPort: 80
       startupProbe:
         httpGet:
           port: 80
           path: /start.html  
           scheme: HTTP
         periodSeconds: 10  #每隔多久探测一次
         failureThreshold: 30  #探测多少次
       livenessProbe: 
         httpGet:
           port: 80 
           path: /index.html
           scheme: HTTP
         initialDelaySeconds: 3 
         periodSeconds: 10
         timeoutSeconds: 3 
       readinessProbe:
         httpGet: 
           port: 80
           path: /index.html
           scheme: HTTP
         initialDelaySeconds: 3
         periodSeconds: 10
         timeoutSeconds: 3
   
   [root@k8snode1 k8s]# kubectl apply -f httpget-startup-liveness-readiness-probe.yaml
   pod/nginx-tcp-startup-liveness-probe created
   
   root@k8snode1 k8s]# kubectl describe pods nginx-tcp-startup-liveness-probe
   Name:         nginx-tcp-startup-liveness-probe
   Namespace:    default
   Priority:     0
   Node:         k8snode3/192.168.10.187
   Start Time:   Wed, 21 Apr 2021 22:43:37 +0800
   Labels:       app=nginx-startup
   Annotations:  cni.projectcalico.org/podIP: 172.16.98.213/32
                 cni.projectcalico.org/podIPs: 172.16.98.213/32
                 kubernetes.io/description: nginx-tcp-liveness-probe
   Status:       Running
   IP:           172.16.98.213
   IPs:
     IP:  172.16.98.213
   Containers:
     nginx-tcp-startup-liveness-probe:
       Container ID:   docker://bfdcaeb46cf1d692f14b3ed69c7d29a7fd1b0545ac1758416cf094c94d582a6a
       Image:          nginx:1.7.9
       Image ID:       docker-pullable://nginx@sha256:e3456c851a152494c3e4ff5fcc26f240206abac0c9d794affb40e0714846c451
       Port:           80/TCP
       Host Port:      0/TCP
       State:          Running
         Started:      Wed, 21 Apr 2021 22:43:38 +0800
       Ready:          False
       Restart Count:  0
       Liveness:       http-get http://:80/index.html delay=3s timeout=3s period=10s #success=1 #failure=3
       Readiness:      http-get http://:80/index.html delay=3s timeout=3s period=10s #success=1 #failure=3
       Startup:        http-get http://:80/start.html delay=0s timeout=1s period=10s #success=1 #failure=30
       Environment:    <none>
       Mounts:
         /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-fmmbs (ro)
   Conditions:
     Type              Status
     Initialized       True 
     Ready             False 
     ContainersReady   False 
     PodScheduled      True 
   Volumes:
     kube-api-access-fmmbs:
       Type:                    Projected (a volume that contains injected data from multiple sources)
       TokenExpirationSeconds:  3607
       ConfigMapName:           kube-root-ca.crt
       ConfigMapOptional:       <nil>
       DownwardAPI:             true
   QoS Class:                   BestEffort
   Node-Selectors:              <none>
   Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                                node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
   Events:
     Type     Reason     Age                      From               Message
     ----     ------     ----                     ----               -------
     Normal   Scheduled  41s                      default-scheduler  Successfully assigned default/nginx-tcp-startup-liveness-probe to k8snode3
     Normal   Pulled     26s                      kubelet            Container image "nginx:1.7.9" already present on machine
     Normal   Created    26s                      kubelet            Created container nginx-tcp-startup-liveness-probe
     Normal   Started    26s                      kubelet            Started container nginx-tcp-startup-liveness-probe
    #可以看到一直在做 Startup探测
    Warning  Unhealthy  <invalid> (x4 over 17s)  kubelet            Startup probe failed: HTTP probe failed with statuscode: 404
   
   
   #进入容器 手动创建一个文件
   [root@k8snode1 k8s]# kubectl exec -it nginx-tcp-startup-liveness-probe /bin/bash
   kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
   root@nginx-tcp-startup-liveness-probe:/# cd /usr/share/nginx/html/
   root@nginx-tcp-startup-liveness-probe:/usr/share/nginx/html# cp index.html start.html
   root@nginx-tcp-startup-liveness-probe:/usr/share/nginx/html# 
   
   #查看容器启动成功
   [root@k8snode1 k8s]# kubectl get pods nginx-tcp-startup-liveness-probe
   NAME                               READY   STATUS    RESTARTS   AGE
   nginx-tcp-startup-liveness-probe   1/1     Running   0          4m4s
   
   
   ```

7. 探针文档：https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

# lablels，标签和标签选择器

参考文档：https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/labels/

*标签（Labels）* 是附加到 Kubernetes 对象（比如 Pods）上的键值对。 标签旨在用于指定对用户有意义且相关的对象的标识属性，但不直接对核心系统有语义含义。 标签可以用于组织和选择对象的子集。标签可以在创建时附加到对象，随后可以随时添加和修改。 每个对象都可以定义一组键/值标签。每个键对于给定对象必须是唯一的。

主要是为了解决多个资源为一个集群的标识，可以指定是哪个控制器创建的

一个标签可以对应多个资源 同样的资源可以有多个标签 标签可以区分节点  pod service 等

通过selector标签选择器来管理标签

```
[root@k8shardway1 kubernetes]# vim web-dev-lables.yaml

#deploy
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-demo
  namespace: dev
spec:
  selector:  #标签选择器 一旦定义了 就不能变了
    matchLabels:  #条件 app=web-demo的时候 还有matchExpressions
      app: web-demo # 表明这个Deployment只负责app=web-demo的pod  这个标签只作用于当前Deployment
  replicas: 1
  template:  #模板 创建pod的时候用于这个模板
    metadata:
      labels:
        app: web-demo  #这个labels必须和上面的labels一样 否则会报错不一致
    spec:
      containers:
      - name: web-demo
        image: hub.mooc.com/kubernetes/web:v1
        ports:
        - containerPort: 8080
      nodeSecector: #还可以配置nodeSecector的select 指定落到哪个node节点上
        disktype: ssd
---
#service  service根据lables选择所有这个标签的pod 不区分Deployment
apiVersion: v1
kind: Service
metadata:
  name: web-demo
  namespace: dev
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app: web-demo
  type: ClusterIP

---
#ingress
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: web-demo
  namespace: dev
spec:
  rules:
  - host: web-dev.mooc.com
    http:
      paths:
      - path: /
        backend:
          serviceName: web-demo
          servicePort: 80

```



# Scheduler

Scheduler调度器，会收集每个work节点的资源，包括监控和运行什么样的内容，用来管理pod运行在哪个节点上的

1. informer会一直监听apiserver 会不会有需要调度的pod 如果有 就会放入到一个优先级队列
2. 有一个cache会一直从apiserver 拿到所有node的节点 来缓存node的资源类型
3. 下面就开始调度，
   1. 先拿到优先级队列的理信息和cache里的信息
   2. 先预选策略，过滤掉不符合资源要求的节点
   3. 下来到优选策略 考虑到亲和性和反亲和，尽量将pod分布在不通节点
   4. 把pod和node进行一个绑定 然后讲信息推送给apiserver
   5. apiserver接收到信息后 就会通知对应的kubelet 来启动pod

![](images\k8s-scheduler.png)

可以包含

1. 节点的亲和性 web-dev-node.yaml

   ```
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: web-demo-node
     namespace: dev
   spec:
     selector:
       matchLabels:
         app: web-demo-node
     replicas: 1
     template:
       metadata:
         labels:
           app: web-demo-node
       spec:
         containers:
         - name: web-demo-node
           image: hub.mooc.com/kubernetes/web:v1
           ports:
           - containerPort: 8080
         affinity:
           nodeAffinity:
             requiredDuringSchedulingIgnoredDuringExecution:  #必须要满足的条件 可以有多个 
               nodeSelectorTerms:
               - matchExpressions:
                 - key: beta.kubernetes.io/arch
                   operator: In
                   values:
                   - amd64
             preferredDuringSchedulingIgnoredDuringExecution:  #最好有这些条件满足 也可以不满足 可选权重条件
             - weight: 1
               preference:
                 matchExpressions:
                 - key: disktype
                   operator: NotIn
                   values:
                   - ssd
   
   ```

   

2. pod的亲和性 比如想和某些pod运行在一起 web-dev-pod.yaml

   ```
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: web-demo-pod
     namespace: dev
   spec:
     selector:
       matchLabels:
         app: web-demo-pod
     replicas: 1
     template:
       metadata:
         labels:
           app: web-demo-pod
       spec:
         containers:
         - name: web-demo-pod
           image: hub.mooc.com/kubernetes/web:v1
           ports:
           - containerPort: 8080
         affinity:
           podAffinity:
             requiredDuringSchedulingIgnoredDuringExecution:
             - labelSelector:
                 matchExpressions:  #要和app=web-demo的pod运行在同一个节点上
                 - key: app
                   operator: In
                   values:
                   - web-demo  #还可以改成自己不和自己运行在一台机器上  就是尽量将pod分布在不通节点
               topologyKey: kubernetes.io/hostname
             preferredDuringSchedulingIgnoredDuringExecution:
             - weight: 100
               podAffinityTerm:
                 labelSelector:
                   matchExpressions:
                   - key: app
                     operator: In
                     values:
                     - web-demo-node
                 topologyKey: kubernetes.io/hostname
   
   ```

   

3. 污点容忍，拒绝pod运行在这个节点上 web-dev-taint

   ```
   #设置一个污点 
   [root@k8shardway1 kubernetes]# kubectl taint nodes k8shardway2 gpu=true:NoSchedule
   
   #==========
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: web-demo-node
     namespace: dev
   spec:
     selector:
       matchLabels:
         app: web-demo-node
     replicas: 1
     template:
       metadata:
         labels:
           app: web-demo-node
       spec:
         containers:
         - name: web-demo-node
           image: hub.mooc.com/kubernetes/web:v1
           ports:
           - containerPort: 8080
        tolerations:  #污点容忍 可以调度到打了当gpu=true的时候并且effect: "NoSchedule"污点的节点上
        - key: "gpu"
          operator: "Equal"
          value: "true"
          effect: "NoSchedule"
   ```

   

# Controller控制器

参考：https://blog.51cto.com/happylab/2504398

集群内部的控制中心，保证集群内部所有服务处于一个正确状态

Pod是kubernetes所有运行应用或部署服务的基础，可以看作是k8s中运行的机器人，应用单独运行在Pod中不具备高级的特性，比如节点故障时Pod无法自动迁移，Pod多副本横向扩展，应用滚动升级RollingUpdate等，因此Pod一般不会单独使用，需要使用控制器来实现。

我们先看一个概念ReplicationController副本控制器，简称RC，副本控制是实现Pod高可用的基础，其通过定义副本的副本数replicas，当运行的Pod数量少于replicas时RC会自动创建Pod，当Pod数量多于replicas时RC会自动删除多余的Pod，确保当前运行的Pod和RC定义的副本数保持一致。

副本控制器包括Deployment，ReplicaSet，ReplicationController，StatefulSet等。其中常用有两个：Deployment和StatefulSet，Deployment用于无状态服务，StatefulSet用于有状态服务，ReplicaSet作为Deployment后端副本控制器，ReplicationController则是旧使用的副本控制器。

为了实现不同的功能，kubernetes中提供多种不同的控制器满足不同的业务场景，可以分为四类：

- Stateless application无状态化应用，如Deployment，ReplicaSet，RC等；
- Stateful application有状态化应用，需要保存数据状态，如数据库，数据库集群；
- Node daemon节点支撑守护，适用于在所有或部分节点运行Daemon，如日志，监控采集；
- Batch批处理任务，非长期运行服务，分为一次性运行Job和计划运行CronJob两种。

本文我们主要介绍无状态服务副本控制器的使用，包括Deployment，ReplicaSet和ReplicationController。

参考：https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/

1. ReplicaSet主要来维护副本数量

   ReplicaSet副本集简称RS，用于实现副本数的控制，通过上面的学习我们可以知道Deployment实际是调用ReplicaSet实现副本的控制，RS不具备滚动升级和回滚的特性，一般推荐使用Deployment，ReplicaSet的定义和Deployment差不多

   RepicaSet 是通过一组字段来定义的，包括一个用来识别可获得的 Pod 的集合的选择算符、一个用来标明应该维护的副本个数的数值、一个用来指定应该创建新 Pod 以满足副本个数条件时要使用的 Pod 模板等等。 每个 ReplicaSet 都通过根据需要创建和 删除 Pod 以使得副本个数达到期望值， 进而实现其存在价值。当 ReplicaSet 需要创建新的 Pod 时，会使用所提供的 Pod 模板。

   ReplicaSet一般和Deployment结合使用，Deployment底层就是调用ReplicaSet

   定义ReplicaSet

   ```
   [root@k8snode1 k8s]# vim  replicaset-demo.yaml 
   
   apiVersion: apps/v1
   kind: ReplicaSet
   metadata:
     name: replicaset-demo
     labels:
       controller: replicaset
     annotations:
       kubernetes.io/description: "Kubernetes Replication Controller Replication"
   spec:
     replicas: 3    #副本数
     selector:      #Pod标签选择器
       matchLabels:
         app: nginx
     template:      #创建Pod的模板
       metadata:
         labels:
           app: nginx  #名字要和matchLabels里的app名字一样
           version: 1.7.9
       spec:        #容器信息
         containers:
         - name: nginx-replicaset-demo
           image: nginx:1.7.9
           imagePullPolicy: IfNotPresent
           ports:
           - name: http-80-port
             protocol: TCP
             containerPort: 80
             
   [root@k8snode1 k8s]# kubectl apply -f replicaset-demo.yaml 
   replicaset.apps/replicaset-demo created
   
   #看到已经创建出来了
   [root@k8snode1 k8s]# kubectl get rs
   NAME                              DESIRED   CURRENT   READY   AGE
   nginx-dashboard-demo-5cd59c4f6d   3         3         3       2d5h
   nginx-dashboard-demo-85c9476ddc   0         0         0       2d5h
   nginxdemo1-5bb6dd6f49             0         0         0       2d8h
   nginxdemo1-6995675b4b             4         4         4       2d23h
   nginxdemo1-855bd4cd9f             0         0         0       2d8h
   replicaset-demo                   3         3         3       27s
   
   #查看详情
   [root@k8snode1 k8s]# kubectl get rs replicaset-demo -o yaml
   apiVersion: apps/v1
   kind: ReplicaSet
   metadata:
     annotations:
       kubectl.kubernetes.io/last-applied-configuration: |
         {"apiVersion":"apps/v1","kind":"ReplicaSet","metadata":{"annotations":{"kubernetes.io/description":"Kubernetes Replication Controller Replication"},"labels":{"controller":"replicaset"},"name":"replicaset-demo","namespace":"default"},"spec":{"replicas":3,"selector":{"matchLabels":{"app":"nginx"}},"template":{"metadata":{"labels":{"app":"nginx","version":"1.7.9"}},"spec":{"containers":[{"image":"nginx:1.7.9","imagePullPolicy":"IfNotPresent","name":"nginx-replicaset-demo","ports":[{"containerPort":80,"name":"http-80-port","protocol":"TCP"}]}]}}}}
       kubernetes.io/description: Kubernetes Replication Controller Replication
     creationTimestamp: "2021-04-21T15:29:10Z"
     generation: 1
     labels:
       controller: replicaset
     name: replicaset-demo
     namespace: default
     resourceVersion: "699090"
     uid: 18defc22-9a18-4d8b-bcde-e4901d3a7e03
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: nginx
     template:
       metadata:
         creationTimestamp: null
         labels:
           app: nginx
           version: 1.7.9
       spec:
         containers:
         - image: nginx:1.7.9
           imagePullPolicy: IfNotPresent
           name: nginx-replicaset-demo
           ports:
           - containerPort: 80
             name: http-80-port
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
     availableReplicas: 3
     fullyLabeledReplicas: 3
     observedGeneration: 1
     readyReplicas: 3
     replicas: 3
   
   #查看 三个副本已经创建出来了
   [root@k8snode1 k8s]# kubectl get pods -l app=nginx --show-labels 
   NAME                    READY   STATUS    RESTARTS   AGE    LABELS
   replicaset-demo-6k5s5   1/1     Running   0          2m5s   app=nginx,version=1.7.9
   replicaset-demo-dzcfx   1/1     Running   0          2m5s   app=nginx,version=1.7.9
   replicaset-demo-hkngm   1/1     Running   0          2m5s   app=nginx,version=1.7.9
   
   [root@k8snode1 k8s]# kubectl describe replicasets.apps replicaset-demo 
   Name:         replicaset-demo
   Namespace:    default
   Selector:     app=nginx   #rs是通过Selector选择器进行筛选的 凡是 app=nginx 的都会被认为是一个标签
   Labels:       controller=replicaset
   Annotations:  kubernetes.io/description: Kubernetes Replication Controller Replication
   Replicas:     3 current / 3 desired
   Pods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 Failed
   Pod Template:
     Labels:  app=nginx
              version=1.7.9
     Containers:
      nginx-replicaset-demo:
       Image:        nginx:1.7.9
       Port:         80/TCP
       Host Port:    0/TCP
       Environment:  <none>
       Mounts:       <none>
     Volumes:        <none>
   Events:
     Type    Reason            Age   From                   Message
     ----    ------            ----  ----                   -------
     Normal  SuccessfulCreate  3m8s  replicaset-controller  Created pod: replicaset-demo-6k5s5
     Normal  SuccessfulCreate  3m8s  replicaset-controller  Created pod: replicaset-demo-dzcfx
     Normal  SuccessfulCreate  3m8s  replicaset-controller  Created pod: replicaset-demo-hkngm
   
   
   #去掉一个pod的标签
   [root@k8snode1 k8s]# kubectl label pods replicaset-demo-6k5s5 app-
   pod/replicaset-demo-6k5s5 labeled
   
   #然后发现又创建了一个
   [root@k8snode1 k8s]# kubectl get pods -l app=nginx --show-labels 
   NAME                    READY   STATUS    RESTARTS   AGE    LABELS
   replicaset-demo-dzcfx   1/1     Running   0          5m8s   app=nginx,version=1.7.9
   replicaset-demo-hkngm   1/1     Running   0          5m8s   app=nginx,version=1.7.9
   replicaset-demo-tsh7f   1/1     Running   0          6s     app=nginx,version=1.7.9
   
   
   #rs扩容 到5个
   [root@k8snode1 k8s]# vim  replicaset-demo.yaml 
   
   apiVersion: apps/v1
   kind: ReplicaSet
   metadata:
     name: replicaset-demo
     labels:
       controller: replicaset
     annotations:
       kubernetes.io/description: "Kubernetes Replication Controller Replication"
   spec:
     replicas: 5    #副本数
     selector:      #Pod标签选择器
       matchLabels:
         app: nginx
     template:      #创建Pod的模板
       metadata:
         labels:
           app: nginx  #名字要和matchLabels里的app名字一样
           version: 1.7.9
       spec:        #容器信息
         containers:
         - name: nginx-replicaset-demo
           image: nginx:1.7.9
           imagePullPolicy: IfNotPresent
           ports:
           - name: http-80-port
             protocol: TCP
             containerPort: 80
             
   [root@k8snode1 k8s]# kubectl apply -f replicaset-demo.yaml 
   replicaset.apps/replicaset-demo configured
   [root@k8snode1 k8s]# kubectl get pods -l app=nginx --show-labels 
   NAME                    READY   STATUS    RESTARTS   AGE     LABELS
   replicaset-demo-dzcfx   1/1     Running   0          6m38s   app=nginx,version=1.7.9
   replicaset-demo-hkngm   1/1     Running   0          6m38s   app=nginx,version=1.7.9
   replicaset-demo-jbwg6   1/1     Running   0          4s      app=nginx,version=1.7.9
   replicaset-demo-kd8x6   1/1     Running   0          4s      app=nginx,version=1.7.9
   replicaset-demo-tsh7f   1/1     Running   0          96s     app=nginx,version=1.7.9
   
   
   [root@k8snode1 k8s]# kubectl get rs replicaset-demo -o yaml
   apiVersion: apps/v1
   kind: ReplicaSet
   metadata:
     annotations:
       kubectl.kubernetes.io/last-applied-configuration: |
         {"apiVersion":"apps/v1","kind":"ReplicaSet","metadata":{"annotations":{"kubernetes.io/description":"Kubernetes Replication Controller Replication"},"labels":{"controller":"replicaset"},"name":"replicaset-demo","namespace":"default"},"spec":{"replicas":5,"selector":{"matchLabels":{"app":"nginx"}},"template":{"metadata":{"labels":{"app":"nginx","version":"1.7.9"}},"spec":{"containers":[{"image":"nginx:1.7.9","imagePullPolicy":"IfNotPresent","name":"nginx-replicaset-demo","ports":[{"containerPort":80,"name":"http-80-port","protocol":"TCP"}]}]}}}}
       kubernetes.io/description: Kubernetes Replication Controller Replication
     creationTimestamp: "2021-04-21T15:29:10Z"
     generation: 2
     labels:
       controller: replicaset
     name: replicaset-demo
     namespace: default
     resourceVersion: "700078"
     uid: 18defc22-9a18-4d8b-bcde-e4901d3a7e03
   spec:
     replicas: 5
     selector:
       matchLabels:
         app: nginx
     template:
       metadata:
         creationTimestamp: null
         labels:
           app: nginx
           version: 1.7.9
       spec:
         containers:
         - image: nginx:1.7.9
           imagePullPolicy: IfNotPresent
           name: nginx-replicaset-demo
           ports:
           - containerPort: 80
             name: http-80-port
             protocol: TCP
           resources: {}
           terminationMessagePath: /dev/termination-log
           terminationMessagePolicy: File
         dnsPolicy: ClusterFirst
         restartPolicy: Always
         schedulerName: default-scheduler
         securityContext: {}
         terminationGracePeriodSeconds: 30
   status:  #也是通过status当前状态和上面的spec.replicas进行对比，对比出来不够了 就基于上面的模板新创建几个
     availableReplicas: 5
     fullyLabeledReplicas: 5
     observedGeneration: 2
     readyReplicas: 5
     replicas: 5
   
   ```

   ReplicaSet缺点：

   ​	无法在线更新，如果更新了模板，并不能更新之前创建的pod，只能修改玩模板之后，把之前的pod删掉。ReplicaSet就会自动基于新的模板创建

     回滚的话也是之前的流程

2. deployment

   Deployment是实现无状态应用副本控制器，其通过declarative申明式的方式定义Pod的副本数，Deployment的副本机制是通过ReplicaSet实现，replicas副本的管理通过在ReplicaSet中添加和删除Pod，RollingUpdate通过新建ReplicaSet，然后逐步移除和添加ReplicaSet中的Pod数量，从而实现滚动更新，使用Deployment的场景如下：

   - 滚动升级RollingUpdate，后台通过ReplicaSet实现
   - 多副本replicas实现，增加副本(高负载)或减少副本(低负载)
   - 应用回滚Rollout，版本更新支持回退

   ```
   [root@k8snode1 k8s]# vim deployment-demo.yaml 
   
   apiVersion: apps/v1
   kind: Deployment
   metadata:            #Deployment的元数据信息，包含名字，标签
     name: deployment-app
     labels:
       app: deployment-app
     annotations:
       kubernetes.io/replicationcontroller: Deployment
       kubernetes.io/description: "ReplicationController Deployment Demo"
   spec:
     replicas: 3          #副本数量，包含有3个Pod副本
     selector:            #标签选择器，选择管理包含指定标签的Pod
       matchLabels:
         app: deployment-app  
     template:       #如下是Pod的模板定义，没有apiVersion，Kind属性，需包含metadata定义
       metadata:          #Pod的元数据信息，必须包含有labels
         labels:
           app: deployment-app  #这个名字必须和metadata里的app一样 这个里所有信息都要保存一致
       spec:              #spec指定容器的属性信息
         containers:
         - name: deployment-app
           image: nginx:1.7.9
           imagePullPolicy: IfNotPresent
           ports:          #容器端口信息
           - name: http-80-port
             protocol: TCP
             containerPort: 80
           resources:      #资源管理,requests请求资源，limits限制资源
             requests:
               cpu: 100m
               memory: 128Mi
             limits:
               cpu: 200m
               memory: 256Mi
           livenessProbe:  #健康检查器livenessProbe，存活检查
             httpGet:
               path: /index.html
               port: 80
               scheme: HTTP
             initialDelaySeconds: 3
             periodSeconds: 5
             timeoutSeconds: 2
           readinessProbe:  #健康检查器readinessProbe，就绪检查
             httpGet:
               path: /index.html
               port: 80
               scheme: HTTP
             initialDelaySeconds: 3
             periodSeconds: 5
             timeoutSeconds: 2
   
   [root@k8snode1 k8s]# kubectl apply -f deployment-demo.yaml 
   deployment.apps/deployment-app created
   
   [root@k8snode1 k8s]# kubectl get deployments.apps deployment-app 
   NAME             READY   UP-TO-DATE   AVAILABLE   AGE
   deployment-app   3/3     3            3           24s
   
   [root@k8snode1 k8s]# kubectl describe deployments.apps deployment-app 
   Name:                   deployment-app
   Namespace:              default
   CreationTimestamp:      Wed, 21 Apr 2021 23:59:56 +0800
   Labels:                 app=deployment-app
   Annotations:            deployment.kubernetes.io/revision: 1
                           kubernetes.io/description: ReplicationController Deployment Demo
                           kubernetes.io/replicationcontroller: Deployment
   Selector:               app=deployment-app
   Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
   StrategyType:           RollingUpdate
   MinReadySeconds:        0
   RollingUpdateStrategy:  25% max unavailable, 25% max surge
   Pod Template:
     Labels:  app=deployment-app
     Containers:
      deployment-app:
       Image:      nginx:1.7.9
       Port:       80/TCP
       Host Port:  0/TCP
       Limits:
         cpu:     200m
         memory:  256Mi
       Requests:
         cpu:        100m
         memory:     128Mi
       Liveness:     http-get http://:80/index.html delay=3s timeout=2s period=5s #success=1 #failure=3
       Readiness:    http-get http://:80/index.html delay=3s timeout=2s period=5s #success=1 #failure=3
       Environment:  <none>
       Mounts:       <none>
     Volumes:        <none>
   Conditions:
     Type           Status  Reason
     ----           ------  ------
     Available      True    MinimumReplicasAvailable
     Progressing    True    NewReplicaSetAvailable
   OldReplicaSets:  <none>
   NewReplicaSet:   deployment-app-569bbb967c (3/3 replicas created)
   Events:
     Type    Reason             Age   From                   Message
     ----    ------             ----  ----                   -------
     Normal  ScalingReplicaSet  34s   deployment-controller  Scaled up replica set deployment-app-569bbb967c to 3 #这个就是创建的replicaset
   
   [root@k8snode1 k8s]# kubectl get replicasets.apps deployment-app-569bbb967c
   NAME                        DESIRED   CURRENT   READY   AGE
   deployment-app-569bbb967c   3         3         3       78s
   
   [root@k8snode1 k8s]# kubectl get deployments.apps deployment-app -o yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     annotations:
       deployment.kubernetes.io/revision: "1"
       kubectl.kubernetes.io/last-applied-configuration: |
         {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{"kubernetes.io/description":"ReplicationController Deployment Demo","kubernetes.io/replicationcontroller":"Deployment"},"labels":{"app":"deployment-app"},"name":"deployment-app","namespace":"default"},"spec":{"replicas":3,"selector":{"matchLabels":{"app":"deployment-app"}},"template":{"metadata":{"labels":{"app":"deployment-app"}},"spec":{"containers":[{"image":"nginx:1.7.9","imagePullPolicy":"IfNotPresent","livenessProbe":{"httpGet":{"path":"/index.html","port":80,"scheme":"HTTP"},"initialDelaySeconds":3,"periodSeconds":5,"timeoutSeconds":2},"name":"deployment-app","ports":[{"containerPort":80,"name":"http-80-port","protocol":"TCP"}],"readinessProbe":{"httpGet":{"path":"/index.html","port":80,"scheme":"HTTP"},"initialDelaySeconds":3,"periodSeconds":5,"timeoutSeconds":2},"resources":{"limits":{"cpu":"200m","memory":"256Mi"},"requests":{"cpu":"100m","memory":"128Mi"}}}]}}}}
       kubernetes.io/description: ReplicationController Deployment Demo
       kubernetes.io/replicationcontroller: Deployment
     creationTimestamp: "2021-04-21T15:59:56Z"
     generation: 1
     labels:
       app: deployment-app
     name: deployment-app
     namespace: default
     resourceVersion: "703619"
     uid: 20ea5e79-4669-43c3-8b02-e1e2fa558986
   spec:
     progressDeadlineSeconds: 600
     replicas: 3
     revisionHistoryLimit: 10
     selector:
       matchLabels:
         app: deployment-app
     strategy:  #可以看到 Deployment会自动添加一个滚动更新的策略
       rollingUpdate:
         maxSurge: 25%
         maxUnavailable: 25%
       type: RollingUpdate
     template:
       metadata:
         creationTimestamp: null
         labels:
           app: deployment-app
       spec:
         containers:
         - image: nginx:1.7.9
           imagePullPolicy: IfNotPresent
           livenessProbe:
             failureThreshold: 3
             httpGet:
               path: /index.html
               port: 80
               scheme: HTTP
             initialDelaySeconds: 3
             periodSeconds: 5
             successThreshold: 1
             timeoutSeconds: 2
           name: deployment-app
           ports:
           - containerPort: 80
             name: http-80-port
             protocol: TCP
           readinessProbe:
             failureThreshold: 3
             httpGet:
               path: /index.html
               port: 80
               scheme: HTTP
             initialDelaySeconds: 3
             periodSeconds: 5
             successThreshold: 1
             timeoutSeconds: 2
           resources:
             limits:
               cpu: 200m
               memory: 256Mi
             requests:
               cpu: 100m
               memory: 128Mi
           terminationMessagePath: /dev/termination-log
           terminationMessagePolicy: File
         dnsPolicy: ClusterFirst
         restartPolicy: Always
         schedulerName: default-scheduler
         securityContext: {}
         terminationGracePeriodSeconds: 30
   status:
     availableReplicas: 3
     conditions:
     - lastTransitionTime: "2021-04-21T16:00:01Z"
       lastUpdateTime: "2021-04-21T16:00:01Z"
       message: Deployment has minimum availability.
       reason: MinimumReplicasAvailable
       status: "True"
       type: Available
     - lastTransitionTime: "2021-04-21T15:59:56Z"
       lastUpdateTime: "2021-04-21T16:00:01Z"
       message: ReplicaSet "deployment-app-569bbb967c" has successfully progressed.
       reason: NewReplicaSetAvailable
       status: "True"
       type: Progressing
     observedGeneration: 1
     readyReplicas: 3
     replicas: 3
     updatedReplicas: 3
   
   
   [root@k8snode1 k8s]# kubectl get replicasets.apps deployment-app-569bbb967c -o yaml
   apiVersion: apps/v1
   kind: ReplicaSet
   metadata:
     annotations:
       deployment.kubernetes.io/desired-replicas: "3"
       deployment.kubernetes.io/max-replicas: "4"
       deployment.kubernetes.io/revision: "1"
       kubernetes.io/description: ReplicationController Deployment Demo
       kubernetes.io/replicationcontroller: Deployment
     creationTimestamp: "2021-04-21T15:59:56Z"
     generation: 1
     labels:
       app: deployment-app
       pod-template-hash: 569bbb967c
     name: deployment-app-569bbb967c
     namespace: default
     ownerReferences:
     - apiVersion: apps/v1
       blockOwnerDeletion: true
       controller: true
       kind: Deployment
       name: deployment-app
       uid: 20ea5e79-4669-43c3-8b02-e1e2fa558986
     resourceVersion: "703617"
     uid: dbbdf1cc-abd5-4573-a742-6158ded7e029
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: deployment-app  
         pod-template-hash: 569bbb967c  #也在rs里添加了一些matchLabels，来标记这个rs属于哪个Deployment
     template:
       metadata:
         creationTimestamp: null
         labels:
           app: deployment-app  #归属于哪个Deployment
           pod-template-hash: 569bbb967c
       spec:
         containers:
         - image: nginx:1.7.9
           imagePullPolicy: IfNotPresent
           livenessProbe:
             failureThreshold: 3
             httpGet:
               path: /index.html
               port: 80
               scheme: HTTP
             initialDelaySeconds: 3
             periodSeconds: 5
             successThreshold: 1
             timeoutSeconds: 2
           name: deployment-app
           ports:
           - containerPort: 80
             name: http-80-port
             protocol: TCP
           readinessProbe:
             failureThreshold: 3
             httpGet:
               path: /index.html
               port: 80
               scheme: HTTP
             initialDelaySeconds: 3
             periodSeconds: 5
             successThreshold: 1
             timeoutSeconds: 2
           resources:
             limits:
               cpu: 200m
               memory: 256Mi
             requests:
               cpu: 100m
               memory: 128Mi
           terminationMessagePath: /dev/termination-log
           terminationMessagePolicy: File
         dnsPolicy: ClusterFirst
         restartPolicy: Always
         schedulerName: default-scheduler
         securityContext: {}
         terminationGracePeriodSeconds: 30
   status:
     availableReplicas: 3
     fullyLabeledReplicas: 3
     observedGeneration: 1
     readyReplicas: 3
     replicas: 3
   
   [root@k8snode1 k8s]# kubectl get pods -l pod-template-hash=569bbb967c --show-labels
   NAME                              READY   STATUS    RESTARTS   AGE    LABELS
   deployment-app-569bbb967c-646tp   1/1     Running   0          7m2s   app=deployment-app,pod-template-hash=569bbb967c
   deployment-app-569bbb967c-c8r9h   1/1     Running   0          7m2s   app=deployment-app,pod-template-hash=569bbb967c
   deployment-app-569bbb967c-sp64k   1/1     Running   0          7m2s   app=deployment-app,pod-template-hash=569bbb967c
   
   
   #扩容deployment
   [root@k8snode1 k8s]# vim deployment-demo.yaml 
   
   apiVersion: apps/v1
   kind: Deployment
   metadata:            #Deployment的元数据信息，包含名字，标签
     name: deployment-app
     labels:
       app: deployment-app
     annotations:
       kubernetes.io/replicationcontroller: Deployment
       kubernetes.io/description: "ReplicationController Deployment Demo"
   spec:
     replicas: 5          #副本数量，包含有3个Pod副本
     selector:            #标签选择器，选择管理包含指定标签的Pod
       matchLabels:
         app: deployment-app
     template:       #如下是Pod的模板定义，没有apiVersion，Kind属性，需包含metadata定义
       metadata:          #Pod的元数据信息，必须包含有labels
         labels:
           app: deployment-app  #这个名字必须和metadata里的app一样 这个里所有信息都要保存一致
       spec:              #spec指定容器的属性信息
         containers:
         - name: deployment-app
           image: nginx:1.7.9
           imagePullPolicy: IfNotPresent
           ports:          #容器端口信息
           - name: http-80-port
             protocol: TCP
             containerPort: 80
           resources:      #资源管理,requests请求资源，limits限制资源
             requests:
               cpu: 100m
               memory: 128Mi
             limits:
               cpu: 200m
               memory: 256Mi
           livenessProbe:  #健康检查器livenessProbe，存活检查
             httpGet:
               path: /index.html
               port: 80
               scheme: HTTP
             initialDelaySeconds: 3
             periodSeconds: 5
             timeoutSeconds: 2
           readinessProbe:  #健康检查器readinessProbe，就绪检查
             httpGet:
               path: /index.html
               port: 80
               scheme: HTTP
             initialDelaySeconds: 3
             periodSeconds: 5
             timeoutSeconds: 2
   
   [root@k8snode1 k8s]# kubectl apply -f deployment-demo.yaml 
   deployment.apps/deployment-app configured
   
   [root@k8snode1 k8s]# kubectl get replicasets.apps deployment-app-569bbb967c
   NAME                        DESIRED   CURRENT   READY   AGE
   deployment-app-569bbb967c   5         5         5       8m37s
   
   #原理一样也是通过status来对比进行扩容的，区别就是 扩容是通过replicaset进行的
   ```

# 三种常见应用发布方式

参考：https://www.jianshu.com/p/e9f45236aabd?utm_campaign=hugo

- 蓝绿发布

  将应用所在集群上的机器从逻辑上分为A/B两组。在新版发布时，首先把A组的机器从负载均衡中摘除，再进行新版本的部署。此时，B组仍然继续提供服务。

  当A组升级完毕后，负载均衡重新接入A组，再把B组从负载列表中摘除，进行新版本的部署。A组重新提供服务。

  特点

  - 如果出问题，影响面较广，或者说很难控制具体的影响面；
  - 发布策略简单；
  - 用户无感知，平滑过渡；
  - 升级/回滚速度快。

  缺点

  - 需要准备正常业务使用资源的两倍以上服务器，防止升级期间单组无法承载业务突发；
  - 短时间内浪费一定资源成本；
  - 基础设施无改动，增大升级稳定性。

- 灰度发布

  灰度发布的基本含义是只升级部分服务，即让一部分用户继续用老版本，一部分用户开始用新版本，如果用户对新版本没什么意见，那么逐步扩大范围，把所有用户都迁移到新版本上面来。例如，阿里这边的典型做法是在用户登录时携带的cookie中埋点，使其携带一些属性信息，例如corpId/zoneID/isInner，然后根据这些埋点的属性进行组合，来实现灰度策略。例如，可以允许isInner=true && corpId=123的用户访问到新功能。

  特点

  - 保证整体系统稳定性，在初始灰度的时候就可以发现、调整问题，影响范围可控；
  - 新功能逐步评估性能，稳定性和健康状况，如果出问题影响范围很小，相对用户体验也少；
  - 用户无感知，平滑过渡。

  缺点

  - 需要灰度平台的支持，对基础设施的自动化水平要求较高

  部署过程

  - 从LB摘掉灰度服务器，升级成功后再加入LB；
  - 少量用户流量到新版本；
  - 如果灰度服务器测试成功，升级剩余服务器。

  灰度发布是通过切换线上并存版本之间的路由权重，逐步从一个版本切换为另一个版本的过程。

  

- 滚动发布

  滚动发布是指每次只升级一个或多个服务，升级完成后加入生产环境，不断执行这个过程，直到集群中的全部旧版本升级新版本。

  特点

  - 用户无感知，平滑过渡；
  - 节约资源。

  缺点

  - 部署时间慢，取决于每阶段更新时间；
  - 发布策略较复杂；
  - 无法确定OK的环境，不易回滚。

  部署过程

  - 先升级1个副本，主要做部署验证；
  - 每次升级副本，自动从LB上摘掉，升级成功后自动加入集群；
  - 事先需要有自动更新策略，分为若干次，每次数量/百分比可配置；
  - 回滚是发布的逆过程，先从LB摘掉新版本，再升级老版本，这个过程一般时间比较长；
  - 自动化要求高。

- 金丝雀发布

  金丝雀发布是灰度发布的一种。灰度发布是指在黑与白之间，能够平滑过渡的一种发布方式。即在发布过程中一部分用户继续使用老版本，一部分用户使用新版本，不断地扩大新版本的访问流量。最终实现老版本到新版本的过度。由于金丝雀对瓦斯极其敏感，因此以前旷工开矿下矿洞前，先会放一只金丝雀进去探是否有有毒气体，看金丝雀能否活下来，金丝雀发布由此得名。发布过程中，先发一台或者一小部分比例的机器作为金丝雀，用于流量验证。如果金丝雀验证通过则把剩余机器全部发掉。如果金丝雀验证失败，则直接回退金丝雀。金丝雀发布的优势在于可以用少量用户来验证新版本功能，这样即使有问题所影响的也是很小的一部分客户。如果对新版本功能或性能缺乏足够信心那么就可以采用这种方式。这种方式也有其缺点，金丝雀发布本质上仍然是一次性的全量发布，发布过程中用户体验并不平滑，有些隐藏深处的bug少量用户可能并不能验证出来问题，需要逐步扩大流量才可以。

  1. 蓝绿发布

     ```
     #把之前的加到services里
     [root@k8snode1 k8s]# kubectl expose deployment deployment-app 
     service/deployment-app exposed
     
     [root@k8snode1 k8s]# kubectl get services deployment-app 
     NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
     deployment-app   ClusterIP   10.111.12.80   <none>        80/TCP    16s
     
     [root@k8snode1 k8s]# kubectl get services deployment-app -o yaml
     apiVersion: v1
     kind: Service
     metadata:
       creationTimestamp: "2021-04-21T16:14:29Z"
       labels:
         app: deployment-app
       name: deployment-app
       namespace: default
       resourceVersion: "705731"
       uid: aea9071a-31c3-4181-bdfa-9e4da1acf52d
     spec:
       clusterIP: 10.111.12.80
       clusterIPs:
       - 10.111.12.80
       ipFamilies:
       - IPv4
       ipFamilyPolicy: SingleStack
       ports:
       - port: 80
         protocol: TCP
         targetPort: 80
       selector:
         app: deployment-app
       sessionAffinity: None
       type: ClusterIP
     status:
       loadBalancer: {}
     
     #可以看到为1.7.9
     [root@k8snode1 k8s]# curl -I 10.111.12.80
     HTTP/1.1 200 OK
     Server: nginx/1.7.9
     Date: Wed, 21 Apr 2021 16:15:43 GMT
     Content-Type: text/html
     Content-Length: 612
     Last-Modified: Tue, 23 Dec 2014 16:25:09 GMT
     Connection: keep-alive
     ETag: "54999765-264"
     Accept-Ranges: bytes
     
     
     #部署一个新版本
     [root@k8snode1 k8s]# vim deployment-demo-new.yaml 
     
     apiVersion: apps/v1
     kind: Deployment
     metadata:            #Deployment的元数据信息，包含名字，标签
       name: deployment-app-new
       labels:
         app: deployment-app-new
       annotations:
         kubernetes.io/replicationcontroller: Deployment
         kubernetes.io/description: "ReplicationController Deployment Demo"
     spec:
       replicas: 3          #副本数量，包含有3个Pod副本
       selector:            #标签选择器，选择管理包含指定标签的Pod
         matchLabels:
           app: deployment-app-new
       template:       #如下是Pod的模板定义，没有apiVersion，Kind属性，需包含metadata定义
         metadata:          #Pod的元数据信息，必须包含有labels
           labels:
             app: deployment-app-new  #这个名字必须和metadata里的app一样 这个里所有信息都要保存一致
         spec:              #spec指定容器的属性信息
           containers:
           - name: deployment-app-new
             image: nginx:1.8.1
             imagePullPolicy: IfNotPresent
             ports:          #容器端口信息
             - name: http-80-port
               protocol: TCP
               containerPort: 80
             resources:      #资源管理,requests请求资源，limits限制资源
               requests:
                 cpu: 100m
                 memory: 128Mi
               limits:
                 cpu: 200m
                 memory: 256Mi
             livenessProbe:  #健康检查器livenessProbe，存活检查
               httpGet:
                 path: /index.html
                 port: 80
                 scheme: HTTP
               initialDelaySeconds: 3
               periodSeconds: 5
               timeoutSeconds: 2
             readinessProbe:  #健康检查器readinessProbe，就绪检查
               httpGet:
                 path: /index.html
                 port: 80
                 scheme: HTTP
               initialDelaySeconds: 3
               periodSeconds: 5
               timeoutSeconds: 2
     
     [root@k8snode1 k8s]# kubectl apply -f deployment-demo-new.yaml 
     deployment.apps/deployment-app-new created
     
     #切换，将旧的应用切换到新的应用中
     #之前发布的版本
     [root@k8snode1 k8s]# kubectl get pods --show-labels -l app=deployment-app
     NAME                              READY   STATUS    RESTARTS   AGE   LABELS
     deployment-app-569bbb967c-646tp   1/1     Running   0          18m   app=deployment-app,pod-template-hash=569bbb967c
     deployment-app-569bbb967c-bgqwh   1/1     Running   0          10m   app=deployment-app,pod-template-hash=569bbb967c
     deployment-app-569bbb967c-c8r9h   1/1     Running   0          18m   app=deployment-app,pod-template-hash=569bbb967c
     deployment-app-569bbb967c-g4lwh   1/1     Running   0          10m   app=deployment-app,pod-template-hash=569bbb967c
     deployment-app-569bbb967c-sp64k   1/1     Running   0          18m   app=deployment-app,pod-template-hash=569bbb967c
     
     #新的
     [root@k8snode1 k8s]# kubectl get pods --show-labels -l app=deployment-app-new
     NAME                                  READY   STATUS    RESTARTS   AGE     LABELS
     deployment-app-new-76dc776c96-7s4g6   1/1     Running   0          2m32s   app=deployment-app-new,pod-template-hash=76dc776c96
     deployment-app-new-76dc776c96-8gtdq   1/1     Running   0          2m32s   app=deployment-app-new,pod-template-hash=76dc776c96
     deployment-app-new-76dc776c96-j69cx   1/1     Running   0          2m32s   app=deployment-app-new,pod-template-hash=76dc776c96
     
     
     #修改service的 直接改标签 流量就过来了
     [root@k8snode1 k8s]# kubectl edit service deployment-app 
     
     # Please edit the object below. Lines beginning with a '#' will be ignored,
     # and an empty file will abort the edit. If an error occurs while saving this file will be
     # reopened with the relevant failures.
     #
     apiVersion: v1
     kind: Service
     metadata:
       creationTimestamp: "2021-04-21T16:14:29Z"
       labels:
         app: deployment-app
       name: deployment-app
       namespace: default
       resourceVersion: "705731"
       uid: aea9071a-31c3-4181-bdfa-9e4da1acf52d
     spec:
       clusterIP: 10.111.12.80
       clusterIPs:
       - 10.111.12.80
       ipFamilies:
       - IPv4
       ipFamilyPolicy: SingleStack
       ports:
       - port: 80
         protocol: TCP
         targetPort: 80
       selector:
         app: deployment-app-new  #修改为新的标签
       sessionAffinity: None
       type: ClusterIP
     status:
       loadBalancer: {}
     
     [root@k8snode1 k8s]# kubectl get service deployment-app
     NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
     deployment-app   ClusterIP   10.111.12.80   <none>        80/TCP    6m43s
     
     #发现已经切换过来了 如果失败了 还可以恢复回去
     [root@k8snode1 k8s]# curl -I 10.111.12.80
     HTTP/1.1 200 OK
     Server: nginx/1.8.1
     Date: Wed, 21 Apr 2021 16:21:37 GMT
     Content-Type: text/html
     Content-Length: 612
     Last-Modified: Tue, 26 Jan 2016 15:24:47 GMT
     Connection: keep-alive
     ETag: "56a78fbf-264"
     Accept-Ranges: bytes
     
     ```

  2. 滚动更新，滚动更新是一个一个进行替换的，先更新一部分，将流量打过来，然后再更新另一部分

     ```
     [root@k8snode1 k8s]# kubectl get deployments.apps deployment-app-new -o yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       annotations:
         deployment.kubernetes.io/revision: "1"
         kubectl.kubernetes.io/last-applied-configuration: |
           {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{"kubernetes.io/description":"ReplicationController Deployment Demo","kubernetes.io/replicationcontroller":"Deployment"},"labels":{"app":"deployment-app-new"},"name":"deployment-app-new","namespace":"default"},"spec":{"replicas":3,"selector":{"matchLabels":{"app":"deployment-app-new"}},"template":{"metadata":{"labels":{"app":"deployment-app-new"}},"spec":{"containers":[{"image":"nginx:1.8.1","imagePullPolicy":"IfNotPresent","livenessProbe":{"httpGet":{"path":"/index.html","port":80,"scheme":"HTTP"},"initialDelaySeconds":3,"periodSeconds":5,"timeoutSeconds":2},"name":"deployment-app-new","ports":[{"containerPort":80,"name":"http-80-port","protocol":"TCP"}],"readinessProbe":{"httpGet":{"path":"/index.html","port":80,"scheme":"HTTP"},"initialDelaySeconds":3,"periodSeconds":5,"timeoutSeconds":2},"resources":{"limits":{"cpu":"200m","memory":"256Mi"},"requests":{"cpu":"100m","memory":"128Mi"}}}]}}}}
         kubernetes.io/description: ReplicationController Deployment Demo
         kubernetes.io/replicationcontroller: Deployment
       creationTimestamp: "2021-04-21T16:17:08Z"
       generation: 1
       labels:
         app: deployment-app-new
       name: deployment-app-new
       namespace: default
       resourceVersion: "706182"
       uid: 93850597-5d6b-44c4-81ff-d403b8318bfc
     spec:
       progressDeadlineSeconds: 600
       replicas: 3
       revisionHistoryLimit: 10
       selector:
         matchLabels:
           app: deployment-app-new
       strategy:   #更新策略
         rollingUpdate:
           maxSurge: 25%  #第一次更新可以创建多少的pod为新的
           maxUnavailable: 25%
         type: RollingUpdate
       template:
         metadata:
           creationTimestamp: null
           labels:
             app: deployment-app-new
         spec:
           containers:
           - image: nginx:1.8.1
             imagePullPolicy: IfNotPresent
             livenessProbe:
               failureThreshold: 3
               httpGet:
                 path: /index.html
                 port: 80
                 scheme: HTTP
               initialDelaySeconds: 3
               periodSeconds: 5
               successThreshold: 1
               timeoutSeconds: 2
             name: deployment-app-new
             ports:
             - containerPort: 80
               name: http-80-port
               protocol: TCP
             readinessProbe:
               failureThreshold: 3
               httpGet:
                 path: /index.html
                 port: 80
                 scheme: HTTP
               initialDelaySeconds: 3
               periodSeconds: 5
               successThreshold: 1
               timeoutSeconds: 2
             resources:
               limits:
                 cpu: 200m
                 memory: 256Mi
               requests:
                 cpu: 100m
                 memory: 128Mi
             terminationMessagePath: /dev/termination-log
             terminationMessagePolicy: File
           dnsPolicy: ClusterFirst
           restartPolicy: Always
           schedulerName: default-scheduler
           securityContext: {}
           terminationGracePeriodSeconds: 30
     status:
       availableReplicas: 3
       conditions:
       - lastTransitionTime: "2021-04-21T16:17:13Z"
         lastUpdateTime: "2021-04-21T16:17:13Z"
         message: Deployment has minimum availability.
         reason: MinimumReplicasAvailable
         status: "True"
         type: Available
       - lastTransitionTime: "2021-04-21T16:17:08Z"
         lastUpdateTime: "2021-04-21T16:17:13Z"
         message: ReplicaSet "deployment-app-new-76dc776c96" has successfully progressed.
         reason: NewReplicaSetAvailable
         status: "True"
         type: Progressing
       observedGeneration: 1
       readyReplicas: 3
       replicas: 3
       updatedReplicas: 3
     
     
     #看看更新策略的方式
     [root@k8snode1 k8s]# kubectl explain deployments.spec.strategy
     KIND:     Deployment
     VERSION:  apps/v1
     
     RESOURCE: strategy <Object>
     
     DESCRIPTION:
          The deployment strategy to use to replace existing pods with new ones.
     
          DeploymentStrategy describes how to replace existing pods with new ones.
     
     FIELDS:
        rollingUpdate	<Object>
          Rolling update config params. Present only if DeploymentStrategyType =
          RollingUpdate.
     
        type	<string>
          Type of deployment. Can be "Recreate" or "RollingUpdate". Default is
          RollingUpdate. #Recreate直接创建新的 RollingUpdate逐步替换
     
     #调整更新策略
     [root@k8snode1 k8s]# vim deployment-demo-new.yaml 
     
     apiVersion: apps/v1
     kind: Deployment
     metadata:            #Deployment的元数据信息，包含名字，标签
       name: deployment-app-new
       labels:
         app: deployment-app-new
       annotations:
         kubernetes.io/replicationcontroller: Deployment
         kubernetes.io/description: "ReplicationController Deployment Demo"
     spec:
       replicas: 3          #副本数量，包含有3个Pod副本
       selector:            #标签选择器，选择管理包含指定标签的Pod
         matchLabels:
           app: deployment-app-new
       strategy:
         type: RollingUpdate  #更新策略
         rollingUpdate:
           maxSurge: 1  #最大更新一个pod
           maxUnavailable: 2 #更新的时候最大多少副本不可用 
       template:       #如下是Pod的模板定义，没有apiVersion，Kind属性，需包含metadata定义
         metadata:          #Pod的元数据信息，必须包含有labels
           labels:
             app: deployment-app-new  #这个名字必须和metadata里的app一样 这个里所有信息都要保存一致
         spec:              #spec指定容器的属性信息
           containers:
           - name: deployment-app-new
             image: nginx:1.7.9
             imagePullPolicy: IfNotPresent
             ports:          #容器端口信息
             - name: http-80-port
               protocol: TCP
               containerPort: 80
             resources:      #资源管理,requests请求资源，limits限制资源
               requests:
                 cpu: 100m
     			memory: 128Mi
               limits:
                 cpu: 200m
                 memory: 256Mi
             livenessProbe:  #健康检查器livenessProbe，存活检查
               httpGet:
                 path: /index.html
                 port: 80
                 scheme: HTTP
               initialDelaySeconds: 3
               periodSeconds: 5
               timeoutSeconds: 2
             readinessProbe:  #健康检查器readinessProbe，就绪检查
               httpGet:
                 path: /index.html
                 port: 80
                 scheme: HTTP
               initialDelaySeconds: 3
               periodSeconds: 5
               timeoutSeconds: 2
     
     #先扩容到3副本
     [root@k8snode1 k8s]# kubectl scale --replicas=3 deployment deployment-app-new
     deployment.apps/deployment-app scaled
     
     NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
     deployment-app         3/3     3            3           12h
     deployment-app-new     3/3     3            3           12h
     nginx-dashboard-demo   3/3     3            3           2d18h
     nginxdemo1             4/4     4            4           3d12h
     
     [root@k8snode1 k8s]# kubectl apply -f deployment-demo-new.yaml 
     deployment.apps/deployment-app-new configured
     
     #从日志可以看到，先创建了一个新的pod 然后运行，在删除删除之前的 在进行更新之后的
     [root@k8snode1 ~]# kubectl get pod -l app=deployment-app-new -w
     NAME                                  READY   STATUS    RESTARTS   AGE
     deployment-app-new-76dc776c96-7s4g6   1/1     Running   0          12h
     deployment-app-new-76dc776c96-8gtdq   1/1     Running   0          12h
     deployment-app-new-76dc776c96-j69cx   1/1     Running   0          12h
     deployment-app-new-7c68576649-dftzd   0/1     Pending   0          0s
     deployment-app-new-7c68576649-dftzd   0/1     Pending   0          0s
     deployment-app-new-76dc776c96-7s4g6   1/1     Terminating   0          12h
     deployment-app-new-76dc776c96-j69cx   1/1     Terminating   0          12h
     deployment-app-new-7c68576649-dftzd   0/1     ContainerCreating   0          0s
     deployment-app-new-7c68576649-fjwrg   0/1     Pending             0          0s
     deployment-app-new-7c68576649-srpkw   0/1     Pending             0          0s
     deployment-app-new-7c68576649-fjwrg   0/1     Pending             0          0s
     deployment-app-new-7c68576649-srpkw   0/1     Pending             0          0s
     deployment-app-new-7c68576649-fjwrg   0/1     ContainerCreating   0          0s
     deployment-app-new-7c68576649-srpkw   0/1     ContainerCreating   0          0s
     deployment-app-new-76dc776c96-j69cx   1/1     Terminating         0          12h
     deployment-app-new-76dc776c96-7s4g6   1/1     Terminating         0          12h
     deployment-app-new-7c68576649-fjwrg   0/1     ContainerCreating   0          1s
     deployment-app-new-7c68576649-dftzd   0/1     ContainerCreating   0          1s
     deployment-app-new-7c68576649-srpkw   0/1     ContainerCreating   0          1s
     deployment-app-new-76dc776c96-j69cx   0/1     Terminating         0          12h
     deployment-app-new-76dc776c96-7s4g6   0/1     Terminating         0          12h
     deployment-app-new-7c68576649-fjwrg   0/1     Running             0          1s
     deployment-app-new-7c68576649-dftzd   0/1     Running             0          3s
     deployment-app-new-76dc776c96-7s4g6   0/1     Terminating         0          12h
     deployment-app-new-76dc776c96-7s4g6   0/1     Terminating         0          12h
     deployment-app-new-7c68576649-srpkw   0/1     Running             0          3s
     deployment-app-new-76dc776c96-j69cx   0/1     Terminating         0          12h
     deployment-app-new-76dc776c96-j69cx   0/1     Terminating         0          12h
     deployment-app-new-7c68576649-dftzd   1/1     Running             0          5s
     deployment-app-new-76dc776c96-8gtdq   1/1     Terminating         0          12h
     deployment-app-new-76dc776c96-8gtdq   1/1     Terminating         0          12h
     deployment-app-new-7c68576649-fjwrg   1/1     Running             0          5s
     deployment-app-new-7c68576649-srpkw   1/1     Running             0          5s
     deployment-app-new-76dc776c96-8gtdq   0/1     Terminating         0          12h
     deployment-app-new-76dc776c96-8gtdq   0/1     Terminating         0          12h
     deployment-app-new-76dc776c96-8gtdq   0/1     Terminating         0          12h
     
     #查看版本状态
     [root@k8snode1 k8s]# kubectl rollout history deployment deployment-app-new 
     deployment.apps/deployment-app-new 
     REVISION  CHANGE-CAUSE
     1         <none>
     2         <none>
     
     
     #退回到某个版本
     [root@k8snode1 ~]# kubectl rollout undo deployment deployment-app-new --to-revision=1
     
     [root@k8snode1 k8s]# kubectl rollout undo deployment deployment-app-new --to-revision=1
     deployment.apps/deployment-app-new rolled back
     ```

  3. 金丝雀发布 先发出一个探测 然后再通过手动的发布停止来达到金丝雀发布的目的

     ```
     #先修改一个容器的image镜像，然后再立马停止，就会发现 只更新了一个 看看这个更新了容器的进程是否正常
     [root@k8snode1 ~]# kubectl set image deployment deployment-app-new deployment-app-new=nginx:1.7.9
     deployment.apps/deployment-app-new image updated
     [root@k8snode1 ~]#  kubectl rollout pause deployment deployment-app-new
     deployment.apps/deployment-app-new paused
     [root@k8snode1 ~]#  curl -I 10.111.12.80
     HTTP/1.1 200 OK
     Server: nginx/1.8.1
     Date: Thu, 22 Apr 2021 08:22:46 GMT
     Content-Type: text/html
     Content-Length: 612
     Last-Modified: Tue, 26 Jan 2016 15:24:47 GMT
     Connection: keep-alive
     ETag: "56a78fbf-264"
     Accept-Ranges: bytes
     
     [root@k8snode1 ~]#  curl -I 10.111.12.80
     HTTP/1.1 200 OK
     Server: nginx/1.8.1
     Date: Thu, 22 Apr 2021 08:22:47 GMT
     Content-Type: text/html
     Content-Length: 612
     Last-Modified: Tue, 26 Jan 2016 15:24:47 GMT
     Connection: keep-alive
     ETag: "56a78fbf-264"
     Accept-Ranges: bytes
     
     [root@k8snode1 ~]#  curl -I 10.111.12.80
     HTTP/1.1 200 OK
     Server: nginx/1.7.9
     Date: Thu, 22 Apr 2021 08:22:48 GMT
     Content-Type: text/html
     Content-Length: 612
     Last-Modified: Tue, 23 Dec 2014 16:25:09 GMT
     Connection: keep-alive
     ETag: "54999765-264"
     Accept-Ranges: bytes
     
     #接着 在恢复 在停止，发现多了一个
     [root@k8snode1 ~]#  kubectl rollout resume deployment deployment-app-new
     deployment.apps/deployment-app-new resumed
     [root@k8snode1 ~]#  kubectl rollout pause deployment deployment-app-new
     deployment.apps/deployment-app-new paused
     
     ```

# DeamonSet控制器

参考：https://blog.51cto.com/happylab/2504721，https://kubernetes.io/zh/docs/concepts/workloads/controllers/daemonset/

介绍DaemonSet时我们先来思考一个问题：相信大家都接触过监控系统比如zabbix，监控系统需要在被监控机安装一个agent，安装agent通常会涉及到以下几个场景：

- 所有节点都必须安装agent以便采集监控数据
- 新加入的节点需要配置agent，手动或者运行脚本
- 节点下线后需要手动在监控系统中删除

DaemonSet可以保证在所有节点上运行pod 或者部分节点运行pod，一旦有新的node节点加入进来，就会直接部署一个pod，当节点要移除的时候 就通过垃圾回收移除对应的节点

kubernetes中经常涉及到在node上安装部署应用，它是如何解决上述的问题的呢？答案是DaemonSet。DaemonSet守护进程简称DS，适用于在所有节点或部分节点运行一个daemon守护进程，如监控我们安装部署时网络插件kube-flannel和kube-proxy，DaemonSet具有如下特点：

- DaemonSet确保所有节点运行一个Pod副本
- 指定节点运行一个Pod副本，通过标签选择器或者节点亲和性
- 新增节点会自动在节点增加一个Pod
- 移除节点时垃圾回收机制会自动清理Pod

DaemonSet适用于每个node节点均需要部署一个守护进程的场景，常见的场景例如：

- 日志采集agent，如fluentd或logstash
- 监控采集agent，如Prometheus Node Exporter,Sysdig Agent,Ganglia gmond
- 分布式集群组件，如Ceph MON，Ceph OSD，glusterd，Hadoop Yarn NodeManager等
- k8s必要运行组件，如网络flannel，weave，calico，kube-proxy等

1. 定义一个daemonsets 

```
#查看daemonsets 当前三个节点，就会有3个
[root@k8snode1 ~]# kubectl get daemonsets.apps -n kube-system 
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
calico-node   3         3         3       3            3           kubernetes.io/os=linux   4d16h
kube-proxy    3         3         3       3            3           kubernetes.io/os=linux   4d17h

#定义一个daemonsets
[root@k8snode1 ~]# vim daemonset-demo.yaml 

apiVersion: apps/v1              #api版本信息
kind: DaemonSet                  #类型为DaemonSet
metadata:                        #元数据信息
  name: fluentd-elasticsearch
  namespace: kube-system        #运行的命名空间
  labels:
    k8s-app: fluentd-logging
spec:                          #DS模版
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch  #这个labels必须和选择器的一样
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:            #容器信息
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        imagePullPolicy: IfNotPresent
        resources:          #resource资源
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:      #挂载存储，agent需要到这些目录采集日志
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:            #将主机的目录以hostPath的形式挂载到容器Pod中。
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers

#可以看到 pod正在部署并且正在创建
[root@k8snode1 ~]# kubectl get ds -n kube-system 
NAME                    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
calico-node             3         3         3       3            3           kubernetes.io/os=linux   4d17h
fluentd-elasticsearch   3         3         1       3            1           <none>                   3m41s
kube-proxy              3         3         3       3            3           kubernetes.io/os=linux   4d18h
[root@k8snode1 ~]# kubectl get pods -n kube-system -o wide
NAME                                       READY   STATUS              RESTARTS   AGE     IP               NODE       NOMINATED NODE   READINESS GATES
calico-kube-controllers-6d8ccdbf46-62qmm   1/1     Running             1          2d18h   172.16.249.8     k8snode1   <none>           <none>
calico-node-249tw                          1/1     Running             1          4d17h   192.168.10.187   k8snode3   <none>           <none>
calico-node-8rxs5                          1/1     Running             2          4d17h   192.168.10.186   k8snode2   <none>           <none>
calico-node-zs4hx                          1/1     Running             2          4d17h   192.168.10.185   k8snode1   <none>           <none>
coredns-558bd4d5db-j4nb4                   1/1     Running             1          2d18h   172.16.249.9     k8snode1   <none>           <none>
coredns-558bd4d5db-q2cbn                   1/1     Running             1          2d18h   172.16.249.10    k8snode1   <none>           <none>
etcd-k8snode1                              1/1     Running             5          4d18h   192.168.10.185   k8snode1   <none>           <none>
fluentd-elasticsearch-gz8hl                1/1     Running             0          4m      172.16.249.12    k8snode1   <none>           <none>
fluentd-elasticsearch-sdljl                0/1     ContainerCreating   0          4m      <none>           k8snode2   <none>           <none>
fluentd-elasticsearch-zj8tf                1/1     Running             0          4m      172.16.98.198    k8snode3   <none>           <none>
kube-apiserver-k8snode1                    1/1     Running             6          4d18h   192.168.10.185   k8snode1   <none>           <none>
kube-controller-manager-k8snode1           1/1     Running             5          3d22h   192.168.10.185   k8snode1   <none>           <none>
kube-proxy-9tvkf                           1/1     Running             1          4d17h   192.168.10.187   k8snode3   <none>           <none>
kube-proxy-c4m82                           1/1     Running             2          4d18h   192.168.10.185   k8snode1   <none>           <none>
kube-proxy-r4xpx                           1/1     Running             2          4d17h   192.168.10.186   k8snode2   <none>           <none>
kube-scheduler-k8snode1                    1/1     Running             6          3d22h   192.168.10.185   k8snode1   <none>           <none>


[root@k8snode1 ~]# kubectl describe pods fluentd-elasticsearch-sdljl
Error from server (NotFound): pods "fluentd-elasticsearch-sdljl" not found
[root@k8snode1 ~]# kubectl describe pods fluentd-elasticsearch-sdljl -n kube-system 
Name:           fluentd-elasticsearch-sdljl
Namespace:      kube-system
Priority:       0
Node:           k8snode2/192.168.10.186
Start Time:     Thu, 22 Apr 2021 16:53:08 +0800
Labels:         controller-revision-hash=84d7969c85
                name=fluentd-elasticsearch
                pod-template-generation=1
Annotations:    cni.projectcalico.org/podIP: 172.16.185.243/32
                cni.projectcalico.org/podIPs: 172.16.185.243/32
Status:         Pending
IP:             
IPs:            <none>
Controlled By:  DaemonSet/fluentd-elasticsearch
Containers:
  fluentd-elasticsearch:
    Container ID:   
    Image:          quay.io/fluentd_elasticsearch/fluentd:v2.5.2
    Image ID:       
    Port:           <none>
    Host Port:      <none>
    State:          Waiting
      Reason:       ContainerCreating
    Ready:          False
    Restart Count:  0
    Limits:
      memory:  200Mi
    Requests:
      cpu:        100m
      memory:     200Mi
    Environment:  <none>
    Mounts:
      /var/lib/docker/containers from varlibdockercontainers (ro)
      /var/log from varlog (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-dx2jw (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             False 
  ContainersReady   False 
  PodScheduled      True 
Volumes:
  varlog:
    Type:          HostPath (bare host directory volume)
    Path:          /var/log
    HostPathType:  
  varlibdockercontainers:
    Type:          HostPath (bare host directory volume)
    Path:          /var/lib/docker/containers
    HostPathType:  
  kube-api-access-dx2jw:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Burstable
Node-Selectors:              <none>
Tolerations:                 node-role.kubernetes.io/master:NoSchedule
                             node.kubernetes.io/disk-pressure:NoSchedule op=Exists
                             node.kubernetes.io/memory-pressure:NoSchedule op=Exists
                             node.kubernetes.io/not-ready:NoExecute op=Exists
                             node.kubernetes.io/pid-pressure:NoSchedule op=Exists
                             node.kubernetes.io/unreachable:NoExecute op=Exists
                             node.kubernetes.io/unschedulable:NoSchedule op=Exists
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  4m56s  default-scheduler  Successfully assigned kube-system/fluentd-elasticsearch-sdljl to k8snode2
 #可以看到正在拉取
 Normal  Pulling    4m56s  kubelet            Pulling image "quay.io/fluentd_elasticsearch/fluentd:v2.5.2"

```

2. daemonsets 调度

   ```
   #查看详情
   [root@k8snode1 ~]# kubectl get ds -n kube-system fluentd-elasticsearch -o yaml
   apiVersion: apps/v1
   kind: DaemonSet
   metadata:
     annotations:
       deprecated.daemonset.template.generation: "1"
       kubectl.kubernetes.io/last-applied-configuration: |
         {"apiVersion":"apps/v1","kind":"DaemonSet","metadata":{"annotations":{},"labels":{"k8s-app":"fluentd-logging"},"name":"fluentd-elasticsearch","namespace":"kube-system"},"spec":{"selector":{"matchLabels":{"name":"fluentd-elasticsearch"}},"template":{"metadata":{"labels":{"name":"fluentd-elasticsearch"}},"spec":{"containers":[{"image":"quay.io/fluentd_elasticsearch/fluentd:v2.5.2","imagePullPolicy":"IfNotPresent","name":"fluentd-elasticsearch","resources":{"limits":{"memory":"200Mi"},"requests":{"cpu":"100m","memory":"200Mi"}},"volumeMounts":[{"mountPath":"/var/log","name":"varlog"},{"mountPath":"/var/lib/docker/containers","name":"varlibdockercontainers","readOnly":true}]}],"terminationGracePeriodSeconds":30,"tolerations":[{"effect":"NoSchedule","key":"node-role.kubernetes.io/master"}],"volumes":[{"hostPath":{"path":"/var/log"},"name":"varlog"},{"hostPath":{"path":"/var/lib/docker/containers"},"name":"varlibdockercontainers"}]}}}}
     creationTimestamp: "2021-04-22T08:53:09Z"
     generation: 1
     labels:
       k8s-app: fluentd-logging
     name: fluentd-elasticsearch
     namespace: kube-system
     resourceVersion: "851094"
     uid: 5f78ab0f-9db0-492c-acc1-6f60735dcd37
   spec:
     revisionHistoryLimit: 10
     selector:
       matchLabels:
         name: fluentd-elasticsearch
     template:
       metadata:
         creationTimestamp: null
         labels:
           name: fluentd-elasticsearch
       spec:
         containers:
         - image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
           imagePullPolicy: IfNotPresent
           name: fluentd-elasticsearch
           resources:
             limits:
               memory: 200Mi
             requests:
               cpu: 100m
               memory: 200Mi
           terminationMessagePath: /dev/termination-log
           terminationMessagePolicy: File
           volumeMounts:
           - mountPath: /var/log
             name: varlog
           - mountPath: /var/lib/docker/containers
             name: varlibdockercontainers
             readOnly: true
         dnsPolicy: ClusterFirst
         restartPolicy: Always
         schedulerName: default-scheduler
         securityContext: {}
         terminationGracePeriodSeconds: 30
         tolerations:
         - effect: NoSchedule
           key: node-role.kubernetes.io/master
         volumes:
         - hostPath:
             path: /var/log
             type: ""
           name: varlog
         - hostPath:
             path: /var/lib/docker/containers
             type: ""
           name: varlibdockercontainers
     updateStrategy:  #更新策略
       rollingUpdate:  #对应的是滚动更新
         maxSurge: 0   
         maxUnavailable: 1 #最大不可用的pod有几个
       type: RollingUpdate
   status:
     currentNumberScheduled: 3
     desiredNumberScheduled: 3
     numberAvailable: 2
     numberMisscheduled: 0
     numberReady: 2
     numberUnavailable: 1
     observedGeneration: 1
     updatedNumberScheduled: 3
   
   
   #滚动更新与回滚
   #更新镜像至最新版本
   [root@node-1 ~]# kubectl set image daemonsets fluentd-elasticsearch fluentd-elasticsearch=quay.io/fluentd_elasticsearch/fluentd:latest -n kube-system
   daemonset.extensions/fluentd-elasticsearch image updated
   
   #查看滚动更新状态
   [root@node-1 ~]# kubectl rollout status daemonset -n kube-system fluentd-elasticsearch 
   Waiting for daemon set "fluentd-elasticsearch" rollout to finish: 1 out of 3 new pods have been updated...
   Waiting for daemon set "fluentd-elasticsearch" rollout to finish: 1 out of 3 new pods have been updated...
   Waiting for daemon set "fluentd-elasticsearch" rollout to finish: 1 out of 3 new pods have been updated...
   Waiting for daemon set "fluentd-elasticsearch" rollout to finish: 2 out of 3 new pods have been updated...
   Waiting for daemon set "fluentd-elasticsearch" rollout to finish: 2 out of 3 new pods have been updated...
   Waiting for daemon set "fluentd-elasticsearch" rollout to finish: 2 out of 3 new pods have been updated...
   Waiting for daemon set "fluentd-elasticsearch" rollout to finish: 2 of 3 updated pods are available...
   daemon set "fluentd-elasticsearch" successfully rolled out
   
   
   #查看DaemonSet滚动更新版本，REVSION 1为初始的版本
   [root@node-1 ~]# kubectl rollout history daemonset -n kube-system fluentd-elasticsearch 
   daemonset.extensions/fluentd-elasticsearch 
   REVISION  CHANGE-CAUSE
   1         <none>
   2         <none>
   
   #更新回退，如果配置没有符合到预期可以回滚到原始的版本
   [root@node-1 ~]# kubectl rollout undo daemonset -n kube-system fluentd-elasticsearch --to-revision=1
   daemonset.extensions/fluentd-elasticsearch rolled back
   ```

3. 删除DaemonSet

   ```
   [root@k8snode1 ~]# kubectl delete daemonsets -n kube-system fluentd-elasticsearch
   [root@k8snode1 ~]# kubectl get ds -n kube-system 
   NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
   calico-node   3         3         3       3            3           kubernetes.io/os=linux   4d17h
   kube-proxy    3         3         3       3            3           kubernetes.io/os=linux   4d18h
   
   ```

4. DaemonSet调度策略

   DaemonSet通过kubernetes默认的调度器scheduler会在所有的node节点上运行一个Pod副本，可以通过如下三种方式将Pod运行在部分节点上：

   - 指定nodeName节点运行
   - 通过标签运行nodeSelector
   - 通过亲和力调度node Affinity和node Anti-affinity

   DaemonSet调度算法用于实现将Pod运行在特定的node节点上，如下以通过node affinity亲和力将Pod调度到部分的节点上node-2上为例。

   ```
   #先给node2和node3加一个标签
   [root@k8snode1 k8s]# kubectl label nodes k8snode2 app=fluentd
   node/k8snode2 labeled
   [root@k8snode1 k8s]# kubectl label nodes k8snode3 app=fluentd
   node/k8snode3 labeled
   
   [root@k8snode1 k8s]#  kubectl get nodes --show-labels 
   NAME       STATUS   ROLES                  AGE     VERSION   LABELS
   k8snode1   Ready    control-plane,master   4d18h   v1.21.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8snode1,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node-role.kubernetes.io/master=,node.kubernetes.io/exclude-from-external-load-balancers=
   k8snode2   Ready    <none>                 4d18h   v1.21.0   app=fluentd,beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8snode2,kubernetes.io/os=linux
   k8snode3   Ready    <none>                 4d18h   v1.21.0   app=fluentd,beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8snode3,kubernetes.io/os=linux
   
   
   [root@k8snode1 k8s]# vim daemonset-demo.yaml 
   
   apiVersion: apps/v1              #api版本信息
   kind: DaemonSet                  #类型为DaemonSet
   metadata:                        #元数据信息
     name: fluentd-elasticsearch
     namespace: kube-system        #运行的命名空间
     labels:
       k8s-app: fluentd-logging
   spec:                          #DS模版
     selector:
       matchLabels:
         name: fluentd-elasticsearch
     template:
       metadata:
         labels:
           name: fluentd-elasticsearch  #这个labels必须和选择器的一样
       spec:
         tolerations:
         - key: node-role.kubernetes.io/master
           effect: NoSchedule
         containers:            #容器信息
         - name: fluentd-elasticsearch
           image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
           imagePullPolicy: IfNotPresent
           resources:          #resource资源
             limits:
               memory: 200Mi
             requests:
               cpu: 100m
               memory: 200Mi
           volumeMounts:      #挂载存储，agent需要到这些目录采集日志
           - name: varlog
             mountPath: /var/log
           - name: varlibdockercontainers
             mountPath: /var/lib/docker/containers
             readOnly: true
         nodeSelector:  #node标签选择器
           app: fluentd  #添加node3的标签
         terminationGracePeriodSeconds: 30
         volumes:            #将主机的目录以hostPath的形式挂载到容器Pod中。
         - name: varlog
           hostPath:
             path: /var/log
         - name: varlibdockercontainers
           hostPath:
             path: /var/lib/docker/containers
   
   [root@k8snode1 k8s]# kubectl apply -f daemonset-demo.yaml 
   daemonset.apps/fluentd-elasticsearch created
   
   #可以看到 只有两个
   [root@k8snode1 k8s]# kubectl get ds -n kube-system 
   NAME                    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
   calico-node             3         3         3       3            3           kubernetes.io/os=linux   4d17h
   fluentd-elasticsearch   2         2         2       2            2           app=fluentd              54s
   kube-proxy              3         3         3       3            3           kubernetes.io/os=linux   4d18h
   
   ```

# Job控制器

参考：https://blog.51cto.com/happylab/2504772，https://kubernetes.io/zh/docs/concepts/workloads/controllers/job/

jobs是单次任务，只跑一次,如果失败了 还可以定制策略来重启或者继续执行

Windows下可以通过批处理脚本完成批处理任务，脚本运行完毕后任务即可终止，从而实现批处理任务运行工作，类似的任务如何在kubernetes中运行呢？答案是Jobs，Jobs是kubernetes中实现一次性计划任务的Pod控制器—JobController，通过控制Pod来执行任务，其特点为：

- 创建Pod运行特定任务，确保任务运行完成
- 任务运行期间节点异常时会自动重新创建Pod
- 支持并发创建Pod任务数和指定任务数

1. 创建一个job

   ```
   [root@k8snode1 k8s]# vim job-demo.yaml 
   
   apiVersion: batch/v1
   kind: Job
   metadata:
     name: jobs-demo
     labels:
       controller: jobs
   spec:
     parallelism: 1        #并发数，默认为1，即创建pod副本的数量i
     completions: 1      #任务运行多少次
     backoffLimit: 2  #失败后重试两次
     template:
       metadata:
         name: jobs-demo
         labels:
           controller: jobs
       spec:
         containers:
         - name: echo-time
           image: centos:latest
           imagePullPolicy: IfNotPresent
           command:
           - /bin/sh
           - -c
           - "for i in `seq 0 100`;do echo ${date} && sleep 1;done"
         restartPolicy: Never  #设置为Never，jobs任务运行完毕即可完成
   
   [root@k8snode1 k8s]# kubectl apply -f job-demo.yaml 
   job.batch/jobs-demo created
   
   #查看 一个任务已经运行完
   [root@k8snode1 k8s]# kubectl get jobs
   NAME        COMPLETIONS   DURATION   AGE
   jobs-demo   1/1           2m7s       5m27s
   
   [root@k8snode1 k8s]# kubectl get jobs
   NAME        COMPLETIONS   DURATION   AGE
   jobs-demo   1/1           2m7s       5m27s
   [root@k8snode1 k8s]# kubectl describe jobs
   Name:           jobs-demo
   Namespace:      default
   Selector:       controller-uid=113df463-6e11-490a-b073-4a40cbed532f
   Labels:         controller=jobs
   Annotations:    <none>
   Parallelism:    1
   Completions:    1
   Start Time:     Thu, 22 Apr 2021 17:31:12 +0800
   Completed At:   Thu, 22 Apr 2021 17:33:19 +0800
   Duration:       2m7s
   Pods Statuses:  0 Running / 1 Succeeded / 0 Failed
   Pod Template:
     Labels:  controller=jobs
              controller-uid=113df463-6e11-490a-b073-4a40cbed532f
              job-name=jobs-demo
     Containers:
      echo-time:
       Image:      centos:latest
       Port:       <none>
       Host Port:  <none>
       Command:
         /bin/sh
         -c
         for i in `seq 0 100`;do echo ${date} && sleep 1;done
       Environment:  <none>
       Mounts:       <none>
     Volumes:        <none>
   #发现运行成功并且完成
   Events:
     Type    Reason            Age    From            Message
     ----    ------            ----   ----            -------
     Normal  SuccessfulCreate  5m51s  job-controller  Created pod: jobs-demo-dskgk
     Normal  Completed         3m44s  job-controller  Job completed
   
   ```

2. job多任务 任务顺序执行多少次

   Jobs控制器提供了两个控制并发数的参数：completions和parallelism，completions表示需要运行任务数的总数，parallelism表示并发运行的个数，如设置为1则会依次运行任务，前面任务运行再运行后面的任务，如下以创建5个任务数为例演示Jobs控制器实现并发数的机制。

   ```
   [root@k8snode1 k8s]# kubectl delete jobs.batch jobs-demo
   job.batch "jobs-demo" deleted
   
   [root@k8snode1 k8s]# vim job-demo.yaml 
   
   apiVersion: batch/v1
   kind: Job
   metadata:
     name: jobs-demo1
     labels:
       controller: jobs
   spec:
     parallelism: 1        #并发数，默认为1，即创建pod副本的数量i
     completions: 5   #任务运行5次
     backoffLimit: 2  #失败后重试两次
     template:
       metadata:
         name: jobs-demo1
         labels:
           controller: jobs
       spec:
         containers:
         - name: echo-time
           image: centos:latest
           imagePullPolicy: IfNotPresent
           command:
           - /bin/sh
           - -c
           - "for i in `seq 0 100`;do echo ${date} && sleep 1;done"
         restartPolicy: Never  #设置为Never，jobs任务运行完毕即可完成
   
   #打开实时监控可以看到每次只跑一个任务，一个完成后在继续执行下一个任务
   ```

3. job多并发任务 

   ```
   [root@k8snode1 k8s]# vim job-demo1.yaml 
   
   apiVersion: batch/v1
   kind: Job
   metadata:
     name: jobs-demo
     labels:
       controller: jobs
   spec:
     parallelism: 4        #并发数，默认为1，即创建pod副本的数量i
     completions: 5
     backoffLimit: 2  #失败后重试两次
     template:
       metadata:
         name: jobs-demo
         labels:
           controller: jobs
       spec:
         containers:
         - name: echo-time
           image: centos:latest
           imagePullPolicy: IfNotPresent
           command:
           - /bin/sh
           - -c
           - "for i in `seq 0 100`;do echo ${date} && sleep 1;done"
         restartPolicy: Never  #设置为Never，jobs任务运行完毕即可完成
   
   [root@k8snode1 k8s]# kubectl apply -f job-demo1.yaml 
   job.batch/jobs-demo1 created
   
   #根据这个命令可以看出 同时创建了4个pod
   [root@k8snode1 ~]#  kubectl get pods -w
   
   [root@k8snode1 k8s]# kubectl get jobs
   NAME         COMPLETIONS   DURATION   AGE
   jobs-demo    3/5           5m48s      5m48s
   jobs-demo1   4/5           2m26s      2m26s
   
   ```

# CronJobs周期性运转

CronJobs用于实现类似Linux下的cronjob周期性计划任务，CronJobs控制器通过时间线创建Jobs任务，从而完成任务的执行处理，其具有如下特点：

- 实现周期性计划任务
- 实际还是调用Jobs控制器创建任务
- CronJobs任务名称小于52个字符
- 应用场景如：定期备份，周期性发送邮件

1. 定义一个CronJobs任务

   ```
   [root@k8snode1 k8s]# vim cronJobs-demo.yaml
   
   apiVersion: batch/v1beta1  #api 定值 每个kind不一样
   kind: CronJob
   metadata:
     name: cronjob-demo
     labels:
       jobgroup: cronjob-demo
   spec:
     schedule: "*/5 * * * *"       #调度任务周期
     jobTemplate:                    #创建Jobs任务模版
       spec:
         template:
           spec:
             containers:
             - name: cronjob-demo
               image: busybox:latest
               imagePullPolicy: IfNotPresent
               command:
               - /bin/sh
               - -c
               - "for i in `seq 0 100`;do echo ${i} && sleep 1;done"
             restartPolicy: Never   #必要值 必须设置Never 任务失败了就失败了 不重试了
              
   [root@k8snode1 k8s]# kubectl apply -f cronJobs-demo.yaml 
   Warning: batch/v1beta1 CronJob is deprecated in v1.21+, unavailable in v1.25+; use batch/v1 CronJob
   cronjob.batch/cronjob-demo created
   
   #已经开始运行 
   [root@k8snode1 k8s]# kubectl get cronjobs.batch 
   NAME           SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
   cronjob-demo   */5 * * * *   False     1        24s             2m16s
   [root@k8snode1 k8s]# kubectl describe cronjobs.batch cronjob-demo 
   Name:                          cronjob-demo
   Namespace:                     default
   Labels:                        jobgroup=cronjob-demo
   Annotations:                   <none>
   Schedule:                      */5 * * * *
   Concurrency Policy:            Allow
   Suspend:                       False
   Successful Job History Limit:  3
   Failed Job History Limit:      1
   Starting Deadline Seconds:     <unset>
   Selector:                      <unset>
   Parallelism:                   <unset>
   Completions:                   <unset>
   Pod Template:
     Labels:  <none>
     Containers:
      cronjob-demo:
       Image:      busybox:latest
       Port:       <none>
       Host Port:  <none>
       Command:
         /bin/sh
         -c
         for i in `seq 0 100`;do echo ${i} && sleep 1;done
       Environment:     <none>
       Mounts:          <none>
     Volumes:           <none>
   Last Schedule Time:  Thu, 22 Apr 2021 18:00:00 +0800
   Active Jobs:         cronjob-demo-26984760
   Events:
     Type    Reason            Age   From                Message
     ----    ------            ----  ----                -------
     #创建了一个pod
     Normal  SuccessfulCreate  36s   cronjob-controller  Created job cronjob-demo-26984760
   
   ```

2. 同样支持并发和任务数

   ```
   [root@k8snode1 k8s]# vim cronJobs-demo1.yaml 
   
   apiVersion: batch/v1  #api 定值 每个kind不一样
   kind: CronJob
   metadata:
     name: cronjob-demo1
     labels:
       jobgroup: cronjob-demo1
   spec:
     schedule: "*/5 * * * *"       #调度任务周期
     jobTemplate:                    #创建Jobs任务模版
       spec:
         parallelism: 3        #并发数，默认为1，即创建pod副本的数量i
         completions: 5   #任务运行5次
         template:
           spec:
             containers:
             - name: cronjob-demo1
               image: busybox:latest
               imagePullPolicy: IfNotPresent
               command:
               - /bin/sh
               - -c
               - "for i in `seq 0 100`;do echo ${i} && sleep 1;done"
             restartPolicy: Never
   
   [root@k8snode1 k8s]# kubectl apply -f cronJobs-demo1.yaml 
   cronjob.batch/cronjob-demo1 created
   
   #可以看到 两个已经创建 还没开始跑
   [root@k8snode1 k8s]# kubectl get pods 
   NAME                                    READY   STATUS      RESTARTS   AGE
   cronjob-demo-26984760-rxg4t             0/1     Completed   0          8m57s
   cronjob-demo-26984765-66c8d             0/1     Completed   0          3m57s
   
   [root@k8snode1 ~]#  kubectl get jobs.batch -w
   ```

# service

参考：https://kubernetes.io/zh/docs/concepts/services-networking/service/

service是通过lablels来管理多个pod容器的

service是通过lable来关联pod的其实和deployment无关的，两个deployment的标签一样 也是可以关联到service的

将运行在一组 [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 上的应用程序公开为网络服务的抽象方法。

使用 Kubernetes，你无需修改应用程序即可使用不熟悉的服务发现机制。 Kubernetes 为 Pods 提供自己的 IP 地址，并为一组 Pod 提供相同的 DNS 名， 并且可以在它们之间进行负载均衡

创建和销毁 Kubernetes [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 以匹配集群状态。 Pod 是非永久性资源。 如果你使用 [Deployment](https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/) 来运行你的应用程序，则它可以动态创建和销毁 Pod。

每个 Pod 都有自己的 IP 地址，但是在 Deployment 中，在同一时刻运行的 Pod 集合可能与稍后运行该应用程序的 Pod 集合不同。

这导致了一个问题： 如果一组 Pod（称为“后端”）为集群内的其他 Pod（称为“前端”）提供功能， 那么前端如何找出并跟踪要连接的 IP 地址，以便前端可以使用提供工作负载的后端部分？

service就是对后端所有的pod进行负载均衡，当pod的ip发生变化的时候 services没有变化就行，pod发生ip变化后 只要将pod的ip注册到service上就可以

服务发现， 把每个service的名称和命名空间这些信息组成一个域名  把域名和注册到coredns中后面服务调用的时候调用服务名字就可以

* userspace 代理模式

  service是借助于leable标签来管理各个pod的，持续监控这些pod的标签，service是通过kube-proxy实现的，kube-proxy会一直监听apiserver只要发生了变化 就会根据iptable把新的转发规则构建好 kube-proxy在每个apiserver上都会 监听一个端口 通过这个端口接收信息把数据转发到后端

  用户流量是通过node的cluserip转发到kube-proxy(这一步是内核态的)，在通过kube-proxy转发到相应的node

*  iptables 代理模式 

  基于apiserver监听规则。kube-proxy直接构建cluserip和podip的规则，cluserip监听到消息后 直接就转发到了podip上，减少了内核态的转发 ，提高了效率，在1.2的版本后默认就是这个了 在规模比较大的情况下，就有了性能的瓶颈

*  IPVS 代理模式

  lvs的模式，规模比较大的情况下使用这种

  在 `ipvs` 模式下，kube-proxy 监视 Kubernetes 服务和端点，调用 `netlink` 接口相应地创建 IPVS 规则， 并定期将 IPVS 规则与 Kubernetes 服务和端点同步。 该控制循环可确保IPVS 状态与所需状态匹配。访问服务时，IPVS 将流量定向到后端Pod之一。

  IPVS代理模式基于类似于 iptables 模式的 netfilter 挂钩函数， 但是使用哈希表作为基础数据结构，并且在内核空间中工作。 这意味着，与 iptables 模式下的 kube-proxy 相比，IPVS 模式下的 kube-proxy 重定向通信的延迟要短，并且在同步代理规则时具有更好的性能。 与其他代理模式相比，IPVS 模式还支持更高的网络流量吞吐量。

  IPVS 提供了更多选项来平衡后端 Pod 的流量。 这些是：

  - `rr`：轮替（Round-Robin）
  - `lc`：最少链接（Least Connection），即打开链接数量最少者优先
  - `dh`：目标地址哈希（Destination Hashing）
  - `sh`：源地址哈希（Source Hashing）
  - `sed`：最短预期延迟（Shortest Expected Delay）
  - `nq`：从不排队（Never Queue）

  > **说明：**
  >
  > 要在 IPVS 模式下运行 kube-proxy，必须在启动 kube-proxy 之前使 IPVS 在节点上可用。
  >
  > 当 kube-proxy 以 IPVS 代理模式启动时，它将验证 IPVS 内核模块是否可用。 如果未检测到 IPVS 内核模块，则 kube-proxy 将退回到以 iptables 代理模式运行。
  >
  > 

1. 创建service

   ```
   # 创建一个deployment
   [root@k8snode1 ~]# kubectl create deployment service-demo --image=nginx:1.7.9
   deployment.apps/service-demo created
   [root@k8snode1 ~]# kubectl get deployments.apps 
   NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
   deployment-app         3/3     3            3           22h
   deployment-app-new     3/3     3            3           21h
   nginx-dashboard-demo   3/3     3            3           3d4h
   nginxdemo1             4/4     4            4           3d22h
   service-demo           1/1     1            1           26s
   [root@k8snode1 ~]# kubectl scale deployment service-demo --replicas=3
   deployment.apps/service-demo scaled
   [root@k8snode1 ~]# kubectl get deployments.apps 
   NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
   deployment-app         3/3     3            3           22h
   deployment-app-new     3/3     3            3           21h
   nginx-dashboard-demo   3/3     3            3           3d4h
   nginxdemo1             4/4     4            4           3d22h
   service-demo           3/3     3            3           42s
   
   [root@k8snode1 ~]# kubectl get pod --show-labels -l  app=service-demo -o wide
   NAME                            READY   STATUS    RESTARTS   AGE    IP               NODE       NOMINATED NODE   READINESS GATES   LABELS
   service-demo-6c5dcb7b86-56wt5   1/1     Running   0          118s   172.16.98.235    k8snode3   <none>           <none>            app=service-demo,pod-template-hash=6c5dcb7b86
   service-demo-6c5dcb7b86-8mszm   1/1     Running   0          79s    172.16.185.237   k8snode2   <none>           <none>            app=service-demo,pod-template-hash=6c5dcb7b86
   service-demo-6c5dcb7b86-dmx8c   1/1     Running   0          79s    172.16.185.234   k8snode2   <none>           <none>            app=service-demo,pod-template-hash=6c5dcb7b86
   
   #创建一个service service对后端是负载均衡的
   [root@k8snode1 k8s]# vim server-demo.yaml
   
   apiVersion: v1
   kind: Service
   metadata:
     name: nginx-server
   spec:
     selector:
       app: service-demo #需要构建pod的标签
     ports:  #对外暴露的端口
       - name: http-80
         protocol: TCP
         port: 80   #需要暴露的端口
         targetPort: 80  #目标pod的端口
   
   [root@k8snode1 k8s]# kubectl apply -f server-demo.yaml 
   service/nginx-server created
   
   [root@k8snode1 k8s]# kubectl get service nginx-server
   NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
   nginx-server   ClusterIP   10.97.122.227   <none>        80/TCP    2m3s
   
   [root@k8snode1 k8s]# kubectl get service nginx-server -o yaml
   apiVersion: v1
   kind: Service
   metadata:
     annotations:
       kubectl.kubernetes.io/last-applied-configuration: |
         {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"nginx-server","namespace":"default"},"spec":{"ports":[{"port":80,"protocol":"TCP","targetPort":9376}],"selector":{"app":"service-demo"}}}
     creationTimestamp: "2021-04-22T14:15:48Z"
     name: nginx-server
     namespace: default
     resourceVersion: "904121"
     uid: c7a7157e-9d7d-4504-b700-29a959961b20
   spec:
     clusterIP: 10.97.122.227
     clusterIPs:
     - 10.97.122.227
     ipFamilies:
     - IPv4
     ipFamilyPolicy: SingleStack
     ports:
     - port: 80
       protocol: TCP
       targetPort: 9376
     selector:
       app: service-demo
     sessionAffinity: None
     type: ClusterIP
   status:
     loadBalancer: {}
   
   [root@k8snode1 k8s]# kubectl describe service nginx-server
   Name:              nginx-server
   Namespace:         default
   Labels:            <none>
   Annotations:       <none>
   Selector:          app=service-demo
   Type:              ClusterIP
   IP Family Policy:  SingleStack
   IP Families:       IPv4
   IP:                10.97.122.227
   IPs:               10.97.122.227
   Port:              <unset>  80/TCP
   TargetPort:        9376/TCP
   Endpoints:         172.16.185.234:9376,172.16.185.237:9376,172.16.98.235:9376  #可以看到 根据标签已经吧对应的pod选择出来了
   Session Affinity:  None
   Events:            <none>
   
   #也能看到这个资源
   [root@k8snode1 k8s]# kubectl get endpoints nginx-server 
   NAME           ENDPOINTS                                                    AGE
   nginx-server   172.16.185.234:9376,172.16.185.237:9376,172.16.98.235:9376   4m5s
   
   #访问成功
   [root@k8snode1 k8s]# curl http://10.97.122.227
   
   
   #容器中访问，--rm 退出后立马删除容器
   [root@k8snode1 k8s]# kubectl run test --rm -it --image=centos:latest -- /bin/bash
   #访问成功
   [root@test /]# curl http://10.97.122.227
   <!DOCTYPE html>
   <html>
   <head>
   <title>Welcome to nginx!</title>
   <style>
       body {
           width: 35em;
           margin: 0 auto;
           font-family: Tahoma, Verdana, Arial, sans-serif;
       }
   </style>
   </head>
   <body>
   <h1>Welcome to nginx!</h1>
   <p>If you see this page, the nginx web server is successfully installed and
   working. Further configuration is required.</p>
   
   <p>For online documentation and support please refer to
   <a href="http://nginx.org/">nginx.org</a>.<br/>
   Commercial support is available at
   <a href="http://nginx.com/">nginx.com</a>.</p>
   
   <p><em>Thank you for using nginx.</em></p>
   </body>
   </html>
   
   #也可以直接访问名称
   [root@test /]# curl http://nginx-server 
   <!DOCTYPE html>
   <html>
   <head>
   <title>Welcome to nginx!</title>
   <style>
       body {
           width: 35em;
           margin: 0 auto;
           font-family: Tahoma, Verdana, Arial, sans-serif;
       }
   </style>
   </head>
   <body>
   <h1>Welcome to nginx!</h1>
   <p>If you see this page, the nginx web server is successfully installed and
   working. Further configuration is required.</p>
   
   <p>For online documentation and support please refer to
   <a href="http://nginx.org/">nginx.org</a>.<br/>
   Commercial support is available at
   <a href="http://nginx.com/">nginx.com</a>.</p>
   
   <p><em>Thank you for using nginx.</em></p>
   </body>
   </html>
   
   ```

2. 会话保持sessionAffinity 默认是None

   ClientIP: 如果是同一个地址请求，会负载到后端的同一个pod

   ```
   [root@k8snode1 k8s]# vim server-demo.yaml 
   
   apiVersion: v1
   kind: Service
   metadata:
     name: nginx-server
   spec:
     selector:
       app: service-demo #所有pod的标签
     ports:
       - protocol: TCP
         port: 80
         targetPort: 80
     sessionAffinity: ClientIP  #会话保持类型如果是同一个地址请求，会负载到后端的同一个pod
     sessionAffinityConfig:
       clientIP:
         timeoutSeconds: 60 #会话保持超时时间
   
   [root@k8snode1 k8s]# kubectl apply -f server-demo.yaml 
   service/nginx-server configured
   [root@k8snode1 k8s]# kubectl get service nginx-server -o yaml
   apiVersion: v1
   kind: Service
   metadata:
     annotations:
       kubectl.kubernetes.io/last-applied-configuration: |
         {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"nginx-server","namespace":"default"},"spec":{"ports":[{"port":80,"protocol":"TCP","targetPort":80}],"selector":{"app":"service-demo"},"sessionAffinity":"ClientIP","sessionAffinityConfig":{"clientIP":{"timeoutSeconds":60}}}}
     creationTimestamp: "2021-04-22T14:15:48Z"
     name: nginx-server
     namespace: default
     resourceVersion: "908552"
     uid: c7a7157e-9d7d-4504-b700-29a959961b20
   spec:
     clusterIP: 10.97.122.227
     clusterIPs:
     - 10.97.122.227
     ipFamilies:
     - IPv4
     ipFamilyPolicy: SingleStack
     ports:
     - port: 80
       protocol: TCP
       targetPort: 80
     selector:
       app: service-demo
     sessionAffinity: ClientIP
     sessionAffinityConfig:
       clientIP:
         timeoutSeconds: 60  #查看设置成功
     type: ClusterIP
   status:
     loadBalancer: {}
   
   [root@k8snode1 k8s]# kubectl run test --rm -it --image=centos:latest -- /bin/bash
   If you don't see a command prompt, try pressing enter.
   [root@test /]# curl http://nginx-server #会一直打到一个pod上
   
   ```

3. server和pod之间的规则关联，是基于kube-proxy构建的iptable规则

   ```
   #可以看到 有两条规则，一个出口一个入口，入口收到消息后做的转发
   [root@k8snode1 k8s]# iptables -t nat -L -n | less
   KUBE-MARK-MASQ  tcp  -- !172.16.0.0/16        10.97.122.227        /* default/nginx-server cluster IP */ tcp dpt:80
   KUBE-SVC-N3JEWD2B456ZVERU  tcp  --  0.0.0.0/0            10.97.122.227        /* default/nginx-server cluster IP */ tcp dpt:80
   
   ```

4. 服务发现

   Kubernetes 支持两种基本的服务发现模式 —— 环境变量和 DNS

   环境变量：当 Pod 运行在 `Node` 上，kubelet 会为每个活跃的 Service 添加一组环境变量。 它同时支持 [Docker links兼容](https://docs.docker.com/userguide/dockerlinks/) 变量 （查看 [makeLinkVariables](https://releases.k8s.io/master/pkg/kubelet/envvars/envvars.go#L49)）、 简单的 `{SVCNAME}_SERVICE_HOST` 和 `{SVCNAME}_SERVICE_PORT` 变量。 这里 Service 的名称需大写，横线被转换成下划线。

   服务发现1：不管pod的扩容还是缩小，service都可以发现进行相应的关联

   环境变量：就是在每个运行的pod上添加一个service的环境变量 这个环境变量标明了当前service的所有配置

   ```
   [root@k8snode1 k8s]# kubectl run test --rm -it --image=centos:latest -- /bin/bash
   If you don't see a command prompt, try pressing enter.
   [root@test /]# set
   BASH=/bin/bash
   BASHOPTS=checkwinsize:cmdhist:complete_fullquote:expand_aliases:extquote:force_fignore:histappend:hostcomplete:interactive_comments:progcomp:promptvars:sourcepath
   BASHRCSOURCED=Y
   BASH_ALIASES=()
   BASH_ARGC=()
   BASH_ARGV=()
   BASH_CMDS=()
   BASH_LINENO=()
   BASH_SOURCE=()
   BASH_VERSINFO=([0]="4" [1]="4" [2]="19" [3]="1" [4]="release" [5]="x86_64-redhat-linux-gnu")
   BASH_VERSION='4.4.19(1)-release'
   COLUMNS=187
   DEPLOYMENT_APP_PORT=tcp://10.111.12.80:80
   DEPLOYMENT_APP_PORT_80_TCP=tcp://10.111.12.80:80
   DEPLOYMENT_APP_PORT_80_TCP_ADDR=10.111.12.80
   DEPLOYMENT_APP_PORT_80_TCP_PORT=80
   DEPLOYMENT_APP_PORT_80_TCP_PROTO=tcp
   DEPLOYMENT_APP_SERVICE_HOST=10.111.12.80
   DEPLOYMENT_APP_SERVICE_PORT=80
   DIRSTACK=()
   EUID=0
   GROUPS=()
   HISTFILE=/root/.bash_history
   HISTFILESIZE=500
   HISTSIZE=500
   HOME=/root
   HOSTNAME=test
   HOSTTYPE=x86_64
   IFS=$' \t\n'
   KUBERNETES_PORT=tcp://10.96.0.1:443
   KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
   KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
   KUBERNETES_PORT_443_TCP_PORT=443
   KUBERNETES_PORT_443_TCP_PROTO=tcp
   KUBERNETES_SERVICE_HOST=10.96.0.1
   KUBERNETES_SERVICE_PORT=443
   KUBERNETES_SERVICE_PORT_HTTPS=443
   LANG=en_US.UTF-8
   LESSOPEN='||/usr/bin/lesspipe.sh %s'
   LINES=47
   MACHTYPE=x86_64-redhat-linux-gnu
   MAILCHECK=60
   NGINXDEMO1_PORT=tcp://10.105.52.111:80
   NGINXDEMO1_PORT_80_TCP=tcp://10.105.52.111:80
   NGINXDEMO1_PORT_80_TCP_ADDR=10.105.52.111
   NGINXDEMO1_PORT_80_TCP_PORT=80
   NGINXDEMO1_PORT_80_TCP_PROTO=tcp
   NGINXDEMO1_SERVICE_HOST=10.105.52.111
   NGINXDEMO1_SERVICE_PORT=80
   NGINX_SERVER_PORT=tcp://10.97.122.227:80
   NGINX_SERVER_PORT_80_TCP=tcp://10.97.122.227:80
   NGINX_SERVER_PORT_80_TCP_ADDR=10.97.122.227
   NGINX_SERVER_PORT_80_TCP_PORT=80
   NGINX_SERVER_PORT_80_TCP_PROTO=tcp
   NGINX_SERVER_SERVICE_HOST=10.97.122.227
   NGINX_SERVER_SERVICE_PORT=80
   NGINX_TCP_READINESS_LIVENESS_PROBE_PORT=tcp://10.104.193.231:80
   NGINX_TCP_READINESS_LIVENESS_PROBE_PORT_80_TCP=tcp://10.104.193.231:80
   NGINX_TCP_READINESS_LIVENESS_PROBE_PORT_80_TCP_ADDR=10.104.193.231
   NGINX_TCP_READINESS_LIVENESS_PROBE_PORT_80_TCP_PORT=80
   NGINX_TCP_READINESS_LIVENESS_PROBE_PORT_80_TCP_PROTO=tcp
   NGINX_TCP_READINESS_LIVENESS_PROBE_SERVICE_HOST=10.104.193.231
   NGINX_TCP_READINESS_LIVENESS_PROBE_SERVICE_PORT=80
   OPTERR=1
   OPTIND=1
   OSTYPE=linux-gnu
   PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
   PIPESTATUS=([0]="0")
   PPID=0
   PROMPT_COMMAND='printf "\033]0;%s@%s:%s\007" "${USER}" "${HOSTNAME%%.*}" "${PWD/#$HOME/\~}"'
   PS1='[\u@\h \W]\$ '
   PS2='> '
   PS4='+ '
   PWD=/
   SHELL=/bin/bash
   SHELLOPTS=braceexpand:emacs:hashall:histexpand:history:interactive-comments:monitor
   SHLVL=1
   TERM=xterm
   UID=0
   _=/etc/bashrc
   gawklibpath_append () 
   { 
       [ -z "$AWKLIBPATH" ] && AWKLIBPATH=`gawk 'BEGIN {print ENVIRON["AWKLIBPATH"]}'`;
       export AWKLIBPATH="$AWKLIBPATH:$*"
   }
   gawklibpath_default () 
   { 
       unset AWKLIBPATH;
       export AWKLIBPATH=`gawk 'BEGIN {print ENVIRON["AWKLIBPATH"]}'`
   }
   gawklibpath_prepend () 
   { 
       [ -z "$AWKLIBPATH" ] && AWKLIBPATH=`gawk 'BEGIN {print ENVIRON["AWKLIBPATH"]}'`;
       export AWKLIBPATH="$*:$AWKLIBPATH"
   }
   gawkpath_append () 
   { 
       [ -z "$AWKPATH" ] && AWKPATH=`gawk 'BEGIN {print ENVIRON["AWKPATH"]}'`;
       export AWKPATH="$AWKPATH:$*"
   }
   gawkpath_default () 
   { 
       unset AWKPATH;
       export AWKPATH=`gawk 'BEGIN {print ENVIRON["AWKPATH"]}'`
   }
   gawkpath_prepend () 
   { 
       [ -z "$AWKPATH" ] && AWKPATH=`gawk 'BEGIN {print ENVIRON["AWKPATH"]}'`;
       export AWKPATH="$*:$AWKPATH"
   }
   
   ```

   DNS服务发现，只要注册一个服务，就会自动在dns里创建一个规则，会把service名称解析为service的ip地址

   ```
   [root@k8snode1 ~]# yum -y install bind-util
   [root@test /]# cat /etc/resolv.conf  #查看容器内的dns
   nameserver 10.96.0.10
   search default.svc.cluster.local svc.cluster.local cluster.local  #查询dns的后缀
   options ndots:5
   
   #容器内直接ping发现也可以解析成功 并且自动加了default.svc.cluster.local后缀
   [root@test /]# ping nginx-server
   PING nginx-server.default.svc.cluster.local (10.97.122.227) 56(84) bytes of data.
   64 bytes from nginx-server.default.svc.cluster.local (10.97.122.227): icmp_seq=1 ttl=59 time=3.48 ms
   64 bytes from nginx-server.default.svc.cluster.local (10.97.122.227): icmp_seq=2 ttl=59 time=4.44 ms
   ^C
   --- nginx-server.default.svc.cluster.local ping statistics ---
   2 packets transmitted, 2 received, 0% packet loss, time 2ms
   rtt min/avg/max/mdev = 3.476/3.956/4.437/0.484 ms
   
   
   #查看当前serverip 和上面解析的一样
   [root@k8snode1 ~]# kubectl get services nginx-server
   NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
   nginx-server   ClusterIP   10.97.122.227   <none>        80/TCP    46m
   
   #查看这个域名下 会有这个ip
   [root@k8snode1 ~]# dig nginx-service.default.svc.cluster.local @10.96.0.10
   
   ; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.4 <<>> nginx-service.default.svc.cluster.local @10.96.0.10
   ;; global options: +cmd
   ;; Got answer:
   ;; WARNING: .local is reserved for Multicast DNS
   ;; You are currently testing what happens when an mDNS query is leaked to DNS
   ;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 12008
   ;; flags: qr aa rd; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1
   ;; WARNING: recursion requested but not available
   
   ;; OPT PSEUDOSECTION:
   ; EDNS: version: 0, flags:; udp: 4096
   ;; QUESTION SECTION:
   ;nginx-service.default.svc.cluster.local. IN A
   
   ;; AUTHORITY SECTION:
   cluster.local.		30	IN	SOA	ns.dns.cluster.local. hostmaster.cluster.local. 1619102473 7200 1800 86400 30
   
   ;; Query time: 0 msec
   ;; SERVER: 10.96.0.10#53(10.96.0.10)
   ;; WHEN: Thu Apr 22 22:58:56 CST 2021
   ;; MSG SIZE  rcvd: 161
   
   [root@k8snode1 ~]# dig kube-dns.default.svc.cluster.local @10.96.0.10
   
   ; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.4 <<>> kube-dns.default.svc.cluster.local @10.96.0.10
   ;; global options: +cmd
   ;; Got answer:
   ;; WARNING: .local is reserved for Multicast DNS
   ;; You are currently testing what happens when an mDNS query is leaked to DNS
   ;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 7450
   ;; flags: qr aa rd; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1
   ;; WARNING: recursion requested but not available
   
   ;; OPT PSEUDOSECTION:
   ; EDNS: version: 0, flags:; udp: 4096
   ;; QUESTION SECTION:
   ;kube-dns.default.svc.cluster.local. IN	A
   
   ;; AUTHORITY SECTION:
   cluster.local.		30	IN	SOA	ns.dns.cluster.local. hostmaster.cluster.local. 1619102473 7200 1800 86400 30
   
   ;; Query time: 0 msec
   ;; SERVER: 10.96.0.10#53(10.96.0.10)
   ;; WHEN: Thu Apr 22 22:59:25 CST 2021
   ;; MSG SIZE  rcvd: 156
   
   ```

5. 服务发布，serviceTypes，将服务暴露到外部

   - `ClusterIP`：通过集群的内部 IP 暴露服务，选择该值时服务只能够在集群内部访问。 这也是默认的 `ServiceType`。
   - [`NodePort`](https://kubernetes.io/zh/docs/concepts/services-networking/service/#nodeport)：通过每个节点上的 IP 和静态端口（`NodePort`）暴露服务。 `NodePort` 服务会路由到自动创建的 `ClusterIP` 服务。 通过请求 `<节点 IP>:<节点端口>`，你可以从集群的外部访问一个 `NodePort` 服务。
   - [`LoadBalancer`](https://kubernetes.io/zh/docs/concepts/services-networking/service/#loadbalancer)：使用云提供商的负载均衡器向外部暴露服务。 外部负载均衡器可以将流量路由到自动创建的 `NodePort` 服务和 `ClusterIP` 服务上。
   - [`ExternalName`](https://kubernetes.io/zh/docs/concepts/services-networking/service/#externalname)：通过返回 `CNAME` 和对应值，可以将服务映射到 `externalName` 字段的内容（例如，`foo.bar.example.com`）。 无需创建任何类型代理。

6. NodePort 在宿主机随机(30000-32000)或者指定一个端口对外暴露

   ```
   [root@k8snode1 k8s]# vim server-demo.yaml 
   
   apiVersion: v1
   kind: Service
   metadata:
     name: nginx-server
   spec:
     type: NodePort
     selector:
       app: service-demo #所有pod的标签
     ports:
       - protocol: TCP
         port: 80
         targetPort: 80
         nodePort: 32000
     sessionAffinity: ClientIP  #会话保持类型如果是同一个地址请求，会负载到后端的同一个pod
     sessionAffinityConfig:
       clientIP:
         timeoutSeconds: 60 #会话保持超时时间
   
   [root@k8snode1 k8s]# kubectl apply -f server-demo.yaml 
   service/nginx-server configured
   
   #就可以通过nodeip进行访问
   [root@k8snode1 k8s]# curl http://k8snode1:32000
   <!DOCTYPE html>
   <html>
   <head>
   <title>Welcome to nginx!</title>
   <style>
       body {
           width: 35em;
           margin: 0 auto;
           font-family: Tahoma, Verdana, Arial, sans-serif;
       }
   </style>
   </head>
   <body>
   <h1>Welcome to nginx!</h1>
   <p>If you see this page, the nginx web server is successfully installed and
   working. Further configuration is required.</p>
   
   <p>For online documentation and support please refer to
   <a href="http://nginx.org/">nginx.org</a>.<br/>
   Commercial support is available at
   <a href="http://nginx.com/">nginx.com</a>.</p>
   
   <p><em>Thank you for using nginx.</em></p>
   </body>
   </html>
   
   #nodePort就是基于每个node都监听一个端口 通过iptable将请求转发到ClusterIP然后再转发到podip
   ```

7. LoadBalancer 与k8s结合 将 将请求转发到nodePort 在转发到对应的请求，自己实现了一个服务发现，监听nodePort 的状态

   详情：https://kubernetes.io/zh/docs/concepts/services-networking/service/

8. ExternalName：集群内部pod访问外部服务，就是将外部服务通过cname映射到集群里

   ```
   [root@k8snode1 k8s]# vim externalName.yaml
   
   apiVersion: v1
   kind: Service
   metadata:
     name: external-demo
   spec:
     type: ExternalName
     externalName: www.baidu.com  #这个域名必须宿主机可以解析
     ports:
     - name: baidu-80
       port: 80
       protocol: TCP
       targetPort: 80 #目标端口
       
   [root@k8snode1 k8s]# kubectl apply -f externalName.yaml 
   service/external-demo created
   
   #将external-demo解析为www.baidu.com
   [root@k8snode1 k8s]# kubectl get service external-demo
   NAME            TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)   AGE
   external-demo   ExternalName   <none>       www.baidu.com   80/TCP    35s
   
   #在容器中解析这个域名
   [root@test /]# ping external-demo
   PING www.a.shifen.com (14.215.177.38) 56(84) bytes of data.
   64 bytes from 14.215.177.38 (14.215.177.38): icmp_seq=1 ttl=54 time=33.7 ms
   64 bytes from 14.215.177.38 (14.215.177.38): icmp_seq=2 ttl=54 time=33.9 ms
   64 bytes from 14.215.177.38 (14.215.177.38): icmp_seq=3 ttl=54 time=33.9 ms
   64 bytes from 14.215.177.38 (14.215.177.38): icmp_seq=4 ttl=54 time=33.9 ms
   64 bytes from 14.215.177.38 (14.215.177.38): icmp_seq=5 ttl=54 time=34.1 ms
   64 bytes from 14.215.177.38 (14.215.177.38): icmp_seq=6 ttl=54 time=33.10 ms
   ^C
   --- www.a.shifen.com ping statistics ---
   6 packets transmitted, 6 received, 0% packet loss, time 10ms
   rtt min/avg/max/mdev = 33.735/33.916/34.079/0.237 ms
   
   ```

9. Headless类型，某些场景不需要service负载均衡，直接在service不设置clusterIP就是Headless service，对于Headless 是不会在kube-server上去构建这个规则的，访问服务就是基于dns来访问的，dns将服务直接解析为podsip

   ```
   [root@k8snode1 k8s]# vim headless-demo.yaml 
   
   apiVersion: v1
   kind: Service
   metadata:
     name: nginx-server
   spec:
     clusterIP: None  #设置clusterIP: None 就是headless
     selector:
       app: service-demo #所有pod的标签
     ports:
       - protocol: TCP
         port: 80
         targetPort: 80
     sessionAffinity: ClientIP  #会话保持类型如果是同一个地址请求，会负载到后端的同一个pod
     sessionAffinityConfig:
       clientIP:
         timeoutSeconds: 60 #会话保持超时时间
   
   
   [root@k8snode1 k8s]# kubectl apply -f headless-demo.yaml 
   service/nginx-server created
   
   #没有生成ClusterIPip
   [root@k8snode1 k8s]# kubectl get services nginx-server --show-labels 
   NAME           TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE   LABELS
   nginx-server   ClusterIP   None         <none>        80/TCP    36s   <none>
   
   #在容器里运行，可以看到将服务名直接解析为了pod的地址
   [root@test /]# ping nginx-server
   PING nginx-server.default.svc.cluster.local (172.16.185.234) 56(84) bytes of data.
   64 bytes from 172-16-185-234.nginx-server.default.svc.cluster.local (172.16.185.234): icmp_seq=1 ttl=62 time=0.467 ms
   64 bytes from 172-16-185-234.nginx-server.default.svc.cluster.local (172.16.185.234): icmp_seq=2 ttl=62 time=0.776 ms
   ^C
   --- nginx-server.default.svc.cluster.local ping statistics ---
   2 packets transmitted, 2 received, 0% packet loss, time 3ms
   rtt min/avg/max/mdev = 0.467/0.621/0.776/0.156 ms
   [root@test /]# ping nginx-server
   PING nginx-server.default.svc.cluster.local (172.16.98.235) 56(84) bytes of data.
   64 bytes from 172-16-98-235.nginx-server.default.svc.cluster.local (172.16.98.235): icmp_seq=1 ttl=63 time=0.146 ms
   ^C
   --- nginx-server.default.svc.cluster.local ping statistics ---
   1 packets transmitted, 1 received, 0% packet loss, time 0ms
   rtt min/avg/max/mdev = 0.146/0.146/0.146/0.000 ms
   [root@test /]# ping nginx-server
   PING nginx-server.default.svc.cluster.local (172.16.185.237) 56(84) bytes of data.
   64 bytes from 172-16-185-237.nginx-server.default.svc.cluster.local (172.16.185.237): icmp_seq=1 ttl=62 time=0.718 ms
   ^C
   --- nginx-server.default.svc.cluster.local ping statistics ---
   1 packets transmitted, 1 received, 0% packet loss, time 0ms
   rtt min/avg/max/mdev = 0.718/0.718/0.718/0.000 ms
   
   [root@k8snode1 k8s]# kubectl describe service nginx-server 
   Name:              nginx-server
   Namespace:         default
   Labels:            <none>
   Annotations:       <none>
   Selector:          app=service-demo
   Type:              ClusterIP
   IP Family Policy:  SingleStack
   IP Families:       IPv4
   IP:                None
   IPs:               None
   Port:              <unset>  80/TCP
   TargetPort:        80/TCP
   Endpoints:         172.16.185.234:80,172.16.185.237:80,172.16.98.235:80
   Session Affinity:  ClientIP
   Events:            <none>
   ```

10. 手动生成Endpoints

    ```
    [root@k8snode1 k8s]# vim server-demo.yaml 
    
    apiVersion: v1
    kind: Service
    metadata:
      name: nginx-server
    spec:
      clusterIP: None
      ports:
        - protocol: TCP
          port: 80
          targetPort: 80
      sessionAffinity: ClientIP  #会话保持类型如果是同一个地址请求，会负载到后端的同一个pod
      sessionAffinityConfig:
        clientIP:
          timeoutSeconds: 60 #会话保持超时时间
          
    [root@k8snode1 k8s]# kubectl apply -f server-demo.yaml 
    service/nginx-server created
    
    [root@k8snode1 k8s]# kubectl describe services nginx-server 
    Name:              nginx-server
    Namespace:         default
    Labels:            <none>
    Annotations:       <none>
    Selector:          <none>
    Type:              ClusterIP
    IP Family Policy:  RequireDualStack
    IP Families:       IPv4,IPv6
    IP:                None
    IPs:               None
    Port:              <unset>  80/TCP
    TargetPort:        80/TCP 
    Endpoints:         <none>   #Endpoints为空
    Session Affinity:  ClientIP
    Events:            <none>
    
    #抄一个实例
    [root@k8snode1 k8s]# kubectl get endpoints deployment-app -o yaml
    apiVersion: v1
    kind: Endpoints
    metadata:
      creationTimestamp: "2021-04-21T16:14:29Z"
      labels:
        app: deployment-app
      name: deployment-app
      namespace: default
      resourceVersion: "846057"
      uid: db448ecb-7543-4dd6-a29f-9119d8627d12
    subsets:
    - addresses:
      - ip: 172.16.98.196
        nodeName: k8snode3
        targetRef:
          kind: Pod
          name: deployment-app-new-7c68576649-m4ptj
          namespace: default
          resourceVersion: "846044"
          uid: b540389f-1528-47dc-b444-6dfadfc687e1
      - ip: 172.16.98.254
        nodeName: k8snode3
        targetRef:
          kind: Pod
          name: deployment-app-new-7c68576649-tmk69
          namespace: default
          resourceVersion: "846039"
          uid: 1e19e547-c839-42ce-acc2-2a280c4ffc73
      - ip: 172.16.98.255
        nodeName: k8snode3
        targetRef:
          kind: Pod
          name: deployment-app-new-7c68576649-sj9ls
          namespace: default
          resourceVersion: "846048"
          uid: 1d196c01-19bb-4e35-be2a-24f86c022c03
      ports:
      - port: 80
        protocol: TCP
    
    [root@k8snode1 k8s]# vim  endpoints-demo.yaml
    
    piVersion: v1
    kind: Endpoints
    metadata:
      name: nginx-server  # 必须和上面定义的services的name相同
    subsets:
    - addresses:
      - ip: 172.16.98.235   #pod的ip
    
    [root@k8snode1 k8s]# kubectl apply -f endpoints-demo.yaml 
    endpoints/nginx-server created
    
    [root@k8snode1 k8s]# kubectl describe services nginx-server 
    Name:              nginx-server
    Namespace:         default
    Labels:            <none>
    Annotations:       <none>
    Selector:          <none>
    Type:              ClusterIP
    IP Family Policy:  RequireDualStack
    IP Families:       IPv4,IPv6
    IP:                None
    IPs:               None
    Port:              <unset>  80/TCP
    TargetPort:        80/TCP
    Endpoints:         172.16.98.235  #查看 ip已经关联
    Session Affinity:  ClientIP
    Events:            <none>
    
    #再次在容器里ping 发现可以通并且已经解析
    [root@test /]# ping nginx-server
    PING nginx-server.default.svc.cluster.local (172.16.98.235) 56(84) bytes of data.
    64 bytes from 172-16-98-235.nginx-server.default.svc.cluster.local (172.16.98.235): icmp_seq=1 ttl=63 time=0.080 ms
    64 bytes from 172-16-98-235.nginx-server.default.svc.cluster.local (172.16.98.235): icmp_seq=2 ttl=63 time=0.104 ms
    ^C
    --- nginx-server.default.svc.cluster.local ping statistics ---
    2 packets transmitted, 2 received, 0% packet loss, time 2ms
    rtt min/avg/max/mdev = 0.080/0.092/0.104/0.012 ms
    
    ```

# ingress

ingreess主要是将服务暴露给外部 提供给外部访问 是将 service暴露出去的 ingreess会根据规则将流量转发到service

参考：https://blog.51cto.com/happylab/2463744，https://kubernetes.io/zh/docs/concepts/services-networking/ingress/

在kubernetes中对外暴露服务的方式有两种：service（NodePort或者外部LoadBalancer）和ingress，其中service是提供四层的负载均衡，通过iptables DNAT或lvs nat模式实现后端Pod的代理请求。如需实现http，域名，URI，证书等请求方式，service是无法实现的，需要借助于ingress来来实现，本文将来介绍ingress相关的内容。相当于一个七层的负载均衡

Ingress 是对集群中服务的外部访问进行管理的 API 对象，典型的访问方式是 HTTP。

Ingress 可以提供负载均衡、SSL 终结和基于名称的虚拟托管。

[Ingress](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.21/#ingress-v1beta1-networking-k8s-io) 公开了从集群外部到集群内[服务](https://kubernetes.io/zh/docs/concepts/services-networking/service/)的 HTTP 和 HTTPS 路由。 流量路由由 Ingress 资源上定义的规则控制。

Ingress 必须有个Ingress controller,Ingress controller就是实现规则的一个定义

Ingress controller默认没有实现的，需要第三方厂商来做，

实现Ingress包含的组件有：

- Ingress，客户端，负责定义ingress配置，将请求转发给Ingress Controller；
- Ingress Controller，Ingress控制器，实现七层转发的Edge Router，通过调用k8s的api动态感知集群中Pod的变化而动态更新配置文件并重载， Controller需要部署在k8s集群中以实现和集群中的pod通信，通常以DaemonSets或Deployments的形式部署，并对外暴露80和443端口，对于DaemonSets来说，一般是以hostNetwork或者hostPort的形式暴露，Deployments则以NodePort的方式暴露，控制器的多个节点则借助外部负载均衡ExternalLB以实现统一接入；
- Ingress配置规则，Controller控制器通过service服务发现机制动态实现后端Pod路由转发规则的实现；
- Service，kuberntes中四层的负载均衡调度机制，Ingress借助service的服务发现机制实现集群中Pod资源的动态感知；
- Pod，后端实际负责响应请求容器，由控制器如Deployment创建，通过标签Labels和service关联，服务发现。

Ingress Controller部署在k8s的pod中，就会和k8s里的集群的pod就通了Ingress Controller会借助于service来实现pod的ip变化来重新映射

简而言之，ingress控制器借助service的服务发现机制实现配置的动态更新以实现Pod的负载均衡机制实现，由于涉及到Ingress Controller的动态更新，目前社区Ingress Controller大体包含两种类型的控制器：

- 传统的七层负载均衡如Nginx，HAproxy，开发了适应微服务应用的插件，具有成熟，高性能等优点；
- 新型微服务负载均衡如Traefik，Envoy，Istio，专门适用于微服务+容器化应用场景，具有动态更新特点；

nginx的Ingress Controller是基于service部署的，在流量到service的nodePort上后，直接将流量转发到pod上没有经过service，参考官网：https://www.nginx.com/products/nginx-ingress-controller/

1. 部署nginx Ingress Controller

   参考：https://github.com/nginxinc/kubernetes-ingress/tree/master/deployments，https://docs.nginx.com/nginx-ingress-controller/installation/

   支持两种manifests和helm，manifests支持deployments和daemon-set方式

   ```
   #拉取代码
   [root@k8snode1 nginxController]# git clone https://github.com/nginxinc/kubernetes-ingress.git
   [root@k8snode1 nginxController]# cd kubernetes-ingress-master/
   build/                        docs-web/                     grafana/                      pkg/
   cmd/                          examples/                     hack/                         tests/
   deployments/                  examples-of-custom-resources/ internal/                     
   docs/                         .github/                      perf-tests/                   
   [root@k8snode1 nginxController]# cd kubernetes-ingress-master/deployments/
   #配置rbac认证
   [root@k8snode1 common]#  kubectl apply -f common/ns-and-sa.yaml 
   namespace/nginx-ingress created
   serviceaccount/nginx-ingress created
   
   #权限绑定 就是clusterrole和clusterrolebinding之间的绑定
   [root@k8snode1 deployments]# kubectl apply -f rbac/rbac.yaml
   clusterrole.rbac.authorization.k8s.io/nginx-ingress created
   clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress created
   
   #配置通用资源
   [root@k8snode1 deployments]#  kubectl apply -f common/default-server-secret.yaml
   secret/default-server-secret created
   [root@k8snode1 deployments]# kubectl apply -f common/nginx-config.yaml
   configmap/nginx-config created
   [root@k8snode1 deployments]# kubectl apply -f common/ingress-class.yaml
   Warning: networking.k8s.io/v1beta1 IngressClass is deprecated in v1.19+, unavailable in v1.22+; use networking.k8s.io/v1 IngressClassList
   ingressclass.networking.k8s.io/nginx created
   
   [root@k8snode1 deployments]# kubectl apply -f common/crds/k8s.nginx.org_virtualservers.yaml
   customresourcedefinition.apiextensions.k8s.io/virtualservers.k8s.nginx.org created
   [root@k8snode1 deployments]# kubectl apply -f common/crds/k8s.nginx.org_virtualserverroutes.yaml
   customresourcedefinition.apiextensions.k8s.io/virtualserverroutes.k8s.nginx.org created
   [root@k8snode1 deployments]#  kubectl apply -f common/crds/k8s.nginx.org_transportservers.yaml
   customresourcedefinition.apiextensions.k8s.io/transportservers.k8s.nginx.org created
   [root@k8snode1 deployments]# kubectl apply -f common/crds/k8s.nginx.org_policies.yaml
   customresourcedefinition.apiextensions.k8s.io/policies.k8s.nginx.org created
   [root@k8snode1 deployments]# kubectl apply -f common/crds/k8s.nginx.org_globalconfigurations.yaml
   customresourcedefinition.apiextensions.k8s.io/globalconfigurations.k8s.nginx.org created
   
   #部署nginx-ingress
   [root@k8snode1 deployments]# kubectl apply -f deployment/nginx-ingress.yaml
   deployment.apps/nginx-ingress created
   
   [root@k8snode1 deployments]# kubectl get ns
   NAME                   STATUS   AGE
   default                Active   5d17h
   kube-node-lease        Active   5d17h
   kube-public            Active   5d17h
   kube-system            Active   5d17h
   kubernetes-dashboard   Active   3d23h
   ndemo                  Active   3d17h
   nginx-ingress          Active   5m49s
   tigera-operator        Active   5d17h
   
   #可有看到有一个副本已经部署成功
   [root@k8snode1 deployments]# kubectl get deployments.apps -n nginx-ingress
   NAME            READY   UP-TO-DATE   AVAILABLE   AGE
   nginx-ingress   1/1     1            1           48s
   
   #查看pod已经run
   [root@k8snode1 deployments]# kubectl get pods --namespace=nginx-ingress
   NAME                             READY   STATUS    RESTARTS   AGE
   nginx-ingress-5497bdccb5-f6qfw   1/1     Running   0          85s
   
   #可以改成3副本
   [root@k8snode1 deployments]# vim deployment/nginx-ingress.yaml
   
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nginx-ingress
     namespace: nginx-ingress
   spec:
     replicas: 3  #三副本
     selector:
       matchLabels:
         app: nginx-ingress
     template:
       metadata:
         labels:
           app: nginx-ingress
        #annotations:
          #prometheus.io/scrape: "true"
          #prometheus.io/port: "9113"
       spec:
         serviceAccountName: nginx-ingress
         containers:
         - image: nginx/nginx-ingress:1.11.1
           imagePullPolicy: IfNotPresent
           name: nginx-ingress
           ports:
           - name: http
             containerPort: 80
           - name: https
             containerPort: 443
           - name: readiness-port
             containerPort: 8081
          #- name: prometheus
            #containerPort: 9113
           readinessProbe:
             httpGet:
               path: /nginx-ready
               port: readiness-port
             periodSeconds: 1
           securityContext:
             allowPrivilegeEscalation: true
             runAsUser: 101 #nginx
             capabilities:
               drop:
               - ALL
               add:
               - NET_BIND_SERVICE
           env:
           - name: POD_NAMESPACE
             valueFrom:
               fieldRef:
                 fieldPath: metadata.namespace
           - name: POD_NAME
             valueFrom:
               fieldRef:
                 fieldPath: metadata.name
           args:
             - -nginx-configmaps=$(POD_NAMESPACE)/nginx-config
             - -default-server-tls-secret=$(POD_NAMESPACE)/default-server-secret
            #- -v=3 # Enables extensive logging. Useful for troubleshooting.
            #- -report-ingress-status
            #- -external-service=nginx-ingress
            #- -enable-prometheus-metrics
            #- -global-configuration=$(POD_NAMESPACE)/nginx-configuration
   
   
   [root@k8snode1 deployments]# kubectl apply -f deployment/nginx-ingress.yaml
   deployment.apps/nginx-ingress configured
   [root@k8snode1 deployments]# kubectl get pods --namespace=nginx-ingress -o wide
   NAME                             READY   STATUS    RESTARTS   AGE     IP              NODE       NOMINATED NODE   READINESS GATES
   nginx-ingress-5497bdccb5-f6qfw   1/1     Running   0          4m22s   172.16.98.228   k8snode3   <none>           <none>
   nginx-ingress-5497bdccb5-tn5sw   1/1     Running   0          21s     172.16.98.239   k8snode3   <none>           <none>
   nginx-ingress-5497bdccb5-wqzmh   1/1     Running   0          21s     172.16.98.252   k8snode3   <none>           <none>
   
   
   
   #暴露service
   #以nodeport方式暴露 在每个节点都会暴露这俩端口
   [root@k8snode1 deployments]# kubectl get services -n nginx-ingress 
   NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
   nginx-ingress   NodePort   10.108.246.97   <none>        80:30556/TCP,443:31631/TCP   14s
   
   [root@k8snode1 deployments]# curl http://k8snode2:30556
   
   #如果是云环境 就以LoadBalancer 暴露
   ```

2. ingress资源定义

   上面的章节已安装了一个Nginx Ingress Controller控制器，有了Ingress控制器后，我们就可以定义Ingress资源来实现七层负载转发了，大体上Ingress支持三种使用方式：1. 基于虚拟主机转发，2. 基于虚拟机主机URI转发，3. 支持TLS加密转发。

   ```
   # 环境准备，先创建一个nginx的Deployment应用，包含2个副本
   [root@k8snode1 deployments]# kubectl create deployment ingress-demo --image=nginx:1.7.9 --port=80 
   deployment.apps/ingress-demo created
   [root@k8snode1 deployments]# kubectl scale deployment ingress-demo --replicas=2
   deployment.apps/ingress-demo scaled
   
   [root@k8snode1 deployments]# kubectl get deployments.apps 
   NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
   deployment-app         3/3     3            3           41h
   deployment-app-new     3/3     3            3           40h
   ingress-demo           2/2     2            2           16s
   nginx-dashboard-demo   3/3     3            3           3d23h
   nginxdemo1             4/4     4            4           4d17h
   service-demo           3/3     3            3           18h
   
   #将端口暴露出去
   [root@k8snode1 deployments]# kubectl expose deployment ingress-demo 
   service/ingress-demo exposed
   
   [root@k8snode1 deployments]# kubectl get service
   NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)        AGE
   deployment-app                       ClusterIP      10.111.12.80     <none>          80/TCP         40h
   external-demo                        ExternalName   <none>           www.baidu.com   80/TCP         17h
   ingress-demo                         ClusterIP      10.99.158.239    <none>          80/TCP         26s
   kubernetes                           ClusterIP      10.96.0.1        <none>          443/TCP        5d18h
   nginx-server                         ClusterIP      None             <none>          80/TCP         17h
   nginx-tcp-readiness-liveness-probe   ClusterIP      10.104.193.231   <none>          80/TCP         42h
   nginxdemo1                           NodePort       10.105.52.111    <none>          80:31204/TCP   4d2h
   
   #单个服务的暴露定义一个ingress对象将起转发至ingress-demo这个service，通过ingress.class指定控制器的类型为nginx
   [root@k8snode1 deployments]# vim nginx-ingress-demo.yaml 
   
   apiVersion: extensions/v1beta1
   kind: Ingress
   metadata:
     name: ingress-demo
     labels:
       ingres-controller: nginx
     annotations:
       kubernets.io/ingress.class: nginx
   spec:
     rules:  #后端多主机 通过host方式定义
     - host: www.happylau.cn
       http:
         paths:
         - path: /
           backend:
             serviceName: ingress-demo
             servicePort: 80
   
   
   #通过ip直接访问后端主机
   [root@k8snode1 deployments]# vim nginx-ingress-demo.yaml 
   
   apiVersion: extensions/v1beta1
   kind: Ingress
   metadata:
     name: ingress-demo
   spec:
     backend:  
       serviceName: ingress-demo
       servicePort: 80
   
   [root@k8snode1 deployments]# kubectl apply -f nginx-ingress-demo.yaml 
   Warning: extensions/v1beta1 Ingress is deprecated in v1.14+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
   ingress.extensions/ingress-demo created
   
   [root@k8snode1 deployments]# kubectl apply -f nginx-ingress-demo.yaml 
   Warning: extensions/v1beta1 Ingress is deprecated in v1.14+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
   ingress.extensions/ingress-demo created
   [root@k8snode1 deployments]# kubectl get ingress
   NAME           CLASS    HOSTS   ADDRESS   PORTS   AGE
   ingress-demo   <none>   *                 80      17s
   
   #查看详情
   [root@k8snode1 deployments]# kubectl describe ingress ingress-demo
   Name:             ingress-demo
   Namespace:        default
   Address:          
   Default backend:  ingress-demo:80 (172.16.185.212:80,172.16.185.214:80) #匹配到两个后端pod的Ip
   Rules:
     Host        Path  Backends
     ----        ----  --------
     *           *     ingress-demo:80 (172.16.185.212:80,172.16.185.214:80)
   Annotations:  <none>
   Events:       <none>
   
   #添加规则
   [root@k8snode1 deployments]# vim nginx-ingress-demo.yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: ingress-demo
     labels:
       ingres-controller: nginx
     annotations:
       kubernets.io/ingress.class: nginx
   spec:
     rules:  #后端多主机 通过host方式定义
     - host: www.imakj.com  #域名 如果匹配到这个域名了，则转发到这里 如果没有 则转发到下面的backend里
       http:
         paths:
         - path: /  #path路径
           pathType: Prefix
           backend:
             service:
               name: ingress-demo
               port:
                 number: 80
     defaultBackend:
       service:
         name: ingress-demo
         port:
           number: 80
   
   #可以看到已经出来了
   [root@k8snode1 deployments]# kubectl get ingress
   NAME           CLASS    HOSTS           ADDRESS   PORTS   AGE
   ingress-demo   <none>   www.imakj.com             80      20m
   
   [root@k8snode1 deployments]# kubectl describe ingress ingress-demo
   Name:             ingress-demo
   Namespace:        default
   Address:          
   Default backend:  ingress-demo:80 (172.16.185.212:80,172.16.185.214:80)
   Rules:
     Host           Path  Backends
     ----           ----  --------
     www.imakj.com  
                    /   ingress-demo:80 (172.16.185.212:80,172.16.185.214:80)
   Annotations:     kubernets.io/ingress.class: nginx
   Events:          <none>
   
   [root@k8snode1 deployments]# kubectl get service -n nginx-ingress 
   NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
   nginx-ingress   NodePort   10.108.246.97   <none>        80:30556/TCP,443:31631/TCP   58m
   
   #访问测试
   [root@k8snode1 deployments]# curl -I www.imakj.com --resolve www.imakj.cn:30556:172.16.185.212
   HTTP/1.1 200 OK
   Server: nginx/1.7.9
   Date: Fri, 23 Apr 2021 09:56:37 GMT
   Content-Type: text/html
   Content-Length: 612
   Last-Modified: Tue, 23 Dec 2014 16:25:09 GMT
   Connection: keep-alive
   ETag: "54999765-264"
   Accept-Ranges: bytes
   
   
   #修改hosts
   [root@k8snode1 deployments]# vim /etc/hosts
   
   127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4
   ::1 localhost localhost.localdomain localhost6 localhost6.localdomain6
   192.168.10.185 k8snode1
   192.168.10.186 k8snode2
   192.168.10.187 k8snode3
   172.16.185.212 www.imakj.com
   
   [root@k8snode1 deployments]# curl www.imakj.com
   <!DOCTYPE html>
   <html>
   <head>
   <title>Welcome to nginx!</title>
   <style>
       body {
           width: 35em;
           margin: 0 auto;
           font-family: Tahoma, Verdana, Arial, sans-serif;
       }
   </style>
   </head>
   <body>
   <h1>Welcome to nginx!</h1>
   <p>If you see this page, the nginx web server is successfully installed and
   working. Further configuration is required.</p>
   
   <p>For online documentation and support please refer to
   <a href="http://nginx.org/">nginx.org</a>.<br/>
   Commercial support is available at
   <a href="http://nginx.com/">nginx.com</a>.</p>
   
   <p><em>Thank you for using nginx.</em></p>
   </body>
   </html>
   
   ```

3. Nginx Ingress Controller实现机制

   其实也是通过nginx的配置反向代理实现的，通过域名映射podip实现，如果pod发生变化了Ingress Controller 就会重新映射并重新加载，所以在大规模情况下 deployments频繁变化的情况下，nginx就会频繁重新加载影响效率

   ```
   #进入到容器里
   [root@k8snode1 deployments]# kubectl get pods -n nginx-ingress 
   NAME                             READY   STATUS    RESTARTS   AGE
   nginx-ingress-5497bdccb5-f6qfw   1/1     Running   0          80m
   nginx-ingress-5497bdccb5-tn5sw   1/1     Running   0          76m
   nginx-ingress-5497bdccb5-wqzmh   1/1     Running   0          76m
   
   [root@k8snode1 deployments]# kubectl exec -it nginx-ingress-5497bdccb5-f6qfw -n nginx-ingress /bin/bash
   kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
   #可以看到nginx配置文件
   nginx@nginx-ingress-5497bdccb5-f6qfw:/$ cd /etc/nginx/
   nginx@nginx-ingress-5497bdccb5-f6qfw:/etc/nginx$ ls
   conf.d		     fastcgi_params  koi-win	 modules     scgi_params  stream-conf.d  win-utf
   config-version.conf  koi-utf	     mime.types  nginx.conf  secrets	  uwsgi_params
   nginx@nginx-ingress-5497bdccb5-f6qfw:/etc/nginx$ cat nginx.conf 
   
   worker_processes  auto;
   daemon off;
   
   error_log  stderr notice;
   pid        /var/lib/nginx/nginx.pid;
   
   events {
       worker_connections  1024;
   }
   
   http {
       include       /etc/nginx/mime.types;
       default_type  application/octet-stream;
   
       log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                         '$status $body_bytes_sent "$http_referer" '
                         '"$http_user_agent" "$http_x_forwarded_for"';
   
       
       access_log  /dev/stdout  main;
       
   
       
   
       sendfile        on;
       #tcp_nopush     on;
   
       keepalive_timeout 65s;
       keepalive_requests 100;
   
       #gzip  on;
   
       server_names_hash_max_size 1024;
       server_names_hash_bucket_size 256;
   
       variables_hash_bucket_size 256;
       variables_hash_max_size 1024;
   
       map $http_upgrade $connection_upgrade {
           default upgrade;
           ''      close;
       }
       map $http_upgrade $vs_connection_header {
           default upgrade;
           ''      $default_connection_header;
       }
       
       
       
       
   
       
       
   
       server {
           # required to support the Websocket protocol in VirtualServer/VirtualServerRoutes
           set $default_connection_header "";
           set $resource_type "";
           set $resource_name "";
           set $resource_namespace "";
           set $service "";
   
           listen 80 default_server;
   
           
           listen 443 ssl default_server;
           
   
           ssl_certificate /etc/nginx/secrets/default;
           ssl_certificate_key /etc/nginx/secrets/default;
   
           
           
           
   
           server_name _;
           server_tokens "on";
           
   
           
   
           
   
           location / {
               return 404;
           }
       }
       # stub_status
       server {
           listen 8080;
           
           allow 127.0.0.1;
           deny all;
           
           location /stub_status {
               stub_status;
           }
       }
   
       include /etc/nginx/config-version.conf;
       include /etc/nginx/conf.d/*.conf;  #看到includ的文件 查看这个文件
   
       server {
           listen unix:/var/lib/nginx/nginx-502-server.sock;
           access_log off;
   
           
   
           return 502;
       }
   
       server {
           listen unix:/var/lib/nginx/nginx-418-server.sock;
           access_log off;
   
           
   
           return 418;
       }
   }
   
   stream {
       log_format  stream-main  '$remote_addr [$time_local] '
                         '$protocol $status $bytes_sent $bytes_received '
                         '$session_time "$ssl_preread_server_name"';
   
       access_log  /dev/stdout  stream-main;
   
       
   
       
   
       include /etc/nginx/stream-conf.d/*.conf;
   }
   
   
   ```

# ingress虚拟主机

1. ingress支持基于名称的虚拟主机，实现单个IP多个域名转发的需求，通过请求头部携带主机名方式区分开

```
#创建一个新的
[root@k8snode1 ~]# kubectl create deployment ingress-demo-1 --image=nginx:1.7.9 --port=80 
deployment.apps/ingress-demo-1 created
[root@k8snode1 ~]# kubectl expose deployment ingress-demo-1 --port=80
service/ingress-demo-1 exposed

#修改内容 便于区分
[root@k8snode1 ~]# kubectl exec -it ingress-demo-1-54c6f87769-rk82q /bin/bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
root@ingress-demo-1-54c6f87769-rk82q:/# echo sports > /usr/share/nginx/html/index.html

[root@k8snode1 deployments]# vim nginx-ingress-demo.yaml 

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-demo
  labels:
    ingres-controller: nginx
  annotations:
    kubernets.io/ingress.class: nginx
spec:
  rules:  #后端多主机 通过host方式定义
  - host: www.imakj.com  #域名 如果匹配到这个域名了，则转发到这里 如果没有 则转发到下面的backend里
    http:
      paths:
      - path: /  #path路径
        pathType: Prefix
        backend:
          service:
            name: ingress-demo
            port:
              number: 80
  - host: sports.imakj.com  #域名 如果匹配到这个域名了，则转发到这里 如果没有 则转发到下面的backend里
    http:
      paths:
      - path: /  #path路径
        pathType: Prefix
        backend:
          service:
            name: ingress-demo-1
            port:
              number: 80
  defaultBackend:   #上面两个都没匹配的话，默认就是这个
    service:
      name: ingress-demo
      port:
        number: 80

#匹配规则有俩了
[root@k8snode1 deployments]# kubectl describe ingress ingress-demo 
Name:             ingress-demo
Namespace:        default
Address:          
Default backend:  ingress-demo:80 (172.16.185.212:80,172.16.185.214:80)
Rules:
  Host              Path  Backends
  ----              ----  --------
  www.imakj.com     
                    /   ingress-demo:80 (172.16.185.212:80,172.16.185.214:80)
  sports.imakj.com  
                    /   ingress-demo-1:80 (172.16.98.246:80)
Annotations:        kubernets.io/ingress.class: nginx
Events:             <none>

```

2. 配置url转发

   ```
   [root@k8snode1 deployments]# vim nginx-ingress-url-demo.yaml 
   
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: ingress-demo-url
   spec:
     rules:  #后端多主机 通过host方式定义
     - host: www.imakj.com  #域名 如果匹配到这个域名了，则转发到这里 如果没有 则转发到下面的backend里
       http:
         paths:
         - path: /new  #path路径匹配
           pathType: Prefix
           backend:
             service:
               name: ingress-demo
               port:
                 number: 80
         - path: /sports  #path路径
           pathType: Prefix
           backend:
             service:
               name: ingress-demo-1
               port:
                 number: 80
     defaultBackend:
       service:
         name: ingress-demo
         port:
           number: 80
   
   #可以看到有俩转发
   [root@k8snode1 deployments]#  kubectl describe ingress ingress-demo-url 
   Name:             ingress-demo-url
   Namespace:        default
   Address:          
   Default backend:  ingress-demo:80 (172.16.185.212:80,172.16.185.214:80)
   Rules:
     Host           Path  Backends
     ----           ----  --------
     www.imakj.com  
                    /new      ingress-demo:80 (172.16.185.212:80,172.16.185.214:80)
                    /sports   ingress-demo-1:80 (172.16.98.246:80)
   Annotations:     <none>
   Events:          <none>
   
   ```

3. tls加密

   ```
   #先用openssl生秘钥
   [root@k8snode1 ~]# openssl req -x509 -newkey rsa:2048 -nodes -days 365 -keyout tls.key -out tls.crt
   Generating a 2048 bit RSA private key
   ....................................................+++
   ........................................+++
   writing new private key to 'tls.key'
   -----
   You are about to be asked to enter information that will be incorporated
   into your certificate request.
   What you are about to enter is what is called a Distinguished Name or a DN.
   There are quite a few fields but you can leave some blank
   For some fields there will be a default value,
   If you enter '.', the field will be left blank.
   -----
   Country Name (2 letter code) [XX]:CN        #国家
   State or Province Name (full name) []:GD    #省份
   Locality Name (eg, city) [Default City]:ShenZhen  #城市
   Organization Name (eg, company) [Default Company Ltd]:Tencent    #公司 
   Organizational Unit Name (eg, section) []:HappyLau  #组织
   Common Name (eg, your name or your server's hostname) []:www.happylau.cn  #域名
   Email Address []:573302346@qq.com       #邮箱地址
   
   #tls.crt为证书，tls.key为私钥
   [root@k8snode1 ~]# ls tls.* -l
   -rw-r--r-- 1 root root 1428 12月 26 13:21 tls.crt
   -rw-r--r-- 1 root root 1708 12月 26 13:21 tls.key
   
   #配置Secrets，将证书和私钥配置到Secrets中
   [root@k8snode1 ~]# kubectl create secret tls makj-sslkey --cert=tls.crt --key=tls.key 
   secret/happylau-sslkey created
   
   查看Secrets详情,证书和私要包含在data中，文件名为两个不同的key：tls.crt和tls.key
   [root@k8snode1 ~]# kubectl describe secrets makj-sslkey 
   Name:         makj-sslkey
   Namespace:    default
   Labels:       <none>
   Annotations:  <none>
   
   Type:  kubernetes.io/tls
   
   Data
   ====
   tls.crt:  1428 bytes
   tls.key:  1708 bytes
   
   [root@k8snode1 deployments]# vim nginx-ingress-tls.yaml 
   
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: ingress-demo
     labels:
       ingres-controller: nginx
     annotations:
       kubernets.io/ingress.class: nginx
   spec:
     tls:
     - hosts:
       - www.imakj.com   #需要匹配的域名
       secretName: makj-tls  #tls名字
     rules:  #
     - host: www.imakj.com  #域名 如果匹配到这个域名了，则转发到这里 如果没有 则转发到下面的backend里
       http:
         paths:
         - path: /  #path路径
           pathType: Prefix
           backend:
             service:
               name: ingress-demo
               port:
                 number: 80
   
   [root@k8snode1 deployments]# kubectl apply -f nginx-ingress-tls.yaml 
   ingress.networking.k8s.io/ingress-demo configured
   
   ```

4. 高级功能

   ingress controller提供了基础反向代理的功能，如果需要定制化nginx的特性或参数，需要通过ConfigMap和Annotations来实现，两者实现的方式有所不同，ConfigMap用于指定整个ingress集群资源的基本参数，修改后会被所有的ingress对象所继承；Annotations则被某个具体的ingress对象所使用，修改只会影响某个具体的ingress资源，冲突时其优先级高于ConfigMap。

5. ConfigMap自定义参数

   安装nginx ingress controller时默认会包含一个空的ConfigMap，可以通过ConfigMap来自定义nginx controller的默认参数，如下以修改一些参数为例：

   ```
   [root@k8snode1 deployments]# kubectl get configmaps -n nginx-ingress 
   NAME                            DATA   AGE
   kube-root-ca.crt                1      6h52m
   nginx-config                    0      6h49m
   nginx-ingress-leader-election   0      6h46m
   
   [root@k8snode1 deployments]# kubectl edit configmaps -n nginx-ingress nginx-config
   
   # Please edit the object below. Lines beginning with a '#' will be ignored,
   # and an empty file will abort the edit. If an error occurs while saving this file will be
   # reopened with the relevant failures.
   #
   apiVersion: v1
   kind: ConfigMap
   metadata:
     annotations:
       kubectl.kubernetes.io/last-applied-configuration: |
         {"apiVersion":"v1","data":null,"kind":"ConfigMap","metadata":{"annotations":{},"name":"nginx-config","namespace":"nginx-ingress"}}
     creationTimestamp: "2021-04-23T08:44:44Z"
     name: nginx-config
     namespace: nginx-ingress
     resourceVersion: "1094203"
     uid: 5a581359-2147-42fe-8d1b-34adbaaf8a25
   data:   #自定义的nginx参数
     client-max-body-size: 3m
     proxy-connect-timeout: 10s
     proxy-read-timeout: 10s
     proxy-send-timeout: 10s                                                                              
   
   Edit cancelled, no changes made
   
   [root@k8snode1 deployments]# kubectl get configmaps -n nginx-ingress 
   NAME                            DATA   AGE
   kube-root-ca.crt                1      6h57m
   nginx-config                    4      6h54m
   nginx-ingress-leader-election   0      6h51m
   
   
   #改完后在nginx的容器里就可以看到配置已经改成功
   [root@k8snode1 deployments]# kubectl get configmaps -n nginx-ingress nginx-config -o yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     annotations:
       kubectl.kubernetes.io/last-applied-configuration: |
         {"apiVersion":"v1","data":null,"kind":"ConfigMap","metadata":{"annotations":{},"name":"nginx-config","namespace":"nginx-ingress"}}
     creationTimestamp: "2021-04-23T08:44:44Z"
     name: nginx-config
     namespace: nginx-ingress
     resourceVersion: "1094203"
     uid: 5a581359-2147-42fe-8d1b-34adbaaf8a25
   [root@k8snode1 deployments]# kubectl get configmaps -n nginx-ingress nginx-config -o yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     annotations:
       kubectl.kubernetes.io/last-applied-configuration: |
         {"apiVersion":"v1","data":null,"kind":"ConfigMap","metadata":{"annotations":{},"name":"nginx-config","namespace":"nginx-ingress"}}
     creationTimestamp: "2021-04-23T08:44:44Z"
     name: nginx-config
     namespace: nginx-ingress
     resourceVersion: "1094203"
     uid: 5a581359-2147-42fe-8d1b-34adbaaf8a25
   [root@k8snode1 deployments]# kubectl edit configmaps -n nginx-ingress nginx-config
   configmap/nginx-config edited
   [root@k8snode1 deployments]# kubectl get configmaps -n nginx-ingress nginx-config -o ymal
   error: unable to match a printer suitable for the output format "ymal", allowed formats are: custom-columns,custom-columns-file,go-template,go-template-file,json,jsonpath,jsonpath-as-json,jsonpath-file,name,template,templatefile,wide,yaml
   [root@k8snode1 deployments]# kubectl get configmaps -n nginx-ingress nginx-config -o yaml
   apiVersion: v1
   data:  #修改的配置
     client-max-body-size: 3m
     proxy-connect-timeout: 10s
     proxy-read-timeout: 10s
     proxy-send-timeout: 10s
   kind: ConfigMap
   metadata:
     annotations:
       kubectl.kubernetes.io/last-applied-configuration: |
         {"apiVersion":"v1","data":null,"kind":"ConfigMap","metadata":{"annotations":{},"name":"nginx-config","namespace":"nginx-ingress"}}
     creationTimestamp: "2021-04-23T08:44:44Z"
     name: nginx-config
     namespace: nginx-ingress
     resourceVersion: "1168663"
     uid: 5a581359-2147-42fe-8d1b-34adbaaf8a25
   
   ```

# PV:PersistentVolume

PV:PersistentVolume ：持久数据卷 对存储资源抽象化 使存储作为集群中的资源来管理 用来对接后端存储

* PersistentVolume 一个pod需要挂载的持久化存储

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs  #存储类型
spec:
  capacity:
    storage: 10Gi #存储大小
  accessModes:
    - ReadWriteOnce  #只能由一个容器访问
  nfs:  #nfs信息
    path: "/tmp"
    server: 172.22.1.2
```

* PersistentVolumeClaim 一个pod希望使用的持久化的存储的需求  例如 下面pod需要一个又10G空间独占的需求

  ```
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: nfs  #存储类型
  spec:
    accessModes:
      - ReadWriteOnce  #只能由一个容器访问
    resources:
      requests:
        storage: 10Gi
  ```

* StorageClass 自动创建pv  StorageClass 可以和pvc绑定 来方便的管理pod的pv 

  每个pv和pvc都会有个StorageClass 如果没明显指定 则会创建一个空的StorageClass 

  pv和pvc使用相同的StorageClass 的时候  就会被自动绑定在一起

  一个pod可以使用多个pvc

  一个pvc可以对多个pod提供服务

  一个pv只能绑定一个pvc 一个pvc只能对应一个后端存储

  ![](images\k8s-存储.png)

  ```
  apiVersion: storage.k8s.io/v1
  kind: StorageClass 
  metadata:
    name: storage-cleass-demo
  provisioner: kubernetes.io/aws-ebs #aws的存储插件
  parameters:
    type: io1
    zone: us-east-1d
    iopsPerGB: "10"
    
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: pvc-demo
  spec:
    accessModes:
      - ReadWriteOnce  #只能由一个容器访问
    stroageClassName:  storage-cleass-demo #关联上面的存储
    resources:
      requests:
        storage: 10Gi
  ```

# PVC

持久数据卷申请，用户自定义使用的存储容量，是的用户不需要关注后端存储将后端存储封装起来 用来对接pv

PVC相当于应用和PV对接的一个接口 应用申请存储资源的时候通过pvc向pv申请

一个pvc只能对应绑定一个pv

pv可以定义多个

关系为：存储 -> pv -> pvc -> pod

1、pv与pvc什么关系？
一对一关系
2、pv与pvc匹配条件？
根据存储容量和访问模式匹配
3、存储容量匹配策略？
匹配最接近的PV容量（向大的匹配）
4、存储容量起到限制的作用？
从pv、pvc资源来讲，不具备限制能力，具体还得根据后端存储而定。
5、是否可以修改PV容量
k8s再不断去完善可以通过调整pv的容量，改变后端的存储的限制。

# StorageClass 

PV的动态创建供给 动态创建pv

deployment直接指定storageclass来实现pv和pvc的创建

 默认不支持nfs和ceph 需要使用社区的插件包

相当于一个插件程序 在pod申请存储的时候 通知storageclass插件，由插件自动创建pv和pvc

# StateFulSet 有状态的应用

StateFulSet

* 多个pod之间具有顺序性

* 对于持久化存储的区分

  StateFulSet的pod不能分配clusterip 是一个无头服务 定义无头服务的clusterIP应该为None 例如：clusterIP: None 依赖的service是headless service

  需要指定serviceName为headless service

* 因为每个pod都是由状态的 所以需要独享存储，需要使用VolumeClaimTemplate

多个pod之间具有顺序性表现：

```
# 先定义一个service
[root@k8shardway1 10-statefulset]# vim headless-service.yaml 

apiVersion: v1
kind: Service
metadata:
  name: springboot-web-svc  #名字为 springboot-web-svc
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  clusterIP: None  #clusterIP 为None
  selector:
    app: springboot-web  #这个leable就是下面的statefulset.yaml的app: springboot-web

#定义一个stateFulset
[root@k8shardway1 10-statefulset]# vim statefulset.yaml 

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: springboot-web
spec:
  serviceName: springboot-web-svc  #定义service name
  replicas: 2
  selector:
    matchLabels:
      app: springboot-web
  template:
    metadata:
      labels:
        app: springboot-web
    spec:
      containers:
      - name: springboot-web
        image: nginx:1.7.9
        ports:
        - containerPort: 80
        livenessProbe:
          tcpSocket:
            port: 80
          initialDelaySeconds: 20
          periodSeconds: 10
          failureThreshold: 3
          successThreshold: 1
          timeoutSeconds: 5
        readinessProbe:
          httpGet:
            path: /index.html
            port: 80
            scheme: HTTP
          initialDelaySeconds: 20
          periodSeconds: 10
          failureThreshold: 1
          successThreshold: 1
          timeoutSeconds: 5
          
[root@k8shardway1 10-statefulset]# kubectl apply -f headless-service.yaml 
service/springboot-web-svc created
[root@k8shardway1 10-statefulset]# kubectl apply -f statefulset.yaml 
statefulset.apps/springboot-web created


#监控一下 可以看到 两个pod是顺序创建的 先第一个创建成功后才会创建第二个
[root@k8snode1 k8s]# kubectl get pods -l app=springboot-web -w
NAME               READY   STATUS    RESTARTS   AGE
springboot-web-0   1/1     Running   0          65s
springboot-web-1   1/1     Running   0          35s


#进入到容器 可以看到 其实名字就叫springboot-web-0 就是名字+编号的行使 标明了容器的顺序性
[root@k8snode1 k8s]# kubectl exec -it springboot-web-0 /bin/bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
root@springboot-web-0:/# cat /etc/host
cat: /etc/host: No such file or directory
root@springboot-web-0:/# cat /etc/hosts
# Kubernetes-managed hosts file.
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
fe00::0	ip6-mcastprefix
fe00::1	ip6-allnodes
fe00::2	ip6-allrouters
172.16.185.254	springboot-web-0.springboot-web-svc.default.svc.cluster.local	springboot-web-0

```

持久化存储

```
# 先定义一个service
[root@k8shardway1 10-statefulset]# vim headless-service.yaml 

apiVersion: v1
kind: Service
metadata:
  name: springboot-web-svc  #名字为 springboot-web-svc
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  clusterIP: None  #clusterIP 为None
  selector:
    app: springboot-web  #这个leable就是下面的statefulset.yaml的app: springboot-web

[root@k8snode1 k8s]# vim statefulset-volume.yaml 

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: springboot-web
spec:
  serviceName: springboot-web-svc  #定义service name
  replicas: 2
  selector:
    matchLabels:
      app: springboot-web
  template:
    metadata:
      labels:
        app: springboot-web
    spec:
      containers:
      - name: springboot-web
        image: nginx:1.7.9
        ports:
        - containerPort: 80
        livenessProbe:
          tcpSocket:
            port: 80
          initialDelaySeconds: 20
          periodSeconds: 10
          failureThreshold: 3
          successThreshold: 1
          timeoutSeconds: 5
        readinessProbe:
          httpGet:
            path: /index.html
            port: 80
            scheme: HTTP
          initialDelaySeconds: 20
          periodSeconds: 10
          failureThreshold: 1
          successThreshold: 1
          timeoutSeconds: 5
  volumeClaimTemplates:   #定义一个volumeClaimTemplates 创建一个pvc的模板 下面就是pvc的消息
  - metadata:
      name: data
    spec:
      accessModes:
      - ReadWriteOnce
      storageClassName: gulustrfs-storage-class
      resources:
        requests:
          storage: 1Gi

```

# 数据卷

empotyDir卷：一个临时的存储卷 和pod生命周期绑定在一起 随着pod删除而删除 就单纯的为了多个container 共享数据目录而存在的 在pod所在的节点创建一个共享目录

hostPath卷：将宿主机的文件目录挂载到pod容器中

# 准入控制器

相当于一个拦截器 拦截每次从apiserver请求到k8s的请求数据 主要用于二次开发使用

# Kubernetes API 自定义api管理

参考文档：https://kubernetes.io/docs/concepts/overview/kubernetes-api/

每个yaml文件里都会有一个apiversion apiversion分为以下两个/api 和/apis

/api下 都是核心资源 是没有分组的 只有两级 一级是版本 下面以及就是资源

/apis 非核心api 有三级  每个资源都用非核心的  一级是版本 二级是组 三级才是资源

![](images\k8s-api.png)

apiservice就是基于rusfapi的 

```
# 打开api
[root@k8shardway1 ~]# vim /etc/systemd/system/kube-apiserver.service 

[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \
  --advertise-address=192.168.10.180 \
  --allow-privileged=true \
  --apiserver-count=2 \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/var/log/audit.log \
  --authorization-mode=Node,RBAC \
  --bind-address=0.0.0.0 \
  --insecure-port=8080   #这个参数
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
  --etcd-cafile=/etc/kubernetes/ssl/ca.pem \
  --etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem \
  --etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem \
  --etcd-servers=https://192.168.10.180:2379,https://192.168.10.181:2379,https://192.168.10.182:2379 \
  --event-ttl=1h \
  --kubelet-certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --kubelet-client-certificate=/etc/kubernetes/ssl/kubernetes.pem \
  --kubelet-client-key=/etc/kubernetes/ssl/kubernetes-key.pem \
  --service-account-issuer=api \
  --service-account-key-file=/etc/kubernetes/ssl/service-account.pem \
  --service-account-signing-key-file=/etc/kubernetes/ssl/service-account-key.pem \
  --api-audiences=api,vault,factors \
  --service-cluster-ip-range=10.233.0.0/16 \
  --service-node-port-range=30000-32767 \
  --proxy-client-cert-file=/etc/kubernetes/ssl/proxy-client.pem \
  --proxy-client-key-file=/etc/kubernetes/ssl/proxy-client-key.pem \
  --runtime-config=api/all=true \
  --requestheader-client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --requestheader-allowed-names=aggregator \
  --requestheader-extra-headers-prefix=X-Remote-Extra- \
  --requestheader-group-headers=X-Remote-Group \
  --requestheader-username-headers=X-Remote-User \
  --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  --v=1
Restart=on-failure
RestartSec=5

[root@k8shardway1 ~]# systemctl daemon-reload
[root@k8shardway1 ~]# service kube-apiserver restart

```

还有相应的api 

https://github.com/kubernetes-client 有各种api

官方的api  https://github.com/kubernetes/client-go

 



# 命令行

* 参考官方文档：https://kubernetes.io/zh/docs/reference/kubectl/kubectl/
* 详细文档:https://kubernetes.io/zh/docs/reference/kubectl/overview/

* 资源文档：https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands
* 查看所有资源：`# kubectl api-resources`
* 查看帮助:`# kubectl -h`
* 查看子命令的帮助：`kubectl <command> --help`，例如：`kubectl apply -h`

# k8s生态

https://landscape.cncf.io/

* kube-api-server的责任
* etcd的责任，负责集群状态的持久化
* kube-scheduler 集群调度的集群，
* kub-controller-manager的责任

