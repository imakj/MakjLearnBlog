# 租户

* 获取租户列表

  ```
  # ./pulsar-admin tenants list
  ```

* 获取单个租户信息

  ```
  # ./pulsar-admin tenants get my-tenant
  ```

* 创建租户

  pulsar-admin tenants create 租户名称

  ```
  ./pulsar-admin tenants create my-tenant
  
  # 在创建租户时，可以使用-r 或者 --admin-roles标志分配管理角色。可以用逗号分隔的列表指定多个角色。
  # ./pulsar-admin tenants create my-tenant --admin-roles role1,role2,role3
  # ./pulsar-admin tenants create my-tenant -r role1
  ```

* 修改租户信息

  ```
  # ./pulsar-admin tenants update my-tenant -r 'dev'
  ```

* 删除租户

  ```
  # ./pulsar-admin tenants delete my-tenant
  ```



# 命名空间

namespace有两种，分别是本地的namespace和全局的namespace：

* 本地namespace——仅对定义它的集群可见。
* 全局namespace——跨集群可见，可以是同一个数据中心的集群，也可以是跨地域中心的集群，这依赖于是否在namespace中设置了跨集群拷贝数据的功能。

* 创建命名空间 在指定的租户下创建名称空间

  ```
  # ./pulsar-admin namespaces create my-tenant/my-namespace
  ```

* 获取租户下所有的名称空间列表

  ```
  # ./pulsar-admin namespaces list my-tenant
  ```

* 删除命名空间

  ```
  # ./pulsar-admin namespaces delete my-tenant/my-namespace
  ```

* 获取命名空间的配置策略

  ```
  # ./pulsar-admin namespaces policies my-tenant/my-namespace
  {
    "auth_policies" : {  # 授权策略
      "namespace_auth" : { },
      "destination_auth" : { },
      "subscription_auth_roles" : { }
    },
    "replication_clusters" : [ "pulsar-cluster" ],  #副本信息
    "bundles" : {
      "boundaries" : [ "0x00000000", "0x40000000", "0x80000000", "0xc0000000", "0xffffffff" ],
      "numBundles" : 4
    },
    "backlog_quota_map" : { },
    "clusterDispatchRate" : { },
    "topicDispatchRate" : { },
    "subscriptionDispatchRate" : { },
    "replicatorDispatchRate" : { },
    "clusterSubscribeRate" : { },
    "publishMaxMessageRate" : { },
    "latency_stats_sample_rate" : { },
    "deleted" : false,
    "encryption_required" : false,
    "subscription_auth_mode" : "None",
    "offload_threshold" : -1,
    "schema_compatibility_strategy" : "UNDEFINED",
    "is_allow_auto_update_schema" : true,
    "schema_validation_enforced" : false,
    "subscription_types_enabled" : [ ],
    "properties" : { }
  }
  ```

* 配置复制集群信息

  可以将命名空间下的数据复制到其他集群

  ```
  # ./pulsar-admin namespaces set-clusters my-tenant/my-namespace --clusters cl2
  
  # 获取给定命名空间复制集群的列表
  # ./pulsar-admin namespaces get-clusters my-tenant/my-namespace
  ```

* 配置backlog quota 策略

  限制当前命名空间的预知 比如带宽限制 存储限制等 达到阈值后就限制后续消息的行动

  --policy 的值选择:

  * producer_request_hold：broker 暂停运行，并不再持久化生产请求负载
  * producer_exception：broker 抛出异常，并与客户端断开连接。
  * consumer_backlog_eviction：broker 丢弃积压消息

  ```
  # 如下 36000秒内 数据的处理最大为10G 如果超过 则暂停
  # ./pulsar-admin namespaces set-backlog-quota --limit 10G --limitTime 36000 --policy producer_request_hold my-tenant/my-namespace
  ```

* 获取backlog quota 策略

  ```
  # ./pulsar-admin namespaces get-backlog-quotas my-tenant/my-namespace
  ```

* 删除backlog quota 策略

  ```
  # ./pulsar-admin namespaces remove-backlog-quota my-tenant/my-namespace
  ```

