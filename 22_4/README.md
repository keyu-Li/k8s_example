## Pod

### Pod的实现原理

需要注意的是，pod只是一个逻辑概念，k8s实际处理的还是linux宿主机操作系统上的Namespace和Cgroups，并不存在一个pod边界和隔离环境。
所以说，pod是一组共享了某些资源的容器！Pod里的所有资源共享一个Network Namespace，并且```可以```共享同一个Volume。

凡是调度，网络，存储，安全相关的属性，基本上都是pod级别的

凡是与linux namespace相关的属性，也一定是pod级别的
我们知道Pod中的容器共享一个Network Namespace，我们可以通过设置spec.shareProcessNamespace:true 来使同一个pod的容器共享一个PID namespace

Pod这个复杂的API对象，其实只是对容器进行进一步的抽象和封装

#### Pod的创建过程

首先会创建一个infra容器，其他的容器会通过join network namespace与infra容器关联在一起

### 容器设计模式

用户想要在一个容器中跑多个不相关的应用时，首先应该考虑的是，是不是更应该被描述成一个Pod里的多个容器。

### linux namespace

linux支持8种namespace：

- cgroup用来隔离根目录
- IPC用于隔离系统消息队列
- Network隔离网络
- Mount隔离挂载点
- PID隔离进程
- User隔离用户和用户组
- UTS隔离主机名nis域名
- time用来隔离时间

### pod的重要字段

- nodeSelector：一个供用户将Pod和Node进行绑定的字段
- nodeName：如果这个字段被赋值，则表示这个pod已经经过了调度，调度的结果就是赋值的节点的名字，```可以通过设置这个字段来骗过调度器```
- hostAliases：用来设置hosts文件，建议使用这种方法来设置hosts文件中的内容，因为如果直接修改hosts文件，当pod迁移之后会覆盖这些内容
- hostNetwork,hostIPC,hostPID:共享宿主机的相关namespace
- containers,initContainers:配置pod中的容器信息
  - imagePullPolicy：镜像的pull策略
  - lifecycle：容器状态发生变化时执行一系列操作
    - postStart：在执行entrypoint之后执行，不保证entrypoint执行完成
    - preStop在容器被杀死之前执行，”优雅退出“

#### pod生命周期

字段为pod.status.phase

- Pending: pod的yaml文件已经提交给k8s，API对象已经被创建并保存在Etcd中，但是这个对象中的某些容器因为一些原因没有被顺利创建，比如调度不成功。
- Running：Pod已经调度成功，并且与一个节点绑定，它包含的容器已经被创建，至少一个容器正在运行
- Succeeded：Pod所有容器正常运行完毕，并且已经退出
- Failed：pod中至少有一个容器以不正常的状态退出
- Unknown：pod的状态不能持续的汇报被kubelet汇报给kube-apiserver，这可能是主从（master，kubelet）节点通信的问题

这些状态还可以继续细分出一组Conditions，比如PodScheduled Ready Initialized UnSchedulable

### Projected Volume

k8s支持的PV共有四种

- Secret
- ConfigMap
- Downward API
- ServiceAccountToken

前三种都可以使用环境变量的方式进行配置

#### Sercret

把Pod想要访问的加密数据保存在Etcd中
实验见./PV/Secret
注意：

- 更新会有延迟，在使用Secret信息连接数据库时，要写好重试和超时的逻辑

#### ConfigMap

configMap和Secret相似，区别是configMap中存储的时不需要加密的配置信息，用法和Secret相似

#### Downward API

他的作用是让用户能获得这个pod本身的信息
实验见/PV/DownwardAPI

Downward API能获得的信息一定是pod启动之前就能确定的信息
如果想要获取pod启动后的内容，可以考虑启动一个sidecar

#### Service Account

Service Account是一个特殊的Secret，它是k8s进行权限分配的对象。像这样的 Service Account 的授权信息和文件，实际上保存在它所绑定的一个特殊的 Secret 对象里的。这个特殊的 Secret 对象，就叫作 ServiceAccountToken。任何运行在k8s上的应用，都需要使用ServiceAccountToken里保存的授权信息，也就是token，这样才能合法地访问API server

对于每一个pod，在创建的时候，k8s都会给这个pod挂载一个默认的ServiceAccountToken：

```bash
default/var/run/secrets/kubernetes.io/serviceaccount # ls
ca.crt     namespace  token
```

这种把k8s客户端以容器的方式运行在及群里，然后使用default Service Account 自动授权的方式，被称作“InClusterConfig”。

### 容器健康检查和恢复机制

pod通过探针Probe的方式来进行健康检测，kubelet会根据Probe的返回值决定容器的状态，而不是直接以容器镜像是否运行为依据。
实验见/PV/Probe

## 编排

https://github.com/kubernetes/kubernetes/tree/master/pkg/controller
所有的编排控制器都在这个文件夹下面 ，他们都遵循k8s项目的一个通用编排模式，即：控制循环

```go

for {
  实际状态 := 获取集群中对象X的实际状态（Actual State）
  期望状态 := 获取集群中对象X的期望状态（Desired State）
  if 实际状态 == 期望状态{
    什么都不做
  } else {
    执行编排动作，将实际状态调整为期望状态
  }
}

```

对象的实际状态来源；

- kubelet通过心跳汇报容器状态和节点状态
- 监控系统保存的应用监控数据
- 控制器主动收集的信息

期望状态一般来自于用户提交的yaml文件

### deployment

Deployment 这个 template 字段里的内容，跟一个标准的 Pod 对象的 API 定义，丝毫不差。

类似 Deployment 这样的一个控制器，实际上都是由上半部分的控制器定义（包括期望状态），加上下半部分的被控制对象的模板组成的。

#### ReplicaSet

