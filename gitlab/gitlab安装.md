1. 下载安装包

<https://about.gitlab.com/install/>

![1585821577476](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/1585821577476.png?x-oss-process=style/makjwatermarks)

<https://mirror.tuna.tsinghua.edu.cn/>

![1585821777344](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/1585821777344.png?x-oss-process=style/makjwatermarks)

![1585821899745](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/1585821899745.png?x-oss-process=style/makjwatermarks)

![1585835665431](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/1585835665431.png?x-oss-process=style/makjwatermarks)

1. 安装gitlab服务所依赖包

   ```
   sudo yum install -y curl policycoreutils-python openssh-server postfix wget
   ```

2. 下载gitlab服务，安装gitlab服务

   ```
   sudo yum localinstall -y gitlab-ce-12.9.2-ce.0.el7.x86_64.rpm
   ```

3. 配置gitlab服务，访问域名以及邮箱

   ```bash
   vi /etc/gitlab/gitlab.rb
   external_url 'http://gitlab.yuanke.com'
   
   # 配置邮箱
   gitlab_rails['gitlab_email_enabled'] = true
   gitlab_rails['gitlab_email_from'] = '530979104@qq.com' #发送邮箱
   gitlab_rails['gitlab_email_display_name'] = 'yuanke-gitlab' #发送人显示名称
   
    
    
   gitlab_rails['smtp_enable'] = true
   gitlab_rails['smtp_address'] = "smtp.qq.com"
   gitlab_rails['smtp_port'] = 465
   gitlab_rails['smtp_user_name'] = "530979104@qq.com"
   gitlab_rails['smtp_password'] = "hqvzmkwywndwbiah"
   gitlab_rails['smtp_domain'] = "qq.com"
   gitlab_rails['smtp_authentication'] = "login"
   gitlab_rails['smtp_enable_starttls_auto'] = true
   gitlab_rails['smtp_tls'] = true
   
   # 修改
   prometheus['enable'] = false
   
   unicorn['port'] = 8101
   ```

   ![1585824852368](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/1585824852368.png?x-oss-process=style/makjwatermarks)

4. 初始化gitlab服务，启动gitlab服务

```
gitlab-ctl reconfigure
gitlab-ctl start | status | restart | stop
```

![1585831361763](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/1585831361763.png?x-oss-process=style/makjwatermarks)

5. 访问gitlab服务，以及gitlab邮箱测试

```
host 做映射  C:\WINDOWS\system32\drivers\etc
192.168.254.208 gitlab.yuanke.com
gitlab.yuanke.com
```

![1585833036644](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/1585833036644.png?x-oss-process=style/makjwatermarks)



# gitlab 汉化

<https://gitlab.com/xhang/gitlab>



停止gitlab，进行中文汉化

```
gitlab-ctl stop
\cp -r gitlab-12-3-stable-zh/* /opt/gitlab/embedded/service/gitlab-rails/
gitlab-ctl start
```

![1586057898970](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/1586057898970.png?x-oss-process=style/makjwatermarks)

![1586057969044](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/1586057969044.png?x-oss-process=style/makjwatermarks)



# 卸载

1.停止gitlab

 ```
gitlab-ctl stop
 ```

2.卸载gitlab（卸载gitlab-ce）

```
rpm -e gitlab-ce
```

3.查看gitlab进程

```
ps aux | grep gitlab
```

![1585890639360](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/1585890639360.png?x-oss-process=style/makjwatermarks)

4.杀死第一个进程

kill -9 32236

5.删除所有gitlab文件

find / -name gitlab | xargs rm -rf



# 注意

