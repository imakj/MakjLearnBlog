# 准备环境

|   系统    |       IP       |  角色   | 主机名 |
| :-------: | :------------: | :-----: | ------ |
| centos7.9 | 192.168.10.190 | master1 | k8sm1  |
| centos7.9 | 192.168.10.191 | master2 | k8sm2  |
| centos7.9 | 192.168.10.192 | master3 | k8sm3  |
| centos7.9 | 192.168.10.193 |  work1  | k8sw1  |
| centos7.9 | 192.168.10.194 |  work1  | k8sm2  |

修改主机名：`hostnamectl set-hostname <主机名>`

# 系统设置

1. 所有节点设置主机名

   ```
   # hostnamectl set-hostname <主机名>
   ```

2. 所有节点安装依赖包

   ```
   # yum install -y conntrack ipvsadm ipset jq sysstat curl libseccomp
   ```

3. 关闭防火墙，swap

   ```
   # 关闭防火墙
   # systemctl stop firewalld && systemctl disable firewalld
   # 重置iptables
   # iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat && iptables -P FORWARD ACCEPT
   # 关闭swap
   # swapoff -a
   # sed -i '/swap/s/^\(.*\)$/#\1/g' /etc/fstab
   # 关闭selinux
   # setenforce 0
   # 关闭dnsmasq(否则可能导致docker容器无法解析域名)
   # service dnsmasq stop && systemctl disable dnsmasq
   ```

4. 所有节点添加系统参数

   ```
   # cat > /etc/sysctl.d/kubernetes.conf <<EOF
   net.bridge.bridge-nf-call-iptables=1
   net.bridge.bridge-nf-call-ip6tables=1
   net.ipv4.ip_forward=1
   vm.swappiness=0
   vm.overcommit_memory=1
   vm.panic_on_oom=0
   fs.inotify.max_user_watches=89100
   EOF
   # 生效文件
   # sysctl -p /etc/sysctl.d/kubernetes.conf
   ```

5. 所有节点安装docker

   ```
   # 配置yum源
   # yum -y install yum-utils
   # yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
   
   # 清理原有版本
   # yum remove docker \
                     docker-client \
                     docker-client-latest \
                     docker-common \
                     docker-latest \
                     docker-latest-logrotate \
                     docker-logrotate \
                     docker-engine \
                     container-selinux
            
   # 查看版本列表
   #yum list docker-ce --showduplicates | sort -r
   
   # 根据kubernetes对docker版本的兼容测试情况，安装18.09版本
   # yum install -y docker-ce-18.09.9 docker-ce-cli-18.09.9 containerd.io -y
   
   # 开机启动
   # systemctl enable docker
   
   # 配置docker目录
   # mkdir -p /data/docer/
   # mkdir /etc/docker/
   # vim /etc/docker/daemon.json
   {
       "graph": "/data/docker"
   }
   
   # 启动docker
   # service docker restart
   ```



# 工具安装

- **kubeadm:** 部署集群用的命令
- **kubelet:** 在集群中每台机器上都要运行的组件，负责管理pod、容器的生命周期
- **kubectl:** 集群管理工具（可选，只要在控制集群的节点上安装即可）

1. 配置yum源

   ```
   # vim /etc/yum.repos.d/kubernetes.repo
   [kubernetes]
   name=Kubernetes
   baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
   enabled=1
   gpgcheck=0
   repo_gpgcheck=0
   gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
          http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
   ```

2. 安装

   ```
   # 找到要安装的版本号
   # yum list kubeadm --showduplicates | sort -r
   
   # 安装指定版本（这里用的是1.14.0）
   $ yum install kubelet-1.14.0-0 -y && yum install kubectl-1.14.0-0 -y && yum install kubeadm-1.14.0-0 -y
   
   # 设置kubelet的cgroupdriver（kubelet的cgroupdriver默认为systemd，如果上面没有设置docker的exec-opts为systemd，这里就需要将kubelet的设置为cgroupfs）
   # 由于各自的系统配置不同，配置位置和内容都不相同
   # 1. /etc/systemd/system/kubelet.service.d/10-kubeadm.conf(如果此配置存在的情况执行下面命令：)
   $ sed -i "s/cgroup-driver=systemd/cgroup-driver=cgroupfs/g" /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
   
   # 2. 如果1中的配置不存在，则此配置应该存在(不需要做任何操)：/usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
   
   # 启动kubelet
   $ systemctl enable kubelet && systemctl start kubelet
   
   ```

