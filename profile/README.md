# 集群部署与运维手册

这份文档面向两类使用者：

- 新接手这套环境的人
- 后续会接管部署流程的 AI / 自动化系统

目标是让执行者在一台已经放好安装包的 Linux 服务器上，按固定顺序完成：

1. 主机初始化
2. LVM 与数据盘挂载
3. NFS Server
4. Kubernetes
5. NFS StorageClass
6. metrics-server
7. Prometheus Stack
8. MySQL / Redis / Nacos / MinIO / RabbitMQ / MongoDB / Milvus
9. HM 业务应用

同时，每装完一个组件，都能知道：

- 组件装在什么命名空间
- 内网访问地址是什么
- 外网访问地址是什么
- 默认账密是什么
- 它依赖了什么组件
- 最低资源需求大概是多少

配套汇总表见：

- [RESOURCE-BASELINE.zh-CN.md](C:/Users/yuanyp8/Desktop/archinfra/RESOURCE-BASELINE.zh-CN.md)

## 1. 约定与目录

本文默认以下约定：

- 所有 `.run` 包、初始化脚本、`bootstrapctl` 都已经放在服务器的 `/opt/release`
- 所有命令默认在 Linux 主机上执行
- 部署控制节点默认是 `master-01`
- 第一台 master 同时承担 NFS Server
- 动态卷的默认 `StorageClass` 名称为 `nfs`
- 中间件默认以 `mid` 资源档位安装
- 除非明确需要外网暴露，否则优先使用 ClusterIP 或内部域名访问

关键目录约定如下：

| 路径 | 含义 |
| --- | --- |
| `/opt/release` | 所有安装器和脚本所在目录 |
| `/opt/cluster-deploy` | 本文档建议的操作工作目录 |
| `/etc/profile.d/ops-environment.sh` | 旧 Shell 工具消费的环境变量文件 |
| `/data` | LVM 默认挂载点 |
| `/data/nfs-share` | NFS 导出目录 |
| `/data/graphroot` | 容器镜像 / graphroot 默认目录 |
| `/data/containerd` | containerd 数据目录 |

## 2. 执行前检查

先在控制节点执行一次统一环境准备：

```bash
set -euo pipefail

export RELEASE_DIR=/opt/release
export WORK_DIR=/opt/cluster-deploy
mkdir -p "${WORK_DIR}"
cd "${WORK_DIR}"

ARCH_RAW="$(uname -m)"
case "${ARCH_RAW}" in
  x86_64|amd64)
    export PKG_ARCH=amd64
    ;;
  aarch64|arm64)
    export PKG_ARCH=arm64
    ;;
  *)
    echo "[ERROR] Unsupported arch: ${ARCH_RAW}" >&2
    exit 1
    ;;
esac

run_bootstrapctl() {
  if [ -x "${RELEASE_DIR}/bootstrapctl" ]; then
    "${RELEASE_DIR}/bootstrapctl" "$@"
    return
  fi

  if [ -d "${RELEASE_DIR}/bootstrapctl" ]; then
    (
      cd "${RELEASE_DIR}/bootstrapctl"
      go run ./cmd/bootstrapctl "$@"
    )
    return
  fi

  echo "[ERROR] bootstrapctl not found under ${RELEASE_DIR}" >&2
  return 1
}

pick_run() {
  local path="$1"
  if [ -f "${path}" ]; then
    printf '%s\n' "${path}"
    return 0
  fi
  return 1
}

find_optional_run() {
  find "${RELEASE_DIR}" -maxdepth 1 -type f "$@" | sort | head -n 1
}

export LVM_SCRIPT="${RELEASE_DIR}/init/lvm.sh"
export NFS_SERVER_SCRIPT="${RELEASE_DIR}/init/nfs-server.sh"

export METRICS_SERVER_RUN="$(pick_run "${RELEASE_DIR}/metrics-server-installer-${PKG_ARCH}.run")"
export PROMETHEUS_RUN="$(pick_run "${RELEASE_DIR}/prometheus-stack-installer-${PKG_ARCH}.run")"
export MYSQL_RUN="$(pick_run "${RELEASE_DIR}/mysql-installer-${PKG_ARCH}.run")"
export REDIS_RUN="$(pick_run "${RELEASE_DIR}/redis-cluster-installer-${PKG_ARCH}.run")"
export NACOS_RUN="$(pick_run "${RELEASE_DIR}/nacos-installer-${PKG_ARCH}.run")"
export MINIO_RUN="$(pick_run "${RELEASE_DIR}/minio-cluster-installer-${PKG_ARCH}.run")"
export RABBITMQ_RUN="$(pick_run "${RELEASE_DIR}/rabbitmq-cluster-installer-${PKG_ARCH}.run")"
export MONGODB_RUN="$(pick_run "${RELEASE_DIR}/mongodb-cluster-installer-${PKG_ARCH}.run")"
export MILVUS_RUN="$(pick_run "${RELEASE_DIR}/milvus-cluster-installer-${PKG_ARCH}.run")"
export HM_RUN="$(pick_run "${RELEASE_DIR}/hm-installer-${PKG_ARCH}.run")"

export K8S_INSTALLER="$(find_optional_run \\( -name "*k8s*${PKG_ARCH}.run" -o -name "*kubernetes*${PKG_ARCH}.run" \\))"
export NFS_SC_INSTALLER="$(find_optional_run \\( -name "*nfs*sc*${PKG_ARCH}.run" -o -name "*nfs*storage*${PKG_ARCH}.run" \\))"

ls -lh "${RELEASE_DIR}"
```

