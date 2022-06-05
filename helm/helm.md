# 介绍

Helm是一个Kubernetes的包管理工具，就像Linux下的包管理器，如yum/apt等，可以很方便的将之前打包好的yaml文件部署到kubernetes上。
Helm有3个重要概念：
•helm：一个命令行客户端工具，主要用于Kubernetes应用chart的创建、打包、发布和管理。
•Chart：应用描述，一系列用于描述k8s 资源相关文件的集合。
•Release：基于Chart的部署实体，一个chart 被Helm 运行后将会生成对应的一个release；将在k8s中创建出真实运行的资源对象。

Helm核心是模板，即模板化K8s YAML文件。
通过模板实现Chart高效复用，当部署多个应用时，可以将差异化的字段进行模板化，在部署时使用-f或者--set动态覆盖默认值，从而适配多个应用

# 下载

helm 就是一个二进制客户端包即可，会通过kubeconfig配置（通常$HOME/.kube/config）来连接Kubernetes。
项目地址：https://github.com/helm/helm
下载Helm客户端：

```
# wget https://get.helm.sh/helm-v3.4.2-linux-amd64.tar.gz
# tar zxvf helm-v3.4.2-linux-amd64.tar.gz
# mv linux-amd64/helm /usr/bin/
```

#  常用命令

## 创建chart

```
# helm create mychar
```

目录详情

•charts：目录里存放这个chart依赖的所有子chart。
•Chart.yaml：用于描述这个Chart的基本信息，包括名字、描述信息以及版本等。
•values.yaml ：用于存储templates 目录中模板文件中用到变量的值。存储默认值 变量文件 所有yaml渲染时候的默认值

•Templates：目录里面存放所有yaml模板文件。
•NOTES.txt ：用于介绍Chart帮助信息，helm install 部署后展示给用户。例如：如何使用这个Chart、列出缺省的设置等。
•_helpers.tpl：放置模板的地方，可以在整个chart 中重复使用。

## 打包chart

```
# helm package mychar
```

## 部署chart

helm install 名字 chart包的目录

```
# helm install nginx-mychart ./mychar/

# 可以看到提示信息 就是从NOTES.txt 抛出的
NAME: nginx-mychart
LAST DEPLOYED: Thu May  5 21:22:08 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=mychar,app.kubernetes.io/instance=nginx-mychart" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace default $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace default port-forward $POD_NAME 8080:$CONTAINER_PORT
```

## 查看chart

```
# helm list
# kubectl get pods,svc
```

## 创建自定义chart

1. 准备chart的自定义目录

   ```
   # pwd
   /opt/chart/demo/chardemo
   
   # tree ./
   ./
   ├── charts
   ├── Chart.yaml
   ├── templates
   │   ├── deployment.yaml
   │   ├── _helpers.tpl
   │   ├── NOTES.txt   # 空文件
   │   └── service.yaml
   └── values.yaml   # 空文件
   
   # cat ./templates/deployment.yaml 
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     creationTimestamp: null
     labels:
       app: chardemo
     name: chardemo
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: chardemo
     strategy: {}
     template:
       metadata:
         creationTimestamp: null
         labels:
           app: chardemo
       spec:
         containers:
         - image: nginx
           name: nginx
           resources: {}
   
   # cat ./templates/service.yaml 
   apiVersion: v1
   kind: Service
   metadata:
     creationTimestamp: null
     labels:
       app.kubernetes.io/instance: nginx-mychart
       app.kubernetes.io/managed-by: Helm
       app.kubernetes.io/name: mychar
       app.kubernetes.io/version: 1.16.0
       helm.sh/chart: mychar-0.1.0
     name: nginx-mychart
   spec:
     ports:
     - port: 80
       protocol: TCP
       targetPort: 80
     selector:
       app: chardemo
       
   # 先部署测试一个 这就是一个chart的版本
   # helm install web-test /opt/chart/demo/chardemo/ -n test
   NAME: web-test
   LAST DEPLOYED: Thu May  5 21:39:47 2022
   NAMESPACE: test
   STATUS: deployed
   REVISION: 1
   TEST SUITE: None
   ```

