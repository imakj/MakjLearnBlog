# pw参考文档

官网：https://goharbor.io/

安装文档：https://goharbor.io/docs/2.2.0/install-config/download-installer/

github：https://github.com/goharbor/harbor

# 环境准备

|  系统类型  |     IP地址     |      节点角色      | CPU  | Memory | Hostname |
| :--------: | :------------: | :----------------: | :--: | :----: | :------: |
| centos-7.9 | 192.168.10.182 | master,worker,etcd | \>=2 | \>=2G  | harbor1  |
| centos-7.9 | 192.168.10.183 |    worker,etcd     | \>=2 | \>=2G  | harbor2  |

* 采用双主复制的，前端负载nginx的方式进行部署

* harbor版本：1.6.0

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

3. 安装docker

   

# 安装

1. 上传harbor-offline-installer-v1.6.0软件包 并解压 两台节点同时执行

   ```
   # mkdir /opt/harbor
   # cd /opt/harbor/
   # ls
   harbor-offline-installer-v1.6.0.tgz
   
   #解压
   # tar vxf harbor-offline-installer-v1.6.0.tgz
   
   # cd harbor
   ```

2. 上传docker compose 并且mv为可执行文件(两台机器执行)

   ```
   # pwd
   /opt/harbor
   # ll
   total 690056
   -rw-r--r-- 1 root root  11750136 May 10 16:14 docker-compose-Linux-x86_64-1.22.0
   drwxr-xr-x 4 root root       294 May 10 16:13 harbor
   -rw-r--r-- 1 root root 694863055 May 10 16:04 harbor-offline-installer-v1.6.0.tgz
   
   # mv docker-compose-Linux-x86_64-1.22.0 /usr/local/bin/docker-compose
   # chmod +x /usr/local/bin/docker-compose 
   # docker-compose --version
   ```

3. 编辑配置文件(两台机器执行)

   ```
   # cd harbor
   
   #修改hostname为当前ip和登录密码 因为要做ha 不能使用域名 其他的不动
   # vim harbor.cfg
   ……
   hostname=192.168.10.160
   harbor_admin_password = Harbor12345
   ……
   ```

4. 修改docker-compose.yml文件(两台机器执行)

   ```
   # 将所有volumes文件放到大磁盘下 如果不需要更改 则默认
   # vim docker-compose.yml 
   ```

5. 安装(两台机器执行)

   ```
   # ./install.sh
   ```

6. 安装完毕后登录harbor

   ```
   # curl http://192.168.10.160
   # curl http://192.168.10.161
   
   #用户名 admin 密码为Harbor12345
   ```

7. 部署nginx 用docker部署 编写配置文件

   ```
   [root@harbor1 harbor]# docker pull nginx:1.7.9
   [root@harbor1 harbor]# mkdir /opt/harbor/nginx
   [root@harbor1 harbor]# cd /opt/harbor/nginx
   #编辑配置文件
   [root@harbor1 nginx]# vim nginx.conf
   
   user nginx;
   worker_processes 2;
   
   error_log /var/log/nginx/error.log warn;
   
   pid /var/run/nginx.pid;
   
   events {
     worker_connections 1024;
   }
   
   http {
     upstream harbor {
       server 192.168.10.160:80;
     }
   
     server {
       listen 8080;
       location / {
         proxy_pass http://harbor;
       }
     }
   }
   
   
   ```

8. 编写启动脚本

   ```
   [root@harbor1 nginx]# vim restart.sh
   
   #!/bin/bash
   docker stop harbornginx
   
   docker rm harbornginx
   
   #挂载本地文件
   docker run -idt --net=host --name harbornginx -v /opt/harbor/nginx/nginx.conf:/etc/nginx/nginx.conf nginx:1.7.9
   
   [root@harbor1 nginx]# sh restart.sh 
   harbornginx
   harbornginx
   f701f495327f318ce8d100b1bf41e9466204db61d3bc19c158ca93576ed2d7ef
   
   #查看日志
   [root@harbor1 nginx]# docker logs f7
   
   #测试访问
   # curl http://192.168.10.160:8080/harbor/sign-in
   ```

9. 编辑docker配置文件 允许指定ip来访问

   ```
   [root@harbor1 nginx]# vim /etc/docker/daemon.json
   
   {
     "insecure-registries": ["192.168.10.160","192.168.10.161"]
   }
   
   [root@harbor1 nginx]# systemctl restart docker
   
   ```

