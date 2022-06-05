# 环境准备

|   系统    |       IP       |                             角色                             |    主机名     |
| :-------: | :------------: | :----------------------------------------------------------: | :-----------: |
| centos7.9 | 192.168.10.170 | kube-apiserver，kube-controller-manager，kube-scheduler，kubelet，kube-proxy，docker，etcd，  nginx，keepalived | k8s-manual-01 |
| centos7.9 | 192.168.10.173 | kube-apiserver，kube-controller-manager，kube-scheduler，kubelet，kube-proxy，docker，etcd，  nginx，keepalived | k8s-manual-04 |
| centos7.9 | 192.168.10.171 |              kubelet，kube-proxy，docker，etcd               | k8s-manual-02 |
| centos7.9 | 192.168.10.172 |                 kubelet，kube-proxy，docker                  | k8s-manual-03 |
| centos7.9 | 192.168.10.172 |                 kubelet，kube-proxy，docker                  | k8s-manual-05 |

## 所有节点设置主机名

```
# hostnamectl set-hostname <主机名>
```

## 安装依赖

```
 yum install -y iptables
```

### 关闭防火墙，swap

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

### 所有节点添加系统参数 

桥接IPv4流量传递到iptables

```
# cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
vm.swappiness=0
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
EOF
# 生效文件
# sysctl --system
```

### 修改host文件

```
# cat >> /etc/hosts << EOF 
192.168.10.170 k8s-manual-01
192.168.10.171 k8s-manual-02
192.168.10.172 k8s-manual-03
EOF

cat >> /etc/hosts << EOF 
192.168.10.173 k8s-manual-04
192.168.10.174 k8s-manual-05
EOF

# cat /etc/hosts
```

# 开始部署

## 生成etcd证书

所有节点的交互都是基于https的交互，需要有证书 这里 在第一个节点生成证书

```
# cd /opt/ca
# wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
# wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
# wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
# chmod +x cfssl_linux-amd64 cfssljson_linux-amd64 cfssl-certinfo_linux-amd64
# mv cfssl_linux-amd64 /usr/bin/cfssl
# mv cfssljson_linux-amd64 /usr/bin/cfssljson
# mv cfssl-certinfo_linux-amd64 /usr/cfssl-certinfo
```

## 部署etcd

k8s依赖etcd做做数据存储 所以etcd需要做集群 也可以独立部署三台节点做etcd集群，需要单数节点，由apiserver对etcd进行操作 是整个集群的统一入口，所有连接基于https连接 

### 生成etcd的证书

用于etcd内部通信apiserver访问etcd

```
# mkdir /opt/ca/etcd/
# cd /opt/ca/etcd/
# 生成自签的跟证书 
# cat > ca-config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "www": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
EOF

# cat > ca-csr.json << EOF
{
    "CN": "etcd CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing"
        }
    ]
}
EOF

# 执行 生成etcd的跟证书
# cfssl gencert -initca ca-csr.json | cfssljson -bare ca 
```

### 签发etcd的https证书

创建证书文件

必须包含所有的etcd节点，可以多签发几个备用

```
cat > server-csr-etcd.json << EOF
{
    "CN": "etcd",
    "hosts": [
    "192.168.10.170",
    "192.168.10.171",
    "192.168.10.172"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing"
        }
    ]
}
EOF

#生成证书
# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr-etcd.json | cfssljson -bare server
# ll
total 36
-rw-r--r-- 1 root root  287 Apr 27 22:01 ca-config.json
-rw-r--r-- 1 root root  956 Apr 27 22:02 ca.csr
-rw-r--r-- 1 root root  209 Apr 27 22:01 ca-csr.json
-rw------- 1 root root 1679 Apr 27 22:02 ca-key.pem
-rw-r--r-- 1 root root 1265 Apr 27 22:02 ca.pem
-rw-r--r-- 1 root root 1013 Apr 27 22:06 server.csr
-rw-r--r-- 1 root root  293 Apr 27 22:05 server-csr-etcd.json
-rw------- 1 root root 1679 Apr 27 22:06 server-key.pem
-rw-r--r-- 1 root root 1338 Apr 27 22:06 server.pem
```

### 部署etcd1 

下载etcd包

https://github.com/etcd-io/etcd/releases/download/v3.4.9/etcd-v3.4.9-linux-amd64.tar.gz

将etcd部署到所有etcd节点 上传etcd到/opt/soft下解压

```
# 创建目录
# mkdir -p /opt/etcd/{bin,cfg,ssl} -p 
# cp /opt/soft/etcd-v3.4.9-linux-amd64/etcd /opt/etcd/bin/
# cp /opt/soft/etcd-v3.4.9-linux-amd64/etcdctl /opt/etcd/bin/
# cp /opt/ca/etcd/server.pem /opt/etcd/ssl/
# cp /opt/ca/etcd/server-key.pem /opt/etcd/ssl/
# cp /opt/ca/etcd/ca.pem /opt/etcd/ssl/

#生成etcd配置文件到cfg文件架里
cat > /opt/etcd/cfg/etcd.conf << EOF
#[Member]
#当前节点名称 集群中唯一
ETCD_NAME="etcd-1"
#工作目录
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
#集群内部通信的地址
ETCD_LISTEN_PEER_URLS="https://192.168.10.170:2380"
#客户端连接的地址
ETCD_LISTEN_CLIENT_URLS="https://192.168.10.170:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.10.170:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.10.170:2379"
#集群的所有节点
ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.10.170:2380,etcd-2=https://192.168.10.171:2380,etcd-3=https://192.168.10.172:2380"
#集群的通信的token
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF

```

•   ETCD_NAME：节点名称，集群中唯一

•   ETCD_DATA_DIR：数据目录

•   ETCD_LISTEN_PEER_URLS：集群通信监听地址

•   ETCD_LISTEN_CLIENT_URLS：客户端访问监听地址

•   ETCD_INITIAL_ADVERTISE_PEERURLS：集群通告地址

•   ETCD_ADVERTISE_CLIENT_URLS：客户端通告地址

•   ETCD_INITIAL_CLUSTER：集群节点地址

•   ETCD_INITIALCLUSTER_TOKEN：集群Token

•   ETCD_INITIALCLUSTER_STATE：加入集群的当前状态，new是新集群，existing表示加入已有集群

### 创建systemd服务

```
cat > /usr/lib/systemd/system/etcd.service << EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=/opt/etcd/cfg/etcd.conf
ExecStart=/opt/etcd/bin/etcd \
--cert-file=/opt/etcd/ssl/server.pem \
--key-file=/opt/etcd/ssl/server-key.pem \
--peer-cert-file=/opt/etcd/ssl/server.pem \
--peer-key-file=/opt/etcd/ssl/server-key.pem \
--trusted-ca-file=/opt/etcd/ssl/ca.pem \
--peer-trusted-ca-file=/opt/etcd/ssl/ca.pem \
--logger=zap
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

# 设置开机起动
# systemctl daemon-reload
# systemctl enable etcd
# systemctl start etcd

# 查看日志 节点出于等待状态，等待其他两个节点起动
# journalctl -u etcd
```

### 部署etcd2和3 

直接将etc整个文件拷贝到171和172

```
# 拷贝到171
# scp -r /opt/etcd/ root@192.168.10.171:/opt/
# scp /usr/lib/systemd/system/etcd.service root@192.168.10.171:/usr/lib/systemd/system/

# 修改节点配置信息
# vim /opt/etcd/cfg/etcd.conf
#[Member]
#当前节点名称 集群中唯一
ETCD_NAME="etcd-2"
#工作目录
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
#集群内部通信的地址
ETCD_LISTEN_PEER_URLS="https://192.168.10.171:2380"
#客户端连接的地址
ETCD_LISTEN_CLIENT_URLS="https://192.168.10.171:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.10.171:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.10.171:2379"
#集群的所有节点
ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.10.170:2380,etcd-2=https://192.168.10.171:2380,etcd-3=https://192.168.10.172:2380"
#集群的通信的token
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
# 设置开机起动
# systemctl daemon-reload
# systemctl enable etcd
# systemctl start etcd


# 拷贝到172 172执行
# scp -r /opt/etcd/ root@192.168.10.172:/opt/
# scp /usr/lib/systemd/system/etcd.service root@192.168.10.172:/usr/lib/systemd/system/
# vim /opt/etcd/cfg/etcd.conf
#[Member]
#当前节点名称 集群中唯一
ETCD_NAME="etcd-3"
#工作目录
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
#集群内部通信的地址
ETCD_LISTEN_PEER_URLS="https://192.168.10.172:2380"
#客户端连接的地址
ETCD_LISTEN_CLIENT_URLS="https://192.168.10.172:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.10.172:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.10.172:2379"
#集群的所有节点
ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.10.170:2380,etcd-2=https://192.168.10.171:2380,etcd-3=https://192.168.10.172:2380"
#集群的通信的token
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
# 设置开机起动
# systemctl daemon-reload
# systemctl enable etcd
# systemctl start etcd

```

