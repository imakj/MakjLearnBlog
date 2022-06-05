# 环境准备

|   系统    |       IP       |  角色   | 主机名 |
| :-------: | :------------: | :-----: | :----: |
| centos7.9 | 192.168.10.180 | master1 | k8sbm1 |
| centos7.9 | 192.168.10.181 | master2 | k8sbm2 |
| centos7.9 | 192.168.10.182 | master3 | k8sbm3 |
| centos7.9 | 192.168.10.183 |  work1  | k8sbw1 |
| centos7.9 | 192.168.10.184 |  work1  | k8sbw2 |

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
   
   # 根据kubernetes对docker版本的兼容测试情况
   # yum install -y docker-ce-18.09.9 docker-ce-cli-19.03.9 containerd.io -y
   
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

# 下载二进制文件

1. 上传kubernetes-ha-kubeadm-master.zip到服务器/opt目录

   ```
   # /opt/
   # tar zxvf kubernetes-bin-1.14.0.tar.gz
   # mv kubernetes-bin-1.14.0 kubernetes
   ```

2. 所有主节点执行

   ```
   # echo "export PATH=/opt/kubernetes/master:$PATH" >> /etc/profile
   # source /etc/profile
   ```

3. 所有工作节点执行

   ```
   # echo "export PATH=/opt/kubernetes/master:$PATH" >> /etc/profile
   # source /etc/profile
   ```

4. 创建bin超链接,所有主节点执行

   ```
   # cd /opt/kubernetes/
   # ln -s master/ bin
   ```

   

5. 上传kubernetes-ha-binary-master.zip文件到服务器

   ```
   # /root/kubernetes-ha-binary
   # ll
   total 24
   drwxr-xr-x 2 root root    99 May 16  2020 addons
   drwxr-xr-x 2 root root   175 May 16  2020 configs
   drwxr-xr-x 2 root root    87 May 16  2020 docs
   -rwxr-xr-x 1 root root   746 May 16  2020 global-config.properties
   -rwxr-xr-x 1 root root  1909 May 16  2020 init.sh
   -rw-r--r-- 1 root root 11357 May 16  2020 LICENSE
   drwxr-xr-x 8 root root   145 May 16  2020 pki
   -rw-r--r-- 1 root root   945 May 16  2020 README.md
   drwxr-xr-x 2 root root   174 May 16  2020 services
   ```

6. #### 文件说明

   - **addons**

     > kubernetes的插件目录，包括calico、coredns、dashboard等。

   - **configs**

     > 这个目录比较 - 凌乱，包含了部署集群过程中用到的杂七杂八的配置文件、脚本文件等。

   - **pki**

     > 各个组件的认证授权相关证书配置。

   - **services**

     > 所有的kubernetes服务(service)配置文件。

   - **global-configs.properties**

     > 全局配置，包含各种易变的配置内容。

   - **init.sh**

     > 初始化脚本，配置好global-config之后，会自动生成所有配置文件。

7. 生成配置（所有节点）

   ```
   # pwd
   /root/kubernetes-ha-binary
   
   # vim global-config.properties
   #3个master节点的ip
   MASTER_0_IP=192.168.10.180
   MASTER_1_IP=192.168.10.181
   MASTER_2_IP=192.168.10.182
   
   #3个master节点的hostname
   MASTER_0_HOSTNAME=k8sbm1
   MASTER_1_HOSTNAME=k8sbm2
   MASTER_2_HOSTNAME=k8sbm3
   
   #api-server的高可用虚拟ip
   MASTER_VIP=192.168.10.185
   
   #keepalived用到的网卡接口名，一般是eth0
   VIP_IF=eth192
   
   #worker节点的ip列表
   WORKER_IPS=192.168.10.183,192.168.10.184
   
   #kubernetes服务ip网段
   SERVICE_CIDR=10.254.0.0/16
   
   #kubernetes的默认服务ip，一般是cidr的第一个
   KUBERNETES_SVC_IP=10.254.0.1
   
   #dns服务的ip地址，一般是cidr的第二个
   CLUSTER_DNS=10.254.0.2
   
   #pod网段
   POD_CIDR=172.22.0.0/16
   
   #NodePort的取值范围
   NODE_PORT_RANGE=8400-8900
   
   # 生成配置
   # ./init.sh
   
   # 查看生成的配置文件，确保脚本执行成功
   # find target/ -type f
   ```

