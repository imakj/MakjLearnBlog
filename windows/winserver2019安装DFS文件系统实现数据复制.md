<b>winserver2019安装DFS文件系统实现数据复制</b>



* 系统：windows server 2019
* 网络:192.168.18.X
* 子网掩码:255.255.255.0
* 网关:192.168.18.1
* DFS1:192.168.18.32/24
* DFS2:192.168.18.33/24
* 域:dcmakj.com

**准备工作**

	1. 装系统
	2. 将两台服务器加入到域

**DFS1安装**

 1. 安装DFS

    ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/1.1.png?x-oss-process=style/makjwatermarks)

    ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/1.2.png?x-oss-process=style/makjwatermarks)

    点击下一步，再点击下一步出现如下界面

    ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/1.3.png?x-oss-process=style/makjwatermarks)

    一直点击下一步，点击安装，等待安装完成。

    ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/1.4.png?x-oss-process=style/makjwatermarks)

    ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/1.5.png?x-oss-process=style/makjwatermarks)

	2. 第二台机器同样安装DFS

    ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/2.png?x-oss-process=style/makjwatermarks)

	3. 配置DFS服务

    	1. 在DFS1中打开DFS管理器

    ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/3.1.png?x-oss-process=style/makjwatermarks)

    2. 新建复制组

       ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/3.2.png?x-oss-process=style/makjwatermarks)

    3. 选择--多用途复制组

       ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/3.3.png?x-oss-process=style/makjwatermarks)

    4. 填写复制组信息

       ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/3.4.png?x-oss-process=style/makjwatermarks)

    5. 选择组成员，将DFS1和DFS2同时添加到组成员中

       ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/3.5.png?x-oss-process=style/makjwatermarks)

    6. 添加完成后如图

       ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/3.6.png?x-oss-process=style/makjwatermarks)

    7. 选择交错复制

       ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/3.7.png?x-oss-process=style/makjwatermarks)

    8. 在此可根据自己的环境进行选择；如果数据量大的话，建议使用指定日期及时间复制,我选择的是指定带宽复制

       ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/3.8.png?x-oss-process=style/makjwatermarks)

    9. 选择主成员 --> master

       ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/3.8.png?x-oss-process=style/makjwatermarks)

    10. 选择同步的文件夹，定义的权限等。这里我用的默认权限和同步整个E盘

        ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/3.10.png?x-oss-process=style/makjwatermarks)

    11. 选择成功后如图

        ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/3.11.png?x-oss-process=style/makjwatermarks)

    12. 选择DFS2成员的文件夹

        ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/3.12.png?x-oss-process=style/makjwatermarks)

    13. 设置成功后如图

        ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/3.13.png?x-oss-process=style/makjwatermarks)

    14. 确认配置，点击创建

        ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/3.14.png?x-oss-process=style/makjwatermarks)

    15. 创建完成

        ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/3.15.png?x-oss-process=style/makjwatermarks)

    16. 完成后在DFS1和DFS2上都可以看到复制组

        ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/3.16_1.png?x-oss-process=style/makjwatermarks)

        ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/3.16_2.png?x-oss-process=style/makjwatermarks)

	4. 测试，在DFS1上创建一个文件夹和文件，在DFS2上看看是否可以同步成功,再次注意，在两台服务器不能同时修改一个文件，这样会产生冲突

    ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/4.png?x-oss-process=style/makjwatermarks)

	5. 









参考<https://blog.51cto.com/gaowenlong/1899283?from=groupmessage>