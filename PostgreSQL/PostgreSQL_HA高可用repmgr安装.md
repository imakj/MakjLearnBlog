# 环境

* centos7  4台 
  * 192.168.10.68 repmgr+master
  * 192.168.10.69 repmgr+standby
  * 192.168.10.70 repmgr+standby
  * 192.168.10.71 repmgr+witness
* postgresql源码包：postgresql-12.6.tar.gz
* repmgr:repmgr-5.0.tar.gz

# 系统变量准备

1. 所有机器修改主机名

   ```
   # 192.168.10.68修改
   # hostnamectl set-hostname postgreSQL_04_68
   # 192.168.10.69修改
   # hostnamectl set-hostname postgreSQL_05_69
   # 192.168.10.70修改
   # hostnamectl set-hostname postgreSQL_06_70
   # 192.168.10.7修改
   # hostnamectl set-hostname postgreSQL_07_71
   ```

2. 所有机器修改host

   ```
   vim /etc/hosts
   192.168.10.68 postgreSQL_04_68
   192.168.10.69 postgreSQL_05_69
   192.168.10.70 postgreSQL_06_70
   192.168.10.71 postgreSQL_07_71
   ```

3. 所有节点全部进行密码认证

   ```
   # su - pgsql
   $ ssh-keygen
   Generating public/private rsa key pair.
   Enter file in which to save the key (/home/pgsql/.ssh/id_rsa): 
   Enter passphrase (empty for no passphrase): 
   Enter same passphrase again: 
   Your identification has been saved in /home/pgsql/.ssh/id_rsa.
   Your public key has been saved in /home/pgsql/.ssh/id_rsa.pub.
   The key fingerprint is:
   SHA256:f/0t8le8xTltqWR68EJy/u31fsc7rmNDGePJyGQiHuQ pgsql@postgresql_04_68
   The key's randomart image is:
   +---[RSA 2048]----+
   |                 |
   |       .         |
   |      o          |
   |       E . o o   |
   |      . S = + =o+|
   |       . o * X +B|
   |          * O oo*|
   |           =.Bo+O|
   |            ==BOX|
   +----[SHA256]-----+
   
   ```

4. 拷贝公钥到四个节点，包括自己

   ```
   $ ssh-copy-id -i /home/pgsql/.ssh/id_rsa.pub postgreSQL_04_68
   $ ssh-copy-id -i /home/pgsql/.ssh/id_rsa.pub postgreSQL_05_69
   $ ssh-copy-id -i /home/pgsql/.ssh/id_rsa.pub postgreSQL_06_70
   $ ssh-copy-id -i /home/pgsql/.ssh/id_rsa.pub postgreSQL_07_71
   ```

   

5. 所有节点执行安装依赖

   ```
   # yum install -y cmake make gcc zlib gcc-c++ perl readline readline-devel zlib zlib-devel perl python36 tcl openssl ncurses-devel openldap pam
   # yum -y groupinstall "Development Tools"
   # yum -y install yum-utils openjade docbook-dtds docbook-style-dsssl docbook-style-xsl
   ```

# 安装

1. 创建用户 

   ```
   # groupadd -g 60000 pgsql
   # useradd -u 60000 -g pgsql pgsql
   # echo "pgsql" |passwd --stdin pgsql
   ```

2. 创建安装目录，

   ```
   # mkdir -p /data/postgresql/{pgdata,archive,scripts,backup,soft,logs}
   # chown -R pgsql:pgsql /data/postgresql
   # chmod -R 700 /data/postgresql
   # chmod 700 /usr/local/src/postgresql-12.6.tar.gz
   ```

3. 解压 安装

   ```
   # cd /usr/local/src/
   # tar zxvf postgresql-12.6.tar.gz
   # cd postgresql-12.6
   # mkdir /usr/local/postgresql
   # --without-readline禁止生成历史记录 防止别人再次登录看到
   # ./configure --prefix=/usr/local/postgresql --without-readline
   # make -j
   # make install
   ```

4. 修改环境变量,最后一行追加

   ```
   # vim /etc/profile
   export PGPORT=5432
   export PGDATA=/data/postgresql/pgdata
   export PGHOME=/usr/local/postgresql
   export LD_LIBRARY_PATH=$PGHOME/lib:/lib64:/usr/lib64:/usr/local/lib64:/lib:/usr/lib:/usr/local/lib:$LD_LIBRARY_PATH
   export PATH=$PGHOME/bin:$PATH:.
   export DATE=`date +"%Y%m%d%H%M"`
   export MANPATH=$PGHOME/share/man:$MANPATH
   export PGHOST=$PGDATA
   export PGUSER=postgres
   export PGDATABASE=postgres
   
   # 保存变更使其生效
   # source /etc/profile
   ```

5. 初始化数据库

   ```
   # su - pgsql
   $ /usr/local/postgresql/bin/initdb -D /data/postgresql/pgdata -E UTF8 --locale=en_US.utf8 -U postgres
   The files belonging to this database system will be owned by user "pgsql".
   This user must also own the server process.
   
   The database cluster will be initialized with locale "en_US.utf8".
   The default text search configuration will be set to "english".
   
   Data page checksums are disabled.
   
   fixing permissions on existing directory /data/postgresql/pgdata ... ok
   creating subdirectories ... ok
   selecting dynamic shared memory implementation ... posix
   selecting default max_connections ... 100
   selecting default shared_buffers ... 128MB
   selecting default time zone ... Asia/Shanghai
   creating configuration files ... ok
   running bootstrap script ... ok
   performing post-bootstrap initialization ... ok
   syncing data to disk ... ok
   
   initdb: warning: enabling "trust" authentication for local connections
   You can change this by editing pg_hba.conf or using the option -A, or
   --auth-local and --auth-host, the next time you run initdb.
   
   Success. You can now start the database server using:
   
       /usr/local/postgresql/bin/pg_ctl -D /data/postgresql/pgdata -l logfile start
   ```

6. 修改配置文件

   ```
   $ cd /data/postgresql/pgdata/
   $ vim postgresql.conf
   listen_addresses = '*'  #监听所有ip
   port = 5432
   max_connections = 1000  #并发数
   logging_collector = on  #打开日志收集开关
   log_directory = '/data/postgresql/logs/'   #日志目录 
   log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'  #日志显示格式
   log_truncate_on_rotation = on  #打开日志循环
   shared_buffers = 2048MB #共享池 物理内存的80%
   max_wal_size = 1GB   #日志最大 
   min_wal_size = 80MB  #日志最小
   #下面是主从复制的
   archive_mode = on  #启用归档模式
   archive_command = 'test ! -f /data/postgresql/archive%f && cp %p /data/postgresql/archive/%f'  #归档路径
   wal_level = replica #日志复制模式
   max_wal_senders = 10
   wal_keep_segments = 0
   wal_sender_timeout = 60s
   ```

7. 修改访问权限配置文件（也就是防火墙）

   ```
   $ cd /data/postgresql/pgdata/
   $ vim pg_hba.conf  
   #最后一行添加，也就是所有人都可以用密码进来 replication必须存在，
   host    all     all     0.0.0.0/0       md5
   ```

8. 启动 

   ```
   $ /usr/local/postgresql/bin/pg_ctl -D /data/postgresql/pgdata start
   waiting for server to start....2021-02-19 16:58:00.890 CST [11010] LOG:  starting PostgreSQL 12.6 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 4.8.5 20150623 (Red Hat 4.8.5-44), 64-bit
   2021-02-19 16:58:00.890 CST [11010] LOG:  listening on IPv4 address "0.0.0.0", port 5432
   2021-02-19 16:58:00.890 CST [11010] LOG:  listening on IPv6 address "::", port 5432
   2021-02-19 16:58:00.902 CST [11010] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5432"
   2021-02-19 16:58:00.990 CST [11010] LOG:  redirecting log output to logging collector process
   2021-02-19 16:58:00.990 CST [11010] HINT:  Future log output will appear in directory "/data/postgresql/logs".
    done
   server started
   ```

9. 设置密码（通过127.0.0.1登录设置密码，前面防火墙已经设置通过127.0.0.1登录不需要密码）

   ```\
   $ psql -h 127.0.0.1 -p 5432 -U postgres
   # 设置postgres的密码为postgres
   postgres=# \password postgres
   # 退出
   postgres-# \q
   # 通过ip登录就可以成功
   $ psql -h 192.168.10.65 -p 5432
   ```

10. 在68 69 70三个节点上安装repmgr

    ```
    # cd /usr/local/src/
    # tar zxvf repmgr-5.0.tar.gz 
    # cd repmgr-5.0.0/
    # ./configure
    # make -j
    # 安装 安装的时候是作为pg的插件来安装的，所以注意上面的环境变量的配置
    # make install  
    Building against PostgreSQL 12
    /usr/bin/mkdir -p '/usr/local/postgresql/lib'
    /usr/bin/mkdir -p '/usr/local/postgresql/share/extension'
    /usr/bin/mkdir -p '/usr/local/postgresql/share/extension'
    /usr/bin/mkdir -p '/usr/local/postgresql/bin'
    /usr/bin/install -c -m 755  repmgr.so '/usr/local/postgresql/lib/repmgr.so'
    /usr/bin/install -c -m 644 .//repmgr.control '/usr/local/postgresql/share/extension/'
    /usr/bin/install -c -m 644 .//repmgr--unpackaged--4.0.sql .//repmgr--4.0.sql .//repmgr--4.0--4.1.sql .//repmgr--4.1.sql .//repmgr--4.1--4.2.sql .//repmgr--4.2.sql .//repmgr--4.2--4.3.sql .//repmgr--4.3.sql .//repmgr--4.3--4.4.sql .//repmgr--4.4.sql .//repmgr--4.4--5.0.sql .//repmgr--5.0.sql  '/usr/local/postgresql/share/extension/'
    /usr/bin/install -c -m 755 repmgr repmgrd '/usr/local/postgresql/bin/'
    
    # repmgr --help
    ```

11. 开始配置，主库创建相关系统用户

    ```
    # su - pgsql
    # 创建repmgr用户
    $ createuser -s repmgr -h 127.0.0.1
    # 创建repmgr数据库
    $ createdb repmgr -O repmgr -h 127.0.0.1
    # 修改repmgr 密码
    $ psql -h 127.0.0.1 -c "alter user repmgr with password 'repmgr'"
    ALTER ROLE
    # 设置repmgr数据的用户为repmgr
    $ psql -h 127.0.0.1 -c "alter user repmgr set search_path to repmgr, \"\$user\",public";
    ALTER ROLE
    ```

