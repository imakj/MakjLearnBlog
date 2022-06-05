# 准备环境

* 下载软件包(社区版)

  https://open.oceanbase.com/softwareCenter/community

* 最低要求

  * 每台observer节点不能少于2c16G
  * observer最少三节点
  * ocpserver不能少于8c16G

* 系统架构

  | 主机名 | IP             | 部署节点           | 配置        |
  | ------ | -------------- | ------------------ | ----------- |
  | obs1   | 192.168.10.160 | OBServer1,OBProxy1 | 4c/16G/500G |
  | obs2   | 192.168.10.161 | OBServer2,OBProxy2 | 4c/16G/500G |
  | obs3   | 192.168.10.162 | OBServer3,OBProxy3 | 4c/16G/500G |
  | obs4   | 192.168.10.163 | OBServer1          | 4c/16G/500G |
  | obs5   | 192.168.10.164 | OBServer2          | 4c/16G/500G |
  | obs6   | 192.168.10.165 | OBServer3          | 4c/16G/500G |
  | obc    | 192.168.10.166 | OBControl,OBClient | 4c/2G/500G  |
  | ocp    | 192.168.10.167 | OCPServer          | 8c/16G/500G |

* 修改主机名，host文件 每个及其分别修改 修改完成后重启电脑

  ```
  # hostnamectl set-hostname obs1
  # hostnamectl set-hostname obs2
  # hostnamectl set-hostname obs3
  # hostnamectl set-hostname obs4
  # hostnamectl set-hostname obs5
  # hostnamectl set-hostname obs6
  # hostnamectl set-hostname obc
  # hostnamectl set-hostname ocp
  
  #修改host文件 所有节点执行
  echo "192.168.10.160 obs1" >> /etc/hosts
  echo "192.168.10.161 obs2" >> /etc/hosts
  echo "192.168.10.162 obs3" >> /etc/hosts
  echo "192.168.10.163 obs4" >> /etc/hosts
  echo "192.168.10.164 obs5" >> /etc/hosts
  echo "192.168.10.165 obs6" >> /etc/hosts
  echo "192.168.10.166 obc" >> /etc/hosts
  echo "192.168.10.167 ocp" >> /etc/hosts
  cat /etc/hosts
  
  ```

* obs1-obs6节点准备文件系统可以使用LVM创建(便于后续扩展) 也可以单独格式化

  ```
  # 创建lvm
  # pvcreate /dev/sdb
  # vgcreate datavg /dev/sdb 
  # lvcreate -n datalv -L 300000M datavg
  # lvcreate -n adminlv -L 100000M datavg
  # lvcreate -n redolv -L 100000M datavg
  
  # 格式化 挂载磁盘
  # mkfs.xfs /dev/datavg/datalv 
  # mkfs.xfs /dev/datavg/adminlv 
  # mkfs.xfs /dev/datavg/redolv 
  # mkdir /data /home/admin /redo -p
  # cat >> /etc/fstab << EOF
  /dev/datavg/datalv /data                       xfs     defaults    0 0
  /dev/datavg/adminlv /home/admin                       xfs     defaults       0 0
  /dev/datavg/redolv /redo                       xfs     defaults      0 0
  EOF
  # cat /etc/fstab
  # 修改完重启
  # reboot
  # 重启完成后查看
  # df -h
  ```

* obs1-obs6节点创建用户 用户组 授权

  ```
  # 删除之前的admin账户
  # rm -rf /var/spool/mail/admin
  # userdel -r admin
  
  # 创建用户和用户组 
  # groupadd -g 66000 admin
  #创建用户 不创建用户的home目录 手动指定
  # useradd -u 66000 -g admin -m -d /home/admin -s /bin/bash admin
  # 此命令有可能提示目录已经存在 无法生产环境文件 直接手动拷贝环境命令即可
  # cp /etc/skel/.bash* /home/admin/
  # 修改密码为admin
  # echo "admin" | passwd --stdin admin
  # 授权
  # chown -R admin:admin /data
  # chown -R admin:admin /redo/
  # chown -R admin:admin /home/admin/
  # 配置sudo权限
  # cat >> /etc/sudoers << EOF
  admin ALL=(ALL) NOPASSWD:ALL
  EOF
  # 测试
  # su - admin
  $ sudo - su root
  # exit
  $ exit
  ```

* 所有节点修改操作系统环境

  ```
  # 修改环境变量
  # echo "export LANG=en_US.UTF8" >> ~/.bash_profile 
  # echo "export LANG=en_US.UTF8" >> /home/admin/.bash_profile 
  # echo "export LANG=en_US.UTF8" >> /etc/profile
  
  #如果有图形界面 则修改默认是命令行
  # systemctl set-default multi-user.target
  ```

* 安装软件包

  ```
  # yum install -y expect mariadb mariadb-devel python-devel openssl-devel gcc gcc-gfortran gcc-c++ python-setuptools bc et-tools mtr chrony bind-utils libaio tree net-tools
  ```

* 修改系统环境，官方建议

  ```
  # cat >> /etc/security/limits.conf  << EOF
  * soft nofile 655360 
  * hard nofile 655360 
  * soft nproc 655360 
  * hard nproc 655360 
  * hard core unlimited 
  * soft core unlimited 
  * hard stack 10240 
  * soft stack 10240 
  * hard cpu unlimited 
  * soft cpu unlimited
  EOF
  
  # cat >> /etc/sysctl.conf  << EOF
  fs.aio-max-nr = 1048576
  fs.file-max = 6815744
  vm.swappiness=1
  vm.max_map_count=655360
  kernel.pid_max=819200   
  #vm.nr_hugepages = 0
  kernel.core_pattern=/data/1/core-%e-%p-%t 
  vm.min_free_kbytes=204800
  net.core.somaxconn=32768 
  net.core.netdev_max_backlog=10000 
  net.core.rmem_default=16777216 
  net.core.wmem_default=16777216 
  net.core.rmem_max=16777216 
  net.core.wmem_max=16777216
  net.ipv4.ip_local_port_range=10000 65535 
  net.ipv4.ip_forward=1 
  net.ipv4.conf.default.rp_filter=1 
  net.ipv4.conf.default.accept_source_route=0 
  net.ipv4.tcp_rmem=4096 87380 16777216 
  net.ipv4.tcp_wmem=4096 65536 16777216 
  net.ipv4.tcp_max_syn_backlog=16384 
  net.ipv4.tcp_fin_timeout=15 
  net.ipv4.tcp_tw_reuse=1 
  net.ipv4.tcp_slow_start_after_idle=0
  EOF
  
  # sysctl -p 
  
  # 时区
  # ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
  # hwclock
  
  # echo "SELINUX=disabled" > /etc/selinux/config
  # echo "#SELINUXTYPE=targeted " >> /etc/selinux/config
  # cat /etc/selinux/config
  # setenforce 0
  
  # cat /sys/kernel/mm/transparent_hugepage/defrag
  # cat /sys/kernel/mm/transparent_hugepage/enabled
  
  # cat >> /etc/rc.d/rc.local  << EOF
  if test -f /sys/kernel/mm/transparent_hugepage/enabled; then
  echo never > /sys/kernel/mm/transparent_hugepage/enabled
  fi
  if test -f /sys/kernel/mm/transparent_hugepage/defrag; then
  echo never > /sys/kernel/mm/transparent_hugepage/defrag
  fi
  EOF
  
  # chmod +x /etc/rc.d/rc.local
  
  #修改磁盘调度
  # vim set_deadline.sh
  for DISK in `ls /sys/block | grep "sd\?"` ; do echo deadline > /sys/block/$DISK/queue/scheduler ; done
  # sh set_deadline.sh 
  
  # echo 'for DISK in `ls /sys/block | grep "sd\?"` ; do echo deadline > /sys/block/$DISK/queue/scheduler ; done' > set_deadline.sh
  sh set_deadline.sh 
  ```

* 安装校时

  ```
  # yum -y install chrony
  
  # 选择ocp当做服务器
  # cp /etc/chrony.conf /etc/chrony.conf.bak
  # cat >> /etc/chrony.conf  << EOF
  server 192.168.10.160
  allow 192.168.10.0/24
  local stratum 10
  EOF
  
  # systemctl restart chronyd
  # systemctl enable chronyd
  
  # timedatectl set-timezone Asia/Shanghai
  # chronyc -a makestep
  200 OK
  # chronyc sources -v
  
  # 其他服务器从obs1同步
  # cp /etc/chrony.conf /etc/chrony.conf.bak
  # cat >> /etc/chrony.conf  << EOF
  server 192.168.10.160
  allow 192.168.10.0/24
  local stratum 10
  EOF
  # systemctl restart chronyd && systemctl enable chronyd
  
  # 在ocp查看和其他系统时差 是否校时成功
  # clockdiff 192.168.10.160
  # clockdiff 192.168.10.161
  # clockdiff 192.168.10.162
  # clockdiff 192.168.10.163
  # clockdiff 192.168.10.164
  # clockdiff 192.168.10.165
  # clockdiff 192.168.10.166
  ```

