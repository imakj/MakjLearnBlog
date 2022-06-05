# 安装NFS

1. 修改hostname 所有需要提供的修改

   client1执行：`hostnamectl set-hostname ceph_gateway`

2. 修改hosts

   集群节点添加两台客户端host三个节点机器分别修改host文件，修改完成后如下

   ```127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
   ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
   
   192.168.10.15 node1
   192.168.10.16 node2
   192.168.10.17 node3
   192.168.10.18 ceph_gateway
   ```

3. 关掉selinux

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

5. 配置阿里云yum源,包括Base、epel、ceph

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

6. ceph yum源 采用nautilus版本

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

7. 所有节点上配置nfs-ganesha源,注意ceph版本

   ```
   vim /etc/yum.repos.d/nfs-ganesha.repo
   
   [nfs-ganesha_x86_64]
   name=nfs-ganesha
   baseurl=https://mirrors.aliyun.com/ceph/nfs-ganesha/rpm-V2.8-stable/nautilus/x86_64/
   enabled=1
   priority=1
   gpgcheck=0
   ```

   

8. 完毕后更新下系统：`sudo yum update -y `

9. 在所有ganesha节点上安装ganesha。

   ```
   # yum install nfs-ganesha nfs-ganesha-ceph nfs-ganesha-rgw -y
   ```

10. 查看ganesha节点查看是否安装成功librgw2和libcephfs2软件包。

    ```
    # rpm -qa |grep libcephfs
    libcephfs2-14.2.16-0.el7.x86_64
    # rpm -qa |grep librgw
    librgw2-14.2.16-0.el7.x86_64
    ```

11. 设置服务启动以及开机启动。

    ```
    #systemctl start nfs-ganesha.service
    #systemctl enable nfs-ganesha.service
    #systemctl status nfs-ganesha.service
    ● nfs-ganesha.service - NFS-Ganesha file server
       Loaded: loaded (/usr/lib/systemd/system/nfs-ganesha.service; disabled; vendor preset: disabled)
       Active: active (running) since Sun 2021-01-17 17:10:09 CST; 3s ago
         Docs: http://github.com/nfs-ganesha/nfs-ganesha/wiki
      Process: 30518 ExecStart=/bin/bash -c ${NUMACTL} ${NUMAOPTS} /usr/bin/ganesha.nfsd ${OPTIONS} ${EPOCH} (code=exited, status=0/SUCCESS)
     Main PID: 30520 (ganesha.nfsd)
       CGroup: /system.slice/nfs-ganesha.service
               └─30520 /usr/bin/ganesha.nfsd -L /var/log/ganesha/ganesha.log -f /etc/ganesha/ganesha.conf -N NIV_EVENT
    
    Jan 17 17:10:09 cephfs_gateway systemd[1]: Starting NFS-Ganesha file server...
    Jan 17 17:10:09 cephfs_gateway bash[30518]: libust[30518/30518]: Warning: HOME environment variable not set. Disablin...c:305)
    Jan 17 17:10:09 cephfs_gateway bash[30518]: libust[30518/30519]: Error: Error opening shm /lttng-ust-wait-5 (in get_w...c:886)
    Jan 17 17:10:09 cephfs_gateway bash[30518]: libust[30518/30519]: Error: Error opening shm /lttng-ust-wait-5 (in get_w...c:886)
    Jan 17 17:10:09 cephfs_gateway systemd[1]: Started NFS-Ganesha file server.
    Hint: Some lines were ellipsized, use -l to show in full.
    ```

12. 新建ganesha_data的pool，此pool专门用来存放一些配置文件，Dashboard管理NFS需要有些配置文件存放在Rados pool中。

    ```
    # ceph osd pool create ganesha_data 16 16 hdd_replicated_rule
    pool 'ganesha_data' created
    # ceph osd pool application enable ganesha_data nfs
    enabled application 'nfs' on pool 'ganesha_data'
    ```

13. 新建空的daemon.txt文本文件
    

    # touch daemon.txt