### 三台节点一起起动

```
# 最后三台节点一起起动
# systemctl start etcd
```

### 查看集群状态

全部为true 代表集群正常

```
# ETCDCTL_API=3 /opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --endpoints="https://192.168.10.170:2379,https://192.168.10.171:2379,https://192.168.10.172:2379" endpoint health --write-out=table
+-----------------------------+--------+-------------+-------+
|          ENDPOINT           | HEALTH |    TOOK     | ERROR |
+-----------------------------+--------+-------------+-------+
| https://192.168.10.172:2379 |   true | 15.376945ms |       |
| https://192.168.10.170:2379 |   true | 14.987919ms |       |
| https://192.168.10.171:2379 |   true | 19.427651ms |       |
+-----------------------------+--------+-------------+-------+
```

1. 集群日志默认在/var/log/message 或 journalctl -u etcd

# 安装docker 

所有节点master和node执行

```
# cd /opt/soft/
# tar zxvf docker-19.03.9.tgz 
# mv docker/* /usr/bin

# 生成systemd服务
cat > /usr/lib/systemd/system/docker.service << EOF
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
EOF

#配置镜像加速
# mkdir /etc/docker
# cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"]
}
EOF

#启动
# systemctl daemon-reload && systemctl enable docker && systemctl start docker

#最后查看docker状态
# docker info
```

# 部署master

## 生成kube-apiserver证书 先生成ca跟证书

```
# 170节点执行
# mkdir /opt/ca/apiserver
# ca /opt/ca/apiserver
# cat > ca-config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
EOF
cat > ca-csr.json << EOF
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF

# 生成ca的pem
# cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
# 会生成ca.pem和ca-key.pem文件
```

## 生成apiserver HTTPS证书

生成CA签发kube-apiserver HTTPS证书 ip需要包含所有的kube节点 包括将来需要扩容的

hosts字段中IP为所有Master/LB/VIP IP，

```
cat > server-csr-apiserver.json << EOF
{
    "CN": "kubernetes",
    "hosts": [
      "10.0.0.1",
      "127.0.0.1",
      "192.168.10.170",
      "192.168.10.171",
      "192.168.10.172",
      "192.168.10.173",
      "192.168.10.174",
      "192.168.10.175",
      "192.168.10.176",
      "192.168.10.177",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF


#生成证书
# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr-apiserver.json | cfssljson -bare server-apiserver

#会生成server-apiserver.pem和server-apiserver-key.pem文件。
# ll
total 36
-rw-r--r-- 1 root root  294 Apr 27 22:52 ca-config.json
-rw-r--r-- 1 root root 1001 Apr 27 22:52 ca.csr
-rw-r--r-- 1 root root  264 Apr 27 22:52 ca-csr.json
-rw------- 1 root root 1679 Apr 27 22:52 ca-key.pem
-rw-r--r-- 1 root root 1359 Apr 27 22:52 ca.pem
-rw-r--r-- 1 root root 1301 Apr 27 22:57 server-apiserver.csr
-rw------- 1 root root 1679 Apr 27 22:57 server-apiserver-key.pem
-rw-r--r-- 1 root root 1667 Apr 27 22:57 server-apiserver.pem
-rw-r--r-- 1 root root  680 Apr 27 22:55 server-csr-apiserver.json
```

## 部署master节点

下载server和node的二进制包： https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.20.md

* kubernetes-server-linux-amd64 就这一个就可以

### 创建目录

```
# mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs} 
# cd /opt/soft/
# tar zxvf kubernetes-server-linux-amd64.tar.gz 
# cd kubernetes/server/bin/
# cp kube-apiserver kube-scheduler kube-controller-manager /opt/kubernetes/bin
# cp kubectl /usr/bin/
```

### 拷贝证书到目录下

```
# cp /opt/ca/apiserver/ca*.pem /opt/kubernetes/ssl/
# cp /opt/ca/apiserver/server-*.pem /opt/kubernetes/ssl/
```

### 准备配置文件

```
# cd /opt/kubernetes/cfg/
# cat > /opt/kubernetes/cfg/kube-apiserver.conf << EOF
KUBE_APISERVER_OPTS="--logtostderr=false \\
--v=2 \\
# 日志
--log-dir=/opt/kubernetes/logs \\
# etcd地址
--etcd-servers=https://192.168.10.170:2379,https://192.168.10.171:2379,https://192.168.10.172:2379 \\
# apiserver绑定的当前ip
--bind-address=192.168.10.170 \\
--secure-port=6443 \\
# apiserver集群通讯ip
--advertise-address=192.168.10.170 \\
--allow-privileged=true \\
# cluster service的ip
--service-cluster-ip-range=10.0.0.0/24 \\
--enable-admission-plugins=NodeRestriction \\
# 授权模式
--authorization-mode=RBAC,Node \\
# 启用认证
--enable-bootstrap-token-auth=true \\
--token-auth-file=/opt/kubernetes/cfg/token.csv \\
# service的断口范围
--service-node-port-range=30000-32767 \\
--kubelet-client-certificate=/opt/kubernetes/ssl/server-apiserver.pem \\
--kubelet-client-key=/opt/kubernetes/ssl/server-apiserver-key.pem \\
--tls-cert-file=/opt/kubernetes/ssl/server-apiserver.pem  \\
--tls-private-key-file=/opt/kubernetes/ssl/server-apiserver-key.pem \\
--client-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \\
# 服务令牌 1.20后新加的 控制service account帐号鉴权
--service-account-issuer=api \\
--service-account-signing-key-file=/opt/kubernetes/ssl/ca-key.pem \\
# etcd证书
--etcd-cafile=/opt/etcd/ssl/ca.pem \\
--etcd-certfile=/opt/etcd/ssl/server.pem \\
--etcd-keyfile=/opt/etcd/ssl/server-key.pem \\
#聚合层
--requestheader-client-ca-file=/opt/kubernetes/ssl/ca.pem \\
--proxy-client-cert-file=/opt/kubernetes/ssl/server-apiserver.pem \\
--proxy-client-key-file=/opt/kubernetes/ssl/server-apiserver-key.pem \\
--requestheader-allowed-names=kubernetes \\
--requestheader-extra-headers-prefix=X-Remote-Extra- \\
--requestheader-group-headers=X-Remote-Group \\
--requestheader-username-headers=X-Remote-User \\
--enable-aggregator-routing=true \\
# 审计相关
--audit-log-maxage=30 \\
--audit-log-maxbackup=3 \\
--audit-log-maxsize=100 \\
--audit-log-path=/opt/kubernetes/logs/k8s-audit.log"
EOF

# cat > /opt/kubernetes/cfg/kube-apiserver.conf << EOF
KUBE_APISERVER_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--etcd-servers=https://192.168.10.170:2379,https://192.168.10.171:2379,https://192.168.10.172:2379 \\
--bind-address=192.168.10.170 \\
--secure-port=6443 \\
--advertise-address=192.168.10.170 \\
--allow-privileged=true \\
--service-cluster-ip-range=10.0.0.0/24 \\
--enable-admission-plugins=NodeRestriction \\
--authorization-mode=RBAC,Node \\
--enable-bootstrap-token-auth=true \\
--token-auth-file=/opt/kubernetes/cfg/token.csv \\
--service-node-port-range=30000-32767 \\
--kubelet-client-certificate=/opt/kubernetes/ssl/server-apiserver.pem \\
--kubelet-client-key=/opt/kubernetes/ssl/server-apiserver-key.pem \\
--tls-cert-file=/opt/kubernetes/ssl/server-apiserver.pem  \\
--tls-private-key-file=/opt/kubernetes/ssl/server-apiserver-key.pem \\
--client-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--service-account-issuer=api \\
--service-account-signing-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--etcd-cafile=/opt/etcd/ssl/ca.pem \\
--etcd-certfile=/opt/etcd/ssl/server.pem \\
--etcd-keyfile=/opt/etcd/ssl/server-key.pem \\
--requestheader-client-ca-file=/opt/kubernetes/ssl/ca.pem \\
--proxy-client-cert-file=/opt/kubernetes/ssl/server-apiserver.pem \\
--proxy-client-key-file=/opt/kubernetes/ssl/server-apiserver-key.pem \\
--requestheader-allowed-names=kubernetes \\
--requestheader-extra-headers-prefix=X-Remote-Extra- \\
--requestheader-group-headers=X-Remote-Group \\
--requestheader-username-headers=X-Remote-User \\
--enable-aggregator-routing=true \\
--audit-log-maxage=30 \\
--audit-log-maxbackup=3 \\
--audit-log-maxsize=100 \\
--audit-log-path=/opt/kubernetes/logs/k8s-audit.log"
EOF
```

