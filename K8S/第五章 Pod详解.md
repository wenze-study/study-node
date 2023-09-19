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

