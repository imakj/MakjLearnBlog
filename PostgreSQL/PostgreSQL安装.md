# 环境

* centos7  4台 
  * 192.168.10.65
  * 192.168.10.66
  * 192.168.10.67
* postgresql源码包：postgresql-12.6.tar.gz

# 系统变量准备

1. 所有机器修改主机名

   ```
   # 192.168.10.65修改
   # hostnamectl set-hostname postgreSQL_01_65
   # 192.168.10.66修改
   # hostnamectl set-hostname postgreSQL_02_66
   # 192.168.10.67修改
   # hostnamectl set-hostname postgreSQL_03_67
   ```
   
2. 所有机器修改host

   ```
   vim /etc/hosts
   192.168.10.65 postgreSQL_01_65
   192.168.10.66 postgreSQL_02_66
   192.168.10.67 postgreSQL_03_67
   192.168.10.68 postgreSQL_04_68
   ```

3. 从官网下载安装包

   网站：https://www.postgresql.org/ftp/source/

   ```
   # https://ftp.postgresql.org/pub/source/v12.6/postgresql-12.6.tar.gz
   ```

4. 所有节点执行安装依赖

   ```
   # yum install -y cmake make gcc zlib gcc-c++ perl readline readline-devel zlib zlib-devel perl python36 tcl openssl ncurses-devel openldap pam
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
   # chmod -R 775 /data/postgresql
   # chmod 775 /usr/local/src/postgresql-12.6.tar.gz
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
   export PATH
   export LANG=en_US.UTF8
   # export PS1="[`whoami`@`hostname`:"'$PWD]\$ '
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
   ```

7. 修改访问权限配置文件（也就是防火墙）

   ```
   $ cd /data/postgresql/pgdata/
   $ vim pg_hba.conf
   #最后一行添加，也就是所有人都可以用密码进来
   host    all     all     0.0.0.0/0       md5
   ```

8. 启动 至此 环境安装完成

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

9. 停止

   ```
   $ /usr/local/postgresql/bin/pg_ctl -D /data/postgresql/pgdata stop
   ```

11. 设置密码（通过127.0.0.1登录设置密码，前面防火墙已经设置通过127.0.0.1登录不需要密码）

    ```\
    $psql -h 127.0.0.1 -p 5432 -U postgres
    # 设置postgres的密码为postgres
    postgres=# \password postgres
    # 退出
    postgres-# \q
    # 通过ip登录就可以成功
    $ psql -h 192.168.10.65 -p 5432
    ```

12. 创建用户

    ```
    $ psql -h 192.168.10.65 -p 5432
    postgres=# create user pgtest with password 'pgtest' nocreatedb;
    CREATE ROLE
    postgres=# create database testdb with owner=pgtest;
    CREATE DATABASE
    postgres=# quit
    
    #登录
    $ psql -h 192.168.10.65 -p 5432 -U pgtest -d testdb
    ```

13. 创建表

    ```
    #登录
    $ psql -h 192.168.10.65 -p 5432 -U pgtest -d testdb
    testdb=> create table test (name varchar(50));
    CREATE TABLE
    testdb=> insert into test values('test1');
    INSERT 0 1
    testdb=> insert into test values('test2');
    INSERT 0 1
    testdb=> insert into test values('test3');
    INSERT 0 1
    testdb=> select * from test;
    name  
    -------
     test1
     test2
     test3
    (3 rows)
    ```

14. 切换数据库

    ```
    #登录
    $ psql -h 192.168.10.65 -p 5432
    #切换命令 \c 目标数据库 目标用户名
    testdb=> \c testdb pgtest
    #查看表结构
    testdb=> \dt test;
    ```

# 高可用方案

* 主从复制
  * postgreSQL+RHCS  红帽官方的方案
  * postgreSQL+Pacemaker+Corosync  开源方案
  * postgreSQL+keepalived
  * postgreSQL+repmgr 相当于MHA 可以支持1主多从 可以加一个观察服务器，解决同时出现主库出现的脑裂问题
  * postgreSQL+pgpool 这个方案bug有点多，但是可以实现读写分离，读的负载均衡，连接池，自动failover等
  * postgreSQL+mycat
* 高可用
  * PostgreSQL+ pg_ shard
  * PostgreSQL+ pl/proxy
  * PostgreSQL +FDWV
  * PostgreSQL+ mycat
  * PostgreSQL +Citus
  * PostgreSQL+ Postgres-XL
  * PostgreSQL + Greenplum  就是这个方案吧Greenplum  带火的 可以支持到PB级别数据

# 主从复制

* 注意：主从复制在链接不上备库的情况下,主库会挂起等待
* PostgreSQL支持物理复制(流复制)、逻辑复制(逻辑订阅)。
* 物理复制(流复制) :可以从实例级复制出一一个与主库一模一样的实例级的从库，流复制同步方式有同步、异步两种。
* 物理复制优点:
  * 物理层面完全-致，是主要的复制方式,其类似于Oracle的DG。
  * 延迟低,事务执行过程中产生REDO record , 实时的在备库apply ,事务结束时,备库立马能见到数据。
  * 物理复制的一致性、可靠性高,不必担心数据逻辑层面不一致。
* 物理复制缺点:
  * 无法满足不同的版本之间、不同库名之间的表同步。
  *  无法满足指定库或部分表的复制需求
  *  无法满足将多个数据库实例同步到一个库,将一个库的数据分 发到多个不同的库。
  *  物理复制场景:
    *  适合于单向同步。
    *  适合于任意事务,任意密度写(重度写)的同步。
    *  适合于HA、容灾、读写分离。
    *  适合于备库没有写,只有读的场景。

1. 修改master(65服务器)配置文件,修改添加如下

   ```
   # vim /data/postgresql/pgdata/postgresql.conf
   archive_mode = on  #启用归档模式
   archive_command = 'test ! -f /data/postgresql/archive%f && cp %p /data/postgresql/archive/%f'  #归档路径
   wal_level = replica #日志复制模式
   max_wal_senders = 10
   wal_keep_segments = 0
   wal_sender_timeout = 60s
   ```

