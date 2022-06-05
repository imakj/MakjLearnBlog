# 数据库分类

* 关系型和非关系型
* 关系型
  * SQL关系型
  * NoSQL非关系型
  * NewSQL新式关系型
* 按照运行架构分类
  * 集中式数据库
  * 分布式数据库
* 按照存储分类
  * 行存储
  * 列存储
* 按照业务应用分类
  * OLTP
  * OLAP
  * HTAP

* ob数据库为分布式的OLTP的数据 支持分布式事务



# 特点

高可用和可扩展

* 高可用

  数据库将数据以多副本的方式存储在集群的各个节点，轻松实现高可用，保证PRO=0 异地多活，也可以满足异地容灾 客服传统数据的主备模式在节点出现异常

* 可扩展

  可以在线平滑扩容和缩容，扩容后自动实现负载均衡，且扩容缩容对系统透明

* 低成本

  不依赖高端硬件，给予LSM-Tree的存储引擎，对数据进行压缩不影响性能

  LSM-Tree 一种分层 有序 面向磁盘的数据结构，首先将数据写到内存然后批量顺序写入磁盘，等到积累到一定的数据后，然后刷到磁盘

* HTAP

  引擎对OLTP和OLAP应用进行优化，并且支持跨书记节点的DQL和DML并发，实现了引擎同事支持混合负载

* 兼容性

  社区版兼容mqsql 企业版兼容oracle

* 多租户

  通过租户实现资源隔离(就是mysql中的数据库) 每个数据库服务的实例感知不到其他实例的存在 并且可以通过权限控制确保不同租户数据的安全性，于扩展性相结合 提供安全灵活的DBaas服务

# 工具

* OceanBase Database Proxy

  数据库代理软件，ob专用的连接代理软件，核心功能为保证最佳路由，避免分布式事务(比如第一次连接到哪个节点 以后都是)，保护高可用能力，单台故障不影响，其实就是一个负载均衡软件

* OceanBase Deployer

  安装部署工具，简称OBD，同时也是包管理器，用来管理OceanBase

* OCP

  社区版本的运维监控工具，是ob的数据库集群管理平台工具，就是Oracle的OEM

* ODC（开发者中心）

  就是ob的管理sql调用工具

* OMS（数据迁移工具）

  OMS支持异构数据库在线不停服迁移到OceanBase数据库，同事切换ob数据库后，可将ob数据库上所有的变更数据实时同步到切换前的源端数据库

* OAT(OceanBase Admin Toolkit)

  进一步方便OCP OMS ODC的自动化部署工具 java开发的一个web程序

* OBClient

  一个交互和批处理的工具，就是一个黑屏的命令行工具(黑屏是客户端 白屏是图形界面)

* OBLogProxy

  ob的增量日志代理服务 是OM的一部分 oblogproxy给予liboblog 以服务的形势提供实时增量链路接入和管理，方便ob增量日志

* (OBLOADER/OBDUMPER) 数据库导入导出工具

  实现ob的快速导入导出

* OBAgent

  监控采集框架，支持promenthes的服务

* od_admin

  配套的运维工具之一，提供了slog_took archive_tool clog_tool dumpsst和dump_backup的功能，主要用于排查数据不一致 丢数据 错误数据等问题

* CDC（Change Data Capture）

  变更数据库捕获，帮助从上次提供后发生的变化 比如历史库 实时缓存 MQ做审计分析

