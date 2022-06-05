# 准备环境

1. 下载软件包：

   这里下载2.10.0

   https://pulsar.apache.org/download

2. 上传到服务器:

3. 安装java 依赖于java 此环境已经安装了jdk8

# 安装

1. 进入到软件目录解压

   ```
   # cd /opt/pulsar/
   # tar zxvf apache-pulsar-2.10.0-bin.tar.gz
   # ln -s ./apache-pulsar-2.10.0 ./pulsar
   ```

2. 启动(单机模式)

   ```
   # cd bin/
   ```

3. 测试

   ```
   # 启动消费者 指定消费者名称
   # ./pulsar-client consume test-topic -s "test-consume"
   
   # 启动生产者生产一条数据
   ./pulsar-client produce test-topic --messages "test-pulsar"
   ```

   