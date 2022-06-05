#GTID复制#
环境：两个库，一主一从
GTID：每个事务都有一个唯一的编号，用于事务提交的时候区分事务，用来区分事务是谁产生的，产生了多少事物
------
##概念，特征##
+ GTID全称：Global transaction identifiers
+ 一个事物对应一个唯一ID，事物id是由server-uuid产生` select @@server_uuid`
+ 一个GTID在一个服务器上只会执行一次
+ GTID是用来替代以前的calssic的复制方法
+ Mysql5.6.2以后支持，5.6.10后完善

###GIID组成###
+ server uuid:sequence number这个uuid在~/data/auto.cnf文件里面，由linux的uuidgen产生
+ 事务ID的组成uuid:N-N,第一个N是当前是哪个机器，第二个N代表事务ID，比如master的事物到了uuid:1-30，这个时候master做了一个mysqldump，然后恢复在了slave机器上，那么slave上的事物开始就是UUID：30-N

------
环境：主库4306 从库：4307

备份当前mysql库`/usr/local/mysql/bin/mysqldump -S /data/mysql/mysql4306/mysql4306.sock --single-transaction --master-data=2 -A  > 4306.sql -uroot -p`
恢复到4307数据库上:`/usr/local/mysql/bin/mysql -S /data/mysql/mysql4307/mysql4307.sock < 4306.sql `


1. 在4306上创建一个复制用的账号：
```
create user 'rpl'@'%';
grant replication slave on *.* to 'rpl'@'%' identified by 'repl4slave';
```
2. 做一个change master 语句将这个语句在4307上执行，然后`start slave；`
```
change master to master_host='192.168.200.131',master_port=4306,master_user='rpl',master_password='repl4slave',master_auto_position=1;
start slave；
```
这里可以用`help change master to;`查看语句帮助
3. 用`show slave status;`查看是否成功
```
*************************** 1. row ***************************
               Slave_IO_State: 
                  Master_Host: 192.168.200.131
                  Master_User: rpl
                  Master_Port: 4306
                Connect_Retry: 60
              Master_Log_File: 
          Read_Master_Log_Pos: 4
               Relay_Log_File: relay-bin.000001
                Relay_Log_Pos: 4
        Relay_Master_Log_File: 
             Slave_IO_Running: No
            Slave_SQL_Running: No
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 0
              Relay_Log_Space: 154
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 0
                  Master_UUID: 
             Master_Info_File: /data/mysql/mysql4307/data/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: 
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 15e0796e-ca94-11e6-bdb1-000c292d0f64:1-10
                Auto_Position: 1
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
1 row in set (0.00 sec)
```

对比，将两个的serveruiid先保存下来
```
master:
mysql> select @@server_uuid;
+--------------------------------------+
| @@server_uuid                        |
+--------------------------------------+
| 15e0796e-ca94-11e6-bdb1-000c292d0f64 |
+--------------------------------------+
1 row in set (0.01 sec)

slave:
mysql> select @@server_uuid;
+--------------------------------------+
| @@server_uuid                        |
+--------------------------------------+
| 29a23ca9-ca97-11e6-9f70-000c292d0f64 |
+--------------------------------------+
1 row in set (0.00 sec)
```

在slave上执行一个`show mstar status`，看看当前状态，然后在slave上创建一个表,在看看状态
```
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------------------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set                         |
+------------------+----------+--------------+------------------+-------------------------------------------+
| mysql-bin.000002 |      436 |              |                  | 15e0796e-ca94-11e6-bdb1-000c292d0f64:1-11 |
+------------------+----------+--------------+------------------+-------------------------------------------+
1 row in set (0.00 sec)

mysql> create table tb2(id int ,c1 varchar(255));
Query OK, 0 rows affected (0.02 sec)

mysql> show master status;
+------------------+----------+--------------+------------------+-----------------------------------------------------------------------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set                                                                 |
+------------------+----------+--------------+------------------+-----------------------------------------------------------------------------------+
| mysql-bin.000002 |      616 |              |                  | 15e0796e-ca94-11e6-bdb1-000c292d0f64:1-11,
29a23ca9-ca97-11e6-9f70-000c292d0f64:1 |
+------------------+----------+--------------+------------------+-----------------------------------------------------------------------------------+
1 row in set (0.01 sec)
```
可以看出，从库上自己的server-uuid发生了变化，明显的定位出从库有没有写入

## 总结 ##
### GTID的两种从库的添加方法：### 
1. 如果master所有的binlog还在，安装slave后，直接change master to 到master
	1. 原理是直接获取master所有的gtid并执行
	2. 优点是简单
	3. 缺点是如果binlog太多，数据完全同步需要的时间长，并且需要master一开始就启用了GTID
	4. 总结：适用于master新建不久的情况
