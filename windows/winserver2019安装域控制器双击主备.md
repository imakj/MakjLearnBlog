<b>环境</b>

* 系统：windows server 2019
* 网络:192.168.18.X
* 子网掩码:255.255.255.0
* 网关:192.168.18.1
* DC1:192.168.18.30/24
* DC2:192.168.18.31/24

**DC1 安装**

 1. 修改DC1的DNS，使DNS指向自己的IP并且修改DC1的计算机名字为DC1，修改完成后如图:

    ![第一步](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/one.png?x-oss-process=style/makjwatermarks)



2. 安装域:服务器管理 -> 添加角色和功能 如图

   ![安装域图1](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/two_1.png?x-oss-process=style/makjwatermarks)

   ![安装域图1](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/two_2.png?x-oss-process=style/makjwatermarks)

   ![安装域图1](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/two_3.png?x-oss-process=style/makjwatermarks)

   ![安装域图1](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/two_4.png?x-oss-process=style/makjwatermarks)

   到这里一直点击下一步到安装，如图

   ![安装域图1](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/two_5.png?x-oss-process=style/makjwatermarks)

   ![安装域图1](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/two_6.png?x-oss-process=style/makjwatermarks)

3. 安装完成后，将此控制器提升为域控制器。

   ​	![提升为域控制器](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/three_1.png?x-oss-process=style/makjwatermarks)
   
   
      3.1 定义域名。
   
   ![提升为域控制器](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/three_2.png?x-oss-process=style/makjwatermarks)3.2 设置功能级别和密码。

 ![提升为域控制器](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/three_3.png?x-oss-process=style/makjwatermarks)

​	3.3 点击下一步，因为是第一个域控制器。默认无法创建DNS服务器委派，忽略并下一步继续

   ![提升为域控制器](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/three_4.png?x-oss-process=style/makjwatermarks)

   3.4 确认NetBIOS域名名称，默认与根域名前缀一致，直接下一步继续。

   ![提升为域控制器](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/three_5.png?x-oss-process=style/makjwatermarks)

   3.5 指定AD DS数据库、日志文件和SYSVOL位置，本例接受默认(生产环境建议将三者分开存放)并下一步继续

   ​	数据库文件夹：用了存储AD数据库

   ​	日志文件文件夹：用了存储AD的更改记录，此记录可以用来修复AD数据库

   ​	SYSVOL文件夹：用了存储域共享文件（例如组策略）

   ![提升为域控制器](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/three_6.png?x-oss-process=style/makjwatermarks)

   3.6 确认所有信息，指定AD DS数据库、日志文件和SYSVOL位置，本例接受默认(生产环境建议将三者分开存放)并下一步继续。

   3.7  系统顺利通过检查，点击安装

   ![域安装](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/three_7.png?x-oss-process=style/makjwatermarks)

   3.8 安装完成。系统会重启。重启完成后。系统会多了了AD的管理指令，最后在管理工具里检查DNS是否成功点击工具如图：

   ​		![域安装](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/three_8.png?x-oss-process=style/makjwatermarks)

![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/three_9.png?x-oss-process=style/makjwatermarks)



域控制器如果注册成功，那么域控制器已经正确的将家里注册到dns服务器，应该还会有_tcp _udp等文件夹。单击_tcp文件夹后可以看到数据类型为服务位置(SRV)的_ldap记录，表示dc1.contoso.com已经正确的注册为域控制器。还能看到_gc记录全局编录也是由dc1.dcmakj.com所扮演.

* 如果注册失败排除注册失败的问题

  如果域成员本身的设置或者网络问题，会造成无法将数据注册到DNS服务器。

  如果有成员计算机的主机与ip美元正确注册到DNS服务器，可以到此机器上运行ipconfig /registerdns来手动注册。完成后，到DNS服务器检查是否已有正确记录，例如server1.dcmakj.com，ip地址192.168.18.100则坚持区域dcmakj.com是否有对应的a记录和ip。

  如果发现域控制器没有将其扮演的角色注册到dns服务器，也就是没有_tcp文件夹与记录，到服务器中重启netlogon服务

  ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/three_10.png?x-oss-process=style/makjwatermarks)



至此，第一台域控制器已经装完。



========================================我是分割线===========================



4. 安装第二台域控制器

   4.1 初始化第二台域控制器。修改IP。在第二服务器系统中，打开计算机属性，修改计算机名为DC2，加入域为dcmakj.com，DNS后缀为dcmakj.com，如图；然后再弹出的加入域授权凭据对话框中输入域控制器的账号和密码并确定，然后重启，完成域的加入。重启。

   ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/four_1.png?x-oss-process=style/makjwatermarks)

   ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/four_2.png?x-oss-process=style/makjwatermarks)

   ![1557405442386](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/four_3.png?x-oss-process=style/makjwatermarks)

5. 按照第一台域控制器的方法，安装Active Directory 域服务和DNS服务器角色。

6. 在Active Directory 域服务配置向导的部署配置标签中，选择将域控制器增加到现有域，填写域名makjdc.com，提供此操作的凭据makjdc\administrator（域管理员账户密码作为凭据）选择下一步，参考如图

   ![1557405442386](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/four_4.png?x-oss-process=style/makjwatermarks)

7. 如图，填入域密码

   ![1557409614466](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/seven.png?x-oss-process=style/makjwatermarks)

8. 点击下一步，再点击下一步，出现如下画面，选择复制自。。。。

   ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/eight.png?x-oss-process=style/makjwatermarks)

9. 填入文件路径![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/nine.png?x-oss-process=style/makjwatermarks)

10. 一直点击下一步，直到安装界面出现

    ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/ten.png?x-oss-process=style/makjwatermarks)

11. 配置主备域控制器

    ![image](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/eleven.png?x-oss-process=style/makjwatermarks)

    ![](images/eleven_2.png)

    ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/eleven_3.png?x-oss-process=style/makjwatermarks)

    接下来在DC1域上配置DNS，使之一样

    ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/eleven_4.png?x-oss-process=style/makjwatermarks)

12. 最后验证两个域是否添加成功

    ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/twelve.png?x-oss-process=style/makjwatermarks)

<https://blog.csdn.net/weixin_40283570/article/details/81184299>

<https://blog.csdn.net/wenzhongxiang/article/details/79338279>