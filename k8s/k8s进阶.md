# 1. etcd备份恢复

## kubeadm方式

### 备份

```
ETCDCTL_API=3 etcdctl \
snapshot save snap.db \
--endpoints=https://127.0.0.1:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key
```

### 恢复

```
1、先暂停kube-apiserver和etcd容器
mv /etc/kubernetes/manifests /etc/kubernetes/manifests.bak
mv /var/lib/etcd/ /var/lib/etcd.bak
2、恢复
ETCDCTL_API=3 etcdctl \
snapshot restore snap.db \
--data-dir=/var/lib/etcd
3、启动kube-apiserver和etcd容器
mv /etc/kubernetes/manifests.bak /etc/kubernetes/manifests
```

## 二进制模式

如果是集群模式 二进制恢复的时候需要带有集群信息 否则恢复会出现不完整 因为备份的时候 只有数据 没有集群信息

## 查看集群状态

```
# ETCDCTL_API=3 /opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --endpoints="https://192.168.10.170:2379,https://192.168.10.171:2379,https://192.168.10.172:2379" endpoint health --write-out=table
+-----------------------------+--------+-------------+-------+
|          ENDPOINT           | HEALTH |    TOOK     | ERROR |
+-----------------------------+--------+-------------+-------+
| https://192.168.10.170:2379 |   true |  9.679618ms |       |
| https://192.168.10.171:2379 |   true |  9.925938ms |       |
| https://192.168.10.172:2379 |   true | 11.388643ms |       |
+-----------------------------+--------+-------------+-------+
```

## 备份

### 创建一个容器 

```
# kubectl create deployment nginxtest --image=nginx
deployment.apps/nginxtest created
# kubectl get deployment
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
nginxtest   0/1     1            0           6s
# kubectl get pods
NAME                       READY   STATUS    RESTARTS   AGE
nginxtest-fc5f8b88-md9nb   1/1     Running   0          10s
```

### 开始备份

备份只需要在一个节点上备份就可以

制定当前节点ip和etcd证书 和etcd api版本

snapshot 命令

save 导出

sanp.db 导出文件名

endpoints 节点ip

```
# ETCDCTL_API=3 /opt/etcd/bin/etcdctl \
snapshot save snap.db \
--endpoints=https://192.168.10.170:2379 \
--cacert=/opt/etcd/ssl/ca.pem \
--cert=/opt/etcd/ssl/server.pem \
--key=/opt/etcd/ssl/server-key.pem

…………
Snapshot saved at snap.db 导出成功
```

### 恢复

恢复需要在所有节点执行 并且制定节点名字 节点名字必须和创建的时候的etcd配置文件一直/opt/etcd/cfg/etcd.conf

### 先停止etcd 和apiserver

所有etcd节点执行

```
# systemctl stop kube-apiserver
# systemctl stop etcd
# mv /var/lib/etcd/default.etcd /var/lib/etcd/default.etcd.bak
```

拷贝备份文件到其他两个节点

```
# scp snap.db 192.168.10.171:/opt/back/
# scp snap.db 192.168.10.171:/opt/back/
```

### 开始恢复

restore 恢复

sanp.db 导出文件名

```
# 192.168.10.170执行
cd /opt/back/ && ETCDCTL_API=3 /opt/etcd/bin/etcdctl snapshot restore snap.db \
--name etcd-1 \
--initial-cluster="etcd-1=https://192.168.10.170:2380,etcd-2=https://192.168.10.171:2380,etcd-3=https://192.168.10.172:2380" \
--initial-cluster-token=etcd-cluster \
--initial-advertise-peer-urls=https://192.168.10.170:2380 \
--data-dir=/var/lib/etcd/default.etcd

# 192.168.10.171执行
cd /opt/back/ && ETCDCTL_API=3 /opt/etcd/bin/etcdctl snapshot restore snap.db \
--name etcd-2 \
--initial-cluster="etcd-1=https://192.168.10.170:2380,etcd-2=https://192.168.10.171:2380,etcd-3=https://192.168.10.172:2380" \
--initial-cluster-token=etcd-cluster \
--initial-advertise-peer-urls=https://192.168.10.171:2380 \
--data-dir=/var/lib/etcd/default.etcd

# 192.168.10.172执行
cd /opt/back/ && ETCDCTL_API=3 /opt/etcd/bin/etcdctl snapshot restore snap.db \
--name etcd-3 \
--initial-cluster="etcd-1=https://192.168.10.170:2380,etcd-2=https://192.168.10.171:2380,etcd-3=https://192.168.10.172:2380" \
--initial-cluster-token=etcd-cluster \
--initial-advertise-peer-urls=https://192.168.10.172:2380 \
--data-dir=/var/lib/etcd/default.etcd
```

恢复完后启动

```
# systemctl start etcd
# 查看集群状态
# ETCDCTL_API=3 /opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --endpoints="https://192.168.10.170:2379,https://192.168.10.171:2379,https://192.168.10.172:2379" endpoint health --write-out=table

# 最后启动apiserver
# systemctl start kube-apiserver

# 查看k8s状态
# kubectl get nodes
```

# 版本升级

## kubeadm升级

1. 查看最新版本号

   可以看到当前版本为1.22.4-0 最新版本为1.23.0-0 

   ```
   # yum list --showduplicates kubeadm
   ```

2. 升级到1.23.0-0 

   ```
   # yum install -y kubeadm-1.23.0-0
   ```

3. 驱逐当前需要升级的节点的pod 只是业务pod 忽略daemonsets

   ```
   [root@k8s01 ~]# kubectl drain k8s01 --ignore-daemonsets
   ```

4. 检查集群是否可升级

   ```
   # kubeadm upgrade plan
   [upgrade/config] Making sure the configuration is correct:
   [upgrade/config] Reading configuration from the cluster...
   [upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
   [preflight] Running pre-flight checks.
   [upgrade] Running cluster health checks
   [upgrade] Fetching available versions to upgrade to
   [upgrade/versions] Cluster version: v1.22.4
   [upgrade/versions] kubeadm version: v1.23.0
   [upgrade/versions] Target version: v1.23.6
   [upgrade/versions] Latest version in the v1.22 series: v1.22.9
   
   Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
   COMPONENT   CURRENT       TARGET
   kubelet     3 x v1.22.4   v1.22.9
   
   Upgrade to the latest version in the v1.22 series:
   
   COMPONENT                 CURRENT   TARGET
   kube-apiserver            v1.22.4   v1.22.9
   kube-controller-manager   v1.22.4   v1.22.9
   kube-scheduler            v1.22.4   v1.22.9
   kube-proxy                v1.22.4   v1.22.9
   CoreDNS                   v1.8.4    v1.8.6
   etcd                      3.5.0-0   3.5.1-0
   
   You can now apply the upgrade by executing the following command:
   
           kubeadm upgrade apply v1.22.9
   
   _____________________________________________________________________
   
   Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
   COMPONENT   CURRENT       TARGET
   kubelet     3 x v1.22.4   v1.23.6
   
   Upgrade to the latest stable version:
   
   COMPONENT                 CURRENT   TARGET
   kube-apiserver            v1.22.4   v1.23.6
   kube-controller-manager   v1.22.4   v1.23.6
   kube-scheduler            v1.22.4   v1.23.6
   kube-proxy                v1.22.4   v1.23.6
   CoreDNS                   v1.8.4    v1.8.6
   etcd                      3.5.0-0   3.5.1-0
   
   You can now apply the upgrade by executing the following command:
   
           kubeadm upgrade apply v1.23.6
   
   Note: Before you can perform this upgrade, you have to update kubeadm to v1.23.6.
   
   _____________________________________________________________________
   
   
   The table below shows the current state of component configs as understood by this version of kubeadm.
   Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
   resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
   upgrade to is denoted in the "PREFERRED VERSION" column.
   
   API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
   kubeproxy.config.k8s.io   v1alpha1          v1alpha1            no
   kubelet.config.k8s.io     v1beta1           v1beta1             no
   _____________________________________________________________________
   
   ```

5. 根据提示升级1.23.6

   ```
   # kubeadm upgrade apply v1.23.6
   ```

