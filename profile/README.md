# 集群统一部署与运维手册

这份文档面向两类使用者：

- 新接手这套环境的人
- 未来会接管部署流程的 AI / 自动化系统

目标是让执行者在一台已经放好所有安装包的 Linux 服务器上，按照统一顺序完成：

1. `boot` 生成 bootstrap 配置
2. LVM 初始化并挂载 `/data`
3. 在 `master-01` 安装 NFS Server
4. 安装 Kubernetes
5. 安装 NFS StorageClass
6. 安装 metrics-server
7. 安装 Prometheus Stack
8. 安装 MySQL、Redis、Nacos、MinIO、RabbitMQ、MongoDB、Milvus
9. 安装 HM 业务应用

同时保证后来的人和 AI 能快速知道：

- 每一步要执行什么命令
- 每个组件的默认地址、账密、依赖和命名空间
- 哪些组件必须先装，哪些组件必须最后装
- 集群的关键存储路径和资源基线是什么

配套汇总表见：

- [RESOURCE-BASELINE.zh-CN.md](./RESOURCE-BASELINE.zh-CN.md)

## 1. 项目与产物映射

当前统一部署链路对应的项目如下：

| 阶段 | 仓库 | 服务器上的默认产物 |
| --- | --- | --- |
| bootstrap host init | `archinfra/bootstrapctl` | `/opt/release/boot` |
| LVM | `archinfra/lvm` | `/opt/release/lvm.sh` |
| NFS Server | `archinfra/nfs-server` | `/opt/release/nfs-server.sh` |
| Kubernetes | `archinfra/apps_kubernetes` | `/opt/release/k8s-sealos-linux-<arch>-full.run` |
| NFS StorageClass | `archinfra/apps_nfs-provisioner` | `/opt/release/nfs-provisioner-installer-<arch>.run` |
| metrics-server | `archinfra/apps_metrics-server` | `/opt/release/metrics-server-installer-<arch>.run` |
| Prometheus Stack | `archinfra/app_prometheus-stack` | `/opt/release/prometheus-stack-installer-<arch>.run` |
| MySQL | `archinfra/apps_mysql` | `/opt/release/mysql-installer-<arch>.run` |
| Redis | `archinfra/apps_redis-cluster` | `/opt/release/redis-cluster-installer-<arch>.run` |
| Nacos | `archinfra/apps_nacos` | `/opt/release/nacos-installer-<arch>.run` |
| MinIO | `archinfra/apps_minio-cluster` | `/opt/release/minio-cluster-installer-<arch>.run` |
| RabbitMQ | `archinfra/apps_rabbitmq-cluster` | `/opt/release/rabbitmq-cluster-installer-<arch>.run` |
| MongoDB | `archinfra/apps_mongodb-cluster` | `/opt/release/mongodb-cluster-installer-<arch>.run` |
| Milvus | `archinfra/apps_milvus-cluster` | `/opt/release/milvus-cluster-installer-<arch>.run` |
| HM | `archinfra/hm` | `/opt/release/hm-installer-<arch>.run` |

## 2. 目录与默认约定

本文默认以下目录与角色约定：

| 路径 / 项目 | 默认值 |
| --- | --- |
| 安装包目录 | `/opt/release` |
| 建议工作目录 | `/opt/cluster-deploy` |
| bootstrap 兼容环境文件 | `/etc/profile.d/ops-environment.sh` |
| LVM 挂载点 | `/data` |
| NFS 导出目录 | `/data/nfs-share` |
| 容器 graph root | `/data/graphroot` |
| containerd 数据目录 | `/data/containerd` |
| NFS Server 节点 | `master-01` |
| 默认 StorageClass | `nfs` |
| 监控标签 | `monitoring.archinfra.io/stack=default` |

部署原则：