2. 改完后重启

   ```
   $ pg_ctl stop
   waiting for server to shut down.... done
   server stopped
   $ pg_ctl start
   waiting for server to start....2021-02-19 22:11:01.661 CST [28050] LOG:  starting PostgreSQL 12.6 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 4.8.5 20150623 (Red Hat 4.8.5-44), 64-bit
   2021-02-19 22:11:01.661 CST [28050] LOG:  listening on IPv4 address "0.0.0.0", port 5432
   2021-02-19 22:11:01.661 CST [28050] LOG:  listening on IPv6 address "::", port 5432
   2021-02-19 22:11:01.673 CST [28050] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5432"
   2021-02-19 22:11:01.784 CST [28050] LOG:  redirecting log output to logging collector process
   2021-02-19 22:11:01.784 CST [28050] HINT:  Future log output will appear in directory "/data/postgresql/logs".
    done
   server started
   
   # 查看有没有归档进程
   $ ps -ef|grep postgre
   pgsql    28050     1  0 22:11 ?        00:00:00 /usr/local/postgresql/bin/postgres
   pgsql    28051 28050  0 22:11 ?        00:00:00 postgres: logger   
   pgsql    28053 28050  0 22:11 ?        00:00:00 postgres: checkpointer   
   pgsql    28054 28050  0 22:11 ?        00:00:00 postgres: background writer   
   pgsql    28055 28050  0 22:11 ?        00:00:00 postgres: walwriter   
   pgsql    28056 28050  0 22:11 ?        00:00:00 postgres: autovacuum launcher   
   pgsql    28057 28050  0 22:11 ?        00:00:00 postgres: archiver    //归档进程
   pgsql    28058 28050  0 22:11 ?        00:00:00 postgres: stats collector   
   pgsql    28059 28050  0 22:11 ?        00:00:00 postgres: logical replication launcher  
   pgsql    28063 11401  0 22:11 pts/0    00:00:00 grep --color=auto postgre
   ```

3. 进入数据库查看归档是否打开,可以看到 已经全部打开

   ```
   $ psql -h 127.0.0.1 -p 5432
   psql (12.6)
   Type "help" for help.
   
   postgres=# show wal_level;
    wal_level 
   -----------
    replica
   (1 row)
   
   postgres=# show archive_mode;
    archive_mode 
   --------------
    on
   (1 row)
   
   postgres=# show archive_command;
                                  archive_command                                
   ------------------------------------------------------------------------------
    test ! -f /data/postgresql/archive/%f && cp %p /data/postgresql/archive/%f
   (1 row)
   
   #切换归档
   postgres=# select pg_switch_wal();
    pg_switch_wal 
   ---------------
    0/1650D50
   (1 row)
   
   #$ ps -ef|grep postgre
   pgsql    28198     1  1 22:19 ?        00:00:00 /usr/local/postgresql/bin/postgres
   pgsql    28199 28198  0 22:19 ?        00:00:00 postgres: logger   
   pgsql    28201 28198  0 22:19 ?        00:00:00 postgres: checkpointer   
   pgsql    28202 28198  0 22:19 ?        00:00:00 postgres: background writer   
   pgsql    28203 28198  0 22:19 ?        00:00:00 postgres: walwriter   
   pgsql    28204 28198  0 22:19 ?        00:00:00 postgres: autovacuum launcher   
   pgsql    28205 28198  0 22:19 ?        00:00:00 postgres: archiver   last was 000000010000000000000004
   pgsql    28206 28198  0 22:19 ?        00:00:00 postgres: stats collector   
   pgsql    28207 28198  0 22:19 ?        00:00:00 postgres: logical replication launcher  
   pgsql    28217 11401  0 22:19 pts/0    00:00:00 grep --color=auto postgre
   $ ll /data/postgresql/archive/
   total 65536
   -rw------- 1 pgsql pgsql 16777216 Feb 19 22:19 000000010000000000000001
   -rw------- 1 pgsql pgsql 16777216 Feb 19 22:19 000000010000000000000002
   -rw------- 1 pgsql pgsql 16777216 Feb 19 22:19 000000010000000000000003
   -rw------- 1 pgsql pgsql 16777216 Feb 19 22:19 000000010000000000000004
   
   ```

4. 创建流复制的角色,注意防火墙是否开启(pg_hba.conf是否添加host all all 0.0.0.0/0 md5)

   ```
   $ psql -h 127.0.0.1 -p 5432
   psql (12.6)
   Type "help" for help.
   # 注意后面的replication权限
   postgres=# create role repuser login encrypted password 'repuser123' replication;
   CREATE ROLE
   # 查看权限
   postgres=# \du
                                      List of roles
    Role name |                         Attributes                         | Member of 
   -----------+------------------------------------------------------------+-----------
    pgtest    |                                                            | {}
    postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
    repuser   | Replication
   ```

5. 添加流复制权限,

   ```
   $ vim pg_hba.conf
   #最后一行添加
   host replication repuser 0.0.0.0/0 md5
   #重新加载
   $ pg_ctl reload
   server signaled
   ```

