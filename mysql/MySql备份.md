#MySql备份#

##备份命令##
`mysqldump --single-transaction --master-data=2 dbname > dbname_data.sql`
 or 
`/usr/local/mysql/bin/mysqldump -S /data/mysql/mysql3306/mysql3306.sock --single-transaction --master-data=2 dbname > dbname_"%Y%m%d-%H%M%S".sql -uroot -proot`
压缩：
`mysqldump --single-transaction --master-data=2 dbname | gzip > dbname_data.sql`
+ --single-transaction参数：这个参数做了一个一次性快照，如果其他链接做了“ ALTER TABLE, DROP TABLE, RENAME TABLE,，TRUNCATE TABLE”做了这些事情，是不受一次性快照保护的，所以在备份的时候不要做DDL操作，他建议用--lock-tables操作，所以用这个命令备份的时候不要用DDL操作，后续用metadata lock 解决了，就是当前访问的表，不能用DDL语句操作，没有访问的表可以用DDL
+ --master-data：看他的描述，实际上是在数据库中做了一个show master status做了一个数据库输入，如果是1，则显示将这个输出打印出来，如果是2，则给加个注释，前提是带着`--single-transaction`和`--lock-tables`参数

###mysqldump一直性备份原理，是怎么拿到数据库一致性的(不停业务)###
表结构：
`create table t1 (id int not null,name varchar(255),primary key(id));`
`create table t2 like t1;`
`insert into t2(id,name) values(1,'mysql');`