- 默认使用 `mid` 资源档位
- 第一台 master 兼任 NFS Server
- 先装监控底座，再装业务中间件
- Nacos 必须在 MySQL ready 后安装
- HM 必须在 MySQL、Redis、MinIO、MongoDB、RabbitMQ、Milvus 都 ready 后安装

## 3. 第 0 步：统一环境变量

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

export BOOT_BIN="${RELEASE_DIR}/boot"
export LVM_SCRIPT="${RELEASE_DIR}/lvm.sh"
export NFS_SERVER_SCRIPT="${RELEASE_DIR}/nfs-server.sh"
export K8S_RUN="${RELEASE_DIR}/k8s-sealos-linux-${PKG_ARCH}-full.run"
export NFS_SC_RUN="${RELEASE_DIR}/nfs-provisioner-installer-${PKG_ARCH}.run"
export METRICS_SERVER_RUN="${RELEASE_DIR}/metrics-server-installer-${PKG_ARCH}.run"
export PROMETHEUS_RUN="${RELEASE_DIR}/prometheus-stack-installer-${PKG_ARCH}.run"
export MYSQL_RUN="${RELEASE_DIR}/mysql-installer-${PKG_ARCH}.run"
export REDIS_RUN="${RELEASE_DIR}/redis-cluster-installer-${PKG_ARCH}.run"
export NACOS_RUN="${RELEASE_DIR}/nacos-installer-${PKG_ARCH}.run"
export MINIO_RUN="${RELEASE_DIR}/minio-cluster-installer-${PKG_ARCH}.run"
export RABBITMQ_RUN="${RELEASE_DIR}/rabbitmq-cluster-installer-${PKG_ARCH}.run"
export MONGODB_RUN="${RELEASE_DIR}/mongodb-cluster-installer-${PKG_ARCH}.run"
export MILVUS_RUN="${RELEASE_DIR}/milvus-cluster-installer-${PKG_ARCH}.run"
export HM_RUN="${RELEASE_DIR}/hm-installer-${PKG_ARCH}.run"

export MASTER1_IP="10.0.0.11"
export MASTER_IPS="10.0.0.11,10.0.0.12,10.0.0.13"
export WORKER_IPS="10.0.0.21,10.0.0.22"
export SSH_USER="root"
export SSH_PRIVATE_KEY="/root/.ssh/id_rsa"
export NFS_SERVER_IP="${MASTER1_IP}"

ls -lh "${RELEASE_DIR}"
```

执行前必须确认：

- `${BOOT_BIN}` 存在
- `${LVM_SCRIPT}` 和 `${NFS_SERVER_SCRIPT}` 存在
- `${K8S_RUN}` 和 `${NFS_SC_RUN}` 存在
- 其余 `.run` 包和当前机器架构一致
- `MASTER_IPS`、`WORKER_IPS`、`MASTER1_IP`、`SSH_PRIVATE_KEY` 已改成真实值

## 4. 第 1 步：使用 `./boot` 生成 bootstrap 配置

### 4.1 生成模板

```bash
cd "${WORK_DIR}"
cp "${BOOT_BIN}" ./boot
chmod +x ./boot

./boot init -d ./bootstrap/demo-env -c demo-env
```

这一步会生成两个文件：

- `./bootstrap/demo-env/inventory.yaml`
- `./bootstrap/demo-env/profile.yaml`

### 4.2 修改 `inventory.yaml`

最小示例如下：

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

建议同时检查 `profile.yaml` 中至少这些关键值：

- `graph_root: /data/graphroot`
- `cri_root: /data/containerd`
- swap / selinux / firewall 处理策略
- 内核网络模块和 sysctl
- 文件句柄和进程数限制

### 4.3 扫描、规划、应用、验收

```bash
cd "${WORK_DIR}"
./boot scan -i ./bootstrap/demo-env/inventory.yaml -t 20s
./boot plan -i ./bootstrap/demo-env/inventory.yaml -p ./bootstrap/demo-env/profile.yaml -t 20s
./boot apply -i ./bootstrap/demo-env/inventory.yaml -p ./bootstrap/demo-env/profile.yaml -t 20s
./boot verify -i ./bootstrap/demo-env/inventory.yaml -p ./bootstrap/demo-env/profile.yaml -t 20s
```

### 4.4 导出旧脚本兼容环境

LVM 和 NFS Server 脚本仍然消费 `ops-environment.sh`，所以这里要额外导出一次：

```bash
cd "${WORK_DIR}"
./boot export-ops-env \
  -i ./bootstrap/demo-env/inventory.yaml \
  -o ./bootstrap/demo-env/ops-environment.sh