2. 查看帮助

   ```
   # cat mychar/templates/hpa.yaml 
   {{- if .Values.autoscaling.enabled }}
   apiVersion: autoscaling/v2beta1
   kind: HorizontalPodAutoscaler
   metadata:
     name: {{ include "mychar.fullname" . }}
     labels:
       {{- include "mychar.labels" . | nindent 4 }}
   spec:
     scaleTargetRef:
       apiVersion: apps/v1
       kind: Deployment
       name: {{ include "mychar.fullname" . }}
     minReplicas: {{ .Values.autoscaling.minReplicas }}
     maxReplicas: {{ .Values.autoscaling.maxReplicas }}
     metrics:
       {{- if .Values.autoscaling.targetCPUUtilizationPercentage }}
       - type: Resource
         resource:
           name: cpu
           targetAverageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
       {{- end }}
       {{- if .Values.autoscaling.targetMemoryUtilizationPercentage }}
       - type: Resource
         resource:
           name: memory
           targetAverageUtilization: {{ .Values.autoscaling.targetMemoryUtilizationPercentage }}
       {{- end }}
   {{- end }}
   ```

3. 定义模板

   ```
   # vim templates/deployment.yaml 
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     creationTimestamp: null
     labels:
       app: {{ .Release.Name }}  # 内置变量
     name: {{ .Release.Name }}
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: {{ .Release.Name }}
     strategy: {}
     template:
       metadata:
         creationTimestamp: null
         labels:
           app: {{ .Release.Name }}
       spec:
         containers:
         - image: {{ .Values.image.repository }}:{{ .Values.image.tag }}  #value默认值里的变量
           name: nginx
           resources: {}
           
   # vim templates/service.yaml 
   apiVersion: v1
   kind: Service
   metadata:
     creationTimestamp: null
     labels:
       app: {{ .Release.Name }}
     name: {{ .Release.Name }}
   spec:
     ports:
     - port: 80
       protocol: TCP
       targetPort: 80
     selector:
       app: {{ .Release.Name }}
       
       
   # 变量文件，所有yaml渲染时候的yaml默认值
   replicaCount: 1
   
   image:
     repository: nginx
     tag: "latest"
   
   # helm install chardemo1 /opt/chart/demo/chardemo/ --set image.repositpry=nginx -n test
   ```

4. 升级

   使用Chart升级应用有两种方法：
   •--values，-f：指定YAML文件覆盖值
   •--set：在命令行上指定覆盖值
   注：如果一起使用，--set优先级高

   ```
   # helm upgrade chardemo1 /opt/chart/demo/chardemo/ --set image.tag=1.7.9 -n test
   
   # 访问 变成了1.7.9
   # curl -I 10.0.0.19
   HTTP/1.1 200 OK
   Server: nginx/1.7.9
   Date: Thu, 05 May 2022 14:24:12 GMT
   Content-Type: text/html
   Content-Length: 612
   Last-Modified: Tue, 23 Dec 2014 16:25:09 GMT
   Connection: keep-alive
   ETag: "54999765-264"
   Accept-Ranges: bytes
   
   #根据文件名更新
   # vim values1.7.9.yaml 
   # 变量文件，所有yaml渲染时候的yaml默认值
   replicaCount: 1
   
   image:
     repository: nginx
     tag: "1.7.9"
   
   # 文件方式升级
   # helm upgrade -f values1.7.9.yaml chardemo1 /opt/chart/demo/chardemo/ -n test
   ```