* 配置持久化策略

  持久化策略可以为给定命名空间下 topic 上的所有消息配置持久等级

  参数说明:

  * Bookkeeper-ack-quorum：每个 entry 在等待的 acks（有保证的副本）数量，默认值：0 保证多少副本成功后才代表成功
  * Bookkeeper-ensemble：单个 topic 使用的 bookie 数量，默认值：0
  * Bookkeeper-write-quorum：每个 entry 要写入的次数，默认值：0 就是写入多少个副本
  * Ml-mark-delete-max-rate：标记-删除操作的限制速率（0表示无限制），默认值：0.0

  ```
  # ./pulsar-admin namespaces set-persistence --bookkeeper-ack-quorum 2 --bookkeeper-ensemble 3 --bookkeeper-write-quorum 2 --ml-mark-delete-max-rate 0 my-tenant/my-namespace
  ```

* 获取持久化策略

  ```
  $ ./pulsar-admin namespaces get-persistence my-tenant/my-namespace
  ```

* 配置消息存活时间(TTL)

  单位：秒 0表示无限制

  存储到当前namespace的消息的最大存入时间

  ```
  # ./pulsar-admin namespaces set-message-ttl --messageTTL 100 my-tenant/my-namespace
  ```

* 获取消息的存活时间

  ```
  # ./pulsar-admin namespaces get-message-ttl my-tenant/my-namespace
  ```

* 删除消息的存活时间

  ```
  # ./pulsar-admin namespaces remove-message-ttl my-tenant/my-namespace
  ```

* 配置整个名称空间中Topic的消息发送速率

  向某个命名空间中的topic 每秒最大的发送数据数量

  * --msg-dispatch-rate : 每dispatch-rate-period秒钟发送的消息数量
  * --byte-dispatch-rate : 每dispatch-rate-period秒钟发送的总字节数
  * --dispatch-rate-period : 设置发送的速率, 比如 1 表示 每秒钟

  ```
  # 如下 每秒最大发送1000条数据 每条数据不超过1048576
  # ./pulsar-admin namespaces set-dispatch-rate my-tenant/my-namespace \
  --msg-dispatch-rate 1000 \
  --byte-dispatch-rate 1048576 \
  --dispatch-rate-period 1
  ```

* 获取topic的消息发送速率

  ```
  # ./pulsar-admin namespaces get-dispatch-rate my-tenant/my-namespace
  # 也可以这么看
  # ./pulsar-admin namespaces policies my-tenant/my-namespace
  ```

* 配置整个名称空间中Topic的消息接收速率

  * --msg-dispatch-rate : 每dispatch-rate-period秒钟接收的消息数量
  * --byte-dispatch-rate : 每dispatch-rate-period秒钟接收的总字节数
  * --dispatch-rate-period : 设置接收的速率, 比如 1 表示 每秒钟

  ```
  # ./pulsar-admin namespaces set-subscription-dispatch-rate my-tenant/my-namespace \
  --msg-dispatch-rate 1000 \
  --byte-dispatch-rate 1048576 \
  --dispatch-rate-period 1
  ```

* 获取topic的消息接收速率

  ```
  # ./pulsar-admin namespaces get-subscription-dispatch-rate my-tenant/my-namespace
  # 也可以这么看
  # ./pulsar-admin namespaces policies my-tenant/my-namespace
  ```

* 配置整个名称空间中Topic的复制集群的速率

  * --msg-dispatch-rate : 每dispatch-rate-period秒钟复制集群的消息数量
  * --byte-dispatch-rate : 每dispatch-rate-period秒钟复制集群的总字节数
  * --dispatch-rate-period : 设置复制集群的速率, 比如 1 表示 每秒钟

  ```
  # ./pulsar-admin namespaces set-replicator-dispatch-rate my-tenant/my-namespace \
  --msg-dispatch-rate 1000 \
  --byte-dispatch-rate 1048576 \
  --dispatch-rate-period 1
  ```

* 获取topic的消息复制集群的速率

  ```
  # ./pulsar-admin namespaces get-replicator-dispatch-rate my-tenant/my-namespace
  ```

  

# topic

在一个名称空间下, 可以定义多个Topic 通过Topic进行数据的分类划分, 将不同的类别的消息放置到不同Topic,
消费者也可以从不同Topic中获取到相关的消息, 是一种更细粒度的消息划分操作, 同时在Topic下可以划分为多个分片, 进行分布式的存储操作, 每个分片下还存在有副本操作, 保证数据不丢失, 当然这些分片副本更多是由bookkeeper来提供支持

Pulsar 提供持久化与非持久化两种topic。 持久化topic是消息发布、消费的逻辑端点。 

* 持久化topic地址的命名格式如下:

  ```
  persistent://tenant/namespace/topic
  ```