12. 修改hba防火墙文件 添加一下权限

    ```
    $ cd /data/postgresql/pgdata/
    $ vim pg_hba.conf
    local repmgr repmgr md5
    host repmgr repmgr 127.0.0.1/32 md5
    # host repmgr repmgr 192.168.10.0/24 md5 #这里可以指定网段
    host repmgr repmgr 0.0.0.0/0 md5
    local replication repmgr md5
    host replication repmgr 127.0.0.1/32 md5
    host replication repmgr 0.0.0.0/0 md5
    # host replication repmgr 192.168.1.0/24 md5
    
    
    #查看文件最后几行配置
    $ cat pg_hba.conf
    # "local" is for Unix domain socket connections only
    local   all             all                                     trust
    # IPv4 local connections:
    host    all             all             127.0.0.1/32            trust
    # IPv6 local connections:
    host    all             all             ::1/128                 trust
    # Allow replication connections from localhost, by a user with the
    # replication privilege.
    local   replication     all                                     trust
    host    replication     all             127.0.0.1/32            trust
    host    replication     all             ::1/128                 trust
    host    all     all     0.0.0.0/0       md5
    # host replication repuser 0.0.0.0/0 md5
    local repmgr repmgr md5
    host repmgr repmgr 127.0.0.1/32 md5
    # host repmgr repmgr 192.168.10.0/24 md5
    host repmgr repmgr 0.0.0.0/0 md5
    local replication repmgr md5
    host replication repmgr 127.0.0.1/32 md5
    host replication repmgr 0.0.0.0/0 md5
    # host replication repmgr 192.168.1.0/24 md5
    
    ```

13. 重新加载配置文件

    ```
    $ pg_ctl reload
    ```

14. 创建repmgr.conf配置文件

    1. 主库68创建repmgr

       ```
       # cd /usr/local/postgresql/
       # vim repmgr.conf
       node_id=1  #节点id
       node_name=postgresql_04_68  #节点名字 
       conninfo='host=192.168.10.68 user=repmgr password=repmgr dbname=repmgr connect_timeout=2'  #连接信息 
       data_directory='/data/postgresql/pgdata'  #数据目录
       pg_bindir='/usr/local/postgresql/bin'  #pg数据库二进制目录
       
       # chmod 755 -R /usr/local/postgresql/
       ```

    2. 从库69创建repmgr

       ```
       # cd /usr/local/postgresql/
       # vim repmgr.conf
       node_id=2
       node_name=postgresql_05_69
       conninfo='host=192.168.10.69 user=repmgr password=repmgr dbname=repmgr connect_timeout=2'
       data_directory='/data/postgresql/pgdata'
       pg_bindir='/usr/local/postgresql/bin'
       
       # chmod 755 -R /usr/local/postgresql/
       ```

    3. 从库70创建repmgr

       ```
       node_id=3
       node_name=postgresql_06_70
       conninfo='host=192.168.10.70 user=repmgr password=repmgr dbname=repmgr connect_timeout=2'
       data_directory='/data/postgresql/pgdata'
       pg_bindir='/usr/local/postgresql/bin'
       
       # chmod 755 -R /usr/local/postgresql/
       ```

15. 注册主库

    ```
    # su - pgsql
    #也可以通过后面添加-F强制注册，用来解决配置文件配置错误的时候 repmgr -f /usr/local/postgresql/repmgr.conf primary register -F
    $ repmgr -f /usr/local/postgresql/repmgr.conf primary register
    INFO: connecting to primary database...
    NOTICE: attempting to install extension "repmgr"
    NOTICE: "repmgr" extension successfully installed
    NOTICE: primary node record (ID: 1) registered
    
    #顺便看看状态
    $ repmgr -f /usr/local/postgresql/repmgr.conf cluster show
     ID | Name             | Role    | Status    | Upstream | Location | Priority | Timeline | Connection string                                                             
    ----+------------------+---------+-----------+----------+----------+----------+----------+--------------------------------------------------------------------------------
     1  | postgresql_04_68 | primary | * running |          | default  | 100      | 1        | host=192.168.10.68 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
     
     #也可以进入数据库看看 两条结果一样
     $ psql -h 192.168.10.68 -p 5432 -U repmgr -d repmgr
    Password for user repmgr: 
    psql (12.6)
    Type "help" for help.
    
    repmgr=# select * from repmgr.nodes;
     node_id | upstream_node_id | active |    node_name     |  type   | location | priority |                                    conninfo 
                                       | repluser | slot_name |            config_file            
    ---------+------------------+--------+------------------+---------+----------+----------+---------------------------------------------
    -----------------------------------+----------+-----------+-----------------------------------
           1 |                  | t      | postgresql_04_68 | primary | default  |      100 | host=192.168.10.68 user=repmgr password=repm
    gr dbname=repmgr connect_timeout=2 | repmgr   |           | /usr/local/postgresql/repmgr.conf
    (1 row)
    
    ```

16. 克隆备库，1.所有机器必须配置一个备注的.pgpass密码文件,文件格式内容为：ip:port:db:user:pwd

    ```
    # 所有机器执行
    # su - pgsql
    $ echo "#ip:port:db:user:pwd" >> ~/.pgpass
    $ echo "192.168.10.68:5432:repmgr:repmgr:repmgr" >> ~/.pgpass
    $ echo "192.168.10.69:5432:repmgr:repmgr:repmgr" >> ~/.pgpass
    $ echo "192.168.10.70:5432:repmgr:repmgr:repmgr" >> ~/.pgpass
    $ chmod 0600 ~/.pgpass
    $ cat ~/.pgpass
    192.168.10.68:5432:repmgr:repmgr:repmgr
    192.168.10.69:5432:repmgr:repmgr:repmgr
    192.168.10.70:5432:repmgr:repmgr:repmgr
    ```

17. 克隆69和70从库

    1. 69机器执行

       ```
       # 不执行 先检查是否成功
       # su - pgsql
       $ repmgr -h 192.168.10.68 -U repmgr -d repmgr -f /usr/local/postgresql/repmgr.conf standby clone --dry-run
       NOTICE: destination directory "/data/postgresql/pgdata" provided
       INFO: connecting to source node
       DETAIL: connection string is: host=192.168.10.68 user=repmgr dbname=repmgr
       DETAIL: current installation size is 31 MB
       INFO: "repmgr" extension is installed in database "repmgr"
       INFO: parameter "max_wal_senders" set to 10
       NOTICE: checking for available walsenders on the source node (2 required)
       INFO: sufficient walsenders available on the source node
       DETAIL: 2 required, 10 available
       NOTICE: checking replication connections can be made to the source server (2 required)
       INFO: required number of replication connections could be made to the source server
       DETAIL: 2 replication connections required
       NOTICE: standby will attach to upstream node 1
       HINT: consider using the -c/--fast-checkpoint option
       INFO: all prerequisites for "standby clone" are met
       
       
       # 没有问题 执行 需要输入repmgr用户的密码
       # $ repmgr -h 192.168.10.68 -U repmgr -d repmgr -f /usr/local/postgresql/repmgr.conf standby clone
       NOTICE: destination directory "/data/postgresql/pgdata" provided
       INFO: connecting to source node
       DETAIL: connection string is: host=192.168.10.68 user=repmgr dbname=repmgr
       DETAIL: current installation size is 31 MB
       NOTICE: checking for available walsenders on the source node (2 required)
       NOTICE: checking replication connections can be made to the source server (2 required)
       INFO: checking and correcting permissions on existing directory "/data/postgresql/pgdata"
       NOTICE: starting backup (using pg_basebackup)...
       HINT: this may take some time; consider using the -c/--fast-checkpoint option
       INFO: executing:
         /usr/local/postgresql/bin/pg_basebackup -l "repmgr base backup"  -D /data/postgresql/pgdata -h 192.168.10.68 -p 5432 -U repmgr -X stream 
       Password: 
       NOTICE: standby clone (using pg_basebackup) complete
       NOTICE: you can now start your PostgreSQL server
       HINT: for example: pg_ctl -D /data/postgresql/pgdata start
       HINT: after starting the server, you need to register this standby with "repmgr standby register"
       
       # 看到上面提示 则克隆成功 
       # 根据提示 启动和注册服务
       # 启动
       $ pg_ctl -D /data/postgresql/pgdata start
       waiting for server to start....2021-02-22 22:29:22.700 CST [2631] LOG:  starting PostgreSQL 12.6 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 4.8.5 20150623 (Red Hat 4.8.5-44), 64-bit
       2021-02-22 22:29:22.700 CST [2631] LOG:  listening on IPv4 address "0.0.0.0", port 5432
       2021-02-22 22:29:22.700 CST [2631] LOG:  listening on IPv6 address "::", port 5432
       2021-02-22 22:29:22.713 CST [2631] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5432"
       2021-02-22 22:29:22.807 CST [2631] LOG:  redirecting log output to logging collector process
       2021-02-22 22:29:22.807 CST [2631] HINT:  Future log output will appear in directory "/data/postgresql/logs".
        done
       server started
       
       # 注册
       $ repmgr -f /usr/local/postgresql/repmgr.conf standby register
       INFO: connecting to local node "postgresql_05_69" (ID: 2)
       INFO: connecting to primary database
       WARNING: --upstream-node-id not supplied, assuming upstream node is primary (node ID 1)
       INFO: standby registration complete
       NOTICE: standby node "postgresql_05_69" (ID: 2) successfully registered
       
       #查看集群状态
       $ repmgr -f /usr/local/postgresql/repmgr.conf cluster show
        ID | Name             | Role    | Status    | Upstream         | Location | Priority | Timeline | Connection string                                                             
       ----+------------------+---------+-----------+------------------+----------+----------+----------+--------------------------------------------------------------------------------
        1  | postgresql_04_68 | primary | * running |                  | default  | 100      | 1        | host=192.168.10.68 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
        2  | postgresql_05_69 | standby |   running | postgresql_04_68 | default  | 100      | 1        | host=192.168.10.69 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
       
       #查看从库复制状态
       $ psql -h 192.168.10.69 -p 5432
       Password for user postgres: 
       psql (12.6)
       Type "help" for help.
       
       postgres=# \x
       Expanded display is on.
       postgres=# select * from pg_stat_wal_receiver;
       -[ RECORD 1 ]---------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
       pid                   | 2637
       status                | streaming
       receive_start_lsn     | 0/3000000
       receive_start_tli     | 1
       received_lsn          | 0/3000698
       received_tli          | 1
       last_msg_send_time    | 2021-02-22 22:33:46.669293+08
       last_msg_receipt_time | 2021-02-22 22:33:46.669488+08
       latest_end_lsn        | 0/3000698
       latest_end_time       | 2021-02-22 22:31:46.383765+08
       slot_name             | 
       sender_host           | 192.168.10.68
       sender_port           | 5432
       conninfo              | user=repmgr password=******** connect_timeout=2 dbname=replication host=192.168.10.68 port=5432 application_name=postgresql_05_69 fallback_application_name=walreceiver sslmode=disable sslcompression=0 gssencmode=disable krbsrvname=postgres target_session_attrs=any
       ```

    2. 70机器执行

       ```
       # 不执行 先检查是否成功
       # su - pgsql
       $ repmgr -h 192.168.10.68 -U repmgr -d repmgr -f /usr/local/postgresql/repmgr.conf standby clone --dry-run
       NOTICE: destination directory "/data/postgresql/pgdata" provided
       INFO: connecting to source node
       DETAIL: connection string is: host=192.168.10.68 user=repmgr dbname=repmgr
       DETAIL: current installation size is 31 MB
       INFO: "repmgr" extension is installed in database "repmgr"
       INFO: parameter "max_wal_senders" set to 10
       NOTICE: checking for available walsenders on the source node (2 required)
       INFO: sufficient walsenders available on the source node
       DETAIL: 2 required, 9 available
       NOTICE: checking replication connections can be made to the source server (2 required)
       INFO: required number of replication connections could be made to the source server
       DETAIL: 2 replication connections required
       NOTICE: standby will attach to upstream node 1
       HINT: consider using the -c/--fast-checkpoint option
       INFO: all prerequisites for "standby clone" are met
       
       
       # 没有问题 执行 需要输入repmgr用户的密码
       $ repmgr -h 192.168.10.68 -U repmgr -d repmgr -f /usr/local/postgresql/repmgr.conf standby clone
       NOTICE: destination directory "/data/postgresql/pgdata" provided
       INFO: connecting to source node
       DETAIL: connection string is: host=192.168.10.68 user=repmgr dbname=repmgr
       DETAIL: current installation size is 31 MB
       NOTICE: checking for available walsenders on the source node (2 required)
       NOTICE: checking replication connections can be made to the source server (2 required)
       INFO: checking and correcting permissions on existing directory "/data/postgresql/pgdata"
       NOTICE: starting backup (using pg_basebackup)...
       HINT: this may take some time; consider using the -c/--fast-checkpoint option
       INFO: executing:
         /usr/local/postgresql/bin/pg_basebackup -l "repmgr base backup"  -D /data/postgresql/pgdata -h 192.168.10.68 -p 5432 -U repmgr -X stream 
       Password: 
       NOTICE: standby clone (using pg_basebackup) complete
       NOTICE: you can now start your PostgreSQL server
       HINT: for example: pg_ctl -D /data/postgresql/pgdata start
       HINT: after starting the server, you need to register this standby with "repmgr standby register"
       
       # 看到上面提示 则克隆成功 
       # 根据提示 启动和注册服务
       # 启动
       $ pg_ctl -D /data/postgresql/pgdata start
       waiting for server to start....2021-02-22 22:36:44.805 CST [2508] LOG:  starting PostgreSQL 12.6 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 4.8.5 20150623 (Red Hat 4.8.5-44), 64-bit
       2021-02-22 22:36:44.806 CST [2508] LOG:  listening on IPv4 address "0.0.0.0", port 5432
       2021-02-22 22:36:44.806 CST [2508] LOG:  listening on IPv6 address "::", port 5432
       2021-02-22 22:36:44.819 CST [2508] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5432"
       2021-02-22 22:36:44.923 CST [2508] LOG:  redirecting log output to logging collector process
       2021-02-22 22:36:44.923 CST [2508] HINT:  Future log output will appear in directory "/data/postgresql/logs".
        done
       server started
       
       
       # 注册
       $ repmgr -f /usr/local/postgresql/repmgr.conf standby register
       INFO: connecting to local node "postgresql_06_70" (ID: 3)
       INFO: connecting to primary database
       WARNING: --upstream-node-id not supplied, assuming upstream node is primary (node ID 1)
       INFO: standby registration complete
       NOTICE: standby node "postgresql_06_70" (ID: 3) successfully registered
       
       #查看集群状态
       $ repmgr -f /usr/local/postgresql/repmgr.conf cluster show
       ID | Name             | Role    | Status    | Upstream         | Location | Priority | Timeline | Connection string                                                             
       ----+------------------+---------+-----------+------------------+----------+----------+----------+--------------------------------------------------------------------------------
        1  | postgresql_04_68 | primary | * running |                  | default  | 100      | 1        | host=192.168.10.68 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
        2  | postgresql_05_69 | standby |   running | postgresql_04_68 | default  | 100      | 1        | host=192.168.10.69 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
        3  | postgresql_06_70 | standby |   running | postgresql_04_68 | default  | 100      | 1        | host=192.168.10.70 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
       
       ```

