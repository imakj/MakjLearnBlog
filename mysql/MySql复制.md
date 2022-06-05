# MySql复制 #
mysql复制在业界又叫：MySql同步，AB复制等，专业名词叫：复制
+ 复制是单向的，只能从Master复制到Slave上
+ Slave上对于Master包含的数据不能进行写的操作
+ 一组复制结构中可以有多个slave，对于master一般场景推荐只有一个
+ msater用户写入数据，生成event记录到binary log中
+ slave接手master传来的binlog，然后按顺序应用，重现master上的用户操作

## 复制介绍 ##
###bin-log格式##
bin log格式在和binlog_format有关系，查看当前binlog格式`show global variables like "%binlog_format%";`
**重点：**生产环境里所有的格式都配置成Row格式，5.1后面的版本都支持ROW格式
+ SBR
+ RBR
+ MIXED

### 复制的实用价值 ###
+ 利用从裤做读能力的提升(重点)
+ 利用从裤做Master故障的接管(重点)
+ 利用从裤做备份较少对业务的影响
+ 利用复制升级
+ 利用专用slave做特殊SQL统计，达到不影响现在的业务

### 复制的配置 ###
+ 一个简单的主从配置
+ SBR复制配置
+ MIX复制配置
+ GTID复制配置
+ 传统复制和GTID的比较
+ 并行复制，多元复制。都是依赖于GTID

#### 简单的主从配置 ####
没有使用GTID的复制的情况下： 4.X-5.X
必备条件：
- Master:ip:port,具备server-id，必须启用log-bin(查看是否开启`show variables like "%log_bin%";`,其中log_bin=ON就是开启状态),主库上创建复制用户(`grant replication slave on *.* to 'repl'@'%' identified by 'repl'`)，禁用GTID
- slave：ip:port,具备server-id

条件：两个server-id不能一样，不然就会各自认为是复制端或者主库段

步骤：
1. 主库上创建复制用户(`grant replication slave on *.* to 'repl'@'%' identified by 'repl'`)
2. 在主库上做一个备份:mysqldump --single-transaction --master-data=2 -A &get;xxxx.sql
3. 查看备份的sql预计，找到master的logfile和pos
4. 修改CHANGE master to语句`mysql&get; change master to master_host='192.168.1.102',master_user='test',master_password='MkFx1FUtLx5mx3fg',master_log_file='binlog.000003’,master_log_pos=1191;`

### 案例 ###

----------
**statement日志格式**
前提条件：讲当前的日志格式改为默认的日志格式，就是注释掉配置文件里的`transaction_isolation = READ-COMMITTED`配置，确定当前事物级别是RR.
通过`select @@tx_isolation;`或者`show variables like '%tx_isolation%';`查看
```
mysql> show variables like '%iso%';
+---------------+-----------------+
| Variable_name | Value           |
+---------------+-----------------+
| tx_isolation  | REPEATABLE-READ |
+---------------+-----------------+
1 row in set (0.00 sec)

mysql> show variables like '%tx_isolation%';
+---------------+-----------------+
| Variable_name | Value           |
+---------------+-----------------+
| tx_isolation  | REPEATABLE-READ |
+---------------+-----------------+
1 row in set (0.00 sec)
```
1. 将当前回话改为statement格式  
`set session binlog_format='statement';`
改完后查看当前回话格式
```
mysql> show variables like "binlog_format";
+---------------+-----------+
| Variable_name | Value     |
+---------------+-----------+
| binlog_format | STATEMENT |
+---------------+-----------+
1 row in set (0.01 sec)
#加上global是查看全局
mysql> show global variables like "binlog_format";
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| binlog_format | ROW   |
+---------------+-------+
1 row in set (0.00 sec)
```
2. 查看当前binlog的日志位置
```
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000007 |      154 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```
3. 刷新日志
```
mysql> flush logs;
Query OK, 0 rows affected (0.01 sec)
```
4. 再次查看日志
```
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000008 |      154 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

此刻，日志文件以及变成了mysql-bin.000008

然后创建一张表，插入几条和依赖于系统函数的数据
```