5. 回滚版本

   ```
   # helm rollback chardemo1 -n test
   
   # helm history chardemo1 -n test
   REVISION        UPDATED                         STATUS          CHART           APP VERSION     DESCRIPTION     
   1               Thu May  5 22:06:56 2022        superseded      mychar-0.1.0    1.16.0          Install complete
   2               Thu May  5 22:20:27 2022        superseded      mychar-0.1.0    1.16.0          Upgrade complete
   3               Thu May  5 22:23:52 2022        superseded      mychar-0.1.0    1.16.0          Upgrade complete
   4               Thu May  5 22:26:28 2022        superseded      mychar-0.1.0    1.16.0          Upgrade complete
   5               Thu May  5 22:29:36 2022        deployed        mychar-0.1.0    1.16.0          Rollback to 3
   
   # helm rollback chardemo1 4 -n test
   Rollback was a success! Happy Helming!
   # helm history chardemo1 -n test
   REVISION        UPDATED                         STATUS          CHART           APP VERSION     DESCRIPTION     
   1               Thu May  5 22:06:56 2022        superseded      mychar-0.1.0    1.16.0          Install complete
   2               Thu May  5 22:20:27 2022        superseded      mychar-0.1.0    1.16.0          Upgrade complete
   3               Thu May  5 22:23:52 2022        superseded      mychar-0.1.0    1.16.0          Upgrade complete
   4               Thu May  5 22:26:28 2022        superseded      mychar-0.1.0    1.16.0          Upgrade complete
   5               Thu May  5 22:29:36 2022        superseded      mychar-0.1.0    1.16.0          Rollback to 3   
   6               Thu May  5 22:30:18 2022        superseded      mychar-0.1.0    1.16.0          Rollback to 4   
   7               Thu May  5 22:30:29 2022        deployed        mychar-0.1.0    1.16.0          Rollback to 4   
   ```

   

6. 查看历史版本

   ```
   # helm history chardemo1 -n test
   REVISION        UPDATED                         STATUS          CHART           APP VERSION     DESCRIPTION     
   1               Thu May  5 22:06:56 2022        superseded      mychar-0.1.0    1.16.0          Install complete
   2               Thu May  5 22:20:27 2022        superseded      mychar-0.1.0    1.16.0          Upgrade complete
   3               Thu May  5 22:23:52 2022        superseded      mychar-0.1.0    1.16.0          Upgrade complete
   4               Thu May  5 22:26:28 2022        deployed        mychar-0.1.0    1.16.0          Upgrade complete
   ```

   

7. 回滚到制定版本

   ```
   # helm rollback chardemo1 4 -n test
   
   # helm history chardemo1 -n test
   REVISION        UPDATED                         STATUS          CHART           APP VERSION     DESCRIPTION     
   1               Thu May  5 22:06:56 2022        superseded      mychar-0.1.0    1.16.0          Install complete
   2               Thu May  5 22:20:27 2022        superseded      mychar-0.1.0    1.16.0          Upgrade complete
   3               Thu May  5 22:23:52 2022        superseded      mychar-0.1.0    1.16.0          Upgrade complete
   4               Thu May  5 22:26:28 2022        superseded      mychar-0.1.0    1.16.0          Upgrade complete
   5               Thu May  5 22:29:36 2022        superseded      mychar-0.1.0    1.16.0          Rollback to 3   
   6               Thu May  5 22:30:18 2022        superseded      mychar-0.1.0    1.16.0          Rollback to 4   
   7               Thu May  5 22:30:29 2022        deployed        mychar-0.1.0    1.16.0          Rollback to 4   
   
   ```

8. 卸载应用

   ```
   helm uninstall chardemo1
   ```

## 常用函数

* quote

  将值转换为字符串，即加双引号 比如一些关键字出现在yaml里 只能以字符串的形式出现 这个时候就需要用quote

  例如：

  ```
    selector:
      app: {{ .Release.Name }}
      gpu: {{ quote .Values.nodeSelector.gpu }}
      
  # 最终就是
    selector:
      app: nginx
      gpu: "true"  #quote 转换为字符串 带双引号
  ```