18. 添加71服务器 witness角色 71服务器添加pgpass配置

    ```
    # su - pgsql
    $ echo "#ip:port:db:user:pwd" >> ~/.pgpass
    $ echo "192.168.10.68:5432:repmgr:repmgr:repmgr" >> ~/.pgpass
    $ echo "192.168.10.69:5432:repmgr:repmgr:repmgr" >> ~/.pgpass
    $ echo "192.168.10.70:5432:repmgr:repmgr:repmgr" >> ~/.pgpass
    $ echo "192.168.10.71:5432:repmgr:repmgr:repmgr" >> ~/.pgpass
    $ chmod 0600 ~/.pgpass
    ```

19. 其他68 69 70服务器添加pgpass配置

    ```
    $ echo "192.168.10.71:5432:repmgr:repmgr:repmgr" >> ~/.pgpass
    ```

20. 71服务器初始化数据库，注意 目前和集群没有关系。（也可以和集群一起安装）

    ```
    $ /usr/local/postgresql/bin/initdb -D /data/postgresql/pgdata -E UTF8 --locale=en_US.utf8 -U postgres
    The files belonging to this database system will be owned by user "pgsql".
    This user must also own the server process.
    
    The database cluster will be initialized with locale "en_US.utf8".
    The default text search configuration will be set to "english".
    
    Data page checksums are disabled.
    
    fixing permissions on existing directory /data/postgresql/pgdata ... ok
    creating subdirectories ... ok
    selecting dynamic shared memory implementation ... posix
    selecting default max_connections ... 100
    selecting default shared_buffers ... 128MB
    selecting default time zone ... Asia/Shanghai
    creating configuration files ... ok
    running bootstrap script ... ok
    performing post-bootstrap initialization ... ok
    syncing data to disk ... ok
    
    initdb: warning: enabling "trust" authentication for local connections
    You can change this by editing pg_hba.conf or using the option -A, or
    --auth-local and --auth-host, the next time you run initdb.
    
    Success. You can now start the database server using:
    
        /usr/local/postgresql/bin/pg_ctl -D /data/postgresql/pgdata -l logfile start
    
    ```

21. 从69从库复制配置文件进来，必须从从库复制

    ```
    $ scp postgresql_05_69:/data/postgresql/pgdata/postgresql.conf /data/postgresql/pgdata/
    postgresql.conf                                                                                     100%   26KB   4.3MB/s   00:00    
    $ scp postgresql_05_69:/data/postgresql/pgdata/pg_hba.conf /data/postgresql/pgdata/
    pg_hba.conf                                
    ```

22. 启动71数据库

    ```
    $ /usr/local/postgresql/bin/pg_ctl -D /data/postgresql/pgdata -l logfile start
    waiting for server to start.... done
    server started
    ```

23. 71上创建repmgr用户，和第11步一样

    ```
    $ createuser -s repmgr -h 127.0.0.1
    $ createdb repmgr -O repmgr -h 127.0.0.1
    $ psql -h 127.0.0.1 -c "alter user repmgr with password 'repmgr'"
    ALTER ROLE
    $ psql -h 127.0.0.1 -c "alter user repmgr set search_path to repmgr, \"\$user\",public";
    ALTER ROLE
    ```

24. 安装repmgr

    ```
    # cd /usr/local/src/
    # tar zxvf repmgr-5.0.tar.gz 
    # cd repmgr-5.0.0/
    # ./configure
    # make -j
    # 安装 安装的时候是作为pg的插件来安装的，所以注意上面的环境变量的配置
    # make install  
    Building against PostgreSQL 12
    /usr/bin/mkdir -p '/usr/local/postgresql/lib'
    /usr/bin/mkdir -p '/usr/local/postgresql/share/extension'
    /usr/bin/mkdir -p '/usr/local/postgresql/share/extension'
    /usr/bin/mkdir -p '/usr/local/postgresql/bin'
    /usr/bin/install -c -m 755  repmgr.so '/usr/local/postgresql/lib/repmgr.so'
    /usr/bin/install -c -m 644 .//repmgr.control '/usr/local/postgresql/share/extension/'
    /usr/bin/install -c -m 644 .//repmgr--unpackaged--4.0.sql .//repmgr--4.0.sql .//repmgr--4.0--4.1.sql .//repmgr--4.1.sql .//repmgr--4.1--4.2.sql .//repmgr--4.2.sql .//repmgr--4.2--4.3.sql .//repmgr--4.3.sql .//repmgr--4.3--4.4.sql .//repmgr--4.4.sql .//repmgr--4.4--5.0.sql .//repmgr--5.0.sql  '/usr/local/postgresql/share/extension/'
    /usr/bin/install -c -m 755 repmgr repmgrd '/usr/local/postgresql/bin/'
    
    # repmgr --help
    ```

25. 配置repmgr.conf文件

    ```
    # vim /usr/local/postgresql/repmgr.conf
    node_id=4
    node_name=postgresql_07_71
    conninfo='host=192.168.10.68 user=repmgr password=repmgr dbname=repmgr connect_timeout=2'
    data_directory='/data/postgresql/pgdata'
    pg_bindir='/usr/local/postgresql/bin'
    # chmod 755 -R /usr/local/postgresql/
    ```

26. 注册witness节点

    ```
    # su - pgsql
    $ repmgr -f /usr/local/postgresql/repmgr.conf -h 192.168.10.68 -U repmgr -d repmgr witness register
    INFO: connecting to witness node "postgresql_07_71" (ID: 4)
    INFO: connecting to primary node
    NOTICE: attempting to install extension "repmgr"
    NOTICE: "repmgr" extension successfully installed
    INFO: witness registration complete
    NOTICE: witness node "postgresql_07_71" (ID: 4) successfully registered
    
    # 再次查看集群状态，多了witness服务，其他节点也可以看到
    [pgsql@postgresql_07_71 ~]$ repmgr -f /usr/local/postgresql/repmgr.conf cluster show
     ID | Name             | Role    | Status    | Upstream         | Location | Priority | Timeline | Connection string                                                             
    ----+------------------+---------+-----------+------------------+----------+----------+----------+--------------------------------------------------------------------------------
     1  | postgresql_04_68 | primary | * running |                  | default  | 100      | 1        | host=192.168.10.68 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
     2  | postgresql_05_69 | standby |   running | postgresql_04_68 | default  | 100      | 1        | host=192.168.10.69 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
     3  | postgresql_06_70 | standby |   running | postgresql_04_68 | default  | 100      | 1        | host=192.168.10.70 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
     4  | postgresql_07_71 | witness | * running | postgresql_04_68 | default  | 0        | 1        | host=192.168.10.71 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    
    ```

28. 启用pg数据库的复制槽,68 69 70节点执行

    pg12之前还需要use_replication_slots=true这个参数，12之后不需要了

    ```
    # 修改postgresql.conf的wal_log_hints=on
    $ vim /data/postgresql/pgdata/postgresql.conf
    ……
    max_replication_slots = 10
    wal_log_hints=on
    
    #执行完成后，重启
    $ pg_ctl stop
    $ pg_ctl -D /data/postgresql/pgdata start
    ```

    