执行前必须确认：

- `bootstrapctl` 可用
- `/opt/release/init/lvm.sh` 和 `/opt/release/init/nfs-server.sh` 存在
- 所有需要的 `.run` 包和当前架构一致
- `K8S_INSTALLER` 与 `NFS_SC_INSTALLER` 能从 `/opt/release` 解析出来

如果 `K8S_INSTALLER` 或 `NFS_SC_INSTALLER` 为空，不要继续，先去 `/opt/release` 补齐包名。

## 3. 第一步：bootstrap host init

### 3.1 生成初始化模板

```bash
cd "${WORK_DIR}"
run_bootstrapctl init -d ./bootstrap/demo-env -c demo-env
```

这一步会生成两个文件：

- `./bootstrap/demo-env/inventory.yaml`
- `./bootstrap/demo-env/profile.yaml`

### 3.2 修改 inventory

需要把真实节点、IP、SSH 信息补进去。一个最小示例：

```yaml
cluster_name: hm-prod

transport:
  ssh_user: root
  ssh_port: 22
  ssh_password: changeme
  use_sudo: false

nodes:
  - name: master-01
    ip: 10.0.0.11
    roles: [master]

  - name: master-02
    ip: 10.0.0.12
    roles: [master]

  - name: master-03
    ip: 10.0.0.13
    roles: [master]

  - name: worker-01
    ip: 10.0.0.21
    roles: [worker]

  - name: worker-02
    ip: 10.0.0.22
    roles: [worker]
```

建议同时检查 `profile.yaml` 中这些关键值：

- `graph_root: /data/graphroot`
- `cri_root: /data/containerd`
- swap / selinux / firewall 处理策略
- 内核网络模块和 sysctl
- 文件句柄和进程数限制

### 3.3 扫描、规划、应用、验收

```bash
cd "${WORK_DIR}"
run_bootstrapctl scan -i ./bootstrap/demo-env/inventory.yaml -t 20s
run_bootstrapctl plan -i ./bootstrap/demo-env/inventory.yaml -p ./bootstrap/demo-env/profile.yaml -t 20s
run_bootstrapctl apply -i ./bootstrap/demo-env/inventory.yaml -p ./bootstrap/demo-env/profile.yaml -t 20s
run_bootstrapctl verify -i ./bootstrap/demo-env/inventory.yaml -p ./bootstrap/demo-env/profile.yaml -t 20s
```

