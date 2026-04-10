# 集群资源、地址与依赖基线

这份文档是部署手册的配套速查表，目标是让后来的人或 AI 不用翻多个仓库 README，就能快速知道：

- 每个组件默认装在哪
- 默认访问地址是什么
- 默认账密是什么
- 它依赖谁
- `mid` 档位下的资源和存储需求大概是多少

部署步骤见：

- [DEPLOY-README.zh-CN.md](./DEPLOY-README.zh-CN.md)

## 1. 仓库与交付物映射

| 阶段 | 仓库 | 服务器上的默认产物 |
| --- | --- | --- |
| bootstrap init | `archinfra/bootstrapctl` | `/opt/release/boot` |
| LVM | `archinfra/lvm` | `/opt/release/lvm.sh` |
| NFS Server | `archinfra/nfs-server` | `/opt/release/nfs-server.sh` |
| Kubernetes | `archinfra/apps_kubernetes` | `/opt/release/k8s-sealos-linux-<arch>-full.run` |
| NFS StorageClass | `archinfra/apps_nfs-provisioner` | `/opt/release/nfs-provisioner-installer-<arch>.run` |
| metrics-server | `archinfra/apps_metrics-server` | `/opt/release/metrics-server-installer-<arch>.run` |
| Prometheus | `archinfra/app_prometheus-stack` | `/opt/release/prometheus-stack-installer-<arch>.run` |
| MySQL | `archinfra/apps_mysql` | `/opt/release/mysql-installer-<arch>.run` |
| Redis | `archinfra/apps_redis-cluster` | `/opt/release/redis-cluster-installer-<arch>.run` |
| Nacos | `archinfra/apps_nacos` | `/opt/release/nacos-installer-<arch>.run` |
| MinIO | `archinfra/apps_minio-cluster` | `/opt/release/minio-cluster-installer-<arch>.run` |
| RabbitMQ | `archinfra/apps_rabbitmq-cluster` | `/opt/release/rabbitmq-cluster-installer-<arch>.run` |
| MongoDB | `archinfra/apps_mongodb-cluster` | `/opt/release/mongodb-cluster-installer-<arch>.run` |
| Milvus | `archinfra/apps_milvus-cluster` | `/opt/release/milvus-cluster-installer-<arch>.run` |
| HM | `archinfra/hm` | `/opt/release/hm-installer-<arch>.run` |

## 2. 主机与基础设施约定

| 项目 | 默认值 |
| --- | --- |
| LVM 挂载点 | `/data` |
| 容器 graph root | `/data/graphroot` |
| containerd 数据目录 | `/data/containerd` |
| NFS Server 所在节点 | `master-01` |
| NFS 导出目录 | `/data/nfs-share` |
| 默认 StorageClass | `nfs` |
| 监控标签 | `monitoring.archinfra.io/stack=default` |

## 3. 组件总览

| 组件 | 命名空间 | 内网访问 | 外网访问 | 默认账密 | 依赖 |
| --- | --- | --- | --- | --- | --- |
| metrics-server | `kube-system` | 聚合 API，不直接暴露业务地址 | 不建议暴露 | 无 | Kubernetes |
| Prometheus Stack | `monitoring` | 通过 `kubectl get svc -n monitoring` 查看 | 默认不暴露 | Grafana `admin / admin@passw0rd` | Kubernetes、NFS SC |
| MySQL | `aict` | `mysql.aict.svc.cluster.local:3306` | `<NODE_IP>:30306` | `root / passw0rd` | Kubernetes、NFS SC |
| Redis Cluster | `aict` | `redis-cluster.aict.svc.cluster.local:6379` | 默认不暴露 | 密码 `Redis@Passw0rd` | Kubernetes、NFS SC |
| Nacos | `aict` | `nacos.aict.svc.cluster.local:8848` | HTTP `30094`，gRPC `30930` | 数据库密码依赖 MySQL | MySQL |
| MinIO Cluster | `aict` | API `minio.aict.svc.cluster.local:9000` | API `30093`，Console `30092` | `minioadmin / minioadmin@123` | Kubernetes、NFS SC |
| RabbitMQ Cluster | `aict` | AMQP `rabbitmq-cluster.aict.svc.cluster.local:5672` | 默认不暴露 | `admin / RabbitMQ@Passw0rd` | Kubernetes、NFS SC |
| MongoDB ReplicaSet | `aict` | `mongodb-cluster-0..2.mongodb-cluster-headless.aict.svc.cluster.local:27017` | 默认不暴露 | `root / MongoDB@Passw0rd` | Kubernetes、NFS SC |
| Milvus Cluster | `milvus-system` | `milvus-cluster.milvus-system.svc.cluster.local:19530` | 默认不暴露 | 无默认业务账号 | Kubernetes、NFS SC |
| HM | `aict` | 见下方业务入口表 | `30080/31721/32136/31285/31196` | 依赖中间件自身账密 | MySQL、Redis、MinIO、MongoDB、RabbitMQ、Milvus |

