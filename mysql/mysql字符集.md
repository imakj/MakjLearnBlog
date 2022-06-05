### 字符集 ###

+ 字符集是一套符号和编码的规则，字符串都必须有相应的字符集
+ 椒盐基是这套符号和编码的额校验规则，定义字符排序规则，字符集之间比较的规则
+ 费ASCII字符在不通字符集中，器所需要的字节数是不一样的
+ 多字节自附件是以字符进行不叫，而非以字节为单位进行比较的
+ 校验集可以用于验证大小写，不同重音灯是否一致
+ 个别椒盐基是二进制的，给予字符对应的数值进行比较
+ xxx_bin将字符串中的每一个字符用于二进制数据存储，区分大小写
+ xxx_general_ci不分区大小写，ci为case insensitive的缩写，既大小写不敏感
+ xxx_general_cs区分大小写，cs为case sensitlice缩写，大小写敏感
+ 查看mysql字符集`show character set;`
+ 设置表明不区分大小写,设置这个参数:lower_case_table_name 
  ,用`show global variables like "%lower%";`查看此参数

### 字符集和校验集 ###

+ 字符集支持多层面：服务器层(server)、数据库层(database)、数据表(table)、字段(column)、链接(connection)、结果集(result)
+ 正常情况下一个请求流程client > server > db > table > 字段 > 返回结果集，所以字符集牵扯的都是这些，任何一个环节不对应都会出现乱码

### 没有字符集的字段 ##

+ binary
+ varbinary
+ blob
+ text

##tips##

+ 建议一般把默认的参数都写出来，不然升级的时候版本不一样的默认的配置值也不一样
+ 对比两个配置文件：用默认的配置文件，在两个版本里执行`show global variables`,然后导出来对比文件
+ 在一个表里，用`select a,char_length(a),length(a) from tb`来查看一个字段几个字符，多大长度
+ 小b是bit 大B是byte 1B=8b
+ 查看系统字符集`show global varables like "%char%";`
+ 字符集转换`iconv -t gbk -f utf8 -c 1.sql > gbk1.sql`
+ 

#字符集的意义
------------------------------

+ 加速对字符型的存储及比较,所以引入字符集
+ 对字符集需要了解的
	+ 常见的字符集
	+ Unicode的支持
	+ 设置字符集的注意事项
	+ 字符集作用于varchar、char、text字段

##常见的字符集
+ latin1 
	+ 按一个字节为单位分配，拉丁1是按照字节存储的，一个中文占2个字节。在latin1中 char(10) 10代表10个字符，只不过在latin1中，一个汉子是两个字节占用两个字符。
+ gbk&gbk2312
	+ 双字节分配
+ utf8
	+ 1-3个字节分配(变长，基本都按照3个字节算。)如果客户端是utf8.但是数据库是latin1 这个时候，如果latin1中 char(10) 就只能存入3个汉子。因为utf8中一个汉子是3个字节算的
+ unicode
	+ 通用编码，支持的最大，性能也比较弱
+ utf8mb4
	+ mysql8.0默认的字符集
	+ 1-4个字节分配
	+ 支持emjo表情
+ **注意：不同的字符集转换全是通过unicode转换的**
	+ utf8 -> gbk就是:utf8 -> unicode -> gbk
	+ latin -> utf8就是：latin1 -> unicode -> utf8

##校验字符集
+ _ai:忽略方言口音(北欧)
+ _as:精确对比方言和口音(北欧)
+ _ci:忽略大小写
+ _cs:区分大小写
+ _bin:二进制格式

像 blob varbinary这种二进制字段是没有字符集的

##总结
+ char(N):N代表字符的个数，像char(10) 在gbk里可以存10个字母，10个中文，但是实际长度会是20
+ varchar(N):N也是字符个数，在N <= 255的时候 需要1 byte的overload，如果N > 255的时候，需要2 byte的overload。所以在gbk下 如果varchar(10)就是 20+1=21，varchar(256)就是 256+2=258，就可以存256个字符 256个中文 占用长度256*2+2=514
+ varchar(N):在utf8情况下，varchar(10)就是 30+1=31
+ latin1情况下char(30),存gbk字符的话可以存15个，存utf8字符可以存10个，长度都是30。因为ASCII码在任何字符集下都是占用1个字节的