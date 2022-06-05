##mysql用户##
+ mysql查看账户:
	 5.7以前：`select user,host,password from mysql.user`
	 5.7及以后：`select user,host,authentication_string from mysql.user`
+ mysql5.7或者5.6创建用户，如果不指定用户，那么默认就映射到了root用户下.例：`create uuser ''@localhost`
+ mysql创建用户:`create user 'username'@'localhost' identified by 'password';`
+ mysql创建IP访问段用户:`create user 'username'@'192.168.11.{1,9}' identified by 'password';`,这里只能改ip段的1-9可以登录。不能用下划线"_"，因为mysql中下划线"_"代表了一个字符
+ mysql创建IP访问段用户:`create user 'username'@'192.168.11.%' identified by 'password';`,允许所有11网段登录
+ mysql创建IP访问段用户:`create user 'username'@'192.168.11.1_' identified by 'password';`,允许所有11网段的10-19登录，如果是两个下划线"_"就是10-99
+ mysql创建限制域名用户：`create user 'username'@'db%.zst.com' identified by 'password';` ,允许db%.zst.com用户登录
+ mysql创建限制子网掩码用户：`create user 'username'@'192.168.11.0/255.255.255.0' identified by 'password';`,允许IP段是192.168.11.0子网掩码是255.255.255.0用户登录
+ 删除用户:`drop user 'wubx'@'%'` 如果drop用户后背不带主机名的话，默认删除的就是'%'用户
+ mysql修改用户名:`rename usesr 'wubx'@'%' to 'zst'@'%'；`
+ 查看当前账户的权限：`show grants for 'wubx'@'192.168.11.%';`
+ 每次权限更新完，需要`flush privileges;`

##mysql权限##
+ 只读用户：只用于select:`grant select on zstdb.* to 'r_zst'@'192.168.11.%';`;5.6之前如果这个账号不存在，直接创建这个账户，5.7会日志找不到这个账号
+ 只读用户：只用于select:`grant all privileges select on zstdb.* to 'r_zst'@'192.168.11.%' identified by;`;这样带密码创建是可以创建成功的，但是会被提示说是一个过时的方法。有可能被遗弃掉
+ 查看mysql所有权限：`show privileges` 
+ 查看用户当前权限:`show grants`
+ 查看用户当前权限:`show grants for current_user()`
+ `user()`和`current_user()`区别，`user()`：显示用户名链接的具体的IP，`current_user()`:显示创建的用户的具体权限，例如：`192.168.11.%'`
+ 查看指定用户权限: `show grants for 'wubx'@'192.168.11.%';`
+ 撤销用户授权:`revoke delete,insert,update on db,* from 'zst'@'%'；`
+ 撤销用户全部授权:`revoke all privileges,grant option from 'zst'@'%'；`,`grant option`:赋予账号赋权权限
+ revoke语法： `revoke ... on ... from`
	+ revoke:关键字
	+ on子句：指定要撤销特权的级别(全局级时可以不用带)
	+ from子句：指定账户名
+ usage权限：无权限。创建的用户只能登陆。

##禁用验证控制##
下列所有参数都是在[mysqld]下面添加的
+ `--skip-grant-table`:无需用户名和密码登录，登录后禁止使用create user,grant,revode,set password。建议配合：禁止使用网络验证一起使用，如果这个时候密码忘了，可以使用`update mysql.user set password=xxx where user='xxx'`表来进行改密码。
+ `--skip-networking`:禁止使用网络验证。用于在升级的过程中，拒绝所有的网络连接
+ `--safe-update`：
	1. 阻止mysql客户端使用没有where的条件语句
	2. update和delete仅在包含where子句，并且该子句通过键值明确的标识了更新或者删除的记录，或者limit子句时才允许使用
	3. 将单标select语句中的输出限制为不超过1000行，单语句包含limit子句的时候除外
	4. 仅当mysql为处理查询所检查的行不超过1000000时，才允许使用多表select语句

##mysql客户端工具##
mysql客户端工具都在`~/mysql/bin/`下面
+ mysql:将SQL语句发送到服务器
+ mysqladmin:用于管理Mysql服务器，在shell层次交互
+ mysqldump:备份数据库
+ mysqlbinlog:解析mysql的binary log及重放binary log
+ mysqllimport:将文件加载到数据库(有点load data的感觉)
+ mysqlsalp:mysql官方自带的一个简单的压力测试工具
+ mysqlsh:待研究,网址：https://dev.mysql.com/downloads/shell/
+ mysql客户端调用中"--"和"-"分表表示："--"：表示长选项，例如-h，-P，"-"表示短选项：例如-hostname,-port
+ mysql客户端的--login-path 参数研究
+ mysql执行sql语句:`mysql -S /tmp/mysql3306.sock -p wubx < filename.sql`
+ mysql修改客户端登录提示符:`mysql_config_editor set --login-path=3306 --user=xx --passwrod=xx --socket=xx`
+ mysql恢复默认客户端登录提示符：`prompt`

##mysql客户端工具终止符##
+ ";"或者"\G"
+ \c 终止语句
+ mysql允许语句多行输入
+ 退出:\q或是quit,exit

##mysqladmin##
+ 强制回应(ping)服务器
+ 关闭服务器
+ 创建和删除数据库
+ 显示服务器和版本信息
+ 显示或充值服务器状态变量
+ 设置口令
+ 重新装入授权表
+ 刷新日志文件和高速缓存
+ 启动和停止复制
+ 显示客户机信息

##总结##
+ 如果主机名给了localhost，那么该账号只能从socket文件登录，localhost不支持TCP登录
+ mysql创建用户不要直接用"%",不校验来源。漏洞较大
+ mysql登录是精确匹配的。每种登录限制权限的可以使用不同的密码
+ create user创建出来的用户，不需要`flush privileges;`
+ mysql 7 有个mysql.sys的账户，使用户sys库的
+ 将mysql用户lock，可以让用户不能登录：`update user set account_locked='N' where user='wubx'`
+ mysql通过`show warnings`来查看上一条语句的警告
+ 改密码的二次方法，将mysql的user表的frm、myd、myi三个文件拷贝到另一个库去进行修改密码，改完后在拷贝回来。重启mysql，或者通过`kill -HUP 'pidof mysqld'`，让mysql重新加载下权限
+ 通过`man mysql`查看mysql的帮助
+ 通过`mysql --help |grep mysql`查看mysql的配置文件加载顺序
+ mysql配置文件中，[mysql]继承[clint]配置文件
+ mysql工具：toad for mysql,sqlyog