* ODP（OceanBase Database Proxy）

  是ob专用的代理服务器 数据库用户以多副本的形式存放在各个OBServer上，ODP接受用户发出的SQL请求，将SQL请求转发至最佳目标的OBServer 最后将执行结果返回给用户

  ODP具有以下特性（如图）

  * 高性能转发

    采用多线程异步框架和透明流式转发设计，保证了数据的高性能转发，同时确保了ODP对及其资源的最小消耗

  * 最佳路由

    ODP充分考虑用户请求设计的副本位置 用户配置的读写分离路由策略 多低部署的最优链路 已经各个机器状态和负载情况，将用户请求转发到最佳的OBServer上，

    路由策略

    * 强一致性读 默认策略需要读取partition主的数据
      * 默认强一致读 即写后读立即可见(READ AFTER WRITER) 强一致读语句 OBPROXY会优先将sql路由到访问表的分区的主副本节点上
      * 如果sql溶蚀访问了两个表，会依据第一个表及其条件判断该分区主副本节点 如果得不到 就随机发 所以sql多表连接时 表的前后顺序对路由策略有影响 间接对性能有有影响
      * 如果判断表是分区表，会看条件是否是分区键等值条件 如果不是 则不确定哪个是分区 就随机发到该表所在分区的任意一个节点
      * 如果开启了事务 则事务里的sql的路由节点会作为事务后面其他sql路由节点目标 直到事务结束
    * 弱一致性读 默认备用策略 备优先度策略 读写分离策略
      * 弱一致性读不要求写后读立即可见 弱一致性读可以路由到分区的主副本和备副本节点 通常有三个副本所在节点可选
      * 开启弱一致性读后 如果observer和obproxy都开启了ldc(逻辑数据中心(Logical Data Center )路由) 特性 那么弱一致性读语句的路由策略就是优先路由到同一个机房或者同一个Region状态的并且不是合并中的节点，其次是合并中节点，最后是其他region的不在合并的或者合并的节点

  * 连接管理

  * 针对一个客户端的物理连接 odp维持自身到后端多odp 采用对会话变量进行版本管理 此阿勇增量同步方案保证每个observer连接的会话一致性 保证了客户端高效访问各个observer

    * 在observer 宕机 升级 重启时 客户端于obproxy的连接不会断开 obproxy可以迅速地切换到正常的server上 对应用透明
    * obproxy支持用户通过同一个obproxy访问多个ob集群
    * server session对于每个client session独占
    * 同一个client session对应server session状态保持相同(session变量同步)

  * 专有协议

    默认采用了专有协议 如增加报文的crc校验 保证于observer链路正确性 增强传输协议支持oracle兼容性的数据类型和交互模型

  * 易运维

    无状态的odp支持无线水平扩展 支持同事访问多个集群 可以通过丰富的内部命令对odp状态实时监控 

    * 周期性汇报统计到ocp 实现了语句级别 事务级别 session级别 obproxy级别的各种统计
    * Xflush日志监控
    * Sql Audit 功能
    * 实现了大量内部命令来实现远程兼容 查询和运维

# 概念

* ob必须是集群 没有单节点概念

* 一个ob集群有多个Zone组成，Zone个数大于3个 

  * 每个Zone又包含多个ob服务器(一个ob服务器就是一个observer进程)(Zone：就是一个区)，比如 9台集群服务 一个三个zone每个zone下有三台机器 每个集群是16c16g 那么集群可用资源就16c16g*3+16c16g*3+16c16g*3 此时架构就是3-3-3
  * 如果有三个zone 每个zone下一台机器 就是1-1-1

* 一般情况下各个Zone内的机器配置数量保持一直，多台ob服务器作为资源组成各个业务所需的资源池

* 管理员可以根据业务情况，将资源划分不同大小资源池分配给租户用 

* 租户在资源池创建表，库 分区等等

* OceanBase集群 Zone和OceanBase服务器关系

  * 一个集群由多个Zone组成，每份数据在各个Zone上都会有一份副本(全功能副本或者日志副本)，并且只能有一份副本，这样一个Zone故障后不会影响业务正常运行
  * 物理上不同的Zone可以对应不同区域或者城市(Region) 也可以对应一个城市的不同机房，还可以对应一个机房的不同机架，实现不同级别的容灾
  * Zone个数必须大于等于3 因为是采用Paxos协议，多数派需要达成一致 3个以以上的Zone可以保证当1个Zone故障后，剩下的2个Zone依然可以构成多数派，所以扩容需要奇数
  * 同一个分区的数据会分布在多个Zone上，每个分区的主副本所在的服务器被称为Leader 所在的Zone被称为PrimayZone 如果不设定PrimaryZone 系统会根据负载均衡的策略 在多个全功能副本里自动选择一个作为Leader
  * 每个Zone会提供两种服务，总控服务(RootService)和分区服务(PartitionService)
  * 每个Zone有一台OBServer会同事使用总控服务，整个集群只存在一个总控服务，总控服务负载整个集群的资源调度 资源分配 数据分布信息管理移机Schema管理
  * 数据库采用Shared-Nothing(非共享)架构，各个节点之前完全对等，每个节点都有自己的SQL引擎，事务引擎，存储引擎，也会有部分数据，用户的SQL查询经过SQL引擎解析优化后转化为事务引擎和存储引擎内部调用
  * 每个OB服务器均可以看做一台传统的集中式数据库，业余访问到这台OB服务器后，如果需要访问到其他OB服务器，会自动调度协商 对业务无感知
  * 对于跨服务器操作，OB还会执行强之一的分布式事务，从而在分布式集群是实现事务的ACID

