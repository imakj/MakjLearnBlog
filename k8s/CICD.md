CICD基本包含：gitlab、maven、jenkins、script

k8s的cicd

![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/k8s-cicd%E5%A5%BD%E5%A4%84.png?x-oss-process=style/makjwatermarks)

# 环境准备

* git
* java
* maven
* jenkins ：官网:https://www.jenkins.io/zh/doc/pipeline/tour/getting-started/

# 安装

1. 上传jenkins

   ```
   [root@cicd opt]# mkdir /opt/jenkins
   [root@cicd jenkins]# ll
   total 69256
   -rw-r--r-- 1 root root 70918017 May 11 22:33 jenkins.war
   ```

2. 启动jenkins

   ```
   [root@cicd jenkins]# nohup java -jar jenkins.war --httpPort=8080 2>&1 &
   [root@cicd jenkins]# tail -100f nohup.out 
   Running from: /opt/jenkins/jenkins.war
   webroot: $user.home/.jenkins
   Running from: /opt/jenkins/jenkins.war
   webroot: $user.home/.jenkins
   2021-05-11 14:35:51.973+0000 [id=1]	INFO	org.eclipse.jetty.util.log.Log#initialized: Logging initialized @281ms to org.eclipse.jetty.util.log.JavaUtilLog
   2021-05-11 14:35:52.087+0000 [id=1]	INFO	winstone.Logger#logInternal: Beginning extraction from war file
   2021-05-11 14:35:52.113+0000 [id=1]	WARNING	o.e.j.s.handler.ContextHandler#setContextPath: Empty contextPath
   2021-05-11 14:35:52.162+0000 [id=1]	INFO	org.eclipse.jetty.server.Server#doStart: jetty-9.4.39.v20210325; built: 2021-03-25T14:42:11.471Z; git: 9fc7ca5a922f2a37b84ec9dbc26a5168cee7e667; jvm 1.8.0_231-b11
   2021-05-11 14:35:52.384+0000 [id=1]	INFO	o.e.j.w.StandardDescriptorProcessor#visitServlet: NO JSP Support for /, did not find org.eclipse.jetty.jsp.JettyJspServlet
   2021-05-11 14:35:52.421+0000 [id=1]	INFO	o.e.j.s.s.DefaultSessionIdManager#doStart: DefaultSessionIdManager workerName=node0
   2021-05-11 14:35:52.422+0000 [id=1]	INFO	o.e.j.s.s.DefaultSessionIdManager#doStart: No SessionScavenger set, using defaults
   2021-05-11 14:35:52.423+0000 [id=1]	INFO	o.e.j.server.session.HouseKeeper#startScavenging: node0 Scavenging every 660000ms
   2021-05-11 14:35:52.761+0000 [id=1]	INFO	hudson.WebAppMain#contextInitialized: Jenkins home directory: /root/.jenkins found at: $user.home/.jenkins
   2021-05-11 14:35:52.848+0000 [id=1]	INFO	o.e.j.s.handler.ContextHandler#doStart: Started w.@329dbdbf{Jenkins v2.277.4,/,file:///root/.jenkins/war/,AVAILABLE}{/root/.jenkins/war}
   2021-05-11 14:35:52.862+0000 [id=1]	INFO	o.e.j.server.AbstractConnector#doStart: Started ServerConnector@90f6bfd{HTTP/1.1, (http/1.1)}{0.0.0.0:8080}
   2021-05-11 14:35:52.862+0000 [id=1]	INFO	org.eclipse.jetty.server.Server#doStart: Started @1170ms
   2021-05-11 14:35:52.863+0000 [id=22]	INFO	winstone.Logger#logInternal: Winstone Servlet Engine running: controlPort=disabled
   2021-05-11 14:35:53.952+0000 [id=29]	INFO	jenkins.InitReactorRunner$1#onAttained: Started initialization
   2021-05-11 14:35:53.996+0000 [id=27]	INFO	jenkins.InitReactorRunner$1#onAttained: Listed all plugins
   2021-05-11 14:35:55.012+0000 [id=33]	INFO	jenkins.InitReactorRunner$1#onAttained: Prepared all plugins
   2021-05-11 14:35:55.016+0000 [id=33]	INFO	jenkins.InitReactorRunner$1#onAttained: Started all plugins
   2021-05-11 14:35:55.020+0000 [id=28]	INFO	jenkins.InitReactorRunner$1#onAttained: Augmented all extensions
   2021-05-11 14:35:55.372+0000 [id=31]	INFO	jenkins.InitReactorRunner$1#onAttained: System config loaded
   2021-05-11 14:35:55.372+0000 [id=31]	INFO	jenkins.InitReactorRunner$1#onAttained: System config adapted
   2021-05-11 14:35:55.372+0000 [id=31]	INFO	jenkins.InitReactorRunner$1#onAttained: Loaded all jobs
   2021-05-11 14:35:55.373+0000 [id=31]	INFO	jenkins.InitReactorRunner$1#onAttained: Configuration for all jobs updated
   2021-05-11 14:35:55.399+0000 [id=47]	INFO	hudson.model.AsyncPeriodicWork#lambda$doRun$0: Started Download metadata
   2021-05-11 14:35:55.406+0000 [id=47]	INFO	hudson.util.Retrier#start: Attempt #1 to do the action check updates server
   2021-05-11 14:35:55.473+0000 [id=34]	INFO	jenkins.install.SetupWizard#init: 
   
   *************************************************************
   *************************************************************
   *************************************************************
   
   Jenkins initial setup is required. An admin user has been created and a password generated.
   Please use the following password to proceed to installation:
   
   #这个就是初始化的密码
   6da14ff8334144f1882e15fd3e2f5351
   
   This may also be found at: /root/.jenkins/secrets/initialAdminPassword
   
   *************************************************************
   *************************************************************
   *************************************************************
   
   ```

