# 资源、地址与依赖基线

这份文档是组织首页 README 的速查表，目标是让后来的人和 AI 不用翻多个仓库 README，也能快速知道：

- 每个组件默认装在哪
- 默认访问地址是什么
- 默认账密是什么
- 它依赖谁
- `mid` 档位下大概需要多少资源和多少持久化空间

主入口见：

- [README.md](./README.md)

## 1. 仓库与默认产物映射

| 分类 | 仓库 | 服务器上的默认产物 |
| --- | --- | --- |
| 发布包下载 | `archinfra/mc` | `/opt/release/hm-release-downloader-<arch>.run` |
| bootstrap | `archinfra/bootstrapctl` | `/opt/release/boot` |
| LVM | `archinfra/lvm` | `/opt/release/lvm.sh` |
| NFS Server | `archinfra/nfs-server` | `/opt/release/nfs-server.sh` |
| Kubernetes | `archinfra/apps_kubernetes` | `/opt/release/k8s-sealos-linux-<arch>-full.run` |
| NFS StorageClass | `archinfra/apps_nfs-provisioner` | `/opt/release/nfs-provisioner-installer-<arch>.run` |
| metrics-server | `archinfra/apps_metrics-server` | `/opt/release/metrics-server-installer-<arch>.run` |
| Prometheus Stack | `archinfra/app_prometheus-stack` | `/opt/release/prometheus-stack-installer-<arch>.run` |
| Data Protection Operator | `archinfra/data-protection-operator` | `/opt/release/data-protection-operator-<arch>.run` |
| Data Protection Core | `archinfra/dataprotection` | samples、addons、quickstart、user cases |
| MySQL | `archinfra/apps_mysql` | `/opt/release/mysql-installer-<arch>.run` |
| Redis | `archinfra/apps_redis-cluster` | `/opt/release/redis-cluster-installer-<arch>.run` |
| Nacos | `archinfra/apps_nacos` | `/opt/release/nacos-installer-<arch>.run` |
| MinIO | `archinfra/apps_minio-cluster` | `/opt/release/minio-cluster-installer-<arch>.run` |
| RabbitMQ | `archinfra/apps_rabbitmq-cluster` | `/opt/release/rabbitmq-cluster-installer-<arch>.run` |
| MongoDB | `archinfra/apps_mongodb-cluster` | `/opt/release/mongodb-cluster-installer-<arch>.run` |
| Milvus | `archinfra/apps_milvus-cluster` | `/opt/release/milvus-cluster-installer-<arch>.run` |
| HM | `archinfra/hm` | `/opt/release/hm-installer-<arch>.run` |

## 2. 平台默认约定

| 项目 | 默认值 |
| --- | --- |
| 安装包目录 | `/opt/release` |
| 工作目录 | `/opt/cluster-deploy` |
| LVM 挂载点 | `/data` |
| 容器 graph root | `/data/graphroot` |
| containerd 数据目录 | `/data/containerd` |
| NFS Server 所在节点 | `master-01` |
| NFS 导出目录 | `/data/nfs-share` |
| 默认 StorageClass | `nfs` |
| 监控命名空间 | `monitoring` |
| 数据保护 operator 命名空间 | `data-protection-system` |
| 数据保护业务命名空间 | `backup-system` |
| 中间件默认命名空间 | `aict` |
| Milvus 默认命名空间 | `milvus-system` |
| Prometheus 选择器标签 | `monitoring.archinfra.io/stack=default` |
| Grafana Dashboard 标签 | `grafana_dashboard=1` |
| Grafana 可选目录注解 | `grafana_folder=<folder>` |

## 3. 组件总览

