# 四、集群运维

## 1. Master节点
#### 增加master节点
```bash
# 1.编辑hosts.yaml，增加master节点配置
$ vi inventory/mycluster/hosts.yaml
# 2.执行cluster.yml（不要用scale.yml）
$ ansible-playbook -i inventory/mycluster/hosts.yaml cluster.yml -b -v
# 3.重启nginx-proxy - 在所有节点执行下面命令重启nginx-proxy 因为nginx里写了所有node节点ip的配置为恩建
$ crictl ps | grep k8s_nginx-proxy_nginx-proxy | awk '{print $1}' | xargs docker restart
```
#### 删除master节点
`如果你要删除的是配置文件中第一个节点，需要先调整配置，将第一行配置下移，再重新运行cluster.yml，使其变成非第一行配置。举例如下：`
```bash
# 场景：下线node-1节点
$ vi inventory/mycluster/hosts.yaml
# 变更前的配置
  children:
    kube-master:
      hosts:
        node-1:
        node-2:
        node-3:
# 变更后的配置
  children:
    kube-master:
      hosts:
        node-2:
        node-1:
        node-3:
# 再执行一次cluster.yml
$ ansible-playbook -i inventory/mycluster/hosts.yaml -b cluster.yml
```
非第一行的master节点下线流程：
```bash
# 执行remove-node.yml(不要在hosts.yaml中删除要下线的节点)
$ ansible-playbook -i inventory/mycluster/hosts.yaml remove-node.yml -b -v -e "node=NODE-NAME"
# 同步hosts.yaml（编辑hosts.yaml将下线的节点删除，保持集群状态和配置文件的一致性）
$ vi inventory/mycluster/hosts.yaml
```


## 2. Worker节点
#### 增加worker节点
```bash
# 刷新缓存
$ ansible-playbook -i inventory/mycluster/hosts.yaml facts.yml -b -v
# 修改配置hosts.yaml，增加节点
$ vi inventory/mycluster/hosts.yaml
# 执行scale添加节点，--limit限制只在某个固定节点执行
$ ansible-playbook -i inventory/mycluster/hosts.yaml scale.yml --limit=NODE-NAME -b -v
```
#### 删除worker节点
```bash
# 此命令可以下线节点，不影响其他正在运行中的节点，并清理节点上所有的容器以及kubelet，恢复初始状态，多个节点逗号分隔
$ ansible-playbook -i inventory/mycluster/hosts.yaml remove-node.yml -b -v -e "node=NODE-NAME-1,NODE-NAME-2,..."
# 同步hosts.yaml（编辑hosts.yaml将下线的节点删除，保持集群状态和配置文件的一致性）
$ vi inventory/mycluster/hosts.yaml
```
## 3. ETCD节点
`如果要变更的etcd节点同时也是master或worker节点，需要先将master/worker节点按照前面的文档操作下线，保留纯粹的etcd节点`

#### 增加etcd节点
```bash
# 编辑hosts.yaml（可以增加1个或2个etcd节点配置）
$ vi inventory/mycluster/hosts.yaml
# 更新etcd集群
$ ansible-playbook -i inventory/mycluster/hosts.yaml upgrade-cluster.yml --limit=etcd,kube-master -e ignore_assert_errors=yes -e etcd_retries=10
```

#### 删除etcd节点
```bash
# 执行remove-node.yml(不要在hosts.yaml中删除要下线的节点)
$ ansible-playbook -i inventory/mycluster/hosts.yaml remove-node.yml -b -v -e "node=NODE-NAME"
# 同步hosts.yaml（编辑hosts.yaml将下线的节点删除，保持集群状态和配置文件的一致性）
$ vi inventory/mycluster/hosts.yaml
# 运行cluster.yml给node节点重新生成etcd节点相关的配置
$ ansible-playbook -i inventory/mycluster/hosts.yaml -b cluster.yml
```

## 4. 其他常用命令
#### 集群reset
```bash
# 运行reset.yml一键清理集群
$ ansible-playbook -i inventory/mycluster/hosts.yaml -b -v reset.yml
```
#### 自定义play起始点
当我们执行play的过程中如果有问题，需要重新的时候，如果重新执行指令会重新经历前面漫长的等待，这个时候“跳过”功能就显得非常有用
```bash
# 通过--start-at-task指定从哪个task处开始执行，会跳过前面的任务，举例如下
$ ansible-playbook --start-at-task="reset | gather mounted kubelet dirs"
```
#### 忽略错误
当有些错误是我们确认可以接受的或误报的，可以配置ignore_errors: true，避免task出现错误后影响整个流程的执行。
```bash
# 示例片段如下：
- name: "Remove physical volume from cluster disks."
  environment:
    PATH: "{{ ansible_env.PATH }}:/sbin"
  become: true
  command: "pvremove {{ disk_volume_device_1 }} --yes"
  ignore_errors: true
```