# 正常情况主从切换测试

1. 先看当前主库是谁

   ```
   $ repmgr -f /usr/local/postgresql/repmgr.conf cluster show
    ID | Name             | Role    | Status    | Upstream         | Location | Priority | Timeline | Connection string                                                             
   ----+------------------+---------+-----------+------------------+----------+----------+----------+--------------------------------------------------------------------------------
    1  | postgresql_04_68 | primary | * running |                  | default  | 100      | 1        | host=192.168.10.68 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    2  | postgresql_05_69 | standby |   running | postgresql_04_68 | default  | 100      | 1        | host=192.168.10.69 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    3  | postgresql_06_70 | standby |   running | postgresql_04_68 | default  | 100      | 1        | host=192.168.10.70 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    4  | postgresql_07_71 | witness | * running | postgresql_04_68 | default  | 0        | 1        | host=192.168.10.71 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
   
   # 由此看出 是68服务器
   ```

2. 提升69为主库,在69服务器上执行命令。(需要提升哪个为主库就在哪个执行)

   --siblings-follow : 所有的从库同步源自动改成最新的主库节点

   --force-rewind : 如果repmgr检测到需要执行pg_rewind (主从数据同步)的时候,在执行pg_rewind之前,在新主节点执行checkpoint;

   ```
   # 先测试下
   $ repmgr -f /usr/local/postgresql/repmgr.conf standby switchover --siblings-follow --dry-run --force-rewind
   NOTICE: checking switchover on node "postgresql_05_69" (ID: 2) in --dry-run mode
   INFO: prerequisites for using pg_rewind are met
   INFO: SSH connection to host "192.168.10.68" succeeded
   INFO: able to execute "repmgr" on remote host "localhost"
   INFO: all sibling nodes are reachable via SSH
   INFO: 3 walsenders required, 10 available
   INFO: demotion candidate is able to make replication connection to promotion candidate
   INFO: 0 pending archive files
   INFO: replication lag on this standby is 0 seconds
   NOTICE: local node "postgresql_05_69" (ID: 2) would be promoted to primary; current primary "postgresql_04_68" (ID: 1) would be demoted to standby
   INFO: following shutdown command would be run on node "postgresql_04_68":
     "/usr/local/postgresql/bin/pg_ctl  -D '/data/postgresql/pgdata' -W -m fast stop"
   INFO: prerequisites for executing STANDBY SWITCHOVER are met
   
   #没有问题 执行
   $ repmgr -f /usr/local/postgresql/repmgr.conf standby switchover --siblings-follow --force-rewind
   NOTICE: executing switchover on node "postgresql_05_69" (ID: 2)
   NOTICE: local node "postgresql_05_69" (ID: 2) will be promoted to primary; current primary "postgresql_04_68" (ID: 1) will be demoted to standby
   NOTICE: stopping current primary node "postgresql_04_68" (ID: 1)
   NOTICE: issuing CHECKPOINT
   DETAIL: executing server command "/usr/local/postgresql/bin/pg_ctl  -D '/data/postgresql/pgdata' -W -m fast stop"
   INFO: checking for primary shutdown; 1 of 60 attempts ("shutdown_check_timeout")
   INFO: checking for primary shutdown; 2 of 60 attempts ("shutdown_check_timeout")
   NOTICE: current primary has been cleanly shut down at location 0/8000028
   NOTICE: promoting standby to primary
   DETAIL: promoting server "postgresql_05_69" (ID: 2) using pg_promote()
   NOTICE: waiting up to 60 seconds (parameter "promote_check_timeout") for promotion to complete
   NOTICE: STANDBY PROMOTE successful
   DETAIL: server "postgresql_05_69" (ID: 2) was successfully promoted to primary
   NOTICE: issuing CHECKPOINT
   INFO: local node 1 can attach to rejoin target node 2
   DETAIL: local node's recovery point: 0/8000028; rejoin target node's fork point: 0/80000A0
   NOTICE: setting node 1's upstream to node 2
   WARNING: unable to ping "host=192.168.10.68 user=repmgr password=repmgr dbname=repmgr connect_timeout=2"
   DETAIL: PQping() returned "PQPING_NO_RESPONSE"
   NOTICE: starting server using "/usr/local/postgresql/bin/pg_ctl  -w -D '/data/postgresql/pgdata' start"
   NOTICE: NODE REJOIN successful
   DETAIL: node 1 is now attached to node 2
   NOTICE: executing STANDBY FOLLOW on 2 of 2 siblings
   INFO: STANDBY FOLLOW successfully executed on all reachable sibling nodes
   NOTICE: switchover was successful
   DETAIL: node "postgresql_05_69" is now primary and node "postgresql_04_68" is attached as standby
   NOTICE: STANDBY SWITCHOVER has completed successfully
   ```

3. 再次查看主库，变成了69

   ```
   $ repmgr -f /usr/local/postgresql/repmgr.conf cluster show
    ID | Name             | Role    | Status    | Upstream         | Location | Priority | Timeline | Connection string                                                             
   ----+------------------+---------+-----------+------------------+----------+----------+----------+--------------------------------------------------------------------------------
    1  | postgresql_04_68 | standby |   running | postgresql_05_69 | default  | 100      | 1        | host=192.168.10.68 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    2  | postgresql_05_69 | primary | * running |                  | default  | 100      | 2        | host=192.168.10.69 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    3  | postgresql_06_70 | standby |   running | postgresql_05_69 | default  | 100      | 1        | host=192.168.10.70 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    4  | postgresql_07_71 | witness | * running | postgresql_05_69 | default  | 0        | 1        | host=192.168.10.71 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
   ```

4. 测试数据

   ```
   $ psql -h 127.0.0.1 -p 5432
   psql (12.6)
   Type "help" for help.
   
   postgres=# create user pgtest with password 'pgtest' nocreatedb;
   CREATE ROLE
   postgres=# create database testdb with owner=pgtest;
   CREATE DATABASE
   postgres=# quit
   
   $ psql -h 192.168.10.68 -p 5432 -U pgtest -d testdb
   create table test1 (name varchar(50));
   testdb=> insert into test1 values('test1');
   INSERT 0 1
   testdb=> select * from test1;
    name  
   -------
    test1
   (1 row)
   
   ```

5. 再讲68切会主库，然后再写入数据

   ```
   $ repmgr -f /usr/local/postgresql/repmgr.conf standby switchover --siblings-follow --force-rewind
   NOTICE: executing switchover on node "postgresql_04_68" (ID: 1)
   NOTICE: local node "postgresql_04_68" (ID: 1) will be promoted to primary; current primary "postgresql_05_69" (ID: 2) will be demoted to standby
   NOTICE: stopping current primary node "postgresql_05_69" (ID: 2)
   NOTICE: issuing CHECKPOINT
   DETAIL: executing server command "/usr/local/postgresql/bin/pg_ctl  -D '/data/postgresql/pgdata' -W -m fast stop"
   INFO: checking for primary shutdown; 1 of 60 attempts ("shutdown_check_timeout")
   INFO: checking for primary shutdown; 2 of 60 attempts ("shutdown_check_timeout")
   NOTICE: current primary has been cleanly shut down at location 0/9000028
   NOTICE: promoting standby to primary
   DETAIL: promoting server "postgresql_04_68" (ID: 1) using pg_promote()
   NOTICE: waiting up to 60 seconds (parameter "promote_check_timeout") for promotion to complete
   NOTICE: STANDBY PROMOTE successful
   DETAIL: server "postgresql_04_68" (ID: 1) was successfully promoted to primary
   NOTICE: issuing CHECKPOINT
   INFO: local node 2 can attach to rejoin target node 1
   DETAIL: local node's recovery point: 0/9000028; rejoin target node's fork point: 0/90000A0
   NOTICE: setting node 2's upstream to node 1
   WARNING: unable to ping "host=192.168.10.69 user=repmgr password=repmgr dbname=repmgr connect_timeout=2"
   DETAIL: PQping() returned "PQPING_NO_RESPONSE"
   NOTICE: starting server using "/usr/local/postgresql/bin/pg_ctl  -w -D '/data/postgresql/pgdata' start"
   NOTICE: NODE REJOIN successful
   DETAIL: node 2 is now attached to node 1
   NOTICE: executing STANDBY FOLLOW on 2 of 2 siblings
   INFO: STANDBY FOLLOW successfully executed on all reachable sibling nodes
   NOTICE: switchover was successful
   DETAIL: node "postgresql_04_68" is now primary and node "postgresql_05_69" is attached as standby
   NOTICE: STANDBY SWITCHOVER has completed successfully
   $ repmgr -f /usr/local/postgresql/repmgr.conf cluster show
    ID | Name             | Role    | Status    | Upstream         | Location | Priority | Timeline | Connection string                                                             
   ----+------------------+---------+-----------+------------------+----------+----------+----------+--------------------------------------------------------------------------------
    1  | postgresql_04_68 | primary | * running |                  | default  | 100      | 3        | host=192.168.10.68 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    2  | postgresql_05_69 | standby |   running | postgresql_04_68 | default  | 100      | 2        | host=192.168.10.69 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    3  | postgresql_06_70 | standby |   running | postgresql_04_68 | default  | 100      | 2        | host=192.168.10.70 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    4  | postgresql_07_71 | witness | * running | postgresql_04_68 | default  | 0        | 1        | host=192.168.10.71 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
   
   ```

6. 测试数据

   ```
   $ psql -h 192.168.10.68 -p 5432 -U pgtest -d testdb
   Password for user pgtest: 
   psql (12.6)
   Type "help" for help.
   
   testdb=> insert into test1 values('test2');
   INSERT 0 1
   testdb=> select * from test1;
    name  
   -------
    test1
    test2
   (2 rows)
   
   ```

# 异常情况主从手动切换测试

环境：模拟主库68出现故障，切换从库70到主库，原主库68恢复后恢复为主库 相当于重新组建一个以68为主的集群