cp ./bootstrap/demo-env/ops-environment.sh /etc/profile.d/ops-environment.sh
source /etc/profile.d/ops-environment.sh
```

### 4.5 验收标准

```bash
test -f /etc/profile.d/ops-environment.sh
grep -E 'NODE_IPS|NODE_NAMES' /etc/profile.d/ops-environment.sh
```

## 5. 第 2 步：LVM 初始化并挂载 `/data`

默认 LVM 值如下：

- VG: `ops_vg_data`
- LV: `ops_lv_data`
- FS: `xfs`
- Mount: `/data`
- Disk: `/dev/vdb`

推荐命令：

```bash
chmod +x "${LVM_SCRIPT}"
"${LVM_SCRIPT}" -y \
  --vg-name ops_vg_data \
  --lv-name ops_lv_data \
  --mount-point /data \
  --fs-type xfs \
  --disks /dev/vdb
```

执行时按下面输入：

1. 节点选择输入 `a`
2. 操作选择输入 `1`
3. 其余回车，使用默认 VG / LV / FS / 挂载点

### 5.1 验收标准

```bash
lsblk
df -h /data
grep '/data' /etc/fstab
```

### 5.2 运维说明

- 后续扩容仍使用同一个脚本，选择 `2`
- 这一阶段只是主机级数据盘准备，不直接创建 Kubernetes PVC

## 6. 第 3 步：在 `master-01` 安装 NFS Server

这一步只在第一台 master 上执行。默认导出目录是 `/data/nfs-share`。

```bash
chmod +x "${NFS_SERVER_SCRIPT}"
"${NFS_SERVER_SCRIPT}" -y
```

如果要显式指定目录：

```bash
"${NFS_SERVER_SCRIPT}" -d /data/nfs-share -y
```

### 6.1 验收标准

```bash
systemctl status nfs-server --no-pager
exportfs -v
showmount -e localhost
test -d /data/nfs-share
cat /etc/exports
```

### 6.2 安装信息

| 项目 | 内容 |
| --- | --- |
| 所在节点 | `master-01` |
| 导出目录 | `/data/nfs-share` |
| 用途 | 作为所有 `nfs` StorageClass PVC 的后端目录 |

## 7. 第 4 步：安装 Kubernetes

这里直接使用 `archinfra/apps_kubernetes` 产出的 `full` 包。

建议流程是：

1. `show-defaults`
2. `precheck`
3. `install`

```bash
chmod +x "${K8S_RUN}"
"${K8S_RUN}" show-defaults

"${K8S_RUN}" precheck \
  --masters "${MASTER_IPS}" \
  --nodes "${WORKER_IPS}" \
  --user "${SSH_USER}" \
  --pk "${SSH_PRIVATE_KEY}" \
  --yes

"${K8S_RUN}" install \
  --masters "${MASTER_IPS}" \
  --nodes "${WORKER_IPS}" \
  --user "${SSH_USER}" \
  --pk "${SSH_PRIVATE_KEY}" \
  --yes
```

如果现场使用的是密码认证，也可以改成：

```bash
"${K8S_RUN}" install \
  --masters "${MASTER_IPS}" \
  --nodes "${WORKER_IPS}" \
  --passwd 'your-password' \
  --yes
```

### 7.1 验收标准

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get sc
```