# 开始安装

1.  安装cfssl

   ```
   # mkdir ~/bin/
   # cd ~/bin/
   # wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -O ~/bin/cfssl
   # wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -O ~/bin/cfssljson
   
   #修改可执行权限
   # chmod +x ~/bin/cfssl ~/bin/cfssljson
   
   # vim ~/.bash_profile
   export PATH=$PATH:~/bin/cfssljson
   
   # source ~/.bash_profile
   
   # cfssl version
   Version: 1.2.0
   Revision: dev
   Runtime: go1.6
   ```

2. 生成根证书

   ```
   # cfssl gencert -initca ca-csr.json | cfssljson -bare ca
   2021/03/16 22:32:31 [INFO] generating a new CA key and certificate from CSR
   2021/03/16 22:32:31 [INFO] generate received request
   2021/03/16 22:32:31 [INFO] received CSR
   2021/03/16 22:32:31 [INFO] generating key: rsa-2048
   2021/03/16 22:32:31 [INFO] encoded CSR
   2021/03/16 22:32:31 [INFO] signed certificate with serial number 413359770422448576053003613960705048321713967734
   
   # 生成完成后会有以下文件（我们最终想要的就是ca-key.pem和ca.pem，一个秘钥，一个证书）
   # ls
   ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem
   ```

3. 分发到主节点

   ```
   # 所有主节点创建目录
   # mkdir -p /etc/kubernetes/pki/
   # scp ca*.pem root@192.168.10.180:/etc/kubernetes/pki/
   # scp ca*.pem root@192.168.10.181:/etc/kubernetes/pki/
   # scp ca*.pem root@192.168.10.182:/etc/kubernetes/pki/
   ```

4.  部署etcd集群 下载etcd

   ```
   # wget https://github.com/coreos/etcd/releases/download/v3.2.18/etcd-v3.2.18-linux-amd64.tar.gz
   ```

5. 生成etcd证书和私钥

   ```
   # cd /root/kubernetes-ha-binary/target/pki/etcd
   # cfssl gencert -ca=../ca.pem \
       -ca-key=../ca-key.pem \
       -config=../ca-config.json \
       -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
      
   # 分发到所有主节点
   # scp etcd*.pem root@192.168.10.180:/etc/kubernetes/pki/
   # scp etcd*.pem root@192.168.10.181:/etc/kubernetes/pki/
   # scp etcd*.pem root@192.168.10.182:/etc/kubernetes/pki/
   ```

6. 创建etcd service 三个主节点执行

   ```
   # m1执行
   # cp ~/kubernetes-ha-binary/target/192.168.10.180/services/etcd.service /etc/systemd/system/
   
   # m2执行
   # cp ~/kubernetes-ha-binary/target/192.168.10.181/services/etcd.service /etc/systemd/system/
   
   # m2执行
   # cp ~/kubernetes-ha-binary/target/192.168.10.182/services/etcd.service /etc/systemd/system/
   
   # 创建工作目录
   # mkdir -p /var/lib/etcd
   ```

7. 启动etcd服务三个主节点执行

   ```
   #启动服务
   # systemctl daemon-reload && systemctl enable etcd && systemctl restart etcd
   
   #查看状态
   # service etcd status
   
   #查看启动日志
   # journalctl -f -u etcd
   ```