### 3.4 导出旧脚本兼容环境

`scan/plan/apply/verify` 会自动同步 `ops-environment.sh`，这里再手工刷新一次，方便后面的 LVM/NFS 脚本消费：

```bash
cd "${WORK_DIR}"
run_bootstrapctl export-ops-env \
  -i ./bootstrap/demo-env/inventory.yaml \
  -o ./bootstrap/demo-env/ops-environment.sh

cp ./bootstrap/demo-env/ops-environment.sh /etc/profile.d/ops-environment.sh
source /etc/profile.d/ops-environment.sh
```

### 3.5 验收标准

```bash
test -f /etc/profile.d/ops-environment.sh
grep -E 'NODE_IPS|NODE_NAMES' /etc/profile.d/ops-environment.sh
```

## 4. 第二步：LVM 挂载 /data

本文默认所有节点都把业务数据盘挂到 `/data`，LVM 默认值如下：

- VG: `ops_vg_data`
- LV: `ops_lv_data`
- FS: `xfs`
- Mount: `/data`
- Disk: `/dev/vdb`

推荐命令：

```bash
chmod +x "${LVM_SCRIPT}"
"${LVM_SCRIPT}" -y --vg-name ops_vg_data --lv-name ops_lv_data --mount-point /data --fs-type xfs --disks /dev/vdb
```

执行时按下面输入：

1. 节点选择输入 `a`
2. 操作选择输入 `1`
3. 其余回车，使用默认 VG / LV / FS / 挂载点

### 4.1 验收标准

```bash
lsblk
df -h /data
grep '/data' /etc/fstab
```

### 4.2 运维说明

- 以后如果需要扩容，还是用同一个脚本，选择 `2`
- 这个阶段只做主机级存储准备，不直接创建 Kubernetes PVC

## 5. 第三步：在 master-01 安装 NFS Server

这一步只在第一台 master 上执行。默认导出目录是 `/data/nfs-share`。

```bash
chmod +x "${NFS_SERVER_SCRIPT}"
"${NFS_SERVER_SCRIPT}" -y
```

如果要显式指定目录：

```bash
"${NFS_SERVER_SCRIPT}" -d /data/nfs-share -y
```

### 5.1 验收标准

```bash
systemctl status nfs-server --no-pager
exportfs -v
showmount -e localhost
test -d /data/nfs-share
cat /etc/exports
```

### 5.2 运维说明

- NFS 默认安装在 `master-01`
- 默认导出目录是 `/data/nfs-share`
- 后续所有使用 `nfs` StorageClass 的 PVC 最终都会落到这里

## 6. 第四步：安装 Kubernetes

当前工作区里没有 Kubernetes 安装器源码，所以这里按服务器上已经放好的 `.run` 包执行。先确认实际包名：

```bash
echo "${K8S_INSTALLER}"
test -n "${K8S_INSTALLER}"
"${K8S_INSTALLER}" --help || "${K8S_INSTALLER}" help
```

如果它遵循统一 `.run` 范式，直接执行：

```bash
"${K8S_INSTALLER}" install -y
```

如果它不是统一 action 风格，就严格按该安装器自己的 `--help` 或 README 执行，不要猜参数。