1. 停止68主库

   ```
   # su - pgsql
   Last login: Mon Feb 22 22:10:25 CST 2021 on pts/0
   $ pg_ctl -m fast stop
   waiting for server to shut down.... done
   server stopped
   
   #查看68状态 可以看出 68连接失败(在69执行，因为68已经停掉了，连不上集群了)
   $  repmgr -f /usr/local/postgresql/repmgr.conf cluster show
    ID | Name             | Role    | Status        | Upstream           | Location | Priority | Timeline | Connection string                                                             
   ----+------------------+---------+---------------+--------------------+----------+----------+----------+--------------------------------------------------------------------------------
    1  | postgresql_04_68 | primary | ? unreachable |                    | default  | 100      | ?        | host=192.168.10.68 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    2  | postgresql_05_69 | standby |   running     | ? postgresql_04_68 | default  | 100      | 3        | host=192.168.10.69 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    3  | postgresql_06_70 | standby |   running     | ? postgresql_04_68 | default  | 100      | 3        | host=192.168.10.70 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    4  | postgresql_07_71 | witness | * running     | ? postgresql_04_68 | default  | 0        | 1        | host=192.168.10.71 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
   
   WARNING: following issues were detected
     - unable to connect to node "postgresql_04_68" (ID: 1)
     - node "postgresql_04_68" (ID: 1) is registered as an active primary but is unreachable
     - unable to connect to node "postgresql_05_69" (ID: 2)'s upstream node "postgresql_04_68" (ID: 1)
     - unable to determine if node "postgresql_05_69" (ID: 2) is attached to its upstream node "postgresql_04_68" (ID: 1)
     - unable to connect to node "postgresql_06_70" (ID: 3)'s upstream node "postgresql_04_68" (ID: 1)
     - unable to determine if node "postgresql_06_70" (ID: 3) is attached to its upstream node "postgresql_04_68" (ID: 1)
     - unable to connect to node "postgresql_07_71" (ID: 4)'s upstream node "postgresql_04_68" (ID: 1)
   
   ```

2. 提升从库70为主库，在70上执行

   ```
   # su - pgsql
   Last login: Mon Feb 22 23:31:33 CST 2021 from postgresql_05_69 on pts/1
   $ repmgr -f /usr/local/postgresql/repmgr.conf --siblings-follow standby promote
   NOTICE: promoting standby to primary
   DETAIL: promoting server "postgresql_06_70" (ID: 3) using pg_promote()
   NOTICE: waiting up to 60 seconds (parameter "promote_check_timeout") for promotion to complete
   NOTICE: STANDBY PROMOTE successful
   DETAIL: server "postgresql_06_70" (ID: 3) was successfully promoted to primary
   NOTICE: executing STANDBY FOLLOW on 2 of 2 siblings
   INFO: STANDBY FOLLOW successfully executed on all reachable sibling nodes
   
   #查看集群状态，可以看到70变成了主库 68已经down掉了
   $ repmgr -f /usr/local/postgresql/repmgr.conf cluster show
    ID | Name             | Role    | Status    | Upstream         | Location | Priority | Timeline | Connection string                                                             
   ----+------------------+---------+-----------+------------------+----------+----------+----------+--------------------------------------------------------------------------------
    1  | postgresql_04_68 | primary | - failed  |                  | default  | 100      | ?        | host=192.168.10.68 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    2  | postgresql_05_69 | standby |   running | postgresql_06_70 | default  | 100      | 3        | host=192.168.10.69 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    3  | postgresql_06_70 | primary | * running |                  | default  | 100      | 4        | host=192.168.10.70 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    4  | postgresql_07_71 | witness | * running | postgresql_06_70 | default  | 0        | 1        | host=192.168.10.71 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
   
   WARNING: following issues were detected
     - unable to connect to node "postgresql_04_68" (ID: 1)
   
   ```

3. 68主库修复，错误做法：如果需要68重新为主库，则需要68重新从70上克隆一份数据，并且提升为主库，不能直接启动为主库，因为会存在68数据不一致情况

   ```
   #启动68主库
   $ pg_ctl -D /data/postgresql/pgdata start
   waiting for server to start....2021-02-23 13:52:48.089 CST [5518] LOG:  starting PostgreSQL 12.6 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 4.8.5 20150623 (Red Hat 4.8.5-44), 64-bit
   2021-02-23 13:52:48.089 CST [5518] LOG:  listening on IPv4 address "0.0.0.0", port 5432
   2021-02-23 13:52:48.089 CST [5518] LOG:  listening on IPv6 address "::", port 5432
   2021-02-23 13:52:48.103 CST [5518] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5432"
   2021-02-23 13:52:48.198 CST [5518] LOG:  redirecting log output to logging collector process
   2021-02-23 13:52:48.198 CST [5518] HINT:  Future log output will appear in directory "/data/postgresql/logs".
    done
   server started
   
   # 68上查看集群状态，可以看到68变成了主库，70变成了从库，running as primary标明以前曾经是主库
   $ repmgr -f /usr/local/postgresql/repmgr.conf cluster show
    ID | Name             | Role    | Status               | Upstream           | Location | Priority | Timeline | Connection string                                                             
   ----+------------------+---------+----------------------+--------------------+----------+----------+----------+--------------------------------------------------------------------------------
    1  | postgresql_04_68 | primary | * running            |                    | default  | 100      | 3        | host=192.168.10.68 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    2  | postgresql_05_69 | standby |   running            | ! postgresql_06_70 | default  | 100      | 3        | host=192.168.10.69 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    3  | postgresql_06_70 | standby | ! running as primary |                    | default  | 100      | 4        | host=192.168.10.70 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    4  | postgresql_07_71 | witness | * running            | ! postgresql_06_70 | default  | 0        | 1        | host=192.168.10.71 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
   
   WARNING: following issues were detected
     - node "postgresql_05_69" (ID: 2) reports a different upstream (reported: "postgresql_06_70", expected "postgresql_04_68")
     - node "postgresql_06_70" (ID: 3) is registered as standby but running as primary
     - node "postgresql_07_71" (ID: 4) reports a different upstream (reported: "postgresql_06_70", expected "postgresql_04_68")
   
   
   # 但是在70上执行查看集群状态，发现有两个主库(此时因为有witness节点，并没有产生脑裂)
   $ repmgr -f /usr/local/postgresql/repmgr.conf cluster show
    ID | Name             | Role    | Status    | Upstream         | Location | Priority | Timeline | Connection string                                                             
   ----+------------------+---------+-----------+------------------+----------+----------+----------+--------------------------------------------------------------------------------
    1  | postgresql_04_68 | primary | ! running |                  | default  | 100      | 3        | host=192.168.10.68 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    2  | postgresql_05_69 | standby |   running | postgresql_06_70 | default  | 100      | 3        | host=192.168.10.69 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    3  | postgresql_06_70 | primary | * running |                  | default  | 100      | 4        | host=192.168.10.70 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    4  | postgresql_07_71 | witness | * running | postgresql_06_70 | default  | 0        | 1        | host=192.168.10.71 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
   
   WARNING: following issues were detected
     - node "postgresql_04_68" (ID: 1) is running but the repmgr node record is inactive
     
   # 此时讲 69作为68的备库就会出现数据库数据比较旧或者比较旧的情况 (此刻是因为witness观察到了有两个主库，并且68数据是有问题的)
   $ repmgr -f /usr/local/postgresql/repmgr.conf node rejoin -d 'host=192.168.10.68 user=repmgr dbname=repmgr connect_timeout=2' --force-rewind --dry-run --verbose
   NOTICE: using provided configuration file "/usr/local/postgresql/repmgr.conf"
   ERROR: database is still running in state "in archive recovery"
   HINT: "repmgr node rejoin" cannot be executed on a running node
   
   # 此时讲 69作为68的备库就会出现数据库数据比较旧或者比较旧的情况 (此刻是因为witness观察到了有两个主库，并且68数据是有问题的)
   $ repmgr -f /usr/local/postgresql/repmgr.conf node rejoin -d 'host=192.168.10.69 user=repmgr dbname=repmgr connect_timeout=2' --force-rewind --dry-run --verbose
   NOTICE: using provided configuration file "/usr/local/postgresql/repmgr.conf"
   INFO: replication connection to the rejoin target node was successful
   INFO: local and rejoin target system identifiers match
   DETAIL: system identifier is 6932084377171332038
   ERROR: this node is ahead of the rejoin target
   DETAIL: local node lsn is 0/14000028, rejoin target lsn is 0/13076078
   ```

4. 68主库修复，正确做法，停止68数据库，先克隆为70的从库(在68上执行)

   ```
   $ pg_ctl stop
   waiting for server to shut down.... done
   server stopped
   
   # 68作为70的备库
   $ repmgr -f /usr/local/postgresql/repmgr.conf node rejoin -d 'host=192.168.10.70 user=repmgr dbname=repmgr connect_timeout=2' --force-rewind --dry-run --verbose
   NOTICE: using provided configuration file "/usr/local/postgresql/repmgr.conf"
   ERROR: database is still running in state "in production"
   HINT: "repmgr node rejoin" cannot be executed on a running node
   [pgsql@postgresql_04_68 ~]$ repmgr -f /usr/local/postgresql/repmgr.conf node rejoin -d 'host=192.168.10.68 user=repmgr dbname=repmgr connect_timeout=2' --force-rewind --dry-run --verbose
   NOTICE: using provided configuration file "/usr/local/postgresql/repmgr.conf"
   ERROR: database is still running in state "in production"
   HINT: "repmgr node rejoin" cannot be executed on a running node
   [pgsql@postgresql_04_68 ~]$ pg_ctl stop
   waiting for server to shut down.... done
   server stopped
   
   
   $ repmgr -f /usr/local/postgresql/repmgr.conf node rejoin -d 'host=192.168.10.70 user=repmgr dbname=repmgr connect_timeout=2' --force-rewind --dry-run --verbose
   NOTICE: using provided configuration file "/usr/local/postgresql/repmgr.conf"
   INFO: replication connection to the rejoin target node was successful
   INFO: local and rejoin target system identifiers match
   DETAIL: system identifier is 6932084377171332038
   NOTICE: pg_rewind execution required for this node to attach to rejoin target node 3
   DETAIL: rejoin target server's timeline 4 forked off current database system timeline 3 before current recovery point 0/B000028
   INFO: prerequisites for using pg_rewind are met
   INFO: temporary archive directory "/tmp/repmgr-config-archive-postgresql_04_68" created
   INFO: 0 files would have been copied to "/tmp/repmgr-config-archive-postgresql_04_68"
   INFO: temporary archive directory "/tmp/repmgr-config-archive-postgresql_04_68" deleted
   INFO: pg_rewind would now be executed
   DETAIL: pg_rewind command is:
     /usr/local/postgresql/bin/pg_rewind -D '/data/postgresql/pgdata' --source-server='host=192.168.10.70 user=repmgr password=repmgr dbname=repmgr connect_timeout=2'
   INFO: prerequisites for executing NODE REJOIN are met
   
   $ repmgr -f /usr/local/postgresql/repmgr.conf node rejoin -d 'host=192.168.10.70 user=repmgr dbname=repmgr connect_timeout=2' --force-rewind --verbose
   NOTICE: using provided configuration file "/usr/local/postgresql/repmgr.conf"
   NOTICE: pg_rewind execution required for this node to attach to rejoin target node 3
   DETAIL: rejoin target server's timeline 4 forked off current database system timeline 3 before current recovery point 0/B000028
   INFO: prerequisites for using pg_rewind are met
   INFO: 0 files copied to "/tmp/repmgr-config-archive-postgresql_04_68"
   NOTICE: executing pg_rewind
   DETAIL: pg_rewind command is "/usr/local/postgresql/bin/pg_rewind -D '/data/postgresql/pgdata' --source-server='host=192.168.10.70 user=repmgr password=repmgr dbname=repmgr connect_timeout=2'"
   pg_rewind: servers diverged at WAL location 0/A0000A0 on timeline 3
   pg_rewind: rewinding from last common checkpoint at 0/A000028 on timeline 3
   pg_rewind: Done!
   NOTICE: 0 files copied to /data/postgresql/pgdata
   INFO: directory "/tmp/repmgr-config-archive-postgresql_04_68" deleted
   NOTICE: setting node 1's upstream to node 3
   WARNING: unable to ping "host=192.168.10.68 user=repmgr password=repmgr dbname=repmgr connect_timeout=2"
   DETAIL: PQping() returned "PQPING_NO_RESPONSE"
   NOTICE: starting server using "/usr/local/postgresql/bin/pg_ctl  -w -D '/data/postgresql/pgdata' start"
   INFO: demoted primary is pingable
   INFO: node 1 has attached to its upstream node
   NOTICE: NODE REJOIN successful
   DETAIL: node 1 is now attached to node 3
   
   
   #查看状态 68已经变成了备库
   $ repmgr -f /usr/local/postgresql/repmgr.conf cluster show
    ID | Name             | Role    | Status    | Upstream         | Location | Priority | Timeline | Connection string                                                             
   ----+------------------+---------+-----------+------------------+----------+----------+----------+--------------------------------------------------------------------------------
    1  | postgresql_04_68 | standby |   running | postgresql_06_70 | default  | 100      | 3        | host=192.168.10.68 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    2  | postgresql_05_69 | standby |   running | postgresql_06_70 | default  | 100      | 4        | host=192.168.10.69 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    3  | postgresql_06_70 | primary | * running |                  | default  | 100      | 4        | host=192.168.10.70 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    4  | postgresql_07_71 | witness | * running | postgresql_06_70 | default  | 0        | 1        | host=192.168.10.71 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
   
   ```