* 配置ocp的admin用户的和其他机的免密登录

  ```
  # ocp执行
  # su - admin
  $ ssh-keygen
  $ ssh-copy-id -i /home/admin/.ssh/id_rsa.pub obs1
  $ ssh-copy-id -i /home/admin/.ssh/id_rsa.pub obs2
  $ ssh-copy-id -i /home/admin/.ssh/id_rsa.pub obs3
  $ ssh-copy-id -i /home/admin/.ssh/id_rsa.pub obs4
  $ ssh-copy-id -i /home/admin/.ssh/id_rsa.pub obs5
  $ ssh-copy-id -i /home/admin/.ssh/id_rsa.pub obs6
  $ ssh-copy-id -i /home/admin/.ssh/id_rsa.pub obc
  $ ssh-copy-id -i /home/admin/.ssh/id_rsa.pub ocp
  ```

* 上传文件到ocp机器

  ```
  [root@ocp soft]# ll /data/soft/
  total 385712
  -rw-r--r-- 1 admin admin 146799982 Jan 24 23:41 jdk-8u311-linux-x64.tar.gz
  -rw-r--r-- 1 admin admin    658620 Apr 10 14:37 libobclient-2.0.0-2.el7.x86_64.rpm
  -rw-r--r-- 1 admin admin   8111800 Apr 10 14:37 obagent-1.1.0-1.el7.x86_64.rpm
  -rw-r--r-- 1 admin admin  41916564 Apr 10 14:37 obclient-2.0.0-2.el7.x86_64.rpm
  -rw-r--r-- 1 admin admin  33305696 Apr 10 14:36 ob-deploy-1.2.1-9.el7.x86_64.rpm
  -rw-r--r-- 1 root  root    1062976 Apr 10 14:37 oblogproxy-1.0.0-1.el7.x86_64.rpm
  -rw-r--r-- 1 admin admin   8179432 Apr 10 14:36 obproxy-3.2.0-1.el7.x86_64.rpm
  -rw-r--r-- 1 root  root   48708456 Apr 10 14:35 oceanbase-ce-3.1.2-10000392021123010.el7.x86_64.rpm
  -rw-r--r-- 1 root  root   57361468 Apr 10 14:37 oceanbase-ce-devel-3.1.2-10000392021123010.el7.x86_64.rpm
  -rw-r--r-- 1 admin admin    158948 Apr 10 14:35 oceanbase-ce-libs-3.1.2-10000392021123010.el7.x86_64.rpm
  -rw-r--r-- 1 admin admin  48687968 Apr 10 14:37 oceanbase-ce-utils-3.1.2-10000392021123010.el7.x86_64.rpm
  ```

* 将依赖写入环境变量

  ```
  $ echo 'export LD_LIBRARY_PATH=\$LD_LIBRARY_PATH:~/oceanbase-ce/lib' >> ~/.bash_profile
  $ source ~/.bash_profile
  
  ```

  

  

# 自动化部署(deploy)

1. ocp节点安装deploy自动化部署工具

   ```
   # 授权
   $ sudo chown -R admin:admin ./soft/
   # 安装 
   $ sudo rpm -ivh ob-deploy-1.2.1-9.el7.x86_64.rpm
   # 查看安装目录 全部在
   $ rpm -ql `rpm -qa| grep ob-deploy`
   ```

2. 安装完成会在/etc/profile.d/有个obd.sh的启动文件 执行这个启动文件

   ```
   $ cat /etc/profile.d/obd.sh 
   $ sh /etc/profile.d/obd.sh 
   ```

3. 执行命令，就会生成相应的工作目录

   ```
   $ obd mirror list
   Update OceanBase-community-stable-el7 ok
   Update OceanBase-development-kit-el7 ok
   +------------------------------------------------------------------+
   |                      Mirror Repository List                      |
   +----------------------------+--------+---------+------------------+
   | SectionName                | Type   | Enabled | Update Time      |
   +----------------------------+--------+---------+------------------+
   | oceanbase.community.stable | remote | True    | 2022-04-12 21:10 |
   | oceanbase.development-kit  | remote | True    | 2022-04-12 21:10 |
   | local                      | local  | -       | 2022-04-12 21:10 |
   +----------------------------+--------+---------+------------------+
   # 查看工作目录
   # cd /home/admin/.obd
   ```

4. 修改源为本地源，用deploy 安装的时候 就会将这些包推送到所有节点

   ```
   $ cd /home/admin/.obd/mirror/
   $ mv remote remote-back
   # 拷贝系统文件到本地源 注意执行用户 否则会找不到包 这里用admin执行
   $ obd mirror clone /data/soft/*.rpm
   # 查看本地源
   $ obd mirror list local
   +----------------------------------------------------------------------------------------------------------+
   |                                            local Package List                                            |
   +--------------------+---------+-----------------------+--------+------------------------------------------+
   | name               | version | release               | arch   | md5                                      |
   +--------------------+---------+-----------------------+--------+------------------------------------------+
   | libobclient        | 2.0.0   | 2.el7                 | x86_64 | f73cae67e2ff5be0682ac2803aba33a7ed26430e |
   | obagent            | 1.1.0   | 1.el7                 | x86_64 | d2416fadeadba35944872467843d55da0999f298 |
   | obclient           | 2.0.0   | 2.el7                 | x86_64 | 1d2c3ee31f40b9d2fbf97f653f549d896b7e7060 |
   | ob-deploy          | 1.2.1   | 9.el7                 | x86_64 | 219823e1119d37f59ce2c906b44e45605acfd3f3 |
   | obproxy            | 3.2.0   | 1.el7                 | x86_64 | 8d5c6978f988935dc3da1dbec208914668dcf3b2 |
   | oceanbase-ce-libs  | 3.1.2   | 10000392021123010.el7 | x86_64 | 94fff0ab31de053051dba66039e3185fa390cad5 |
   | oceanbase-ce-utils | 3.1.2   | 10000392021123010.el7 | x86_64 | 6ca7db146fee526f4201508f9bd30901e487b7c5 |
   +--------------------+---------+-----------------------+--------+------------------------------------------+
   
   ```

5. 开始部署 环境

   * OBD：obd节点

   * OBServer: obs1,obs2,obs3节点
   * OBProxy: obs1,obs2,obs3节点
   * OBClient: obd节点

