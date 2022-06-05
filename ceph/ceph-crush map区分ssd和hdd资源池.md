# 环境

* 3块SSD

* 6块HDD

  ```
  # ceph -s
    cluster:
      id:     b4c125fd-60ab-41ce-b51c-88833089a3ad
      health: HEALTH_OK
   
    services:
      mon: 3 daemons, quorum node1,node2,node3 (age 47m)
      mgr: node1(active, since 56m), standbys: node2, node3
      osd: 9 osds: 9 up (since 44m), 9 in (since 44m)
   
    data:
      pools:   1 pools, 1 pgs
      objects: 0 objects, 0 B
      usage:   9.1 GiB used, 30 TiB / 30 TiB avail
      pgs:     1 active+clean
     
  # ceph osd tree
  ID  CLASS  WEIGHT    TYPE NAME       STATUS  REWEIGHT  PRI-AFF
  -1         29.83720  root default                             
  -3          8.73286      host node1                           
   0    hdd   3.63869          osd.0       up   1.00000  1.00000
   1    hdd   3.63869          osd.1       up   1.00000  1.00000
   6    ssd   1.45549          osd.6       up   1.00000  1.00000
  -5         12.37148      host node2                           
   2    hdd   5.45799          osd.2       up   1.00000  1.00000
   3    hdd   5.45799          osd.3       up   1.00000  1.00000
   7    ssd   1.45549          osd.7       up   1.00000  1.00000
  -7          8.73286      host node3                           
   4    hdd   3.63869          osd.4       up   1.00000  1.00000
   5    hdd   3.63869          osd.5       up   1.00000  1.00000
   8    ssd   1.45549          osd.8       up   1.00000  1.00000
  ```

  

# 需求

根据crush map将ssd和hdd分为两个资源池，iops高的放在ssd的pool，不高的放在hdd的pool



# 部署

1. 查看crush map分布：

```
# ceph osd crush tree
ID  CLASS  WEIGHT    TYPE NAME     
-1         29.83720  root default  
-3          8.73286      host node1
 0    hdd   3.63869          osd.0 
 1    hdd   3.63869          osd.1 
 6    ssd   1.45549          osd.6 
-5         12.37148      host node2
 2    hdd   5.45799          osd.2 
 3    hdd   5.45799          osd.3 
 7    ssd   1.45549          osd.7 
-7          8.73286      host node3
 4    hdd   3.63869          osd.4 
 5    hdd   3.63869          osd.5 
 8    ssd   1.45549          osd.8 
```



