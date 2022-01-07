### Docker核心技术

1. 系统架构

传统分层架构 ---> 微服务

微服务通讯

- API 网关

- RPC通信

- 基于sidecar通信

2. Docker

##### 实现原理

基于Linux内核的cgroup，Namespace，以及Union FS等技术，对进程进行封装隔离，属于os的虚拟化技术。

由于隔离的进程独立于宿主和其他的隔离进程，因此称之为容器

##### 容器常用操作

启动：

​	docker run

​	-it 交互

​	-d 后台运行

​    -p 端口映射

​    -v 磁盘挂载

启动已终止容器

​	docker start

停止容器

​	docker stop

查看容器进程

​	docker ps

查看容器细节：

​	docker inspect 

拷贝文件到容器内

​	docker cp file <containerid>:/path

##### 初识容器

cat Dockerfile

```
FROM ubuntu
ENV SERVICE_PORT=80
ADD bin/httpserver /httpserver ENTRYPOINT /httpserver
```

将Dockerfile打包成镜像

```
docker build -t httpserver:${tag} .
docker push httpserver:v1.0
```

运行容器

```
docker run -d httpserver:v1.0
```



##### 容器标准

Open Container intiative （OCI） 轻量级开放式管理组织

OCI定义了两个规范

- Runtime Specification
- Image Specification

容器主要特性：安全性，隔离型，可配额和便携性



##### Namespace

Linux Namespace 是一种Linux Kernel提供的资源隔离方案

- 系统为进程分配不同的Namespace

- 保证不同的Namespace资源独立分配，进程彼此隔离



###### 内核中Namespace的实现

- 进程数据结构

  ```c
  struct task_struct {
  	...
  	/*namespaces*/
  	struct nsproxy *nsproxy;
  	...
  }
  ```

- Namespace数据结构

```
struct nsproxy {
	atomic_t count;
	struct uts_namespace *uts_ns;
	struct ipc_namespace *ipc_ns;
	struct mnt_namespace *mnt_ns;
	struct pid_namespace *pid_ns_for_children;
	struct net *net_ns
}
```

###### Namespace操作方法

- clone

在创建新进程的系统调用时，可以通过flags参数指定需要创建的Namespace类型：

// CLONE_NEWCGROUP/CLONE_NEWIPC/CLONE_NEWNET/CLONE_NEWNS/CLONE_NEWPID/CLONE_NEWUSER/CLONE_NEWUTS

int clone (int(*fn)(void*), void *child_stack, int flags, void *arg)

- setns

该系统调用可以让调用进程加入某个已经存在的Namespace中：

int setons(int fd, int nstype)

- unshare

该系统调用可以将调用进程移动到新的Namespace下

int unshare(int flags)

###### Namespace隔离性

| 类型                   | 隔离资源                     | kernel版本 |
| ---------------------- | ---------------------------- | ---------- |
| IPC                    | System V IPC和Posix消息队列  | 2.6.19     |
| Network                | 网络资源，协议栈，网络端口等 | 2.6.29     |
| PID                    | 进程                         | 2.6.14     |
| Mount                  | 挂载点                       | 2.4.19     |
| UTS(Unix Time Sharing) | 主机名和域名                 | 2.6.19     |
| USR                    | 用户和用户组                 | 3.8        |

- 查看当前系统的namespace:

  lsns -t type

- 查看某进程的namespace

  ls -la /proc/pid/ns/

- 进入某namespace运行命令

  nsenter -t pid -n ip addr

##### Cgroups

Cgroups是Linux下用于对一个或一组进程进行资源控制和监控的机制

可以对诸如cpu使用时间、内存，磁盘IO等进程所需的资源进行限制

不同资源的具体工作由相应的Cgroup子系统来实现

针对不同类型的资源限制，只要将限制策略在不同的子系统上进行关联即可

###### Linux内核代码Cgroups的实现

- 进程数据结构

```c
 struct task_struct {
 	#ifdef CONFIG_CGROUPS
 	struct css_set __rtcu *cgroups;
 	struct list_head cg_list;
 	#endif
 }
```

- css_set是cgroup_subsys_state对象的集合数据结构

```c
struct css_set {
	struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT]
};
```

###### Control Groups可配额/可度量

- blkio：      这个子系统限制每个块设备的输入输出控制。例如磁盘，光盘以及usb

- cpu：        提供cpu的访问

- cpuacct： 产生cgroup任务的cpu资源报告

- cpuset：   如果是多核cpu，为cgroup任务分配单独的cpu和内存

- devices：  控制对设备的访问

- freezer：  暂停或恢复cgroup任务

- memory：设置每个cgroup的内存限制

- net_cts：   标记每个网络包

- ns：           命名空间

- pid：          进程标识子系统



Linux CPU调度器

内核默认提供了5个调度器，使用struct sched_class对调度器进行抽象

1. stop调度器，stop_sched_class优先级最高的调度类，可以抢占其他所有进程
2. deadline调度器，dl_sched_class 使用红黑树，把进程按照绝对时间期限进行排序，选择最小进程进行调度
3. RT调度器，rt_sched_class实时调度器，为每个优先级维护一个队列
4. CFS调度器，cfs_sched_class完全公平调度器，采用完全公平调度算法，引入虚拟运行时间概念
5. IDLE-Task调度器，idle_sched_class空闲调度器，每个cpu都有一个idle线程，当没有其他进程调度时，调度运行idle

###### Cgroup driver

systemd：

当操作系统使用systemd作为init system时，初始化进程生成一个根cgroup目录结构并作为cgroup管理器

systemd和cgroup紧密结合，并且为每个sytemd unit分配cgroup

cgroupfs：

docker默认用cgroupfs作为cgroup驱动

存在问题：

在systemd作为init system的系统中，默认存在两套groupdriver。系统中docker和kubelet管理的进程被cgroupfs驱动管，而systemd拉起的服务有systemd驱动管，就会出现管理混乱，容易在资源紧张时引发问题。

解决方法：

kubelet会默认--cgroup-driver=systemd



##### UnionFS文件系统