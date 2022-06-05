##MySQL备份##
+ mysqldump&mysqlpump使用
+ mydumper使用
+ xtrabackup使用
+ 关于meta data lock
+ 使用备份工具，在线建从库
+ innodb表空间传输的实现
+ 备份策略定制

##备份时用不用停机分为##
+ 冷备
+ 热备

##从备份的量分为##
+ 全量
+ 增量

##备份方法分为##
+ 逻辑备份
+ 物理备份

###基于SQL层的备份：###
+ 官方工具：mysqldump 
+ 社会工具：mydumper

###基于物理文件copy的备份，针对于innodb：###
+ 官方工具：mysql enterprise backup
+ xtrabackup

###利用从库备份###
Replication salve(Delay Replication)

mysqldump的实现原理：http://url.cn/43sMQhB

##Xtrabackup##
percona公司的www.percona.com 
###备份策略###
+ 首先需要一种策略保证数据不会丢
+ 对备份使用的定义
+ 备份校验机制
------

研究东西：
+ ceph。
+ innodb_ruby:一个可以校验innodb表空间的东西
+ mysqlpump是给予gtid的，离开gtid就是没用
+ 集合运算

##meta data ock##
mysql5.5之前DDL语句不在事物框架之内，比如会话1正在进行查询，会话2突然修改了表结构，就会发现查询不一致，这个要验证，5.5之后加了一个S锁，其他会话可以读，但是不能修改表结构，等待S锁释放了，拿到了X锁，才能修改。8.0已经将meta data lock纳入到了事物框架之内

##案例##
如果用netstat -an看到有大量的time_wait的时候，有可能是TCP回收问题，是个万年坑。
如果在mysql出现大量的time_wait，估计就是结果集太大了
处理方法
```
vi /etc/sysctl.conf 
#添加：
net.ipv4.tcp_suncookies=1
net.ipv4.tcp_tw_reuse=1
net.ipv4.tcp_tw_recycle=1
net.ipv4.tcp_fin_timeout=30
最后执行/sbin/sysctk -p 使参数生效
```


