# 准备环境

* 三节点
  * 192.168.10.140 (zookeeper1)
  * 192.168.10.141 (zookeeper2)
  * 192.168.10.142 (zookeeper3)

* 版本：apache-zookeeper-3.7.1-bin.tar
* jdk:jdk-8u333
* 上传安装包到三节点的/opt/zookeeper目录
* 停止/关闭防火墙

# 配置环境

配置hostname 三台节点配置

```
# echo "192.168.10.140 zookeeper1" >> /etc/hosts
# echo "192.168.10.141 zookeeper2" >> /etc/hosts
# echo "192.168.10.142 zookeeper3" >> /etc/hosts
```

# 安装

1. 解压 三台节点全部执行

   ```
   # tar zxvf apache-zookeeper-3.7.1-bin.tar.gz 
   # ln -s ./apache-zookeeper-3.7.1-bin ./zookeeper
   ```

2. 创建data目录 三台节点全部执行

   ```
   # mkdir -p /data/zookeeper
   ```

3. 创建 myid 文件

   ```
   # zk1节点
   [root@zookeeper1 conf]# cd /data/zookeeper/
   [root@zookeeper1 zookeeper]# echo 1 > myid
   [root@zookeeper1 zookeeper]# cat myid 
   1
   
   # zk2节点
   [root@zookeeper1 conf]# cd /data/zookeeper/
   [root@zookeeper1 zookeeper]# echo 2 > myid
   [root@zookeeper1 zookeeper]# cat myid 
   2
   
   # zk3节点
   [root@zookeeper1 conf]# cd /data/zookeeper/
   [root@zookeeper1 zookeeper]# echo 3 > myid
   [root@zookeeper1 zookeeper]# cat myid 
   3
   ```

   

4. 进入到conf目录 修改配置文件 三台节点都需要修改

   ```
   # cd /opt/zookeeper/zookeeper/conf/
   # cp zoo_sample.cfg zoo.cfg 
   
   #添加dataDir和添加server
   # vim zoo.cfg
   ……
   dataDir=/data/zookeeper
   # server.1的1 代表每台节点的myid中数值 2888：访问zookeeper的端口；3888：重新选举leader的端口
   server.1=zookeeper1:2888:3888
   server.2=zookeeper2:2888:3888
   server.3=zookeeper3:2888:3888
   ……
   
   # 完整配置文件
   # vim zoo.cfg 
   
   # The number of milliseconds of each tick
   tickTime=2000
   # The number of ticks that the initial 
   # synchronization phase can take
   initLimit=10
   # The number of ticks that can pass between 
   # sending a request and getting an acknowledgement
   syncLimit=5
   # the directory where the snapshot is stored.
   # do not use /tmp for storage, /tmp here is just 
   # example sakes.
   dataDir=/data/zookeeper
   # the port at which the clients will connect
   clientPort=2181
   # the maximum number of client connections.
   # increase this if you need to handle more clients
   #maxClientCnxns=60
   #
   # Be sure to read the maintenance section of the 
   # administrator guide before turning on autopurge.
   #
   # http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
   #
   # The number of snapshots to retain in dataDir
   #autopurge.snapRetainCount=3
   # Purge task interval in hours
   # Set to "0" to disable auto purge feature
   #autopurge.purgeInterval=1
   
   ## Metrics Providers
   #
   # https://prometheus.io Metrics Exporter
   #metricsProvider.className=org.apache.zookeeper.metrics.prometheus.PrometheusMetricsProvider
   #metricsProvider.httpPort=7000
   #metricsProvider.exportJvmInfo=true
   server.1=zookeeper1:2888:3888
   server.2=zookeeper2:2888:3888
   server.3=zookeeper3:2888:3888
   ```

5. 启动

   三台节点进入cd /opt/zookeeper/zookeeper/目录 并且启动

   ```
   # cd /opt/zookeeper/zookeeper/
   # ./bin/zkServer.sh start
   # 查看状态是否启动城关
   # ./bin/zkServer.sh status
   
   # 停止命令
   # ./bin/zkServer.sh stop
   
   #根据日志看到 节点2被选为主
   # ./bin/zkServer.sh status
   ZooKeeper JMX enabled by default
   Using config: /opt/zookeeper/zookeeper/bin/../conf/zoo.cfg
   Client port found: 2181. Client address: localhost. Client SSL: false.
   Mode: follower
   
   # ./bin/zkServer.sh status
   ZooKeeper JMX enabled by default
   Using config: /opt/zookeeper/zookeeper/bin/../conf/zoo.cfg
   Client port found: 2181. Client address: localhost. Client SSL: false.
   Mode: leader
   
   # ./bin/zkServer.sh status
   ZooKeeper JMX enabled by default
   Using config: /opt/zookeeper/zookeeper/bin/../conf/zoo.cfg
   Client port found: 2181. Client address: localhost. Client SSL: false.
   Mode: follower
   ```

# 测试

客户端链接任意一个节点 所有的写入(增删改)都会转发到主节点进行操作，其他查询都是在从节点

```
# 连接单节点
# bin/zkCli.sh -server 192.168.10.140:2181
[zk: 192.168.10.140:2181(CONNECTED) 0] ls /
[zookeeper]
[zk: 192.168.10.140:2181(CONNECTED) 5] create /test
Created /test
[zk: 192.168.10.140:2181(CONNECTED) 6] ls /

#连接集群 干掉任何一个节点 集群正常
# bin/zkCli.sh -server 192.168.10.140:2181,192.168.10.141:2181,192.168.10.142:2181
[zk: 192.168.10.140:2181,192.168.10.141:2181,192.168.10.142:2181(CONNECTED) 0] ls /
[test, zookeeper]
```