6. 升级kubelet和kubectl

   ```
   # yum install -y kubelet-1.23.6 kubectl-1.23.6
   ```

7. 重启kubelet

   ```
   # systemctl daemon-reload && systemctl restart kubelet
   ```

8. 取消不可调度污点

   ```
   # kubectl uncordon k8s01
   ```

9. 查看 当前节点已经升级完成

   ```
   # kubectl get nodes
   NAME    STATUS   ROLES                  AGE    VERSION
   k8s01   Ready    control-plane,master   138d   v1.23.6
   k8s02   Ready    control-plane,master   138d   v1.22.4
   k8s03   Ready    control-plane,master   138d   v1.22.4
   ```

10. 其他节点上升级(k8s02)

    ```
    1.升级kubeadm
    # yum install -y kubeadm-1.23.0-0
    
    2.驱逐node上的pod，且不可调度 master节点执行
    # kubectl drain k8s02 --ignore-daemonsets
    
    3.升级kubelet配置 需要升级的节点执行 也就是k8s02
    # kubeadm upgrade node
    
    4.升级kubelet和kubectl
    # yum install -y kubelet-1.23.6 kubectl-1.23.6
    
    5.重启kubelet
    # systemctl daemon-reload && systemctl restart kubelet
    
    6.取消不可调度，节点重新上线
    # kubectl uncordon k8s02
    
    7. 查看节点状态
    # kubectl get nodes
    NAME    STATUS   ROLES                  AGE    VERSION
    k8s01   Ready    control-plane,master   138d   v1.23.6
    k8s02   Ready    control-plane,master   138d   v1.23.6
    k8s03   Ready    control-plane,master   138d   v1.22.4
    ```

11. 升级第三个节点(k8s03)

    ```
    # 升级完成后查看 三个节点均已正常
    # kubectl get nodes
    NAME    STATUS   ROLES                  AGE    VERSION
    k8s01   Ready    control-plane,master   138d   v1.23.6
    k8s02   Ready    control-plane,master   138d   v1.23.6
    k8s03   Ready    control-plane,master   138d   v1.23.6
    ```

## 二进制升级

基于上述升级基本流程，大致流程如下：

1. 下载二进制包：https://github.com/kubernetes/kubernetes
2. 替换对应组件二进制文件
3. 重启服务



# 证书

## kubeadm续签

查看证书有效期

```
# kubeadm certs check-expiration

# kubectl -n kube-system get cm kubeadm-config -o yaml
# openssl x509 -in server-apiserver.csr -noout -dates
```

1. 查备份之前的证书和环境

   ```
   # cp -r /etc/kubernetes/ /etc/kubernetes.bak
   # cp -r /var/lib/etcd/ /var/lib/etcd.bak
   ```

2. 续签

   ```
   # kubeadm certs renew all 
   ```

3. 重启 kube-apiserver, kube-controller-manager, kube-scheduler and etcd

   kubeadm是容器方式部署的。直接把这几个pod删掉就可以

   ```
   # kubectl delete pod etcd-k8s01 etcd-k8s02 etcd-k8s03 kube-apiserver-k8s01 kube-apiserver-k8s02 kube-apiserver-k8s03 kube-controller-manager-k8s01 kube-controller-manager-k8s02 kube-controller-manager-k8s03 kube-scheduler-k8s01 kube-scheduler-k8s02 kube-scheduler-k8s03 -n kube-system
   ```

4. 查看证书是否已经生效

   ```
   # echo | openssl s_client -showcerts -connect 127.0.0.1:6443 -servername api 2>/dev/null | openssl x509 -noout -enddate
   ```

5. 使用新的配置文件

   ```
   # cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
   ```

## 二进制

直接按照安装流程将证书重新生成续签就可以了



# 扩容

手动创建比当前所有资源大的应用资源，引起整个集群资源不足

## 缩容

自动减少Node：
如果你想从Kubernetes集群中删除节点，正确流程如下：
1、获取节点列表
kubectl get node
2、设置不可调度
kubectl cordon <node_name>
3、驱逐节点上的Pod
kubectl drain <node_name> --ignore-daemonsets
4、移除节点
kubectl delete node <node_name>

# HPA

HPA扩容

HPA需要获取阈值 阈值从metric-server获取，metric-server又要从kubelet(cadvisor)获取

metric-server和HPA对接是采用 由于metric-server是第三方服务 需要和HPA对接 也就是和apiserver对接然后apiserver再次代理metric-server

## HPA前提条件

需要在kube-apiserver.conf配置文件添加一下几个参数

```
# vim /opt/kubernetes/cfg/kube-apiserver.conf
...
--requestheader-client-ca-file=/opt/kubernetes/ssl/ca.pem \
--proxy-client-cert-file=/opt/kubernetes/ssl/server.pem \
--proxy-client-key-file=/opt/kubernetes/ssl/server-key.pem \
--requestheader-allowed-names=kubernetes \
--requestheader-extra-headers-prefix=X-Remote-Extra-\
--requestheader-group-headers=X-Remote-Group \
--requestheader-username-headers=X-Remote-User \
--enable-aggregator-routing=true \
...
```

配置后好重启

## 部署Metrics Server

项目地址：https://github.com/kubernetes-sigs/metrics-server

1. 下载Metrics Server

   ```
   # wget https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.7/components.yaml
   ```

2. 修改配置文件，添加---kubelet-insecure-tls参数 忽略证书验证

   ```
   vi components.yaml
   ...
   containers:
   -args:
   ---cert-dir=/tmp
   ---secure-port=4443
   ---kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
   ---kubelet-use-node-status-port
   ---kubelet-insecure-tls
   ...
   ```

3. 执行文件

   ```
   # kubectl apply -f metrics-server.yaml 
   
   # 运行成功
   # kubectl get pods -n kube-system
   NAME                                     READY   STATUS    RESTARTS       AGE
   calico-kube-controllers-8db96c76-q2hxz   1/1     Running   2 (26h ago)    5d19h
   calico-node-82xgz                        1/1     Running   14 (47h ago)   5d18h
   calico-node-8dqvq                        1/1     Running   1 (47h ago)    5d17h
   calico-node-9n9br                        1/1     Running   0              17h
   calico-node-dxrth                        1/1     Running   31 (47h ago)   5d19h
   calico-node-m2l9t                        1/1     Running   14 (47h ago)   5d18h
   coredns-5c5b78b748-vhttd                 1/1     Running   2 (26h ago)    5d18h
   metrics-server-864cd9cfb-nzx7m           1/1     Running   0              2m34s
   ```

4. 查看已经注册上来

   ```
   # kubectl get apiservice| grep metrics
   v1beta1.metrics.k8s.io                 kube-system/metrics-server   True        67s
   
   # 可以看到metrics地址 这个是apiserver代理metrics 所以下面地址是metrics直接返回的
   # kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes
   # kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods
   
   # 也可以查看pod的top命令 查看资源消耗 此命令依赖于metrics
   # kubectl top node
   # kubectl top pod
   ```