#  准备配置文件（m1节点）

1. 文件说明

   - **addons**

     > kubernetes的插件，比如calico和dashboard。

   - **configs**

     > 包含了部署集群过程中用到的各种配置文件。

   - **scripts**

     > 包含部署集群过程中用到的脚本，如keepalive检查脚本。

   - **global-configs.properties**

     > 全局配置，包含各种易变的配置内容。

   - **init.sh**

     > 初始化脚本，配置好global-config之后，会自动生成所有配置文件。

2. 生成配置

   这里会根据大家各自的环境生成kubernetes部署过程需要的配置文件。 在每个节点上都生成一遍，把所有配置都生成好，后面会根据节点类型去使用相关的配置

   ```
   # cd到之前下载的git代码目录
   # cd /opt/kubernetes-ha-kubeadm
   
   # 编辑属性配置（根据文件注释中的说明填写好每个key-value）
   $ vim global-config.properties
   #kubernetes版本
   VERSION=v1.14.0
   
   #POD网段
   POD_CIDR=172.22.0.0/16
   
   #master虚拟ip
   MASTER_VIP=192.168.10.195
   
   #keepalived用到的网卡接口名
   VIP_IF=ens192
   
   # 生成配置文件，确保执行过程没有异常信息
   # ./init.sh
   
   # 查看生成的配置文件，确保脚本执行成功
   # find target/ -type f
   ```

# 高可用集群搭建

1. 两台节点安装keepalived(这里m1和m2)

   ```
   #  yum install -y keepalived
   ```

2. 创建目录(这里m1和m2)

   ```
   # mkdir -p /etc/keepalived
   ```

3. 复制配置文件，和检测脚本(这里m1和m2)

   ```
   # m1执行
   # cp /opt/kubernetes-ha-kubeadm/target/configs/keepalived-master.conf /etc/keepalived/keepalived.conf 
   
   # m2执行
   # cp /opt/kubernetes-ha-kubeadm/target/configs/keepalived-backup.conf /etc/keepalived/keepalived.conf 
   
   # m1和m2执行
   # cp /opt/kubernetes-ha-kubeadm/target/scripts/check-apiserver.sh /etc/keepalived/
   ```

4. 启动keepalived(这里m1和m2)

   ```
   # systemctl enable keepalived && service keepalived start
   
   # service keepalived status
   
   #查看日志
   # journalctl -f -u keepalived
   
   #查看ip 发现ip已经挂载上
   # ip a
   ```

