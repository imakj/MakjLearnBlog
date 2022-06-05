# 准备

jenkins.zip

# 安装

1. 解压

   ```
   # cd /opt/jenkins
   # unzip jenkins
   ```

2. k8s部署 注意本地卷的挂载 这里挂载的hostpath 需要在每个节点创建/data/jenkins并且给与权限

   ```
   # vim jenkins.yml 
   ……
   volumes:
         - name: jenkins-home
           hostPath:
             path: /data/jenkins
             type: Directory
   ……
   # kubectl apply -f jenkins.yml 
   ```

3. 等待一会进入

   ```
   # kubectl get pods,svc
   NAME                                     READY   STATUS    RESTARTS       AGE
   pod/httpbin-7f678fdb89-dbvpw             2/2     Running   2 (4h2m ago)   4d5h
   pod/jenkins-86d45d8c76-g2kmq             1/1     Running   4 (4m ago)     4m42s
   pod/metrics-flask-app-66c9995b58-ngtm4   1/1     Running   3 (4h2m ago)   16d
   pod/mychar-webtest8-86b4f99b6b-wsh2w     1/1     Running   3 (4h2m ago)   14d
   pod/nginx-hpa-588d4dccb6-568ns           1/1     Running   3 (4h2m ago)   16d
   pod/nginx-hpa-588d4dccb6-z4zzh           1/1     Running   3 (4h2m ago)   16d
   pod/nginx-mychart-55f9579b65-hjs8l       1/1     Running   3 (4h2m ago)   15d
   pod/nginxtest-fc5f8b88-md9nb             1/1     Running   4 (4h2m ago)   17d
   pod/webtest-9f8cc89f7-nctrv              1/1     Running   3 (4h2m ago)   14d
   
   NAME                        TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                        AGE
   service/httpbin             NodePort    10.0.0.146   <none>        8000:31576/TCP                 4d5h
   service/jenkins             NodePort    10.0.0.52    <none>        80:30008/TCP,50000:31889/TCP   4m42s
   service/kubernetes          ClusterIP   10.0.0.1     <none>        443/TCP                        22d
   service/metrics-flask-app   ClusterIP   10.0.0.114   <none>        80/TCP                         16d
   service/mychar-webtest8     ClusterIP   10.0.0.169   <none>        80/TCP                         14d
   service/nginx-hpa           ClusterIP   10.0.0.75    <none>        18080/TCP                      16d
   service/nginx-mychart       ClusterIP   10.0.0.59    <none>        80/TCP                         15d
   service/webtest             ClusterIP   10.0.0.184   <none>        80/TCP                         14d
   
   # curl http://192.168.10.170:30008/login?from=%2F
   ```

4.  查看密码

   ```
   # kubectl get pods -o wide | grep jenkins
   jenkins-86d45d8c76-g2kmq             1/1     Running   4 (5m19s ago)   6m1s   10.244.121.10    k8s-manual-05   <none>           <none>
   
   # 因为jenkins home是挂载在/data/jenkins 故目录是下面的
   # cat /data/jenkins/secrets/initialAdminPassword 
   5e80d59d4b15451fac2ab5d32496bbb5
   ```

5. 选择 “选择插件安装” 不要推荐安装 然后进去后点击无 最后自己安装插件

6. 设置管理员账号密码admin1/admin 完成

7. 修改源为清华源

   ```
   # cd /data/jenkins/updates/
   # sed -i 's/http:\/\/updates.jenkins-ci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' default.json && sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' default.json
   
   # 重启
   http://192.168.10.170:30008/restart
   ```

8. 安装插件

   * git
   * git parmeter
   * pipeline
   * kubernetes
   * config file provider

   安装完成后重启

