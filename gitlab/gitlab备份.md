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

   