2. 通过master或者其他slave的备份搭建新的slave
	1. 原理：获取master的数据，和这些数据对应的GTID范围，然后通过slave设置@@GLOBAL.GTID_PURGED从而跳过备份包含的GTID
	2. 有点事可以避免第一种方法的不足
	3. 缺点操作相对复杂
	4. 总结：适用于较大数据集的情况 
3. MySqldump
	1. 在备份的时候需要制定master-data
	2. 到处的语句中包含有：set @@GLOBAL.GTID_PURGED，恢复的时候在slave上执行一个reset master;
	3. 导入数据后做change master to 语句
4. Percona xtrabackup
	1. Xtrabackup_binlog_info包含GTID的信息
	2. 做从库恢复后，需要手工设置:set global gtid_purged='uuid:N'
	3. 导入数据后做change master to 语句

### GTID添加从库 ### 
1. change master to 
	+ master_host='masterip' ,
	+ master_port='masterport',
	+ master_user='user' ,
	+ master_password='password',
	+ master_auto_position=1 #自动查找同步到哪里了，也可以改为0，手动指定位置，需要添加下面两个参数
	+ master_log_file='binlog.xxxxxx', 
	+ master_log_pos=xxx

### GTID限制 ### 
+ 不支持非事务引擎(从库会报错 stop slave;start slave 忽略)
+ 不支持create table …… select语句复制(主库会直接报错)
+ 不允许一个sql同事更新一个事物引擎和非事务引擎的表
+ 在一个复制组中，不许要求统一开启GTID或者关闭GTID
+ 开启GTID需要重启
+ 开启GTID后，就不在使用原来传统的复制方式
+ 对于create temporary table 和drop temporary table的语句不支持
+ 不支持sql_slave_skip_counter

### GTID 复制中遇到的错误解决 ###
由于oracle中GTID在复制中GTID要求必须是连续性的（主库上的GTID，必须在从上也有的），所以不能skip过去一个事物，只能通过注入一个空的事物去跳过这个事物
1. stop slave
2. set gitd_next='xxxx:N';
3. begin;commit;
4. set gtid_next='AUTOMATIC';
5. start slave;
``` sql
#在从库上创建一个表
4306：mysql> create table tb2(id int ,c1 varchar(255));
#在主库上也创建这个表
4307：mysql> create table tb2(id int ,c1 varchar(255));
#在从库上查看从库状态，就会出现报错
Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.200.131
                  Master_User: rpl
                  Master_Port: 4306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000003
          Read_Master_Log_Pos: 835
               Relay_Log_File: relay-bin.000002
                Relay_Log_Pos: 828
        Relay_Master_Log_File: mysql-bin.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: No
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 1050
                   Last_Error: Error 'Table 'tb2' already exists' on query. Default database: 'wubx'. Query: 'create table tb2(id int ,c1 varchar(255))'
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 655
              Relay_Log_Space: 1209
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 1050
               Last_SQL_Error: Error 'Table 'tb2' already exists' on query. Default database: 'wubx'. Query: 'create table tb2(id int ,c1 varchar(255))'
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 24306
                  Master_UUID: 15e0796e-ca94-11e6-bdb1-000c292d0f64
             Master_Info_File: /data/mysql/mysql4307/data/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: 
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 161225 23:41:53
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 15e0796e-ca94-11e6-bdb1-000c292d0f64:11-13
            Executed_Gtid_Set: 15e0796e-ca94-11e6-bdb1-000c292d0f64:1-12,
29a23ca9-ca97-11e6-9f70-000c292d0f64:1
                Auto_Position: 1
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
1 row in set (0.00 sec)
```

处理方案：
1. 以主库为准，那么就在从库把这个表drop掉没，如果处理的话，不想让处理的数据记录到binlog里可以这样
	+ set sql_log_bin=0;
	+ drop table tb2;
	+ set sql_log_bin=1;
```
mysql> set sql_log_bin=0;
ERROR 2006 (HY000): MySQL server has gone away
No connection. Trying to reconnect...
Connection id:    15
Current database: wubx
\
Query OK, 0 rows affected (0.02 sec)
\
mysql> drop table tb2;
Query OK, 0 rows affected (0.00 sec)
\
mysql> set sql_log_bin=1;
Query OK, 0 rows affected (0.00 sec)
```
恢复从库，将Slave_SQL_Running启动起来
```
mysql> start slave sql_thread;
```
在看看slave状态，并且查看是否恢复成功
```
mysql> show slave status\G;
mysql> show tables;
```
以上操作是不记录到binlog日志的，如果将操作保存在binlog的时候，就会出现当这个从库是主库的时候，下一个从库就会将这些事物复制过去，造成下一个从把这个数据删掉，如果是主从切换的时候，就哭了

