<b>winserver2019安装vCenter</b>

此文参考:https://blog.51cto.com/3701740/2112464  感谢作者

**准备环境**

* windows环境一台
* VCSA安装文件:VMware-VCSA-all-6.7.0-8217866.iso
* ESXI服务器

**安装**

**1. 第一阶段安装**

1. 虚拟光驱运行安装文件,点击里面的vcsa-ui-installer文件夹，运行安装文件

   ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/1.1.png?x-oss-process=style/makjwatermarks)

2. 先进行第一阶段，部署

   ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/1.2.png?x-oss-process=style/makjwatermarks)

3. 点击下一步

   ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/1.3.png?x-oss-process=style/makjwatermarks)

4. 选择“嵌入式PSC”。

   ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/1.4.png?x-oss-process=style/makjwatermarks)

5. 填入将vcsa安装到esxi的连接信息

   ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/1.5.png?x-oss-process=style/makjwatermarks)

6. 提示证书警告，选择“是”

   ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/1.6.png?x-oss-process=style/makjwatermarks)

7. 填入要安装vcsa虚拟机的相关信息，这里相当于新建虚拟机

   ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/1.7.png?x-oss-process=style/makjwatermarks)

8. 选择部署大小，这里只有两台exsi，故所以选择微型

   ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/1.8.png?x-oss-process=style/makjwatermarks)

9. 选择安装的磁盘。

   ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/1.9.png?x-oss-process=style/makjwatermarks)

10. 填写网络相关信息

    ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/1.10.png?x-oss-process=style/makjwatermarks)

11. 确认信息，点击完成，完成第一阶段安装

    ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/1.11.png?x-oss-process=style/makjwatermarks)

12. 第一阶段安装完成

    ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/1.12.png?x-oss-process=style/makjwatermarks)

13. 安装完成后，IP可以ping通，ESXI主机上会多一个VCSA的虚拟机。

    ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/1.13.png?x-oss-process=style/makjwatermarks)

**2. 第二阶段安装**

 1. 点击第一阶段12接的继续，开配置第二阶段配置

    ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/2.1.png?x-oss-process=style/makjwatermarks)

 2. 配置NTP服务器，这里暂时不启用SSH

    ![](images\2.2.png)

 3. 创建SSO参数

    ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/2.3.png?x-oss-process=style/makjwatermarks)

 4. 默认，点击下一步

    ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/2.4.png?x-oss-process=style/makjwatermarks)

 5. 确认参数，点击完成

    ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/2.5.png?x-oss-process=style/makjwatermarks)

 6. 点击确定

    ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/2.6.png?x-oss-process=style/makjwatermarks)

 7. 安装完成

    ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/2.7.png?x-oss-process=style/makjwatermarks)

	8. 控制台

    ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/2.8.png?x-oss-process=style/makjwatermarks)

**3.验证**

 1. VCSA 6.7提供H5以及FLASH两个选择，这里查看H5界面,输入用户名密码登录

    ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/3.1.png?x-oss-process=style/makjwatermarks)

 2. 成功进入

    ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/3.2.png?x-oss-process=style/makjwatermarks)

 3. 