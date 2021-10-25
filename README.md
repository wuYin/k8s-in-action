## 2. Get Started

### 2.1 Docker 容器

```shell
> docker run busybox ls -lh # 运行标准的 unix 命令
> docker run <image>:<tag>  # 运行指定版本的 image，tag 默认 latest

# Dockerfile 包含构建 docker 镜像的命令
FROM node # 基础镜像
ADD app.js /app.js # 将本地文件添加到镜像的根目录
ENTRYPOINT ["node", "app.js"] # 镜像被执行时需被执行的命令

> docker build -t kubia . # 在当前目录根据 Dockerfile 构建指定 tag 的镜像
> docker images # 列出本地所有镜像

# 执行基于 kubia 镜像，映射主机 8081 到容器内 8080 端口，并在后台运行的容器
> docker run --name kubia-container -p 8081:8080 -d kubia
> docker ps # 列出 running 容器
> docker ps -a # 列出 running, exited 容器

> docker exec -it kubia-container bash # 在容器内执行 shell 命令，如 ls/sh
> docker stop kubia-container # 停止容器
> docker rm kubia-container # 删除容器
> docker tag kubia wuyinio/kubia # 给本地镜像打标签

> docker login
> docker push wuyinio/kubia # push 到 DockerHub
```

### 2.2 配置 k8s 集群

```shell
> minikube start # 本地启动 minikube 单节点虚拟机
> kubectl cluster-info # 查看集群各组件的 URL，是否工作正常

> kubectl get nodes # get 命令可列出各种 k8s 对象的基本信息
> kubectl describe node <NODE_ID> # describe 命令显示 k8s 对象更详细的信息
```

### 2.3 在 k8s 上运行应用

```shell
> kubectl run kubia --image=wuyinio/kubia --port=8080 --generator=run/v1  # 创建 rc 并拉取镜像运行
> kubectl get pods # 列出 pods
> kubectl expose rc kubia --type=LoadBalancer --name kubia-http # 通过 LoadBalancer 服务暴露 ClusterIP pod 服务给外部访问
> kubectl get svc # 列出 services
> kubectl get rc  # 列出 replication controller
> minikube service kubia-http # minikube 单节点不支持 LoadBalancer 服务，需手动获取服务地址
# kubia-http service 的 EXTERNAL_IP 一直为 PENDING 状态

> kubectl scale rc kubia --replicas=5 # 修改 rc 期望的副本数，来水平伸缩应用
> kubectl get pod kubia-1ic8j -o wide # 显示 pod 详细列
```



## 3. Pod

### 3.2 创建 Pod

yaml pod 定义模块

- apiVersion 与 kind 资源类型。
- metadata 元数据： pod 名称、标签、注解。
- spec 规格内部元件信息：容器镜像名称、卷等。

kubia-manual.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-manual
spec:
  containers:
    - image: wuyinio/kubia:2
      name: kubia
      ports:
        - containerPort: 8080  # pod 对外暴露的端口
          protocol: TCP # supported values: "SCTP", "TCP", "UDP"
```

```shell
> kubectl create -f kubia-manual.yaml # 从 yaml 文件创建 k8s 资源
> kubectl get pod kubia-manual -o yaml # 导出 pod 定义
> kubectl logs kubia-manual -c kubia # 查看 kubia-manual Pod 中 kubia 容器的日志，-c 显式指定容器名称
> kubectl logs kubia-manual --previous # 查看崩溃前的上一个容器日志

> minikube ssh && docker ps
> docker logs bdb67198848d  # 登录到 pod 运行时的节点 minikube，手动 docker logs 查看日志

> kubectl port-forward kubia-manual 9090:8080 # 配置多重端口转发，将本机的 9090 转发至 pod 的 8080，可用于调试等
port-forward kubia-manual 9090:8080 
Forwarding from 127.0.0.1:9090 -> 8080 
Forwarding from [::1]:9090 -> 8080
Handling connection for 9090 # curl 127.0.0.1:9090
```



### 3.3 标签（label）

基于组操作 pod 而非单个操作，metadata.labels 的 kv pair 标签可组织任何 k8s 资源，保存标识信息。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-manual-v2
  labels:
    creation_method: manual
    env: prod
spec: # ...
```

```shell
# 基于 Label 的增删改查操作
> kubectl get pods --show-labels # 显示 labels
> kubectl get pods -L env # 只显示 env 标签
> kubectl label pod kubia-manual env=debug # 为指定的 pod 资源添加新标签
> kubectl label pod kubia-manual env=online --overwrite=true # 修改标签
> kubectl label pod kubia-manual env- # - 号删除标签
```



### 3.5 标签选择器（nodeSelector）

label selector 可筛选出具有指定值的  k8s 资源。

```shell
> kubectl get pods -l env=debug # 筛选 env 为 debug 的 pods  # get pod 与 get pods 无异
> kubectl get pod -l creation_method!=manual # 不等
> kubectl get pods -l '!env' # 不含 env 标签的 pods # -l 筛选 -L 显示 # "" 双引号会转义
> kubectl get pods -l 'env in (debug)' # in 筛选 # -l 接受整体字符串为参数
> kubectl get pods -l 'env notin (debug,online)' # notin 筛选
```

在指定标签 node 上运行 pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-gpu
spec:
  nodeSelector:
    gpu: "true" # 当无可用 node 时 pod 一直处于 Pending 状态
  containers: #...