从库如果有super权限的时候，5.7建议改为super-read-only
继续.......

** 找不到数据错误：1032  **
这个数据会从update找不到记录或者delete找不到记录的时候
解决办法1：
停掉主从同步，在主库上基于主键伪造这么一条记录，然后启动
解决办法2：
跳过这个事物,
```
假设：主库上有一条记录id=3 从库上没有，在主库上执行delete from tb3 where id = 3;
从库的复制就会报1032错误
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.200.131
                  Master_User: rpl
                  Master_Port: 4306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000003
          Read_Master_Log_Pos: 1811
               Relay_Log_File: relay-bin.000003
                Relay_Log_Pos: 454
        Relay_Master_Log_File: mysql-bin.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: No
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 1032
                   Last_Error: Could not execute Delete_rows event on table wubx.t3; Can't find record in 't3', Error_code: 1032; handler error HA_ERR_END_OF_FILE; the event's master log mysql-bin.000003, end_log_pos 1780
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 1550
              Relay_Log_Space: 2485
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 1032
               Last_SQL_Error: Could not execute Delete_rows event on table wubx.t3; Can't find record in 't3', Error_code: 1032; handler error HA_ERR_END_OF_FILE; the event's master log mysql-bin.000003, end_log_pos 1780
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 24306
                  Master_UUID: 15e0796e-ca94-11e6-bdb1-000c292d0f64
             Master_Info_File: /data/mysql/mysql4307/data/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: 
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 161226 01:28:22
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 15e0796e-ca94-11e6-bdb1-000c292d0f64:11-17
            Executed_Gtid_Set: 15e0796e-ca94-11e6-bdb1-000c292d0f64:1-16,
29a23ca9-ca97-11e6-9f70-000c292d0f64:1-3
                Auto_Position: 1
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
1 row in set (0.00 sec)

这个时候，需要跳过这个事物，那么事物ID就是"Executed_Gtid_Set: 15e0796e-ca94-11e6-bdb1-000c292d0f64:1-16,29a23ca9-ca97-11e6-9f70-000c292d0f64:1-3"基于Executed_Gtid_Set事物ID的下一个事物id，也就是15e0796e-ca94-11e6-bdb1-000c292d0f64:1-16，Executed_Gtid_Set是代表当前执行的事物ID

mysql> stop slave;

mysql> set gtid_next="15e0796e-ca94-11e6-bdb1-000c292d0f64:17"; #事物+1
Query OK, 0 rows affected (0.00 sec)

#跳过一个空事物
mysql> begin;commit;
Query OK, 0 rows affected (0.00 sec)

#设置为自动
mysql> set gtid_next='automatic';
Query OK, 0 rows affected (0.00 sec)

#启动
mysql> start slave;
```

如果是update的1032错误的话，需要根据这个下面参数解析binlog，看看是哪条记录出错，需要相应的补回来
Last_Error: Could not execute Delete_rows event on table wubx.t3; Can't find record in 't3', Error_code: 1032; handler error HA_ERR_END_OF_FILE; the event's master log mysql-bin.000003, end_log_pos 1780
然后通过`mysqlbinlog`解析到相应的位置其中--start-position的参数为上面 Exec_Master_Log_Pos: 1550的值1550 --stop-position=1780为end_log_pos 1780的值
`mysqlbinlog -v --base64-output=decode-rows --start-position=1550 --stop-position=1780  mysql-bin.000003`

上面“Retrieved_Gtid_Set: 15e0796e-ca94-11e6-bdb1-000c292d0f64:11-17”中：11代表这个库是备份恢复过来的，备份的时候他的事务到了11，后面的17是获取到的主库当前的事物ID

** 主键冲突错误：1062  **
同上，选择跳过或者插入

**注意**
binlog中，这种语句
UPDATE `zhishu`.`wubx`
WHERE
@1=2
@2='wubx'
SET
@1=2
@2='python'
where后的是old的值，set后的是new的值

------

###传统复制和GTID的对比###

**注意：**一旦发生主从库的故障切换，一定要做数据一致性的校验

###在GTID的环境可以非常方便的Change任何一个存在者的身上，为什么###
因为GTID每个复制都是基于事务的，在做change的时候，看看自己和主库差多少事务，获取到binlog去执行就行了


问题：
group replication是什么
single-master
multi-master

再有GTID的时候，MHA已经不建议使用了