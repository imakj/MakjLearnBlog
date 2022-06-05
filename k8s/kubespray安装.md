# 环境准备

官网：https://kubespray.io/#/

|  系统类型  |     IP地址     | 节点角色 | CPU  | Memory | Hostname  |
| :--------: | :------------: | :------: | :--: | :----: | :-------: |
| centos-7.9 | 192.168.10.175 |  master  | \>=2 | \>=2G  | k8sspray1 |
| centos-7.9 | 192.168.10.175 |  master  | \>=2 | \>=2G  | k8sspray2 |
| centos-7.9 | 192.168.10.175 |  worker  | \>=2 | \>=2G  | k8sspray3 |

# 系统设置(所有节点执行)

1. 修改hostname

   ```bash
   # hostnamectl set-hostname k8sspray1
   # hostnamectl set-hostname k8sspray2
   # hostnamectl set-hostname k8sspray3
   ```

2. 关闭防火墙、selinux、swap，重置iptables

   ```bash
   # 关闭selinux
   # setenforce 0
   # sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config
   # 关闭防火墙
   # systemctl stop firewalld && systemctl disable firewalld
   
   # 设置iptables规则
   # iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat && iptables -P FORWARD ACCEPT
   # 关闭swap
   # swapoff -a && free –h
   
   # 关闭dnsmasq(否则可能导致容器无法解析域名)
   # service dnsmasq stop && systemctl disable dnsmasq
   ```

3. k8s相关参数设置

   ```
   # cat > /etc/sysctl.d/kubernetes.conf <<EOF
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
   net.ipv4.ip_nonlocal_bind = 1
   net.ipv4.ip_forward = 1
   vm.swappiness = 0
   vm.overcommit_memory = 0
   EOF
   # sysctl -p /etc/sysctl.d/kubernetes.conf
   sysctl: cannot stat /proc/sys/net/bridge/bridge-nf-call-ip6tables: No such file or directory
   sysctl: cannot stat /proc/sys/net/bridge/bridge-nf-call-iptables: No such file or directory
   net.ipv4.ip_nonlocal_bind = 1
   net.ipv4.ip_forward = 1
   vm.swappiness = 0
   vm.overcommit_memory = 0
   
   ```

4. 移除docker相关

   ```
   # yum remove -y docker*
   # rm -f /etc/docker/daemon.json
   ```

5. 节点配置hosts

   ```
   [root@k8sspray1 ~]# vim /etc/hosts
   
   127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
   ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
   192.168.10.175 k8sspray1
   192.168.10.176 k8sspray2
   192.168.10.177 k8sspray3
   ```

# kubespray配置

1. k8sspray1节点配置免密登录(或者任意一个节点)

   ```
   [root@k8sspray1 ~]# ssh-keygen
   [root@k8sspray1 ~]# ssh-copy-id -i /root/.ssh/id_rsa.pub k8sspray1
   [root@k8sspray1 ~]# ssh-copy-id -i /root/.ssh/id_rsa.pub k8sspray2
   [root@k8sspray1 ~]# ssh-copy-id -i /root/.ssh/id_rsa.pub k8sspray3
   ```

2. k8sspray1节点安装依赖

   ```
   [root@k8sspray1 kubespray]# yum install -y epel-release python36 python36-pip git
   [root@k8sspray1 kubespray-2.15.0]# yum install gcc libffi-devel python36-devel openssl-devel -y
   ```

3. k8sspray1节点下载源码

   ```
   [root@k8sspray1 kubespray]# wget https://github.com/kubernetes-sigs/kubespray/archive/refs/tags/v2.15.0.tar.gz
   
   [root@k8sspray1 kubespray]# tar zxvf v2.15.0.tar.gz 
   
   #查看pip依赖
   [root@k8sspray1 kubespray-2.15.0]# cd kubespray-2.15.0/
   [root@k8sspray1 kubespray-2.15.0]# cat requirements.txt
   ansible==2.9.16
   jinja2==2.11.1
   netaddr==0.7.19
   pbr==5.4.4
   jmespath==0.9.5
   ruamel.yaml==0.16.10
   
   [root@k8sspray1 kubespray-2.15.0]# pip3.6 install setuptools-rust
   [root@k8sspray1 kubespray-2.15.0]# pip3 install wheel
   
   
   [root@k8sspray1 kubespray-2.15.0]# pip3.6 install -r requirements.txt
   
   #如果安装失败，根据提示升级pip
   [root@k8sspray1 ~]# pip3.6 install --upgrade pip
   #然后进行逐个安装
   [root@k8sspray1 ~]# pip3 install ansible==2.9.16
   [root@k8sspray1 ~]# pip3 install jinja2==2.11.1
   [root@k8sspray1 ~]# pip3 install netaddr==0.7.19
   [root@k8sspray1 ~]# pip3 install pbr==5.4.4
   [root@k8sspray1 ~]# pip3 install jmespath==0.9.5
   [root@k8sspray1 ~]# pip3 install ruamel.yaml==0.16.10
   ```

   