mysql> create table rpl(id int,c1 varchar(200));
Query OK, 0 rows affected (0.01 sec)

mysql> insert into rpl(id,c1) values(1,'abc');
Query OK, 1 row affected (0.01 sec)

mysql> insert into rpl(id,c1) values(2,uuid());
Query OK, 1 row affected, 1 warning (0.00 sec)

mysql> insert into rpl(id,c1) values(3,now());
Query OK, 1 row affected (0.00 sec)

mysql> insert into rpl(id,c1) values(4,sysdate());
Query OK, 1 row affected, 1 warning (0.00 sec)

mysql> select * from rpl;
+------+--------------------------------------+
| id   | c1                                   |
+------+--------------------------------------+
|    1 | abc                                  |
|    2 | b51e5de3-ca55-11e6-9a1b-000c292d0f64 |
|    3 | 2016-12-25 11:53:40                  |
|    4 | 2016-12-25 11:53:46                  |
+------+--------------------------------------+
4 rows in set (0.00 sec)
```
再次写入一个uuid,查看它的警告信息
```
mysql> insert into rpl(id,c1) values(5,uuid());
Query OK, 1 row affected, 1 warning (0.00 sec)

mysql> show warnings;
+-------+------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Level | Code | Message                                                                                                                                                                                                  |
+-------+------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Note  | 1592 | Unsafe statement written to the binary log using statement format since BINLOG_FORMAT = STATEMENT. Statement is unsafe because it uses a system function that may return a different value on the slave. |
+-------+------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
上面解释就是如果使用了statement格式，并且使用了系统函数，会在slave上得到不通的值
```

看看我们刚才做的操作的那个binlog日志内容
`[root@localhost ~]# /usr/local/mysql/bin/mysqlbinlog /data/mysql/mysql3306/logs/mysql-bin.000008 > mysql-bin.000008.sql`