## 4. `mid` 档位资源基线

说明：

- 这里只统计仓库文档里已经固定下来的默认 `request/limit`
- Prometheus Stack 的计算资源未在安装器层统一钉死，因此不计入“硬性显式总量”
- 数值是运维容量规划用的近似值

| 组件 | `mid` request | `mid` limit | 默认持久化 |
| --- | --- | --- | --- |
| metrics-server | `0.1 CPU / 200Mi` | 以清单为准 | 无 |
| MySQL | `0.6 CPU / 1152Mi` | `1.2 CPU / 2304Mi` | `10Gi` |
| Redis Cluster | `3.6 CPU / 6.75Gi` | `7.2 CPU / 13.5Gi` | `60Gi` |
| Nacos | `0.5 CPU / 1Gi` | `1 CPU / 2Gi` | 默认不单独声明 PVC |
| MinIO Cluster | `2.1 CPU / 4.25Gi` | 约 `16.7 CPU / 32.75Gi` | `2Ti` |
| RabbitMQ Cluster | `1.5 CPU / 3Gi` | `3 CPU / 6Gi` | `24Gi` |
| MongoDB ReplicaSet | `1.8 CPU / 3456Mi` | `3.6 CPU / 6912Mi` | `60Gi` |
| Milvus Cluster | `4 CPU / 13Gi` | 视组件汇总而定 | `460Gi` |
| HM | `11.81 CPU / 11.9Gi` | `41.92 CPU / 51Gi` | `10Gi` |

## 5. 业务层显式资源总量

不含 Prometheus Stack 的计算资源，仅统计当前仓库已经固定了 `request` 的业务层组件时，默认 `mid` 档位的最低请求量约为：

| 合计项 | 数值 |
| --- | --- |
| 总 CPU request | 约 `26.01 CPU` |
| 总 Memory request | 约 `44.6Gi` |
| 总显式持久化存储 | 约 `2.78Ti` |

显式持久化存储由以下部分组成：

| 组件 | 存储 |
| --- | --- |
| Prometheus Stack | `220Gi` |
| MySQL | `10Gi` |
| Redis Cluster | `60Gi` |
| MinIO Cluster | `2Ti` |
| RabbitMQ Cluster | `24Gi` |
| MongoDB ReplicaSet | `60Gi` |
| Milvus Cluster | `460Gi` |
| HM kbase PVC | `10Gi` |

因此，作为 NFS 后端的 `master-01:/data`，建议至少规划：

- 测试环境：`3Ti` 可用空间以上
- 生产环境：`4Ti` 可用空间以上，并预留增长余量

## 6. 组件详细信息卡

### 6.1 metrics-server

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `kube-system` |
| 核心用途 | 为 `kubectl top`、HPA、监控基线提供资源使用数据 |
| 访问方式 | 聚合 API，不直接提供业务入口 |
| 依赖 | Kubernetes |
| 备注 | kubelet 自签 TLS 环境建议开启 `--kubelet-insecure-tls` |

### 6.2 Prometheus Stack

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `monitoring` |
| 核心用途 | 监控底座、Grafana、Prometheus、Alertmanager |
| 监控发现规则 | 只抓带 `monitoring.archinfra.io/stack=default` 的资源 |
| 默认账密 | Grafana `admin / admin@passw0rd` |
| 外网访问 | 默认不暴露，建议使用 ingress / NodePort / port-forward |
| 依赖 | Kubernetes、NFS SC |

### 6.3 MySQL

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `aict` |
| 内网地址 | `mysql.aict.svc.cluster.local:3306` |
| 主节点直连 | `mysql-0.mysql.aict.svc.cluster.local:3306` |
| 外网地址 | `<NODE_IP>:30306` |
| 默认账密 | `root / passw0rd` |
| 其他默认账号 | `repl / repl@passw0rd`，`orch / orch@passw0rd`，`mysqld_exporter / exporter@passw0rd` |
| 依赖 | Kubernetes、NFS SC |
| 被谁依赖 | Nacos、HM |

### 6.4 Redis Cluster

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `aict` |
| 内网地址 | `redis-cluster.aict.svc.cluster.local:6379` |
| Headless | `redis-cluster-headless.aict.svc.cluster.local` |
| 外网地址 | 默认不暴露 |
| 默认密码 | `Redis@Passw0rd` |
| 依赖 | Kubernetes、NFS SC |
| 被谁依赖 | HM |

### 6.5 Nacos

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `aict` |
| 内网地址 | `http://nacos.aict.svc.cluster.local:8848/nacos` |
| 外网地址 | HTTP `<NODE_IP>:30094`，gRPC `<NODE_IP>:30930` |
| 默认数据库 | `frame_nacos_demo` |
| 默认 MySQL 连接 | `mysql-0.mysql.aict:3306` |
| 默认初始化 | 导入幂等 SQL 与 `cmict-share.yaml` |
| 依赖 | MySQL |
| 被谁依赖 | 配置中心使用方 |