### 6.1 验收标准

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get sc
```

## 7. 第五步：安装 NFS StorageClass

同样先确认实际包名：

```bash
echo "${NFS_SC_INSTALLER}"
test -n "${NFS_SC_INSTALLER}"
"${NFS_SC_INSTALLER}" --help || "${NFS_SC_INSTALLER}" help
```

如果它也是统一 `.run` 范式，优先使用：

```bash
"${NFS_SC_INSTALLER}" install -y
```

### 7.1 验收标准

```bash
kubectl get sc
kubectl get pods -A | grep -i nfs
```

至少要看到：

- `StorageClass/nfs`
- NFS provisioner 处于 `Running`

## 8. 第六步：安装 metrics-server

建议在 kubelet 自签 TLS 环境直接启用 `--kubelet-insecure-tls`。

```bash
"${METRICS_SERVER_RUN}" install --kubelet-insecure-tls -y
```

### 安装信息

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `kube-system` |
| 依赖 | Kubernetes 已安装 |
| 内网访问 | 作为聚合 API 服务，不直接对外提供业务地址 |
| 外网访问 | 不建议暴露 |
| 默认资源 | `100m CPU / 200Mi` request |

### 验收标准

```bash
"${METRICS_SERVER_RUN}" status
kubectl top nodes
kubectl top pods -A
```

## 9. 第七步：安装 Prometheus Stack

Prometheus Stack 建议在安装各业务组件前先装好，这样后续默认开启的 `ServiceMonitor` / `PodMonitor` 会自动接入。

```bash
"${PROMETHEUS_RUN}" install \
  --namespace monitoring \
  --grafana-admin-password 'admin@passw0rd' \
  -y
```

### 安装信息

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `monitoring` |
| 依赖 | Kubernetes、NFS StorageClass |
| 监控发现规则 | 只抓带 `monitoring.archinfra.io/stack=default` 的资源 |
| Grafana 默认密码 | `admin@passw0rd` |
| 默认持久化 | Prometheus `200Gi`，Grafana `10Gi`，Alertmanager `10Gi` |
| 内网访问 | 建议通过 `kubectl get svc -n monitoring` 查看实际 Service |
| 外网访问 | 默认不暴露，建议通过 ingress / NodePort / port-forward 二次发布 |

### 建议访问命令

```bash
kubectl get svc -n monitoring -l app.kubernetes.io/instance=prometheus-stack
kubectl -n monitoring port-forward svc/prometheus-stack-grafana 3000:80
kubectl -n monitoring port-forward svc/prometheus-operated 9090:9090
```

### 验收标准

```bash
"${PROMETHEUS_RUN}" status
kubectl get pods,svc -n monitoring
kubectl get servicemonitor,podmonitor,prometheusrule -A -l monitoring.archinfra.io/stack=default
```

## 10. 第八步：安装 MySQL

```bash
"${MYSQL_RUN}" install --resource-profile mid -y
```

### 安装信息

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `aict` |
| 内网访问 | `mysql.aict.svc.cluster.local:3306` |
| 主节点直连 | `mysql-0.mysql.aict.svc.cluster.local:3306` |
| 外网访问 | `<任意节点IP>:30306` |
| 默认账号 | `root / passw0rd` |
| 其他默认账号 | `repl / repl@passw0rd`，`orch / orch@passw0rd`，`mysqld_exporter / exporter@passw0rd` |
| 依赖 | Kubernetes、NFS StorageClass、Prometheus 可选 |
| 默认监控 | 已开启 exporter + ServiceMonitor |
| `mid` 资源 | request `600m / 1152Mi`，limit `1200m / 2304Mi` |
| 默认存储 | `10Gi` |

### 验收标准

```bash
"${MYSQL_RUN}" status
kubectl get pods,svc,pvc -n aict
kubectl logs -n aict mysql-0 -c mysql --tail=50
```

## 11. 第九步：安装 Redis Cluster

```bash
"${REDIS_RUN}" install --resource-profile mid -y
```

### 安装信息

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `aict` |
| 内网访问 | `redis-cluster.aict.svc.cluster.local:6379` |
| Headless | `redis-cluster-headless.aict.svc.cluster.local` |
| 外网访问 | 默认不暴露 |
| 默认密码 | `Redis@Passw0rd` |
| 依赖 | Kubernetes、NFS StorageClass、Prometheus 可选 |
| 默认监控 | exporter + ServiceMonitor 默认开启 |
| `mid` 资源 | 总 request 约 `3.6 CPU / 6.75Gi`，总 limit 约 `7.2 CPU / 13.5Gi` |
| 默认存储 | 总计约 `60Gi` |

### 验收标准

```bash
"${REDIS_RUN}" status
kubectl get pods,svc,pvc -n aict
```

## 12. 第十步：安装 Nacos

Nacos 依赖 MySQL，默认会连接到 `mysql-0.mysql.aict`，并初始化 `frame_nacos_demo` 数据库。

```bash
"${NACOS_RUN}" install \
  --mysql-host mysql-0.mysql.aict \
  --mysql-password 'passw0rd' \
  --resource-profile mid \
  -y