5. 测试

   1. 给予cpu资源做资源伸缩

      增加resources.requests.cpu 因为是基于cpu扩展 所以要修改这个值 

      ```
      # kubectl create deployment nginx-hpa --image=nginx --dry-run=client -o yaml > deployment-hpa.yaml
      
      # 修改yaml，增加resources.requests.cpu 因为是基于cpu扩展 所以要修改这个值
      # 创建deployment 默认1核1副本
      # vim deploymeny-hpa.yaml
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        creationTimestamp: null
        labels:
          app: nginx-hpa
        name: nginx-hpa
      spec:
        replicas: 1
        selector:
          matchLabels:
            app: nginx-hpa
        strategy: {}
        template:
          metadata:
            creationTimestamp: null
            labels:
              app: nginx-hpa
          spec:
            containers:
            - image: nginx
              name: nginx
              resources: 
                requests:
                   cpu: 1
      
      # 将端口共享出去
      # kubectl expose deployment nginx-hpa --port=80 --target-port=80 --dry-run=client -o yaml > service-hpa.yaml
      
      # vim service-hpa.yaml 
      apiVersion: v1
      kind: Service
      metadata:
        annotations: # 加入prometheus支持 但是需要提供指标接口 不然不通
          prometheus.io/port: "9153"
          prometheus.io/scrape: "true"
        creationTimestamp: null
        labels:
          app: nginx-hpa
        name: nginx-hpa
      spec:
        ports:
        - port: 18080  # 暴露的端口
          protocol: TCP
          targetPort: 80 # 内部端口
        selector:
          app: nginx-hpa
      status:
        loadBalancer: {}
        
      # kubectl apply -f service-hpa.yaml 
      service/nginx-hpa created
      
      # kubectl get pods
      ```
   
   
   
   ## hpa扩容
   
   1. 创建hpa
   
      --min 最小pod
   
      --max 最大pod
   
      -cpu-percent cpu阈值 默认80% 基于百分比 这里给5% 这里的计算是所有的pod的cpu利用率的评价值 如果超过5就扩容
   
      ```
      # kubectl autoscale deployment web --min=2 --max=10 --cpu-percent=5
      # 导出yaml
      # kubectl autoscale deployment nginx-hpa --min=2 --max=10 --cpu-percent=5 --dry-run=client -o yaml > nginx-hpa-v1.yaml
      
      # vim nginx-hpa-v1.yaml 
      status:
        currentReplicas: 0
      apiVersion: autoscaling/v1
      kind: HorizontalPodAutoscaler
      metadata:
        creationTimestamp: null
        name: nginx-hpa
      spec:
        maxReplicas: 10  # 最大pod
        minReplicas: 2  # 最小pod
        scaleTargetRef:
          apiVersion: apps/v1
          kind: Deployment
          name: nginx-hpa  #deployment名字
        targetCPUUtilizationPercentage: 5  #资源阈值 5%
        
      # 执行
      # kubectl apply -f nginx-hpa-v1.yaml 
      
      # 过一会 可以看到当前的负载和设置的阈值
      # kubectl get hpa
      NAME        REFERENCE              TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
      nginx-hpa   Deployment/nginx-hpa   0%/5%     2         10        2          34s
      
      # 已经把第二pod创建了 因为默认最小两个pod
      # kubectl get pods
      NAME                         READY   STATUS    RESTARTS   AGE
      nginx-hpa-588d4dccb6-568ns   1/1     Running   0          86s
      nginx-hpa-588d4dccb6-z4zzh   1/1     Running   0          7m18s
      nginxtest-fc5f8b88-md9nb     1/1     Running   0          26h
      ```
   
   2. 压测
   
      ```
      # kubectl get svc
      NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)     AGE
      kubernetes   ClusterIP   10.0.0.1     <none>        443/TCP     6d17h
      nginx-hpa    ClusterIP   10.0.0.75    <none>        18080/TCP   2m53s
      
      # yum install -y httpd-tools
      # ab -n 300000 -c 2000 http://10.0.0.75:18080/index.html # 总30w请求，并发2000
      
      # 监听 hpa和pod的扩容变化 可以看到 hba负载和pod正在扩容
      # kubectl get hpa -w
      NAME        REFERENCE              TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
      nginx-hpa   Deployment/nginx-hpa   0%/5%     2         10        2          5m40s
      nginx-hpa   Deployment/nginx-hpa   0%/5%     2         10        2          6m15s
      nginx-hpa   Deployment/nginx-hpa   43%/5%    2         10        2          6m30s
      
      # kubectl get pods -w
      NAME                         READY   STATUS    RESTARTS   AGE
      nginx-hpa-588d4dccb6-568ns   1/1     Running   0          5m17s
      nginx-hpa-588d4dccb6-z4zzh   1/1     Running   0          11m
      nginxtest-fc5f8b88-md9nb     1/1     Running   0          26h
      nginx-hpa-588d4dccb6-25ghk   0/1     Pending   0          0s
      nginx-hpa-588d4dccb6-k68j5   0/1     Pending   0          0s
      nginx-hpa-588d4dccb6-25ghk   0/1     Pending   0          0s
      nginx-hpa-588d4dccb6-k68j5   0/1     Pending   0          0s
      nginx-hpa-588d4dccb6-25ghk   0/1     ContainerCreating   0          0s
      nginx-hpa-588d4dccb6-k68j5   0/1     ContainerCreating   0          0s
      nginx-hpa-588d4dccb6-25ghk   0/1     ContainerCreating   0          2s
      nginx-hpa-588d4dccb6-k68j5   0/1     ContainerCreating   0          2s
      
      # kubectl get pod
      NAME                         READY   STATUS    RESTARTS   AGE
      nginx-hpa-588d4dccb6-25ghk   1/1     Running   0          78s
      nginx-hpa-588d4dccb6-568ns   1/1     Running   0          7m33s
      nginx-hpa-588d4dccb6-5rnh2   1/1     Running   0          63s
      nginx-hpa-588d4dccb6-bkbdf   1/1     Running   0          48s
      nginx-hpa-588d4dccb6-d24ps   1/1     Running   0          63s
      nginx-hpa-588d4dccb6-h874d   1/1     Running   0          48s
      nginx-hpa-588d4dccb6-k68j5   1/1     Running   0          78s
      nginx-hpa-588d4dccb6-rz7mh   1/1     Running   0          63s
      nginx-hpa-588d4dccb6-z4zzh   1/1     Running   0          13m
      nginx-hpa-588d4dccb6-zczp2   1/1     Running   0          63s
      nginxtest-fc5f8b88-md9nb     1/1     Running   0          26h
      ```
   

## hpa缩容

在弹性伸缩中，程序不可能一直扩容和缩容 需要有一个冷却周期 不然副本的数量可能会不断波动，造成丢失流量，所以不应该在任意时间扩容和缩容。

在HPA 中，通过两个参数控制扩容和缩容：
---horizontal-pod-autoscaler-downscale-delay ：当前操作完成后等待多次时间才能执行缩容操作，默认5分钟
---horizontal-pod-autoscaler-upscale-delay ：当前操作完成后等待多长时间才能执行扩容操作，默认3分钟
通过调整kube-controller-manager组件启动参数调整 修改 /opt/kubernetes/cfg/kube-controller-manager.conf 这个文件



## hpa自定义扩容缩容指标

HPA也支持自定义指标，例如QPS、5xx错误状态码等，自定义指标的实现由autoscaling/v2版本提供，而v2版本又分为beta1和beta2两个版本。
这两个版本的区别是autoscaling/v1beta1支持了：
•Resource Metrics（资源指标 比如 cpu内存）
•Custom Metrics（自定义指标）
而在autoscaling/v2beta2的版本中额外增加了External Metrics（扩展指标）的支持。

自定义指标

是通过k8s的prometheus一个适配器采集数据，转换成k8s api标准获取指标 实现扩容和缩容

自定义指标（例如QPS），将使用custom.metrics.k8s.io API，由相关适配器（Adapter）服务提供。
已知适配器列表：https://github.com/kubernetes/metrics/blob/master/IMPLEMENTATIONS.md#custom-metrics-api

1. 部署prometheus

   prometheus-deployment.yaml # 部署Prometheus
   prometheus-configmap.yaml # Prometheus配置文件，主要配置基于Kubernetes服务发现
   prometheus-rules.yaml # Prometheus告警规则

   ```
   # cd /opt/yaml/prometheus
   # kubectl apply -f prometheus-deployment.yaml 
   # kubectl apply -f prometheus-configmap.yaml 
   # kubectl apply -f prometheus-rules.yaml 
   
   # kubectl get svc -A
   查看端口 通过30090 
   ```

2. 部署应用测试

   应用需要实现对Prometheus采集指标的服务暴露

   对应用暴露指标，部署应用，并让Prometheus采集暴露的指标。
   在做这步之前先了解下Prometheus如何监控应用的。
   如果要想监控，前提是能获取被监控端指标数据，并且这个数据格式必须遵循Prometheus数据模型，这样才能识别和采集，一般使用exporter提供监控指标数据。但对于自己开发的项目，是需要自己实现类似于exporter的指标采集程序。
   exporter列表：https://prometheus.io/docs/instrumenting/exporters

   ```
   # kubectl apply -f metrics-flask-app.yaml 
   
   # 部署完成后 http://192.168.10.170:30090/targets就有了指标
   # curl 10.0.0.114:80/metrics
   ```