6. 创建yaml部署集群文件

   配置文件案例可以再https://gitee.com/ycsitcn/obdeploy.git查看

   ```
   $ cd /data/
   $ vim makjobce-3zones.yaml  #可以用复制模式:set paste
   # Only need to configure when remote login is required
   user:
      username: admin  #部署的用户名 
      password: admin  #密码
      key_file: /home/admin/.ssh/id_rsa.pub
      port: your ssh port, default 22  #端口
   #   timeout: ssh connection timeout (second), default 30
   oceanbase-ce:  #配置集群区域名字
     servers:  #服务器参数
       - name: obs1  #节点名字 共三台
         # Please don use hostname, only IP can be supported
         ip: 192.168.10.160
       - name: obs2
         ip: 192.168.10.161
       - name: obs3
         ip: 192.168.10.162
     global:  # 全局配置
       # Please set devname as the network adaptor name whose ip is  in the setting of severs.
       # if set severs as "127.0.0.1", please set devname as "lo"
       # if current ip is 192.168.1.10, and the ip's network adaptor's name is "eth0", please use "eth0"
       devname: ens192  #指定网卡
       cluster_id: 2  #集群id 可以自定义 整数 默认2
       # please set memory limit to a suitable value which is matching resource.
       memory_limit: 8G # The maximum running memory for an observer  #observer 内存限制，最低8G 这里给8G 默认系统的80%
       system_memory: 3G # The reserved system memory. system_memory is reserved for general tenants. The default value is 30G. #observer 系统内存 算在8G里面的 这部分是所有租户的共享内存
       stack_size: 512K  #堆栈大小 
       cpu_count: 16   #cpu数量 这里给默认 也可以根据真实环境改变 最多16个
       cache_wash_threshold: 1G  #缓存清洗的值 默认就好
       __min_full_resource_pool_memory: 268435456  #最小资源池大小
       workers_per_cpu_quota: 10 #每个cpu工作的线程数
       schema_history_expire_time: 1d #scheam清洗的时间 1天
       # The value of net_thread_count had better be same as cpu core number.
       net_thread_count: 4 #网络线程总数
       major_freeze_duty_time: Disable
       minor_freeze_times: 10
       enable_separate_sys_clog: 0 # 是否启用clog阻塞
       enable_merge_by_turn: FALSE # 是否启用按照顺序合并
       #datafile_disk_percentage: 20 # The percentage of the data_dir space to the total disk space. This value takes effect only when datafile_size is 0. The default value is 90.
       datafile_size: 50G  # 数据初始文件大小 最小就是50G
       syslog_level: WARN # System log level. The default value is INFO. # 系统日志级别
       enable_syslog_wf: false # Print system logs whose levels are higher than WARNING to a separate log file. The default value is true. # 是否启用日志打印
       enable_syslog_recycle: true # Enable auto system log recycling or not. The default value is false. # 是否启用日志回收
       max_syslog_file_count: 10 # The maximum number of reserved log files before enabling auto recycling. The default value is 0.# 日志回收最大文件数量
       # observer cluster name, consistent with obproxy cluster_name
       appname: makjobce   # 集群名字需要和集群标识不一样
       root_password: mkj123. # root user password, can be empty  # 集群管理员密码 就是root@sys的账户密码
       proxyro_password: mkj123. # proxyro user pasword, consistent with obproxy observer_sys_password, can be empty
     obs1:
       mysql_port: 2881 # External port for OceanBase Database. The default value is 2881. # mysql端口
       rpc_port: 2882 # Internal port for OceanBase Database. The default value is 2882. # 集群rpc的端口
       #  The working directory for OceanBase Database. OceanBase Database is started under this directory. This is a required field.# obproxy连接集群使用的密码
       home_path: /home/admin/oceanbase-ce #安装目录
       # The directory for data storage. The default value is $home_path/store.
       data_dir: /data  # 数据目录
       # The directory for clog, ilog, and slog. The default value is the same as the data_dir value.
       redo_dir: /redo # redo目录
       zone: zone1 # 属于第几个zone 
     obs2:
       mysql_port: 2881 # External port for OceanBase Database. The default value is 2881.
       rpc_port: 2882 # Internal port for OceanBase Database. The default value is 2882.
       #  The working directory for OceanBase Database. OceanBase Database is started under this directory. This is a required field.
       home_path: /home/admin/oceanbase-ce
       # The directory for data storage. The default value is $home_path/store.
       data_dir: /data
       # The directory for clog, ilog, and slog. The default value is the same as the data_dir value.
       redo_dir: /redo
       zone: zone2
     obs3:
       mysql_port: 2881 # External port for OceanBase Database. The default value is 2881.
       rpc_port: 2882 # Internal port for OceanBase Database. The default value is 2882.
       #  The working directory for OceanBase Database. OceanBase Database is started under this directory. This is a required field.
       home_path: /home/admin/oceanbase-ce
       # The directory for data storage. The default value is $home_path/store.
       data_dir: /data
       # The directory for clog, ilog, and slog. The default value is the same as the data_dir value.
       redo_dir: /redo
       zone: zone3
   obproxy:  # obproxy配置
     servers:
       - 192.168.10.160
       - 192.168.10.161
       - 192.168.10.162
     # Set dependent components for the component.
     # When the associated configurations are not done, OBD will automatically get the these configurations from the dependent components.
     depends: # obproxy依赖的配置 这个名字就对应上面的 #配置集群区域名字oceanbase-ce
       - oceanbase-ce
     global: # obproxy 全局配置
       listen_port: 2883 # External port. The default value is 2883. # 监听端口
       prometheus_listen_port: 2884 # The Prometheus port. The default value is 2884. # 和prometheus的通信端口
       home_path: /home/admin/obproxy # 安装目录
       # oceanbase root server list
       # format: ip:mysql_port;ip:mysql_port # 对应需要代理分发的数据库服务
       rs_list: 192.168.10.160:2881;192.168.10.161:2881;192.168.10.162:2881
       enable_cluster_checkout: false # 是否做集群检测
       # observer cluster name, consistent with oceanbase-ce appname
       cluster_name: makjobce # 集群名字
       obproxy_sys_password: mkj123. # obproxy sys user password, can be empty  # obproxy的sys用户的密码
       observer_sys_password: mkj123. # proxyro user pasword, consistent with oceanbase-ce proxyro_password, can be empty  # observer的sys用户的密码
   
   ```

7. 部署集群

   ```
   # makjobce是配置文件里的集群名字
   $ obd cluster deploy makjobce -c makjobce-3zones.yaml 
   
   # 查看集群信息 查看配置文件就是在/home/admin/.obd/cluster/makjobce下面 将此配置文件拷贝到任何机器都可以管理这个集群 和k8s的config文件一样
   $ obd cluster list
   +----------------------------------------------------------------+
   |                          Cluster List                          |
   +----------+-----------------------------------+-----------------+
   | Name     | Configuration Path                | Status (Cached) |
   +----------+-----------------------------------+-----------------+
   | makjobce | /home/admin/.obd/cluster/makjobce | deployed        |
   +----------+-----------------------------------+-----------------+
   
   ```

8. 启动并且初始化集群 启动中必须所有状态OK

   ```
   $ obd cluster start makjobce
   Get local repositories and plugins ok
   Open ssh connection ok
   Load cluster param plugin ok
   Check before start observer ok
   Check before start obproxy ok
   Start observer ok
   observer program health check ok
   Connect to observer ok
   Initialize cluster
   Cluster bootstrap ok
   Wait for observer init ok
   +--------------------------------------------------+
   |                     observer                     |
   +----------------+---------+------+-------+--------+
   | ip             | version | port | zone  | status |
   +----------------+---------+------+-------+--------+
   | 192.168.10.160 | 3.1.2   | 2881 | zone1 | active |
   | 192.168.10.161 | 3.1.2   | 2881 | zone2 | active |
   | 192.168.10.162 | 3.1.2   | 2881 | zone3 | active |
   +----------------+---------+------+-------+--------+
   
   Start obproxy ok
   obproxy program health check ok
   Connect to obproxy ok
   Initialize cluster  #检查所有状态是否是 active
   +--------------------------------------------------+
   |                     obproxy                      |
   +----------------+------+-----------------+--------+
   | ip             | port | prometheus_port | status |
   +----------------+------+-----------------+--------+
   | 192.168.10.160 | 2883 | 2884            | active |
   | 192.168.10.161 | 2883 | 2884            | active |
   | 192.168.10.162 | 2883 | 2884            | active |
   +----------------+------+-----------------+--------+
   makjobce running
   
   # 初始化完成后所有的集群数据目录下都会有 /data/sstable的目录
   
   #部署完成稿查看每台节点下是否正常
   $ ps -ef|grep observer
   $ ps -ef|grep admin
   
   # 实际的启动文件就是一个软连接
   $ ls -las /home/admin/oceanbase-ce/bin/observer 
   0 lrwxrwxrwx 1 admin admin 100 Apr 12 22:12 /home/admin/oceanbase-ce/bin/observer -> /home/admin/.obd/repository/oceanbase-ce/3.1.2/7fafba0fac1e90cbd1b5b7ae5fa129b64dc63aed/bin/observer
   
   #查看日志和其他文件目录
   $ /home/admin/oceanbase-ce
   ```

# 手工部署

企业版没有OBD 有OAT(就是企业版的OCP) 在企业版里要么OCP部署要么手工部署

* 准备环境

  参考上面的准备环境

* 相关目录
  * observer 部署/启动目录/home/admin/oceanbase RPM包自动创建
  * observer 数据总目录/home/admin/oceanbase/store/fgeduobce 手动创建
  * observer 数据文件实际目录/data/fgeduobce/sstable 手动创建，通过软链接映射到数据总目录下
  * observer 事务日志实际目录/redo/fgeduobce/{clog,slog,ilog} 手动创建，通过软链接映射到数据总目录下
  * observer 参数文件目录/home/admin/oceanbase/etc 启动时在启动目录自动创建或自动读取
  * observer 运行日志目录/home/admin/oceanbase/log 启动时在启动目录自动创建

* 上传软件包 所有节点上传

  * oceanbase-ce-3.1.2-10000392021123010.el7.x86_64.rpm

  * oceanbase-ce-libs-3.1.2-10000392021123010.el7.x86_64.rpm

    上传到/home/admin下