* default

  设置默认值，如果获取的值为空则为默认值

  例如：:

  ```
    selector:
      app: {{ .Release.Name }}
      gpu: {{ .Values.nodeSelector.gpu | defaule "web" }}
      
  # 最终就是 如果yaml里没有指定gpu 值 value文件里也没有 最终就是默认值web
    selector:
      app: nginx
      gpu: "web"  
      
  # 当没有值得时候  从Chart文件里引用
    selector:
      app: {{ .Release.Name }}
      gpu: {{ .Values.nodeSelector.gpu | defaule .Chart.AppVersion }}
  ```

* indent和nindent

  缩进字符串

  例如

  ```
    selector:
      app: {{ .Release.Name | indent 5 }}
      gpu: {{ .Values.nodeSelector.gpu | defaule "web" }}
      
  # 就会在app后面加入5个空格
  
    selector:
      app:      nginx
      gpu: "web"  
      
    selector:
      app: {{ .Release.Name | nindent 5 }}
      gpu: {{ .Values.nodeSelector.gpu | defaule "web" }}
      
  # nindent 换行后在家5个空格
    selector:
      app: 
       nginx
      gpu: "web"  
  ```

* toYaml

  引用一块YAML内容

  例如：

  ```
    selector:
      app: {{ .Release.Name | nindent 5 }}
      gpu: {{ toYaml .Values.nodeSelector.gpu | nindent 10 }}
      
  # 就会将gpu下的所有数据字符串引用到这里 这里需要配合nindent使用
    selector:
      app: 
       nginx
      gpu: 
       web: nginx
       app: app1
  ```

* upper

  全部转换大写

* title

  首字母转换大写

* .File对象

  读取一个对象 渲染到yaml里

​		 Files.Glob方法返回所有匹配的文件路径列表，当做个文件时，可以更灵活提取某些文件。 可以引用某个文件里的所有参数 一般配合configmap使用

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: configmap-test
data:
  {{-$root := . }}
  {{-range $path,$bytes := .Files.Glob "files/*.properties" }}  # range 循环关键字
  {{ base $path }}: |           # base路径 值保留base的最后一个文件名
  {{ $root.Files.Get $path |indent 4 }} # 引用files/*.properties文件 并且缩进4空格
  {{-end -}} 
  