3. hpa集成

   就是将Prometheus的数据做一个Prometheus适配器 转换成hpa可以识别的格式进行数据采集来实现扩容和缩容

4. 部署Prometheus Adapter
   ```
   # kubectl apply -f prometheus-adapter.yaml
   
   #验证是否成功
   # kubectl get apiservices |grep custom
   # 这个接口才是hpa要使用的
   # kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1"
   ```

5. 为指定HPA配置Prometheus Adapter

   其实就是为每个不通的应用的采集数据实现Prometheus 和HPA的转换规则

   例如 hpa请求到Adapter然后根据规则去查询Prometheus 

   • seriesQuery：Prometheus查询语句，查询应用系列指标。
   • resources：Kubernetes资源标签映射到Prometheus标签。
   • name：将Prometheus指标名称在自定义指标API中重命名，matches正则匹配，as指定新名称。
   • metricsQuery：一个Go模板，对调用自定义指标API转换为Prometheus查询语句。

   ```
   # 创建adapter
   # vim prometheus-adapter-configmap.yaml 
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: prometheus-adapter
     labels:
       app: prometheus-adapter
       chart: prometheus-adapter-2.5.1
       release: prometheus-adapter
       heritage: Helm
     namespace: kube-system
   data:
     config.yaml: |
       rules:
       - seriesQuery: 'request_count_total{app="flask-app"}'
         resources:
           overrides:
             kubernetes_namespace: {resource: "namespace"}
             kubernetes_pod_name: {resource: "pod"}
         name:
           matches: "request_count_total"
           as: "qps"
         metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'
   ```

6. 执行config配置并且重建pod 使pod生效

   ```
   # kubectl apply -f prometheus-adapter-configmap.yaml 
   # kubectl delete pod prometheus-adapter-bbcf477c6-vzv5f  -n kube-system
   ```

   Adapter向Prometheus查询语句最终是：

   ```
   # sum(rate(request_count_total{app="flask-app", kubernetes_namespace="default", kubernetes_pod_name=~"pod1|pod2"}[2m])) by (kubernetes_pod_name)
   
   # sum(rate(request_count_total{app="flask-app", kubernetes_namespace="default"}[2m])) by (kubernetes_pod_name)
   ```

   由于HTTP请求统计是累计的，对HPA自动缩放不是特别有用，因此将其转为速率指标。
   这条语句意思是：查询每个Pod在2分钟内访问速率，即QPS（每秒查询率）
   向自定义指标API访问：
   kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/qps"
   如果配置没问题，会返回JSON数据，注意里面的value字段，HPA控制器会拿这个值计算然后比对阈值。这个值单位是m，表示毫秒，千分之一，例如值为500m是每秒0.5个请求，10000m是每秒10个请求（并发）。

7. 创建HPA

   基于V1版本导出V2 因为本地之前有了

   ```
   # kubectl get hpa.v2beta2.autoscaling -o yaml > hpa-v2.yzml
   
   # vim hpa-v2-qps.yaml 
   apiVersion: autoscaling/v2beta2
   kind: HorizontalPodAutoscaler
   metadata:
     name: metrics-flask-app
     namespace: default
   spec:
     minReplicas: 2
     maxReplicas: 10
     scaleTargetRef:
       apiVersion: apps/v1
       kind: Deployment
       name: metrics-flask-app
     metrics:
     - type: Pods
       pods:
         metric:
           name: qps
         target:
           type: AverageValue  #类型 求一个平均值
           averageValue: 10000m # 所有Pod平均值为10000m触发扩容，即每秒10个请求
           
   # kubectl apply -f hpa-v2-qps.yaml 
   ```

8. 压测

   ```
   # kubectl get svc
   NAME                TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)     AGE
   kubernetes          ClusterIP   10.0.0.1     <none>        443/TCP     6d22h
   metrics-flask-app   ClusterIP   10.0.0.114   <none>        80/TCP      3h52m
   nginx-hpa           ClusterIP   10.0.0.75    <none>        18080/TCP   4h52m
   
   # ab -n 200000 -c 1000 http://10.0.0.114:80/index.html
   
   # 可以看到请求已经满足阈值 自动创建了扩容
   # kubectl get hpa -w
   NAME                REFERENCE                      TARGETS      MINPODS   MAXPODS   REPLICAS   AGE
   metrics-flask-app   Deployment/metrics-flask-app   121699m/10   1         10        6          5m46s
   nginx-hpa           Deployment/nginx-hpa           0%/5%        2         10        2          4h58m
   metrics-flask-app   Deployment/metrics-flask-app   121699m/10   1         10        10         6m
   metrics-flask-app   Deployment/metrics-flask-app   16m/10       1         10        10         6m15s
   # kubectl get pods -w
   NAME                                 READY   STATUS    RESTARTS   AGE
   metrics-flask-app-66c9995b58-8mtxp   1/1     Running   0          3h52m
   metrics-flask-app-66c9995b58-ngtm4   1/1     Running   0          3h52m
   metrics-flask-app-66c9995b58-wd7zb   1/1     Running   0          3h52m
   nginx-hpa-588d4dccb6-568ns           1/1     Running   0          4h53m
   nginx-hpa-588d4dccb6-z4zzh           1/1     Running   0          4h58m
   nginxtest-fc5f8b88-md9nb             1/1     Running   0          31h
   metrics-flask-app-66c9995b58-sz9dd   0/1     Pending   0          0s
   metrics-flask-app-66c9995b58-nrg2b   0/1     Pending   0          0s
   metrics-flask-app-66c9995b58-2f4k7   0/1     Pending   0          0s
   metrics-flask-app-66c9995b58-sz9dd   0/1     Pending   0          0s
   metrics-flask-app-66c9995b58-nrg2b   0/1     Pending   0          0s
   metrics-flask-app-66c9995b58-2f4k7   0/1     Pending   0          0s
   metrics-flask-app-66c9995b58-nrg2b   0/1     ContainerCreating   0          0s
   metrics-flask-app-66c9995b58-sz9dd   0/1     ContainerCreating   0          0s
   metrics-flask-app-66c9995b58-2f4k7   0/1     ContainerCreating   0          0s
   metrics-flask-app-66c9995b58-sz9dd   0/1     ContainerCreating   0          1s
   metrics-flask-app-66c9995b58-nrg2b   0/1     ContainerCreating   0          1s
   metrics-flask-app-66c9995b58-2f4k7   0/1     ContainerCreating   0          1s
   metrics-flask-app-66c9995b58-nrg2b   1/1     Running             0          3s
   metrics-flask-app-66c9995b58-vzcpf   0/1     Pending             0          0s
   metrics-flask-app-66c9995b58-vzcpf   0/1     Pending             0          0s
   metrics-flask-app-66c9995b58-zfftc   0/1     Pending             0          0s
   metrics-flask-app-66c9995b58-zfftc   0/1     Pending             0          0s
   metrics-flask-app-66c9995b58-qbrth   0/1     Pending             0          0s
   metrics-flask-app-66c9995b58-qbrth   0/1     Pending             0          0s
   metrics-flask-app-66c9995b58-vzcpf   0/1     ContainerCreating   0          0s
   metrics-flask-app-66c9995b58-q5gjs   0/1     Pending             0          0s
   metrics-flask-app-66c9995b58-zfftc   0/1     ContainerCreating   0          0s
   metrics-flask-app-66c9995b58-q5gjs   0/1     Pending             0          0s
   metrics-flask-app-66c9995b58-qbrth   0/1     ContainerCreating   0          0s
   metrics-flask-app-66c9995b58-q5gjs   0/1     ContainerCreating   0          0s
   metrics-flask-app-66c9995b58-q5gjs   0/1     ContainerCreating   0          1s
   metrics-flask-app-66c9995b58-qbrth   0/1     ContainerCreating   0          1s
   metrics-flask-app-66c9995b58-zfftc   0/1     ContainerCreating   0          1s
   metrics-flask-app-66c9995b58-vzcpf   0/1     ContainerCreating   0          1s
   metrics-flask-app-66c9995b58-qbrth   1/1     Running             0          2s
   metrics-flask-app-66c9995b58-sz9dd   1/1     Running             0          17s
   metrics-flask-app-66c9995b58-2f4k7   1/1     Running             0          18s
   metrics-flask-app-66c9995b58-vzcpf   1/1     Running             0          18s
   metrics-flask-app-66c9995b58-q5gjs   1/1     Running             0          18s
   metrics-flask-app-66c9995b58-zfftc   1/1     Running             0          20s
   ```