上面两个\ \ 第一个是转义符，第二个是换行符，使用转义符是为了使用EOF保留换行符

•   --logtostderr：启用日志

•   ---v：日志等级

•   --log-dir：日志目录

•   --etcd-servers：etcd集群地址

•   --bind-address：监听地址

•   --secure-port：https安全端口

•   --advertise-address：集群通告地址

•   --allow-privileged：启用授权

•   --service-cluster-ip-range：Service虚拟IP地址段

•   --enable-admission-plugins：准入控制模块

•   --authorization-mode：认证授权，启用RBAC授权和节点自管理

•   --enable-bootstrap-token-auth：启用TLS bootstrap机制 就是为了kubelet颁发证书的

•   --token-auth-file：bootstrap token文件

•   --service-node-port-range：Service nodeport类型默认分配端口范围

•   --kubelet-client-xxx：apiserver访问kubelet客户端证书

•   --tls-xxx-file：apiserver https证书

•   1.20版本必须加的参数：--service-account-issuer，--service-account-signing-key-file

•   --etcd-xxxfile：连接Etcd集群证书

•   --audit-log-xxx：审计日志

•   启动聚合层相关配置：--requestheader-client-ca-file，--proxy-client-cert-file，--proxy-client-key-file，--requestheader-allowed-names，--requestheader-extra-headers-prefix，--requestheader-group-headers，--requestheader-username-headers，--enable-aggregator-routing

### 生成 bootstrap token文件

生成上面配置文件的  bootstrap token文件token 用于整个集群token

格式：token，用户名，UID，用户组

相当于一个最小权限的用户 K8s拿着这个用户去请求颁发证书

```
# head -c 16 /dev/urandom | od -An -t x | tr -d ' '
6fb1839685ab7dd63a6c24f24e2320d5
# cat > /opt/kubernetes/cfg/token.csv << EOF
6fb1839685ab7dd63a6c24f24e2320d5,kubelet-bootstrap,10001,"system:node-bootstrapper"
EOF
```

### 配置systemd

```
# 这里\$KUBE_APISERVER_OPTS参数就是对应的/opt/kubernetes/cfg/kube-apiserver.conf文件里的KUBE_APISERVER_OPTS变量
# cat > /usr/lib/systemd/system/kube-apiserver.service << EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-apiserver.conf
ExecStart=/opt/kubernetes/bin/kube-apiserver \$KUBE_APISERVER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# systemctl daemon-reload && systemctl enable kube-apiserver 
```

### 起动apiserver

```
# systemctl start kube-apiserver 
# ps -ef|grep kube
```

### 部署kube-controller-manager

也是同过https访问apiserver的

#### 创建配置文件

```
# 创建配置文件
cat > /opt/kubernetes/cfg/kube-controller-manager.conf << EOF
KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--leader-elect=true \\
--kubeconfig=/opt/kubernetes/cfg/kube-controller-manager.kubeconfig \\
--bind-address=127.0.0.1 \\
--allocate-node-cidrs=true \\
--cluster-cidr=10.244.0.0/16 \\
--service-cluster-ip-range=10.0.0.0/24 \\
--cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \\
--cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem  \\
--root-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--cluster-signing-duration=87600h0m0s"
EOF


```

•	--kubeconfig：连接apiserver配置文件
•	--leader-elect：当该组件启动多个时，自动选举（HA）
•	--cluster-signing-cert-file/--cluster-signing-key-file：自动为•	kubelet颁发证书的CA，与apiserver保持一致
•	--logtostderr=false：日志
•	leader-elect=true：leader选举
•	bind-address=127.0.0.1: 监听地址
•	cluster-cidr=10.244.0.0/16: cidr网段
•	cluster-signing-duration：证书有效期
•	cluster-signing-cert-file：为kubernete颁发证书所需要的ca 同过跟证书颁发

#### 生成kube-controller-manager证书

```
# mkdir /opt/ca/kubecontrollermanager
# cd /opt/ca/kubecontrollermanager
# cat > kube-controller-manager-csr.json << EOF
{
  "CN": "system:kube-controller-manager",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing", 
      "ST": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF

# 生成证书
# cfssl gencert -ca=/opt/ca/apiserver/ca.pem -ca-key=/opt/ca/apiserver/ca-key.pem -config=/opt/ca/apiserver/ca-config.json -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

# ll
total 16
-rw-r--r-- 1 root root 1045 Apr 27 23:33 kube-controller-manager.csr
-rw-r--r-- 1 root root  255 Apr 27 23:31 kube-controller-manager-csr.json
-rw------- 1 root root 1675 Apr 27 23:33 kube-controller-manager-key.pem
-rw-r--r-- 1 root root 1436 Apr 27 23:33 kube-controller-manager.pem
```

#### 生成kubeconfig文件

（以下是shell命令，直接在终端执行）

生成的kubeconfig文件就在/opt/kubernetes/cfg/kube-controller-manager.kubeconfig 

里面就包含了ip 断口和客户端证书

```
KUBE_CONFIG="/opt/kubernetes/cfg/kube-controller-manager.kubeconfig"
KUBE_APISERVER="https://192.168.10.170:6443"

kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
kubectl config set-credentials kube-controller-manager \
  --client-certificate=./kube-controller-manager.pem \
  --client-key=./kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-controller-manager \
  --kubeconfig=${KUBE_CONFIG}
kubectl config use-context default --kubeconfig=${KUBE_CONFIG}

```

#### 生成systemd

```
cat > /usr/lib/systemd/system/kube-controller-manager.service << EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-controller-manager.conf
ExecStart=/opt/kubernetes/bin/kube-controller-manager \$KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# systemctl daemon-reload
# systemctl enable kube-controller-manager

```

#### 启动

```
# systemctl start kube-controller-manager
# ps -ef|grep kube-controller-manager
```

### 部署kube-scheduler

#### 生成kube-scheduler证书

```
# mkdir /opt/ca/kubescheduler/
# cd /opt/ca/kubescheduler/
# cat > kube-scheduler-csr.json << EOF
{
  "CN": "system:kube-scheduler",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF

# cfssl gencert -ca=/opt/ca/apiserver/ca.pem -ca-key=/opt/ca/apiserver/ca-key.pem -config=/opt/ca/apiserver/ca-config.json -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler
[root@k8s-manual-01 kubescheduler]# ll
total 16
-rw-r--r-- 1 root root 1029 Apr 27 23:39 kube-scheduler.csr
-rw-r--r-- 1 root root  245 Apr 27 23:38 kube-scheduler-csr.json
-rw------- 1 root root 1675 Apr 27 23:39 kube-scheduler-key.pem
-rw-r--r-- 1 root root 1424 Apr 27 23:39 kube-scheduler.pem
```

#### 创建配置文件

```
# cat > /opt/kubernetes/cfg/kube-scheduler.conf << EOF
KUBE_SCHEDULER_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--leader-elect \\
--kubeconfig=/opt/kubernetes/cfg/kube-scheduler.kubeconfig \\
--bind-address=127.0.0.1"
EOF
```

•   --kubeconfig：连接apiserver配置文件

•   --leader-elect：当该组件启动多个时，自动选举（HA）

#### 生成kube-scheduler.conf文件 

直接执行脚本：

脚本一样 在/opt/kubernetes/cfg/kube-scheduler.kubeconfig下

```
# cd /opt/ca/kubescheduler/
KUBE_CONFIG="/opt/kubernetes/cfg/kube-scheduler.kubeconfig"
KUBE_APISERVER="https://192.168.10.170:6443"

kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
kubectl config set-credentials kube-scheduler \
  --client-certificate=./kube-scheduler.pem \
  --client-key=./kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-scheduler \
  --kubeconfig=${KUBE_CONFIG}
kubectl config use-context default --kubeconfig=${KUBE_CONFIG}

# cat /opt/kubernetes/cfg/kube-scheduler.kubeconfig
```

