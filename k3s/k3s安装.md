# 主机环境

k3s要求每台机器节点的主机名不能一样

|   系统    |       IP       |  角色  | 主机名 |
| :-------: | :------------: | :----: | ------ |
| centos7.9 | 192.168.10.200 | master | k3s1   |
| centos7.9 | 192.168.10.201 | master | k3s2   |
| centos7.9 | 192.168.10.202 | master | k3s2   |

* k3s work只要注册到一台k3s server就可以，因为server之间会自己同步数据
* k3s 依赖于hostname 在server上一个hostname上对应一个password，所以不能出现相同的hostname

# server安装

参考文档：https://docs.rancher.cn/docs/k3s/quick-start/_index

```
[root@k3sdemo k3s]# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -
[INFO]  Finding release for channel stable
[INFO]  Using v1.20.6+k3s1 as release
#下载sha文件和二进制文件
[INFO]  Downloading hash http://rancher-mirror.cnrancher.com/k3s/v1.20.6-k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary http://rancher-mirror.cnrancher.com/k3s/v1.20.6-k3s1/k3s
[INFO]  Verifying binary download
#将k3s安装到 /usr/local/bin/k3s
[INFO]  Installing k3s to /usr/local/bin/k3s
#创建了三个软连接到和别名到k3s方便命令直接使用
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
#清理下安装过程中的网络配置和脚本
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
#创建一个k3s的服务
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
[INFO]  systemd: Starting k3s

[root@k3sdemo k3s]# kubectl get nodes
NAME      STATUS   ROLES                  AGE     VERSION
k3sdemo   Ready    control-plane,master   3m59s   v1.20.6+k3s1

```

* k3s默认的kubectl.conf文件在/etc/rancher/k3s/k3s.yaml 下

  ```
  [root@k3sdemo ~]# cat /etc/rancher/k3s/k3s.yaml 
  apiVersion: v1
  clusters:
  - cluster:
      certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUJkakNDQVIyZ0F3SUJBZ0lCQURBS0JnZ3Foa2pPUFFRREFqQWpNU0V3SHdZRFZRUUREQmhyTTNNdGMyVnkKZG1WeUxXTmhRREUyTWpBd05UWXdNemd3SGhjTk1qRXdOVEF6TVRVek16VTRXaGNOTXpFd05UQXhNVFV6TXpVNApXakFqTVNFd0h3WURWUVFEREJock0zTXRjMlZ5ZG1WeUxXTmhRREUyTWpBd05UWXdNemd3V1RBVEJnY3Foa2pPClBRSUJCZ2dxaGtqT1BRTUJCd05DQUFUNjZweDI1UEZPTEh2Rm9wcCs4eFZ1V0VyRDVCZXpsakN1a1lnYnBtZFcKQ1ZQc2pHcjVoaVVQdU1QSnRSMFRpQ2NrZnVPUlo2WitNTHB1SEhvTTBhUmxvMEl3UURBT0JnTlZIUThCQWY4RQpCQU1DQXFRd0R3WURWUjBUQVFIL0JBVXdBd0VCL3pBZEJnTlZIUTRFRmdRVWpSWmpETk5iOVByR2F5YjlYYUtyCmsyUUVFRGd3Q2dZSUtvWkl6ajBFQXdJRFJ3QXdSQUlnVmxDa2cyN1BWSzN2dnZHT0Jwelp2cVlqcFVmUHA5Y28KSXpxb1JKc0s4cVFDSUV3alBSeUgweTc2bmwyZ2RkcFp6S0JsYStNTEtVQStST01KY2ZkR0lxcGsKLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
      server: https://127.0.0.1:6443
    name: default
  contexts:
  - context:
      cluster: default
      user: default
    name: default
  current-context: default
  kind: Config
  preferences: {}
  users:
  - name: default
    user:
      client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUJrakNDQVRlZ0F3SUJBZ0lJWm1ZUDhVbzFCODB3Q2dZSUtvWkl6ajBFQXdJd0l6RWhNQjhHQTFVRUF3d1kKYXpOekxXTnNhV1Z1ZEMxallVQXhOakl3TURVMk1ETTRNQjRYRFRJeE1EVXdNekUxTXpNMU9Gb1hEVEl5TURVdwpNekUxTXpNMU9Gb3dNREVYTUJVR0ExVUVDaE1PYzNsemRHVnRPbTFoYzNSbGNuTXhGVEFUQmdOVkJBTVRESE41CmMzUmxiVHBoWkcxcGJqQlpNQk1HQnlxR1NNNDlBZ0VHQ0NxR1NNNDlBd0VIQTBJQUJCOEE2dmZSWi9YazZZYmwKV1VXMzVHa3ZZeFFUNVE4ZnJyOGx2SzNqL09XdzkwUGxmdDAzREpsV3hJOGVXczJLeFlYbkM5UnhEQXlUa2xESgpEb1Q4aDVPalNEQkdNQTRHQTFVZER3RUIvd1FFQXdJRm9EQVRCZ05WSFNVRUREQUtCZ2dyQmdFRkJRY0RBakFmCkJnTlZIU01FR0RBV2dCVEllTHZkQ0d0VVRhcEZsZkZtdVh1bk4rWnV5VEFLQmdncWhrak9QUVFEQWdOSkFEQkcKQWlFQXhMVUZiSEU3Vms1UkpHQXNiWkZ6MFBPZ1VpUnJRSzJKRlhQcEV5Q2taSzBDSVFDd1lNV09aaHBic3VDZApWYUd2T20rUHR2bUgxS2JsblZGSUFvS0h1OWpMM1E9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCi0tLS0tQkVHSU4gQ0VSVElGSUNBVEUtLS0tLQpNSUlCZHpDQ0FSMmdBd0lCQWdJQkFEQUtCZ2dxaGtqT1BRUURBakFqTVNFd0h3WURWUVFEREJock0zTXRZMnhwClpXNTBMV05oUURFMk1qQXdOVFl3TXpnd0hoY05NakV3TlRBek1UVXpNelU0V2hjTk16RXdOVEF4TVRVek16VTQKV2pBak1TRXdId1lEVlFRRERCaHJNM010WTJ4cFpXNTBMV05oUURFMk1qQXdOVFl3TXpnd1dUQVRCZ2NxaGtqTwpQUUlCQmdncWhrak9QUU1CQndOQ0FBVEcyUHczb2JqelpiUFVIVm00TU14Qmh0NmtoMG5tc0RjVWxYamRvZ29vCmpCcVZzVHdYajQzdXlQbXpoeDFsZHV1T0FiOE5tdHZ4NndlQndvSVA1YkFPbzBJd1FEQU9CZ05WSFE4QkFmOEUKQkFNQ0FxUXdEd1lEVlIwVEFRSC9CQVV3QXdFQi96QWRCZ05WSFE0RUZnUVV5SGk3M1FoclZFMnFSWlh4WnJsNwpwemZtYnNrd0NnWUlLb1pJemowRUF3SURTQUF3UlFJZ0duZkRVZjdzMFpXeExLazRHNzB1blg3NmxhK25vME1qCmRhOWF0V1NDMGw0Q0lRQ3FIVk9QMWV6NE4rYlUwV0hjT0NhMTJGWjc4ZnVMbEZOZXRWSkVqL0g2Rnc9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
      client-key-data: LS0tLS1CRUdJTiBFQyBQUklWQVRFIEtFWS0tLS0tCk1IY0NBUUVFSUJoQ01SNHdYM3dkZFFNREVZZlVGU01PaitDOWNpSFhHeDhveTZSc2pvTkRvQW9HQ0NxR1NNNDkKQXdFSG9VUURRZ0FFSHdEcTk5Rm45ZVRwaHVWWlJiZmthUzlqRkJQbER4K3V2eVc4cmVQODViRDNRK1YrM1RjTQptVmJFang1YXpZckZoZWNMMUhFTURKT1NVTWtPaFB5SGt3PT0KLS0tLS1FTkQgRUMgUFJJVkFURSBLRVktLS0tLQo=
  
  ```