* 非持久topic应用在仅消费实时发布消息与不需要持久化保证的应用程序。 通过这种方式，它通过删除持久消息的开销来减少消息发布延迟。 非持久化topic地址的命名格式如下:

  ```
  non-persistent://tenant/namespace/topic
  ```

* 创建topic

  注意: 不管是有分区还是没有分区, 创建topic后,如果没有任何操作, 60s后pulsar会认为此topic是不活动的, 会
  自动进行删除, 以避免生成垃圾数据

  可以配置broker的配置 来确定是否自动删除:
  Brokerdeleteinactivetopicsenabenabled : 默认值为true 表示是否启动自动删除
  BrokerDeleteInactiveTopicsFrequencySeconds: 默认为60s 表示检测未活动的时间

  * 创建一个没有分区的topic

    ```
    # ./pulsar-admin topics create persistent://my-tenant/my-namespace/my-topic
    ```

  * 创建一个有分区的topic

    ```
    # ./pulsar-admin topics create-partitioned-topic persistent://my-tenant/my-namespace/my-topic --partitions 4
    ```

* 列出当前某个名称空间下的所有Topic

  ```
  # ./pulsar-admin topics list my-tenant/my-namespace
  ```

* 更新Topic操作

  ```
  # 针对有分区的topic去更新其分区的数量
  # ./pulsar-admin topics update-partitioned-topic persistent://my-tenant/my-namespace/my-topic --partitions 8
  ```

* 删除Topic操作

  * 删除没有分区的topic

    ```
    # ./pulsar-admin topics delete persistent://my-tenant/my-namespace/my-topic
    ```

  * 删除有分区的topic

    ```
    # ./pulsar-admin topics delete-partitioned-topic persistent://my-tenant/my-namespace/my-topic
    ```

* topic授权

  ```
  # ./pulsar-admin topics grant-permission --actions produce,consume --role application1 persistent://my-tenant/my-namespace/my-topic
  ```

* 获取topic权限

  ```
  # ./pulsar-admin topics grant-permission --actions produce,consume --role application1 persistent://my-tenant/my-namespace/my-topic
  ```

* 取消权限

  ```
  # ./pulsar-admin topics revoke-permission --role application1 persistent://my-tenant/my-namespace/my-topic
  ```

  

# Function

Pulsar Functions 相当于一个流集群处理

1. 启用function 集群中所有节点修改配文件 启用function

   ```
   # cd /opt/pulsar/puslar/conf
   
   # vim broker.conf 
   ……
   functionsWorkerEnabled=true
   ……
   ```

2. 修改完成后重启broker

   所有节点执行 

   ```
   # 停止 三台节点全部停止 
   # cd /opt/pulsar/puslar/bin && ./pulsar-daemon stop broker
   
   # 启动 
   # cd /opt/pulsar/puslar/bin && ./pulsar-daemon start broker
   ```

3. 测试是否正常 使用自带的测试工具

   * create: 创建function可以下面几个
     * localrun: 创建本地function进行运行
     * create: 在集群模式下创建
     * delete: 删除在集群中运行的function
     * get: 获取function的相关信息
     * restart: 重启
     * stop : 停止运行
     * start: 启动
     * status: 检查状态
     * stats: 查看状态
     * list: 查看特定租户和名称空间下的所有的function
   *  jar：指定运行jar文件
   * classname 启动主函数
   * inputs ：读取的topic
   * output:处理完成后输出的topic 
   * tenant：以哪个租户执行
   * namespace：运行在那个名称空间中
   * name: function的名字

   ```
   # cd /opt/pulsar/puslar
   
   # 如果输出Created successfully则代表成功 最终是运行在某一个节点的
   # ./bin/pulsar-admin functions create \
   --jar examples/api-examples.jar \
   --classname org.apache.pulsar.functions.api.examples.ExclamationFunction \
   --inputs persistent://public/default/exclamation-input \
   --output persistent://public/default/exclamation-output \
   --tenant public \
   --namespace default \
   --name exclamation
   
   #检查是否按照预期的执行正常 相当于向persistent://public/default/exclamation-input写入了一条数据 检查是否在persistent://public/default/exclamation-output有输出
   # bin/pulsar-admin functions trigger --name exclamation --trigger-value "hello world"
   ```

4. 自定义function 组件

   见FormatDateFunction类
   
   

# Connector连接器

兼容了主流的flink mysql等主流的连接器适配器，可以直接从mysql或者flink等工具中倒入数据到pulsar