4. 生成集群配置

   示例配置在目录inventory/sample中，复制一份出来作为自己集群的配置

   ```
   [root@k8sspray1 kubespray-2.15.0]# cp -rpf inventory/sample inventory/mycluster
   ```

5. kubespray有py脚本，可以直接根据环境变量自动生成配置文件，所以我们现在只需要设定好环境变量就可以

   ```
   # 使用真实的hostname（否则会自动把你的hostname改成node1/node2...这种）
   [root@k8sspray1 kubespray-2.15.0]# export USE_REAL_HOSTNAME=true
   # 指定配置文件位置
   [root@k8sspray1 kubespray-2.15.0]# export CONFIG_FILE=inventory/mycluster/hosts.yaml
   # 定义ip列表（你的服务器内网ip地址列表，3台及以上，前两台默认为master节点）
   [root@k8sspray1 kubespray-2.15.0]# declare -a IPS=(192.168.10.175 192.168.10.176 192.168.10.177)
   # 生成配置文件
   [root@k8sspray1 kubespray-2.15.0]# python3 contrib/inventory_builder/inventory.py ${IPS[@]}
   DEBUG: Adding group all
   DEBUG: Adding group kube-master
   DEBUG: Adding group kube-node
   DEBUG: Adding group etcd
   DEBUG: Adding group k8s-cluster
   DEBUG: Adding group calico-rr
   DEBUG: adding host k8sspray1 to group all
   DEBUG: adding host k8sspray2 to group all
   DEBUG: adding host k8sspray3 to group all
   DEBUG: adding host k8sspray1 to group etcd
   DEBUG: adding host k8sspray2 to group etcd
   DEBUG: adding host k8sspray3 to group etcd
   DEBUG: adding host k8sspray1 to group kube-master
   DEBUG: adding host k8sspray2 to group kube-master
   DEBUG: adding host k8sspray1 to group kube-node
   DEBUG: adding host k8sspray2 to group kube-node
   DEBUG: adding host k8sspray3 to group kube-node
   
   ```

6. 进行个性化配置，比如用docker还是containerd，docker的工作目录等

   ```
   # 定制化配置文件
   # 1. 节点组织配置（这里可以调整每个节点的角色）
   [root@k8sspray1 kubespray-2.15.0]# vim inventory/mycluster/hosts.yaml
   # 2. containerd配置（教程使用containerd作为容器引擎）
   [root@k8sspray1 kubespray-2.15.0]# vim inventory/mycluster/group_vars/all/containerd.yml
   # 3. 全局配置（可以在这配置http(s)代理实现外网访问）
   [root@k8sspray1 kubespray-2.15.0]# vim inventory/mycluster/group_vars/all/all.yml
   #大约在57 58行 配置一个可以发访问外网的https
   ……
   57 http_proxy: "http://192.168.10.106:11180"
   58 https_proxy: "https://192.168.10.106:11180"
   ……
   
   # 4. k8s集群配置（包括设置容器运行时、svc网段、pod网段等）
   [root@k8sspray1 kubespray-2.15.0]# vim inventory/mycluster/group_vars/k8s-cluster/k8s-cluster.yml
   #可以改个网段
   ……
    73 kube_service_addresses: 10.200.0.0/16
    74 
    75 # internal network. When used, it will assign IP
    76 # addresses from this range to individual pods.
    77 # This network must be unused in your network infrastructure!
    78 kube_pods_subnet: 10.233.0.0/16
    
    #容器改为containerd
    176 container_manager: containerd
   ……
   
   # 5. 修改etcd部署类型为host（默认是docker）因为上层我们用的containerd
   [root@k8sspray1 kubespray-2.15.0]# vim ./inventory/mycluster/group_vars/etcd.yml
   ……
   etcd_deployment_type: host
   ……
   
   # 6. 附加组件（ingress、dashboard等）
   [root@k8sspray1 kubespray-2.15.0]# vim ./inventory/mycluster/group_vars/k8s-cluster/addons.yml
   ……
    4 dashboard_enabled: true #打开dashboard
     
    87 ingress_nginx_enabled: true
   ……
   ```

