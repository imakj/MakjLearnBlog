1. 参考文档:https://kubernetes.github.io/ingress-nginx/

2. 可以采用ds来管理ingress 给需要部署ingress的node打一个标签来不是ingress 

3. 对外暴露tcp端口服务

   ```
   # vim tcp-config.yaml
   # 创建一个ConfigMap 来暴露
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: tcp-services
     namespace: ingress-nginx
   data:
     "30000": dev/web-demo:80  #暴露一个30000的端口 对应的服务是dev命名空间下的web-demo
   
   ```

4. 定义nginx ingress的配置 需要配置一个configmap 来定义nginx的全局配置

   ```
   # 这样就会自己吧配置加到nginx里
   # vim nginx-config.yaml
   
   kind: ConfigMap
   apiVersion: v1
   metadata:
     name: nginx-configuration
     namespace: ingress-nginx
     labels:
       app: ingress-nginx
   data:
     proxy-body-size: "64m"
     proxy-read-timeout: "180"
     proxy-send-timeout: "180"
   
   ```

5. 还可以自定义header

   ```
   # vim custom-header-global.yaml
   
   apiVersion: v1
   kind: ConfigMap
   data:
     proxy-set-headers: "ingress-nginx/custom-headers" #添加一个 proxy-set-headers 使用ingress-nginx命名空间下的custom-headers 这个custom-headers就下面定义的custom-headers
   metadata:
     name: nginx-configuration
     namespace: ingress-nginx
     labels:
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
   ---
   apiVersion: v1
   kind: ConfigMap
   data:   #添加header的配置
     X-Different-Name: "true"
     X-Request-Start: t=${msec}
     X-Using-Nginx-Controller: "true"
   metadata:
     name: custom-headers  #定义一个ingress-nginx命名空间下的名字叫 custom-headers的配置
     namespace: ingress-nginx
   
   ```

6. 还可以定制某个监听的域名下的配置

   ```
   # custom-header-spec-ingress.yaml
   
   apiVersion: extensions/v1beta1
   kind: Ingress
   metadata:
     annotations:  #定义一个annotations 内容就是nginx.ingress.kubernetes.io/configuration-snippet（这个是定值）
       nginx.ingress.kubernetes.io/configuration-snippet: |
         more_set_headers "Request-Id: $req_id";  #配置一个nginx里的参数
     name: web-demo
     namespace: dev
   spec:
     rules:
     - host: web-dev.mooc.com
       http:
         paths:
         - backend:
             serviceName: web-demo
             servicePort: 80
           path: /
   
   ```