5. 部署第一个主节点(m1)，这段执行的命令需要保存下来，后面添加节点用

   ```
   # cp /opt/kubernetes-ha-kubeadm/target/configs/kubeadm-config.yaml ~
   
   # 节点初始化
   # kubeadm init --config=kubeadm-config.yaml --experimental-upload-certs
   [init] Using Kubernetes version: v1.14.0
   [preflight] Running pre-flight checks
   	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
   	[WARNING Hostname]: hostname "k8sm1" could not be reached
   	[WARNING Hostname]: hostname "k8sm1": lookup k8sm1 on 223.5.5.5:53: no such host
   [preflight] Pulling images required for setting up a Kubernetes cluster
   [preflight] This might take a minute or two, depending on the speed of your internet connection
   [preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
   [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
   [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
   [kubelet-start] Activating the kubelet service
   [certs] Using certificateDir folder "/etc/kubernetes/pki"
   [certs] Generating "ca" certificate and key
   [certs] Generating "apiserver-kubelet-client" certificate and key
   [certs] Generating "apiserver" certificate and key
   [certs] apiserver serving cert is signed for DNS names [k8sm1 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.10.190 192.168.10.195]
   [certs] Generating "front-proxy-ca" certificate and key
   [certs] Generating "front-proxy-client" certificate and key
   [certs] Generating "etcd/ca" certificate and key
   [certs] Generating "apiserver-etcd-client" certificate and key
   [certs] Generating "etcd/peer" certificate and key
   [certs] etcd/peer serving cert is signed for DNS names [k8sm1 localhost] and IPs [192.168.10.190 127.0.0.1 ::1]
   [certs] Generating "etcd/healthcheck-client" certificate and key
   [certs] Generating "etcd/server" certificate and key
   [certs] etcd/server serving cert is signed for DNS names [k8sm1 localhost] and IPs [192.168.10.190 127.0.0.1 ::1]
   [certs] Generating "sa" key and public key
   [kubeconfig] Using kubeconfig folder "/etc/kubernetes"
   [kubeconfig] Writing "admin.conf" kubeconfig file
   [kubeconfig] Writing "kubelet.conf" kubeconfig file
   [kubeconfig] Writing "controller-manager.conf" kubeconfig file
   [kubeconfig] Writing "scheduler.conf" kubeconfig file
   [control-plane] Using manifest folder "/etc/kubernetes/manifests"
   [control-plane] Creating static Pod manifest for "kube-apiserver"
   [control-plane] Creating static Pod manifest for "kube-controller-manager"
   [control-plane] Creating static Pod manifest for "kube-scheduler"
   [etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
   [wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
   [apiclient] All control plane components are healthy after 15.502124 seconds
   [upload-config] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
   [kubelet] Creating a ConfigMap "kubelet-config-1.14" in namespace kube-system with the configuration for the kubelets in the cluster
   [upload-certs] Storing the certificates in ConfigMap "kubeadm-certs" in the "kube-system" Namespace
   [upload-certs] Using certificate key:
   0d8db125ade3d5081a69ce9f2ab3f9f3019147d8c8822debdbdb63f9a39567e7
   [mark-control-plane] Marking the node k8sm1 as control-plane by adding the label "node-role.kubernetes.io/master=''"
   [mark-control-plane] Marking the node k8sm1 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
   [bootstrap-token] Using token: j3all8.qvi0krq7mf4zoxdw
   [bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
   [bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
   [bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
   [bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
   [bootstrap-token] creating the "cluster-info" ConfigMap in the "kube-public" namespace
   [addons] Applied essential addon: CoreDNS
   [addons] Applied essential addon: kube-proxy
   
   Your Kubernetes control-plane has initialized successfully!
   
   To start using your cluster, you need to run the following as a regular user:
   
     mkdir -p $HOME/.kube
     sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
     sudo chown $(id -u):$(id -g) $HOME/.kube/config
   
   You should now deploy a pod network to the cluster.
   Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
     https://kubernetes.io/docs/concepts/cluster-administration/addons/
   
   You can now join any number of the control-plane node running the following command on each as root:
   
     kubeadm join 192.168.10.195:6443 --token j3all8.qvi0krq7mf4zoxdw \
       --discovery-token-ca-cert-hash sha256:80261e0de30cb02bf9a3933f00f616c72b27adada16e564d096b04b226aebc94 \
       --experimental-control-plane --certificate-key 0d8db125ade3d5081a69ce9f2ab3f9f3019147d8c8822debdbdb63f9a39567e7
   
   Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
   As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use 
   "kubeadm init phase upload-certs --experimental-upload-certs" to reload certs afterward.
   
   Then you can join any number of worker nodes by running the following on each as root:
   
   kubeadm join 192.168.10.195:6443 --token j3all8.qvi0krq7mf4zoxdw \
       --discovery-token-ca-cert-hash sha256:80261e0de30cb02bf9a3933f00f616c72b27adada16e564d096b04b226aebc94 
   
   # 根据日志发现，copy而配置文件
   # mkdir -p ~/.kube
   # cp -i /etc/kubernetes/admin.conf ~/.kube/config
   
   ```