2. 查看详细crush规则

   ```
   # ceph osd crush dump
   {
       "devices": [
           {
               "id": 0,
               "name": "osd.0",
               "class": "hdd"
           },
           {
               "id": 1,
               "name": "osd.1",
               "class": "hdd"
           },
           {
               "id": 2,
               "name": "osd.2",
               "class": "hdd"
           },
           {
               "id": 3,
               "name": "osd.3",
               "class": "hdd"
           },
           {
               "id": 4,
               "name": "osd.4",
               "class": "hdd"
           },
           {
               "id": 5,
               "name": "osd.5",
               "class": "hdd"
           },
           {
               "id": 6,
               "name": "osd.6",
               "class": "ssd"
           },
           {
               "id": 7,
               "name": "osd.7",
               "class": "ssd"
           },
           {
               "id": 8,
               "name": "osd.8",
               "class": "ssd"
           }
       ],
       "types": [
           {
               "type_id": 0,
               "name": "osd"
           },
           {
               "type_id": 1,
               "name": "host"
           },
           {
               "type_id": 2,
               "name": "chassis"
           },
           {
               "type_id": 3,
               "name": "rack"
           },
           {
               "type_id": 4,
               "name": "row"
           },
           {
               "type_id": 5,
               "name": "pdu"
           },
           {
               "type_id": 6,
               "name": "pod"
           },
           {
               "type_id": 7,
               "name": "room"
           },
           {
               "type_id": 8,
               "name": "datacenter"
           },
           {
               "type_id": 9,
               "name": "zone"
           },
           {
               "type_id": 10,
               "name": "region"
           },
           {
               "type_id": 11,
               "name": "root"
           }
       ],
       "buckets": [
           {
               "id": -1,
               "name": "default",
               "type_id": 11,
               "type_name": "root",
               "weight": 1955411,
               "alg": "straw2",
               "hash": "rjenkins1",
               "items": [
                   {
                       "id": -3,
                       "weight": 572317,
                       "pos": 0
                   },
                   {
                       "id": -5,
                       "weight": 810777,
                       "pos": 1
                   },
                   {
                       "id": -7,
                       "weight": 572317,
                       "pos": 2
                   }
               ]
           },
           {
               "id": -2,
               "name": "default~hdd",
               "type_id": 11,
               "type_name": "root",
               "weight": 1669250,
               "alg": "straw2",
               "hash": "rjenkins1",
               "items": [
                   {
                       "id": -4,
                       "weight": 476930,
                       "pos": 0
                   },
                   {
                       "id": -6,
                       "weight": 715390,
                       "pos": 1
                   },
                   {
                       "id": -8,
                       "weight": 476930,
                       "pos": 2
                   }
               ]
           },
           {
               "id": -3,
               "name": "node1",
               "type_id": 1,
               "type_name": "host",
               "weight": 572317,
               "alg": "straw2",
               "hash": "rjenkins1",
               "items": [
                   {
                       "id": 0,
                       "weight": 238465,
                       "pos": 0
                   },
                   {
                       "id": 1,
                       "weight": 238465,
                       "pos": 1
                   },
                   {
                       "id": 6,
                       "weight": 95387,
                       "pos": 2
                   }
               ]
           },
           {
               "id": -4,
               "name": "node1~hdd",
               "type_id": 1,
               "type_name": "host",
               "weight": 476930,
               "alg": "straw2",
               "hash": "rjenkins1",
               "items": [
                   {
                       "id": 0,
                       "weight": 238465,
                       "pos": 0
                   },
                   {
                       "id": 1,
                       "weight": 238465,
                       "pos": 1
                   }
               ]
           },
           {
               "id": -5,
               "name": "node2",
               "type_id": 1,
               "type_name": "host",
               "weight": 810777,
               "alg": "straw2",
               "hash": "rjenkins1",
               "items": [
                   {
                       "id": 2,
                       "weight": 357695,
                       "pos": 0
                   },
                   {
                       "id": 3,
                       "weight": 357695,
                       "pos": 1
                   },
                   {
                       "id": 7,
                       "weight": 95387,
                       "pos": 2
                   }
               ]
           },
           {
               "id": -6,
               "name": "node2~hdd",
               "type_id": 1,
               "type_name": "host",
               "weight": 715390,
               "alg": "straw2",
               "hash": "rjenkins1",
               "items": [
                   {
                       "id": 2,
                       "weight": 357695,
                       "pos": 0
                   },
                   {
                       "id": 3,
                       "weight": 357695,
                       "pos": 1
                   }
               ]
           },
           {
               "id": -7,
               "name": "node3",
               "type_id": 1,
               "type_name": "host",
               "weight": 572317,
               "alg": "straw2",
               "hash": "rjenkins1",
               "items": [
                   {
                       "id": 4,
                       "weight": 238465,
                       "pos": 0
                   },
                   {
                       "id": 5,
                       "weight": 238465,
                       "pos": 1
                   },
                   {
                       "id": 8,
                       "weight": 95387,
                       "pos": 2
                   }
               ]
           },
           {
               "id": -8,
               "name": "node3~hdd",
               "type_id": 1,
               "type_name": "host",
               "weight": 476930,
               "alg": "straw2",
               "hash": "rjenkins1",
               "items": [
                   {
                       "id": 4,
                       "weight": 238465,
                       "pos": 0
                   },
                   {
                       "id": 5,
                       "weight": 238465,
                       "pos": 1
                   }
               ]
           },
           {
               "id": -9,
               "name": "node1~ssd",
               "type_id": 1,
               "type_name": "host",
               "weight": 95387,
               "alg": "straw2",
               "hash": "rjenkins1",
               "items": [
                   {
                       "id": 6,
                       "weight": 95387,
                       "pos": 0
                   }
               ]
           },
           {
               "id": -10,
               "name": "node2~ssd",
               "type_id": 1,
               "type_name": "host",
               "weight": 95387,
               "alg": "straw2",
               "hash": "rjenkins1",
               "items": [
                   {
                       "id": 7,
                       "weight": 95387,
                       "pos": 0
                   }
               ]
           },
           {
               "id": -11,
               "name": "node3~ssd",
               "type_id": 1,
               "type_name": "host",
               "weight": 95387,
               "alg": "straw2",
               "hash": "rjenkins1",
               "items": [
                   {
                       "id": 8,
                       "weight": 95387,
                       "pos": 0
                   }
               ]
           },
           {
               "id": -12,
               "name": "default~ssd",
               "type_id": 11,
               "type_name": "root",
               "weight": 286161,
               "alg": "straw2",
               "hash": "rjenkins1",
               "items": [
                   {
                       "id": -9,
                       "weight": 95387,
                       "pos": 0
                   },
                   {
                       "id": -10,
                       "weight": 95387,
                       "pos": 1
                   },
                   {
                       "id": -11,
                       "weight": 95387,
                       "pos": 2
                   }
               ]
           }
       ],
       "rules": [
           {
               "rule_id": 0,
               "rule_name": "replicated_rule",
               "ruleset": 0,
               "type": 1,
               "min_size": 1,
               "max_size": 10,
               "steps": [
                   {
                       "op": "take",
                       "item": -1,
                       "item_name": "default"
                   },
                   {
                       "op": "chooseleaf_firstn",
                       "num": 0,
                       "type": "host"
                   },
                   {
                       "op": "emit"
                   }
               ]
           }
       ],
       "tunables": {
           "choose_local_tries": 0,
           "choose_local_fallback_tries": 0,
           "choose_total_tries": 50,
           "chooseleaf_descend_once": 1,
           "chooseleaf_vary_r": 1,
           "chooseleaf_stable": 1,
           "straw_calc_version": 1,
           "allowed_bucket_algs": 54,
           "profile": "jewel",
           "optimal_tunables": 1,
           "legacy_tunables": 0,
           "minimum_required_version": "jewel",
           "require_feature_tunables": 1,
           "require_feature_tunables2": 1,
           "has_v2_rules": 0,
           "require_feature_tunables3": 1,
           "has_v3_rules": 0,
           "has_v4_buckets": 1,
           "require_feature_tunables5": 1,
           "has_v5_rules": 0
       },
       "choose_args": {}
   }
   ```

   