```

可以使用 ` kubernetes.io/hostname: minikube` 的 nodeSelector 将 pod 运行在指定的物理机上（不建议）



### 3.6  注解（annotation）

类似 label 的 kv pair 注释，但不用作标识，用作资源说明（所以才会放在 pod metadata 的第一个子节点），可添加大量的数据块：

```shell
# 增删改查和 label 操作一样
> kubectl annotate pod kubia-gpu yinzige.com/gpu=10G
> kubectl describe pod kubia-gpu # 出现在 metadata.annotation
```

注：为避免标签或注解冲突，和 Java Class 使用倒序域名的方式类似，建议 key 中添加域名信息。



### 3.7 命名空间（namespace）

labels 会导致资源重叠，可用 namespace 将对象分配到集群级别的隔离区域，相当于多租户的概念，以 namespace 为操作单位。

```shell
> kubectl get ns # 获取所有 namespace，默认 default 下
> kubectl get pods -n kube-system # 获取指定 namespace 下的资源
```

```yaml
# kubectl create namespace custom-namespace # 创建 namespace 资源
apiVersion: v1
kind: Namespace
metadata:
  name: custom-namespace
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-manual
  namespace: custom-namespace # 创建资源时在 metadata.namespace 中指定资源的命名空间
spec: # ...
```

```shell
# 切换上下文命名空间
> kubectl config set-context $(kubectl config current-context) --namespace custom-namespace 
```



### 3.8  删除 pod

删除原理：向 pod 所有容器进程定期发送 SIGTERM 信号，超时则发送 SIGKILL 强制终止。需在程序内部捕捉信号正确处理，如 Go 注册捕捉信号  `signal.Notify()` 后 select 监听该 channel

```shell
> kubectl delete pod -l env=debug # 删除指定标签的 pod
> kubectl delete pod --all # 删除当前 namespace 下的所有 pod （慎用）
> kubectl delete all --all # 删除所有类型资源的所有对象（慎用）
```



## 4. ReplicationController

### 4.1 容器存活探针（Liveness Probe）

将进程的重启监控从程序监控级别提升到 k8s 集群功能级别，使进程 OOM，死锁或死循环时能自动重启。pod 中各容器的探针用于暴露给 k8s ，来检查容器中的应用进程是否正常。分为 3 类：

- HTTP Get：指定 IP:Port 和 Path，GET 请求返回 5xx 或超时则认为失败。
- TCP Socket：是否能建立 TCP 连接。
- Exec：在容器中执行任意命令，检查 `$?` 是否为 0

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-liveness
spec:
  containers:
    - image: wuyinio/kubia-unhealthy
      name: kubia
      livenessProbe:
        httpGet:   # 定义 http get 探针
          path: /  # 指定路径和端口号
          port: 8080
        initialDelaySeconds: 11 # 初次探测延迟 11s
```

存活探针原则：

- 为检查设立子路径 `/health` ，确保无认证。
- 保证探针返回失败时，错误发生在应用内且重启可恢复，而非应用外的组件导致的失败，那重启也没用。

注：非托管 Pod 仅由 Worker 节点的 kubelet 负责通过探针监控并重启，但整个节点崩溃会丢失该 Pod



### 4.2  ReplicationController

RC 监控 Pod 列表并根据模板增删、迁移 Pod。分为 3 部分：

- 标签选择器 label selector：确定要管理哪些 pod
- 副本数量 replica count：指定要运行的 pod 数量
- pod 模板 pod template：创建新的 pod 副本

注：修改标签选择器，会导致 rc 不再关注之前匹配的所有 pod。修改模板则只对新 Pod 生效（如手动 delete）

RC 的两个功能：

- 监控：确保符合标签选择器的 Pod 以指定的副本数量运行，多了则删除，少了则按 Pod 模板创建。
- 扩缩容：能对监控的某组 Pod 进行动态修改副本数量进行扩缩容。

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: kubia
spec:
  replicas: 3 # pod 实例的期望数
  selector:   # 决定 rc 的操作对象，可省略。必须与模板 label 匹配，否则 `selector` does not match template `labels`
    app: kubia
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
        - name: kubia
          image: wuyinio/kubia
          ports:
          - containerPort: 8080
```

操作 RC：

```shell
> kubectl describe rc kubia # 查看 rc 详细信息如 events
> kubectl edit rc kubia # 修改 rc 的 yaml 配置
> kubectl scale rc kubia --replicas=5 # 扩缩容
> kubectl delete rc kubia --ascade=false # 删除 rc 时保留运行中的 pod # ascade 级联（关联删除）
```



### 4.3  ReplicaSet

 ReplicaSet = ReplicationController + 扩展的 label selector ，即对 pod 的 label selector 表达力更强 。

RS 能通过 `selector.matchLabels` 和 `selector.matchExpressio` 来扩展对 pod label 的筛选：

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    matchLabels: # 与 RC 一样必须完整匹配
      app: kubia
    matchExpressions:
      - key: app
        operator: In # 必须有 KEY 且 VALUE 在列表中
        values:
          - kubia
          - kubia-v2
      - key: app
        operator: NotIn # 有 KEY 则不能在如下列表中
        values:
          - KUBIA
      - key: env # 必须存在的 KEY，不能有 VALUE
        operator: Exists
      - key: ENV
        operator: DoesNotExist # 必须不能存在的 KEY，也不能有 VALUE
  template:
    metadata:
      labels: # Pod 模板的 label 必须能和 RS 的 selector 匹配上
        app: kubia
        env: env_exists
    spec:
      containers:
        - name: kubia
          image: yinzige/kubia
          ports:
            - protocol: TCP
              containerPort: 8080
```



### 4.4  DaemonSet

- 功能：保证标签匹配的 Pod 在符合 selector 的一个节点上运行一个，没有目标 pod 数量的概念，无法 scale
- 场景：部署系统级组件，如 Node 监控，如 kube-proxy 处理各节点网络代理等。

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ssd-monitor
spec:
  selector:
    matchLabels:
      app: ssd-monitor # 指定要控制运行的一组 Pod
  template:
    metadata:
      labels:
        app: ssd-monitor # 被控制的 Pod 的标签
    spec:
      nodeSelector: # 选择 Pod 要运行的节点标签
        disk: ssd # 注意 YAML 文件的 true 类型是布尔型，如 ssd: true 是无法被解析为 String 的
      containers:
        - name: main
          image: yinzige/ssd-monitor