# 最后使用测试输出yaml查看下
# helm install webtest /opt/chart/demo/chardemo/ -n test --dry-run
```

## 流程函数

* if/else：条件判断

  部署一个应用，在没明确启用ingress时，默认情况下不启用

  ```
  # values.yaml
  ingress:
    enabled: false
    
  # templates/ingress.yaml
  {{ if .Values.ingress.enabled }}
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: web
  spec:
    rules:
    - host: www.aliangedu.cn
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
           service:
            name: web
            port:
             number: 80
  {{ end }}
  
  # helm install test /opt/chart/demo/chardemo/ -n test --dry-run
  ```

  如果值为以下几种情况则为false：
  • 一个布尔类型false
  • 一个数字0
  • 一个空的字符串
  • 一个nil（空或null）
  • 一个空的集合（map、slice、tuple、dict、array）

  

  条件表达式也支持操作符：
  • eq 等于
  • ne 不等于
  • lt 小于
  • gt 大于
  • and 逻辑与
  • or 逻辑或

  例：如果是一个空的集合则不启用资源配额

  ```
  # cat values.yaml
  resources: {}
  # templates/ingress.yaml
  ...
      {{ if .Values.resources }}
      resources:
       {{ toYaml .Values.resources | indent 10 }}
      {{ end }}
  ...
  
  # helm install test --dry-run mychart
   
  ```

  渲染结果会发现有多余的空行，这是因为模板渲染时会将指令删除，所以原有的位置就空白了。可以使用横杠“-”消除空行。

  ```
  # cat values.yaml
  resources: {}
  # templates/ingress.yaml
  ...
      {{- if .Values.resources }}
      resources:
       {{ toYaml .Values.resources | indent 10 }}
      {{- end }}
  ...
  
  # helm install test --dry-run mychart
  ```

  

* range：循环

  一般用于遍历序列结构的数据。例如序列、键值

  语法：
  {{ range <值> }}

  引用内容

  {{ end }}

  示例：遍历数据

  ```
  # cat values.yaml
  test:
  -1
  -2
  -3
  
  # templates/configmap.yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
   name: {{ .Release.Name }}
  data:
   test: |
   {{-range .Values.test }}
    {{ . }} # 引用当前元素
   {{-end }}
  ```

  

* with：指定变量作用域

  with可以允许将当前范围. 设置为特定的对象，比如我们前面一直使用的.Values.nodeSelecotr，我们可以使用with来将. 范围指向.Values.nodeSelecotr：

  指定变量作用域

  语法：
  {{ with <值> }}

  限制范围

  {{ end }}

  ```
  # cat values.yaml
  ...
  nodeSelector:
   team: a
   gpu: yes
   
   
  # cat templates/deployment.yaml
  ...
      {{-with .Values.nodeSelector }}
      nodeSelector:
       team: {{ .team }} # 这里就可以直接写变量名
       gpu: {{ .gpu }}
      {{-end }}
  ```

  with块限制了变量作用域，也就是无法直接引用模板对象，例如.Values、.Release，如果还想使用，可以定义变量来解决该问题。

  ```
  {{-$releaseName := .Release.Name -}}
    {{-with .Values.nodeSelector }}
    team: {{ .team }}
    gpu: {{ .gpu }}
    test: {{ $releaseName }}
  {{-end }}
  ```

## 模板

命名模板类似于开发语言中的函数，指一段可以直接被另一段程序或代码引用的程序或代码。
在编写chart时，可以将一些重复使用的内容写在命名模板文件中供公共使用，这样可减少重复编写程序段和简化代码结构。
命名模块使用define定义，template（不支持管道）或include引入，在templates目录中默认下划线开头的文件为公共模板（helpers.tpl）。

资源名称生成指令放到公共模板文件，作为所有资源名称

```
# cat templates/_helpers.tpl #定义模板文件 可以支持多个模板
{{-define "fullname" -}}  #模板名字
{{-.Chart.Name -}}-{{ .Release.Name }}
{{-end -}}

# cat templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
 name: {{ template "fullname" . }}
...
```

## 创建一个通用chart

```
# tree ./
./
├── Chart.yaml
├── files
├── templates
│   ├── configmap.yaml
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── ingress.yaml
│   ├── NOTES.txt
│   └── service.yaml
└── values.yaml

# cat Chart.yaml 
apiVersion: v2
name: mychar
description: A Helm chart for Kubernetes
version: 0.1.0
appVersion: "1.16.0"

# cat values.yaml 
# 变量文件，所有yaml渲染时候的yaml默认值
replicaCount: 1

image:
  repository: nginx
  tag: "latest"

resources: 
  limits: 
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

ingress:
  enabled: false
  annotations: {}
  host: www.default.cn
  paht: /
  tls:
    secretName: default
    host: www.default.cn

service:
  port: 80
  targetport: 80
  type: ClusterIP

env:
  NAME: "gateway"
  JAVA_OPTS: "-Xmx1G"

livenessProbe:
  httpGet:
    path: /
    port: 80
readinessProbe:
  httpGet:
    path: /
    port: 80


# mkdir templates
# touch values.yaml

# cd  templates/
# touch NOTES.txt
# touch _helpers.tpl

# cat configmap.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "fullname" . }}
data:
  {{- $root := . }}
  {{- range $path,$bytes := .Files.Glob "files/*.properties" }}
  {{base $path }}:
{{ $root.File.Get $path | indent 4 }}
  {{- end -}}