```

### 安装信息

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `aict` |
| 内网访问 | `http://nacos.aict.svc.cluster.local:8848/nacos` |
| 外网访问 | HTTP `<任意节点IP>:30094`，gRPC `<任意节点IP>:30930` |
| 数据库依赖 | MySQL `root / passw0rd`，DB `frame_nacos_demo` |
| 默认监控 | metrics + ServiceMonitor 默认开启 |
| 默认导入 | 初始化 SQL + `cmict-share.yaml` |
| `mid` 资源 | request `500m / 1Gi`，limit `1 CPU / 2Gi` |
| 默认持久化 | 不单独声明 PVC，数据持久化主要落在 MySQL |

### 验收标准

```bash
"${NACOS_RUN}" status
kubectl get pods,svc -n aict
curl -sf "http://127.0.0.1:30094/nacos/"
```

## 13. 第十一步：安装 MinIO Cluster

```bash
"${MINIO_RUN}" install --resource-profile mid -y
```

### 安装信息

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `aict` |
| 内网 API | `minio.aict.svc.cluster.local:9000` |
| 内网 Console | `minio-console.aict.svc.cluster.local:9090` |
| 外网 API | `<任意节点IP>:30093` |
| 外网 Console | `<任意节点IP>:30092` |
| 默认账密 | `minioadmin / minioadmin@123` |
| 依赖 | Kubernetes、NFS StorageClass、Prometheus 可选 |
| 默认监控 | metrics + ServiceMonitor 默认开启 |
| `mid` 资源 | 稳态总 request 约 `2.1 CPU / 4.25Gi` |
| 默认存储 | `4 x 500Gi = 2Ti` |

### 验收标准

```bash
"${MINIO_RUN}" status
kubectl get pods,svc,pvc -n aict
```

## 14. 第十二步：安装 RabbitMQ Cluster

```bash
"${RABBITMQ_RUN}" install --resource-profile mid -y
```

### 安装信息

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `aict` |
| 内网 AMQP | `rabbitmq-cluster.aict.svc.cluster.local:5672` |
| 内网管理台 | `rabbitmq-cluster.aict.svc.cluster.local:15672` |
| 外网访问 | 默认不暴露，改成 NodePort 时常用 `30672 / 31672` |
| 默认账号 | `admin / RabbitMQ@Passw0rd` |
| Erlang Cookie | `ArchInfraRabbitMQCookie2026` |
| 依赖 | Kubernetes、NFS StorageClass、Prometheus 可选 |
| 默认监控 | metrics + ServiceMonitor 默认开启 |
| `mid` 资源 | 总 request 约 `1.5 CPU / 3Gi`，总 limit 约 `3 CPU / 6Gi` |
| 默认存储 | `3 x 8Gi = 24Gi` |

### 验收标准

```bash
"${RABBITMQ_RUN}" status
kubectl get pods,svc,pvc -n aict
```

## 15. 第十三步：安装 MongoDB ReplicaSet

```bash
"${MONGODB_RUN}" install --resource-profile mid -y
```