```



### 4.5 Job

- 功能：保证任务以指定的并发数执行指定次数，任务执行失败后按配置策略处理。
- 场景：执行一次性任务。

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  completions: 5 # 总任务数量
  parallelism: 2 # 并发执行任务数
  template:
    metadata:
      labels:
        app: batch-job # 要执行的 pod job label
    spec:
      restartPolicy: OnFailure # 任务异常结束或节点异常时处理方式："Always", "OnFailure", "Never"
      containers:
        - name: main
          image: yinzige/batch-job
```



### 4.6 CronJob

对标 Linux 的 crontab 的定时任务。

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: cron-batch-job
spec:
  schedule: "*/1 * * * *" # 每分钟运行一次
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: cron-batch-job
        spec:
          restartPolicy: OnFailure
          containers:
            - name: cron-batch-job
              image: yinzige/batch-job
```



## 5. Service

### 5.1 Service 内部解析

#### 接入点隔离

由于 pod 调度后 IP 会变化，需使用 Service 服务给一组 Pod 提供不变的单一接入点 entrypoint，即  `IP:Port`

```shell
> kubectl expose rc kubia --type=LoadBalancer --name kubia-http # 通过服务暴露 ClusterIP pod 服务给外部访问
```

创建名为 kubia 的 Service，将其 80 端口的请求分发给有 `app: kubia` 标签 Pod 的自定义的 http 端口上：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  sessionAffinity: ClientIP
  ports:
    - name: http
      port: 80
      targetPort: http
    - name: https
      port: 443
      targetPort: https # defined in pod template.spec.containers.ports array
  selector:
    app: kubia
```

设置请求亲和性：保证一个 Client 的所有请求都只会落到同一个 Pod 上：

```yaml
apiVersion: v1
kind: Service
metadata: 
  name: kubia-svc-session
spec:
  sessionAffinity: ClientIP # or None default 
  ports:
    - port: 80
      targetPort: 8080
```



#### 服务发现

客户端和 Pod 都需知道服务本身的 IP 和 Port，才能与其背后的 Pod 进行交互。

- 环境变量：`kubectl exec kubia-qgtmw env` 会看到 Pod 的环境变量列出了 Pod 创建时的所有服务地址和端口，如 SVCNAME_SERVICE_HOST 和 SVCNAME_SERVICE_PORT 指向服务。

- DNS 发现：Pod 上通过全限定域名 FQDN 访问服务：`<service_name>.<namespace>.svc.cluster.local`

  ```shell
  > kubectl exec kubia-qgtmw cat /etc/resolv.conf
  nameserver 10.96.0.10
  search default.svc.cluster.local svc.cluster.local cluster.local # 会
  options ndots:5
  ```

  在 Pod kubia-qgtmw 中可通过访问 `kubia.default.svc.cluster.local` 来访问 kubia 服务，在 `/etc/resolv.conf` 中指明了域名解析服务器地址，以及主机名补全规则，是在 Pod 创建时候，根据 namespace 手动导入的。



### 5.2  Service 对内部解析外部

集群内部的 Pod 不直连到外部的 IP:Port，而是同样定义 Service 结合外部 endpoints 做代理中间层解耦。如获取百度首页的向外解析：

1.1  建立外部目标的 endpoints 资源：

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: baidu-endpoints
  
subsets:
  - addresses:
      - ip: 220.181.38.148 # baidu.com
      - ip: 39.156.69.79
    ports:
      - port: 80
```

1.2  或者建立外部解析别名

```yaml
apiVersion: v1
kind: Service
metadata:
  name: baidu-endpoints
spec:
  type: ExternalName
  externalName: www.baidu.com
  ports:
    - port: 80
```

2. 再建立同名的 Service 代理，标识使用上边这组 endpoints

```yaml
apiVersion: v1
kind: Service
metadata:
  name: baidu-endpoints
spec:
  ports:
    - port: 80
```

3. 效果：在集群内部 Pod 上可透过名为 baidu-endpoints 的 Service 连接到百度首页：

```shell
# root@kubia-72sxt:/# curl 10.103.134.52
root@kubia-72sxt:/# curl baidu-endpoints
<html>
<meta http-equiv="refresh" content="0;url=http://www.baidu.com/">
</html>
```

注意 Service 类型：

```shell
> kubectl get svc                      
NAME              TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)          AGE
baidu-endpoints   ExternalName   <none>         www.baidu.com   80/TCP           30m
kubernetes        ClusterIP      10.96.0.1      <none>          443/TCP          5d2h
kubia             ClusterIP      10.96.239.1    <none>          80/TCP,443/TCP   87m
```



### 5.3  Service 对外部解析内部

#### 5.3.1  NameNode

使用：外部客户端直连宿主机端口访问服务。

原理：在集群所有节点暴露指定的端口给外部客户端，该端口会将请求转发 Service 进一步转发给节点上符合 label 的 Pod，即 Service 从所有节点收集指定端口的请求并分发给能处理的 Pod

缺点：高可用性需由外部客户端保证，若节点下线需及时切换。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia-nodeport
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30001
  selector:
    app: kubia
```

三个端口号，节点转发 30001，Service 转发 80：

宿主机即外部，执行  `curl MINIKUBE_NODE_IP:30001` 会被转发到有 `app:kubia` 标签的 Pod 的  8080 端口。执行 `curl NAME_PORT:80` 同理。