# 部署

1. 所有节点执行 可以事先准备好安装包 也可以在线下载 上传到所有节点根目录:kubespray-k8s-releases-v2.15.0.tar.gz

   ```
   # tar zxvf kubespray-k8s-releases-v2.15.0.tar.gz
   # ll /tmp/releases/
   total 325248
   -rwxr-xr-x 1 root root       40288256 Feb  2 18:07 calicoctl
   -rwxr-xr-x 1 root root       39677175 Feb  2 17:29 cni-plugins-linux-amd64-v0.9.0.tgz
   -rwxr-xr-x 1 root users      30473598 Aug 28  2020 crictl
   -rwxr-xr-x 1 root root       13287324 Feb  2 16:26 crictl-v1.19.0-linux-amd64.tar.gz
   drwxr-xr-x 3 root 600260513       123 Aug 25  2020 etcd-v3.4.13-linux-amd64
   -rwxr-xr-x 1 root root       17373136 Feb  2 17:27 etcd-v3.4.13-linux-amd64.tar.gz
   drwxr-xr-x 2 root root              6 Feb  2 16:34 images
   -rwxr-xr-x 1 root root       39059456 Feb  2 16:42 kubeadm-v1.19.7-amd64
   -rwxr-xr-x 1 root root       42950656 Feb  3 16:33 kubectl-v1.19.7-amd64
   -rwxr-xr-x 1 root root      109937800 Feb  2 17:55 kubelet-v1.19.7-amd64
   
   # ansible_python_interpreter=/usr/bin/python2.7
   ```

2. k8sspray1执行一键部署

   ```
   [root@k8sspray1 kubespray-2.15.0]# ansible-playbook -i inventory/mycluster/hosts.yaml  -b cluster.yml -vvvv
   ```

   为了减少等待时间，等所有节点都部署好containerd的时候  可以提前拉取镜像
   
   ```
   # curl https://gitee.com/pa/pub-doc/raw/master/kubespray-v2.15.0-images.sh|bash -x
   ```
   