# Flannel

Flannel是CoreOS维护的一个网络组件，Flannel为每个Pod提供全局唯一的IP，Flannel使用ETCD来存储Pod子网与Node IP之间的关系。flanneld守护进程在每台主机上运行，并负责维护ETCD信息和路由数据包。
项目地址：https://github.com/coreos/flannel
YAML地址：https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

切换网络组建的时候 pod需要重建

## 先卸载calico

```
# pwd
/opt/soft
# kubectl delete -f calico.yaml 

# 清理calico配置文件
# down calico的虚拟网卡 以来于ipip的模块
# ip link set tunl0 down
# 清理cni配置文件
# rm -f /etc/cni/net.d/*
# 删掉nodename
# rm -f /var/lib/calico/nodename 
# 清空路由表

```

## 部署Flannel

Network：指定Pod IP分配的网段，与controller-manager配置的保持一样。
--allocate-node-cidrs=true

--cluster-cidr=10.244.0.0/16

* kubeadm部署：/etc/kubernetes/manifests/kube-controller-manager.yaml
* 二进制部署：/opt/kubernetes/cfg/kube-controller-manager.conf
* Backend：指定工作模式

```
# 修改网段
……
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
……

# cat kube-flannel.yml 
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
rules:
- apiGroups: ['extensions']
  resources: ['podsecuritypolicies']
  verbs: ['use']
  resourceNames: ['psp.flannel.unprivileged']
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes/status
  verbs:
  - patch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flannel
  namespace: kube-system
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
      hostNetwork: true
      priorityClassName: system-node-critical
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image: lizhenliang/flannel:v0.14.0
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: lizhenliang/flannel:v0.14.0
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
            add: ["NET_ADMIN", "NET_RAW"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
      - name: run
        hostPath:
          path: /run/flannel
      - name: cni
        hostPath:
          path: /etc/cni/net.d
      - name: flannel-cfg
        configMap:
          name: kube-flannel-cfg
```

部署

```
# kubectl apply -f kube-flannel.yml 
# kubectl get pods -n kube-system
NAME                                 READY   STATUS    RESTARTS      AGE
coredns-5c5b78b748-vhttd             1/1     Running   3 (12m ago)   10d
kube-flannel-ds-4c9b9                1/1     Running   0             25s
kube-flannel-ds-6cc8l                1/1     Running   0             25s
kube-flannel-ds-7ktv4                1/1     Running   0             25s
kube-flannel-ds-928wg                1/1     Running   0             25s
kube-flannel-ds-gtjdd                1/1     Running   0             25s
metrics-server-864cd9cfb-nzx7m       1/1     Running   1 (12m ago)   5d1h
prometheus-8d4b7b96d-4295b           2/2     Running   2 (12m ago)   5d
prometheus-adapter-bbcf477c6-6x459   1/1     Running   1 (12m ago)   4d20h
```



## Flannel支持多种工作模式

* UDP：最早支持的一种方式，由于性能最差，目前已经弃用。

* VXLAN：Overlay Network方案，源数据包封装在另一种网络包里面进行路由转发和通信

  VXLAN，即Virtual Extensible LAN（虚拟可扩展局域网），是Linux 内核本身就支持的一种网络虚似化技术。VXLAN 可以完全在内核态实现上述封装和解封装的工作，从而通过与前面相似的“隧道”机制，构建出覆盖网络（Overlay Network）。
  VXLAN的覆盖网络设计思想：在现有的三层网络之上，覆盖一层二层网络，使得连接在这个VXLAN二层网络上的主机之间，可以像在同一局域网里通信。
  为了能够在二层网络上打通“隧道”，VXLAN 会在宿主机上设置一个特殊的网络设备作为“隧道”的两端。这个设备就叫作VTEP，即：VXLAN Tunnel End Point（虚拟隧道端点）。

  VTEP设备进行封装和解封装的对象是二层数据帧，这个工作是在Linux内核中完成的。

  是针对二层数据包再次封包 以来二层数据包 相当于将数据打包在宿主机的二层上进行传输 会新启动一个网桥 桥接到宿主机网络上 根据bridge-utils查看

  ```
  # yum -y install bridge-utils
  ```

  加入node1和node2 各自有个pod1和pod2 这两个pod传输

  如果Pod 1访问Pod 2，源地址10.244.1.10，目的地址10.244.2.10 ，数据包传输流程如下：

  1. 从container到cni0（容器路由）：容器根据路由表，将数据包发送下一跳10.244.0.1，从eth0网卡出。可以使用ip route命令查看路由表
  2. 从cni0到flannel.1（主机路由）：数据包进入到宿主机虚拟网卡cni0，根据路由表转发到flannel.1虚拟网卡。
  3. flannel.1封装内部数据帧：flannel.1收到IP包后，由于flannel.1工作在二层网络，二层网络又需要目的MAC地址，因此要想封装完整的二层数据帧，这个目标容器宿主机上flannel.1的MAC地址从哪获取到呢？其实在flanneld进程启动后，就会自动添加其他节点ARP记录，可以通过ip neigh show dev flannel.1命令查看。
  4. flannel.1二次封装：知道了目的MAC地址，Linux内核就可以进行二层封包了。但是，对于宿主机网络来说这个二层帧并不能在宿主机二层网络里传输。所以接下来，还要把这个数据帧进一步封装成为宿主机网络的一个普通数据帧，好让它载着内部数据帧，通过宿主机的eth0网卡进行传输。
  5. flannel.1发起UDP连接：flannel.1准备向目标容器宿主机flannel.1发送UDP连接，但这时还不知道目标宿主机是谁，也就是说这个UDP包该发给哪台宿主机呢？
  flanneld进程也维护着一个叫做FDB的转发数据库，可以通过bridge fdb show dev flannel.1命令查看，里面记录了目标容器宿主机flannel.1的MAC对应的宿主机IP，也就是UDP要发往的目的地。接下来就是一个正常的宿主机于宿主机之间的传输了。
  6. 数据包到达目的宿主机：数据包从Node1的eth0网卡发出去，Node2接收到数据包，解封装发现是VXLAN数据包，把它交给flannel.1设备。flannel.1设备则会进一步拆包，取出原始IP包（源容器IP和目标容器IP），通过cni0网桥二层转发给容器。

* Host-GW：Flannel通过在各个节点上的Agent进程，将容器网络的路由信息写到主机的路由表上，这样一来所有的主机都有整个容器网络的路由数据了。

* Directrouting：同时支持VXLAN和Host-GW工作模式

* 公有云VPC：aliyun，AWS

## HOST-GW模式

host-gw模式相比vxlan简单了许多，直接添加路由，将目的主机当做网关，直接路由原始封包。

修改配置文件：

```
# kube-flannel.yml
……
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "host-gw"
      }
    }
……

```

当你设置flannel使用host-gw模式，flanneld会在宿主机上创建节点的路由表

目的IP 地址属于10.244.1.0/24 网段的IP 包，应该经过本机的eth0 设备发出去（即：dev eth0）；并且，它下一跳地址是192.168.31.63（即：via 192.168.31.63）。

一旦配置了下一跳地址，那么接下来，当IP 包从网络层进入链路层封装成帧的时候，eth0 设备就会使用下一跳地址对应的MAC 地址，作为该数据帧的目的MAC 地址。