​        
14. 导入daemon文件到ganesha_data pool中,1.存入rados的文件名必须要是conf-xxxx,原因是要Ceph Dashboard支持NFS Ganesha管理功能，需要遵循关于每个服务守护进程的RADOS对象名称的约定。对象的名称必须是conf-<daemon_id>格式，其中<daemon_id>对应于运行此服务的节点名称。<daemon_id>是一个任意字符串，应唯一地标识该守护程序实例（例如，运行守护程序的主机名）。
    2.当然我们创建这个文件现在是空的，后续通过Dashboard创建导出后，conf-<daemon_id>会有内容，每个conf-<daemon_id>都包含指向NFS-Ganesha守护程序应服务的导出的RADOS URL。这些URL的格式为：%url rados://<pool_name>[/<namespace>]/export-<id>，在创建新的导出时也同时会创建export-id的文件，这个文件内容存放实际的导出的配置内容，也就是之前没有配置Dashboard时，直接配置在ganesha配置文件中的EXPORT{}的内容。
    3.conf-<daemon_id>和export-<id>对象必须存储在同一个RADOS池/命名空间，当然如果是通过Dashboard配置的这两个文件肯定是在同个pool，如果手工创建的话就需要注意这点。

    ```
    # rados -p ganesha_data put conf-ceph_node1 daemon.txt
    # rados -p ganesha_data put conf-ceph_node2 daemon.txt
    # rados -p ganesha_data put conf-ceph_node3 daemon.txt
    #也可以单独加一台对外服务网关
    # rados -p ganesha_data put conf-ceph_gateway daemon.txt 
    ```

15. 查看gaensha pool中存

    ```
    # rados -p ganesha_data ls
    conf-ceph_gateway
    conf-ceph_node1
    conf-ceph_node2
    conf-ceph_node3
    ```

16. 删除conf

    ```
    # rados rm -p ganesha_data conf-ceph_gateway
    ```

17. 如果有单独的主要服务器用来对外共享nfs，则在此服务器安装rgw网关

    1. 推送配置文件到节点中

    ```
    # yum -y install ceph-radosgw
    ```

    2. 推送配置文件到cephfs_gateway

       ```
       # ceph-deploy --overwrite-conf config push ceph_gateway
       ```

    3. 添加创建rgw

       ```
       ceph-deploy rgw create ceph_gateway
       ```

       

