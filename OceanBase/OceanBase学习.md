#  集群管理和租户管理

* 如果业务系统要使用oceanbase 必须创建租户 分为三步

1. 创建资源单元规格(可选)：如果有合适的规格可以复用 就不用了
2. 创建资源池：可以每个zone创建一个资源池，使用独立的资源单元规格，也可以所有zone使用同一个资源单元规格，都在一个资源池下
3. 创建租户：关联到这个资源池

逻辑资源标识

(1-1-1)就是每个zone下面一个副本占用一个机器

(3-3-3)就是每个zone三个副本占用三台机器

一个zone下面有三个节点机器

* 系统中包含两大类租户：系统租户和普通租户

  * 系统租户

    系统租户属于系统内置租户 主要有三个功能

    1. 系统表的容器：所有系统表都存放在系统租户的空间中

    2. 具备集群管理功能的用户的容器：集群级别的管理功能，不如增加 删除租户 修改系统配置项 每日合并等操作  只允许系统租户下的用户来做
    3. 提供执行系统维护和管理行为所需的资源，像选主 日志同步 每日合并的灯操作没有按租户分离，这些操作所需的资源都系统租户统一提供

  * 普通租户

    相当于一个mysql数据库实例，由系统租户根据需要创建

    在创建租户的时候 除了指定租户名字外 最重要的还是指定占用资源

    普通租户具有一个实例所有的特性

    1. 可以创建自己的用户
    2. 可以创建数据库，表 等对象
    3. 有自己独立的information_schema等系统数据库
    4. 有自己独立的系统变量
    5. 数据库实例所具备的所有其他特性

# 资源单元

创建资源单元并不会立即分配资源，创建的单元规格元数据在__all_unit_config里 就是每个节点的最小资源单元 资源单算算是最小的用户资源单位

https://open.oceanbase.com/docs/observer-cn/V3.1.0/10000000000017096

语法如下

```
# 创建
CREATE RESOURCE UNIT [unit_name]
  MAX_CPU [=] {INT_VALUE | DECIMAL_VALUE},
  | MAX_MEMORY [=] STORAGE_SIZE,
  | MAX_IOPS [=] INT_VALUE,
  | MAX_DISK_SIZE [=] STORAGE_SIZE,
  | MAX_SESSION_NUM [=] INT_VALUE
  | MIN_CPU [=] {INT_VALUE | DECIMAL_VALUE}
  | MIN_MEMORY [=] STORAGE_SIZE
  | MIN_IOPS [=] INT_VALUE

# 修改
ALTER RESOURCE UNIT [unit_name]
  MAX_CPU [=] {INT_VALUE | DECIMAL_VALUE},
  | MAX_MEMORY [=] STORAGE_SIZE,
  | MAX_IOPS [=] INT_VALUE,
  | MAX_DISK_SIZE [=] STORAGE_SIZE,
  | MAX_SESSION_NUM [=] INT_VALUE
  | MIN_CPU [=] {INT_VALUE | DECIMAL_VALUE}
  | MIN_MEMORY [=] STORAGE_SIZE
  | MIN_IOPS [=] INT_VALUE

```

1. 创建资源单元

   ```
   create resource unit ocp_unit_config min_cpu=2, max_cpu=2, min_memory='1G', max_memory='1G',max_iops=1000, min_iops=128, max_disk_size='10G', max_session_num=100;
   ```

2. 查看单元

   ```
   select * from __all_unit_config;
   ```

3. 修改单元 可以修改cpu 内存 存储空间和ops的选项 语法如下 必须使用root用户登录sys租户

   ```
   alter resource unit ocp_unit_config min_cpu=3, max_cpu=3, min_memory='1G', max_memory='1G',max_iops=1000, min_iops=128, max_disk_size='10G', max_session_num=100;
   ```

4. 删除单元 如果单元已经被指定给资源池 则需要为源资源池指定新的资源单元后 在删除

   ```
   alter resource pool pool1 UNIT='ocp_unit_config'
   drop resource unit ocp_unit_config;
   ```

# 资源池

资源池从节点的资源中分配 资源池在每个节点里的部分被称为"资源单元" 每个资源单元的大小通过创建的时候指定资源单元的规格决定 资源在视图 oceanbase.__all_resource_pool 资源池和资源单元就是为了满足租户资源隔离和负载均衡而存在的。 一个资源池可以有多个资源单元，根据资源单元的个数 会在每个zone的节点下创建对应个数的资源单元 比如一个资源池有1个资源单元 并且是三个zone 那么在1-1-1环境下 每个zone下的节点都会有一个资源单元被占用

* 语法

  ```
  CREATE RESOURCE POOL poolname 
        UNIT [=] unit_name
      | UNIT_NUM [=] INT_VALUE
      | ZONE_LIST [=] zone_list
      
  ALTER RESOURCE POOL poolname 
        UNIT [=] unit_name
      | UNIT_NUM [=] INT_VALUE
      | ZONE_LIST [=] zone_list
  ```

* 创建资源池 指定资源单元为unit_makj1，并且有三个zone 生产的zone必须大于2

  ```
  create resource pool pool_nakj1 unit='unit_makj2',unit_num=1,zone_list=('zone1','zone2','zone3');
  create resource pool pool_nakj2 unit='unit_makj2',unit_num=1,zone_list=('zone1','zone2');
  ```

* 查看资源池

  ```
  select * from oceanbase.__all_resource_pool
  ```

* 修改资源池所属的资源单元

  ```
  alter resource pool pool_nakj1 unit='unit_makj1';
  ```

* 修改资源池每个zone下的资源单位为2个 必须有足够的空间

  ```
  alter resource pool pool_nakj1 unit_num=2;
  ```

* 删除指定资源池中的unit_id为1001 1002 1003的资源单元 使每个zone下的资源单元个数为1个

  ```
  alter resource pool pool_nakj2 unit_num 1 delete unit=(1001,1002,1003);
  ```

* 修改资源池 使资源池中的zone_list范围扩大到3个 注意 每次只能扩一个 比如第一次是1个 扩展只能扩展两个 第三次在扩展三个

  ```
  alter resource pool pool_nakj2 zone_list=('zone1','zone2','zone3');
  ```

# 资源池合并

可以将租户内的相同的资源池的配置的多个资源吃合并为一个资源池,合并资源池的时候不会影响资源池的租户使用，仅在rootservice的管理层面看来，合并多个资源池是为了统一维护管理科

合并资源池的限制规则：

* 被合并的资源池的unit_num需要相等
* 被合并的资源池的资源单元配置必须是同一个 
* 并且zone没有冲突

* 语法

  ```
  ALTER RESOURCE POOL MERGE ('pool_name'[,'pool_name' ...]) INFO ('merge_pool_name')
  ```

* 合并资源池 合并的时候 两个资源池的zone必须不冲突

  ```
  select * from oceanbase.__all_resource_pool;
  alter resource pool pool_nakj2 zone_list=('zone1','zone2');
  alter resource pool merge ('pool_nakj2','pool_nakj3') into ('pool_merge');
  ```

* 分裂资源池

  将资源池pool_merge分裂为pool_nakj4 pool_nakj5 并且为新的资源池指定新的资源单元·

  ```
  alter resource pool pool_merge split into ('pool_nakj4','pool_nakj5') on ('zone1','zone2','zone3');
  alter resource pool pool_nakj4 UNIT='unit_makj2'
  alter resource pool pool_nakj5 UNIT='unit_makj2'
  ```

* 删除资源池  先查看要删除的资源池是否被租户使用 然后再删除

  ```
  select * from oceanbase.gv$unit;
  select tenant_id,tenant_name,resource_pool_name from oceanbase.gv$unit;
  drop resource pool pool_merge;
  select * from oceanbase.__all_resource_pool;
  ```

* 查看资源分配

  ```
  select t1.name resource_pool_name,
         t2. name unit_config_name,
         t2.max_cpu,
         t2.min_cpu,
         t2.max_memory / 1024 / 1024 / 1024 max_mem_gb,
         t2.min_memory / 1024 / 1024 / 1024 min_mem_gb,
         t3.unit_id,
         t3.zone,
         concat(t3.svr_ip, ':', t3. svr_port) observer,
         t4.tenant_id,
         t4.tenant_name
    from __all_resource_pool t1
    join __all_unit_config t2
      on (t1.unit_config_id = t2.unit_config_id)
    join __all_unit t3
      on (t1. resource_pool_id = t3. resource_pool_id)
    left join __all_tenant t4
      on (t1.tenant_id = t4.tenant_id)
   order by t1. resource_pool_id, t2. unit_config_id, t3.unit_id;
  ```

* 查看整个zone的资源 看出来每个zone的空闲资源 资源综合是每个zone里面的cpu和内存总和 比如每个zone的cpu是1个 内存是1G 那么总和就是3cpu 3G

  ```sql
  select a.zone,
         concat(a.svr_ip, ':', a.svr_port) observer,
         cpu_total,
         cpu_assigned,
         (cpu_total - cpu_assigned) cpu_free,
         mem_total / 1024 / 1024 / 1024 mem_total_gb,
         mem_assigned / 1024 / 1024 / 1024 mem_assign_gb,
         (mem_total - mem_assigned) / 1024 / 1024 / 1024 mem_free_gb
    from __all_virtual_server_stat a
    join __all_server b
      on (a.svr_ip = b.svr_ip and a.svr_port = b.svr_port)
   order by a.zone, a.svr_ip;
   
   # 可以看出 可用的
   select t1.name resource_pool_name,
         t2. name unit_config_name,
         t2.max_cpu,
         t2.min_cpu,
         t2.max_memory / 1024 / 1024 / 1024 max_mem_gb,
         t2.min_memory / 1024 / 1024 / 1024 min_mem_gb,
         t3.unit_id,
         t3.zone,
         concat(t3.svr_ip, ':', t3. svr_port) observer,
         t4.tenant_id,
         t4.tenant_name
    from __all_resource_pool t1
    join __all_unit_config t2
      on (t1.unit_config_id = t2.unit_config_id)
    join __all_unit t3
      on (t1. resource_pool_id = t3. resource_pool_id)
    left join __all_tenant t4
      on (t1.tenant_id = t4.tenant_id)
   order by t1. resource_pool_id, t2. unit_config_id, t3.unit_id;
  ```

# 租户

租户就是实例 就是一个数据库 创建租户需要关联到某个资源池 租户之间是完全隔离的，sys看不到其他租户

* 创建租户 第一次创建租户的密码是空的

  ```sql
  # 创建租户 创建一个名字为tent_makj1的3副本租户 主副本为三个zone 资源池为pool_nakj1 优先zone1为主 如果zone1失败 则2和3
  create tenant tent_makj1 charset='utf8mb4',zone_list=('zone1','zone2','zone3'), primary_zone='zone1;zone2,zone3',resource_pool_list=('pool_nakj1');
  
  #主副本也可以写随机
  create tenant tent_makj1 charset='utf8mb4',zone_list=('zone1','zone2','zone3'), primary_zone='RANDOM',resource_pool_list=('pool_nakj1');
  
  #指定任何客户端 或者ip
  create tenant tent_makj1 charset='utf8mb4',zone_list=('zone1','zone2','zone3'), primary_zone='zone1,zone2,zone3',resource_pool_list=('pool_nakj1') set ob_tcp_invited_nodes='%';
  
  #企业版需要指定兼容mysql还是oracle 社区办只有mysql
  create tenant tent_makj1 charset='utf8mb4',resource_pool_list=('pool_nakj1'),primary_zone='RANDOM',comment '测试',set ob_tcp_invited_nodes='%',ob_compatibility='mysql';
  ```

* 删除租户 强制删除

  ```sql
  drop tenant tent_makj1 force;
  ```

* 查看租户

  ```sql
  use oceanbase;
  select * from gv$unit;
  
  #查看某个租户所有的资源池的资源单元个数
  select resource_pool_id,name,unit_count,unit_config_id,tenant_id from __all_resource_pool where tenant_id='1002';
  #查看租户会话
  show processlist
  # 终止租户会话
  KILL CONNECTION SESSION_ID
  ```

* 复制租户 

  仅ocp提供了复制租户的功能，可以根据指定租户复制一个新的租户 只会复制租户信息 数据不会被复制

* 修改租户

  ```
  #修改租户的主副本
  alter tenant tent_makj1 primary_zone='zone2';
  
  #修改租户的locality 增加副本数 全副本
  alter tenant tent_makj1 locality='F@zone1,F@zone2,F@zone3';
  
  #修改租户的locality 减少副本数 全副本
  alter tenant tent_makj1 locality='F@zone1,F@zone2';
  
  #修改资源池给租户
  alter tenant tent_makj1 resource_pool_list=('pool_nakj2')
  ```

* 租户访问权限 支持模糊匹配

  ```
  # 查看租户白名单
  show variables like 'ob_tcp_invited_nodes';
  # 设置白名单
  set global ob_tcp_invited_nodes='192.168.10.%';
  set global ob_tcp_invited_nodes='%'
  ```

* 登录租户 

  ```
  # 新创建的租户的密码是空的 可以不指定密码
  $ obclient -h192.168.10.160 -P2883 -uroot@tent_makj1#makjobce -c -A oceanbase
  
  #登录进去后修改密码
  MySQL [oceanbase]> alter user root identified by 'mkj123.';
  
  #修改密码后重登录
  $ obclient -h192.168.10.160 -P2883 -uroot@tent_makj1#makjobce -pmkj123. -c -A oceanbase
  ```

* 查看租户里的所有用户

  ```
  MySQL [oceanbase]> select user_id,user_name from __all_user;
  ```

* 创建一个普通用户 并且授权 

  ```
  MySQL [oceanbase]> set global ob_tcp_invited_nodes='%';
  MySQL [oceanbase]> create user makj identified by 'makj';
  MySQL [oceanbase]> grant all on *.* to makj;
  ```

* 普通用户登录

  ```
  # 先创建一个数据库
  MySQL [oceanbase]> create database makjdb;
  #普通用户登录
  obclient -h192.168.10.160 -P2883 -umakj@tent_makj1#makjobce -pmakj -c -A makjdb
  ```

* 至此 就可以用普通用户进行建表查询

# 集群组织架构

* 查看当前集群组织架构 可以修改idc 机房等信息

  ```
  select * from __all_zone
  ```

# 集群参数

集群支持参数定制 以适应不同的场景需求 参数查看和修改主要在集群内部的sys租户里

