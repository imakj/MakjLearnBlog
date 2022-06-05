##MySQL日志分类##
查询mysql所有日志类型`show global variables like "%log%"`
+ 错误日志.通过`--log-error=pathfile`指定,默认名称:host_name.err。此日志需要严重关注。
+ 慢查询日志.通过`--show_query_log=pathfile --long_query_timt=慢查询时间，#默认10秒,建议1-1.5秒`来指定。默认名称:host_name-slow.log。在mysql官方的分析程序有`mysqldumpslow`,第三方分许:`pt-query-digest`程序来输出。这个计时是这样的：当sql语句被解析完成后，进入到引擎层去执行的时候。就开始计时，如果时间超过long_query_timt的值，就将此sql输出出来。但是，在delete,update,insert执行中的锁等待的时间，是不计入long_query_timt时间的。只记录了执行的时间。在阿里的版本里改善了这个问题，锁等待的时间也记录了。
+ 常规日志.通过`--general_log=pathfile`指定，默认名称:host_name.log或者general.log
+ 二进制日志.通过`--log-bin=pathfile --expire-log-days`指定，默认名字:host_name-bin.000000。在mysql中用mysqlbinlog来解析、binlog2sql来解析。二进制日志是一个轮换事件，在服务器重启，执行刷新日志命令，达到设置的日志最大值的时候，会执行切换操作。但是mysql有个约定。一个事物不能跨binary log。就会造成一个binary log日志比设定的大小大。如果日志文件非常多的时候，数据库在重启的时候就会将所有的日志扫描一遍，然后在产生下一个日志。会造成启动的非常慢。**Tips:**可以通过编辑`mysql-bin.index`来制定文件索引。可以通过直接造一个mysql-bin.999999文件来测试，重新启动的时候就会产生一个mysql-bin.10000000。在以前的老版本里,日志会从mysql-bin.0000001重新开始。
+ 审计日志.`--audit_log --audit_log_fil=pathfile`来指定。文件名字:audit.log
+ DDL日志(5.7新增的日志)
+ Relay-log:中继日志。就是从库的io_thread从主库拉下来的binlog日志存放在本地的用来提供给sql_thread执行的一个日志。需要设置Relay-log大小。因为如果在主从复制中，如果sql_thread终止了，但是io_thread没有停止，这样很快就将磁盘给写满了
+ 当日志被移走的时候，可以用`fluse logs`来刷新一次日志。所有日志通用。

##MySQL二进制格式日志##
mysql二进制搁置有Statement和Row格式还有Mixed格式
+ Statement格式：是基于语句的格式。会记录原始SQL，日志文件：日志文件小。复制限制：并非所有的语句都可以复制。主/从 Mysql版本限制：从系统可以是采用不同行结构的的较新的版本。锁：在insert和select要求数量更多的行锁。时间点恢复：可以基于语句恢复
+ Row格式：基于行的语句格式，日志文件：日志文件大。复制限制：所有语句都可以复制。主/从 Mysql版本限制：从系统结构来讲必须是完全相同的版本和行结构。锁：insert、update、delete在系统上需要较少的锁。时间点恢复：由于日志时间是基于二进制格式。难度较高。
+ Mixed格式：Statement和Row格式的混合模式。是在Statement模式下，将一些特定的数据。例如UUID(),where id > 10，这类的sql执行。将会转换成Row格式进行记录。也就是在复制过程中数据不确定的时候，自动转换为Row格式。
+ mysql的二进制日志是以事件（event）为单位存储到日志中的，一个insert update由多个事件组成
+ 专业名词：日志文件：mysql-bin.000010,字节偏移量(位置)
****
**-------------------------------------------binlog命令-------------------------------------------------**
+ 查看解析binary log:`mysqlbinlog -v --base64 -output=decode-rows mysql-bin.000010`
+ 删除二进制日志，基于时间删除：`set global expire_log_days=7`保存7天的 或者 `purge binary logs before now() -interval 3 days` 删除3天前的，或者`purge binary logs before '2017-05-01'`删除5月1号之前的
+ 删除二进制日志，基于文件删除：`purge binary logs to 'mysql-bin.0000010'`
+ 日志删除完成后，刷新下日志、。

##MySQL审计日志##
官方收费组件，基于策略的日志记录，默认是ALL
+ 通过audit_log_policy设置
+ 提供机制记录选项ALL、NONE、LOGINS或QUERIES
+ 在日志文件中生成一个服务器活动审计记录
+ 内容取决于策略，可能包括
	+ 在系统上发生的错误记录
	+ 客户机连接和断开链接的时间
	+ 客户机在连接期间执行的操作
	+ 客户机访问的数据库和表
+ 审计日志格式是一个XML文件

##information_schema数据库##
+ 查看information_schema数据库:`select table_name from information_schema.tables where table_schema='information_schema'`
+ 一些基于"show"命令的操作，都是展示的这张表的信息
+ 查询当前库的信息:`select * from tables where table_schema='db'`.可以每天定时访问这张表，收集数据库信息
+ 查询当前库的表的信息:`select table_name,engine from tables where table_schema='db'`
+ `select character_set_name,collation_name from collations where is_default='Yes'`
+ `select table_schema,count(*) form tables group by tables_schema`


##总结##
+ 认识information_schema数据库
+ 学习利用information_schema的字典信息生成语句
+ information_schema相当于Mysql的中央信息库模式和模式对象、服务器的统计信息(状态变量，设置，连接)。该库不持久化，是一个"虚拟数据库"。可以通过SELECT访问。
+ general和binlog的区别：select、update连接到mysql输入的所有的sql语句都会记录到general log中。binlog只会记录对数据库相关改或者写的东西，也就是只会记录相关变化的日志。
+ 通过`desc table_name`查看一个表有哪些字段
+ 生成kill连接语句:`select concat("kill",ID,";") form information_schema.PROCESSLIST`。放到文件:`select concat("kill",ID,";") form PROCESSLIST info outfile '/tmp/kill.sql'`。要写入文件，需要配置写文件权限，在配置文件中的[mysqld]下面添加`secure-file-priv=path`例如`secure-file-priv=/tmp`只能在tmp下面写。也可以配置任意目录`secure-file-priv=""`,添加次权限必须重启。
+ 查看mysql所有配置文件:`show global variables like "%file%"`

error_log、general_log、slow_log等这些日志都是在磁盘顺序写入的。
