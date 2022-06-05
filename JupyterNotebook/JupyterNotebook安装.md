# 准备环境

## python

1. 准备python3.8

   ```
   # cd /usr/local/src
   # mkdir /usr/local/pythin38
   # tar zxvf Python-3.8.13.tgz 
   # 安装依赖
   # yum install -y gcc patch libffi-devel python-devel  zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gdbm-devel db4-devel libpcap-devel xz-devel
   ```

2. 编译安装

   ```
   # cd Python-3.8.13
   # ./configure --prefix=/usr/local/pythin38/
   # make -j
   # make install
   # ls /usr/local/pythin38/
   ```

3. 配置系统环境变量

   ```
   # vim  /etc/profile
   # 最后一行添加
   ……
   PATH=$PATH:/usr/local/pythin38/bin
   ……
   # python3 --version
   Python 3.8.13
   ```

4. 或者添加软连接

   ```
   # ln -s /usr/local/pythin38/bin/python3 /usr/bin/python3
   # ln -s /usr/local/pythin38/bin/pip3 /usr/bin/pip3
   ```

# 安装jupterNotebook

```
# pip3 install jupyter -i https://pypi.tuna.tsinghua.edu.cn/simple
# 查看是否成功 官方不建议以root用户运行 下面有登录的token
# jupyter notebook --ip=192.168.10.169 --no-browser --allow-root
[I 15:35:53.478 NotebookApp] Serving notebooks from local directory: /usr/local/src/Python-3.8.13
[I 15:35:53.478 NotebookApp] Jupyter Notebook 6.4.11 is running at:
[I 15:35:53.478 NotebookApp] http://192.168.10.169:8888/?token=0611072ca9ddac547ec039a6d78ddbd3bb8e870da16b0d09
[I 15:35:53.478 NotebookApp]  or http://127.0.0.1:8888/?token=0611072ca9ddac547ec039a6d78ddbd3bb8e870da16b0d09
[I 15:35:53.478 NotebookApp] Use Control-C to stop this server and shut down all kernels (twice to skip confirmation).
[C 15:35:53.481 NotebookApp] 
    
    To access the notebook, open this file in a browser:
        file:///root/.local/share/jupyter/runtime/nbserver-14102-open.html
    Or copy and paste one of these URLs:
        http://192.168.10.169:8888/?token=0611072ca9ddac547ec039a6d78ddbd3bb8e870da16b0d09
     or http://127.0.0.1:8888/?token=0611072ca9ddac547ec039a6d78ddbd3bb8e870da16b0d09
[I 15:35:57.272 NotebookApp] 302 GET / (192.168.10.90) 1.050000ms
[I 15:35:57.275 NotebookApp] 302 GET /tree? (192.168.10.90) 1.170000ms
[I 15:36:04.323 NotebookApp] 302 GET /?token=0611072ca9ddac547ec039a6d78ddbd3bb8e870da16b0d09 (192.168.10.90) 0.350000ms

# 也可以修改密码，直接密码登录
# jupyter notebook password

# 访问url进入
http://192.168.10.169:8888/
```

# 配置

默认情况下，在哪个目录启动jupterNotebook 当前的工作目录就是哪个目录，可以设置指定目录

1. 生成配置目录

   ```
   # jupyter notebook --generate-config
   Writing default config to: /root/.jupyter/jupyter_notebook_config.py
   ```

2. 编辑配置文件 ，修改目录 完成后 工作目录就是修改的目录了

   在/opt/jupyternotebook创建文件 jupyternotebook页面就会有相应的目录

   ```
   # vim  /root/.jupyter/jupyter_notebook_config.py
   ……
   c.NotebookApp.notebook_dir = '/opt/jupyternotebook'
   ……
   ```

# 使用

1. 新建项目 python3环境

   ![image-20220604163514576](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/image-20220604163514576.png?x-oss-process=style/makjwatermarks)

