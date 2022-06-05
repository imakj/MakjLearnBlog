+ mysql推荐单实例单库，可以更好利用资源，每个业务隔离，分担连接数，因为随着mysq连接数上升，性能会出现下降。
+ mysql默认读取配置文件加载顺序:/etc/my.cnf -> /etc/mysql/my.cnf -> /usr/local/mysql/my.cnf -> ~/.my.cnf
+ mysql启动方式:service mysqld start 调用 mysqld_safe,mysqld_safe调用mysqld
+ mysql配置文件生成器：http://zhishutech.net/my-cnf-wizard.html
+ mysql.cnf的文件里[mysql]下prompt="\\u@\\h\\p [\\d]>"参数，显示登录时候提示
+ mysql登录流程
![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/mysqlyonghulianjiehechaxu.png?x-oss-process=style/makjwatermarks) 