而Node 2 的内核网络栈从二层数据帧里拿到IP 包后，会“看到”这个IP 包的目的IP 地址是10.244.1.20，即container-2 的IP 地址。这时候，根据Node 2 上的路由表，该目的地址会匹配到第二条路由规则（也就是10.244.1.0 对应的路由规则），从而进入cni0 网桥，进而进入到container-2 当中。

可见，数据包是封装成帧发送出去的，会使用路由表的下一跳来设置目的MAC地址，会经过二层网络到达目的节点，因此，host-gw模式必须要求集群宿主机之间二层连通。



VXLAN特点：

* 先进行二层帧封装，再通过宿主机网络封装，解封装也一样，所以增加性能开销
* 对宿主机网络要求低，只要三层网络可达

Host-GW特点：

* 直接路由转发，性能损失很小
* 对宿主机网络要求二层可达



# Calico

Calico是一个纯三层的数据中心网络方案，Calico支持广泛的平台，包括Kubernetes、OpenStack等。

Calico 在每一个计算节点利用Linux Kernel 实现了一个高效的虚拟路由器（vRouter）来负责数据转发，而每个vRouter 通过BGP 协议负责把自己上运行的workload 的路由信息向整个Calico 网络内传播。
此外，Calico 项目还实现了Kubernetes 网络策略，提供ACL功能。

实际上，Calico项目提供的网络解决方案，与Flannel的host-gw模式几乎一样。也就是说，Calico也是基于路由表实现容器数据包转发，但不同于Flannel使用flanneld进程来维护路由信息的做法，而Calico项目使用BGP协议来自动维护整个集群的路由信息。

BGP英文全称是Border Gateway Protocol，即边界网关协议，它是一种自治系统间的动态路由发现协议，与其他BGP 系统交换网络可达信息。

在这个图中，有两个自治系统（autonomous system，简称为AS）：AS 1 和AS 2。
在互联网中，一个自治系统(AS)是一个有权自主地决定在本系统中应采用何种路由协议的小型单位。这个网络单位可以是一个简单的网络也可以是一个由一个或多个普通的网络管理员来控制的网络群体，它是一个单独的可管理的网络单元（例如一所大学，一个企业或者一个公司个体）。一个自治系统有时也被称为是一个路由选择域（routing domain）。一个自治系统将会分配一个全局的唯一的16位号码，有时我们把这个号码叫做自治系统号（ASN）。
在正常情况下，自治系统之间不会有任何来往。如果两个自治系统里的主机，要通过IP 地址直接进行通信，我们就必须使用路由器把这两个自治系统连接起来。BGP协议就是让他们互联的一种方式。

在了解了BGP 之后，Calico 项目的架构就非常容易理解了，Calico主要由三个部分组成：

* Felix：以DaemonSet方式部署，运行在每一个Node节点上，主要负责维护宿主机上路由规则以及ACL规则。
* BGP Client（BIRD）：主要负责把Felix 写入Kernel 的路由信息分发到集群Calico 网络。
* Etcd：分布式键值存储，保存Calico的策略和网络配置状态。
* calicoctl：命令行管理Calico。



Calico存储有两种方式：

* 数据存储在etcd
  https://docs.projectcalico.org/v3.9/manifests/calico-etcd.yaml
* 数据存储在Kubernetes API Datastore服务中
  https://docs.projectcalico.org/manifests/calico.yaml

数据存储在etcd中还需要修改yaml：

* 配置连接etcd地址，如果使用https，还需要配置证书。（ConfigMap和Secret位置）
* 根据实际网络规划修改Pod CIDR（CALICOIPV4POOLCIDR）

## 部署：

删除Flannel

删除flannel和一流的网桥 还有配置文件 还有路由表

```
# kubectl delete -f kube-flannel.yml 
# ip link delete cni0
# ip link delete flannel.1
# rm -f /etc/cni/net.d/*
# 删除所有Flannel路由
# ip route
# ip route del 10.244.0.0/24 via 10.244.0.0 dev flannel.1 onlink
# ip route del 10.244.2.0/24 via 10.244.2.0 dev flannel.1 onlink
# ip route del 10.244.3.0/24 via 10.244.3.0 dev flannel.1 onlink 
# ip route del 10.244.4.0/24 via 10.244.4.0 dev flannel.1 onlink
# ip route
```

部署

```
# kubectl apply -f calico.yaml 
# kubectl get pods -n kube-system
```

## 管理工具

calicoctl工具用于管理calico，可通过命令行读取、创建、更新和删除Calico 的存储对象。
项目地址：https://github.com/projectcalico/calicoctl
calicoctl 在使用过程中，需要从配置文件中读取Calico 对象存储地址等信息。默认配置文件路径/etc/calico/calicoctl.cfg

* 方式1

  链接k8s api方式

  ```
  # 创建配置文件
  # vim /etc/calio/calicoctl.cfg
  apiVersion: projectcalico.org/v3
  kind: CalicoAPIConfig
  metadata:
  spec:
   datastoreType: "kubernetes"
   kubeconfig: "/root/.kube/config"  #k8s配置文件
  ```

* 方式2

  链接etcd

  ```
  # vim /etc/calio/calicoctl.cfg
  apiVersion: projectcalico.org/v3
  kind: CalicoAPIConfig
  metadata:
  spec:
   datastoreType: "etcdv3"
   etcdEndpoints: "https://192.168.10.170:2379,https://192.168.10.171:2379,https://192.168.10.172:2379"
   etcdKeyFile: "/opt/etcd/ssl/server-key.pem"
   etcdCertFile: "/opt/etcd/ssl/server.pem"
   etcdCACertFile: "/opt/etcd/ssl/ca.pem"
  ```

* 命令

  ```
  # calicoctl node status
  Calico process is running.
  
  IPv4 BGP status
  +----------------+-------------------+-------+----------+-------------+
  |  PEER ADDRESS  |     PEER TYPE     | STATE |  SINCE   |    INFO     |
  +----------------+-------------------+-------+----------+-------------+
  | 192.168.10.171 | node-to-node mesh | up    | 13:31:03 | Established |
  | 192.168.10.172 | node-to-node mesh | up    | 13:31:02 | Established |
  | 192.168.10.173 | node-to-node mesh | up    | 13:31:03 | Established |
  | 192.168.10.174 | node-to-node mesh | up    | 13:31:03 | Established |
  +----------------+-------------------+-------+----------+-------------+
  
  IPv6 BGP status
  No IPv6 peers found.
  
  # calicoctl get nodes
  NAME            
  k8s-manual-01   
  k8s-manual-02   
  k8s-manual-03   
  k8s-manual-04   
  k8s-manual-05   
  
  #  calicoctl get ippool -o wide
  NAME                  CIDR            NAT    IPIPMODE   VXLANMODE   DISABLED   SELECTOR   
  default-ipv4-ippool   10.244.0.0/16   true   Always     Never       false      all()      
  ```

## 工作模式

* IPIP：Overlay Network方案，源数据包封装在宿主机网络包里进行转发和通信。（默认）

  采用Linux IPIP隧道技术实现的数据包封装与转发。
  IP 隧道（IP tunneling）是将一个IP报文封装在另一个IP报文的技术，Linux系统内核实现的IP隧道技术主要有三种：IPIP、GRE、SIT。

  Pod 1 访问Pod 2 大致流程如下：
  1.数据包（原始数据包）从容器出去到达宿主机，宿主机根据路由表发送到tunl0设备（IP隧道设备）
  2.Linux内核IPIP驱动将原始数据包封装在宿主机网络的IP包中（新的IP包目的地之是原IP包的下一跳地址，即192.168.31.63）。
  3.数据包根据宿主机网络到达Node2；
  4.Node2收到数据包后，使用IPIP驱动进行解包，从中拿到原始数据包；
  5.然后根据路由规则，根据路由规则将数据包转发给cali设备，从而到达容器2。