3. 部署成功

   ```
   [root@k8sspray1 kubespray-2.15.0]# kubectl get all -n kube-system
   NAME                                              READY   STATUS    RESTARTS   AGE
   pod/calico-kube-controllers-8b5ff5d58-4h9ls       1/1     Running   0          40m
   pod/calico-node-c4nn7                             1/1     Running   0          41m
   pod/calico-node-j4b64                             1/1     Running   0          41m
   pod/calico-node-qjc5v                             1/1     Running   0          41m
   pod/coredns-85967d65-7x6hb                        1/1     Running   0          40m
   pod/coredns-85967d65-bswtt                        1/1     Running   0          40m
   pod/dns-autoscaler-5b7b5c9b6f-n792z               1/1     Running   0          40m
   pod/kube-apiserver-k8sspray1                      1/1     Running   0          43m
   pod/kube-apiserver-k8sspray2                      1/1     Running   0          42m
   pod/kube-controller-manager-k8sspray1             1/1     Running   0          43m
   pod/kube-controller-manager-k8sspray2             1/1     Running   0          42m
   pod/kube-proxy-54gsg                              1/1     Running   0          41m
   pod/kube-proxy-mbpvg                              1/1     Running   0          41m
   pod/kube-proxy-rqsfh                              1/1     Running   0          41m
   pod/kube-scheduler-k8sspray1                      1/1     Running   0          43m
   pod/kube-scheduler-k8sspray2                      1/1     Running   0          42m
   pod/kubernetes-dashboard-86c6f9df5b-s48jn         1/1     Running   0          40m
   pod/kubernetes-metrics-scraper-678c97765c-cqrld   1/1     Running   0          40m
   pod/nginx-proxy-k8sspray3                         1/1     Running   0          41m
   pod/nodelocaldns-jhk2j                            1/1     Running   0          40m
   pod/nodelocaldns-lf6q8                            1/1     Running   0          40m
   pod/nodelocaldns-s77zx                            1/1     Running   0          40m
   
   NAME                                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                  AGE
   service/coredns                     ClusterIP   10.200.0.3       <none>        53/UDP,53/TCP,9153/TCP   40m
   service/dashboard-metrics-scraper   ClusterIP   10.200.252.85    <none>        8000/TCP                 40m
   service/kubernetes-dashboard        ClusterIP   10.200.115.248   <none>        443/TCP                  40m
   
   NAME                          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
   daemonset.apps/calico-node    3         3         3       3            3           <none>                   41m
   daemonset.apps/kube-proxy     3         3         3       3            3           kubernetes.io/os=linux   43m
   daemonset.apps/nodelocaldns   3         3         3       3            3           <none>                   40m
   
   NAME                                         READY   UP-TO-DATE   AVAILABLE   AGE
   deployment.apps/calico-kube-controllers      1/1     1            1           40m
   deployment.apps/coredns                      2/2     2            2           40m
   deployment.apps/dns-autoscaler               1/1     1            1           40m
   deployment.apps/kubernetes-dashboard         1/1     1            1           40m
   deployment.apps/kubernetes-metrics-scraper   1/1     1            1           40m
   
   NAME                                                    DESIRED   CURRENT   READY   AGE
   replicaset.apps/calico-kube-controllers-8b5ff5d58       1         1         1       40m
   replicaset.apps/coredns-85967d65                        2         2         2       40m
   replicaset.apps/dns-autoscaler-5b7b5c9b6f               1         1         1       40m
   replicaset.apps/kubernetes-dashboard-86c6f9df5b         1         1         1       40m
   replicaset.apps/kubernetes-metrics-scraper-678c97765c   1         1         1       40m
   [root@k8sspray1 kubespray-2.15.0]# kubectl get all -n kube-system
   NAME                                              READY   STATUS    RESTARTS   AGE
   pod/calico-kube-controllers-8b5ff5d58-4h9ls       1/1     Running   0          41m
   pod/calico-node-c4nn7                             1/1     Running   0          41m
   pod/calico-node-j4b64                             1/1     Running   0          41m
   pod/calico-node-qjc5v                             1/1     Running   0          41m
   pod/coredns-85967d65-7x6hb                        1/1     Running   0          40m
   pod/coredns-85967d65-bswtt                        1/1     Running   0          40m
   pod/dns-autoscaler-5b7b5c9b6f-n792z               1/1     Running   0          40m
   pod/kube-apiserver-k8sspray1                      1/1     Running   0          43m
   pod/kube-apiserver-k8sspray2                      1/1     Running   0          42m
   pod/kube-controller-manager-k8sspray1             1/1     Running   0          43m
   pod/kube-controller-manager-k8sspray2             1/1     Running   0          42m
   pod/kube-proxy-54gsg                              1/1     Running   0          41m
   pod/kube-proxy-mbpvg                              1/1     Running   0          41m
   pod/kube-proxy-rqsfh                              1/1     Running   0          41m
   pod/kube-scheduler-k8sspray1                      1/1     Running   0          43m
   pod/kube-scheduler-k8sspray2                      1/1     Running   0          42m
   pod/kubernetes-dashboard-86c6f9df5b-s48jn         1/1     Running   0          40m
   pod/kubernetes-metrics-scraper-678c97765c-cqrld   1/1     Running   0          40m
   #可以看到 在k8sspray3有一个nginx proxy的node
   pod/nginx-proxy-k8sspray3                         1/1     Running   0          41m
   pod/nodelocaldns-jhk2j                            1/1     Running   0          40m
   pod/nodelocaldns-lf6q8                            1/1     Running   0          40m
   pod/nodelocaldns-s77zx                            1/1     Running   0          40m
   
   NAME                                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                  AGE
   service/coredns                     ClusterIP   10.200.0.3       <none>        53/UDP,53/TCP,9153/TCP   40m
   service/dashboard-metrics-scraper   ClusterIP   10.200.252.85    <none>        8000/TCP                 40m
   service/kubernetes-dashboard        ClusterIP   10.200.115.248   <none>        443/TCP                  40m
   
   NAME                          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
   daemonset.apps/calico-node    3         3         3       3            3           <none>                   41m
   daemonset.apps/kube-proxy     3         3         3       3            3           kubernetes.io/os=linux   43m
   daemonset.apps/nodelocaldns   3         3         3       3            3           <none>                   40m
   
   NAME                                         READY   UP-TO-DATE   AVAILABLE   AGE
   deployment.apps/calico-kube-controllers      1/1     1            1           41m
   deployment.apps/coredns                      2/2     2            2           40m
   deployment.apps/dns-autoscaler               1/1     1            1           40m
   deployment.apps/kubernetes-dashboard         1/1     1            1           40m
   deployment.apps/kubernetes-metrics-scraper   1/1     1            1           40m
   
   NAME                                                    DESIRED   CURRENT   READY   AGE
   replicaset.apps/calico-kube-controllers-8b5ff5d58       1         1         1       41m
   replicaset.apps/coredns-85967d65                        2         2         2       40m
   replicaset.apps/dns-autoscaler-5b7b5c9b6f               1         1         1       40m
   replicaset.apps/kubernetes-dashboard-86c6f9df5b         1         1         1       40m
   replicaset.apps/kubernetes-metrics-scraper-678c97765c   1         1         1       40m
   
   ```

