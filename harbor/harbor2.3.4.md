# pw参考文档

官网：https://goharbor.io/

安装文档：https://goharbor.io/docs/2.2.0/install-config/download-installer/

github：https://github.com/goharbor/harbor

# 环境准备

|  系统类型  |    IP地址     | 节点角色 | CPU  | Memory | Hostname |
| :--------: | :-----------: | :------: | :--: | :----: | :------: |
| centos-7.9 | 192.168.10.60 |          | \>=2 | \>=2G  | harbor1  |
| centos-7.9 | 192.168.10.61 |          | \>=2 | \>=2G  | harbor2  |

* 采用双主复制的，前端负载nginx的方式进行部署

* harbor版本：2.3.4

1. 修改hostname

   ```
   # hostnamectl set-hostname harbor1
   # hostnamectl set-hostname harbor2
   ```

2. 关闭防火墙、selinux、swap，重置iptables

   ```bash
   # 关闭selinux
   # setenforce 0
   # sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config
   # 关闭防火墙
   # systemctl stop firewalld && systemctl disable firewalld
   
   # 关闭swap
   # swapoff -a && free –h
   
   #或者在这里删除swap挂载
   # vim /etc/fstab
   
   # 关闭dnsmasq(否则可能导致容器无法解析域名)
   # service dnsmasq stop && systemctl disable dnsmasq
   ```

3. 安装docker 两台机器同时安装

   ```
   # yum install -y yum-utils
   # yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
   # yum install -y docker-ce
   # systemctl start docker && systemctl enable docker
   
   #配置镜像加速
   # cat /etc/docker/daemon.json 
   {
           "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"]
   }
   
   
   ```

4. 安装docker compose 并且mv为可执行文件(两台机器执行)

   地址：https://github.com/docker/compose/releases

   ```
   # cd /opt
   # wget https://github.com/docker/compose/releases/download/v2.1.1/docker-compose-linux-x86_64
   
   # mv docker-compose-linux-x86_64 /usr/local/bin/docker-compose
   # chmod +x /usr/local/bin/docker-compose 
   # docker-compose --version
   
   ```



# 安装

1. 上传harbor-offline-installer-v1.6.0软件包 并解压 两台节点同时执行

   ```
   # cd /opt/
   # ls
   harbor-offline-installer-v2.3.4.tgz
   
   #解压
   # tar vxf harbor-offline-installer-v2.3.4.tgz
   
   # cd harbor
   ```

2. 编辑配置文件(两台机器执行)

   ```
   # cd harbor
   # cp harbor.yml.tmpl harbor.yml
   #修改hostname为当前ip和登录密码 因为要做ha 不能使用域名 其他的不动
   # vim harbor.yml
   ……
   hostname: 192.168.10.61 #也可以写域名
   harbor_admin_password = Harbor12345
   
   #注释掉https
   #  port: 443
     # The path of cert and key files for nginx
   #  certificate: /your/certificate/path
   #  private_key: /your/private/key/path
   
   ……
   
   # 执行准备脚本
   # ./prepare
   #执行安装 如果没有这个目录 就创建/var/log/harbor/
   # ./install.sh
   ```

3. 配置https

   1. 先通过cfssljson生产一个证书

      ```
      # wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
      # wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
      # wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
      # chmod +x cfssl*
      # mv cfssl_linux-amd64 /usr/bin/cfssl
      # mv cfssljson_linux-amd64 /usr/bin/cfssljson
      # mv cfssl-certinfo_linux-amd64 /usr/bin/cfssl-certinfo
      
      ```

   2. 生成证书

      ```
      # vim ca-config.json
      
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
      
      # vim ca-csr.json 
      
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
                  "ST": "Beijing"
              }
          ]
      }
      
      #执行
      cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
      
      #准备证书信息
      # cat harbor.makj.com-csr.json 
      {
        "CN": "harbor.makj.com",
        "hosts": [],
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
      
      #生成证书
      # cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes harbor.makj.com-csr.json | cfssljson -bare harbor.makj.com
      
      #最终信息如下
      [root@harbor01 cfssl]# ll
      total 36
      -rw-r--r-- 1 root root  294 Nov 18 21:18 ca-config.json
      -rw-r--r-- 1 root root  960 Nov 18 21:18 ca.csr
      -rw-r--r-- 1 root root  212 Nov 18 21:18 ca-csr.json
      -rw------- 1 root root 1679 Nov 18 21:18 ca-key.pem
      -rw-r--r-- 1 root root 1273 Nov 18 21:18 ca.pem
      -rw-r--r-- 1 root root  964 Nov 18 21:21 harbor.makj.com.csr
      -rw-r--r-- 1 root root  188 Nov 18 21:19 harbor.makj.com-csr.json
      -rw------- 1 root root 1675 Nov 18 21:21 harbor.makj.com-key.pem
      -rw-r--r-- 1 root root 1314 Nov 18 21:21 harbor.makj.com.pem
      ```

