# 第五章 Pod详解

本章节将详细介绍 Pod 资源的各种配置和原理。

## 一、Pod 介绍

### 1、Pod 结构

![K8S-Pod](https://study-node-md.oss-cn-beijing.aliyuncs.com/2023%2F09%2F15%2F1694746653-af4273dac14d12abe70de1bfdf39b5ef-image-20230915105733304.png)

每个 Pod 中都可以包含一个或者多个容器，这些容器可以分为两类：

- 用户程序所在的容器，数量可多可少

- Pause 容器，这是每个 Pod 都会有的一个**根容器**，它的作用有两个：

    - 可以以它为依据，评估整个 Pod 的健康状态

    - 可以在根容器上设置 IP 地址，其他容器都以此 IP（Pod IP），来实现 Pod 内部的网络通信

        **这里是 Pod 内部的通讯，Pod 之间的通讯采用的是虚拟二层网络技术来实现，我们当前环境用的是 Flannel**。

### 2、Pod 定义

下面是 Pod 的资源清单：

```yaml
apiVersion: v1 # 必选，版本号，例如：v1
kind: Pod # 必选，资源类型，例如：Pod
metadata: # 必选，元数据
  name: string # 必选，Pod 名称
  namespace: string # Pod 所属的命名空间，默认为：default
  labels: # 自定义标签列表
    - name: string
spec: # 必选，Pod 中容器的详细定义
  containers: # 必选，Pod 中容器列表
    - name: string # 必选，容器名称
      image: string # 必选，容器的镜像名称
      imagePullPolicy: [Always|Never|IfNotPresent] # 获取镜像的策略
      command: [string] # 容器的启动命令列表，如不指定，使用打包时使用的启动命令
      args: [string] # 容器的启动命令参数列表
      workingDir: string # 容器的工作目录
      volumeMounts: # 挂载到容器内部的存储卷配置
        - name: string # 引用 pod 定义的共享存储卷的名称，需用 volumes[] 部分定义的卷名
          mountPath: string # 存储卷在容器内 mount 的绝对路径，应少于 512 字符
          readOnly: boolean # 是否为只读模式
      ports: # 需要暴露的端口号列表
        - name: string # 端口的名称
          containerPort: int # 容器需要监听的端口号
          hostPort: int # 容器所在主机需要监听的端口号，默认与 Container 相同
          protocol: string # 端口协议，支持 TCP 和 UDP，默认为：TCP
      env: # 容器运行前需设置的环境变量列表
        - name: string # 环境变量的名称
          value: string # 环境变量的值
      resources: # 资源限制和请求的设置
        limits: # 资源限制的设置
          cpu: string # CPU 的限制，单位为 core 数，将用于 docker run --cpu-shares 参数
          memory: string # 内存限制，单位可以为 Mib/Gib，将用于 docker run --memory 参数
        requests: # 资源请求的设置
          cpu: string # CPU 请求，容器启动的初始可用数量
          memory: string # 内存请求，容器启动的初始可用数量
      lifecycle: # 生命周期钩子
        postStart: # 容器启动后立即执行此钩子，如果执行失败，会根据重启策略进行重启
        preStop: # 容器终止前执行此钩子，无论结果如何，容器都会终止
      livenessProbe: # 对 Pod 内个容器健康检查的设置，当探测无响应几次后将自动重启该容器
        exec: # 对 Pod 容器内检查方式设置为 exec 方式
          command: [string] # exec 方式需要指定的命令或者脚本
        httpGet: # 对 Pod 容器健康检查方法设置为 HttpGet，需要指定 Path、port
        	path: string
        	port: number
        	host: string
        	scheme: string
        	HttpHeaders:
        	  name: string
        	  value: string
        tcpSocket: # 对 Pod 内容器健康检查方式设置为 tcpSocket 方式
          port: number
        initialDelaySeconds: 0 # 容器启动完成后首次探测的时间，单位为秒
        timeoutSeconds: 0 # 对容器健康检查探测等待响应的超时时间，单位秒，默认 1 秒
        periodSeconds: 0 # 对容器监控检查的定期探测时间设置，单位秒，默认 10 秒一次
        successThreshold: 0
        failureThreshold: 0
        securityContext:
          previleged: false
  restartPolicy: [Always|Never|OnFailure] # Pod 的重启策略
  nodeName: string # 设置 NodeName 表示将该 Pod 调度到指定的名称的 node 节点上
  nodeSelector: object # 设置 NodeSelector 表示将该 Pod 调度到包含这个 label 的 node 上
  imagePullSecrets: # Pull 镜像时使用的 secret 名称，以 key: secretkey 格式指定
    - name: string
  hostNetwork: false # 是否使用主机网络模式，默认为 false，如果设置为 true，表示使用宿主机网络
  volumes: # 在该 pod 上定义共享存储卷列表
    name: string # 共享存储卷名称（volumes 类型有很多种）
    emptyDir: {} # 类型为 emptyDir 的存储卷，与 Pod 同生命周期的一个临时目录。为空值
    hostPath: string # 类型为 hostPath 的存储卷，表示挂载 Pod 所在宿主机的目录
      path: string # Pod 所在宿主机的目录，将被用于同期中 mount 的目录
    secret: # 类型为 secret 的存储卷，挂载集群与定义的 secret 对象到容器内部
      secretname: string
      items:
        - key: string
          path: string
    configMap: # 类型为 configMap 的存储卷，挂载与定义的 configMap 对象到容器内部
      name: string
      items:
        - key: string
          path: string
```



```shell
# 小提示：
#   在这里，可通过一个命令来查看各种资源的可配置项
#   kubectl explain 资源类型          查看某种资源可以配置的一级属性
#   kubectl explain 资源类型，属性     查看属性的子属性
kubectl explain pod

kubectl explain pod.metadata
```

在 kubernetes 中基本所有资源的一级属性都是一样的，主要包含 5 部分：

| 属性字段   | 类型     | 说明                                                         |
| ---------- | -------- | ------------------------------------------------------------ |
| apiVersion | <string> | 版本，由 kubernetes 内部定义，版本必须可以用`kubectl api-versions`查询到 |
| kind       | <string> | 类型，由 kubernetes 内部定义，类型必须可以用`kubectl api-resources`查询到 |
| metadata   | <object> | 元数据，主要是资源标识和说明，常用的有 name、namespace、labels 等 |
| spec       | <object> | 描述，这是配置中最重要的一部分，里面是对各种资源配置的详细描述 |
| status     | <object> | 状态信息，里面的内容不需要定义，由 kubernetes 自动生成       |

在上面的属性中，spec 是接下来研究的重点，继续看下它的常见子属性：

| 属性字段      | 类型       | 说明                                                         |
| ------------- | ---------- | ------------------------------------------------------------ |
| containers    | <[]Object> | 容器列表，用于定义容器的详细信息                             |
| nodeName      | <string>   | 根据 nodeName 的值将 Pod 调度到指定的 Node 节点上            |
| nodeSelector  | <map[]>    | 根据 NodeSelector 中定义的信息选择将该 Pod 调度到包含这些 label 的 Node 上 |
| hostNetwork   | <boolean>  | 是否使用主机网络模式，默认为 false，如果设置为 true，表示使用宿主机网络 |
| volumes       | <[]Object> | 存储卷，用于定义 Pod 上面挂载的存储信息                      |
| restartPolicy | <string>   | 重启策略，表示 Pod 在遇到故障的时候的处理策略                |

## 二、Pod 配置

本小节主要研究`pod.spec.containers`属性，这也是 pod 配置中最为关键的一项配置。

```shell
kubectl explain pod.spec.containers

KIND:       Pod
VERSION:    v1

RESOURCE: containers <[]Container> # 数组，代表可以有多个容器

FIELDS:
  name	<string> -required- # 容器名称
  image	<string> # 容器需要的镜像地址
  imagePullPolicy	<string> # 镜像拉取策略
  command	<[]string> # 容器的启动命令列表，如不指定，使用打包时使用的启动命令
  args	<[]string> # 容器的启动命令需要的参数列表
  env	<[]EnvVar> # 容器环境变量的配置
  ports	<[]ContainerPort> # 容器需要暴露的端口号列表
  resources	<ResourceRequirements> # 资源限制和资源请求的设置
```

### 1、基本配置

创建 pod-base.yaml，内容如下：

```yaml
apiVersion: v1
kind: Namespace
metadata: 
  name: dev
  
---

apiVersion: v1
kind: Pod
metadata:
  name: pod-base
  namespace: dev
  labels:
    user: wenze
spec:
  containers:
    - name: nginx
      image: nginx:1.24
    - name: busybox
      image: busybox:1.30
```

上面定义了一个比较简单的 Pod 配置，里面有两个容器：

- nginx：用 1.24 版本的 nginx 镜像创建，（nginx 是一个轻量级 web 容器）
- busybox：用 1.30 版本的 busybox 镜像创建，（busybox 是一个小巧的 linux 命令集合）

```shell
# 创建 pod
kubectl create -f pod-base.yaml

# 查看 pod 状况
# READY 1/2 : 表示当前 Pod 中有两个容器，其中一个准备就绪，一个未就绪
# RESARTS   : 重启次数，因为有 1 个容器故障了，Pod 一直在重启试图恢复它
kubectl get pod -n dev
NAME       READY   STATUS             RESTARTS      AGE
pod-base   1/2     CrashLoopBackOff   1 (10s ago)   11s

# 可以通过 describe 查看内部的详情
# 此时已经运行起来了一个基本的 Pod，虽然它暂时有问题
kubectl describe pod pod-base -n dev
```

### 2、镜像拉取

创建 pod-imagepullpolicy.yaml 文件，内容如下：

```yaml
apiVersion: v1
kind: Namespace
metadata: 
  name: dev
  
---

apiVersion: v1
kind: Pod
metadata:
  name: pod-imagepullpolicy
  namespace: dev
  labels:
    user: wenze
spec:
  containers:
    - name: nginx
      image: nginx:1.24
      imagePullPolicy: Always # 用于设置镜像拉取策略
    - name: busybox
      image: busybox:1.30
```

`imagePullPolicy`，用于设置镜像拉取策略，kubernetes 支持配置三种拉取策略：

- `Always`：总是从远程仓库拉取镜像（一直远程下载）
- `IfNotPresent`：本地有则使用本地镜像，本地没有则从远程仓库拉取镜像（本地有就本地，本地没远程下载）
- `Never`：只使用本地镜像，从不去远程仓库拉取，本地没有就报错（一直使用本地）

> 默认值说明：
>
> ​	如果镜像 tag 为具体版本号，默认策略是：`IfNotPresent`
>
> ​	如果镜像 tag 为：latest（最终版本），默认策略是：`Always`

```shell
# 创建 pod
kubectl create -f pod-imagepullpolicy.yaml

# 查看 pod 详情
# 此时明显可以看到 nginx 镜像有一步 Pulling image 'nginx:1.17.2' 的过程
kubectl describe pod pod-imagepullpolicy -n dev
```

### 3、启动命令

​	在前面的案例中，一直有一个问题没有解决，就是 busybox 容器一直没有成功运行，那么到底是什么原因导致这个容器的故障呢？

​	原来 busybox 并不是一个程序，而是类似于一个工具类的集合，kubernetes 集群启动管理后，它会自动关闭。解决办法就是让其一直在运行，这就用到了 command 配置。

创建 pod-command.yaml 文件，内容如下：

```yaml
apiVersion: v1
kind: Namespace
metadata: 
  name: dev
  
---

apiVersion: v1
kind: Pod
metadata:
  name: pod-command
  namespace: dev
  labels:
    user: wenze
spec:
  containers:
    - name: nginx
      image: nginx:1.24
    - name: busybox
      image: busybox:1.30
      command: ["/bin/sh", "-c", "touch /tmp/hello.txt;while true; do /bin/echo $(date + %T) >> /tmp/hello.txt; sleep 3; done;"]
```

command，用于在 pod 中的容器初始化完毕之后运行的一个命令。

> 稍微解释下上面命令的意思：
>
> ​	`"/bin/sh", "-c"`，使用 sh 执行命令
>
> ​	`touch /tmp/hello.txt; ` 创建一个 `/tmp/hello.txt` 文件
>
> ​	`while true;do /bin/echo $(data + %T) >> /tmp/hello.txt;sleep 3; done;`每隔3 秒文件中写入当前时间

```shell
# 创建 pod
kubectl create -f pod-command.yaml

# 查看 pod 状态
# 此时发现两个 pod 都正常运行了
kuectl get pods pod-command -n dev

# 进入 pod 中的 busybox 容器，查看文件内容
# 补充一个命令：kubectl exec pod名称 -n 命名空间 -it -c 容器名称 /bin/sh 在容器内部执行命令
# 使用这个命令就可以进入某个容器的内部，然后进行相关操作了
# 比如，可以查看 txt 文件的内容
kubectl exec pod-command -n dev -it -c busybox /bin/sh

tail -f /tmp/hello.txt
```

```txt
特别说明：
  通过上面发现 command 已经可以完成启动命令和传递参数的功能，为什么这里还要提供一个 args 选项，用于传递参数呢？
  1. 如果 command 和 args 均没有写，那么用 Dockerfile 的配置。
  2. 如果 command 写了，但 args 没有写，那么 Dockerfile 默认的配置会被忽略，执行输入的 command。
  3. 如果 command 没写，但 args 写了，那么 Dockerfile 中配置的 ENTRYPOINT 的命令会被执行，使用当前 args 的参数
  4. 如果 command 和 args 都写了，那么 Dockerfile 的配置被忽略，执行 command 并追加上 args 参数
```

### 4、环境变量

创建 pod-env,yaml 文件，内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-env
  namespace: dev
  labels:
    user: wenze
spec:
  containers:
    - name: busybox
      image: busybox:1.30
      command: ["/bin/sh", "-c", "touch /tmp/hello.txt;while true; do /bin/echo $(date + %T) >> /tmp/hello.txt; sleep 3; done;"]
      env: # 设置环境变量列表
      	- name: "username"
      	  value: "admin"
      	- name: "password"
      	  value: "123456"
```

env，环境变量，用于在 pod 中的容器设置环境变量。

```shell
# 创建 pod
kubectl create -f pod-env.yaml

# 进入容器，输出环境变量
kubectl exec pod-env -n dev -c busybox -it /bin/sh

echo $username
admin

echo $password
123456
```

这种方式不是很推荐，推荐将这些配置单独存储在配置文件中，这种方式将在后面介绍。

### 5、端口设置

本小节来介绍容器的端口设置，也就是 containers 的 ports 选项。

首先看下 ports 支持的子选项：

```shell
kubectl explain pod.spec.containers.ports

KIND:       Pod
VERSION:    v1

FIELD: ports <[]ContainerPort>

FIELDS:
  name	<string> # 端口名称，如果指定，必须保证 name 在 pod 中是唯一的
  containerPort	<integer> -required- # 容器要监听的端口(0<x<65536)
  hostPort	<integer> # 容器要在主机上公开的端口，如果设置，主机上只能运行容器的一个副本（一般省略）
  hostIP	<string> # 要将外部端口绑定到的主机 IP（一般省略）
  protocol	<string> # 端口协议。必须是 UDP、TCP 或者 SCTP。默认为：TCP。
```

接下来，编写一个测试案例，创建 pod-ports.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-ports
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx:1.24
      ports: # 设置容器暴露的端口列表
        - name: nginx-port
          containerPort: 80
          protocol: TCP
```

```shell
# 创建 pod
kubectl create -f pod-ports.yaml

# 查看 pod
# 在下面可以明显看到配置信息
kubectl get pod pod-ports -n dev -o yaml

kubectl describe pod pod-ports -n dev
```

访问容器中的程序需要使用的是：`PodIP:containerPort`

### 6、资源配额

​	容器中的程序要运行，肯定是要占用一定资源的，比如 CPU 和内存等，如果不对某个容器的资源做限制，那么它就可能吃掉大量资源，导致其他容器无法运行。针对这种情况，kubernetes 提供了对内存和 CPU 的资源进行配额的机制，这种机制主要通过 resources 选项实现，他有两个子选项：

- limits：用于限制运行时容器的最大占用资源，当容器占用资源超过 limits 时会被终止，并进行重启。
- requests：用于设置容器需要的最小资源，如果环境资源不够，容器将无法启动。

可以通过上面两个选项设置资源的上下限。

接下来，编写一个测试案例，创建 pod-resources.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-resources
  namespace: dev
spec:
  containers:
    - name: nginx:
      image: nginx:1.24
      resources: # 资源配额
        limits: # 限制资源（上限）
          cpu: "2" # CPU 限制，单位是 core 数
          memory: "10Gi" # 内存限制
        requests: # 限制资源（下限）
          cpu: "1" # CPU 限制，单位是 core 数
          memory: "10Mi" # 内存限制
```

在这对 cpu 和 memory 的单位做一个说明：

- cpu：core 数，可以为整数或小数
- memory：内存大小，可以使用 Gi、Mi、G、M 等形式

```shell
# 运行 pod
kubectl create -f pod-resources.yaml

# 查看发现 pod 运行正常
kubectl get pods pod-resources -n dev

# 接下来，停止 Pod
kubectl delete -f pod-resources.yaml

# 编辑 pod，修改 resources.requests.memory 的值为 10Gi

# 再次启动 pod
kubectl create -f pod-resources.yaml

# 查看 pod 状态，发现 pod 启动失败
kubectl get pod pod-resources -n dev -o wide

# 查看 pod 详情会发现，如下提示
kuebctl describe pod pod-resources -n dev

Warning  FailedScheduling  17s   default-scheduler  0/1 nodes are available: 1 Insufficient memory. `(内存不足)`
```

## 三、Pod 生命周期

我们一般将 pod 对象从创建至终的这段时间范围称为 pod 的生命周期，他主要包含下面的过程：

- pod 创建过程
- 运行初始化容器（init container）过程
- 运行主容器（main container）过程
    - 容器启动后钩子（post start）、容器终止前钩子（pre stop）
    - 容器的存活性探测（liveness probe）、就绪性探测（readiness probe）
- pod 终止过程

![K8S-Pod生命周期](https://study-node-md.oss-cn-beijing.aliyuncs.com/2023%2F09%2F19%2F1695116536-762ba92624f980abe7295348a5f85e0a-20230919174214.png)

在整个生命周期中，Pod 会出现 5 种**状态（相位）**，分别如下：

- 挂起（Pending）：apiserver 已经创建了 pod 资源对象，但它尚未被调度完成或者仍处于下载镜像的过程中
- 运行中（Running）：pod 已经被调度至某节点，并且所有容器都已经被 kubelet 创建完成
- 成功（Succeeded）：pod 中的所有容器都已经成功终止并且不会被重启
- 失败（Failed）：所有容器都已经终止，但至少有一个容器终止失败，即容器返回了非 0 值的退出状态
- 未知（Unknown）：apiserver 无法正常获取到 pod 对象的状态信息，通常由网络通信失败所导致

### 1、创建和终止

#### pod的创建过程

1. 用户通过 kubectl 或者其他 api 客户端提交需要创建的 pod 信息给 apiServer
2. apiServer 开始生成 pod 对象的信息，并将信息存入 etcd，然后返回确认信息至客户端
3. apiServer 开始反映 etcd 中的 pod 对象的变化，其他组件使用 watch 机制来跟踪检查 apiServer 上的变动
4. scheduler 发现有新的 pod 对象要创建，开始为 pod 分配主机并将结果信息更新至 apiSever
5. node 节点上的 kubelet 发现有 pod 调度过来，尝试调用 docker 启动容器，并将结果回送至 apiServer
6. apiServer 将接收到的 pod 状态信息存入 etcd 中

![K8S-Kubernetes组件](https://study-node-md.oss-cn-beijing.aliyuncs.com/2023%2F09%2F08%2F1694165394-161421ed07c7365693fbffdc5949df50-image-20230908172953610.png)

#### pod 的终止过程

1. 用户向 apiServer 发送删除 pod 对象的命令
2. apiServer 中的 pod 对象信息会随着时间的推移而更新，在宽限期内（默认 30s），pod 被视为 dead
3. 将 pod 标记为 terminating 状态
4. kubelet 在监控到 pod 对象转为 terminating 状态的同时启动 pod 关闭过程
5. 端点控制器监控到 pod 对象的关闭行为时将其从所有匹配到此端点的 service 资源的端点列表中移除
6. 如果当前 pod 对象定义了 preStop 钩子处理器，则在其标记为 terminating 后即会以同步的方式启动执行
7. pod 对象中的容器进程收到停止信号
8. 宽限期结束后，若 pod 中还存在仍在运行的进程，那么 pod 对象会收到立即终止的信号
9. kubelet 请求 apiServer 将此 pod 资源的宽限期设置为 0 从而完成删除操作，此时 pod 对于用户已不可见

### 2、初始化容器

初始化容器是在 pod 的主容器启动之前要运行的容器，主要是做一些主容器的前置工作，它具有两大特性：

1. 初始化容器必须运行完成直至结束，若某初始化容器运行失败，那么 kubernetes 需要重启它直到成功完成
2. 初始化容器必须按照定义的顺利执行，当且仅当前一个成功之后，后面的一个才能运行

初始化容器有很多的应用场景，下面列出的是最常见的几个：

- 提供主容器镜像中不具备的工具程序或自定义代码
- 初始化容器要先于应用容器串行启动并运行完成，因此可用于延后应用容器的启动直至其依赖的条件得到满足

接下来做一个案例，模拟下面这个需求：

​	假设要以主容器来运行 nginx，但是要求在运行 nginx 之前先要能够连接上 mysql 和 redis 所在服务器

​	为了简化测试，事先规定好 mysql`(192.168.109.201)`和 redis`(192.168.109.202)`服务器的地址

创建 pod-initcontainer.yaml，内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-initcontainer
  namespace: dev
spec:
  containers:
    - name: main-container
      image: nginx:1.17.1
      ports:
        - name: nginx-port
          containerPort: 80
  initContainers:
    - name: test-mysql
      image: busybox:1.30
      command: ["sh", "-c", "until ping 172.18.0.4 -c 1; do echo waiting for mysql ...; sleep 2; done;"]
    - name: test-redis
      image: busybox:1.30
      command: ["sh", "-c", "until ping 172.18.0.6 -c 1; do echo waiting for redis ...; sleep 2; done;"]
```

```shell
# 创建 pod
kubectl create -f pod-initcontainer.yaml

# 查看 pod
# 发现 pod 卡在启动第一个初始化容器过程中，后面的容器不会运行
kubectl describe pod pod-initcontainer -n dev

# 动态查看 pod
kubectl get pods pod-initcontainer -n dev -w

# 接下来新开一个 shell，为当前服务器新增两个 ip，观察 pod 的变化
ifconfig ens33:1 192.168.109.201 netmask 255.255.255.0 up
ifconfig ens33:1 192.168.109.202 netmask 255.255.255.0 up
```

### 3、钩子函数

钩子函数能够感知自身生命周期中的事件，并在相应的时刻到来时运行用户指定的程序代码。

kubernetes 在主容器的启动之后和停止之前提供了两个钩子函数：

- `post start`： 容器创建之后执行，如果失败了会重启容器
- `pre stop`：容器终止之前执行，执行完成之后容器将成功终止，在其完成之前会阻塞删除容器的操作

钩子处理器支持使用下面三种方式定义动作：

- Exec 命令：在容器内执行一次命令

    ```yaml
    ...
      lifecycle:
        postStart:
          exec:
            command:
              - cat
              - /tmp/healthy
    ...
    ```

- TCPSocket：在当前容器尝试访问指定的 socket

    ```yaml
    ...
      lifecycle:
        postStart:
          tcpSocket:
            port: 8080
    ...
    ```

- HTTPGet：在当前容器中向某 url 发起 http 请求

    ```yaml
    ...
      lifecycle:
        postStart:
          httpGet:
            path: / # URI地址
            port: 80 # 端口号
            host: 192.168.109.100 # 主机地址
            scheme: HTTP # 支持的协议，http 或者 https
    ...
    ```

接下来，以 exec 方式为例，演示下钩子函数的使用，创建 pod-hook-exec.yaml 文件，内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-hook-exec
  namespace: dev
spec:
  containers:
    - name: main-container
      image: nginx:1.24
      ports:
        - name: nginx-port
          containerPort: 80
      lifecycle:
        postStart:
          exec: # 在容器启动的时候执行一个命令，修改掉 nginx 的默认首页内容
            command: ["/bin/sh", "-c", "echo postStart... > /usr/share/nginx/html/index.html"]
        preStop:
          exec: # 在容器停止之前停止 nginx 服务
            command: ["/usr/sbin/nginx", "-s", "quit"]
```

```shell
# 创建 pod
kubectl create -f pod-hook-exec.yaml

# 查看 pod
kubectl get pods pod-hook-exec -n dev -o wide
NAME            READY   STATUS    RESTARTS   AGE   IP              NODE      
pod-hook-exec   1/1     Running   0          22s   192.168.194.5   orbstack  

# 访问 nginx 页面
curl 192.168.194.5:80
```

### 4、容器探测

​	容器探测用于检测容器中的应用实例是否正常工作，是保障业务可用性的一种传统机制。如果经过探测，实例的状态不符合预期，那么 kubernetes 就会把该问题实例“摘除”，不承担业务流量。kubernetes 提供了两种探针来实现容器探测，分别是：

- liveness probes：存活性探针，用于检测应用实例当前是否处于正常运行状态，如果不是，k8s 会重启容器
- readiness probes：就绪性探针，用于检测应用实例当前是否可以接收请求，如果不能，k8s 不会转发流量

> linvenessProbe 决定是否重启容器，readinessProbe 决定是否将请求转发给容器

上面两种探针目前均支持三种探测方式：

- Exec 命令：在容器内执行一次命令，如果命令执行的退出码为 0，则认为程序正常，否则不正常

    ```yaml
    ---
      livenessProbe:
        exec:
          command: 
            - /cat
            - /tmp/healthy
    ---
    ```

- TCPSocket：将会尝试访问一个用户容器的端口，如果能够建立这条连接，则认为程序正常，否则不正常

    ```yaml
    ---
      livenessProbe:
        tcpSocket:
          port: 8080
    ---
    ```

- HttpGet：调用容器内 Web 应用的 URL，如果返回的状态码在 200 到 399 之间，则认为程序正常，否则不正常

    ```yaml
    ---
      livenessProbe:
        httpGet:
          path: / # URI地址
          port: 80 # 端口号
          host: 127.0.0.1 # 主机地址
          scheme: HTTP # 支持的协议，HTTP 或者 HTTPS
    ---
    ```

下面以 liveness probes 为例，做几个演示：

**方式一：Exec**

创建 pod-liveness-exec.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-liveness-exec
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx:1.24
      ports:
        - name: nginx-port
          containerPort: 80
      livenessProbe:
        exec:
          command: ["/bin/cat", "/tmp/hello.txt"] # 执行一个查看文件的命令
```

创建 pod，观察效果

```shell
# 创建 Pod
kubectl create -f pod-liveness-exec.yaml

# 查看 pod 的详情
kubectl describe pod pod-liveness-exec -n dev
# 观察上面的信息就会发现 nginx 容器启动之后就进行了健康检查
# 检查失败之后，容器被 kill 掉，然后尝试进行重启（这是重启策略的作用，后面讲解）
# 稍等一会之后，再观察 pod 信息，就可以看到 RESTARTS 不再是 0，而是一直增长

# 当然接下来，可以修改成一个存在的文件，或修改命令，比如 /bin/ls /tmp，再试，结果就正常了......
```

**方式二：TCPSocket**

创建 pod-liveness-tcpsocket.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-liveness-tcpsocket
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx:1.24
      ports:
        - name: nginx-port
          containerPort: 80
      livenessProbe:
        tcpSocket:
          port: 8080 # 尝试访问的端口
```

创建 pod，观察效果

```shell
# 创建Pod
kubectl create -f pod-liveness-tcpsocket.yaml

# 查看 Pod 的详情
kubectl describe pod pod-liveness-tcpsocket -n dev

# 观察上面的细腻系，发现尝试访问 8080 端口，但是失败了
# 稍等一会之后，再观察 pod 信息，就可以看到 RESTARTS 不再是 0，而是一直增长

# 当然接下来，可以修改成一个可以访问的端口，比如 80，再试，结果就正常了
```

**方式三：HttpGet**

创建 pod-liveness-httpget.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-liveness-httpget
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx:1.24
      ports:
        - name: nginx-port
          containerPort: 80
      livenessProbe:
        httpGet: # 其实就是访问 http://127.0.0.1:80/hello
          scheme: HTTP # 支持的协议，http 或者 https
          port: 80 # 端口号
          path: /hello # URI地址
```

创建 pod，观察效果

```shell
# 创建 pod
kubectl create -f pod-liveness-httpget.yaml

# 查看 Pod 详情
kubectl describe pod pod-liveness-http -n dev

# 观察上面信息，尝试访问路径，但是未找到，出现 404 错误
# 稍等一会之后，再观察 pod 信息，就可以看到 RESTART 不再是 0，而是一直增长

# 当然接下来，可以修改成一个可以访问的路径 path，比如 /，再试，结果就正常了......
```

​	至此，已经使用 livenessProbe 演示了三种探测方式，但是查看 livenessProbe 的子属性，会发现除了这三种方式，还有一些其他的配置，在这里一并解释下：

```shell
kubectl explain pod.spec.containers.livenessProbe

FIELD: livenessProbe <Probe>
FIELDS:
  exec	<ExecAction>
  tcpSocket	<TCPSocketAction>
  httpGet	<HTTPGetAction>
  initialDelaySeconds	<integer> # 容器启动后等待多少秒执行第一次探测
  timeoutSeconds	<integer> # 探测超时时间，默认 1 秒，最小 1 秒
  periodSeconds	<integer> # 执行探测的频率，默认是 10 秒，最小 1 秒
  failureThreshold	<integer> # 连续探测失败多少次才被认定为失败，默认是 3，最小值是 1
  successThreshold	<integer> # 连续探测成功多少次才被认定为成功，默认是 1
  grpc	<GRPCAction> # 指定涉及到 grpc 的操作
```

下面稍微配置两个，演示下效果即可：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-liveness-httpget
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx:1.24
      ports:
        - name: nginx-port
          containerPort: 80
      livenessProbe:
        httpGet: # 其实就是访问 http://127.0.0.1:80/hello
          scheme: HTTP # 支持的协议，http 或者 https
          port: 80 # 端口号
          path: /hello # URI地址
        initialDelaySeconds: 30 # 容器启动后 30s 开始探测
        timeoutSeconds: 5 # 探测超时时间为 5s
```

### 5、重启策略

​	在上一节中，一旦容器探测出现了问题，kubernetes 就会对容器所在的 Pod 进行重启，其实这是由 pod 的重启策略决定的，pod 的重启策略有 3 种，分别如下：

- Always：容器失效时，自动重启该容器，这也是默认值。
- OnFailure：容器终止运行切退出码不为 0 时重启
- Never：不论状态为何，都不重启该容器

​	重启策略适用于 Pod 对象中的所有容器，首次需要重启的容器，将在其需要时立即进行重启，随后再次需要重启的操作将由 kubelet 延迟一段时间后进行，且反复的重启操作的延迟时长以此为 10s、20s、40s、80s、160s 和 300s，300s 是最大延迟时长。

创建 pod-restartpolicy.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-restartpolicy
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx:1.24
      ports:
        - name: nginx-port
          containerPort: 80
      livenessProbe:
        httpGet: 
          scheme: HTTP
          port: 80
          path: /hello
  restartPolicy: Never # 设置重启策略为 Never
```

运行 Pod 测试

```shell
# 创建 Pod
kubectl create -f pod-restartpolicy.yaml

# 查看 pod 详情，发现 nginx 容器失败
kubectl describe pods pod-restartpolicy -n dev

# 多等一会，再观察 pod 的重启次数，发现一直是 0，并未重启
kubectl get pods pod-restartpolicy -n dev
```

## 四、Pod 调度

​	在默认情况下，一个 Pod 在哪个 Node 节点上运行，是由 Scheduler 组件采用相应的算法计算出来的，这个过程是不受人工控制的。但是在实际使用中，这并不满足的需求，因为很多情况下，我们想控制某些 Pod 到达某些节点上，那么应该怎么做呢？这就要求了解 kubernetes 对 Pod 的调度规则，kubernetes 提供了四大类调度方式：

- 自动调度：运行在哪个节点上完全由 Scheduler 经过一系列的算法计算得出
- 定向调度：NodeName、NodeSelector
- 亲和性调度：NodeAffinity、PodAffinity、PodAntiAffinity
- 污点（容忍）调度：Taints、Toleration

### 1、定向调度

​	定向调度，指的是利用在 pod 上声明 nodeName 或者 nodeSelector，以此将 Pod 调度到期望的 node 节点上。注意，这里的调度是强制的，这就意味着即使要调度的目标 Node 不存在，也会向上面进行调度，只不过 pod 运行失败而已。

#### NodeName

​	NodeName 用于强制约束将 Pod 调度到指定的 Name 的 Node 节点上。这种方式，其实是直接跳过 Scheduler 的调度逻辑，直接写入 PodList 列表。

接下来，实验一下：创建一个 pod-nodename.yaml 文件

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodename
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx:1.24
  nodeName: node1 # 指定调度到 node1 节点上
```

```shell
# 创建 pod
kubectl create -f pod-nodename.yaml

# 查看 pod 调度到 node 属性，确实是调度到了 node1 节点上
kubectl get pods pod-nodename -n dev -o wide

# 接下来，删除 pod，修改 nodeName 的值为 node3（并没有 node3 节点）
kubectl delete -f pod-nodename.yaml

vim pod-nodename.yaml
kubectl create -f pod-nodename.yaml

# 再次查看，发现已经向 Node3 节点调度，但是由于不存在 node3 节点，所以 pod 无法正常运行
kubectl get pods pod-nodename -n dev -o wide
```

#### NodeSelector

​	NodeSelector用于将 pod 调度到添加了指定标签的 node 节点上。它是通过 kubernetes 的 label-selector 机制实现的，也就是说，在 pod 创建之前，会由scheduler 使用 MatchNodeSelector 调度策略进行 label 匹配，找出目标 node，然后将 pod 调度到目标节点，该匹配规则是强制约束。

接下来，实验一下：

1. 首先分别为 node 节点添加标签：

    ```shell
    kubectl label nodes node1 nodeenv=pro
    
    kubectl label nodes node2 nodeenv=test
    ```

2. 创建一个 pod-nodeselector.yaml 文件，并使用它创建 pod

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: pod-nodeselector
      namespace: dev
    spec:
      containers:
        - name: nginx
          image: nginx:1.24
      nodeSelector:
        nodeenv: pro # 指定调度到具有 nodeenv=pro 的标签上
    ```

    ```shell
    # 创建pod
    kubectl create -f pod-nodeselector.yaml
    
    # 查看 pod 调度到 NODE 属性，确实是调度到了 node1 节点上
    kubectl get pods pod-nodeselector -n dev -o wide
    
    # 接下来，删除 pod，修改 nodeSelector 的值为 nodeenv:uat(不存在打此标签的节点)
    kubectl delete -f pod-nodeselector.yaml
    vim pod-nodeselector.yaml
    kubectl create -f pod-nodeselector.yaml
    
    # 再次查看，发现 pod 无法正常运行，NODE 的值为 none
    kubectl get pods -n dev -o wide
    
    # 查看详情，发现 node selector 匹配失败的提示
    kubectl describe pods pod-nodeselector -n dev
    ```

### 2、亲和性调度

​	上一节，介绍了两种定向调度的方式，使用起来非常方便，但是也有一定的问题，那就是如果没有满足条件的 Node，那么 Pod 将不会被运行，即使在即群众还有可用 Node 列表也不行，这就限制了它的使用场景。

​	基于上面的问题，kubernetes 还提供了一种亲和性调度（Affinity）。它在 NodeSelector 的基础之上的进行了扩展，可以通过配置的形式，实现优先选择满足条件的 Node 进行调度，如果没有，也可以调度到不满足条件的节点上，使调度更加灵活。

Affinity 主要分为三类：

- nodeAffinity(node 亲和性)：以 node 为目标，解决 pod 可以调度到哪些 node 的问题
- podAffinity(pod 亲和性)：以 pod 为目标，解决 pod 可以和哪些已存在的 pod 部署在同一个拓补域中的问题。
- podAntiAffinity(pod 反亲和性)：以 pod 为目标，解决 pod 不能和哪些已存在 pod 部署在同一个拓补域中的问题。

> 关于亲和性（反亲和性）使用场景的说明：
>
> **亲和性**：如果两个应用频繁交互，那就有必要利用亲和性让两个应用的尽可能的靠近，这样可以减少因网络通信而带来的性能损耗。
>
> **反亲和性**：当应用的采用多副本部署时，有必要采用反亲和性让各个应用实例打散分布在各个 node 上，这样可以提高服务的高可用性。

#### NodeAffinity

首先来看一下`NodeAffinity`的可配置项：

```markdown
pod.spec.affinity.nodeAffinity
  requiredDuringSchedulingIgnoredDuringExecution Node节点必须满足指定的所有规则才可以，相当于硬限制
    nodeSelectorTerms  节点选择列表
      matchFields  按节点字段列出的节点选择器要求列表
      matchExpressions  按节点标签列出的节点选择器要求列表（推荐）
        key  键
        values 值
        operator 关系符 支持 Exists，DoesNotExist，In，NotIn，Gt，Lt
  preferredDuringSchedulingIgnoredDuringExecution 优先调度到满足指定的规则的 Node，相当于软限制（倾向）
    preference 一个节点选择器项，与相应的权重相关联
      matchFields  按节点字段列出的节点选择器要求列表
      matchExpressions 按节点标签列出的节点选择器要求列表（推荐）
        key  键
        values 值
        operator 关系符 支持 In，NotIn，Exists，DoesNotExist，Gt，Lt
    weight 倾向权重，在范围 1-100
```

```markdown
关系户的使用说明：

- matchExpressions:
  - key: nodeenv              # 匹配存在标签的 key 位 nodeenv 的节点
    operator: Exists
  - key: nodeenv              # 匹配标签的 key 位 nodeenv，且 value 是"xxx"或"yyy"的节点
    operator: In
    values: ["xxx", "yyy"]
  - key: nodeenv              # 匹配标签的 key 位 nodeenv，且 value 大于"xxx"的节点
    operator: Gt
    values: "xxx"
```

接下来首先演示一下`requiredDuringSchedulingIgnoredDuringExecution`，

创建 pod-nodeaffinity-required.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodeaffinity-required
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx:1.24
  affinity:  # 亲和性设置
    nodeAffinity: # 设置 node 亲和性
      requiredDuringSchedulingIgnoredDuringExecution: # 硬限制
        nodeSelectorTerms:
          - matchExpressions: # 匹配 env 的值在 ["xxx","yyy"]中的标签
            - key: nodeenv
              operator: In
              values: ["xxx","yyy"]
```

```shell
# 创建 pod
kubectl create -f pod-nodeaffinity-required.yaml

# 查看 Pod 的详情
# 发现调度失败，提示 node 选择失败
kubectl describe pod pod-nodeaffinity-required -n dev

# 接下来，停止 pod
kubectl delete -f pod-nodeaffinity-required.yaml

# 修改文件，将 values: ["xxx","yyy"] ---> ["pro", "yyy"]
vim pod-nodeaffinity-required.yaml

# 再次启动
kubectl create -f pod-nodeaffinity-required.yaml

# 此时查看，发现调度成功，已经将 pod 调度到了 node1 上
kubectl get pods pod-nodeaffinity-required -n dev -o wide
```

接下来再演示一下 `requiredDuringSchedulingIgnoredDuringExecution`

创建 pod-nodeaffinity-preferred.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodeaffinity-required
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx:1.24
  affinity:  # 亲和性设置
    nodeAffinity: # 设置 node 亲和性
      requiredDuringSchedulingIgnoredDuringExecution: # 软限制
        nodeSelectorTerms:
          - matchExpressions: # 匹配 env 的值在 ["xxx","yyy"]中的标签
            - key: nodeenv
              operator: In
              values: ["xxx","yyy"]
```

```shell
# 创建 pod
kubectl create -f pod-nodeaffinity-preferred.yaml

# 查看 pod 状态（运行成功）
kubectl get pod pod-nodeaffinity-perferred -n dev
```

```markdown
NodeAffinity规则设置注意事项：
  1 如果同时定义了 nodeSerector 和 nodeAffinity，那么必须两个条件都得到满足，Pod 才能运行在指定的 Node 上
  2 如果 nodeAffinity 指定了多个 nodeSelectorTerms，那么只需要其中一个能够匹配成功即可
  3 如果 nodeSelectorTerms 中有多个 matchExperssions，则一个节点必须满足所有的才能匹配成功
  4 如果一个 pod 所在的 Node 在 Pod 运行期间其标签发生了改变，不再符合该 Pod 的节点亲和性需求，则系统将忽略此变化
```

#### PodAffinity

PodAffinity 主要实现以运行的 Pod 为参照，实现让新创建的 Pod 跟参照 Pod 在一个区域的功能。

首先来看一下`PodAffinity`的可配置项：

```markdown
pod.spec.affinity.podAffinity
  requiredDuringSchedulingIgnoredDuringExecution 硬限制
    namespaces 指定参照 pod 的 namespace
    topologyKey  指定调度作用域
    labelSelector  标签选择器
      matchExpressions  按节点标签列出的节点选择器要求列表（推荐）
        key  键
        values 值
        operator 关系符 支持 In，NotIn，Exists，DoesNotExist
      matchLabels  指多个 matchExpressions 映射的内容
  preferredDuringSchedulingIgnoredDuringExecution 软限制 
    podAffinityTerm 选项  
      namespaces
      topologyKey
      labelSelector
        matchExpressions
          key  键
          values 值
          operator
        matchLabels
    weight  倾向权重，在范围 1-100
```

```markdown
topologyKey用于指定调度时作用域，例如：
  如果指定为 kubernetes.io/hostname，那就是以 Node 节点为区分范围
  如果指定为 beta.kubernetes.io/os，则以 Node 节点的操作系统类型来区分
```

接下来，演示下`requiredDuringSchedulingIgnoredDuringExecution`

1）首先创建一个参照 Pod，pod-podaffinity-target.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-podaffinity-target
  namespace: dev
  labels:
    podenv: pro # 设置标签
spec:
  containers:
    - name: nginx
      image: nginx:1.24
  nodeName: node1 # 将目标 pod 明确指定到 node1 上
```

```shell
# 启动目标 pod
kubectl create -f pod-podaffinity-target.yaml

# 查看 pod 状况
kubectl get pods pod-podaffinity-target -n dev
```

2）创建 pod-podaffinity-required.yaml，内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-podaffinity-required
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx:1.24
  affinity: # 亲和性设置
    podAffinity: # 设置亲和性
      requiredDuringSchedulingIgnoredDuringExecution: # 硬限制
        - labelSelector:
            matchExpressions: # 匹配 env 的值在 ["xxx", "yyy"]中的标签
              - key: podenv
                operator: In
                values: ["xxx","yyy"]
          topologyKey: kubernetes.io/hostname
```

上面配置表达的意思是：新 Pod 必须要与拥有标签 nodeenv=xxx 或者 nodeenv=yyy的 pod在同一个 Node 上，显然现在没有这样的 Pod，接下来运行测试一下：

```shell
# 启动 pod
kubectl create -f pod-podaffinity-required.yaml

# 查看 pod 状态，发现未运行
kubectl get pods pod-podaffinity-required -n dev

# 查看详细信息
kubectl describe pods pod-podaffinity-required -n dev

# 接下来修改 values: ["xxx","yyy"] --> values: ["pro", "yyy"]
# 意思是：新 Pod 必须要与拥有标签 nodeenv=xxx 或者 nodeenv=yyy 的 pod 在同一 Node 上
vim pod-podaffinity-required.yaml

# 然后重新创建 pod，查看效果
kubectl delete -f pod-podaffinity-required.yaml
kubectl create -f pod-podaffinity-required.yaml

# 发现此时 Pod 运行正常
kubectl get pods pod-podaffinity-required -n dev
```

关于`PodAffinity`的`preferredDuringSchedulingIgnoredDuringExecution`，这里不再演示。

#### PodAntiAffinity

PodAntiAffinity 主要实现以运行的 Pod 为参照，让新创建的 Pod 跟参照 Pod 不在一个区域中的功能。

它的配置方式和选项跟 PodAffinity 是一样的，这里不再做详细解释，直接做一个测试案例。

1）继续使用上个案例中目标 pod

```shell
kubectl get pods -n dev -o wide --show-labels
```

2）创建 pod-podantiaffinity-required.yaml，内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-podantiaffinity-required
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx:1.24
  affinity: # 亲和性设置
    podAntiAffinity: # 设置亲和性
      requiredDuringSchedulingIgnoredDuringExecution: # 硬限制
        - labelSelector:
            matchExpressions: # 匹配 env 的值在 ["pro"]中的标签
              - key: podenv
                operator: In
                values: ["pro"]
          topologyKey: kubernetes.io/hostname
```

上面配置表达的意思是：新 Pod 必须要与拥有标签 nodeenv=pro的 pod 不在同一 Node 上，运行测试一下。

```shell
# 创建 pod
kubectl create -f pod-podantiaffinity-required.yaml

# 查看 pod
# 发现调度到了 node2 上
kubectl get pods pod-podantiaffinity-required -n dev -o wide
```

### 3、污点和容忍

#### 污点（Taints）

​	前面的调度方式都是站在 Pod 的角度上，通过在 Pod 上添加属性，来确定 Pod 是否要调度到指定的 Node 上，其实我们也可以站在 Node 的角度上，通过在 Node 上添加**污点**属性，来决定是否允许 Pod 调度过来。

​	Node 被设置上污点之后就和 Pod 之间存在了一种相斥的关系，进而拒绝 Pod 调度进来，设置可以将已经存在的 Pod 驱逐出去。

污点的格式为：`key=value:effect`，key 和 value 是污点的标签，effect 描述污点的作用，支持如下三个选项：

- PreferNoSchedule：kubernetes 将尽量避免把 Pod 调度到具有该污点的 Node 上，除非没有其他节点可调度
- NoSchedule：kubernets 将不会把 Pod 调度到具有该污点的 Node 上，但不会影响当前 Node 上已存在的 Pod
- NoExecute：kubernetes 将不会把 Pod 调度到具有该污点的 Node 上，同时也会讲 Node 上已存在的 Pod 隔离

![K8S-污点](https://study-node-md.oss-cn-beijing.aliyuncs.com/2023%2F10%2F07%2F1696689890-bf0a9ef24d28083afe26297c7cdc93c0-image-20231007224449162.png)

使用 kuberctl 设置和去除污点的命令示例如下：

```shell
# 设置污点
kubectl taint nodes node1 key=value:effect

# 去除 node1 单个污点
kubectl taint nodes node1 key:effect-

# 去除所有污点
kubectl taint nodes node1 key-
```

接下来，演示下污点的效果：

1. 准备节点 node1（为了演示效果更加明显，暂时停止 node2 节点）
2. 为 node1 节点设置一个污点：`tag=wenze:PreferNoSchedule`；然后创建 pod1（pod1 可以）
3. 修改为 node1 节点设置一个污点：`tag=wenze:NoSchedule`；然后创建 pod2（pod1 正常，pod2 失败）
4. 修改为 node1 节点设置一个污点：`tag=wenze:NoExecute`；然后创建 pod3（3 个 pod 都失败）

```shell
# 为 node1 设置污点（PreferNoSchedule）
kubectl taint nodes node1 tag=wenze:PreferNoSchedule

# 创建 pod1
kubectl run taint1 --image=nginx:1.24 -n dev
kubectl get pods -n dev -o wide

# 为 node1 设置污点（取消 PreferNoSchedule，设置 NoSchedule）
kubectl taint nodes node1 tag:PreferNoSchedule-
kubectl taint nodes node1 tag=wenze:NoSchedule

# 创建 pod2
kubectl run taint2 --image=nginx:1.24 -n dev
kubectl get pods taint2 -n dev -o wide

# 为 node1 设置污点（取消 NoSchedule，设置 NoExecute）
kubectl taint nodes node1 tag:NoSchedule-
kubectl taint nodes node1 tag=wenze:NoExecute

# 创建 pod3
kubectl run taint3 --image=nginx:1.24 -n dev
kubectl get pods -n dev -o wide
```

```markdown
小提示：
  使用 kubeadm 搭建的集群，默认就会给 master 节点添加一个污点标记，所以 pod 就不会调度到 master 节点上。
```

#### 容忍（Toleration）

​	上面介绍了污点的作用，我们可以在 node 上添加污点用于拒绝 pod 调度上来，但是如果就是想将一个 pod调度到一个有污点的 node 上去，这时候应该怎么做呢？这就要使用到**容忍**。

![K8S-容忍](https://study-node-md.oss-cn-beijing.aliyuncs.com/2023%2F10%2F07%2F1696690856-c2f39342b64f1bfc2bf61bb8dd242bba-image-20231007230056006.png)

> 污点就是拒绝，容忍就是忽略，Node 通过污点拒绝 pod 调度上去，Pod 通过容忍忽略拒绝

下面先通过一个案例看下效果：

1. 上一小节，已经在 node1 节点上打上了`NoExecute`的污点，此时 pod 是调度不上去的
2. 本小节，可以通过给 pod 添加容忍，然后将其调度上去

创建 pod-toleration.yaml，内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-toleration
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx:1.24
  tolerations:  # 添加容忍
    - key: "tag" # 要容忍的污点的 key
      operator: "Equal" # 操作符
      value: "wenze" # 容忍的污点的 value
      effect: "NoExecute" # 添加容忍的规则，这里必须和标记规则相同
```

```shell
# 添加容忍之前的 pod
kubectl get pods -n dev -o wide

# 添加容忍之后的 pod
kubectl get pods -n dev -o wide
```

下面看一下容忍的详细配置：

```shell
kubectl explain pod.spec.tolerations

---

FIELDS:
  key	<string> # 对应着要容忍的污点的键，空意味着匹配所有的键
  value	<string> # 对应着要容忍的污点的值
  operator	<string> # key-value 的运算符，支持 Equal 和 Exists（默认）
  effect	<string> # 对应污点的 effect，空意味着匹配所有影响
  tolerationSeconds	<integer> # 容忍时间，当 effect 为 NoExecute 时生效，表示 pod 在 Node 上的停留时间
```









