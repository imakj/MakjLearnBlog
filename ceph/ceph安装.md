# 环境

* 系统环境：centos7.6 2003

* 节点：

  * ceph_node1: 
    * 安装组件：ceph-deploy,ceph-admin
    * 组件：mon1,mgr1,ods
    * ntp服务器
    * IP :192.168.10.15 
    * 硬盘：两块4T(后续会添加)
  * ceph_node2: 
    * 组件：mon2,mgr2,ods
    * IP :192.168.10.16
    * 硬盘：两块4T(后续会添加)
  * ceph_node3: 
  * 组件：mon2,mgr2,ods
    * IP :192.168.10.17 
    * 硬盘：两块6T 两块1T
  * 40G网卡，MTU改为MTU=9000
  
  

# 部署

## 环境准备

1. 修改hostname

   node1执行：`# hostnamectl set-hostname node1 `

   node2执行：`# hostnamectl set-hostname node2 `

   node3执行：`# hostnamectl set-hostname node3`

2. 修改hosts

   三个节点机器分别修改host文件，修改完成后如下

   ```127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
   
   ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
   
   192.168.10.15 node1
   192.168.10.16 node2
   192.168.10.17 node3
   ```
   
3. 密码认证

   node1节点生成公钥：` # ssh-keygen` 一路默认确认即可

   ```
   # ssh-keygen
   Generating public/private rsa key pair.
   Enter file in which to save the key (/root/.ssh/id_rsa): 
   /root/.ssh/id_rsa already exists.
   Overwrite (y/n)? y
   Enter passphrase (empty for no passphrase): 
   Enter same passphrase again: 
   Your identification has been saved in /root/.ssh/id_rsa.
   Your public key has been saved in /root/.ssh/id_rsa.pub.
   The key fingerprint is:
   SHA256:qB7Gr/kL8ZCRM3ZHfVQSbAT0D8qXBdoVr7cD5eMYuFA root@node1
   The key's randomart image is:
   +---[RSA 2048]----+
   |        .oo=*o+. |
   |     . .  .+++ . |
   |    * . . .E+ ...|
   |   . * o ....=o. |
   |    + . S.o.oooo.|
   |   . =    ... =.o|
   |    * .    . . + |
   |   o =          .|
   |    +o+.         |
   +----[SHA256]-----+
   ```

   

   拷贝公钥到三个节点，包括自己

   ```
   # ssh-copy-id -i /root/.ssh/id_rsa.pub node2
   # ssh-copy-id -i /root/.ssh/id_rsa.pub node3
   # ssh-copy-id -i /root/.ssh/id_rsa.pub node1
   ```

   如果出现下面错误

   ```
   # ssh-copy-id -i /root/.ssh/id_rsa.pub node2
   /usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
   /usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
   
   /usr/bin/ssh-copy-id: ERROR: @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
   ERROR: @    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
   ERROR: @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
   ERROR: IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
   ERROR: Someone could be eavesdropping on you right now (man-in-the-middle attack)!
   ERROR: It is also possible that a host key has just been changed.
   ERROR: The fingerprint for the ECDSA key sent by the remote host is
   ERROR: SHA256:r4tJ0ZjBL7lIZoy35+OgyCVT5O4PPnoXm6skektunaU.
   ERROR: Please contact your system administrator.
   ERROR: Add correct host key in /root/.ssh/known_hosts to get rid of this message.
   ERROR: Offending ECDSA key in /root/.ssh/known_hosts:1
   ERROR: ECDSA host key for node2 has changed and you have requested strict checking.
   ERROR: Host key verification failed.
   
   #删除know_hosts文件
   # rm -f ~/.ssh/known_hosts 
   ```

   

4. 关掉selinux

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

5. 关闭防火墙

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

6. 配置ntp服务器

   https://blog.csdn.net/zzy5066/article/details/79036674

   三台节点同时安装ntp服务:` sudo yum install -y ntp`

   在node1启动ntp服务并且配置开机启动，注意修改时区为北京时间东八区()

   ```\
   # 
   # systemctl start ntpd
   # systemctl enable ntpd
   Created symlink from /etc/systemd/system/multi-user.target.wants/ntpd.service to /usr/lib/systemd/system/ntpd.service.
   ```

   在node1上配置ntp任务

   ```
   # crontab -e
   */1 * * * * root /usr/sbin/ntpdate -q 0.asia.pool.ntp.org 1.asia.pool.ntp.org 2.asia.pool.ntp.org 3.asia.pool.ntp.org
   
   #定时任务开机启动
   # systemctl start crond
   # systemctl enable crond
   ```

   

   在node2和node3节点上，配置ntp任务

   ```
   #  crontab -e
   */1 * * * * /usr/sbin/ntpdate 192.168.10.9
   
   #定时任务开机启动
   # systemctl start crond
   # systemctl enable crond
   ```

   

   在node2和node3节点上，修改ntp服务器的指向为node1

   ```
   # vim /etc/ntp.conf   #修改server
   ……
   # Use public servers from the pool.ntp.org project.
   # Please consider joining the pool (http://www.pool.ntp.org/join.html).
   #server 0.centos.pool.ntp.org iburst
   #server 1.centos.pool.
   ntp.org iburst
   #server 2.centos.pool.ntp.org iburst
   #server 3.centos.pool.ntp.org iburst
   server 192.168.10.15 iburst
   ……
   
   #修改完成后启动服务
   # systemctl start ntpd
   # systemctl enable ntpd
   #查看服务状态
   # ntpq -pn
        remote           refid      st t when poll reach   delay   offset  jitter
   ==============================================================================
   *192.168.10.15   193.182.111.143  3 u    6   64    1    0.237   -3.759   0.027
   ```

