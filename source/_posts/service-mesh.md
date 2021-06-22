---
title: Service Mesh和服务Mesh化的认知和理解
date: 2021-09-05 10:33:29
tags: ServiceMesh
---
> 为什么要做这次分享？
已经了解到公司基础架构部门和很多业务部门的技术已经在落地Service Mesh了，Service Mesh本身是个基础设施，最终肯定会推向各个业务，那对业务开发人员或多或少会有影响，
为了使平台开发工程师和业务开发工程师那时候能更好的沟通，就想对Service Mesh做一个入门的介绍。
Service Mesh的实践会涉及到容器相关的知识，比如docker和k8s，如果不了解并不影响这次分享，因为这个分享更多的是讲述如何理解Service Mesh。


通过本次分享希望大家了解：

 - 什么是service mesh？
   
 -  它和微服务有什么关系？
   
 -  基于服务mesh化的思想我们能做些什么？

## 什么是Service Mesh
![服务网格](https://img-blog.csdnimg.cn/20200827204853858.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70#pic_center)
“网”是Service Mesh称呼的由来
微服务当前是什么样子，和目前的微服务之间的调用相比Service Mesh的区别在哪。
不需要知道目标服务在哪，只需要知道目标sidecar在哪

### 概述
一个专注于服务间通信的基础设施。”

Linkerd的CEO William对Service Mesh的描述：
       对于数百个服务或数千个实例，以及不时需要重新调度的业务层实例，单个请求通过的调用链可能变的非常复杂，而且服务可能由不同的语言编写，服务通信的复杂性和重要性导致我们急需一个专门的基础设施层来处理服务间的通信，该层需要与业务代码解耦，并且具有捕获底层环境的动态机制。这就是 Service Mesh 。


注： Linkerd是业界第一个Service Mesh，也是他们第一次提出Service Mesh概念。
### 诞生背景
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200827205402744.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70#pic_center)
机器之间频繁使用网络交换信息的时候需要处理网络通信的问题，比如数据丢失，顺序错误延时阻塞等问题，需要进行流控
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200827205416289.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70#pic_center)
TCP和HTTP等标准协议出现以后，对于网络的处理下沉到了网络栈里面，和操作系统集成在一起
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200827205514202.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70#pic_center)
#### 初代微服务模式
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200827205540248.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70#pic_center)
初代微服务-All-in-one
初代微服务模式的时候，应用程序里面加上了大量的非业务性的代码，业务开发人员除了需要关注业务
的实现还要关注微服务中的服务注册发现、重试、熔断等等。

#### 有了微服务开发组件/库“全家桶以后”
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200827205613876.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70#pic_center)
引入了微服务开发“全家桶”，例如Spring Cloud、Netflix OSS套件以后，开发人员从繁重
的保证微服务治理的任务中解放出来，只需要关注自己的业务实现即可。大大简化了微服务开发的难度，
这也是Spring Cloud等框架后面流行的重要原因之一。

**但是，这样就完美了吗？大家可以想想当下在进行微服务开发的时候，
是否真的只需要关心业务逻辑。**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200828152646352.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70#pic_center)

> 现阶段微服务痛点：我们的服务变成了集成了中间件各种功能的胖客户端

**“事实上，微服务开发做了这么多额外的努力仅仅是为了保证请求能够成功到达正确的服务”**
比如写一个用户服务，对用户做CRUD操作，和刚才说的这些东西有一毛钱关系吗？这些和服务本身没关系，那统一处理服务间的通讯就变成了我们需要解决的根源问题。
#### Sidecar
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200828153039967.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70#pic_center)

> “它是一个代理，将关于服务通讯的功能抽离出来独立运行。”

战争片里面的战车。吃鸡里面也有
这种在微服务中将业务逻辑与服务通信解藕，并分离为两个独立运行组件的做法，正是 Service Mesh 概念的雏形。
那我们再来看一下Sidecar在服务里是以什么形式存在？
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200828153208644.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70#pic_center)
这个时期的Sidecar是有局限性的，各个企业为自己的基础架构定制的。
#### Service Mesh
有了Sidecar以后，那就有人提出了基于Sidecar的通用型的解决方案
Service Mesh更强调Sidecar连接形成的网络
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200828153337921.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70#pic_center)

> “Service Mesh是基于Sidecar的通用方案。”
“Service Mesh 的愿景是希望开发者再也不需要将精力花费在服务通信上。
服务通信由每个微服务的 Sidecar 负责”
### Service Mesh-主要优势
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200828153543742.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70#pic_center)

 - 去中心化：集中式代理变为分布式代理，避免单点故障，比如路由转发、认证鉴权由统一的网关服务变为以进程的方式和业务服务进程共存
- 跨语言：无需一个语言一个客户端，均可以HTTP、TCP、gRPC等方式通信
- 和业务解耦：将管理和运维功能从业务服务中抽离，避免中间件客户端或服务端版本升级对业务造成的影响
- 其他：不需要应用程序做大量改动

**Service Mesh是一个基于Sidecar代理模型来处理服务间通信的基础设施。**
## Service Mesh的实现：Istio
### 概述
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200828153931427.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70#pic_center)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200828153942486.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70#pic_center)

 应用集装箱被docker这个货船运载着在大海上无忧无虑的遨游。
K8s是掌舵者，他来管理和控制货船的行为。
想在大海上无忧无虑的航行，还需要什么，对，帆，有了帆，船在能走的更稳，那这个帆就是我们要说的Istio

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200828154044207.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70#pic_center)
是Service Mesh的一个完整的解决方案，作为透明的一层接入到现有的分布式应用程序里。并提供保护、连接和监控微服务的统一方法。
### 总体架构
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200828154241531.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70#pic_center)
Istio 服务网格从逻辑上分为数据平面和控制平面。
**数据平面** 由一组智能代理（Envoy）组成，被部署为 sidecar。这些代理负责协调和控制微服务之间的所有网络通信。他们还收集和报告所有网格流量的遥测数据。
**控制平面** 管理并配置代理来进行流量路由。

服务发现指的是发现sidecar暴露的ip和端口。Istio有一个注册表（存放服务的地址）来支撑服务路由和负载均衡，它本身不提供服务发现，它是基于底层平台的服务注册发现能力，
顺便将服务的信息通过Pilot组件注册到其自身的注册表里。
Sidecar流量劫持，Service和envoy的通信：iptables捕获和重定向
#### Envoy
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200828154456113.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70#pic_center)
“用于协调服务网格中所有服务的入站和出站流量。”
Envoy（使者的意思） 是 istio 中负责"干活"的模块,如果将整个 istio 体系比喻为一个施工队,那么 Envoy 就是最底层负责搬砖的民工, 所有体力活都由 Envoy 完成. 所有需要控制,决策,管理的功能都是其他模块来负责,然后配置给 Envoy。
#### Pilot
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200828170911887.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70#pic_center)

Envoy在其中扮演的负责搬砖的民工角色, 而指挥Envoy工作的民工头就是Pilot模块.
Pilot（领航员的意思）提供了连接，控制的功能。

1、平台启动一个服务的新实例，该实例通知其平台适配器。
2、平台适配器使用 Pilot 抽象模型注册实例。
3、Pilot 将流量规则和配置派发给 Envoy 代理，来传达此次更改。

- Envoy API负责和Envoy的通讯, 主要是发送服务发现
信息和流量控制规则给Envoy，这些API将istio和Envoy
的实现解耦，使得Envoy可以被其他厂商的数据平面产品接管。

- Abstract Model 定了一个抽象模型,定义在istio中什么是
服务以从特定平台细节中解耦, 为跨平台提供基础。

- Platform Adapter则是这个抽象模型的现实实现版本, 用
于对接外部的不同平台。

- Rules API 提供接口给外部调用以管理 Pilot, 包括
命令行工具istioctl以及未来可能出现的第三方管理界面。

**基于上述的架构设计, Pilot提供以下重要功能：**
- 请求路由
- 服务发现和负载均衡
- 故障处理
- 故障注入
- 规则配置
#### Citadel
“堡垒”的意思。
“之前的版本叫做Istio-Auth。
Citadel通过内置的身份和证书管理，可以支持强大的服务到服务以及最终用户的身份验证。”

**重要功能：**
- 加密
- 身份认证
- 访问控制
- 密钥管理
#### Galley
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200828171655905.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70#pic_center)
“之前叫做Mixer。
Galley 是 Istio 的配置验证、提取、处理和分发组件。它负责将其余的 Istio 组件与从底层平台
（例如 Kubernetes）获取用户配置的细节隔离开来。”
**重要功能：**

- 前提条件检查，如：认证，黑白名单，ACL检查
- 配额管理，如：限流
- 遥测报告，如：日志，监控，指标
### 核心特性
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200828172819549.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70#pic_center)
**在前面的组件加持下提供的核心特性：**
- 流量管理。路由，负载均衡，定义流量配比等。
- 可观察性。监控，日志，追踪等。
- 服务身份和安全。身份认证，访问控制和流量安全防护等。

**Istio 是基于Service Mesh，对微服务提供连接，保护，控制，观测的完整解决方案。**
## 服务Mesh化
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200828174246940.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70#pic_center)
先来思考一个问题：
当Service Mesh将业务和服务间的通信解耦之后，当前的微服务还能不能再纯粹一点？
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200828174328345.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70#pic_center)
Service Mesh中的Sidecar的关键在于“解耦”。

而Sidecar这个车舱里不仅仅可以“坐”流量管理的逻辑，也可以将中间件放入其中，
实现业务和中间件的解耦。

即服务Mesh化 = 服务（中间件） + Sidecar
## API Gateway Mesh
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200828174603532.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70#pic_center)
上面这张图是一个云原生，南北+东西流量的架构图，在Service Mesh环境下使用集中式API Gateway的架构图
为什么有了service mesh还需要API Gateway？
#### API Gateway VS Service Mesh
之前了解到Service Mesh也是提供了路由转发，负载均衡，认证、限流等功能，那他和传统的API Gateway有什么区别呢？

- API Gateway将内部服务以可管理的方式暴露出去，Service Mesh将业务逻辑和应用网络解耦；
- API Gateway管理南北流量（外部流量），Service Mesh管理东西流量（服务内部流量）；
- API Gateway是中心化的路由转发，Service Mesh是类似分布式的，客户端下的路由转发，Sidecar 之间是点对点的调用；
- API Gateway里面可能会包含偏业务的一些逻辑， Service Mesh只管理服务间的流量；

**随着两者的发展，他们开始有了融合，功能上逐渐出现了重叠，那我们能不能把两者合为一体呢？**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200828174756340.png#pic_center)
结合前面对服务Mesh化的理解，可以得出一个结论：
API Gateway Mesh = API Gateway + Sidecar，
类似的消息队列（MessageQueue）+ Sidecar = Message Mesh。
#### 带来的好处
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200828174850564.png#pic_center)
中心化，集中式网关，目前最常见的
中心化网关：单点问题，多一次网络消耗
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200828174933474.png#pic_center)
客户端嵌入式，去中心化，Rout提供简单的路由，将更偏业务的认证鉴权，协议转换，复杂的路由抽象成一个gateway的jar包或者js文件，和语言有关
客户端嵌入式网关（去中心化）：
不同语言需要提供不同的client，
接入困难，客户端升级困难
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200828175015343.png#pic_center)
API Gateway Mesh，去中心化，基于Sidecar代理模型，将gateway以独立进程和应用部署在一起，达到低成本接入，平滑升级，支持多语言系统
**API Gateway Mesh = API Gateway + Sidecar**
## 回顾
1、什么是service mesh？
答：基于Sidecar代理模式的处理服务间通信和流量管理的基础设施。

2、Service Mesh和微服务有什么关系？
答：它是云原生时代下保证微服务架构之间通信的一种解决方案，使微服务本身只需关注业务实现。

3、基于服务mesh化的思想我们能做些什么？
答：我们可以借鉴Sidecar的解耦思想，将业务变得更纯粹，如API Gateway Mesh，MessageQueue Mesh等。