* 可以讲k3s默认到拷贝到k8s的链接内，就可以执行K8s的相关依赖文件

  ```
  [root@k3sdemo ~]# cp /etc/rancher/k3s/k3s.yaml .kube/config
  ```

# agent安装

Agent 节点用`k3s agent`进程发起的 websocket 连接注册，连接由作为代理进程一部分运行的客户端负载均衡器维护。

Agent 将使用节点集群 secret 以及随机生成的节点密码向 k3s server 注册，密码存储在 `/etc/rancher/node/password`路径下。K3s server 将把各个节点的密码存储为 Kubernetes secrets，随后的任何尝试都必须使用相同的密码。节点密码秘密存储在`kube-system`命名空间中，名称使用模板`<host>.node-password.k3s`。

1. 查询server的token

   ```
   [root@k3sdemo ~]# cat /var/lib/rancher/k3s/server/token 
   K10e7afe124bd05faa95f403d4008a76c2e20f0dbe734b469e406ac34322976c59b::server:97bae6753b46b4ed2fcd804d464d824b
   ```

2. 在agent节点上安装，流程和server都一样，就是将服务换成了agent

   ```
   [root@k3sdemo2 ~]# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_URL=https://192.168.10.198:6443 K3S_TOKEN=K10e7afe124bd05faa95f403d4008a76c2e20f0dbe734b469e406ac34322976c59b::server:97bae6753b46b4ed2fcd804d464d824b sh -
   [INFO]  Finding release for channel stable
   [INFO]  Using v1.20.6+k3s1 as release
   [INFO]  Downloading hash http://rancher-mirror.cnrancher.com/k3s/v1.20.6-k3s1/sha256sum-amd64.txt
   [INFO]  Downloading binary http://rancher-mirror.cnrancher.com/k3s/v1.20.6-k3s1/k3s
   [INFO]  Verifying binary download
   [INFO]  Installing k3s to /usr/local/bin/k3s
   [INFO]  Creating /usr/local/bin/kubectl symlink to k3s
   [INFO]  Creating /usr/local/bin/crictl symlink to k3s
   [INFO]  Creating /usr/local/bin/ctr symlink to k3s
   [INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
   [INFO]  Creating uninstall script /usr/local/bin/k3s-agent-uninstall.sh
   [INFO]  env: Creating environment file /etc/systemd/system/k3s-agent.service.env
   [INFO]  systemd: Creating service file /etc/systemd/system/k3s-agent.service
   [INFO]  systemd: Enabling k3s-agent unit
   [INFO]  systemd: Starting k3s-agent
   
   ```