### 7.2 安装信息

| 项目 | 内容 |
| --- | --- |
| 安装器 | `k8s-sealos-linux-${PKG_ARCH}-full.run` |
| 核心参数 | `--masters`、`--nodes`、`--user`/`--passwd` |
| 备注 | 推荐优先使用私钥认证 |

## 8. 第 5 步：安装 NFS StorageClass

这里使用 `archinfra/apps_nfs-provisioner`。

默认会创建：

- release: `nfs-subdir-external-provisioner`
- namespace: `kube-system`
- storageClass: `nfs`
- 默认导出路径: `/data/nfs-share`

安装命令：

```bash
chmod +x "${NFS_SC_RUN}"
"${NFS_SC_RUN}" install \
  --nfs-server "${NFS_SERVER_IP}" \
  --nfs-path /data/nfs-share \
  -y
```

### 8.1 验收标准

```bash
kubectl get sc
kubectl get pods -n kube-system | grep -i nfs
```

至少要看到：

- `StorageClass/nfs`
- `nfs-subdir-external-provisioner` 处于 `Running`

### 8.2 推荐的简单 PVC 验证

```bash
cat <<'YAML' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-smoke-pvc
  namespace: default
spec:
  storageClassName: nfs
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
YAML

kubectl get pvc nfs-smoke-pvc
```

### 8.3 安装信息

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `kube-system` |
| 安装器 | `nfs-provisioner-installer-${PKG_ARCH}.run` |
| NFS 服务端 | `${MASTER1_IP}` |
| NFS 目录 | `/data/nfs-share` |
| StorageClass | `nfs` |

## 9. 第 6 步：安装 metrics-server

建议在 kubelet 自签 TLS 环境直接启用 `--kubelet-insecure-tls`。

```bash
"${METRICS_SERVER_RUN}" install --kubelet-insecure-tls -y
```

### 安装信息

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `kube-system` |
| 安装器 | `metrics-server-installer-${PKG_ARCH}.run` |
| 依赖 | Kubernetes 已安装 |
| 内网访问 | 聚合 API，不直接对外提供业务地址 |
| 外网访问 | 不建议暴露 |
| 默认资源 | `100m CPU / 200Mi` request |

### 验收标准

```bash
"${METRICS_SERVER_RUN}" status
kubectl top nodes
kubectl top pods -A
```

## 10. 第 7 步：安装 Prometheus Stack

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
| 安装器 | `prometheus-stack-installer-${PKG_ARCH}.run` |
| 依赖 | Kubernetes、NFS StorageClass |
| 监控发现规则 | 只抓带 `monitoring.archinfra.io/stack=default` 的资源 |
| Grafana 默认密码 | `admin@passw0rd` |
| 默认持久化 | Prometheus `200Gi`，Grafana `10Gi`，Alertmanager `10Gi` |
| 外网访问 | 默认不暴露，建议使用 ingress / NodePort / port-forward |

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

## 11. 第 8 步：安装 MySQL

```bash
"${MYSQL_RUN}" install --resource-profile mid -y
```

### 安装信息

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `aict` |
| 安装器 | `mysql-installer-${PKG_ARCH}.run` |
| 内网访问 | `mysql.aict.svc.cluster.local:3306` |
| 主节点直连 | `mysql-0.mysql.aict.svc.cluster.local:3306` |
| 外网访问 | `<任意节点IP>:30306` |
| 默认账号 | `root / passw0rd` |
| 其他默认账号 | `repl / repl@passw0rd`，`orch / orch@passw0rd`，`mysqld_exporter / exporter@passw0rd` |
| 默认监控 | exporter + ServiceMonitor 已开启 |
| `mid` 资源 | request `600m / 1152Mi`，limit `1200m / 2304Mi` |
| 默认存储 | `10Gi` |

### 验收标准

```bash
"${MYSQL_RUN}" status
kubectl get pods,svc,pvc -n aict
kubectl logs -n aict mysql-0 -c mysql --tail=50
```