6. 从库上执行(66从库)，从库执行之前需要先安装完成环境，主库必须在启动状态

   ```
   # su - pgsql
   $ pg_basebackup -D /data/postgresql/pgdata -F p -P -R -h 192.168.10.65 -p 5432 -U repuser -l backup20210219
   Password: 
   32560/32560 kB (100%), 1/1 tablespace
   
   #查看是否成功
   $ ll /data/postgresql/pgdata/
   total 64
   -rw------- 1 pgsql pgsql   213 Feb 19 22:38 backup_label
   drwx------ 6 pgsql pgsql    54 Feb 19 22:38 base
   -rw------- 1 pgsql pgsql    62 Feb 19 22:38 current_logfiles
   drwx------ 2 pgsql pgsql  4096 Feb 19 22:38 global
   -rw------- 1 pgsql pgsql   631 Feb 19 22:38 logfile
   drwx------ 2 pgsql pgsql     6 Feb 19 22:38 pg_commit_ts
   drwx------ 2 pgsql pgsql     6 Feb 19 22:38 pg_dynshmem
   -rw------- 1 pgsql pgsql  4827 Feb 19 22:38 pg_hba.conf
   -rw------- 1 pgsql pgsql  1636 Feb 19 22:38 pg_ident.conf
   drwx------ 4 pgsql pgsql    68 Feb 19 22:38 pg_logical
   drwx------ 4 pgsql pgsql    36 Feb 19 22:38 pg_multixact
   drwx------ 2 pgsql pgsql     6 Feb 19 22:38 pg_notify
   drwx------ 2 pgsql pgsql     6 Feb 19 22:38 pg_replslot
   drwx------ 2 pgsql pgsql     6 Feb 19 22:38 pg_serial
   drwx------ 2 pgsql pgsql     6 Feb 19 22:38 pg_snapshots
   drwx------ 2 pgsql pgsql     6 Feb 19 22:38 pg_stat
   drwx------ 2 pgsql pgsql     6 Feb 19 22:38 pg_stat_tmp
   drwx------ 2 pgsql pgsql     6 Feb 19 22:38 pg_subtrans
   drwx------ 2 pgsql pgsql     6 Feb 19 22:38 pg_tblspc
   drwx------ 2 pgsql pgsql     6 Feb 19 22:38 pg_twophase
   -rw------- 1 pgsql pgsql     3 Feb 19 22:38 PG_VERSION
   drwx------ 3 pgsql pgsql    60 Feb 19 22:38 pg_wal
   drwx------ 2 pgsql pgsql    18 Feb 19 22:38 pg_xact
   -rw------- 1 pgsql pgsql   268 Feb 19 22:38 postgresql.auto.conf
   -rw------- 1 pgsql pgsql 26670 Feb 19 22:38 postgresql.conf
   -rw------- 1 pgsql pgsql     0 Feb 19 22:38 standby.signal  #备库的标识
   ```

7. 修改从库的配置文件,从中主库上复制一份就好了，从库上执行(66从库)

   ```
   $ vim postgresql.conf
   #添加修改下面文件
   primary_conninfo = 'host=192.168.10.65 port=5432 user=repuser passowrd=repuser123'
   ```

8. 启动从库(66从库) 如果碰到权限问题，推出重新赋值700权限（# chmod -R 700 /data/postgresql）

   ```
   $ /usr/local/postgresql/bin/pg_ctl -D /data/postgresql/pgdata start
   waiting for server to start....2021-02-19 22:47:56.421 CST [26803] LOG:  starting PostgreSQL 12.6 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 4.8.5 20150623 (Red Hat 4.8.5-44), 64-bit
   2021-02-19 22:47:56.421 CST [26803] LOG:  listening on IPv4 address "0.0.0.0", port 5432
   2021-02-19 22:47:56.421 CST [26803] LOG:  listening on IPv6 address "::", port 5432
   2021-02-19 22:47:56.434 CST [26803] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5432"
   2021-02-19 22:47:56.550 CST [26803] LOG:  redirecting log output to logging collector process
   2021-02-19 22:47:56.550 CST [26803] HINT:  Future log output will appear in directory "/data/postgresql/logs".
    done
   server started
   
   ```

9. 查看主从状态，可以看到主从复制的三个进程

   ```
   # 主节点状态
   $ ps -ef|grep postgre
   pgsql    28198     1  0 22:19 ?        00:00:00 /usr/local/postgresql/bin/postgres
   pgsql    28199 28198  0 22:19 ?        00:00:00 postgres: logger   
   pgsql    28201 28198  0 22:19 ?        00:00:00 postgres: checkpointer   
   pgsql    28202 28198  0 22:19 ?        00:00:00 postgres: background writer   
   pgsql    28203 28198  0 22:19 ?        00:00:00 postgres: walwriter   
   pgsql    28204 28198  0 22:19 ?        00:00:00 postgres: autovacuum launcher   
   pgsql    28205 28198  0 22:19 ?        00:00:00 postgres: archiver   last was 000000010000000000000006.00000028.backup
   pgsql    28206 28198  0 22:19 ?        00:00:00 postgres: stats collector   
   pgsql    28207 28198  0 22:19 ?        00:00:00 postgres: logical replication launcher  
   pgsql    28298 28198  0 22:47 ?        00:00:00 postgres: walsender repuser 192.168.10.66(40052) streaming 0/7000148    #主节点的发送进程
   pgsql    28305 11401  0 22:49 pts/0    00:00:00 grep --color=auto postgre
   
   # 从节点状态
   $ ps -ef|grep postgre
   pgsql    26803     1  0 22:47 ?        00:00:00 /usr/local/postgresql/bin/postgres -D /data/postgresql/pgdata
   pgsql    26804 26803  0 22:47 ?        00:00:00 postgres: logger   #日志收集进程
   pgsql    26805 26803  0 22:47 ?        00:00:00 postgres: startup   recovering 000000010000000000000007   #2 接收后应用
   pgsql    26806 26803  0 22:47 ?        00:00:00 postgres: checkpointer   
   pgsql    26807 26803  0 22:47 ?        00:00:00 postgres: background writer   
   pgsql    26808 26803  0 22:47 ?        00:00:00 postgres: stats collector   #信息收集
   pgsql    26809 26803  0 22:47 ?        00:00:00 postgres: walreceiver   streaming 0/7000148  #1 从节点接受进程
   pgsql    26811 26786  0 22:48 pts/0    00:00:00 grep --color=auto postgre
   ```