### 5.3.2  LoadBalancer

场景：外部客户端直连 LB 访问服务。其是 K8S 集群端高可用的 NameNode 扩展。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia-loadbalancer
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: kubia
```

k8s `app:kubia` 所在的所有节点打开随机端口 **32148**，进一步转发给 Pod 的 8080 端口。

```
kubia-loadbalancer   LoadBalancer   10.108.104.22   <pending>       80:32148/TCP     4s
```



### 5.4  Ingress

顶级转发代理资源，仅通过一个 IP 即可代理转发后端多个 Service 的请求。需要开启 nginx controller 功能

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: kubia
spec:
  rules:
    - host: "kubia.example.com"
      http:
        paths:
          - path: /kubia # 将 /kubia 子路径请求转发到 kubia-nodeport 服务的 80 端口
            backend:
              serviceName: kubia-nodeport
              servicePort: 80
          - path: /user # 可配置多个 path 对应到 service
            backend:
              serviceName: user-svc
              servicePort: 90
    - host: "new.example.com" # 可配置多个 host
      http:
        paths:
          - path: /
            backend:
              serviceName: gooele
              servicePort: 8080
```



### 5.5 就绪探针

场景：pod 启动后并非立刻就绪，需延迟接收来自 service 的请求。若不定义就绪探针，pod 启动就会暴露给 service 使用，所以需像存活探针一样添加指定类型的就绪探针：

```yaml
#...
    spec:
      containers:
        - name: kubia-container
          image: yinzige/kubia
          readinessProbe:
            exec:
              command:
                - ls
                - /var/ready_now
```



### 5.6  headless

场景：向客户端暴露所有 pod 的 IP，将 ClusterIP 置为 None 即可：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia-headless
spec:
  clusterIP: None
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: kubia
```

k8s 不会为 headless 服务分配 IP，通过 DNS 可直接发现后端的所有 Pod

```shell
> kubectl get svc                       
NAME             TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubia            LoadBalancer   10.104.138.112   <pending>     80:32110/TCP   123m
kubia-headless   ClusterIP      None             <none>        80/TCP         10m

root@dnsutils:/# nslookup kubia-headless
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   kubia-headless.default.svc.cluster.local
Address: 172.17.0.6
Name:   kubia-headless.default.svc.cluster.local
Address: 172.17.0.12
Name:   kubia-headless.default.svc.cluster.local
Address: 172.17.0.10
Name:   kubia-headless.default.svc.cluster.local
Address: 172.17.0.8

root@dnsutils:/# nslookup kubia
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   kubia.default.svc.cluster.local
Address: 10.104.138.112
```

## Ch6. Volume

### 6.1 卷介绍

- 问题：Pod 中每个容器的文件系统来自镜像，相互独立。
- 解决：使用存储卷，让容器访问外部磁盘空间、容器间共享存储。

卷是 Pod 生命周期的一部分，不是 k8s 资源对象。Pod 启动时创建，删除时销毁（文件可选保留）。用于 Pod 中挂载到多个容器进行文件共享。

卷类型：

- emptyDir：存放临时数据的临时空目录。
- hostPath：将 k8s worker 节点的系统文件挂载到 Pod 中，常用于单节点集群的持久化存储。
- persistentVolumeClaim：PVC 持久卷声明，用于预配置 PV

### 6.2 emptyDir 卷

1 个 Pod 中 2 个容器使用同一个 emptyDir 卷 html，来共享文件夹，随 Pod 删除而清除，属于非持久化存储。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune
spec:
  containers:
    - name: html-generator
      image: yinzige/fortuneloop
      volumeMounts:
        - name: html
          mountPath: /var/htdocs # 将 html 的卷挂载到 html-generator 容器的 /var/htdocs 目录
    - name: web-server
      image: nginx:alpine
      volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html  # 将 html 的卷挂载 web-server 容器到 /usr/share/nginx/html 目录
          readOnly: true # 设置只读
  volumes:
    - name: html # 声明名为 emptyDir 的 emptyDir 卷
      emptyDir: {}
```

emptyDir 卷跟随 Pod 被 k8s 自动分配在宿主机指定目录：`/var/lib/kubelet/pods/PODUID/volumes/kubernetes.io~empty-dir/VOLUMENAME`

如上的 html 卷位置在 minikube 节点：

```shell
$ sudo ls -l /var/lib/kubelet/pods/144c55eb-edf5-4b44-a2f6-a0d9cfe04f7c/volumes/kubernetes.io~empty-dir/html
total 4
-rw-r--r-- 1 root root 80 Apr 26 05:01 index.html
```



### 6.3 hostPath 卷

hostPath 卷的数据不跟随 Pod 生命周期，下一个调度至此节点的 Pod 能继续使用前一个 Pod 留下的数据，pod 和节点是强耦合的，只适合单节点部署。



### 6.5  持久化卷 PV、持久化卷声明 PVC

PV 与 PVC 用于解耦 Pod 与底层存储。PV、PVC 与底层存储关系：

