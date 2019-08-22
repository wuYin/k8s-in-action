## Ch2. Start

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

> kubectl scale rc kubia --replicas=5 # 修改 rc 期望的副本数，来水平伸缩应用
> kubectl get pod kubia-1ic8j -o wide # 显示 pod 详细列
```



## Ch3. Pod

### 3.2 创建 Pod

yaml pod 定义模块

- apiVersion：k8s API 版本
- kind：资源类型，如 Pod，Service
- metadata：元数据。如 pod 名称、标签、注解
- spec：规格。pod 内容的实际说明，如容器名称、卷

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
          protocol: TCP
```

```shell
> kubectl create -f kubia-manual.yaml # 从 yaml 文件创建 k8s 资源
> kubectl get pod kubia-manual -o yaml # 导出 pod 定义
> kubectl logs kubia-manual -c kubia # 查看 kubia-manual Pod 中 kubia 容器的日志，-c 显式指定容器名称
> kubectl logs kubia-manual --previous # 查看崩溃前的上一个容器日志

> minikube ssh && docker ps
> docker logs bdb67198848d  # 登录到 pod 运行时的节点，查看日志

> kubectl port-forward kubia-manual 8081:8080 # 配置端口转发，将本机的 8081 转发至 pod 的 8080，可用于调试等
```



### 3.3 标签组织 Pod

metadata.labels 的 kv pair 标签可组织任何 k8s 资源，保存标识信息。

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
> kubectl get pods --show-labels # 显示 labels
> kubectl get pods -L env # 只显示指定的标签
> kubectl label pod kubia-manual env=debug # 为指定的 pod 资源添加新标签
> kubectl label pod kubia-manual env=online --overwrite=true # 修改标签
> kubectl label pod kubia-manual env- # 删除标签
```



### 3.4 使用标签选择器

label selector 可筛选指定的  k8s 资源。

```shell
> kubectl get pods -l env=debug # 筛选 env 为 debug 的 pods  # get pod 与 get pods 无异
> kubectl get pods -l '!env' # 不含 env 标签的 pods # -L 筛选并显示标签值
> kubectl get pods -l 'env in (debug)' # in 筛选
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

### 3.5 使用注解

类似 label 的 kv pair 注释，但不用作标识，可以添加大量的数据块：

```shell
> kubectl annotate pod kubia-manual creator='k8s'
> kubectl describe pod kubia-manual # 出现在 annotation
```

### 3.6 命名空间

labels 会导致资源重叠，可用 namespace 将对象分配到不同组避免。

```shell
> s # 获取所有 namespace，默认 default 下
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

### 3.8 移除 pod

```shell
> kubectl delete pod -l env=debug # 删除指定标签的 pod
> kubectl delete pod --all # 删除当前 namespace 下的所有 pod 
> kubectl delete all --all # 删除所有类型资源的所有对象
```



## Ch4. ReplicationController

### 4.1 容器存活探针 Liveness Probe

容器的探针用于暴露给 k8s 检察容器中的应用进程是否正常。分为 3 类：

- HTTP Get：指定 IP:Port 和 Path，GET 请求的 response code 4xx / 5xx 则认为失败
- TCP Socket：是否能建立 TCP 连接
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

注：带探针的 Pod 仅由 Worker 节点的 kubelet 负责监控并重启，整个节点崩溃将丢失所有 Pod

### 4.2 ReplicationController

RC 监控 Pod 列表并根据模板增删、迁移 Pod。分为 3 部分：

- 标签选择器 label selector：确定要管理哪些 pod
- 副本数量 replica count：指定要运行的 pod 数量
- pod 模板 pod template：创建新的 pod 副本

注：修改标签选择器，会丢失无新标签的 pod 的管理，修改模板则不影响，可手动删除新建生效。

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
> kubectl delete rc kubia  --ascade=false # 删除 rc 时保留运行中的 pod
```



### 4.3 ReplicaSet

ReplicationController + 扩展的 label selector  = ReplicaSet