* 部署observer

  1. 安装软件包 root执行 所有节点执行

     ```shell
     # rpm -ivh /home/admin/*.rpm
     
     # tree /home/admin/oceanbase
     ```

  2. 创建目录 所有节点执行

     相关目录解释
     参数或目录值备注
     observer 部署/启动目录/home/admin/oceanbase RPM包自动创建
     observer 数据总目录/home/admin/oceanbase/store/makjobce 手动创建
     observer 数据文件实际目录/data/makjobce/sstable 手动创建，通过软链接映射到数据总目录下
     observer 事务日志实际目录/redo/makjobce/{clog,slog,ilog} 手动创建，通过软链接映射到数据总目录下
     observer 参数文件目录/home/admin/oceanbase/etc 启动时在启动目录自动创建或自动读取
     observer 运行日志目录/home/admin/oceanbase/log 启动时在启动目录自动创建

     ```
     # su - admin
     
     $ sudo chown -R admin:admin /home/admin
     
     $ mkdir -p ~/oceanbase/store/makjobce /data/makjobce/{sstable,etc3} /redo/makjobce/{clog,ilog,slog,etc2}
     
     $ for f in {clog,ilog,slog,etc2}; do ln -s /redo/makjobce/$f ~/oceanbase/store/makjobce/$f ; done
     
     $ for f in {sstable,etc3}; do ln -s /data/makjobce/$f ~/oceanbase/store/makjobce/$f; done
     ```

  3. 启动  

     -i :网卡信息
     -p 外部端口
     -P 内部rpc端口
     -z 当前节点zone名字
     -d 数据目录
     -r 集群节点 
     -c 集群id
     -n 集群名字
     -d 实际数据存储目录
     -o 各种参数  里面的参数不能有空格 不能有空格 不能有空格
     	memory_limit 内存限制
     	cache_wash_threshold 清洗缓存大小
     	__min_full_resource_pool_memory 资源池最小内存
     	system_memory 系统保留内存
     	memory_chunk_cache_size 内存缓存块大小
     	cpu_count cpu数量
     	net_thread_count 网络线程数
     	datafile_size 数据文件大小
     	stack_size 堆大小
     	config_additional_dir 额外配置目录 这个貌似没什么用

     

     ```
     # 先配置环境变量
     # su - admin
     $ echo 'export LD_LIBRARY_PATH=\$LD_LIBRARY_PATH:~/oceanbase/lib' >> ~/.bash_profile
     $ source ~/.bash_profile
     
     $ cd ~/oceanbase
     
     # 每个obs就是zone不一样
     # obs1启动 
     cd ~/oceanbase && bin/observer -i ens192 -p 2881 -P 2882 -z zone1 -d ~/oceanbase/store/makjobce -r '192.168.10.160:2882:2881;192.168.10.161:2882:2881;192.168.10.162:2882:2881' -c 3 -n makjobce -o "memory_limit=8G,cache_wash_threshold=1G,__min_full_resource_pool_memory=268435456,system_memory=3G,memory_chunk_cache_size=128M,cpu_count=16,net_thread_count=4,datafile_size=50G,stack_size=1536K,config_additional_dir=/data/makjobce/etc3;/redo/makjobce/etc2" -d ~/oceanbase/store/makjobce
     
     
     # obs2启动
     cd ~/oceanbase && bin/observer -i ens192 -p 2881 -P 2882 -z zone2 -d ~/oceanbase/store/makjobce -r '192.168.10.160:2882:2881;192.168.10.161:2882:2881;192.168.10.162:2882:2881' -c 3 -n makjobce -o "memory_limit=8G,cache_wash_threshold=1G,__min_full_resource_pool_memory=268435456,system_memory=3G,memory_chunk_cache_size=128M,cpu_count=16,net_thread_count=4,datafile_size=50G,stack_size=1536K,config_additional_dir=/data/makjobce/etc3;/redo/makjobce/etc2" -d ~/oceanbase/store/makjobce
     
     
     # obs3启动
     cd ~/oceanbase && bin/observer -i ens192 -p 2881 -P 2882 -z zone3 -d ~/oceanbase/store/makjobce -r '192.168.10.160:2882:2881;192.168.10.161:2882:2881;192.168.10.162:2882:2881' -c 3 -n makjobce -o "memory_limit=8G,cache_wash_threshold=1G,__min_full_resource_pool_memory=268435456,system_memory=3G,memory_chunk_cache_size=128M,cpu_count=16,net_thread_count=4,datafile_size=50G,stack_size=1536K,config_additional_dir=/data/makjobce/etc3;/redo/makjobce/etc2" -d ~/oceanbase/store/makjobce
     
     ```

  4. 启动玩后检查节点信息

     ```
     $ IPS="192.168.10.160 192.168.10.161 192.168.10.162"
     $ for ob in $IPS;do echo $ob; ssh $ob "ps -ef | grep observer | grep -v grep "; done
     $ for ob in $IPS;do echo $ob; ssh $ob "netstat -ntlp"; done
     ```

  5. 初始化集群，现在都是单节点的mysql 还没有系统租户什么的 密码是空的

     ```
     $ mysql -h 192.168.10.160 -u root -P 2881 -p -c -A
     
     #初始化zone 一定要等初始化完成 返回状态ok
     alter system bootstrap ZONE 'zone1' SERVER '192.168.10.160:2882',ZONE 'zone2' SERVER '192.168.10.161:2882', ZONE 'zone3' SERVER'192.168.10.162:2882';
     ```

  6. 初始化完成后 就可以使用sys租户进入 改root密码 授权

     ```
     $ mysql -h 192.168.10.160 -u root@sys -P 2881 -p -c -A
     
     # 集群管理员（root@sys）密码
     alter user root identified by 'mkj123.' ;
     
     # obproxy 用户（proxyro）密码
     # 默认obproxy 连接OceanBase 集群时使用用户proxyro 。
     grant select on oceanbase.* to proxyro identified by 'mkj123.' ;
     ```

  7. observer log 自清理设置

     如果/home/admin 目录空间紧张，则设置运行日志滚动输出，根据实际情况修改。

     ```
     alter system set enable_syslog_recycle=true;
     alter system set max_syslog_file_count=10;
     show parameters where name in ('enable_syslog_recycle','max_syslog_file_count');
     ```

* 部署obporxy

  	1. 安装包 传到三台机器

      * obproxy-3.2.0-1.el7.x86_64.rpm

        ```
        # rpm -ivh /home/admin/obproxy-3.2.0-1.el7.x86_64.rpm
        ```

  	2. 相关目录

      参数或目录值备注
      obproxy 部署/启动目录/home/admin/obproxy-版本号RPM 包自动创建
      obproxy 参数文件目录/home/admin/obproxy/etc 启动时在启动目录自动创建或自动读取
      obproxy 运行日志目录/home/admin/obproxy/log 启动时在启动目录自动创建

  	3. 启动obproxy  三台节点一起执行

      enable_strict_kernel_release 是否检测内核版本
      enable_cluster_checkout 集群退出后是否需要检测
      enable_metadb_used 是否启用matedb数据库

      ```
      # su - admin
      
      $ cd /home/admin/obproxy-3.2.0/
      
      $ bin/obproxy -r "192.168.10.160:2881;192.168.10.161:2881;192.168.10.162:2881" -p 2883 -o "enable_strict_kernel_release=false,enable_cluster_checkout=false,enable_metadb_used=false" -c makjobce
      ```

  	4. 检查三台节点状态 是否运行正常

      ```
      $ ps -ef|grep obproxy
      
      # 查看2881 2882 2883 2884端口是否正常
      $ netstat -ntlp
      ```

  	5. 修改proxysys密码 三台机器都要改 否则只改一个 其他的认证不成功 不能进行转发 报'reading authorization packet'

      对应上面obs安装的第六步的这个proxyro 用户的密码`grant select on oceanbase.* to proxyro identified by 'mkj123.' ;`

      ```
      $ mysql -h 192.168.10.160 -u root@proxysys -P 2883 -p
      
      # 查看密码 需要修改observer_sys_password和obproxy_sys_password
      show proxyconfig like '%sys_password%';
      
      #改密码
      alter proxyconfig set obproxy_sys_password = 'mkj123.' ;
      alter proxyconfig set observer_sys_password = 'mkj123.' ;
      
      # 第二台
      $ mysql -h 192.168.10.161 -u root@proxysys -P 2883 -p
      alter proxyconfig set obproxy_sys_password = 'mkj123.' ;
      alter proxyconfig set observer_sys_password = 'mkj123.' ;
      
      # 第三台
      $ mysql -h 192.168.10.162 -u root@proxysys -P 2883 -p
      alter proxyconfig set obproxy_sys_password = 'mkj123.' ;
      alter proxyconfig set observer_sys_password = 'mkj123.' ;
      
      ```

  	6. 集群的启动和停止

      直接kill进程 `kill `pidof observer

       等待60s，等进程完全退出 反复确认进程完全退出 然后按照上面的启动方式重启

      一个重启完成后叜重启第二个

  	7. 查看集群状态

      ```
      # 集群可用资源
      select a.zone,
      concat(a.svr_ip, ':', a.svr_port) observer,
      cpu_total,
      (cpu_total - cpu_assigned) cpu_free,
      round(mem_total / 1024 / 1024 / 1024) mem_total_gb,
      round((mem_total - mem_assigned) / 1024 / 1024 / 1024)
      mem_free_gb,
      usec_to_time(b.last_offline_time) last_offline_time,
      usec_to_time(b.start_service_time) start_service_time,
      b.status,
      usec_to_time(b.stop_time) stop_time,
      b.build_version
      from __all_virtual_server_stat a
      join __all_server b
      on (a.svr_ip = b.svr_ip and a.svr_port = b.svr_port)
      order by a.zone, a.svr_ip;
      
      
      select zone,svr_ip,svr_port,inner_port,with_rootserver,status,gmt_create from __all_server order by zone, svr_ip;
      
      # 查看OB 集群所有租户信息
      select tenant_id, tenant_name, zone_list, locality ,gmt_modified from __all_tenant;
      
      # 查看OceanBase 集群所有节点可用资源情况
      select zone, svr_ip, svr_port,inner_port, cpu_total, cpu_assigned,
      round(mem_total/1024/1024/1024) mem_total_gb,
      round(mem_assigned/1024/1024/1024) mem_ass_gb,
      round(disk_total/1024/1024/1024) disk_total_gb,
      unit_num, substr(build_version,1,6) version
      from __all_virtual_server_stat
      order by zone, svr_ip, inner_port;
      
      select a.zone,concat(a.svr_ip,':',a.svr_port) observer, cpu_total,
      (cpu_total-cpu_assigned) cpu_free,
      round(mem_total/1024/1024/1024) mem_total_gb,
      round((mem_total-mem_assigned)/1024/1024/1024) mem_free_gb,
      round(disk_total/1024/1024/1024) disk_total_gb,
      substr(a.build_version,1,6)
      version,usec_to_time(b.start_service_time) start_service_time
      from __all_virtual_server_stat a join __all_server b on
      (a.svr_ip=b.svr_ip and a.svr_port=b.svr_port)
      order by a.zone, a.svr_ip;
      
      # 查看集群资源池具体使用情况
      select t1.name resource_pool_name, t2.`name` unit_config_name,
      t2.max_cpu, t2.min_cpu,
      round(t2.max_memory/1024/1024/1024) max_mem_gb,
      round(t2.min_memory/1024/1024/1024) min_mem_gb,
      t3.unit_id, t3.zone, concat(t3.svr_ip,':',t3.`svr_port`)
      observer,t4.tenant_id, t4.tenant_name
      from __all_resource_pool t1 join __all_unit_config t2 on
      (t1.unit_config_id=t2.unit_config_id)
      join __all_unit t3 on (t1.`resource_pool_id` =
      t3.`resource_pool_id`)
      left join __all_tenant t4 on (t1.tenant_id=t4.tenant_id)
      order by t1.`resource_pool_id`, t2.`unit_config_id`, t3.unit_id;
      
      #查看OB 集群资源单元unit 配置情况
      select
      unit_config_id,name,max_cpu,min_cpu,round(max_memory/1024/1024/
      1024) max_mem_gb,
      round(min_memory/1024/1024/1024) min_mem_gb,
      round(max_disk_size/1024/1024/1024) max_disk_size_gb
      from __all_unit_config
      order by unit_config_id;
      ```

      

      