1. 打开general_log:`set global general_log=1;`，查看general_log是否打开`show global variables like "%gen%";`
2. 备份数据库:`/usr/local/mysql/bin/mysqldump -S /data/mysql/mysql3306/mysql3306.sock --single-transaction --master-data=2 wubx > wubx_3306_2306.sql -uroot -p`
3. 备份文件:wubx_3306_2306.sql
4. 进入到general_log文件里，查看开始备份的文件日志记录:`vi /data/mysql/mysql3306/data/localhost.log` 
```
2016-12-24T23:34:36.839845Z        34 Connect   root@localhost on  using Socket
2016-12-24T23:34:36.840791Z        34 Query     /*!40100 SET @@SQL_MODE='' */
2016-12-24T23:34:36.841958Z        34 Query     /*!40103 SET TIME_ZONE='+00:00' */
2016-12-24T23:34:36.842630Z        34 Query     FLUSH /*!40101 LOCAL */ TABLES
2016-12-24T23:34:36.867630Z        34 Query     FLUSH TABLES WITH READ LOCK
2016-12-24T23:34:36.868604Z        34 Query     SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ
2016-12-24T23:34:36.868774Z        34 Query     START TRANSACTION /*!40100 WITH CONSISTENT SNAPSHOT */
2016-12-24T23:34:36.868928Z        34 Query     SHOW VARIABLES LIKE 'gtid\_mode'
2016-12-24T23:34:36.901249Z        34 Query     SHOW MASTER STATUS
2016-12-24T23:34:36.901451Z        34 Query     UNLOCK TABLES
2016-12-24T23:34:36.901945Z        34 Query     SELECT LOGFILE_GROUP_NAME, FILE_NAME, TOTAL_EXTENTS, INITIAL_SIZE, ENGINE, EXTRA FROM INFORMATION_SCHEMA.FILES WHERE FILE_TYPE = 'UNDO LOG' AND FILE_NAME IS NOT NULL AND LOGFILE_GROUP_NAME IS NOT NULL AND LOGFILE_GROUP_NAME IN (SELECT DISTINCT LOGFILE_GROUP_NAME FROM INFORMATION_SCHEMA.FILES WHERE FILE_TYPE = 'DATAFILE' AND TABLESPACE_NAME IN (SELECT DISTINCT TABLESPACE_NAME FROM INFORMATION_SCHEMA.PARTITIONS WHERE TABLE_SCHEMA IN ('wubx'))) GROUP BY LOGFILE_GROUP_NAME, FILE_NAME, ENGINE, TOTAL_EXTENTS, INITIAL_SIZE ORDER BY LOGFILE_GROUP_NAME
2016-12-24T23:34:36.904563Z        34 Query     SELECT DISTINCT TABLESPACE_NAME, FILE_NAME, LOGFILE_GROUP_NAME, EXTENT_SIZE, INITIAL_SIZE, ENGINE FROM INFORMATION_SCHEMA.FILES WHERE FILE_TYPE = 'DATAFILE' AND TABLESPACE_NAME IN (SELECT DISTINCT TABLESPACE_NAME FROM INFORMATION_SCHEMA.PARTITIONS WHERE TABLE_SCHEMA IN ('wubx')) ORDER BY TABLESPACE_NAME, LOGFILE_GROUP_NAME
2016-12-24T23:34:36.905372Z        34 Query     SHOW VARIABLES LIKE 'ndbinfo\_version'
2016-12-24T23:34:36.907547Z        34 Init DB   wubx
2016-12-24T23:34:36.907718Z        34 Query     SAVEPOINT sp
2016-12-24T23:34:36.907955Z        34 Query     show tables
2016-12-24T23:34:36.908279Z        34 Query     show table status like 't1'
2016-12-24T23:34:36.909501Z        34 Query     SET SQL_QUOTE_SHOW_CREATE=1
2016-12-24T23:34:36.909698Z        34 Query     SET SESSION character_set_results = 'binary'
2016-12-24T23:34:36.909925Z        34 Query     show create table `t1`
2016-12-24T23:34:36.911365Z        34 Query     SET SESSION character_set_results = 'utf8'
2016-12-24T23:34:36.911569Z        34 Query     show fields from `t1`
2016-12-24T23:34:36.913380Z        34 Query     show fields from `t1`
2016-12-24T23:34:36.913821Z        34 Query     SELECT /*!40001 SQL_NO_CACHE */ * FROM `t1`
2016-12-24T23:34:36.915523Z        34 Query     SET SESSION character_set_results = 'binary'
2016-12-24T23:34:36.915652Z        34 Query     use `wubx`
2016-12-24T23:34:36.915795Z        34 Query     select @@collation_database
2016-12-24T23:34:36.915926Z        34 Query     SHOW TRIGGERS LIKE 't1'
2016-12-24T23:34:36.916304Z        34 Query     SET SESSION character_set_results = 'utf8'
2016-12-24T23:34:36.916425Z        34 Query     ROLLBACK TO SAVEPOINT sp
2016-12-24T23:34:36.916553Z        34 Query     show table status like 't2'
2016-12-24T23:34:36.916859Z        34 Query     SET SQL_QUOTE_SHOW_CREATE=1
2016-12-24T23:34:36.917022Z        34 Query     SET SESSION character_set_results = 'binary'
2016-12-24T23:34:36.917160Z        34 Query     show create table `t2`
2016-12-24T23:34:36.917302Z        34 Query     SET SESSION character_set_results = 'utf8'
2016-12-24T23:34:36.917442Z        34 Query     show fields from `t2`
2016-12-24T23:34:36.917797Z        34 Query     show fields from `t2`
2016-12-24T23:34:36.918132Z        34 Query     SELECT /*!40001 SQL_NO_CACHE */ * FROM `t2`
2016-12-24T23:34:36.918390Z        34 Query     SET SESSION character_set_results = 'binary'
2016-12-24T23:34:36.918475Z        34 Query     use `wubx`
2016-12-24T23:34:36.918540Z        34 Query     select @@collation_database
2016-12-24T23:34:36.918693Z        34 Query     SHOW TRIGGERS LIKE 't2'
2016-12-24T23:34:36.919044Z        34 Query     SET SESSION character_set_results = 'utf8'
2016-12-24T23:34:36.919163Z        34 Query     ROLLBACK TO SAVEPOINT sp
2016-12-24T23:34:36.919322Z        34 Query     RELEASE SAVEPOINT sp
2016-12-24T23:34:36.920240Z        34 Quit
#这一行就是备份的时候链接进来的开始
2016-12-24T23:34:43.213912Z        35 Connect   root@localhost on  using Socket
2016-12-24T23:34:43.217215Z        35 Query     /*!40100 SET @@SQL_MODE='' */
2016-12-24T23:34:43.218299Z        35 Query     /*!40103 SET TIME_ZONE='+00:00' */
#做了一个FLUSH TABLES 防止有DDL操作，如果这里有DDL操作，FLUSH TABLES 是进行不下去的
2016-12-24T23:34:43.218795Z        35 Query     FLUSH /*!40101 LOCAL */ TABLES
# 如果FLUSH TABLES没问题，就做一个FLUSH TABLES WITH READ LOCK，用了这个命令的话，就是数据库被整体锁定，是不能写入的，让数据库处于一个一致性快照的状态下
2016-12-24T23:34:43.219861Z        35 Query     FLUSH TABLES WITH READ LOCK
# 重点：这一句，把事物级别改成了RR(REPEATABLE READ)可重复读锁，利用RRZ在事务下获取一个一致性的读
2016-12-24T23:34:43.220148Z        35 Query     SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ
# 打开一个事物
2016-12-24T23:34:43.220346Z        35 Query     START TRANSACTION /*!40100 WITH CONSISTENT SNAPSHOT */
# 查看一次当前gtid有没有打开，如果打开了就获取出来，此案例没有打开
2016-12-24T23:34:43.220536Z        35 Query     SHOW VARIABLES LIKE 'gtid\_mode'
#做了一个SHOW MASTER STATUS
2016-12-24T23:34:43.227464Z        35 Query     SHOW MASTER STATUS
# 做完这些操作，解除锁定，基本锁了几毫秒
2016-12-24T23:34:43.228212Z        35 Query     UNLOCK TABLES
2016-12-24T23:34:43.228652Z        35 Query     SELECT LOGFILE_GROUP_NAME, FILE_NAME, TOTAL_EXTENTS, INITIAL_SIZE, ENGINE, EXTRA FROM INFORMATION_SCHEMA.FILES WHERE FILE_TYPE = 'UNDO LOG' AND FILE_NAME IS NOT NULL AND LOGFILE_GROUP_NAME IS NOT NULL AND LOGFILE_GROUP_NAME IN (SELECT DISTINCT LOGFILE_GROUP_NAME FROM INFORMATION_SCHEMA.FILES WHERE FILE_TYPE = 'DATAFILE' AND TABLESPACE_NAME IN (SELECT DISTINCT TABLESPACE_NAME FROM INFORMATION_SCHEMA.PARTITIONS WHERE TABLE_SCHEMA IN ('wubx'))) GROUP BY LOGFILE_GROUP_NAME, FILE_NAME, ENGINE, TOTAL_EXTENTS, INITIAL_SIZE ORDER BY LOGFILE_GROUP_NAME
2016-12-24T23:34:43.231705Z        35 Query     SELECT DISTINCT TABLESPACE_NAME, FILE_NAME, LOGFILE_GROUP_NAME, EXTENT_SIZE, INITIAL_SIZE, ENGINE FROM INFORMATION_SCHEMA.FILES WHERE FILE_TYPE = 'DATAFILE' AND TABLESPACE_NAME IN (SELECT DISTINCT TABLESPACE_NAME FROM INFORMATION_SCHEMA.PARTITIONS WHERE TABLE_SCHEMA IN ('wubx')) ORDER BY TABLESPACE_NAME, LOGFILE_GROUP_NAME
2016-12-24T23:34:43.233216Z        35 Query     SHOW VARIABLES LIKE 'ndbinfo\_version'
# 这里，进入到备份库里
2016-12-24T23:34:43.236155Z        35 Init DB   wubx
# 创建一个事物的起始点：sp
2016-12-24T23:34:43.236777Z        35 Query     SAVEPOINT sp
# 下面，检查表结构，检查数据
2016-12-24T23:34:43.238626Z        35 Query     show tables
2016-12-24T23:34:43.239748Z        35 Query     show table status like 't1'
2016-12-24T23:34:43.240318Z        35 Query     SET SQL_QUOTE_SHOW_CREATE=1
2016-12-24T23:34:43.240469Z        35 Query     SET SESSION character_set_results = 'binary'
2016-12-24T23:34:43.240610Z        35 Query     show create table `t1`
2016-12-24T23:34:43.241141Z        35 Query     SET SESSION character_set_results = 'utf8'
2016-12-24T23:34:43.241325Z        35 Query     show fields from `t1`
2016-12-24T23:34:43.241788Z        35 Query     show fields from `t1`
2016-12-24T23:34:43.242212Z        35 Query     SELECT /*!40001 SQL_NO_CACHE */ * FROM `t1`
2016-12-24T23:34:43.242528Z        35 Query     SET SESSION character_set_results = 'binary'
2016-12-24T23:34:43.242679Z        35 Query     use `wubx`
2016-12-24T23:34:43.242819Z        35 Query     select @@collation_database
2016-12-24T23:34:43.242997Z        35 Query     SHOW TRIGGERS LIKE 't1'
2016-12-24T23:34:43.243438Z        35 Query     SET SESSION character_set_results = 'utf8'
# 这里，备份完了第一个表，回到事物的开始点，
2016-12-24T23:34:43.243566Z        35 Query     ROLLBACK TO SAVEPOINT sp
2016-12-24T23:34:43.243740Z        35 Query     show table status like 't2'
2016-12-24T23:34:43.244003Z        35 Query     SET SQL_QUOTE_SHOW_CREATE=1
2016-12-24T23:34:43.244101Z        35 Query     SET SESSION character_set_results = 'binary'
2016-12-24T23:34:43.244189Z        35 Query     show create table `t2`
2016-12-24T23:34:43.244319Z        35 Query     SET SESSION character_set_results = 'utf8'
2016-12-24T23:34:43.244431Z        35 Query     show fields from `t2`
2016-12-24T23:34:43.244771Z        35 Query     show fields from `t2`
2016-12-24T23:34:43.245092Z        35 Query     SELECT /*!40001 SQL_NO_CACHE */ * FROM `t2`
2016-12-24T23:34:43.245304Z        35 Query     SET SESSION character_set_results = 'binary'
2016-12-24T23:34:43.245407Z        35 Query     use `wubx`
2016-12-24T23:34:43.245546Z        35 Query     select @@collation_database
2016-12-24T23:34:43.245699Z        35 Query     SHOW TRIGGERS LIKE 't2'
2016-12-24T23:34:43.246028Z        35 Query     SET SESSION character_set_results = 'utf8'
# 第二张表完了，也回到了事物的开始点
2016-12-24T23:34:43.246161Z        35 Query     ROLLBACK TO SAVEPOINT sp
# 所有表备份完了后，释放事物点
2016-12-24T23:34:43.246208Z        35 Query     RELEASE SAVEPOINT sp
2016-12-24T23:34:43.246991Z        35 Quit
```
总结：
1. 做了一个FLUSH TABLES 防止有DDL操作，如果这里有DDL操作，FLUSH TABLES 是进行不下去的
2. 如果FLUSH TABLES没问题，就做一个FLUSH TABLES WITH READ LOCK，用了这个命令的话，就是数据库被整体锁定，是不能写入的，让数据库处于一个一致性快照的状态下
3. 把事物级别改成了RR(REPEATABLE READ)可重复读锁，利用RR在事务下获取一个一致性的读
4. 打开一个事物
5. 查看一次当前gtid有没有打开，如果打开了就获取出来
6. 做了一个SHOW MASTER STATUS
7. 做完这些操作，解除锁定，基本锁了几毫秒，以上的操作中，数据库是不能访问的
8. 创建一个事物的起始点：sp
9. 开始一张一张备份表
10. 所有表备份完了后，释放事物点

