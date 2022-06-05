# 参考文档

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

# 环境

|   系统    |       IP       |  角色  | 主机名   |
| :-------: | :------------: | :----: | -------- |
| centos7.9 | 192.168.10.185 | master | k8snode1 |
| centos7.9 | 192.168.10.186 |  work  | k8snode2 |
| centos7.9 | 192.168.10.187 |  work  | k8snode3 |

# 安装docker(全部节点)

1. 见docker安装文档。

2. 修改docekr cgroup driver类型为systemd

   ```
   [root@localhost yum.repos.d]# sudo mkdir /etc/docker
   [root@localhost yum.repos.d]# cat <<EOF | sudo tee /etc/docker/daemon.json
   > {
   >   "exec-opts": ["native.cgroupdriver=systemd"],
   >   "log-driver": "json-file",
   >   "log-opts": {
   >     "max-size": "100m"
   >   },
   >   "storage-driver": "overlay2"
   > }
   > EOF
   {
     "exec-opts": ["native.cgroupdriver=systemd"],
     "log-driver": "json-file",
     "log-opts": {
       "max-size": "100m"
     },
     "storage-driver": "overlay2"
   }
   [root@localhost yum.repos.d]# cat /etc/docker/daemon.json
   {
     "exec-opts": ["native.cgroupdriver=systemd"],
     "log-driver": "json-file",
     "log-opts": {
       "max-size": "100m"
     },
     "storage-driver": "overlay2"
   }
   ```

3. 重启docker

   ```
   [root@localhost yum.repos.d]# systemctl daemon-reload
   或者
   [root@localhost yum.repos.d]# systemctl restart docker
   ```

4. 确定是是否为systemd

   ```
   [root@localhost yum.repos.d]# docker info
   Client:
    Debug Mode: false
   
   Server:
    Containers: 0
     Running: 0
     Paused: 0
     Stopped: 0
    Images: 0
    Server Version: 19.03.9
    Storage Driver: overlay2
     Backing Filesystem: xfs
     Supports d_type: true
     Native Overlay Diff: true
    Logging Driver: json-file
    Cgroup Driver: systemd
    Plugins:
     Volume: local
     Network: bridge host ipvlan macvlan null overlay
     Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
    Swarm: inactive
    Runtimes: runc
    Default Runtime: runc
    Init Binary: docker-init
    containerd version: 05f951a3781f4f2c1911b05e61c160e9c30eaa8e
    runc version: 12644e614e25b05da6fd08a38ffa0cfe1903fdec
    init version: fec3683
    Security Options:
     seccomp
      Profile: default
    Kernel Version: 3.10.0-1160.15.2.el7.x86_64
    Operating System: CentOS Linux 7 (Core)
    OSType: linux
    Architecture: x86_64
    CPUs: 4
    Total Memory: 3.7GiB
    Name: localhost.localdomain
    ID: 2GLH:4FEH:6NSH:PAWD:IEYP:7NM3:AYTD:3JLO:CVII:VNBJ:Z7DN:2TBV
    Docker Root Dir: /var/lib/docker
    Debug Mode: false
    Registry: https://index.docker.io/v1/
    Labels:
    Experimental: false
    Insecure Registries:
     127.0.0.0/8
    Live Restore Enabled: false
   ```

# 系统环境准备

1. 关闭防火墙,所有节点

   ```
   # systemctl stop firewalld && systemctl disable firewalld
   ```

2. 重置iptables，设置没有规则,设置iptable可以被桥接

   ```
   # iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat && iptables -P FORWARD ACCEPT
   # cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
   > net.bridge.bridge-nf-call-ip6tables = 1
   > net.bridge.bridge-nf-call-iptables = 1
   > EOF
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
   
   ```

3. 关闭swap和关闭selinux

   ```
   # 关闭swap
   # swapoff -a
   # sed -i '/swap/s/^\(.*\)$/#\1/g' /etc/fstab
   # 关闭selinux
   # setenforce 0
   # 关闭dnsmasq(否则可能导致docker容器无法解析域名)
   # service dnsmasq stop && systemctl disable dnsmasq
   ```

4. 设置系统参数

   ```
   # cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
   br_netfilter
   EOF
   
   
   # sysctl --system
   * Applying /usr/lib/sysctl.d/00-system.conf ...
   net.bridge.bridge-nf-call-ip6tables = 0
   net.bridge.bridge-nf-call-iptables = 0
   net.bridge.bridge-nf-call-arptables = 0
   * Applying /usr/lib/sysctl.d/10-default-yama-scope.conf ...
   kernel.yama.ptrace_scope = 0
   * Applying /usr/lib/sysctl.d/50-default.conf ...
   kernel.sysrq = 16
   kernel.core_uses_pid = 1
   kernel.kptr_restrict = 1
   net.ipv4.conf.default.rp_filter = 1
   net.ipv4.conf.all.rp_filter = 1
   net.ipv4.conf.default.accept_source_route = 0
   net.ipv4.conf.all.accept_source_route = 0
   net.ipv4.conf.default.promote_secondaries = 1
   net.ipv4.conf.all.promote_secondaries = 1
   fs.protected_hardlinks = 1
   fs.protected_symlinks = 1
   * Applying /etc/sysctl.d/99-sysctl.conf ...
   vm.swappiness = 0
   * Applying /etc/sysctl.d/k8s.conf ...
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
   * Applying /etc/sysctl.conf ...
   vm.swappiness = 0
   
   ```

5. k8s需要的端口,要保证所有端口都畅通

   ### 控制节点

   | 协议 | 方向 | 端口范围  | 作用                    | 使用者                       |
   | ---- | ---- | --------- | ----------------------- | ---------------------------- |
   | TCP  | 入站 | 6443      | Kubernetes API 服务器   | 所有组件                     |
   | TCP  | 入站 | 2379-2380 | etcd 服务器客户端 API   | kube-apiserver, etcd         |
   | TCP  | 入站 | 10250     | Kubelet API             | kubelet 自身、控制平面组件   |
   | TCP  | 入站 | 10251     | kube-scheduler          | kube-scheduler 自身          |
   | TCP  | 入站 | 10252     | kube-controller-manager | kube-controller-manager 自身 |

   ### 工作节点

   | 协议 | 方向 | 端口范围    | 作用           | 使用者                     |
   | ---- | ---- | ----------- | -------------- | -------------------------- |
   | TCP  | 入站 | 10250       | Kubelet API    | kubelet 自身、控制平面组件 |
   | TCP  | 入站 | 30000-32767 | NodePort 服务† | 所有组件                   |

6. 配置hsots 所有节点执行

   ```
   # cat << EOF>> /etc/hosts 
   192.168.10.185 k8snode1
   192.168.10.186 k8snode2
   192.168.10.187 k8snode3
   EOF
   
   # cat /etc/hosts
   127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4
   ::1 localhost localhost.localdomain localhost6 localhost6.localdomain6
   192.168.10.185 k8snode1
   192.168.10.186 k8snode2
   192.168.10.187 k8snode3
   ```

   

7. 配置系统安全认证，node1节点生成公钥：` # ssh-keygen` 一路默认确认即可

   ```
   [root@k8snode1 ~]# ssh-keygen
   Generating public/private rsa key pair.
   Enter file in which to save the key (/root/.ssh/id_rsa): 
   Created directory '/root/.ssh'.
   Enter passphrase (empty for no passphrase): 
   Enter same passphrase again: 
   Your identification has been saved in /root/.ssh/id_rsa.
   Your public key has been saved in /root/.ssh/id_rsa.pub.
   The key fingerprint is:
   SHA256:8rbA386IsPrAOfmdexMIVeb4TcL5CJPvF7+UqUFEJI0 root@k8snode1
   The key's randomart image is:
   +---[RSA 2048]----+
   |       .++o      |
   |      .*E+.      |
   |     .= = o      |
   |    .  = B       |
   |     ...S =      |
   | . o ..+.. o o   |
   |  * . o +.o =    |
   |   + + *o* + .   |
   |  .o+ =o+o= .    |
   +----[SHA256]-----+
   ```