参数介绍(parameter)

* 部分参数的生效范围是集群内所有实例 可以通过命令修改哪些节点生效

* 少数参数在租户范围内生效

* 大部分参数变更是立即生效 有些可能要重启节点 可以通过命令查看是否重启

* 查看修改参数是否需要重启命令

  ```
  show parameters
  show parameters [like '%参数名%']
  ```

* 修改参数

  ```
  alter system set 参数名=参数值 例如（scope='xxx.xxx.xxx.xxx:2882'）
  ```

* 修改集群参数

  ```
  # 编辑集群参数
  $ obd cluster edit-config makjobce
  
  # 改完后reload
  $ obd cluster reload makjobce
  ```

* 还可以在启动集群的时候指定参数

  可以再命令后面通过-o指定 但是这个是单独修改的 不建议这么做

* 除了集群参数可以定制 租户也可以定制 通过变量(variable)设置 修改租户参数 

  * root登录到租户里修改 租户参数修改完成后只能在租户里看到
  * 也可以租户自己登录到租户里面进行修改

  变量值 描述可以通过命令查看

  ```
  show global variables
  show global |[session] variables [like '%参数名%']
  show global |[session] variables where variable_name in ('参数',...)
  ```

* 修改变量

  ```
  set global |[session] 变量名=变量值 (字符串需要用单引号)
  global 全局生效，对当前会话不生效 对新建会话生效
  session 或者没有限定 对当前会话生效，断开重连后失效
  ```

* 参数文件

  查看参数除了登录到sys租户实例 还可以通过查看文件的方法 ob集群的参数文件名字固定是:observer.config.bin 在etc/目录下

# 内存管理

ob支持多租户架构的准内存分布式数据库 对大容量内存的管理和使用有很高的要求 会占据物理服务器的大部分内存进行统一管理

通过参数设定observer的内存占用上线 最大内存占用系统的80%

memeory_limit_percentage和memory_limit

通过两种方式设置内存上限

1. 按照物理机总内存的百分比计算内存上线 由memeory_limit_percentage设置
2. 直接设置observer内存上线 由memory_limit参数设置 memory_limit=0的时候 memeory_limit_percentage决定observer内存大小 否则由memory_limit决定
3. 以128G内存为例 
   1. 如果memory_limit=0 由memeory_limit_percentage决定observer的内存大小 即128*0.6=76.8
   2. 如果memory_limit=85G 因为observer内存上线就是85 memeory_limit_percentage失效

* 系统内部内存

  * 每个observer都包含多个租户(sys租户和其他租户)数据，但是observer的内存并不是全部分配给租户的
  * observer有些内存不属于任何租户，属于所有租户的共享资源 称为“系统内部内存”
  * 系统内部内存上线由system_memory设定
  * 租户可用的总内存=observer内存上线-系统内部内存

* 每个租户内部内存总体分为两部分

  * MemStore：不可动态伸缩的内存用来保存DML产生的增量数据 空间不可被占用。MemStore大小由memeory_limit_percentage决定 标识租户的MemStore部分占租户总内存的百分比(默认为50% 占用租户内存的50%) 当MemStore内存使用超过freezze_trigger_percentage定义的百分比的时候(70%) 触发冻结及后续的转储/合并等行为
  * KVCache:可动态伸缩的内存 空间会被其他众多内存模块复用

* 修改memory_limit和memeory_limit_percentage

  ```
  alter system set memeory_limit_percentage=90
  alter system set memory_limit=8G
  ```

  一般来说要维持mem_usage在90%一下，如果超过90% 有可能就会触发租户内写入限速，降低写入速度

* 查看租户的内存使用情况

  ```
  select * from gv$memstore where tenant_id=1002
  ```

# 表分区

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

    RANGE分区根在分区表定义时为每个分区建立的分区键值范围，将数据映射到相应的分区中。
    它是常见的分区类型，经常跟日期类型一起使用。
    比如说，可以将业务日志表按日/周/月分区。

    RANGE分区要求表拆分键表达式的结果必须为整型，如果要按时间类型列做RANGE分区，则必须使用timestamp类型，并且使用函数UNIX.TIMESTAMP将时间类型转换为数值。

    RANGE分区简单的语法格式如下：

    ```
    CREATE TABLE table.name (
    column_namel column_type
    3 column_nameN column_type]
    )PARTITION BY RANGE C expr(column_namel))
    (
    	PARTITION p0 VALUES LESS THAN ( expr )
    	[,PARTITION pN VALUES LESS THAN (expr )]
    	PARTITION pX VALUES LESS THAN (maxvalue)]
    );
    ```

    示例：

    ```
    # 创建按照时间分区表
    CREATE TABLE makj_range(id INT, username VARCHAR(20),createdate
    TIMESTAMP, PRIMARY KEY (createdate))
    PARTITION BY RANGE(UNIX_TIMESTAMP(createdate))
    (
    PARTITION p0 VALUES LESS THAN (UNIX_TIMESTAMP('2020-01-01 00:00:00')), #小于2020-01-01放到第一个分区 < 2020-01-01
    PARTITION p1 VALUES LESS THAN (UNIX_TIMESTAMP('2021-01-01 00:00:00')), #小于2021-01-01放到第一个分区 2020-01-01 < x < 2021-01-01
    PARTITION p2 VALUES LESS THAN (UNIX_TIMESTAMP('2022-01-01 00:00:00')) #小于2022-01-01放到第一个分区  > 2022-01-01
    );
    
    insert into makj_range values('1','2019','2019-12-01 01:03:10');
    insert into makj_range values('2','2020','2020-11-21 05:02:20');
    insert into makj_range values('3','2021','2021-09-20 09:09:50');
    
    select * from makj_range;
    # 查看指定分区
    select * from makj_range partition (p0);
    select * from makj_range partition (p1);
    
    Range Columns 分区与Range 分区的作用基本类似，不同之处在于：
    Range Columns 分区的分区键的结果不要求是整型，可以是任意类型。
    Range Columns 分区的分区键不能使用表达式。
    Range Columns 分区的分区键可以写多个列（即列向量）
    ```

    

  * Range Columns 分区

    Range Columns 分区与Range 分区的作用基本类似，不同之处在于：
    Range Columns 分区的分区键的结果不要求是整型，可以是任意类型。
    Range Columns 分区的分区键不能使用表达式。
    Range Columns 分区的分区键可以写多个列（即列向量）

  * llist列表分区

    按照字段的值范围进行分区

    List 分区可以显式的控制记录行如何映射到分区，具体方法是为每个分区的分区键指定一组离散值列表。
    List 分区的优点是可以方便的对无序或无关的数据集进行分区。

    示例

    ```
    create table makj_list
    (
    ID int not null,
    Name varchar(20) not null,
    Age int not null
    )
    partition by list (Age)
    (
    partition par_01 values in (20), #20岁的一个分区
    partition par_02 values in (30,40)  # 30和40 的一个
    );
    insert into makj_list values('1','fgedu2019','20');
    insert into makj_list values('2','fgedu2020','30');
    insert into makj_list values('3','fgedu2021','40');
    commit;
    select * from makj_list;
    select * from makj_list partition (par_02) ;
    ```

  * List Columns 分区

    List Columns 分区与List 分区的作用基本相同，不同之处在于：
    List Columns 分区的分区键不要求是整型，可以是任意类型。
    List Columns 分区的分区键可以是多列（即列向量）。

  * Hash分区

    HASH 分区适合于对不能用RANGE 分区、LIST 分区方法的场景，
    它的实现方法简单，通过对分区键上的HASH 函数值来散列记录到不同分区中。
    如果数据符合下列特点，使用HASH 分区是个很好的选择：

    1. 不能指定数据的分区键的列表特征。
    2. 不同范围内的数据大小相差非常大，并且很难手动调整均衡。
    3. 使用RANGE 分区后数据聚集严重。
    4. 并行DML、分区剪枝和分区连接等性能非常重要。
    5. HASH 分区不能做新增或删除分区操作。

    示例：按照id进行hash分区 分到5个区域

    ```
    CREATE TABLE makj_hash(id INT, username VARCHAR(20), Age int,PRIMARY KEY (id))
    PARTITION by hash(id) partitions 5;
    insert into makj_hash values('1','fgedu2019','10');
    insert into makj_hash values('2','fgedu2020','20');
    insert into makj_hash values('3','fgedu2021','30');
    insert into makj_hash values('4','fgedu2021','40');
    insert into makj_hash values('5','fgedu2021','50');
    insert into makj_hash values('6','fgedu2021','60');
    commit;
    select * from makj_hash;
    select * from makj_hash partition (p2) ;
    ```

  * Key分区

    Key 分区与Hash 分区类似，也是通过对分区个数取模的方式来确定数据属于哪个分区，不同的是系统会对Key 分区键做一个内部默认的Hash 函数后再取模。
    Key 分区有如下特点：

    * Key 分区的分区键不要求为整型，可以为任意类型
    * Key 分区的分区键不能使用表达式
    * Key 分区的分区键支持向量
    * Key 分区的分区键中不指定任何列时，表示Key 分区的分区键是主键。

    分区表的查询性能跟SQL 中条件有关。
    当SQL 中带上拆分键时，OceanBase 会根据条件做分区剪枝，只用搜索特定的分区即可；如果没有拆分键，则要扫描所有分区。
    分区表也可以通过创建索引来提升性能，跟分区表一样，分区表的索引也可以分区或者不分区。
    如果分区表的索引不分区，就是一个全局索引（ GLOBAL ），是一个独立的分区，索引数据覆盖整个分区表。
    如果分区表的索引分区了，可以分区表保持一致的分区策略，则每个索引分区的索引数据覆盖相应的分区表的分区，这个索引又叫本地索引（ LOCAL ）。
    注意：通常创建索引时默认都是全局索引，本地索引需要在后面增加关键字LOCAL 。
    建议:
    尽可能的使用本地索引，只有在有必要的时候才使用全局索引。
    其原因是全局索引会降低DML 的性能，DML 可能会因此产生分布式事务。
    创建分区表的本地索引和全局索引:

    ```
    # 创建本地索引
    CREATE INDEX idx_makj_hash_range_date ON makj_hash_range(createdate) LOCAL;
    
    # 创建全局索引
    CREATE INDEX makj_range_hash_id_date ON makj_range_hash(id,logindate) GLOBAL;
    
    #查看索引
    show indexes from makj_range_hash;
    ```

    

  * 组合分区

    ```
    #组合分区 先分范围 在分hash
    CREATE TABLE makj_hash_range (
    id int
    , createdate TIMESTAMP NOT NULL
    , name varchar(30)
    , primary key (id,createdate)
    )
    PARTITION BY hash(id)
    SUBPARTITION BY RANGE(UNIX_TIMESTAMP(createdate))
    SUBPARTITION template
    (
    	SUBPARTITION p202101 VALUES LESS THAN(UNIX_TIMESTAMP('2021/02/01'))
    	, SUBPARTITION p202102 VALUES LESS THAN(UNIX_TIMESTAMP('2021/03/01'))
    	, SUBPARTITION p202103 VALUES LESS THAN(UNIX_TIMESTAMP('2021/04/01'))
    	, SUBPARTITION p202104 VALUES LESS THAN(UNIX_TIMESTAMP('2021/05/01'))
    	, SUBPARTITION p202105 VALUES LESS THAN(UNIX_TIMESTAMP('2021/06/01'))
    	, SUBPARTITION p202106 VALUES LESS THAN(UNIX_TIMESTAMP('2021/07/01'))
    	, SUBPARTITION p202107 VALUES LESS THAN(UNIX_TIMESTAMP('2021/08/01'))
    	, SUBPARTITION p202108 VALUES LESS THAN(UNIX_TIMESTAMP('2021/09/01'))
    	, SUBPARTITION p202109 VALUES LESS THAN(UNIX_TIMESTAMP('2021/10/01'))
    	, SUBPARTITION p202110 VALUES LESS THAN(UNIX_TIMESTAMP('2021/11/01'))
    	, SUBPARTITION p202111 VALUES LESS THAN(UNIX_TIMESTAMP('2021/12/01'))
    	, SUBPARTITION p202112 VALUES LESS THAN(UNIX_TIMESTAMP('2022/01/01'))
    )
    partitions 5;
    
    #组合分区 先分hash 在分范围
    CREATE TABLE makj_range_hash (
    id int NOT NULL
    , address varchar(50)
    , logindate TIMESTAMP NOT NULL
    , PRIMARY key(id, logindate)
    )
    PARTITION BY RANGE(UNIX_TIMESTAMP(logindate))
    SUBPARTITION BY HASH(id)
    SUBPARTITIONS 5
    (
    PARTITION p202101 VALUES LESS THAN(UNIX_TIMESTAMP('2021/02/01'))
    , PARTITION p202102 VALUES LESS THAN(UNIX_TIMESTAMP('2021/03/01'))
    , PARTITION p202103 VALUES LESS THAN(UNIX_TIMESTAMP('2021/04/01'))
    , PARTITION p202104 VALUES LESS THAN(UNIX_TIMESTAMP('2021/05/01'))
    , PARTITION p202105 VALUES LESS THAN(UNIX_TIMESTAMP('2021/06/01'))
    , PARTITION p202106 VALUES LESS THAN(UNIX_TIMESTAMP('2021/07/01'))
    , PARTITION p202107 VALUES LESS THAN(UNIX_TIMESTAMP('2021/08/01'))
    , PARTITION p202108 VALUES LESS THAN(UNIX_TIMESTAMP('2021/09/01'))
    , PARTITION p202109 VALUES LESS THAN(UNIX_TIMESTAMP('2021/10/01'))
    , PARTITION p202110 VALUES LESS THAN(UNIX_TIMESTAMP('2021/11/01'))
    , PARTITION p202111 VALUES LESS THAN(UNIX_TIMESTAMP('2021/12/01'))
    , PARTITION p202112 VALUES LESS THAN(UNIX_TIMESTAMP('2022/01/01'))
    );
    
    ```

# 表分组

表分组（TABLE GROUP）是OceanBase 作为分布式数据库的一个特色功能。
表分组是表的属性，会影响多个表的分区在OceanBase 机器上的分布特征。
不同表的分区有可能分布在不同的节点上，当两个表做表连接查询时，OceanBase 会跨节点请求数据，执行时间就跟节点间请求延时有关。
在SQL 调优时，OceanBase 建议对业务上关系密切的表，设置相同的表分组。
OceanBase 对于同一个表分组中的表的同号分区会管理为一个分区组。
同一个分区组中的分区，OceanBase 会尽可能的分配到同一个节点内部，这样就可以规避跨节点的请求。