6. 测试一下kubectl

   ```
   # kubectl get pods --all-namespaces
   NAMESPACE     NAME                            READY   STATUS    RESTARTS   AGE
   kube-system   coredns-8567978547-989h4        0/1     Pending   0          76s
   kube-system   coredns-8567978547-v566k        0/1     Pending   0          76s
   kube-system   etcd-k8sm1                      1/1     Running   0          20s
   kube-system   kube-apiserver-k8sm1            1/1     Running   0          11s
   kube-system   kube-controller-manager-k8sm1   1/1     Running   0          31s
   kube-system   kube-proxy-sj62v                1/1     Running   0          76s
   kube-system   kube-scheduler-k8sm1            1/1     Running   0          23s
   
   # curl -k https://localhost:6443/healthz
   ok
   ```

7. 部署网络插件

   ```
   # 创建目录(m1执行)
   # mkdir -p /etc/kubernetes/addons
   
   #复制calico配置到配置好kubectl的节点(m1执行)
   # cp /opt/kubernetes-ha-kubeadm/target/addons/calico* /etc/kubernetes/addons/
   
   # 部署calico
   # kubectl apply -f /etc/kubernetes/addons/calico-rbac-kdd.yaml
   clusterrole.rbac.authorization.k8s.io/calico-node created
   clusterrolebinding.rbac.authorization.k8s.io/calico-node created
   # kubectl apply -f /etc/kubernetes/addons/calico.yaml
   configmap/calico-config created
   service/calico-typha created
   deployment.apps/calico-typha created
   daemonset.extensions/calico-node created
   customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
   customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
   customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
   customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
   customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
   customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
   
   customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
   customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
   customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
   serviceaccount/calico-node created
   
   # 查看状态
   # kubectl get pods -n kube-system
   NAME                            READY   STATUS    RESTARTS   AGE
   calico-node-q5hgc               1/2     Running   0          51s
   calico-typha-666749994b-4tfzg   0/1     Pending   0          50s
   coredns-8567978547-989h4        1/1     Running   0          7m37s
   coredns-8567978547-v566k        1/1     Running   0          7m37s
   etcd-k8sm1                      1/1     Running   0          6m41s
   kube-apiserver-k8sm1            1/1     Running   0          6m32s
   kube-controller-manager-k8sm1   1/1     Running   0          6m52s
   kube-proxy-sj62v                1/1     Running   0          7m37s
   kube-scheduler-k8sm1            1/1     Running   0          6m44s
   
   # kubectl describe pods -n kube-system calico-typha-666749994b-4tfzg
   Name:               calico-typha-666749994b-4tfzg
   Namespace:          kube-system
   Priority:           0
   PriorityClassName:  <none>
   Node:               <none>
   Labels:             k8s-app=calico-typha
                       pod-template-hash=666749994b
   Annotations:        scheduler.alpha.kubernetes.io/critical-pod: 
   Status:             Pending
   IP:                 
   Controlled By:      ReplicaSet/calico-typha-666749994b
   Containers:
     calico-typha:
       Image:      quay.io/calico/typha:v0.7.4
       Port:       5473/TCP
       Host Port:  5473/TCP
       Liveness:   http-get http://:9098/liveness delay=30s timeout=1s period=30s #success=1 #failure=3
       Readiness:  http-get http://:9098/readiness delay=0s timeout=1s period=10s #success=1 #failure=3
       Environment:
         TYPHA_LOGSEVERITYSCREEN:          info
         TYPHA_LOGFILEPATH:                none
         TYPHA_LOGSEVERITYSYS:             none
         TYPHA_CONNECTIONREBALANCINGMODE:  kubernetes
         TYPHA_DATASTORETYPE:              kubernetes
         TYPHA_HEALTHENABLED:              true
       Mounts:
         /var/run/secrets/kubernetes.io/serviceaccount from calico-node-token-knbmj (ro)
   Conditions:
     Type           Status
     PodScheduled   False 
   Volumes:
     calico-node-token-knbmj:
       Type:        Secret (a volume populated by a Secret)
       SecretName:  calico-node-token-knbmj
       Optional:    false
   QoS Class:       BestEffort
   Node-Selectors:  <none>
   Tolerations:     CriticalAddonsOnly
                    node.kubernetes.io/not-ready:NoExecute for 300s
                    node.kubernetes.io/unreachable:NoExecute for 300s
   Events:
     Type     Reason            Age                From               Message
     ----     ------            ----               ----               -------
     Warning  FailedScheduling  57s (x3 over 90s)  default-scheduler  0/1 nodes are available: 1 node(s) had taints that the pod didn't tolerate.  # 发现是没有工作节点
    
   ```