7. 三台节点全部配置阿里云yum源,包括Base、epel、ceph

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
   
   [ceph-source]
   name=ceph package for $basearch
   baseurl=https://mirrors.aliyun.com/ceph/rpm-nautilus/el7/SRPMS/
   enabled=1
   gpgcheck=0
   
   ```
   
   配置nfs-ganesha
   
   ```
   vim /etc/yum.repos.d/nfs-ganesha.repo
   [nfs-ganesha_x86_64]
   name=x86 64
   baseurl=https://mirrors.aliyun.com/ceph/nfs-ganesha/rpm-V2.8-stable/nautilus/x86_64/
   enabled=1
   gpgcheck=0
   ```
   
   配置sicsi
   
   ```
   [ceph-iscsi]
   name=ceph-iscsi noarch packages
   baseurl=http://download.ceph.com/ceph-iscsi/3/rpm/el7/noarch
   enabled=1
   gpgcheck=1
   gpgkey=https://download.ceph.com/keys/release.asc
   type=rpm-md
   
   [ceph-iscsi-source]
   name=ceph-iscsi source packages
   baseurl=http://download.ceph.com/ceph-iscsi/3/rpm/el7/SRPMS
   enabled=1
   gpgcheck=1
   gpgkey=https://download.ceph.com/keys/release.asc
   type=rpm-md
   
   [tcmu-runner]
   name=tcmu-runner
   baseurl=https://3.chacra.ceph.com/r/tcmu-runner/master/eef511565078fb4e2ed52caaff16e6c7e75ed6c3/centos/7/flavors/default/x86_64/
   enabled=1
   priority=1
   gpgcheck=0
   
   [ceph-iscsi-conf]
   name=ceph-iscsi-config
   baseurl=https://3.chacra.ceph.com/r/ceph-iscsi-config/master/7496f1bc418137230d8d45b19c47eab3165c756a/centos/7/flavors/default/noarch/
   enabled=1
   priority=1
   gpgcheck=0 
   ```
   
   

   完毕后更新下系统：`sudo yum -y update`

8. 升级内核

   ```
   #rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
   #yum install https://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm -y
   #yum --enablerepo=elrepo-kernel install  kernel-ml-devel kernel-ml -y
   ```

9. 查看安装的内核

   ```
   # rpm -qa | grep kernel
   kernel-3.10.0-1160.el7.x86_64
   kernel-tools-libs-3.10.0-1160.11.1.el7.x86_64
   kernel-headers-3.10.0-1160.11.1.el7.x86_64
   kernel-ml-5.10.10-1.el7.elrepo.x86_64
   kernel-tools-3.10.0-1160.11.1.el7.x86_64
   kernel-3.10.0-1160.11.1.el7.x86_64
   kernel-devel-3.10.0-1160.11.1.el7.x86_64
   kernel-devel-3.10.0-1160.el7.x86_64
   kernel-ml-devel-5.10.10-1.el7.elrepo.x86_64
   ```

10. 设置kernel默认启动项。后重启

    ```
    # grub2-set-default "kernel-ml-5.10.10-1"
    # 查看新设置的默认的启动项，并重启系统。
    # grub2-editenv list
    ```

11. 重启后验证是否设置成功

    ```
    # uname -r
    5.10.11-1.el7.elrepo.x86_64
    ```

12. 关闭NUMA和透明大页。简单来说，NUMA思路就是将内存和CPU分割为多个区域，每个区域叫做NODE,然后将NODE高速互联。 node内cpu与内存访问速度快于访问其他node的内存，NUMA可能会在某些情况下影响ceph-osd。解决的方案，一种是通过BIOS关闭NUMA，另外一种就是通过cgroup将ceph-osd进程与某一个CPU Core以及同一NODE下的内存进行绑定。但是第二种看起来更麻烦，所以一般部署的时候可以在系统层面关闭NUMA。CentOS系统下，通过修改/etc/grub.conf文件，添加numa=off来关闭NUMA。

13. 编辑 /etc/default/grub 文件，如下图所示加上：numa=off

    ```
    vim /etc/default/grub
    GRUB_TIMEOUT=5
    GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
    GRUB_DEFAULT=saved
    GRUB_DISABLE_SUBMENU=true
    GRUB_TERMINAL_OUTPUT="console"
    GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet numa=off transparent_hugepage=never"
    GRUB_DISABLE_RECOVERY="true"
    ```

14. 重新生成 /etc/grub2.cfg 配置文件

    ```
    # grub2-mkconfig -o /etc/grub2-efi.cfg
    Generating grub configuration file ...
    Found linux image: /boot/vmlinuz-5.10.11-1.el7.elrepo.x86_64
    Found initrd image: /boot/initramfs-5.10.11-1.el7.elrepo.x86_64.img
    Found linux image: /boot/vmlinuz-3.10.0-1160.11.1.el7.x86_64
    Found initrd image: /boot/initramfs-3.10.0-1160.11.1.el7.x86_64.img
    Found linux image: /boot/vmlinuz-3.10.0-1160.el7.x86_64
    Found initrd image: /boot/initramfs-3.10.0-1160.el7.x86_64.img
    Found linux image: /boot/vmlinuz-0-rescue-b1135b09685e4ec3beefda208269c426
    Found initrd image: /boot/initramfs-0-rescue-b1135b09685e4ec3beefda208269c426.img
    done
    ```

15. 重启之后进行确认

    ```
    # dmesg | grep -i numa
    [    0.000000] Command line: BOOT_IMAGE=/vmlinuz-5.10.11-1.el7.elrepo.x86_64 root=/dev/mapper/centos-root ro crashkernel=auto rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet numa=off
    [    0.009711] NUMA turned off
    [    0.035473] Kernel command line: BOOT_IMAGE=/vmlinuz-5.10.11-1.el7.elrepo.x86_64 root=/dev/mapper/centos-root ro crashkernel=auto rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet numa=off
    # cat /proc/cmdline
    BOOT_IMAGE=/vmlinuz-5.10.11-1.el7.elrepo.x86_64 root=/dev/mapper/centos-root ro crashkernel=auto rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet numa=off
    
    # yum install numactl
    # numastat
                               node0
    numa_hit                  390271
    numa_miss                      0
    numa_foreign                   0
    interleave_hit             15680
    local_node                390271
    other_node                     0
    # numactl --show
    policy: default
    preferred node: current
    physcpubind: 0 1 2 3 
    cpubind: 0 
    nodebind: 0 
    membind: 0 
    # numactl --hardware
    available: 1 nodes (0)
    node 0 cpus: 0 1 2 3 
    node 0 size: 3788 MB   //一个cpu用所有内存
    node 0 free: 3217 MB
    node distances:
    node   0 
      0:  10 
    
    ```

16. linux系统参数优化

    ```
    # echo "4194303" > /proc/sys/kernel/pid_max
    # echo "8192" > /sys/block/sda/queue/read_ahead_kb
    # 设置系统最大打开文文件数
    # echo "3285342" > /proc/sys/fs/file-max
    # echo "vm.swappiness = 0" >> /etc/sysctl.conf
    # sysctl -p
    # vim /etc/security/limits.conf
    * soft nofile 65535
    * hard nofile 65535
    
    或者
    * soft nofile 655350 
    * hard nofile 655350
    * soft nproc  655350
    * hard nproc  655350
    * soft core   unlimited
    * hard core   unlimited
    
    #或者临时修改
    ulimit -HSn 655350
    
    #查看
    # ulimit -n
    ```
    
    


​    

## 开始部署

1. node1节点上安装ceph-deploy

   ```
   # yum install -y python-setuptools ceph-deploy
   # ceph-deploy -V
   usage: ceph-deploy [-h] [-v | -q] [--version] [--username USERNAME]
                      [--overwrite-conf] [--ceph-conf CEPH_CONF]
                      COMMAND ...
   ceph-deploy: error: too few arguments
   [root@node1 ~]# ceph-deploy -c
   usage: ceph-deploy [-h] [-v | -q] [--version] [--username USERNAME]
                      [--overwrite-conf] [--ceph-conf CEPH_CONF]
                      COMMAND ...
   ceph-deploy: error: too few arguments
   [root@node1 ~]# ceph-deploy -v
   usage: ceph-deploy [-h] [-v | -q] [--version] [--username USERNAME]
                      [--overwrite-conf] [--ceph-conf CEPH_CONF]
                      COMMAND ...
   ceph-deploy: error: too few arguments
   [root@node1 ~]# ceph-deploy
   usage: ceph-deploy [-h] [-v | -q] [--version] [--username USERNAME]
                      [--overwrite-conf] [--ceph-conf CEPH_CONF]
                      COMMAND ...
   
   Easy Ceph deployment
   
       -^-
      /   \
      |O o|  ceph-deploy v2.0.1
      ).-.(
     '/|||\`
     | '|` |
       '|`
   
   Full documentation can be found at: http://ceph.com/ceph-deploy/docs
   
   optional arguments:
     -h, --help            show this help message and exit
     -v, --verbose         be more verbose
     -q, --quiet           be less verbose
     --version             the current installed version of ceph-deploy
     --username USERNAME   the username to connect to the remote host
     --overwrite-conf      overwrite an existing conf file on remote host (if
                           present)
     --ceph-conf CEPH_CONF
                           use (or reuse) a given ceph.conf file
   
   commands:
     COMMAND               description
       new                 Start deploying a new cluster, and write a
                           CLUSTER.conf and keyring for it.
       install             Install Ceph packages on remote hosts.
       rgw                 Ceph RGW daemon management
       mgr                 Ceph MGR daemon management
       mds                 Ceph MDS daemon management
       mon                 Ceph MON Daemon management
       gatherkeys          Gather authentication keys for provisioning new nodes.
       disk                Manage disks on a remote host.
       osd                 Prepare a data disk on remote host.
       repo                Repo definition management
       admin               Push configuration and client.admin key to a remote
                           host.
       config              Copy ceph.conf to/from remote host(s)
       uninstall           Remove Ceph packages from remote hosts.
       purgedata           Purge (delete, destroy, discard, shred) any Ceph data
                           from /var/lib/ceph
       purge               Remove Ceph packages from remote hosts and purge all
                           data.
       forgetkeys          Remove authentication keys from the local directory.
       pkg                 Manage packages on remote hosts.
       calamari            Install and configure Calamari nodes. Assumes that a
                           repository with Calamari packages is already
                           configured. Refer to the docs for examples
                           (http://ceph.com/ceph-deploy/docs/conf.html)
   
   See 'ceph-deploy <command> --help' for help on a specific command
   
   ```


2. node1上初始化mon节点

   ```
   #创建一个目录
   # mkdir ceph-deploy
   # cd ceph-deploy/
   # 创建一个集群，指定访问网络和osd网络
   # ceph-deploy new --cluster-network=168.192.10.0/24 --public-network=192.168.10.0/24 node1
   [ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
   [ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy new --cluster-network=192.168.10.0/24 --public-network=192.168.10.0/24 node1
   [ceph_deploy.cli][INFO  ] ceph-deploy options:
   [ceph_deploy.cli][INFO  ]  username                      : None
   [ceph_deploy.cli][INFO  ]  func                          : <function new at 0x7fd9487022a8>
   [ceph_deploy.cli][INFO  ]  verbose                       : False
   [ceph_deploy.cli][INFO  ]  overwrite_conf                : False
   [ceph_deploy.cli][INFO  ]  quiet                         : False
   [ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7fd947e6ebd8>
   [ceph_deploy.cli][INFO  ]  cluster                       : ceph
   [ceph_deploy.cli][INFO  ]  ssh_copykey                   : True
   [ceph_deploy.cli][INFO  ]  mon                           : ['node1']
   [ceph_deploy.cli][INFO  ]  public_network                : 192.168.10.0/24
   [ceph_deploy.cli][INFO  ]  ceph_conf                     : None
   [ceph_deploy.cli][INFO  ]  cluster_network               : 192.168.10.0/24
   [ceph_deploy.cli][INFO  ]  default_release               : False
   [ceph_deploy.cli][INFO  ]  fsid                          : None
   [ceph_deploy.new][DEBUG ] Creating new cluster named ceph
   [ceph_deploy.new][INFO  ] making sure passwordless SSH succeeds
   [node1][DEBUG ] connected to host: node1
   [node1][DEBUG ] detect platform information from remote host
   [node1][DEBUG ] detect machine type
   [node1][DEBUG ] find the location of an executable
   [node1][INFO  ] Running command: /usr/sbin/ip link show
   [node1][INFO  ] Running command: /usr/sbin/ip addr show
   [node1][DEBUG ] IP addresses found: [u'192.168.10.15']
   [ceph_deploy.new][DEBUG ] Resolving host node1
   [ceph_deploy.new][DEBUG ] Monitor node1 at 192.168.10.15
   [ceph_deploy.new][DEBUG ] Monitor initial members are ['node1']
   [ceph_deploy.new][DEBUG ] Monitor addrs are [u'192.168.10.15']
   [ceph_deploy.new][DEBUG ] Creating a random mon key...
   [ceph_deploy.new][DEBUG ] Writing monitor keyring to ceph.mon.keyring...
   [ceph_deploy.new][DEBUG ] Writing initial config to ceph.conf...
   [root@node1 ceph-deploy]# ll
   total 12
   -rw-r--r-- 1 root root  263 Jul 12 02:03 ceph.conf
   -rw-r--r-- 1 root root 3007 Jul 12 02:03 ceph-deploy-ceph.log
   -rw------- 1 root root   73 Jul 12 02:03 ceph.mon.keyring

   ```

3. 手动安装ceph安装包(三个节点全部手动安装)，如果使用自动安装的话，ceph会修改yum源，导致安装非常慢

   ```
   # yum -y install ceph ceph-mon ceph-mgr ceph-radosgw ceph-mds
   ```

4. 初始化mon,会生成秘钥文件

   ```
   # ceph-deploy mon create-initial
   [ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
   [ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy mon create-initial
   [ceph_deploy.cli][INFO  ] ceph-deploy options:
   [ceph_deploy.cli][INFO  ]  username                      : None
   [ceph_deploy.cli][INFO  ]  verbose                       : False
   [ceph_deploy.cli][INFO  ]  overwrite_conf                : False
   [ceph_deploy.cli][INFO  ]  subcommand                    : create-initial
   [ceph_deploy.cli][INFO  ]  quiet                         : False
   [ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7fcf869caf80>
   [ceph_deploy.cli][INFO  ]  cluster                       : ceph
   [ceph_deploy.cli][INFO  ]  func                          : <function mon at 0x7fcf86e39848>
   [ceph_deploy.cli][INFO  ]  ceph_conf                     : None
   [ceph_deploy.cli][INFO  ]  default_release               : False
   [ceph_deploy.cli][INFO  ]  keyrings                      : None
   [ceph_deploy.mon][DEBUG ] Deploying mon, cluster ceph hosts node1
   [ceph_deploy.mon][DEBUG ] detecting platform for host node1 ...
   [node1][DEBUG ] connected to host: node1
   [node1][DEBUG ] detect platform information from remote host
   [node1][DEBUG ] detect machine type
   [node1][DEBUG ] find the location of an executable
   [ceph_deploy.mon][INFO  ] distro info: CentOS Linux 7.8.2003 Core
   [node1][DEBUG ] determining if provided host has same hostname in remote
   [node1][DEBUG ] get remote short hostname
   [node1][DEBUG ] deploying mon to node1
   [node1][DEBUG ] get remote short hostname
   [node1][DEBUG ] remote hostname: node1
   [node1][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
   [node1][DEBUG ] create the mon path if it does not exist
   [node1][DEBUG ] checking for done path: /var/lib/ceph/mon/ceph-node1/done
   [node1][DEBUG ] done path does not exist: /var/lib/ceph/mon/ceph-node1/done
   [node1][INFO  ] creating keyring file: /var/lib/ceph/tmp/ceph-node1.mon.keyring
   [node1][DEBUG ] create the monitor keyring file
   [node1][INFO  ] Running command: ceph-mon --cluster ceph --mkfs -i node1 --keyring /var/lib/ceph/tmp/ceph-node1.mon.keyring --setuser 167 --setgroup 167
   [node1][INFO  ] unlinking keyring file /var/lib/ceph/tmp/ceph-node1.mon.keyring
   [node1][DEBUG ] create a done file to avoid re-doing the mon deployment
   [node1][DEBUG ] create the init path if it does not exist
   [node1][INFO  ] Running command: systemctl enable ceph.target
   [node1][INFO  ] Running command: systemctl enable ceph-mon@node1
   [node1][WARNIN] Created symlink from /etc/systemd/system/ceph-mon.target.wants/ceph-mon@node1.service to /usr/lib/systemd/system/ceph-mon@.service.
   [node1][INFO  ] Running command: systemctl start ceph-mon@node1
   [node1][INFO  ] Running command: ceph --cluster=ceph --admin-daemon /var/run/ceph/ceph-mon.node1.asok mon_status
   [node1][DEBUG ] ********************************************************************************
   [node1][DEBUG ] status for monitor: mon.node1
   [node1][DEBUG ] {
   [node1][DEBUG ]   "election_epoch": 3,
   [node1][DEBUG ]   "extra_probe_peers": [],
   [node1][DEBUG ]   "feature_map": {
   [node1][DEBUG ]     "mon": [
   [node1][DEBUG ]       {
   [node1][DEBUG ]         "features": "0x3ffddff8ffacffff",
   [node1][DEBUG ]         "num": 1,
   [node1][DEBUG ]         "release": "luminous"
   [node1][DEBUG ]       }
   [node1][DEBUG ]     ]
   [node1][DEBUG ]   },
   [node1][DEBUG ]   "features": {
   [node1][DEBUG ]     "quorum_con": "4611087854031667199",
   [node1][DEBUG ]     "quorum_mon": [
   [node1][DEBUG ]       "kraken",
   [node1][DEBUG ]       "luminous",
   [node1][DEBUG ]       "mimic",
   [node1][DEBUG ]       "osdmap-prune",
   [node1][DEBUG ]       "nautilus"
   [node1][DEBUG ]     ],
   [node1][DEBUG ]     "required_con": "2449958747315912708",
   [node1][DEBUG ]     "required_mon": [
   [node1][DEBUG ]       "kraken",
   [node1][DEBUG ]       "luminous",
   [node1][DEBUG ]       "mimic",
   [node1][DEBUG ]       "osdmap-prune",
   [node1][DEBUG ]       "nautilus"
   [node1][DEBUG ]     ]
   [node1][DEBUG ]   },
   [node1][DEBUG ]   "monmap": {
   [node1][DEBUG ]     "created": "2020-07-12 04:39:25.334373",
   [node1][DEBUG ]     "epoch": 1,
   [node1][DEBUG ]     "features": {
   [node1][DEBUG ]       "optional": [],
   [node1][DEBUG ]       "persistent": [
   [node1][DEBUG ]         "kraken",
   [node1][DEBUG ]         "luminous",
   [node1][DEBUG ]         "mimic",
   [node1][DEBUG ]         "osdmap-prune",
   [node1][DEBUG ]         "nautilus"
   [node1][DEBUG ]       ]
   [node1][DEBUG ]     },
   [node1][DEBUG ]     "fsid": "3ae84859-aea9-4b37-81c9-a36ee6b4ce1b",
   [node1][DEBUG ]     "min_mon_release": 14,
   [node1][DEBUG ]     "min_mon_release_name": "nautilus",
   [node1][DEBUG ]     "modified": "2020-07-12 04:39:25.334373",
   [node1][DEBUG ]     "mons": [
   [node1][DEBUG ]       {
   [node1][DEBUG ]         "addr": "192.168.10.15:6789/0",
   [node1][DEBUG ]         "name": "node1",
   [node1][DEBUG ]         "public_addr": "192.168.10.15:6789/0",
   [node1][DEBUG ]         "public_addrs": {
   [node1][DEBUG ]           "addrvec": [
   [node1][DEBUG ]             {
   [node1][DEBUG ]               "addr": "192.168.10.15:3300",
   [node1][DEBUG ]               "nonce": 0,
   [node1][DEBUG ]               "type": "v2"
   [node1][DEBUG ]             },
   [node1][DEBUG ]             {
   [node1][DEBUG ]               "addr": "192.168.10.15:6789",
   [node1][DEBUG ]               "nonce": 0,
   [node1][DEBUG ]               "type": "v1"
   [node1][DEBUG ]             }
   [node1][DEBUG ]           ]
   [node1][DEBUG ]         },
   [node1][DEBUG ]         "rank": 0
   [node1][DEBUG ]       }
   [node1][DEBUG ]     ]
   [node1][DEBUG ]   },
   [node1][DEBUG ]   "name": "node1",
   [node1][DEBUG ]   "outside_quorum": [],
   [node1][DEBUG ]   "quorum": [
   [node1][DEBUG ]     0
   [node1][DEBUG ]   ],
   [node1][DEBUG ]   "quorum_age": 2,
   [node1][DEBUG ]   "rank": 0,
   [node1][DEBUG ]   "state": "leader",
   [node1][DEBUG ]   "sync_provider": []
   [node1][DEBUG ] }
   [node1][DEBUG ] ********************************************************************************
   [node1][INFO  ] monitor: mon.node1 is running
   [node1][INFO  ] Running command: ceph --cluster=ceph --admin-daemon /var/run/ceph/ceph-mon.node1.asok mon_status
   [ceph_deploy.mon][INFO  ] processing monitor mon.node1
   [node1][DEBUG ] connected to host: node1
   [node1][DEBUG ] detect platform information from remote host
   [node1][DEBUG ] detect machine type
   [node1][DEBUG ] find the location of an executable
   [node1][INFO  ] Running command: ceph --cluster=ceph --admin-daemon /var/run/ceph/ceph-mon.node1.asok mon_status
   [ceph_deploy.mon][INFO  ] mon.node1 monitor has reached quorum!
   [ceph_deploy.mon][INFO  ] all initial monitors are running and have formed quorum
   [ceph_deploy.mon][INFO  ] Running gatherkeys...
   [ceph_deploy.gatherkeys][INFO  ] Storing keys in temp directory /tmp/tmpit1Ubk
   [node1][DEBUG ] connected to host: node1
   [node1][DEBUG ] detect platform information from remote host
   [node1][DEBUG ] detect machine type
   [node1][DEBUG ] get remote short hostname
   [node1][DEBUG ] fetch remote file
   [node1][INFO  ] Running command: /usr/bin/ceph --connect-timeout=25 --cluster=ceph --admin-daemon=/var/run/ceph/ceph-mon.node1.asok mon_status
   [node1][INFO  ] Running command: /usr/bin/ceph --connect-timeout=25 --cluster=ceph --name mon. --keyring=/var/lib/ceph/mon/ceph-node1/keyring auth get client.admin
   [node1][INFO  ] Running command: /usr/bin/ceph --connect-timeout=25 --cluster=ceph --name mon. --keyring=/var/lib/ceph/mon/ceph-node1/keyring auth get client.bootstrap-mds
   [node1][INFO  ] Running command: /usr/bin/ceph --connect-timeout=25 --cluster=ceph --name mon. --keyring=/var/lib/ceph/mon/ceph-node1/keyring auth get client.bootstrap-mgr
   [node1][INFO  ] Running command: /usr/bin/ceph --connect-timeout=25 --cluster=ceph --name mon. --keyring=/var/lib/ceph/mon/ceph-node1/keyring auth get client.bootstrap-osd
   [node1][INFO  ] Running command: /usr/bin/ceph --connect-timeout=25 --cluster=ceph --name mon. --keyring=/var/lib/ceph/mon/ceph-node1/keyring auth get client.bootstrap-rgw
   [ceph_deploy.gatherkeys][INFO  ] Storing ceph.client.admin.keyring
   [ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-mds.keyring
   [ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-mgr.keyring
   [ceph_deploy.gatherkeys][INFO  ] keyring 'ceph.mon.keyring' already exists
   [ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-osd.keyring
   [ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-rgw.keyring
   [ceph_deploy.gatherkeys][INFO  ] Destroy temp directory /tmp/tmpit1Ubk
   
   #查看生成的秘钥文件
   # ll
   total 44
   -rw------- 1 root root   113 Jul 12 04:39 ceph.bootstrap-mds.keyring
   -rw------- 1 root root   113 Jul 12 04:39 ceph.bootstrap-mgr.keyring
   -rw------- 1 root root   113 Jul 12 04:39 ceph.bootstrap-osd.keyring
   -rw------- 1 root root   113 Jul 12 04:39 ceph.bootstrap-rgw.keyring
   -rw------- 1 root root   151 Jul 12 04:39 ceph.client.admin.keyring
   -rw-r--r-- 1 root root   263 Jul 12 02:03 ceph.conf
   -rw-r--r-- 1 root root 15354 Jul 12 04:39 ceph-deploy-ceph.log
   -rw------- 1 root root    73 Jul 12 02:03 ceph.mon.keyring
   
   ```
   
5. 将生成的秘钥文件和配置文件拷贝到其他两个节点

   ```
   # ceph-deploy admin node1 node2 node3
   [ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
   [ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy admin node1 node2 node3
   [ceph_deploy.cli][INFO  ] ceph-deploy options:
   [ceph_deploy.cli][INFO  ]  username                      : None
   [ceph_deploy.cli][INFO  ]  verbose                       : False
   [ceph_deploy.cli][INFO  ]  overwrite_conf                : False
   [ceph_deploy.cli][INFO  ]  quiet                         : False
   [ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7f85900d5560>
   [ceph_deploy.cli][INFO  ]  cluster                       : ceph
   [ceph_deploy.cli][INFO  ]  client                        : ['node1', 'node2', 'node3']
   [ceph_deploy.cli][INFO  ]  func                          : <function admin at 0x7f8590975668>
   [ceph_deploy.cli][INFO  ]  ceph_conf                     : None
   [ceph_deploy.cli][INFO  ]  default_release               : False
   [ceph_deploy.admin][DEBUG ] Pushing admin keys and conf to node1
   [node1][DEBUG ] connected to host: node1
   [node1][DEBUG ] detect platform information from remote host
   [node1][DEBUG ] detect machine type
   [node1][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
   [ceph_deploy.admin][DEBUG ] Pushing admin keys and conf to node2
   [node2][DEBUG ] connected to host: node2
   [node2][DEBUG ] detect platform information from remote host
   [node2][DEBUG ] detect machine type
   [node2][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
   [ceph_deploy.admin][DEBUG ] Pushing admin keys and conf to node3
   [node3][DEBUG ] connected to host: node3
   [node3][DEBUG ] detect platform information from remote host
   [node3][DEBUG ] detect machine type
   [node3][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
	```

6. 执行ceph -s查看集群状态

   ```
   #  ceph -s
     cluster:
       id:     3ae84859-aea9-4b37-81c9-a36ee6b4ce1b
       health: HEALTH_OK
       
     services:
       mon: 1 daemons, quorum node1 (age 11m)
       mgr: no daemons active
       osd: 0 osds: 0 up, 0 in
       
     data:
       pools:   0 pools, 0 pgs
       objects: 0 objects, 0 B
       usage:   0 B used, 0 B / 0 B avail
       pgs:

   ```

7. 部署mgr监控节点，在node1上执行

   ```
   # ceph-deploy mgr create node1
   [ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
   [ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy mgr create node1
   [ceph_deploy.cli][INFO  ] ceph-deploy options:
   [ceph_deploy.cli][INFO  ]  username                      : None
   [ceph_deploy.cli][INFO  ]  verbose                       : False
   [ceph_deploy.cli][INFO  ]  mgr                           : [('node1', 'node1')]
   [ceph_deploy.cli][INFO  ]  overwrite_conf                : False
   [ceph_deploy.cli][INFO  ]  subcommand                    : create
   [ceph_deploy.cli][INFO  ]  quiet                         : False
   [ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7f606d7758c0>
   [ceph_deploy.cli][INFO  ]  cluster                       : ceph
   [ceph_deploy.cli][INFO  ]  func                          : <function mgr at 0x7f606d738578>
   [ceph_deploy.cli][INFO  ]  ceph_conf                     : None
   [ceph_deploy.cli][INFO  ]  default_release               : False
   [ceph_deploy.mgr][DEBUG ] Deploying mgr, cluster ceph hosts node1:node1
   [node1][DEBUG ] connected to host: node1
   [node1][DEBUG ] detect platform information from remote host
   [node1][DEBUG ] detect machine type
   [ceph_deploy.mgr][INFO  ] Distro info: CentOS Linux 7.8.2003 Core
   [ceph_deploy.mgr][DEBUG ] remote host will use systemd
   [ceph_deploy.mgr][DEBUG ] deploying mgr bootstrap to node1
   [node1][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
   [node1][WARNIN] mgr keyring does not exist yet, creating one
   [node1][DEBUG ] create a keyring file
   [node1][DEBUG ] create path recursively if it doesn't exist
   [node1][INFO  ] Running command: ceph --cluster ceph --name client.bootstrap-mgr --keyring /var/lib/ceph/bootstrap-mgr/ceph.keyring auth get-or-create mgr.node1 mon allow profile mgr osd allow * mds allow * -o /var/lib/ceph/mgr/ceph-node1/keyring
   [node1][INFO  ] Running command: systemctl enable ceph-mgr@node1
   [node1][WARNIN] Created symlink from /etc/systemd/system/ceph-mgr.target.wants/ceph-mgr@node1.service to /usr/lib/systemd/system/ceph-mgr@.service.
   [node1][INFO  ] Running command: systemctl start ceph-mgr@node1
   [node1][INFO  ] Running command: systemctl enable ceph.target

   ```

   

8. 添加OSD磁盘,在node1上节点执行

   说明：Ceph到了L版本开始支持bluestore，之前均使用filestore，对于fielstore来说，将ssd以分区的形式指定为journal盘，用于加速是业界标准的方案，ssd的大小通常为10G；filestore逐渐废弃，推荐使用bluestore的模式，对于bluestore模式来说，其使用wal（日志分区）和db（元数据分区）的方式来存储，这两个分区分别存储BlueStore后端产生的元数据和日志文件，这样整个存储系统通过元数据对数据的操作效率极高，同时通过日志事务来维持系统的稳定性。官方推荐，wal通常为10G就足够了，db分区不低于数据盘的4%。
   以你环境来说，将ssd划分4个10G的分区用于wal日志存储，同时划分三个160G分区用于db，osd创建和添加的时候采用以下模式添加即可。
   ceph-deploy osd create ${node} --data /dev/sd${i} --block-wal /dev/nvme0n1p${j} --block-db /dev/nvme0n1p${k}

   

   三台节点先格式化ssd日志分区（分区链接：https://blog.csdn.net/qq_35036995/article/details/80532351）

   ```
   #### 三台节点根据磁盘格式化分区 wal每个分区20G db每个160G
   #格式化ssd 为gpt
   [root@node1 ~]# parted -s /dev/sdb mklabel gpt
   [root@node1 ~]# fdisk -l -u /dev/sdb
   WARNING: fdisk GPT support is currently new, and therefore in an experimental phase. Use at your own discretion.
   
   Disk /dev/sdb: 800.2 GB, 800166076416 bytes, 1562824368 sectors
   Units = sectors of 1 * 512 = 512 bytes
   Sector size (logical/physical): 512 bytes / 4096 bytes
   I/O size (minimum/optimal): 4096 bytes / 4096 bytes
   Disk label type: gpt
   Disk identifier: 213BC572-48D0-4DE9-93F4-5B41264FFA82
   
   ###最后分区结果如下，预留一下，防止最后扩充
   
   #         Start          End    Size  Type            Name
    1         2048     41945087     20G  Linux filesyste
    2     41945088    377489407    160G  Linux filesyste
    3    377489408    419432447     20G  Linux filesyste
    4    419432448    754976767    160G  Linux filesyste
    
   node1执行添加node1的osd
   [root@node1 ceph-deploy]# ceph-deploy osd create node1 --data /dev/sdc --block-wal /dev/sdb1 --block-db /dev/sdb2
   [root@node1 ceph-deploy]# ceph-deploy osd create node1 --data /dev/sdd --block-wal /dev/sdb3 --block-db /dev/sdb4
   
   node1执行添加node2的osd
   [root@node1 ceph-deploy]# ceph-deploy osd create node2 --data /dev/sdc --block-wal /dev/sdb1 --block-db /dev/sdb2
   [root@node1 ceph-deploy]# ceph-deploy osd create node2 --data /dev/sdd --block-wal /dev/sdb3 --block-db /dev/sdb4
   
   node1执行添加node3的osd
   [root@node1 ceph-deploy]# ceph-deploy osd create node3 --data /dev/sdc --block-wal /dev/sdb1 --block-db /dev/sdb2
   [root@node1 ceph-deploy]# ceph-deploy osd create node3 --data /dev/sdd --block-wal /dev/sdb3 --block-db /dev/sdb4
   ```

   

1. 进入 ceph-deploy目录。添加mon节点，将node2和node3添加到集群里

   ```
   # ceph-deploy mon add node2 --address 192.168.10.16
   # ceph-deploy mon add node3 --address 192.168.10.17
   # 查看节点添加成功并且已经进入选举集群内
   # ceph quorum_status --format json-pretty

   {
       "election_epoch": 16,
       "quorum": [
           0,
           1,
           2
       ],
       "quorum_names": [
           "node1",
           "node2",
           "node3"
       ],
       "quorum_leader_name": "node1",
       "quorum_age": 29,
       "monmap": {
           "epoch": 3,
           "fsid": "3ae84859-aea9-4b37-81c9-a36ee6b4ce1b",
           "modified": "2020-07-12 00:07:48.483888",
           "created": "2020-07-12 04:39:25.334373",
           "min_mon_release": 14,
           "min_mon_release_name": "nautilus",
           "features": {
               "persistent": [
                   "kraken",
                   "luminous",
                   "mimic",
                   "osdmap-prune",
                   "nautilus"
               ],
               "optional": []
           },
           "mons": [
               {
                   "rank": 0,
                   "name": "node1",
                   "public_addrs": {
                       "addrvec": [
                           {
                               "type": "v2",
                               "addr": "192.168.10.15:3300",
                               "nonce": 0
                           },
                           {
                               "type": "v1",
                               "addr": "192.168.10.15:6789",
                               "nonce": 0
                           }
                       ]
                   },
                   "addr": "192.168.10.15:6789/0",
                   "public_addr": "192.168.10.15:6789/0"
               },
               {
                   "rank": 1,
                   "name": "node2",
                   "public_addrs": {
                       "addrvec": [
                           {
                               "type": "v2",
                               "addr": "192.168.10.108:3300",
                               "nonce": 0
                           },
                           {
                               "type": "v1",
                               "addr": "192.168.10.108:6789",
                               "nonce": 0
                           }
                       ]
                   },
                   "addr": "192.168.10.108:6789/0",
                   "public_addr": "192.168.10.108:6789/0"
               },
               {
                   "rank": 2,
                   "name": "node3",
                   "public_addrs": {
                       "addrvec": [
                           {
                               "type": "v2",
                               "addr": "192.168.10.109:3300",
                               "nonce": 0
                           },
                           {
                               "type": "v1",
                               "addr": "192.168.10.109:6789",
                               "nonce": 0
                           }
                       ]
                   },
                   "addr": "192.168.10.109:6789/0",
                   "public_addr": "192.168.10.109:6789/0"
               }
           ]
       }
   }

   # ceph mon dump
   dumped monmap epoch 3
   epoch 3
   fsid 3ae84859-aea9-4b37-81c9-a36ee6b4ce1b
   last_changed 2020-07-12 00:07:48.483888
   created 2020-07-12 04:39:25.334373
   min_mon_release 14 (nautilus)
   0: [v2:192.168.10.15:3300/0,v1:192.168.10.15:6789/0] mon.node1
   1: [v2:192.168.10.108:3300/0,v1:192.168.10.108:6789/0] mon.node2
   2: [v2:192.168.10.109:3300/0,v1:192.168.10.109:6789/0] mon.node3

   ```

2. 扩容mgr 扩容 node2和node3

   ```
   # ceph-deploy mgr create node2 node3
   [ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
   [ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy mgr create node1 node2
   [ceph_deploy.cli][INFO  ] ceph-deploy options:
   [ceph_deploy.cli][INFO  ]  username                      : None
   [ceph_deploy.cli][INFO  ]  verbose                       : False
   [ceph_deploy.cli][INFO  ]  mgr                           : [('node1', 'node1'), ('node2', 'node2')]
   [ceph_deploy.cli][INFO  ]  overwrite_conf                : False
   [ceph_deploy.cli][INFO  ]  subcommand                    : create
   [ceph_deploy.cli][INFO  ]  quiet                         : False
   [ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7f4aa035f908>
   [ceph_deploy.cli][INFO  ]  cluster                       : ceph
   [ceph_deploy.cli][INFO  ]  func                          : <function mgr at 0x7f4aa0322578>
   [ceph_deploy.cli][INFO  ]  ceph_conf                     : None
   [ceph_deploy.cli][INFO  ]  default_release               : False
   [ceph_deploy.mgr][DEBUG ] Deploying mgr, cluster ceph hosts node1:node1 node2:node2
   [node1][DEBUG ] connected to host: node1
   [node1][DEBUG ] detect platform information from remote host
   [node1][DEBUG ] detect machine type
   [ceph_deploy.mgr][INFO  ] Distro info: CentOS Linux 7.8.2003 Core
   [ceph_deploy.mgr][DEBUG ] remote host will use systemd
   [ceph_deploy.mgr][DEBUG ] deploying mgr bootstrap to node1
   [node1][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
   [node1][DEBUG ] create path recursively if it doesn't exist
   [node1][INFO  ] Running command: ceph --cluster ceph --name client.bootstrap-mgr --keyring /var/lib/ceph/bootstrap-mgr/ceph.keyring auth get-or-create mgr.node1 mon allow profile mgr osd allow * mds allow * -o /var/lib/ceph/mgr/ceph-node1/keyring
   [node1][INFO  ] Running command: systemctl enable ceph-mgr@node1
   [node1][INFO  ] Running command: systemctl start ceph-mgr@node1
   [node1][INFO  ] Running command: systemctl enable ceph.target
   [node2][DEBUG ] connected to host: node2
   [node2][DEBUG ] detect platform information from remote host
   [node2][DEBUG ] detect machine type
   [ceph_deploy.mgr][INFO  ] Distro info: CentOS Linux 7.8.2003 Core
   [ceph_deploy.mgr][DEBUG ] remote host will use systemd
   [ceph_deploy.mgr][DEBUG ] deploying mgr bootstrap to node2
   [node2][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
   [node2][WARNIN] mgr keyring does not exist yet, creating one
   [node2][DEBUG ] create a keyring file
   [node2][DEBUG ] create path recursively if it doesn't exist
   [node2][INFO  ] Running command: ceph --cluster ceph --name client.bootstrap-mgr --keyring /var/lib/ceph/bootstrap-mgr/ceph.keyring auth get-or-create mgr.node2 mon allow profile mgr osd allow * mds allow * -o /var/lib/ceph/mgr/ceph-node2/keyring
   [node2][INFO  ] Running command: systemctl enable ceph-mgr@node2
   [node2][WARNIN] Created symlink from /etc/systemd/system/ceph-mgr.target.wants/ceph-mgr@node2.service to /usr/lib/systemd/system/ceph-mgr@.service.
   [node2][INFO  ] Running command: systemctl start ceph-mgr@node2
   [node2][INFO  ] Running command: systemctl enable ceph.target
   
   ```
   
11. 也可以开启pg_autoscale

    ```
    # ceph mgr module enable pg_autoscaler
    # ceph osd pool set nvme_pool pg_autoscale_mode on
    # ceph osd pool autoscale-status
    POOL                  SIZE TARGET SIZE RATE RAW CAPACITY  RATIO TARGET RATIO EFFECTIVE RATIO BIAS PG_NUM NEW PG_NUM AUTOSCALE 
    .rgw.root            1245               2.0       44712G 0.0000                               1.0     32            warn      
    default.rgw.meta        0               2.0       44712G 0.0000                               1.0     32            warn      
    default.rgw.log         0               2.0       44712G 0.0000                               1.0     32            warn      
    nvme_pool           347.8G              2.0       17884G 0.0389                               1.0     32            on        
    default.rgw.control     0               2.0       44712G 0.0000                               1.0     32            warn
    
    让我们看一下每一列：
    
    SIZE列仅报告存储在池中的数据总量（以字节为单位）。 这包括对象数据和omap键/值数据。
    TARGET SIZE报告有关池的预期大小的任何管理员输入。如果池刚刚创建，则最初不会存储任何数据，但管理员通常对最终将存储多少数据有所了解。如果提供，则使用此值或实际大小中的较大者来计算池的理想PG数。
    RATE值是原始存储占用与存储的用户数据的比率，对于复本桶（bucket）来说，它只是一个复制因子，对于纠删码池来说，它是一个(通常)较小的比率。
    RAW CAPACITY是池由CRUSH映射到的存储设备上的原始存储容量。
    RATIO是池占用的总存储空间的比率。
    TARGET RATIO是管理员提供的值，类似于TARGET SIZE，它表示用户预计池将占用集群总存储的多大部分。
    最后，PG_NUM是池的当前PG数量，而NEW PG_NUM（如果存在）是建议值。
    AUTOSCALE列指示池的模式，可以是禁用、警告或启用。
    ```

    

## 创建块存储

创建资源池:

```
[root@node1 ceph-deploy]# ceph osd pool create ceph-demo 64 64 <ssd_rule>
pool 'ceph-demo' created
```

官方建议pg和pgs数量

- 少于 5 个 OSD 时可把 pg_num 设置为 128
- OSD 数量在 5 到 10 个时，可把 pg_num 设置为 512
- OSD 数量在 10 到 50 个时，可把 pg_num 设置为 4096
- OSD 数量大于 50 时，你得理解权衡方法、以及如何自己计算 pg_num 取值
- 自己计算 pg_num 取值时可借助 [pgcalc](http://ceph.com/pgcalc/) 工具

查看资源池

```
[root@node1 ceph-deploy]# ceph osd pool get ceph-demo pg_num
pg_num: 64
[root@node1 ceph-deploy]# ceph osd pool get ceph-demo pgp_num
pgp_num: 64
[root@node1 ceph-deploy]# ceph osd pool get ceph-demo size  #查看副本数量
size: 3
```

设置资源池(也可以调整pg和pgp，pg和pgp必须一样)

```
[root@node1 ceph-deploy]# ceph osd pool set ceph-demo size 2 #调整副本数
set pool 2 size to 2
[root@node1 ceph-deploy]# ceph osd pool get ceph-demo size
size: 2
```

删除资源池

1. 打开mon节点的配置文件：

   ```
   [root@node1 ceph]# vim /root/ceph-deploy/ceph.conf
   ```

2. 在配置文件中添加如下内容：

   ```
   [mon]
   mon allow pool delete = true
   ```

3. 重启ceph-mon服务：

   ```
   [root@node1 ceph]# systemctl restart ceph-mon.target
   ```

4. 执行删除pool命令：

   ```
   [root@node1 mnt]# ceph osd pool delete ceph-demo ceph-demo --yes-i-really-really-mean-it
   pool '64' removed
   [root@node1 mnt]# rados lspools
   ceph-demo
   ```

注意：修改完配置文件后要将配置文件推送到其他节点

```
[root@node1 ceph-deploy]# ceph-deploy --overwrite-conf config push node1 node2 node3
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy --overwrite-conf config push node2
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  overwrite_conf                : True
[ceph_deploy.cli][INFO  ]  subcommand                    : push
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7f043b2adbd8>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  client                        : ['node2']
[ceph_deploy.cli][INFO  ]  func                          : <function config at 0x7f043b6f40c8>
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.config][DEBUG ] Pushing config to node2
[node2][DEBUG ] connected to host: node2 
[node2][DEBUG ] detect platform information from remote host
[node2][DEBUG ] detect machine type
[node2][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[root@node1 ceph-deploy]# ceph-deploy --overwrite-conf config push node3
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy --overwrite-conf config push node3
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  overwrite_conf                : True
[ceph_deploy.cli][INFO  ]  subcommand                    : push
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7fb5d68e1bd8>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  client                        : ['node3']
[ceph_deploy.cli][INFO  ]  func                          : <function config at 0x7fb5d6d280c8>
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.config][DEBUG ] Pushing config to node3
[node3][DEBUG ] connected to host: node3 
[node3][DEBUG ] detect platform information from remote host
[node3][DEBUG ] detect machine type
[node3][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
```



创建快储存 一下两种方法都可以 -p指定资源池 --image指定块存储名字(创建的是瘦模式，用多少占多少资源)

```
[root@node1 ceph-deploy]# rbd create -p ceph-demo --image rbd-demo.img --size 10G --image-feature  layering
[root@node1 ceph-deploy]# rbd create ceph-demo/rbd-demo1.img --size 10G
```

创建块存储的时候linux内核7版本features命令有些命令不支持，创建的时候需要去掉，也可以后面去掉

```
[root@node1 ceph-deploy]# rbd feature disable ceph-demo/rbd-demo.img deep-flatten
[root@node1 ceph-deploy]# rbd feature disable ceph-demo/rbd-demo.img fast-diff
[root@node1 ceph-deploy]# rbd feature disable ceph-demo/rbd-demo.img object-map
[root@node1 ceph-deploy]# rbd feature disable ceph-demo/rbd-demo.img exclusive-lock
[root@node1 ceph-deploy]# rbd info ceph-demo/rbd-demo.img #再次查看features特性就只剩下layering
rbd image 'rbd-demo.img':
        size 10 GiB in 2560 objects
        order 22 (4 MiB objects)
        snapshot_count: 0
        id: 38d61c80322b
        block_name_prefix: rbd_data.38d61c80322b
        format: 2
        features: layering
        op_features: 
        flags: 
        create_timestamp: Tue Aug 18 21:38:41 2020
        access_timestamp: Tue Aug 18 21:38:41 2020
        modify_timestamp: Tue Aug 18 21:38:41 2020
```



查看快储存

```
[root@node1 ceph-deploy]# rbd -p ceph-demo ls
rbd-demo.img
[root@node1 ceph-deploy]# rbd info ceph-demo/rbd-demo.img
rbd image 'rbd-demo.img':
        size 10 GiB in 2560 objects
        order 22 (4 MiB objects)
        snapshot_count: 0
        id: 38d61c80322b
        block_name_prefix: rbd_data.38d61c80322b
        format: 2
        features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
        op_features: 
        flags: 
        create_timestamp: Tue Aug 18 21:38:41 2020
        access_timestamp: Tue Aug 18 21:38:41 2020
        modify_timestamp: Tue Aug 18 21:38:41 2020
```

删除块存储

```
[root@node1 ceph-deploy]# rbd rm -p ceph-demo --image rbd-demo1.img
Removing image: 100% complete...done.
[root@node1 ceph-deploy]# rbd -p ceph-demo ls
rbd-demo.img
```

在本地挂载块设备 云盘使用 不建议用磁盘分区

```
[root@node1 ceph-deploy]# rbd map ceph-demo/rbd-demo.img
/dev/rbd0
[root@node1 ceph-deploy]# fdisk -l
Disk /dev/rbd0: 10.7 GB, 10737418240 bytes, 20971520 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 4194304 bytes / 4194304 bytes
#格式化
[root@node1 ceph-deploy]# mkfs.xfs /dev/rbd0
meta-data=/dev/rbd0              isize=512    agcount=16, agsize=163840 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2621440, imaxpct=25
         =                       sunit=1024   swidth=1024 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=8 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@node1 ceph-deploy]# mount /dev/rbd0 /mnt/rdb-demo
[root@node1 ceph-deploy]# cd /mnt/rdb-demo/
[root@node1 rdb-demo]# touch test
[root@node1 rdb-demo]# echo test > test
[root@node1 rdb-demo]# cat test
test

#取消映射
[root@node1 ceph-deploy]# rbd unmap /dev/rbd0
```

rados查看rbd块存储

```
[root@node1 rdb-demo]# rbd info ceph-demo/rbd-demo.img
rbd image 'rbd-demo.img':
        size 10 GiB in 2560 objects
        order 22 (4 MiB objects)
        snapshot_count: 0
        id: 38d61c80322b
        block_name_prefix: rbd_data.38d61c80322b
        format: 2
        features: layering
        op_features: 
        flags: 
        create_timestamp: Tue Aug 18 21:38:41 2020
        access_timestamp: Tue Aug 18 21:38:41 2020
        modify_timestamp: Tue Aug 18 21:38:41 2020
[root@node1 rdb-demo]# rados -p ceph-demo
rados: you must give an action. Try --help
[root@node1 rdb-demo]# rados -p ceph-demo ls
rbd_data.38d61c80322b.0000000000000501
rbd_data.38d61c80322b.0000000000000820
rbd_data.38d61c80322b.0000000000000502
rbd_directory
rbd_data.38d61c80322b.0000000000000001
rbd_data.38d61c80322b.0000000000000780
rbd_data.38d61c80322b.0000000000000960
rbd_data.38d61c80322b.00000000000000a0
rbd_id.rbd-demo.img
rbd_info
rbd_data.38d61c80322b.00000000000006e0
rbd_data.38d61c80322b.0000000000000500
rbd_data.38d61c80322b.0000000000000140
rbd_data.38d61c80322b.00000000000009ff
rbd_data.38d61c80322b.00000000000005a0
rbd_data.38d61c80322b.00000000000008c0
rbd_data.38d61c80322b.00000000000003c0
rbd_data.38d61c80322b.0000000000000320
rbd_data.38d61c80322b.0000000000000460
rbd_data.38d61c80322b.0000000000000000
rbd_data.38d61c80322b.0000000000000640
rbd_data.38d61c80322b.0000000000000280
rbd_trash
rbd_header.38d61c80322b
rbd_data.38d61c80322b.00000000000001e0
[root@node1 rdb-demo]# rados -p ceph-demo ls | grep rbd_data.38d61c80322b
rbd_data.38d61c80322b.0000000000000501
rbd_data.38d61c80322b.0000000000000820
rbd_data.38d61c80322b.0000000000000502
rbd_data.38d61c80322b.0000000000000001
rbd_data.38d61c80322b.0000000000000780
rbd_data.38d61c80322b.0000000000000960
rbd_data.38d61c80322b.00000000000000a0
rbd_data.38d61c80322b.00000000000006e0
rbd_data.38d61c80322b.0000000000000500
rbd_data.38d61c80322b.0000000000000140
rbd_data.38d61c80322b.00000000000009ff
rbd_data.38d61c80322b.00000000000005a0
rbd_data.38d61c80322b.00000000000008c0
rbd_data.38d61c80322b.00000000000003c0
rbd_data.38d61c80322b.0000000000000320
rbd_data.38d61c80322b.0000000000000460
rbd_data.38d61c80322b.0000000000000000
rbd_data.38d61c80322b.0000000000000640
rbd_data.38d61c80322b.0000000000000280
rbd_data.38d61c80322b.00000000000001e0

#查看大小
[root@node1 rdb-demo]# rados -p ceph-demo stat rbd_data.38d61c80322b.0000000000000460
ceph-demo/rbd_data.38d61c80322b.0000000000000460 mtime 2020-08-18 22:04:24.000000, size 16384
```

查看每个对象最终落到哪个pg哪个osd.由下图可以看出 落在了两个节点的两个osd上

```
[root@node1 rdb-demo]# ceph osd map ceph-demo rbd_data.38d61c80322b.0000000000000460 
osdmap e38 pool 'ceph-demo' (2) object 'rbd_data.38d61c80322b.0000000000000460' -> pg 2.7a596a69 (2.29) -> up ([4,3], p4) acting ([4,3], p4)
[root@node1 rdb-demo]# ceph osd tree
ID CLASS WEIGHT   TYPE NAME      STATUS REWEIGHT PRI-AFF 
-1       26.56436 root default                           
-3       15.17957     host node1                         
 0   hdd  3.79489         osd.0      up  1.00000 1.00000 
 1   hdd  3.79489         osd.1      up  1.00000 1.00000 
 2   hdd  3.79489         osd.2      up  1.00000 1.00000 
 3   hdd  3.79489         osd.3      up  1.00000 1.00000 
-5       11.38480     host node2                         
 4   hdd  5.69240         osd.4      up  1.00000 1.00000 
 5   hdd  5.69240         osd.5      up  1.00000 1.00000 
```

查看当前资源池所有的落盘信息

```
[root@node1 rdb-demo]# for i in `rados -p ceph-demo ls | grep rbd_data.38d61c80322b`; do ceph osd map ceph-demo ${i}; done
```

 rbd resize命令RBD块存储扩容

```
[root@node1 rdb-demo]# rbd resize ceph-demo/rbd-demo.img --size 20G
Resizing image: 100% complete...done.
[root@node1 rdb-demo]# rbd info ceph-demo/rbd-demo.img
rbd image 'rbd-demo.img':
        size 20 GiB in 5120 objects
        order 22 (4 MiB objects)
        snapshot_count: 0
        id: 38d61c80322b
        block_name_prefix: rbd_data.38d61c80322b
        format: 2
        features: layering
        op_features: 
        flags: 
        create_timestamp: Tue Aug 18 21:38:41 2020
        access_timestamp: Tue Aug 18 21:38:41 2020
        modify_timestamp: Tue Aug 18 21:38:41 2020

#扩展文件系统
[root@node1 mnt]# xfs_growfs /dev/rbd0 
meta-data=/dev/rbd0              isize=512    agcount=16, agsize=163840 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=2621440, imaxpct=25
         =                       sunit=1024   swidth=1024 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=8 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 2621440 to 5242880
[root@node1 mnt]# df -h
Filesystem               Size  Used Avail Use% Mounted on
devtmpfs                 3.9G     0  3.9G   0% /dev
tmpfs                    3.9G     0  3.9G   0% /dev/shm
tmpfs                    3.9G  205M  3.7G   6% /run
tmpfs                    3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/mapper/centos-root   75G  2.3G   73G   4% /
/dev/sda1               1014M  194M  821M  20% /boot
tmpfs                    3.9G   52K  3.9G   1% /var/lib/ceph/osd/ceph-0
tmpfs                    3.9G   52K  3.9G   1% /var/lib/ceph/osd/ceph-1
tmpfs                    3.9G   52K  3.9G   1% /var/lib/ceph/osd/ceph-2
tmpfs                    3.9G   52K  3.9G   1% /var/lib/ceph/osd/ceph-3
tmpfs                    783M     0  783M   0% /run/user/0
/dev/rbd0                 20G  2.1G   18G  11% /mnt/rdb-demo
```



## 创建rgw(radosgw)存储网关

node1上创建rgw 端口为7480

```
[root@node1 ceph-deploy]# ceph-deploy rgw create node1
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy rgw create node1
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  rgw                           : [('node1', 'rgw.node1')]
[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
[ceph_deploy.cli][INFO  ]  subcommand                    : create
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7f6a82692cb0>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  func                          : <function rgw at 0x7f6a82acf488>
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.rgw][DEBUG ] Deploying rgw, cluster ceph hosts node1:rgw.node1
[node1][DEBUG ] connected to host: node1 
[node1][DEBUG ] detect platform information from remote host
[node1][DEBUG ] detect machine type
[ceph_deploy.rgw][INFO  ] Distro info: CentOS Linux 7.8.2003 Core
[ceph_deploy.rgw][DEBUG ] remote host will use systemd
[ceph_deploy.rgw][DEBUG ] deploying rgw bootstrap to node1
[node1][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[node1][WARNIN] rgw keyring does not exist yet, creating one
[node1][DEBUG ] create a keyring file
[node1][DEBUG ] create path recursively if it doesn't exist
[node1][INFO  ] Running command: ceph --cluster ceph --name client.bootstrap-rgw --keyring /var/lib/ceph/bootstrap-rgw/ceph.keyring auth get-or-create client.rgw.node1 osd allow rwx mon allow rw -o /var/lib/ceph/radosgw/ceph-rgw.node1/keyring
[node1][INFO  ] Running command: systemctl enable ceph-radosgw@rgw.node1
[node1][WARNIN] Created symlink from /etc/systemd/system/ceph-radosgw.target.wants/ceph-radosgw@rgw.node1.service to /usr/lib/systemd/system/ceph-radosgw@.service.
[node1][INFO  ] Running command: systemctl start ceph-radosgw@rgw.node1
[node1][INFO  ] Running command: systemctl enable ceph.target
[ceph_deploy.rgw][INFO  ] The Ceph Object Gateway (RGW) is now running on host node1 and default port 7480
[root@node1 ceph-deploy]# curl http://node1:7480
<?xml version="1.0" encoding="UTF-8"?><ListAllMyBucketsResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/"><Owner><ID>anonymous</ID><DisplayName></DisplayName></Owner><Buckets></Buckets></ListAllMyBucketsResult>
```

修改默认端口,配置文件添加rgw_frontends = "civetweb port= 7480"

```
[global]
fsid = eadc10e1-6249-4cc3-bf5a-c897572a06ae
public_network = 192.168.10.0/24
cluster_network = 192.168.10.0/24
mon_initial_members = node1, node2, node3
mon_host = 192.168.10.15,192.168.10.16,192.168.10.17
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
osd pool default size = 2
osd pool default min size =1

[mon]
mon allow pool delete = true

[client.rgw.node1]
rgw_frontends = "civetweb port= 7480"

#改完后覆盖
[root@node1 ceph-deploy]# ceph-deploy --overwrite-conf config push node1 node2 node3
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy --overwrite-conf config push node1 node2 node3
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  overwrite_conf                : True
[ceph_deploy.cli][INFO  ]  subcommand                    : push
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7ff387c21bd8>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  client                        : ['node1', 'node2', 'node3']
[ceph_deploy.cli][INFO  ]  func                          : <function config at 0x7ff3880680c8>
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.config][DEBUG ] Pushing config to node1
[node1][DEBUG ] connected to host: node1 
[node1][DEBUG ] detect platform information from remote host
[node1][DEBUG ] detect machine type
[node1][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[ceph_deploy.config][DEBUG ] Pushing config to node2
[node2][DEBUG ] connected to host: node2 
[node2][DEBUG ] detect platform information from remote host
[node2][DEBUG ] detect machine type
[node2][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[ceph_deploy.config][DEBUG ] Pushing config to node3
[node3][DEBUG ] connected to host: node3 
[node3][DEBUG ] detect platform information from remote host
[node3][DEBUG ] detect machine type
[node3][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf

#重启服务
[root@node1 ceph-deploy]# systemctl restart ceph-radosgw.target
```

创建rgw用户 创建完成后可以使用s3工具s3cmd操作ceph对象网关

```
[root@node1 ceph-deploy]# radosgw-admin user create --uid ceph-s3-user --display-name "ceph s3 user demo"
{
    "user_id": "ceph-s3-user",
    "display_name": "ceph s3 user demo",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "subusers": [],
    "keys": [
        {
            "user": "ceph-s3-user",
            "access_key": "JA6LYJWSA7NPABGLR15W",
            "secret_key": "bbhcfnhMlyUDxMLz57dL7zuUU4GmTA8Q3QgN5rjL"
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "default_storage_class": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": []
}
#查看用户
[root@node1 ceph-deploy]# radosgw-admin  user list
[
    "ceph-s3-user"
]
```

创建swift用户，需要在原有的用户基础上创建

```
[root@node1 ceph-deploy]# radosgw-admin subuser create --uid ceph-s3-user --subuser=ceph-s3-user --access=full
{
    "user_id": "ceph-s3-user",
    "display_name": "ceph s3 user demo",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "subusers": [
        {
            "id": "ceph-s3-user:ceph-s3-user",
            "permissions": "full-control"
        }
    ],
    "keys": [
        {
            "user": "ceph-s3-user",
            "access_key": "JA6LYJWSA7NPABGLR15W",
            "secret_key": "bbhcfnhMlyUDxMLz57dL7zuUU4GmTA8Q3QgN5rjL"
        }
    ],
    "swift_keys": [
        {
            "user": "ceph-s3-user:ceph-s3-user",
            "secret_key": "AqU7qUZ2gdQss58xUKDNO6yYPsrnOW6H8Rmnl9nS"
        }
    ],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "default_storage_class": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": []
}

#生成swift用户的secret_key
[root@node1 ceph-deploy]# radosgw-admin key create --subuser=ceph-s3-user:swift --key-type=swift --gen-secret
{
    "user_id": "ceph-s3-user",
    "display_name": "ceph s3 user demo",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "subusers": [
        {
            "id": "ceph-s3-user:ceph-s3-user",
            "permissions": "full-control"
        }
    ],
    "keys": [
        {
            "user": "ceph-s3-user",
            "access_key": "JA6LYJWSA7NPABGLR15W",
            "secret_key": "bbhcfnhMlyUDxMLz57dL7zuUU4GmTA8Q3QgN5rjL"
        }
    ],
    "swift_keys": [
        {
            "user": "ceph-s3-user:ceph-s3-user",
            "secret_key": "AqU7qUZ2gdQss58xUKDNO6yYPsrnOW6H8Rmnl9nS"
        },
        {
            "user": "ceph-s3-user:swift",
            "secret_key": "sAlycvMc0zCd85yvQeAIKWUH5PtyONsYon9VWZqJ"
        }
    ],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "default_storage_class": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": []
}

```



## 创建对象存储

创建mds

```
[root@node1 ceph-deploy]# ceph-deploy mds create node1
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy mds create node1
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
[ceph_deploy.cli][INFO  ]  subcommand                    : create
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7fe52ed6f488>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  func                          : <function mds at 0x7fe52f1d3398>
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  mds                           : [('node1', 'node1')]
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.mds][DEBUG ] Deploying mds, cluster ceph hosts node1:node1
[node1][DEBUG ] connected to host: node1 
[node1][DEBUG ] detect platform information from remote host
[node1][DEBUG ] detect machine type
[ceph_deploy.mds][INFO  ] Distro info: CentOS Linux 7.8.2003 Core
[ceph_deploy.mds][DEBUG ] remote host will use systemd
[ceph_deploy.mds][DEBUG ] deploying mds bootstrap to node1
[node1][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[node1][WARNIN] mds keyring does not exist yet, creating one
[node1][DEBUG ] create a keyring file
[node1][DEBUG ] create path if it doesn't exist
[node1][INFO  ] Running command: ceph --cluster ceph --name client.bootstrap-mds --keyring /var/lib/ceph/bootstrap-mds/ceph.keyring auth get-or-create mds.node1 osd allow rwx mds allow mon allow profile mds -o /var/lib/ceph/mds/ceph-node1/keyring
[node1][INFO  ] Running command: systemctl enable ceph-mds@node1
[node1][WARNIN] Created symlink from /etc/systemd/system/ceph-mds.target.wants/ceph-mds@node1.service to /usr/lib/systemd/system/ceph-mds@.service.
[node1][INFO  ] Running command: systemctl start ceph-mds@node1
[node1][INFO  ] Running command: systemctl enable ceph.target

[root@node1 ceph-deploy]# ceph-deploy mds create node2 node3
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy mds create node2 node3
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
[ceph_deploy.cli][INFO  ]  subcommand                    : create
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7f2cc4201488>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  func                          : <function mds at 0x7f2cc4665398>
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  mds                           : [('node2', 'node2'), ('node3', 'node3')]
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.mds][DEBUG ] Deploying mds, cluster ceph hosts node2:node2 node3:node3
[node2][DEBUG ] connected to host: node2 
[node2][DEBUG ] detect platform information from remote host
[node2][DEBUG ] detect machine type
[ceph_deploy.mds][INFO  ] Distro info: CentOS Linux 7.8.2003 Core
[ceph_deploy.mds][DEBUG ] remote host will use systemd
[ceph_deploy.mds][DEBUG ] deploying mds bootstrap to node2
[node2][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[node2][WARNIN] mds keyring does not exist yet, creating one
[node2][DEBUG ] create a keyring file
[node2][DEBUG ] create path if it doesn't exist
[node2][INFO  ] Running command: ceph --cluster ceph --name client.bootstrap-mds --keyring /var/lib/ceph/bootstrap-mds/ceph.keyring auth get-or-create mds.node2 osd allow rwx mds allow mon allow profile mds -o /var/lib/ceph/mds/ceph-node2/keyring
[node2][INFO  ] Running command: systemctl enable ceph-mds@node2
[node2][WARNIN] Created symlink from /etc/systemd/system/ceph-mds.target.wants/ceph-mds@node2.service to /usr/lib/systemd/system/ceph-mds@.service.
[node2][INFO  ] Running command: systemctl start ceph-mds@node2
[node2][INFO  ] Running command: systemctl enable ceph.target
[node3][DEBUG ] connected to host: node3 
[node3][DEBUG ] detect platform information from remote host
[node3][DEBUG ] detect machine type
[ceph_deploy.mds][INFO  ] Distro info: CentOS Linux 7.8.2003 Core
[ceph_deploy.mds][DEBUG ] remote host will use systemd
[ceph_deploy.mds][DEBUG ] deploying mds bootstrap to node3
[node3][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[node3][WARNIN] mds keyring does not exist yet, creating one
[node3][DEBUG ] create a keyring file
[node3][DEBUG ] create path if it doesn't exist
[node3][INFO  ] Running command: ceph --cluster ceph --name client.bootstrap-mds --keyring /var/lib/ceph/bootstrap-mds/ceph.keyring auth get-or-create mds.node3 osd allow rwx mds allow mon allow profile mds -o /var/lib/ceph/mds/ceph-node3/keyring
[node3][INFO  ] Running command: systemctl enable ceph-mds@node3
[node3][WARNIN] Created symlink from /etc/systemd/system/ceph-mds.target.wants/ceph-mds@node3.service to /usr/lib/systemd/system/ceph-mds@.service.
[node3][INFO  ] Running command: systemctl start ceph-mds@node3
[node3][INFO  ] Running command: systemctl enable ceph.target
[root@node1 ceph-deploy]# ceph -s
  cluster:
    id:     eadc10e1-6249-4cc3-bf5a-c897572a06ae
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum node1,node2,node3 (age 25h)
    mgr: node1(active, since 41h), standbys: node2, node3
    mds:  3 up:standby
    osd: 6 osds: 6 up (since 2d), 6 in (since 4d)
    rgw: 1 daemon active (node1)
 
  task status:
 
  data:
    pools:   4 pools, 128 pgs
    objects: 223 objects, 2.0 KiB
    usage:   1.1 TiB used, 25 TiB / 27 TiB avail
    pgs:     128 active+clean
 
 
```

创建一个用于存储对象的pool资源池metadata和data

```
[root@node1 ceph-deploy]# ceph osd pool create ceph-metadata 16 16
pool 'ceph-metadata' created
[root@node1 ceph-deploy]# ceph osd pool create ceph-data 16 16
pool 'ceph-data' created
[root@node1 ceph-deploy]# ceph osd lspools
3 .rgw.root
4 default.rgw.control
5 default.rgw.meta
6 default.rgw.log
8 ceph-metadata
9 ceph-data
```

创建一个对象存储的文件系统 指定metadata和data

```
# ceph fs new cephfs-demo ceph-metadata ceph-data
new fs with metadata pool 8 and data pool 9
[root@node1 ceph-deploy]# ceph fs ls
name: cephfs-demo, metadata pool: ceph-metadata, data pools: [ceph-data ]
```

ceph默认状态下之只能创建一个fs文件系统， 如果要创建多个文件系统，则需要开启多个文件系统

```
# ceph fs flag set enable_multiple true
```



ceph创建出来后，ceph的mds会进行选举

```
[root@node1 ceph-deploy]# ceph -s
  cluster:
    id:     eadc10e1-6249-4cc3-bf5a-c897572a06ae
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum node1,node2,node3 (age 25h)
    mgr: node1(active, since 41h), standbys: node2, node3
    mds: cephfs-demo:1 {0=node2=up:active} 2 up:standby
    osd: 6 osds: 6 up (since 2d), 6 in (since 4d)
    rgw: 1 daemon active (node1)
 
  task status:
    scrub status:
        mds.node2: idle
 
  data:
    pools:   6 pools, 160 pgs
    objects: 245 objects, 4.2 KiB
    usage:   1.1 TiB used, 25 TiB / 27 TiB avail
    pgs:     160 active+clean
 
```

挂载对象存储，直接通过内核挂载

```
[root@node1 ceph-deploy]#  mount -t ceph 192.168.10.15:6789:/ /mnt/cephfs/ -o name=admin
[root@node1 ceph-deploy]# df -h
Filesystem               Size  Used Avail Use% Mounted on
devtmpfs                 3.9G     0  3.9G   0% /dev
tmpfs                    3.9G     0  3.9G   0% /dev/shm
tmpfs                    3.9G   25M  3.8G   1% /run
tmpfs                    3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/mapper/centos-root   75G  2.3G   73G   4% /
/dev/sda1               1014M  194M  821M  20% /boot
tmpfs                    3.9G   24K  3.9G   1% /var/lib/ceph/osd/ceph-3
tmpfs                    3.9G   24K  3.9G   1% /var/lib/ceph/osd/ceph-0
tmpfs                    3.9G   24K  3.9G   1% /var/lib/ceph/osd/ceph-1
tmpfs                    3.9G   24K  3.9G   1% /var/lib/ceph/osd/ceph-2
tmpfs                    783M     0  783M   0% /run/user/0
192.168.10.15:6789:/      13T     0   13T   0% /mnt/cephfs
```

fuse挂载：

```
[root@localhost ~]# yum -y install ceph-common ceph-fuse
```

也可以开启多活

```
ceph fs set cephfs allow_standby_replay true
```






# 运维 

## 删除osd

​	参考：https://www.cnblogs.com/ajunyu/p/11165950.html

1. ceph-deploy节点把 OSD 踢出集群 

   ```
   ceph osd out osd.4
   ```

2. 在相应的节点，停止ceph-osd服务

   ```
   systemctl stop ceph-osd@4.service
   systemctl disable ceph-osd@4.service
   ```

3. 删除 CRUSH 图的对应 OSD 条目，它就不再接收数据了

   ```
   ceph osd crush remove osd.4
   ```

4. 删除 OSD 认证密钥

   ```
   ceph auth del osd.4
   ```

5. 通过ceph命令强行标记为down

   ```
    ceph osd down osd.4
   ```

6. 删除osd.4

   ```
   ceph osd rm osd.4
   ```

7. 最后通过命令查看最新osd

   ```
   ceph osd tree
   ID CLASS WEIGHT   TYPE NAME      STATUS REWEIGHT PRI-AFF
   -1       12.73537 root default
   -3        3.63869     host node1
    0   hdd  3.63869         osd.0      up  1.00000 1.00000
   -5        3.63869     host node2
    1   hdd  3.63869         osd.1      up  1.00000 1.00000
   -7        5.45799     host node3
    2   hdd  5.45799         osd.2      up  1.00000 1.00000
   
   ```

   



# 问题

## 问题1

创建osd的时候：Failed to execute command: /usr/sbin/ceph-volume --cluster ceph lvm create --bluestore --data /dev/sdc

解决：

​	ceph-deploy重置sdb分区： ceph-deploy disk zap node2 /dev/sdb

​	node2节点删掉分区： pvremove /dev/sdb

​	然后在创建： ceph-deploy osd create node2 --data /dev/sdb

​	如果卸载过程中还报错直接手动执行命令：dd if=/dev/zero of=/dev/sdc bs=10M count=1 然后重启



## 问题2

ceph -s 出现 clock skew detected on mon.node2 问题

```
[root@node1 ceph-deploy]# ceph -s
  cluster:
    id:     3ae84859-aea9-4b37-81c9-a36ee6b4ce1b
    health: HEALTH_WARN
    		clock skew detected on mon.node2
            1/3 mons down, quorum node1,node2

  services:
    mon: 3 daemons, quorum node1,node2 (age 0.431165s), out of quorum: node3
    mgr: node1(active, since 30m), standbys: node2, node3
    osd: 6 osds: 6 up (since 5m), 6 in (since 5m)

  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   1.1 TiB used, 25 TiB / 27 TiB avail
    pgs:

```

解决：

**1. 在admin部署节点修改配置参数：**

```
# vi ~/my-cluster/ceph.conf
```

在global字段下添加,修改ceph配置中的时间偏差阈值：

```
mon clock drift allowed = 2
mon clock drift warn backoff = 30    
```

**2. 向需要同步的mon节点推送配置文件：**

```
# ceph-deploy --overwrite-conf config push node{1..3}
```

这里是向node1 node2 node3推送，也可以后跟其它不连续节点
**3. 全部节点重启mon服务（centos7环境下）**

```
# systemctl restart ceph-mon.target
```

**4.验证：**

```
# ceph -s
```

问题解决



ceph卸载

```
# ceph-deploy purge node1 node2 node3
# ceph-deploy purgedata node1 node2 node3
# ceph-deploy forgetkeys
# rm -f ceph*
# rm -rf /var/lib/ceph/* && rm -rf /etc/ceph/* && rm -rf /var/log/ceph/* && rm -rf /var/run/ceph/*

```



ceph警告排查

1 这个警告是因为资源池没有初始化。没有被分类，需要做分类

```
[root@node1 mnt]# ceph health detail
HEALTH_WARN application not enabled on 1 pool(s)
POOL_APP_NOT_ENABLED application not enabled on 1 pool(s)
    application not enabled on pool 'ceph-demo'
    use 'ceph osd pool application enable <pool-name> <app-name>', where <app-name> is 'cephfs', 'rbd', 'rgw', or freeform for custom applications.
```

 解决, 将ceph-demo设置为rbd类型：

```
[root@node1 mnt]# ceph osd pool application enable ceph-demo rbd
enabled application 'rbd' on pool 'ceph-demo'
#查看是否设置成功
[root@node1 mnt]# ceph osd pool application get ceph-demo
{
    "rbd": {}
}
[root@node1 mnt]# ceph -s
  cluster:
    id:     eadc10e1-6249-4cc3-bf5a-c897572a06ae
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum node1,node2,node3 (age 9m)
    mgr: node1(active, since 39h), standbys: node2, node3
    osd: 6 osds: 6 up (since 47h), 6 in (since 47h)
 
  data:
    pools:   1 pools, 64 pgs
    objects: 553 objects, 2.0 GiB
    usage:   1.1 TiB used, 25 TiB / 27 TiB avail
    pgs:     64 active+clean
```



ceph清除告警

```
[root@node1 mnt]# ceph crash ls   #查看告警详情
crash info <id>
[root@node1 mnt]# ceph crash archive <id>    #根据id清除告警
```



重启后因为时间不同步造成问题:

 ceph health detail
HEALTH_WARN 183 slow ops, oldest one blocked for 94 sec, mon.node3 has slow ops
[WRN] SLOW_OPS: 183 slow ops, oldest one blocked for 94 sec, mon.node3 has slow ops

https://blog.csdn.net/qq_29974229/article/details/103729288

## 问题3

ceph 集群报错：mds0: Client failing to respond to capability release

  CephFS 客户端收到了 MDS 发出的能力（ capabilities ） ，它就像锁。有时候，比如一个客户端需要访问权， MDS 就会让别的客户端释放它们的能力，如果有客户端没响应、或者有缺陷，它就有可能没及时释放、或者根本不释放。如果某个客户端的响应时间超过了 mds_revoke_cap_timeout （默认为 60s ），这条消息就会出现。

解决办法：

1：修改集群超时配置：

查看集群现配置参数： 在每个节点上执行，由此看出 没有超时：

```
# ceph --admin-daemon /var/run/ceph/ceph-mds.node1.asok config show |grep revoke
    "mds_cap_revoke_eviction_timeout": "0.000000",
```

2:修改超时180秒:

```
# ceph --admin-daemon /var/run/ceph/ceph-mds.node1.asok config set mds_cap_revoke_eviction_timeout 180
{
    "success": "mds_cap_revoke_eviction_timeout = '180.000000' "
}
# ceph --admin-daemon /var/run/ceph/ceph-mds.node1.asok config show |grep revoke
    "mds_cap_revoke_eviction_timeout": "180.000000",
```

3:修改配置文件,global下添加mds_cap_revoke_eviction_timeout = 180

```
# pwd
/opt/ceph/ceph-deploy
# vim ceph.conf 
[global]
fsid = b188a963-daf3-4f17-90a4-9d81c665d65d
public_network = 192.168.10.0/24
cluster_network = 168.192.10.0/24
mon_initial_members = node1, node2, node3
mon_host = 192.168.10.15,192.168.10.16,192.168.10.17
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
mon clock drift allowed = 2
mon clock drift warn backoff = 30
mds_cap_revoke_eviction_timeout = 180

[mon]
mon allow pool delete = true

[osd]
osd crush update on start = false
~                                             
```

4： 推送到所有节点

```
# ceph-deploy --overwrite-conf config push node1 node2 node3 client1 client2
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy --overwrite-conf config push node1 node2 node3 client1 client2
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  overwrite_conf                : True
[ceph_deploy.cli][INFO  ]  subcommand                    : push
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7f7b55166bd8>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  client                        : ['node1', 'node2', 'node3', 'client1', 'client2']
[ceph_deploy.cli][INFO  ]  func                          : <function config at 0x7f7b55397c08>
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.config][DEBUG ] Pushing config to node1
[node1][DEBUG ] connected to host: node1 
[node1][DEBUG ] detect platform information from remote host
[node1][DEBUG ] detect machine type
[node1][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[ceph_deploy.config][DEBUG ] Pushing config to node2
[node2][DEBUG ] connected to host: node2 
[node2][DEBUG ] detect platform information from remote host
[node2][DEBUG ] detect machine type
[node2][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[ceph_deploy.config][DEBUG ] Pushing config to node3
[node3][DEBUG ] connected to host: node3 
[node3][DEBUG ] detect platform information from remote host
[node3][DEBUG ] detect machine type
[node3][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[ceph_deploy.config][DEBUG ] Pushing config to client1
[client1][DEBUG ] connected to host: client1 
[client1][DEBUG ] detect platform information from remote host
[client1][DEBUG ] detect machine type
[client1][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[ceph_deploy.config][DEBUG ] Pushing config to client2
[client2][DEBUG ] connected to host: client2 
[client2][DEBUG ] detect platform information from remote host
[client2][DEBUG ] detect machine type
[client2][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
```

最后查看集群状态：

```
# ceph -s
  cluster:
    id:     b188a963-daf3-4f17-90a4-9d81c665d65d
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum node1,node2,node3 (age 2d)
    mgr: node1(active, since 3d), standbys: node2, node3
    mds: cephfs-ssd:1 cephfs-hdd:1 {cephfs-hdd:0=node3=up:active,cephfs-ssd:0=node1=up:active} 1 up:standby
    osd: 10 osds: 10 up (since 22h), 10 in (since 2d)
 
  task status:
    scrub status:
        mds.node1: idle
        mds.node3: idle
 
  data:
    pools:   4 pools, 256 pgs
    objects: 485.87k objects, 1.0 TiB
    usage:   3.1 TiB used, 28 TiB / 31 TiB avail
    pgs:     256 active+clean
 
  io:
    client:   128 KiB/s wr, 0 op/s rd, 4 op/s wr
```

## 问题4 

实验环境：centos7 服务器

问题：之前服务器 做过ceph，之后格式化磁盘，数据盘作raid0。系统装好后，查看设备信息。

lsblk，显示部分磁盘正常，部分下面有-ceph-**等标识，用ilo多次格式化磁盘作raid0均无效果。

直接parted /dev/sdm , 做好分区/dev/sdm1，格式化/dev/sdm1 mkfs.xfs 出错，cannot open /dev/sdm: Device or resource busy



解决方法：

dmsetup ls 查看谁在占用，找到ceph-**字样（ceph-**为lsblk显示的块设备具体信息）

使用dmsetup 删除字样

dmsetup remove ceph-**



lsblk 查看设备信息，可以看到ceph-**等标识等标识消失

## mkfs.xfs -f /dev/sdm 成功通过







