+ 通过`--pring-default`输出mysql的加载参数.例如:`/usr/local/mysql/bin/mysqld --pring-default`
+ mysql多实例启动的三种方式:
1.mysqld启动:`/usr/local/mysql/bin/mysqld --defaults-file=/path/my.cnf &`
2.mysqld_safe启动:`/usr/local/mysql/bin/mysqld_safe --defaults-file=/path/my.cnf &`
3.mysqld_multi启动:`/usr/local/mysql/bin/mysqld_multi start 3306`  
+ my_print_defaults打印当前配置文件参数:`/usr/local/mysql/bin/my_print_defaults mysqld`
+ mysql命令：
	1. \G：按照key value显示
	2. \c:终止当前命令 


****

+ 查询mysql所有日志类型`show global variables like "%log%"`
+ 列出当前的日志文件及大小，已字节为单位:`show binary logs`
+ 显示mysql当前的日志及状态[需要super、replication、client权限]：`show master status`
+ 查看binary log日志内容:`show binary log ecents in 'mysql-bin.000010'`
+ 查看解析binary log:`mysqlbinlog -v --base64 -output=decode-rows mysql-bin.000010`
+ 查看mysql所有配置文件:`show global variables like "%file%"`
+ 查看数据库:`show databases`
+ 查看表：`show tables`
+ 查看链接进程：`show processlist`
+ 查看建表DDL:`show create table <tb_name>`
+ 查看表索引：`show index from <tb_name>`
+ 查看当前数据库打开多少表:`show open tables`
+ 查看数据库的表的信息:`show table status`，这个语句会对数据库性能产生影响。
+ 查看表的列:`show columus from db.table`
+ 查看表的列:`show full columus from db.table`.加了full会展示出当前账号对列的权限。
+ 查看表的列:`desc table_name`
+ 查看字符集：`show characser set`
+ 查看支持的字符集：`show charset`
+ `show collation`
+ 查看mysql的引擎：`show engines;`

+ 查看所有show命令：`help show`
+ show命令支持like和where使用
+ 查看当前系统信息:`state` or `/s`