2. 重命名项目

   需要先停止项目 然后重命名

   ![image-20220604163942947](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/image-20220604163942947.png?x-oss-process=style/makjwatermarks)

   ![image-20220604163959672](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/image-20220604163959672.png?x-oss-process=style/makjwatermarks)

3. 项目运行

   如图，中括号中的数字表示当前运行的第几个程序 如果是*号 代表正在运行

   ![image-20220604164337406](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/image-20220604164337406.png?x-oss-process=style/makjwatermarks)

4. 检查点，可以手动保存创建检查点，用于出错的时候回退 如下图 保存和回退

   自动保存不会创建记录点 只有手动保存才会

   ![image-20220604164535900](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/image-20220604164535900.png?x-oss-process=style/makjwatermarks)

   ![image-20220604164620536](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/image-20220604164620536.png?x-oss-process=style/makjwatermarks)

5. 停止

   每个notebook都对应一个kernel内核 停止notebook就是停止一个内核

   ![image-20220604165455300](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/image-20220604165455300.png?x-oss-process=style/makjwatermarks)

# 主题

1. 安装主题库

   ```
   # pip3 install jupyterthemes -i https://pypi.tuna.tsinghua.edu.cn/simple
   ```

2. 查看主题

   ```
   # jt -l
   Available Themes: 
      chesterish
      grade3
      gruvboxd
      gruvboxl
      monokai
      oceans16
      onedork
      solarizedd
      solarizedl
   ```

3. 设置主题

   ```
   # jt -t chesterish
   
   # 恢复默认主题
   # jt -r
   ```

4. 设置完成后重启

   ```
   # jupyter notebook --ip=192.168.10.169 --no-browser --allow-root
   ```

# 插件

1. 安装插件库

   ```
   # pip3 install jupyter_nbextensions_configurator -i https://pypi.tuna.tsinghua.edu.cn/simple
   
   # 启用
   # jupyter nbextensions_configurator enable --user
   ```

2. 启动

   会显示一个配置器 还需要安装插件

   ```
   # jupyter notebook --ip=192.168.10.169 --no-browser --allow-root
   ```

3. 安装第三方插件

   ```
   # pip3 install jupyter_contrib_nbextensions -i https://pypi.tuna.tsinghua.edu.cn/simple
   # jupyter contrib nbextension install --user
   ```

4. 至此插件就可用了

   ![image-20220604215522713](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/image-20220604215522713.png?x-oss-process=style/makjwatermarks)

# 安装java内核

1. 在官方下载java内核

   https://github.com/jupyter/jupyter/wiki/Jupyter-kernels

   目前最低支持到java9

   下载内核包：https://github.com/SpencerPark/IJava/releases/ijava-1.3.0.zip

2. 解压安装

   ```
   # unzip ijava-1.3.0.zip 
   # python3 install.py --sys-prefix
   install.py:164: DeprecationWarning: replace is ignored. Installing a kernelspec always replaces an existing installation
     install_dest = KernelSpecManager().install_kernel_spec(
   Installed java kernel into "/usr/local/pythin38/share/jupyter/kernels/java"
   
   # 查看是否安装成功 有没有java内核
   # jupyter kernelspec list
   Available kernels:
     java       /usr/local/pythin38/share/jupyter/kernels/java
     python3    /usr/local/pythin38/share/jupyter/kernels/python3
   ```

3. 重启jupyter

   ```
   # jupyter notebook --ip=192.168.10.169 --no-browser --allow-root
   ```

4. 创建的时候就会有java内核了

   ![image-20220604224004645](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/image-20220604224004645.png?x-oss-process=style/makjwatermarks)



# tips

1. 使用？查看对象的注释或者文档

   ```
   import time
   time.sleep?
   ```

2. 执行shell命令

   在命令前加上叹号(!) 就可以直接执行shell命令 还可以将返回直接赋值给变量

   ```
   !ls ./
   
   #赋值给变量
   file = !ls
   print(file)
   ```

3. 单元格的最后一行值会被默认输出，如果不想输出 需要在最后一行添加分号;

   ```
   a = 18
   a;
   ```

   