| 组件 | 命名空间 | 内网访问 | 外网访问 | 默认账密 / 备注 | 依赖 |
| --- | --- | --- | --- | --- | --- |
| metrics-server | `kube-system` | 聚合 API | 不建议暴露 | 无 | Kubernetes |
| Prometheus Stack | `monitoring` | `prometheus-stack-kube-prom-prometheus.monitoring.svc:9090` | Grafana `30090`，Prometheus `30091` | Grafana `admin / admin@passw0rd` | Kubernetes、NFS SC |
| Data Protection Operator | `data-protection-system` | 控制器，不提供业务地址 | 不暴露 | 业务 CR 常用在 `backup-system` | Kubernetes |
| MySQL | `aict` | `mysql.aict.svc.cluster.local:3306` | `<NODE_IP>:30306` | `root / passw0rd` | Kubernetes、NFS SC |
| Redis Cluster | `aict` | `redis-cluster.aict.svc.cluster.local:6379` | 默认不暴露 | `Redis@Passw0rd` | Kubernetes、NFS SC |
| Nacos | `aict` | `http://nacos.aict.svc.cluster.local:8848/nacos` | HTTP `30094`，gRPC `30930` | 依赖 MySQL `frame_nacos_demo` | MySQL |
| MinIO Cluster | `aict` | `minio.aict.svc.cluster.local:9000` | API `30093`，Console `30092` | `minioadmin / minioadmin@123` | Kubernetes、NFS SC |
| RabbitMQ Cluster | `aict` | `rabbitmq-cluster.aict.svc.cluster.local:5672` | 默认不暴露 | `admin / RabbitMQ@Passw0rd` | Kubernetes、NFS SC |
| MongoDB ReplicaSet | `aict` | `mongodb-cluster-0..2.mongodb-cluster-headless.aict.svc.cluster.local:27017` | 默认不暴露 | `root / MongoDB@Passw0rd` | Kubernetes、NFS SC |
| Milvus Cluster | `milvus-system` | `milvus-cluster.milvus-system.svc.cluster.local:19530` | 默认不暴露 | 默认 `cluster` 模式 | Kubernetes、NFS SC |
| HM | `aict` | 依赖各中间件 service | `30080 / 31721 / 32136 / 31285 / 31196` | 依赖中间件自身账密 | MySQL、Redis、MinIO、MongoDB、RabbitMQ、Milvus |

## 4. `mid` 档位资源基线

说明：

- 这里只统计各安装器当前已经相对固定下来的默认 request/limit
- Prometheus Stack 与 Data Protection Controller 没有在所有组件层面完全统一钉死，不计入硬性总量
- 数值用于容量规划，不替代现场压测和实际业务画像

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

## 5. 默认显式资源总量

不含 Prometheus Stack 与 Data Protection Controller，仅统计当前业务层已经相对固定 request 的组件时，默认 `mid` 档位的最低请求量约为：

| 合计项 | 数值 |
| --- | --- |
| 总 CPU request | 约 `26.01 CPU` |
| 总 Memory request | 约 `44.6Gi` |
| 总显式持久化存储 | 约 `2.78Ti` |

显式持久化存储由这些部分构成：

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

## 6. 监控接入基线

### 6.1 Prometheus 自动发现

Prometheus 只会抓取带下列标签的对象：

```text
monitoring.archinfra.io/stack=default
```

适用资源：

- `ServiceMonitor`
- `PodMonitor`
- `Probe`
- `PrometheusRule`

### 6.2 Grafana 自动 Dashboard 导入

Grafana sidecar 默认跨 namespace 搜索 Dashboard ConfigMap，契约如下：

- 标签：`grafana_dashboard=1`
- 可选注解：`grafana_folder`

建议各中间件仓库在安装器里一起创建：

- 指标暴露资源
- `ServiceMonitor` / `PodMonitor`
- `PrometheusRule`
- Dashboard ConfigMap

### 6.3 当前默认监控开启情况

| 组件 | 默认监控状态 |
| --- | --- |
| MySQL | exporter + ServiceMonitor |
| Redis | exporter + ServiceMonitor |
| Nacos | metrics + ServiceMonitor |
| MinIO | metrics + ServiceMonitor |
| RabbitMQ | metrics + ServiceMonitor + PrometheusRule |
| MongoDB | metrics + ServiceMonitor |
| Milvus | Milvus ServiceMonitor + embedded etcd/MinIO 监控 |

## 7. 数据保护接入基线

### 7.1 当前模型

数据保护平台当前由两部分组成：

- `data-protection-operator`
- `dataprotection`

核心资源模型包括：

- `BackupAddon`
- `BackupSource`
- `BackupStorage`
- `RetentionPolicy`
- `BackupPolicy`
- `BackupJob`
- `RestoreJob`
- `Snapshot`
- `NotificationEndpoint`

### 7.2 当前能力边界

已经明确支持：

- `BackupStorage` 后端：`nfs`、`minio`
- 多存储 fan-out 调度
- 快照恢复
- 导入恢复
- 后端文件保留清理联动
- 通知统一收敛

当前最建议先验证的业务路径：

- MySQL + MinIO
- MySQL + NFS

### 7.3 备份平台推荐理解方式

不要把它理解成“某个中间件内部的备份脚本”。更适合的理解方式是：

- 平台层负责调度、上传下载、保留策略、通知、恢复入口
- 具体中间件通过 addon 提供导入导出逻辑

## 8. 详细信息卡