最后，删除设个表数据，删除设个表，并且刷新日志
```
[root@localhost ~]# more mysql-bin.000008.sql 
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 4
#161225 11:45:49 server id 23306  end_log_pos 123 CRC32 0xc06ea30d      Start: binlog v 4, server v 5.7.17-log cr
eated 161225 11:45:49
# Warning: this binlog is either in use or was not closed properly.
BINLOG '
7UBfWA8KWwAAdwAAAHsAAAABAAQANS43LjE3LWxvZwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAEzgNAAgAEgAEBAQEEgAAXwAEGggAAAAICAgCAAAACgoKKioAEjQA
AQ2jbsA=
'/*!*/;
# at 123
#161225 11:45:49 server id 23306  end_log_pos 154 CRC32 0x129da2bd      Previous-GTIDs
# [empty]
# at 154
#161225 11:49:59 server id 23306  end_log_pos 219 CRC32 0x0903388c      Anonymous_GTID  last_committed=0        s
equence_number=1
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 219
#161225 11:49:59 server id 23306  end_log_pos 333 CRC32 0xa1284164      Query   thread_id=3     exec_time=0     e
rror_code=0
use `wubx`/*!*/;
SET TIMESTAMP=1482637799/*!*/;
SET @@session.pseudo_thread_id=3/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit
=1/*!*/;
SET @@session.sql_mode=1436549152/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8 *//*!*/;
SET @@session.character_set_client=33,@@session.collation_connection=33,@@session.collation_server=33/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
create table rp1(id int,c1 varchar(200))
/*!*/;
# at 333
#161225 11:50:30 server id 23306  end_log_pos 398 CRC32 0xeb0c160e      Anonymous_GTID  last_committed=1        s
equence_number=2
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 398
#161225 11:50:30 server id 23306  end_log_pos 477 CRC32 0xad3091e9      Query   thread_id=3     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482637830/*!*/;
BEGIN
/*!*/;
# at 477
#161225 11:50:30 server id 23306  end_log_pos 589 CRC32 0x236c31ee      Query   thread_id=3     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482637830/*!*/;
insert into rp1(id,c1) values(1,'abc')
/*!*/;
# at 589
#161225 11:50:30 server id 23306  end_log_pos 620 CRC32 0x3b168d63      Xid = 24
COMMIT/*!*/;
# at 620
#161225 11:50:43 server id 23306  end_log_pos 685 CRC32 0xc857bd5b      Anonymous_GTID  last_committed=2        s
equence_number=3
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 685
#161225 11:50:43 server id 23306  end_log_pos 801 CRC32 0xbb666d37      Query   thread_id=3     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482637843/*!*/;
DROP TABLE `rp1` /* generated by server */
/*!*/;
# at 801
#161225 11:50:54 server id 23306  end_log_pos 866 CRC32 0x3fefd720      Anonymous_GTID  last_committed=3        s
equence_number=4
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 866
#161225 11:50:54 server id 23306  end_log_pos 980 CRC32 0x6b604cb4      Query   thread_id=3     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482637854/*!*/;
create table rpl(id int,c1 varchar(200))
/*!*/;
# at 980
#161225 11:50:59 server id 23306  end_log_pos 1045 CRC32 0xa0719c8f     Anonymous_GTID  last_committed=4        s
equence_number=5
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 1045
#161225 11:50:59 server id 23306  end_log_pos 1124 CRC32 0xd63a1eeb     Query   thread_id=3     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482637859/*!*/;
BEGIN
/*!*/;
# at 1124
#161225 11:50:59 server id 23306  end_log_pos 1236 CRC32 0xd59dbb96     Query   thread_id=3     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482637859/*!*/;
insert into rpl(id,c1) values(1,'abc')
/*!*/;
# at 1236
#161225 11:50:59 server id 23306  end_log_pos 1267 CRC32 0x150d148b     Xid = 27
COMMIT/*!*/;
# at 1267
#161225 11:51:06 server id 23306  end_log_pos 1332 CRC32 0x2ae24785     Anonymous_GTID  last_committed=5        s
equence_number=6
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 1332
#161225 11:51:06 server id 23306  end_log_pos 1411 CRC32 0x94c0e3e8     Query   thread_id=3     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482637866/*!*/;
BEGIN
/*!*/;
# at 1411
#161225 11:51:06 server id 23306  end_log_pos 1524 CRC32 0xeefccb42     Query   thread_id=3     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482637866/*!*/;
insert into rpl(id,c1) values(1,uuid())
/*!*/;
# at 1524
#161225 11:51:06 server id 23306  end_log_pos 1555 CRC32 0xdc3405d9     Xid = 28
COMMIT/*!*/;
# at 1555
#161225 11:51:11 server id 23306  end_log_pos 1620 CRC32 0x92f91009     Anonymous_GTID  last_committed=6        s
equence_number=7
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 1620
#161225 11:51:11 server id 23306  end_log_pos 1707 CRC32 0x1abdd798     Query   thread_id=3     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482637871/*!*/;
SET @@session.time_zone='SYSTEM'/*!*/;
BEGIN
/*!*/;
# at 1707
#161225 11:51:11 server id 23306  end_log_pos 1827 CRC32 0x6ba92f28     Query   thread_id=3     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482637871/*!*/;
insert into rpl(id,c1) values(1,now())
/*!*/;
# at 1827
#161225 11:51:11 server id 23306  end_log_pos 1858 CRC32 0x78c1862d     Xid = 29
COMMIT/*!*/;
# at 1858
#161225 11:51:18 server id 23306  end_log_pos 1923 CRC32 0x70f2386d     Anonymous_GTID  last_committed=7        s
equence_number=8
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 1923
#161225 11:51:18 server id 23306  end_log_pos 2010 CRC32 0x90cf49a6     Query   thread_id=3     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482637878/*!*/;
BEGIN
/*!*/;
# at 2010
#161225 11:51:18 server id 23306  end_log_pos 2134 CRC32 0x3981a79c     Query   thread_id=3     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482637878/*!*/;
insert into rpl(id,c1) values(1,sysdate())
/*!*/;
# at 2134
#161225 11:51:18 server id 23306  end_log_pos 2165 CRC32 0xed5fdc11     Xid = 30
COMMIT/*!*/;
# at 2165
#161225 11:53:00 server id 23306  end_log_pos 2230 CRC32 0xe5660ab9     Anonymous_GTID  last_committed=8        s
equence_number=9
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 2230
#161225 11:53:00 server id 23306  end_log_pos 2346 CRC32 0x7afbfaf2     Query   thread_id=3     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482637980/*!*/;
DROP TABLE `rpl` /* generated by server */
/*!*/;
# at 2346
#161225 11:53:11 server id 23306  end_log_pos 2411 CRC32 0x5f95875c     Anonymous_GTID  last_committed=9        s
equence_number=10
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 2411
#161225 11:53:11 server id 23306  end_log_pos 2525 CRC32 0x6fa1c1ad     Query   thread_id=3     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482637991/*!*/;
create table rpl(id int,c1 varchar(200))
/*!*/;
# at 2525
#161225 11:53:17 server id 23306  end_log_pos 2590 CRC32 0xb39fceb8     Anonymous_GTID  last_committed=10       s
equence_number=11
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 2590
#161225 11:53:17 server id 23306  end_log_pos 2669 CRC32 0x2a87ee89     Query   thread_id=3     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482637997/*!*/;
BEGIN
/*!*/;
# at 2669
#161225 11:53:17 server id 23306  end_log_pos 2781 CRC32 0x7d12c7dc     Query   thread_id=3     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482637997/*!*/;
insert into rpl(id,c1) values(1,'abc')
/*!*/;
# at 2781
#161225 11:53:17 server id 23306  end_log_pos 2812 CRC32 0x0fba3f23     Xid = 35
COMMIT/*!*/;
# at 2812
#161225 11:53:34 server id 23306  end_log_pos 2877 CRC32 0xd427f700     Anonymous_GTID  last_committed=11       s
equence_number=12
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 2877
#161225 11:53:34 server id 23306  end_log_pos 2956 CRC32 0x032cbf26     Query   thread_id=3     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482638014/*!*/;
BEGIN
/*!*/;
# at 2956
#161225 11:53:34 server id 23306  end_log_pos 3069 CRC32 0x6cd38f56     Query   thread_id=3     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482638014/*!*/;
insert into rpl(id,c1) values(2,uuid())
/*!*/;
# at 3069
#161225 11:53:34 server id 23306  end_log_pos 3100 CRC32 0x47d40fba     Xid = 36
COMMIT/*!*/;
# at 3100
#161225 11:53:40 server id 23306  end_log_pos 3165 CRC32 0x95933f16     Anonymous_GTID  last_committed=12       s
equence_number=13
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 3165
#161225 11:53:40 server id 23306  end_log_pos 3252 CRC32 0x2b852949     Query   thread_id=3     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482638020/*!*/;
BEGIN
/*!*/;
# at 3252
#161225 11:53:40 server id 23306  end_log_pos 3372 CRC32 0xc3accb70     Query   thread_id=3     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482638020/*!*/;
insert into rpl(id,c1) values(3,now())
/*!*/;
# at 3372
#161225 11:53:40 server id 23306  end_log_pos 3403 CRC32 0x4b8a011e     Xid = 37
COMMIT/*!*/;
# at 3403
#161225 11:53:46 server id 23306  end_log_pos 3468 CRC32 0xb69047fd     Anonymous_GTID  last_committed=13       s
equence_number=14
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 3468
#161225 11:53:46 server id 23306  end_log_pos 3555 CRC32 0x2e486df9     Query   thread_id=3     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482638026/*!*/;
BEGIN
/*!*/;
# at 3555
#161225 11:53:46 server id 23306  end_log_pos 3679 CRC32 0xca9eaebe     Query   thread_id=3     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482638026/*!*/;
insert into rpl(id,c1) values(4,sysdate())
/*!*/;
# at 3679
#161225 11:53:46 server id 23306  end_log_pos 3710 CRC32 0x2961cba7     Xid = 38
COMMIT/*!*/;
# at 3710
#161225 11:54:43 server id 23306  end_log_pos 3775 CRC32 0xd6395b22     Anonymous_GTID  last_committed=14       s
equence_number=15
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 3775
#161225 11:54:43 server id 23306  end_log_pos 3854 CRC32 0x929bdc3e     Query   thread_id=3     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482638083/*!*/;
BEGIN
/*!*/;
# at 3854
#161225 11:54:43 server id 23306  end_log_pos 3967 CRC32 0xbb450414     Query   thread_id=3     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482638083/*!*/;
insert into rpl(id,c1) values(5,uuid())
/*!*/;
# at 3967
#161225 11:54:43 server id 23306  end_log_pos 3998 CRC32 0x9955f0f9     Xid = 40
COMMIT/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```
看到上面的bonlog日志，将系统函数写到了日志里，当从库执行这些日志的时候，就会得到不一样的值；但是`now()`函数例外，因为在做`now()`函数的时候，在前面加了一个`SET TIMESTAMP=1482638083/*!*`，如上面的
```
SET TIMESTAMP=1482638020/*!*/;
insert into rpl(id,c1) values(3,now())
```
所以，基于binlog=statement的日志格式下
**优点：**
+ binlog文件较小
+ 日志是包含用户执行的原始SQL，方便统计和审计
+ 和早起的binlog的兼容好
+ binlog方便阅读，故障修复