* 资源池和租户

  * 集群的多个服务器组成一个大的资源池，管理员会根据各个租户的要求，创建与之对应的虚拟资源池给租户用 资源池包括指定规格的cpu 内存 存储 tps qps等等

  * 为了避免租户之间争抢资源，租户之间的资源相互隔离，内存是物理隔离，cpu是逻辑隔离

  * 租户相当于一个mysql实例，系统租户保持到系统表，系统租户id在1000以下，以上的是业务租户

  * 社区版只支持mysql的租户模式

  * 租户的硬件资源是以资源池(Resource Pool)进行描述的，资源池是由资源单元(Resource Unit)组成

    *  资源单元：对应cpu 内存 存储 iops这几个物理资源，是数据库服务内部的资源容器
    * 单元配置(Resource Unit Config)

    描述资源单元的配置信息 每一个资源单元配置绘描述一种规格的CPU 内存 存储空间 iops等

    * 资源池(Resource Pool)

      具有相同资源单元配置的若干个资源单元组成 一个资源池只能属于一个租户 一个租户可以拥有一个或者多个资源池，这些资源池集合描述了这个租户所能使用的所有物理机资源，一个租户在同一个服务器最多有一个资源单元

  * 资源单元的均衡

    会将一个租户的资源单元在集群的服务器之间均衡，一个租户拥有一个或者多个资源池描述的多个资源单元，每个Zone内的资源单元基本均衡策略如下

    * 当一个租户的若干个资源单元 会均匀分布在不同的服务器上
    * 当服务器存储使用率超过一个阈值时 通过交换或迁移资源单元，降低存储使用率水位线
    * 根据资源单元的cpu和内存规格，交换或者迁移单元 降低cpu和内存的平均水位线

* 存储引擎

  * 存储引擎给予LSM-Tree的架构 把基线数据和增量数据分别保持在磁盘(SSTable)和内存(MemTable)中，具备读写分离的特点，对数据的修改都是增量数据，只写内存，所以DML是完全的内存操作 性能非常高
  * 读的时候 数据可能会在内存里有更新过的版本，在持久化存储里有基线版本们需要把两个版本合并，获得一个新的版本

* SQL引擎

  * SQL引起是整个数据的数据计算中枢和传统数据类似 整个引擎分为解析器 优化器 执行器三部分 区别在于分布式数据库里，查询优化器会依据数据的分布信息生成分布式执行计划
  * 当引擎接收到SQL请求后 经过语法解析 语义分析 查询重写 查询优化等一系列过程 再由执行器负责执行
  * 查询优化器做了很多优化 如算子下推 只能连接 分区裁剪 等
  * 如果SQL的设计的数据量很大，查询执行引擎也做了并行处理，任务拆分，动态分区，流水调度，任务裁剪 只任务合并 并发限制等优化

* 高可用 可靠性

  * 当节点发生宕机或者意外中断时候 能够自动恢复数据的可用性 减少业务受影响的时间，避免因为业务数据节点故障导致终端
  * 当少数节点故障无法读取的时候 保证业务不丢失
  * 不同于传统的主备或者一主多从 使用性价比较高 可靠性略低的服务器 同一数据保存在多台(>=3)服务器的半数服务器以上(例如三台中的2台 5台中的3台) 每一笔写事务必须达到半数以上的服务器才能生效 因此当少出服务故障不会有任何数据丢失
  * 采用Paxos协议，主库宕机后，剩余服务器会很快选出主库，继续提供服务
  * 以分区为单位组件Paxos协议组 每个分区都有多副本 通过维护成员组关系 自动监理Paxos组 同事以分区为单位的Paxos协议组基础上 自动选举主副本 Paxos协议组每个副本在正常工作时分为两种不同角色
    * Leader：主副本 数据库服务的强一致性读移以及写都是由Leader提供 在此基础上提供分布式事务等多方面能力
    * Follower：从副本 为Leader同步数据进行投票 提供弱一致性的读服务