```bash

There was an error running gitlab-ctl reconfigure:

gitlab_sysctl[kernel.shmall] (postgresql::enable line 66) had an error: Mixlib:                                                :ShellOut::ShellCommandFailed: execute[load sysctl conf kernel.shmall] (/opt/gi                                                tlab/embedded/cookbooks/cache/cookbooks/package/resources/gitlab_sysctl.rb line                                                 46) had an error: Mixlib::ShellOut::ShellCommandFailed: Expected process to ex                                                it with [0], but received '255'
---- Begin output of sysctl -e --system ----
STDOUT: * Applying /usr/lib/sysctl.d/00-system.conf ...
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-arptables = 0
* Applying /usr/lib/sysctl.d/50-default.conf ...
kernel.sysrq = 16
kernel.core_uses_pid = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.promote_secondaries = 1
net.ipv4.conf.all.promote_secondaries = 1
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
* Applying /etc/sysctl.d/90-omnibus-gitlab-kernel.sem.conf ...
* Applying /etc/sysctl.d/90-omnibus-gitlab-kernel.shmall.conf ...
kernel.shmall = 4194304
* Applying /etc/sysctl.d/90-omnibus-gitlab-kernel.shmmax.conf ...
kernel.shmmax = 17179869184
* Applying /etc/sysctl.d/90-omnibus-gitlab-net.core.somaxconn.conf ...
* Applying /etc/sysctl.d/99-sysctl.conf ...
* Applying /etc/sysctl.conf ...
STDERR: sysctl: cannot open "/etc/sysctl.d/90-omnibus-gitlab-kernel.sem.conf":                                                 No such file or directory
sysctl: cannot open "/etc/sysctl.d/90-omnibus-gitlab-net.core.somaxconn.conf":                                                 No such file or directory
---- End output of sysctl -e --system ----
Ran sysctl -e --system returned 255

```

[root@s208 ~]# touch /opt/gitlab/embedded/etc/90-omnibus-gitlab-kernel.sem.conf
[root@s208 ~]# touch /opt/gitlab/embedded/etc/90-omnibus-gitlab-kernel.shmall.conf
[root@s208 ~]# touch /opt/gitlab/embedded/etc/90-omnibus-gitlab-net.core.somaxconn.conf



3. 编写gitlab

   ```
   Host github
   HostName github.com
   User git
   IdentityFile ~/.ssh/id_rsa
   
   
   Host coding
   HostName coding.net
   User git
   IdentityFile ~/.ssh/id_rsa_coding
   
   Host gitlab
   HostName gitlab.yuanke.com
   User git
   IdentityFile ~/.ssh/id_rsa_gitlab
   
   ```

   $ ssh-agent.exe bash

   $ ssh-add.exe ~/.ssh/id_rsa_gitlab

4. 修改完配置后需要重新加载下

   ```
   # gitlab-ctl reconfigure
   ```

   

# 备份

1. 默认备份配置文件在: vim /etc/gitlab/gitlab.rb

   ```
   # vim /etc/gitlab/gitlab.rb 
   
   
   # gitlab_rails['manage_backup_path'] = true
   # gitlab_rails['backup_path'] = "/var/opt/gitlab/backups"  #备份目录
   
   ###! Docs: https://docs.gitlab.com/ce/raketasks/backup_restore.html#backup-archive-permissions
   # gitlab_rails['backup_archive_permissions'] = 0644
   
   # gitlab_rails['backup_pg_schema'] = 'public'
   
   ###! The duration in seconds to keep backups before they are allowed to be deleted
   # gitlab_rails['backup_keep_time'] = 604800  #默认保存时间 7天
   
   # gitlab_rails['backup_upload_connection'] = {
   #   'provider' => 'AWS',
   #   'region' => 'eu-west-1',
   #   'aws_access_key_id' => 'AKIAKIAKI',
   #   'aws_secret_access_key' => 'secret123'
   # }
   # gitlab_rails['backup_upload_remote_directory'] = 'my.s3.bucket'
   # gitlab_rails['backup_multipart_chunk_size'] = 104857600
   
   ```

2. 执行备份命令

   ```
   # gitlab-rake gitlab:backup:create
   ```

3. 可以看到文件会存到:/var/opt/gitlab/backups

   ```
   # cd /var/opt/gitlab/backups
   # ll
   total 2402464
   -rw------- 1 gitlab gitlab 2460119040 Jun 20 21:47 1624196864_2021_06_20_12.3.5_gitlab_backup.tar
   ```

4. 恢复

   1. 停止数据写入服务

      ```
      # gitlab-ctl stop unicorn
      # gitlab-ctl  stop sidekiq
      ```

   2.  通过gitlab-rake命令进行恢复，恢复时需要指定此前备份的名称。（单不需要写后缀的.tar后缀）

      ```
      # gitlab-rake gitlab:backup:restore BACKUP=1624196864_2021_06_20_12.3.5_gitlab_backup
      ```

   3. 为了保险起见，重启gitlab，简称是否恢复

      ```
      # gitlab-ctl restart
      ```

      

   