兼容数据源支持：http://pulsar.apache.org/docs/zh-CN/io-connectors/#source-connector

输出到目标支持：http://pulsar.apache.org/docs/zh-CN/io-connectors/#sink-connecto

详情见com.makj.pulsar.connectorFink包下面demo



# Flume Connector

Flume 就是一个数据采集的方案

1. 安装Flume 

   ```
   # cd /opt/flume/
   # tar zxvf apache-flume-1.9.0-bin.tar.gz 
   # cd apache-flume-1.9.0-bin/conf/
   # cp flume-env.sh.template flume-env.sh
   # 修改java_home环境
   ……
   export JAVA_HOME=/usr/java/jdk1.8.0_333
   ……
   ```

2. github上下载flume源码并且编译

   ```
    # git clone https://github.com/streamnative/flume-ng-pulsar-sink.git
    
    # 编译
    # cd pulsar-flume-ng-sink
    # mvn clean package
   ```

3. 将编译的jar flume-ng-pulsar-sink-1.9.0.jar包上传到/opt/flume/apache-flume-1.9.0-bin/lib下

4. 配置Pulsar的采集文件

   ```
   # vim netcat_source_to_pulsar_sink.conf
   # 输出到哪个目的地管道
   a1.sources = r1 
   a1.channels = c1  
   a1.sinks = k1  
   
   # 基于端口号的监听 
   a1.sources.r1.type = netcat 
   a1.sources.r1.bind = 192.168.10.181
   a1.sources.r1.port = 44444
   
   # 基于内存进行数据传输 
   a1.channels.c1.type = memory
   a1.channels.c1.capacity = 100
   a1.channels.c1.transactionCapacity = 100
   
   # pulsar信息
   a1.sinks.k1.type = org.apache.flume.sink.pulsar.PulsarSink
   a1.sinks.k1.serviceUrl = 192.168.10.181:6650,192.168.10.182:6650,192.168.10.183:6650
   a1.sinks.k1.topicName = persistent://makj_java_tenant/makj_java_ns/flume-test-topic  # 对应的topic信息
   a1.sinks.k1.producerName = flume-test-producer #对应的生产者名字
   a1.sources.r1.channels = c1
   a1.sinks.k1.channel = c1
   ```

5. 运行启动

   ```
   # ./bin/flume-ng agent -n a1 -c ./conf/ -f ./conf/netcat_source_to_pulsar_sink.conf -Dflume.root.logger=INFO,console
   ```

6. 测试

   ```
   # 向44444端口发送数据 通过telnet发送
   # telnet 192.168.10.181 44444
   # 启动监听
   # ./pulsar-client consume -n 0 -s 'flumeconsume' persistent://makj_java_tenant/makj_java_ns/flume-test-topic
   ```



# 事务

生产者发送的消息的消息语义可分为

* 至少一次

* 最多一次

* 精确一次(Exactly-once)

  Pulsar在最新的版本中, 通过事务API 实现跨topic消息生产和确认的原子性操作, 通过这个功能，Producer
  可以确保一条消息同时发送到多个 Topic，要么这些消息都发送成功，在所有 Topic 上都可以被消费，要么所有消
  息都不能被消费。这个功能也允许在一个事务操作中对多个 Topic 上的消息进行 ACK 确认，从而实现端到端的
  Exactly-once 语义

事务语义允许事件流应用将消费，处理，生产消息整个过程定义为一个原子操作。 在 Pulsar 中，生产者或消费者能够处理跨多个主题和分区的消息, 允许一个原子操作写入多个主题和分区, 同时事务中的批量消息可以被以多分区接收、生产和确认, 一个事务涉及的所有操作都作为整体成功或失败。事务中几个概念需要大家了解:

1. 开启事务

   修改pulsar的broker.conf文件: 开启事务支持 3台节点都需要开启

   ```
   # vim conf/broker.conf 
   
   ……
   # 开启事务支持
   transactionCoordinatorEnabled=true
   # 开启批量确认
   acknowledgmentAtBatchIndexLevelEnabled=true
   ……
   ```

2. 初始化事务协调器元数据, 

   初始化到zk集群 指定信息

   输出Transaction coordinator metadata setup success即成功

   ```
   # bin/pulsar initialize-transaction-coordinator-metadata -cs 192.168.10.140:2181,192.168.10.141:2181,192.168.10.142:2181 -c pulsar-cluster
   ```