* Redo日志

  * 用于宕机恢复以及维护多副本数据一致性的关键组件
  * 会自动负责redo日志的管理控制 包括创建日志文件 日志文件在多个副本之间同步 日志文件复用 宕机恢复等
  * redo日志分为两部分
    * clog：全程commit log 记录redo日志的日志内存
    * ilog：全程 index log 记录相同分区相同log id的已经形成多数派日志的commit log位置信息

* 架构图，如图

* 部署模式 如图

# 云平台

可以直接使用ocp部署 

分为六个模块

* 管理Agent(Manager Agent)

  通常安装在计算环境中每台受监视的主机 这些代理程序通过ocp管理控制台进行统一部署和升级 用于控制目标主机的启停 远程执行任务和收集指标等 然后将信息提供给云服务管理凭条

* 管理服务(Manager  Service)

  给予java的大型应用 管理angt和元信息数据库通信 已便收集和存储相关远程主机的信息 此服务还可以于ob集群通讯用于远程执行运维命令

* 元信息数据库(Manager Repository)

  可称为元信息库或者MateDB 用于存储管理Agent程序收集的所有信息 元信息数据库存放目标主机 数据库集群 租户 数据库实例 数据库用户 调度任务和软件版本等信息 安装ocp前 要求元信息库存在

* 监控数据库(Monitor Repository )

  称为元信息或者MonitorDB 用于存储OCP采集的监控数据 存放了主机 集群 租户 会话 sql等性能指标 统计和诊断信息等

* 管理控制台(Manager Console)

  就是一个web界面

* OBProxy(OceanBase 专用反向代理)

  是用户连接到ob的代理服务 负责将ocp管理程序向数据库发送的各种请求路由到云信息集群 并将返回的信息发送给ocp管理服务

架构图



# 目录结构