* 集群验证

  1. 创建资源测试下

     ```
     $ mysql -h192.168.10.160- -uroot@sys#makjobce -P2883 -pmkj123. -c -A
     
     #创建单元
     CREATE resource unit unit_makj1 max_cpu=4, min_cpu=4,max_memory='1G', min_memory='1G', max_iops=10000, min_iops=1000,max_session_num=1000000,max_disk_size='20G';
     
     #创建资源池
     CREATE resource pool pool_makj1 unit = 'unit_makj1', unit_num = 1,zone_list=('zone1','zone2','zone3');
     
     #创建租户
     create tenant tent_fgedu resource_pool_list=('pool_makj1'),charset='utf8' set ob_tcp_invited_nodes='%';
     --如果创建是oracle 的，则加上参数
     ob_compatibility_mode='oracle';，社区版这里只有mysql,默认也是mysql
     
     #查看租户
     SELECT * FROM oceanbase.gv$tenant;
     
     
     # 连接租户
     $  mysql -h192.168.10.160 -uroot@tent_makj1#makjobce -P2883 -p -c -A
     
     #修改密码
     alter user root identified by 'mkj123.';
     
     #创建数据库 建表 插入数据
     create database makjdb;
     show databases;
     use makjdb;
     set global ob_tcp_invited_nodes = '%';
     create user makj identified by 'makj';
     grant all on *.* to makjdb;
     GRANT ALL ON makjdb.* to 'makj'@'%';
     
     # 使用业务用户登录
     mysql -h192.168.10.160 -umakj@tent_makj#makjobce -P2883 -p -c -A
     
     # 在创建一个分区表
     CREATE TABLE tab_hash1(id INT, username VARCHAR(20), Age int,
     PRIMARY KEY (id))
     PARTITION by hash(id) partitions 6;
     insert into tab_hash1 values('1','fgedu2019','10');
     insert into tab_hash1 values('2','fgedu2020','20');
     insert into tab_hash1 values('3','fgedu2021','30');
     insert into tab_hash1 values('4','fgedu2021','40');
     insert into tab_hash1 values('5','fgedu2021','50');
     ```

     