8. m2加入master节点(m2执行)，就是上面第5步生成的文件

   ```
   # kubeadm join 192.168.10.195:6443 --token j3all8.qvi0krq7mf4zoxdw \
   >     --discovery-token-ca-cert-hash sha256:80261e0de30cb02bf9a3933f00f616c72b27adada16e564d096b04b226aebc94 \
   >     --experimental-control-plane --certificate-key 0d8db125ade3d5081a69ce9f2ab3f9f3019147d8c8822debdbdb63f9a39567e7
   
   [preflight] Running pre-flight checks
   	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
   	[WARNING Hostname]: hostname "k8sm2" could not be reached
   	[WARNING Hostname]: hostname "k8sm2": lookup k8sm2 on 223.5.5.5:53: no such host
   [preflight] Reading configuration from the cluster...
   [preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
   [preflight] Running pre-flight checks before initializing the new control plane instance
   [preflight] Pulling images required for setting up a Kubernetes cluster
   [preflight] This might take a minute or two, depending on the speed of your internet connection
   [preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
   [download-certs] Downloading the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
   [certs] Using certificateDir folder "/etc/kubernetes/pki"
   [certs] Generating "apiserver-kubelet-client" certificate and key
   [certs] Generating "apiserver" certificate and key
   [certs] apiserver serving cert is signed for DNS names [k8sm2 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.10.191 192.168.10.195]
   [certs] Generating "etcd/server" certificate and key
   [certs] etcd/server serving cert is signed for DNS names [k8sm2 localhost] and IPs [192.168.10.191 127.0.0.1 ::1]
   [certs] Generating "etcd/healthcheck-client" certificate and key
   [certs] Generating "etcd/peer" certificate and key
   [certs] etcd/peer serving cert is signed for DNS names [k8sm2 localhost] and IPs [192.168.10.191 127.0.0.1 ::1]
   [certs] Generating "apiserver-etcd-client" certificate and key
   [certs] Generating "front-proxy-client" certificate and key
   [certs] Valid certificates and keys now exist in "/etc/kubernetes/pki"
   [certs] Using the existing "sa" key
   [kubeconfig] Generating kubeconfig files
   [kubeconfig] Using kubeconfig folder "/etc/kubernetes"
   [kubeconfig] Writing "admin.conf" kubeconfig file
   [kubeconfig] Writing "controller-manager.conf" kubeconfig file
   [kubeconfig] Writing "scheduler.conf" kubeconfig file
   [control-plane] Using manifest folder "/etc/kubernetes/manifests"
   [control-plane] Creating static Pod manifest for "kube-apiserver"
   [control-plane] Creating static Pod manifest for "kube-controller-manager"
   [control-plane] Creating static Pod manifest for "kube-scheduler"
   [check-etcd] Checking that the etcd cluster is healthy
   [kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.14" ConfigMap in the kube-system namespace
   [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
   [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
   [kubelet-start] Activating the kubelet service
   [kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
   [etcd] Announced new etcd member joining to the existing etcd cluster
   [etcd] Wrote Static Pod manifest for a local etcd member to "/etc/kubernetes/manifests/etcd.yaml"
   [etcd] Waiting for the new etcd member to join the cluster. This can take up to 40s
   [upload-config] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
   [mark-control-plane] Marking the node k8sm2 as control-plane by adding the label "node-role.kubernetes.io/master=''"
   [mark-control-plane] Marking the node k8sm2 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
   
   This node has joined the cluster and a new control plane instance was created:
   
   * Certificate signing request was sent to apiserver and approval was received.
   * The Kubelet was informed of the new secure connection details.
   * Control plane (master) label and taint were applied to the new node.
   * The Kubernetes control plane instances scaled up.
   * A new etcd member was added to the local/stacked etcd cluster.
   
   To start administering your cluster from this node, you need to run the following as a regular user:
   
   	mkdir -p $HOME/.kube
   	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   	sudo chown $(id -u):$(id -g) $HOME/.kube/config
   
   Run 'kubectl get nodes' to see this node join the cluster.
   
   
   #在m1节点查看 当前状态
   # kubectl get nodes
   NAME    STATUS   ROLES    AGE   VERSION
   k8sm1   Ready    master   12m   v1.14.0
   k8sm2   Ready    master   65s   v1.14.0
   ```

