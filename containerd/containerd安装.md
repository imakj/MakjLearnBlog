# 官网

https://containerd.io/

https://github.com/containerd

https://github.com/containerd/containerd

* 最小发行版:[containerd-1.5.0-linux-amd64.tar.gz](https://github.com/containerd/containerd/releases/download/v1.5.0/containerd-1.5.0-linux-amd64.tar.gz)

* 包含cri和cni标准的发行版：[cri-containerd-cni-1.5.0-linux-amd64.tar.gz](https://github.com/containerd/containerd/releases/download/v1.5.0/cri-containerd-cni-1.5.0-linux-amd64.tar.gz)

# 安装

1. 下载带有cri标准的版本,注意 目前1.5版本没有crtctl 官方已经修复：https://github.com/containerd/containerd/pull/5462

   ```
   [root@crontainerd ~]# mkdir /opt/crontainerd
   [root@crontainerd ~]# wget https://github.com/containerd/containerd/releases/download/v1.5.0/cri-containerd-cni-1.5.0-linux-amd64.tar.gz
   ```

2. 解压

   ```
   [root@crontainerd crontainerd]# tar zxvf cri-containerd-cni-1.5.0-linux-amd64.tar.gz
   ```

3. 配置配置文件

   ```
   #拷贝命令
   [root@crontainerd crontainerd]# cp -r usr/ /
   [root@crontainerd crontainerd]# cp -r etc/ /
   
   #生产配置文件写入到默认配置文件位置
   [root@crontainerd crontainerd]# mkdir /etc/containerd
   [root@crontainerd crontainerd]# containerd config default > /etc/containerd/config.toml
   [root@crontainerd crontainerd]# vim /etc/containerd/config.toml
   
   disabled_plugins = []
   imports = []
   oom_score = 0    #在系统内存不足的 越小越不容易被kill掉 可以改为-999
   plugin_dir = ""
   required_plugins = []
   root = "/var/lib/containerd"   #根目录 可以修改为大磁盘的目录
   state = "/run/containerd"   
   version = 2
   
   [cgroup]
     path = ""
   
   [debug]
     address = ""
     format = ""
     gid = 0
     level = ""
     uid = 0
   
   [grpc]
     address = "/run/containerd/containerd.sock"
     gid = 0
     max_recv_message_size = 16777216
     max_send_message_size = 16777216
     tcp_address = ""
     tcp_tls_cert = ""
     tcp_tls_key = ""
     uid = 0
   
   [metrics]
     address = ""
     grpc_histogram = false
   
   [plugins]
   
     [plugins."io.containerd.gc.v1.scheduler"]
       deletion_threshold = 0
       mutation_threshold = 100
       pause_threshold = 0.02
       schedule_delay = "0s"
       startup_delay = "100ms"
   
     [plugins."io.containerd.grpc.v1.cri"]
       disable_apparmor = false
       disable_cgroup = false
       disable_hugetlb_controller = true
       disable_proc_mount = false
       disable_tcp_service = true
       enable_selinux = false
       enable_tls_streaming = false
       ignore_image_defined_volumes = false
       max_concurrent_downloads = 3
       max_container_log_line_size = 16384
       netns_mounts_under_state_dir = false
       restrict_oom_score_adj = false
       sandbox_image = "k8s.gcr.io/pause:3.5"
       selinux_category_range = 1024
       stats_collect_period = 10
       stream_idle_timeout = "4h0m0s"
       stream_server_address = "127.0.0.1"
       stream_server_port = "0"
       systemd_cgroup = false
       tolerate_missing_hugetlb_controller = true
       unset_seccomp_profile = ""
   
       [plugins."io.containerd.grpc.v1.cri".cni]
         bin_dir = "/opt/cni/bin"
         conf_dir = "/etc/cni/net.d"
         conf_template = ""
         max_conf_num = 1
   
       [plugins."io.containerd.grpc.v1.cri".containerd]
         default_runtime_name = "runc"
         disable_snapshot_annotations = true
         discard_unpacked_layers = false
         no_pivot = false
         snapshotter = "overlayfs"
   
         [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime]
           base_runtime_spec = ""
           container_annotations = []
           pod_annotations = []
           privileged_without_host_devices = false
           runtime_engine = ""
           runtime_root = ""
           runtime_type = ""
   
           [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime.options]
   
         [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
   
           [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
             base_runtime_spec = ""
             container_annotations = []
             pod_annotations = []
           privileged_without_host_devices = false
           runtime_engine = ""
           runtime_root = ""
           runtime_type = ""
   
           [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime.options]
   
         [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
   
           [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
             base_runtime_spec = ""
             container_annotations = []
             pod_annotations = []
             privileged_without_host_devices = false
             runtime_engine = ""
             runtime_root = ""
             runtime_type = "io.containerd.runc.v2"
   
             [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
               BinaryName = ""
               CriuImagePath = ""
               CriuPath = ""
               CriuWorkPath = ""
               IoGid = 0
               IoUid = 0
               NoNewKeyring = false
               NoPivotRoot = false
               Root = ""
               ShimCgroup = ""
               SystemdCgroup = false
   
         [plugins."io.containerd.grpc.v1.cri".containerd.untrusted_workload_runtime]
           base_runtime_spec = ""
           container_annotations = []
           pod_annotations = []
           privileged_without_host_devices = false
           runtime_engine = ""
           runtime_root = ""
           runtime_type = ""
   
           [plugins."io.containerd.grpc.v1.cri".containerd.untrusted_workload_runtime.options]
   
       [plugins."io.containerd.grpc.v1.cri".image_decryption]
         key_model = "node"
   
       [plugins."io.containerd.grpc.v1.cri".registry]
         config_path = ""
   
         [plugins."io.containerd.grpc.v1.cri".registry.auths]
   
         [plugins."io.containerd.grpc.v1.cri".registry.configs]
   
         [plugins."io.containerd.grpc.v1.cri".registry.headers]
   
         [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
   
       [plugins."io.containerd.grpc.v1.cri".x509_key_pair_streaming]
         tls_cert_file = ""
         tls_key_file = ""
   
     [plugins."io.containerd.internal.v1.opt"]
       path = "/opt/containerd"
   
     [plugins."io.containerd.internal.v1.restart"]
       interval = "10s"
   
     [plugins."io.containerd.metadata.v1.bolt"]
       content_sharing_policy = "shared"
   
     [plugins."io.containerd.monitor.v1.cgroups"]
       no_prometheus = false
   
     [plugins."io.containerd.runtime.v1.linux"]
       no_shim = false
       runtime = "runc"
       runtime_root = ""
       shim = "containerd-shim"
       shim_debug = false
   
     [plugins."io.containerd.runtime.v2.task"]
       platforms = ["linux/amd64"]
   
     [plugins."io.containerd.service.v1.diff-service"]
       default = ["walking"]
   
     [plugins."io.containerd.snapshotter.v1.aufs"]
       root_path = ""
   
     [plugins."io.containerd.runtime.v2.task"]
       platforms = ["linux/amd64"]
   
     [plugins."io.containerd.service.v1.diff-service"]
       default = ["walking"]
   
     [plugins."io.containerd.snapshotter.v1.aufs"]
       root_path = ""
   
     [plugins."io.containerd.snapshotter.v1.btrfs"]
       root_path = ""
   
     [plugins."io.containerd.snapshotter.v1.devmapper"]
       async_remove = false
       base_image_size = ""
       pool_name = ""
       root_path = ""
   
     [plugins."io.containerd.snapshotter.v1.native"]
       root_path = ""
   
     [plugins."io.containerd.snapshotter.v1.overlayfs"]
       root_path = ""
   
     [plugins."io.containerd.snapshotter.v1.zfs"]
       root_path = ""
   
   [proxy_plugins]
   
   [stream_processors]
   
     [stream_processors."io.containerd.ocicrypt.decoder.v1.tar"]
       accepts = ["application/vnd.oci.image.layer.v1.tar+encrypted"]
       args = ["--decryption-keys-path", "/etc/containerd/ocicrypt/keys"]
       env = ["OCICRYPT_KEYPROVIDER_CONFIG=/etc/containerd/ocicrypt/ocicrypt_keyprovider.conf"]
       path = "ctd-decoder"
       returns = "application/vnd.oci.image.layer.v1.tar"
   
     [stream_processors."io.containerd.ocicrypt.decoder.v1.tar.gzip"]
       accepts = ["application/vnd.oci.image.layer.v1.tar+gzip+encrypted"]
       args = ["--decryption-keys-path", "/etc/containerd/ocicrypt/keys"]
       env = ["OCICRYPT_KEYPROVIDER_CONFIG=/etc/containerd/ocicrypt/ocicrypt_keyprovider.conf"]
       path = "ctd-decoder"
       returns = "application/vnd.oci.image.layer.v1.tar+gzip"
   
   [timeouts]
     "io.containerd.timeout.shim.cleanup" = "5s"
     "io.containerd.timeout.shim.load" = "5s"
     "io.containerd.timeout.shim.shutdown" = "3s"
     "io.containerd.timeout.task.state" = "2s"
   
   [ttrpc]
     address = ""
     gid = 0
     uid = 0
                       
   #启动
   [root@crontainerd crontainerd]# systemctl restart containerd
   [root@crontainerd crontainerd]# systemctl status containerd
   ● containerd.service - containerd container runtime
      Loaded: loaded (/etc/systemd/system/containerd.service; disabled; vendor preset: disabled)
      Active: active (running) since Thu 2021-05-06 23:18:32 CST; 3s ago
        Docs: https://containerd.io
     Process: 1597 ExecStartPre=/sbin/modprobe overlay (code=exited, status=0/SUCCESS)
    Main PID: 1601 (containerd)
       Tasks: 11
      Memory: 13.9M
      CGroup: /system.slice/containerd.service
              └─1601 /usr/local/bin/containerd
   
   May 06 23:18:32 crontainerd containerd[1601]: time="2021-05-06T23:18:32.643856880+08:00" l...v1
   May 06 23:18:32 crontainerd containerd[1601]: time="2021-05-06T23:18:32.644022066+08:00" l...t"
   May 06 23:18:32 crontainerd containerd[1601]: time="2021-05-06T23:18:32.644099844+08:00" l...e"
   May 06 23:18:32 crontainerd containerd[1601]: time="2021-05-06T23:18:32.644149366+08:00" l...pc
   May 06 23:18:32 crontainerd containerd[1601]: time="2021-05-06T23:18:32.644197070+08:00" l...ck
   May 06 23:18:32 crontainerd containerd[1601]: time="2021-05-06T23:18:32.644222396+08:00" l...r"
   May 06 23:18:32 crontainerd containerd[1601]: time="2021-05-06T23:18:32.644247760+08:00" l...r"
   May 06 23:18:32 crontainerd containerd[1601]: time="2021-05-06T23:18:32.644259732+08:00" l...r"
   May 06 23:18:32 crontainerd containerd[1601]: time="2021-05-06T23:18:32.644267345+08:00" l...r"
   May 06 23:18:32 crontainerd containerd[1601]: time="2021-05-06T23:18:32.644268215+08:00" l...s"
   Hint: Some lines were ellipsized, use -l to show in full.
   
   ```

# ctr命令

```
# 查看镜像
[root@crontainerd crontainerd]# ctr i ls

#pull 一个镜像 必须是仓库地址+镜像名
[root@crontainerd crontainerd]# ctr i pull docker.io/library/nginx:1.7.9

#查看命名空间 如果有k8s的话 就会有一个k8s.io的命名空间
[root@crontainerd crontainerd]# ctr ns ls
NAME    LABELS 
default

#如果同时存在docker 就会出现docker的命名空间moby
[root@crontainerd containerd]# ctr ns ls
NAME    LABELS 
default        
moby     

#根据命名空间查看容器
[root@crontainerd containerd]# ctr -n default i ls
REF                           TYPE                                       DIGEST                                                                  SIZE     PLATFORMS   LABELS 
docker.io/library/nginx:1.7.9 application/vnd.oci.image.manifest.v1+json sha256:3ca56ebcfdbc7f6e23db3800cc489fcf597a896e1fc8f7f0919ebc0d5c97f418 38.1 MiB linux/amd64 - 

#这里查看docker的看不到，因为两个容器存储的目录不一样  如果需要一样 就要把当前docker的镜像推送到docker仓库然后用containerd拉下来
containerd]# ctr -n moby i ls
REF TYPE DIGEST SIZE PLATFORMS LABELS 
[root@crontainerd containerd]# ll /var/lib/containerd/
total 0
drwxr-xr-x 4 root root 33 May  6 23:22 io.containerd.content.v1.content
drwx--x--x 2 root root 21 May  6 23:18 io.containerd.metadata.v1.bolt
drwx--x--x 2 root root  6 May  6 23:18 io.containerd.runtime.v1.linux
drwx--x--x 2 root root  6 May  6 23:18 io.containerd.runtime.v2.task
drwxr-xr-x 2 root root  6 May  6 23:18 io.containerd.snapshotter.v1.btrfs
drwx------ 3 root root 23 May  6 23:18 io.containerd.snapshotter.v1.native
drwx------ 3 root root 42 May  6 23:22 io.containerd.snapshotter.v1.overlayfs
drwx------ 2 root root  6 May  6 23:22 tmpmounts
[root@crontainerd containerd]# ll /var/lib/docker/
total 4
drwx--x--x  4 root root  120 May  6 23:25 buildkit
drwx-----x  2 root root    6 May  6 23:25 containers
drwx------  3 root root   22 May  6 23:25 image
drwxr-x---  3 root root   19 May  6 23:25 network
drwx-----x 16 root root 4096 May  6 23:26 overlay2
drwx------  4 root root   32 May  6 23:25 plugins
drwx------  2 root root    6 May  6 23:25 runtimes
drwx------  2 root root    6 May  6 23:25 swarm
drwx------  2 root root    6 May  6 23:26 tmp
drwx------  2 root root    6 May  6 23:25 trust
drwx-----x  2 root root   50 May  6 23:25 volumes


#启动一个容器 同样需要全称
[root@crontainerd containerd]# ctr run -t -d docker.io/library/nginx:1.7.9 nginx

#查看容器
[root@crontainerd containerd]# ctr c ls
CONTAINER    IMAGE                            RUNTIME                  
nginx        docker.io/library/nginx:1.7.9    io.containerd.runc.v2 
#任务的方式查看
[root@crontainerd containerd]# ctr t ls
TASK     PID     STATUS    
nginx    2195    RUNNING

#杀掉一个容器
[root@crontainerd containerd]# ctr t kill nginx
[root@crontainerd containerd]# ctr t ls
TASK     PID     STATUS    
nginx    2195    STOPPED

#删除一个容器任务
[root@crontainerd containerd]# ctr t rm nginx
#查看 容器还在 
[root@crontainerd containerd]# ctr c ls
CONTAINER    IMAGE                            RUNTIME                  
nginx        docker.io/library/nginx:1.7.9    io.containerd.runc.v2   
#删除容器
[root@crontainerd containerd]# ctr c rm nginx
[root@crontainerd containerd]# ctr c ls

```

docker和containerd在ctr的本质上只是命名空间的不一样

# crtctl 命令

crtctl的命令和docker相似。为了配合k8s而存在的,crtctl默认使用k8s.io命名空间