* 手动扩容

  1-1-1扩容到2-2-2

  扩容的三个节点为 obs4 obs5 obs6 proxy不扩容

  1. 扩容前的集群检查

     ```
     # 查看zone使用情况
     select zone,svr_ip,svr_port,inner_port,with_rootserver,status,gmt_create from __all_server order by zone, svr_ip;
     
     select t1.name resource_pool_name, t2.`name` unit_config_name,
     t2.max_cpu, t2.min_cpu,
     round(t2.max_memory/1024/1024/1024) max_mem_gb,
     round(t2.min_memory/1024/1024/1024) min_mem_gb,
     t3.unit_id, t3.zone, concat(t3.svr_ip,':',t3.`svr_port`)
     observer,t4.tenant_id, t4.tenant_name
     from __all_resource_pool t1 join __all_unit_config t2 on
     (t1.unit_config_id=t2.unit_config_id)
     join __all_unit t3 on (t1.`resource_pool_id` =
     t3.`resource_pool_id`)
     left join __all_tenant t4 on (t1.tenant_id=t4.tenant_id)
     order by t1.`resource_pool_id`, t2.`unit_config_id`, t3.unit_id;
     select a.zone,concat(a.svr_ip,':',a.svr_port) observer, cpu_total,
     (cpu_total-cpu_assigned) cpu_free,
     round(mem_total/1024/1024/1024) mem_total_gb,
     round((mem_total-mem_assigned)/1024/1024/1024) mem_free_gb,
     round(disk_total/1024/1024/1024) disk_total_gb,unit_num,
     substr(a.build_version,1,6)
     version,usec_to_time(b.start_service_time)
     start_service_time,b.status
     from __all_virtual_server_stat a join __all_server b on
     (a.svr_ip=b.svr_ip and a.svr_port=b.svr_port)
     order by a.zone, a.svr_ip;
     
     
     #查看租户使用情况
     select tenant_id, tenant_name, compatibility_mode,zone_list,
     locality ,primary_zone ,gmt_modified from __all_tenant;
     
     # 查看资源使用情况
     select
     unit_config_id,name,max_cpu,min_cpu,round(max_memory/1024/1024/
     1024) max_mem_gb,
     round(min_memory/1024/1024/1024) min_mem_gb,
     round(max_disk_size/1024/1024/1024) max_disk_size_gb
     from __all_unit_config order by unit_config_id;
     
     #查看租户表的分布 可以看到三副本 一个机器一个副本 需要root登录到租户执行 分区表有6个分区 每个分区有三个副本 所以会产生18条记录 分布在不同机器上
     use oceanbase;
     SELECT t1.tenant_id,
     t1.tenant_name,
     t2.database_name,
     t3.table_id,
     t3.table_Name,
     t3.tablegroup_id,
     t3.part_num,
     t4.partition_Id,
     t4.zone,
     t4.svr_ip,
     t4.role,
     round(t4.data_size / 1024 / 1024) data_size_mb
     from gv$tenant t1
     join gv$database t2
     on (t1.tenant_id = t2.tenant_id)
     join gv$table t3
     on (t2.tenant_id = t3.tenant_id and t2.database_id =
     t3.database_id and
     t3.index_type = 0)
     left join gv$partition t4
     on (t2.tenant_id = t4.tenant_id and
     (t3.table_id = t4.table_id or t3.tablegroup_id =
     t4.table_id) and
     t4.role in (1, 2))
     where t1.tenant_id = 1002
     order by t1.tenant_id,
     t3.tablegroup_id,
     t3.table_name,
     t4.partition_Id,
     t4.role;
     ```

  2. 开始扩容

  3. 上传到/home/admin下 然后安装 obs4 obs5 obs6 执行

     ```
     # rpm -ivh /home/admin/*.rpm
     
     # tree /home/admin/oceanbase
     ```

  4. 创建目录 

     所有节点执行  obs4 obs5 obs6 执行

     ```
     # su - admin
     
     $ sudo chown -R admin:admin /home/admin
     
     $ mkdir -p ~/oceanbase/store/makjobce /data/makjobce/{sstable,etc3} /redo/makjobce/{clog,ilog,slog,etc2}
     
     $ for f in {clog,ilog,slog,etc2}; do ln -s /redo/makjobce/$f ~/oceanbase/store/makjobce/$f ; done
     
     $ for f in {sstable,etc3}; do ln -s /data/makjobce/$f ~/oceanbase/store/makjobce/$f; done
     ```

  5. 启动 

     ```
     # 先配置环境变量
     # su - admin
     $ echo 'export LD_LIBRARY_PATH=\$LD_LIBRARY_PATH:~/oceanbase/lib' >> ~/.bash_profile
     $ source ~/.bash_profile
     
     $ cd ~/oceanbase
     
     # obs4扩容 注意 这里是扩zone的资源 不是扩展zone 所以obs4对应的是zone1
     cd ~/oceanbase && bin/observer -i ens192 -p 2881 -P 2882 -z zone1 -d ~/oceanbase/store/makjobce -r '192.168.10.160:2882:2881;192.168.10.161:2882:2881;192.168.10.162:2882:2881;192.168.10.163:2882:2881;192.168.10.164:2882:2881;192.168.10.165:2882:2881;' -c 3 -n makjobce -o "memory_limit=8G,cache_wash_threshold=1G,__min_full_resource_pool_memory=268435456,system_memory=3G,memory_chunk_cache_size=128M,cpu_count=16,net_thread_count=4,datafile_size=50G,stack_size=1536K,config_additional_dir=/data/makjobce/etc3;/redo/makjobce/etc2" -d ~/oceanbase/store/makjobce
     
     # obs5扩容 注意 所以obs5对应的是zone2
     cd ~/oceanbase && bin/observer -i ens192 -p 2881 -P 2882 -z zone2 -d ~/oceanbase/store/makjobce -r '192.168.10.160:2882:2881;192.168.10.161:2882:2881;192.168.10.162:2882:2881;192.168.10.163:2882:2881;192.168.10.164:2882:2881;192.168.10.165:2882:2881;' -c 3 -n makjobce -o "memory_limit=8G,cache_wash_threshold=1G,__min_full_resource_pool_memory=268435456,system_memory=3G,memory_chunk_cache_size=128M,cpu_count=16,net_thread_count=4,datafile_size=50G,stack_size=1536K,config_additional_dir=/data/makjobce/etc3;/redo/makjobce/etc2" -d ~/oceanbase/store/makjobce
     
     # obs5扩容 注意 所以obs6对应的是zone3
     cd ~/oceanbase && bin/observer -i ens192 -p 2881 -P 2882 -z zone3 -d ~/oceanbase/store/makjobce -r '192.168.10.160:2882:2881;192.168.10.161:2882:2881;192.168.10.162:2882:2881;192.168.10.163:2882:2881;192.168.10.164:2882:2881;192.168.10.165:2882:2881;' -c 3 -n makjobce -o "memory_limit=8G,cache_wash_threshold=1G,__min_full_resource_pool_memory=268435456,system_memory=3G,memory_chunk_cache_size=128M,cpu_count=16,net_thread_count=4,datafile_size=50G,stack_size=1536K,config_additional_dir=/data/makjobce/etc3;/redo/makjobce/etc2" -d ~/oceanbase/store/makjobce
     
     ```

  6. 启动玩后检查节点信息

     ```
     $ IPS="192.168.10.163 192.168.10.164 192.168.10.165"
     $ for ob in $IPS;do echo $ob; ssh $ob "ps -ef | grep observer | grep -v grep "; done
     
     #查看端口 2881和2882是否启动正常
     $ for ob in $IPS;do echo $ob; ssh $ob "netstat -ntlp"; done
     ```

  7. 将新的observer添加到新的zone 是扩副本的资源 不是扩副本所以要将对应的机器加到zone1 zone2 zone4

     ```
     添加新节点：
     mysql -h192.168.10.161 -P2883 -uroot@sys#makjobce -pmkj123. -c -A
     
     use oceanbase;
     # 163添加到zone1
     alter system add server '192.168.10.163:2882' zone 'zone1';
     
     # 164添加到zone2
     alter system add server '192.168.10.164:2882' zone 'zone2';
     
     # 165添加到zone3
     alter system add server '192.168.10.165:2882' zone 'zone3';
     ```

  8. 添加完成后查看server情况 

     ```
     # 节点添加完成后，查看OB 集群所有节点信息 是否是active
     select zone,svr_ip,svr_port,inner_port,with_rootserver,status,gmt_create from __all_server order by zone, svr_ip;
     
     # 可以查看分区副本迁移重新选主的过程，可以通过视图 __all_rootservice_event_history 查看集群event 事件信息。 需要集群自动触发  从添加时间后查询
     SELECT *
     FROM __all_rootservice_event_history
     WHERE 1 = 1
     and gmt_create > date_format('2022-04-15 15:36:17','%Y%m%d%H%i%s')
     ORDER BY gmt_create LIMIT 50;
     
     # 等待资源平衡分区副本迁移完成后，查看资源使用情况 可以看到刚加进来的资源 cpu空闲14c 内存空闲5G 还没用
     select a.zone,concat(a.svr_ip,':',a.svr_port) observer, cpu_total,
     (cpu_total-cpu_assigned) cpu_free,
     round(mem_total/1024/1024/1024) mem_total_gb,
     round((mem_total-mem_assigned)/1024/1024/1024) mem_free_gb,
     round(disk_total/1024/1024/1024) disk_total_gb,unit_num,
     substr(a.build_version,1,6)
     version,usec_to_time(b.start_service_time)
     start_service_time,b.status
     from __all_virtual_server_stat a join __all_server b on
     (a.svr_ip=b.svr_ip and a.svr_port=b.svr_port)
     order by a.zone, a.svr_ip;
     
     # 查看单元
     select t1.name resource_pool_name, t2.`name` unit_config_name,
     t2.max_cpu, t2.min_cpu,
     round(t2.max_memory/1024/1024/1024) max_mem_gb,
     round(t2.min_memory/1024/1024/1024) min_mem_gb,
     t3.unit_id, t3.zone, concat(t3.svr_ip,':',t3.`svr_port`)
     observer,t4.tenant_id, t4.tenant_name
     from __all_resource_pool t1 join __all_unit_config t2 on
     (t1.unit_config_id=t2.unit_config_id)
     join __all_unit t3 on (t1.`resource_pool_id` =
     t3.`resource_pool_id`)
     left join __all_tenant t4 on (t1.tenant_id=t4.tenant_id)
     order by t1.`resource_pool_id`, t2.`unit_config_id`, t3.unit_id;
     
     select
     unit_config_id,name,max_cpu,min_cpu,round(max_memory/1024/1024/
     1024) max_mem_gb,
     round(min_memory/1024/1024/1024) min_mem_gb,
     round(max_disk_size/1024/1024/1024) max_disk_size_gb
     from __all_unit_config
     order by unit_config_id;
     
     #查看租户情况
     select t1.tenant_id,t1.tenant_name,t2.database_name,t3.table_id,t3.table_Name,t3.tablegroup_id,t3.part_num,t4.partition_Id,
     t4.zone,t4.svr_ip,t4.role, round(t4.data_size/1024/1024) data_size_mb
     from `gv$tenant` t1
     join `gv$database` t2 on (t1.tenant_id = t2.tenant_id)
     join gv$table t3 on (t2.tenant_id = t3.tenant_id and
     t2.database_id = t3.database_id and t3.index_type = 0)
     left join `gv$partition` t4 on (t2.tenant_id = t4.tenant_id
     and ( t3.table_id = t4.table_id or t3.tablegroup_id = t4.table_id )
     and t4.role in (1,2))
     where t1.tenant_id >1000
     order by t1.tenant_id,t3.tablegroup_id,
     t3.table_name,t4.partition_Id ,t4.role;
     ```

  9. 查看原来的租户情况 并没有扩容

     ```
     # 需要登录到对应租户的实例 root用户登录
     use oceanbase;
     SELECT t1.tenant_id,
     t1.tenant_name,
     t2.database_name,
     t3.table_id,
     t3.table_Name,
     t3.tablegroup_id,
     t3.part_num,
     t4.partition_Id,
     t4.zone,
     t4.svr_ip,
     t4.role,
     round(t4.data_size / 1024 / 1024) data_size_mb
     from gv$tenant t1
     join gv$database t2
     on (t1.tenant_id = t2.tenant_id)
     join gv$table t3
     on (t2.tenant_id = t3.tenant_id and t2.database_id =
     t3.database_id and
     t3.index_type = 0)
     left join gv$partition t4
     on (t2.tenant_id = t4.tenant_id and
     (t3.table_id = t4.table_id or t3.tablegroup_id =
     t4.table_id) and
     t4.role in (1, 2))
     where t1.tenant_id = 1002
     order by t1.tenant_id,
     t3.tablegroup_id,
     t3.table_name,
     t4.partition_Id,
     t4.role;
     ```

  10. 扩容租户 

      扩展租户两个方法 

      * 1是提升单元资源的规格
      * 2是增加资源单元的数量

  ​	这里采用增加单元数量 由原来的1个增加到2个，由于集群是2-2-2的架构 每个zone下有两台observer，因此 每个zone下都会有2个租户的unit资源，集群会自动迁移副本和leader重新选主

  ```
  # 将tent_makj1所属的池的资源单元扩展为2个
  alter resource pool pool_makj1 unit_num = 2;
  
  # 可以看到 新加的obs4 5 6的资源已经开始使用了 少了1个cpu和1G内存
  select a.zone,concat(a.svr_ip,':',a.svr_port) observer, cpu_total,
  (cpu_total-cpu_assigned) cpu_free,
  round(mem_total/1024/1024/1024) mem_total_gb,
  round((mem_total-mem_assigned)/1024/1024/1024) mem_free_gb,
  round(disk_total/1024/1024/1024) disk_total_gb,unit_num,
  substr(a.build_version,1,6)
  version,usec_to_time(b.start_service_time)
  start_service_time,b.status
  from __all_virtual_server_stat a join __all_server b on
  (a.svr_ip=b.svr_ip and a.svr_port=b.svr_port)
  order by a.zone, a.svr_ip;
  
  #这里pool_makj1 变化成了6 台主机，CPU、内存都增加了，等待后台自动平衡
  select t1.name resource_pool_name, t2.`name`
  unit_config_name, t2.max_cpu, t2.min_cpu,
  round(t2.max_memory/1024/1024/1024) max_mem_gb,
  round(t2.min_memory/1024/1024/1024) min_mem_gb,
  t3.unit_id, t3.zone, concat(t3.svr_ip,':',t3.`svr_port`)
  observer,t4.tenant_id, t4.tenant_name
  from __all_resource_pool t1 join __all_unit_config t2 on
  (t1.unit_config_id=t2.unit_config_id)
  join __all_unit t3 on (t1.`resource_pool_id` =
  t3.`resource_pool_id`)
  left join __all_tenant t4 on (t1.tenant_id=t4.tenant_id)
  order by t1.`resource_pool_id`, t2.`unit_config_id`, t3.unit_id;
  
  #查询是否平衡完成 可以看到先重新选主switch_leader 然后增加单元池 alter_resource_pool 做数据平衡迁移start_migrate_replica 一直平衡迁移 一直到租户平衡完成tenant_balance_finished
  SELECT *
  FROM __all_rootservice_event_history
  WHERE 1 = 1
  and gmt_create> date_format('2022-04-15 15:36:17','%Y%m%d%H%i%s')
  ORDER BY gmt_create DESC
  LIMIT 50;
  
  # 最后查看租户的表的使用情况 有6个分区的那张表已经自动迁移到新的obs节点上了 扩容后对普通机器已经没什么作用了 但是对分区表作用很大 所以分区表在集群扩容后 查询速度就更快了
  use oceanbase;
  SELECT t1.tenant_id,
  t1.tenant_name,
  t2.database_name,
  t3.table_id,
  t3.table_Name,
  t3.tablegroup_id,
  t3.part_num,
  t4.partition_Id,
  t4.zone,
  t4.svr_ip,
  t4.role,
  round(t4.data_size / 1024 / 1024) data_size_mb
  from gv$tenant t1
  join gv$database t2
  on (t1.tenant_id = t2.tenant_id)
  join gv$table t3
  on (t2.tenant_id = t3.tenant_id and t2.database_id =
  t3.database_id and
  t3.index_type = 0)
  left join gv$partition t4
  on (t2.tenant_id = t4.tenant_id and
  (t3.table_id = t4.table_id or t3.tablegroup_id =
  t4.table_id) and
  t4.role in (1, 2))
  where t1.tenant_id = 1002
  order by t1.tenant_id,
  t3.tablegroup_id,
  t3.table_name,
  t4.partition_Id,
  t4.role;
  
  ```

  11. 修改之前三台节点的observer和obproxy 的列表加入扩容的三台节点 一台一台来

      ```
      # obs1修改
      $ kill -9 1728
      $ cd ~/oceanbase && bin/observer -i ens192 -p 2881 -P 2882 -z zone1 -d ~/oceanbase/store/makjobce -r '192.168.10.160:2882:2881;192.168.10.161:2882:2881;192.168.10.162:2882:2881;192.168.10.163:2882:2881;192.168.10.164:2882:2881;192.168.10.165:2882:2881;' -c 3 -n makjobce -o "memory_limit=8G,cache_wash_threshold=1G,__min_full_resource_pool_memory=268435456,system_memory=3G,memory_chunk_cache_size=128M,cpu_count=16,net_thread_count=4,datafile_size=50G,stack_size=1536K,config_additional_dir=/data/makjobce/etc3;/redo/makjobce/etc2" -d ~/oceanbase/store/makjobce
      
      # 修改obproxy
      $ kill -9 2389
      cd /home/admin/obproxy-3.2.0/ && bin/obproxy -r "192.168.10.160:2881;192.168.10.161:2881;192.168.10.162:2881;192.168.10.163:2881;192.168.10.164:2881;192.168.10.165:2881" -p 2883 -o "enable_strict_kernel_release=false,enable_cluster_checkout=false,enable_metadb_used=false" -c makjobce
      
      
      # obs2修改
      cd ~/oceanbase && bin/observer -i ens192 -p 2881 -P 2882 -z zone2 -d ~/oceanbase/store/makjobce -r '192.168.10.160:2882:2881;192.168.10.161:2882:2881;192.168.10.162:2882:2881;192.168.10.163:2882:2881;192.168.10.164:2882:2881;192.168.10.165:2882:2881;' -c 3 -n makjobce -o "memory_limit=8G,cache_wash_threshold=1G,__min_full_resource_pool_memory=268435456,system_memory=3G,memory_chunk_cache_size=128M,cpu_count=16,net_thread_count=4,datafile_size=50G,stack_size=1536K,config_additional_dir=/data/makjobce/etc3;/redo/makjobce/etc2" -d ~/oceanbase/store/makjobce
      
      # 修改obproxy
      cd /home/admin/obproxy-3.2.0/ && bin/obproxy -r "192.168.10.160:2881;192.168.10.161:2881;192.168.10.162:2881;192.168.10.163:2881;192.168.10.164:2881;192.168.10.165:2881" -p 2883 -o "enable_strict_kernel_release=false,enable_cluster_checkout=false,enable_metadb_used=false" -c makjobce
      
      # obs3修改
      cd ~/oceanbase && bin/observer -i ens192 -p 2881 -P 2882 -z zone3 -d ~/oceanbase/store/makjobce -r '192.168.10.160:2882:2881;192.168.10.161:2882:2881;192.168.10.162:2882:2881;192.168.10.163:2882:2881;192.168.10.164:2882:2881;192.168.10.165:2882:2881;' -c 3 -n makjobce -o "memory_limit=8G,cache_wash_threshold=1G,__min_full_resource_pool_memory=268435456,system_memory=3G,memory_chunk_cache_size=128M,cpu_count=16,net_thread_count=4,datafile_size=50G,stack_size=1536K,config_additional_dir=/data/makjobce/etc3;/redo/makjobce/etc2" -d ~/oceanbase/store/makjobce
      
      # 修改obproxy
      cd /home/admin/obproxy-3.2.0/ && bin/obproxy -r "192.168.10.160:2881;192.168.10.161:2881;192.168.10.162:2881;192.168.10.163:2881;192.168.10.164:2881;192.168.10.165:2881" -p 2883 -o "enable_strict_kernel_release=false,enable_cluster_checkout=false,enable_metadb_used=false" -c makjobce
      
      #验证
      $ mysql -h192.168.10.160 -uroot@sys#makjobce -P2883 -pmkj123. -c -A
      $ mysql -h192.168.10.161 -uroot@sys#makjobce -P2883 -pmkj123. -c -A
      $ mysql -h192.168.10.162 -uroot@sys#makjobce -P2883 -pmkj123. -c -A
      ```