10. 在harbor新建一个项目

    ![](https://makj-blogs-images.oss-cn-beijing.aliyuncs.com/%E6%96%B0%E5%BB%BA%E9%A1%B9%E7%9B%AE.png)

11. 创建一个harbor用户:testpush/Ttestpush123.

    ![](https://makj-blogs-images.oss-cn-beijing.aliyuncs.com/%E5%88%9B%E5%BB%BA%E7%94%A8%E6%88%B7.png)

12. 将新用户添加到项目里

    创建一个harbor用户:testpush/Ttestpush123.

    ![](https://makj-blogs-images.oss-cn-beijing.aliyuncs.com/%E6%B7%BB%E5%8A%A0%E9%A1%B9%E7%9B%AE%E6%88%90%E5%91%98.png)

13. 在harbor新建一个公开项目test, 然后推送一个镜像

    ```
    # 先打一个tag
    [root@harbor1 nginx]# docker tag nginx:1.7.9 192.168.10.160/test/nginx:1.7.9
    
    #登录
    [root@harbor1 nginx]# docker login 192.168.10.160
    Username: testpush
    Password: 
    WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
    Configure a credential helper to remove this warning. See
    https://docs.docker.com/engine/reference/commandline/login/#credentials-store
    
    Login Succeeded
    
    #push 成功
    [root@harbor1 nginx]# docker push 192.168.10.160/test/nginx:1.7.9
    
    ```

14. 查看仓库是否有了

    ![](https://makj-blogs-images.oss-cn-beijing.aliyuncs.com/%E6%9F%A5%E7%9C%8B%E4%BB%93%E5%BA%931.png)

15. 测试 161节点pull一下

    ```
    [root@harbor2 harbor]# docker login 192.168.10.160
    Username: testpush
    Password: 
    WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
    Configure a credential helper to remove this warning. See
    https://docs.docker.com/engine/reference/commandline/login/#credentials-store
    
    Login Succeeded
    
    #pull成功
    [root@harbor2 harbor]# docker pull 192.168.10.160/test/nginx:1.7.9
    1.7.9: Pulling from test/nginx
    4f4fb700ef54: Pull complete 
    8c71e11b018e: Pull complete 
    32a497444d35: Pull complete 
    8f6a37a5f8b6: Pull complete 
    b0568fa3217a: Pull complete 
    2188268d060d: Pull complete 
    5af4b0ff64b0: Pull complete 
    Digest: sha256:b1f5935eb2e9e2ae89c0b3e2e148c19068d91ca502e857052f14db230443e4c2
    Status: Downloaded newer image for 192.168.10.160/test/nginx:1.7.9
    192.168.10.160/test/nginx:1.7.9
    ```

16. 双主复制配置 登录http://192.168.10.160/ 

    1. 仓库管理->新建目标 填写需要复制的仓库目标

       ![](https://makj-blogs-images.oss-cn-beijing.aliyuncs.com/%E4%BB%93%E5%BA%93%E7%AE%A1%E7%90%86%E6%96%B0%E5%BB%BA%E7%9B%AE%E6%A0%87.png)

    2. 复制管理->新建规则，关联上面创建的161目标仓库 

       ![](https://makj-blogs-images.oss-cn-beijing.aliyuncs.com/%E5%A4%8D%E5%88%B6%E7%AE%A1%E7%90%86%E6%96%B0%E5%BB%BA%E8%A7%84%E5%88%99.png)

    3. 160查看复制结果

       ![](https://makj-blogs-images.oss-cn-beijing.aliyuncs.com/160%E6%9F%A5%E7%9C%8B%E5%A4%8D%E5%88%B6%E7%BB%93%E6%9E%9C.png)

    4. 同理，161创建复制规则到160 达到双向复制

       ![](https://makj-blogs-images.oss-cn-beijing.aliyuncs.com/161%E5%88%9B%E5%BB%BA%E7%9B%AE%E6%A0%87%E4%BB%93%E5%BA%93.png)

    5. 161复制管理新建规则到160

       ![](https://makj-blogs-images.oss-cn-beijing.aliyuncs.com/161%E5%A4%8D%E5%88%B6%E7%AE%A1%E7%90%86%E6%96%B0%E5%BB%BA%E8%A7%84%E5%88%99.png)

    6. 161查看复制结果

       ![](https://makj-blogs-images.oss-cn-beijing.aliyuncs.com/161%E6%9F%A5%E7%9C%8B%E5%A4%8D%E5%88%B6%E7%BB%93%E6%9E%9C.png)

# 结论

双主复制的时候两边镜像是通过对比镜像签名来完成是否是同一个镜像的，如果是同一个镜像则不尽兴复制