9. m3加入master节点(m3执行)，就是上面第5步生成的文件

   ```
   # kubeadm join 192.168.10.195:6443 --token j3all8.qvi0krq7mf4zoxdw \
       --discovery-token-ca-cert-hash sha256:80261e0de30cb02bf9a3933f00f616c72b27adada16e564d096b04b226aebc94 \
       --experimental-control-plane --certificate-key 0d8db125ade3d5081a69ce9f2ab3f9f3019147d8c8822debdbdb63f9a39567e7
       
   # 在m1节点查看 当前状态
   # kubectl get nodes
   NAME    STATUS     ROLES    AGE     VERSION
   k8sm1   Ready      master   14m     v1.14.0
   k8sm2   Ready      master   3m27s   v1.14.0
   k8sm3   NotReady   master   18s     v1.14.0
   ```

10. 连个work节点添加到集群命令(w1和w2执行)

    ```
    # kubeadm join 192.168.10.195:6443 --token j3all8.qvi0krq7mf4zoxdw \
        --discovery-token-ca-cert-hash sha256:80261e0de30cb02bf9a3933f00f616c72b27adada16e564d096b04b226aebc94 
    ```

11. 查看集群状态

    ```
    # kubectl get pods --all-namespaces
    NAMESPACE     NAME                            READY   STATUS    RESTARTS   AGE
    kube-system   calico-node-4rb85               2/2     Running   3          7m56s
    kube-system   calico-node-7mn9k               2/2     Running   0          3m47s
    kube-system   calico-node-cgcwj               2/2     Running   1          4m47s
    kube-system   calico-node-nq4x2               2/2     Running   0          3m41s
    kube-system   calico-node-q5hgc               1/2     Running   7          12m
    kube-system   calico-typha-666749994b-4tfzg   1/1     Running   0          12m
    kube-system   coredns-8567978547-989h4        1/1     Running   0          19m
    kube-system   coredns-8567978547-v566k        1/1     Running   0          19m
    kube-system   etcd-k8sm1                      1/1     Running   0          18m
    kube-system   etcd-k8sm2                      1/1     Running   0          7m55s
    kube-system   etcd-k8sm3                      1/1     Running   0          4m46s
    kube-system   kube-apiserver-k8sm1            1/1     Running   0          17m
    kube-system   kube-apiserver-k8sm2            1/1     Running   0          7m55s
    kube-system   kube-apiserver-k8sm3            1/1     Running   0          4m47s
    kube-system   kube-controller-manager-k8sm1   1/1     Running   1          18m
    kube-system   kube-controller-manager-k8sm2   1/1     Running   0          7m55s
    kube-system   kube-controller-manager-k8sm3   1/1     Running   0          4m46s
    kube-system   kube-proxy-6hccb                1/1     Running   0          4m47s
    kube-system   kube-proxy-gkvpq                1/1     Running   0          7m56s
    kube-system   kube-proxy-pphnw                1/1     Running   0          3m47s
    kube-system   kube-proxy-sfffj                1/1     Running   0          3m41s
    kube-system   kube-proxy-sj62v                1/1     Running   0          19m
    kube-system   kube-scheduler-k8sm1            1/1     Running   1          18m
    kube-system   kube-scheduler-k8sm2            1/1     Running   0          7m55s
    kube-system   kube-scheduler-k8sm3            1/1     Running   0          4m46s
    
    # kubectl get nodes
    NAME     STATUS   ROLES    AGE     VERSION
    k8sm1    Ready    master   24m     v1.14.0
    k8sm2    Ready    master   12m     v1.14.0
    k8sm3    Ready    master   9m49s   v1.14.0
    k8smw2   Ready    <none>   8m43s   v1.14.0
    k8sw1    Ready    <none>   8m49s   v1.14.0
    ```