* 手工收缩 收缩回三台节点

  由于上面3副本没变 只是扩展了副本的资源和unit数量 不需要调整副本

  1. 查看整个集群信息，以及业务租户下表分区副本分布情况，便于后续减副本对比

     ```
     select zone,svr_ip,svr_port,inner_port,with_rootserver,status,gmt_create from __all_server order by zone, svr_ip;
     
     select
     unit_config_id,name,max_cpu,min_cpu,round(max_memory/1024/1024/
     1024) max_mem_gb,
     round(min_memory/1024/1024/1024) min_mem_gb,
     round(max_disk_size/1024/1024/1024) max_disk_size_gb
     from __all_unit_config
     order by unit_config_id;
     
     select tenant_id, tenant_name, compatibility_mode,zone_list,
     locality ,primary_zone ,gmt_modified from __all_tenant;
     select a.zone,concat(a.svr_ip,':',a.svr_port) observer, cpu_total,
     (cpu_total-cpu_assigned) cpu_free,
     round(mem_total/1024/1024/1024) mem_total_gb,
     round((mem_total-mem_assigned)/1024/1024/1024) mem_free_gb,
     round(disk_total/1024/1024/1024) disk_total_gb,unit_num,
     substr(a.build_version,1,6)
     version,usec_to_time(b.start_service_time)
     start_service_time,b.status
     from __all_virtual_server_stat a join __all_server b on
     (a.svr_ip=b.svr_ip and a.svr_port=b.svr_port)
     order by a.zone, a.svr_ip;
     
     select t1.name resource_pool_name, t2.`name` unit_config_name,
     t2.max_cpu, t2.min_cpu,
     round(t2.max_memory/1024/1024/1024) max_mem_gb,
     round(t2.min_memory/1024/1024/1024) min_mem_gb,
     t3.unit_id, t3.zone, concat(t3.svr_ip,':',t3.`svr_port`)
     observer,t4.tenant_id, t4.tenant_name
     from __all_resource_pool t1 join __all_unit_config t2 on
     (t1.unit_config_id=t2.unit_config_id)
     join __all_unit t3 on (t1.`resource_pool_id` =
     t3.`resource_pool_id`) left join __all_tenant t4 on (t1.tenant_id=t4.tenant_id)
     order by t1.`resource_pool_id`, t2.`unit_config_id`, t3.unit_id;
     
     
     SELECT t1.tenant_id,
     t1.tenant_name,
     t2.database_name,
     t3.table_id,
     t3.table_Name,
     t3.tablegroup_id,
     t3.part_num,
     t4.partition_Id,
     t4.zone,
     t4.svr_ip,
     t4.role,
     round(t4.data_size / 1024 / 1024) data_size_mb
     from gv$tenant t1
     join gv$database t2
     on (t1.tenant_id = t2.tenant_id)
     join gv$table t3
     on (t2.tenant_id = t3.tenant_id and t2.database_id =
     t3.database_id and
     t3.index_type = 0)
     left join gv$partition t4
     on (t2.tenant_id = t4.tenant_id and
     (t3.table_id = t4.table_id or t3.tablegroup_id =
     t4.table_id) and
     t4.role in (1, 2))
     where t1.tenant_id > 1000
     order by t1.tenant_id,
     t3.tablegroup_id,
     t3.table_name,
     t4.partition_Id,
     t4.role;
     
     ```

  2. 减少unit数量

     ```
     alter resource pool pool_makj1 unit_num = 1;
     
     # 修改完成后 等待资源迁移完成 直到出现最新的数据的module为balancer
     SELECT *
     FROM __all_rootservice_event_history
     WHERE 1 = 1
     and gmt_create> date_format('2022-04-15 16:06:00','%Y%m%d%H%i%s')
     ORDER BY gmt_create DESC
     LIMIT 50;
     ```

  3. 等待所有的自护副本和资源池释放后，删除需要下架的observer和zone

     ```
     # 删除observer和zone
     alter system delete server '192.168.10.163:2882' zone zone1;
     alter system delete server '192.168.10.164:2882' zone zone2;
     alter system delete server '192.168.10.165:2882' zone zone3;
     
     #查看zone是否删除完成 等到三台机器删除完成后，资源就会自动平衡到其他节点上
     select zone,svr_ip,svr_port,inner_port,with_rootserver,status,gmt_create from __all_server order by zone, svr_ip;
     
     # 查看zone
     select a.zone,concat(a.svr_ip,':',a.svr_port) observer, cpu_total,
     (cpu_total-cpu_assigned) cpu_free,
     round(mem_total/1024/1024/1024) mem_total_gb,
     round((mem_total-mem_assigned)/1024/1024/1024) mem_free_gb,
     round(disk_total/1024/1024/1024) disk_total_gb,unit_num,
     substr(a.build_version,1,6)
     version,usec_to_time(b.start_service_time)
     start_service_time,b.status
     from __all_virtual_server_stat a join __all_server b on
     (a.svr_ip=b.svr_ip and a.svr_port=b.svr_port)
     order by a.zone, a.svr_ip;
     
     #查看租户表所对应的资源 是否已经迁移到在线的节点上
     SELECT
     t1.tenant_id,t1.tenant_name,t2.database_name,t3.table_id,t3.table_Name,t3.tablegroup_id,t3.part_num,t4.partition_Id,
     t4.zone,t4.svr_ip,t4.role, round(t4.data_size/1024/1024)
     data_size_mb
     from `gv$tenant` t1
     join `gv$database` t2 on (t1.tenant_id = t2.tenant_id)
     join gv$table t3 on (t2.tenant_id = t3.tenant_id and
     t2.database_id = t3.database_id and t3.index_type = 0)
     left join `gv$partition` t4 on (t2.tenant_id = t4.tenant_id
     and ( t3.table_id = t4.table_id or t3.tablegroup_id = t4.table_id )
     and t4.role in (1,2))
     where t1.tenant_id >1000
     order by t1.tenant_id,t3.tablegroup_id,
     t3.table_name,t4.partition_Id ,t4.role;
     
     select zone,svr_ip,svr_port,inner_port,with_rootserver,status,gmt_create from __all_server order by zone, svr_ip;
     SELECT t1.tenant_id,
     t1.tenant_name,
     t2.database_name,
     t3.table_id,
     t3.table_Name,
     t3.tablegroup_id,
     t3.part_num,
     t4.partition_Id,
     t4.zone,
     t4.svr_ip,
     t4.role,
     round(t4.data_size / 1024 / 1024) data_size_mb
     from gv$tenant t1
     join gv$database t2
     on (t1.tenant_id = t2.tenant_id)
     join gv$table t3
     on (t2.tenant_id = t3.tenant_id and t2.database_id =
     t3.database_id and
     t3.index_type = 0)
     left join gv$partition t4
     on (t2.tenant_id = t4.tenant_id and
     (t3.table_id = t4.table_id or t3.tablegroup_id =
     t4.table_id) and
     t4.role in (1, 2))
     where t1.tenant_id > 1000
     order by t1.tenant_id,
     t3.tablegroup_id,
     t3.table_name,
     t4.partition_Id,
     t4.role;
     ```

  4. 修改obproxy和在线的observer

     ```
     ```

     