5. 测试数据

   ```
   $ psql -h 192.168.10.70 -p 5432 -U pgtest -d testdb
   Password for user pgtest: 
   psql (12.6)
   Type "help" for help.
   
   testdb=> insert into test1 values('test2');
   INSERT 0 1
   testdb=> select * from test1;
    name  
   -------
    test1
    test2
   (2 rows)
   
   $ psql -h 192.168.10.68 -p 5432 -U pgtest -d testdb
   Password for user pgtest: 
   psql (12.6)
   Type "help" for help.
   
   testdb=> select * from test1;
    name  
   -------
    test1
    test2
   (2 rows)
   
   ```

6. 在讲68切回主库

   ```
   $ repmgr -f /usr/local/postgresql/repmgr.conf standby switchover --siblings-follow --dry-run --force-rewind
   NOTICE: checking switchover on node "postgresql_04_68" (ID: 1) in --dry-run mode
   INFO: prerequisites for using pg_rewind are met
   INFO: SSH connection to host "192.168.10.70" succeeded
   INFO: able to execute "repmgr" on remote host "localhost"
   INFO: all sibling nodes are reachable via SSH
   INFO: 3 walsenders required, 10 available
   INFO: demotion candidate is able to make replication connection to promotion candidate
   INFO: 0 pending archive files
   INFO: replication lag on this standby is 0 seconds
   NOTICE: local node "postgresql_04_68" (ID: 1) would be promoted to primary; current primary "postgresql_06_70" (ID: 3) would be demoted to standby
   INFO: following shutdown command would be run on node "postgresql_06_70":
     "/usr/local/postgresql/bin/pg_ctl  -D '/data/postgresql/pgdata' -W -m fast stop"
   INFO: prerequisites for executing STANDBY SWITCHOVER are met
   
   
   $ repmgr -f /usr/local/postgresql/repmgr.conf standby switchover --siblings-follow --force-rewind
   NOTICE: executing switchover on node "postgresql_04_68" (ID: 1)
   NOTICE: local node "postgresql_04_68" (ID: 1) will be promoted to primary; current primary "postgresql_06_70" (ID: 3) will be demoted to standby
   NOTICE: stopping current primary node "postgresql_06_70" (ID: 3)
   NOTICE: issuing CHECKPOINT
   DETAIL: executing server command "/usr/local/postgresql/bin/pg_ctl  -D '/data/postgresql/pgdata' -W -m fast stop"
   INFO: checking for primary shutdown; 1 of 60 attempts ("shutdown_check_timeout")
   INFO: checking for primary shutdown; 2 of 60 attempts ("shutdown_check_timeout")
   NOTICE: current primary has been cleanly shut down at location 0/B000028
   NOTICE: promoting standby to primary
   DETAIL: promoting server "postgresql_04_68" (ID: 1) using pg_promote()
   NOTICE: waiting up to 60 seconds (parameter "promote_check_timeout") for promotion to complete
   NOTICE: STANDBY PROMOTE successful
   DETAIL: server "postgresql_04_68" (ID: 1) was successfully promoted to primary
   NOTICE: issuing CHECKPOINT
   INFO: local node 3 can attach to rejoin target node 1
   DETAIL: local node's recovery point: 0/B000028; rejoin target node's fork point: 0/B0000A0
   NOTICE: setting node 3's upstream to node 1
   WARNING: unable to ping "host=192.168.10.70 user=repmgr password=repmgr dbname=repmgr connect_timeout=2"
   DETAIL: PQping() returned "PQPING_NO_RESPONSE"
   NOTICE: starting server using "/usr/local/postgresql/bin/pg_ctl  -w -D '/data/postgresql/pgdata' start"
   NOTICE: NODE REJOIN successful
   DETAIL: node 3 is now attached to node 1
   NOTICE: executing STANDBY FOLLOW on 2 of 2 siblings
   INFO: STANDBY FOLLOW successfully executed on all reachable sibling nodes
   NOTICE: switchover was successful
   DETAIL: node "postgresql_04_68" is now primary and node "postgresql_06_70" is attached as standby
   NOTICE: STANDBY SWITCHOVER has completed successfully
   
   #查看状态 68又变回了主库
   $ repmgr -f /usr/local/postgresql/repmgr.conf cluster show
    ID | Name             | Role    | Status    | Upstream         | Location | Priority | Timeline | Connection string                                                             
   ----+------------------+---------+-----------+------------------+----------+----------+----------+--------------------------------------------------------------------------------
    1  | postgresql_04_68 | primary | * running |                  | default  | 100      | 5        | host=192.168.10.68 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    2  | postgresql_05_69 | standby |   running | postgresql_04_68 | default  | 100      | 4        | host=192.168.10.69 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    3  | postgresql_06_70 | standby |   running | postgresql_04_68 | default  | 100      | 4        | host=192.168.10.70 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    4  | postgresql_07_71 | witness | * running | postgresql_04_68 | default  | 0        | 1        | host=192.168.10.71 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
   
   ```

# 异常情况主从自动切换测试-failover

1. 启用repmgrd进程（自动切换，需要启用repmgrd）,68 69 70配置shared_preload_libraries='repmgr' 配置

   ```
   $ vim /data/postgresql/pgdata/postgresql.conf
   ……
   # 修改shared_preload_libraries
   shared_preload_libraries = 'repmgr' 
   ```

2. 修改完成后重启68 69 70三个节点执行

   ```
   $ pg_ctl stop
   waiting for server to shut down.... done
   server stopped
   $ pg_ctl -D /data/postgresql/pgdata start
   waiting for server to start....2021-02-23 14:32:53.041 CST [5698] LOG:  starting PostgreSQL 12.6 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 4.8.5 20150623 (Red Hat 4.8.5-44), 64-bit
   2021-02-23 14:32:53.042 CST [5698] LOG:  listening on IPv4 address "0.0.0.0", port 5432
   2021-02-23 14:32:53.042 CST [5698] LOG:  listening on IPv6 address "::", port 5432
   2021-02-23 14:32:53.062 CST [5698] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5432"
   2021-02-23 14:32:53.159 CST [5698] LOG:  redirecting log output to logging collector process
   2021-02-23 14:32:53.159 CST [5698] HINT:  Future log output will appear in directory "/data/postgresql/logs".
    done
   server started
   ```

3. 配置repmgrd自动failover,68 69 70三个节点执行

   ```
   # vim /usr/local/postgresql/repmgr.conf
   node_id=1
   node_name=postgresql_04_68
   conninfo='host=192.168.10.68 user=repmgr password=repmgr dbname=repmgr connect_timeout=2'
   data_directory='/data/postgresql/pgdata'
   pg_bindir='/usr/local/postgresql/bin'
   
   #添加如下
   monitoring_history=yes   #监控集群的所有节点复制状态
   monitor_interval_secs=5	 #监控时间默认2s
   failover=automatic       #是否设置为自动切换模式
   reconnect_attempts=6     #当主连接异常的时候，尝试连接次数
   reconnect_interval=5     # 当主连接异常的时候，尝试连接次数 每次连接间隔为5s
   promote_command='repmgr standby promote -f /usr/local/postgresql/repmgr.conf --log-to-file' #当本机提升为主库的命令
   follow_command='repmgr standby follow -f /usr/local/postgresql/repmgr.conf --log-to-file --upstream-node-id=%n' #当出现新的主库执行的命令
   log_level=INFO           # 日志级别
   log_status_interval=10  #写日志间隔时间10s
   log_file='/usr/local/postgresql/log/repmgr.log' #日志配置文件 需要创建这么一个配置文件
   
   # mkdir /usr/local/postgresql/log/
   # chown pgsql:pgsql -R /usr/local/postgresql
   ```

4. 配置repmgr.log日志文件，logrotate.conf末尾添加配置信息，68 69 70三个节点执行

   ```
   # vim /etc/logrotate.conf
   ……
   /usr/local/postgresql/log/repmgr.log {
       missingok
       compress
       rotate 30
       daily
       dateext
       create 0600 pgsql pgsql
   }
   
   ```

5. 启动repmgrd进程，68 69 70三个节点执行

   ```
   # su - pgsql
   $ repmgrd -f /usr/local/postgresql/repmgr.conf --pid-file /tmp/repmgrd.pid --daemonize
   [2021-02-23 14:48:37] [NOTICE] redirecting logging output to "/usr/local/postgresql/log/repmgr.log"
   
   #查看日志是否启动成功
   $ tail -100f /usr/local/postgresql/log/repmgr.log
   [2021-02-23 14:48:37] [NOTICE] repmgrd (repmgrd 5.0.0) starting up
   [2021-02-23 14:48:37] [INFO] connecting to database "host=192.168.10.68 user=repmgr password=repmgr dbname=repmgr connect_timeout=2"
   INFO:  set_repmgrd_pid(): provided pidfile is /tmp/repmgrd.pid
   [2021-02-23 14:48:37] [NOTICE] starting monitoring of node "postgresql_04_68" (ID: 1)
   [2021-02-23 14:48:37] [INFO] "connection_check_type" set to "ping"
   [2021-02-23 14:48:37] [NOTICE] monitoring cluster primary "postgresql_04_68" (ID: 1)
   [2021-02-23 14:48:37] [INFO] child node "postgresql_06_70" (ID: 3) is attached
   [2021-02-23 14:48:37] [INFO] child node "postgresql_05_69" (ID: 2) is attached
   [2021-02-23 14:48:37] [INFO] child node "postgresql_07_71" (ID: 4) is not yet attached
   [2021-02-23 14:48:47] [INFO] monitoring primary node "postgresql_04_68" (ID: 1) in normal state
   
   #也可以写到开机启动里
   echo "repmgrd -f /usr/local/postgresql/repmgr.conf --pid-file /tmp/repmgrd.pid --daemonize" >> /etc/rc.local
   #停止repmgrd进程，直接kill掉
   kill `cat /tmp/repmgrd.pid`
   ```