12. 测试集群可用性,添加个nginx试试

    ```
    # cat > nginx-ds.yml <<EOF
    > apiVersion: v1
    > kind: Service
    > metadata:
    >   name: nginx-ds
    >   labels:
    >     app: nginx-ds
    > spec:
    >   type: NodePort
    >   selector:
    >     app: nginx-ds
    >   ports:
    >   - name: http
    >     port: 80
    >     targetPort: 80
    > ---
    > apiVersion: extensions/v1beta1
    > kind: DaemonSet
    > metadata:
    >   name: nginx-ds
    >   labels:
    >     addonmanager.kubernetes.io/mode: Reconcile
    > spec:
    >   template:
    >     metadata:
    >       labels:
    >         app: nginx-ds
    >     spec:
    >       containers:
    >       - name: my-nginx
    >         image: nginx:1.7.9
    >         ports:
    >         - containerPort: 80
    > EOF
    
    # 创建ds
    # kubectl create -f nginx-ds.yml
    service/nginx-ds created
    daemonset.extensions/nginx-ds created
    
    #检查各种ip连通性 在w1和w2上屏一下ip
    # kubectl get pods  -o wide
    NAME             READY   STATUS      RESTARTS   AGE     IP           NODE     NOMINATED NODE   READINESS GATES
    nginx-ds-84n29   1/1     Running     0          4h57m   172.22.3.2   k8sw1    <none>           <none>
    nginx-ds-glwm2   0/1     Completed   0          4h57m   <none>       k8smw2   <none>           <none>
    
    # w1上
    # ping 172.22.3.2
    PING 172.22.3.2 (172.22.3.2) 56(84) bytes of data.
    64 bytes from 172.22.3.2: icmp_seq=1 ttl=64 time=0.072 ms
    64 bytes from 172.22.3.2: icmp_seq=2 ttl=64 time=0.078 ms
    ^C
    --- 172.22.3.2 ping statistics ---
    2 packets transmitted, 2 received, 0% packet loss, time 999ms
    rtt min/avg/max/mdev = 0.072/0.075/0.078/0.003 ms
    [root@k8sw1 ~]# ping 172.22.4.3
    PING 172.22.4.3 (172.22.4.3) 56(84) bytes of data.
    64 bytes from 172.22.4.3: icmp_seq=1 ttl=63 time=0.570 ms
    64 bytes from 172.22.4.3: icmp_seq=2 ttl=63 time=0.387 ms
    
    
    #检查service可达性
    # kubectl get svc
    NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
    kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        5h30m
    nginx-ds     NodePort    10.106.81.198   <none>        80:32398/TCP   4h58m
    
    #所有节点查看nginx是否可用
    # curl http://10.106.81.198
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
    
    #检查每个节点的对外服务
    # curl http://192.168.10.194:32398
    
    #在测试dns是否可用 再创建一个容器
    # cat > pod-nginx.yaml <<EOF
    > apiVersion: v1
    > kind: Pod
    > metadata:
    >   name: nginx
    > spec:
    >   containers:
    >   - name: nginx
    >     image: nginx:1.7.9
    >     ports:
    >     - containerPort: 80
    > EOF
    
    # 启动
    # kubectl create -f pod-nginx.yaml
    pod/nginx created
    
    # 查看是否启动成功 
    # kubectl get pods  -o wide
    NAME             READY   STATUS    RESTARTS   AGE    IP           NODE     NOMINATED NODE   READINESS GATES
    nginx            1/1     Running   0          7s     172.22.3.3   k8sw1    <none>           <none>
    nginx-ds-84n29   1/1     Running   0          5h2m   172.22.3.2   k8sw1    <none>           <none>
    nginx-ds-glwm2   1/1     Running   1          5h2m   172.22.4.3   k8smw2   <none>           <none>
    
    # 进入到容器 执行一个ping
    [root@k8sm1 yml]# kubectl exec nginx -i -t -- /bin/bash
    root@nginx:/# cat /etc/resolv.conf 
    nameserver 10.96.0.10
    search default.svc.cluster.local svc.cluster.local cluster.local
    options ndots:5
    root@nginx:/# ping nginx-ds
    PING nginx-ds.default.svc.cluster.local (10.106.81.198): 48 data bytes  #可以看到 解析和真是ip相等
    ^C--- nginx-ds.default.svc.cluster.local ping statistics ---
    92 packets transmitted, 0 packets received, 100% packet loss
    
    # kubectl get svc
    NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
    kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        5h38m
    nginx-ds     NodePort    10.106.81.198   <none>        80:32398/TCP   5h6m
    
    ```

