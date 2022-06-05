# 环境：

* centos7

# 软件包准备

* 安装包：redis-6.0.10.tar.gz

# 环境准备

* 安装依赖

  ```
  # yum -y install gcc-c++ autoconf automake
  ```

  

* 升级gcc，redis6版本需要高版本的gcc centos7版本的gcc是4.8.x

  ```
  # gcc -v
  Using built-in specs.
  COLLECT_GCC=gcc
  COLLECT_LTO_WRAPPER=/usr/libexec/gcc/x86_64-redhat-linux/4.8.5/lto-wrapper
  Target: x86_64-redhat-linux
  Configured with: ../configure --prefix=/usr --mandir=/usr/share/man --infodir=/usr/share/info --with-bugurl=http://bugzilla.redhat.com/bugzilla --enable-bootstrap --enable-shared --enable-threads=posix --enable-checking=release --with-system-zlib --enable-__cxa_atexit --disable-libunwind-exceptions --enable-gnu-unique-object --enable-linker-build-id --with-linker-hash-style=gnu --enable-languages=c,c++,objc,obj-c++,java,fortran,ada,go,lto --enable-plugin --enable-initfini-array --disable-libgcj --with-isl=/builddir/build/BUILD/gcc-4.8.5-20150702/obj-x86_64-redhat-linux/isl-install --with-cloog=/builddir/build/BUILD/gcc-4.8.5-20150702/obj-x86_64-redhat-linux/cloog-install --enable-gnu-indirect-function --with-tune=generic --with-arch_32=x86-64 --build=x86_64-redhat-linux
  Thread model: posix
  gcc version 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC)
  
  # 安装 scl 源 也可以只安装centos-release-scl
  yum install -y centos-release-scl scl-utils-build
  # 安装 9 版本的 gcc、gcc-c++、gdb 工具链（toolchian）
  yum install -y devtoolset-9-toolchain
  # 临时覆盖系统原有的 gcc 引用(临时生效)
  scl enable devtoolset-9 bash
  # 查看 gcc 当前版本
  gcc -v
  Using built-in specs.
  COLLECT_GCC=gcc
  COLLECT_LTO_WRAPPER=/opt/rh/devtoolset-9/root/usr/libexec/gcc/x86_64-redhat-linux/9/lto-wrapper
  Target: x86_64-redhat-linux
  Configured with: ../configure --enable-bootstrap --enable-languages=c,c++,fortran,lto --prefix=/opt/rh/devtoolset-9/root/usr --mandir=/opt/rh/devtoolset-9/root/usr/share/man --infodir=/opt/rh/devtoolset-9/root/usr/share/info --with-bugurl=http://bugzilla.redhat.com/bugzilla --enable-shared --enable-threads=posix --enable-checking=release --enable-multilib --with-system-zlib --enable-__cxa_atexit --disable-libunwind-exceptions --enable-gnu-unique-object --enable-linker-build-id --with-gcc-major-version-only --with-linker-hash-style=gnu --with-default-libstdcxx-abi=gcc4-compatible --enable-plugin --enable-initfini-array --with-isl=/builddir/build/BUILD/gcc-9.3.1-20200408/obj-x86_64-redhat-linux/isl-install --disable-libmpx --enable-gnu-indirect-function --with-tune=generic --with-arch_32=x86-64 --build=x86_64-redhat-linux
  Thread model: posix
  gcc version 9.3.1 20200408 (Red Hat 9.3.1-2) (GCC)
  
  #修改文件使其永久生效
  # vim /etc/profile
  #最后一行添加
  source /opt/rh/devtoolset-9/enable
  
  # source /etc/profile
  ```



# 开始安装

1. 解压 编译 (编译前确认gcc版本)

   ```
   # cd /usr/local/src
   # tar zxvf redis-6.0.10.tar.gz
   # cd redis-6.0.10
   # make -j
   # 创建redis安装目录
   # mkdir /usr/local/redis
   # 安装(带安装目录的,make install默认安装在/usr/local/bin下) 注意大小写
   # make PREFIX=/usr/local/redis/ install
   # 查看安装目录安装完成
   # ll /usr/local/redis/bin/
   total 37968
   -rwxr-xr-x 1 root root 4739592 Feb 15 18:00 redis-benchmark
   -rwxr-xr-x 1 root root 9687576 Feb 15 18:00 redis-check-aof
   -rwxr-xr-x 1 root root 9687576 Feb 15 18:00 redis-check-rdb
   -rwxr-xr-x 1 root root 5059888 Feb 15 18:00 redis-cli
   lrwxrwxrwx 1 root root      12 Feb 15 18:00 redis-sentinel -> redis-server
   -rwxr-xr-x 1 root root 9687576 Feb 15 18:00 redis-server
   ```

