# 首行感谢大佬:

[立花泷bili](https://blog.csdn.net/qq_41859728)

https://blog.csdn.net/qq_41859728/article/details/109187748

本教程全程参照大佬的方案

# 提前结果 最后改用git大佬的编译结果

https://github.com/RikudouPatrickstar/JetBrainsRuntime-for-Linux-x64/releases/tag/202110301849

https://blog.csdn.net/NoodleMaster/article/details/117267275

# 问题

ubuntu20.04环境下 JetBrains idea打字输入法总在最下面，没有跟随光标一起定位

# 解决

手动编译

JetBrainsRuntime + OpenJFX 修复bug

# 环境配置

* 系统：ubuntu 20.04 LTS
* 内存：10 G
* java版本：[OpenJDK](https://so.csdn.net/so/search?q=OpenJDK&spm=1001.2101.3001.7020) 11.0.13 
* gcc版本：gcc 7.5.0
* idea版本：2021.3.1

# 开始

1. 安装依赖

   ```
   $ sudo apt install -y ksh bison flex gperf build-essential libasound2-dev libgl1-mesa-dev \
       libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev libjpeg-dev \
       libpng-dev libx11-dev libxml2-dev libxslt1-dev libxt-dev \
       libxxf86vm-dev pkg-config x11proto-core-dev \
       x11proto-xf86vidmode-dev libavcodec-dev mercurial \
       libgtk2.0-dev libgtk-3-dev \
       libxtst-dev libudev-dev libavformat-dev ant
    
   $ sudo apt install -y cmake ruby
   ```

2. 安装openJDK11

   ```
   $ sudo apt install openjdk-11-jdk
   # 设置环境变量
   $ export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
   $ export JDK_HOME=$JAVA_HOME
   ```

3. 获取OpenJFX源码 目前只获取到11.0.11的

   ```
   $ sudo apt install mercurial
   $ hg clone http://hg.openjdk.java.net/openjfx/11-dev/rt
   ```

4. 编译

   ```
   $ cd rt
   $ chmod a+x gradlew
   $ ./gradlew -PCOMPILE_WEBKIT=true #注意一定要编译Webkit，不然Markdown一样无法预览
   ```

# 编译JetBrainsRuntime

1. 下载JetBrainsRuntime源码

   ```
   $ git clone https://github.com/JetBrains/JetBrainsRuntime.git
   ```

2. 下载并应用patch

   ```
   $ cd JetBrainsRuntime
   # $ git checkout cfc3e87f2ac27a0b8c78c729c113aa52535feff6 #大佬这一步是checkout11.0.7版本 可以不要 经过验证idea.patch可以应用到11.0.11
   $ wget https://raw.githubusercontent.com/prehonor/myJetBrainsRuntime/master/idea.patch
   $ git apply idea.patch
   ```

3. 安装依赖

   ```
   $ sudo apt install autoconf make build-essential libx11-dev libxext-dev libxrender-dev libxtst-dev libxt-dev libxrandr-dev libcups2-dev libfontconfig1-dev libasound2-dev 
   ```

4. 编译并整合OpenJFX

   ```
   # sh ./configure --disable-warnings-as-errors  --with-import-modules=_path_to_jfx-dev_/rt/build/modular-sdk #_path_to_jfx-dev_是第一步获取的OpenJFX源码即rt文件夹的绝对路径, 下面path_to_JetBrainsRuntime同理
   $ sh ./configure --disable-warnings-as-errors  --with-import-modules=/home/makj/rt/build/modular-sdk/
   
   $ make images
   ```

# 安装JetBrainsRuntime

1. 重命名jdk为jbr

   ```
   $ cd /home/makj/JetBrainsRuntime/build/linux-x86_64-normal-server-release/images/
   mv jdk jbr
   ```

2. 直接替换idea环境下的jbr文件