一个 ReplicaSet 对象，其实就是由副本数目的定义和一个 Pod 模板组成的。
更重要的是，Deployment 控制器实际操纵的，正是这样的 ReplicaSet 对象，而不是 Pod 对象。

ReplicaSet通过控制器模式，保证pod的数量永远等于指定的个数
deployment 同样通过“控制器模式”，来操作 ReplicaSet 的个数和属性，进而实现“水平扩展 / 收缩”和“滚动更新”这两个编排动作。

#### 滚动更新

用户提交一个deployment后，deployment controller会立即生成一个replicaset，replicaset的名称由deployment和随机字符串组成。
这个随机字符串可以保证pod的名称不会与其他的pod混淆

将一个集群中正在运行的多个 Pod 版本，交替地逐一升级的过程，就是“滚动更新”。这个过程通过创建新的replicaset来完成，旧的relicaset相当于history，遇到问题可以回退。

Pod的health check是滚动更新发挥作用的关键，单纯依靠判断pod是否处于Running状态，不能判断容器中的服务是否已经正常启动

在spec.strategy中配置滚动更新的参数

```bash
# 直接编辑etcd中的api对象
kubectl edit deployment/name 
# 查看滚动更新的状态
kubectl rollout status deployment/name
# 里面的Events可以看到滚动更新的流程
kubectl describe deployment name
# rs的状态查看
kubectl get rs
# 只更改image
kubectl set image deployment/name image_key=image_value
# 回退到上一个版本
kubectl rollout undo deployment/name
# 查看history
kubectl rollout history deployment/name
# 回退到具体的版本
kubectl rollout undo deployment/name --revision=num
```

如果每个更新操作都生成一个rs，可能会浪费资源，k8s提供一种操作只生成一个rs

kubectl rollout pause：进入”暂停状态“，防止之后的每一步都进行滚动更新
修改deployment
kubectl rollout resume

spec.revisionHistoryLimit可以限制rs的数量，如果数量为0就不能进行回滚操作

### StatefulSet

对于分布式应用，它的多个实例之间，往往有依赖关系，比如：主从关系、主备关系

还有就是数据存储类应用，它的多个实例，往往都会在本地磁盘上保存一份数据。而这些实例一旦被杀掉，即便重建出来，实例与数据之间的对应关系也已经丢失，从而导致应用失败。

这种实例之间有不对等关系，以及实例对外部数据有依赖关系的应用，就被称为“有状态应用”（Stateful Application）。

StatefulSet有两种状态

- 拓扑状态：应用的多个实例之间不是对等的关系。这些应用必须按照一定的顺序启动。
- 存储状态：应用的多个实例分别绑定不同的存储数据。对于这些应用实例来说，Pod A 第一次读取到的数据，和隔了十分钟之后再次读取到的数据，应该是同一份，哪怕在此期间 Pod A 被重新创建过。这种情况最典型的例子，就是一个数据库应用的多个存储实例。这个概念不是很理解！

StatefulSet的核心功能就是通过某种方式记录这些状态，然后在pod被重新创建时，能够为新pod恢复状态。


#### Headless Service

service是k8s项目中将一组pod暴露给外界访问的一种机制。比如我们定义一个3 pod的deployment，我们可以定义一个Service，，用户可以通过这个service访问某个具体的pod。

service如何被访问到

- 通过VIP（虚拟IP）的方式，我们访问service的VIP，service会将请求转发到具体的pod中。
- DNS方式，只要访问my-svc.my-namespace.svc.cluster.local这条记录，就可以访问到service代理的pod。这种方式又可以细分为两种处理方式
  - 第一种方式访问my-svc.my-namespace.svc.cluster.local解析到的是service的VIP
  - 第二种方式就是Headless Service！，访问my-svc.my-namespace.svc.cluster.local解析到的是service代理的某个pod的IP。这种方式和VIP的区别是Headless Service 不需要分配一个 VIP，而是可以直接以 DNS 记录的方式解析出被代理 Pod 的 IP 地址。service
通过Label Selector机制选择需要代理的pod，pod的ip地址被绑定到<pod-name>.<svc-name>.<namespace>.svc.cluster.local

那么，StatefulSet 又是如何使用这个 DNS 记录来维持 Pod 的拓扑状态的呢？

StatefulSet 就保证了 Pod 网络标识的稳定性。

StatefulSet 这个控制器的主要作用之一，就是使用 Pod 模板创建 Pod 的时候，对它们进行编号，并且按照编号顺序逐一完成创建工作。而当 StatefulSet 的“控制循环”发现 Pod 的“实际状态”与“期望状态”不一致，需要新建或者删除 Pod 进行“调谐”的时候，它会严格按照这些 Pod 编号的顺序，逐一完成这些操作。

所以，StatefulSet 其实可以认为是对 Deployment 的改良。与此同时，通过 Headless Service 的方式，StatefulSet 为每个 Pod 创建了一个固定并且稳定的 DNS 记录，来作为它的访问入口。实际上，在部署“有状态应用”的时候，应用的每个实例拥有唯一并且稳定的“网络标识”，是一个非常重要的假设。
实验见/Service

### Persistent Volume Claim

场景：不知道那些类型的Volume可用
比如，开发者不懂一些持久化存储项目（如Ceph， ClusterFs），自然无法编写对应的Volume定义文件

K8s项目引入了一组叫作 Persistent Volume Claim（PVC）和 Persistent Volume（PV）的 API 对象，大大降低了用户声明和使用持久化 Volume 的门槛。

所以，Kubernetes 中 PVC 和 PV 的设计，实际上类似于“接口”和“实现”的思想。开发者只要知道并会使用“接口”，即：PVC；而运维人员则负责给“接口”绑定具体的实现，即：PV。