3. 打开浏览器，输入http://192.168.10.170:8080/地址，输入上面的密码

   ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/%E7%99%BB%E5%BD%95.png?x-oss-process=style/makjwatermarks)

4. 点击安装插件

   ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/%E5%AE%89%E8%A3%85%E6%8F%92%E4%BB%B6.png?x-oss-process=style/makjwatermarks)

5. 等待 正在安装中。。。

   ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/%E6%8F%92%E4%BB%B6%E5%AE%89%E8%A3%85%E4%B8%AD.png?x-oss-process=style/makjwatermarks)

6. 可以创建一个新用户

   ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/%E5%88%9B%E5%BB%BA%E7%94%A8%E6%88%B7.png?x-oss-process=style/makjwatermarks)

7. 继续点击保存完成

   ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/%E4%BF%9D%E5%AD%98%E5%AE%8C%E6%88%90.png?x-oss-process=style/makjwatermarks)

8. 安装完成

   ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/%E5%AE%89%E8%A3%85%E5%AE%8C%E6%88%90.png?x-oss-process=style/makjwatermarks)

# 使用

jenkins主要流程都是任务 所有的cicd都是通过job完成的

1. 创建job

   ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/%E5%88%9B%E5%BB%BAjob.png?x-oss-process=style/makjwatermarks)

   ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/%E5%88%9B%E5%BB%BAjob1.png?x-oss-process=style/makjwatermarks)

      ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/%E5%88%9B%E5%BB%BAjob2.png?x-oss-process=style/makjwatermarks)

   脚本写完后，点击保存 立即构建 在下面就会看到构建的日志 点击配置会返回流水线继续修改脚本

   ![](https://makj-imagehost.oss-cn-hangzhou.aliyuncs.com/imagehosts/%E5%88%9B%E5%BB%BAjob3.png?x-oss-process=style/makjwatermarks)

 2. pipeline构建脚本

    ```
    node {
        env.BUILD_DIR = "/root/build-workspace"  //定义 一个工作目录 
        env.MODULE = "client"   //定义一个模块名字 
        env.HOST="K8S.host.com"  //镜像仓库 
        stage('Preparation') { // 1. git拉取代码 
            git 'http://makj:mkj13418854559@gitlab.yzltech.com:34678/makj/dataserver.git'
        }
        stage('Maven Build') {  //2.maven 构建
            sh "mvn clean package -pl ${MODULE} -am"
        }
        stage('Build Image') {  //3.构建 docker image docker build的脚本路径 
            sh "/root/script/dockerbuild.sh" 
        }
        stage('deploy') {  //4.发布到k8s  
            sh "/root/script/k8sdeploy.sh" 
        }
    }
    ```

3. dockerbuild.sh脚本

   ```
   #!/bin/bash
   
   #BUILD_ DIR 就是脚本里定义的env.BUILD_DIR
   if ["${BUILD_DIR}" == ""] ; then
   	echo "env 'BUILD_ DIR' is not set "
   	exit 1
   fi
   
   #JOB_NAME 就是创建pipline的流水线的名字
   DOCKER_DIR = ${BUILD_DIR}/${JOB_NAME}
   
   #判断docker工作目录是否存在
   if[! -d ${DOCKER_DIR}];then
   	mkdir -p $ {DOCKER_ DIR} 
   fi
   
   echo "docker workspace: ${DOCKER_ DIR}"
   
   #${WORKSPACE}就是当前jenkins的工作空间 ${MODULE}就是env.MODULE
   
   JENKINS_DIR = ${WORKSPACE}/${MODULE}
   echo "jenkins workspace: ${JENKINS_DIR} "
   
   if [ ! -f ${JENKINS_DIR}/target/*.war ] ; then
   	echo "target war file not found"
   	exit 1
   fi
   
   #将构建打包好的文件解压
   cd ${DOCKER_DIR}
   rm -fr *
   unzip -0q ${JENKINS_DIR}/target/*.war -d ./ROOT
   
   mv ${JENKINS_DIR} /Dockerfile
   if [ -d ${JENKINS_DIR}/dockerfiles ] ; then 
   	mv ${JENKINS_DIR}/dockerfiles .
   fi
   
   #至此 docker文件就准备好了 下面写Dockerfile
   VERSION=$(date +%Y%m%d%H%M%S) #版本号
   IMAGE_NAME = hub.mooc.com/kubernetes/${JOB_NAME}:${VERSION}  #镜像名字
   echo "building image: ${IMAGE_ NAME}"
   #将镜像名字写到一个文件里
   echo "${IMAGE_NAME}" > ${WORKSPACE}/IMAGE
   docker build -t $ {IMAGE_NAME }  #打包
   docker push $ {IMAGE_NAME}  #push到仓库
   
   ```

4. k8sdeploy.sh脚本

   ```
   #!/bin/bash
   
   name=${JOB_NAME}  #任务名字
   image=$(cat ${WORKSPACE}/IMAGE)  #镜像名字 从文件中读
   host=${HOST}  #镜像host
   echo "deploying name: ${name}, image: ${imaage},host: ${host}"
   
   rm -f web.yaml
   
   #拷贝一个目标yaml 也可以自己写一个
   cp $(dirname "${BASH_ SOURCE [0]}")/template/web.yaml .
   
   #替换模板
   sed -i "s, { {name}} ,${name} ,g" web.yaml
   sed -i "s, {{name}}，${name} ,g" web.yaml
   
   #k8s里生效
   kucectl apply -y web.yaml
   
   cat web.yaml
   
   
   #健康检查 2秒一次 最多60次
   #健康检查
   success=0 #状态
   count=60  #检查总数
   IFS=","
   sleep 5  #等待5秒后开始健康检查
   while[ ${count} -gt 0 0]
   do
   	#执行一个kubectl命令 拿到容器的status下面的所有状态值 如果都等于1 则代表容器启动成功
   	replicas=$(kubectl get deploy ${name} -o go-template='{{.status.replicas}},{{.status.updatedReplicas}}, {{.status.readyReplicas}} ,{{.status.availableReplicas}}')
   	echo "replicas: ${replicas}"
   	arr=(${replicas})
   	if [ "${arr[0]}" == "${arr[1]}" -a "${arr[1]}" == "${arr[2]}" -a "${arr[2]}" == "${arr[3]]}" ] ; then
   		echo "health check success !"
   		success=1
   		break 
   	fi
   	#可以在取到deployment.kubernetes.io/revision这个 值 这个值在deployment每启动一次的时候 就会加1 对比前后是否发生了变化得到是否更新成功
   	((count--))
   	sleep 2
   done
   if [ ${success} -ne 1 ]; then
   	echo "healte check failed"
   	exit 1
   fi
   
   ```

   

