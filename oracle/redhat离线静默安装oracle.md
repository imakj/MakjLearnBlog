<b>redhat离线静默安装oracle11g</b>

**1. 背景需求**

* 在最小安装的redhat下安装oracle 11gR2,没有图形界面，需要静默安装



**2. 准备工作**

1. oracle11gR2安装包

2. yum依赖包

   ![](images\1.1.png)

3. oracle rpm依赖包

   ![](images/2.3.png)

**3. 没互联网情况下依赖包环境初始化**

1. 卸载现有的yum工具

   ```
   # rpm -qa |grep yum
   yum-rhn-plugin-0.9.1-48.el6.noarch
   yum-utils-1.1.30-14.el6.noarch
   PackageKit-yum-plugin-0.5.8-21.el6.x86_64
   yum-3.2.29-40.el6.noarch
   yum-plugin-security-1.1.30-14.el6.noarch
   PackageKit-yum-0.5.8-21.el6.x86_64
   yum-metadata-parser-1.1.2-16.el6.x86_64
   
   # rpm -aq | grep yum | xargs rpm -e --nodeps
   # rpm -qa |grep yum
   ```

2. 安装准备的好的oracle安装包，yum包，rpm包，当前在/opt/oracle/installPackage/下

   ![](images/3.2.png)

3. 安装yum工具

   先安装python-urlgrabber-3.9.1-11，然后在安装yum其他包，否则会报依赖包错误

   ![](images/3.3.png)

   ```
   # cd /opt/oracle/installPackage/yum/
   
   # rpm -Uvh python-urlgrabber-3.9.1-11.el6.noarch.rpm   
   
   # rpm -ivh yum-*
   ```

   ![](images/3.3_1.png)

4. 安装rpm包，需要按照顺序装，否则会出现依赖问题

   ```
   # cd /opt/oracle/installPackage/rpm/gcc/
   # rpm -Uvh *
   # cd /opt/oracle/installPackage/rpm/gcc-/
   # rpm -Uvh *
   # cd /opt/oracle/installPackage/rpm/libaio/
   # rpm -Uvh *
   # cd /opt/oracle/installPackage/rpm/unixODBC/
   # rpm -Uvh *
   # cd /opt/oracle/installPackage/rpm/elfutils
   # rpm -Uvh *
   # cd /opt/oracle/installPackage/rpm/
   # rpm -Uvh compat-libstdc++-33-3.2.3-69.el6.x86_64.rpm
   # rpm -Uvh pdksh-5.2.14-37.el5_8.1.x86_64.rpm
   
   ```

   若安装elfutils出现相互依赖错误的情况下（如下图），可以强制安装。

   ![](images/3.4.png)

   强制安装

   ```
   # cd /opt/oracle/installPackage/rpm/elfutils
   # rpm -Uvh *  --nodeps --force
   ```

5. 检查依赖包是否完整.

   ```
   # rpm -q \
   binutils \
   compat-libstdc++-33 \
   elfutils-libelf \
   elfutils-libelf-devel \
   expat \
   gcc \
   gcc-c++ \
   glibc \
   glibc-common \
   glibc-devel \
   glibc-headers \
   libaio \
   libaio-devel \
   libgcc \
   libstdc++ \
   libstdc++-devel \
   make \
   pdksh \
   sysstat \
   unixODBC \
   unixODBC-devel | grep "not installed"
   ```