10. 查看主从状态

    ```
    $ psql -h 127.0.0.1 -p 5432
    psql (12.6)
    Type "help" for help.
    
    postgres=# \x
    Expanded display is on.
    postgres=# select * from pg_stat_replication; 
    -[ RECORD 1 ]----+------------------------------
    pid              | 28298 #发送进程id
    usesysid         | 16389 #复制用户id
    usename          | repuser #复制的用户
    application_name | walreceiver
    client_addr      | 192.168.10.66 #从节点接收地址
    client_hostname  | 
    client_port      | 40052  #从节点接收端口
    backend_start    | 2021-02-19 22:47:56.623189+08  #从节点开始接收的时间 也就是搭建环境时间
    backend_xmin     | 
    state            | streaming  #同步状态，当前在同步过程中
    sent_lsn         | 0/7000148 #主库发送的位置
    write_lsn        | 0/7000148 #备库接收的位置
    flush_lsn        | 0/7000148 #备库同步到磁盘的位置
    replay_lsn       | 0/7000148 #备注同步到数据库的位置 这四个完全一样的话，代表主从同步一致
    write_lag        | 
    flush_lag        | 
    replay_lag       | 
    sync_priority    | 0   #同步的优先级，0代表异步 1代表同步
    sync_state       | async #当前同步状态，可能会变
    reply_time       | 2021-02-19 22:51:57.279284+08
    ```

11. 最后验证主从数据是否同步，主库插入一条，从库看看有没有

# 主从切换

P12之前: pg_ ctl promote 自己写个shell脚本、触发器方式，recovery.conf: triggre_file

P12和之后: 执行pg_ promote()函数 ( true,60 )，第一个参数为是否等待备库提升成功后返回，第二个参数为等待时间

select pg_promote ( true,60 ) ;

1. 从库提升为主库，原主库以新从库方式存在。65为从库, 66为主库 

   删除从库的标记信息，以66为主65位从重新做一遍主从（可以通过执行pg_ctl stop -m fast停止主库）

   1. 原主库65停止服务

      ```
      $ pg_ctl stop -m fast
      ```

   2. 原从库66提升为主

      ```
      $ psql -h 127.0.0.1 -p 5432
      psql (12.6)
      Type "help" for help.
      
      postgres=# select pg_promote(true,60);
       pg_promote 
      ------------
       t
      (1 row)
      ```

   3. 查看新主库66状态

      ```
      $ pg_controldata
      pg_control version number:            1201
      Catalog version number:               201909212
      Database system identifier:           6930892158948932168
      Database cluster state:               in production  #生产状态
      pg_control last modified:             Fri 19 Feb 2021 11:25:22 PM CST
      Latest checkpoint location:           0/8000108
      Latest checkpoint's REDO location:    0/80000D0
      Latest checkpoint's REDO WAL file:    000000020000000000000008
      Latest checkpoint's TimeLineID:       2
      Latest checkpoint's PrevTimeLineID:   2
      Latest checkpoint's full_page_writes: on
      Latest checkpoint's NextXID:          0:495
      Latest checkpoint's NextOID:          16390
      Latest checkpoint's NextMultiXactId:  1
      Latest checkpoint's NextMultiOffset:  0
      Latest checkpoint's oldestXID:        479
      Latest checkpoint's oldestXID's DB:   1
      Latest checkpoint's oldestActiveXID:  495
      Latest checkpoint's oldestMultiXid:   1
      Latest checkpoint's oldestMulti's DB: 1
      Latest checkpoint's oldestCommitTsXid:0
      Latest checkpoint's newestCommitTsXid:0
      Time of latest checkpoint:            Fri 19 Feb 2021 11:25:22 PM CST
      Fake LSN counter for unlogged rels:   0/3E8
      Minimum recovery ending location:     0/0
      Min recovery ending loc's timeline:   0
      Backup start location:                0/0
      Backup end location:                  0/0
      End-of-backup record required:        no
      wal_level setting:                    replica
      wal_log_hints setting:                off
      max_connections setting:              1000
      max_worker_processes setting:         8
      max_wal_senders setting:              10
      max_prepared_xacts setting:           0
      max_locks_per_xact setting:           64
      track_commit_timestamp setting:       off
      Maximum data alignment:               8
      Database block size:                  8192
      Blocks per segment of large relation: 131072
      WAL block size:                       8192
      Bytes per WAL segment:                16777216
      Maximum length of identifiers:        64
      Maximum columns in an index:          32
      Maximum size of a TOAST chunk:        1996
      Size of a large-object chunk:         2048
      Date/time type storage:               64-bit integers
      Float4 argument passing:              by value
      Float8 argument passing:              by value
      Data page checksum version:           0
      Mock authentication nonce:            cfbef73178832e0707468ce5fdf15e5e540c9a4d72cfd334f8e6d464b9deda25
      
      #变成了主库的归档模式
      $ ps -ef|grep postgre
      pgsql    26803     1  0 22:47 ?        00:00:00 /usr/local/postgresql/bin/postgres -D /data/postgresql/pgdata
      pgsql    26804 26803  0 22:47 ?        00:00:00 postgres: logger   
      pgsql    26806 26803  0 22:47 ?        00:00:00 postgres: checkpointer   
      pgsql    26807 26803  0 22:47 ?        00:00:00 postgres: background writer   
      pgsql    26808 26803  0 22:47 ?        00:00:00 postgres: stats collector   
      pgsql    26855 26803  0 23:25 ?        00:00:00 postgres: walwriter   
      pgsql    26856 26803  0 23:25 ?        00:00:00 postgres: autovacuum launcher   
      pgsql    26857 26803  0 23:25 ?        00:00:00 postgres: archiver   last was 000000010000000000000008.partial
      pgsql    26858 26803  0 23:25 ?        00:00:00 postgres: logical replication launcher   
      pgsql    26867 26786  0 23:26 pts/0    00:00:00 grep --color=auto postgre
      ```

   4. 新主库写入一条数据测试下，因为从库不允许写入数据

      ```
      $ psql -h 127.0.0.1 -p 5432
      psql (12.6)
      Type "help" for help.
      
      postgres=# \c testdb pgtest
      You are now connected to database "testdb" as user "pgtest".
      testdb=> insert into test values('test5');                
      INSERT 0 1
      ```

      

   5. 以下开始是旧主库65变从库修复过程

   6. 新主库66修改配置文件 添加权限信息

      ```
      # vim pg_hba.conf
      host replication repuser 0.0.0.0/0 md5
      ```

   7. 新主库66注释掉同步信息primary_conninfo

      ```
      # vim postgresql.conf
      # primary_conninfo = 'host=192.168.10.65 port=5432 user=repuser passowrd=repuser123'
      ```

   8. 新备库65删除data目录(确认数据没有运行)

      ```
      $ rm -rf /data/postgresql/pgdata
      ```

   9. 新备库65执行同步

      ```
      $ pg_basebackup -D /data/postgresql/pgdata -F p -P -R -h 192.168.10.66 -p 5432 -U repuser -l backup20210219_1
      ```

   10.  新备库65添加备库的标识文件

       ```
       $ cd /data/postgresql/pgdata
       $ echo "standby_mode='on' > standby.signal 
       $ cat standby.signal 
       standby_mode='on'
       ```

   11. 新备库65修改配置文件 添加新主库66信息

       ```
       # vim postgresql.conf
       primary_conninfo = 'host=192.168.1.66 port=5432 user=repuser passowrd=repuser123'
       ```

   12. 启动新备库65

       ```
       $ /usr/local/postgresql/bin/pg_ctl -D /data/postgresql/pgdata start
       waiting for server to start....2021-02-19 23:33:39.744 CST [28422] LOG:  starting PostgreSQL 12.6 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 4.8.5 20150623 (Red Hat 4.8.5-44), 64-bit
       2021-02-19 23:33:39.744 CST [28422] LOG:  listening on IPv4 address "0.0.0.0", port 5432
       2021-02-19 23:33:39.744 CST [28422] LOG:  listening on IPv6 address "::", port 5432
       2021-02-19 23:33:39.758 CST [28422] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5432"
       2021-02-19 23:33:39.858 CST [28422] LOG:  redirecting log output to logging collector process
       2021-02-19 23:33:39.858 CST [28422] HINT:  Future log output will appear in directory "/data/postgresql/logs".
        done
       server started
       ```

   13. 新主库66写入一条数据测试下 看看从库有没有

       ```
       $ psql -h 127.0.0.1 -p 5432
       psql (12.6)
       Type "help" for help.
       
       postgres=# \c testdb pgtest
       You are now connected to database "testdb" as user "pgtest".
       testdb=> insert into test values('test6');                
       INSERT 0 1
       
       $ psql -h 127.0.0.1 -p 5432
       psql (12.6)
       Type "help" for help.
       
       postgres=# \c testdb pgtest
       You are now connected to database "testdb" as user "pgtest".
       testdb=> select * from test; 
        name  
       -------
        test1
        test2
        test3
        test4
        test5
        test6
       (6 rows)
       
       ```

   14. 查看新主执行状态

       ```
       $ psql -h 127.0.0.1 -p 5432
       postgres=#  \x
       Expanded display is on.
       postgres=# select * from pg_stat_replication; 
       -[ RECORD 1 ]----+------------------------------
       pid              | 26898
       usesysid         | 16389
       usename          | repuser
       application_name | walreceiver
       client_addr      | 192.168.10.65
       client_hostname  | 
       client_port      | 38922
       backend_start    | 2021-02-19 23:33:39.928283+08
       backend_xmin     | 
       state            | streaming
       sent_lsn         | 0/A0001E8
       write_lsn        | 0/A0001E8
       flush_lsn        | 0/A0001E8
       replay_lsn       | 0/A0001E8
       write_lag        | 
       flush_lag        | 
       replay_lag       | 
       sync_priority    | 0
       sync_state       | async
       reply_time       | 2021-02-19 23:35:53.639687+08
       
       ```

       