6. 模拟测试,停掉68主库，集群会自动选举主库

   ```
   # 68上执行
   $ pg_ctl stop
   
   # 任意一个节点查看日志，每5秒执行一次连接主库检查一共执行6次 还连不上了后重新选举主库，可以看到最终选举了69为主库(attempting to follow new primary "postgresql_05_69" (node ID: 2))
   $ tail -100f /usr/local/postgresql/log/repmgr.log
   [2021-02-23 14:50:43] [NOTICE] repmgrd (repmgrd 5.0.0) starting up
   [2021-02-23 14:50:43] [INFO] connecting to database "host=192.168.10.70 user=repmgr password=repmgr dbname=repmgr connect_timeout=2"
   INFO:  set_repmgrd_pid(): provided pidfile is /tmp/repmgrd.pid
   [2021-02-23 14:50:43] [NOTICE] starting monitoring of node "postgresql_06_70" (ID: 3)
   [2021-02-23 14:50:43] [INFO] "connection_check_type" set to "ping"
   [2021-02-23 14:50:43] [INFO] monitoring connection to upstream node "postgresql_04_68" (ID: 1)
   [2021-02-23 14:50:53] [INFO] node "postgresql_06_70" (ID: 3) monitoring upstream node "postgresql_04_68" (ID: 1) in normal state
   [2021-02-23 14:50:53] [DETAIL] last monitoring statistics update was 5 seconds ago
   [2021-02-23 14:51:04] [INFO] node "postgresql_06_70" (ID: 3) monitoring upstream node "postgresql_04_68" (ID: 1) in normal state
   [2021-02-23 14:51:04] [DETAIL] last monitoring statistics update was 5 seconds ago
   [2021-02-23 14:51:14] [INFO] node "postgresql_06_70" (ID: 3) monitoring upstream node "postgresql_04_68" (ID: 1) in normal state
   [2021-02-23 14:51:14] [DETAIL] last monitoring statistics update was 5 seconds ago
   [2021-02-23 14:51:24] [INFO] node "postgresql_06_70" (ID: 3) monitoring upstream node "postgresql_04_68" (ID: 1) in normal state
   [2021-02-23 14:51:24] [DETAIL] last monitoring statistics update was 5 seconds ago
   [2021-02-23 14:51:34] [INFO] node "postgresql_06_70" (ID: 3) monitoring upstream node "postgresql_04_68" (ID: 1) in normal state
   [2021-02-23 14:51:34] [DETAIL] last monitoring statistics update was 5 seconds ago
   [2021-02-23 14:51:44] [INFO] node "postgresql_06_70" (ID: 3) monitoring upstream node "postgresql_04_68" (ID: 1) in normal state
   [2021-02-23 14:51:44] [DETAIL] last monitoring statistics update was 5 seconds ago
   [2021-02-23 14:51:54] [INFO] node "postgresql_06_70" (ID: 3) monitoring upstream node "postgresql_04_68" (ID: 1) in normal state
   [2021-02-23 14:51:54] [DETAIL] last monitoring statistics update was 5 seconds ago
   [2021-02-23 14:52:04] [INFO] node "postgresql_06_70" (ID: 3) monitoring upstream node "postgresql_04_68" (ID: 1) in normal state
   [2021-02-23 14:52:04] [DETAIL] last monitoring statistics update was 5 seconds ago
   [2021-02-23 14:52:14] [INFO] node "postgresql_06_70" (ID: 3) monitoring upstream node "postgresql_04_68" (ID: 1) in normal state
   [2021-02-23 14:52:14] [DETAIL] last monitoring statistics update was 5 seconds ago
   [2021-02-23 14:52:24] [WARNING] unable to ping "host=192.168.10.68 user=repmgr password=repmgr dbname=repmgr connect_timeout=2"
   [2021-02-23 14:52:24] [DETAIL] PQping() returned "PQPING_NO_RESPONSE"
   [2021-02-23 14:52:24] [WARNING] unable to connect to upstream node "postgresql_04_68" (ID: 1)
   [2021-02-23 14:52:24] [INFO] checking state of node 1, 1 of 6 attempts
   [2021-02-23 14:52:24] [WARNING] unable to ping "user=repmgr password=repmgr connect_timeout=2 dbname=repmgr host=192.168.10.68 fallback_application_name=repmgr"
   [2021-02-23 14:52:24] [DETAIL] PQping() returned "PQPING_NO_RESPONSE"
   [2021-02-23 14:52:24] [INFO] sleeping 5 seconds until next reconnection attempt
   [2021-02-23 14:52:29] [INFO] checking state of node 1, 2 of 6 attempts
   [2021-02-23 14:52:29] [WARNING] unable to ping "user=repmgr password=repmgr connect_timeout=2 dbname=repmgr host=192.168.10.68 fallback_application_name=repmgr"
   [2021-02-23 14:52:29] [DETAIL] PQping() returned "PQPING_NO_RESPONSE"
   [2021-02-23 14:52:29] [INFO] sleeping 5 seconds until next reconnection attempt
   [2021-02-23 14:52:34] [INFO] checking state of node 1, 3 of 6 attempts
   [2021-02-23 14:52:34] [WARNING] unable to ping "user=repmgr password=repmgr connect_timeout=2 dbname=repmgr host=192.168.10.68 fallback_application_name=repmgr"
   [2021-02-23 14:52:34] [DETAIL] PQping() returned "PQPING_NO_RESPONSE"
   [2021-02-23 14:52:34] [INFO] sleeping 5 seconds until next reconnection attempt
   [2021-02-23 14:52:39] [INFO] checking state of node 1, 4 of 6 attempts
   [2021-02-23 14:52:39] [WARNING] unable to ping "user=repmgr password=repmgr connect_timeout=2 dbname=repmgr host=192.168.10.68 fallback_application_name=repmgr"
   [2021-02-23 14:52:39] [DETAIL] PQping() returned "PQPING_NO_RESPONSE"
   [2021-02-23 14:52:39] [INFO] sleeping 5 seconds until next reconnection attempt
   [2021-02-23 14:52:44] [INFO] checking state of node 1, 5 of 6 attempts
   [2021-02-23 14:52:44] [WARNING] unable to ping "user=repmgr password=repmgr connect_timeout=2 dbname=repmgr host=192.168.10.68 fallback_application_name=repmgr"
   [2021-02-23 14:52:44] [DETAIL] PQping() returned "PQPING_NO_RESPONSE"
   [2021-02-23 14:52:44] [INFO] sleeping 5 seconds until next reconnection attempt
   [2021-02-23 14:52:49] [INFO] checking state of node 1, 6 of 6 attempts
   [2021-02-23 14:52:49] [WARNING] unable to ping "user=repmgr password=repmgr connect_timeout=2 dbname=repmgr host=192.168.10.68 fallback_application_name=repmgr"
   [2021-02-23 14:52:49] [DETAIL] PQping() returned "PQPING_NO_RESPONSE"
   [2021-02-23 14:52:49] [WARNING] unable to reconnect to node 1 after 6 attempts
   [2021-02-23 14:52:49] [INFO] 2 active sibling nodes registered
   [2021-02-23 14:52:49] [INFO] primary and this node have the same location ("default")
   [2021-02-23 14:52:49] [INFO] local node's last receive lsn: 0/D0000A0
   [2021-02-23 14:52:49] [INFO] checking state of sibling node "postgresql_05_69" (ID: 2)
   [2021-02-23 14:52:49] [WARNING] node "postgresql_05_69" (ID: 2) is not in recovery
   [2021-02-23 14:52:49] [INFO] local node "postgresql_06_70" (ID: 3) can attach to follow target node "postgresql_05_69" (ID: 2)
   [2021-02-23 14:52:49] [DETAIL] local node's recovery point: 0/D0000A0; follow target node's fork point: 0/D0000A0
   [2021-02-23 14:52:49] [INFO] follower node intending to follow new primary 2
   [2021-02-23 14:52:49] [NOTICE] attempting to follow new primary "postgresql_05_69" (node ID: 2)
   [2021-02-23 14:52:49] [NOTICE] redirecting logging output to "/usr/local/postgresql/log/repmgr.log"
   
   [2021-02-23 14:52:49] [INFO] local node 3 can attach to follow target node 2
   [2021-02-23 14:52:49] [DETAIL] local node's recovery point: 0/D0000A0; follow target node's fork point: 0/D0000A0
   [2021-02-23 14:52:49] [NOTICE] setting node 3's upstream to node 2
   [2021-02-23 14:52:49] [NOTICE] stopping server using "/usr/local/postgresql/bin/pg_ctl  -D '/data/postgresql/pgdata' -w -m fast stop"
   [2021-02-23 14:52:54] [NOTICE] starting server using "/usr/local/postgresql/bin/pg_ctl  -w -D '/data/postgresql/pgdata' start"
   [2021-02-23 14:52:54] [NOTICE] STANDBY FOLLOW successful
   [2021-02-23 14:52:54] [DETAIL] standby attached to upstream node "postgresql_05_69" (ID: 2)
   INFO:  set_repmgrd_pid(): provided pidfile is /tmp/repmgrd.pid
   [2021-02-23 14:52:54] [NOTICE] node 3 now following new upstream node 2
   [2021-02-23 14:52:54] [INFO] resuming standby monitoring mode
   [2021-02-23 14:52:54] [DETAIL] following new primary "postgresql_05_69" (ID: 2)
   
   
   # 69或者70上执行查看集群状态
   $ repmgr -f /usr/local/postgresql/repmgr.conf cluster show
    ID | Name             | Role    | Status    | Upstream           | Location | Priority | Timeline | Connection string                                                             
   ----+------------------+---------+-----------+--------------------+----------+----------+----------+--------------------------------------------------------------------------------
    1  | postgresql_04_68 | primary | - failed  |                    | default  | 100      | ?        | host=192.168.10.68 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    2  | postgresql_05_69 | primary | * running |                    | default  | 100      | 6        | host=192.168.10.69 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    3  | postgresql_06_70 | standby |   running | postgresql_05_69   | default  | 100      | 5        | host=192.168.10.70 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    4  | postgresql_07_71 | witness | * running | ? postgresql_04_68 | default  | 0        | 1        | host=192.168.10.71 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
   
   WARNING: following issues were detected
     - unable to connect to node "postgresql_04_68" (ID: 1)
     - unable to connect to node "postgresql_07_71" (ID: 4)'s upstream node "postgresql_04_68" (ID: 1)
   
   ```