**3. 有互联网情况下依赖包环境初始化**

 1. 配置源，删除旧的源，更新新的源

    ```
    # cd /etc
    # mv yum.repos.d yum.repos.d.bak
    # mkdir yum.repos.d
    # wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
    # yum clean all
    # yum makecache
    ```

	2. 安装依赖包

    ```
    yum -y install binutils \
    compat-libstdc++-33 \
    elfutils-libelf \
    elfutils-libelf-devel \
    expat \
    gcc \
    gcc-c++ \
    glibc \
    glibc-common \
    glibc-devel \
    glibc-headers \
    libaio \
    libaio-devel \
    libgcc \
    libstdc++ \
    libstdc++-devel \
    make \
    pdksh \
    sysstat \
    unixODBC \
    unixODBC-devel
    ```

	3. 检查依赖是否安装完整

    ```
    rpm -q \
    binutils \
    compat-libstdc++-33 \
    elfutils-libelf \
    elfutils-libelf-devel \
    expat \
    gcc \
    gcc-c++ \
    glibc \
    glibc-common \
    glibc-devel \
    glibc-headers \
    libaio \
    libaio-devel \
    libgcc \
    libstdc++ \
    libstdc++-devel \
    make \
    pdksh \
    sysstat \
    unixODBC \
    unixODBC-devel | grep "not installed"
    ```

    发现pdksh未安装

    ![](images/3.5.png)

    通过yum install pdksh -y 安装缺少 package ；通过wget命令直接下载pdksh的rpm包，下载到至/tmp/ 然后安装

    ```
    # wget -O /tmp/pdksh-5.2.14-37.el5_8.1.x86_64.rpm http://vault.centos.org/5.11/os/x86_64/CentOS/pdksh-5.2.14-37.el5_8.1.x86_64.rpm
    # rpm -ivh pdksh-5.2.14-37.el5_8.1.x86_64.rpm
    ```

    

**4. 系统环境配置**

1. 修改hostname主机名

   ```
   # vim /etc/hosts
   ```

   ![](images/4.1_1.png)

   ```
   # vim /etc/sysconfig/network
   ```

   ![](images/4.1_2.png)

   修改完成后重启服务器。或者不重启的情况下，用hostname命令修改当前环境

   ```
   # hostname oracledb
   ```

2. 关闭防火墙，关闭selinux

   ```
   # vi /etc/selinux/config
   # setenforce 0
   # service iptables stop
   ```

   设置SELINUX=disabled

   ![](images/4.2_1.png)

3. 创建oracle用户

   ```
   # groupadd oinstall
   # groupadd dba
   # useradd -g oinstall -G dba oracle
   # passwd oracle
   # id oracle
   uid=501(oracle) gid=501(oinstall) groups=501(oinstall),502(dba)
   ```

4. 创建软件安装目录

   ```
   # mkdir -p /opt/oracle/product/11.2.0.1/dbname_1
   # mkdir /opt/oracle/oradata
   # mkdir /opt/oracle/flash_recovery_area
   # mkdir /opt/oracle/inventory
   # chown -R oracle:oinstall /opt/oracle
   # chmod -R 775 /opt/oracle
   ```

5. 将oracle使用者加入到sudo群组中

   ```
   # vim /etc/sudoers
   ```

   输入上面的命令后，打开sudoers文件进行编辑，找到
   root       ALL=(ALL)       ALL 
   这行，并且在底下再加入以下命令：（按esc退出insert插入模式，按下i进入编辑模式）

   ![](images/4.5.png)

   按下esc，直到退出insert模式，在最底下空白行输入“:wq!”（由于这是一份只读文档所以需要再加上!）并且按下Enter

   

6. 修改系统内核参数

   ```
   # vi /etc/sysctl.conf
   ```

   

   ```
   # Controls the maximum shared segment size, in bytes
   kernel.shmmax = 1073741824
   
   # Controls the maximum number of shared memory segments, in pages
   kernel.shmall = 2097152
   fs.aio-max-nr = 1048576
   fs.file-max = 6815744
   kernel.shmmni = 4096
   kernel.sem = 250 32000 100 128
   net.ipv4.ip_local_port_range = 9000 65500
   net.core.rmem_default = 262144
   net.core.rmem_max = 4194304
   net.core.wmem_default = 262144
   net.core.wmem_max = 1048576
   ```

   ![](images/4.6.png)

   修改完毕后，启用配置

   ```
   # sysctl -p
   ```

7. 修改用户限制文件

   ```
   # vim /etc/security/limits.conf
   ```

   ```
   oracle soft nproc 2047
   oracle hard nproc 16384
   oracle soft nofile 1024
   oracle hard nofile 65536
   oracle soft stack 10240
   oracle hard stack 32768
   ```

   ![](images/4.7.png)