4. 在k8sspray3的manifests查看这个文件，这个文件夹下的所有yaml文件在k8s启动的时候都会加载

   ```
   [root@k8sspray3 ~]# cat /etc/kubernetes/manifests/nginx-proxy.yml 
   apiVersion: v1
   kind: Pod
   metadata:
     name: nginx-proxy
     namespace: kube-system
     labels:
       addonmanager.kubernetes.io/mode: Reconcile
       k8s-app: kube-nginx
     annotations:
       nginx-cfg-checksum: "1bb31b8ae44b9bb522c8a8d5ffbd1a501714a100"
   spec:
     hostNetwork: true
     dnsPolicy: ClusterFirstWithHostNet
     nodeSelector:
       kubernetes.io/os: linux
     priorityClassName: system-node-critical
     containers:
     - name: nginx-proxy
       image: docker.io/library/nginx:1.19
       imagePullPolicy: IfNotPresent
       resources:
         requests:
           cpu: 25m
           memory: 32M
       securityContext:
         privileged: true
       livenessProbe:
         httpGet:
           path: /healthz
           port: 8081
       readinessProbe:
         httpGet:
           path: /healthz
           port: 8081
       volumeMounts:
       - mountPath: /etc/nginx
         name: etc-nginx
         readOnly: true
     volumes:   #挂载了一个配置文件
     - name: etc-nginx
       hostPath:
         path: /etc/nginx
   
   ```

5. 查看这个配置文件 可以看到对两个master做了负载均衡，这样又可以负载均衡又可以高可用

   ```
   [root@k8sspray3 nginx]# cat /etc/nginx/nginx.conf 
   error_log stderr notice;
   
   worker_processes 2;
   worker_rlimit_nofile 130048;
   worker_shutdown_timeout 10s;
   
   events {
     multi_accept on;
     use epoll;
     worker_connections 16384;
   }
   
   stream {
     upstream kube_apiserver {
       least_conn;
       server 192.168.10.175:6443;
       server 192.168.10.176:6443;
       }
   
     server {
       listen        127.0.0.1:6443;
       proxy_pass    kube_apiserver;
       proxy_timeout 10m;
       proxy_connect_timeout 1s;
     }
   }
   
   http {
     aio threads;
     aio_write on;
     tcp_nopush on;
     tcp_nodelay on;
   
     keepalive_timeout 5m;
     keepalive_requests 100;
     reset_timedout_connection on;
     server_tokens off;
     autoindex off;
   
     server {
       listen 8081;
       location /healthz {
         access_log off;
         return 200;
       }
       location /stub_status {
         stub_status on;
         access_log off;
       }
     }
     }
   
   ```