8. 部署api-server，m1执行

   ```
   #生成证书
   # cd ~/kubernetes-ha-binary/target/pki/apiserver
   
   # cfssl gencert -ca=../ca.pem \
      -ca-key=../ca-key.pem \
      -config=../ca-config.json \
      -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
   2021/03/16 22:54:24 [INFO] generate received request
   2021/03/16 22:54:24 [INFO] received CSR
   2021/03/16 22:54:24 [INFO] generating key: rsa-2048
   2021/03/16 22:54:24 [INFO] encoded CSR
   2021/03/16 22:54:24 [INFO] signed certificate with serial number 386026752012225804486821457846100490751819234351
   2021/03/16 22:54:24 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
   websites. For more information see the Baseline Requirements for the Issuance and Management
   of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
   specifically, section 10.2.3 ("Information Requirements").
   
   # scp kubernetes*.pem root@192.168.10.180:/etc/kubernetes/pki/
   # scp kubernetes*.pem root@192.168.10.181:/etc/kubernetes/pki/
   # scp kubernetes*.pem root@192.168.10.182:/etc/kubernetes/pki/
   ```

9. 创建service文件

   ```
   # m1执行
   # cp ~/kubernetes-ha-binary/target/192.168.10.180/services/kube-apiserver.service /etc/systemd/system/
   
   # m2执行
   # cp ~/kubernetes-ha-binary/target/192.168.10.181/services/kube-apiserver.service /etc/systemd/system/
   
   # m3执行
   # cp ~/kubernetes-ha-binary/target/192.168.10.182/services/kube-apiserver.service /etc/systemd/system/
   
   #创建日志文件，所有主节点执行
   mkdir -p /var/log/kubernetes
   ```

10. 启动服务，所有节点执行

    ```
    # systemctl daemon-reload && systemctl enable kube-apiserver && systemctl restart kube-apiserver
    
    #查看运行状态
    # service kube-apiserver status
    
    #查看日志
    # journalctl -f -u kube-apiserver
    
    #检查监听端口
    # netstat -ntlp
    ```

11. m1和m2两个主节点安装keepalived

    ```
    # yum install -y keepalived
    ```

12. m1和m2两个主节点创建keepalived配置文件

    ```
    # mkdir -p /etc/keepalived
    
    #m1执行
    cp ~/kubernetes-ha-binary/target/configs/keepalived-master.conf /etc/keepalived/keepalived.conf
    
    #m2执行
    # cp ~/kubernetes-ha-binary/target/configs/keepalived-backup.conf /etc/keepalived/keepalived.conf
    
    # 分别在master和backup上启动服务
    # systemctl enable keepalived && service keepalived start
    
    # 检查状态
    # service keepalived status
    
    # 查看日志
    # journalctl -f -u keepalived
    
    # 访问测试
    # curl --insecure https://192.168.10.185:6443/
    ```

13. 创建kubeconfig配置文件

    kubeconfig 为 kubectl 的配置文件，包含访问 apiserver 的所有信息，如 apiserver 地址、CA 证书和自身使用的证书

    ```
    # 生成证书
    # cd ~/kubernetes-ha-binary/target/pki/admin/
    # cfssl gencert -ca=../ca.pem \
      -ca-key=../ca-key.pem \
      -config=../ca-config.json \
      -profile=kubernetes admin-csr.json | cfssljson -bare admin
    
    # kubectl config set-cluster kubernetes \
      --certificate-authority=../ca.pem \
      --embed-certs=true \
      --server=https://192.168.10.185:6443 \
      --kubeconfig=kube.config
      
      # 设置集群参数
    # kubectl config set-cluster kubernetes \
      --certificate-authority=../ca.pem \
      --embed-certs=true \
      --server=https://<MASTER_VIP>:6443 \
      --kubeconfig=kube.config
    
    # 设置客户端认证参数
    # kubectl config set-credentials admin \
      --client-certificate=admin.pem \
      --client-key=admin-key.pem \
      --embed-certs=true \
      --kubeconfig=kube.config
    
    # 设置上下文参数
    # kubectl config set-context kubernetes \
      --cluster=kubernetes \
      --user=admin \
      --kubeconfig=kube.config
      
    # 设置默认上下文
    # kubectl config use-context kubernetes --kubeconfig=kube.config
    
    # 分发到目标节点
    # scp kube.config <user>@<node-ip>:~/.kube/config
    ```