3. 然后将Pulsar(bookie和Broker)进行重启

   三台节点执行

   ```
   # 停止 三台节点全部停止 
   # cd /opt/pulsar/puslar/bin && ./pulsar-daemon stop broker
   
   # 启动 
   # cd /opt/pulsar/puslar/bin && ./pulsar-daemon start broker
   ```

4. 示例详见demo (com.makj.pulsar.transaction.PulsarTransactionTest)



# kafka适配器

可以最小的无损将kafka的接入信息切换成pulsar

1. 依赖包 删除kafka的包

   ```
   <dependency>
       <groupId>org.apache.pulsar</groupId>
       <artifactId>pulsar-client-kafka</artifactId>
       <version>2.8.0</version>
   </dependency>
   ```

2. 代码详见demo下的com.makj.pulsar.kafkaAdaptor.*



# sprk适配器

1. 导入依赖包

   ```
           <dependency>
               <groupId>org.apache.pulsar</groupId>
               <artifactId>pulsar-spark</artifactId>
               <version>2.8.0</version>
   
               <exclusions>
                   <exclusion>
                       <groupId>org.apache.spark</groupId>
                       <artifactId>spark-streaming_2.10</artifactId>
                   </exclusion>
               </exclusions>
           </dependency>
   
           <dependency>
               <groupId>org.apache.spark</groupId>
               <artifactId>spark-streaming_2.11</artifactId>
               <version>2.1.0</version>
           </dependency>
           <dependency>
               <groupId>com.fasterxml.jackson.core</groupId>
               <artifactId>jackson-annotations</artifactId>
               <version>2.6.1</version>
           </dependency>
   
           <dependency>
               <groupId>com.fasterxml.jackson.core</groupId>
               <artifactId>jackson-databind</artifactId>
               <version>2.6.1</version>
           </dependency>
   
           <dependency>
               <groupId>org.scala-lang</groupId>
               <artifactId>scala-xml</artifactId>
               <version>2.11.0-M4</version>
           </dependency>
   ```

2. 代码详见demo下的com.makj.pulsar.sparkAdaptor.*



# KOP

Pulsar实现了原生的kafka协议，可以直接和kafka兼容，直接无损将kafka迁移到pulsar

1. 下载kop的包

   https://github.com/streamnative/kop/releases/download/v2.9.2.18/pulsar-protocol-handler-kafka-2.9.2.18.nar

   ```
   https://github.com/streamnative/kop/releases/download/v2.8.1.30/pulsar-protocol-handler-kafka-2.8.1.30.nar
   ```

2. 将包上传到Pulsar的protocols目录中 所有节点都要上传

   ```
   # ll /opt/pulsar/apache-pulsar-2.10.0/protocols/
   total 6788
   -rw-r--r-- 1 root root 6949222 Jun  1 17:13 pulsar-protocol-handler-kafka-2.9.2.18.nar
   ```

3. 修改配置文件 所有节点都需要修改

   ```
   # cd /opt/pulsar/apache-pulsar-2.10.0/conf
   # vim broker.conf 
   ……
   
   # 修改以下内容
   allowAutoTopicCreationType=partitioned
   brokerDeleteInactiveTopicsEnabled=false
   # 添加以下内容:
   messagingProtocols=kafka
   protocolHandlerDirectory=./protocols
   # 每个节点的主机名要改成对应节点的ip
   kafkaListeners=PLAINTEXT://192.168.10.181:9092
   kafkaTransactionCoordinatorEnabled=true
   brokerEntryMetadataInterceptors=org.apache.pulsar.common.intercept.AppendIndexMetadataInterceptor
   
   brokerDeduplicationEnabled=true
   kafkaTransactionCoordinatorEnabled=true
   ……
   ```

4. 重启各个Broker节点

   ```
   # cd /opt/pulsar/puslar/bin && ./pulsar-daemon stop broker
   
   # cd /opt/pulsar/puslar/bin && ./pulsar-daemon start broker
   ```

5. 测试

   1. 使用kafka工具测试

      ```
      # tar zxvf kafka_2.12-2.4.1.tgz 
      
      ```

   2. 生产者和消费者测试

      ```
      # cd /opt/kafka/kafka_2.12-2.4.1/bin/
      # ./kafka-console-producer.sh --broker-list 192.168.10.181:90,192.168.10.182:9092,192.168.10.183:9092 --topic kafka_topic
      
      # ./kafka-console-consumer.sh  --bootstrap-server 192.168.10.181:9092,192.168.10.182:9092,192.168.10.183:9092 --topic kafka_topic
      ```

   3. 代码测试 详见org.apache.kafka.clients.consumer.ConsumerRecord.*