3. 查看当前pool写入osd的规则

   ```
   # ceph osd crush rule ls
   replicated_rule
   ```

   

# 手动编辑crush规则

1. 导出crushmap规则

   ```
   # ceph osd getcrushmap -o crushmap20200922.bin
   19
   ```

2. 编译导出的crushmap文件

   ```
   # crushtool -d crushmap20200922.bin -o crushmap20200922.txt
   [root@node1 ceph]# ll
   total 8
   -rw-r--r-- 1 root root 1326 Sep 22 23:06 crushmap20200922.bin
   -rw-r--r-- 1 root root 1935 Sep 22 23:07 crushmap20200922.txt
   ```

3. 修改规则

   1. 添加三个buckets来定义ssd host 并且将ssd host从原来的hdd host中删除 并且将以前host顺便改名

      ```
      # 原来的host
      ...
      
      # buckets  定义osd怎么存放的
      host node1 {
      	id -3		# do not change unnecessarily
      	id -4 class hdd		# do not change unnecessarily
      	id -9 class ssd		# do not change unnecessarily
      	# weight 8.733
      	alg straw2
      	hash 0	# rjenkins1
      	item osd.0 weight 3.639
      	item osd.1 weight 3.639
      	item osd.6 weight 1.455
      }
      host node2 {
      	id -5		# do not change unnecessarily
      	id -6 class hdd		# do not change unnecessarily
      	id -10 class ssd		# do not change unnecessarily
      	# weight 12.371
      	alg straw2
      	hash 0	# rjenkins1
      	item osd.2 weight 5.458
      	item osd.3 weight 5.458
      	item osd.7 weight 1.455
      }
      host node3 {
      	id -7		# do not change unnecessarily
      	id -8 class hdd		# do not change unnecessarily
      	id -11 class ssd		# do not change unnecessarily
      	# weight 8.733
      	alg straw2
      	hash 0	# rjenkins1
      	item osd.4 weight 3.639
      	item osd.5 weight 3.639
      	item osd.8 weight 1.455
      }
      ...
      
      # 修改后的host
      ...
      # buckets  定义osd怎么存放的
      host node1-hdd {
      	id -3		# do not change unnecessarily
      	id -4 class hdd		# do not change unnecessarily
      	# weight 8.733
      	alg straw2
      	hash 0	# rjenkins1
      	item osd.0 weight 3.639
      	item osd.1 weight 3.639
      }
      host node2-hdd {
      	id -5		# do not change unnecessarily
      	id -6 class hdd		# do not change unnecessarily
      	# weight 12.371
      	alg straw2
      	hash 0	# rjenkins1
      	item osd.2 weight 5.458
      	item osd.3 weight 5.458
      }
      host node3-hdd {
      	id -7		# do not change unnecessarily
      	id -8 class hdd		# do not change unnecessarily
      	# weight 8.733
      	alg straw2
      	hash 0	# rjenkins1
      	item osd.4 weight 3.639
      	item osd.5 weight 3.639
      }
      host node1-ssd {
      	id -9 class ssd		# do not change unnecessarily
      	# weight 8.733
      	alg straw2
      	hash 0	# rjenkins1
      	item osd.6 weight 1.455
      }
      host node2-ssd {
      	id -10 class ssd		# do not change unnecessarily
      	# weight 12.371
      	alg straw2
      	hash 0	# rjenkins1
      	item osd.7 weight 1.455
      }
      host node3-ssd {
      	id -11 class ssd		# do not change unnecessarily
      	# weight 8.733
      	alg straw2
      	hash 0	# rjenkins1
      	item osd.8 weight 1.455
      }
      ...
      ```

   2. 定义一个上层规则去调用新加的三个host，并且将以前的root default改为root hdd 并且调节权重，权重就是所有osd容量大小相加，比如单盘1.8TB，系统容量为1.635，权重为1.63539 

      ```
      #之前的default调用
      ...
      root default {
      	id -1		# do not change unnecessarily
      	id -2 class hdd		# do not change unnecessarily
      	id -12 class ssd		# do not change unnecessarily
      	# weight 29.837
      	alg straw2
      	hash 0	# rjenkins1
      	item node1 weight 8.733
      	item node2 weight 12.371
      	item node3 weight 8.733
      }
      ...
      
      #修改后的调用
      ...
      root hdd {
      	id -1		# do not change unnecessarily
      	id -2 class hdd		# do not change unnecessarily
      	# weight 29.837
      	alg straw2
      	hash 0	# rjenkins1
      	item node1-hdd weight 7.278
      	item node2-hdd weight 10.916
      	item node3-hdd weight 7.278
      }
      root ssd {
      	# weight 29.837
      	alg straw2
      	hash 0	# rjenkins1
      	item node1-ssd weight 1.456
      	item node2-ssd weight 1.456
      	item node3-ssd weight 1.456
      }
      
      ...
      ```

   3. 最后定义一个rule规则关联到上层规则,并且将以前的replicated_rule改为hdd_rule

      ```
      #之前的rule
      ...
      rule replicated_rule { 
      	id 0
      	type replicated
      	min_size 1
      	max_size 10
      	step take default
      	step chooseleaf firstn 0 type host
      	step emit
      }
      ...
      #修改后rule
      ...
      rule hdd_rule { 
      	id 0
      	type replicated
      	min_size 1
      	max_size 10
      	step take hdd
      	step chooseleaf firstn 0 type host
      	step emit
      }
      rule ssd_rule { 
      	id 1
      	type replicated
      	min_size 1
      	max_size 10
      	step take ssd
      	step chooseleaf firstn 0 type host
      	step emit
      }
      ...
      ```

   4. 修改完成后的规则

      ```
      # begin crush map
      tunable choose_local_tries 0
      tunable choose_local_fallback_tries 0
      tunable choose_total_tries 50
      tunable chooseleaf_descend_once 1
      tunable chooseleaf_vary_r 1
      tunable chooseleaf_stable 1
      tunable straw_calc_version 1
      tunable allowed_bucket_algs 54
      
      # devices 定义osd信息
      device 0 osd.0 class hdd
      device 1 osd.1 class hdd
      device 2 osd.2 class hdd
      device 3 osd.3 class hdd
      device 4 osd.4 class hdd
      device 5 osd.5 class hdd
      device 6 osd.6 class ssd
      device 7 osd.7 class ssd
      device 8 osd.8 class ssd
      
      # types  定义数组组织类型  osd 主机 机柜 机房等等
      type 0 osd
      type 1 host
      type 2 chassis
      type 3 rack
      type 4 row
      type 5 pdu
      type 6 pod
      type 7 room
      type 8 datacenter
      type 9 zone
      type 10 region
      type 11 root
      
      # buckets  定义osd怎么存放的
      host node1-hdd {
      	id -3		# do not change unnecessarily
      	id -4 class hdd		# do not change unnecessarily
      	# weight 8.733
      	alg straw2
      	hash 0	# rjenkins1
      	item osd.0 weight 3.639
      	item osd.1 weight 3.639
      }
      host node2-hdd {
      	id -5		# do not change unnecessarily
      	id -6 class hdd		# do not change unnecessarily
      	# weight 12.371
      	alg straw2
      	hash 0	# rjenkins1
      	item osd.2 weight 5.458
      	item osd.3 weight 5.458
      }
      host node3-hdd {
      	id -7		# do not change unnecessarily
      	id -8 class hdd		# do not change unnecessarily
      	# weight 8.733
      	alg straw2
      	hash 0	# rjenkins1
      	item osd.4 weight 3.639
      	item osd.5 weight 3.639
      }
      host node1-ssd {
      	id -9 class ssd		# do not change unnecessarily
      	# weight 8.733
      	alg straw2
      	hash 0	# rjenkins1
      	item osd.6 weight 1.455
      }
      host node2-ssd {
      	id -10 class ssd		# do not change unnecessarily
      	# weight 12.371
      	alg straw2
      	hash 0	# rjenkins1
      	item osd.7 weight 1.455
      }
      host node3-ssd {
      	id -11 class ssd		# do not change unnecessarily
      	# weight 8.733
      	alg straw2
      	hash 0	# rjenkins1
      	item osd.8 weight 1.455
      }
      root hdd {
      	id -1		# do not change unnecessarily
      	id -2 class hdd		# do not change unnecessarily
      	# weight 29.837
      	alg straw2
      	hash 0	# rjenkins1
      	item node1-hdd weight 7.278
      	item node2-hdd weight 10.916
      	item node3-hdd weight 7.278
      }
      root ssd {
      	# weight 29.837
      	alg straw2
      	hash 0	# rjenkins1
      	item node1-ssd weight 1.456
      	item node2-ssd weight 1.456
      	item node3-ssd weight 1.456
      }
      
      # rules 最终的root规则 pool应用到哪个
      rule hdd_rule { 
      	id 0
      	type replicated
      	min_size 1
      	max_size 10
      	step take hdd
      	step chooseleaf firstn 0 type host
      	step emit
      }
      rule ssd_rule { 
      	id 1
      	type replicated
      	min_size 1
      	max_size 10
      	step take ssd
      	step chooseleaf firstn 0 type host
      	step emit
      }
      
      # end crush map
      
      ```

      

   5. 将修改完的规则编译为bin文件

      ```
      # crushtool -c crushmap20200922.txt  -o crushmap20200922-new.bin
      # ll
      total 12
      -rw-r--r-- 1 root root 1326 Sep 22 23:06 crushmap20200922.bin
      -rw-r--r-- 1 root root 2113 Sep 22 23:33 crushmap20200922-new.bin
      -rw-r--r-- 1 root root 2516 Sep 22 23:33 crushmap20200922.txt
      ```

   6. 应用新的规则

      ```
      # ceph osd setcrushmap -i crushmap20200922-new.bin 
      20
      ```

   7. 应用后新老规则对比，已经可以区分出ssd和hdd规则

      ```
      # 应用前
      # ceph osd tree
      ID  CLASS  WEIGHT    TYPE NAME       STATUS  REWEIGHT  PRI-AFF
      -1         29.83720  root default                             
      -3          8.73286      host node1                           
       0    hdd   3.63869          osd.0       up   1.00000  1.00000
       1    hdd   3.63869          osd.1       up   1.00000  1.00000
       6    ssd   1.45549          osd.6       up   1.00000  1.00000
      -5         12.37148      host node2                           
       2    hdd   5.45799          osd.2       up   1.00000  1.00000
       3    hdd   5.45799          osd.3       up   1.00000  1.00000
       7    ssd   1.45549          osd.7       up   1.00000  1.00000
      -7          8.73286      host node3                           
       4    hdd   3.63869          osd.4       up   1.00000  1.00000
       5    hdd   3.63869          osd.5       up   1.00000  1.00000
       8    ssd   1.45549          osd.8       up   1.00000  1.00000
       
       # 应用后 可以看出有两个root规则，分为hdd和ssd
       ceph osd tree
      ID   CLASS  WEIGHT    TYPE NAME           STATUS  REWEIGHT  PRI-AFF
      -15          4.36798  root ssd                                     
      -12          1.45599      host node1-ssd                           
        6    ssd   1.45499          osd.6           up   1.00000  1.00000
      -13          1.45599      host node2-ssd                           
        7    ssd   1.45499          osd.7           up   1.00000  1.00000
      -14          1.45599      host node3-ssd                           
        8    ssd   1.45499          osd.8           up   1.00000  1.00000
       -1         25.47200  root hdd                                     
       -3          7.27800      host node1-hdd                           
        0    hdd   3.63899          osd.0           up   1.00000  1.00000
        1    hdd   3.63899          osd.1           up   1.00000  1.00000
       -5         10.91600      host node2-hdd                           
        2    hdd   5.45799          osd.2           up   1.00000  1.00000
        3    hdd   5.45799          osd.3           up   1.00000  1.00000
       -7          7.27800      host node3-hdd                           
        4    hdd   3.63899          osd.4           up   1.00000  1.00000
        5    hdd   3.63899          osd.5           up   1.00000  1.00000
      ```

   8. 验证，将之前的ceph-demo资源池的规则改为ssd的

      ```
      # ceph osd pool get ceph-demo crush_rule
      crush_rule: hdd_rule
      # ceph osd pool set ceph-demo crush_rule ssd_rule
      set pool 2 crush_rule to ssd_rule
      # ceph osd pool get ceph-demo crush_rule
      crush_rule: ssd_rule
      
      #创建一个100G的rbd
      # rbd create ceph-demo/rbd-demo.img --size 100G
      # rbd info ceph-demo/rbd-demo.img
      rbd image 'rbd-demo.img':
              size 100 GiB in 25600 objects
              order 22 (4 MiB objects)
              snapshot_count: 0
              id: 39a814402607
              block_name_prefix: rbd_data.39a814402607
              format: 2
              features: layering
              op_features: 
              flags: 
              create_timestamp: Tue Sep 22 23:42:55 2020
              access_timestamp: Tue Sep 22 23:42:55 2020
              modify_timestamp: Tue Sep 22 23:42:55 2020
      #将ceph-demo设置为rbd类型：
      # ceph osd pool application enable ceph-demo rbd
      
      # 挂载到本地
      # rbd map ceph-demo/rbd-demo.img
      /dev/rbd0
      ]# mkfs.xfs /dev/rbd0
      meta-data=/dev/rbd0              isize=512    agcount=16, agsize=1638400 blks
               =                       sectsz=512   attr=2, projid32bit=1
               =                       crc=1        finobt=0, sparse=0
      data     =                       bsize=4096   blocks=26214400, imaxpct=25
               =                       sunit=1024   swidth=1024 blks
      naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
      log      =internal log           bsize=4096   blocks=12800, version=2
               =                       sectsz=512   sunit=8 blks, lazy-count=1
      realtime =none                   extsz=4096   blocks=0, rtextents=0
      [root@node1 ceph]# mkdir /mnt/rdb-demo
      [root@node1 ceph]# mount /dev/rbd0 /mnt/rdb-demo
      
      #最后测速测一把 写入1.2G 符合ssd
      # time dd if=/dev/zero of=test.dbf bs=8k count=300000
      300000+0 records in
      300000+0 records out
      2457600000 bytes (2.5 GB) copied, 2.06784 s, 1.2 GB/s
      
      real    0m2.072s
      user    0m0.318s
      sys     0m1.742s
      
      #同时也可以看到文件落在三个ssd上
      # ceph osd map ceph-demo rbd-demo.img
      osdmap e81 pool 'ceph-demo' (2) object 'rbd-demo.img' -> pg 2.7d92bf55 (2.15) -> up ([6,8,7], p6) acting ([6,8,7], p6)
      ```

   9. 禁用掉禁用在增加或者修改osd的时候自动分配curshmap，否在在重启服务或者添加osd的时候 会有大量的pg迁移。出现事故,在配置文件中添加osd crush update on start = false 添加了这个参数后，后面加了osd后 需要手动将osd添加到curshmap规则里 重新应用

      ```
      # vim ceph.conf 
      ...
      [osd]
      osd crush update on start = false
      ...
      
      #推送到其他节点
      # ceph-deploy --overwrite-conf config push node1 node2 node3
      [ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
      [ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy --overwrite-conf config push node1 node2 node3
      [ceph_deploy.cli][INFO  ] ceph-deploy options:
      [ceph_deploy.cli][INFO  ]  username                      : None
      [ceph_deploy.cli][INFO  ]  verbose                       : False
      [ceph_deploy.cli][INFO  ]  overwrite_conf                : True
      [ceph_deploy.cli][INFO  ]  subcommand                    : push
      [ceph_deploy.cli][INFO  ]  quiet                         : False
      [ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7f1e65ed7b00>
      [ceph_deploy.cli][INFO  ]  cluster                       : ceph
      [ceph_deploy.cli][INFO  ]  client                        : ['node1', 'node2', 'node3']
      [ceph_deploy.cli][INFO  ]  func                          : <function config at 0x7f1e667c0c08>
      [ceph_deploy.cli][INFO  ]  ceph_conf                     : None
      [ceph_deploy.cli][INFO  ]  default_release               : False
      [ceph_deploy.config][DEBUG ] Pushing config to node1
      [node1][DEBUG ] connected to host: node1 
      [node1][DEBUG ] detect platform information from remote host
      [node1][DEBUG ] detect machine type
      [node1][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
      [ceph_deploy.config][DEBUG ] Pushing config to node2
      [node2][DEBUG ] connected to host: node2 
      [node2][DEBUG ] detect platform information from remote host
      [node2][DEBUG ] detect machine type
      [node2][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
      [ceph_deploy.config][DEBUG ] Pushing config to node3
      [node3][DEBUG ] connected to host: node3 
      [node3][DEBUG ] detect platform information from remote host
      [node3][DEBUG ] detect machine type
      [node3][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
      
      #重启osd 三台节点全部重启
      # systemctl restart ceph-osd.target
      ```

      

      # 注意事项

      * 在扩容和删除osd还有编辑curshmap的时候最好备份一个

      * 初始化就规划好curshmap的规则，否则应用过程中改变的话，就会有大量的pg迁移

      * 重启服务的时候就会自动修改curshmap，所以需要备份，也可以禁用在增加或者修改osd的时候自动分配curshmap

        