创建表分组
创建表分组时，首先要规划好表分组的用途。
如果是用于普通表的属性，表分组就不用分区；
如果是用于分区表的属性，表分组就要指定分区策略，并且要跟分区表的
分区策略保持一致。

* 创建表分组

  ```
  create tablegroup makj_group;
  show create tablegroup makj_group;
  ```

* 查看表分组

  ```
  show tablegroups;
  ```

* 将表加入表分组

  ```
  CREATE TABLE makj_t1(id INT, username VARCHAR(20), Age int,PRIMARY KEY (id)) ;
  CREATE TABLE makj_t2(id INT, username VARCHAR(20), Age int,PRIMARY KEY (id));
  CREATE TABLE makj_t3(id INT, username VARCHAR(20), Age int,PRIMARY KEY (id));
  
  向表分组里加入表
  alter tablegroup makj_group add makj_t1, makj_t2;
  
  也可以直接修改表的表分组
  alter table makj_t3 tablegroup='makj_group';
  
  从表分组里删除表
  alter table makj_t1 tablegroup='';
  alter table makj_t2 tablegroup='';
  alter table makj_t3 tablegroup='';
  ```

* 删除表分组

  ```
  drop tablegroup makj_group;
  
  drop tablegroup makj_group;
  # 普通的表分组是不能加入分区表的，会报错：
  ERROR 4179 (HY000): table and tablegroup use different partition options not allowed
  
  ```

普通的表分组是不能加入分区表的，会报错：

* 创建分区表表分组

  ```
  # 创建两张表
  CREATE TABLE makj_gt1(id INT, username VARCHAR(20), Age int,PRIMARY KEY (id)) PARTITION by hash(id) partitions 5;
  CREATE TABLE makj_gt2(id INT, username VARCHAR(20), Age int,PRIMARY KEY (id)) PARTITION by hash(id) partitions 5;
  
  # 创建分区表表分组 并且将表加入其中
  create tablegroup makj_pgroup partition by hash partitions 5;
  alter tablegroup makj_pgroup add makj_gt1, makj_gt2;
  ```

# 复制表

复制表指的是一种特殊的表。
普通的表在生产环境，默认有三副本，其中一个主副本和两个备副本。
备副本通过同步主副本的事务日志clog 保持同步，同步协议是Paxos协议，主副本的事务日志只有在多数成员里确认落盘后，事务修改才会生效。
通常默认情况下，读写都是在主副本上，备副本是不提供读写服务。
应用如果开启会话或语句级别的弱一致性读后，备副本可能会提供只读服务,风险就是备副本的读会有些许延迟。
普通表可以变为复制表，然后主副本和所有备副本之间使用全同步协议，主副本的事务日志只有在所有副本成员里确认落盘后，事务修改才会生效。
所以主副本跟所有备副本的数据理论上都是强一致的。
复制表最有用的场景是业务数据库做了水平拆分后，有部分业务表不适合拆分。
前者的数据主副本有可能在所有机器上，后者的主副本只会在某台机器上。
OceanBase 里一个事务的SQL 都会跟随到事务开始时那条SQL 的路由，如果某个SQL 被路由到的节点不是该SQL 访问的分区的主副本节点，这个SQL 就是个远程SQL 。
如果这个分区所在的表是复制表，则这条SQL 就会在本机执行，从而提升性能。
复制表使用的前提是表的修改频率不能太高，每个事务的平均延时会比普通的表的事务延时要大。
可以在创建表的时候就指定复制表属性DUPLICATE_SCOPE 。
这个属性有下面几个值：

* NONE : 这个是默认值，表示是普通的表。
* CLUSTER ：表的备副本分布在租户资源池所在的所有机器上。

创建复制表

```
create table makj_d1(id bigint not null auto_increment , name varchar(50),createdate timestamp not null default current_timestamp) duplicate_scope='cluster';

# 也可以在表创建好后修改这个属性。
alter table makj_d1 duplicate_scope = 'CLUSTER';
alter table makj_d1 duplicate_scope = 'NONE';
```

# 备份

日志归档是指日志数据的自动归档功能，observer 会定期将日志数据归档到指定的备份路径。
这个动作是全自动的，只需要用户发起一次alter system archivelog，日志备份就会在后台持续进行，不需要外部定期触发。
数据备份指的是备份基线数据的功能，该功能分为全量备份和增量备份两种
注意事项：

* 开始备份前一定要执行一次合并major_freeze，不然会由于版本1 没有
  冻结时间而导致备份失败。

* 合并完成后，才能进行备份，否则会报错。

* 恢复的最小粒度为租户级别，执行恢复命令恢复租户的时候是不会自动
  创建资源的，要先创建要恢复的租户的资源池resource pool

* 备份前建议将资源池的数据备份一份出来 通过sql语句查询备份

  ```
  select t1.name resource_pool_name,
  t2. name unit_config_name,
  t2.max_cpu,
  t2.min_cpu,
  t2.max_memory / 1024 / 1024 / 1024 max_mem_gb,
  t2.min_memory / 1024 / 1024 / 1024 min_mem_gb,
  t3.unit_id,
  t3.zone,
  concat(t3.svr_ip, ':', t3. svr_port) observer,
  t4.tenant_id,
  t4.tenant_name
  from __all_resource_pool t1
  join __all_unit_config t2
  on (t1.unit_config_id = t2.unit_config_id)
  join __all_unit t3
  on (t1. resource_pool_id = t3. resource_pool_id)
  left join __all_tenant t4
  on (t1.tenant_id = t4.tenant_id)
  order by t1. resource_pool_id, t2. unit_config_id, t3.unit_id;
  ```

  

1. 创建nfs

   1. 服务端 在192.168.10.167创建

      ```
      # mkdir /data/backup
      # chmod -R 777 /data/backup
      # echo "/data/backup *(rw,sync,all_squash,anonuid=500,anongid=500)" >> /etc/exports
      # 重启服务：
      # systemctl enable nfs
      # systemctl restart nfs
      # exportfs -r
      # showmount -e 192.168.10.167
      ```

   2. 客户端 所有客户端执行

      ```
      # yum install rpcbind nfs-utils -y
      # mkdir /data/backup
      # echo "ocp:/data/backup /data/backup nfs rw,timeo=7,hard,intr,bg,suid,lock 0 0" >> /etc/fstab
      # mount -a
      # df -h
      ```

   3. 备份 登录到任意一台数据库中 必须用sys租户的root用户执行备份 可以备份到nfs或者oss

      ```
      # 设置nfs或者oss
      nfs: obclient> alter system set backup_dest='file:///data/backup';
      oss: obclient> alter system set backup_dest='oss://xxxxxxxxxxx';
      
      #设置备份目录 需要每台机器都有
      alter system set backup_dest='file:///data/backup';
      
      #查看备份目标地 每台都有
      show parameters like 'backup_dest';
      ```

   4. 启动数据库的归档贵能

      ```
      alter system archivelog;
      
      #查看规定那记录 需要等到状态为DOING的时候 才能开始备份
      SELECT * FROM CDB_OB_BACKUP_ARCHIVELOG_SUMMARY;
      ```

   5. 执行全量或者增量备份，第一次备份前 一定要执行一次合并，执行全量备份前 对集群进行一次合并，等合并完成后 才能执行下面动作 否则出现ERROR 9040 (HY000): backup can not start

      ```
      alter system major freeze;
      
      # 查看合并进度 如果info信息显示IDLE 代表合并完成
      SELECT * FROM __all_zone WHERE name='merge_status';
      ```

   6. 可以设置备份密码

      ```
      set encryption on identified by 'rootroot' only;
      ```

   7. 开始执行全量备份

      ```
      alter system backup database;
      ```

   8. 查看备份进度 必须cdb_ob_backup_progress没有数据 算备份完成 

      ```
      select * from cdb_ob_backup_progress;
      
      #查看历史详细信息 其中BACK_TYPE字段 I代表增量 D代表全量
      select * from cdb_ob_backup_set_details;
      ```

   9. 等到上面备份是否完成  在向表里插入数据 进行增量备份

      ```
      alter system backup incremental database;
      ```

   10. 然后再次查看是否备份完成 必须cdb_ob_backup_progress没有数据 算备份完成

       ```
       select * from cdb_ob_backup_progress;
       #查看历史详细信息
       select * from cdb_ob_backup_set_details;
       ```

   11. 如果备份过程中不行备份了，可以停止

       ```
       # 停止数据备份任务
       ALTER SYSTEM CANCEL BACKUP;
       
       #停止日志备份任务
       ALTER SYSTEM NOARCHIVELOG;
       ```

   12. 查看根据现有的Recovery Window 计算出的过期备份：

       ```
       SELECT * FROM oceanbase.CDB_OB_BACKUP_SET_EXPIRED;
       ```

   13. 获取备份目的地

       ```
       SHOW PARAMETERS LIKE '%backup_dest%';
       ```

   14. 获取备份的恢复窗口

       ```
       SHOW PARAMETERS LIKE '%recovery_window%';
       ```

   15. 查看与备份相关的RootService 日志事件

       ```
       SELECT *
       FROM oceanbase.__all_rootservice_event_history
       WHERE module LIKE '%backup'
       OR module LIKE '%archive%'
       ORDER BY gmt_create DESC LIMIT 30;
       ```

# 恢复

恢复前一定要停止日志归档

执行恢复命令的恢复租户是不会自动创建资源的，所以必须先创建租户的资源池resource pool

创建恢复目标租户需要用到unit resource pool

1. 恢复前一定要停止日志归档

   ```
   alter system noarchivelog;
   ```

2. 创建恢复的资源池和资源单元

   ```
   CREATE resource unit unit_makj4 max_cpu=2, min_cpu=2,max_memory='1G', min_memory='1G', max_iops=10000,min_iops=1000, max_session_num=1000000, max_disk_size='10G';
   
   create resource pool pool_makj4 unit='unit_makj4' , unit_num=1,zone_list=('zone1' ,'zone2' ,'zone3') ;
   ```

3. 如果之前设置了加密密码 需要设置解密

   ```
   SET @kms_encrypt_info = '<加密string>';
   
   SET DECRYPTION IDENTIFIED BY 'password1','password2';
   ```

4. 设置恢复并行度 如果restore_concurrency为0  该大些

   ```
   show parameters like 'restore_concurrency';
   
   alter system set restore_concurrency = 20;
   ```

5. 默认恢复等待时间_restore_idle_time 为1 分钟，整个恢复期间会有3 次等待，即3 分钟的等待时间。
   对于测试恢复性能的场景，为了减少恢复的空闲时间，将等待时间调整为10s。

   ```
   ALTER SYSTEM SET _restore_idle_time = '10s';
   ```

6. 开始恢复

   Restore tenant 命令内部的流程如下：
   1. 创建恢复用的租户
   2. 恢复租户的系统表数据
   3. 恢复租户的系统表日志
   4. 调整恢复租户的元信息
   5. 恢复租户的用户表数据
   6. 恢复租户的用户表日志

   语法

   ```
   ALTER SYSTEM RESTORE <dest_tenant_name> FROM <source_tenan_tname> at 'uri' UNTIL 'timestamp' WITH 'restore_option';
   
   # dest_tenant_name:恢复到哪个租户
   # source_tenan_tname：从哪个租户恢复
   # uri：在什么目录
   # timestamp：采用时间戳的方式
   # restore_option恢复到哪个资源池
   ```

   恢复：

   ```
   # 查看归档时间 时间是 > CDB_OB_BACKUP_SET_DETAILS的start_time的最大时间 <= CDB_OB_BACKUP_ARCHIVELOG_SUMMARY的max_next_time的最大时间
   
   SELECT * FROM oceanbase.CDB_OB_BACKUP_SET_DETAILS;
   SELECT * FROM oceanbase.CDB_OB_BACKUP_ARCHIVELOG_SUMMARY;
   
   #执行恢复命令
   alter system restore tent_makj2 from tent_makj1 at 'file:///data/backup' until '2022-04-13 22:05:42.884200'
   with 'backup_cluster_name=makjobce&backup_cluster_id=2&pool_list=pool_makj4';
   ```

7. 查看恢复进度

   其中，is_restore 的取值含义如下：
   0：表示正常副本
   1：表示逻辑恢复的副本
   2：表示物理恢复需要恢复基线的副本
   3：表示物理恢复需要恢复转储的副本
   4：物理恢复需要恢复clog 的副本
   5：物理恢复需要转储的副本
   6：物理恢复等待所有副本转储完成的副本
   7：物理恢复设置member list 的副本
   role 的取值含义如下：
   1：表示Leader
   2：表示Follower
   3：表示恢复中的Leader

   ```
   SELECT svr_ip, role, is_restore, COUNT(*)
   FROM __all_root_table AS a,
   (SELECT value FROM __all_restore_info WHERE name =
   'tenant_id') AS b
   WHERE a.tenant_id = b.value
   GROUP BY role, is_restore, svr_ip
   ORDER BY svr_ip, is_restore;
   ```

8. 查看整个恢复信息 如果没记录了 代表恢复完成了

   ```
   SELECT * FROM __all_restore_info;
   ```

9. 恢复完成后检查(没有数据了代表正常)

   ```
   select svr_ip, role, is_restore, count(*)
   from __all_virtual_meta_table as a,
   (select value from __all_restore_info where name =
   'tenant_id') as b
   where a.tenant_id = b.value
   group by role, is_restore, svr_ip
   order by svr_ip, is_restore;
   
   # 或者这个 查看恢复信息
   SELECT svr_ip, is_restore, COUNT(*)
   FROM __all_virtual_partition_store_info
   WHERE tenant_id > 1002
   group by svr_ip, is_restore
   order by svr_ip, is_restore;
   ```

10. 查看恢复结果信息

    ```
    SELECT * FROM __all_restore_info;
    SELECT * FROM __all_restore_history;
    ```

11. 验证恢复结果 登录到新的租户里去看

    ```
    $ obclient -h192.168.10.160 -P2883 -uroot@tent_makj2#makjobce  -p
    
    MySQL [(none)]> show databases;
    MySQL [mysql]> use makjdb;
    MySQL [makjdb]> select * from makj_t4;
    ```

# 逻辑备份与恢复

导出的时候先要导出结构 然后数据 导入的时候先导入结构 然后数据