14. 授予 kubernetes 证书访问 kubelet API 的权限，在执行 kubectl exec、run、logs 等命令时，apiserver 会转发到 kubelet。这里定义 RBAC 规则，授权 apiserver 调用 kubelet API

    ```
    # kubectl create clusterrolebinding kube-apiserver:kubelet-apis --clusterrole=system:kubelet-api-admin --user kubernetes
    clusterrolebinding.rbac.authorization.k8s.io/kube-apiserver:kubelet-apis created
    
    #测试
    # kubectl cluster-info
    Kubernetes master is running at https://192.168.10.185:6443
    
    To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
    #  kubectl get all --all-namespaces
    NAMESPACE   NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
    default     service/kubernetes   ClusterIP   10.254.0.1   <none>        443/TCP   33m
    #  kubectl get componentstatuses
    NAME                 STATUS      MESSAGE                                                                                     ERROR
    scheduler            Unhealthy   Get http://127.0.0.1:10251/healthz: dial tcp 127.0.0.1:10251: connect: connection refused   
    controller-manager   Unhealthy   Get http://127.0.0.1:10252/healthz: dial tcp 127.0.0.1:10252: connect: connection refused   
    etcd-2               Healthy     {"health": "true"}                                                                          
    etcd-0               Healthy     {"health": "true"}                                                                          
    etcd-1               Healthy     {"health": "true"}      
    ```

15. 部署controller-manager（所有master节点）

    controller-manager启动后将通过竞争选举机制产生一个 leader 节点，其它节点为阻塞状态。当 leader 节点不可用后，剩余节点将再次进行选举产生新的 leader 节点，从而保证服务的可用性。

    ```
    # 创建证书和私钥
    # cd ~/kubernetes-ha-binary/target/pki/controller-manager
    # cfssl gencert -ca=../ca.pem \
      -ca-key=../ca-key.pem \
      -config=../ca-config.json \
      -profile=kubernetes controller-manager-csr.json | cfssljson -bare controller-manager
      
    # 分发到每个master节点
    # scp controller-manager*.pem root@192.168.10.180:/etc/kubernetes/pki/
    # scp controller-manager*.pem root@192.168.10.181:/etc/kubernetes/pki/
    # scp controller-manager*.pem root@192.168.10.182:/etc/kubernetes/pki/
    ```

16. 创建controller-manager的kubeconfig

    ```
    # kubectl config set-cluster kubernetes \
      --certificate-authority=../ca.pem \
      --embed-certs=true \
      --server=https://192.168.10.185:6443 \
      --kubeconfig=controller-manager.kubeconfig
    
    # kubectl config set-credentials system:kube-controller-manager \
      --client-certificate=controller-manager.pem \
      --client-key=controller-manager-key.pem \
      --embed-certs=true \
      --kubeconfig=controller-manager.kubeconfig
    
    # kubectl config set-context system:kube-controller-manager \
      --cluster=kubernetes \
      --user=system:kube-controller-manager \
      --kubeconfig=controller-manager.kubeconfig
    
    # kubectl config use-context system:kube-controller-manager --kubeconfig=controller-manager.kubeconfig
    
    # 分发controller-manager.kubeconfig
    # scp controller-manager.kubeconfig root@192.168.10.180:/etc/kubernetes/
    # scp controller-manager.kubeconfig root@192.168.10.181:/etc/kubernetes/
    # scp controller-manager.kubeconfig root@192.168.10.182:/etc/kubernetes/
    
    # 创建service文件
    # scp ~/kubernetes-ha-binary/target/services/kube-controller-manager.service root@192.168.10.180:/etc/systemd/system/
    # scp ~/kubernetes-ha-binary/target/services/kube-controller-manager.service root@192.168.10.181:/etc/systemd/system/
    # scp ~/kubernetes-ha-binary/target/services/kube-controller-manager.service root@192.168.10.182:/etc/systemd/system/
    ```