# AOP

AoP 是基于 Pulsar 特性实现的。但是，使用 Pulsar 和使用 AMQP 的方法是不同的。以下是 AoP 的一些限制。

* 目前，AoP 协议处理程序支持 AMQP0-9-1 协议，仅支持持久交换和持久队列。
* 一个 Vhost 由一个只能有一个包的命名空间支持。您需要提前为 Vhost 创建一个命名空间。
* Pulsar 2.6.1 或更高版本支持 AoP。

1. 下载aop的包

   https://github.com/streamnative/aop/releases/download/v2.9.2.18/pulsar-protocol-handler-amqp-2.9.2.18.nar

2. 修改配置

   ```
   # cd /opt/pulsar/apache-pulsar-2.10.0/conf
   # vim broker.conf 
   ……
   
   messagingProtocols=kafka,amqp
   protocolHandlerDirectory=./protocols
   # 每个节点的主机名要改成对应节点的ip
   amqpListeners=amqp://192.168.10.181:5672
   
   ……
   ```

3. 重启各个Broker节点

   ```
   # cd /opt/pulsar/puslar/bin && ./pulsar-daemon stop broker
   
   # ./pulsar-daemon start broker
   ```

4. 测试

   ```
   # 添加依赖
   
   ```

   

# pulsar分层存储

单个 Pulsar 集群由以下三部分组成：

* 多个 broker 负责处理和负载均衡 producer 发出的消息，并将这些消息分派给 consumer；Broker 与 Pulsar 配
  置存储交互来处理相应的任务，并将消息存储在 BookKeeper 实例中（又称 bookies 就是负责将数据存储）；Broker 依赖 ZooKeeper
  集群处理特定的任务，等等。
* 多个 bookie 的 BookKeeper 集群负责消息的持久化存储。
* 一个zookeeper集群，用来处理多个Pulsar集群之间的协调任务。

分层存储相当于将热数据存储到内存或者一级缓存中，其他数据存储到hdd中


Pulsar 通过提供分层存储（Apache Pulsar 2.1 起新增的特性）减少了成本/大小的损失。分层存储为用户提供大小不受限制的 backlog，且无需添加存储节点；卸载较旧的 topic 数据到长期存储中，长期存储的成本比在 Pulsar 集群中存储的成本低一个数量级。对于终端用户来说，消费存储在 Pulsar 集群或分层存储中的 topic数据没有明显差别。位于 Pulsar 集群和分层存储中的 topic 生产和消费消息的方式也完全相同。
Pulsar 通过分片架构实现了分层存储。Pulsar topic 的消息日志由一系列分片组成。序列中的最后一个分
片是 Pulsar 当前写入的分片。当前序列之前的所有分片都已封装，也就是说，这些分片中的数据不可变。由于数
据不可变，因此可以轻易地将数据复制到另一个存储系统，而不必担心一致性的问题。复制完成后，可以立即更新
消息日志元数据中的数据指针，并且可以删除 Pulsar 在 Apache BookKeeper 中存储的数据副本。

Pulsar 是基于分片存储的，已经写完的片可以看做一个整体 不会在进行修改，可以将已经写完成的片存入其他存储，或者发送到其他节点进行同步

分层存储只需要在broker.conf中配置卸载地址和路径, 并开启卸载自动运行即可,

https://pulsar.apache.org/docs/en/cookbooks-tiered-storage/

# 生产数据的流程

1. 客户端调用pulsar提供给客户端的API, 进行数据的生产操作, 将生产的消息传递给producer

2. 在生产端内部有一个MessageWriter的类, 基于这个类实现数据分发操作, 默认方案为round-robin(轮询),同时为了提高效率, 在一定的时间内,只会选择一个partition
   除了支持轮询方案外, 如果在传递消息指定key, 会采用hash取模的方式确定要发送到那个partition , 同时pulsar支持自定义分发策略

3. 客户端在此连接broker, 根据要发送的partition获取对应服务的broker节点

4. broker收到消息后调用bookkeeper的客户端并发去写多个副本