使用PBDUMPER/OBLOADER工具导出导入

1. 准备java环境 配置jdk

   ```
   export JAVA_HOME=/data/soft/jdk1.8.0_311
   export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
   export PATH=$PATH:$JAVA_HOME/bin
   ```

   

2. 装备工具

   ```
    # unzip ob-loader-dumper-3.0.0-RELEASE-ce.zip 
    # ob-loader-dumper-3.0.0-RELEASE-ce/bin
   ```

3. 导出ddl

   ```
   # 格式
   bin/obdumper -h <主机IP> -P <主机端口> -u <用户> -p <密码> --sys-user <sys 租户下的root 用户或proxyro 用户> --sys-password <sys 租户下的账号密码> -c <集群> -t <租户> -D <Schema 库名> [--ddl] [--csv|--sql] [--all|--table '表名'] -f<数据目录>
   
   # 创建导出目录
   # mkdir  /data/obdumper
   # ./bin/obdumper -h 192.168.10.161 -P 2883 -u root -p mkj123. --sys-user=root --sys-password=mkj123. -c makjobce -t tent_makj1 -D makjdb --ddl --all -f /data/obdumper
   ```

4. 导出数据

   ```
   # mkdir  /data/obdumper2
   # ./bin/obdumper -h 192.168.10.161 -P 2883 -u root -p mkj123. --sys-user=root --sys-password=mkj123. -c makjobce -t tent_makj1 -D makjdb --csv --all -f /data/obdumper2
   ```

5. 恢复数据 先导入ddl

   ```
   # 恢复之前 先把数据库干掉
   drop database makjdb;
   create database makjdb;
   
   # 导入命令
   # bin/obloader -h <主机IP> -P <端口> -u <用户> -p <密码> --sys-user <sys 租户下的root 用户或proxyro 用户> --sys-password <sys 租户下的账号密码> -c <集群> -t <租户> -D <Schema 库名> [--ddl] [--csv|--sql] [--all|--table '表名'] -f<数据目录>
   
   # 执行导入
   # ./bin/obloader -h 192.168.10.161 -P 2883 -u root -p mkj123. --sys-user=root --sys-password=mkj123. -c makjobce -t tent_makj1 -D makjdb --ddl --all -f /data/obdumper
   ```

6. 导入数据

   ```
   ./bin/obloader -h 192.168.10.161 -P 2883 -u root -p mkj123. --sys-user=root --sys-password=mkj123. -c makjobce -t tent_makj1 -D makjdb --csv --all -f /data/obdumper2
   ```

7. 问题总结

   导出表结构的时候，如果表很多，不能开启太多并发， --threads 控制在4 以内
   太高的并发没有意义，会加重SYS 租户访问内部视图的负担，，进而出现导出超时报错。导出表数据的时候。
   如果表很多，可以开启并发。
   --threads
   默认会根据CPU 动态调整，也可以手动指定，以控制导出对主机性能的影响。
   导出或导入大量的数据时，可能瓶颈处在OBLOADER 和OBDUMPER 自身，可以编辑这两个文件，调大里面的JAVA 参数Xms 和Xmx 后面的值，分别表示JAVA 初始内存和最大可用内存。
   vim bin/obdumper 或vim bin/obloader
   JAVA_OPTS="$JAVA_OPTS -server -Xms4G -Xmx4G
   -XX:MetaspaceSize=512M -XX:MaxMetaspaceSize=512M -Xss352K"
   导入性能问题
   导入表结构时，指定--threads 并发数不要超过2

8. 从mysql迁移到oceanbase说明

   mysqldump
   dbcat
   Canal
   DATAX
   PDI/KETTLE
   CDC 全称是Change Data Capture，即变更数据捕获。

# 集群扩容

1. 准备

   给予obd进行扩容(官方推荐)

   1-1-1扩容只1-1-1-1-1-1 3副本扩容至5副本

   扩容节点为

   obs4和obs5

* 扩容过程：

  需要创建一个新的集群，然后把旧的集群迁移到新的集群 新的集群只包含obs4和obs5 然后将新的集群整合到之前旧的集群

1. 查看当前集群信息

   ```
   # su - admin
   Last login: Wed Apr 13 22:31:26 CST 2022 on pts/0
   $ obd cluster display makjobce
   
   # 检查版本
   $ obd --version
   ```

2. 准备新的集群，准备完成后不要启动

3. 准备配置文件

   ```
   $ vim makjobce2-5zones.yaml 
   # Only need to configure when remote login is required
   user:
      username: admin  #部署的用户名 
      password: admin  #密码
      key_file: /home/admin/.ssh/id_rsa.pub
      port: your ssh port, default 22  #端口
   #   timeout: ssh connection timeout (second), default 30
   oceanbase-ce:  #配置集群区域名字
     servers:  #服务器参数
       - name: obs4  #节点名字 共三台
         # Please don use hostname, only IP can be supported
         ip: 192.168.10.163
       - name: obs5
         ip: 192.168.10.164
     global:  # 全局配置
       # Please set devname as the network adaptor name whose ip is  in the setting of severs.
       # if set severs as "127.0.0.1", please set devname as "lo"
       # if current ip is 192.168.1.10, and the ip's network adaptor's name is "eth0", please use "eth0"
       devname: ens192  #指定网卡
       cluster_id: 2  #集群id 可以自定义 整数 默认2
       # please set memory limit to a suitable value which is matching resource.
       memory_limit: 8G # The maximum running memory for an observer  #observer 内存限制，最低8G 这里给8G 默认系统的80%
       system_memory: 3G # The reserved system memory. system_memory is reserved for general tenants. The default value is 30G. #observer 系统内存 算在8G里面的 这部分是所有租户的共享内存
       stack_size: 512K  #堆栈大小 
       cpu_count: 16   #cpu数量 这里给默认 也可以根据真实环境改变 最多16个
       cache_wash_threshold: 1G  #缓存清洗的值 默认就好
       __min_full_resource_pool_memory: 268435456  #最小资源池大小
       workers_per_cpu_quota: 10 #每个cpu工作的线程数
       schema_history_expire_time: 1d #scheam清洗的时间 1天
       # The value of net_thread_count had better be same as cpu core number.
       net_thread_count: 4 #网络线程总数
       major_freeze_duty_time: Disable
       minor_freeze_times: 10
       enable_separate_sys_clog: 0 # 是否启用clog阻塞
       enable_merge_by_turn: FALSE # 是否启用按照顺序合并
       #datafile_disk_percentage: 20 # The percentage of the data_dir space to the total disk space. This value takes effect only when datafile_size is 0. The default value is 90.
       datafile_size: 50G  # 数据初始文件大小 最小就是50G
       syslog_level: WARN # System log level. The default value is INFO. # 系统日志级别
       enable_syslog_wf: false # Print system logs whose levels are higher than WARNING to a separate log file. The default value is true. # 是否启用日志打印
       enable_syslog_recycle: true # Enable auto system log recycling or not. The default value is false. # 是否启用日志回收
       max_syslog_file_count: 10 # The maximum number of reserved log files before enabling auto recycling. The default value is 0.# 日志回收最大文件数量
       # observer cluster name, consistent with obproxy cluster_name
       appname: makjobce2   # 集群名字需要和集群标识不一样
       root_password: mkj123. # root user password, can be empty  # 集群管理员密码 就是root@sys的账户密码
       proxyro_password: mkj123. # proxyro user pasword, consistent with obproxy observer_sys_password, can be empty
     obs4:
       mysql_port: 2881 # External port for OceanBase Database. The default value is 2881. # mysql端口
       rpc_port: 2882 # Internal port for OceanBase Database. The default value is 2882. # 集群rpc的端口
       #  The working directory for OceanBase Database. OceanBase Database is started under this directory. This is a required field.# obproxy连接集群使用的密码
       home_path: /home/admin/oceanbase-ce #安装目录
       # The directory for data storage. The default value is $home_path/store.
       data_dir: /data  # 数据目录
       # The directory for clog, ilog, and slog. The default value is the same as the data_dir value.
       redo_dir: /redo # redo目录
       zone: zone4 # 属于第几个zone 
     obs5:
       mysql_port: 2881 # External port for OceanBase Database. The default value is 2881.
       rpc_port: 2882 # Internal port for OceanBase Database. The default value is 2882.
       #  The working directory for OceanBase Database. OceanBase Database is started under this directory. This is a required field.
       home_path: /home/admin/oceanbase-ce
       # The directory for data storage. The default value is $home_path/store.
       data_dir: /data
       # The directory for clog, ilog, and slog. The default value is the same as the data_dir value.
       redo_dir: /redo
       zone: zone5
   ```

4. 部署新集群

   ```
   $ obd cluster deploy makjobce2 -c makjobce2-5zones.yaml
   
   #部署完成查看集群信息
   $ obd cluster list
   ```

5. 合并配置 将新集群的配置添加到老集群的配置文件里

   ```
   $ cd /home/admin/.obd/cluster/makjobce
   $ vim config.yaml 
   # Only need to configure when remote login is required
   user:
      username: admin  #部署的用户名 
      password: admin  #密码
      key_file: /home/admin/.ssh/id_rsa.pub
      port: your ssh port, default 22  #端口
   #   timeout: ssh connection timeout (second), default 30
   oceanbase-ce:  #配置集群区域名字
     servers:  #服务器参数
       - name: obs1  #节点名字 共三台
         # Please don use hostname, only IP can be supported
         ip: 192.168.10.160
       - name: obs2
         ip: 192.168.10.161
       - name: obs3
         ip: 192.168.10.162
       - name: obs4 
         ip: 192.168.10.163
       - name: obs5
         ip: 192.168.10.164
     global:  # 全局配置
       # Please set devname as the network adaptor name whose ip is  in the setting of severs.
       # if set severs as "127.0.0.1", please set devname as "lo"
       # if current ip is 192.168.1.10, and the ip's network adaptor's name is "eth0", please use "eth0"
       devname: ens192  #指定网卡
       cluster_id: 2  #集群id 可以自定义 整数 默认2
       # please set memory limit to a suitable value which is matching resource.
       memory_limit: 8G # The maximum running memory for an observer  #observer 内存限制，最低8G 这里给8G 默认系统的80%
       system_memory: 3G # The reserved system memory. system_memory is reserved for general tenants. The default value is 30G. #observer 系统内存 算在8G里面的 这部分是所有租户的共享内存
       stack_size: 512K  #堆栈大小 
       cpu_count: 16   #cpu数量 这里给默认 也可以根据真实环境改变 最多16个
       cache_wash_threshold: 1G  #缓存清洗的值 默认就好
       __min_full_resource_pool_memory: 268435456  #最小资源池大小
       workers_per_cpu_quota: 10 #每个cpu工作的线程数
       schema_history_expire_time: 1d #scheam清洗的时间 1天
       # The value of net_thread_count had better be same as cpu core number.
       net_thread_count: 4 #网络线程总数
       major_freeze_duty_time: Disable
       minor_freeze_times: 10
       enable_separate_sys_clog: 0 # 是否启用clog阻塞
       enable_merge_by_turn: FALSE # 是否启用按照顺序合并
       #datafile_disk_percentage: 20 # The percentage of the data_dir space to the total disk space. This value takes effect only when datafile_size is 0. The default value is 90.
       datafile_size: 50G  # 数据初始文件大小 最小就是50G
       syslog_level: WARN # System log level. The default value is INFO. # 系统日志级别
       enable_syslog_wf: false # Print system logs whose levels are higher than WARNING to a separate log file. The default value is true. # 是否启用日志打印
       enable_syslog_recycle: true # Enable auto system log recycling or not. The default value is false. # 是否启用日志回收
       max_syslog_file_count: 10 # The maximum number of reserved log files before enabling auto recycling. The default value is 0.# 日志回收最大文件数量
       # observer cluster name, consistent with obproxy cluster_name
       appname: makjobce   # 集群名字需要和集群标识不一样
       root_password: mkj123. # root user password, can be empty  # 集群管理员密码 就是root@sys的账户密码
       proxyro_password: mkj123. # proxyro user pasword, consistent with obproxy observer_sys_password, can be empty
     obs1:
       mysql_port: 2881 # External port for OceanBase Database. The default value is 2881. # mysql端口
       rpc_port: 2882 # Internal port for OceanBase Database. The default value is 2882. # 集群rpc的端口
       #  The working directory for OceanBase Database. OceanBase Database is started under this directory. This is a required field.# obproxy连接集群使用的密码
       home_path: /home/admin/oceanbase-ce #安装目录
       # The directory for data storage. The default value is $home_path/store.
       data_dir: /data  # 数据目录
       # The directory for clog, ilog, and slog. The default value is the same as the data_dir value.
       redo_dir: /redo # redo目录
       zone: zone1 # 属于第几个zone 
     obs2:
       mysql_port: 2881 # External port for OceanBase Database. The default value is 2881.
       rpc_port: 2882 # Internal port for OceanBase Database. The default value is 2882.
       #  The working directory for OceanBase Database. OceanBase Database is started under this directory. This is a required field.
       home_path: /home/admin/oceanbase-ce
       # The directory for data storage. The default value is $home_path/store.
       data_dir: /data
       # The directory for clog, ilog, and slog. The default value is the same as the data_dir value.
       redo_dir: /redo
       zone: zone2
     obs3:
       mysql_port: 2881 # External port for OceanBase Database. The default value is 2881.
       rpc_port: 2882 # Internal port for OceanBase Database. The default value is 2882.
       #  The working directory for OceanBase Database. OceanBase Database is started under this directory. This is a required field.
       home_path: /home/admin/oceanbase-ce
       # The directory for data storage. The default value is $home_path/store.
       data_dir: /data
       # The directory for clog, ilog, and slog. The default value is the same as the data_dir value.
       redo_dir: /redo
       zone: zone3
     obs4:
       mysql_port: 2881 # External port for OceanBase Database. The default value is 2881. # mysql端口
       rpc_port: 2882 # Internal port for OceanBase Database. The default value is 2882. # 集群rpc的端口
       #  The working directory for OceanBase Database. OceanBase Database is started under this directory. This is a required field.# obproxy连接集群使用的密码
       home_path: /home/admin/oceanbase-ce #安装目录
       # The directory for data storage. The default value is $home_path/store.
       data_dir: /data  # 数据目录
       # The directory for clog, ilog, and slog. The default value is the same as the data_dir value.
       redo_dir: /redo # redo目录
       zone: zone4 # 属于第几个zone 
     obs5:
       mysql_port: 2881 # External port for OceanBase Database. The default value is 2881.
       rpc_port: 2882 # Internal port for OceanBase Database. The default value is 2882.
       #  The working directory for OceanBase Database. OceanBase Database is started under this directory. This is a required field.
       home_path: /home/admin/oceanbase-ce
       # The directory for data storage. The default value is $home_path/store.
       data_dir: /data
       # The directory for clog, ilog, and slog. The default value is the same as the data_dir value.
       redo_dir: /redo
       zone: zone5
   obproxy:  # obproxy配置
     servers:
       - 192.168.10.160
       - 192.168.10.161
       - 192.168.10.162
     # Set dependent components for the component.
     # When the associated configurations are not done, OBD will automatically get the these configurations from the dependent components.
     depends: # obproxy依赖的配置 这个名字就对应上面的 #配置集群区域名字oceanbase-ce
       - oceanbase-ce
     global: # obproxy 全局配置
       listen_port: 2883 # External port. The default value is 2883. # 监听端口
       prometheus_listen_port: 2884 # The Prometheus port. The default value is 2884. # 和prometheus的通信端口
       home_path: /home/admin/obproxy # 安装目录
       # oceanbase root server list
       # format: ip:mysql_port;ip:mysql_port # 对应需要代理分发的数据库服务
       rs_list: 192.168.10.160:2881;192.168.10.161:2881;192.168.10.162:2881;192.168.10.163:2881;192.168.10.164:2881
       enable_cluster_checkout: false # 是否做集群检测
       # observer cluster name, consistent with oceanbase-ce appname
       cluster_name: makjobce # 集群名字
       obproxy_sys_password: mkj123. # obproxy sys user password, can be empty  # obproxy的sys用户的密码
       observer_sys_password: mkj123. # proxyro user pasword, consistent with oceanbase-ce proxyro_password, can be empty  # observer的sys用户的密码
   ```