* 查看目录结构

  ```
  $ tree -L 3 --filelimit 10 /home/admin/obproxy/ /home/admin/oceanbase-ce/ /data/ /redo/
  /home/admin/obproxy/
  |-- bin
  |   `-- obproxy -> /home/admin/.obd/repository/obproxy/3.2.0/8d5c6978f988935dc3da1dbec208914668dcf3b2/bin/obproxy
  |-- control-config
  |-- etc  # 配置文件
  |   |-- obproxy_config.bin
  |   `-- obproxy_config.bin.old
  |-- lib
  |-- log [11 entries exceeds filelimit, not opening dir]
  |-- obproxyd.sh
  |-- run
  |   |-- obproxy-192.168.10.160-2883.pid
  |   `-- obproxyd-192.168.10.160-2883.pid
  `-- sharding-config
  /home/admin/oceanbase-ce/
  |-- admin  
  |-- bin
  |   `-- observer -> /home/admin/.obd/repository/oceanbase-ce/3.1.2/7fafba0fac1e90cbd1b5b7ae5fa129b64dc63aed/bin/observer
  |-- etc  # 三个配置文件 2和3是备份 是一个二进制的配置文件
  |   |-- observer.config.bin
  |   `-- observer.config.bin.history
  |-- etc2
  |   |-- observer.conf.bin
  |   `-- observer.conf.bin.history
  |-- etc3
  |   |-- observer.conf.bin
  |   `-- observer.conf.bin.history
  |-- lib
  |   |-- libaio.so -> /home/admin/.obd/repository/oceanbase-ce/3.1.2/7fafba0fac1e90cbd1b5b7ae5fa129b64dc63aed/lib/libaio.so
  |   |-- libaio.so.1 -> /home/admin/.obd/repository/oceanbase-ce/3.1.2/7fafba0fac1e90cbd1b5b7ae5fa129b64dc63aed/lib/libaio.so.1
  |   |-- libaio.so.1.0.1 -> /home/admin/.obd/repository/oceanbase-ce/3.1.2/7fafba0fac1e90cbd1b5b7ae5fa129b64dc63aed/lib/libaio.so.1.0.1
  |   |-- libmariadb.so -> /home/admin/.obd/repository/oceanbase-ce/3.1.2/7fafba0fac1e90cbd1b5b7ae5fa129b64dc63aed/lib/libmariadb.so
  |   `-- libmariadb.so.3 -> /home/admin/.obd/repository/oceanbase-ce/3.1.2/7fafba0fac1e90cbd1b5b7ae5fa129b64dc63aed/lib/libmariadb.so.3
  |-- log [17 entries exceeds filelimit, not opening dir]
  |-- run  #运行文件
  |   |-- mysql.sock
  |   `-- observer.pid
  `-- store -> /data
  /data/
  |-- clog -> /redo/clog
  |-- ilog -> /redo/ilog
  |-- slog -> /redo/slog
  `-- sstable
      `-- block_file
  /redo/
  |-- clog
  |   |-- 1
  |   |-- 2
  |   |-- 3
  |   |-- 4
  |   |-- 5
  |   `-- 6
  |-- ilog
  |   `-- 5
  `-- slog
      `-- 1
  
  ```

# 数据类型介绍

文档：https://www.oceanbase.eom/docs/oceanbase-database/oceanbase-database/V3.2.2/about-sql-data-types
OceanBase社区版兼容MySQL数据库常用语法，具体主要是5.5/5.6常用语法。
OceanBase数据类型有

* 数值类型
  整数类型：BOOL/BOOLEAN、TINYINT、SMALLINT、MEDIUMINT、INT/INTEGER、BIGINT
  定点类型：DECIMAL/NUMERIC
  浮点类型：FLOAT、DOUBLE
  Bit-Value 类型：BIT

* 日期时间类型 DATETIME、 TIMESTAMP. DATE、TIME、YEAR

* 字符类型
  VARCHAR. VARBINARY、CHAR、BINARY、enum、set
  大对象类型
  TINYTEXT、TINYBLOB. TEXT、BLOB、MEDIUMTEXT. MEDIUMBLOB、LONGTEXT、LONGBLOB

* OceanBase的大对象目前大小不能超过48MB。
  与MySQL数据库对比，OceanBase数据库暂不支持空间数据类型和JSON数据类型,
  其他类别的数据类型除了大对象外，支持情况是等于或大于MySQL数据库的。

# OceanBase数据库SQL基础介绍

* 查询语句：SELECT
  支持大部分查询功能，包括：
  支持单、多表查询；
  支持子查询；
  支持内连接、半连接以及外连接；
  支持分组、聚合；
  常见的概率、线性回归等数据挖掘函数等
  支持如下集合操作union、union all. intersect x minus
  不支持SELECT ... FOR SHARE ...语法

* DDL数据定义语言
  create
  alter
  drop
  truncate

* DML数据操作语言:
  insert
  delete
  update
  REPLACE INTO

# 分区表技术

分区技术(Partitioning)是OceanBase非常重要的分布式能力之一，
它能解决大表的容量问题和高并发访问时性能问题，主要思想就是将大表拆分为更多更小的结构相同的独立对象，即分区普通的表只有一个分区，可以看作分区表的特例。
每个分区只能存在于一个节点内部，分区表的不同分区可以分散在不同节点上。
通常当表的数据量非常大，以致于可能使数据库空间紧张，或者由于表非常大导致相关SQL查询性能变慢时，可以考虑布

* 分区路由

  OceanBase的分区表是内建功能，您只需要在建表的时候指定分区策略和分区数即可。
  分区表的查询SQL跟普通表是一样的，OceanBase的OBProxy或observer会自动将用户SQL路由到相应节点内，因此，分区表的分区细节对业务是透明的。
  如果知道要读取的数据所在的分区号，可以通过SQL直接访问分区表的某个分区。
  简单语法格式如下：

  ```
  part_table partition ( p[0,l,...J [sp[0,l,..)
  ```

  默认情况下，除非表定义了分区名，分区名都是按一定规则编号，例如：
  一级分区名为：p0 , pl , p2 ,...
  默认情况下，除非表定义了分区名，分区名都是按一定规则编号，例如：
  一级分区名为：p0 , pl , p2 ,...
  二级分区名为：p0sp0 , p0spl , p0sp2 , ... ; plsp0 , plspl , plsp2 ,
  示例：访问分区表的具体分区。

  ```
  select * from tl partition (p0);
  select * from tl partition (p5sp0);
  ```

  分区策略
  OceanBase支持多种分区策略:

  * Range分区(范围分区)
  * Range Columns 分区
  * llist分区
  * List Columns 分区
  * Hash分区
  * Key分区
  * 组合分区
