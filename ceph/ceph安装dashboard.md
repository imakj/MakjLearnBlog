# dashboard安装

1. 环境

   ```
   # ceph -s
     cluster:
       id:     f885f732-2d84-4d5f-a6a4-8d02c8ae3834
       health: HEALTH_OK
    
     services:
       mon: 1 daemons, quorum node1 (age 26m)
       mgr: node1(active, since 2m), standbys: node2, node3
       osd: 9 osds: 9 up (since 15m), 9 in (since 15m)
    
     data:
       pools:   0 pools, 0 pgs
       objects: 0 objects, 0 B
       usage:   9.0 GiB used, 30 TiB / 30 TiB avail
       pgs: 
   ```

   

2. 所有mgr节点安装ceph-mgr-dashboard

   ```
   # yum install -y ceph-mgr-dashboard
   ```

3. 开启插件 禁用SSL

   ```
   # ceph mgr module enable dashboard
   # ceph config set mgr mgr/dashboard/ssl false
   ```

4. 配置监听IP 0.0.0.0和端口

   ```
   # ceph config set mgr mgr/dashboard/server_addr 0.0.0.0
   # ceph config set mgr mgr/dashboard/server_port 8443
   ```

5. 设置用户及密码

   ```
   # ceph dashboard ac-user-create admin mkj123. administrator
   ```

6. 查看模块是否开启

   ```
   # ceph mgr services
   {
       "dashboard": "http://node1:8080/"
   }
   ```

7. 重启

   ```
   # ceph mgr module disable dashboard
   # ceph mgr module enable dashboard
   ```

8. 再次查看配置

   ```
   # ceph mgr services
   {
       "dashboard": "http://node1:8443/"
   }
   ```

   

https://blog.51cto.com/renlixing/2487852?source=dra



# dashboard启用rgw

1. 准备，安装rgw

   ```
   ceph-deploy rgw create node1 node2 node3
   ```

2. 推送配置文件和认证文件

   ```
   # ceph-deploy --overwrite-conf config push ceph_gateway
   # ceph-deploy admin ceph_gateway
   ```

   

3. 查看是否安装成功

```
# curl http://node1:7480
<?xml version="1.0" encoding="UTF-8"?><ListAllMyBucketsResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/"><Owner><ID>anonymous</ID><DisplayName></DisplayName></Owner><Buckets></Buckets></ListAllMyBucketsResult>
```

3. 创建rgw系统账户

   ```
   # radosgw-admin user create --uid=rgw --display-name=rgw --system
   {
       "user_id": "rgw",
       "display_name": "rgw",
       "email": "",
       "suspended": 0,
       "max_buckets": 1000,
       "subusers": [],
       "keys": [
           {
               "user": "rgw",
               "access_key": "8WIF1QXSA2JQCFKG8VTD",
               "secret_key": "SjMlQDclW75RnRuPp0FJM3tuYsCxOZ6P8to6ifgL"
           }
       ],
       "swift_keys": [],
       "caps": [],
       "op_mask": "read, write, delete",
       "system": "true",
       "default_placement": "",
       "default_storage_class": "",
       "placement_tags": [],
       "bucket_quota": {
           "enabled": false,
           "check_on_raw": false,
           "max_size": -1,
           "max_size_kb": 0,
           "max_objects": -1
       },
       "user_quota": {
           "enabled": false,
           "check_on_raw": false,
           "max_size": -1,
           "max_size_kb": 0,
           "max_objects": -1
       },
       "temp_url_keys": [],
       "type": "rgw",
       "mfa_ids": []
   }
   
   ```

4. 查看是否创建成功，并且记录下key

   ```
   # radosgw-admin user info --uid=rgw
   {
       "user_id": "rgw",
       "display_name": "rgw",
       "email": "",
       "suspended": 0,
       "max_buckets": 1000,
       "subusers": [],
       "keys": [
           {
               "user": "rgw",
               "access_key": "8WIF1QXSA2JQCFKG8VTD",
               "secret_key": "SjMlQDclW75RnRuPp0FJM3tuYsCxOZ6P8to6ifgL"
           }
       ],
       "swift_keys": [],
       "caps": [],
       "op_mask": "read, write, delete",
       "system": "true",
       "default_placement": "",
       "default_storage_class": "",
       "placement_tags": [],
       "bucket_quota": {
           "enabled": false,
           "check_on_raw": false,
           "max_size": -1,
           "max_size_kb": 0,
           "max_objects": -1
       },
       "user_quota": {
           "enabled": false,
           "check_on_raw": false,
           "max_size": -1,
           "max_size_kb": 0,
           "max_objects": -1
       },
       "temp_url_keys": [],
       "type": "rgw",
       "mfa_ids": []
   }
   
   ```