6. 配置完成后再次启动老集群 不是重启 直接启动 就是如果原来已经存在，现在启动只启动新增的节点，此刻启动的时候 节点并没有加到集群里，集群列表里还没这俩节点

   ```
   $ obd cluster start makjobce
   ```

7. 集群添加新节点

   1. root进入到sys租户  查看集群状态 看到只有三个节点

      ```
      $ obclient -h192.168.10.160 -P2883 -uroot@sys#makjobce -pmkj123. -c -A oceanbase
      MySQL [oceanbase]> select svr_ip,id,zone,status from __all_server;
      ```

   2. 添加zone,默认sys_region,一个一个添加

      ```
      #添加zone
      alter system add zone 'zone4' region 'sys_region';
      
      alter system add zone 'zone5' region 'sys_region';
      ```

   3. 启动zone

      ```
      # 启动zone
      alter system start zone 'zone4';
      alter system start zone 'zone5';
      ```

   4. 将节点添加到zone中

      ```
      alter system add server '192.168.10.163:2882' zone 'zone4';
      
      alter system add server '192.168.10.164:2882' zone 'zone5';
      
      #如果添加错了可以删除
      alter system delete server '192.168.10.164:2882' zone 'zone5';
      ```

   5. 将服务启动起来

      ```
      alter system start server '192.168.10.163:2882' zone 'zone4';
      alter system start server '192.168.10.164:2882' zone 'zone5'
      ```

   6. 查看状态是否active

      ```
      MySQL [oceanbase]> select svr_ip,id,zone,status from __all_server;
      +----------------+----+-------+--------+
      | svr_ip         | id | zone  | status |
      +----------------+----+-------+--------+
      | 192.168.10.160 |  1 | zone1 | active |
      | 192.168.10.161 |  2 | zone2 | active |
      | 192.168.10.162 |  3 | zone3 | active |
      | 192.168.10.163 |  4 | zone4 | active |
      | 192.168.10.164 |  5 | zone5 | active |
      +----------------+----+-------+--------+
      5 rows in set (0.001 sec)
      
      [admin@ocp ~]$ obd cluster display makjobce
      Get local repositories and plugins ok
      Open ssh connection ok
      Cluster status check ok
      Connect to observer ok
      Wait for observer init ok
      +--------------------------------------------------+
      |                     observer                     |
      +----------------+---------+------+-------+--------+
      | ip             | version | port | zone  | status |
      +----------------+---------+------+-------+--------+
      | 192.168.10.160 | 3.1.2   | 2881 | zone1 | active |
      | 192.168.10.161 | 3.1.2   | 2881 | zone2 | active |
      | 192.168.10.162 | 3.1.2   | 2881 | zone3 | active |
      | 192.168.10.163 | 3.1.2   | 2881 | zone4 | active |
      | 192.168.10.164 | 3.1.2   | 2881 | zone5 | active |
      +----------------+---------+------+-------+--------+
      
      Connect to obproxy ok
      +--------------------------------------------------+
      |                     obproxy                      |
      +----------------+------+-----------------+--------+
      | ip             | port | prometheus_port | status |
      +----------------+------+-----------------+--------+
      | 192.168.10.160 | 2883 | 2884            | active |
      | 192.168.10.161 | 2883 | 2884            | active |
      | 192.168.10.162 | 2883 | 2884            | active |
      +----------------+------+-----------------+--------+
      ```

   7. 查看zone信息 确认是否都存在正常

      ```
      select * from __all_zone where name in ('region','status','zone_type');
      ```

   8. 查看租户资源情况，发现租户tent_makj2并没有扩展到zone4和zong5

      ```
      select t1.name resource_pool_name, t2.`name` unit_config_name,
      t2.max_cpu, t2.min_cpu, t2.max_memory/1024/1024/1024 max_mem_gb,
      t2.min_memory/1024/1024/1024 min_mem_gb, t3.unit_id, t3.zone,
      concat(t3.svr_ip,':',t3.`svr_port`) observer,t4.tenant_id,
      t4.tenant_name
      from __all_resource_pool t1 join __all_unit_config t2 on
      (t1.unit_config_id=t2.unit_config_id)
      join __all_unit t3 on (t1.`resource_pool_id` =
      t3.`resource_pool_id`)
      left join __all_tenant t4 on (t1.tenant_id=t4.tenant_id)
      order by t1.`resource_pool_id`, t2.`unit_config_id`, t3.unit_id;
      
      select t1.name resource_pool_name,
      t2. name unit_config_name,
      t2.max_cpu,t2.min_cpu,
      t2.max_memory / 1024 / 1024 / 1024 max_mem_gb,
      t2.min_memory / 1024 / 1024 / 1024 min_mem_gb,
      t3.unit_id,
      t3.zone,
      concat(t3.svr_ip, ':', t3. svr_port) observer,
      t4.tenant_id,
      t4.tenant_name
      from __all_resource_pool t1
      join __all_unit_config t2
      on (t1.unit_config_id = t2.unit_config_id)
      join __all_unit t3
      on (t1. resource_pool_id = t3. resource_pool_id)
      left join __all_tenant t4
      on (t1.tenant_id = t4.tenant_id)
      order by t1. resource_pool_id, t2. unit_config_id, t3.unit_id;
      ```

   9. 扩展租户所对应的资源池tent_makj2对应 pool_makj4 将zone4和zone5扩容进去

      ```
      alter resource pool pool_makj4 zone_list=('zone1','zone2','zone3','zone4') ;
      alter resource pool pool_makj4 zone_list=('zone1','zone2','zone3','zone4','zone5') ;
      ```

   10. 扩容租户tent_makj1的locality资源

       ```
       alter tenant tent_makj2 locality='FULL{1}@zone1, FULL{1}@zone2, FULL{1}@zone3';
       alter tenant tent_makj2 locality='FULL{1}@zone1, FULL{1}@zone2, FULL{1}@zone3, FULL{1}@zone4';
       alter tenant tent_makj2 locality='FULL{1}@zone1, FULL{1}@zone2, FULL{1}@zone3, FULL{1}@zone4, FULL{1}@zone5';
       
       #在扩展途中查看任务日志是否完成 完成后才能进行下一步
       select gmt_create,gmt_modified,job_id,job_type,job_status,return_code,progress,tenant_id from __all_rootservice_job;
       ```

   11. 扩展完成后查看租户情况 资源情况是否都是5副本了  去确认数据库中的数据是否还存在

       ```
       # 先查看任务是否执行完成 
       select gmt_create,gmt_modified,job_id,job_type,job_status,return_code,progress,tenant_id from __all_rootservice_job;
       
       #查看租户情况
       select * from __all_tenant;
       
       #查看集群zone扩展情况
       select a.zone,
       concat(a.svr_ip, ':', a.svr_port) observer,
       cpu_total,
       (cpu_total - cpu_assigned) cpu_free,
       round(mem_total / 1024 / 1024 / 1024) mem_total_gb,
       round((mem_total - mem_assigned) / 1024 / 1024 / 1024)
       mem_free_gb,
       usec_to_time(b.last_offline_time) last_offline_time,
       usec_to_time(b.start_service_time) start_service_time,
       b.status,
       usec_to_time(b.stop_time) stop_time
       from __all_virtual_server_stat a
       join __all_server b
       on (a.svr_ip = b.svr_ip and a.svr_port = b.svr_port)
       order by a.zone, a.svr_ip;
       
       #查看租户扩展结果
       select t1.name resource_pool_name,
       t2. name unit_config_name,
       t2.max_cpu,
       t2.min_cpu,
       t2.max_memory / 1024 / 1024 / 1024 max_mem_gb,
       t2.min_memory / 1024 / 1024 / 1024 min_mem_gb,
       t3.unit_id,
       t3.zone,
       concat(t3.svr_ip, ':', t3. svr_port) observer,
       t4.tenant_id,
       t4.tenant_name
       from __all_resource_pool t1
       join __all_unit_config t2
       on (t1.unit_config_id = t2.unit_config_id)
       join __all_unit t3
       on (t1. resource_pool_id = t3. resource_pool_id)
       left join __all_tenant t4
       on (t1.tenant_id = t4.tenant_id)
       order by t1. resource_pool_id, t2. unit_config_id, t3.unit_id;
       select * from __all_tenant;
       
       ```

   12. 也可以新建一个资源池 然后新建一个租户 把租户指定到新的池里 达到5副本效果

       ```
       create resource unit unit_makj2 min_cpu=4,max_cpu=4,min_memory='1G',max_memory='1G';
       create resource pool pool_makj2 unit='unit_makj2' , unit_num=1,zone_list=('zone1','zone2','zone3','zone4','zone5');
       create tenant tenant_makj2 resource_pool_list=('pool_makj2'),primary_zone='zone1',comment 'mysql tenant/instance', charset='utf8' set ob_tcp_invited_nodes='%' ;
       ```

   13. 清理之前扩容的时候创建的临时集群

       ```
       # 直接删除掉配置文件
       $ obd cluster list
       +------------------------------------------------------------------+
       |                           Cluster List                           |
       +-----------+------------------------------------+-----------------+
       | Name      | Configuration Path                 | Status (Cached) |
       +-----------+------------------------------------+-----------------+
       | makjobce  | /home/admin/.obd/cluster/makjobce  | running         |
       | makjobce2 | /home/admin/.obd/cluster/makjobce2 | deployed        |
       +-----------+------------------------------------+-----------------+
       # 删掉配置文件 然后销毁
       $ rm -rf /home/admin/.obd/cluster/makjobce2 ^C
       $ obd cluster destroy makjobce2
       
       # 最后在查询 没了
       $ obd cluster list
       ```

# 集群收缩

集群收缩，和扩容相反 删掉之前扩容的zone5和zone4 ,root@sys下执行

1. 收缩前先进行一次合并 使内存数据刷入磁盘

   ```
   alter system major freeze;
   ```

2. 先将刚才扩容的tent_makj2的locality改为三个zone

   ```
   alter tenant tent_makj2 locality='FULL{1}@zone1, FULL{1}@zone2,FULL{1}@zone3, FULL{1}@zone4';
   
   alter tenant tent_makj2 locality='FULL{1}@zone1, FULL{1}@zone2,FULL{1}@zone3';
   
   select gmt_create,gmt_modified,job_id,job_type,job_status,return_code,progress,tenant_id from __all_rootservice_job;
   ```

3. 查看租户是否收缩成功

   ```
   select * from __all_tenant;
   ```

4. 收缩资源池

   ```
   alter resource pool pool_makj4 zone_list=('zone1','zone2','zone3','zone4');
   alter resource pool pool_makj4 zone_list=('zone1','zone2','zone3');
   ```

5. 确认资源池是否收缩成功

   ```
   select t1.name resource_pool_name, t2.`name` unit_config_name,
   t2.max_cpu, t2.min_cpu, t2.max_memory/1024/1024/1024 max_mem_gb,
   t2.min_memory/1024/1024/1024 min_mem_gb, t3.unit_id, t3.zone,
   concat(t3.svr_ip,':',t3.`svr_port`) observer,t4.tenant_id,
   t4.tenant_name
   from __all_resource_pool t1 join __all_unit_config t2 on
   (t1.unit_config_id=t2.unit_config_id)
   join __all_unit t3 on (t1.`resource_pool_id` =
   t3.`resource_pool_id`)
   left join __all_tenant t4 on (t1.tenant_id=t4.tenant_id)
   order by t1.`resource_pool_id`, t2.`unit_config_id`, t3.unit_id;
   
   select t1.name resource_pool_name,
   t2. name unit_config_name,
   t2.max_cpu,t2.min_cpu,
   t2.max_memory / 1024 / 1024 / 1024 max_mem_gb,
   t2.min_memory / 1024 / 1024 / 1024 min_mem_gb,
   t3.unit_id,
   t3.zone,
   concat(t3.svr_ip, ':', t3. svr_port) observer,
   t4.tenant_id,
   t4.tenant_name
   from __all_resource_pool t1
   join __all_unit_config t2
   on (t1.unit_config_id = t2.unit_config_id)
   join __all_unit t3
   on (t1. resource_pool_id = t3. resource_pool_id)
   left join __all_tenant t4
   on (t1.tenant_id = t4.tenant_id)
   order by t1. resource_pool_id, t2. unit_config_id, t3.unit_id;
   ```