18. 查看当前Ceph节点的rgw认证信息，如，下图输出client.rgw.node3.localdomain，为后续每一台虚拟机ganesha配置文件中RGW部分name的值。

    ```
    # ceph auth list
    installed auth entries:
    
    mds.node1
    	key: AQBP2QNgusqMMhAAmuZ+AzYoPGU8pAc506BtGg==
    	caps: [mds] allow
    	caps: [mon] allow profile mds
    	caps: [osd] allow rwx
    mds.node2
    	key: AQBR2QNgSBniExAAhiQb3e7QOtMY4hTy0zn1FQ==
    	caps: [mds] allow
    	caps: [mon] allow profile mds
    	caps: [osd] allow rwx
    mds.node3
    	key: AQBS2QNgOkvYLRAA5FANQQlVK1ota8xhG/ep7Q==
    	caps: [mds] allow
    	caps: [mon] allow profile mds
    	caps: [osd] allow rwx
    osd.0
    	key: AQBlFwNgj65tMhAA0iY3XWG97qBqJQo1hBrdfQ==
    	caps: [mgr] allow profile osd
    	caps: [mon] allow profile osd
    	caps: [osd] allow *
    osd.1
    	key: AQCGFwNgZb87DRAAjhov+W1LxqHQhpxkq0F3Rg==
    	caps: [mgr] allow profile osd
    	caps: [mon] allow profile osd
    	caps: [osd] allow *
    osd.10
    	key: AQCNGANgE98+BhAA7YC0FXJQewGuy3HEQH4lbQ==
    	caps: [mgr] allow profile osd
    	caps: [mon] allow profile osd
    	caps: [osd] allow *
    osd.11
    	key: AQCYGANginqxNhAABpP8kIQAIY1y5EiV51sSzw==
    	caps: [mgr] allow profile osd
    	caps: [mon] allow profile osd
    	caps: [osd] allow *
    osd.12
    	key: AQClGANgMDLMAhAANFeG4pEe7XZrrs90XNOnRw==
    	caps: [mgr] allow profile osd
    	caps: [mon] allow profile osd
    	caps: [osd] allow *
    osd.13
    	key: AQCwGANg7AEPOBAAfnoiQvrJ+pDtpaBBcP2A4g==
    	caps: [mgr] allow profile osd
    	caps: [mon] allow profile osd
    	caps: [osd] allow *
    osd.2
    	key: AQCtFwNgDRLBFxAA3O0ghMQNltGw3u1JLd0SCA==
    	caps: [mgr] allow profile osd
    	caps: [mon] allow profile osd
    	caps: [osd] allow *
    osd.3
    	key: AQD2FwNgR0w0DBAAJ8i72ZLgvLduM9s+QskxBg==
    	caps: [mgr] allow profile osd
    	caps: [mon] allow profile osd
    	caps: [osd] allow *
    osd.4
    	key: AQAGGANgnhiOLBAAH+FPY767mF4AWwTxuNzV9g==
    	caps: [mgr] allow profile osd
    	caps: [mon] allow profile osd
    	caps: [osd] allow *
    osd.5
    	key: AQAXGANgPJFbHhAATexjFmc01e30cYLrncsYIg==
    	caps: [mgr] allow profile osd
    	caps: [mon] allow profile osd
    	caps: [osd] allow *
    osd.6
    	key: AQBbGANg9WeeNhAAUI915nu2yqUtJojBP+7Kdw==
    	caps: [mgr] allow profile osd
    	caps: [mon] allow profile osd
    	caps: [osd] allow *
    osd.7
    	key: AQBoGANgpnqRHRAAhdECgvpe92eSGrNo+DdTqA==
    	caps: [mgr] allow profile osd
    	caps: [mon] allow profile osd
    	caps: [osd] allow *
    osd.8
    	key: AQB1GANgo+3iBBAATHWB9rxmAEzsg0Qakeknrw==
    	caps: [mgr] allow profile osd
    	caps: [mon] allow profile osd
    	caps: [osd] allow *
    osd.9
    	key: AQCAGANgGJ6LHxAA9f5gsNybjAoVKE7DKiK6WQ==
    	caps: [mgr] allow profile osd
    	caps: [mon] allow profile osd
    	caps: [osd] allow *
    client.admin
    	key: AQDOFANgo6jDHxAAc/4zsWsrrrJB/YpGIF8JaQ==
    	caps: [mds] allow *
    	caps: [mgr] allow *
    	caps: [mon] allow *
    	caps: [osd] allow *
    client.bootstrap-mds
    	key: AQDOFANgntDDHxAAigbleA4OKTohDW6k5xKxRw==
    	caps: [mon] allow profile bootstrap-mds
    client.bootstrap-mgr
    	key: AQDOFANgPuvDHxAAOfbWFDIahM7LO70z0OhIog==
    	caps: [mon] allow profile bootstrap-mgr
    client.bootstrap-osd
    	key: AQDOFANglQTEHxAAhRJRVpRP3eM55YCTmX36Zw==
    	caps: [mon] allow profile bootstrap-osd
    client.bootstrap-rbd
    	key: AQDOFANg3h3EHxAAlCbZIV6rbwbO9NBNVwVnJg==
    	caps: [mon] allow profile bootstrap-rbd
    client.bootstrap-rbd-mirror
    	key: AQDOFANgJjrEHxAAOk5/4hZ/jGgw7voT9bQ/rA==
    	caps: [mon] allow profile bootstrap-rbd-mirror
    client.bootstrap-rgw
    	key: AQDOFANgNVXEHxAA3Wxz+nLmjTMfoIEzKLG2AQ==
    	caps: [mon] allow profile bootstrap-rgw
    client.rgw.node1
    	key: AQBp7gNg0OWuCBAAB8jJQiKxRyjf4CaO6KSRxA==
    	caps: [mon] allow rw
    	caps: [osd] allow rwx
    client.rgw.node2
    	key: AQBq7gNgWzaPGRAA3jFsSW++pX8lk7+p7SFWcQ==
    	caps: [mon] allow rw
    	caps: [osd] allow rwx
    client.rgw.node3
    	key: AQBs7gNgCk2BKxAALU5mWDvWbYAPTE7Pppvg7Q==
    	caps: [mon] allow rw
    	caps: [osd] allow rwx
    mgr.node1
    	key: AQAGFQNgz0mfJxAA3tpLKe35j/8Mi7EKCDAAuw==
    	caps: [mds] allow *
    	caps: [mon] allow profile mgr
    	caps: [osd] allow *
    mgr.node2
    	key: AQAHFQNgsvzXMhAACJyEch+2HskM9J7wtIUDpg==
    	caps: [mds] allow *
    	caps: [mon] allow profile mgr
    	caps: [osd] allow *
    mgr.node3
    	key: AQAJFQNg4UqrAhAAGfZh5suLHrxZwTJql+Trgg==
    	caps: [mds] allow *
    	caps: [mon] allow profile mgr
    	caps: [osd] allow *
    ```

    ```
    主要看这一部分 记住这几个名字 下面配置RGW的时候就是这几个名字
    client.rgw.node1
    	key: AQBp7gNg0OWuCBAAB8jJQiKxRyjf4CaO6KSRxA==
    	caps: [mon] allow rw
    	caps: [osd] allow rwx
    client.rgw.node2
    	key: AQBq7gNgWzaPGRAA3jFsSW++pX8lk7+p7SFWcQ==
    	caps: [mon] allow rw
    	caps: [osd] allow rwx
    client.rgw.node3
    	key: AQBs7gNgCk2BKxAALU5mWDvWbYAPTE7Pppvg7Q==
    	caps: [mon] allow rw
    	caps: [osd] allow rwx
    ```