#### 生成systemd服务

```
cat > /usr/lib/systemd/system/kube-scheduler.service << EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-scheduler.conf
ExecStart=/opt/kubernetes/bin/kube-scheduler \$KUBE_SCHEDULER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# systemctl daemon-reload
# systemctl enable kube-scheduler
```

#### 启动

```
# systemctl start kube-scheduler
# ps -ef|grep  kube-scheduler
```

### 生成.kube文件 

用于连接apiserver的管理文件

#### 生成admin的证书

 生成一个admin的证书绑定到system:masters组下，这样就俱备了连接k8s管理员权限

```
# mkdir /opt/ca/admin/
# cd /opt/ca/admin/
# cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF

# cfssl gencert -ca=/opt/ca/apiserver/ca.pem -ca-key=/opt/ca/apiserver/ca-key.pem -config=/opt/ca/apiserver/ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin

# ll
total 16
-rw-r--r-- 1 root root 1009 Apr 27 23:46 admin.csr
-rw-r--r-- 1 root root  229 Apr 27 23:45 admin-csr.json
-rw------- 1 root root 1675 Apr 27 23:46 admin-key.pem
-rw-r--r-- 1 root root 1399 Apr 27 23:46 admin.pem
```

#### 生成.kube文件

 放倒默认的目录下/root/.kube/

```
# mkdir /root/.kube/
# cd /opt/ca/admin/
# 直接执行脚本
KUBE_CONFIG="/root/.kube/config"
KUBE_APISERVER="https://192.168.10.170:6443"

kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
kubectl config set-credentials cluster-admin \
  --client-certificate=./admin.pem \
  --client-key=./admin-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}
kubectl config set-context default \
  --cluster=kubernetes \
  --user=cluster-admin \
  --kubeconfig=${KUBE_CONFIG}
kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
```

#### 验证master正常不正常

```
# kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-1               Healthy   {"health":"true"}   
etcd-0               Healthy   {"health":"true"}   
etcd-2               Healthy   {"health":"true"}   
```

#### 授权kubelet-bootstrap用户允许请求证书

最后 授权kubelet-bootstrap用户允许请求证书

就是上面创建apiserver的时候指定的bootstrap 最小权限的颁发证书的用户  将这个用户绑定到system:node-bootstrapper

```
# kubectl create clusterrolebinding kubelet-bootstrap \
--clusterrole=system:node-bootstrapper \
--user=kubelet-bootstrap

clusterrolebinding.rbac.authorization.k8s.io/kubelet-bootstrap created
```

# 部署work node节点

将master也可以当作work node

## 拷贝软件包

拷贝kubelet和kube-proxy软件包

```
# cp /opt/soft/kubernetes/server/bin/kubelet /opt/kubernetes/bin
# cp /opt/soft/kubernetes/server/bin/kube-proxy /opt/kubernetes/bin
```

## 部署kubelet

### 创建配置文件

注意hostname-override名称

```
# cat > /opt/kubernetes/cfg/kubelet.conf << EOF
KUBELET_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--hostname-override=k8s-manual-01 \\
--network-plugin=cni \\
--kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \\
--bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \\
--config=/opt/kubernetes/cfg/kubelet-config.yml \\
--cert-dir=/opt/kubernetes/ssl \\
--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google-containers/pause-amd64:3.0"
EOF

```

•   --hostname-override：显示名称，集群中唯一 同主机名

•   --network-plugin：启用CNI

•   --kubeconfig：空路径，会自动生成，后面用于连接apiserver

•   --bootstrap-kubeconfig：首次启动向apiserver申请证书

•   --config：配置参数文件

•   --cert-dir：kubelet证书生成目录

•   --pod-infra-container-image：管理Pod网络容器的镜像



### 配置参数文件

```
# cat > /opt/kubernetes/cfg/kubelet-config.yml << EOF
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 0.0.0.0
port: 10250
readOnlyPort: 10255
cgroupDriver: cgroupfs
clusterDNS:
- 10.0.0.2
clusterDomain: cluster.local 
failSwapOn: false
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /opt/kubernetes/ssl/ca.pem 
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
maxOpenFiles: 1000000
maxPods: 110
EOF

```

### 生成kubelet初次加入集群引导kubeconfig文件

TOKEN 和token.csv里保持一致

```
# KUBE_CONFIG="/opt/kubernetes/cfg/bootstrap.kubeconfig"
# KUBE_APISERVER="https://192.168.10.170:6443"
# TOKEN="6fb1839685ab7dd63a6c24f24e2320d5"

# 生成 kubelet bootstrap kubeconfig 配置文件
# kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
# kubectl config set-credentials "kubelet-bootstrap" \
  --token=${TOKEN} \
  --kubeconfig=${KUBE_CONFIG}
# kubectl config set-context default \
  --cluster=kubernetes \
  --user="kubelet-bootstrap" \
  --kubeconfig=${KUBE_CONFIG}
# kubectl config use-context default --kubeconfig=${KUBE_CONFIG}

# cat /opt/kubernetes/cfg/bootstrap.kubeconfig
```

### 配置systemd服务并且起动

```
# cat > /usr/lib/systemd/system/kubelet.service << EOF
[Unit]
Description=Kubernetes Kubelet
After=docker.service

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kubelet.conf
ExecStart=/opt/kubernetes/bin/kubelet \$KUBELET_OPTS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

# systemctl daemon-reload
# systemctl enable kubelet
# systemctl start kubelet

```

起动后kubelet已经向apiserver发起了证书请求

### 将kubelet加入

在上面起动kubelet的时候，kubeletapiserver发起了证书请求

#### 同过kubectl get csr查看kubelet发起的请求

```
# kubectl get csr
NAME                                                   AGE   SIGNERNAME                                    REQUESTOR           REQUESTEDDURATION   CONDITION
node-csr-02u189QmkyD9vTS0vzw_RYgj2iKLmEdepOErvF6KylI   2s    kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   <none>              Pending
```

#### 批准kubelet证书申请并加入集群

```
# kubectl certificate approve node-csr-02u189QmkyD9vTS0vzw_RYgj2iKLmEdepOErvF6KylI
```

#### 查看节点是否加入进来

由于网络插件还没有部署，节点会没有准备就绪 NotReady

```
# kubectl get node
NAME          STATUS     ROLES    AGE   VERSION
k8s-master1   NotReady   <none>   5s    v1.22.4
```

#### tps: 如果这里-hostname-override名称写错了，需要删除node节点 并且重新部署

```
# 停止节点 删除节点
# systemctl stop kubelet
# kubectl delete node k8s-master1

#进入到证书目录 删除对应的证书
# cd /opt/kubernetes/ssl/
# rm -f kubelet-client-2022-04-28-15-28-04.pem 
# rm -f kubelet-client-current.pem

#修改正确的节点名 按照流程重新加入
```



## 部署kube-proxy

### 创建配置文件

```
# cat > /opt/kubernetes/cfg/kube-proxy.conf << EOF
KUBE_PROXY_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--config=/opt/kubernetes/cfg/kube-proxy-config.yml"
EOF

```

### 创建参数文件

注意节点名称

这里的clusterCIDR网段需要和kube-controller-manager.conf 一致

```
# cat > /opt/kubernetes/cfg/kube-proxy-config.yml << EOF
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
metricsBindAddress: 0.0.0.0:10249
clientConnection:
  kubeconfig: /opt/kubernetes/cfg/kube-proxy.kubeconfig
hostnameOverride: k8s-manual-01
clusterCIDR: 10.244.0.0/16
EOF

```

### 生成kube-proxy.kubeconfig文件

```
# mkdir /opt/ca/kubeproxy
# cd /opt/ca/kubeproxy

# 创建证书请求文件
# cat > kube-proxy-csr.json << EOF
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF

# 生成证书
# cfssl gencert -ca=/opt/ca/apiserver/ca.pem -ca-key=/opt/ca/apiserver/ca-key.pem -config=/opt/ca/apiserver/ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy

# ll
total 16
-rw-r--r-- 1 root root 1009 Apr 28 15:43 kube-proxy.csr
-rw-r--r-- 1 root root  230 Apr 28 15:42 kube-proxy-csr.json
-rw------- 1 root root 1679 Apr 28 15:43 kube-proxy-key.pem
-rw-r--r-- 1 root root 1403 Apr 28 15:43 kube-proxy.pem

```