17. 启动验证服务,所有主节点

    ```
    # 启动服务
    $ systemctl daemon-reload && systemctl enable kube-controller-manager && systemctl restart kube-controller-manager
    
    # 检查状态
    # service kube-controller-manager status
    
    # 查看日志
    # journalctl -f -u kube-controller-manager
    
    # 查看leader
    # kubectl get endpoints kube-controller-manager --namespace=kube-system -o yaml
    ```

18. 部署scheduler（master节点）

    scheduler启动后将通过竞争选举机制产生一个 leader 节点，其它节点为阻塞状态。当 leader 节点不可用后，剩余节点将再次进行选举产生新的 leader 节点，从而保证服务的可用性。

    ```
    # 创建证书和私钥
    # 生成证书、私钥
    $ cd ~/kubernetes-ha-binary/target/pki/scheduler
    $ cfssl gencert -ca=../ca.pem \
      -ca-key=../ca-key.pem \
      -config=../ca-config.json \
      -profile=kubernetes scheduler-csr.json | cfssljson -bare kube-scheduler
    # 创建scheduler的kubeconfig
    # 创建kubeconfig
    # kubectl config set-cluster kubernetes \
      --certificate-authority=../ca.pem \
      --embed-certs=true \
      --server=https://192.168.10.185:6443 \
      --kubeconfig=kube-scheduler.kubeconfig
    
    # kubectl config set-credentials system:kube-scheduler \
      --client-certificate=kube-scheduler.pem \
      --client-key=kube-scheduler-key.pem \
      --embed-certs=true \
      --kubeconfig=kube-scheduler.kubeconfig
    
    # kubectl config set-context system:kube-scheduler \
      --cluster=kubernetes \
      --user=system:kube-scheduler \
      --kubeconfig=kube-scheduler.kubeconfig
    
    # kubectl config use-context system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig
    
    # 分发kubeconfig
    # scp kube-scheduler.kubeconfig root@192.168.10.180:/etc/kubernetes/
    # scp kube-scheduler.kubeconfig root@192.168.10.181:/etc/kubernetes/
    # scp kube-scheduler.kubeconfig root@192.168.10.182:/etc/kubernetes/
    
    # 创建service文件
    # scp配置文件到每个master节点
    $ scp target/services/kube-scheduler.service <user>@<node-ip>:/etc/systemd/system/
    7.4 启动服务
    # 启动服务
    $ systemctl daemon-reload && systemctl enable kube-scheduler && systemctl restart kube-scheduler
    
    # 检查状态
    $ service kube-scheduler status
    
    # 查看日志
    $ journalctl -f -u kube-scheduler
    
    # 查看leader
    $ kubectl get endpoints kube-scheduler --namespace=kube-system -o yaml
    ```

19.  部署work节点

     ```
     #两台work同时下载镜像(两台节点同时执行)
     # cp ~/kubernetes-ha-binary/configs/download-images.sh ~/
     # cd ~/
     # sh download-images.sh
     # docker images
     REPOSITORY                              TAG       IMAGE ID       CREATED       SIZE
     quay.io/calico/node                     v3.1.3    7eca10056c8e   2 years ago   248MB
     quay.io/calico/typha                    v0.7.4    c8f53c1b7957   2 years ago   56.9MB
     quay.io/calico/cni                      v3.1.3    9f355e076ea7   2 years ago   68.8MB
     k8s.gcr.io/coredns                      1.1.3     b3b94275d97c   2 years ago   45.6MB
     k8s.gcr.io/kubernetes-dashboard-amd64   v1.8.3    0c60bcf89900   3 years ago   102MB
     k8s.gcr.io/pause-amd64                  3.1       da86e6ba6ca1   3 years ago   742kB
     ```