## 12. 第 9 步：安装 Redis Cluster

```bash
"${REDIS_RUN}" install --resource-profile mid -y
```

### 安装信息

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `aict` |
| 安装器 | `redis-cluster-installer-${PKG_ARCH}.run` |
| 内网访问 | `redis-cluster.aict.svc.cluster.local:6379` |
| Headless | `redis-cluster-headless.aict.svc.cluster.local` |
| 外网访问 | 默认不暴露 |
| 默认密码 | `Redis@Passw0rd` |
| 默认监控 | exporter + ServiceMonitor 默认开启 |
| `mid` 资源 | 总 request 约 `3.6 CPU / 6.75Gi`，总 limit 约 `7.2 CPU / 13.5Gi` |
| 默认存储 | 总计约 `60Gi` |

### 验收标准

```bash
"${REDIS_RUN}" status
kubectl get pods,svc,pvc -n aict
```

## 13. 第 10 步：安装 Nacos

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
| 安装器 | `nacos-installer-${PKG_ARCH}.run` |
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
curl -sf "http://127.0.0.1:30094/nacos/" || true
```

## 14. 第 11 步：安装 MinIO Cluster

```bash
"${MINIO_RUN}" install --resource-profile mid -y
```

### 安装信息

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `aict` |
| 安装器 | `minio-cluster-installer-${PKG_ARCH}.run` |
| 内网 API | `minio.aict.svc.cluster.local:9000` |
| 内网 Console | `minio-console.aict.svc.cluster.local:9090` |
| 外网 API | `<任意节点IP>:30093` |
| 外网 Console | `<任意节点IP>:30092` |
| 默认账密 | `minioadmin / minioadmin@123` |
| 默认监控 | metrics + ServiceMonitor 默认开启 |
| `mid` 资源 | 稳态总 request 约 `2.1 CPU / 4.25Gi` |
| 默认存储 | `4 x 500Gi = 2Ti` |

### 验收标准

```bash
"${MINIO_RUN}" status
kubectl get pods,svc,pvc -n aict
```

## 15. 第 12 步：安装 RabbitMQ Cluster

```bash
"${RABBITMQ_RUN}" install --resource-profile mid -y
```

### 安装信息

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `aict` |
| 安装器 | `rabbitmq-cluster-installer-${PKG_ARCH}.run` |
| 内网 AMQP | `rabbitmq-cluster.aict.svc.cluster.local:5672` |
| 内网管理台 | `rabbitmq-cluster.aict.svc.cluster.local:15672` |
| 外网访问 | 默认不暴露，NodePort 常用 `30672 / 31672` |
| 默认账号 | `admin / RabbitMQ@Passw0rd` |
| Erlang Cookie | `ArchInfraRabbitMQCookie2026` |
| 默认监控 | metrics + ServiceMonitor 默认开启 |
| `mid` 资源 | 总 request 约 `1.5 CPU / 3Gi`，总 limit 约 `3 CPU / 6Gi` |
| 默认存储 | `3 x 8Gi = 24Gi` |

### 验收标准

```bash
"${RABBITMQ_RUN}" status
kubectl get pods,svc,pvc -n aict
```

## 16. 第 13 步：安装 MongoDB ReplicaSet

```bash
"${MONGODB_RUN}" install --resource-profile mid -y
```

### 安装信息

| 项目 | 内容 |
| --- | --- |
| 命名空间 | `aict` |
| 安装器 | `mongodb-cluster-installer-${PKG_ARCH}.run` |
| 内网访问 | `mongodb-cluster-0.mongodb-cluster-headless.aict.svc.cluster.local:27017` 等 3 节点 |
| 推荐连接串 | `mongodb://root:MongoDB%40Passw0rd@mongodb-cluster-0.mongodb-cluster-headless.aict.svc.cluster.local:27017,mongodb-cluster-1.mongodb-cluster-headless.aict.svc.cluster.local:27017,mongodb-cluster-2.mongodb-cluster-headless.aict.svc.cluster.local:27017/admin?replicaSet=rs0&authSource=admin` |
| 外网访问 | 默认不暴露 |
| 默认账号 | `root / MongoDB@Passw0rd` |
| 副本集密钥 | `ArchInfraMongoReplicaSetKey2026` |
| 默认监控 | metrics + ServiceMonitor 默认开启 |
| `mid` 资源 | 总 request 约 `1.8 CPU / 3456Mi`，总 limit 约 `3.6 CPU / 6912Mi` |
| 默认存储 | `3 x 20Gi = 60Gi` |