2. 原主库恢复后仍以主库运行。把上门的步骤全部重新执行一遍

   1. 新主库66 :停止服务
   2. 新备库65 :停止服务,删除standby.signal ,注释primary_conninfo ,启动服务。
   3. 新主库65 :创建standby signal ,创建primary. conninfo ,启动服务。

# 主从复制(扩容从节点)

安装主从方式在做一个从库就可以了(第6步开始)，最后在主节点查看是否成功

```
$ psql -h 127.0.0.1 -p 5432
postgres=#  \x
Expanded display is on.
postgres=# select client_addr,sync_state from pg_stat_replication; 
psql (12.6)
Type "help" for help.

postgres=# select client_addr,sync_state from pg_stat_replication; 
  client_addr  | sync_state 
---------------+------------
 192.168.10.66 | async
 192.168.10.67 | async
(2 rows)

```

插入数据验证是否成功



# 主从复制tips

1. 关于pg_basebackup实现主从复制,自定义表空间目录,会出错。

2. 在主从复制环境中,主库创建表空间目录,备库也需要手工创建,才可以生效。

3. 举例:
   创建用户test1 创建表空间testtbs

   1. 主、从库创建目录:

   ```
   # su - pgsql
   $ mkdir /data/postgresql/pgdata/testtbs
   ```

   2. 主创建用户:

      ```
      # psql -h 127.0.0.1 -p 5432
      psql (12.6)
      Type "help" for help.
      
      #主库创建用户
      postgres=# create user test1 with password 'test123' nocreatedb;
      CREATE ROLE
      #主库创建表空间:
      postgres=# create tablespace testtbs owner test1 location '/data/postgresql/pgdata/testtbs'; 
      WARNING:  tablespace location should not be inside the data directory
      CREATE TABLESPACE
      #主库创建数据库:
      postgres=# create database test1db with owner=test1 tablespace=testtbs;
      CREATE DATABASE
      ```

   3. 验证主从是否同步-主库执行

      ```
      # 主库执行
      # psql -h 127.0.0.1 -p 5432
      psql (12.6)
      Type "help" for help.
      
      postgres=# create tablespace testtbs owner test1 location '/data/postgresql/pgdata/testtbs'; 
      WARNING:  tablespace location should not be inside the data directory
      CREATE TABLESPACE
      postgres=# create database test1db with owner=test1 tablespace=testtbs;
      CREATE DATABASE
      postgres=# \c test1db test1;
      You are now connected to database "test1db" as user "test1".
      test1db=> create table test(name varchar(50));
      CREATE TABLE
      test1db=> insert into test values('fg01');
      INSERT 0 1
      test1db=> select * from test;
       name 
      ------
       fg01
      (1 row)
      
      ```
      
   4. 验证主从是否同步-从库执行
   
      ```
      
      ```
   #  psql -h 127.0.0.1 -p 5432
      psql (12.6)
   Type "help" for help.
   
      postgres=# psql -h 127.0.0.1 -p 5432
      test1db=> \c test1db test1;
      You are now connected to database "test1db" as user "test1".
      test1db=> select * from test;
       name 
      ------
       fg01
      (1 row)
   
      ```
   
   * PostgreSQL主从复制介绍：物理复制异步与同步复制

   * 异步流复制模式中,主库提交的事务不会等待备库接收WAL日志流并返回确认信息，因此异步流复制模式下主库与备库的数据版本上会存在一定的处理延迟(毫秒级）。当主库宕机,这个延迟就主要受到故障发现与切换时间的影响而拉长。

   * 同步流复制模式中，要求主库把WAL日志写入磁盘 ，同时等待WAL日志记录复制到备库、并且WAL日志记录在任何一个备库写入磁盘后,才能向应用返回Commit结果。一旦所有备库故障,在主库的应用操作则会被挂起,所以此方式建议起码是

   * 同步和异步复制通过synchronous_commit和synchronous_standby_names参数来设置

     * synchronous_commit，设置同步或者异步方式取值：
   
       * on:同步 一定要等到从库写入磁盘后确认  
    * off:异步 默认值
   
      ```
  * synchronous_standby_names，指定某台机器进行同步或者异步
   
       * synchronous_standby_names="*" 代表全部同步

       * synchronous_standby_names = 'postgresql_02_66,postgresql_02_67' 指定某两台机器同步或者异步

       * synchronous_standby_names = 'postgresql_02_66' 指定某一台机器同步或者异步

       * synchronous_standby_names 定义机器的机器名字,例如：

         ```
        
        ```