20.  创建bootstrap配置文件(m1执行)

     ```
     # cd ~/kubernetes-ha-binary/target/pki/admin
     # export BOOTSTRAP_TOKEN=$(kubeadm token create \
     >       --description kubelet-bootstrap-token \
     >       --groups system:bootstrappers:worker \
     >       --kubeconfig kube.config)
     
     # echo $BOOTSTRAP_TOKEN
     bkyttc.vuwr1uqsdn2ldbc7
     
     # 设置集群参数
     # kubectl config set-cluster kubernetes \
           --certificate-authority=../ca.pem \
           --embed-certs=true \
           --server=https://192.168.10.185:6443 \
           --kubeconfig=kubelet-bootstrap.kubeconfig
           
     # 设置客户端认证参数
     # kubectl config set-credentials kubelet-bootstrap \
           --token=${BOOTSTRAP_TOKEN} \
           --kubeconfig=kubelet-bootstrap.kubeconfig
     
     # 设置上下文参数
     # kubectl config set-context default \
           --cluster=kubernetes \
           --user=kubelet-bootstrap \
           --kubeconfig=kubelet-bootstrap.kubeconfig
     
     # 设置默认上下文
     # kubectl config use-context default --kubeconfig=kubelet-bootstrap.kubeconfig
     
     # 两个work节点创建目录
     # mkdir /etc/kubernetes/
     # mkdir /etc/kubernetes/pki/
     
     # 把生成的配置copy到每个worker节点上
     # scp ~/kubernetes-ha-binary/target/pki/admin/kubelet-bootstrap.kubeconfig root@192.168.10.183:/etc/kubernetes/kubelet-bootstrap.kubeconfig
     # scp ~/kubernetes-ha-binary/target/pki/admin/kubelet-bootstrap.kubeconfig root@192.168.10.184:/etc/kubernetes/kubelet-bootstrap.kubeconfig
     
     # 把ca分发到每个worker节点
     # scp ~/kubernetes-ha-binary/target/pki/ca.pem root@192.168.10.183:/etc/kubernetes/pki/
     # scp ~/kubernetes-ha-binary/target/pki/ca.pem root@192.168.10.184:/etc/kubernetes/pki/
     ```

21.  kubelet配置文件

     把kubelet配置文件分发到每个worker节点上

     ```
     # scp ~/kubernetes-ha-binary/target/worker-192.168.10.183/kubelet.config.json root@192.168.10.183:/etc/kubernetes/
     # scp ~/kubernetes-ha-binary/target/worker-192.168.10.184/kubelet.config.json root@192.168.10.184:/etc/kubernetes/
     ```

22.  kubelet服务文件

     把kubelet服务文件分发到每个worker节点上

     ```
     # scp ~/kubernetes-ha-binary/target/worker-192.168.10.183/kubelet.service root@192.168.10.183:/etc/systemd/system/
     # scp ~/kubernetes-ha-binary/target/worker-192.168.10.184/kubelet.service root@192.168.10.184:/etc/systemd/system/
     ```