RS 能通过 `selector.matchLabels` 和 `selector.matchExpressio` 来扩展对 pod label 的筛选：

```yaml
apiVersion: apps/v1beta2
kind: ReplicaSet
metadata:
  name: kubia-matchexpression
spec:
  replicas: 3
  selector:
    matchExpressions: # 为 selector 添加额外的表达式
      - key: app  
        operator: In # 操作符：In 在标签列表中 / NotIn 不在标签列表中 / Exists 必须包含此 key / DoseNotExists 不包含 此 key
        values:
          - kubia
          - foo
      - key: type
        operator: Exists # 只对 key 的 Exists 操作无 values
  template:
    metadata:
      labels:
        app: kubia
        type: normal
    spec:
      containers:
        - name: kubia
          image: wuyinio/kubia
          ports:
          - containerPort: 8080
```

### 4.4 DaemonSet

DS 用于限制在所有节点或指定节点上运行一个指定的 Pod，常用于部署系统级用于如 kube-proxy

```yaml
apiVersion: apps/v1beta2
kind: DaemonSet
metadata:
  name: ssd-monitor
spec:
  selector:
    matchLabels:
      app: ssd-monitor
  template:
    metadata:
      labels:
        app: ssd-monitor
    spec:
      nodeSelector: # 指定只在 disk: ssd 的节点上运行当前的 pod
        disk: ssd
      containers:
        - name: main
          image: wuyinio/ssd-monitor
```

注：DS 资源不存在扩缩容的概念，它只确保每个选中的节点运行 1 个指定的 Pod，所以 `kubectl scale ds` 报错。

### 4.4 Job 与 CronJob

Job 资源指定任务完成后主动关闭 Pod 为 Completed，可配置为失败后重启或不重启，可配置为并发执行。

CronJob 资源会定期创建 Job 资源，Job 创建 Pod 运行，可配置 DeadlineSeconds



## Ch5. Service

### 5.1 介绍服务

由于 pod 调度后 IP 会变化，需使用 Service 服务给一组 Pod 提供不变的单一接入点（IP:Port）

```shell
> kubectl expose rc kubia --type=LoadBalancer --name kubia-http # 通过服务暴露 ClusterIP pod 服务给外部访问
```

```yaml
apiVersion: v1
kind: Service
metadata: 
  name: kubia
spec:
  ports:
    - name: http
      port: 80
      targetPort: 8080
    - name: https
      port: 443
      targetPort: 8443
  selector:
    app: kubia
```

注：创建名为 kubia 的 Service，将其 80 端口的请求分发给有 `app: kubia` 标签 Pod 的 8080 端口上。

```yaml
apiVersion: v1
kind: Service
metadata: 
  name: kubia-svc-session
spec:
  sessionAffinity: ClientIP # 指定了会话的亲和性，则每个 client 的请求会指向同一个 Pod
  ports:
    - port: 80
      targetPort: 8080
```

2 种服务发现：Pod 需知道服务的 IP 和 Port

- 环境变量：`kubectl exec kubia-qgtmw env` 会有 SERVICE_HOST 和 SERVICE_PORT 指向服务。

- DNS 发现：在 Pod 上可通过全限定域名 FQDN 访问服务：`<service_name>.<namespace>.svc.cluster.local`

  ```shell
  > kubectl exec kubia-qgtmw cat /etc/resolv.conf
  nameserver 10.96.0.10
  search default.svc.cluster.local svc.cluster.local cluster.local # 会
  options ndots:5
  ```

  在 Pod kubia-qgtmw 中可通过访问 `kubia.default.svc.cluster.local` 来访问 kubia 服务。



### 5.2 连接集群外部的服务

TODO: 重新梳理



## Ch6. Volume

### 6.1 卷介绍

- 问题：Pod 中每个容器的文件系统来自镜像，相互独立。
- 解决：使用存储卷，让容器访问外部磁盘空间、容器间共享存储。

