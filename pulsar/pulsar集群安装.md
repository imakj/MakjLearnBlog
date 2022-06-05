# 准备环境

* 三节点
  * 192.168.10.181 (pulsar1)
  * 192.168.10.182 (pulsar2)
  * 192.168.10.183 (pulsar3)

* zookeeper版本：apache-zookeeper-3.7.1-bin.tar
  * 192.168.10.140 (zookeeper1)
  * 192.168.10.141 (zookeeper2)
  * 192.168.10.142 (zookeeper3)
* jdk: jdk-8u333
* pulsar: apache-pulsar-2.10.0-bin.tar.gz
* 上传安装包到三节点的/opt/pulsar目录
* 停止/关闭防火墙

# 配置环境

配置hostname 三台节点配置

```she
# echo "192.168.10.181 pulsar1" >> /etc/hosts
# echo "192.168.10.182 pulsar2" >> /etc/hosts
# echo "192.168.10.183 pulsar3" >> /etc/hosts

# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.10.181 pulsar1
192.168.10.182 pulsar2
192.168.10.183 pulsar3

```

# 安装

1. 三节点解压

   ```
   # cd /opt/pulsar
   # tar zxvf apache-pulsar-2.10.0-bin.tar.gz
   # ln -s ./apache-pulsar-2.10.0 ./puslar
   ```

2. 创建目录

   ```
   # mkdir -p /data/pulsar/
   ```

3. 修改bookkeeper配置文件 

   所有节点修改 其中advertisedAddress的值每个节点不一致

   ```
   # vim conf/bookkeeper.conf 
   ……
   # 更改为本地ip地址 每个节点不一样  节点2为 192.168.10.182  节点3为192.168.10.183
   advertisedAddress=192.168.10.181
   # 修改日志目录
   journalDirectory=/data/pulsar/data/bookkeeper/journal
   # 修改ledger目录
   ledgerDirectories=/data/pulsar/data/pulsar/tmp/ledger
   # 修改zk配置
   zkServers=192.168.10.140:2181,192.168.10.141:2181,192.168.10.142:2181
   ……
   ```

4. 修改broker配置文件

   所有节点修改 其中advertisedAddress的值每个节点不一致

   ```
   # vim conf/broker.conf
   ……
   # 修改集群的名称
   clusterName=pulsar-cluster
   # 配置zookeeper地址
   zookeeperServers=192.168.10.140:2181,192.168.10.141:2181,192.168.10.142:2181
   # 配置存储的地址 还是zookeeper地址
   globalZookeeperServers=192.168.10.140:2181,192.168.10.141:2181,192.168.10.142:21
   81
   configurationStoreServers=192.168.10.140:2181,192.168.10.141:2181,192.168.10.142:2181
   # 更改为本地ip地址 每个节点不一样  节点2为 192.168.10.182  节点3为192.168.10.183
   advertisedAddress=192.168.10.181
   ……
   ```

5. 启动zk集群 

   确保zk集群正常

   ```
   # cd /opt/zookeeper/zookeeper && bin/zkServer.sh status
   ```

6. 初始化集群信息

   * cluster: 集群名称
   * zookeeper：zookeeper地址
   * configuration：配置文件存储地址
   * web-service-url：web服务地址  建议写ip地址 不然其他应用没有hosts链接不成功
   * web-service-url-tls： web服务tls地址  建议写ip地址 不然其他应用没有hosts链接不成功
   * broker-service-url：broker地址 建议写ip地址 不然其他应用没有hosts链接不成功 
   * broker-service-url-tls ：broker的tls地址 建议写ip地址 不然其他应用没有hosts链接不成功

   ```
   # cd /opt/pulsar/puslar/bin
   
   # ./pulsar initialize-cluster-metadata \
   --cluster pulsar-cluster \
   --zookeeper 192.168.10.140:2181,192.168.10.141:2181,192.168.10.142:2181 \
   --configuration-store 192.168.10.140:2181,192.168.10.141:2181,192.168.10.142:2181 \
   --web-service-url http://192.168.10.181:8080,192.168.10.182:8080,192.168.10.183:8080 \
   --web-service-url-tls https://192.168.10.181:8443,192.168.10.182:8443,192.168.10.183:8443 \
   --broker-service-url pulsar://192.168.10.181:6650,192.168.10.182:6650,192.168.10.183:6650 \
   --broker-service-url-tls pulsar+ssl://192.168.10.181:6651,192.168.10.182:6651,192.168.10.183:6651
   ```

