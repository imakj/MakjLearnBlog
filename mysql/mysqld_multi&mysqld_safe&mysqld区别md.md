+ mysqld_safe：mysqld_safe是mysqld的一个守护进程，或者说是一个管理脚本，当mysqld被干掉的时候，mysqld_safe会自动拉起mysqld.如果一个异常引起mysqld退出，mysqld_safe会再将mysqld拉起来，不好找问题
+ mysqld:mysql的直接启动进程
+ mysqld_multi：mysqld_multi是一个perl脚本。mysqld_multi可以调用mysqld_safe,也可以调用mysqld。通过调用mysqladmin实现多实例关机。
mysqld_multi使用需要在配置文件中声明：[mysqld_multi] & [mysqldN]，其中[mysqldN]这部分定义会覆盖[mysqld]这部分
mysqld_multi启动的实例N的参数是[mysqld]+[mysqldN]两部分的组合
如果用mysqld_multi启动，配置文件只能放到/etc/my.cnf下，或者用`--defaults-file`指定
mysqld_multi主要配置文件
```
[mysql_multi]
mysqld = /usr/local/mysql/bin/mysqld_safe #指定以什么方式启动
mysqldadmin= /usr/local/mysql/bin/mysqladmin #mysqladmin命令
user=mysql_multi #mysql启动用户
password=mysql_multi # mysql启动用户密码
[mysqld3306]
port=3306
datadir=/data/mysql/mysql3306/data #data目录
socket=/tmp/mysql3306.sock #socket文件
server-id=1003306 
login-bin=/data/mysql/mysql3306/logs/mysql-bin #binlog目录

```
mysqld_multi启动方式：`mysqld_multi{start|gtop|reload|report}[GNR[,GNR]...]` `/usr/local/mysql/bin/mysqld_multi start 3306`

+ mysqld_multi关闭的话会有个问题，需要在mysqld_multi的代码里添加一个-s。源代码:`my $com= join ' ', 'my_print_defaults -s', @defaults_options, $group; ` 修改后：`my $com= join ' ', 'my_print_defaults ', @defaults_options, $group; ` 