1. strip命令，可以把一些换行符，或者没有用的符号给过滤掉。命令:`[root@localhost ~]# strip mysqld`
2. 各个应用的tmp和系统的tmp分开，防止应用将tmp写满，ssh登录不上去了。
3. `ss -nlp | grep mysqld` 查看mysql端口
4. alias命令，定义别名。例如：`alial mr=/usr/local/mysql/bin/mysqld --defaults-file=/data/mysql/mysql3306/my3306.cnf` 然后输入`mr`就可以直接启动。将这些命令写到`.bash_profile`就可以持久化
5. `kill -HUP 'pidof mysqld'`:通知进程，重新加载配置文件。
6. 查看所有kill命令`kill -l`
7. `kill -0 pid`不是杀进程，只是检测这个进程在不在。
8. kill命令默认是`kill -15 pid`。例如`kill 123`就是`kill -15 123`，mysql默认就是`kill -15 mysqlid`