7. 配置模板

   参考https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/custom-template/

   模板的默认位置在容器里的`/etc/nginx/template/nginx.tmpl` 也有可能在根目录`/nginx.tmpl`

   ```
   # 进入到容器 拿到这个文件
   [root@k8shardway1 ~]# kubectl exec -it nginx-ingress-747cd8c4db-d5thp  -n nginx-ingress /bin/bash
   
   #放到/opt/kubernetes/文件夹下
   
   #根据这个文件创建一个configmap
   [root@k8shardway1 kubernetes]# kubectl create cm nginx-template --from-file nginx.tmpl -n nginx-ingress 
   configmap/nginx-template created
   
   [root@k8shardway1 kubernetes]# kubectl get cm nginx-template -n nginx-ingress  -o yaml
   
   #导出nginx的deploy
   [root@k8shardway1 kubernetes]# kubectl get deploy nginx-ingress -n nginx-ingress -o yaml > nginx-ingress-controller.yaml
   
   
   #修改下这个文件
   
   [root@k8shardway1 kubernetes]# vim nginx-ingress-controller.yaml 
   
   spec:
     progressDeadlineSeconds: 600
     replicas: 2
     revisionHistoryLimit: 10
     selector:
       matchLabels:
         app: nginx-ingress
     strategy:
       rollingUpdate:
         maxSurge: 25%
         maxUnavailable: 25%
       type: RollingUpdate
     template:
       metadata:
         creationTimestamp: null
         labels:
           app: nginx-ingress
       spec:
         containers:
         - args:
           - -nginx-configmaps=$(POD_NAMESPACE)/nginx-config
           - -default-server-tls-secret=$(POD_NAMESPACE)/default-server-secret
           env:
           - name: POD_NAMESPACE
             valueFrom:
               fieldRef:
                 apiVersion: v1
                 fieldPath: metadata.namespace
           - name: POD_NAME
             valueFrom:
               fieldRef:
                 apiVersion: v1
                 fieldPath: metadata.name
           image: nginx/nginx-ingress:1.11.1
           imagePullPolicy: IfNotPresent
           name: nginx-ingress
           ports:
           - containerPort: 80
             name: http
             protocol: TCP
           - containerPort: 443
             name: https
             protocol: TCP
           - containerPort: 8081
             name: readiness-port
       matchLabels:
         app: nginx-ingress
     strategy:
       rollingUpdate:
         maxSurge: 25%
         maxUnavailable: 25%
       type: RollingUpdate
     template:
       metadata:
         creationTimestamp: null
         labels:
           app: nginx-ingress
       spec:
         containers:
         - args:
           - -nginx-configmaps=$(POD_NAMESPACE)/nginx-config
           - -default-server-tls-secret=$(POD_NAMESPACE)/default-server-secret
           volumeMounts:  #安装官网提示 加一个volumeMounts
             - mountPath: /etc/nginx/template
               name: nginx-template-volume
               readOnly: true
           env:
           - name: POD_NAMESPACE
             valueFrom:
               fieldRef:
                 apiVersion: v1
                 fieldPath: metadata.namespace
           - name: POD_NAME
             valueFrom:
               fieldRef:
                 apiVersion: v1
                 fieldPath: metadata.name
           image: nginx/nginx-ingress:1.11.1
           imagePullPolicy: IfNotPresent
           name: nginx-ingress
           ports:
           - containerPort: 80
             name: http
             protocol: TCP
           - containerPort: 443
             name: https
             protocol: TCP
           - containerPort: 8081
             name: readiness-port
             protocol: TCP
           readinessProbe:
             failureThreshold: 3
             httpGet:
               path: /nginx-ready
               port: readiness-port
               scheme: HTTP
             periodSeconds: 1
             successThreshold: 1
             timeoutSeconds: 1
           resources: {}
           securityContext:
             allowPrivilegeEscalation: true
             capabilities:
               add:
               - NET_BIND_SERVICE
               drop:
               - ALL
             runAsUser: 101
           terminationMessagePath: /dev/termination-log
           terminationMessagePolicy: File
         dnsPolicy: ClusterFirst
         restartPolicy: Always
         schedulerName: default-scheduler
         securityContext: {}
         serviceAccount: nginx-ingress
         serviceAccountName: nginx-ingress
         terminationGracePeriodSeconds: 30
         volumes: #安装官网提示 加一个volumes
           - name: nginx-template-volume
             configMap:
               name: nginx-template
               items:
               - key: nginx.tmpl
                 path: nginx.tmpl
   status:
     availableReplicas: 2
     conditions:
     - lastTransitionTime: "2021-05-10T14:37:02Z"
       lastUpdateTime: "2021-05-10T14:37:26Z"
       message: ReplicaSet "nginx-ingress-747cd8c4db" has successfully progressed.
       reason: NewReplicaSetAvailable
       status: "True"
       type: Progressing
     - lastTransitionTime: "2021-05-13T15:00:58Z"
       lastUpdateTime: "2021-05-13T15:00:58Z"
       message: Deployment has minimum availability.
       reason: MinimumReplicasAvailable
       status: "True"
       type: Available
     observedGeneration: 2
     readyReplicas: 2
     replicas: 2
     updatedReplicas: 2
   
   
   [root@k8shardway1 kubernetes]# kubectl apply -f nginx-ingress-controller.yaml
   deployment.apps/nginx-ingress configured
   
   #进入到容器查看配置文件
   [root@k8shardway1 kubernetes]# kubectl exec -it nginx-ingress-754bd6bb7-66wqb /bin/bash -n nginx-ingress
   kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.   
   nginx@nginx-ingress-754bd6bb7-66wqb:/$ cat /etc/nginx/nginx.conf 
   
   
   
   #修改一下模板 随便修改下
   [root@k8shardway1 kubernetes]# kubectl edit cm -n nginx-ingress nginx-template
   configmap/nginx-template edited
   
   #在进入到容器里查看
   ```