# primary_conninfo = 'application_name=postgresql_02_66 host=192.168.10.65 port=5432 user=repuser passowrd=repuser123'
         ```
      
         
         ```

# 主从读写分离 pgpoo-II

支持连接池、主备切换、负载均衡、读写分离
写性能不好,不支持部分查询
原始模式,复制模式,主/备模式,并行模式。.
pgpool-II建议只用在读写分离，且必须装在主库上:

环境如下

192.168.10.65 : postgreSQL_01_65,主库,写: pgpool-II
192.168.10.66 : postgreSQL_02_66,从库1,读
192.168.10.67 : postgreSQL_03_67,从库2 ,读

1. 安装

   ```
   # cd /usr/local/src/
   # tar zxvf pgpool-II-4.1.1.tar.gz
   # cd pgpool-II-4.1.1
   # mkdir /usr/local/pgpool
   # 需要带postgresql数据库安装目录
   # ./configure --prefix=/usr/local/pgpool --with-pgsql=/usr/local/postgresql
   # make -j
   # make install
   ```

2. 环境变量配置

   ```
   # vim /etc/profile
   
   # 添加如下
   export PATH=$PATH:/usr/local/pgpool/bin
   
   # source /etc/profile
   ```

3. 验证

   ```
   # pgpool -version
   pgpool-II version 4.1.1 (karasukiboshi)
   ```

4. 修改配置文件

   ```
   #主要修改下面三个
   # cd /usr/local/pgpool/etc/
   # cp pgpool.conf.sample pgpool.conf
   # cp pcp.conf.sample pcp.conf
   # cp pool_hba.conf.sample pool_hba.conf
   ```

5. 修改pgpool.conf

   ```
   # vim pgpool.conf
   listen_addresses = '*' #监听所有地址
   port = 9999  #监听端口
   pcp_listen_addresses = '*'  #pcp监听地址
   pcp_port = 9898  #pcp监听端口
   
   #配置监听哪些数据，
   backend_hostname0 = '192.168.10.65'
   backend_port0 = 5432
   backend_weight0 = 1
   backend_data_directory0 = '/data/postgresql/pgdata/'  #数据库目录
   backend_flag0 = 'ALLOW_TO_FAILOVER' #是否允许故障转移
   backend_application_name0 = 'postgresql_01_65'  #主机名
   
   backend_hostname1 = '192.168.10.66'
   backend_port1 = 5432
   backend_weight1 = 1
   backend_data_directory1 = '/data/postgresql/pgdata/'  #数据库目录
   backend_flag1 = 'ALLOW_TO_FAILOVER'
   backend_application_name1 = 'postgresql_02_66'
   
   backend_hostname2 = '192.168.10.67'
   backend_port2 = 5432
   backend_weight2 = 1
   backend_data_directory2 = '/data/postgresql/pgdata/'  #数据库目录
   backend_flag2 = 'ALLOW_TO_FAILOVER'
   backend_application_name2 = 'postgresql_03_67'
   
   enable_pool_hba = on #是否启用防火墙认证
   pool_passwd = 'pool_passwd' #认证密码
   
   num_init_children = 128  #连接池
   
   log_destination = 'syslog' #日志模式
   log_connections = on #日志收集打开
   
   pid_file_name = '/var/run/pgpool/pgpool.pid' #pid路径
   logdir = '/var/log/pgpool' #日志文件路径
   
   load_balance_mode = on #打开负载均衡 因为有俩从库
   
   master_slave_mode = on #主从模式打开
   master_slave_sub_mode = 'stream' #主从模式为流复制
   sr_check_user = 'nobody'   #主从模式检测数据库 需要在库里创建这个用户
   
   #调试跟踪日志配置，会产生大量日志
   log_statement = all   #输出所有sql语句
   log_per_node_statement = on #输出每个节点的所有sql语句
   client_min_messages = log  #输出客户端最小日志
   log_min_messages = info  #输出节点最小日志
   
   #下面两个配置实现某个数据进行单独的读
   database_redirect_preference_list #指定某个数据库在某一个节点上负载
   app_name_redirect_preference_list #指定某个APP在某一个节点上负载
   ```