### 8.1 Prometheus Stack

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `monitoring` |
| Grafana 外网地址 | `http://<NODE_IP>:30090` |
| Prometheus 外网地址 | `http://<NODE_IP>:30091` |
| 默认账密 | Grafana `admin / admin@passw0rd` |
| 监控发现规则 | 只抓带 `monitoring.archinfra.io/stack=default` 的对象 |
| Dashboard 规则 | 自动搜 `grafana_dashboard=1` 的 ConfigMap |

### 8.2 Data Protection Operator

| 项目 | 内容 |
| --- | --- |
| operator 命名空间 | `data-protection-system` |
| 常用业务命名空间 | `backup-system` |
| 外网访问 | 不暴露 |
| 依赖 | Kubernetes；具体备份任务依赖可用的 NFS 或 MinIO |
| 当前建议 | 先走 MySQL 备份恢复样例链路 |

### 8.3 MySQL

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `aict` |
| 内网地址 | `mysql.aict.svc.cluster.local:3306` |
| 主节点直连 | `mysql-0.mysql.aict.svc.cluster.local:3306` |
| 外网地址 | `<NODE_IP>:30306` |
| 默认账密 | `root / passw0rd` |
| 其他默认账号 | `repl / repl@passw0rd`，`orch / orch@passw0rd`，`mysqld_exporter / exporter@passw0rd` |
| 被谁依赖 | Nacos、HM、备份恢复链路 |

### 8.4 Redis Cluster

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `aict` |
| 内网地址 | `redis-cluster.aict.svc.cluster.local:6379` |
| Headless | `redis-cluster-headless.aict.svc.cluster.local` |
| 外网地址 | 默认不暴露 |
| 默认密码 | `Redis@Passw0rd` |
| 被谁依赖 | HM |

### 8.5 Nacos

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `aict` |
| 内网地址 | `http://nacos.aict.svc.cluster.local:8848/nacos` |
| 外网地址 | HTTP `<NODE_IP>:30094`，gRPC `<NODE_IP>:30930` |
| 默认数据库 | `frame_nacos_demo` |
| 默认 MySQL 连接 | `mysql-0.mysql.aict:3306` |
| 默认初始化 | 幂等 SQL + `cmict-share.yaml` |

### 8.6 MinIO Cluster

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `aict` |
| 内网 API | `minio.aict.svc.cluster.local:9000` |
| 内网 Console | `minio-console.aict.svc.cluster.local:9090` |
| 外网 API | `<NODE_IP>:30093` |
| 外网 Console | `<NODE_IP>:30092` |
| 默认账密 | `minioadmin / minioadmin@123` |
| 被谁依赖 | HM、对象存储使用方、数据保护备份后端 |

### 8.7 RabbitMQ Cluster

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `aict` |
| 内网 AMQP | `rabbitmq-cluster.aict.svc.cluster.local:5672` |
| 内网管理台 | `rabbitmq-cluster.aict.svc.cluster.local:15672` |
| 默认账密 | `admin / RabbitMQ@Passw0rd` |
| Erlang Cookie | `ArchInfraRabbitMQCookie2026` |
| 被谁依赖 | HM、异步任务使用方 |

### 8.8 MongoDB ReplicaSet

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `aict` |
| 内网地址 | `mongodb-cluster-0..2.mongodb-cluster-headless.aict.svc.cluster.local:27017` |
| 默认账密 | `root / MongoDB@Passw0rd` |
| 外网地址 | 默认不暴露 |
| 被谁依赖 | HM、文档数据库使用方 |

### 8.9 Milvus Cluster

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `milvus-system` |
| 内网地址 | `milvus-cluster.milvus-system.svc.cluster.local:19530` |
| 默认模式 | `cluster` |
| 默认消息队列 | `woodpecker` |
| 外网地址 | 默认不暴露 |
| 被谁依赖 | HM、向量检索使用方 |

### 8.10 HM

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `aict` |
| chat-frontend | `<NODE_IP>:30080` |
| gateway-server | `<NODE_IP>:32136` |
| auth-server | `<NODE_IP>:31285` |
| admin-server | `<NODE_IP>:31196` |
| springai-api | `<NODE_IP>:31721` |
| 依赖 | MySQL、Redis、MinIO、MongoDB、RabbitMQ、Milvus |

## 9. 运维与 AI 最小规则

1. 任何安装前先执行 `help`。
2. 任何安装后必须执行 `status`。
3. 任何状态异常先检查依赖链是否满足。
4. 看到 PVC `Pending` 时先查 NFS。
5. 看到监控未发现时先查标签契约。
6. 看到 Dashboard 未出现时先查 `grafana_dashboard=1` 和 `grafana_folder`。
7. 看到备份未成功时先查 operator、storage probe、addon/source 注册状态。
8. 生产环境部署完成后立刻轮换默认密码。