6. 删除zone4和zone5,这里如果还有别的租户和资源池使用zone4和zone5 要先收缩掉，删除zone4和zone5之前，确保没有池和租户使用这俩zone

   ```
   # 查看zone信息
   select gmt_create,gmt_modified,job_id,job_type,job_status,return_code,
   progress,tenant_id from __all_rootservice_job;
   select a.zone,
   concat(a.svr_ip, ':', a.svr_port) observer,
   cpu_total,
   (cpu_total - cpu_assigned) cpu_free,
   round(mem_total / 1024 / 1024 / 1024) mem_total_gb,
   round((mem_total - mem_assigned) / 1024 / 1024 / 1024)
   mem_free_gb,
   usec_to_time(b.last_offline_time) last_offline_time,
   usec_to_time(b.start_service_time) start_service_time,
   b.status,
   usec_to_time(b.stop_time) stop_time
   from __all_virtual_server_stat a
   join __all_server b
   on (a.svr_ip = b.svr_ip and a.svr_port = b.svr_port)
   order by a.zone, a.svr_ip;
   
   # 删除zone4和zone5
   alter system stop server '192.168.10.163:2882' zone 'zone4';
   alter system delete server '192.168.10.163:2882' zone 'zone4';
   alter system stop zone 'zone4';
   alter system delete zone 'zone4';
   
   alter system stop server '192.168.10.164:2882' zone 'zone5';
   alter system delete server '192.168.10.164:2882' zone 'zone5';
   alter system stop zone 'zone5';
   alter system delete zone 'zone5';
   
   select gmt_create,gmt_modified,job_id,job_type,job_status,return_code,progress,tenant_id from __all_rootservice_job;
   ```

7. 最后清理主机 然后关机下线

   ```
   for obid in `pidof observer`; do ls -l /proc/$obid/cwd; done
   kill -9 9648
   rm -rf /data/
   rm -rf /home/admin/.obd
   卸载软件：
   rpm -e `rpm -qa|grep oceanbase`
   rm /home/admin/oceanbase*
   ```

8. 修改配置文件，删掉zone4和zone5信息

   ```
   $ vim /home/admin/.obd/cluster/makjobce/config.yaml 
   # Only need to configure when remote login is required
   user:
      username: admin  #部署的用户名 
      password: admin  #密码
      key_file: /home/admin/.ssh/id_rsa.pub
      port: your ssh port, default 22  #端口
   #   timeout: ssh connection timeout (second), default 30
   oceanbase-ce:  #配置集群区域名字
     servers:  #服务器参数
       - name: obs1  #节点名字 共三台
         # Please don use hostname, only IP can be supported
         ip: 192.168.10.160
       - name: obs2
         ip: 192.168.10.161
       - name: obs3
         ip: 192.168.10.162
     global:  # 全局配置
       # Please set devname as the network adaptor name whose ip is  in the setting of severs.
       # if set severs as "127.0.0.1", please set devname as "lo"
       # if current ip is 192.168.1.10, and the ip's network adaptor's name is "eth0", please use "eth0"
       devname: ens192  #指定网卡
       cluster_id: 2  #集群id 可以自定义 整数 默认2
       # please set memory limit to a suitable value which is matching resource.
       memory_limit: 8G # The maximum running memory for an observer  #observer 内存限制，最低8G 这里给8G 默认系统的80%
       system_memory: 3G # The reserved system memory. system_memory is reserved for general tenants. The default value is 30G. #observer 系统内存 算在8G里面的 这部分是所有租户的共享内存
       stack_size: 512K  #堆栈大小 
       cpu_count: 16   #cpu数量 这里给默认 也可以根据真实环境改变 最多16个
       cache_wash_threshold: 1G  #缓存清洗的值 默认就好
       __min_full_resource_pool_memory: 268435456  #最小资源池大小
       workers_per_cpu_quota: 10 #每个cpu工作的线程数
       schema_history_expire_time: 1d #scheam清洗的时间 1天
       # The value of net_thread_count had better be same as cpu core number.
       net_thread_count: 4 #网络线程总数
       major_freeze_duty_time: Disable
       minor_freeze_times: 10
       enable_separate_sys_clog: 0 # 是否启用clog阻塞
       enable_merge_by_turn: FALSE # 是否启用按照顺序合并
       #datafile_disk_percentage: 20 # The percentage of the data_dir space to the total disk space. This value takes effect only when datafile_size is 0. The default value is 90.
       datafile_size: 50G  # 数据初始文件大小 最小就是50G
       syslog_level: WARN # System log level. The default value is INFO. # 系统日志级别
       enable_syslog_wf: false # Print system logs whose levels are higher than WARNING to a separate log file. The default value is true. # 是否启用日志打印
       enable_syslog_recycle: true # Enable auto system log recycling or not. The default value is false. # 是否启用日志回收
       max_syslog_file_count: 10 # The maximum number of reserved log files before enabling auto recycling. The default value is 0.# 日志回收最大文件数量
       # observer cluster name, consistent with obproxy cluster_name
       appname: makjobce   # 集群名字需要和集群标识不一样
       root_password: mkj123. # root user password, can be empty  # 集群管理员密码 就是root@sys的账户密码
       proxyro_password: mkj123. # proxyro user pasword, consistent with obproxy observer_sys_password, can be empty
     obs1:
       mysql_port: 2881 # External port for OceanBase Database. The default value is 2881. # mysql端口
       rpc_port: 2882 # Internal port for OceanBase Database. The default value is 2882. # 集群rpc的端口
       #  The working directory for OceanBase Database. OceanBase Database is started under this directory. This is a required field.# obproxy连接集群使用的密码
       home_path: /home/admin/oceanbase-ce #安装目录
       # The directory for data storage. The default value is $home_path/store.
       data_dir: /data  # 数据目录
       # The directory for clog, ilog, and slog. The default value is the same as the data_dir value.
       redo_dir: /redo # redo目录
       zone: zone1 # 属于第几个zone 
     obs2:
       mysql_port: 2881 # External port for OceanBase Database. The default value is 2881.
       rpc_port: 2882 # Internal port for OceanBase Database. The default value is 2882.
       #  The working directory for OceanBase Database. OceanBase Database is started under this directory. This is a required field.
       home_path: /home/admin/oceanbase-ce
       # The directory for data storage. The default value is $home_path/store.
       data_dir: /data
       # The directory for clog, ilog, and slog. The default value is the same as the data_dir value.
       redo_dir: /redo
       zone: zone2
     obs3:
       mysql_port: 2881 # External port for OceanBase Database. The default value is 2881.
       rpc_port: 2882 # Internal port for OceanBase Database. The default value is 2882.
       #  The working directory for OceanBase Database. OceanBase Database is started under this directory. This is a required field.
       home_path: /home/admin/oceanbase-ce
       # The directory for data storage. The default value is $home_path/store.
       data_dir: /data
       # The directory for clog, ilog, and slog. The default value is the same as the data_dir value.
       redo_dir: /redo
       zone: zone3
   obproxy:  # obproxy配置
     servers:
       - 192.168.10.160
       - 192.168.10.161
       - 192.168.10.162
     # Set dependent components for the component.
     # When the associated configurations are not done, OBD will automatically get the these configurations from the dependent components.
     depends: # obproxy依赖的配置 这个名字就对应上面的 #配置集群区域名字oceanbase-ce
       - oceanbase-ce
     global: # obproxy 全局配置
       listen_port: 2883 # External port. The default value is 2883. # 监听端口
       prometheus_listen_port: 2884 # The Prometheus port. The default value is 2884. # 和prometheus的通信端口
       home_path: /home/admin/obproxy # 安装目录
       # oceanbase root server list
       # format: ip:mysql_port;ip:mysql_port # 对应需要代理分发的数据库服务
       rs_list: 192.168.10.160:2881;192.168.10.161:2881;192.168.10.162:2881;
       enable_cluster_checkout: false # 是否做集群检测
       # observer cluster name, consistent with oceanbase-ce appname
       cluster_name: makjobce # 集群名字
       obproxy_sys_password: mkj123. # obproxy sys user password, can be empty  # obproxy的sys用户的密码
       observer_sys_password: mkj123. # proxyro user pasword, consistent with oceanbase-ce proxyro_password, can be empty  # observer的sys用户的密码
   
   ```

9. 修改完成后重新执行start加载新的配置

   ```
   $ obd  cluster start makjobce
   ```

10. 最后查看集群是否收缩完成

    ```
    $ obd  cluster list
    +----------------------------------------------------------------+
    |                          Cluster List                          |
    +----------+-----------------------------------+-----------------+
    | Name     | Configuration Path                | Status (Cached) |
    +----------+-----------------------------------+-----------------+
    | makjobce | /home/admin/.obd/cluster/makjobce | running         |
    +----------+-----------------------------------+-----------------+
    ```

# OBAgent

就是一个监控的采集框架

git信息：https://github.com/oceanbase/obagent/blob/master/docs/install-and-deploy/deploy-obagent-with-obd.md

示例配置：https://github.com/oceanbase/obdeploy/blob/master/example/obagent/obagent-only-example.yaml

OBAgent 是用GO 语言开发的监控采集框架，通常部署在observer 节点上。
OBAgent 支持推、拉两种数据采集模式，可以满足不同的应用场景。
OBAgent 默认支持的插件包括主机数据采集、OceanBase 数据库指标的采集、监控数据标签处理和Prometheus 协议的HTTP 服务。
要使OBAgent 支持其他数据源的采集，或者自定义数据的处理流程，您只需要开发对应的插件即可。
01.编辑OBAgent 部署配置文件
OBAgent 部署配置文件可以跟OceanBase 集群部署配置文件一起，也可以后期单独部署。
采用单独的配置文件部署OBAgent
OBAgent 的部署配置文件风格跟OceanBase 集群部署配置文件一样。

1. 创建监控用户 并且授权

   ```
   obclient -h 192.168.1.161 -uroot@sys#makjobce -P2883 -pmkj123. -c -A oceanbase
   
   use oceanbase;
   create user monitor identified by 'monitor';
   grant select on oceanbase.* to monitor;
   ```

2. 编辑配置文件

   ```
   vim obagent-only.yzml
   ## Only need to configure when remote login is required
   # user:
   #   username: your username
   #   password: your password if need
   #   key_file: your ssh-key file path if need
   #   port: your ssh port, default 22
   #   timeout: ssh connection timeout (second), default 30
   obagent:
     servers:  #对应的需要部署的obs的节点ip 写ip不要用主机名
       # Please don't use hostname, only IP can be supported
       - 192.168.10.160
       - 192.168.10.161
       - 192.168.10.162
     global:
       # The working directory for obagent. obagent is started under this directory. This is a required field.
       home_path: /home/admin/obagent #配置的目录
       # The port that pulls and manages the metrics. The default port number is 8088.
       server_port: 8088    #对外服务端口
       # Debug port for pprof. The default port number is 8089.
       pprof_port: 8089  #自己的默认端口
       # Log level. The default value is INFO.
       log_level: INFO  #日志级别
       # Log path. The default value is log/monagent.log.
       log_path: log/monagent.log  #日志路径
       # Encryption method. OBD supports aes and plain. The default value is plain.
       crypto_method: plain
       # Path to store the crypto key. The default value is conf/.config_secret.key.
       # crypto_path: conf/.config_secret.key
       # Size for a single log file. Log size is measured in Megabytes. The default value is 30M.
       log_size: 30  #日志大小
       # Expiration time for logs. The default value is 7 days.
       log_expire_day: 7  #保留7天
       # The maximum number for log files. The default value is 10.
       log_file_count: 10 #日志文件最大个数
       # Whether to use local time for log files. The default value is true.
       # log_use_localtime: true
       # Whether to enable log compression. The default value is true.
       # log_compress: true
       # Username for HTTP authentication. The default value is admin.
       http_basic_auth_user: admin  # 认证用户密码
       # Password for HTTP authentication. The default value is root.
       http_basic_auth_password: admin # 认证用户密码
       # Username for debug service. The default value is admin.
       pprof_basic_auth_user: admin # 认证用户密码
       # Password for debug service. The default value is root.
       pprof_basic_auth_password: admin # 认证用户密码
       # Monitor username for OceanBase Database. The user must have read access to OceanBase Database as a system tenant. The default value is root.
       monitor_user: monitor   #监控用户，oceanbase必须有这个用户
       # Monitor password for OceanBase Database. The default value is empty. When a depends exists, OBD gets this value from the oceanbase-ce of the depends. The value is the same as the root_password in oceanbase-ce.
       monitor_password: monitor  #监控用户密码，oceanbase必须有这个用户
       # The SQL port for observer. The default value is 2881. When a depends exists, OBD gets this value from the oceanbase-ce of the depends. The value is the same as the mysql_port in oceanbase-ce.
       sql_port: 2881  #obs 实例对外的端口
       # The RPC port for observer. The default value is 2882. When a depends exists, OBD gets this value from the oceanbase-ce of the depends. The value is the same as the rpc_port in oceanbase-ce.
       rpc_port: 2882 #obs 实例内部通信的rpc的端口
       # Cluster name for OceanBase Database. When a depends exists, OBD gets this value from the oceanbase-ce of the depends. The value is the same as the appname in oceanbase-ce.
       cluster_name: makjobce  #集群名字 必须和集群一致
       # Cluster ID for OceanBase Database. When a depends exists, OBD gets this value from the oceanbase-ce of the depends. The value is the same as the cluster_id in oceanbase-ce.
       cluster_id: 2  #集群id 必须和集群一致
       # Zone name for your observer. The default value is zone1. When a depends exists, OBD gets this value from the oceanbase-ce of the depends. The value is the same as the zone name in oceanbase-ce.
       zone_name: zone1,zone2,zone3  # 所有的zone信息
       # Monitor status for OceanBase Database.  Active is to enable. Inactive is to disable. The default value is active. When you deploy an cluster automatically, OBD decides whether to enable this parameter based on depends.
       ob_monitor_status: active
       # Monitor status for your host. Active is to enable. Inactive is to disable. The default value is active.
       host_monitor_status: active
       # Whether to disable the basic authentication for HTTP service. True is to disable. False is to enable. The default value is false.
       disable_http_basic_auth: false
       # Whether to disable the basic authentication for the debug interface. True is to disable. False is to enable. The default value is false.
       disable_pprof_basic_auth: false
   
   
   ```

3. 部署

   ```
   # 查看集群镜像里是否有obagent
   $ obd mirror list local
   
   #部署
   $ obd cluster deploy makjobagent -c obagent-only.yzml
   ```