7. 修复68主库，并且同步为新的主库，重复执行上面的异常情况主从手动切换测试流程(步骤:4-6)，全部在68上执行

   ```
   $ rm -rf /data/postgresql/pgdata/
   $ repmgr -f /usr/local/postgresql/repmgr.conf node rejoin -d 'host=192.168.10.69 user=repmgr dbname=repmgr connect_timeout=2' --force-rewind --dry-run --verbose
   # 68为69的备库
   $ repmgr -f /usr/local/postgresql/repmgr.conf node rejoin -d 'host=192.168.10.69 user=repmgr dbname=repmgr connect_timeout=2' --force-rewind --verbose  
   # 68上再次提升为主库
   $ repmgr -f /usr/local/postgresql/repmgr.conf standby switchover --siblings-follow --force-rewind
   # 最后查看状态，68位主
   $ repmgr -f /usr/local/postgresql/repmgr.conf cluster show
    ID | Name             | Role    | Status    | Upstream         | Location | Priority | Timeline | Connection string                                                             
   ----+------------------+---------+-----------+------------------+----------+----------+----------+--------------------------------------------------------------------------------
    1  | postgresql_04_68 | primary | * running |                  | default  | 100      | 7        | host=192.168.10.68 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    2  | postgresql_05_69 | standby |   running | postgresql_04_68 | default  | 100      | 6        | host=192.168.10.69 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    3  | postgresql_06_70 | standby |   running | postgresql_04_68 | default  | 100      | 6        | host=192.168.10.70 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
    4  | postgresql_07_71 | witness | * running | postgresql_04_68 | default  | 0        | 1        | host=192.168.10.71 user=repmgr password=repmgr dbname=repmgr connect_timeout=2
   ```

8. 测试数据

   ```
   $ psql -h 192.168.10.68 -p 5432 -U pgtest -d testdb
   Password for user pgtest: 
   psql (12.6)
   Type "help" for help.
   
   testdb=> insert into test1 values('test3');
   INSERT 0 1
   testdb=> select * from test1;
    name  
   -------
    test1
    test2
    test3
   (3 rows)
   
   
   $ psql -h 192.168.10.69 -p 5432 -U pgtest -d testdb
   Password for user pgtest: 
   psql (12.6)
   Type "help" for help.
   
   testdb=> select * from test1;
    name  
   -------
    test1
    test2
    test3
   (3 rows)
   
   ```

   

# 备份恢复

1. psql导入

   ```
   #导出数据
   $ pg_dump -U pgtest testdb -h 127.0.0.1 > testdb.sql
   
   #psql 导入数据
   #先创建数据库
   $ createdb testdb -h 127.0.0.1
   #然后再导入
   $ psql -d testdb -U pgtest -f testdb.sql -h 127.0.0.1
   ```

2. pg_restore导入

   ```
   #导出数据
   $ pg_dump -U pgtest -Fc testdb -h 127.0.0.1 > testdb.dump
   #先创建数据库
   $ createdb testdb -h 127.0.0.1
   #然后再导入
   $ pg_restore -d testdb -U pgtest -h 127.0.0.1 testdb.dump
   ```

3. pg_dumpall导出所有数据库和配置

   ```
   $ pg_dumpall -h 127.0.0.1  > pg_all.sql
   $ pg_dumpall -h -t 127.0.0.1  > pg_all.sql  #-t 添加权限和表空间
   # 也可以通过pgsql导入全部
   $ psql -f pg_all.sql -h 127.0.0.1
   ```

4. pg_rman安装

   ```
   # cd /usr/local/src/
   # tar zxvf pg_rman-1.3.11-pg12.tar.gz
   # cd pg_rman-1.3.11-pg12
   # make -j
   # make install  #会自动安装在pg的目录下
   # pg_rman --version
   pg_rman 1.3.11
   ```

5. 配置一个pg_rman备份目录的环境变量

   ```
   # mkdir -p /data/backup/pg_rman/
   # chown pgsql:pgsql -R /data/backup/pg_rman/
   
   # vim /etc/profile
   #添加如下
   export BACKUP_PATH=/data/backup/pg_rman/
   
   # source /etc/profile
   ```

6. 初始化备份目录

   ```
   # su - pgsql
   $ pg_rman init -B /data/backup/pg_rman/
   INFO: ARCLOG_PATH is set to '/data/postgresql/archiv' #这个目录有问题。不存在 注意修改 这俩目录影响恢复成功后启动不起来
   INFO: SRVLOG_PATH is set to '/data/postgresql/logs/'
   $ ll /data/backup/pg_rman/
   total 8
   drwx------ 4 pgsql pgsql 34 Feb 23 15:58 backup
   -rw-rw-r-- 1 pgsql pgsql 76 Feb 23 15:58 pg_rman.ini
   -rw-rw-r-- 1 pgsql pgsql 40 Feb 23 15:58 system_identifier
   drwx------ 2 pgsql pgsql  6 Feb 23 15:58 timeline_history
   
   #如果上面的俩目录有问题，则在pg_rman.ini上修改
   $ vim pg_rman.ini
   ARCLOG_PATH='/data/postgresql/archive'
   SRVLOG_PATH='/data/postgresql/logs/'
   ```

7. 全备

   ```
   $ pg_rman backup -b full -h 127.0.0.1
   $ pg_rman backup --backup-mode=full --with-serverlog -h 127.0.0.1
   INFO: copying database files
   INFO: copying archived WAL files
   INFO: backup complete
   INFO: Please execute 'pg_rman validate' to verify the files are correctly copied.
   
   $ pg_rman validate
   INFO: validate: "2021-02-23 15:59:51" backup and archive log files by CRC
   INFO: backup "2021-02-23 15:59:51" is valid
   ```

8. 增备

   ```
   $ pg_rman backup -b incremental -h 127.0.0.1
   INFO: copying database files
   INFO: copying archived WAL files
   INFO: backup complete
   INFO: Please execute 'pg_rman validate' to verify the files are correctly copied.
   
   $ pg_rman validate
   INFO: validate: "2021-02-23 16:01:22" backup and archive log files by CRC
   INFO: backup "2021-02-23 16:01:22" is valid
   ```

9. 查看备份记录

   ```
   $ pg_rman show
   =====================================================================
    StartTime           EndTime              Mode    Size   TLI  Status 
   =====================================================================
   2021-02-23 16:01:22  2021-02-23 16:01:25  INCR    33kB     7  OK
   2021-02-23 15:59:51  2021-02-23 15:59:53  FULL    28MB     7  OK
   
   #查看目录也多了相应的目录文件
   $ ll /data/backup/pg_rman
   total 8
   drwx------ 6 pgsql pgsql 62 Feb 23 16:01 20210223
   drwx------ 4 pgsql pgsql 34 Feb 23 15:58 backup
   -rw-rw-r-- 1 pgsql pgsql 76 Feb 23 15:58 pg_rman.ini
   -rw-rw-r-- 1 pgsql pgsql 40 Feb 23 15:58 system_identifier
   drwx------ 2 pgsql pgsql  6 Feb 23 15:58 timeline_history
   
   ```

10. 备份恢复，如果恢复失败，查看log下的日志，注意postgresql.conf文件的最后两行的目录是否存在

    ```
    # 先停掉所有数据库
    $ pg_ctl stop
    $ rm -rf /data/postgresql/pgdata/
    
    #恢复
    $ pg_rman restore;
    WARNING: pg_controldata file "/data/postgresql/pgdata/global/pg_control" does not exist
    INFO: the recovery target timeline ID is not given
    INFO: use timeline ID of latest full backup as recovery target: 7
    INFO: calculating timeline branches to be used to recovery target point
    INFO: searching latest full backup which can be used as restore start point
    INFO: found the full backup can be used as base in recovery: "2021-02-23 15:59:51"
    INFO: copying online WAL files and server log files
    INFO: clearing restore destination
    INFO: validate: "2021-02-23 15:59:51" backup and archive log files by SIZE
    INFO: backup "2021-02-23 15:59:51" is valid
    INFO: restoring database files from the full mode backup "2021-02-23 15:59:51"
    INFO: searching incremental backup to be restored
    INFO: validate: "2021-02-23 16:01:22" backup and archive log files by SIZE
    INFO: backup "2021-02-23 16:01:22" is valid
    INFO: restoring database files from the incremental mode backup "2021-02-23 16:01:22"
    INFO: searching backup which contained archived WAL files to be restored
    INFO: backup "2021-02-23 16:01:22" is valid
    INFO: restoring WAL files from backup "2021-02-23 16:01:22"
    INFO: restoring online WAL files and server log files
    INFO: add recovery related options to postgresql.conf
    INFO: generating recovery.signal
    INFO: restore complete
    HINT: Recovery will start automatically when the PostgreSQL server is started.
    
    #启动
    $ pg_ctl -D /data/postgresql/pgdata start
    ```

11. 删除备份集

    ```
    $ pg_rman show
    =====================================================================
     StartTime           EndTime              Mode    Size   TLI  Status 
    =====================================================================
    2021-02-23 16:01:22  2021-02-23 16:01:25  INCR    33kB     7  OK
    2021-02-23 16:01:17  2021-02-23 16:01:17  INCR      0B     0  ERROR
    2021-02-23 15:59:51  2021-02-23 15:59:53  FULL    28MB     7  OK
    2021-02-23 15:59:41  2021-02-23 15:59:41  FULL      0B     0  ERROR
    
    $ pg_rman delete 2021-02-23 15:59:41
    INFO: delete the backup with start time: "2021-02-23 15:59:41"
    
    $ pg_rman show
    =====================================================================
     StartTime           EndTime              Mode    Size   TLI  Status 
    =====================================================================
    2021-02-23 16:01:22  2021-02-23 16:01:25  INCR    33kB     7  OK
    2021-02-23 16:01:17  2021-02-23 16:01:17  INCR      0B     0  ERROR
    2021-02-23 15:59:51  2021-02-23 15:59:53  FULL    28MB     7  OK
    
    ```

12. 清空备份集,如果数据做过恢复。怎不允许清空

    ```
    $ pg_rman purge
    ```

# PostgreSQL数据库监控分析工具

* pgAdmin :开源PostgreSQL图形界面管理工具
* psql :执行查询命令行的工具
* pgcluu 性能监控和审计工具
* pgstatsinfo :数据库监控与信息收集工具
* pgwatch2 :数据库监控与信息收集工具
* pgsnap :提供的性能统计数据
* pgstatspack :提供的性能统计数据
* pgcenter :实时top类工具
* pgtop :实时top类工具
* pg_activity :用于PostgreSQL服务器活动监控的命令行工具
  pgbadger :对postgresql的日志进行分析,并且以网页的方式信息展示 