* BGP：基于路由转发，每个节点通过BGP协议同步路由表，写到宿主机。（值设置Never）

  基于路由转发，每个节点通过BGP协议同步路由表，将每个宿主机当做路由器，实现数据包转发

  calicoctl工具修改为BGP模式：

  ```
  # calicoctl get ipPool -o yaml > ippool.yaml
  # vi ipip.yaml
  apiVersion: projectcalico.org/v3
  kind: IPPool
  metadata:
   name: default-ipv4-ippool
  spec:
   blockSize: 26
   cidr: 10.244.0.0/16
   ipipMode: Never
   natOutgoing: true
  
  # calicoctl apply -f ippool.yaml
  # calicoctl get ippool -o wide
  ```

  Pod 1 访问Pod 2 大致流程如下：
  1.数据包从容器出去到达宿主机；
  2.宿主机根据路由规则，将数据包转发给下一跳（网关）；
  3.到达Node2，根据路由规则将数据包转发给cali设备，从而到达容器2。

* CrossSubnet：同时支持BGP和IPIP，即根据子网选择转发方式。

* 修改工作模式

  ```
  # vim calico.yaml 
  - name: CALICO_IPV4POOL_IPIP
    value: "Always"
  ```

* Route Reflector 模式（RR）

  Calico 维护的网络在默认是（Node-to-Node Mesh）全互联模式，Calico集群中的节点之间都会相互建立连接，用于路由交换。但是随着集群规模的扩大，mesh模式将形成一个巨大服务网格，连接数成倍增加，就会产生性能问题。
  这时就需要使用Route Reflector（路由器反射）模式解决这个问题。

  为了解决大量的节点有BGP 造成网络链接过多 需要配置单独的两个节点用来存储这些信息 进行路由反射

  所以需要将node-to-node mesh 改为由制定节点充当路由反射器，充当一个专门的路由表分发，实际的网络还是走的bgp

  在进行切换的时候 需要关闭Calico 会影响节点的之间的网络

  1. 关闭node-to-node模式
     添加default BGP配置，调整nodeToNodeMeshEnabled和asNumber：

     ```
     # calicoctl apply -f ippool.yaml
     # calicoctl get ippool -o wide
     
     cat << EOF | calicoctl create -f -
     apiVersion: projectcalico.org/v3
     kind: BGPConfiguration
     metadata:
      name: default
     spec:
      logSeverityScreen: Info
      nodeToNodeMeshEnabled: false
      asNumber: 63400
     EOF
     ```

  2. 配置指定节点充当路由反射器
     为方便让BGPPeer轻松选择节点，通过标签选择器匹配。
     给路由器反射器节点打标签：
     然后配置路由器反射器节点routeReflectorClusterID

     ```
     # calicoctl get node k8s-node2 -o yaml > rr-node.yaml
     # vi rr-node.yaml
     ...
     bgp:
      ipv4Address: 192.168.31.63/24
      routeReflectorClusterID: 244.0.0.1 # 添加集群ID
     ...
     # calicoctl apply -f rr-node.yaml
     ```

  3. 使用标签选择器将路由反射器节点与其他非路由反射器节点配置为对等

     ```
     # vi bgppeer.yaml
     apiVersion: projectcalico.org/v3
     kind: BGPPeer
     metadata:
      name: peer-with-route-reflectors
     spec:
      nodeSelector: all()
      peerSelector: route-reflector == 'true'
      
     # calicoctl apply -f bgppeer.yaml
     查看节点的BGP连接状态：
     # calicoctl node status
     ```

# Sercice Mesh

服务网格 对于每个应用进行代理

Service Mesh 的中文译为“服务网格”，是一个用于处理服务和服务之间通信的基础设施层，它负责为构建复杂的云原生应用传递可靠的网络请求，并为服务通信实现了微服务所需的基本组件功能，例如服务发现、负载均衡、监控、流量管理、访问控制等。在实践中，服务网格通常实现为一组和应用程序部署在一起的轻量级的网络代理，但对应用程序来说是透明的。
右图，绿色方块为应用服务，蓝色方块为Sidecar Proxy，应用服务之间通过Sidecar Proxy 进行通信，整个服务通信形成图中的蓝色网络连线，图中所有蓝色部分就形成一个网络，这个就是服务网格名字的由来。
Service Mesh 部署网络结构图

Service Mesh有以下特点：
•治理能力独立（Sidecar）
•应用程序无感知
•服务通信的基础设施层
•解耦应用程序的重试/超时、监控、追踪和服务发现



# Istio

* 连接（Connect）
  -流量管理
  -负载均衡
  -灰度发布
* 安全（Secure）
  -认证
  -鉴权
* 控制（Control）
  -限流
  -ACL
* 观察（Observe）
  -监控
  -调用链

Istio服务网格在逻辑上分为数据平面和控制平面。

* 控制平面：使用全新的部署模式：istiod，这个组件负责处理Sidecar注入、证书分发、配置管理等功能，替代原有组件，降低复杂度，提高易用性。

  * Pilot：策略配置组件，为Proxy提供服务发现、智能路由、错误处理等。
  * Citadel：安全组件，提供证书生成下发、加密通信、访问控制。
  * Galley：配置管理、验证、分发。

* 数据平面：

  由一组Proxy组成，这些Proxy负责所有微服务网络通信，实现高效转发和策略。使用envoy实现，envoy是一个基于C++实现的L4/L7 Proxy转发器，是Istio在数据平面唯一的组件。

* Istio有4个配置资源，落地所有流量管理需求：

  * VirtualService（虚拟服务）：实现服务请求路由规则的功能。

    * 定义路由规则

    * 描述满足条件的请求去哪里

      ```
      apiVersion: networking.istio.io/v1alpha3
      kind: VirtualService
      metadata:
      name: httpbin
      spec:
      hosts:
      -"*"
      gateways:
      -httpbin-gateway
      http:
      -route:
      -destination:
      host: httpbin # 指定Service名称
      port:
      number: 8000 # service端口
      kubectl get vs # 查看已创建的虚拟服务
      ```

      

  * DestinationRule（目标规则）：实现目标服务的负载均衡、服务发现、故障处理和故障注入的功能。

    定义虚拟服务路由目标地址的真实地址，即子集（subset），支持多种负载均衡策略：

    不只是实现负载均衡 还有管理流量

    * 随机
    * 权重
    * 最小请求数

  * Gateway（网关）：让服务网格内的服务，可以被全世界看到。

    Gateway（网关）与Kubernetes Ingress有什么区别？
    Kubernetes Ingress与Getaway都是用于为集群内服务提供访问入口，但Ingress主要功能比较单一，不易于Istio现有流量管理功能集成。
    目前Gateway支持的功能：
    •支持L4-L7的负载均衡
    •支持HTTPS和mTLS
    •支持流量镜像、熔断等

  * ServiceEntry（服务入口）：允许管理网格外的服务的流量。



## 安装

下载软件包：https://github.com/istio/istio/releases

```
# tar zxvf istio-1.13.3-linux-amd64.tar.gz 
# cd istio-1.13.3
# mv bin/istioctl /usr/bin/	
# istioctl install

# 最后 部署完成查看
# istio-ingressgateway 负责应用中所有的通信的入口
# istiod istio的控制面板
# istio-ingressgateway 是LoadBalancer是对接公有云的 如果没有公有云 就降级一个NodePort形式
# kubectl get pods,svc -n istio-system
NAME                                       READY   STATUS    RESTARTS   AGE
pod/istio-ingressgateway-b7ffbd9c6-c54fr   1/1     Running   0          70s
pod/istiod-7c595445b6-jtz5q                1/1     Running   0          96s

NAME                           TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)                                      AGE
service/istio-ingressgateway   LoadBalancer   10.0.0.232   <pending>     15021:31411/TCP,80:31583/TCP,443:32363/TCP   70s
service/istiod                 ClusterIP      10.0.0.89    <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP        96s

```

## 验证

```
# 使用示例
# cd samples/httpbin
# kubectl apply -f httpbin-nodeport.yaml 
```

istio的服务管理需要手动注入istioctl服务 需要动态的注入istio的proxy

### Sidecar 手动注入

```
kubectl apply -f <(istioctl kube-inject -f httpbin-nodeport.yaml)
或者
istioctl kube-inject -f httpbin-nodeport.yaml |kubectl apply -f –

```

