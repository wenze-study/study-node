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