卷是 Pod 生命周期的一部分，不是 k8s 资源对象。Pod 启动时创建，删除时销毁。用于 Pod 中挂载到多个容器进行文件共享。

卷类型：

- emptyDir：存放临时数据的空目录。
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
    - image: luksa/fortune
      name: html-generator
      volumeMounts:
        - name: html
          mountPath: /var/htdocs # 名为 html 的卷挂载到容器的 /var/htdocs 目录
    - image: nginx:alpine
      name: web-server
      volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html # 名为 html 的卷挂载到 /usr/share/nginx/html
          readOnly: true # 设置卷为只读
      ports:
        - containerPort: 80
          protocol: TCP
  volumes:
    - name: html
      emptyDir: {} # 在 Pod 中设置名为 html 的 emptyDir 卷，等待挂载
```

### 6.3 hostPath 卷

hostPath 卷的数据不跟随 Pod 生命周期，下一个调度至此节点的 Pod 能继续使用前一个 Pod 留下的数据。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb
spec:
  volumes:
    - name: mongodb-data
      hostPath:
        path: /tmp/mongodb # 使用 hostPath 节点本地文件系统来存储 mongodb 的文件
  containers:
    - image: mongo
      name: mongodb
      volumeMounts:
        - name: mongodb-data
          mountPath: /data/db
      ports:
        - containerPort: 27017
          protocol: TCP
```

重新创建 Pod 后发现 mongodb 的数据被保留：

```shell
> use mystore;
switched to db mystore
> db.foo.insert({k:'v'})
WriteResult({ "nInserted" : 1 })
> db.foo.find()
{ "_id" : ObjectId("5d5cd1a33251e991fc9bb9bb"), "k" : "v" }
{ "_id" : ObjectId("5d5cd209dcc650a35b6ac406"), "k" : "v" }
```



### 6.5 持久化卷 PV、持久化卷声明 PVC

PV 与 PVC 用于解耦 Pod 与底层存储。PV、PVC 与底层存储关系：