6. 安装完成后在每个节点删掉代理设置

   ```
   [root@k8sspray1 kubespray-2.15.0]# rm -f /etc/systemd/system/containerd.service.d/http-proxy.conf
   [root@k8sspray1 kubespray-2.15.0]# systemctl daemon-reload
   [root@k8sspray1 kubespray-2.15.0]# systemctl restart containerd
   
   [root@k8sspray1 kubespray-2.15.0]# grep 11180 -r /etc/yum*
   /etc/yum.conf:proxy=http://192.168.10.105:11180
   /etc/yum.repos.d/containerd.repo:proxy=http://192.168.10.105:11180
   
   [root@k8sspray1 kubespray-2.15.0]# vim /etc/yum.conf 
   
   [main]
   cachedir=/var/cache/yum/$basearch/$releasever
   keepcache=0
   debuglevel=2
   logfile=/var/log/yum.log
   exactarch=1
   obsoletes=1
   gpgcheck=1
   plugins=1
   installonly_limit=5
   bugtracker_url=http://bugs.centos.org/set_project.php?project_id=23&ref=http://bugs.centos.org/bug_report_page.php?category=yum
   distroverpkg=centos-release
   proxy=http://192.168.10.105:11180  #删掉这一行
   
   # rm -f /etc/yum.repos.d/containerd.repo
   ```

# 集群冒烟测试

1. 创建一个nginx ds

   ```
   [root@k8sspray1 kubespray]# cat > nginx-ds.yml <<EOF
   apiVersion: v1
   kind: Service
   metadata:
     name: nginx-ds
     labels:
       app: nginx-ds
   spec:
     type: NodePort
     selector:
       app: nginx-ds
     ports:
     - name: http
       port: 80
       targetPort: 80
   ---
   apiVersion: apps/v1
   kind: DaemonSet
   metadata:
     name: nginx-ds
   spec:
     selector:
       matchLabels:
         app: nginx-ds
     template:
       metadata:
         labels:
           app: nginx-ds
       spec:
         containers:
         - name: my-nginx
           image: nginx:1.19
           ports:
           - containerPort: 80
   EOF
   
   [root@k8sspray1 kubespray]# kubectl apply -f nginx-ds.yml 
   service/nginx-ds created
   daemonset.apps/nginx-ds created
   
   ```

2. 检查各种ip连通性是否正常

   ```
   # ping 10.233.216.4
   # ping 10.233.118.3
   # ping 10.233.72.4
   ```

3. 检查service

   ```
   [root@k8sspray1 kubespray]# kubectl get service
   NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
   kubernetes   ClusterIP   10.200.0.1     <none>        443/TCP        104m
   nginx-ds     NodePort    10.200.91.97   <none>        80:32096/TCP   2m2s
   [root@k8sspray1 kubespray]# curl 10.200.91.97
   
   [root@k8sspray1 kubespray]# curl k8sspray1:32096
   [root@k8sspray1 kubespray]# curl k8sspray2:32096
   [root@k8sspray1 kubespray]# curl k8sspray3:32096
   ```

4. 检查dns

   ```
   [root@k8sspray1 kubespray]#cat > pod-nginx.yaml <<EOF
   apiVersion: v1
   kind: Pod
   metadata:
     name: nginx
   spec:
     containers:
     - name: nginx
       image: docker.io/library/nginx:1.19
       ports:
       - containerPort: 80
   EOF
   
   [root@k8sspray1 kubespray]#  kubectl apply -f pod-nginx.yaml
   pod/nginx created
   
   [root@k8sspray1 kubespray]# kubectl exec nginx -it -- /bin/bash
   root@nginx:/# cat /etc/resolv.conf
   search default.svc.cluster.local svc.cluster.local cluster.local
   nameserver 169.254.25.10
   options ndots:5
   
   root@nginx:/# curl nginx-ds
   ```

5. 查看日志

   ```
   [root@k8sspray1 kubespray]# kubectl get pods
   NAME             READY   STATUS    RESTARTS   AGE
   nginx            1/1     Running   0          83s
   nginx-ds-2sp78   1/1     Running   0          5m32s
   nginx-ds-fkdlm   1/1     Running   0          5m32s
   nginx-ds-w9v64   1/1     Running   0          5m32s
   
   [root@k8sspray1 kubespray]# kubectl logs nginx
   /docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
   /docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
   /docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
   10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
   10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
   /docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
   /docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
   /docker-entrypoint.sh: Configuration complete; ready for start up
   
   ```