# 升级

直接替换掉observer和obproxy的二进制文件就可以

# 客户端

1. 安装客户端软件包

   ```
   $ cd /data/soft/
   $ sudo rpm -ivh obclient-2.0.0-2.el7.x86_64.rpm libobclient-2.0.0-2.el7.x86_64.rpm 
   ```

2. 其实mysql也可以直接登录

   1. 登录单节点

      ```
      # mysql -h 192.168.10.160 -uroot -P2881 -pmkj123. -c -A oceanbase
      # obclient -h 192.168.10.160 -uroot -P2881 -pmkj123. -c -A oceanbase
      ```

   2. 通过proxy登录到集群，这样proxy就会自动路由到任何一个节点，通过proxy负载均衡登录端口后面必须为proxy的2993 -u后面的格式为-u用户名@租户#集群名字 或者-u集群名字:租户:用户名

      ```
      # mysql -h192.168.10.160 -P2883 -uroot@sys#makjobce -pmkj123. -c -A oceanbase
      # obclient -h192.168.10.160 -P2883 -uroot@sys#makjobce -pmkj123. -c -A oceanbase
      
      # obclient -h192.168.10.160 -P2883 -umakjobce:sys:root -pmkj123. -c -A oceanbase
      # mysql -h192.168.10.160 -P2883 -umakjobce:sys:root -pmkj123. -c -A oceanbase
      ```

   3. 参数解释

      * -h:提供。ceanBase数据库连接IP,通常是一个OBProxy地址。
      * -u:提供租户的连接账户，格式有两种：用户名@租户名#集群名或者集群名:租户名:用户名。mysql租户的管理员用户名默认是root。
      * -P：提供OceanBase数据库连接端口，也是OBProxy的监听端口，默认是2883,可以自定义。

      - -p：提供账户密码，为了安全可以不提供，改为在后面提示符下输入，密码文本不可见。
      - -c：表示在mysql运行环境中不要忽略注释。
      - -A:表 示在mysql连接数据库时不自 劭获取统计信息。

   4. 使用第三方工具登录的时候 用户名必须是root@sys#makjobce 端口是2883 可以使用mysql的工具登录，登录的是proxy的地址

3. 查看集群信息

   ```
   select a.zone,
          concat(a.svr_ip, ':', a.svr_port) observer,
          cpu_total,
          cpu_assigned,
          (cpu_total - cpu_assigned) cpu_free,
          mem_total / 1024 / 1024 / 1024 mem_total_gb,
          mem_assigned / 1024 / 1024 / 1024 mem_assign_gb,
          (mem_total - mem_assigned) / 1024 / 1024 / 1024 mem_free_gb
     from __all_virtual_server_stat a
     join __all_server b
       on (a.svr_ip = b.svr_ip and a.svr_port = b.svr_port)
    order by a.zone, a.svr_ip;
   ```

4. 客户端工具

   使用第三方工具登录的时候 用户名必须是root@sys#makjobce 端口是2883 可以使用mysql的工具登录，登录的是proxy的地址

