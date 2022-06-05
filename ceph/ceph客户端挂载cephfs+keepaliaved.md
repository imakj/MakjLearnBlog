## 环境准备

1. 修改hostname

   client1执行：`hostnamectl set-hostname client1`

   client2执行：`hostnamectl set-hostname client2`

2. 修改hosts

   集群节点添加两台客户端host三个节点机器分别修改host文件，修改完成后如下

   ```127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
   ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
   
   192.168.10.15 node1
   192.168.10.16 node2
   192.168.10.17 node3
   192.168.10.18 client1
   192.168.10.19 client2
   ```

3. 关掉selinux

   三个节点机器分别修改selinux文件关闭selinux

   ```
   # vim /etc/selinux/config
   # This file controls the state of SELinux on the system.
   # SELINUX= can take one of these three values:
   #     enforcing - SELinux security policy is enforced.
   #     permissive - SELinux prints warnings instead of enforcing.
   #     disabled - No SELinux policy is loaded.
   SELINUX=disabled
   # SELINUXTYPE= can take one of three values:
   #     targeted - Targeted processes are protected,
   #     minimum - Modification of targeted policy. Only selected processes are protected.
   #     mls - Multi Level Security protection.
   SELINUXTYPE=targeted
   
   # setenforce 0
   # getenforce
   Disabled
   ```

4. 关闭防火墙

   三个节点同时关闭防火墙并且删除

   ```
   # systemctl stop firewalld.service
   # systemctl disable firewalld.service
   Removed symlink /etc/systemd/system/multi-user.target.wants/firewalld.service.
   Removed symlink /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.
   # rpm -qa|grep firewall
   firewalld-filesystem-0.6.3-8.el7.noarch
   firewalld-0.6.3-8.el7.noarch
   python-firewall-0.6.3-8.el7.noarch
   # rpm -e --nodeps firewalld-0.6.3-8.el7.noarch firewalld-filesystem-0.6.3-8.el7.noarch
   
   ```

5. 两台节点全部配置阿里云yum源,包括Base、epel、ceph

   阿里云

   ```
   # curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
     % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                    Dload  Upload   Total   Spent    Left  Speed
   100  2523  100  2523    0     0  12040      0 --:--:-- --:--:-- --:--:-- 12014
   
   # curl -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
     % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                    Dload  Upload   Total   Spent    Left  Speed
   100   664  100   664    0     0   9454      0 --:--:-- --:--:-- --:--:--  9485
   
   # ll
   total 8
   -rw-r--r--. 1 root root 2523 Jul 11 16:49 CentOS-Base.repo
   -rw-r--r--. 1 root root  664 Jul 11 16:49 epel.repo
   ```

   ceph yum源 采用nautilus版本

   ```
   vim /etc/yum.repos.d/ceph.repo
   [norch]
   name=norch
   baseurl=https://mirrors.aliyun.com/ceph/rpm-nautilus/el7/noarch/
   enabled=1
   gpgcheck=0
   type=rpm-md
   
   [x86_64]
   name=x86 64
   baseurl=https://mirrors.aliyun.com/ceph/rpm-nautilus/el7/x86_64/
   enabled=1
   gpgcheck=0
   
   [ceph]
   name=ceph package for $basearch
   baseurl=https://mirrors.aliyun.com/ceph/rpm-nautilus/el7/$basearch
   enabled=1
   gpgcheck=0
   
   ```

   完毕后更新下系统：`sudo yum update -y `

6. 在client1和client2节点上，配置ntp任务

   ```
   # sudo yum install -y ntp
   # timedatectl set-timezone Asia/Shanghai
   #  crontab -e
   */1 * * * * /usr/sbin/ntpdate 192.168.10.15
   
   #定时任务开机启动
   # systemctl start crond
   # systemctl enable crond
   ```

7. 两台机器上安装ceph-fuse

   ```
   # yum -y install ceph-common ceph-fuse
   ```

8. 从ceph-deploy节点拷贝公钥到两个节点

   ```
   # ssh-copy-id -i /root/.ssh/id_rsa.pub client1
   # ssh-copy-id -i /root/.ssh/id_rsa.pub client2
   ```