5. 为Dashboard设置access_key 和 secret_key

   ```
   # ceph dashboard set-rgw-api-access-key 8WIF1QXSA2JQCFKG8VTD
   Option RGW_API_ACCESS_KEY updated
   # ceph dashboard set-rgw-api-secret-key SjMlQDclW75RnRuPp0FJM3tuYsCxOZ6P8to6ifgL
   Option RGW_API_SECRET_KEY updated
   ```

6. 禁用SSL

   ```
   # ceph dashboard set-rgw-api-ssl-verify False
   ```

7. 刷新dashboard即可看到rgw添加成功，最后查看集群状态

   ```
   # ceph -s
     cluster:
       id:     cd2f3511-3d10-4ef7-81fd-ae5f75a68427
       health: HEALTH_OK
    
     services:
       mon: 3 daemons, quorum node1,node2,node3 (age 6h)
       mgr: node2(active, since 18m), standbys: node3, node1
       mds: cephfs:1 {0=node2=up:active} 2 up:standby
       osd: 14 osds: 14 up (since 6h), 14 in (since 15h)
       rgw: 3 daemons active (node1, node2, node3)
    
     task status:
       scrub status:
           mds.node2: idle
    
     data:
       pools:   10 pools, 832 pgs
       objects: 15.92k objects, 55 GiB
       usage:   132 GiB used, 52 TiB / 52 TiB avail
       pgs:     832 active+clean
   
   ```

# dashboard安装grafana

Ceph Dashboard中集成了grafana&prometheus，但需要手工启用，熟悉prometheus的人都知道其监控需要有exporter,ceph mgr模块中内置了prometheus exporter模块，所以无需要手工单独安装exporter,由于Ceph Dashboard中grafana还监控了Ceph存储节点的监控信息，所以每台存储节点中需要安装prometheus node exporter，借用redhat官方文档说明下这个架构：

![图片](images/配置grafana.png)

1. 在grafana(admin)节点上配置grafana yum源。 如果配置了阿里源 则不需要配置此源

   ```
   # vim /etc/yum.repos.d/grafana.repo
   [grafana]
   name=grafana
   baseurl=https://mirrors.cloud.tencent.com/grafana/yum/el7/
   enabled=1
   gpgcheck=0
   ```

2. 安装grafana。

   ```
   # yum -y install grafana -y
   ```

3. 在/etc/grafana/grafana.ini中配置Grafana 以适应匿名模式,并修改grafana默认风格，否者默认为暗黑，集成到ceph dashboard中风格不匹配。

   ```
   # vim /etc/grafana/grafana.ini
   
   default_theme = light
   [auth.anonymous]
   enabled = true
   org_name = Main Org.
   org_role = Viewer
   ```

   注意Main Org后面的个点“.”不要忘记！

   在较新版本的Grafana（从6.2.0-beta1开始）中，`allow_embedding`引入了一个名为的新设置 。该设置需要明确设置`true`，Ceph Dashboard中的Grafana集成才能正常使用，因为其默认值为`false`。

4. 启动grafana并设为开机自启。

   ```
   #systemctl start grafana-server.service 
   #systemctl status grafana-server.service 
   #systemctl enable grafana-server.service
   ```

5. 安装grafana插件。

   ```
   # grafana-cli plugins install vonage-status-panel
   installing vonage-status-panel @ 1.0.10
   from url: https://grafana.com/api/plugins/vonage-status-panel/versions/1.0.10/download
   into: /var/lib/grafana/plugins
   
   ✔ Installed vonage-status-panel successfully 
   
   Restart grafana after installing plugins . <service grafana-server restart>
   
   # grafana-cli plugins install grafana-piechart-panel
   installing grafana-piechart-panel @ 1.6.1
   from url: https://grafana.com/api/plugins/grafana-piechart-panel/versions/1.6.1/download
   into: /var/lib/grafana/plugins
   
   ✔ Installed grafana-piechart-panel successfully 
   
   Restart grafana after installing plugins . <service grafana-server restart>
   
   ```

6. 重启服务

   ```
   #systemctl restart grafana-server
   ```

7. 登录: http://192.168.10.15:3000/?orgId=1进去后默认用户名密码admin/admin,查看安装的插件

   ![图片](images/配置grafana1.png)



# dashboard安装prometheus

1. 从官方下载prometheus软件包并解压。https://prometheus.io/download/