8. 关联设置

   ```
   # vi /etc/pam.d/login
   ```

   行末添加以下内容:

   ```
   session required  /lib64/security/pam_limits.so
   session required   pam_limits.so
   ```

   

   ![](images/4.8.png)

9. 修改/etc/profile

   ```
   # vi /etc/profile
   ```

   末尾添加一下数据

   ```
   if [ $USER = "oracle" ]; then
    if [ $SHELL = "/bin/ksh" ]; then
     ulimit -p 16384
     ulimit -n 65536
    else
     ulimit -u 16384 -n 65536
    fi
   fi
   ```

   ![](images/4.9.png)

   ​	

   root执行以下命令，使环境变量生效

   ```
   # source /etc/profile
   ```

10. 修改oracle用户环境变量

    ```
    # vi /home/oracle/.bash_profile
    ```

    在下面添加以下接口

    ```
    # For Oracle
    export  ORACLE_BASE=/opt/oracle
    export  ORACLE_HOME=/opt/oracle/product/11.2.0.1/dbname_1
    export  ORACLE_SID=ORCL
    export  PATH=$PATH:$HOME/bin:$ORACLE_HOME/bin
    export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/usr/lib
    if [ $USER = "oracle" ]; then
     if [ $SHELL = "/bin/ksh" ]; then
      ulimit -p 16384
      ulimit -n 65536
     else
      ulimit -u 16384 -n 65536
     fi
     umask 022
    fi
    ```

    ![](images/4.10.png)

    执行下面命令，环境生效

    ```	
    # source /home/oracle/.bash_profile
    ```

    可以用`# env`命令查看环境变量是否生效