**缺点：**
+ 存在安全隐患，导致主从不一致
+ 对一些系统函数不能准确复制或者不能复制，如上面的load_file(),uuid(),user(),sysdate()
+ 执行类似于这种SQL的时候`delete from tb where c1=xx limit 1`因为存储引擎的排序是不一样的，导致数据删除错误。
+ 类似这种SQL的话`delete from tb where id <= 10`,如果主从的数据不一致的话，删除的数据也就不一致了

----------
**ROW格式**
将当前会话的binlog格式改为row，然后干上面的事情
```
mysql> show variables like "binlog_format";
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| binlog_format | ROW   |
+---------------+-------+
1 row in set (0.03 sec)

mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000011 |      154 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

mysql> flush logs;
Query OK, 0 rows affected (0.00 sec)

mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000012 |      154 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.01 sec)

mysql> create table rpl(id int,c1 varchar(200));
Query OK, 0 rows affected (0.01 sec)

mysql> insert into rpl(id,c1) values(1,'abc');
Query OK, 1 row affected (0.00 sec)

mysql> insert into rpl(id,c1) values(2,uuid());
Query OK, 1 row affected (0.00 sec)

mysql> insert into rpl(id,c1) values(3,now());
Query OK, 1 row affected (0.00 sec)

mysql> insert into rpl(id,c1) values(4,sysdate());
Query OK, 1 row affected (0.00 sec)

mysql> insert into rpl(id,c1) values(10,uuid());
Query OK, 1 row affected (0.00 sec)

mysql> select * from rpl;
+------+--------------------------------------+
| id   | c1                                   |
+------+--------------------------------------+
|    1 | abc                                  |
|    2 | 8f0f67a7-ca5b-11e6-9a1b-000c292d0f64 |
|    3 | 2016-12-25 12:35:31                  |
|    4 | 2016-12-25 12:35:37                  |
|   10 | 97fa717c-ca5b-11e6-9a1b-000c292d0f64 |
+------+--------------------------------------+
5 rows in set (0.00 sec)

mysql> delete from rpl where id <= 10;
Query OK, 5 rows affected (0.00 sec)

mysql> drop table rpl;
Query OK, 0 rows affected (0.01 sec)

mysql> flush logs;
Query OK, 0 rows affected (0.01 sec)
```