6. 创建pgpool的log

   ```
   # mkdir -p /var/run/pgpool/
   # mkdir -p /var/log/pgpool
   ```

   

7. 主库创建nobody用户的账号和密码

   ```
   # psql -h 127.0.0.1 -p 5432
   psql (12.6)
   Type "help" for help.
   
   postgres=# create role nobody login encrypted password 'nobody123';
   CREATE ROLE
   
   ```

8. 生成pool_passwd文件，注意 所有连接用户都需要认证生成

   ```
   # cd /usr/local/pgpool/etc
   # pg_md5 --md5auth --username=pgtest "pgtest"  #业务用户 
   # pg_md5 --md5auth --username=test1 "test123"  #业务用户
   # pg_md5 --md5auth --username=pgpool "pgpool123"  #pgpool管理用户
   # pg_md5 --md5auth --username=nobody "nobody123"  #健康检测的
   
   #查看文件
   # cat pool_passwd 
   pgtest:md59fe902b9525bd41695dcf708956e2559
   test1:md554e48ca388df890d4a48f1c2704c92c3
   pgpool:md541e61037ae2fa9abc8d79bcfc287f609
   nobody:md5b37c69bb745cf4a35ce4ce16a6f6d56f
   ```

9. 所有数据库hba防火墙文件添加nobody用户的权限

   ```
   # vim /data/postgresql/pgdata/pg_hba.conf
   ……
   host    all     nobody     0.0.0.0/0       md5
   
   #查看配置文件最后几行配置
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
   host replication repuser 0.0.0.0/0 md5
   host    all     nobody     0.0.0.0/0       md5
   ```

10. 所有库重新加载下配置文件

    ```
    # su - pgsql
    $ pg_ctl reload
    server signaled
    ```

11. 配置pgpool管理用户密码(管理pgpool的用户名和密码，不是数据库的用户，不要再数据里建立了)

    ```
    #先生成密码
    # pg_md5 pgpool123
    fa039bd52c3b2090d86b0904021a5e33
    
    #写入到pcp.conf文件 尾行添加
    # cd /usr/local/pgpool/etc
    # vim pcp.conf
    ……
    pgpool:fa039bd52c3b2090d86b0904021a5e33
    ```

12. 启动pgpool

    ```
    # pgpool &
    或者
    # pgpool -n -d > pgpool.log 2>&1 &
    ```

13. 停止pgpool

    ```
    # pgpool stop
    ```

14. 配置为系统服务

    ```
    # cp /usr/local/src/pgpool-II-4.1.1/src/redhat/pgpool.service /lib/systemd/system/
    # chmod +x /lib/systemd/system/pgpool.service
    # vim /lib/systemd/system/pgpool.service
    [Unit]
    Description=Pgpool-II
    After=syslog.target network.target
    [Service]
    #User=pgsql
    #Group=pgsql
    EnvironmentFile=-/etc/sysconfig/pgpool
    ExecStart=/usr/local/pgpool/bin/pgpool -f /usr/local/pgpool/etc/pgpool.conf -n -d
    ExecStop=/usr/local/pgpool/bin/pgpool -f /usr/local/pgpool/etc/pgpool.conf -m fast stop
    ExecReload=/usr/local/pgpool/bin/pgpool -f /usr/local/pgpool/etc/pgpool.conf reload
    LimitNOFILE=65536
    KillMode=process
    KillSignal=SIGINT
    Restart=on-abnormal
    RestartSec=30s
    TimeoutSec=0
    [Install]
    WantedBy=multi-user.target
    
    # systemctl daemon-reload
    #先停掉之前命令行启动的
    # pgpool stop
    
    # systemctl start pgpool
    # systemctl status pgpool
    # systemctl stop pgpool
    ```

15. 重置pgpool

    ```
    # pgpool -m fast stop #先停止
    # pgpool -C -D #重置状态 并且启动
    ```

16. 使用pgpool管理用户查看节点状态

    ```
    #查看0号节点 对应上面配置文件的0号配置
    # pcp_node_info -U pgpool -h 192.168.10.65 -p 9898 -n 0 -v
    Password: 
    Hostname               : 192.168.10.65
    Port                   : 5432
    Status                 : 2
    Weight                 : 0.333333
    Status Name            : up
    Role                   : primary
    Replication Delay      : 0
    Replication State      : 
    Replication Sync State : 
    Last Status Change     : 2021-02-22 17:07:22
    # pcp_node_info -U pgpool -h 192.168.10.65 -p 9898 -n 1 -v
    Password: 
    Hostname               : 192.168.10.66
    Port                   : 5432
    Status                 : 2
    Weight                 : 0.333333
    Status Name            : up
    Role                   : standby
    Replication Delay      : 0
    Replication State      : 
    Replication Sync State : 
    Last Status Change     : 2021-02-22 17:07:22
    # pcp_node_info -U pgpool -h 192.168.10.65 -p 9898 -n 2 -v
    Password: 
    Hostname               : 192.168.10.67
    Port                   : 5432
    Status                 : 2
    Weight                 : 0.333333
    Status Name            : up
    Role                   : standby
    Replication Delay      : 0
    Replication State      : 
    Replication Sync State : 
    Last Status Change     : 2021-02-22 17:07:22
    
    ```