```
# 使用 istioctl kube-inject注入 就是生成一个需要istio的yaml文件
# istioctl kube-inject -f httpbin-nodeport.yaml
# 执行
# kubectl apply -f <(istioctl kube-inject -f httpbin-nodeport.yaml) 
# 或者istioctl kube-inject -f httpbin-nodeport.yaml |kubectl apply -f –

# 可以看到httpbin已经有俩容器了
# kubectl get pods
NAME                                 READY   STATUS     RESTARTS        AGE
httpbin-7f678fdb89-dbvpw             0/2     Init:0/1   0               4s

# kubectl get pods,svc -n istio-system
```

2. 配置gateway访问

   ```
   # kubectl apply -f httpbin-gateway.yaml 
   # kubectl get gateway,virtualservice
   NAME                                          AGE
   gateway.networking.istio.io/httpbin-gateway   12s
   
   NAME                                         GATEWAYS              HOSTS   AGE
   virtualservice.networking.istio.io/httpbin   ["httpbin-gateway"]   ["*"]   12s
   
   # kubectl get pods,svc 
   NAME                                     READY   STATUS    RESTARTS        AGE
   pod/httpbin-7f678fdb89-dbvpw             2/2     Running   0               3m35s
   pod/metrics-flask-app-66c9995b58-ngtm4   1/1     Running   2 (6d17h ago)   11d
   pod/mychar-webtest8-86b4f99b6b-wsh2w     1/1     Running   2 (6d17h ago)   9d
   pod/nginx-hpa-588d4dccb6-568ns           1/1     Running   2 (6d17h ago)   12d
   pod/nginx-hpa-588d4dccb6-z4zzh           1/1     Running   2 (6d17h ago)   12d
   pod/nginx-mychart-55f9579b65-hjs8l       1/1     Running   2 (6d17h ago)   10d
   pod/nginxtest-fc5f8b88-md9nb             1/1     Running   3 (6d17h ago)   13d
   pod/webtest-9f8cc89f7-nctrv              1/1     Running   2 (6d17h ago)   9d
   
   NAME                        TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)          AGE
   service/httpbin             NodePort    10.0.0.146   <none>        8000:31576/TCP   34m
   service/kubernetes          ClusterIP   10.0.0.1     <none>        443/TCP          18d
   service/metrics-flask-app   ClusterIP   10.0.0.114   <none>        80/TCP           11d
   service/mychar-webtest8     ClusterIP   10.0.0.169   <none>        80/TCP           9d
   service/nginx-hpa           ClusterIP   10.0.0.75    <none>        18080/TCP        12d
   service/nginx-mychart       ClusterIP   10.0.0.59    <none>        80/TCP           10d
   service/webtest             ClusterIP   10.0.0.184   <none>        80/TCP           9d
   
   # curl http://192.168.10.170:31576/
   ```

### 自动注入

指定命名空间下所有pod自动注入

```
# kubectl create ns istio-test

# kubectl apply -f httpbin-nodeport.yaml -n istio-test

```

给当前命名空间打一个标签启用自动注入 就实现了这个命名空间下的所有pod都会启用istio

是通过准入控制webhook实现的 只要有当前标签都会自动注入

```
# kubectl label namespace istio-test istio-injection=enabled
# kubectl get ns --show-labels | grep istio-test
istio-test             Active   107s   istio-injection=enabled,kubernetes.io/metadata.name=istio-test

# kubectl get pods -n istio-test 
NAME                       READY   STATUS    RESTARTS   AGE
httpbin-84cddc85d4-985zz   2/2     Running   0          4s
```



## 卸载

```
# istioctl x uninstall --purge
```

## 查看部署背后YAML文件：

```
# istioctl manifest generate
```





如果报错 error occurred forwarding 

是因为istio以来socat端口转发 安装socat即可

报错信息

```
2022-05-16T08:44:58.930891Z     error   klog    an error occurred forwarding 41959 -> 15017: error forwarding port 15017 to pod 97c7ffed678673a69a2f8b2e83468f3797f4e3e9be255341fb60b0cdc88aab52, uid : unable to do port forwarding: socat not found
2022-05-16T08:44:58.931121Z     error   klog    lost connection to pod
Error: Post "https://localhost:41959/inject": EOF
```

解决

```
#  yum install -y socat
```



## 部署官方示例

1. 创建命名空间并开启自动注入

   ```
   # kubectl create ns bookinfo
   # kubectl label namespace bookinfo istio-injection=enabled
   ```

   

2. 部署应用YAML

   ```
   # cd /opt/istio/istio-1.14.0-beta.0/samples/bookinfo/
   # kubectl apply -f platform/kube/bookinfo.yaml -n bookinfo
   # kubectl get pod -n bookinfo
   ```

   

3. 创建Ingress网关

   ```
   # kubectl apply -f networking/bookinfo-gateway.yaml -n bookinfo
   ```

   

4. 确认网关和访问地址，访问应用页面

   ```
   # kubectl get pods,svc -n istio-system
   http://192.168.10.170:31471/productpage
   ```

   

5. reviews 微服务部署3 个版本，用于测试灰度发布效果：
   1. v1 版本不会调用ratings 服务 现部署v1
   2. v2 版本会调用ratings 服务，并使用5个黑色五角星来显示评分信息 升级的时候部署v2
   3. v3 版本会调用ratings 服务，并使用5个红色五角星来显示评分信息 升级的时候部署v3

6. 流量全部发送到reviews v1版本（不带五角星）默认

7. 将90%的流量发送到reviews v1版本，另外10%的流量发送到reviews v2版本（5个黑色五角星），最后完全切换到v2版本

   1. 创建规则 将所有规则转发到v1版本 

      访问http://192.168.10.170:31471/productpage 没有星了

      ```
      # kubectl apply -f networking/virtual-service-all-v1.yaml -n bookinfo
      ```

   2. 将10%的流量转发到v2 就是destination文件 通过reviews的pod标签找到的

      ```
      # kubectl apply -f networking/destination-rule-all.yaml -n bookinfo
      ```

   3. 切换流量

      访问http://192.168.10.170:31471/productpage  会有一定几率访问到

      ```
      # kubectl apply -f networking/virtual-service-reviews-90-10.yaml -n bookinfo
      ```

   4. 将50%的流量发送到v2版本，另外50%的流量发送到v3版本（5个红色五角星）

      ```
      # kubectl apply -f networking/virtual-service-reviews-v2-v3.yaml -n bookinfo
      ```

      

# 流量镜像

流量镜像：将请求复制一份，并根据策略来处理这个请求，不会影响真实请求。



## 监控

Istio集成了多维度监控系统：
•使用Kiali观测应用
•使用Prometheus+Grafana查看系统状态
•使用Jaeger进行链路追踪

```
# 修改文件service 为nodeport
# vim addons/kiali.yaml 
……
spec:
  ports:
  - name: http
    protocol: TCP
    port: 20001
  - name: http-metrics
    protocol: TCP
    port: 9090
  selector:
    app.kubernetes.io/name: kiali
    app.kubernetes.io/instance: kiali
  type: NodePort
……

# kubectl apply -f addons/kiali.yaml -n istio-system

# 修改文件service 为nodeport
# kubectl apply -f addons/prometheus.yaml -n istio-system
# kubectl apply -f addons/grafana.yaml -n istio-system
# kubectl apply -f addons/jaeger.yaml -n istio-system

访问：
http://192.168.10.170:32199/kiali/console/overview?duration=60&refresh=15000
```

放



# 网络策略概述

podSelector：目标Pod，根据标签选择。
policyTypes：策略类型，指定策略用于入站、出站流量。
Ingress：from是可以访问的白名单，可以来自于IP段、命名空间、Pod标签等，ports是可以访问的端口。
Egress：这个Pod组可以访问外部的IP段和端口。

网络策略工作流程：
1、创建Network Policy资源
2、Policy Controller监控网络策略，同步并通知节点上程序
3、节点上DaemonSet运行的程序从etcd中获取Policy，调用本地Iptables创建防火墙规则

# tips

tcpdump工具

-nn ip方式显示

```
# tcpdump udp -i ens192 -nn -e
```