# cat deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels: {{ include "labels" . | nindent 6 }}
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels: {{ include "labels" . | nindent 8 }}
    spec:
      containers:
      - image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
        name: web
        resources: 
          {{- toYaml .Values.resources | nindent 10 }}
        env: 
        {{- range $k,$v := .Values.env }}
          - name: {{ $k }}
            value: {{ $v }}
        {{- end}}
        {{- if .Values.livenessProbe }}
        livenessProbe: {{ toYaml .Values.livenessProbe |  nindent 10 }}
        {{- end }}
        {{- if .Values.readinessProbe }}
        readinessProbe: {{ toYaml .Values.readinessProbe |  nindent 10 }}
        {{- end }}
        
        
# cat _helpers.tpl 
{{/*
资源名称
*/}}
{{- define "fullname" -}}
{{ .Chart.Name }}-{{ .Release.Name }}
{{- end -}}

{{/*
标签
*/}}
{{- define "labels" -}}
chart_name: {{ .Chart.Name }}
instance_name: {{ .Release.Name }}
{{- end -}}
    
# cat ingress.yaml 
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "fullname" . }}
  annotations: 
    {{- toYaml .Values.ingress.annotations | nindent 4 }}
spec:
  {{- if .Values.ingress.tls }}
  tls:
  - hosts:
      - {{ .Values.ingress.tls.host }}
    secretName: {{ .Values.ingress.tls.secretName }}
  {{- end }}
  rules:
  - http:
      paths: 
      - path: {{ .Values.ingress.paths }}
        pathType: Prefix
        backend:
          service:
            name: {{ include "fullname" . }}
            port:
              number: {{ .Values.service.port }}
{{- end }}
  
# cat service.yaml 
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: web
  name: {{ include "fullname" . }}
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: {{ .Values.service.targetport }}
  selector: {{ include "labels" . | nindent 4 }}
  type: {{ .Values.service.type }}
  
# cat NOTES.txt 
我是使用方法
  
# 创建一个files文件夹 放一些要导入的配置
# mkdir files/
# cd files/
```

测试

```
# 默认值
# helm install webtest1 --dry-run /opt/chart/pord-chart/

# 不带健康检查
# helm install webtest --dry-run /opt/chart/pord-chart/ --set livenessProbe=false