2. 安装文件解释

   * redis-benchmark：性能跑分工具 
   * redis-check-aof：AOF文件修复工具
   * redis-check-rdb：RDB文件修复工具
   * redis-cli ： 客户端命令行
   * redis-sentinel：集群管理工具
   * redis-server : 服务进程指令

# 单机模式

1. 单机启动(前置启动)

   ```
   # cd /usr/local/redis/bin/
   # ./redis-server 
   6158:C 15 Feb 2021 18:03:28.314 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
   6158:C 15 Feb 2021 18:03:28.314 # Redis version=6.0.10, bits=64, commit=00000000, modified=0, pid=6158, just started
   6158:C 15 Feb 2021 18:03:28.314 # Warning: no config file specified, using the default config. In order to specify a config file use ./redis-server /path/to/redis.conf
   6158:M 15 Feb 2021 18:03:28.315 * Increased maximum number of open files to 10032 (it was originally set to 1024).
                   _._                                                  
              _.-``__ ''-._                                             
         _.-``    `.  `_.  ''-._           Redis 6.0.10 (00000000/0) 64 bit
     .-`` .-```.  ```\/    _.,_ ''-._                                   
    (    '      ,       .-`  | `,    )     Running in standalone mode(单机模式)
    |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
    |    `-._   `._    /     _.-'    |     PID: 6158
     `-._    `-._  `-./  _.-'    _.-'                                   
    |`-._`-._    `-.__.-'    _.-'_.-'|                                  
    |    `-._`-._        _.-'_.-'    |           http://redis.io        
     `-._    `-._`-.__.-'_.-'    _.-'                                   
    |`-._`-._    `-.__.-'    _.-'_.-'|                                  
    |    `-._`-._        _.-'_.-'    |                                  
     `-._    `-._`-.__.-'_.-'    _.-'                                   
         `-._    `-.__.-'    _.-'                                       
             `-._        _.-'                                           
                 `-.__.-'                                               
   
   6158:M 15 Feb 2021 18:03:28.315 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
   6158:M 15 Feb 2021 18:03:28.315 # Server initialized
   6158:M 15 Feb 2021 18:03:28.315 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
   6158:M 15 Feb 2021 18:03:28.315 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo madvise > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled (set to 'madvise' or 'never').
   6158:M 15 Feb 2021 18:03:28.316 * Ready to accept connections
   ```

2. 单机启动(守护进程启动)

   1. 修改配置文件

      ```
      # 修改配置文件的daemonize参数为yes 大约在224行
      # cp /usr/local/src/redis-6.0.10/redis.conf /usr/local/redis/
      # vim redis.conf
      ……
      daemonize yes
      ……
      ```

   2. 启动

      ```
      # bin/redis-server ./redis.conf
      # ps -ef|grep redis
      root     20785     1  0 18:17 ?        00:00:00 bin/redis-server 127.0.0.1:6379
      root     20791  5479  0 18:18 pts/0    00:00:00 grep --color=auto redis
      ```

   3. 也可以配置开机启动

      ```
      # vim /etc/systemd/system/redis.service
      [Unit]
      Description=redis-server
      After=network.target
      
      [Service]
      Type=forking
      ExecStart=/usr/local/redis/bin/redis-server /usr/local/redis/redis.conf
      PrivateTmp=true
      
      [Install]
      WantedBy=multi-user.target
      
      #重新加载系统服务
      # systemctl daemon-reload
      
      #服务命令
      # systemctl enable redis.service
      # systemctl disable redis.service
      # systemctl stop redis.service
      # systemctl restart redis.service
      # systemctl status redis.service
      ```

      * Description：服务描述
      * After：描述服务类别
      * [Service]服务运行参数设置
      * Type=forking后台运行的模式
      * ExecStart：服务具体的命令（要求绝对路径）
      * ExecReload：服务重启命令（要求绝对路径）
      * ExecStop：服务停止命令（要求绝对路径）
      * WantedBy=multi-user.target：给服务分配独立的临时空间

# 主从模式

1. 准备两台机器

   主：192.168.10.60

   从：192.168.10.61

2. 主从两台机器创建数据，配置，日志目录

   ```
   # 创建数据目录
   mkdir -p /data/redis/6379/data
   # 创建日志目录
   mkdir -p /data/redis/6379/logs
   ```

3. 主节点创建配置文件

   ```
   vim /data/redis/6379/redis.conf
   # 放行访问IP限制
   bind 0.0.0.0
   # 后台启动
   daemonize yes
   # 日志存储目录及日志文件名
   logfile "/data/redis/6379/logs/redis.log"
   # rdb数据文件名
   dbfilename dump.rdb
   # aof模式开启和aof数据文件名
   appendonly yes
   appendfilename "appendonly.aof"
   # rdb数据文件和aof数据文件的存储目录
   dir /data/redis/6379/data
   # 设置密码
   requirepass 123456
   # 从节点访问主节点密码(必须与 requirepass 一致)
   masterauth 123456
   # 从节点只读模式
   replica-read-only yes
   ```

4. 从节点创建配置文件

   ```
   vim /data/redis/6379/redis.conf
   # 放行访问IP限制
   bind 0.0.0.0
   # 后台启动
   daemonize yes
   # 日志存储目录及日志文件名
   logfile "/data/redis/6379/logs/redis.log"
   # rdb数据文件名
   dbfilename dump.rdb
   # aof模式开启和aof数据文件名
   appendonly yes
   appendfilename "appendonly.aof"
   # rdb数据文件和aof数据文件的存储目录
   dir /usr/local/redis/data
   # 设置密码
   requirepass 123456
   # 从节点访问主节点密码(必须与 requirepass 一致)
   masterauth 123456
   # 从节点只读模式
   replica-read-only yes
   # 下面的配置无需在主节点中配置
   # 从节点属于哪个主节点
   slaveof 192.168.10.60 6379
   ```

5. 依次启动主节点和从节点 可以配置俩从节点

   ```
   #60 主节点
   # /usr/local/redis/bin/redis-server /data/redis/6379/redis.conf
   #61 从节点
   # /usr/local/redis/bin/redis-server /data/redis/6379/redis.conf
   ```

6. 验证

   ```
   # 60 主节点
   # /usr/local/redis/bin/redis-cli
   127.0.0.1:6379> auth 123456
   OK
   127.0.0.1:6379> info replication
   # Replication
   # 当前节点角色
   role:master
   # 从节点的连接数也就是几个从节点
   connected_slaves:1
   # 从节点详细信息 IP PORT 状态 命令(单位:字节长度)偏移量 延迟秒数
   # 主节点每次处理完写操作，会把命令的字节长度累加到master_repl_offset中。
   # 从节点在接收到主节点发送的命令后，会累加记录子什么偏移量信息slave_repl_offset，同时，也会每秒钟上报自身的复制偏移量到主节点，以供主节点记录存储。
   # 在实际应用中，可以通过对比主从复制偏移量信息来监控主从复制健康状况。
   slave0:ip=192.168.10.61,port=6379,state=online,offset=168,lag=1
   # master启动时生成的40位16进制的随机字符串，用来标识master节点
   master_replid:8ecb03300e944924d523c5d13acaf3394b597137
   # 当采用哨兵模式的时候，重新选举后产生的新的id会替换到master_replid，master_replid2则记录以前的masterid
   master_replid2:0000000000000000000000000000000000000000
   # master 命令(单位:字节长度)已写入的偏移量 主节点会把要同步的数据命令转换成字节，master_repl_offset用来记录当前主节点已经写入了多少字节数据，看上面的从节点的复制信息也是168，代表现在主从数据是一致的
   master_repl_offset:168
   second_repl_offset:-1
   # 0/1：关闭/开启复制积压缓冲区标志(2.8+)，主要用于增量复制及丢失命令补救
   repl_backlog_active:1
   # 缓冲区最大长度，默认 1M
   repl_backlog_size:1048576
   # 缓冲区起始偏移量
   repl_backlog_first_byte_offset:1
   # 缓冲区已存储的数据长度
   repl_backlog_histlen:168
   
   # 61 从节点 从节点可以看到复制的信息
   # /usr/local/redis/bin/redis-cli
   127.0.0.1:6379> auth 123456
   OK
   127.0.0.1:6379> info replication
   # Replication
   # 角色
   role:slave
   # 主节点详细信息
   master_host:192.168.10.60
   master_port:6379
   # slave端可查看它与master之间同步状态,当复制断开后表示down
   master_link_status:up
   # 主库多少秒未发送数据到从库
   master_last_io_seconds_ago:9
   # 从服务器是否在与主服务器进行同步 0否/1是
   master_sync_in_progress:0
   # slave复制的时候的(单位:字节长度)偏移量
   slave_repl_offset:252
   # 选举时，成为主节点的优先级，数字越大优先级越高，0 永远不会成为主节点
   slave_priority:100
   # 从库是否设置只读，0读写/1只读
   slave_read_only:1
   # 连接的slave实例个数，从节点下面也可以在接入从节点
   connected_slaves:0
   # master启动时生成的40位16进制的随机字符串，用来标识master节点
   master_replid:8ecb03300e944924d523c5d13acaf3394b597137
   # slave切换master之后，会生成了自己的master标识，之前的master节点的标识存到了master_replid2的位置
   master_replid2:0000000000000000000000000000000000000000
   # master 命令(单位:字节长度)已写入的偏移量
   master_repl_offset:252
   # 主从切换时记录主节点的命令偏移量+1，为了避免全量复制，用来记录每次主从改变后最后一次同步的数据的值
   second_repl_offset:-1
   # 0/1：关闭/开启复制积压缓冲区标志(2.8+版本之后)，主要用于增量复制及丢失命令补救
   repl_backlog_active:1
   # 缓冲区最大长度，默认 1M
   repl_backlog_size:1048576
   # 缓冲区起始偏移量
   repl_backlog_first_byte_offset:1
   # 缓冲区已存储的数据长度
   repl_backlog_histlen:252
   ```

7. 日志查看

   ```
   tail -f -n 1000 /usr/local/redis/log/redis.log
   # 准备就绪，接受客户端连接
   * Ready to accept connections
   # 102 从节点发起 SYNC 请求
   * Replica 192.168.10.102:6379 asks for synchronization
   # 全量
   # 从节点发起全量复制请求
   * Full resync requested by replica 192.168.10.60:6379
   # 创建 repl_backlog 文件及生成 master_replid
   * Replication backlog created, my new replication IDs are 'acc2aaa1f0bb0fd79d7d3302f16bddcbe4add423' and '0000000000000000000000000000000000000000'
   # 通过 BGSAVE 指令将数据写入磁盘(RBD操作)
   * Starting BGSAVE for SYNC with target: disk
   # 开启一个子守护进程执行写入
   * Background saving started by pid 1377
   # 数据已写入磁盘
   * DB saved on disk
   # 有 4MB 数据已写入磁盘
   * RDB: 4 MB of memory used by copy-on-write
   # 保存结束
   * Background saving terminated with success
   # 从节点同步数据结束
   * Synchronization with replica 192.168.10.60:6379 succeeded
   # 103 从节点发起 SYNC 请求，执行同步数据操作
   * Replica 192.168.10.103:6379 asks for synchronization
   # 从节点发起全量复制请求
   * Full resync requested by replica 192.168.10.61:6379
   
   # 增量
   # 当前一个客户端连接，执行了两个复制
   1 clients connected (2 replicas), 1955144 bytes in use
   
   ```

# 哨兵模式

三台节点为哨兵：192.168.10.62，192.168.10.63，192.168.10.64

1. 修改三台节点名字

   ```
   # hostnamectl set-hostname redis_sentinel_01
   # hostnamectl set-hostname redis_sentinel_02
   # hostnamectl set-hostname redis_sentinel_03
   ```

2. 三台节点创建配置文件目录

   ```
   # mkdir -p /data/redis/sentinel/
   # mkdir -p /data/redis/sentinel/log
   ```

3. 三台节点创建配置文件

   ```
   # vim /data/redis/sentinel/sentinel.conf
   #IP 限制
   bind 0.0.0.0
   # 进程端口号
   port 26379
   # 后台启动
   daemonize yes
   # 日志记录文件
   logfile "/data/redis/sentinel/log/sentinel.log"
   # 进程编号记录文件
   pidfile /var/run/sentinel.pid
   # 指示 Sentinel 去监视一个名为 mymaster 的主服务器 ip 端口 权重(几台机器同时决定说开始切换需要超过半数)
   sentinel monitor mymaster 192.168.10.60 6379 2
   #故障转移后重新主从复制，1表示串行，>1并行
   sentinel parallel-syncs mymaster 1
   # 访问主节点的密码
   sentinel auth-pass mymaster 123456
   # Sentinel 认为服务器已经断线所需的毫秒数
   sentinel down-after-milliseconds mymaster 10000
   # 若 Sentinel 在该配置值内未能完成 failover 操作，则认为本次 failover 失败
   sentinel failover-timeout mymaster 180000
   
   ```

4. 三个节点启动sentinel服务

   ```
   # /usr/local/redis/bin/redis-sentinel /data/redis/sentinel/sentinel.conf
   ```

5. 查看日志 可以看到sentinel启动 监控了62节点，并且添加了63 64两个sentinel

   ```
   # tail -f /data/redis/sentinel/log/sentinel.log 
   20551:X 27 Feb 2021 21:46:45.421 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
   20551:X 27 Feb 2021 21:46:45.431 # Sentinel ID is b06764dc501d8d30036030fa6d9ef34db537c5da
   20551:X 27 Feb 2021 21:46:45.431 # +monitor master mymaster 192.168.10.61 6379 quorum 2
   20551:X 27 Feb 2021 21:46:57.948 * +sentinel sentinel 68cbb06e61271c76aff602fd4f7a86bcf995395f 192.168.10.63 26379 @ mymaster 192.168.10.61 6379
   20551:X 27 Feb 2021 21:46:59.824 * +sentinel sentinel 3c7d0190143e6e2084a2e88a169113c3a32e674e 192.168.10.64 26379 @ mymaster 192.168.10.61 6379
   20551:X 27 Feb 2021 21:47:15.508 # +sdown master mymaster 192.168.10.61 6379
   20551:X 27 Feb 2021 21:47:25.990 # +new-epoch 1
   20551:X 27 Feb 2021 21:47:25.996 # +vote-for-leader 68cbb06e61271c76aff602fd4f7a86bcf995395f 1
   20551:X 27 Feb 2021 21:47:26.985 # +odown master mymaster 192.168.10.61 6379 #quorum 2/2
   20551:X 27 Feb 2021 21:47:26.985 # Next failover delay: I will not start a failover before Sat Feb 27 21:53:26 2021
   
   ```

   

# redis.conf配置文件详细解读

* Redis支持很多的参数，但都有默认值。
* daemonize
  * 默认情况下，redis 不是在后台运行的，如果 需要在后台运行，把该项的值更改为yes。
* pidfile
  * 当Redis在后台运行的时候，Redis 默认会把pid文件放在/var/run/redis.pid，你可以配置到其他地
    址。当运行多个redis服务时，需要指定不同的pid文件和端口
* bind
  * 指定Redis只接收来自于该IP地址的请求，如果不进行设置，那么将处理所有请求，在生产环境中 最好设
    置该项
  * 0.0.0.0任何机器都可以连接也可以指定某个ip，例如:192.168.10.60
* save
  * 设置Redis进行数据库镜像的频率。
    if(在60秒之内有10000个keys发生变化时){
    进行镜像备份
    }else if(在300秒之内有10个keys发生了变化){
    进行镜像备份
    }else if(在900秒之内有1个keys发生了变化){
    讲行错像各份
* port
  * 监听端口，默认为6379
* timeout
  * 设置客户端连接时的超时时间，单位为秒。当客户端在这段时间内没有发出任何指令，那么关闭该连接
* loglevel
  * log等级分为4级，debug, verbose, notice, 和warning。生产环境 下一般开启notice
* logfile
  * 配置log文件地址，默认使用标准输出，即打印在命令行终端的窗口上
* databases
  * 设置数据库的个数，可以使用SELECT命令来切换数据库。默认使用的数据库是0 
* rdbcompression
  * 在进行镜像备份时，是否进行压缩
* dbfilename
  * 镜像备份文件的文件名
* dir
  * 数据库镜像备份的文件放置的路径。
    这里的路径跟文件名要分开配置是因为Redis在进行备份时，先会将当前数据库的状态写入到一-个临时
    文件中等备份完成时，再把该临时文件替换为上面所指定的文件,而这里的临时文件和上面所配置的备份文件都会放在这个指定的路径当中
* slaveof
  * 设置该数据库为其他数据库的从数据库
* masterauth
  * 当主数据库连接需要密码验证时，在这里指定
* requirepass
  * 设置客户端连接后进行任何其他指定前需要使用的密码。
    警告:因为redis 速度相当快，所以在一- 台比较好的服务器下，- -个外部的用户可以在一秒钟进行 150K 次的密
    码尝试，这意味着你需要指定非常非常强大的密码来防止暴力破解。
* maxclients
  * 限制同时连接的客户数量。当连接数超过这个值时，redis 将不再接收其他连接请求,
    客户端尝试连接时将收到error信息。
* maxmemory
  * 设置redis能够使用的最大内存。
* appendonly
  * 默认情况下，redis 会在后台异步的把数据库镜像备份到磁盘,但是该备份是非常耗时的，且备份也不
    能很频繁，如果发 生诸如拉闸限电、拔插头等状况，那么将造成比较大范围的数据丢失。
    所以redis提供了另外一种更加高效的数据库备份及灾难恢复方式。开启append only模式之后，redis
    会把所接收到的每-次写操作请求都追加到appendonly.aof文件中，当redis重新启动时，会从该文件恢
    复出之前的状态。
    但是这样会造成appendonly.aof 文件过大，所以redis 还支持了BGREWRITEAOF 指令，对
    appendonly.aof进行重新整理。
    所以我认为推荐生产环境下的做法为关闭镜像，开启appendonly . aof，同时可以选择在访问较少的时间每天对ap
    pendonly.aof进行重写- - 次。
* appendfsync
  * 设置对appendonly.aof文件进行同步的频率。always 示每次有写操作都进行同步,
    everysec表示对写操作进行累积，每秒同步- -次。 这个需要根据实际业务场景进行配置
* vm-enabled
  * 是否开启虚拟内存支持。因为redis是一个内存数据库, 且当内存满的时候，无法接收新的写请求，所以
    在redis 2.0中，提供了虚拟内存的支持。但是需要注意的是，redis中， 所有的key都会放在内存中，在内存不够时，只会把value值放入交换
    区。这样保证了虽然使用虚拟内存,但性能基本不受影响
    同时，你需要注意的是你要把vm-max memory设置到足够来放下你的所有的key
* vm-swap-file
  * 设置虚拟内存的交换文件路径
* vm-max-memory
  * 这里设置开启虚拟内存之后，redis 将使用的最大物理内存的大小。默认为0，redis 将把他所有的能放
    到交换文件的都放到交换文件中，以尽量少的使用物理内存。
    在生产环境下，需要根据实际情况设置该值，最好不要使用默认的0
* vm-page-size
  * 设置虚拟内存的页大小，如果你的value值比较大,比如说你要在value中放置博客、新闻之类的所有文
    章内容，就设大一点，如果要放置的都是很小的内容,那就设小一点。
* vm-pages
  * 设置交换文件的总的page数量，要注意的是，page table信息会放在物理内存中，每8个page就会
    占据RAM中的1个byte。总的虚拟内存大小 = vm-page-size * vm-pages
* vm-max-threads
  * 设置VM I0同时使用的线程数量。因为在进行内存交换时，对数据有编码和解码的过程，所以尽管I0设
    备在硬件_上本上不能支持很多的并发读写，但是还是如果你所保存的vlaue 值比较大，将该值设大一些,
    还是能够提升性能的
* glueoutputbuf
  * 把小的输出缓存放在一起，以便能够在-个TCP packet中为客户端发送多个响应，具体原理和真实效果
    我不是很清楚。所以根据注释，你不是很确定的时候就设置成yes
* hash-max-zipmap-entries
  * 在redis 2.0中引入了hash数据结构。当hash中包含超过指定元素个数并且最大的元素没有超过临界
    时，hash 将以一种特殊的编码方式(大大减少内存使用)来存储,这里可以设置这两个临界值
* activerehashing
  * 开启之后，redis 将在每100毫秒时使用1毫秒的CPU时间来对redis的hash表进行重新hash，可以
    降低内存的使用。
    当你的使用场景中，有 非常严格的实时性需要，不能够接受Redis时不时的对请求有2毫秒的延迟的话,
    把这项配置为no。
    如果没有这么严格的实时性要求，可以设置为yes,以便能够尽可能快的释放内存