用`/usr/local/mysql/bin/mysqlbinlog --base64-output=DECODE-ROWS -v /data/mysql/mysql3306/logs/mysql-bin.000012 > mysql-bin.000012';`看看执行结果,,一定要加上`--base64-output=DECODE-ROWS -v`查看

```
[root@localhost ~]# more mysql-bin.000012 
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 4
#161225 12:34:59 server id 23306  end_log_pos 123 CRC32 0x54ae2000      Start: binlog v 4, server v 5.7.17-log cr
eated 161225 12:34:59
# at 123
#161225 12:34:59 server id 23306  end_log_pos 154 CRC32 0xd3b969ac      Previous-GTIDs
# [empty]
# at 154
#161225 12:35:14 server id 23306  end_log_pos 219 CRC32 0xff5c26b8      Anonymous_GTID  last_committed=0        s
equence_number=1
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 219
#161225 12:35:14 server id 23306  end_log_pos 333 CRC32 0xe4ef2931      Query   thread_id=6     exec_time=0     e
rror_code=0
use `wubx`/*!*/;
SET TIMESTAMP=1482640514/*!*/;
SET @@session.pseudo_thread_id=6/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit
=1/*!*/;
SET @@session.sql_mode=1436549152/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8 *//*!*/;
SET @@session.character_set_client=33,@@session.collation_connection=33,@@session.collation_server=33/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
create table rpl(id int,c1 varchar(200))
/*!*/;
# at 333
#161225 12:35:19 server id 23306  end_log_pos 398 CRC32 0x2d43598a      Anonymous_GTID  last_committed=1        s
equence_number=2
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 398
#161225 12:35:19 server id 23306  end_log_pos 470 CRC32 0x58827fad      Query   thread_id=6     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482640519/*!*/;
BEGIN
/*!*/;
# at 470
#161225 12:35:19 server id 23306  end_log_pos 519 CRC32 0xa16a1eeb      Table_map: `wubx`.`rpl` mapped to number 
226
# at 519
#161225 12:35:19 server id 23306  end_log_pos 564 CRC32 0x45fd2cea      Write_rows: table id 226 flags: STMT_END_
F
### INSERT INTO `wubx`.`rpl`
### SET
###   @1=1
###   @2='abc'
# at 564
#161225 12:35:19 server id 23306  end_log_pos 595 CRC32 0xe4441823      Xid = 82
COMMIT/*!*/;
# at 595
#161225 12:35:27 server id 23306  end_log_pos 660 CRC32 0x0b43ed68      Anonymous_GTID  last_committed=2        s
equence_number=3
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 660
#161225 12:35:27 server id 23306  end_log_pos 732 CRC32 0x4a09b004      Query   thread_id=6     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482640527/*!*/;
BEGIN
/*!*/;
# at 732
#161225 12:35:27 server id 23306  end_log_pos 781 CRC32 0x68ba9edc      Table_map: `wubx`.`rpl` mapped to number 
226
# at 781
#161225 12:35:27 server id 23306  end_log_pos 859 CRC32 0x4fed818b      Write_rows: table id 226 flags: STMT_END_
F
### INSERT INTO `wubx`.`rpl`
### SET
###   @1=2
###   @2='8f0f67a7-ca5b-11e6-9a1b-000c292d0f64'
# at 859
#161225 12:35:27 server id 23306  end_log_pos 890 CRC32 0x54c8902a      Xid = 83
COMMIT/*!*/;
# at 890
#161225 12:35:31 server id 23306  end_log_pos 955 CRC32 0x5e598787      Anonymous_GTID  last_committed=3        s
equence_number=4
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 955
#161225 12:35:31 server id 23306  end_log_pos 1035 CRC32 0x5bf882e0     Query   thread_id=6     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482640531/*!*/;
SET @@session.time_zone='SYSTEM'/*!*/;
BEGIN
/*!*/;
# at 1035
#161225 12:35:31 server id 23306  end_log_pos 1084 CRC32 0x50ddeb60     Table_map: `wubx`.`rpl` mapped to number 
226
# at 1084
#161225 12:35:31 server id 23306  end_log_pos 1145 CRC32 0x714f8f7b     Write_rows: table id 226 flags: STMT_END_
F
### INSERT INTO `wubx`.`rpl`
### SET
###   @1=3
###   @2='2016-12-25 12:35:31'
# at 1145
#161225 12:35:31 server id 23306  end_log_pos 1176 CRC32 0x30092d97     Xid = 84
COMMIT/*!*/;
# at 1176
#161225 12:35:37 server id 23306  end_log_pos 1241 CRC32 0xc51eee67     Anonymous_GTID  last_committed=4        s
equence_number=5
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 1241
#161225 12:35:37 server id 23306  end_log_pos 1321 CRC32 0x3481336d     Query   thread_id=6     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482640537/*!*/;
BEGIN
/*!*/;
# at 1321
#161225 12:35:37 server id 23306  end_log_pos 1370 CRC32 0xd8197586     Table_map: `wubx`.`rpl` mapped to number 
226
# at 1370
#161225 12:35:37 server id 23306  end_log_pos 1431 CRC32 0x926d1b3c     Write_rows: table id 226 flags: STMT_END_
F
### INSERT INTO `wubx`.`rpl`
### SET
###   @1=4
###   @2='2016-12-25 12:35:37'
# at 1431
#161225 12:35:37 server id 23306  end_log_pos 1462 CRC32 0x385039dc     Xid = 85
COMMIT/*!*/;
# at 1462
#161225 12:35:42 server id 23306  end_log_pos 1527 CRC32 0x46301e61     Anonymous_GTID  last_committed=5        s
equence_number=6
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 1527
#161225 12:35:42 server id 23306  end_log_pos 1599 CRC32 0xdc998cbb     Query   thread_id=6     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482640542/*!*/;
BEGIN
/*!*/;
# at 1599
#161225 12:35:42 server id 23306  end_log_pos 1648 CRC32 0xf6527d00     Table_map: `wubx`.`rpl` mapped to number 
226
# at 1648
#161225 12:35:42 server id 23306  end_log_pos 1726 CRC32 0x18289fbc     Write_rows: table id 226 flags: STMT_END_
F
### INSERT INTO `wubx`.`rpl`
### SET
###   @1=10
###   @2='97fa717c-ca5b-11e6-9a1b-000c292d0f64'
# at 1726
#161225 12:35:42 server id 23306  end_log_pos 1757 CRC32 0x1a25a0b3     Xid = 86
COMMIT/*!*/;
# at 1757
#161225 12:36:01 server id 23306  end_log_pos 1822 CRC32 0x5f5ba4ca     Anonymous_GTID  last_committed=6        s
equence_number=7
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 1822
#161225 12:36:01 server id 23306  end_log_pos 1894 CRC32 0xfa4de5e4     Query   thread_id=6     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482640561/*!*/;
BEGIN
/*!*/;
# at 1894
#161225 12:36:01 server id 23306  end_log_pos 1943 CRC32 0x56435427     Table_map: `wubx`.`rpl` mapped to number 
226
# at 1943
#161225 12:36:01 server id 23306  end_log_pos 2126 CRC32 0x9017705e     Delete_rows: table id 226 flags: STMT_END
_F
# 删除语句，将旧的数据值作为条件执行的，但是在执行删除语句的时候，会单独删除
### DELETE FROM `wubx`.`rpl`
### WHERE
###   @1=1
###   @2='abc'
### DELETE FROM `wubx`.`rpl`
### WHERE
###   @1=2
###   @2='8f0f67a7-ca5b-11e6-9a1b-000c292d0f64'
### DELETE FROM `wubx`.`rpl`
### WHERE
###   @1=3
###   @2='2016-12-25 12:35:31'
### DELETE FROM `wubx`.`rpl`
### WHERE
###   @1=4
###   @2='2016-12-25 12:35:37'
### DELETE FROM `wubx`.`rpl`
### WHERE
###   @1=10
###   @2='97fa717c-ca5b-11e6-9a1b-000c292d0f64'
# at 2126
#161225 12:36:01 server id 23306  end_log_pos 2157 CRC32 0x7152d3c1     Xid = 88
COMMIT/*!*/;
# at 2157
#161225 12:36:09 server id 23306  end_log_pos 2222 CRC32 0x27ae6609     Anonymous_GTID  last_committed=7        s
equence_number=8
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 2222
#161225 12:36:09 server id 23306  end_log_pos 2338 CRC32 0xa18f0e2c     Query   thread_id=6     exec_time=0     e
rror_code=0
SET TIMESTAMP=1482640569/*!*/;
DROP TABLE `rpl` /* generated by server */
/*!*/;
# at 2338
#161225 12:36:12 server id 23306  end_log_pos 2385 CRC32 0x56e803a5     Rotate to mysql-bin.000013  pos: 4
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```

