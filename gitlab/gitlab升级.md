1. 备份代码

   ```
   # gitlab-rake gitlab:backup:create
   # ll /var/opt/gitlab/backups/
   total 3060996
   -rw------- 1 gitlab gitlab 3134453760 Nov  9 20:11 1636459899_2021_11_09_12.3.5_gitlab_backup.tar
   ```

2. 备份配置文件

   ```
   # cd /etc/gitlab/
   # cp gitlab* /opt/gitbackup/
   # ll /opt/gitbackup/
   total 112
   -rw------- 1 root root 95390 Nov  9 20:17 gitlab.rb
   -rw------- 1 root root 15615 Nov  9 20:17 gitlab-secrets.json
   
   #可以添加一下权限
   gitlab_rails[‘manage_backup_path’] = true
   gitlab_rails[‘backup_path’] = “/data/gitlab/backups” -----------------------//gitlab备份目录
   gitlab_rails[‘backup_archive_permissions’] = 0644 ------------------------//生成的备份文件权限
   gitlab_rails[‘backup_keep_time’] = 604800 ----------------------------------//备份保留天数为7天
   ```

3. 修改源

   ```
   # cat /etc/yum.repos.d/gitlab-ce.repo
   [gitlab-ce]
   name=gitlab-ce
   baseurl=https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/
   repo_gpgcheck=0
   gpgcheck=0
   enable=1
   gpgkey=https://packages.gitlab.com/gpg.key
   
   ```

   

4. 大版本跨版本需要逐渐升级,大版本可以从仓库看(https://mirror.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/)

   ```
   # yum install -y gitlab-ce-12.10.9
   # yum install -y gitlab-ce-13.0.0
   # yum install -y gitlab-ce-13.12.15
   
   #查看版本
   # cat /opt/gitlab/embedded/service/gitlab-rails/VERSION
   ```

   

5. 重载配置文件并且启动

   ```
   # gitlab-ctl reconfigure
   # gitlab-ctl restart
   ```

   