### 生成kubeconfig文件

```
# KUBE_CONFIG="/opt/kubernetes/cfg/kube-proxy.kubeconfig"
# KUBE_APISERVER="https://192.168.10.170:6443"

# kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
# kubectl config set-credentials kube-proxy \
  --client-certificate=./kube-proxy.pem \
  --client-key=./kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}
# kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=${KUBE_CONFIG}
# kubectl config use-context default --kubeconfig=${KUBE_CONFIG}

# cat /opt/kubernetes/cfg/kube-proxy.kubeconfig
```

### 创建systemd服务

```
# cat > /usr/lib/systemd/system/kube-proxy.service << EOF
[Unit]
Description=Kubernetes Proxy
After=network.target

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-proxy.conf
ExecStart=/opt/kubernetes/bin/kube-proxy \$KUBE_PROXY_OPTS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF


# systemctl daemon-reload
# systemctl enable kube-proxy
```

### 启动

```
# systemctl start kube-proxy
# ps -ef|grep kube-proxy
```

## 部署calico网络组件

Calico是一个纯三层的数据中心网络方案，是目前Kubernetes主流的网络方案。

严重依赖kubelet和kube-proxy

### 部署Calico

calico.yaml需要修改CALICO_IPV4POOL_CIDR 为clusterCIDR网段

如果是多网卡在calico-node下 需要指定网卡

```
- name: CALICO_IPV4POOL_VXLAN
  value: "Never"
```



```
# kubectl apply -f calico.yaml

#查看是否成功 等待节点running状态
# kubectl get pods -n kube-system
```

### 查看node节点是否为ready

calico部署完成后 节点就会是Ready  

```
# kubectl get nodes
NAME            STATUS   ROLES    AGE    VERSION
k8s-manual-01   Ready    <none>   3h8m   v1.22.4
```



## 授权apiserver访问kubelet

例如kubectl logs 可以使用

```
# cat > apiserver-to-kubelet-rbac.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
      - pods/log
    verbs:
      - "*"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF

kubectl apply -f apiserver-to-kubelet-rbac.yaml

```

## 新增work节点

### 将msater节点文件拷贝到work上

```
# scp -r /opt/kubernetes root@192.168.10.171:/opt/
# scp -r /usr/lib/systemd/system/{kubelet,kube-proxy}.service root@192.168.10.171:/usr/lib/systemd/system
# scp /opt/kubernetes/ssl/ca.pem root@192.168.10.171:/opt/kubernetes/ssl
```

### 删除多余的文件

在新增的节点里操作

删除kubelet证书和kubeconfig文件

这几个文件是证书申请后自动生成的，每个Node不同，必须删除

```
# rm -f /opt/kubernetes/cfg/kubelet.kubeconfig 
# rm -f /opt/kubernetes/ssl/kubelet*

#删除其他日志
# rm -f /opt/kubernetes/logs/*

#删除master的文件
# rm -f /opt/kubernetes/bin/kube-apiserver
# rm -f /opt/kubernetes/bin/kube-controller-manager
# rm -f /opt/kubernetes/bin/kubectl
# rm -f /opt/kubernetes/bin/kube-scheduler
# rm -f /opt/kubernetes/cfg/kube-apiserver.conf
# rm -f /opt/kubernetes/cfg/kube-controller-manager.*
# rm -f /opt/kubernetes/cfg/kube-scheduler.*
```

### 修改配置文件的节点名

kubelet.conf和kube-proxy-config

--hostname-override=k8s-manual-02 

```
# vim /opt/kubernetes/cfg/kubelet.conf
KUBELET_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--hostname-override=k8s-manual-02 \
--network-plugin=cni \
--kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \
--bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \
--config=/opt/kubernetes/cfg/kubelet-config.yml \
--cert-dir=/opt/kubernetes/ssl \
--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google-containers/pause-amd64:3.0"

# vim /opt/kubernetes/cfg/kube-proxy-config.yml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
metricsBindAddress: 0.0.0.0:10249
clientConnection:
  kubeconfig: /opt/kubernetes/cfg/kube-proxy.kubeconfig
hostnameOverride: k8s-manual-02
clusterCIDR: 10.244.0.0/16
```

### 设置systemd服务 并且起动

```
# systemctl daemon-reload
# systemctl enable kubelet kube-proxy
# systemctl start kubelet kube-proxy
```

#### master节点查看请求并且同过

```
# kubectl get csr
NAME                                                   AGE   SIGNERNAME                                    REQUESTOR           REQUESTEDDURATION   CONDITION
node-csr-HvX9KjbwnF51RSW40yi9Qai9fKRZunzcfyBO7gvWQOc   10m   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   <none>              Approved,Issued
node-csr-yfJh4ehpnWVPYPv1ZuIBxdHWG84ksvcn4OEg_zMBDjo   8s    kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   <none>              Pending

#同过授权请求
# kubectl certificate approve node-csr-yfJh4ehpnWVPYPv1ZuIBxdHWG84ksvcn4OEg_zMBDjo
```

#### 查看状态

```
# kubectl get node
NAME            STATUS     ROLES    AGE     VERSION
k8s-manual-01   Ready      <none>   3h42m   v1.22.4
k8s-manual-02   Ready      <none>   4m52s   v1.22.4
```

### 同理 新增k8s-manual-03

# 部署Dashboard

## 部署

```
# cd /opt/soft
# kubectl apply -f kubernetes-dashboard.yaml
# 查看部署
# kubectl get pods,svc -n kubernetes-dashboard
```

## 创建管理员角色

创建service account并绑定默认cluster-admin管理员集群角色

```
kubectl create serviceaccount dashboard-admin -n kube-system
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')
```

# 部署CoreDNS

k8s内部的一个dns 主要为内部service资源做dns解析的

```
# cd /opt/soft
# kubectl apply -f coredns.yaml 

# kubectl get pods -n kube-system  
```

# 扩容多Master

扩容多Master（高可用架构）

Kubernetes作为容器集群系统，通过健康检查+重启策略实现了Pod故障自我修复能力，通过调度算法实现将Pod分布式部署，并保持预期副本数，根据Node失效状态自动在其他Node拉起Pod，实现了应用层的高可用性。 

针对Kubernetes集群，高可用性还应包含以下两个层面的考虑：Etcd数据库的高可用性和Kubernetes Master组件的高可用性。 而Etcd我们已经采用3个节点组建集群实现高可用，本节将对Master节点高可用进行说明和实施。

Master节点扮演着总控中心的角色，通过不断与工作节点上的Kubelet和kube-proxy进行通信来维护整个集群的健康工作状态。如果Master节点故障，将无法使用kubectl工具或者API做任何集群管理。

Master节点主要有三个服务kube-apiserver、kube-controller-manager和kube-scheduler，其中kube-controller-manager和kube-scheduler组件自身通过选择机制已经实现了高可用，所以Master高可用主要针对kube-apiserver组件，而该组件是以HTTP API提供服务，因此对他高可用与Web服务器类似，增加负载均衡器对其负载均衡即可，并且可水平扩容。

## 说明

将k8s-manual-04作为新增的master节点

## 准备程序包

将k8s-manual-01的master相关组件全部拷贝到k8s-manual-04

```
# k8s-manual-01执行
# scp /usr/bin/docker* root@192.168.10.173:/usr/bin
# scp /usr/bin/runc root@192.168.10.173:/usr/bin
# scp /usr/bin/containerd* root@192.168.10.173:/usr/bin
# scp /usr/lib/systemd/system/docker.service root@192.168.10.173:/usr/lib/systemd/system
# scp -r /etc/docker root@192.168.10.173:/etc

# 在k8s-manual-04启动Docker
# systemctl daemon-reload
# systemctl enable docker
# systemctl start docker
# docker info
```

## 准备etcd证书和master的配置程序

```
# 在k8s-manual-04执行
# mkdir -p /opt/etcd/ssl

# k8s-manual-01执行
# scp -r /opt/kubernetes root@192.168.10.173:/opt
# scp -r /opt/etcd/ssl root@192.168.10.173:/opt/etcd
# scp /usr/lib/systemd/system/kube* root@192.168.10.173:/usr/lib/systemd/system
# scp /usr/bin/kubectl  root@192.168.10.173:/usr/bin
# scp -r ~/.kube root@192.168.10.173:~

#删除kubelet证书和kubeconfig文件 k8s-manual-04执行
# rm -f /opt/kubernetes/cfg/kubelet.kubeconfig 
# rm -f /opt/kubernetes/ssl/kubelet*

```