### 安装信息

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `aict` |
| 内网访问 | `mongodb-cluster-0.mongodb-cluster-headless.aict.svc.cluster.local:27017` 等 3 节点 |
| 推荐连接串 | `mongodb://root:MongoDB%40Passw0rd@mongodb-cluster-0.mongodb-cluster-headless.aict.svc.cluster.local:27017,mongodb-cluster-1.mongodb-cluster-headless.aict.svc.cluster.local:27017,mongodb-cluster-2.mongodb-cluster-headless.aict.svc.cluster.local:27017/admin?replicaSet=rs0&authSource=admin` |
| 外网访问 | 默认不暴露 |
| 默认账号 | `root / MongoDB@Passw0rd` |
| 副本集密钥 | `ArchInfraMongoReplicaSetKey2026` |
| 依赖 | Kubernetes、NFS StorageClass、Prometheus 可选 |
| 默认监控 | metrics + ServiceMonitor 默认开启 |
| `mid` 资源 | 总 request 约 `1.8 CPU / 3456Mi`，总 limit 约 `3.6 CPU / 6912Mi` |
| 默认存储 | `3 x 20Gi = 60Gi` |

### 验收标准

```bash
"${MONGODB_RUN}" status
kubectl get pods,svc,pvc -n aict
```

## 16. 第十四步：安装 Milvus Cluster

Milvus 默认装在独立命名空间 `milvus-system`。如果是测试机或单机资源比较紧，可以改用 `--compact`。

标准部署：

```bash
"${MILVUS_RUN}" install --resource-profile mid -y
```

单机紧凑部署：

```bash
"${MILVUS_RUN}" install --compact --resource-profile mid -y
```

### 安装信息

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `milvus-system` |
| 内网访问 | `milvus-cluster.milvus-system.svc.cluster.local:19530` |
| 外网访问 | 默认不暴露 |
| 默认模式 | `cluster` |
| 默认消息队列 | `woodpecker` |
| 依赖 | Kubernetes、NFS StorageClass、Prometheus 可选 |
| 默认监控 | Milvus ServiceMonitor + embedded etcd PodMonitor + embedded MinIO ServiceMonitor |
| `mid` 资源 | 默认 cluster 模式总 request 约 `4 CPU / 13Gi` |
| 默认存储 | 默认约 `460Gi` |

### 验收标准

```bash
"${MILVUS_RUN}" status
kubectl get pods,svc,pvc -n milvus-system
kubectl get servicemonitor,podmonitor -n milvus-system
```

## 17. 第十五步：安装 HM 业务应用

HM 是业务整合层，默认依赖以下组件都已可用：

- MySQL
- Redis
- MinIO
- MongoDB
- RabbitMQ
- Milvus

安装命令：

```bash
"${HM_RUN}" install -y
```

### 安装信息

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `aict` |
| 依赖 MySQL | `mysql.aict.svc.cluster.local:3306`，`root / passw0rd` |
| 依赖 Redis | `redis-cluster-headless.aict.svc.cluster.local:6379`，密码 `Redis@Passw0rd` |
| 依赖 MinIO | `http://minio.aict.svc.cluster.local:9000`，`minioadmin / minioadmin@123` |
| 依赖 MongoDB | `root / MongoDB@Passw0rd` |
| 依赖 RabbitMQ | `admin / RabbitMQ@Passw0rd` |
| 依赖 Milvus | `milvus-cluster.milvus-system.svc.cluster.local:19530` |
| 默认外网入口 | `chat-frontend` `<任意节点IP>:30080` |
| 其他默认入口 | `gateway-server` `<任意节点IP>:32136`，`auth-server` `<任意节点IP>:31285`，`admin-server` `<任意节点IP>:31196`，`springai-api` `<任意节点IP>:31721` |
| 默认 PVC | `kbase` 使用 `10Gi` NFS PVC |
| 默认总 request | 约 `11.81 CPU / 11.9Gi` |

### 验收标准

```bash
"${HM_RUN}" status
kubectl get pods,svc,pvc -n aict
```

## 18. 推荐安装顺序总表