**5.正式安装**

 1. 解压oracle安装包，进入到oracle的安装目录

    ```
    # unzip linux.x64_11gR2_database_1of2.zip
    # unzip linux.x64_11gR2_database_2of2.zip
    # cd database/
    # pwd
    /opt/oracle/installPackage/database
    ```

	2. 编辑oracle数据库安装的应答文件

    解压后的oracle安装文件中的~/database/response目录下，db_install.rsp、dbca.rsp和netca.rsp三个应答文件，分别数据库安装文件、建立数据库实例和监听配置安装文件。

    编辑db_install.rsp文件，修改一下内容

    ```
    # vim response/db_install.rsp
    ```

    ```
    oracle.install.option=INSTALL_DB_SWONLY   //29 行 安装类型
    ORACLE_HOSTNAME=oracledb //37 行 主机名称
    UNIX_GROUP_NAME=oinstall //42 行 安装组
    INVENTORY_LOCATION=/opt/oracle/inventory //47 行 INVENTORY目录
    SELECTED_LANGUAGES=en,zh_CN //78 行 选择语言
    ORACLE_HOME=/opt/oracle/product/11.2.0.1/dbname_1 //83 行 oracle_home
    ORACLE_BASE=/opt/oracle //88 行 oracle_base
    oracle.install.db.InstallEdition=EE //99 行 oracle版本
    oracle.install.db.DBA_GROUP=dba //142行dba用户组
    oracle.install.db.OPER_GROUP=oinstall //147行oper用户组
    oracle.install.db.config.starterdb.type=GENERAL_PURPOSE //160行 数据库类型
    oracle.install.db.config.starterdb.globalDBName=orcl //165行globalDBName
    oracle.install.db.config.starterdb.SID=orcl //170行SID
    oracle.install.db.config.starterdb.memoryLimit=1024  //200行 自动管理内存的最小内存(M)
    oracle.install.db.config.starterdb.password.ALL=oracle //233行 设定所有数据库用户使用同一个密码
    DECLINE_SECURITY_UPDATES=true //385行 设置安全更新
    ```

	3. 开始安装

    	1. 使用oracle用户安装

        ```
        # su oracle
        ```

        

    	2. 进入刚才解压的database目录

        ```
        $ cd /opt/oracle/installPackage/database/
        ```

    	3. 执行安装文件

        ```
        [oracle@oracledb database]$ ./runInstaller -silent -responseFile /opt/oracle/installPackage/database/response/db_install.rsp -ignorePrereq
        Starting Oracle Universal Installer...
        
        Checking Temp space: must be greater than 120 MB.   Actual 25747 MB    Passed
        Checking swap space: must be greater than 150 MB.   Actual 4095 MB    Passed
        Preparing to launch Oracle Universal Installer from /tmp/OraInstall2019-07-05_09-21-54PM. Please wait ...[oracle@oracledb database]$ [WARNING] [INS-32055] The Central Inventory is located in the Oracle base.
           CAUSE: The Central Inventory is located in the Oracle base.
           ACTION: Oracle recommends placing this Central Inventory in a location outside the Oracle base directory.
        [WARNING] [INS-32055] The Central Inventory is located in the Oracle base.
           CAUSE: The Central Inventory is located in the Oracle base.
           ACTION: Oracle recommends placing this Central Inventory in a location outside the Oracle base directory.
        You can find the log of this install session at:
         /opt/oracle/inventory/logs/installActions2019-07-05_09-21-54PM.log
        /opt/oracle/inventory/logs/installActions2019-07-05_09-21-54PM.log
        ```

        接下来就是等待，时间有点长，安装过程中，如果提示[WARNING]不必理会，此时安装程序仍在后台进行，如果出现[FATAL]，则安装程序已经停止了。
        可以在以下位置找到本次安装会话的日志:
        /opt/oracle/inventory/logs/installActions2019-07-05_09-21-54PM.log
        可以使用命令查看日志：后面的地址应该以安装过程中的提示为准

        ```
        # tail -f /optoracle/oraInventory/logs/installActions2015-06-08_04-00-25PM.log
        ```

        **注意:**这里如果使用MobaXterm的情况下，MobaXterm默认X11的转发，可能会出现如下错误：

        解决方案，换个终端，用putty或者SecureCRT即可

        安装成功后，提示如下,即算安装成功。这个时候不要做任何操作

        ```
        #!/bin/sh 
         #Root scripts to run
        
        /opt/oracle/inventory/orainstRoot.sh
        /opt/oracle/product/11.2.0.1/dbname_1/root.sh
        To execute the configuration scripts:
                 1. Open a terminal window 
                 2. Log in as "root" 
                 3. Run the scripts 
                 4. Return to this window and hit "Enter" key to continue 
        
        Successfully Setup Software.
        ```

    	4. 根据上面提示，重新开启一个终端，root用户下执行上面两个脚本

        ```
        # /opt/oracle/inventory/orainstRoot.sh
        Changing permissions of /opt/oracle/inventory.
        Adding read,write permissions for group.
        Removing read,write,execute permissions for world.
        
        Changing groupname of /opt/oracle/inventory to oinstall.
        The execution of the script is complete.
        
        # /opt/oracle/product/11.2.0.1/dbname_1/root.sh
        Check /opt/oracle/product/11.2.0.1/dbname_1/install/root_oracledb_2019-07-05_21-27-51.log for the output of root script
        
        ```

        执行完成后，安装终端页面按下回车键，安装结束。

**6.配置监听**

 1. 编辑oracle安装目录下的netca.rsp文件

    ```
    $ vim /opt/oracle/installPackage/database/response/netca.rsp 
    ```

    核对一下参数配置。

    ```
    INSTALL_TYPE=""custom""	#安装的类型
    LISTENER_NUMBER=1	#监听器数量
    LISTENER_NAMES={"LISTENER"}	#监听器的名称列表
    LISTENER_PROTOCOLS={"TCP;1521"}	#监听器使用的通讯协议列表
    LISTENER_START=""LISTENER""	#监听器启动的名称
    ```

	2. 检查完成后，oracle执行命令

    ```
    $ netca /silent /responseFile /opt/oracle/installPackage/database/response/netca.rsp
    ```

    执行成功后，提示如下

    ```
    Parsing command line arguments:
        Parameter "silent" = true
        Parameter "responsefile" = /opt/oracle/installPackage/database/response/netca.rsp
    Done parsing command line arguments.
    Oracle Net Services Configuration:
    Configuring Listener:LISTENER
    Listener configuration complete.
    Oracle Net Listener Startup:
        Running Listener Control:
          /opt/oracle/product/11.2.0.1/dbname_1/bin/lsnrctl start LISTENER
        Listener Control complete.
        Listener started successfully.
    Profile configuration complete.
    Oracle Net Services configuration successful. The exit code is 0
    ```

    ![](images/6.2.png)

    **此处如果启动监听失败，查看上面的hostname是否设置正确。监听和hostname有直接关系**

	3. 执行成功后，在成功运行后，在/opt/oracle/product/11.2.0.1/dbname_1/network/admin/中生成listener.ora和sqlnet.ora

    ![](images/6.3.png)