7. 初始化bookeeper信息

    接着初始化bookkeeper集群: 若出现提示输入Y/N: 请输入Y

   ```
   # cd /opt/pulsar/puslar/bin
   # ./bookkeeper shell metaformat
   ```

8. 启动bookkeeper

   注意: 三个节点都需要依次启动

   ```
   # cd /opt/pulsar/puslar/bin && ./pulsar-daemon start bookie
   
   # 验证是否启动: 可三台都检测 有Bookie sanity test succeeded即城关
   # ./bookkeeper shell bookiesanity
   ```

9. 启动broker

   注意: 三个节点都需要依次启动

   ```
   # cd /opt/pulsar/puslar/bin && ./pulsar-daemon start broker
   
   # 检测是否启动 可以三台都验证
   # ./pulsar-admin brokers list pulsar-cluster
   192.168.10.183:8080
   192.168.10.182:8080
   192.168.10.181:8080
   ```

10. 测试

    pulsar的topic名称长 比如persistent://public/default/test 代表:永久的还是临时的://哪一个租户的/哪一个命名空间的/topic

    * persistent: 表示永久的还是临时的
    * public：哪一个租户的
    * default：哪一个命名空间的

    在任意一个节点中启动消费者和者生产者

    ```
    # 启动一个消费者
    # ./pulsar-client consume persistent://public/default/test -s "consumer-test"
    
    # 一直监听消息
    # ./pulsar-client consume persistent://public/default/test -s "consumer-test" -n 0
    
    # 启动一个生产者模拟一条数据
    # ./pulsar-client produce persistent://public/default/test --messages "test-pulsar"
    ```

# 监控

1. 下载安装pulsar-manager

   https://dist.apache.org/repos/dist/release/pulsar/pulsar-manager/pulsar-manager-0.3.0/apache-pulsar-manager-0.3.0-bin.tar.gz

2. 上传到pulsar1节点 解压

   ```
   # cd /opt/pulsar
   # tar -zxvf apache-pulsar-manager-0.3.0-bin.tar.gz 
   
   # cd pulsar-manager/
   # tar -xvf pulsar-manager.tar 
   
   # 拷贝ui界面到当manager目录
   # cd /opt/pulsar/pulsar-manager/pulsar-manager
   # cp -r ../dist ./ui
   ```

3. 启动manager

   ```
   # 只能在这个目录启动
   # cd /opt/pulsar/pulsar-manager/pulsar-manager
   
   # ./bin/pulsar-manager
   ```

4. 配置超级用户密码

   ```
   # CSRF_TOKEN=$(curl http://192.168.10.181:7750/pulsar-manager/csrf-token)
   
   # curl \
   -H "X-XSRF-TOKEN: $CSRF_TOKEN" \
   -H "Cookie: XSRF-TOKEN=$CSRF_TOKEN;" \
   -H 'Content-Type: application/json' \
   -X PUT http://192.168.10.181:7750/pulsar-manager/users/superuser \
   -d '{"name": "pulsar", "password": "pulsar", "description": "test", "email": "makj@test.org"}'
   {"message":"Add super user success, please login"}
   ```

5. 访问

   http://192.168.10.181:7750/ui/index.html 

   添加集群 集群地址为 

   Service URL：http://192.168.10.181:8080,192.168.10.182:8080,192.168.10.183:8080

   Bookie URL:  http://192.168.10.181:8000,192.168.10.182:8000,192.168.10.183:8000