8. tls证书

   ```
   [root@k8shardway1 kubernetes]# vim gen-secret.sh
   
   #!/bin/bash
   
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout mooc.key -out mooc.crt -subj "/CN=*.mooc.com/O=*.mooc.com"
   
   kubectl create secret tls mooc-tls --key mooc.key --cert mooc.crt
   
   
   [root@k8shardway1 kubernetes]# vim gen-secret.sh
   [root@k8shardway1 kubernetes]# sh gen-secret.sh 
   Generating a 2048 bit RSA private key
   ............................................................................................................................+++
   ...............+++
   writing new private key to 'mooc.key'
   -----
   secret/mooc-tls created
   
   #查看多了一个secret
   [root@k8shardway1 kubernetes]# kubectl get secret
   NAME                  TYPE                                  DATA   AGE
   dbpass                Opaque                                2      16h
   default-token-d4vmc   kubernetes.io/service-account-token   3      5d16h
   mooc-tls              kubernetes.io/tls                     2      23s
   
   
   [root@k8shardway1 kubernetes]# vim nginx-ingress-controller.yaml 
   
   spec:
     progressDeadlineSeconds: 600
     replicas: 2
     revisionHistoryLimit: 10
     selector:
       matchLabels:
         app: nginx-ingress
     strategy:
       rollingUpdate:
         maxSurge: 25%
         maxUnavailable: 25%
       type: RollingUpdate
     template:
       metadata:
         creationTimestamp: null
         labels:
           app: nginx-ingress
       spec:
         containers:
         - args:
           - -nginx-configmaps=$(POD_NAMESPACE)/nginx-config
           # - -default-server-tls-secret=$(POD_NAMESPACE)/default-server-secret
           - -default-server-tls-secret=default/mooc-tls  #改成自己的tls
           env:
           - name: POD_NAMESPACE
             valueFrom:
               fieldRef:
                 apiVersion: v1
                 fieldPath: metadata.namespace
           - name: POD_NAME
             valueFrom:
               fieldRef:
                 apiVersion: v1
                 fieldPath: metadata.name
           image: nginx/nginx-ingress:1.11.1
           imagePullPolicy: IfNotPresent
           name: nginx-ingress
           ports:
           - containerPort: 80
             name: http
             protocol: TCP
           - containerPort: 443
             name: https
             protocol: TCP
           - containerPort: 8081
             name: readiness-port
       matchLabels:
         app: nginx-ingress
     strategy:
       rollingUpdate:
         maxSurge: 25%
         maxUnavailable: 25%
       type: RollingUpdate
     template:
       metadata:
         creationTimestamp: null
         labels:
           app: nginx-ingress
       spec:
         containers:
         - args:
           - -nginx-configmaps=$(POD_NAMESPACE)/nginx-config
           - -default-server-tls-secret=$(POD_NAMESPACE)/default-server-secret
           volumeMounts:  #安装官网提示 加一个volumeMounts
             - mountPath: /etc/nginx/template
               name: nginx-template-volume
               readOnly: true
           env:
           - name: POD_NAMESPACE
             valueFrom:
               fieldRef:
                 apiVersion: v1
                 fieldPath: metadata.namespace
           - name: POD_NAME
             valueFrom:
               fieldRef:
                 apiVersion: v1
                 fieldPath: metadata.name
           image: nginx/nginx-ingress:1.11.1
           imagePullPolicy: IfNotPresent
           name: nginx-ingress
           ports:
           - containerPort: 80
             name: http
             protocol: TCP
           - containerPort: 443
             name: https
             protocol: TCP
           - containerPort: 8081
             name: readiness-port
             protocol: TCP
           readinessProbe:
             failureThreshold: 3
             httpGet:
               path: /nginx-ready
               port: readiness-port
               scheme: HTTP
             periodSeconds: 1
             successThreshold: 1
             timeoutSeconds: 1
           resources: {}
           securityContext:
             allowPrivilegeEscalation: true
             capabilities:
               add:
               - NET_BIND_SERVICE
               drop:
               - ALL
             runAsUser: 101
           terminationMessagePath: /dev/termination-log
           terminationMessagePolicy: File
         dnsPolicy: ClusterFirst
         restartPolicy: Always
         schedulerName: default-scheduler
         securityContext: {}
         serviceAccount: nginx-ingress
         serviceAccountName: nginx-ingress
         terminationGracePeriodSeconds: 30
         volumes: #安装官网提示 加一个volumes
           - name: nginx-template-volume
             configMap:
               name: nginx-template
               items:
               - key: nginx.tmpl
                 path: nginx.tmpl
   status:
     availableReplicas: 2
     conditions:
     - lastTransitionTime: "2021-05-10T14:37:02Z"
       lastUpdateTime: "2021-05-10T14:37:26Z"
       message: ReplicaSet "nginx-ingress-747cd8c4db" has successfully progressed.
       reason: NewReplicaSetAvailable
       status: "True"
       type: Progressing
     - lastTransitionTime: "2021-05-13T15:00:58Z"
       lastUpdateTime: "2021-05-13T15:00:58Z"
       message: Deployment has minimum availability.
       reason: MinimumReplicasAvailable
       status: "True"
       type: Available
     observedGeneration: 2
     readyReplicas: 2
     replicas: 2
     updatedReplicas: 2
   
   
   [root@k8shardway1 kubernetes]# kubectl apply -f  nginx-ingress-controller.yaml 
   deployment.apps/nginx-ingress created
   
   #后面进入到网站可以看到是自定义证书
   
   # 在Ingress里指定需要用哪个证书
   [root@k8shardway1 kubernetes]# vim nginx-ingress-demo.yaml 
   
   apiVersion: extensions/v1beta1
   kind: Ingress
   metadata:
     name: ingress-demo
   spec:
     rules:  #后端多主机 通过host方式定义
     - host: www.imakj.com
       http:
         paths:
         - path: / #访问根目录的时候转发到下面的serverName
           backend:
             serviceName: ingress-demo  #上面启动的nginx的名字
             servicePort: 80
     tls:  #指定tls证书
       - host:
         - www.imakj.com
         secretName: mooc-tls
     backend:  #默认规则 如果上面规则没匹配到 则默认这个
       serviceName: ingress-demo  #上面启动的nginx的名字
       servicePort: 80
   
   ```