4. 检查：用过netstat命令查看1521端口是否正常

   ```
   # netstat -tnulp | grep 1521
   ```

   ![](images/6.4.png)

5. 上面执行netca的时候已经生成了listener监听了，所以我们需要修改下面文件

   ```
   vim $ORACLE_HOME/network/admin/listener.ora
   ```

   ![](images/6.5.png)

6. 下面查看监听是否正常：

   ```
   $ lsnrctl status
   ```

   ![](images/6.6.png)

**7. 添加数据库实例**

1. 修改dbca.rsp文件

   ```
   $ vim /opt/oracle/installPackage/database/response/dbca.rsp
   ```

   修改一下内容

   ```
   RESPONSEFILE_VERSION ="11.2.0"//不能更改
   OPERATION_TYPE ="createDatabase"
   GDBNAME ="orcl"//数据库的名字
   SID ="orcl"//对应的实例名字
   TEMPLATENAME ="General_Purpose.dbc"//建库用的模板文件
   SYSPASSWORD ="oracle"//SYS管理员密码
   SYSTEMPASSWORD ="oracle"//SYSTEM管理员密码
   SYSMANPASSWORD= "oracle"
   DBSNMPPASSWORD= "oracle"
   DATAFILEDESTINATION =/opt/oracle/oradata//数据文件存放目录
   RECOVERYAREADESTINATION=/opt/oracle/flash_recovery_area//恢复数据存放目录
   CHARACTERSET ="UTF8"//字符集，重要!!!建库后一般不能更改，所以建库前要确定清楚。
   TOTALMEMORY ="2560"//2560MB，建议物理内存的80%。
   ```

2. 数据库初始化

   进入oracle安装目录的bin下，执行dbca命令

   ```
   $ dbca -silent -responseFile /opt/oracle/installPackage/database/response/dbca.rsp
   ```

   这里界面可能会出现闪动，可以等全部东西都不见了，是要输入SYS密码，但不知道为什么看不见提示，一闪而过。
   然后输入完毕按下回车，又看见SYSTEM密码一闪而过，再次输入密码回车，这时就开始建库了。

   完成后如下图：

   ![](images/7.2.png)

3. 建库完成后进行实例进程检查

   ```
   $ ps -ef | grep ora_ | grep -v grep
   ```

   ![](images/7.3_1.png)

   查看监听状态

   ```
   $ lsnrctl status
   ```

   ![](images/7.3_2.png)

4. 配置dbstart和dbshut

   修改 dbstart文件

   ```
   $ vim /opt/oracle/product/11.2.0.1/dbname_1/bin/dbstart
   ```

   将ORACLE_HOME_LISTNER=$1修改为ORACLE_HOME_LISTNER=$ORACLE_HOME

   ![](images/7.4_1.png)

   修改 dbshut文件

   ```
   $ vim /opt/oracle/product/11.2.0.1/dbname_1/bin/dbshut
   ```

   ![](images/7.4_2.png)

   修改oratab文件

   ```
   $ vim /etc/oratab
   ```

   将orcl:/data/oracle/product/11.2.0:N中最后的N改为Y，成为orcl:/data/oracle/product/11.2.0:Y

   ![](images/7.4_3.png)

   最后，输入dbshut和dbstart测试

   dbshut停止数据库

   ```
   $ dbshut 
   ```

   oracle监听停止，进程消失。查看进程

   ```
   $ ps -ef |grep ora_ |grep -v grep
   $ lsnrctl status
   ```

   ![](images/7.4_4.png)

   dbstart启动数据库

   ```
   $ dbstart
   Processing Database instance "ORCL": log file /opt/oracle/product/11.2.0.1/dbname_1/startup.log
   
   ```

   oracle监听启动，查看进程，查看监听

   ```
   $ lsnrctl status
   $ ps -ef |grep ora_ |grep -v grep
   ```

   ![](images/7.4_5.png)