从上面可以看出，在binlog里，不管是系统函数还是删除或者更改的语句，都是讲值带入到SQL里执行的，但是在执行删除语句的时候，会单独删除，这样的话，万一不小心更新了错误的数据，可以从日志里找回主键并且声称update脚本。
在ROW格式下DDL语句也是明文的

**优点：**
+ 相比STATEMENT更加安全的复制格式
+ 在有些情况下复制速度更快，因为是基于主键复制的
+ 系统的特殊函数也可以复制
+ 因为有主键，所以不容易产生锁等待
+ mysql5.6会记录old_row和new_row，并且只记录产生变更的数据


**缺点：**
+ boinary log比较大(mysql5.6支持binlog_row_image)
+ 他会记录每条语句，所以单语句更新和删除的表的行数多，会形成大量的binlog，索引row格式下不要更新太多的行
+ 无法从binlog看见用户执行的SQL(mysql5.6增加了了一个新的event binlog_rows_query_log_events记录用户的SQL，这个参数默认是关闭的，需要打开,如果低版本做高版本的从库，需要将`log_bin_use_v1_row_events`这个参数也打开，不然5.5就链接不上来)

---------

### MIXED格式 ###
**前提，事物隔离级别是RR**
+ 混合使用ROW和STATEMENT格式，对于DDL记录会以STATEMENT格式记录，对于TABLES里的行操作记录会以ROW格式记录
+ 如果使用innodb表，事物级别使用READ COMMITTEN或者READ UNCOMMITTED日志级别的话，日志只能使用ROW格式记录
+ 但是在ROW格式中DDL语句还是会记录STATEMENT格式
+ MIXED模式在一下集中情况会将binlog的模式由SBR改为RBR
	+ 当DML语句更新一个NDB表的时候
	+ 的那个函数中包含UUID()的时候
	+ 2个及以上包含AUTO_INCREMENT字段的表被更新时
	+ 行任何INSERT DELAYED语句的时候
	+ 用UDF的时候
	+ 视图中必须要求使用RBR是，例如创建视图使用UUID()函数