## 修改k8s-manual-04的master的配置文件

### 修改kube-apiserver.conf

...
 --bind-address=192.168.10.173 \
 --advertise-address=192.168.10.173 \
 ...

```
# vim /opt/kubernetes/cfg/kube-apiserver.conf
KUBE_APISERVER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--etcd-servers=https://192.168.10.170:2379,https://192.168.10.171:2379,https://192.168.10.172:2379 \
KUBE_APISERVER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--etcd-servers=https://192.168.10.170:2379,https://192.168.10.171:2379,https://192.168.10.172:2379 \
--bind-address=192.168.10.173 \
--secure-port=6443 \
--advertise-address=192.168.10.173 \
--allow-privileged=true \
--service-cluster-ip-range=10.0.0.0/24 \
--enable-admission-plugins=NodeRestriction \
--authorization-mode=RBAC,Node \
--enable-bootstrap-token-auth=true \
--token-auth-file=/opt/kubernetes/cfg/token.csv \
--service-node-port-range=30000-32767 \
--kubelet-client-certificate=/opt/kubernetes/ssl/server-apiserver.pem \
--kubelet-client-key=/opt/kubernetes/ssl/server-apiserver-key.pem \
--tls-cert-file=/opt/kubernetes/ssl/server-apiserver.pem  \
--tls-private-key-file=/opt/kubernetes/ssl/server-apiserver-key.pem \
--client-ca-file=/opt/kubernetes/ssl/ca.pem \
--service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \
--service-account-issuer=api \
--service-account-signing-key-file=/opt/kubernetes/ssl/ca-key.pem \
--etcd-cafile=/opt/etcd/ssl/ca.pem \
--etcd-certfile=/opt/etcd/ssl/server.pem \
--etcd-keyfile=/opt/etcd/ssl/server-key.pem \
--requestheader-client-ca-file=/opt/kubernetes/ssl/ca.pem \
--proxy-client-cert-file=/opt/kubernetes/ssl/server-apiserver.pem \
--proxy-client-key-file=/opt/kubernetes/ssl/server-apiserver-key.pem \
--requestheader-allowed-names=kubernetes \
--requestheader-extra-headers-prefix=X-Remote-Extra- \
--requestheader-group-headers=X-Remote-Group \
--requestheader-username-headers=X-Remote-User \
--enable-aggregator-routing=true \
--audit-log-maxage=30 \
--audit-log-maxbackup=3 \
--audit-log-maxsize=100 \
--audit-log-path=/opt/kubernetes/logs/k8s-audit.log"


```

### 修改kube-controller-manager.kubeconfig

​    server: https://192.168.10.173:6443

```
# vim /opt/kubernetes/cfg/kube-controller-manager.kubeconfig
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUR2akNDQXFhZ0F3SUJBZ0lVQ3M2T3VBSWpTZlZTRVNQdzk5Y09QZWtzN3o4d0RRWUpLb1pJaHZjTkFRRU
wKQlFBd1pURUxNQWtHQTFVRUJoTUNRMDR4RURBT0JnTlZCQWdUQjBKbGFXcHBibWN4RURBT0JnTlZCQWNUQjBKbAphV3BwYm1jeEREQUtCZ05WQkFvVEEyczRjekVQTUEwR0ExVUVDeE1HVTNsemRHVnRN
Uk13RVFZRFZRUURFd3ByCmRXSmxjbTVsZEdWek1CNFhEVEl5TURReU56RTBORGd3TUZvWERUSTNNRFF5TmpFME5EZ3dNRm93WlRFTE1Ba0cKQTFVRUJoTUNRMDR4RURBT0JnTlZCQWdUQjBKbGFXcHBibW
N4RURBT0JnTlZCQWNUQjBKbGFXcHBibWN4RERBSwpCZ05WQkFvVEEyczRjekVQTUEwR0ExVUVDeE1HVTNsemRHVnRNUk13RVFZRFZRUURFd3ByZFdKbGNtNWxkR1Z6Ck1JSUJJakFOQmdrcWhraUc5dzBC
QVFFRkFBT0NBUThBTUlJQkNnS0NBUUVBMXJnSElDRzdqZ1JFTEdmcFIvSGUKNXJHS1o4eS9wQlVQZTZRcllLL2drN0RoeDYrM2p1Y2ZyN0UxbzVBYk0vQmlPSzNXR3ZqLytJTUVtTjcwYmFSeQpqWXNGUG
FGUnp1dG51K2pWYzdSYStkTzB0cllmZE9jVVIyWCtnU1BsZmgrQWt0OXVCSGJhOFVYSXZBMHhNWVlPClZsZEx6Z0RKMjMxem8xQjF5Mng2aFU5QWlTWXI2elhBb0M5bWhxYStNV3M4NkZhZk9iYTNvd2Vv
VUNacWYvNnoKcUJhU25Kb21wUlhuNmVXU1poam1idmFvU0hQb2VNZHRTeHhNT3FwQy9saFNlaThVaEozb1BtZWgxT05pUlRnagpwM0ZMQ3gwSVBjT0txNk5RYW9ZMHIxVUhBTTZpcFdiMCtVdE9HOGtNRk
NWcGo5am9HVEdSeFFkd2M0Vm9KY0J6ClZ3SURBUUFCbzJZd1pEQU9CZ05WSFE4QkFmOEVCQU1DQVFZd0VnWURWUjBUQVFIL0JBZ3dCZ0VCL3dJQkFqQWQKQmdOVkhRNEVGZ1FVV1VXUTdRYzNaMDNwK0RU
V1puZmExdDFaVmFzd0h3WURWUjBqQkJnd0ZvQVVXVVdRN1FjMwpaMDNwK0RUV1puZmExdDFaVmFzd0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFLQ2cvUmI4dk1ZeldnQ3hqWWo1ClBkdmQ5WUFIQVN1MF
hTRVZQUEpwS3JjYndaRHppNmhwUDhWanRpK1cwQTg3eldUL1plSXdJc0hRTlRYMGpKVUsKYzdoM2JIazEyTEhLbkdHTjgxZ05MSytxaEowNnorUFdLK2dkVCtMcWpoNFNWdm00L2R3endwcmg3a0pITlBu
cwpxR3BLcnlyald5MUtldE1nMDhlOVk2QjhDUFdqc0FMSWc5Nm9hVk9pWjdDdEMxd2lqOEFBa2FrRWhad0psSEpICkxEU1dWMUQ5UWtYZDJEOWZCajFXbEJmMzA5eFBtWlVPYkVIcmkrWHJVbjB0V1VveD
JZQVluOXdpVVFGQlFOZTgKcHdNZlJLZGlmNkxJQ1BHUGlHUTNDTmZZRjFjSlBMN3pGd0NqT00wMG9xNWZ2clpaY1pYOHUyaXBwaDJxVENGagpXWk09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    server: https://192.168.10.173:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kube-controller-manager
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: kube-controller-manager
  user:
```

### 修改kube-scheduler.kubeconfig

​    server: https://192.168.10.173:6443