19. 编辑每一台节点ganesha配置文件，并根据当前所在节点输入以下内容，如下图显示的是存储节点3的配置信息，请根据情况替换成其它存储节点配置信息。

    此配置文件包括3部分内容：
    1.RADOS_URLS部分
    ceph_confi主要是配置ceph的配置文件路径
    Userid主要是配置访问rados的用户名admin
    watch_url主要是配置，当通过Dashboard生成新的配置文件存入在rados中，ganesha进程可以读取新的内容并通过SIGHUP重新加载配置文件。
    2.%url部分
    NFS Ganesha支持从RADOS对象读取配置。该 %url指令允许指定一个RADOS URL，该URL标识RADOS对象的位置。
    3.RGW部分
    cluster 设置一个Ceph集群名称（必须与正在导出的集群匹配，默认使用ceph-deploy部署的ceph群集名称为ceph）
    name 设置RGW实例名称（必须与正在导出的集群中的rgw节点的认证信息匹配，使用ceph auth list可以查看以client.rgw.开头的信息）
    ceph_conf 给出了要使用的非默认ceph.conf文件的路径,默认路径可以省略此行。

    修改后的配置文件

    node1

    ```
    # mv /etc/ganesha/ganesha.conf /etc/ganesha/ganesha.conf.20210117
    # vim /etc/ganesha/ganesha.conf
    RADOS_URLS {
        ceph_conf = "/etc/ceph/ceph.conf";
        Userid = "admin";
        watch_url = "rados://ganesha_data/conf-node1";
    }
    %url rados://ganesha_data/conf-node1
    RGW {
        ceph_conf = "/etc/ceph/ceph.conf";
        name = "client.rgw.node1";
        cluster = "ceph";
    }
    
    ```

    node2

    ```
    # mv /etc/ganesha/ganesha.conf /etc/ganesha/ganesha.conf.20210117
    # vim /etc/ganesha/ganesha.conf
    RADOS_URLS {
        ceph_conf = "/etc/ceph/ceph.conf";
        Userid = "admin";
        watch_url = "rados://ganesha_data/conf-node2";
    }
    %url rados://ganesha_data/conf-node2
    RGW {
        ceph_conf = "/etc/ceph/ceph.conf";
        name = "client.rgw.node2";
        cluster = "ceph";
    }
    ```

    node3

    ```
    # mv /etc/ganesha/ganesha.conf /etc/ganesha/ganesha.conf.20210117
    # vim /etc/ganesha/ganesha.conf
    RADOS_URLS {
        ceph_conf = "/etc/ceph/ceph.conf";
        Userid = "admin";
        watch_url = "rados://ganesha_data/conf-node3";
    }
    %url rados://ganesha_data/conf-node3
    RGW {
            ceph_conf = "/etc/ceph/ceph.conf";
            name = "client.rgw.node3";
            cluster = "ceph";
    }
    ```

20. 要在Ceph仪表板中启用NFS-Ganesha管理，我们只需要告诉仪表板要导出哪个pool,比如以下是导出cephfs_data pool。

    ```
    ceph dashboard set-ganesha-clusters-rados-pool-namespace ganesha_data
    ```