23.  启动服务

     kublet 启动时查找配置的 --kubeletconfig 文件是否存在，如果不存在则使用 --bootstrap-kubeconfig 向 kube-apiserver 发送证书签名请求 (CSR)。 kube-apiserver 收到 CSR 请求后，对其中的 Token 进行认证（事先使用 kubeadm 创建的 token），认证通过后将请求的 user 设置为 system:bootstrap:，group 设置为 system:bootstrappers，这就是Bootstrap Token Auth。

     ```
     # bootstrap附权(m1执行)
     # kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --group=system:bootstrappers
     
     # 启动服务(俩word执行)
     # mkdir -p /var/lib/kubelet
     # cd /opt/kubernetes/
     # ln -s worker/ bin
     # systemctl daemon-reload && systemctl enable kubelet && systemctl restart kubelet
     
     # 在master上Approve bootstrap请求，查看有没有work请求
     # kubectl get csr
     NAME                                                   AGE   REQUESTOR                 CONDITION
     node-csr-Qj27milSWi09grr_tV6ThKGb8DptdwWXEs4V-wYPTHI   48s   system:bootstrap:bkyttc   Pending
     node-csr-SlydMOPsFIv1mBOZ5T6wuc58WMTMRLT5QdlJtmhc2O0   64s   system:bootstrap:bkyttc   Pending
     
     # kubectl certificate approve node-csr-Qj27milSWi09grr_tV6ThKGb8DptdwWXEs4V-wYPTHI
     certificatesigningrequest.certificates.k8s.io/node-csr-Qj27milSWi09grr_tV6ThKGb8DptdwWXEs4V-wYPTHI approved
     # kubectl certificate approve node-csr-SlydMOPsFIv1mBOZ5T6wuc58WMTMRLT5QdlJtmhc2O0
     certificatesigningrequest.certificates.k8s.io/node-csr-SlydMOPsFIv1mBOZ5T6wuc58WMTMRLT5QdlJtmhc2O0 approved
     
     # 查看服务状态
     # service kubelet status
     
     # 查看日志
     # journalctl -f -u kubelet
     ```

24.  部署kube-proxy（worker节点）

     ```
     # 创建证书和私钥(m1执行)
     # cd ~/kubernetes-ha-binary/target/pki/proxy
     # cfssl gencert -ca=../ca.pem \
       -ca-key=../ca-key.pem \
       -config=../ca-config.json \
       -profile=kubernetes  kube-proxy-csr.json | cfssljson -bare kube-proxy
     
     # 创建和分发 kubeconfig 文件
     # 创建kube-proxy.kubeconfig
     # kubectl config set-cluster kubernetes \
       --certificate-authority=../ca.pem \
       --embed-certs=true \
       --server=https://192.168.10.185:6443 \
       --kubeconfig=kube-proxy.kubeconfig
     
     # kubectl config set-credentials kube-proxy \
       --client-certificate=kube-proxy.pem \
       --client-key=kube-proxy-key.pem \
       --embed-certs=true \
       --kubeconfig=kube-proxy.kubeconfig
       
     # kubectl config set-context default \
       --cluster=kubernetes \
       --user=kube-proxy \
       --kubeconfig=kube-proxy.kubeconfig
       
     # kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
     
     # 分发kube-proxy.kubeconfig到两个work节点
     # scp kube-proxy.kubeconfig root@192.168.10.183:/etc/kubernetes/
     # scp kube-proxy.kubeconfig root@192.168.10.184:/etc/kubernetes/
     
     # 分发kube-proxy.config
     # scp  ~/kubernetes-ha-binary/target/worker-192.168.10.183/kube-proxy.config.yaml root@192.168.10.183:/etc/kubernetes/
     # scp  ~/kubernetes-ha-binary/target/worker-192.168.10.184/kube-proxy.config.yaml root@192.168.10.184:/etc/kubernetes/
     
     # 分发kube-proxy服务文件
     # scp ~/kubernetes-ha-binary/target/services/kube-proxy.service root@192.168.10.183:/etc/systemd/system/
     # scp ~/kubernetes-ha-binary/target/services/kube-proxy.service root@192.168.10.184:/etc/systemd/system/
     
     # 启动服务
     # 创建依赖目录(两个work节点执行)
     # mkdir -p /var/lib/kube-proxy && mkdir -p /var/log/kubernetes
     
     # 启动服务
     $ systemctl daemon-reload && systemctl enable kube-proxy && systemctl restart kube-proxy
     
     # 查看状态
     # service kube-proxy status
     
     # 查看日志
     # journalctl -f -u kube-proxy
     ```