5. broker端会等待bookkeeper写入完成, 当broker收到所有副本的ack之后, 会认为这条消息已经写入成功, broker会返回客户端, 告知这条消息已经被持久化完成

   说明: 整个写入操作, 客户端不会跟zookeeper打交道,也不会和bookkeeper打交道, 只需要和broker即可



# 读取数据的流程

Consumer在消费数据时候, 主要有二种情况, 一种为broker中已经缓存了消息, 一种为broker中没有缓存信息

1. 消费者连接broker地址, 根据要读取的对应topic的分配, 确定要连接的最终的broker地址, 如果没有指定分片, 那么就连接每一个分片对应的broker地址
2. 对应的broker首先判断消息是否已经有缓存数据, 如果存在, 直接从内存中采用推的方式发送给消费者, 将消息放置在一个 receiver 队列中,消费端从队列中读取即可, 如果没有缓存,此时broker端通过bookkeeper的客户端到bookie中读取数据(内部可以读取任意副本的数据)
3. 消费者默认是一个订阅的独享模式 不允许多个客户端同时订阅一个单副本的topic 除非拥有多个副本 但是消费者也是一个主多个从 并不是负载均衡的方式 在单副本情况下 单个副本可以拥有备份模式 当主客户端挂机后  备份客户端可以自动顶上

# 读写异常故障处理流程

* 生产端产生失败:
  当出现 [发消息网络断开, broker宕机] 等情况时候, 这个时候producer有 pending 队列, 会在设置的超时时间内进行重试策略
* Broker端出现宕机
  因为broker是没有状态的, 所以它不保存任何数据, 一旦宕机后, topic的管理权会被其他broker掌管, 这个时候, 服务会被快速恢复
* Bookkeeper出现宕机
  存储节点只负责数据存储, bookkeeper本身是一个集群, 故如果只挂掉一个bookie, 并不影响, 所以broker是不会感知的,除非所有的bookie都挂掉, 没有足够的副本去写入数据.
* 消费端产生失败
  一个订阅同时只有一个消费者, 但是可以拥有多个备份消费者, 一旦主消费者故障, 则备份消费者接管, 进行消费即可, 同时pulsar还支持一个分区对应多个消费者, 或者一个消费端对应多个分片的情况
  同时只要消息没有被消费者所消息, 在pulsar中消息就没有变成确认状态, 下次依然是可以再次消费的

# Bookkeepeer

Apache BookKeeper 是企业级存储系统，旨在保证高持久性、一致性与低延迟。

企业级的实时存储平台需要具备的特点:

* 以极低的延迟（小于 5 毫秒）读写 entry 流
* 能够持久、一致、容 错地存储数据
* 在写数据时，能够进行流式传输或追尾传输
* 有效地存储、访问历史数据与实时数据

bookkeeper具有分布式系统提供高可用性或多副本,在单个集群中或多个集群间（多个数据中心）提供跨机器复制,为发布/订阅（pub-sub）消息系统提供存储服务,为流工作存储不可变对象

BookKeeper 复制并持久存储日志流。日志流是形成良好序列的记录流。

Bookkeeper中比较核心的就两个元素: 日志(ledger/stream)和记录(entry)

* 记录(entry)
  数据以不可分割记录的序列，而不是单个字节写入 Apache BookKeeper 的日志。记录是 BookKeeper 中最小的I/O 单元，也被称作地址单元。单条记录中包含与该记录相关或分配给该记录的序列号（例如递增的长数）。客户端总是从特定记录开始读取，或者追尾序列。也就是说，客户端通过监听序列来寻找下一条要添加到日志中的记录。客户端可以单次接收单条记录，也可以接收包含多条记录的数据块。序列号也可以用于随机检索记录。
* 日志(ledger/stream)
  BookKeeper 中提供了两个表示日志存储的名词：一个是 ledger（又称日志段）；另一个是 stream（又称日志流）。Ledger 用于记录或存储一系列数据记录（日志）。当客户端主动关闭或者当充当 writer 的客户端宕机时，正在写入此 ledger 的记录会丢失，而之前存储在 ledger 中的数据不会丢失。Ledger 一旦被关闭就不可变，也就是说，不允许向已关闭的ledger 中添加数据记录（日志）。