### 6.6 MinIO Cluster

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `aict` |
| 内网 API | `minio.aict.svc.cluster.local:9000` |
| 内网 Console | `minio-console.aict.svc.cluster.local:9090` |
| 外网 API | `<NODE_IP>:30093` |
| 外网 Console | `<NODE_IP>:30092` |
| 默认账密 | `minioadmin / minioadmin@123` |
| 依赖 | Kubernetes、NFS SC |
| 被谁依赖 | HM、对象存储使用方 |

### 6.7 RabbitMQ Cluster

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `aict` |
| 内网 AMQP | `rabbitmq-cluster.aict.svc.cluster.local:5672` |
| 内网管理台 | `rabbitmq-cluster.aict.svc.cluster.local:15672` |
| 默认账密 | `admin / RabbitMQ@Passw0rd` |
| Erlang Cookie | `ArchInfraRabbitMQCookie2026` |
| 外网地址 | 默认不暴露 |
| 依赖 | Kubernetes、NFS SC |
| 被谁依赖 | HM、异步任务使用方 |

### 6.8 MongoDB ReplicaSet

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `aict` |
| 内网地址 | `mongodb-cluster-0..2.mongodb-cluster-headless.aict.svc.cluster.local:27017` |
| 默认账密 | `root / MongoDB@Passw0rd` |
| 默认连接串 | `mongodb://root:<password>@mongodb-cluster-0.../admin?replicaSet=rs0&authSource=admin` |
| 外网地址 | 默认不暴露 |
| 依赖 | Kubernetes、NFS SC |
| 被谁依赖 | HM、文档数据库使用方 |

### 6.9 Milvus Cluster

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `milvus-system` |
| 内网地址 | `milvus-cluster.milvus-system.svc.cluster.local:19530` |
| 默认模式 | `cluster` |
| 默认消息队列 | `woodpecker` |
| 外网地址 | 默认不暴露 |
| 依赖 | Kubernetes、NFS SC |
| 被谁依赖 | HM、向量检索使用方 |

### 6.10 HM

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `aict` |
| chat-frontend | `<NODE_IP>:30080` |
| gateway-server | `<NODE_IP>:32136` |
| auth-server | `<NODE_IP>:31285` |
| admin-server | `<NODE_IP>:31196` |
| springai-api | `<NODE_IP>:31721` |
| 依赖 MySQL | `root / passw0rd` |
| 依赖 Redis | `Redis@Passw0rd` |
| 依赖 MinIO | `minioadmin / minioadmin@123` |
| 依赖 MongoDB | `root / MongoDB@Passw0rd` |
| 依赖 RabbitMQ | `admin / RabbitMQ@Passw0rd` |
| 依赖 Milvus | `milvus-cluster.milvus-system.svc.cluster.local:19530` |

## 7. 监控接入基线

当前这批中间件默认监控能力如下：

| 组件 | 默认监控状态 |
| --- | --- |
| MySQL | exporter + ServiceMonitor 已开启 |
| Redis | exporter + ServiceMonitor 已开启 |
| Nacos | metrics + ServiceMonitor 已开启 |
| MinIO | metrics + ServiceMonitor 已开启 |
| RabbitMQ | metrics + ServiceMonitor 已开启 |
| MongoDB | metrics + ServiceMonitor 已开启 |
| Milvus | Milvus ServiceMonitor + embedded etcd PodMonitor + embedded MinIO ServiceMonitor |

Prometheus 只会选择带下列标签的监控资源：

```text
monitoring.archinfra.io/stack=default
```

## 8. 密码轮换建议

这些默认密码只适合交付初始化或测试环境。生产环境建议安装完成后立刻轮换至少这些项：

- Grafana `admin`
- MySQL `root`
- Redis `requirepass`
- MinIO root access key / secret key
- RabbitMQ `admin`
- MongoDB `root`
- Nacos 依赖数据库密码

## 9. 给运维与 AI 的最小行动规则

1. 任何安装前先读当前组件 `--help`。
2. `boot` 用来生成和维护 `inventory.yaml`、`profile.yaml`、`ops-environment.sh`，它不是 Kubernetes 安装器。
3. 任何安装后必须执行 `status` 和 `kubectl get pods,svc,pvc`。
4. 修改默认密码后，要同步更新所有依赖它的上游组件参数。
5. 组件依赖链必须满足：
   - Nacos 依赖 MySQL
   - HM 依赖 MySQL、Redis、MinIO、MongoDB、RabbitMQ、Milvus
6. 看到 PVC `Pending` 时优先检查 NFS Server、NFS SC、`/data/nfs-share`。
7. 看到监控没有被 Prometheus 发现时，先检查 `ServiceMonitor / PodMonitor` 是否带 `monitoring.archinfra.io/stack=default`。
