# 介绍

k8s各个容器之间的访问会涉及到三种情况

* 集群内部访问
* 集群内 --> 集群外的访问
* 集群外 -> 集群内的访问

# 集群内部访问

1. service

   pod到pod的访问，pod自己本身的ip是一直在变化的，所以需要service来对pod进行管理 service对pod是负载均衡的 service的ip是固定不变的。又通过DNS解析service的名字和service ip对应关系 供集群内pod通过名字访问

2. HeadlessService

   HeadlessService并不会对pod进行负载均衡 而是把对应pod的ip列表返回给调用者 调用者根据返回的ip列表进行访问

![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/k8s%E6%9C%8D%E5%8A%A1%E5%8F%91%E7%8E%B0-%E9%9B%86%E7%BE%A4%E5%86%85%E8%AE%BF%E9%97%AE.png?x-oss-process=style/makjwatermarks)

# 集群内访问集群外

1. 直接根据ip端口直接访问

2. 定义一个EndPoing Service

   这个EndPoing 里的ip就是要访问的外部服务的ip地址和端口，然后通过service的dns进行名字和service 的映射，这样pod访问外部服务的时候 直接访问这个service就可以了，对于pod是无感知的

![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/k8s%E6%9C%8D%E5%8A%A1%E5%8F%91%E7%8E%B0-%E9%9B%86%E7%BE%A4%E5%86%85%E8%AE%BF%E9%97%AE%E5%A4%96%E9%83%A8%E6%9C%8D%E5%8A%A1.png?x-oss-process=style/makjwatermarks)

# 集群外访问内部服务

1. NodePort访问

   根据service 的NodePort访问，NodePort会在每个节点暴露一个端口来映射转发到当前需要访问的pod的ip，会有多层转发

2. HostPort访问

   不会在所有节点上开启端口转发，只会在当前pod运行的节点上开启一个端口映射。pod运行在哪个节点，哪个节点就会开启一个端口，客户端请求的时候只能访问有这个实例的节点和端口

3. ingressController访问

   相当于在k8s内置一个nginx，这个nginx映射了所以所有需要请求的服务，外部服务请求的时候只需要请求这个nginx就可以得到相应的转发

   ingress需要提供各个host和域名的对应关系，访问path和api的对应关系  service和api-service的对应关系

   ingressController可以自定义实现，ingressController可以监听到k8s内部事件 监听的service和pod的变化来实施修改nginx的对应关系的变化

   ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/k8s%E6%9C%8D%E5%8A%A1%E5%8F%91%E7%8E%B0-%E9%9B%86%E7%BE%A4%E5%A4%96%E9%83%A8%E8%AE%BF%E9%97%AE%E9%9B%86%E7%BE%A4%E5%86%85%E9%83%A8.png?x-oss-process=style/makjwatermarks)