6. Exec功能

   ```
   [root@k8sspray1 kubespray]# kubectl get pods -l app=nginx-ds
   NAME             READY   STATUS    RESTARTS   AGE
   nginx-ds-2sp78   1/1     Running   0          6m19s
   nginx-ds-fkdlm   1/1     Running   0          6m19s
   nginx-ds-w9v64   1/1     Running   0          6m19s
   
   # --后面为在容器里执行的指令
   [root@k8sspray1 kubespray]# kubectl exec -it nginx-ds-2sp78 -- nginx -v
   nginx version: nginx/1.19.10
   
   ```

# dashboard

1. 创建service

   ```
   [root@k8sspray1 kubespray]# cat > dashboard-svc.yaml <<EOF
   apiVersion: v1
   kind: Service
   metadata:
     namespace: kube-system
     name: dashboard
     labels:
       app: dashboard
   spec:
     type: NodePort
     selector:
       k8s-app: kubernetes-dashboard
     ports:
     - name: https
       nodePort: 30000
       port: 443
       targetPort: 8443
   EOF
   
   [root@k8sspray1 kubespray]# kubectl apply -f dashboard-svc.yaml
   service/dashboard created
   
   ```

2. 访问

   为了集群安全，从 1.7 开始，dashboard 只允许通过 https 访问，我们使用nodeport的方式暴露服务，可以使用 https://NodeIP:NodePort 地址访问 
   关于自定义证书 
   默认dashboard的证书是自动生成的，肯定是非安全的证书，如果大家有域名和对应的安全证书可以自己替换掉。使用安全的域名方式访问dashboard。 
   在dashboard-all.yaml中增加dashboard启动参数，可以指定证书文件，其中证书文件是通过secret注进来的。

   \- –tls-cert-file  
   \- dashboard.cer  
   \- –tls-key-file  
   \- dashboard.key

   ```
   [root@k8sspray1 kubespray]# kubectl get service -n kube-system 
   NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                  AGE
   coredns                     ClusterIP   10.200.0.3       <none>        53/UDP,53/TCP,9153/TCP   110m
   dashboard                   NodePort    10.200.194.51    <none>        443:30000/TCP            71s
   dashboard-metrics-scraper   ClusterIP   10.200.252.85    <none>        8000/TCP                 110m
   kubernetes-dashboard        ClusterIP   10.200.115.248   <none>        443/TCP                  110m
   
   ```

   右上可见登录地址为https://192.168.10.176:30000

3. 登录

   Dashboard 默认只支持 token 认证，所以如果使用 KubeConfig 文件，需要在该文件中指定 token

   ```
   [root@k8sspray1 kubespray]#  kubectl create sa dashboard-admin -n kube-system
   serviceaccount/dashboard-admin created
   [root@k8sspray1 kubespray]#  kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
   clusterrolebinding.rbac.authorization.k8s.io/dashboard-admin created
   [root@k8sspray1 kubespray]# ADMIN_SECRET=$(kubectl get secrets -n kube-system | grep dashboard-admin | awk '{print $1}')
   [root@k8sspray1 kubespray]#  kubectl describe secret -n kube-system ${ADMIN_SECRET} | grep -E '^token' | awk '{print $2}'
   eyJhbGciOiJSUzI1NiIsImtpZCI6IklCU1RReXRYV1dtSDZadUtBYWpCWHlhcFV1M1llVzVFbGtxR0tDZl84V2sifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tN2RndHQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiYTVlZDg0ODAtODg2NS00NmNkLTk5MzAtODdiZmQ0MWJlODRkIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1hZG1pbiJ9.JAXKyzDbHSMWhgHFlJnNNExDsYvsCEbanLhC-HneEeAR8rUUMo8V0Fes1vfbmlGe7Ad9TP4CgxCKHbAWKINxzRXUWOFc1Ud5MyoTqhjItB7GkJ3EqJIOhFjyXUtbOVB3A0_CIltShpmA3piJjuWoSRWpko2Wq1k41BWbbcFKr8Nt8-cfDIrqzTJhQLGcscI5abTz7kNhM2f1g6my4DxGrxRLBtSXDFLAMbRWDk8SwuhWaGC56231OXnOwrUE_CZ8x1J3UoCkR1msZ5g6JWr3hidsqEbctusn08bUqyy_vDSCU5FcxmyyZa3QdJja1X2y4D5Ei8LkQjHuNcKbvkO48Q
   
   ```

# 集群运维