### 验收标准

```bash
"${MONGODB_RUN}" status
kubectl get pods,svc,pvc -n aict
```

## 17. 第 14 步：安装 Milvus Cluster

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
| 安装器 | `milvus-cluster-installer-${PKG_ARCH}.run` |
| 内网访问 | `milvus-cluster.milvus-system.svc.cluster.local:19530` |
| 外网访问 | 默认不暴露 |
| 默认模式 | `cluster` |
| 默认消息队列 | `woodpecker` |
| 默认监控 | Milvus ServiceMonitor + embedded etcd PodMonitor + embedded MinIO ServiceMonitor |
| `mid` 资源 | 默认 cluster 模式总 request 约 `4 CPU / 13Gi` |
| 默认存储 | 默认约 `460Gi` |

### 验收标准

```bash
"${MILVUS_RUN}" status
kubectl get pods,svc,pvc -n milvus-system
kubectl get servicemonitor,podmonitor -n milvus-system
```

## 18. 第 15 步：安装 HM 业务应用

HM 默认依赖以下组件都已 ready：

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
| 安装器 | `hm-installer-${PKG_ARCH}.run` |
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

## 19. 推荐安装顺序总表

| 顺序 | 组件 | 必须依赖 |
| --- | --- | --- |
| 1 | `boot` / bootstrap host init | 无 |
| 2 | LVM `/data` | inventory / profile 已完成 |
| 3 | NFS Server | `/data` 已挂载 |
| 4 | `apps_kubernetes` | 主机初始化已完成 |
| 5 | `apps_nfs-provisioner` / NFS StorageClass | Kubernetes + NFS Server |
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

## 20. 组件互相调用关系

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

## 21. 安装后必须记录的信息

每装完一个组件，都建议补一张信息卡：

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
status 结果:
备注:
```

## 22. 给 AI / 自动化系统的执行规约

1. 先检测架构，再选 `amd64` 或 `arm64` 安装包。
2. 先列出 `/opt/release`，确认实际产物都在，再执行。
3. `boot` 只负责 host init、inventory、profile、ops-environment，不负责安装 Kubernetes。
4. 每装完一个组件都必须执行：
   - `<installer> status`
   - `kubectl get pods,svc,pvc -n <namespace>`
   - 记录内部地址、外部地址、账密、依赖
5. 安装 Nacos 之前必须确认 MySQL 已 ready。
6. 安装 HM 之前必须确认 MySQL、Redis、MinIO、MongoDB、RabbitMQ、Milvus 已 ready。
7. 看到 `StorageClass` 不存在、PVC `Pending`、Pod `CrashLoopBackOff` 时必须先排障，不能跳过。
8. 生产环境部署完成后要尽快轮换默认密码。

## 23. 常用运维命令

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

## 24. 生产前必须再次确认

- `master-01:/data` 容量足够承载所有 NFS 数据
- 默认密码已经按环境要求修改
- Prometheus 已经发现各组件监控对象
- 业务入口已经通过 NodePort / LB / Ingress 正确暴露
- [RESOURCE-BASELINE.zh-CN.md](./RESOURCE-BASELINE.zh-CN.md) 中的资源与存储估算与你的节点规模匹配