13. 部署

    1. 上传dashboard

       ```
       # cp /opt/kubernetes-ha-kubeadm/addons/dashboard-all.yaml /etc/kubernetes/addons/
       ```

    2.  创建服务

       ```
       # kubectl apply -f /etc/kubernetes/addons/dashboard-all.yaml
       ```

    3. 查看服务运行情况

       ```
       # kubectl get deployment kubernetes-dashboard -n kube-system
       NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
       kubernetes-dashboard   1/1     1            1           11s
       
       # kubectl --namespace kube-system get pods -o wide
       # 可以看到 服务在第k8smw2上
       kubernetes-dashboard-5bd4bfc87-2twwj   1/1     Running   0          2m      172.22.4.4       k8smw2   <none>           <none>
       
       # kubectl get services kubernetes-dashboard -n kube-system
       NAME                   TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
       kubernetes-dashboard   NodePort   10.107.152.105   <none>        443:30005/TCP   29s
       # netstat -ntlp|grep 30005
       ```

    4. 为了集群安全，从 1.7 开始，dashboard 只允许通过 https 访问，我们使用nodeport的方式暴露服务，可以使用 [https://NodeIP:NodePort](https://nodeip:NodePort/) 地址访问 关于自定义证书 默认dashboard的证书是自动生成的，肯定是非安全的证书，如果大家有域名和对应的安全证书可以自己替换掉。使用安全的域名方式访问dashboard。 在dashboard-all.yaml中增加dashboard启动参数，可以指定证书文件，其中证书文件是通过secret注进来的。

       > \- –tls-cert-file
       > \- dashboard.cer
       > \- –tls-key-file
       > \- dashboard.key

    5.  登录dashboard

       Dashboard 默认只支持 token 认证，所以如果使用 KubeConfig 文件，需要在该文件中指定 token，我们这里使用token的方式登录

       ```
       # 创建service account
       # kubectl create sa dashboard-admin -n kube-system
       
       # 创建角色绑定关系
       # kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
       
       # 查看dashboard-admin的secret名字
       # ADMIN_SECRET=$(kubectl get secrets -n kube-system | grep dashboard-admin | awk '{print $1}')
       
       # 打印secret的token
       # kubectl describe secret -n kube-system ${ADMIN_SECRET} | grep -E '^token' | awk '{print $2}'
       kubectl describe secret -n kube-system ${ADMIN_SECRET} | grep -E '^token' | awk '{print $2}'
       eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tN2dzdm0iLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiY2U0NTYyMTMtODYzYi0xMWViLWIyMTYtMDA1MDU2Yjk4YWY1Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1hZG1pbiJ9.vXV2OZo-_NxGv0u5zAEaa83CHcQjfEL-o6nQu7P6U9ktOunziVxJklZ582oG_5ZE6_P9AaIozS02q5mfQuTVpbgdmB1_lrTNPd3Ah75hi87bzsEWjuQwF52clOiKhnQfO4KCgVYiwkSOXmB9ielADHBdXgdHECo6ueUzJZ9N3B5g6_n0mie5rNtg8q2Q3aTNOAWZRJxZzh18Pf2fyIBf_YSe7HCLMyHtbucrNR_c2ALXaHKHiDaWkQrFp0TvZyKeOn9c7PKr_kOPCRrJhdjY5DPI9NsMhteuYgXD6kHF2zo0f_BRC8XkN0QKVMOYm_1YXzlu-9ThG5khjC7KmL3Sjw
       
       ```

       