9. ceph-deploy 推送秘钥和配置文件到client1和client2

    或者直接scp将ceph.conf和 ceph.client.admin.keyring拷贝到client1的/etc/ceph下

   ```
   # ceph-deploy admin client1 client2
   [ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
   [ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy admin client1
   [ceph_deploy.cli][INFO  ] ceph-deploy options:
   [ceph_deploy.cli][INFO  ]  username                      : None
   [ceph_deploy.cli][INFO  ]  verbose                       : False
   [ceph_deploy.cli][INFO  ]  overwrite_conf                : False
   [ceph_deploy.cli][INFO  ]  quiet                         : False
   [ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7f0ab575f560>
   [ceph_deploy.cli][INFO  ]  cluster                       : ceph
   [ceph_deploy.cli][INFO  ]  client                        : ['client1']
   [ceph_deploy.cli][INFO  ]  func                          : <function admin at 0x7f0ab5ff9230>
   [ceph_deploy.cli][INFO  ]  ceph_conf                     : None
   [ceph_deploy.cli][INFO  ]  default_release               : False
   [ceph_deploy.admin][DEBUG ] Pushing admin keys and conf to client1
   [client1][DEBUG ] connected to host: client1 
   [client1][DEBUG ] detect platform information from remote host
   [client1][DEBUG ] detect machine type
   [client1][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
   ```

10. 挂载cephfs

    ```
    # mkdir /mnt/cephfs_ssd/
    # mount -t ceph node1:6789,node2:6789,node3:6789:/ /mnt/cephfs_ssd/ -o name=admin 
    # df -h
    Filesystem               Size  Used Avail Use% Mounted on
    devtmpfs                 1.9G     0  1.9G   0% /dev
    tmpfs                    1.9G     0  1.9G   0% /dev/shm
    tmpfs                    1.9G  8.8M  1.9G   1% /run
    tmpfs                    1.9G     0  1.9G   0% /sys/fs/cgroup
    /dev/mapper/centos-root   78G  2.1G   76G   3% /
    /dev/sda2                2.0G  185M  1.9G  10% /boot
    /dev/sda1                500M   12M  489M   3% /boot/efi
    tmpfs                    379M     0  379M   0% /run/user/0
    192.168.10.15:6789:/     1.7T     0  1.7T   0% /mnt/cephfs_ssd
    ```

    为了便于区分多个cephfs，挂载两个节点，挂载的集群有所不一样
    
    ```
    # mount -t ceph node1:6789,node2:6789:/ /mnt/cephfs_ssd/ -o mds_namespace=cephfs-ssd,name=admin
    # mount -t ceph node2:6789,node3:6789:/ /mnt/cephfs_hdd/ -o mds_namespace=cephfs-hdd,name=admin
    You have new mail in /var/spool/mail/root
    [root@client1 mnt]# df -h
    Filesystem                               Size  Used Avail Use% Mounted on
    devtmpfs                                 1.9G     0  1.9G   0% /dev
    tmpfs                                    1.9G     0  1.9G   0% /dev/shm
    tmpfs                                    1.9G   41M  1.9G   3% /run
    tmpfs                                    1.9G     0  1.9G   0% /sys/fs/cgroup
    /dev/mapper/centos-root                   78G  2.1G   76G   3% /
    /dev/sda2                                2.0G  185M  1.9G  10% /boot
    /dev/sda1                                500M   12M  489M   3% /boot/efi
    tmpfs                                    379M     0  379M   0% /run/user/0
    192.168.10.15:6789,192.168.10.16:6789:/  1.6T  629G  917G  41% /mnt/cephfs_ssd
    192.168.10.16:6789,192.168.10.17:6789:/  8.1T     0  8.1T   0% /mnt/cephfs_hdd
    ```
    
11. 挂载nfs

12. 添加nfs源

     ```
     # vim /etc/yum.repos.d/nfs-ganesha.repo
     [nfs-ganesha_x86_64]
     name=x86 64
     baseurl=https://mirrors.aliyun.com/ceph/nfs-ganesha/rpm-V3.3-stable/octopus/el7/x86_64/
     enabled=1
     gpgcheck=0
     ```

     ```
     # yum update -y
     # yum -y install libntirpc nfs-ganesha nfs-ganesha-ceph
     ```

13. 启动rpc

    ```
    # systemctl start rpcbind; systemctl enable rpcbind
    # systemctl status rpcbind
    ● rpcbind.service - RPC bind service
       Loaded: loaded (/usr/lib/systemd/system/rpcbind.service; enabled; vendor preset: enabled)
       Active: active (running) since Tue 2020-09-29 22:54:20 CST; 10s ago
     Main PID: 25527 (rpcbind)
       CGroup: /system.slice/rpcbind.service
               └─25527 /sbin/rpcbind -w
    
    Sep 29 22:54:20 client1 systemd[1]: Starting RPC bind service...
    Sep 29 22:54:20 client1 systemd[1]: Started RPC bind service.
    
    ```

