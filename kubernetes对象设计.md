#### etcd

etcd是CoreOS基于raft开发的分布式key-value存储，用于服务发现、共享配置以及一致性保障（如数据库选主、分布式锁等）

- 基本的key-value存储

- 监听机制

- key的过期及续约机制，用于监控和服务发现

- 原子CAS和CAD，用于分布式锁和leader选举

![etcd集群](https://github.com/RTCFoundation/k8s.github.io/blob/main/images/etcd集群.png)

#### 直接访问etcd数据

- 通过etcd进程查看启动参数

- 进入容器

```
ps -ef|grep etcd

etcd --advertise-client-urls=https://172.17.0.76:2379 --cert-file=/var/lib/minikube/certs/etcd/server.crt --client-cert-auth=true --data-dir=/var/lib/minikube/etcd --initial-advertise-peer-urls=https://172.17.0.76:2380 --initial-cluster=minikube=https://172.17.0.76:2380 --key-file=/var/lib/minikube/certs/etcd/server.key --listen-client-urls=https://127.0.0.1:2379,https://172.17.0.76:2379 --listen-metrics-urls=http://127.0.0.1:2381 --listen-peer-urls=https://172.17.0.76:2380 --name=minikube --peer-cert-file=/var/lib/minikube/certs/etcd/peer.crt --peer-client-cert-auth=true --peer-key-file=/var/lib/minikube/certs/etcd/peer.key --peer-trusted-ca-file=/var/lib/minikube/certs/etcd/ca.crt --proxy-refresh-interval=70000 --snapshot-count=10000 --trusted-ca-file=/var/lib/minikube/certs/etcd/ca.crt
```

- 查询数据

```
etcdctl --endpoints https://172.17.0.76:2379 --cert /var/lib/minikube/certs/etcd/server.crt --key /var/lib/minikube/certs/etcd/server.key --cacert /var/lib/minikube/certs/etcd/ca.crt get --keys-only --prefix /
```

- 监控对象变化

```
etcdctl --endpoints https://172.17.0.76:2379 --cert /var/lib/minikube/certs/etcd/server.crt --key /var/lib/minikube/certs/etcd/server.key --cacert /var/lib/minikube/certs/etcd/ca.crt get watch --prefix /registry/services/specs/default/mynginx
```

#### API Server

Kube-APIServer是k8s最重要的核心组件之一，主要提供以下功能：

1. 提供集群管理的REST API接口，包括

- 认证Authentication

- 授权Authorization

- 准入Admission

2. 提供其他模块之前的数据交互和通讯的枢纽

3. APIServer提供etcd数据缓存以减少集群对etcd的访问

#### Controller Manager

Controller Manager是集群的大脑，确保整个集群正常运行的关键

Controller Manager是多个控制器的组合，每个controller事实上都是一个control loop，负责侦听其管控对象。当对象发生变更时完成配置

controller配置失败通常会触发自动重试

![controllerManager](https://github.com/RTCFoundation/k8s.github.io/blob/main/images/controllerManager.png)