25.  部署CNI插件 - calico

     使用calico官方的安装方式来部署。

     ```
     # 创建目录（在配置了kubectl的节点上执行，也就是m1）
     # mkdir -p /etc/kubernetes/addons
     
     # 拷贝calico配置到配置好kubectl的节点(m1)（一个节点即可）
     # cp ~/kubernetes-ha-binary/target/addons/calico* /etc/kubernetes/addons/
     
     # 部署calico
     # kubectl create -f /etc/kubernetes/addons/calico-rbac-kdd.yaml
     # kubectl create -f /etc/kubernetes/addons/calico.yaml
     
     # 查看状态
     # kubectl get pods -n kube-system
     ```

26.  部署DNS插件 - coredns（在配置了kubectl的节点上执行，也就是m1）

     ```
     # 上传配置文件
     # scp target/addons/coredns.yaml <user>@<node-ip>:/etc/kubernetes/addons/
     
     # 部署coredns
     $ kubectl create -f /etc/kubernetes/addons/coredns.yaml
     ```

# 检查集群可用性

1. 创建一个容器

   ```
   # 写入配置
   $ cat > nginx-ds.yml <<EOF
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
   apiVersion: extensions/v1beta1
   kind: DaemonSet
   metadata:
     name: nginx-ds
     labels:
       addonmanager.kubernetes.io/mode: Reconcile
   spec:
     template:
       metadata:
         labels:
           app: nginx-ds
       spec:
         containers:
         - name: my-nginx
           image: nginx:1.7.9
           ports:
           - containerPort: 80
   EOF
   
   # 创建ds
   # kubectl create -f nginx-ds.yml
   ```

2. 检查各种ip连通性

   ```
   # 检查各 Node 上的 Pod IP 连通性（主节点没有calico所以不能访问podip）
   # kubectl get pods  -o wide
   
   # 在每个worker节点上ping pod ip
   # ping <pod-ip>
   
   # 检查service可达性
   # kubectl get svc
   
   # 在每个worker节点上访问服务（主节点没有proxy所以不能访问service-ip）
   # curl <service-ip>:<port>
   
   # 在每个节点检查node-port可用性
   # curl <node-ip>:<port>
   ```

3. 检查dns可用性

   ```
   # 创建一个nginx pod
   # cat > pod-nginx.yaml <<EOF
   apiVersion: v1
   kind: Pod
   metadata:
     name: nginx
   spec:
     containers:
     - name: nginx
       image: nginx:1.7.9
       ports:
       - containerPort: 80
   EOF
   
   # 创建pod
   # kubectl create -f pod-nginx.yaml
   
   # 进入pod，查看dns
   # kubectl exec  nginx -i -t -- /bin/bash
   
   # 查看dns配置
   root@nginx:/# cat /etc/resolv.conf
   
   # 查看名字是否可以正确解析
   root@nginx:/# ping nginx-ds
   root@nginx:/# ping kubernetes
   ```

# 部署dashboard

1. 创建服务

   ```
   # 上传dashboard配置
   # cp ~/kubernetes-ha-binary/target/addons/dashboard-all.yaml /etc/kubernetes/addons/
   
   # 创建服务
   # kubectl apply -f /etc/kubernetes/addons/dashboard-all.yaml
   
   # 查看服务运行情况
   # kubectl get deployment kubernetes-dashboard -n kube-system
   # kubectl --namespace kube-system get pods -o wide
   # kubectl get services kubernetes-dashboard -n kube-system
   # netstat -ntlp|grep 8401
   ```

2. 访问dashboard

   为了集群安全，从 1.7 开始，dashboard 只允许通过 https 访问，我们使用nodeport的方式暴露服务，可以使用 [https://NodeIP](https://nodeip/):NodePort 地址访问
   关于自定义证书
   默认dashboard的证书是自动生成的，肯定是非安全的证书，如果大家有域名和对应的安全证书可以自己替换掉。使用安全的域名方式访问dashboard。
   在dashboard-all.yaml中增加dashboard启动参数，可以指定证书文件，其中证书文件是通过secret注进来的。

   - dashboard.cer
   - –tls-key-file
   - dashboard.key

3. 登录dashboard

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
   ```

   

   