上面备份中的坑：
+ 如果在做FLUSH TABLES的时候，刚好有个连接在做一个alter table操作，这个时候FLUSH TABLES就会断掉，FLUSH TABLES和DDL有冲突
+ 在备份中如果做flush logs的时候，就是在连接到mysql的时候，先进行一个flush logs，在进行备份，如果这个时候正在进行一个长事务，日志就不会切换，这个坑在innobackupex也有，所以在大的事务下带有flush logs的dump是不成功的
+ 在大的事务更新的情况下是拿不到metadata lock的

## MySql复制 ## 
![](http://i.imgur.com/2t3R19c.png)
![](http://i.imgur.com/Ab627Df.png)

案例：一主两从的结构，如果主挂了，怎么选取一个从来顶上来呢

## 复制结构 ##

### mysql proxy ###
+ java：MyCAT
+ golang：kingshard 
+ python:Oneproxy 

有了proxy后，所有的读和写都会经过proxy，如果是写的语句，就是用master，如果是读就用slave，如果有业务压力的话 一个机器的proxy也就代理三个DB

### Percona-xtraDB-Cluster(PXC) ###
PXC结构，每个节点都可以去写入数据，类似于oracle的rac，但是所有的写，都在一个节点上写

### 问题 ###
![](http://i.imgur.com/21NrWIk.png)

#### 业界的基本方案 ####
- 淘宝：TMA
- 腾讯游戏：spider 
- 财富通：半同步
- sohu : keepalived+lvs 主从
- sina: python实现一版mha 
- 京东和滴滴：jd, didi :ｍｈａ　
- 360：GTID

## tips ##
+ 可以用多核的压缩工具：pigz
+ mydumper：并行备份
+ mysqlpump必须和GTID一起用，不然拿不到一致性
+ www.frieflyclub.org上面有一个可重复读锁的解释
+ 在程序执行事物的时候，如果想把相关的事物保护起来，不要丢失，可以执行`savepiont s1`保存一个事物点，就可以`rollback s1`回到这个事物点
+ 什么是数据库一致性：在数据库里所有的操作都有一个编号，这个编号就是LSN(log seq number),假如在事务开始的时候对应的LSN是5(可以看下checkpoint)，那么我在做下面所有的操作都是5和5以前的数据，5以后的数据都看不到。，这些只针对事务表
+ 在8.0里有个基于事务的DDL，以前的DDL是不受事务保护的
+ 隔离级别推荐用RC
+ 国内现在用的最多的就是MHA，gtid出现后，MHA就没啥用了。就不需要MHA了


