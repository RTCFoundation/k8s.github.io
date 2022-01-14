### kubenetes架构原则和对象设计

云计算平台的分类

1. 以openstack为典型的虚拟化平台

- 虚拟机构建和业务代码部署分离

- 可变的基础架构使后续维护风险变大

2. 以谷歌borg为典型的基于进程的作业调度平台

##### Google borg简介

1. workload

prod：在线任务，长期运行，对延时敏感，面向终端用户，比如Gmail，Google docs，web Search服务等

non-prod：离线任务，也称为批处理任务（Batch），比如一些分布式计算服务等

2. Cell

一个Cell跑一个集群管理系统Borg

通过定义Cell可以让Borg对服务资源进行统一虫响，作为用户就无需知道自己的应用跑在那台机器上，不用关心资源分配，程序安装，依赖管理，健康检查及故障恢复等

3. Job和Task

用户以Job的形式提交应用部署请求。一个Job包含一个或多个相同的Task，每个Task运行的应用程序

4. Naming

Borg的服务发现通过BNS（Borg Name Service）来实现的

50.jfoo.ubar.cc.borg.google.com表示，在一个名为cc的Cell中由用户ubar部署的一个名为jfoo的Job下的第50个Task

##### Borg架构

![borg](/Users/menglingfeng/Documents/GitHub/k8s.github.io/borg.png)

borg master主进程：

- 处理客户端rpc请求，比如创建job，查询job等

- 维护系统组件和服务的状态，比如服务器、Task等

- 负责与borglet通信

scheduler进程：

- 调度策略：worst fit，best fit和hybrid
- 调度优化

Borglet：

​	borglet是部署在所有服务器上的agent，负责接收borg master进程的指令

##### 隔离性

安全性隔离

- 早期采用chroot jail，后期版本基于namesapce

性能隔离

- 采用cgroup的容器技术

- 在线任务（prod）是延时敏感的，优先级高，而离线任务优先级低

- borg通过不同优先级之间的抢占式调度来有限保障在线任务的新能，牺牲离线任务

##### 什么是kubernetes（k8s）

kubernetes是谷歌开源的容器集群管理系统，是google多年大规模管理技术borg的开源版本，主要功能包括：

- 基于容器的应用部署，维护和滚动升级

- 负载均衡和服务发现

- 跨机房和跨地区的集群调度

- 自动伸缩

- 无状态和有状态服务

- 插件机制保证扩展性

##### kubernetes：声明式系统

kubernetes的所有管理能力构建在对象抽象的基础上，核心对象包括

- Node：计算节点的抽象，用来描述计算节点的资源抽象、健康状态等

- Namespace：资源隔离的基本单位，可以简单理解为文件系统中的目录结构

- Pod：用来描述应用实例，包含镜像地址、资源需求等。它是kubernetes中最核心的对象，也是打通应用和基础架构的秘密武器

- Service：服务如何将应用发布成服务，本质上是负载均衡和域名服务的声明

##### kubernetes架构

![k8s](/Users/menglingfeng/Documents/GitHub/k8s.github.io/k8s.png)

k8s Master node

- API Server：

这是k8s控制面板中唯一用户可以访问API以及交互的组件。API服务器会暴露一个RESTful的k8s api并使用json格式的清单文件

- Cluster Data Store：

k8s使用etcd，这是一个强大、稳定、高可用的键值存储，被k8s用于长久储存所有的API对象

- Controller Manager：

它运行着所有处理集群日常任务的控制器，包含了节点控制器、副本控制器、端点控制器以及服务账户等

- Scheduler

调度器会监控新建的pods，并将其分配给节点

k8s工作节点（Worker Node）

- kubelet

负责调度到对应节点的Pod的生命周期管理，执行任务并将Pod状态报告给主节点的渠道，通过容器运行时来运行这些容器，它还会定期执行健康探测程序

- kube-proxy
- 

负责节点的网络，在主机上维护网络规则并执行连接转发，还负责对正在服务的pods进行负载均衡