9. session保持 访问控制

   ```
   #配置一个ingress-session
   [root@k8shardway1 kubernetes]# vim nginx-ingress-demo.yaml 
   
   apiVersion: extensions/v1beta1
   kind: Ingress
   metadata:
     annotations:  #session保持 就是下面的参数 定制
       nginx.ingress.kubernetes.io/affinity: cookie
       nginx.ingress.kubernetes.io/session-cookie-hash: sha1
       nginx.ingress.kubernetes.io/session-cookie-name: route  #这个是随便一个名字
     name: ingress-demo
   spec:
     rules:  #后端多主机 通过host方式定义
     - host: www.imakj.com
       http:
         paths:
         - path: / #访问根目录的时候转发到下面的serverName
           backend:
             serviceName: ingress-demo  #上面启动的nginx的名字
             servicePort: 80
     tls:
       - host:
         - www.imakj.com
         secretName: mooc-tls
     backend:  #默认规则 如果上面规则没匹配到 则默认这个
       serviceName: ingress-demo  #上面启动的nginx的名字
       servicePort: 80
   
   #这样 在浏览器访问的时候  就会有一个route的cookie
   ```

10. 滚定更新和AB测试

    ```
    # 创建服务的时候 添加以下俩注解
    # vim ingress-weight.yaml
    
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: web-canary-b
      namespace: canary
      annotations:  # 添加俩注解 
        nginx.ingress.kubernetes.io/canary: "true"  #开启注解
        nginx.ingress.kubernetes.io/canary-weight: "10"  #有10%的流量转发到这个场景
    spec:
      rules:
      - host: www.imakj.com  #域名需要一样 不然没法确定是一个服务
        http:
          paths:
          - path: /
            backend:
              serviceName: web-canary-b
              servicePort: 80
    
    ```

11. 定向流量控制

    ```
    # vim ingress-cookie.yaml
    
    #ingress
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: web-canary-b
      namespace: canary
      annotations:  #和上面一样
        nginx.ingress.kubernetes.io/canary: "true"
        nginx.ingress.kubernetes.io/canary-by-cookie: "web-canary"  #这里改成canary-by-cookie  #指定cookie  web-canary值为always 直接访问到这个流量
    spec:
      rules:
      - host:  www.imakj.com 
        http:
          paths:
          - path: /
            backend:
              serviceName: web-canary-b
              servicePort: 80
    
    ```

    还可以使用header控制

    ```
    # vim ingress-cookie.yaml
    
    #ingress
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: web-canary-b
      namespace: canary
      annotations:  #和上面一样
        nginx.ingress.kubernetes.io/canary: "true"
        nginx.ingress.kubernetes.io/canary-by-header: "web-canary"  #通过添加一个hader来控制
    spec:
      rules:
      - host:  www.imakj.com 
        http:
          paths:
          - path: /
            backend:
              serviceName: web-canary-b
              servicePort: 80
    ```

    还可以组合

    ```
    # vim ingress-cookie.yaml
    
    #ingress
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: web-canary-b
      namespace: canary
      annotations:  # 控制权限的流程会依次向下判断 转发给谁
        nginx.ingress.kubernetes.io/canary: "true"
        nginx.ingress.kubernetes.io/canary-by-header: "web-canary"
        nginx.ingress.kubernetes.io/canary-by-cookie: "web-canary"
        nginx.ingress.kubernetes.io/canary-weight: "90"
    spec:
      rules:
      - host:  www.imakj.com 
        http:
          paths:
          - path: /
            backend:
              serviceName: web-canary-b
              servicePort: 80
    ```

    

    