8. 拷贝公钥到所有节点

   ```
   [root@k8snode1 ~]# ssh-copy-id -i /root/.ssh/id_rsa.pub k8snode1
   [root@k8snode1 ~]# ssh-copy-id -i /root/.ssh/id_rsa.pub k8snode2
   [root@k8snode1 ~]# ssh-copy-id -i /root/.ssh/id_rsa.pub k8snode3
   ```

   



# 容器运行时

k8s支持多种支持CRI的容器，在k8s运行时，会扫描默认容器的sock路径来匹配。

| 运行时     | 域套接字                        |
| ---------- | ------------------------------- |
| Docker     | /var/run/docker.sock            |
| containerd | /run/containerd/containerd.sock |
| CRI-O      | /var/run/crio/crio.sock         |

如果同时检测到 Docker 和 containerd，则优先选择 Docker。 这是必然的，因为 Docker 18.09 附带了 containerd 并且两者都是可以检测到的， 即使你仅安装了 Docker。 如果检测到其他两个或多个运行时，kubeadm 输出错误信息并退出。

kubelet 通过内置的 `dockershim` CRI 实现与 Docker 集成。

# 安装

安装 kubeadm、kubelet 和 kubectl

* kubeadm：一个初始化集群的工具和命令
* kubelet ：在集群中的每个节点上用来启动 Pod 和容器等。
* kubectl：负责和k8s进行交互的,用来与集群通信的命令行工具。

1. 准备k8s源(阿里源,三台节点执行)

   ```
   # cat <<EOF > /etc/yum.repos.d/kubernetes.repo
   [kubernetes]
   name=Kubernetes
   baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
   enabled=1
   gpgcheck=1
   repo_gpgcheck=1
   gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
   EOF
   
   # wget https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
   
   # rpm --import rpm-package-key.gpg
   
   # yum repolist  （更新本地源）
   ```

2. 安装kubeadm，全部节点

   ```
   [root@k8snode1 ~]# yum install kubeadm kubectl kubelet --disableexcludes=kubernetes -y
   ```

3. 启动kubelet

   ```
   [root@k8snode1 ~]# systemctl start kubelet && systemctl enable kubelet
   ```

4. 查看版本

   ```
   [root@k8snode1 ~]# rpm -qa| grep kub
   kubectl-1.21.0-0.x86_64
   kubernetes-cni-0.8.7-0.x86_64
   kubeadm-1.21.0-0.x86_64
   kubelet-1.21.0-0.x86_64
   
   ```