* Stream（又称日志流）

  是无界、无限的数据记录序列。默认情况下，stream 永远不会丢失。stream 和 ledger有所不同。在追加记录时，ledger 只能运行一次，而 stream 可以运行多次。
  一个 stream 由多个 ledger 组成；每个 ledger 根据基于时间或空间的滚动策略循环。在 stream 被删除之前，stream 有可能存在相对较长的时间（几天、几个月，甚至几年）。Stream 的主要数据保留机制是截断，包括根据基于时间或空间的保留策略删除最早的 ledger。
  Ledger 和 stream 为历史数据和实时数据提供统一的存储抽象。在写入数据时，日志流流式传输或追尾传输实时数据记录。存储在 ledger 的实时数据成为历史数据。累积在 stream 中的数据不受单机容量的限制。

* 命名空间

  通常情况下，用户在命名空间分类、管理日志流。命名空间是租户用来创建 stream 的一种机制，也是一个部署或管理单元。用户可以配置命名空间级别的数据放置策略。
  同一命名空间的所有 stream 都拥有相同的命名空间的设置，并将记录存放在根据数据放置策略配置的存储节点中。这为同时管理多个 stream 的机制提供了强有力的支持。

* Bookies:
  Bookies 即存储服务器。一个 bookie 是一个单独的 BookKeeper 存储服务器，用于存储数据记录。BookKeeper跨 bookies 复制并存储数据 entries。出于性能考虑，单个 bookie 上存储 ledger 段，而不是整个 ledger。
  因此，bookie 就像是整个集成的一部分。对于任意给定 ledger L，集成指存储 L 中 entries 的一组 bookies。将 entries 写入 ledger 时，entries 就会跨集成分段（写入 bookies 的一个分组而不是所有的 bookies）。





# 跨机房复制(异地灾被)

1. 有三个数据中心：Cluster-A、 Cluster-B、 Cluster-C。用户创建的一个 Topic 主题 T1 设置了跨越三个数据中心做互备。在三个数据中心中，分别有三个生产者：P1、P2、P3，它们往主题 T1 中发布消息；有两个消费者：C1、C2，订阅了这个主题，接收主题中的消息。
2. 当消息由本数据中心的生产者发布成功后，会立即复制到其他两个数据中心。消息复制完成后，消费者不仅可以收到本数据中心产生的消息，也可以收到从其他数据中心复制过来的消息。它的工作机制是在 Broker 内部，为跨地域的数据复制启动了一组内嵌的额外生产者和消费者。当外部消息产生后，内嵌的消费者会读取消息；读取完成后，调用内嵌的生产者将消息立即发送到远端的数据中心。
3. 跨地域复制需要设置“租户”在数据中心之间的访问权限。
4. 在配置了跨地域复制后，每个发送进来的消息，首先被保存在本地集群中；然后异步地推送到远端的集群。如果本地集群和远端集群之间没有网络问题，消息会被立即推送给远端集群。这样端到端的发送延迟主要由集群之间网络的决定。
5. 无论生产者（Producer）P1、P2 和 P3 在什么时候分别将消息发布给 Cluster A、 Cluster B 和 Cluster C 中的主题 T1，这些消息均会立刻复制到整个集群。一旦完成复制，消费者（Consumer）C1 和 C2 即可从自己所在的集群消耗这些消息，并且保持消息在每个 Producer 内部的发送顺序。
6. 因为消息已经从其他远端集群发送到本地集群的 Topic，所以每个集群内部都会保留这个 Topic 中产生的所有消息。对于每个 Consumer 来说，Consumer 的订阅（subscription，维护 Consumer 对 Topic 的消费和 ack 的位置）是针对本地集群的 Topic，相当于 Consumer 消费本地集群的消息。

配置三机房跨机房复制(cluster1, cluster2, cluster3)

1. 首先创建一个租户, 并给予三个数据中心的权限

   ```
   ./pulsar-admin tenants create replication-tenant --allowed-clusters cluster1, cluster2, cluster3
   ```

2. 创建namespace

   ```
   ./pulsar-admin namespaces create replication-tenant/replication-namespace
   ```

3. 设置namespace中topic在那些数据中心之间进行互备

   ```
   ./pulsar-admin namespaces set-clusters replication-tenant/replication-namespace --clusters cluster1, cluster2, cluster3
   ```

4. 新增了数据中心,或者关闭数据中心, 可以随时进行配置调整操作, 而且pulsar表示这样的操作并不会对流量有任何影响

   ```
   ./pulsar-admin namespaces set-clusters replication-tenant/replication-namespace --clusters cluster1, cluster2, cluster3,cluster
   ```


tips

1. python客户端在windows下支持不好

   https://stackoverflow.com/questions/57057858/pulsar-client-couldnt-be-installed