4. 查看部署详情

   ```
   $ obd cluster list
   +----------------------------------------------------------------------+
   |                             Cluster List                             |
   +-------------+--------------------------------------+-----------------+
   | Name        | Configuration Path                   | Status (Cached) |
   +-------------+--------------------------------------+-----------------+
   | makjobce    | /home/admin/.obd/cluster/makjobce    | running         |
   | makjobagent | /home/admin/.obd/cluster/makjobagent | deployed        |
   +-------------+--------------------------------------+-----------------+
   ```

5. 启动agent 查看是否都是active

   ```
   $ obd cluster start makjobagent
   Get local repositories and plugins ok
   Open ssh connection ok
   Load cluster param plugin ok
   Check before start obagent ok
   Start obproxy ok
   obagent program health check ok
   +----------------------------------------------------+
   |                      obagent                       |
   +----------------+-------------+------------+--------+
   | ip             | server_port | pprof_port | status |
   +----------------+-------------+------------+--------+
   | 192.168.10.160 | 8088        | 8089       | active |
   | 192.168.10.161 | 8088        | 8089       | active |
   | 192.168.10.162 | 8088        | 8089       | active |
   +----------------+-------------+------------+--------+
   makjobagent running
   
   #检查三台机器进程是否存在
   # ps -ef|grep agent
   ```

6. 重启

   ```
   $ obd cluster restart makjobagent
   
   # 还可以杀掉节点进程 手动重启
   # kill -9 308962
   # cd /home/admin/obagent
   # nohup ./bin/monagent -c conf/monagent.yaml
   # ps -ef|grep agent
   ```

# OCP

1. MetaDB 配置要求

   MetaDB 用于存储除了监控数据的其它持久化数据，需要的资源相对较少，
   推荐在OceanBase 中创建独立的租户用于MetaDB，
   并最低分配给该租户4 个CPU、8G 内存。
   按照OCP 管理的observer 数量，推荐MetaDB 租户每个副本的CPU 和
   内存资源规则如下：
   管理的机器数量（台）CPU（核）内存（GB）
   ≤ 10 4 8
   ≤ 50 8 16
   ≤ 100 16 32
   ≤ 200 32 64
   ≤ 400 64 128

2. MonitorDB 配置要求

   OCP 在运行过程中，会采集管理目标的各种性能数据，主要是OceanBase
   集群的数据，这些数据都会存储到MonitorDB。
   推荐在OceanBase 中创建独立的租户用于MonitorDB，并最低分配给该
   租户4 个CPU、16G 内存。
   按照OCP 管理的observer 数量，推荐MonitorDB 租户每个副本的CPU
   和内存资源规则如下：
   管理的机器数量（台）CPU（核）内存（GB）
   ≤ 10 4 16
   ≤ 50 16 64
   ≤ 100 32 128
   ≤ 200 64 256
   ≤ 400 128 512

3. OCP 配置要求

   OCP-Server 机器的CPU 和内存资源配置参考如下表所示。
   以下配置是以每个observer 中包含10 个租户为标准，进行计算而得出
   的数据，请您根据实际情况进行计算，选择合适的CPU 和内存。
   管理的机器数量（台）CPU（核）内存（GB）
   ≤ 10 4 8
   ≤ 50 8 16
   ≤ 100 16 32
   ≤ 200 32 64
   ≤ 400 64 128
   OCP-Server 机器要求的最低配置为4 核8 GB 以及500 GB 可用磁盘空
   间，当单个主机的租户数量≤ 10 时，
   仍建议CPU 和内存保持为4 核8 GB。
   规划示例
   部署内容CPU（核）内存（GB）
   ocp-server 4 8
   ocp-metadb 租户4 8
   ocp-monitordb 租户4 16
   sys 租户5 28
   即这台物理机需要有24 个CPU 和64G 内存

4. OCP 磁盘规划

   OCP 以Docker 形式部署，OCP-Server 在运行过程中会写入文件到磁盘，
   这些数据会占用磁盘空间，
   推荐至少需要预留350 GB 的空间给OCP-Server Docker 使用，以免因
   磁盘写满导致功能受影响。
   OCP 程序10G
   /home/admin/ocp-server/
   /home/admin/ocp-init/
   OCP 主进程日志/home/admin/logs/ocp/ 100G
   OCP 系统参数配置了log 文件大小和保留的文件数量。
   默认配置为单个文件100 MB，每天至少5-10 个文件，每个文件最多保
   留100 天，建议预留100 GB 空间。
   OCP 任务日志/home/admin/logs/task/ 100G
   未设置容量限制，OCP 会定时清理任务日志，清理任务按照磁盘剩余空间
   大于20 % 的目标清理日志文件。
   清理时，首先清理更新时间更早的日志文件，需预留100 GB 空间。
   OCP 文件本地缓存/home/admin/data/files/ 100G
   未设置容量限制，OCP 文件主要是OCP 管理对象相关的安装包，且主要
   是OceanBase 安装包，安装包大小平均在500 MB 左右，
   按照100 个安装包预估，需预留100 GB 空间。

5. 网络端口

   MetaDB & MonitorDB OceanBase 数据库
   2881 observer SQL 监听端口。不支持修改tcp
   2882 远程访问端口。不支持修改tcp

   OCP OCP-Server
   8080 http
   修改OCP 系统参数server.port,重启生效。
   OCP-Server web 服务监听端口，通常其它组件通过SLB/DNS 地址访问

6. 依赖安装包

   * OBClient 的依赖包和安装包
     libobclient-2.0.0-2.el7.x86_64.rpm
     obclient-2.0.0-2.el7.x86_64.rpm
   * OBD 安装包
     ob-deploy-1.1.0-1.el7.x86_64.rpm
   * OCP 安装包
     ocp-3.1.1-ce.tar.gz
   * Docker 离线安装（Linux）

7. 安装docker

   ```
   # yum install -y yum-utils
   # yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
   # yum install -y docker-ce
   # service docker start
   ```

8. 安装sshpass 也可以配置免密

   ```
   # yum -y install sshpass
   # yum -y install net-tools
   ```

9. 配置root和admin用户免密

   ```
   # root用户配置自己免密
   # ssh-keygen
   # ssh-copy-id -i /root/.ssh/id_rsa.pub opc
   
   # admin用户配置全部免密
   $ ssh-keygen
   $ ssh-copy-id -i /home/admin/.ssh/id_rsa.pub obs1
   $ ssh-copy-id -i /home/admin/.ssh/id_rsa.pub obs2
   $ ssh-copy-id -i /home/admin/.ssh/id_rsa.pub obs3
   $ ssh-copy-id -i /home/admin/.ssh/id_rsa.pub obs4
   $ ssh-copy-id -i /home/admin/.ssh/id_rsa.pub obs5
   $ ssh-copy-id -i /home/admin/.ssh/id_rsa.pub obs6
   $ ssh-copy-id -i /home/admin/.ssh/id_rsa.pub obc
   $ ssh-copy-id -i /home/admin/.ssh/id_rsa.pub ocp
   ```

   

10. 安装ob-deploy

    ```
    # rpm -i ob-deploy-1.2.1-9.el7.x86_64.rpm 
    # 安装完后启动
    # source /etc/profile.d/obd.sh
    # 执行任意一个命令 产生本地文件
    $ obd mirror list
    ```

11. 生成本地源

    ```
    # su - admin
    $ cd /home/admin/.obd/mirror/
    $ mv remote remote-back
    # 拷贝系统文件到本地源 注意执行用户 否则会找不到包 这里用admin执行
    $ obd mirror clone /data/soft/*.rpm
    # 查看本地源 主要加载数据库包oceanbase-ce和oceanbase-ce-libs
    $ obd mirror list local
    ```

12. 准备配置文件

    ```
    $ vim mini-local.yaml
    :set paste
    oceanbase-ce:
      servers:
        # Please don't use hostname, only IP can be supported
        - 192.168.10.167  #ip地址 写本地地址
      global:
        #  The working directory for OceanBase Database. OceanBase Database is started under this directory. This is a required field.
        home_path: /home/admin/observer  #安装目录
        # The directory for data storage. The default value is $home_path/store.
        # data_dir: /data
        # The directory for clog, ilog, and slog. The default value is the same as the data_dir value.
        # redo_dir: /redo
        # Please set devname as the network adaptor's name whose ip is  in the setting of severs.
        # if set severs as "127.0.0.1", please set devname as "lo"
        # if current ip is 192.168.1.10, and the ip's network adaptor's name is "eth0", please use "eth0"
        devname: ens192 # 网卡名字
        mysql_port: 2881 # External port for OceanBase Database. The default value is 2881. DO NOT change this value after the cluster is started.
        rpc_port: 2882 # Internal port for OceanBase Database. The default value is 2882. DO NOT change this value after the cluster is started.
        zone: zone1
        cluster_id: 1
        # please set memory limit to a suitable value which is matching resource. 
        memory_limit: 8G # The maximum running memory for an observer #内存限制 最大8G
        system_memory: 4G # The reserved system memory. system_memory is reserved for general tenants. The default value is 30G. #内存限制 最小4G
        stack_size: 512K
        cpu_count: 16
        cache_wash_threshold: 1G
        __min_full_resource_pool_memory: 268435456
        workers_per_cpu_quota: 10
        schema_history_expire_time: 1d
        # The value of net_thread_count had better be same as cpu's core number. 
        net_thread_count: 4
        sys_bkgd_migration_retry_num: 3
        minor_freeze_times: 10
        enable_separate_sys_clog: 0
        enable_merge_by_turn: FALSE
        datafile_disk_percentage: 20 # The percentage of the data_dir space to the total disk space. This value takes effect only when datafile_size is 0. The default value is 90.
        syslog_level: INFO # System log level. The default value is INFO.
        enable_syslog_wf: false # Print system logs whose levels are higher than WARNING to a separate log file. The default value is true.
        enable_syslog_recycle: true # Enable auto system log recycling or not. The default value is false.
        max_syslog_file_count: 4 # The maximum number of reserved log files before enabling auto recycling. The default value is 0.
        root_password: mkj123. # root user password, can be empty  #当前节点sys租户的root密码
    ```

13. 部署数据库

    ```
    $ obd cluster deploy makjocpdb -c mini-local.yaml
    ```

14. 部署完成后启动

    ```
    $ obd cluster start makjocpdb
    ```

15. 安装ob客户端

    ```
    $ cd /data/soft/
    $ sudo rpm -ivh libobclient-2.0.0-2.el7.x86_64.rpm obclient-2.0.0-2.el7.x86_64.rpm 
    ```

16. 登录到ocp节点数据库

    ```
    $ obclient -h192.168.10.167 -P2881 -uroot@sys -pmkj123.
    ```

17. 查看服务

    ```
    use oceanbase;
    #查看资源
    select * from __all_server;
    select * from __all_unit_config;
    
    # 查看当前系统的zone1资源单元
    select a.zone,
    concat(a.svr_ip, ':', a.svr_port) observer,
    cpu_total,
    (cpu_total - cpu_assigned) cpu_free,
    round(mem_total / 1024 / 1024 / 1024) mem_total_gb,
    round((mem_total - mem_assigned) / 1024 / 1024 / 1024)
    mem_free_gb,
    round(disk_total / 1024 / 1024 / 1024) disk_total_gb,
    substr(a.build_version, 1, 6) version,
    usec_to_time(b.start_service_time) start_service_time
    from __all_virtual_server_stat a
    join __all_server b
    on (a.svr_ip = b.svr_ip and a.svr_port = b.svr_port)
    order by a.zone, a.svr_ip;
    
    # 查看已经分配的资源
    select zone,
    cpu_total,
    cpu_assigned,
    mem_total / 1024 / 1024 / 1024,
    mem_assigned / 1024 / 1024 / 1024
    from __all_virtual_server_stat;
    
    select t1.name resource_pool_name,
    t2. name unit_config_name,
    t2.max_cpu,
    t2.min_cpu,
    round(t2.max_memory / 1024 / 1024 / 1024) max_mem_gb,
    round(t2.min_memory / 1024 / 1024 / 1024) min_mem_gb,
    t3.unit_id,
    t3.zone,
    concat(t3.svr_ip, ':', t3. svr_port) observer,
    t4.tenant_id,
    t4.tenant_name
    from __all_resource_pool t1
    join __all_unit_config t2
    on (t1.unit_config_id = t2.unit_config_id)
    join __all_unit t3
    on (t1. resource_pool_id = t3. resource_pool_id)
    left join __all_tenant t4
    on (t1.tenant_id = t4.tenant_id)
    order by t1. resource_pool_id, t2. unit_config_id, t3.unit_id;
    ```

18. 创建ocp资源单元

    ```
    create resource unit ocp_unit_config min_cpu=2, max_cpu=2,min_memory=2147483648, max_memory=2147483648,
    max_iops=1000, min_iops=128, max_disk_size='500G',
    max_session_num=100;
    
    #修改成1G 不然资源不够用了 一个单元一共就2G 一个资源池占用2G 就只能创建一个租户了 生产环境需要给足够大的内存 20G 30G
    alter resource unit ocp_unit_config min_cpu=2, max_cpu=2,min_memory='1G', max_memory='1G',
    max_iops=1000, min_iops=128, max_disk_size='500G',
    max_session_num=100;
    
    #创建完后查看
    select unit_config_id,
    name,
    max_cpu,
    min_cpu,
    round(max_memory / 1024 / 1024 / 1024) max_mem_gb,
    round(min_memory / 1024 / 1024 / 1024) min_mem_gb,
    round(max_disk_size / 1024 / 1024 / 1024) max_disk_size_gb
    from __all_unit_config
    order by unit_config_id;
    ```

19. 创建资源池和租户

    ```
    # 创建租户资源池
    create resource pool ocp_pool unit='ocp_unit_config',zone_list=('zone1'), unit_num=1;
    
    #创建meta租户
    create tenant ocp_meta resource_pool_list = ('ocp_pool');
    
    #创建monitot池
    create resource pool ocp_monitor_pool unit='ocp_unit_config',zone_list=('zone1'), unit_num=1;
    
    #创建monitot租户
    create tenant ocp_monitor resource_pool_list = ('ocp_monitor_pool');
    ```

20. 创建用户

    ```
    # 创建mate的admin 
    $ obclient -h127.0.0.1 -P2881 -uroot@ocp_meta
    set global ob_tcp_invited_nodes = '%';
    create user ocp identified by 'admin';
    grant all on *.* to ocp;
    
    #创建monitor的admin 
    $ obclient -h127.0.0.1 -P2881 -uroot@ocp_monitor 
    set global ob_tcp_invited_nodes = '%';
    create user ocp identified by 'admin';
    grant all on *.* to ocp;
    ```