```
# vim /opt/kubernetes/cfg/kube-scheduler.kubeconfig
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUR2akNDQXFhZ0F3SUJBZ0lVQ3M2T3VBSWpTZlZTRVNQdzk5Y09QZWtzN3o4d0RRWUpLb1pJaHZjTkFRRU
wKQlFBd1pURUxNQWtHQTFVRUJoTUNRMDR4RURBT0JnTlZCQWdUQjBKbGFXcHBibWN4RURBT0JnTlZCQWNUQjBKbAphV3BwYm1jeEREQUtCZ05WQkFvVEEyczRjekVQTUEwR0ExVUVDeE1HVTNsemRHVnRN
Uk13RVFZRFZRUURFd3ByCmRXSmxjbTVsZEdWek1CNFhEVEl5TURReU56RTBORGd3TUZvWERUSTNNRFF5TmpFME5EZ3dNRm93WlRFTE1Ba0cKQTFVRUJoTUNRMDR4RURBT0JnTlZCQWdUQjBKbGFXcHBibW
N4RURBT0JnTlZCQWNUQjBKbGFXcHBibWN4RERBSwpCZ05WQkFvVEEyczRjekVQTUEwR0ExVUVDeE1HVTNsemRHVnRNUk13RVFZRFZRUURFd3ByZFdKbGNtNWxkR1Z6Ck1JSUJJakFOQmdrcWhraUc5dzBC
QVFFRkFBT0NBUThBTUlJQkNnS0NBUUVBMXJnSElDRzdqZ1JFTEdmcFIvSGUKNXJHS1o4eS9wQlVQZTZRcllLL2drN0RoeDYrM2p1Y2ZyN0UxbzVBYk0vQmlPSzNXR3ZqLytJTUVtTjcwYmFSeQpqWXNGUG
FGUnp1dG51K2pWYzdSYStkTzB0cllmZE9jVVIyWCtnU1BsZmgrQWt0OXVCSGJhOFVYSXZBMHhNWVlPClZsZEx6Z0RKMjMxem8xQjF5Mng2aFU5QWlTWXI2elhBb0M5bWhxYStNV3M4NkZhZk9iYTNvd2Vv
VUNacWYvNnoKcUJhU25Kb21wUlhuNmVXU1poam1idmFvU0hQb2VNZHRTeHhNT3FwQy9saFNlaThVaEozb1BtZWgxT05pUlRnagpwM0ZMQ3gwSVBjT0txNk5RYW9ZMHIxVUhBTTZpcFdiMCtVdE9HOGtNRk
NWcGo5am9HVEdSeFFkd2M0Vm9KY0J6ClZ3SURBUUFCbzJZd1pEQU9CZ05WSFE4QkFmOEVCQU1DQVFZd0VnWURWUjBUQVFIL0JBZ3dCZ0VCL3dJQkFqQWQKQmdOVkhRNEVGZ1FVV1VXUTdRYzNaMDNwK0RU
V1puZmExdDFaVmFzd0h3WURWUjBqQkJnd0ZvQVVXVVdRN1FjMwpaMDNwK0RUV1puZmExdDFaVmFzd0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFLQ2cvUmI4dk1ZeldnQ3hqWWo1ClBkdmQ5WUFIQVN1MF
hTRVZQUEpwS3JjYndaRHppNmhwUDhWanRpK1cwQTg3eldUL1plSXdJc0hRTlRYMGpKVUsKYzdoM2JIazEyTEhLbkdHTjgxZ05MSytxaEowNnorUFdLK2dkVCtMcWpoNFNWdm00L2R3endwcmg3a0pITlBu
cwpxR3BLcnlyald5MUtldE1nMDhlOVk2QjhDUFdqc0FMSWc5Nm9hVk9pWjdDdEMxd2lqOEFBa2FrRWhad0psSEpICkxEU1dWMUQ5UWtYZDJEOWZCajFXbEJmMzA5eFBtWlVPYkVIcmkrWHJVbjB0V1VveD
JZQVluOXdpVVFGQlFOZTgKcHdNZlJLZGlmNkxJQ1BHUGlHUTNDTmZZRjFjSlBMN3pGd0NqT00wMG9xNWZ2clpaY1pYOHUyaXBwaDJxVENGagpXWk09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    server: https://192.168.10.173:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kube-scheduler
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: kube-scheduler
  user:
```



### 修改kubelet.conf

--hostname-override=k8s-manual-04

```
# vim /opt/kubernetes/cfg/kubelet.conf
KUBELET_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--hostname-override=k8s-manual-04 \
--network-plugin=cni \
--kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \
--bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \
--config=/opt/kubernetes/cfg/kubelet-config.yml \
--cert-dir=/opt/kubernetes/ssl \
--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google-containers/pause-amd64:3.0"
```

### 修改kube-proxy-config.yml

hostnameOverride: k8s-manual-04

```
# vim /opt/kubernetes/cfg/kube-proxy-config.yml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
metricsBindAddress: 0.0.0.0:10249
clientConnection:
  kubeconfig: /opt/kubernetes/cfg/kube-proxy.kubeconfig
hostnameOverride: k8s-manual-04
clusterCIDR: 10.244.0.0/16
```

### 修改~/.kube/config

​    server: https://192.168.10.173:6443

```
# vim  ~/.kube/config
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUR2akNDQXFhZ0F3SUJBZ0lVQ3M2T3VBSWpTZlZTRVNQdzk5Y09QZWtzN3o4d0RRWUpLb1pJaHZjTkFRRU
wKQlFBd1pURUxNQWtHQTFVRUJoTUNRMDR4RURBT0JnTlZCQWdUQjBKbGFXcHBibWN4RURBT0JnTlZCQWNUQjBKbAphV3BwYm1jeEREQUtCZ05WQkFvVEEyczRjekVQTUEwR0ExVUVDeE1HVTNsemRHVnRN
Uk13RVFZRFZRUURFd3ByCmRXSmxjbTVsZEdWek1CNFhEVEl5TURReU56RTBORGd3TUZvWERUSTNNRFF5TmpFME5EZ3dNRm93WlRFTE1Ba0cKQTFVRUJoTUNRMDR4RURBT0JnTlZCQWdUQjBKbGFXcHBibW
N4RURBT0JnTlZCQWNUQjBKbGFXcHBibWN4RERBSwpCZ05WQkFvVEEyczRjekVQTUEwR0ExVUVDeE1HVTNsemRHVnRNUk13RVFZRFZRUURFd3ByZFdKbGNtNWxkR1Z6Ck1JSUJJakFOQmdrcWhraUc5dzBC
QVFFRkFBT0NBUThBTUlJQkNnS0NBUUVBMXJnSElDRzdqZ1JFTEdmcFIvSGUKNXJHS1o4eS9wQlVQZTZRcllLL2drN0RoeDYrM2p1Y2ZyN0UxbzVBYk0vQmlPSzNXR3ZqLytJTUVtTjcwYmFSeQpqWXNGUG
FGUnp1dG51K2pWYzdSYStkTzB0cllmZE9jVVIyWCtnU1BsZmgrQWt0OXVCSGJhOFVYSXZBMHhNWVlPClZsZEx6Z0RKMjMxem8xQjF5Mng2aFU5QWlTWXI2elhBb0M5bWhxYStNV3M4NkZhZk9iYTNvd2Vv
VUNacWYvNnoKcUJhU25Kb21wUlhuNmVXU1poam1idmFvU0hQb2VNZHRTeHhNT3FwQy9saFNlaThVaEozb1BtZWgxT05pUlRnagpwM0ZMQ3gwSVBjT0txNk5RYW9ZMHIxVUhBTTZpcFdiMCtVdE9HOGtNRk
NWcGo5am9HVEdSeFFkd2M0Vm9KY0J6ClZ3SURBUUFCbzJZd1pEQU9CZ05WSFE4QkFmOEVCQU1DQVFZd0VnWURWUjBUQVFIL0JBZ3dCZ0VCL3dJQkFqQWQKQmdOVkhRNEVGZ1FVV1VXUTdRYzNaMDNwK0RU
V1puZmExdDFaVmFzd0h3WURWUjBqQkJnd0ZvQVVXVVdRN1FjMwpaMDNwK0RUV1puZmExdDFaVmFzd0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFLQ2cvUmI4dk1ZeldnQ3hqWWo1ClBkdmQ5WUFIQVN1MF
hTRVZQUEpwS3JjYndaRHppNmhwUDhWanRpK1cwQTg3eldUL1plSXdJc0hRTlRYMGpKVUsKYzdoM2JIazEyTEhLbkdHTjgxZ05MSytxaEowNnorUFdLK2dkVCtMcWpoNFNWdm00L2R3endwcmg3a0pITlBu
cwpxR3BLcnlyald5MUtldE1nMDhlOVk2QjhDUFdqc0FMSWc5Nm9hVk9pWjdDdEMxd2lqOEFBa2FrRWhad0psSEpICkxEU1dWMUQ5UWtYZDJEOWZCajFXbEJmMzA5eFBtWlVPYkVIcmkrWHJVbjB0V1VveD
JZQVluOXdpVVFGQlFOZTgKcHdNZlJLZGlmNkxJQ1BHUGlHUTNDTmZZRjFjSlBMN3pGd0NqT00wMG9xNWZ2clpaY1pYOHUyaXBwaDJxVENGagpXWk09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    server: https://192.168.10.173:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: cluster-admin
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: cluster-admin
  user:
```

## 配置systemd服务并且开机起动

