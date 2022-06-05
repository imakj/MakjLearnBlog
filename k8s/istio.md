官网：https://istio.io/latest/docs/concepts/traffic-management/

官网：https://istio.io/latest/zh/

istio就是一个代理 接管所有pod的代理 通过pod之间可以相互通讯让服务接管到所有pod

如图 所有的pod的网络都被service Mesh Control plane管理起来

![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/istio-%E6%A8%A1%E5%9E%8B.png?x-oss-process=style/makjwatermarks)

如图每个service都通过proxy进行通讯

* envoy:istio采用envoy来完成代理组件，有动态服务发现，负载均衡 断路器等这些功能，所有的网络负载都是由envoy来完成的
* pilot: 给envoy提供配置信息的。 和envoy是核心组件
* mixer: 是一个独立的组件 分为两个 一个是策略 一个是遥测 策略 为集群提供访问控制，比如那些应用访问那些服务和访问限制等  遥测主要是数据收集和回报 收集服务之间流转数据 相当于链路监控
* galley：在1.1版本之后负责整个配置管理中心 负责整个istio的配置是否正确
* citadel：安全相关的 包括rbac的访问控制 服务之间的访问访问授权 http升级为https

![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/istio-%E6%9E%B6%E6%9E%84.png?x-oss-process=style/makjwatermarks)

![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/istio-%E8%A7%A3%E5%86%B3%E9%97%AE%E9%A2%981.png?x-oss-process=style/makjwatermarks)

![](images\istio-解决问题2.png)

![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/istio-%E8%A7%A3%E5%86%B3%E9%97%AE%E9%A2%983.png?x-oss-process=style/makjwatermarks)

![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/istio-%E8%A7%A3%E5%86%B3%E9%97%AE%E9%A2%984.png?x-oss-process=style/makjwatermarks)

# 安装 

入门文档：https://istio.io/latest/zh/docs/setup/getting-started/

安装要求：https://istio.io/latest/zh/docs/setup/platform-setup/gardener/

配置要求：https://istio.io/latest/zh/docs/setup/additional-setup/config-profiles/

1. 下载, github地址：https://github.com/istio/istio/releases/tag/1.9.5

   ```
   [root@k8shardway1 istio]# wget https://github.com/istio/istio/releases/download/1.9.5/istio-1.9.5-linux-amd64.tar.gz
   
   [root@k8shardway1 istio]# tar zxvf istio-1.9.5-linux-amd64.tar.gz 
   
   [root@k8shardway1 istio]# cd istio-1.9.5
   
   ```

   