5. 初始化k8s，初始化的时候指定ip地址和ip的pod的地址

   * 控制平面节点是运行控制平面组件的机器， 包括 [etcd](https://kubernetes.io/zh/docs/tasks/administer-cluster/configure-upgrade-etcd/) （集群数据库） 和 [API Server](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-apiserver/) （命令行工具 [kubectl](https://kubernetes.io/docs/user-guide/kubectl-overview/) 与之通信）。

   * （推荐）如果计划将单个控制平面 kubeadm 集群升级成高可用， 你应该指定 `--control-plane-endpoint` 为所有控制平面节点设置共享端点。 端点可以是负载均衡器的 DNS 名称或 IP 地址。
   * 选择一个Pod网络插件，并验证是否需要为 `kubeadm init` 传递参数。 根据你选择的第三方网络插件，你可能需要设置 `--pod-network-cidr` 的值。 请参阅 [安装Pod网络附加组件](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network)。
   * （可选）从版本1.14开始，`kubeadm` 尝试使用一系列众所周知的域套接字路径来检测 Linux 上的容器运行时。 要使用不同的容器运行时， 或者如果在预配置的节点上安装了多个容器，请为 `kubeadm init` 指定 `--cri-socket` 参数。 请参阅[安装运行时](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-runtime)。
   * （可选）除非另有说明，否则 `kubeadm` 使用与默认网关关联的网络接口来设置此控制平面节点 API server 的广播地址。 要使用其他网络接口，请为 `kubeadm init` 设置 `--apiserver-advertise-address=<ip-address>` 参数。 要部署使用 IPv6 地址的 Kubernetes 集群， 必须指定一个 IPv6 地址，例如 `--apiserver-advertise-address=fd00::101`

   1. 在 `kubeadm init` 之前运行 `kubeadm config images pull`，以验证与 gcr.io 容器镜像仓库的连通性。如果拉去不下来镜像，通过`kubeadm config images list`进行查看手动下载

      ```
      [root@k8snode1 ~]# kubeadm config images list
      k8s.gcr.io/kube-apiserver:v1.21.0
      k8s.gcr.io/kube-controller-manager:v1.21.0
      k8s.gcr.io/kube-scheduler:v1.21.0
      k8s.gcr.io/kube-proxy:v1.21.0
      k8s.gcr.io/pause:3.4.1
      k8s.gcr.io/etcd:3.4.13-0
      k8s.gcr.io/coredns/coredns:v1.8.0
      ```

      使用阿里云的镜像拉去

      ```
      # docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.21.0
      # docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.21.0
      # docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.21.0
      # docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.21.0
      # docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.4.1
      # docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0
      # docker pull coredns/coredns:1.8.0  #coredns需要从官网下载
      
      #查看
      # docker images
      REPOSITORY                                                                    TAG                 IMAGE ID            CREATED             SIZE
      registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver            v1.21.0             4d217480042e        8 days ago          126MB
      registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy                v1.21.0             38ddd85fe90e        8 days ago          122MB
      registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager   v1.21.0             09708983cc37        8 days ago          120MB
      registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler            v1.21.0             62ad3129eca8        8 days ago          50.6MB
      registry.cn-hangzhou.aliyuncs.com/google_containers/pause                     3.4.1               0f8457a4c2ec        3 months ago        683kB
      coredns/coredns                                                               1.8.0               296a6d5035e2        5 months ago        42.5MB
      registry.cn-hangzhou.aliyuncs.com/google_containers/etcd                      3.4.13-0            0369cf4303ff        7 months ago        253MB
      
      
      # 标记改名
      # docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.21.0 k8s.gcr.io/kube-apiserver:v1.21.0
      # docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.21.0 k8s.gcr.io/kube-controller-manager:v1.21.0
      # docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.21.0 k8s.gcr.io/kube-scheduler:v1.21.0
      # docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.21.0 k8s.gcr.io/kube-proxy:v1.21.0
      # docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.4.1 k8s.gcr.io/pause:3.4.1
      # docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0 k8s.gcr.io/etcd:3.4.13-0
      # docker tag coredns/coredns:1.8.0  k8s.gcr.io/coredns/coredns:v1.8.0
      
      #删除之前的
      # docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.21.0
      # docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.21.0
      # docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.21.0
      # docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.21.0
      # docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.4.1
      # docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0
      # docker rmi coredns/coredns:1.8.0 
      
      #最后检查一遍
      # docker images
      REPOSITORY                           TAG                 IMAGE ID            CREATED             SIZE
      k8s.gcr.io/kube-apiserver            v1.21.0             4d217480042e        8 days ago          126MB
      k8s.gcr.io/kube-proxy                v1.21.0             38ddd85fe90e        8 days ago          122MB
      k8s.gcr.io/kube-controller-manager   v1.21.0             09708983cc37        8 days ago          120MB
      k8s.gcr.io/kube-scheduler            v1.21.0             62ad3129eca8        8 days ago          50.6MB
      k8s.gcr.io/pause                     3.4.1               0f8457a4c2ec        3 months ago        683kB
      k8s.gcr.io/coredns/coredns           v1.8.0              296a6d5035e2        5 months ago        42.5MB
      k8s.gcr.io/etcd                      3.4.13-0            0369cf4303ff        7 months ago        253MB
      
      #最后导出包。复制到其他两个节点
      [root@k8snode1 k8s]# docker save > kube-apiserver:v1.21.0.tar k8s.gcr.io/kube-apiserver:v1.21.0
      [root@k8snode1 k8s]# docker save > kube-controller-manager:v1.21.0.tar k8s.gcr.io/kube-controller-manager:v1.21.0
      [root@k8snode1 k8s]# docker save > kube-scheduler:v1.21.0.tar k8s.gcr.io/kube-scheduler:v1.21.0
      [root@k8snode1 k8s]# docker save > kube-proxy:v1.21.0.tar k8s.gcr.io/kube-proxy:v1.21.0
      [root@k8snode1 k8s]# docker save > pause:3.4.1.tar k8s.gcr.io/pause:3.4.1
      [root@k8snode1 k8s]# docker save > etcd:3.4.13-0.tar k8s.gcr.io/etcd:3.4.13-0
      [root@k8snode1 k8s]# docker save > coredns:v1.8.0.tar k8s.gcr.io/coredns/coredns:v1.8.0
      
      [root@k8snode1 k8s]# scp /data/k8s/k8simages.zip root@k8snode2:/data/k8s/
      [root@k8snode1 k8s]# scp /data/k8s/k8simages.zip root@k8snode3:/data/k8s/
      
      #node2执行
      [root@k8snode2 k8simages]# cd /data/k8s/
      [root@k8snode2 k8s]# unzip k8simages.zip
      [root@k8snode2 k8simages]# cd k8simages
      [root@k8snode2 k8simages]# for i in `ls`; do docker image load -i ${i} ;done
      
      #node3执行
      [root@k8snode3 k8simages]# cd /data/k8s/
      [root@k8snode3 k8s]# unzip k8simages.zip
      [root@k8snode3 k8simages]# cd k8simages
      [root@k8snode3 k8simages]# for i in `ls`; do docker image load -i ${i} ;done
      ```

   2. 重新初始化k8s

      ```
      [root@k8snode1 k8s]# kubeadm init --pod-network-cidr 172.16.0.0/16 --apiserver-advertise-address 192.168.10.185
      [init] Using Kubernetes version: v1.21.0
      [preflight] Running pre-flight checks  #环境监测，比如swap关闭
      [preflight] Pulling images required for setting up a Kubernetes cluster
      [preflight] This might take a minute or two, depending on the speed of your internet connection
      [preflight] You can also perform this action in beforehand using 'kubeadm config images pull'  #拉去镜像，刚才已经导入了
      #下面生成一些证书
      [certs] Using certificateDir folder "/etc/kubernetes/pki"  
      [certs] Generating "ca" certificate and key
      [certs] Generating "apiserver" certificate and key
      [certs] apiserver serving cert is signed for DNS names [k8snode1 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.10.185]
      [certs] Generating "apiserver-kubelet-client" certificate and key
      [certs] Generating "front-proxy-ca" certificate and key
      [certs] Generating "front-proxy-client" certificate and key
      [certs] Generating "etcd/ca" certificate and key
      [certs] Generating "etcd/server" certificate and key
      [certs] etcd/server serving cert is signed for DNS names [k8snode1 localhost] and IPs [192.168.10.185 127.0.0.1 ::1]
      [certs] Generating "etcd/peer" certificate and key
      [certs] etcd/peer serving cert is signed for DNS names [k8snode1 localhost] and IPs [192.168.10.185 127.0.0.1 ::1]
      [certs] Generating "etcd/healthcheck-client" certificate and key
      [certs] Generating "apiserver-etcd-client" certificate and key
      [certs] Generating "sa" key and public key
      #生成配置文件
      [kubeconfig] Using kubeconfig folder "/etc/kubernetes"
      [kubeconfig] Writing "admin.conf" kubeconfig file
      [kubeconfig] Writing "kubelet.conf" kubeconfig file
      [kubeconfig] Writing "controller-manager.conf" kubeconfig file
      [kubeconfig] Writing "scheduler.conf" kubeconfig file
      [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
      [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
      #启动kubelet
      [kubelet-start] Starting the kubelet
      #进入静态目录启动相关容器， 放在manifests这里的都是静态容器
      [control-plane] Using manifest folder "/etc/kubernetes/manifests"
      [control-plane] Creating static Pod manifest for "kube-apiserver"
      [control-plane] Creating static Pod manifest for "kube-controller-manager"
      [control-plane] Creating static Pod manifest for "kube-scheduler"
      [etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
      [wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
      [kubelet-check] Initial timeout of 40s passed.
      [apiclient] All control plane components are healthy after 85.502557 seconds
      [upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
      [kubelet] Creating a ConfigMap "kubelet-config-1.21" in namespace kube-system with the configuration for the kubelets in the cluster
      [upload-certs] Skipping phase. Please see --upload-certs
      [mark-control-plane] Marking the node k8snode1 as control-plane by adding the labels: [node-role.kubernetes.io/master(deprecated) node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
      [mark-control-plane] Marking the node k8snode1 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
      [bootstrap-token] Using token: 3iwxdt.2s17bwrc5jtcokqk
      #进行认证
      [bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
      [bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
      [bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
      [bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
      [bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
      [bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
      [kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
      #启动俩dns和proxy容器
      [addons] Applied essential addon: CoreDNS
      [addons] Applied essential addon: kube-proxy
      #安装完成
      Your Kubernetes control-plane has initialized successfully!
      
      #根据提示需要执行的三条命令
      To start using your cluster, you need to run the following as a regular user:
      
        mkdir -p $HOME/.kube
        sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        sudo chown $(id -u):$(id -g) $HOME/.kube/config
      
      Alternatively, if you are the root user, you can run:
      
        export KUBECONFIG=/etc/kubernetes/admin.conf
      
      You should now deploy a pod network to the cluster.
      Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
        https://kubernetes.io/docs/concepts/cluster-administration/addons/
      
      Then you can join any number of worker nodes by running the following on each as root:
      
      #添加集群的方法
      kubeadm join 192.168.10.185:6443 --token 3iwxdt.2s17bwrc5jtcokqk \
      	--discovery-token-ca-cert-hash sha256:34fbd3ea8f9157e36b4794ef56692b9dd2ad3fa18c97dc28622fa80186a1fbe0 
      
      
      #最后查看容器是否启动成功
      [root@k8snode1 k8s]# docker ps -a
      CONTAINER ID        IMAGE                    COMMAND                  CREATED             STATUS              PORTS               NAMES
      dfeb2c5173bb        38ddd85fe90e             "/usr/local/bin/kube…"   3 minutes ago       Up 3 minutes                            k8s_kube-proxy_kube-proxy-c4m82_kube-system_1428a2f0-4f90-4619-82b2-484ad5b4a7c2_0
      38af914df1ae        k8s.gcr.io/pause:3.4.1   "/pause"                 3 minutes ago       Up 3 minutes                            k8s_POD_kube-proxy-c4m82_kube-system_1428a2f0-4f90-4619-82b2-484ad5b4a7c2_0
      69444092c2ee        4d217480042e             "kube-apiserver --ad…"   3 minutes ago       Up 3 minutes                            k8s_kube-apiserver_kube-apiserver-k8snode1_kube-system_45b9c9571b561573a462cbbb349ad36e_0
      797899d7a2eb        k8s.gcr.io/pause:3.4.1   "/pause"                 3 minutes ago       Up 3 minutes                            k8s_POD_kube-apiserver-k8snode1_kube-system_45b9c9571b561573a462cbbb349ad36e_0
      98f6a023eee9        0369cf4303ff             "etcd --advertise-cl…"   3 minutes ago       Up 3 minutes                            k8s_etcd_etcd-k8snode1_kube-system_5d52c7d516d5438ca27bbb218374c99c_0
      f280af565b21        k8s.gcr.io/pause:3.4.1   "/pause"                 3 minutes ago       Up 3 minutes                            k8s_POD_etcd-k8snode1_kube-system_5d52c7d516d5438ca27bbb218374c99c_0
      c9e20651acf9        62ad3129eca8             "kube-scheduler --au…"   4 minutes ago       Up 4 minutes                            k8s_kube-scheduler_kube-scheduler-k8snode1_kube-system_99bd4c84a25bacfce7bf14963cb53e93_0
      7b0a4174d3df        k8s.gcr.io/pause:3.4.1   "/pause"                 4 minutes ago       Up 4 minutes                            k8s_POD_kube-scheduler-k8snode1_kube-system_99bd4c84a25bacfce7bf14963cb53e93_0
      384ff6b9b761        09708983cc37             "kube-controller-man…"   4 minutes ago       Up 4 minutes                            k8s_kube-controller-manager_kube-controller-manager-k8snode1_kube-system_52ccbd93e40a9e1fa58a4e49843fa785_0
      b9ef0d9c7b35        k8s.gcr.io/pause:3.4.1   "/pause"                 4 minutes ago       Up 4 minutes                            k8s_POD_kube-controller-manager-k8snode1_kube-system_52ccbd93e40a9e1fa58a4e49843fa785_0
      
      #查看集群节点
      [root@k8snode1 k8s]# kubectl get nodes
      NAME       STATUS     ROLES                  AGE    VERSION
      k8snode1   NotReady   control-plane,master   8m8s   v1.21.0
      ```

   3. 根据上面的提示，执行三条命令

      ```
      [root@k8snode1 k8s]# mkdir -p $HOME/.kube
      [root@k8snode1 k8s]# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      [root@k8snode1 k8s]# sudo chown $(id -u):$(id -g) $HOME/.kube/config
      [root@k8snode1 k8s]# cat $HOME/.kube/config
      ```

   4. 将node1和node2加入到集群里

      ```
      #node2
      [root@k8snode2 k8simages]# kubeadm join 192.168.10.185:6443 --token 3iwxdt.2s17bwrc5jtcokqk --discovery-token-ca-cert-hash sha256:34fbd3ea8f9157e36b4794ef56692b9dd2ad3fa18c97dc28622fa80186a1fbe0 
      [preflight] Running pre-flight checks
      [preflight] Reading configuration from the cluster...
      [preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
      [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
      [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
      [kubelet-start] Starting the kubelet
      [kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
      
      This node has joined the cluster:
      * Certificate signing request was sent to apiserver and a response was received.
      * The Kubelet was informed of the new secure connection details.
      
      Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
      
      #node3
      [root@k8snode3 k8simages]# kubeadm join 192.168.10.185:6443 --token 3iwxdt.2s17bwrc5jtcokqk --discovery-token-ca-cert-hash sha256:34fbd3ea8f9157e36b4794ef56692b9dd2ad3fa18c97dc28622fa80186a1fbe0 
      [preflight] Running pre-flight checks
      	[WARNING Service-Kubelet]: kubelet service is not enabled, please run 'systemctl enable kubelet.service'
      [preflight] Reading configuration from the cluster...
      [preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
      [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
      [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
      [kubelet-start] Starting the kubelet
      [kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
      
      This node has joined the cluster:
      * Certificate signing request was sent to apiserver and a response was received.
      * The Kubelet was informed of the new secure connection details.
      
      Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
      
      #查看集群节点
      [root@k8snode1 k8s]# kubectl get nodes
      NAME       STATUS     ROLES                  AGE   VERSION
      k8snode1   NotReady   control-plane,master   10m   v1.21.0
      k8snode2   NotReady   <none>                 51s   v1.21.0
      k8snode3   NotReady   <none>       
      ```

   5. 安装⽹络plugin，kubernetes⽀持多种类型⽹络插件，要求⽹络⽀持CNI插件即可，CNI是Container Network Interface，要求kubernetes的中pod⽹络访问⽅式，这样整个环境的网络就通了

      * node和node之间⽹络互通
      * pod和pod之间⽹络互通
      * node和pod之间⽹络互通

      不同的CNI plugin⽀持的特性有所差别。kubernetes⽀持多种开源的⽹络CNI插件，常⻅的有flannel、calico、canal、weave等.

      1. 在calico上下载最新的:https://docs.projectcalico.org/getting-started/kubernetes/quickstart

         ```
         # 根据k8s版本下载相应的版本的calico.yaml
         [root@k8snode1 k8s]# wget https://docs.projectcalico.org/v3.18/manifests/calico.yaml
         ```

      2. 安装

         ```
         [root@k8snode1 k8s]# kubectl apply -f calico.yaml
         configmap/calico-config created
         customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org configured
         customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org configured
         customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org configured
         customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org configured
         customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org configured
         customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org configured
         customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org configured
         customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org configured
         customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org configured
         customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org configured
         customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org configured
         customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org configured
         customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org configured
         customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org configured
         customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org configured
         clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
         clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
         clusterrole.rbac.authorization.k8s.io/calico-node created
         clusterrolebinding.rbac.authorization.k8s.io/calico-node created
         daemonset.apps/calico-node created
         serviceaccount/calico-node created
         deployment.apps/calico-kube-controllers created
         serviceaccount/calico-kube-controllers created
         Warning: policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
         poddisruptionbudget.policy/calico-kube-controllers created
         
         [root@k8snode1 k8s]# kubectl apply -f custom-resources.yaml 
         installation.operator.tigera.io/default created
         
         ```

      3. 如果报错，通过命令查看日志

         ```
         [root@k8snode1 k8s]# journalctl -f -u kubelet.service
         ```

      4. 查看 CoreDNS Pod 是否是否运行

         ```
         [root@k8snode1 k8s]# kubectl get pods --all-namespaces
         NAMESPACE         NAME                                       READY   STATUS    RESTARTS   AGE
         kube-system       calico-kube-controllers-6d8ccdbf46-s6qdx   1/1     Running   0          23m
         kube-system       calico-node-249tw                          1/1     Running   0          14m
         kube-system       calico-node-8rxs5                          1/1     Running   0          18m
         kube-system       calico-node-zs4hx                          1/1     Running   0          18m
         kube-system       coredns-558bd4d5db-56pr7                   1/1     Running   0          78m
         kube-system       coredns-558bd4d5db-wbgr6                   1/1     Running   0          78m
         kube-system       etcd-k8snode1                              1/1     Running   0          79m
         kube-system       kube-apiserver-k8snode1                    1/1     Running   0          79m
         kube-system       kube-controller-manager-k8snode1           1/1     Running   0          79m
         kube-system       kube-proxy-9tvkf                           1/1     Running   0          68m
         kube-system       kube-proxy-c4m82                           1/1     Running   0          78m
         kube-system       kube-proxy-r4xpx                           1/1     Running   0          69m
         kube-system       kube-scheduler-k8snode1                    1/1     Running   0          79m
         tigera-operator   tigera-operator-675ccbb69c-ptdmm           1/1     Running   0          51m
         
         ```

      5. 校验集群状态

         ```
         [root@k8snode1 ~]# kubectl get nodes
         NAME       STATUS   ROLES                  AGE   VERSION
         k8snode1   Ready    control-plane,master   19h   v1.21.0
         k8snode2   Ready    <none>                 19h   v1.21.0
         k8snode3   Ready    <none>                 19h   v1.21.0
         
         [root@k8snode1 ~]# kubectl get cs
         Warning: v1 ComponentStatus is deprecated in v1.19+
         NAME                 STATUS    MESSAGE             ERROR
         controller-manager   Healthy   ok                  
         scheduler            Healthy   ok                  
         etcd-0               Healthy   {"health":"true"}   
         
         
         [root@k8snode1 ~]# kubectl get pods -n kube-system
         NAME                                       READY   STATUS    RESTARTS   AGE
         calico-kube-controllers-6d8ccdbf46-s6qdx   1/1     Running   0          18h
         calico-node-249tw                          1/1     Running   0          18h
         calico-node-8rxs5                          1/1     Running   0          18h
         calico-node-zs4hx                          1/1     Running   0          18h
         #完成服务之间的域名解析
         coredns-558bd4d5db-56pr7                   1/1     Running   0          19h
         coredns-558bd4d5db-wbgr6                   1/1     Running   0          19h
         etcd-k8snode1                              1/1     Running   0          19h
         kube-apiserver-k8snode1                    1/1     Running   0          19h
         kube-controller-manager-k8snode1           1/1     Running   0          19h
         #kube-proxy  完成server的网络构建 讲容器的ip暴露给外面使用
         kube-proxy-9tvkf                           1/1     Running   0          19h
         kube-proxy-c4m82                           1/1     Running   0          19h
         kube-proxy-r4xpx                           1/1     Running   0          19h
         kube-scheduler-k8snode1                    1/1     Running   0          19h
         
         [root@k8snode1 ~]# kubectl get pods -n kube-system -o wide
         NAME                                       READY   STATUS    RESTARTS   AGE   IP               NODE       NOMINATED NODE   READINESS GATES
         calico-kube-controllers-6d8ccdbf46-s6qdx   1/1     Running   0          18h   172.16.98.194    k8snode3   <none>           <none>
         calico-node-249tw                          1/1     Running   0          18h   192.168.10.187   k8snode3   <none>           <none>
         calico-node-8rxs5                          1/1     Running   0          18h   192.168.10.186   k8snode2   <none>           <none>
         calico-node-zs4hx                          1/1     Running   0          18h   192.168.10.185   k8snode1   <none>           <none>
         coredns-558bd4d5db-56pr7                   1/1     Running   0          19h   172.16.98.193    k8snode3   <none>           <none>
         coredns-558bd4d5db-wbgr6                   1/1     Running   0          19h   172.16.98.195    k8snode3   <none>           <none>
         etcd-k8snode1                              1/1     Running   0          19h   192.168.10.185   k8snode1   <none>           <none>
         kube-apiserver-k8snode1                    1/1     Running   0          19h   192.168.10.185   k8snode1   <none>           <none>
         kube-controller-manager-k8snode1           1/1     Running   0          19h   192.168.10.185   k8snode1   <none>           <none>
         kube-proxy-9tvkf                           1/1     Running   0          19h   192.168.10.187   k8snode3   <none>           <none>
         kube-proxy-c4m82                           1/1     Running   0          19h   192.168.10.185   k8snode1   <none>           <none>
         kube-proxy-r4xpx                           1/1     Running   0          19h   192.168.10.186   k8snode2   <none>           <none>
         kube-scheduler-k8snode1                    1/1     Running   0          19h   192.168.10.185   k8snode1   <none>           <none>
         
         # 这两个会在所有节点运行
         [root@k8snode1 ~]# kubectl get daemonsets -n kube-system
         NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
         calico-node   3         3         3       3            3           kubernetes.io/os=linux   18h
         kube-proxy    3         3         3       3            3           kubernetes.io/os=linux   19h
         
         [root@k8snode1 ~]# kubectl get deployments -n kube-system
         NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
         calico-kube-controllers   1/1     1            1           18h
         coredns                   2/2     2            2           19h
         
         ```

# 高可用配置

参考官网：https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/ha-topology/



# dashboard图形界面

参考文档：https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/

1. 安装，下载yam文件

   ```
   [root@k8snode1 k8s]# wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
   ```

2. 提交资源应用，也可以将recommended.yaml 中的type改为NodePort

   ```
   [root@k8snode1 k8s]# kubectl apply -f recommended.yaml 
   namespace/kubernetes-dashboard created
   serviceaccount/kubernetes-dashboard created
   service/kubernetes-dashboard created
   secret/kubernetes-dashboard-certs created
   secret/kubernetes-dashboard-csrf created
   secret/kubernetes-dashboard-key-holder created
   configmap/kubernetes-dashboard-settings created
   role.rbac.authorization.k8s.io/kubernetes-dashboard created
   clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
   rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
   clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
   deployment.apps/kubernetes-dashboard created
   service/dashboard-metrics-scraper created
   deployment.apps/dashboard-metrics-scraper created
   ```

3. 查看是否部署成功

   ```
   [root@k8snode1 k8s]# kubectl get deployments.apps --all-namespaces 
   NAMESPACE              NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
   default                nginxdemo1                  4/4     4            4           17h
   kube-system            calico-kube-controllers     1/1     1            1           41h
   kube-system            coredns                     2/2     2            2           42h
   kubernetes-dashboard   dashboard-metrics-scraper   1/1     1            1           92s
   kubernetes-dashboard   kubernetes-dashboard        1/1     1            1           92s
   tigera-operator        tigera-operator             1/1     1            1           41h
   
   [root@k8snode1 k8s]# kubectl get deployments.apps -n kubernetes-dashboard
   NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
   dashboard-metrics-scraper   1/1     1            1           111s
   kubernetes-dashboard        1/1     1            1           111s
   
   [root@k8snode1 k8s]# kubectl describe deployments.apps -n kubernetes-dashboard
   ```

4. 查看services,看到services是ClusterIP，需要改成NodePort

   ```
   [root@k8snode1 k8s]# kubectl get services -n kubernetes-dashboard
   NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
   dashboard-metrics-scraper   ClusterIP   10.100.64.124    <none>        8000/TCP   2m44s
   kubernetes-dashboard        ClusterIP   10.110.177.184   <none>        443/TCP    2m44s
   
   ```

5. 修改type为NodePort

   ```
   [root@k8snode1 k8s]# kubectl edit services kubernetes-dashboard -n kubernetes-dashboard
   # Please edit the object below. Lines beginning with a '#' will be ignored,
   # and an empty file will abort the edit. If an error occurs while saving this file will be
   # reopened with the relevant failures.
   #
   apiVersion: v1
   kind: Service
   metadata:
     annotations:
       kubectl.kubernetes.io/last-applied-configuration: |
         {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"k8s-app":"kubernetes-dashboard"},"name":"kubernetes-dashboard","namespace":"kubernetes-dashboard"},"spec":{"ports":[{"port":443,"targetPort":8443}],"selector":{"k8s-app":"kubernetes-dashboard"}}}
     creationTimestamp: "2021-04-19T09:03:52Z"
     labels:
       k8s-app: kubernetes-dashboard
     name: kubernetes-dashboard
     namespace: kubernetes-dashboard
     resourceVersion: "363254"
     uid: b7046bca-60d2-49d5-8e86-a18ccb7f03a4
   spec:
     clusterIP: 10.110.177.184
     clusterIPs:
     - 10.110.177.184
     ipFamilies:
     - IPv4
     ipFamilyPolicy: SingleStack
     ports:
     - port: 443
       protocol: TCP
       targetPort: 8443
     selector:
       k8s-app: kubernetes-dashboard
     sessionAffinity: None
     type: ClusterIP  #改为NodePort
   status:
     loadBalancer: {}
   
   #查看是否成功
   [root@k8snode1 k8s]# kubectl get services -n kubernetes-dashboard
   NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
   dashboard-metrics-scraper   ClusterIP   10.100.64.124    <none>        8000/TCP        5m37s
   kubernetes-dashboard        NodePort    10.110.177.184   <none>        443:32520/TCP   5m37s
   
   #进行访问
   [root@k8snode1 k8s]# curl https://192.168.10.185:32520
   ```

6. dashboard默认是没有登录权限的需要创建一个超级管理员来登录

   参考文档：https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md

   ```
   [root@k8snode1 k8s]# kubectl get clusterrole
   NAME                                                                   CREATED AT
   admin                                                                  2021-04-17T14:50:23Z
   calico-kube-controllers                                                2021-04-17T15:45:51Z
   calico-node                                                            2021-04-17T15:45:51Z
   # 这个就是管理员角色
   cluster-admin                                                          2021-04-17T14:50:23Z
   edit                                                                   2021-04-17T14:50:23Z
   kubeadm:get-nodes                                                      2021-04-17T14:50:25Z
   kubernetes-dashboard                                                   2021-04-19T09:03:52Z
   system:aggregate-to-admin                                              2021-04-17T14:50:23Z
   system:aggregate-to-edit                                               2021-04-17T14:50:23Z
   system:aggregate-to-view                                               2021-04-17T14:50:23Z
   system:auth-delegator                                                  2021-04-17T14:50:23Z
   system:basic-user                                                      2021-04-17T14:50:23Z
   system:certificates.k8s.io:certificatesigningrequests:nodeclient       2021-04-17T14:50:23Z
   system:certificates.k8s.io:certificatesigningrequests:selfnodeclient   2021-04-17T14:50:23Z
   system:certificates.k8s.io:kube-apiserver-client-approver              2021-04-17T14:50:24Z
   system:certificates.k8s.io:kube-apiserver-client-kubelet-approver      2021-04-17T14:50:24Z
   system:certificates.k8s.io:kubelet-serving-approver                    2021-04-17T14:50:24Z
   system:certificates.k8s.io:legacy-unknown-approver                     2021-04-17T14:50:23Z
   system:controller:attachdetach-controller                              2021-04-17T14:50:24Z
   system:controller:certificate-controller                               2021-04-17T14:50:24Z
   system:controller:clusterrole-aggregation-controller                   2021-04-17T14:50:24Z
   system:controller:cronjob-controller                                   2021-04-17T14:50:24Z
   system:controller:daemon-set-controller                                2021-04-17T14:50:24Z
   system:controller:deployment-controller                                2021-04-17T14:50:24Z
   system:controller:disruption-controller                                2021-04-17T14:50:24Z
   system:controller:endpoint-controller                                  2021-04-17T14:50:24Z
   system:controller:endpointslice-controller                             2021-04-17T14:50:24Z
   system:controller:endpointslicemirroring-controller                    2021-04-17T14:50:24Z
   system:controller:ephemeral-volume-controller                          2021-04-17T14:50:24Z
   system:controller:expand-controller                                    2021-04-17T14:50:24Z
   system:controller:generic-garbage-collector                            2021-04-17T14:50:24Z
   system:controller:horizontal-pod-autoscaler                            2021-04-17T14:50:24Z
   system:controller:job-controller                                       2021-04-17T14:50:24Z
   system:controller:namespace-controller                                 2021-04-17T14:50:24Z
   system:controller:node-controller                                      2021-04-17T14:50:24Z
   system:controller:persistent-volume-binder                             2021-04-17T14:50:24Z
   system:controller:pod-garbage-collector                                2021-04-17T14:50:24Z
   system:controller:pv-protection-controller                             2021-04-17T14:50:24Z
   system:controller:pvc-protection-controller                            2021-04-17T14:50:24Z
   system:controller:replicaset-controller                                2021-04-17T14:50:24Z
   system:controller:replication-controller                               2021-04-17T14:50:24Z
   system:controller:resourcequota-controller                             2021-04-17T14:50:24Z
   system:controller:root-ca-cert-publisher                               2021-04-17T14:50:24Z
   system:controller:route-controller                                     2021-04-17T14:50:24Z
   system:controller:service-account-controller                           2021-04-17T14:50:24Z
   system:controller:service-controller                                   2021-04-17T14:50:24Z
   system:controller:statefulset-controller                               2021-04-17T14:50:24Z
   system:controller:ttl-after-finished-controller                        2021-04-17T14:50:24Z
   system:controller:ttl-controller                                       2021-04-17T14:50:24Z
   system:coredns                                                         2021-04-17T14:50:25Z
   system:discovery                                                       2021-04-17T14:50:23Z
   system:heapster                                                        2021-04-17T14:50:23Z
   system:kube-aggregator                                                 2021-04-17T14:50:23Z
   system:kube-controller-manager                                         2021-04-17T14:50:23Z
   system:kube-dns                                                        2021-04-17T14:50:23Z
   system:kube-scheduler                                                  2021-04-17T14:50:24Z
   system:kubelet-api-admin                                               2021-04-17T14:50:23Z
   system:monitoring                                                      2021-04-17T14:50:23Z
   system:node                                                            2021-04-17T14:50:23Z
   system:node-bootstrapper                                               2021-04-17T14:50:23Z
   system:node-problem-detector                                           2021-04-17T14:50:23Z
   system:node-proxier                                                    2021-04-17T14:50:24Z
   system:persistent-volume-provisioner                                   2021-04-17T14:50:23Z
   system:public-info-viewer                                              2021-04-17T14:50:23Z
   system:service-account-issuer-discovery                                2021-04-17T14:50:24Z
   system:volume-scheduler                                                2021-04-17T14:50:23Z
   tigera-operator                                                        2021-04-17T15:18:33Z
   view                                                                   2021-04-17T14:50:23Z
   
   ```

7. 创建一个yaml文件

   ```
   [root@k8snode1 k8s]# vim dashboard-adminuser.yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: admin-user #用户名
     namespace: kubernetes-dashboard
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: admin-user  #用户名
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: ClusterRole
     name: cluster-admin #关联的角色
   subjects:
   - kind: ServiceAccount
     name: admin-user
     namespace: kubernetes-dashboard
   ```

8. 提交文件

   ```
   [root@k8snode1 k8s]# kubectl apply -f dashboard-adminuser.yaml 
   serviceaccount/admin-user created
   clusterrolebinding.rbac.authorization.k8s.io/admin-user created
   
   #查看是否创建成功
   [root@k8snode1 k8s]# kubectl get secrets -n kubernetes-dashboard 
   NAME                               TYPE                                  DATA   AGE
   admin-user-token-srljt             kubernetes.io/service-account-token   3      23s  #已经创建成功
   default-token-p9r6r                kubernetes.io/service-account-token   3      13m
   kubernetes-dashboard-certs         Opaque                                0      13m
   kubernetes-dashboard-csrf          Opaque                                1      13m
   kubernetes-dashboard-key-holder    Opaque                                2      13m
   kubernetes-dashboard-token-gqk79   kubernetes.io/service-account-token   3      13m
   
   ```

9. 打开这个权限文件

   ```
   [root@k8snode1 k8s]# kubectl get secrets -n kubernetes-dashboard admin-user-token-srljt -o yaml
   apiVersion: v1
   data:
     ca.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM1ekNDQWMrZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJeE1EUXhOekUwTkRnMU5sb1hEVE14TURReE5URTBORGcxTmxvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTDhrCmJsczdiRUxUdnJPTzlMbWJpZlNNandkL0pqMVZkU1pMTkU1VFJNanQ5YU9QQnp3eVdnTkowUmRTc3djamlKOFEKVnhiVHJvZ2RwMWhXVG5Pb3FMdDlwMWFtSGMzSXNlT096RGd5L1EwOU5TQ0Qwa2tSVXB0MW80em1LZCtxUDJ6NwoyMklkcTRxZE9wMDhGN2l2SUFRRHZhczR6eTBKU3VoYk9XNzN5WWhUdTZnK3Rsa2xJWldhWlI5dVowTjZOSzhlCkptWEZVY2pWV1VaSnRuNzhJM2Q3SWlmTTNQWmVzb1BIMVhnd2xVWHpQN2hUbXk2dHhpSGUvVTdEUGdBWU5HcGsKR0xPa0NKVUd6L1hYTEtDY2h6S0Z6ZkVIYWNITE5VRjFmd0tQdHNxdE8rbkJMUm8wMnluUGNXSlIzQ05OMGV1TQp6cFp6L2owZHBNQWsvSkgySEY4Q0F3RUFBYU5DTUVBd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZHNncySGpUWGorSHhLUVBHcGUzdDJVZHRRMlBNQTBHQ1NxR1NJYjMKRFFFQkN3VUFBNElCQVFBbGpyanFPdEo0ZDh6djQ0d1NCNGJWbDJUc3liQ1oySy8yQ2ZtRzIrYU9uM2M5N2RwRwpTYzRqQ3dKRmQ3R3I1NUJPUW1UU01qYkVBMjZ2NDRQNWJhcjJUSlVjY01DS05QSWNwVHB3eE9Td2FFdm0xN2JxCnZ6WThXMWNHNFJyVjBrM09qQnVNN09rSVBIRkJOM1E2cXVGam56dURpWHpTVFRhSG4zQjZPSHJmK1FiZ0NuSTgKdFp1UHRNOGVoY1Q5ZTFmNjF4Q1pHTHRYZUxEdFJiclhaeFh4R2xKOTJhYU8vbDVzYm5lV2thdkVNaTVZbzJ4ZgovQ0h0MmJJbWVXSTBaK21FZjBPT0VHaVo3S1Qyc0xLdzdPSjM4OWttSFFZZDhIcTJGRXBBMXdTVGgwdjZ2dENCCmJwK3FhSkVQRkNxQjVvcGorcktUYllDa0d0TWptdmV4NE1KcgotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
     namespace: a3ViZXJuZXRlcy1kYXNoYm9hcmQ=
     # 看到一个base64加密的token
     token:   ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNkltTmFNVVZ5TTBSa1JqRTNhamRLT1ZGaFprVjRXV2xEV2paUFEzRjFjemRHT0ZkWk5GVm1UMVZzYTJjaWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUpyZFdKbGNtNWxkR1Z6TFdSaGMyaGliMkZ5WkNJc0ltdDFZbVZ5Ym1WMFpYTXVhVzh2YzJWeWRtbGpaV0ZqWTI5MWJuUXZjMlZqY21WMExtNWhiV1VpT2lKaFpHMXBiaTExYzJWeUxYUnZhMlZ1TFhOeWJHcDBJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5elpYSjJhV05sTFdGalkyOTFiblF1Ym1GdFpTSTZJbUZrYldsdUxYVnpaWElpTENKcmRXSmxjbTVsZEdWekxtbHZMM05sY25acFkyVmhZMk52ZFc1MEwzTmxjblpwWTJVdFlXTmpiM1Z1ZEM1MWFXUWlPaUl4WkdWalpHRmpaaTFpTXpVNExUUmtZakF0WW1SbU5pMHpNekE1TVRrMVptVmhObVFpTENKemRXSWlPaUp6ZVhOMFpXMDZjMlZ5ZG1salpXRmpZMjkxYm5RNmEzVmlaWEp1WlhSbGN5MWtZWE5vWW05aGNtUTZZV1J0YVc0dGRYTmxjaUo5LlBHV1ZrU3NXWHpUUFFIU2Y3QlBPRDM4WDJoTWZDUG1LSUpJaFJGckxTX3otSDdENEdyNC1JT25xck5WV1pvaVZlcnhxRHBnNFBKQVZqcDdQQkw0WGtCc19nQUI2allUV0ZuVzJrLVZqQUFCSS1SU09hYXhmZ0tURWJMd0xnZnkzWXJqeDBRdjBtU29NSml1UjU3UHNMckE1V0FCZkdQZy1fQ2FuMEdIR2NFbk9PYm83bmx2bU1tWnpuRjd0OTdZTnBpc0NlSnhqUGlVbzdodGpSbC1KY1UzV2VRYjFoQks3V2twd3JQRE1CUk1sTndldVpLLVlaXzFJWHNUS1AxdlQwc2pka0ptOTJ0NF9mb2JIQUpGYl9DLURZUkhSbWgwcWR0emlBX3ZsMkhmLTZIamktUWFxdjZubnVncXJpNDV5cWxDYllxS1dYNVpOaEZ3RWRyU3JJZw==
   kind: Secret
   metadata:
     annotations:
       kubernetes.io/service-account.name: admin-user
       kubernetes.io/service-account.uid: 1decdacf-b358-4db0-bdf6-3309195fea6d
     creationTimestamp: "2021-04-19T09:16:58Z"
     name: admin-user-token-srljt
     namespace: kubernetes-dashboard
     resourceVersion: "365207"
     uid: acdffabb-6c20-480a-ac1d-9fa6f53e50c0
   type: kubernetes.io/service-account-token
   
   ```

10. 解密base64,得到的token就是登录token了，就可以在页面登录了

    ```
    [root@k8snode1 k8s]# echo 'ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNkltTmFNVVZ5TTBSa1JqRTNhamRLT1ZGaFprVjRXV2xEV2paUFEzRjFjemRHT0ZkWk5GVm1UMVZzYTJjaWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUpyZFdKbGNtNWxkR1Z6TFdSaGMyaGliMkZ5WkNJc0ltdDFZbVZ5Ym1WMFpYTXVhVzh2YzJWeWRtbGpaV0ZqWTI5MWJuUXZjMlZqY21WMExtNWhiV1VpT2lKaFpHMXBiaTExYzJWeUxYUnZhMlZ1TFhOeWJHcDBJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5elpYSjJhV05sTFdGalkyOTFiblF1Ym1GdFpTSTZJbUZrYldsdUxYVnpaWElpTENKcmRXSmxjbTVsZEdWekxtbHZMM05sY25acFkyVmhZMk52ZFc1MEwzTmxjblpwWTJVdFlXTmpiM1Z1ZEM1MWFXUWlPaUl4WkdWalpHRmpaaTFpTXpVNExUUmtZakF0WW1SbU5pMHpNekE1TVRrMVptVmhObVFpTENKemRXSWlPaUp6ZVhOMFpXMDZjMlZ5ZG1salpXRmpZMjkxYm5RNmEzVmlaWEp1WlhSbGN5MWtZWE5vWW05aGNtUTZZV1J0YVc0dGRYTmxjaUo5LlBHV1ZrU3NXWHpUUFFIU2Y3QlBPRDM4WDJoTWZDUG1LSUpJaFJGckxTX3otSDdENEdyNC1JT25xck5WV1pvaVZlcnhxRHBnNFBKQVZqcDdQQkw0WGtCc19nQUI2allUV0ZuVzJrLVZqQUFCSS1SU09hYXhmZ0tURWJMd0xnZnkzWXJqeDBRdjBtU29NSml1UjU3UHNMckE1V0FCZkdQZy1fQ2FuMEdIR2NFbk9PYm83bmx2bU1tWnpuRjd0OTdZTnBpc0NlSnhqUGlVbzdodGpSbC1KY1UzV2VRYjFoQks3V2twd3JQRE1CUk1sTndldVpLLVlaXzFJWHNUS1AxdlQwc2pka0ptOTJ0NF9mb2JIQUpGYl9DLURZUkhSbWgwcWR0emlBX3ZsMkhmLTZIamktUWFxdjZubnVncXJpNDV5cWxDYllxS1dYNVpOaEZ3RWRyU3JJZw==' | base64 -d
    eyJhbGciOiJSUzI1NiIsImtpZCI6ImNaMUVyM0RkRjE3ajdKOVFhZkV4WWlDWjZPQ3F1czdGOFdZNFVmT1Vsa2cifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLXNybGp0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiIxZGVjZGFjZi1iMzU4LTRkYjAtYmRmNi0zMzA5MTk1ZmVhNmQiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.PGWVkSsWXzTPQHSf7BPOD38X2hMfCPmKIJIhRFrLS_z-H7D4Gr4-IOnqrNVWZoiVerxqDpg4PJAVjp7PBL4XkBs_gAB6jYTWFnW2k-VjAABI-RSOaaxfgKTEbLwLgfy3Yrjx0Qv0mSoMJiuR57PsLrA5WABfGPg-_Can0GHGcEnOObo7nlvmMmZznF7t97YNpisCeJxjPiUo7htjRl-JcU3WeQb1hBK7WkpwrPDMBRMlNweuZK-YZ_1IXsTKP1vT0sjdkJm92t4_fobHAJFb_C-DYRHRmh0qdtziA_vl2Hf-6Hji-Qaqv6nnugqri45yqlCbYqKWX5ZNhFwEdrSrIg
    ```

    





# 问题

1. kubectl get cs异常

   ```
   # 现象
   [root@k8snode1 ~]# kubectl get cs
   Warning: v1 ComponentStatus is deprecated in v1.19+
   NAME                 STATUS      MESSAGE                                                                                       ERROR
   controller-manager   Unhealthy   Get "http://127.0.0.1:10252/healthz": dial tcp 127.0.0.1:10252: connect: connection refused   
   scheduler            Unhealthy   Get "http://127.0.0.1:10251/healthz": dial tcp 127.0.0.1:10251: connect: connection refused   
   etcd-0               Healthy     {"health":"true"}                                         
   
   
   #查看组件运行是否正常
   [root@k8snode1 ~]# kubectl get pods -A
   NAMESPACE         NAME                                       READY   STATUS    RESTARTS   AGE
   kube-system       calico-kube-controllers-6d8ccdbf46-s6qdx   1/1     Running   0          18h
   kube-system       calico-node-249tw                          1/1     Running   0          18h
   kube-system       calico-node-8rxs5                          1/1     Running   0          18h
   kube-system       calico-node-zs4hx                          1/1     Running   0          18h
   kube-system       coredns-558bd4d5db-56pr7                   1/1     Running   0          19h
   kube-system       coredns-558bd4d5db-wbgr6                   1/1     Running   0          19h
   kube-system       etcd-k8snode1                              1/1     Running   0          19h
   kube-system       kube-apiserver-k8snode1                    1/1     Running   0          19h
   kube-system       kube-controller-manager-k8snode1           1/1     Running   0          19h
   kube-system       kube-proxy-9tvkf                           1/1     Running   0          19h
   kube-system       kube-proxy-c4m82                           1/1     Running   0          19h
   kube-system       kube-proxy-r4xpx                           1/1     Running   0          19h
   kube-system       kube-scheduler-k8snode1                    1/1     Running   0          19h
   tigera-operator   tigera-operator-675ccbb69c-ptdmm           1/1     Running   0          18h
   
   #查看本地端口 这俩不存在
   [root@k8snode1 ~]# netstat -tlunp
   Active Internet connections (only servers)
   Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
   tcp        0      0 127.0.0.1:37088         0.0.0.0:*               LISTEN      29488/kubelet       
   tcp        0      0 127.0.0.1:10248         0.0.0.0:*               LISTEN      29488/kubelet       
   tcp        0      0 127.0.0.1:10249         0.0.0.0:*               LISTEN      29854/kube-proxy    
   tcp        0      0 127.0.0.1:9099          0.0.0.0:*               LISTEN      19658/calico-node   
   tcp        0      0 192.168.10.185:2379     0.0.0.0:*               LISTEN      29215/etcd          
   tcp        0      0 127.0.0.1:2379          0.0.0.0:*               LISTEN      29215/etcd          
   tcp        0      0 192.168.10.185:2380     0.0.0.0:*               LISTEN      29215/etcd          
   tcp        0      0 127.0.0.1:2381          0.0.0.0:*               LISTEN      29215/etcd          
   tcp        0      0 127.0.0.1:10257         0.0.0.0:*               LISTEN      28913/kube-controll 
   tcp        0      0 0.0.0.0:179             0.0.0.0:*               LISTEN      19773/bird          
   tcp        0      0 127.0.0.1:10259         0.0.0.0:*               LISTEN      29060/kube-schedule 
   tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      1045/sshd           
   tcp6       0      0 :::10250                :::*                    LISTEN      29488/kubelet       
   tcp6       0      0 :::6443                 :::*                    LISTEN      29372/kube-apiserve 
   tcp6       0      0 :::10256                :::*                    LISTEN      29854/kube-proxy    
   tcp6       0      0 :::22                   :::*                    LISTEN      1045/sshd           
   udp        0      0 127.0.0.1:323           0.0.0.0:*                           793/chronyd         
   udp6       0      0 ::1:323                 :::*                                793/chronyd         
   
   #确认kube-scheduler和kube-controller-manager组件配置是否禁用了非安全端口
   找到kube-controller-manager.yaml查看是否禁用了端口
   [root@k8snode1 ~]# vim /etc/kubernetes/manifests/kube-controller-manager.yaml
   apiVersion: v1
   kind: Pod
   metadata:
     creationTimestamp: null
     labels:
       component: kube-controller-manager
       tier: control-plane
     name: kube-controller-manager
     namespace: kube-system
   spec:
     containers:
     - command:
       - kube-controller-manager
       - --allocate-node-cidrs=true
       - --authentication-kubeconfig=/etc/kubernetes/controller-manager.conf
       - --authorization-kubeconfig=/etc/kubernetes/controller-manager.conf
       - --bind-address=127.0.0.1
       - --client-ca-file=/etc/kubernetes/pki/ca.crt
       - --cluster-cidr=172.16.0.0/16
       - --cluster-name=kubernetes
       - --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
       - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
       - --controllers=*,bootstrapsigner,tokencleaner
       - --kubeconfig=/etc/kubernetes/controller-manager.conf
       - --leader-elect=true
   #    - --port=0   #注释这一行
       - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
       - --root-ca-file=/etc/kubernetes/pki/ca.crt
       - --service-account-private-key-file=/etc/kubernetes/pki/sa.key
       - --service-cluster-ip-range=10.96.0.0/12
       - --use-service-account-credentials=true
       image: k8s.gcr.io/kube-controller-manager:v1.21.0
       imagePullPolicy: IfNotPresent
       livenessProbe:
         failureThreshold: 8
   
   #同样找到文件kube-scheduler.yam注释掉prot这一行 
   [root@k8snode1 ~]# vim /etc/kubernetes/manifests/kube-scheduler.yam
   apiVersion: v1
   kind: Pod
   metadata:
     creationTimestamp: null
     labels:
       component: kube-scheduler
       tier: control-plane
     name: kube-scheduler
     namespace: kube-system
   spec:
     containers:
     - command:
       - kube-scheduler
       - --authentication-kubeconfig=/etc/kubernetes/scheduler.conf
       - --authorization-kubeconfig=/etc/kubernetes/scheduler.conf
       - --bind-address=127.0.0.1
       - --kubeconfig=/etc/kubernetes/scheduler.conf
       - --leader-elect=true
       #- --port=0
       image: k8s.gcr.io/kube-scheduler:v1.21.0
       imagePullPolicy: IfNotPresent
       livenessProbe:
         failureThreshold: 8
         httpGet:
           host: 127.0.0.1
           path: /healthz
           port: 10259
           scheme: HTTPS
         initialDelaySeconds: 10
         periodSeconds: 10
         timeoutSeconds: 15
       name: kube-scheduler
       resources:
         requests:
           cpu: 100m
   
   #最后重启kubelet 查看状态
   [root@k8snode1 ~]# systemctl restart kubelet
   [root@k8snode1 ~]# kubectl get cs
   Warning: v1 ComponentStatus is deprecated in v1.19+
   NAME                 STATUS    MESSAGE             ERROR
   controller-manager   Healthy   ok                  
   scheduler            Healthy   ok                  
   etcd-0               Healthy   {"health":"true"}   
   
   ```

#  kbuectl命令自动补全功能

1. 查询所有api命令

   ```
   [root@k8snode1 ~]# kubectl api-resources
   ```

2. 安装命令包

   ```
   [root@k8snode1 ~]# yum -y install bash-completion
   [root@k8snode1 ~]# source /usr/share/bash-completion/bash_completion
   ```

3. 生成命令提示

   ```
   [root@k8snode1 ~]# kubectl completion bash
   ```

4. 将命令提示保存到文件

   ```
   [root@k8snode1 ~]# kubectl completion bash >/etc/kubernetes/kubectl.sh
   ```

5. 将文件写入到环境变量里

   ```
   [root@k8snode1 ~]# vim /root/.bashrc
   # .bashrc
   
   # User specific aliases and functions
   
   alias rm='rm -i'
   alias cp='cp -i'
   alias mv='mv -i'
   
   # Source global definitions
   if [ -f /etc/bashrc ]; then
           . /etc/bashrc
   fi
   source /etc/kubernetes/kubectl.sh
   
   [root@k8snode1 ~]# source /etc/kubernetes/kubectl.sh
   ```

6. 验证是否补全成功 

   ```
   [root@k8snode1 ~]# kubectl get co
   componentstatuses         configmaps                controllerrevisions.apps
   ```

7. 除了⽀持命令⾏补全之外，kubectl还⽀持命令简写，如下是⼀些常⻅的命令⾏检测操作,更多通过kubectl api-resources命令获取，SHORTNAMES显示的是⼦命令中的简短⽤法。
   kubectl get componentstatuses，简写kubectl get cs获取组件状态
   kubectl get nodes，简写kubectl get no获取node节点列表
   kubectl get services，简写kubectl get svc获取服务列表
   kubectl get deployments,简写kubectl get deploy获取deployment列表
   kubectl get statefulsets,简写kubectl get sts获取有状态服务列表

   

   