| 顺序 | 组件 | 必须依赖 |
| --- | --- | --- |
| 1 | bootstrapctl host init | 无 |
| 2 | LVM `/data` | bootstrap inventory 已完成 |
| 3 | NFS Server | `/data` 已挂载 |
| 4 | Kubernetes | 主机初始化完成 |
| 5 | NFS StorageClass | Kubernetes + NFS Server |
| 6 | metrics-server | Kubernetes |
| 7 | Prometheus Stack | Kubernetes + NFS StorageClass |
| 8 | MySQL | Kubernetes + NFS StorageClass |
| 9 | Redis | Kubernetes + NFS StorageClass |
| 10 | Nacos | MySQL |
| 11 | MinIO | Kubernetes + NFS StorageClass |
| 12 | RabbitMQ | Kubernetes + NFS StorageClass |
| 13 | MongoDB | Kubernetes + NFS StorageClass |
| 14 | Milvus | Kubernetes + NFS StorageClass |
| 15 | HM | MySQL + Redis + MinIO + MongoDB + RabbitMQ + Milvus |

## 19. 组件互相调用关系

```mermaid
graph TD
  HM["HM 业务应用"] --> MySQL["MySQL"]
  HM --> Redis["Redis Cluster"]
  HM --> MinIO["MinIO Cluster"]
  HM --> MongoDB["MongoDB ReplicaSet"]
  HM --> RabbitMQ["RabbitMQ Cluster"]
  HM --> Milvus["Milvus Cluster"]
  Nacos["Nacos"] --> MySQL
  Prom["Prometheus Stack"] --> SM["ServiceMonitor / PodMonitor"]
  MySQL --> SM
  Redis --> SM
  Nacos --> SM
  MinIO --> SM
  RabbitMQ --> SM
  MongoDB --> SM
  Milvus --> SM
```

## 20. 安装后必须记录的运维信息

每装完一个组件，都建议把下面这张信息卡补到你的 CMDB、交付文档或自动化状态文件里：

```text
组件名:
安装时间:
安装命令:
命名空间:
内部地址:
外部地址:
默认账密:
依赖组件:
PVC / 存储需求:
kubectl status 结果:
备注:
```

## 21. 给 AI / 自动化系统的执行规约

如果后面是 AI 接手部署，建议严格遵循下面规则：

1. 先检测架构，再选 `amd64` 或 `arm64` 安装包。
2. 先列出 `/opt/release`，确认实际包名，再执行。
3. 对当前工作区没有源码的安装器，不要猜参数，先执行 `--help` 或 `help`。
4. 每装完一个组件都必须执行：
   - `<installer> status`
   - `kubectl get pods,svc,pvc -n <namespace>`
   - 记录内部地址、外部地址、账密、依赖
5. 安装 Nacos 之前必须确认 MySQL 已 ready。
6. 安装 HM 之前必须确认 MySQL、Redis、MinIO、MongoDB、RabbitMQ、Milvus 已 ready。
7. 看到 `StorageClass` 不存在、PVC `Pending`、Pod `CrashLoopBackOff` 时必须停下来排障，不能跳过。
8. 生产环境应在部署完成后尽快轮换默认密码。

## 22. 常用运维命令

集群总体巡检：

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get pvc -A
kubectl get sc
```

监控资源巡检：

```bash
kubectl get servicemonitor,podmonitor,prometheusrule -A -l monitoring.archinfra.io/stack=default
```

NFS 与本地数据盘巡检：

```bash
df -h /data
df -h /data/nfs-share
exportfs -v
```

容器数据目录巡检：

```bash
du -sh /data/graphroot || true
du -sh /data/containerd || true
```

## 23. 生产前必须再次确认

- `master-01` 的 `/data` 容量足够承载 NFS 总体数据
- 所有默认密码已按环境要求修改
- Prometheus 已经发现各组件的监控对象
- 业务入口已经通过 NodePort / LB / Ingress 正确暴露
- `RESOURCE-BASELINE.zh-CN.md` 中的资源和存储估算与你的节点规模匹配