21. 所有ganesha-nfs节点重启ganesha-nfs

    ```
    # systemctl restart nfs-ganesha.service
    # systemctl status nfs-ganesha.service
    ● nfs-ganesha.service - NFS-Ganesha file server
       Loaded: loaded (/usr/lib/systemd/system/nfs-ganesha.service; enabled; vendor preset: disabled)
       Active: active (running) since Sun 2021-01-17 18:10:32 CST; 1s ago
         Docs: http://github.com/nfs-ganesha/nfs-ganesha/wiki
      Process: 5669 ExecStop=/bin/dbus-send --system --dest=org.ganesha.nfsd --type=method_call /org/ganesha/nfsd/admin org.ganesha.nfsd.admin.shutdown (code=exited, status=0/SUCCESS)
      Process: 5689 ExecStartPost=/bin/bash -c /usr/bin/sleep 2 && /bin/dbus-send --system   --dest=org.ganesha.nfsd --type=method_call /org/ganesha/nfsd/admin  org.ganesha.nfsd.admin.init_fds_limit (code=exited, status=0/SUCCESS)
      Process: 5687 ExecStartPost=/bin/bash -c prlimit --pid $MAINPID --nofile=$NOFILE:$NOFILE (code=exited, status=0/SUCCESS)
      Process: 5684 ExecStart=/bin/bash -c ${NUMACTL} ${NUMAOPTS} /usr/bin/ganesha.nfsd ${OPTIONS} ${EPOCH} (code=exited, status=0/SUCCESS)
     Main PID: 5686 (ganesha.nfsd)
       CGroup: /system.slice/nfs-ganesha.service
               └─5686 /usr/bin/ganesha.nfsd -L /var/log/ganesha/ganesha.log -f /etc/ganesha/ganesha.conf -N NIV_EVENT
    
    Jan 17 18:10:30 node3 systemd[1]: Starting NFS-Ganesha file server...
    Jan 17 18:10:30 node3 bash[5684]: libust[5684/5684]: Warning: HOME environment variable not set. Disabling LTTng-UST....c:305)
    Jan 17 18:10:32 node3 systemd[1]: Started NFS-Ganesha file server.
    Hint: Some lines were ellipsized, use -l to show in full.
    
    ```

22. 至此 页面上可以看到nfs管理界面

# 安装ISCSI

之前Ceph存储集群的块存储不支持iscsi，从Ceph Luminous版本开始支持iSCSI。

Ceph中实现iscsi 方式有两种，一种是通过Linux target framework(tgt)实现，一种是通过Linux-IO Target（lio）实现，本文是使用的方式是LIO，LIO现在也是官方推荐的方式。

LIO的实现方式主要是利用TCMU与Ceph的librbd库进行交互，并将RBD images映射给iSCSI客户端，所以需要有TCMU软件包安装在系统中。

启用iscsi gateway需要满足以下条件：

1. 正在运行的Ceph Luminous（12.2.x）集群或更高版本
2. CentOS 7.5（或更高版本）；Linux内核v4.16（或更高版本）
3. 该ceph-iscsi软件包安装在所有iSCSI网关节点上
4. 如果Ceph iSCSI网关未位于OSD节点上，则将位于中的Ceph配置文件/etc/ceph/从存储集群中正在运行的Ceph节点复制到iSCSI Gateway节点。Ceph配置文件必须存在于iSCSI网关节点下的/etc/ceph/。