---------

###复制的流程###
![](http://i.imgur.com/QpyaZXw.png)

###案例###
**场景1：**当做大批量的更新或者删除的时候，不要一次处理太多，不会会将日志卡死，或者将session的格式改成statement `set session binlog_format=statement`

**场景2：**
对一个字段去做crc32(c1)校验，如果主从数据不一致的话，返回的值也不一样,pt-table-checksum工具就是这个原理，对主从库的值获取一个hash值来对比，但是，他在校验text类型的字段的时候，他只获取前几位的值来校验，如果前几位一样的话，就会出现text值不一致他也校验一样的


###为什么要给表里加主键###
1. 方便用row格式下复制
2. 方便顺序写入顺序度

##根据日志格式复制##
+ 根据日志定义的格式不一样，可以分为：statement(SBR)格式，ROW()格式或者MINED格式
+ 记录最小单位是一个Event，日志钱4个字节是一个magic number，接下来19个字节记录的是Format desc event:FDE
+ 一个事物由多个Event组成
+ binlog相关包含：binary log和binary log index文件

## binary log 里的position ##
binary log 里的position是文件内部的字节偏移量

## tips ##
+ server-id命名：ip最后一位+端口，serverid的最大值是int 2的32次方-1
+ 在mysql中，也可以用`show binlog events in 'mysql-bin.000008';`这个语句解析binlog
+ 用echo命令讲查询的结果输出出来`echo "select id,c1 from rpl where id < 100 " | mysql -p wubx > /tmp/100.sql`,然后将输出来的语句拼成sql`sed '1d' /tmp/100.sql |awk '{print "update rpl set c1="$2," where id="$1,";"}' `或者`awk '{print "update rpl set c1=\"$2\", where id="$1,";"}'`
+ 生产环境建议大家选用了 row
+ RC的事物隔离级别下的MIXED日志格式就是ROW格式
+ 学会percona-tools工具集
+ mysql论坛：http://fireflyclub.org/
+ binlog文件的最后一个位置是一个指向，指向下一个文件的文件名和开始position位置


问题：
	Xid是个啥，查看Xid：`/usr/local/mysql/bin/mysqlbinlog /data/mysql/mysql3306/logs/mysql-bin.* | grep "Xid"`