4. 配置https

   1. 配置证书

      ```
      # vim harbor.yml
      # 修改域名
      hostname: harbor.makj.com  
      
      # http related config
      http:
        # port for http, default is 80. If https enabled, this port will redirect to https port
        port: 80
      
      # https related config
      #打开https 并且配置证书
      https:
        # https port for harbor, default is 443
        port: 443
        # The path of cert and key files for nginx
        certificate: /opt/cfssl/harbor.makj.com.pem
        private_key: /opt/cfssl/harbor.makj.com-key.pem
      ```

   2. 重新生成配置并且重新配置容器

      ```
      [root@harbor01 harbor]# ./prepare 
      [root@harbor01 harbor]# docker-compose down
      [root@harbor01 harbor]# docker-compose up -d
      # 查看容器是否启动成功
      [root@harbor01 harbor]# docker ps -a
      ```

   3. 再次进去harbor 就是https了

      ![harbor2.2.4-2](images\harbor2.2.4-2.jpg)

   ```
   
   
   ```

   

5. docker推送镜像测试

   1. http测试因为现在是http的 如果是http需要先将http地址设置为可信任

      ```
      # 修改docker配置文件
      # vim /etc/docker/daemon.json
      {
              "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"],
              "insecure-registries": ["192.168.10.60","192.168.10.61"]
      }
      
      
      
      # 查看是否配置成功
      # docker login 192.168.10.61
      Username: admin
      Password: 
      WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
      Configure a credential helper to remove this warning. See
      https://docs.docker.com/engine/reference/commandline/login/#credentials-store
      
      Login Succeeded
      
      ```

   2. 推送镜像到harbor 推送完成后查看是否存在

      ```
      # 先将镜像打个tag 注意 library这个项目目录必须在hatbor存在
      # docker tag ticket-seize-true:1.0 192.168.10.61/library/ticket-seize-true:1.0
      
      # 然后推送到仓库
      # docker push 192.168.10.61/library/ticket-seize-true:1.0
      ```

      ![harbor2.2.4-1](images\harbor2.2.4-1.jpg)

   3. https测试

      1. 将证书拷贝到docker主机

         ```
         [root@harbor01 cfssl]# scp harbor.makj.com.pem 192.168.10.50:/opt/cfssl
         ```

      2. 在docker主机上创建证书存放位置

         ```
         #目录为固定目录(/etc/docker/certs.d) docker默认会在这个目录下读取相应域名的证书
         # mkdir /etc/docker/certs.d/harbor.makj.com -p 
         # 将证书的后缀重命名为crt
         # cp ./harbor.makj.com.pem /etc/docker/certs.d/harbor.makj.com/harbor.makj.com.crt
         ```

      3. 登录测试

         ```
         # docker login harbor.makj.com
         Username: admin
         Password: 
         WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
         Configure a credential helper to remove this warning. See
         https://docs.docker.com/engine/reference/commandline/login/#credentials-store
         
         Login Succeeded
         ```

      4. 推送镜像测试

         ```
         # docker tag container-registry.oracle.com/java/serverjre:8 harbor.makj.com/library/orcalre:8
         # docker push harbor.makj.com/library/orcalre:8
         ```

      5. 页面查看

         ![harbor2.2.4-3](images\harbor2.2.4-3.jpg)

6. 主从同步

   1. 6新建一个目标

      ![harbor2.2.4-4](images\harbor2.2.4-4.jpg)

      2.新建同步规则

      ![harbor2.2.4-4](images\harbor2.2.4-5.jpg)

   3. 登录到62从查看是否复制成功

      ![harbor2.2.4-4](images\harbor2.2.4-6.jpg)

   

# 结论

双主复制的时候两边镜像是通过对比镜像签名来完成是否是同一个镜像的，如果是同一个镜像则不尽兴复制