```
# systemctl daemon-reload
# systemctl start kube-apiserver kube-controller-manager kube-scheduler kubelet kube-proxy
# systemctl enable kube-apiserver kube-controller-manager kube-scheduler kubelet kube-proxy
```

## 查看集群状态

查看是否运行成功,此刻作为一个node加入到集群的

```
# kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"}   
etcd-2               Healthy   {"health":"true"}   
```

## 批准kubelet证书 加入到集群

```
# kubectl get csr
NAME                                                   AGE    SIGNERNAME                                    REQUESTOR           REQUESTEDDURATION   CONDITION
node-csr-HvX9KjbwnF51RSW40yi9Qai9fKRZunzcfyBO7gvWQOc   106m   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   <none>              Approved,Issued
node-csr-S3KLTzLtIsnPUkjMIG2b5wVljaNR0uEE62fWUOTMRk0   91m    kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   <none>              Approved,Issued
node-csr-i1BJW6ksXwjnfvp_Qf-J7Z8jtZLw3ZKaiBGcvcc6C28   84s    kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   <none>              Pending
node-csr-yfJh4ehpnWVPYPv1ZuIBxdHWG84ksvcn4OEg_zMBDjo   96m    kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   <none>              Approved,Issued


[root@k8s-manual-04 ~]# kubectl certificate approve node-csr-i1BJW6ksXwjnfvp_Qf-J7Z8jtZLw3ZKaiBGcvcc6C28

# 此刻就有了
# kubectl get nodes
NAME            STATUS     ROLES    AGE     VERSION
k8s-manual-01   Ready      <none>   5h14m   v1.22.4
k8s-manual-02   Ready      <none>   96m     v1.22.4
k8s-manual-03   Ready      <none>   91m     v1.22.4
k8s-manual-04   Ready   <none>   28s     v1.22.4
```

至此 master第二台已经创建成功

# 部署master高可用

Nginx+Keepalived实现  实现代理apiserver的请求地址

## 安装组件

 k8s-manual-01和 k8s-manual-04执行

```
# yum install epel-release -y
# yum install nginx nginx-all-modules.noarch keepalived -y
```

## 配置nginx

两台master执行 k8s-manual-01和 k8s-manual-04

```
# cat > /etc/nginx/nginx.conf << "EOF"
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

# 四层负载均衡，为两台Master apiserver组件提供负载均衡
stream {

    log_format  main  '$remote_addr $upstream_addr - [$time_local] $status $upstream_bytes_sent';

    access_log  /var/log/nginx/k8s-access.log  main;

    upstream k8s-apiserver {
       server 192.168.10.170:6443;   # Master1 APISERVER IP:PORT
       server 192.168.10.173:6443;   # Master2 APISERVER IP:PORT
    }
    
    server {
       listen 16443; # 由于nginx与master节点复用，这个监听端口不能是6443，否则会冲突
       proxy_pass k8s-apiserver;
    }
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    server {
        listen       80 default_server;
        server_name  _;

        location / {
        }
    }
}
EOF

```

## 配置keepalived

两台master执行 k8s-manual-01和 k8s-manual-04

•   vrrp_script：指定检查nginx工作状态脚本（根据nginx状态判断是否故障转移）

•   virtual_ipaddress：虚拟IP（VIP）

注意：vip必须要包含在签发的证书里

```
# cat > /etc/keepalived/keepalived.conf << EOF
global_defs { 
   notification_email { 
     acassen@firewall.loc 
     failover@firewall.loc 
     sysadmin@firewall.loc 
   } 
   notification_email_from Alexandre.Cassen@firewall.loc  
   smtp_server 127.0.0.1 
   smtp_connect_timeout 30 
   router_id NGINX_MASTER
} 

vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
}

vrrp_instance VI_1 { 
    state MASTER 
    interface ens192  # 修改为实际网卡名
    virtual_router_id 51 # VRRP 路由 ID实例，每个实例是唯一的 
    priority 100    # 优先级，备服务器设置 90 这里 k8s-manual-04是备
    advert_int 1    # 指定VRRP 心跳包通告间隔时间，默认1秒 
    authentication { 
        auth_type PASS      
        auth_pass 1111 
    }  
    # 虚拟IP
    virtual_ipaddress { 
        192.168.10.177/24
    } 
    track_script {
        check_nginx
    } 
}
EOF

# chmod +x /etc/keepalived/check_nginx.sh
```

## nginx健康检查

检查nginx运行状态的脚本 检查断口是否存在 如果不存在kill掉

```
# cat > /etc/keepalived/check_nginx.sh  << "EOF"
#!/bin/bash
count=$(ss -antp |grep 16443 |egrep -cv "grep|$$")

if [ "$count" -eq 0 ];then
    exit 1
else
    exit 0
fi
EOF

# chmod +x /etc/keepalived/check_nginx.sh

```

## 设置systemd服务

```
# systemctl daemon-reload
# systemctl enable nginx keepalived
# systemctl start nginx keepalived
```

## 查看keepalived工作状态

ip已经绑定

```
# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:0c:29:a3:f7:1c brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.170/24 brd 192.168.10.255 scope global noprefixroute ens192
       valid_lft forever preferred_lft forever
    inet 192.168.10.177/24 scope global secondary ens192
       valid_lft forever preferred_lft forever
    inet6 fe80::ed5a:2331:cb29:a940/64 scope link tentative noprefixroute dadfailed 
       valid_lft forever preferred_lft forever
    inet6 fe80::e38a:e3dd:7339:6bd8/64 scope link tentative noprefixroute dadfailed 
       valid_lft forever preferred_lft forever
    inet6 fe80::5253:dcd7:bd69:92a7/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:a9:93:1b:e0 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
4: calida63d37ce8f@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link 
       valid_lft forever preferred_lft forever
5: cali1cb02fa15f7@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link 
       valid_lft forever preferred_lft forever
6: tunl0@NONE: <NOARP,UP,LOWER_UP> mtu 1440 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
    inet 10.244.143.194/32 brd 10.244.143.194 scope global tunl0
       valid_lft forever preferred_lft forever
```

## 停止任一一台nginx 检查是否切换vip成功

```
# systemctl start nginx
```

## 检查是否能获取到apiserver信息

```
# curl -k https://192.168.10.177:16443/version
{
  "major": "1",
  "minor": "22",
  "gitVersion": "v1.22.4",
  "gitCommit": "b695d79d4f967c403a96986f1750a35eb75e75f1",
  "gitTreeState": "clean",
  "buildDate": "2021-11-17T15:42:41Z",
  "goVersion": "go1.16.10",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

## 最后，将所有node节点的apiserver地址

### 将所有node节点的apiserver地址修改到vip上 4台节点的work node都要执行

```
# sed -i 's#192.168.10.170:6443#192.168.10.177:16443#' /opt/kubernetes/cfg/*
```

### 重启服务

```
# systemctl restart kubelet kube-proxy
```

### 查看集群状态是否正常

```
# kubectl get nodes
NAME            STATUS   ROLES    AGE     VERSION
k8s-manual-01   Ready    <none>   5h47m   v1.22.4
k8s-manual-02   Ready    <none>   129m    v1.22.4
k8s-manual-03   Ready    <none>   124m    v1.22.4
k8s-manual-04   Ready    <none>   33m     v1.22.4
```

### 回退

```
sed -i 's#192.168.10.177:16443#192.168.10.170:6443#' /opt/kubernetes/cfg/*
systemctl restart kubelet kube-proxy
```



# 问题集和

## 问题1

```
# 创建calico包错
Warning  Unhealthy  6m29s (x265 over 91m)  kubelet  Readiness probe failed: calico/node is not ready: BIRD is not ready: Failed to stat() nodename file: stat /var/lib/calico/nodename: no such file or directory
 
# 解决
手动创建/var/lib/calico/nodename既可
主要原因:calico 没有自己创建的nodename文件导致检测是未发现nodename信息所以导致报错
# echo "k8s-manual-01"  >   /var/lib/calico/nodename
```

## 问题2

如果pod删不掉可以强制删除

```
kubectl delete pods <pod> --grace-period=0 --force
```

## 问题3

问题：retry error=Get https://10.0.0.1:443/api/v1/nodes/foo: dial tcp 10.0.0.1:443: i/o timeout

解决方案

查看iptable的转发：iptables -t nat -nL

检查/kube-apiserver.conf配置文件的ip是否正确，这几个ip直接应现iptable转发规则包括

--advertise-address

-service-cluster-ip-range

--bind-address