**8.登录实例验证**  

```
$ sqlplus / as sysdba
```

```
SQL> select status from v$instance;
```

![](images/8.1_1.png)

如果出现如下错误，则是SID设置混乱造成的，检查SID

![](images/8.1_2.png)

![](images/8.1_3.png)

​	**解决：**
​	该问题一般是认为sid设置混乱造成，oracle安装过程中有几个地方都设置sid和数据库名称之类的，很	容易记错。

```
$ export ORACLE_SID=ORCL
$ echo $ORACLE_SID$env |grep ORACLE^C
$ env |grep ORACLE

```

**9.检查listener.ora**

```
$ cat /opt/oracle/product/11.2.0.1/dbname_1/network/admin/listener.ora
```

![](images/9.1.png)

**注意红框部分，如果listener.ora文件中没有这个值得话，需要手动添加。否则客户端会有登录不上的情况**

下面是listener.ora内容

```
SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = orcl)
      (ORACLE_HOME = /opt/oracle/product/11.2.0.1/dbname_1)
      (SID_NAME = ORCL)
    )
  )

LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = 10.229.1.12)(PORT = 1521))
    )
  )

ADR_BASE_LISTENER = /opt/oracle
```



接下来：
如果在lsnrctl start 或是lsnrctl status中有看到下面红色部分：

![](images/9.2.png)

sql> alter system register
sql> show parameter local_listener
(address=(protocol=tcp)(host=192.168.129.201)(port=1521))
而listener实际用的ip是192.168.155.100。
发现这台机器有两张网卡，ip分别为：192.168.155.100和192.168.129.201，之前有维护人员大概想将listener绑定到192.168.129.201这个ip上，但采用的方法不对。
修改local_listener参数，sql> alter system set local_listener='';
再重新注册服务，sql> alter system register;
查看注册情况，$ lsnrctl status

至此，所有oracle安装完毕



***********

**========================ORACLE 卸载======================================**

那么需要执行以下步骤：

Oracle卸载

1.停止监听服务（oracle用户登录）

[oracle@tsp-rls-dbserver ~]$ lsnrctl stop

2.停止数据库

[oracle@tsp-rls-dbserver ~]$ sqlplus / as sysdba

SQL>shutdown

3.删除oracle安装路径（root用户登录）

[root@tsp-rls-dbserver deps]# rm -rf /opt/oracle/product

[root@tsp-rls-dbserver deps]# rm -rf /opt/oracle/inventory

……安装前创建的和安装后生成的都删掉（oracle解压文件database不要误删）

4.删除系统路径文件（root用户登录）

[root@tsp-rls-dbserver deps]# rm -rf /usr/local/bin/dbhome

[root@tsp-rls-dbserver deps]# rm -rf /usr/local/bin/oraenv

[root@tsp-rls-dbserver deps]# rm -rf /usr/local/bin/coraenv

5.删除数据库实例表（root用户登录）

[root@tsp-rls-dbserver deps]# rm -rf /etc/oratab

6.删除数据库实例lock文件（root用户登录）

[root@tsp-rls-dbserver deps]# rm -rf /etc/oraInst.loc

7.删除oracle用户及用户组（root用户登录）重装oracle的话就不用删了

[root@tsp-rls-dbserver deps]# userdel -r oracle

[root@tsp-rls-dbserver deps]# groupdel oinstall

[root@tsp-rls-dbserver deps]# groupdel dba



****

**========================其他一些操作======================================**

1. 开启oracle服务：

   ```
   $dbstart
   $lsnrctl start
   $sqlplus / as sysdba
   SQL>startup
   ```

2. 关闭oracle服务：

   ```
   $dbshut
   $lsnrctl stop
   $sqlplus / as sysdba
   SQL>shutdown
   ```

   