![](http://images.yinzige.com/2019-08-21-052202.png)

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
  hostPath:
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
  storageClassName: ""
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

注：PV 是集群范围的，PVC 和 Pod 是命名空间范围的。

可创建 StorageClass 存储类资源，用于分类 PV，在 PV 中引用指定的 PVC

![](http://images.yinzige.com/2019-08-21-054515.png)





## Ch7. ConfigMap 与 Secret

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

二者等同于 Pod 中的 `command` 和 `args`

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

注：args 中的数字必须用 `""` 界定，字符串反而不用。

### 7.3 为容器设置环境变量

在 Pod 配置容器部分 `spec.containers.env` 指定即可：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune3s
spec:
  containers:
    - image: luksa/fortune:env
      env:
        - name: INTERVAL # 对应到镜像 fortune 中的 $INTERVAL
          value: "5"
      name: html-generator
# ...      
```

缺点：多个环境下无法复用 Pod 定义，需将配置项解耦。

### 7.4 ConfigMap

```shell
# 可从 kv 字面量、独立文件、目录下所有文件创建 cm
> kubectl create configmap fortune-config --from-literal=sleep-interval=25 # 从 kv 字面量创建 cm
> kubectl create configmap fortune-config --from-file=nginx-conf=my-nginx-config.conf # 指定 k 的文件创建 cm
```

两种方式将 cm 中的值传递给 Pod 中的容器：

- 设置环境变量或命令行参数

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
          - name: INTERVAL # 为 html-generator 容器设置环境变量，值取 CM fortune-config 中的 sleep-interval 值
            valueFrom:
              configMapKeyRef:
                name: fortune-config
                key: sleep-interval 
  # ...
  ```

  可使用 `kubectl get cm fortune-config -o yaml` 查看 CM 的配置项。

- 使用 CM 将配置暴露为文件（配置项长），创建 Pod 时挂载 CM 配置项作为文件：

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: fortune-configmap-volume
  spec:
    volumes:
      - name: html
        emptyDir: {} 
      - name: config
        configMap:
          name: fortune-config # 使用指定名字的 cm 卷
    containers:
      - image: nginx:alpine 
        name: web-server
        volumeMounts:
          - name: config
            mountPath: /etc/nginx/conf.d # 将 CM fortune-config 所有配置项挂载至此目录下
            readOnly: true
  # ...          
  ```

  注：可添加 `items` 来挂载单一文件，`subPath` 来挂载至某一文件或文件夹而不隐藏容器的原有文件。

使用 `kubectl edit cm fortune-config` 修 CM 后，容器中对应挂载的配置项会自动修改，但应用程序不会自动重新加载。



### 7.5 Secret

存储敏感的配置数据，其配置条目会以 Base64 编码二进制后存储：

```shell
> kubectl create secret generic fortune-https --from-file=fortune-https
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-https
spec:
  volumes:
    - name: certs
      secret:
        secretName: fortune-https # 声明卷
# ...        
  containers:
    - image: nginx:alpine
      name: web-server
      volumeMounts:
        - name: config
          mountPath: /etc/nginx/conf.d
          readOnly: true
        - name: certs
          mountPath: /etc/nginx/certs/ # 将 secret 卷挂载到指定位置
          readOnly: true
      ports:
        - containerPort: 80
        - containerPort: 443
# ...
```

对应到 nginx 的配置：

```nginx
server {
  listen 443;
	server_name demo.com;
	ssl_certificate certs/https.cert;
	ssl_certificate_key certs/https.key; # 使用挂载到 certs 目录下的证书文件
	# ...
}
```



## Ch8. Pod Metadata 与 k8s API

场景：从 Pod 中的容器应用进程访问 Pod 元数据及其他资源。

### 8.1 使用 Downward API 传递 Pod Metadata

问题：配置数据如环境变量、ConfigMap 都是预设的，应用内无法直接获取如 Pod IP 等动态数据。

解决：Downward API 通过环境变量、downward API 卷来传递 Pod 的元数据。如：Pod 名称、IP、命名空间、标签、节点名、每个容器的 CPU、内存限制及其使用量。

- 通过环境变量

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: downward
  spec:
    containers:
      - image: busybox
        command: ["sleep", "99999"]
        resources:
          requests:
            cpu: 15m
            memory: 100Ki
          limits: #...
        env:
          - name: POD_IP
            valueFrom: 
              fieldRef:
                fieldPath: status.podIP # 直接引用 pod manifest 元数据的字段，而非具体值
          - name: CONTAINER_CPU_REQUEST_MILLICORES
            valueFrom:
              resourceFieldRef: # 引用容器级别的数据，如请求的 CPU、内存用量等需引用 resourceFieldRef
                resource: requests.cpu
                divisor: 1m # 设置资源单位
  ```

- 通过 downwardAPI 卷

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
          - path: "podName" # "namespace"
            fieldRef:
              fieldPath: metadata.name
          - path: "labels" # "annotations" # pod 的标签和注解必须使用 downwardAPI 卷去访问
            fieldRef:
              fieldPath: metadata.labels
  ```



### 8.2 与 k8s API 交互

问题：downward API 只能向应用暴露 1 个 Pod 的部分元数据，无法提供其他 Pod 和其他资源信息。

解决：通过 kubectl proxy 中间请求转发代理、从 Pod 内验证 Secret 证书的方式来交互。

Pod 与 API 服务器交互流程：

- 应用通过 secret 卷下的 `ca.crt` 验证 API 地址
- 应用带上 TOKEN 授权
- 操作 pod 所在命名空间内的对象

```shell
> curl --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt https://kubernetes # 验证 API 地址
> export CURL_CA_BUNDLE=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

> TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
> curl -H "Authorization: Bearer $TOKEN" https://kubernetes # 授权

> NS=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)
> curl -H "Authorization: Bearer $TOKEN" https://kubernetes/api/v1/namespaces/$NS/pods # API 交互
```



## Ch9. Deployment

场景：零停机升级应用。

### 9.1 更新运行在 Pod 内的应用程序

版本升级方式：

- 删除所有旧 Pod，创建新 Pod：存在不可用间隔。

  修改 Pod spec.template 中的标签选择器，选择新版 tag 对应的应用。手动 delete 旧版后自动创建新版。

- 逐步创建新 Pod，逐步删除旧 Pod：需两个版本应用都能对外服务。

  短时间内需两倍 Pod 数量，后修改 Service 蓝绿部署将流量切换到新 Pod 上。

### 9.2 基于 RC 的滚动升级

问题：手动脚本将旧 Pod 缩容，新 Pod 扩容，易出错。

解决：使用 kubectl 请求 k8s API 来执行滚动升级：

```shell
# 指定需更新的 RC kubia-v1，用指定的 image 创建的新 RC 来替换
> kubectl rolling-update kubia-v1 kubia-v2 --image=wuyinio/kubia:v2 
```

过程：kubectl 为新旧 Pod、新旧 RC 添加 deployment 标签，并向 k8s API 请求对旧 Pod 进行缩容，对新 Pod 扩容。

### 9.3 使用 Deployment 声明式升级

问题：kubectl 是客户端升级，若网络断开连接则升级中断（如关闭终端），Pod 和 RC 会处于多版本混合态。

解决：在服务端使用高级资源 Deployment 声明来协调新旧两个 RC 的扩缩容。

deployment 也分为 3 部分：标签选择器、期望副本数、Pod 模板：

```yaml
apiVersion: apps/v1beta1 # 属于 apps API 组
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
        - image: wuyinio/kubia:v1
          name: nodejs
```

```shell
# 创建 deployment
> kubectl create -f kubia-deployment-v1.yaml --record # record 选项将记录历史版本号，用于后续回滚
> kubectl rollout status deployment kubia # rollout 显示部署状态

# deployment 用 pod 模板的哈希值创建 RS，再由 RS 创建 Pod 并管理
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

```shell
> kubectl set image deployment kubia nodejs=wuyinio/kubia:v3  # 使用 set image 修改包含指定容器的镜像，保留旧 RS
> kubectl rollout undo deployment kubia  # 回滚到上一次 deployment 部署的版本
> kubectl rollout history deployment kubia  # 创建 deployment 时 --record，此处显示版本
> kubectl rollout undo deployment kubia --to-reversion=1  # 回滚到指定 REVERSION，若手动删除了 RS 则无法回滚
> kubectl rollout pause deployment kubia # 暂停升级，在新 Pod 上进行金丝雀发布验证
> kubectl rollout resume deployment kubia # 恢复
```

### 9.4 结合探针控制升级速度

可配置 `maxSurge` 和 `maxIUnavailable` 来控制升级的最大超意外的 Pod 数量、最多容忍不可用 Pod 数量。

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: kubia
spec:
  replicas: 3
  minReadySeconds: 10 # 新 Pod 创建后需等待 10s，才能继续滚动升级
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0 # 不允许有不可用的 Pod，以确保新 Pod 能逐个替换旧的 Pod
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
        - image: wuyinio/kubia:v3 # v3 的 kubia 在五次 HTTP 请求后出错返回 500 
          name: nodejs
          readinessProbe:
            periodSeconds: 1 # 定义 HTTP Get 就绪探针每隔 1s 执行一次
            httpGet:
              path: /
              port: 8080
```

```shell
# apply 对象不存在则创建，否则修改对象属性
> kubectl apply -f kubia-deployment-v3-with-readinesscheck.yaml
```

使用上述 deployment 从正常版 v2 升级到 bug 版 v3，据配置 v3 的第一个 Pod 会创建并在第 5s 被 Service 标记为不可用，将其从 endpoint 中移除，请求不会分发到该 v3 Pod，10s 内未就绪，最终部署自动停止。



## Ch10. Stateful Set



























































