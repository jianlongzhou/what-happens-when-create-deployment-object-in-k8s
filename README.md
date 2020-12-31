# what-happens-when-create-deployment-object-in-k8s
## 前言
受到github仓库what-happens-when-k8s的影响，加上近期正在准备工作交界，闲暇之余，想自己也来按照自己的理解总结整理一下k8s相关的知识脉络，以前总会有突然无法想起某个知识点的时候，希望以这里为起点，之后养成多记录多总结的习惯，仅作为自己的笔记，相对前面提到的repo作为一个补充。

k8s作为一个容器编排系统，有很多组件组成并谐调工作，我想，如果能利用一个问题点把各个组件的工作流程衔接起来，会更加有助于我们理解k8s。
以下描述的是用户通过以下命令创建一个deployment对象时所发生的事情。
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.12.2
```
```bash
kubectl create -f nginx-deployment.yaml
```

## 步骤

### kubectl
#### 客户端验证
当使用kubectl命令通过yaml文件创建一个对象时，kubectl首先做客户端验证，检查过滤一些很明显统一的错误（比如镜像的格式错误），而不是无脑的发给apiserver增加apiserver端的压力。
#### 构造请求
kubectl开始构造http用于请求apiserver，请求时需要携带身份信息用于apiserver端对自己进行认证，一般使用的是https双向证书认证，kubectl访问apiserver，apiserver会返回自己的服务端证书，kubectl会利用ca证书来验证服务端证书是否合法有效，kubectl也会发送自己的证书给apiserver用于表明自己的身份，证书存在于~/。kube/config文件中，这个文件也是kubectl默认读取的文件。

### apiserver
#### 认证 Authentication
k8s集群中的资源访问都是通过apiserver提供的rest api实现，所以需要验证客户端的身份信息，并为后面的鉴权提供基础。k8s提供了多种客户端身份认证方式，如下：
- https证书认证：基于ca根证书签名的数字证书验证，证书里面已经包含了客户端的身份信息，kubectl就是使用这种方式。
- http token认证：token中记录了客户端身份信息。

#### 鉴权 Authorization
对应的用户是否有请求对应的权限，比如这里的depoyment的创建请求，k8s通过rbac进行权限管理，我们可以通过role/clusterrole对象创建角色、rolebingding/clusterrolebinding对象把用户与角色进行绑定。

#### 准入控制 Admission Controller
客户端的请求经过了前面的认证和鉴权之后，还需要经历一个个的准入控制检查，全部检查通过之后，请求才会被apiserver真正接受并持久化到etcd中。apiserver中内置了很多的准入控制项目，比如DenyExecOnPrivileged、ResourceQuota、ServiceAccount、LimitRanger等，以ResourceQuota为例，我们可以通过创建ResourceQuota对象来配置某个namespace所能够使用的资源，比如限制namespace中最多有p个pod、d个deployment、c个cpu配额、m GB的memory配额等，通过admission controller拦截请求，判断如果当前请求成功是否会超过对象中配置的限额，会的话这个admission项目就会失败，整个请求也就失败，内置的准入控制项目详细介绍可以参考https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#what-does-each-admission-controller-do。
我们看到，k8s中的很多高级特性都是通过admission controller来实现的，可以通以下命令查看默认的准入控制项目
```bash
kube-apiserver -h | grep enable-admission-plugins
```
MutatingAdmissionWebhook和ValidatingAdmissionWebhook是两个比较特殊的准入控制项目，其允许用户根据自己的实际需求自定义准入控制的逻辑，分别可以用作对象变更和对象验证。以MutatingAdmissionWebhook为例，可以通过编写mutatingwebhookconfiguration对象，配置一个暴露https接口（我们自己写代码开发程序）和需要拦截的对象，例如实现一个需求：拦截所有的Pod创建请求，为其添加Hostpath挂载用于存储业务应用日志。

经过准入控制链的层层筛选之后，用户请求就会最终被apiserver解析成对应的对象，并持久化到etcd中

#### controller-manager
controller-manager中内置了很多常见对象的controller，比如deploymentController、serviceController、endpointController等，controller作用可以理解为不断的将实际状态调谐成期望状态，期望状态一般体现在用户提交创建的对象上（yaml）。
- 如上当apiserver持久化用户创建的deployment对象之后，deploymentController就可以通过informer机制watch到这个deployment对象，deploymentController开始工作，期望状态为有一个新的deployment被创建了，期待有一个对应的replicaset对象存在，实际并没有，所以deploymentController开始调谐，即通过apiserver创建一个replicaset对象.
- 当replicaset对象被创建后，controller-manager中的replicasetcontroller开始工作，期望状态是当前存在replicaset对象的spec.replicas字段中配置的pod数，实际状态也是不存在相应的pod，那么replicasetcontroller开始调谐并创建出对应数量的pod

### scheduler
kube-scheduler使用informer机制listAndWatch apiserver中nodeName字段为空的pod对象加入到自己的待调度队列，开始为pod选择节点。如上controller-manager创建出来的pod就会被scheduler watch到

#### 预选
scheduler中的预选过程是为了通过内置的一些必须满足的算法策略来过滤不合适的节点，比如节点剩余可分配资源不足以pod调度、节点不满足nodeAffinity亲和性中的require属性等。一些常见的预选策略如下：
- 基础的检查项（GeneralPredicates）
PodFitsResources node剩余可分配资源（Allocatable-sum(request))是否足够pod调度
PodFitsHost node的名称是否跟pod的spec.nodeName字段一致
PodFitsHostPorts pod指定的spec.nodePort端口在待考察node上是否已被占用
PodMatchNodeSelector pod的nodeSelector或者nodeAffinity指定的节点是否与待考察节点匹配
- Volume相关：
NoDiskConflict 待调度pod声明使用的Volume是否与待考察node上的已有pod声明挂载的Volume有冲突（AWS EBS类型的Volume，是不允许被两个Pod同时使用的）
MaxPDVolumeCountPredicate node上某种类型的持久化Volume是不是已经超过了一定数目
VolumeZonePredicate pod声明使用的Volume的Zone（高可用域）标签是否与待考察节点的Zone标签相匹配
VolumeBindingPredicate Pod对应的pvc所绑定的pv的nodeAffinity字段，是否跟某个节点的标签相匹配（LPV的延迟绑定机制）
- Node相关：
CheckNodeCondition：校验节点是否准备好被调度，校验node.condition的condition type ：Ready为true和NetworkUnavailable为false以及Node.Spec.Unschedulable为false
PodToleratesNodeTaints pod是否能够容忍待考察节点的污点
NodeMemoryPressurePredicate 待考察节点的内存是不是已经不够充足
- Pod相关：
PodAffinityPredicate 检查待调度Pod与Node上的已有Pod之间的亲密（affinity）和反亲密（anti-affinity）关系（pod中配置的spec.affinity.podAntiAffinity.requiredDuringSchedulingIgnoredDuringExecution属性)

具体到算法执行过程，输入是一个pod、m个预选策略、n个节点，输出是x个节点（x<=n），每个节点都需要执行这m个策略，但彼此是独立的，所以节点间可以并发进行（默认起16个gorouting），但是每个节点必须顺序执行策略，因为存在来两个策略有强制的关联顺序要求。

#### 优选
经过预选过程得到的节点都是可以用来部署pod的，为了在这些节点中选择一个最优的节点，会通过内置的优选算法来给每个节点打分，每个算法可以给予给个节点0-10分，节点获得的分数累加最为最终得分，分数高的节点将作为用来部署pod的节点，如果分数相同那么会随机选择一个，一些常见的优选策略如下
- LeastRequestedPriority 选择cpu剩余量和memory剩余量的和最多的宿主机，即把 Pod 分到资源空闲率最高的节点上，而非空闲资源最大的节点（相反的是MostRequestedPriority)
```bash
score = (cpu((capacity-sum(requested))10/capacity) + memory((capacity-sum(requested))10/capacity))/2
```
- BalancedResourceAllocation 各种资源分配最均衡的那个节点（避免一个节点上CPU被大量分配、而 Memory 大量剩余）
```bash
选择资源Fraction差距最小的节点
Fraction = Pod请求的资源 / 节点上的可用资源
variance = 每两种资源 Fraction 之间的距离
score = 10 - variance(cpuFraction,memoryFraction,volumeFraction)*10
```
- ServiceSpreadingPriority 同一个service管控下的pod尽量分散在不同的节点

优选过程依然是使用默认的16个并发来进行节点间的算法执行，代码实现还采用了map-reduce的思想进行分数的归一处理，不过这不是重点。

#### 异步绑定
经过预选和优选过程之后，scheduler就为pod选出了一个合适的节点，但是scheduler并不会等待着把结果成功提交到apiserver，而是立刻开始调度下一个pod，调度结果是通过起一个gorouting异步的访问apiserver进行更新的(async bind)，因为我们没必要把bind这个过程放在主调度路径中。而如果async bind失败，下次调度器会重新为其调度，所以是没有影响的。

### kubelet
kubelet运行在每个worker节点上面，其仍然通过informer机制watch apiserver来获取pod对象中nodeName字段值是自己的pod，整个kubelet有很多个模块组成，如ContainerManager、PLEG、StatusManager、ProberManager、EvictionManager等，这些模块相互配合.

- 根据pod对象的定义创建linux namespace配置
- 根据pod resources创建linux cgroup配置
- 通过CSI初始化volume配置，并准备attach到容器
- 通过CNI接口调用配置的网络插件分配podIP、初始化网络配置
- 拉取pod对象配置的镜像
- 通过CRI接口调用配置的runtime，结合上述初始化的namespace、cgroup、network、volume，把容器运行起来


cAdvisor负责收集容器资源使用等信息
ProbeManager负责根据pod中配置的探针定时检测上报容器状态
StatusManager接受其他模块发送过来的容器状态信息再上报到apiserver
EvictionManager检测节点资源使用是否触发到用户配置的资源驱逐水位线，根据具体的策略驱逐pod
ImageGC/ContainerGC根据用户的GC配置判断节点上的镜像/死掉的容器是否需要清理
VolumeManager负责容器创建过程中设计的volume的attach/detach，以及同步volume状态
PLEG(Pod LifeCycle Event Generator)负责通过runtime定时relist节点上运行的容器，判断容器状态是否异常，并上报给其他模块处理



以上是目前总结的一个Deployment创建之后各个组件的协同工作过程，设计到的很多细节并没有说清楚，之后完善。