17. 测试读写分离(业务系统直接连接pgpool的地址和端口)

    ```
    # 使用业务用户连接pgpool
    $ psql -U pgtest -h 192.168.10.65 -p 9999
    Password for user pgtest: 
    psql (12.6)
    Type "help" for help.
    
    #查看状态
    postgres=> \x
    Expanded display is on.
    postgres=> show pool_nodes;
    -[ RECORD 1 ]----------+--------------------
    node_id                | 0
    hostname               | 192.168.10.65
    port                   | 5432
    status                 | up
    lb_weight              | 0.333333
    role                   | primary
    select_cnt             | 0
    load_balance_node      | false
    replication_delay      | 0
    replication_state      | 
    replication_sync_state | 
    last_status_change     | 2021-02-22 17:07:22
    -[ RECORD 2 ]----------+--------------------
    node_id                | 1
    hostname               | 192.168.10.66
    port                   | 5432
    status                 | up
    lb_weight              | 0.333333
    role                   | standby
    select_cnt             | 0
    load_balance_node      | true
    replication_delay      | 0
    replication_state      | 
    replication_sync_state | 
    last_status_change     | 2021-02-22 17:07:22
    -[ RECORD 3 ]----------+--------------------
    node_id                | 2
    hostname               | 192.168.10.67
    port                   | 5432
    status                 | up
    lb_weight              | 0.333333
    role                   | standby
    select_cnt             | 0
    load_balance_node      | false
    replication_delay      | 0
    replication_state      | 
    replication_sync_state | 
    last_status_change     | 2021-02-22 17:07:22
    
    ```

18. 多个节点同时练到数据库

    ```
    psql -U pgtest -h 192.168.10.65 -p 9999
    postgres=# \c testdb pgtest;
    
    #插入数据 可以看到是节点0
    testdb=> insert into test values('test9'); 
    LOG:  statement: insert into test values('test9');
    LOG:  DB node id: 0 backend pid: 14410 statement: insert into test values('test9');
    INSERT 0 1
    
    #查询数据 可以看到是节点1或者0或者2
    testdb=> select * from test;
    LOG:  statement: select * from test;
    LOG:  DB node id: 0 backend pid: 14421 statement: SELECT version()
    LOG:  DB node id: 0 backend pid: 14421 statement: SELECT count(*) FROM pg_class AS c, pg_namespace AS n WHERE c.oid = pg_catalog.to_regclass('"test"') AND c.relnamespace = n.oid AND n.nspname = 'pg_catalog'
    LOG:  DB node id: 0 backend pid: 14421 statement: SELECT count(*) FROM pg_catalog.pg_class AS c, pg_namespace AS n WHERE c.relname = 'test' AND c.relnamespace = n.oid AND n.nspname ~ '^pg_temp_'
    LOG:  DB node id: 0 backend pid: 14421 statement: SELECT count(*) FROM pg_catalog.pg_class AS c WHERE c.oid = pg_catalog.to_regclass('"test"') AND c.relpersistence = 'u'
    LOG:  DB node id: 0 backend pid: 14421 statement: select * from test;  #0节点
    
    
    testdb=> select * from test;
    LOG:  statement: select * from test;
    LOG:  DB node id: 0 backend pid: 14409 statement: SELECT count(*) FROM pg_catalog.pg_class AS c, pg_namespace AS n WHERE c.relname = 'test' AND c.relnamespace = n.oid AND n.nspname ~ '^pg_temp_'
    LOG:  DB node id: 2 backend pid: 28048 statement: select * from test;  #2节点
    
    testdb=> select * from test;
    LOG:  statement: select * from test;
    LOG:  DB node id: 0 backend pid: 14467 statement: SELECT version()
    LOG:  DB node id: 0 backend pid: 14467 statement: SELECT count(*) FROM pg_class AS c, pg_namespace AS n WHERE c.oid = pg_catalog.to_regclass('"test"') AND c.relnamespace = n.oid AND n.nspname = 'pg_catalog'
    LOG:  DB node id: 0 backend pid: 14467 statement: SELECT count(*) FROM pg_catalog.pg_class AS c, pg_namespace AS n WHERE c.relname = 'test' AND c.relnamespace = n.oid AND n.nspname ~ '^pg_temp_'
    LOG:  DB node id: 0 backend pid: 14467 statement: SELECT count(*) FROM pg_catalog.pg_class AS c WHERE c.oid = pg_catalog.to_regclass('"test"') AND c.relpersistence = 'u'
    LOG:  DB node id: 1 backend pid: 28579 statement: select * from test;  #1节点
    ```

# keepaliaved双机主从高可用

1. keepaliaved安装,两台机器都安装

   ```
   # yum install keepalived -y
   # rpm -qc keepalived
   /etc/keepalived/keepalived.conf
   /etc/sysconfig/keepalived
   ```

2. master配置信息

   ```
   # mv /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak
   # vim /etc/keepalived/keepalived.conf
   ! Configuration File for keepalived
   global_defs {
       router_id postgres
   }
   
   vrrp_script check {
   }
   vrrp_instance VI_1 {
       state BACKUP
       interface eth0 
       virtual_router_id 50  
       priority 101            
       advert_int 1            
       authentication {
           auth_type PASS     
           auth_pass 2222    
   }
       virtual_ipaddress {
           10.149.229.38/25 brd 10.149.229.127 dev eth0 label eth0:vip            #虚拟ip
       }
   }
   ```

3. BACKUP配置信息

   ```
   # mv /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak
   # vim /etc/keepalived/keepalived.conf
   ! Configuration File for keepalived
   global_defs {
       router_id postgres
   }
   
   vrrp_script check {
   }
   vrrp_instance VI_1 {
       state BACKUP
       interface eth0 
       virtual_router_id 50  
       priority 100            
       advert_int 1            
       authentication {
           auth_type PASS     
           auth_pass 2222    
   }
       virtual_ipaddress {
           10.149.229.38/25 brd 10.149.229.127 dev eth0 label eth0:vip            #虚拟ip
       }
   }
   ```

   

   