21. 生成ocp配置文件

    ```
    # 解压包
    $ tar zxvf ocp-3.1.1-ce-bp1.tar.gz 
    # 生成配置文件
    $ cd ocp-ce-3.1.1-bp1/
    $ ./ocp_installer.sh genconf -c ../ocp.yaml
    ```

22. 修改配置文件

    ```
    $ vim /data/ocp.yaml
    # The ip address to deploy ocp, config multiple ip with an array (IP1 IP2 IP3)
    # The ip address should not be 127.0.0.1 or localhost
    # If the server has both public ip and private ip, private ip is ok
    OCP_IP_ARRAY=(192.168.10.167)  # ip地址 有几个写几个
    
    SSH_USER=root    # if not root, make sure remote user can use sudo without password    admin   ALL=(ALL)   NOPASSWD:ALL
    SSH_PORT=22
    SSH_AUTH=password     # can be password or pubkey
    SSH_PASSWORD='mkj123.'     # password for passowrd auth, when use pubkey auth, is passphrase
    SSH_KEY_FILE='/root/.ssh/id_rsa'   # pubkey auth
    
    # it's highly recommended to use separate tenant for metadb and monitordb
    # metadb host address, should not be 127.0.0.1 or localhost
    OCP_METADB_HOST=192.168.10.167 #mate数据库的数据库地址 
    OCP_METADB_PORT=2881
    OCP_METADB_USER=ocp@ocp_meta #用户
    OCP_METADB_PASSWORD='admin'   # 密码 password may contains special char, make sure correctly quote
    OCP_METADB_DBNAME=ocp_meta
    OCP_MONITORDB_USER=ocp@ocp_monitor   # monitor用户
    OCP_MONITORDB_PASSWORD='admin'    # password may contains special char, make sure correctly quote
    OCP_MONITORDB_DBNAME=ocp_monitor
    OCP_WEB_PORT=8080
    
    OCP_LB_VIP=       # HA OCP VIP
    OCP_LB_VPORT=     # HA OCP VPORT
    
    OCP_IMAGE=/data/soft/ocp-ce-3.1.1-bp1/ocp.tar.gz  # 部署到docker的images absoulute path of ocp image file
    OCP_CPU=4 #cpu
    OCP_MEMORY=8G #内存
    OCP_LOG_DIR=/data/ocp_logs
    ```

23. 部署images

    ```
    $ ./ocp_installer.sh install -c /data/ocp.yaml 
    # 部署完成后查看docker的ocp情况
    # docker stats ocp
    ```

24. 检查部署情况 检查docker信息和集群信息是否正常

    ```
    # 进入docker检查 docker信息
    # docker stats ocp
    # docker exec -it ocp bash
    [root@obc admin]# pgrep -a java
    39 /usr/lib/jvm/java-1.8.0/bin/java -server -XX:+UseG1GC -Xms5734m -Xmx5734m -Xss512k -XX:+PrintCommandLineFlags -XX:MetaspaceSize=1024m -XX:MaxMetaspaceSize=1024m -XX:+PrintAdaptiveSizePolicy -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/home/admin/ocp-server/bin/../log/gc.log -XX:+UseGCLogFileRotation -XX:GCLogFileSize=50M -XX:NumberOfGCLogFiles=2 -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/home/admin/ocp-server/bin/../log/ -Dfile.encoding=UTF-8 -jar /home/admin/ocp-server/bin/../lib/ocp-server-3.1.1-20210916.jar
    
    # 检查集群信息
    $ obd cluster list
    +------------------------------------------------------------------+
    |                           Cluster List                           |
    +-----------+------------------------------------+-----------------+
    | Name      | Configuration Path                 | Status (Cached) |
    +-----------+------------------------------------+-----------------+
    | makjocpdb | /home/admin/.obd/cluster/makjocpdb | running         |
    +-----------+------------------------------------+-----------------+
    
    $ obd cluster display makjocpdb
    Get local repositories and plugins ok
    Open ssh connection ok
    Cluster status check ok
    Connect to observer ok
    Wait for observer init ok
    +--------------------------------------------------+
    |                     observer                     |
    +----------------+---------+------+-------+--------+
    | ip             | version | port | zone  | status |
    +----------------+---------+------+-------+--------+
    | 192.168.10.167 | 3.1.2   | 2881 | zone1 | active |
    +----------------+---------+------+-------+--------+
    ```

25. 登录

    地址：http://192.168.10.167:8080/cluster

    用户名/密码：admin/root

26. 启动停止

    * 停止 先停止docker 然后停止数据库

      ```
      # 停止docker
      # systemctl stop docker.socket
      # systemctl stop docker
      
      #停止数据库
      $ obd cluster stop makjocpdb
      ```

      

    * 启动 先启动数据库 然后启动docker

      ```
      # 先启动数据库 
      $ obd cluster start makjocpdb
      
      #在启动docker
      # systemctl start docker
      ```

27. 可以通过ocp管理平台 创建集群 管理集群

    1. 在管理界面里选择接管集群，选择直连 填入任何一个集群的ip 就可以自动接管 接管完成后，会在各个节点上安装ocp-agent安装服务等信息
    2. 如果初始化失败了页面上删除失败的情况下，直接去数据库情况这几个表
       * ocp.compute_host
       * ocp.compute_idc
       * ocp.compute_region
       * ocp.ob_cluster
       * ocp.ob_server
       * ocp.obproxy_server
       * ocp_zone

# 命令

```
# 查看集群详情
$ obd cluster display makjobce

# 启动集群
$ obd cluster start makjobce 

# 停止集群 关机前一定要停止 否则内存的数据就会丢掉
$ obd cluster stop makjobce

# 重启集群
$ obd cluster restart makjobce

# 销毁集群
$ obd cluster destroy makjobce -f

# 停止某一个节点 先通过proxy登录到数据 然后停止
$ obclient -h192.168.10.160 -P2883 -uroot:sys:makjobce -pmkj123. -c -A oceanbase
MySQL [oceanbase]> alter system stop server '192.168.10.160:2882'
# 查看事件
MySQL [oceanbase]> select * from __all_rootservice_event_history where module in ('leader_coodinator','balancer');
#启动服务
MySQL [oceanbase]> alter system start server '192.168.10.160:2882';Query OK, 0 rows affected (0.01 sec)
```

1. 杀掉单节点进程

   ```
   $ ps -ef|grep observer # 或者pidof observer
   admin      3099      1 89 22:16 ?        00:32:29 /home/admin/oceanbase-ce/bin/observer -r 192.168.10.160:2882:2881;192.168.10.161:2882:2881;192.168.10.162:2882:2881 -o __min_full_resource_pool_memory=268435456,memory_limit=8G,system_memory=3G,stack_size=512K,cpu_count=16,cache_wash_threshold=1G,workers_per_cpu_quota=10,schema_history_expire_time=1d,net_thread_count=4,major_freeze_duty_time=Disable,minor_freeze_times=10,enable_separate_sys_clog=0,enable_merge_by_turn=False,datafile_size=50G,enable_syslog_wf=False,enable_syslog_recycle=True,max_syslog_file_count=10 -z zone1 -p 2881 -P 2882 -n makjobce -c 2 -d /data -i ens192 -l WARN
   admin      8392   2816  0 22:52 pts/0    00:00:00 grep --color=auto observer
   
   $ kill -9 3099
   
   #启动 就是上面的进程命令 把参数加进去
   $ cd /home/admin/oceanbase-ce/
   $ ./bin/observer -o datafile_size=50g # 随便带个参数
   
   # 查看集群状态是否都是active
   select a.zone,concat(a.svr_ip,':',a.svr_port) observer, cpu_total, (cpu_total-cpu_assigned) cpu_free, 
   round(mem_total/1024/1024/1024) mem_total_gb, round((mem_total-mem_assigned)/1024/1024/1024) mem_free_gb, 
   round(disk_total/1024/1024/1024) disk_total_gb,unit_num,
   substr(a.build_version,1,6) version,usec_to_time(b.start_service_time) start_service_time,b.status
   from __all_virtual_server_stat a join __all_server b on (a.svr_ip=b.svr_ip and a.svr_port=b.svr_port)
   order by a.zone, a.svr_ip;
   ```



# 集群启动参数

参数名 参数值 参数含义
cpu_count 16 默认取操作系统的CPU数减去2 ，也可以自定义数目。
memory_limit 8G 进程默认使用的最大内存。如果设置为。就是这个参数不限制。跟memory_limit_percentage二选一
memory_limit_percentage 80 进程默认使用的最大内存占总可用内存的百分比。跟memoryjimit二选一
datafile_size 0 进程的数据文件(block_file)的初始化大小。如果设置为0就是这个参数不限制。
datafile_disk_percentage 75 进程的数据文件(block_fi⑹的初始化大小占数据目录(sstable)所在文件系统可用空间大小的百分比。
clog_disk_usage_limit_percentage 95 进程的dog文件所在文件系统空间使用率上限百分比。达到这个值就被认为"空间满”，clog会停写。进程。bserver取得的资源中CPU个数是声明式的，内存资源是独占的，磁盘空间是独占的(预分配)。

# 集群信息查看

* 查看集群资源

  ```
  use oceanbase;
  select a.zone,concat(a.svr_ip,':',a.svr_port) observer, cpu_total, (cpu_total-cpu_assigned) cpu_free,
  round(mem_total/1024/1024/1024) mem_total_gb, round((mem_total-mem_assigned)/1024/1024/1024) mem_free_gb,
  round(disk_total/1024/1024/1024) disk_total_gb,unit_num,
  substr(a.build_version,1,6) version,usec_to_time(b.start_service_time) start_service_time,b.status
  from __all_virtual_server_stat a join __all_server b on (a.svr_ip=b.svr_ip and a.svr_port=b.svr_port)
  order by a.zone, a.svr_ip;
  ```

* 查看资源详情

  ```
  select t1.name resource_pool_name,
         t2. name unit_config_name,
         t2.max_cpu,
         t2.min_cpu,
         t2.max_memory / 1024 / 1024 / 1024 max_mem_gb,
         t2.min_memory / 1024 / 1024 / 1024 min_mem_gb,
         t3.unit_id,
         t3.zone,
         concat(t3.svr_ip, ':', t3. svr_port) observer,
         t4.tenant_id,
         t4.tenant_name
    from __all_resource_pool t1
    join __all_unit_config t2
      on (t1.unit_config_id = t2.unit_config_id)
    join __all_unit t3
      on (t1. resource_pool_id = t3. resource_pool_id)
    left join __all_tenant t4
      on (t1.tenant_id = t4.tenant_id)
   order by t1. resource_pool_id, t2. unit_config_id, t3.unit_id;
   
   # 建议把资源的最大值和最小值持平  修改系统资源
   alter resource unit sys_unit_config min_cpu=5,max_cpu=5,min_memory='1000000B',max_memory='1000000B'
  ```

# 日常监控

1. 使用linux自带的监控

   1. top
   2. free -h
   3. vmstat
   4. iostat -x -t 8关注iowait
   5. df
   6. du

2. tsar命令采集软件

   安装后会把采集数据放到/var/log下 文件名是tsar.data

   ```
   #查看历史性能 每分钟一次
   # tsar -l -i 1
   
   # 查看cpu load 网络磁性能等 每分钟一次
   # tsar --cpu --load --traficc -l -i 1
   
   # 看cpu性能
   # tsar --cpu --load -l -i 1
   
   #看网络性能
   # tsar --traficc -l -i 1
   
   #看io性能
   # tsar --io -l dm -0 -l -i 3
   
   #看内存
   # tsar --mem -l -i 3
   ```

# 故障分析

1. 查看集群节点状态

   ```
   select a.zone,
          concat(a.svr_ip, ':', a.svr_port) observer,
          cpu_total,
          cpu_assigned,
          (cpu_total - cpu_assigned) cpu_free,
          mem_total / 1024 / 1024 / 1024 mem_total_gb,
          mem_assigned / 1024 / 1024 / 1024 mem_assign_gb,
          (mem_total - mem_assigned) / 1024 / 1024 / 1024 mem_free_gb
     from __all_virtual_server_stat a
     join __all_server b
       on (a.svr_ip = b.svr_ip and a.svr_port = b.svr_port)
    order by a.zone, a.svr_ip;
   ```

   

2. 其次确认集群近期的事件

   ```
   SELECT *
     FROM __all_rootservice_event_history
    WHERE 1 = 1
      and gmt_create > date_format('2022-02-18 18:35:00', '%Y%m%d%H%i%s')
    ORDER BY gmt_create LIMIT 50;
   ```

3. 查看日志

   切换到admin用户下 直接执行 查看observer.log的error日志

   ```
   $ cd /home/admin/oceanbase-ce/
   $ ./bin/observer 
   ```

4. 查看时间是否同步

# 其他工具


DataX数据交换平台:
DataX也是阿里巴巴开源的一款实用产品，项目地址是：https://github.com/alibaba/datax。
DataX是阿里云DataWorks数据集成的开平版本，在阿里巴巴集团内被广泛使用的离线数据同步工具/平台。
Canal数据增量工具
Canal也是阿里巴巴开源的一款数据同步产品Canal主要用途是基于mysql数据库增量日志解析，提供增量数据订阅和消费。
dooba性能监控
dooba是OceanBase内部的一个运维脚本，用Python语言开发，支持Python 2.7 o
ob_admin 工具
ob_admin是OceanBase内部开发和运维使用一个工具，用于处理一些疑难问题。
ob_error 工具
ob_error是随着OceanBase一起开源的工具，

dooba性能监控
dooba是OceanBase内部的一个运维脚本，用Python语言开发，支持Python 2.7。
ob_admin 工具
ob_admin是OceanBase内部开发和运维使用一个工具，用于处理一些疑难问题。
ob_error 工具
ob_error是随着OceanBase一起开源的工具，主要用来查看OceanBase错误码含义，节省查文档时间。

OceanBase性能测试工具
对CPU/内存/线程/I0/数据库等方面的性能测试，用于评估系统在运行高负载的数据库时相关核心参数的性能表现
sriygql性能基准测试I _mysql数据库性能优化与运维诊断01
https://edu.51cto.com/course/14782.html
SYSBENCH 测试
TPC-C测试
TPC-H测试

使用Jmeter跑业务测试