#数据库分区，主要目的就是在特定的SQl操作中减少数据读写的总量以及缩减响应时间  
------------------------------
mysql5.6支持1024个分区 mysql5.7支持8192个分区

+ 数据库分区分为：  
 + 水平分区：以行为单位把表进行分区， mysql目前支持分局
 + 垂直分区：以列为单位把表进行分区，mysql暂时不支持

------------------------------
+ 分区历史：
	+ mysql5.6之前是因为在server层模拟的分区，如果分区过多的话，容易造成内存浪费，因为打开一个分区需要2-10M的内存，如果10个表，每个表1000个分区，连续打开10个表，就会造成10*1000*10的内存
	+ mysql5.7直接在innodb引擎层支持了Native分区，一个分区就是一个表了，就不会有那么多的开销

------------------------------
+ 分区类型
	+ Range分区
	+ List分区
	+ Columns分区
	+ Key分区
	+ Hash分区
	+ 子分区 
	
	用的比较多的分区为Range分区和Columns分区

------------------------------
+ Range分区
	+ 支持分区的列的类型：
		+ 整型
		+ datetime需要借助于时间函数转换成证书，例如year
		+ timestamp借助于unix_timestamp进行转换
	+ 特别提示对时间进行分区，如果不想进行转换，就使用range columns进行分区
	+ 创建分区例子详见图：Range分区创建的例子.png

------------------------------
+ List分区
	+ 对于List分区，分区列只能是整型
	+ 其他列如果想用List分区，需要使用List column
	+ 创建分区例子详见图：List分区创建的例子.png

------------------------------
+ columns分区
	+ range columns和list columns支持更多列类型
		+ 所有的整型
		+ data,datatime
		+ 但对于char,varchar,binary,barbinary,text,blob列还是不支持
	+ 约束
		+ range columns不能接受表达式，只能接受列名
		+ range columns可以接受一个或多个列
		+ range colums对于分区列支持更宽泛一些 
	+ columns分区测试：详见columns分区测试.png，columns分区创建的例子.png