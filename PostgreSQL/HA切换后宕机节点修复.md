1. 确定需要修复的主库已经停止

   ```
   $ pg_ctl stop -m fast
   $ pg_controldata
   ```

2. 修改新主库配置文件 添加权限信息

   ```
   $ vim pg_hba.conf
   host replication repuser 0.0.0.0/0 md5
   ```

3. 注释掉新主库的同步信息(以前作为从库的同步信息)

   ```
   $ vim postgresql.conf
   #注释掉此行
   #primary_conninfo = 'host=10.149.229.9 port=5432 user=repluser passowrd=repuser123'
   ```

4. 删除备份宕机节点的原始的pgdata目录

   ```
   $ rm -rf /data/postgresql/pgdata/*
   ```

5. 新从库执行同步信息

   ```
   $ pg_basebackup -D /data/postgresql/pgdata -F p -P -R -h 10.149.229.10 -p 5432 -U repluser -l backup20210227
   ```

6. 新从库修改同步信息

   ```
   $ cd /data/postgresql/pgdata/
   $ vim postgresql.conf
   #添加此行
   primary_conninfo = 'host=10.149.229.10 port=5432 user=repluser passowrd=repuser123'
   ```

7. 启动新的备库

   ```
   $ /usr/local/postgresql/bin/pg_ctl -D /data/postgresql/pgdata start
   ```

8. 查看新从库节点信息

   ```
   $ pg_controldata
   ```

9. 检查keepalived状态

   ```
   # systemctl status keepalived
   # systemctl restart keepalived
   ```

   

10. 检查同步状态和数据写入状态，检查监控监本是否正常

    ```
    # tail -f /home/pgsql/postgresql_monitor.log
    
    ```

    