1. 在所有iscsi gw节点上配置ceph-iscsi yum源。

   ```
   # vim /etc/yum.repos.d/ceph-iscsi.repo
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

   注意：tcmul软件包没有包括在常用的第三方的yum源中，只有redhat官方的源，但没有订阅的话不能使用，所以有个人用户搞了tcmu-runner 源，但个人源不能保证一直有效。

2. 所有节点安装iSCSI

   ```
   # yum install ceph-iscsi -y
   ```

3. 重启tcmu-runner(此步也可省略，因为启动rbd-target-api会自动启动tcmu-runner服务)

   ```
   #systemctl start tcmu-runner.service
   #systemctl enable tcmu-runner.service 
   # systemctl status tcmu-runner.service 
   Created symlink from /etc/systemd/system/multi-user.target.wants/tcmu-runner.service to /usr/lib/systemd/system/tcmu-runner.service.
   You have new mail in /var/spool/mail/root
   [root@ceph_node1 ceph-deploy]# systemctl enable tcmu-runner.service ^C
   You have new mail in /var/spool/mail/root
   [root@ceph_node1 ceph-deploy]# ^C
   [root@ceph_node1 ceph-deploy]# systemctl status tcmu-runner.service
   ● tcmu-runner.service - LIO Userspace-passthrough daemon
      Loaded: loaded (/usr/lib/systemd/system/tcmu-runner.service; enabled; vendor preset: disabled)
      Active: active (running) since Fri 2021-01-29 21:44:19 CST; 1min 54s ago
    Main PID: 26100 (tcmu-runner)
      CGroup: /system.slice/tcmu-runner.service
              └─26100 /usr/bin/tcmu-runner
   
   Jan 29 21:44:19 ceph_node1 systemd[1]: Starting LIO Userspace-passthrough daemon...
   Jan 29 21:44:19 ceph_node1 tcmu-runner[26100]: Inotify is watching "/etc/tcmu/tcmu.conf", wd: 1, mask: IN_ALL_EVENTS
   Jan 29 21:44:19 ceph_node1 tcmu-runner[26100]: 2021-01-29 21:44:19.066 26100 [INFO] load_our_module:534: Inserted module '..._user'
   Jan 29 21:44:19 ceph_node1 tcmu-runner[26100]: load_our_module:534: Inserted module 'target_core_user'
   Jan 29 21:44:19 ceph_node1 systemd[1]: Started LIO Userspace-passthrough daemon.
   Hint: Some lines were ellipsized, use -l to show in full.
   
   ```

4. 创建iscsi pool。

   ```
   # ceph osd pool create iscsi_pool 256 256 nvme_replicated_rule
   pool 'iscsi_pool' created
   # ceph osd pool application enable iscsi_pool rbd
   enabled application 'rbd' on pool 'iscsi_pool'
   ```

5. 配置每一个iscsi gw节点上iscsi gateway配置文件，cluster_client_name为client.admin用户，trusted_ip_list 为所有iscsi gateway IP地址，api端口为5000，user为admin。Trusted_ip_list是每个iscsi网关上IP地址的列表，这些IP地址将用于管理操作，例如目标创建，lun导出等。

   ```
   # vim /etc/ceph/iscsi-gateway.cfg
   [config]
   cluster_client_name = client.admin
   pool = iscsi_pool
   trusted_ip_list = 192.168.10.15,192.168.10.16,192.168.10.17,192.168.10.18
   minimum_gateways = 1
   fqdn_enabled=true
   #Additional API configuration options are as follows, defaults shown.
   api_port = 5000
   api_user = admin
   api_password = admin
   api_secure = false
   #Log level
   logger_level = WARNING   
   ```

6. 重启rbd-target服务并设置开机启动。

   ```
   #systemctl restart rbd-target-api.service
   #systemctl status rbd-target-api.service
   ```

7. 查看所有节点gw服务状态

   ```
   # gwcli info
   Warning: Could not load preferences file /root/.gwcli/prefs.bin.
   HTTP mode          : http
   Rest API port      : 5000
   Local endpoint     : http://localhost:5000/api
   Local Ceph Cluster : ceph
   2ndary API IP's    : 192.168.10.15,192.168.10.16,192.168.10.17,192.168.10.18
   ```

   注意：iscsi-gateway命令行工具gwcli用于创建/配置iscsi-target与rbd image；其余较低级别命令行工具，如targetcli或rbd等，可用于查询配置，但不能用于修改gwcli所做的配置。

   可以查看当前iscsi gateway配置，当然gwcli只是命令行工具，当我们配置了Dashboard集成iscsi后，就不一定要用这个命令行工具配置了，可以使用图形界面配置也是一样的。

8. Dashboard启用用iscsi。

   要禁用API SSL验证。

   ```
   # ceph dashboard set-iscsi-api-ssl-verification false
   Option ISCSI_API_SSL_VERIFICATION updated
   ```

   使用以下命令定义可用的iSCSI网关，添加iscsi-gateway之前，需要在每一个网关上启动rbd-api服务。

   ```
   #ceph dashboard iscsi-gateway-add http://admin:admin@192.168.10.15:5000
   Success
   #ceph dashboard iscsi-gateway-add http://admin:admin@192.168.10.16:5000
   Success
   #ceph dashboard iscsi-gateway-add http://admin:admin@192.168.10.17:5000
   Success
   #ceph dashboard iscsi-gateway-add http://admin:admin@192.168.10.18:5000
   Success
   ```

   在本文的开始，说明了各节点的hosts配置文件中一定要是FQDN，就是因为添加每一台节点是默认都解析成了localhost.localdomain,所以会导致只能添加成功一个iscsi gateway节点（原因是默认只有127.0.0.1配置FQDN）。

   添加iscsi gw网关的用户名admin,密码admin是根据iscsi gw配置文件中定义的api_user以及api_password。

9. 查看配置。

   ```
   # ceph dashboard iscsi-gateway-list
   {"gateways": {"ceph_node3": {"service_url": "http://admin:admin@192.168.10.17:5000"}, "ceph_node2": {"service_url": "http://admin:admin@192.168.10.16:5000"}, "ceph_node1": {"service_url": "http://admin:admin@192.168.10.15:5000"}, "ceph_gateway": {"service_url": "http://admin:admin@192.168.10.18:5000"}}}
   ```

   