14. 备份原配置文件 根目录创建ceph文件夹

    ```
    # mkdir /cephfs/
    # mv /etc/ganesha/ganesha.conf /etc/ganesha/ganesha.conf.bak
    # vim /etc/ganesha/ganesha.conf
    NFS_Core_Param
    {
           Bind_Addr=0.0.0.0;
    }
    
    EXPORT_DEFAULTS {
            Attr_Expiration_Time = 0;
    }
    
    CACHEINODE {
            Dir_Max = 1;
            Dir_Chunk = 0;
    
            Cache_FDs = false;
    
            NParts = 1;
            Cache_Size = 1;
    }
    
    
    EXPORT
    {
            Export_id=1;
            Path = "/";   #cephfs的根目录 会映射本地的一个/cephfs 可以导出多目录
            Pseudo = /cephnfs-ssd;    #虚拟目录名
            Access_Type = RW;
            Protocols = 3,4;
            Transports = TCP;
            SecType = sys,krb5,krb5i,krb5p;
            Squash = no_root_squash;
            Attr_Expiration_Time = 0;
            FSAL {
                    Name = CEPH;
                    User_Id = "admin";
                    Filesystem = "cephfs-ssd";    #cephfs文件名
            }
    
    }
    
    EXPORT
    {
            Export_id=2;
            Path = "/";
            Pseudo = /cephnfs-hdd;
            Access_Type = RW;
            Protocols = 3,4;
            Transports = TCP;
            # SecType = sys,krb5,krb5i,krb5p;
            Squash = no_root_squash;
            Attr_Expiration_Time = 0;
            FSAL {
                    Name = CEPH;
                    User_Id = "admin";
                    Filesystem = "cephfs-hdd";  #cephfs文件名
            }
    
    }
    LOG {
            Facility {
                    name = FILE;
                    destination = "/var/log/ganesha/ganesha.log";
                    enable = active;
            }
    
    
    }
    ```

15. 重启nfs-ganesha

    ```
    # systemctl restart nfs-ganesha
    # showmount -e
    # systemctl enable nfs-ganesha
    ```

    

16. keepaliaved安装

    ```
    # yum install keepalived -y
    # rpm -qc keepalived
    /etc/keepalived/keepalived.conf
    /etc/sysconfig/keepalived
    ```

17. master配置信息

    ```
    # mv /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak
    # vim /etc/keepalived/keepalived.conf
    ! Configuration File for keepalived
    global_defs {
        router_id cephnfs #标识信息
    }
    # 可以添加检测脚本防止脑裂
    vrrp_script check {
    }
    vrrp_instance VI_1 {
        state MASTER    #角色是master
        interface ens192  #vip 绑定端口
        virtual_router_id 50    #让master 和backup在同一个虚拟路由里，id 号必须相同；
        priority 101            #优先级,谁的优先级高谁就是master ;
        advert_int 1            #心跳间隔时间
        authentication {
            auth_type PASS      #认证
            auth_pass 1111      #密码
    }
        virtual_ipaddress {
            192.168.10.20/24 brd 192.168.10.255 dev ens192 label eth192:vip            #虚拟ip
        }
    }
    ```

18. BACKUP配置信息

    ```
    # mv /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak
    # vim /etc/keepalived/keepalived.conf
    ! Configuration File for keepalived
    global_defs {
        router_id cephnfs
    }
    # 可以添加检测脚本防止脑裂
    vrrp_script check {
    }
    vrrp_instance VI_1 {
        state BACKUP
        interface ens192
        virtual_router_id 50
        priority 100
        #将两个都设置为BACKUP BACKUP添加nopreempt 标记为非抢占模式
        nopreempt    
        advert_int 1
        authentication {
            auth_type PASS
            auth_pass 1111
    }
        virtual_ipaddress {
            192.168.10.20/24 brd 192.168.10.255 dev ens192 label eth192:vip
        }
    }
    
    ```

19. 两台机器重启keepalived

    ```
    # systemctl restart keepalived 
    # systemctl enable keepalived 
    # ip addr
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host 
           valid_lft forever preferred_lft forever
    2: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
        link/ether 00:0c:29:3a:5d:e2 brd ff:ff:ff:ff:ff:ff
        inet 192.168.10.18/24 brd 192.168.10.255 scope global noprefixroute ens192
           valid_lft forever preferred_lft forever
        inet 192.168.10.20/24 brd 192.168.10.255 scope global secondary ens192:vip
           valid_lft forever preferred_lft forever
        inet6 fe80::e66e:58a8:2d9a:c8aa/64 scope link noprefixroute 
           valid_lft forever preferred_lft forever
    ```

    