3. 查看节点,节点已经进来了

   ```
   [root@k3sdemo1 ~]# kubectl get nodes
   NAME       STATUS   ROLES                  AGE     VERSION
   k3sdemo1   Ready    control-plane,master   4m43s   v1.20.6+k3s1
   k3sdemo2   Ready    <none>                 114s    v1.20.6+k3s1
   
   ```

4. agent和server通讯的是密码文件在认证，如果这个文件不存在了，通讯就是失败

   ```
   [root@k3sdemo2 ~]# cat /etc/rancher/node/password 
   d1a5c21bafafbb097aeb0eb7cca610b0
   [root@k3sdemo2 ~]# service k3s-agent restart
   
   #删掉这个文件
   [root@k3sdemo2 ~]# rm -f /etc/rancher/node/password
   
   #查看节点变成了NoReady状态
   [root@k3sdemo1 ~]# kubectl get nodes
   NAME       STATUS     ROLES                  AGE     VERSION
   k3sdemo1   Ready      control-plane,master   9m25s   v1.20.6+k3s1
   k3sdemo2   NotReady   <none>                 6m36s   v1.20.6+k3s1
   
   #解决，在server删掉k3sdemo2在server保存的在secret的password文件 这样就会重新在k3sdemo2里在进行password的获取
   [root@k3sdemo1 ~]# kubectl delete secret k3sdemo2.node-password.k3s -n kube-system
   secret "k3sdemo2.node-password.k3s" deleted
   
   #这时候发现 节点已经恢复了
   [root@k3sdemo1 ~]# kubectl get nodes
   NAME       STATUS   ROLES                  AGE     VERSION
   k3sdemo1   Ready    control-plane,master   11m     v1.20.6+k3s1
   k3sdemo2   Ready    <none>                 8m43s   v1.20.6+k3s1
   ```

# 自动部署清单

k3s在每次启动的时候，都会查询`/var/lib/rancher/k3s/server/manifests/`下有没有相应的yaml文件，如果有，就会把这些yaml文件部署起来

# 配置镜像加速

```
[root@k3sdemo1 manifests]# cat >> /etc/rancher/k3s/registries.yaml <<EOF
mirrors:
  "docker.io":
    endpoint:
      - "https://fogjl973.mirror.aliyuncs.com"
      - "https://registry-1.docker.io"
EOF
[root@k3sdemo1 manifests]# systemctl restart k3s
[root@k3sdemo1 manifests]# crictl info

#私有镜像 暂时http 不是是https 要有host映射
# vim registries.yaml
mirrors:
  "harbor.makj.com": # 仓库前缀  也可以写成*
    endpoint:
      - "http://harbor.makj.com"
```

# 网络

**K3s Server 节点的入站规则：**

| 协议 | 端口      | 源                       | 描述                         |
| ---- | --------- | ------------------------ | ---------------------------- |
| TCP  | 6443      | K3s agent 节点           | Kubernetes API Server        |
| UDP  | 8472      | K3s server 和 agent 节点 | 仅对 Flannel VXLAN 需要      |
| TCP  | 10250     | K3s server 和 agent 节点 | Kubelet metrics              |
| TCP  | 2379-2380 | K3s server 节点          | 只有嵌入式 etcd 高可用才需要 |



登录token

eyJhbGciOiJSUzI1NiIsImtpZCI6Ik9pYy12UW1DNGR6WHhodFZpLXNwdnh1dldnSmZ6UG5RNUllYjdmaEtKRDQifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLXdqbnRqIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJhNTZlNjE2ZS04ZWMyLTQ0MzctYjk2MS0yY2RmYzhmNjVlZTEiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.c6R-U8uk_grAbaIuJfKcszZEZBWp6au843lLUFpRaZZk9QN0I_nRfaymXiMGHbIu70JIA5_ToiLzo4HMQ2o-i5T9pqOtZnTH_mu6TrWNhBEA40YhrhJ8WwyEY_4rYDVIUI74P3B5dkteU9qCHLSEAK3gp-BuDKmkakSAhoEtayEZr9CofUzN5EMmruiRmLfAi6r3SOx3i7JdD-34hNkhErVV9361E-uHkNYHykE6bsQjoGP3KT4s5oP_ZIMF2nwnjaiX-EAuTj51HgtBIC9afiWikaQPAi_L7mO2JI_5mSKaPdMUpgu0PV6y_G0Dth-zoAaQn7RXRQIhOdhE33UQoQ