# 部署一个ngixn
# helm install webtest3 /opt/chart/pord-chart/ 
```

最后 打个包

```
# helm package pord-chart/
```



# 推送harbor仓库

1. 启用Harbor的Chart仓库服务 安装后默认仓库就有了Chart服务了

   ```
   # harbor执行
   # ./install.sh --with-chartmuseum
   ```

2. helm安装插件

   ```
   # cd /opt/chart
   # tar zxvf helm-push_0.9.0_linux_amd64.tar.gz 
   # helm plugin install https://github.com/chartmuseum/helm-push
   
   #或者手动安装
   # 查看当前默认目录在哪 linux 默认在$HOME/.local/share/helm 
   # helm -h
   # 将helm-push_0.9.0_linux_amd64解压放到 /root/.local/share/helm/plugins
   # mv /opt/chart/push/plugin.yaml /root/.local/share/helm/plugins/helm-push/
   # mv /opt/chart/push/bin/ /root/.local/share/helm/plugins/helm-push/
   
   # 至此 命令就可以了
   # helm push
   ```

3. 添加repo

   ```
   # helm repo add --username admin --password Harbor12345 myrepo http://192.168.10.61/chartrepo/library
   
   # helm repo update
   ```

4. 推送

   ```
   # helm push mychar-0.1.0.tgz --username=admin --password=Harbor12345 http://192.168.10.61/chartrepo/library
   ```

5. 部署

   ```
   # helm install web --version 0.1.0 myrepo/mychar
   ```

# Tiller安装 

Tiller 是以 Deployment 方式部署在 Kubernetes 集群中的，由于 Helm 默认会去 [storage.googleapis.com](http://storage.googleapis.com/) 拉取镜像 所以需要将仓库指向阿里云

1. 执行阿里云

   ```
   [root@k8shardway1 local]# helm init --client-only --stable-repo-url https://aliacs-app-catalog.oss-cn-hangzhou.aliyuncs.com/charts/
   
   [root@k8shardway1 local]# helm repo add incubator https://aliacs-app-catalog.oss-cn-hangzhou.aliyuncs.com/charts-incubator/
   
   [root@k8shardway1 local]# helm repo update
   Hang tight while we grab the latest from your chart repositories...
   ...Skip local chart repository
   ...Successfully got an update from the "stable" chart repository
   ...Successfully got an update from the "incubator" chart repository
   Update Complete. ⎈ Happy Helming!⎈ 
   
   # 因为官方的镜像无法拉取，使用-i指定自己的镜像
   $ helm init --service-account tiller --upgrade -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.13.1  --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
   
   # 创建TLS认证服务端
   $ helm init --service-account tiller --upgrade -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.13.1 --tiller-tls-cert /etc/kubernetes/ssl/tiller001.pem --tiller-tls-key /etc/kubernetes/ssl/tiller001-key.pem --tls-ca-cert /etc/kubernetes/ssl/ca.pem --tiller-namespace kube-system --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/chart
   ```

2. 安装socat

   ```
   # yum install -y socat
   ```

3. 创建tiller的Deployment 

   ```
   [root@k8shardway1 helm]#  helm init --output yaml > tiller.yaml 
   
   #修改api和
   [root@k8shardway1 helm]# vim tiller.yaml 
   
   ---
   apiVersion: apps/v1  #修改
   kind: Deployment
   metadata:
     creationTimestamp: null
     labels:
       app: helm
       name: tiller
     name: tiller-deploy
     namespace: kube-system
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: helm  #添加matchLabels
     strategy: {}
     template:
       metadata:
         creationTimestamp: null
         labels:
           app: helm
           name: tiller
       spec:
         automountServiceAccountToken: true
         containers:
         - env:
           - name: TILLER_NAMESPACE
             value: kube-system
           - name: TILLER_HISTORY_MAX
             value: "0"
           image: registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.13.1
           imagePullPolicy: IfNotPresent
           livenessProbe:
             httpGet:
               path: /liveness
               port: 44135
             initialDelaySeconds: 1
             timeoutSeconds: 1
           name: tiller
           ports:
           - containerPort: 44134
             name: tiller
           - containerPort: 44135
             name: http
           readinessProbe:
             httpGet:
               path: /readiness
               port: 44135
             initialDelaySeconds: 1
             timeoutSeconds: 1
           resources: {}
   status: {}
   
   ---
   apiVersion: v1
   kind: Service
   metadata:
     creationTimestamp: null
     labels:
       app: helm
       name: tiller
     name: tiller-deploy
     namespace: kube-system
   spec:
     ports:
     - name: tiller
       port: 44134
       targetPort: tiller
     selector:
       app: helm
       name: tiller
     type: ClusterIP
   status:
     loadBalancer: {}
   
   ...
                                          
   ```

   

4. 给Tiller授权

   ```
   # 创建serviceaccount tiller用户
   [root@k8shardway1 helm]# kubectl create serviceaccount --namespace kube-system tiller
   serviceaccount/tiller created
   
   #创建角色绑定
   [root@k8shardway1 helm]# kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
   clusterrolebinding.rbac.authorization.k8s.io/tiller-cluster-rule created
   
   
   ```

   



# 其他

官方维护一个公共仓库，可直接使用它们制作好的包。
官方仓库：https://artifacthub.io
例如使用Chart部署Dashboard：https://artifacthub.io/packages/helm/k8s-dashboard/kubernetes-dashboard