![](http://images.yinzige.com/2019-08-21-052202.png)

流程：

- 管理员向集群加入节点时准备 NFS 等存储资源（TODO ）
- 管理员创建指定大小和访问模式的 PV
- 用户创建需要大小的 PVC
- K8S 寻找符合 PVC 的 PV 并绑定
- 用户在 Pod 中通过卷引用 PVC，从而使用存储 PV 资源

Admin 通过网络存储创建 PV:

```yaml
apiVersion: v1
kind: PersistentVolume # 创建持久卷
metadata:
  name: mongodb-pv
spec:
  capacity:
    storage: 1Gi # 告诉 k8s 容量大小和多个客户端挂载时的访问模式
  accessModes:
    - ReadWriteOnce
    - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain # Cycle / Delete 标识 PV 删除后数据的处理方式
  hostPath: # 持久卷绑定到本地的 hostPath
    path: /tmp/mongodb
```

User 通过创建 PVC 来找到大小、容量均匹配的 PV 并绑定：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc # pvc 名称将在 pod 中引用
spec:
  resources:
    requests:
      storage: 1Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: "" # 手动绑定 PVC 到已存在的 PV，否则有值就是等待绑定到匹配的新 PV
```

User 创建 Pod 使用 PVC：

```yaml
kind: Pod
metadata:
  name: mongodb
spec:
  containers:
    - image: mongo
      name: mongodb
      volumeMounts:
        - name: mongodb-data
          mountPath: /tmp/data
      ports:
        - containerPort: 27017
          protocol: TCP
  volumes:
    - name: mongodb-data
      persistentVolumeClaim: # pod 中通过 claimName 指定要引用的 PVC 名称
        claimName: mongodb-pvc
```

PV 设置卷的三种访问模式：

- RWO：ReadWriteOnly：仅允许单个节点挂载读写
- ROX：ReadOnlyMany ：允许多个节点挂载只读
- RWX：ReadWriteMany：允许多个节点挂载读写

注：PV 是集群级别的存储资源，PVC 和 Pod 是命名空间范围的。所以，在 A 命名空间的 PVC 和在 B 命名空间的 PVC 都有可能绑到同一个 PV 上。



### 6.6  动态 PV：存储类 StorageClass

场景：进一步解耦 Pod 与 PVC，使 Pod 不依赖 PVC 名称，而且跨集群移植只需保证 SC 一致即可，不用管 PVC 和 PV。同时还能给不同 PV 进行归档如按硬盘属性进行分类。

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: k8s.io/minikube-hostpath # 指定 SC 收到创建 PVC 请求时应调用哪个组件进行处理并返回 PV
parameters:
  type: pd-ssd
```



总流程：可创建 StorageClass 存储类资源，用于分类 PV，在  PVC 中绑定到符合条件的 PV 上。

![](http://images.yinzige.com/2019-08-21-054515.png)



## 7.  配置传递：ConfigMap 与 Secret

### 7.1 配置容器化应用程序

三种配置方式：

- 向容器传递命令行参数。
- 为每个容器设置环境变量。
- 通过卷将配置文件挂载至容器中。

容器传递配置文件的问题：修改配置需重新构建镜像，配置文件完全公开。解决：配置使用 ConfigMap 或 Secret 卷挂载。

### 7.2 向容器传递命令行参数

Dockerfile 中 `ENTRYPOINT` 为命令，`CMD` 为其参数。但参数能被 `docker run <image> <arg_values>` 中的参数覆盖。

```yaml
ENTRYPOINT ["/bin/fortuneloop.sh"] # 在脚本中通过 $1 获取 CMD 第一个参数，Go 中 os.Args[1] 类似
CMD ["10", "11"]
```

二者等同于 Pod 中的 `command` 和 `args`，但 pod 可通过 image 的 command 和 args 子标签进行覆盖，注意参数必须是字符串：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune2s
spec:
  containers:
    - image: luksa/fortune:args
      args: ["2"]
# ...
```

### 7.3 为容器设置环境变量

只能在各容器级别注入环境变量，而非 Pod 级别。配置容器部分 `spec.containers.env` 指定即可：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune3s
spec:
  containers:
    - image: luksa/fortune:env
      env:
        - name: INTERVAL # 对应到容器 html-generator 中的 $INTERVAL
          value: "5"
        - name: "NESTED_VAR"
          value: "$(INTERVAL)_1" # 可引用其他环境变量          
      name: html-generator
# ...      
```

缺点：硬编码环境变量可能在多个环境下值不同无法复用，需将配置项解耦。

### 7.4 ConfigMap 卷

存储非敏感信息的文本配置文件。

```shell
# 创建 cm 的四种方式：可从 kv 字面量、配置文件、有命名配置文件、目录下所有配置文件
> kubectl create configmap fortune-config --from-literal=sleep-interval=25 # 从 kv 字面量创建 cm
> kubectl create configmap fortune-config --from-file=nginx-conf=my-nginx-config.conf # 指定 k 的文件创建 cm
```

两种方式将 cm 中的值传递给 Pod 中的容器：

#### 7.4.1  设置环境变量或命令行参数

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-env-from-configmap
spec:
  containers:
    - image: luksa/fortune:env
      name: html-generator
      env:
        - name: INTERVAL # 取 CM fortune-config 中的 sleep-interval，作为 html-generator 容器环境变量 INTERVAL 的值
          valueFrom:
            configMapKeyRef:
              name: fortune-config-cm
              key: sleep-interval 
      envFrom: # 批量导入 cm 的所有 kv 作为环境变量，并加上前缀
        - prefix: CONF_
          configMapRef:
            name: fortune-config-cm              
# ...
```

可使用 `kubectl get cm fortune-config -o yaml` 查看 CM 的 data 配置项。

#### 7.4.2  配置 ConfigMap 卷

当配置项过长需放入配置文件时，可将配置文件暴露为 cm 并用卷引用，从而在各容器内部挂载读取。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-configmap-volume
spec:
  containers:
    - name: html-generator
      image: yinzige/fortuneloop:env
      env:
        - name: INTERVAL
          valueFrom:
            configMapKeyRef:
              key: sleep-interval # raw key file name
              name: fortune-config # cm name
      volumeMounts:
        - mountPath: /var/htdocs
          name: html
    - name: web-server
      image: nginx:alpine
      volumeMounts:
        - mountPath: /usr/share/nginx/html
          name: html
          readOnly: true
        - mountPath: /etc/nginx/conf.d/gzip_in.conf
          name: config
          subPath: gzip.conf # 使用 subPath 只挂载部分卷 gzip.conf 到指定目录下指定文件 gzip_in.conf
          readOnly: true
      ports:
        - containerPort: 80
          name: http
          protocol: TCP
  volumes:
    - name: html
      emptyDir: {}
    - name: config
      configMap:
        name: fortune-config # cm name
        defaultMode: 0666 # 设置卷文件读写权限
        items: # 使用 items 限制从 cm 暴露给卷的文件
          - key: my-nginx-config.conf
            path: gzip.conf # 把 key 文件的值 copy 一份到新文件中

```

添加 `items` 来暴露指定的文件到卷中，`subPath` 用来挂载部分卷，而不隐藏容器目录原有的初始文件。

```shell
> kubectl exec fortune-configmap-volume -c web-server -it -- ls -lA /etc/nginx/conf.d
total 8
-rw-r--r--    1 root     root          1093 Apr 14 14:46 default.conf # subPath
-rw-rw-rw-    1 root     root           242 Apr 27 16:49 gzip_in.conf
```

#### 7.4.3  ConfigMap 场景

使用 `kubectl edit cm fortune-config` 修后，容器中对应挂载的卷文件会延迟将修改同步。问题：若 pod 应用不支持配置文件的热更新，那同步了的修改并不会再旧 pod 生效，反而新起的 pod 会生效，造成新旧配置共存的问题。

场景：cm 的特性是不变性，若 pod 应用本身支持热更新，则可修改 cm 动态更新，但注意有 k8s 的监听延迟。



### 7.5 Secret

存储敏感的配置数据，大小限制 1MB，其配置条目会以 Base64 编码二进制后存储：

```shell
> kubectl create secret generic fortune-auth --from-file=fortune-auth/ # password.txt
```

在 pod 中加载：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-with-serect
spec:
  containers:
    - name: fortune-auth-main
      image: yinzige/fortuneloop
      volumeMounts:
        - mountPath: /tmp/no_password.txt
          subPath: password.txt
          name: auth
  volumes:
    - name: auth
      secret:
        secretName: fortune-auth
```

读取正常：

```shell
> kubectl exec fortune-with-serect -it -- ls -lh /tmp            
total 4.0K
-rw-r--r-- 1 root root 10 Apr 27 17:51 no_password.txt
> kubectl exec fortune-with-serect -it -- mount | grep password
tmpfs on /tmp/no_password.txt type tmpfs (ro,relatime) # secret 仅存储在内存中
```

secret 可用于从镜像仓库中拉取 private 镜像，需配置专用的 secret 使用：

```yaml
apiVersion: v1
kind: Pod
spec:
  imagePullSecrets:
    - name: dockerhub-secret
  containers: # ...
```



## 8. Pod Metadata 与 k8s API

场景：从 Pod 中的容器应用进程访问 Pod 元数据及其他资源。

### 8.1 使用 Downward API 传递 Pod Metadata

问题：配置数据如环境变量、ConfigMap 都是预设的，应用内无法直接获取如 Pod IP 等动态数据。

解决：Downward API 通过环境变量、downward API 卷来传递 Pod 的元数据。如：Pod 名称、IP、命名空间、标签、节点名、每个容器的 CPU、内存限制及其使用量。

#### 8.1.1  环境变量透传

在 Pod 定义中手动将容器需要的元数据，以环境变量的形式手动透传给容器：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward
spec:
  containers:
    - name: main
      image: busybox
      command:
        - "sleep"
        - "1000"
      env:
        - name: E_POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: E_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP # 运行时元数据
        - name: E_REQ_CPU
          valueFrom:
            resourceFieldRef: # 引用容器级别的数据，如请求的 CPU、内存用量等需引用 resourceFieldRef
              resource: requests.cpu
              divisor: 1m # 资源单位
        - name: E_LIMIT_MEM
          valueFrom:
            resourceFieldRef:
              resource: limits.memory
              divisor: 1Ki
```

效果：

```shell
k exec downward -it -- env | grep -e "^E_"
E_POD_NAMESPACE=default
E_POD_IP=172.17.0.8
E_REQ_CPU=0
E_LIMIT_MEM=2085844
```

缺点：无法通过环境传递 pod 标签和注解等可在运行时动态修改的元数据。

```shell
ERROR: error converting fieldPath: field label not supported: metadata.lebels.app
```



#### 8.1.2  通过 downwardAPI 卷

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward
  labels:
    foo: bar
  annotations:
    k1: v1
spec:
  containers:
    - name: main
      # ...
      volumeMounts:
        - name: downward
          mountPath: /etc/downward
  volumes:
    - name: downward # 定义一个名为 downward 的 DownwardAPI 卷，将元数据写入 items 下指定路径的文件中
      downwardAPI:
        items:
        - path: "container_request_memory"
          resourceFieldRef:
            containerName: main # 容器级别的元数据需指定容器名
            resource: requests.cpu
            divisor: 1m
        divisor: 1m
        - path: "labels" # "annotations" # pod 的标签和注解必须使用 downwardAPI 卷去访问
          fieldRef:
            fieldPath: metadata.labels
```

效果：k8s 会自动地将 pod 的标签和注解同步到 downward API 卷的指定文件中。

```shell
> kubectl exec downward-volume-pod -it -- ls /etc/downward
container_request_memory  pod_annotations           pod_labels

> kubectl exec downward-volume-pod -it -- cat /etc/downward/pod_annotations
key1="VALUE1"
key2="VALUE2\nVALUE20\nVALUE200\n"
kubernetes.io/config.seen="2020-04-28T04:15:29.722938998Z"
kubernetes.io/config.source="api"

> kubectl annotate pod downward-volume-pod new_key=NEW_VALUE               
pod/downward-volume-pod annotated

> kubectl exec downward-volume-pod -it -- tail -2 /etc/downward/pod_annotations
kubernetes.io/config.source="api"
new_key="NEW_VALUE"
```



### 8.2  与 k8s API 交互

问题：downward API 只能向应用暴露 1 个 Pod 的部分元数据，无法提供其他 Pod 和其他资源信息。

请求  k8s API：

- 外部：通过 kubectl proxy 中间请求转发代理。
- 内部：从 Pod 内验证 Secret 卷的 crt 证书、传递 token 到来对本 namespace 内的资源进行操作。

Pod 与 API 服务器交互流程：

- 应用通过 secret 卷下的 `ca.crt` 验证 API 地址
- 应用带上 TOKEN 授权
- 操作 pod 所在命名空间内的资源

```shell
> ls /var/run/secrets/kubernetes.io/serviceaccount/
ca.crt  # 验证 API 服务器证书，避免中间人攻击
namespace # 获取本地 pod 的 namespace: default
token # 通过 header 添加 "Authorization: Bearer $TOKEN" 方式来获取授权

> curl --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt https://kubernetes # 验证 API 地址
> export CURL_CA_BUNDLE=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

> TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
> curl -H "Authorization: Bearer $TOKEN" https://kubernetes # 授权

> NS=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)
> curl -H "Authorization: Bearer $TOKEN" https://kubernetes/api/v1/namespaces/$NS/pods # 操作本地命名空间资源
```

折中的 Pod 内部公共透明代理模式：在 pod 内部通过 ambassador 公共容器简化 API 服务器验证流程。普通容器走 HTTP 到 ambassador 容器，后者走 secret 流程走 HTTPS 与 k8s API 交互。



## 9. Deployment

场景：定义比例滚动升级，出错自动回滚。

### 9.1  纯手动更新运行在 Pod 内的应用程序

版本升级方式：

- 先删旧 Pod，再自动创建新 Pod：存在不可用间隔。

  修改 Pod spec.template 中的标签选择器，选择新版 tag 对应的应用，先手动删除旧版本，RS 自动创建新版容器。

- 创建新 Pod，同时逐步剔除旧 Pod：需两个版本应用都能对外服务，短时间内 Pod 数量翻倍。

  修改 Service 的 pod selector 蓝绿部署进度可控地将流量切换到新 Pod 上。

### 9.2  基于 RC 的滚动升级

问题：手动脚本将旧 Pod 缩容，新 Pod 扩容，易出错。

解决：使用 kubectl 请求 k8s API 来执行滚动升级：

```shell
# 指定需更新的 RC kubia-v1，用指定的 image 创建的新 RC 来替换
> kubectl rolling-update kubia-v1 kubia-v2 --image=wuyinio/kubia:v2  # 从 1.8 已移除
```

过程：kubectl 为新旧 Pod、新旧 RC 添加 deployment 标签，并向 k8s API 请求对旧 Pod 进行缩容，对新 Pod 扩容，透明地将 service 的标签匹配到新 pod 上。

原理：由客户端 kebectl 动态地修改两个 RC 的标签，缩容旧 Pod，扩容新 Pod，最终完成流量转移。



### 9.3 使用 Deployment 声明式升级

问题：RC 滚动升级是 kubectl 客户端控制升级，若网络断开连接则升级中断（如关闭终端），Pod 和 RC 会处于多版本混合态。

解决：在服务端使用高级资源 Deployment 声明来协调新旧两个 RS 的扩缩容，用户只定义最终目标收敛状态。

deployment 也分为 3 部分：标签选择器、期望副本数、Pod 模板：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubia
spec:
  replicas: 3
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
        - name: nodejs
          image: yinzige/kubia:v1
  selector:
    matchLabels:
      app: kubia
```

```shell
# 创建 deployment
> kubectl create -f kubia-deployment-v1.yaml --record # record 选项将记录历史版本号，用于后续回滚
> kubectl rollout status deployment kubia # rollout 显示部署状态

# deployment 用 pod 模板的哈希值创建 RS，再由 RS 创建 Pod 并管理，哈希值一致
> kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
kubia-5b9f8f4d84-nxmqd   1/1     Running   0          2s
kubia-5b9f8f4d84-q5wc5   1/1     Running   0          2s
kubia-5b9f8f4d84-r866t   1/1     Running   0          2s
> kubectl get rs
NAME               DESIRED   CURRENT   READY   AGE
kubia-5b9f8f4d84   3         3         3       9s
```

deployment 的升级策略：`spec.stratagy`

- RollingUpdate：渐进式删除旧 Pod。新旧版混合
- Recreate：一次性删除所有旧 Pod，重建新 Pod。中间服务不可用

#### 9.3.1  触发滚动升级

先指定 Pod 就绪后的等待时间：` kubectl patch deployment kubia -p '{"spec": {"minReadySeconds": 10}}'`

修改某个容器的镜像来触发升级：`kubectl set image deployment kubia nodejs=yinzige/kubia:v2`

注：触发升级需真正修改到 deployment 的字段。

原理：`kubectl get rs` 可看到保留了的新旧版 rs，deployment 资源在 k8s master 端会自动控制新旧 RS 的扩缩容。

#### 9.3.2  回滚

```shell
> kubectl rollout undo deployment kubia  # 回滚到上一次 deployment 部署的版本
> kubectl rollout history deployment kubia  # 创建 deployment 时 --record，此处显示版本
> kubectl rollout undo deployment kubia --to-reversion=1  # 回滚到指定 REVERSION，若手动删除了 RS 则无法回滚
> kubectl rollout pause deployment kubia # 暂停升级，在新 Pod 上进行金丝雀发布验证
> kubectl rollout resume deployment kubia # 恢复
```



### 9.4 结合探针控制升级速度

可配置 `maxSurge` 和 `maxIUnavailable` 来控制升级的最大超意外的 Pod 数量、最多容忍不可用 Pod 数量。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubia
spec:
  replicas: 3
  minReadySeconds: 10 # 新 Pod 就绪需等待 10s，才能继续滚动升级
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1 # 只允许最多超出一个非预期 pod，即只能逐个更新
      maxUnavailable: 0 # 不允许有不可用的 Pod，以确保新 Pod 能逐个替换旧的 Pod
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
        - name: nodejs
          image: yinzige/kubia:v3 # 接收请求 5s 后返回 500
          readinessProbe:
            periodSeconds: 1 # 定义 HTTP Get 就绪探针每隔 1s 执行一次
            httpGet:
              port: 8080
              path: /
  selector:
    matchLabels:
      app: kubia
```

```shell
# apply 对象不存在则创建，否则修改对象属性
> kubectl apply -f kubia-deployment-v3-with-readinesscheck.yaml
```

使用上述 deployment 从正常版 v2 升级到 bug 版 v3，据配置 v3 的第一个 Pod 会创建并在第 5s 被 Service 标记为不可用，将其从 endpoint 中移除，请求不会分发到该 v3 Pod，10s 内未就绪，最终部署自动停止。

如下：kubia-54f54bf655 并未加入到 kubia Service 的 endpoints 中

```yaml
> kubectl get endpoints
NAME             ENDPOINTS                                          AGE
kubia            172.17.0.12:8080,172.17.0.6:8080,172.17.0.8:8080   140m

> kubectl get pod -o wide                                   
NAME                     READY   STATUS    RESTARTS   AGE     IP            NODE   NOMINATED NODE  
kubia-54f54bf655-tgvt9   0/1     Running   0          63s     172.17.0.7    m01    <none>          
kubia-b669c877-8kx8c     1/1     Running   0          2m17s   172.17.0.6    m01    <none>          
kubia-b669c877-957fl     1/1     Running   0          113s    172.17.0.12   m01    <none>          
kubia-b669c877-n7hnb     1/1     Running   0          2m4s    172.17.0.8    m01    <none>          
```



## 10.  Stateful Set

场景：在有状态分部署存储应用中，Pod 的多副本有各自独立的 PVC 和 PV，pod 在新节点重建后需保证状态一致。

### 10.2  保证状态一致

- 一致的网络标识

  sts 创建的 pod 名字后缀按顺序从 0 递增，通常通过 headless Service 暴露整个集群的 pod，每个 pod 有独立的 DNS 记录，pod 重建后保证名称和主机名一致。

- 一致的存储

  sts 有 pod 模板和 PVC 模板，每个 pod 会绑定到唯一的 PVC 和 PV，重建后新 pod 会绑定到旧 PVC 和 PV 复用旧的存储。

### 10.3  使用 sts

三种必需资源：PV（若没有默认的 provisioner，则须手动创建）、控制 Service、sts 自身

- 创建 headless 

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: kubia
  spec:
    clusterIP: None
    selector:
      app: kubia
    ports:
      - port: 80
        name: http
  ```

- 创建 sts：PVC 模板会在 pod 运行前创建，并且绑定到默认 storage class 的 provisioner 创建的 PV 上

  ```yaml
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: kubia
  spec:
    serviceName: kubia # 绑定到 kubia 的 headless service
    replicas: 2
    template:
      metadata:
        labels:
          app: kubia
      spec:
        containers:
          - name: kubia
            image: yinzige/kubia-pet
            ports:
              - containerPort: 8080
                name: http
            volumeMounts:
              - mountPath: /var/data # PVC 绑定到 pod 目录
                name: data
  
    volumeClaimTemplates: # 动态 PVC 模板，运行时提前创建
      - metadata:
          name: data
        spec:
          resources:
            requests:
              storage: 1Mi
          accessModes:
            - ReadWriteOnce
    selector:
      matchLabels:
        app: kubia
  ```

  注：sts 的创建或 scale 扩缩容，都是一次只操作一个 Pod 避免出现竞争、数据不一致的情况，pod 操作顺序与副本数顺序增减一致。



## 11.  K8S 组件

k8s 中各组件通过 API 服务器的 event 事件流的通知机制进行解耦，各组件之间不会直接通信，相互透明。[组件](https://asksendai.com/how-to-detect-kubernetes-vulnerability-cve-2019-11246-using-falco/)：

![](https://mysieve-img.s3.amazonaws.com/pub/1564843777_2019_08_03_0dac5abc-584c-4be0-a63e-5eeaeca41452.png)

- etcd：分布式一致性 KV 存储，只与 API Server 交互，存储集群各种资源元数据。
- API Server：提供对集群资源的 CURD 接口，推送资源变更的事件流到监听端。
- Scheduler：监听 Pod 创建事件，筛选出符合条件的节点并选出最优节点，更新 Pod 定义后发布给 API Server

- 各种资源的 Controller：监听资源更新的事件流，检查副本数，同样更新元数据发布给 API Server
- kubectl：注册 Node 等待分配 Pod，告知 Docker 拉取镜像运行容器，随后向 API Server 会报 Pod 状态及指标
- kube-proxy：代理 Service 暴露的 IP 和 Port



### 11.2  Pod 创建流程

各控制器间通过 API Server 进行解耦：

![](https://images.yinzige.com/chain.png)



### 11.6  高可用

API Server 和 etcd 可有多节点。

为避免并发竞争，各控制器同一时间只能有一个实例运行，通过竞争写注解字段的方式进行选举，调度器同理。





## 12.  API Server 安全

