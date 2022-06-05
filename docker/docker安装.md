# docker安装

1. 下载阿里docker镜像源(阿里云,阿里云docker镜像站:https://developer.aliyun.com/article/110806)

   ```
   [root@localhost yum.repos.d]# cd /etc/yum.repos.d
   [root@localhost yum.repos.d]# wget -P /etc/yum.repos.d/ https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
   ```

2. 更新软件包

   ```
   [root@localhost yum.repos.d]# yum -y upgrade
   ```

3. 清理现有版本

   ```
   # yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine container-selinux
   ```

4. 列出可以安装的版本选项

   ```
   [root@localhost yum.repos.d]# yum list docker-ce --showduplicates | sort -r
   ```

5. 选择版本安装,这里选择19.03.9

   ```
   [root@localhost yum.repos.d]# yum install -y docker-ce-19.03.9 docker-ce-cli-19.03.9 containerd.io -y
   ```

6. 启动docker

   ```
   [root@localhost yum.repos.d]# service docker restart
   [root@localhost yum.repos.d]# systemctl enable docker
   ```

7. 查看docker信息

   ```
   [root@localhost yum.repos.d]# docker version
   Client: Docker Engine - Community
    Version:           19.03.9
    API version:       1.40
    Go version:        go1.13.10
    Git commit:        9d988398e7
    Built:             Fri May 15 00:25:27 2020
    OS/Arch:           linux/amd64
    Experimental:      false
   
   Server: Docker Engine - Community
    Engine:
     Version:          19.03.9
     API version:      1.40 (minimum version 1.12)
     Go version:       go1.13.10
     Git commit:       9d988398e7
     Built:            Fri May 15 00:24:05 2020
     OS/Arch:          linux/amd64
     Experimental:     false
    containerd:
     Version:          1.4.4
     GitCommit:        05f951a3781f4f2c1911b05e61c160e9c30eaa8e
    runc:
     Version:          1.0.0-rc93
     GitCommit:        12644e614e25b05da6fd08a38ffa0cfe1903fdec
    docker-init:
     Version:          0.18.0
     GitCommit:        fec3683
   ```

8. 修改存储位置

   ```
   [root@k8snode4b200 docker]# vim /etc/docker/daemon.json 
   
   {
     "exec-opts": ["native.cgroupdriver=systemd"],
     "log-driver": "json-file",
     "log-opts": {
       "max-size": "100m"
     },
     "storage-driver": "overlay2"  
      "graph": "/nvme/docker" #主要是这一行
   }
   
   ```

   



# 另一种安装方式

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



# 注意

 centos8 epel 安装方式

```
[root@sm8250 yum.repos.d]# yum install -y https://mirrors.aliyun.com/epel/epel-release-latest-8.noarch.rpm
Last metadata expiration check: 0:04:23 ago on Sun 25 Apr 2021 03:54:41 PM CST.
epel-release-latest-8.noarch.rpm                                                    92 kB/s |  22 kB     00:00    
Dependencies resolved.
===================================================================================================================
 Package                      Architecture           Version                    Repository                    Size
===================================================================================================================
Upgrading:
 epel-release                 noarch                 8-10.el8                   @commandline                  22 k

Transaction Summary
===================================================================================================================
Upgrade  1 Package

Total size: 22 k
Downloading Packages:
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                           1/1 
  Running scriptlet: epel-release-8-10.el8.noarch                                                              1/1 
  Upgrading        : epel-release-8-10.el8.noarch                                                              1/2 
  Cleanup          : epel-release-8-8.el8.noarch                                                               2/2 
  Running scriptlet: epel-release-8-8.el8.noarch                                                               2/2 
  Verifying        : epel-release-8-10.el8.noarch                                                              1/2 
  Verifying        : epel-release-8-8.el8.noarch                                                               2/2 

Upgraded:
  epel-release-8-10.el8.noarch                                                                                     

Complete!
[root@sm8250 yum.repos.d]# ll
total 32
drwxr-xr-x 2 root root 4096 Apr 25 15:52 backup
-rw-r--r-- 1 root root 2595 Apr 25 15:52 CentOS-Base.repo
-rw-r--r-- 1 root root 2081 Apr 25 07:43 docker-ce.repo
-rw-r--r-- 1 root root 1177 Dec  6 05:13 epel-modular.repo
-rw-r--r-- 1 root root 1259 Dec  6 05:13 epel-playground.repo
-rw-r--r-- 1 root root 1114 Dec  6 05:13 epel.repo
-rw-r--r-- 1 root root 1276 Dec  6 05:13 epel-testing-modular.repo
-rw-r--r-- 1 root root 1213 Dec  6 05:13 epel-testing.repo
[root@sm8250 yum.repos.d]# vim epel-modular.repo 
[root@sm8250 yum.repos.d]# sed -i 's|^#baseurl=https://download.fedoraproject.org/pub|baseurl=https://mirrors.aliyun.com|' /etc/yum.repos.d/epel*
[root@sm8250 yum.repos.d]# sed -i 's|^metalink|#metalink|' /etc/yum.repos.d/epel*

```

# 镜像加速

```
# cat /etc/docker/daemon.json 
{
"exec-opts":["native.cgroupdriver=systemd"],
 "registry-mirrors": ["http://,"192.168.10.61"], //镜像加速，默认http 顺序从前到后
 "insecure-registries": ["harbor.makj.com"]  //给予http的镜像
}
```



# 常用命令

* 启动全部: `docker start $(docker ps -a | awk '{ print $1}' | tail -n +2)`
* 查看容器:`docker ps -a`
