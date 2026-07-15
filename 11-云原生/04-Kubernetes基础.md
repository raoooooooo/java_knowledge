# Kubernetes 基础

## 一、核心概念

### 1.1 什么是 Kubernetes

**一句话定义**：Kubernetes（简称 K8s）是一个开源的**容器编排系统**，负责自动部署、扩展和管理大量容器化应用。

> **名字由来**：K8s = K + 8 个字母（ubernete）+ s，即 Kubernetes 的缩写。

**为什么需要 K8s？**（Docker 解决了"单机跑容器"，K8s 解决"多机管容器"）

| 痛点 | Docker 单机 | K8s 集群 |
|------|------------|---------|
| 容器挂了 | 不会自动重启/重建 | 自动监测并拉起新副本 |
| 流量增大 | 手动扩容 | 一行配置水平扩展副本数 |
| 滚动升级 | 手动停旧起新 | 自动滚动更新、支持回滚 |
| 跨机调度 | 不关心放哪台机器 | 按资源/亲和性智能调度 |
| 服务发现 | 容器 IP 随机变 | 固定 Service IP + DNS 名访问 |

**K8s 和 Docker 的关系**：
- Docker 负责"把应用打包成镜像、跑成容器"（构建与运行）
- K8s 负责"管理成百上千个容器组成的集群"（编排与调度）
- ⚠️ **重要更正**：早期 K8s 内置 `dockershim` 直接对接 Docker，但**自 v1.24（2022 年 5 月）起已移除 dockershim**。如今 K8s 通过 **CRI（容器运行时接口）** 标准对接容器运行时，主流是 **Containerd、CRI-O**，Docker 本身不再是受支持的运行时（但 Docker 构建出的镜像仍可被 Containerd 运行）。

### 1.2 K8s 集群架构

K8s 集群由**控制平面（Master）**和**工作节点（Worker Node）**两部分组成。

```
        ┌──────────────── 控制平面 (Master) ─────────────────┐
        │   kube-apiserver  ◄──►  etcd（集群状态存储）        │
        │   kube-scheduler          kube-controller-manager │
        └───────────────────────────┬───────────────────────┘
                                    │ （都是通过 apiserver 通信）
        ┌───────────────────────────┴───────────────────────┐
   ┌────┴──────────────────┐                    ┌────────────┴──────────┐
   │     Worker Node 1     │      ...           │     Worker Node N     │
   │  kubelet / kube-proxy │                    │  kubelet / kube-proxy │
   │  容器运行时(Containerd)│                    │  容器运行时(Containerd)│
   │    Pod   Pod   Pod     │                    │    Pod   Pod   Pod    │
   └───────────────────────┘                    └───────────────────────┘
```

#### 控制平面组件（大脑）

| 组件 | 职责 | 通俗理解 |
|------|------|---------|
| **kube-apiserver** | 所有操作的唯一入口，唯一能直接读写 etcd 的组件，负责认证/鉴权 | 集群"前台"，所有请求都经它 |
| **etcd** | 高可用键值数据库，持久化存储整个集群的所有状态数据 | 集群的"数据库/记忆" |
| **kube-scheduler** | 监听新创建但未分配节点的 Pod，根据资源/亲和性/约束选一个最合适的节点 | 集群"调度员" |
| **kube-controller-manager** | 运行各种控制器（副本控制器、节点控制器、端点控制器等），不断让"实际状态"趋近"期望状态" | 集群"大管家" |

#### 工作节点组件（手脚）

| 组件 | 职责 | 通俗理解 |
|------|------|---------|
| **kubelet** | 节点上的代理，接收 apiserver 下发的 Pod 规格，确保容器按规约运行 | 节点"包工头" |
| **kube-proxy** | 维护节点上的网络规则，实现 Service 的负载均衡与转发（iptables/IPVS） | 节点"网络路由器" |
| **容器运行时** | 真正跑容器（Containerd、CRI-O） | 干活的"引擎" |

> **记忆要点**：所有组件都只和 apiserver 通信，apiserver 只和 etcd 通信——这是理解 K8s 的关键架构线索。

### 1.3 声明式 API 与控制循环（核心设计思想）

K8s 区别于传统运维的根本范式：**声明式**而非命令式。

- **命令式**：告诉系统"去做什么"（启动 3 个 Pod、删除 1 个 Pod）——你得自己盯着。
- **声明式**：告诉系统"我期望的状态是什么"（期望 3 个副本），系统**持续**努力让"实际状态 = 期望状态"。

这个持续对账的过程叫**控制循环（Reconcile Loop）**：控制器不断 `观察当前状态 → 对比期望状态 → 执行动作缩小差距`。

> 通俗类比：你给恒温器设定 25℃（期望状态），它自己决定开制冷还是制热（控制循环），你不用每次都下命令。Pod 副本数、Deployment 版本、Service 端点都是这么自动维持的。

### 1.4 核心资源对象

#### Node -- 工作节点（Pod 的载体）

- **Node 是 K8s 集群中的一台工作机器**（可以是物理机或虚拟机），Pod 最终都运行在某个 Node 上。
- 每个 Node 运行 **kubelet**（与控制平面通信、管理 Pod 生命周期）和**容器运行时**（真正跑容器）。
- Node 分两类：**Master 节点**（跑控制平面组件，一般不跑业务 Pod）和 **Worker 节点**（专门跑业务 Pod）。
- **Node 状态**：调度器据此判断能否往该节点放 Pod，常见如 `Ready`（就绪）、`MemoryPressure`（内存紧张）、`DiskPressure`（磁盘紧张）。
- **调度影响机制**：通过**污点（Taint）**让节点排斥 Pod、**节点亲和性（Node Affinity）**让 Pod 偏好或限定某些节点，从而决定 Pod 落在哪台 Node。

> 通俗理解：Node 是"工位"，Pod 是"派到工位上的员工"，调度器（scheduler）负责把 Pod 派到合适的工位上。

#### Pod —— 最小调度单元

- **Pod 是 K8s 中最小的可部署计算单元**，容器不能脱离 Pod 单独调度。
- 一个 Pod 可含一个或多个容器，它们**共享网络命名空间**（同一 IP、同一端口空间）和**存储卷**，但**进程/文件系统隔离**。
- 一个 Pod 的所有容器必然运行在**同一个节点**上。
- **Pod 生命周期短暂**：Pod IP 随销毁而变，不保证长期存活。

**为什么需要 Pod？（容器单一职责 vs 紧耦合）**
- 原则上"一个容器一个主进程"（单一职责，便于故障隔离）。
- 但有些场景需要多个进程紧密协作（如主容器 + 日志收集 sidecar），它们需要共享网络/存储。
- Pod 就是把这几个容器"打包"成一个逻辑主机统一调度，既保持容器级隔离，又能共享资源。

#### Label 与 Label Selector

- **Label**：附加在资源上的任意 `key=value` 标签（如 `app=order`、`tier=backend`）。
- **Selector**：通过标签筛选资源（如 `app=order`）。
- 作用：实现资源间**松耦合关联**——Service 选哪些 Pod、Deployment 管哪些 Pod，都靠 Label，而非写死 IP/名字。

#### ReplicaSet —— 副本控制器

- 声明式地维护**期望数量的 Pod 副本**始终运行。
- 某个 Pod 挂了 → 自动新建补齐；多了 → 自动删除。
- **期望式**：只说"要几个"，不说"加几个还是减几个"。

#### Deployment —— 应用发布与升级（最常用）

- **Deployment 管理 ReplicaSet，ReplicaSet 管理 Pod**（层层委托）。
- 核心价值：**滚动更新**与**版本回滚**。
- 只要 Pod 模板（如镜像版本）变更即触发升级；旧 ReplicaSet 不会删除，便于回滚。

**两种升级策略**：
| 策略 | 行为 | 是否中断 |
|------|------|---------|
| RollingUpdate（默认） | 逐步起新 Pod、停旧 Pod，新旧版本短暂共存 | 不中断 |
| Recreate | 先全部停旧 Pod，再一次性起新 Pod | 中断 |

**控制滚动速率的两个参数**（高频考点）：
- `maxUnavailable`：升级过程中允许"不可用 Pod 数"上限，**向下取整**。
- `maxSurge`：升级过程中允许"超出期望副本数"的上限，**向上取整**。
- 例：replicas=3，maxSurge=30%（→1），maxUnavailable=15%（→0）。则升级中**最多 4 个 Pod、最少保持 3 个可用**。

#### Service —— 服务暴露与发现

Pod IP 会变，客户端不能直连 Pod。**Service 提供一个固定的访问入口（稳定 IP + DNS 名），背后负载均衡到一组 Pod**。

| 类型 | 说明 | 场景 |
|------|------|------|
| **ClusterIP**（默认） | 集群内部虚拟 IP | 集群内 Pod 互访 |
| **NodePort** | 在每个节点开放同一静态端口 | 简单对外暴露（端口范围 30000-32767） |
| **LoadBalancer** | 借助云厂商负载均衡器对外 | 云上生产对外服务 |
| **Headless**（clusterIP: None） | 不分配 VIP，DNS 直接返回所有 Pod IP | 客户端直连特定 Pod（如 StatefulSet） |

> 集群内 Pod 可直接用 **Service 的 DNS 名**（如 `test-svc.default.svc.cluster.local`）访问，无需记 IP。

#### Namespace —— 逻辑隔离

- 用于在**一个物理集群**内划分多个虚拟集群，实现资源/权限的逻辑隔离（如 dev/test/prod 环境）。

---

> 补充：Pod、副本、Service 的 IP 与端口如何关联？

**三类 IP 的角色**：Pod IP（单个 Pod 网卡 IP，Pod 销毁即变，不稳定）；ClusterIP（Service 虚拟 IP，Service 存活期内固定）；NodeIP（工作节点的物理 IP）。核心矛盾是 Pod IP 会变而客户端要稳定入口，Service 就是来"钉住"不变 IP 的。

**两类 Service 流量路径**（关键：内部访问和对外暴露路径不同）：

**① ClusterIP 类型（默认，集群内访问）** -- 最常用，不经过 nodePort：
```
集群内 Pod 访问 ClusterIP:port
        │
        ▼ kube-proxy 转发规则（负载均衡）
   PodIP:targetPort
```
就两步：访问 Service 的 ClusterIP -> kube-proxy 负载均衡落到某个 PodIP。

**② NodePort / LoadBalancer 类型（对外暴露）** -- 多一个"节点入口"前置：
```
外部访问 NodeIP:nodePort          （或 LB 公网 IP）
        │
        ▼ kube-proxy 转发到 ClusterIP:port
        ▼ 负载均衡落到 PodIP:targetPort
```

> ⚠️ 注意：`NodeIP:nodePort` 只是**对外暴露时的节点入口**，集群内部访问走的是 `ClusterIP:port`，**不经过 nodePort**。原"三段链路"把 nodePort 写成必经环节是错的。另外"负载均衡到 Pod"和"落 PodIP"其实是 kube-proxy 一次转发的两面，不是独立两步。

**端口对应关系**：

| 字段 | 含义 | 归属 |
|------|------|------|
| `nodePort` | 节点对外开放的端口 | 节点开的口（仅 NodePort/LoadBalancer 才有） |
| `port` | Service 暴露端口 | Service 的虚拟口，集群内 `curl clusterIP:port` 用 |
| `targetPort` | 落到 Pod 容器的端口 | 必须等于容器 `listen` 的端口 |
| `containerPort` | Pod 模板里声明的容器端口 | 声明用，与 targetPort 对应 |

```yaml
spec:
  type: NodePort
  ports:
    - port: 80          # Service 对外门面端口
      targetPort: 8080  # Pod 容器真实监听端口
      nodePort: 30036   # 节点对外开放端口
  selector: {app: order}  # 按 label 把 3 个副本圈为一组
```

**副本在哪里体现**：`replicas: 3` = 3 个独立 PodIP，分散在不同节点。Service 用 selector 把它们圈为一组，背后负载均衡；任一 PodIP 挂了被自动从后端剔除，新 Pod 起来再加入。**客户端只认 ClusterIP / Service 名，完全感知不到背后 PodIP 的增减**，这就是 Service 抽象的价值。

---
- 资源名在**同一 Namespace 内唯一**，跨 Namespace 可重名。

#### 其他高频资源简介

| 资源 | 作用 | 典型场景 |
|------|------|---------|
| **ConfigMap / Secret** | 注入配置 / 敏感信息 | 应用配置、密码证书 |
| **StatefulSet** | 有状态应用，Pod 有稳定身份（pod-0,1,2）和独立存储 | 数据库、消息队列 |
| **DaemonSet** | 每个节点跑一个副本 | 日志采集、监控 Agent、网络插件 |
| **Job / CronJob** | 跑完即退 / 定时任务 | 数据迁移、定时备份 |
| **Ingress** | 七层（HTTP/HTTPS）路由入口，一个 LB 对外、按域名/路径分发到多 Service | 对外 Web 流量入口 |
| **PV / PVC / StorageClass** | 持久化存储抽象 | 数据库持久卷 |

### 1.5 DaemonSet 详解（日志采集与监控实战）

> 术语备注：下文"节点"即前面 1.4 讲的 **Node**（工作机器），下文同。

#### 是什么
DaemonSet 确保集群里**每个（符合条件的）节点上都恰好运行一个 Pod 副本**。
> 类比：每个工位派一个固定"值班员"，不管工位多少个、谁来了都得有一个。

#### 怎么跑的（核心机制）
1. **每节点一个**：不看 replicas，节点数 = Pod 数。新加 5 个节点就自动多 5 个 Pod。
2. **自动跟随节点**：
   - 新节点加入集群 -> 自动在其上创建 Pod
   - 节点被移除/宕机 -> 该节点 Pod 自动回收（不会在别处重建，因为它绑定节点）
3. **调度方式**：由 kube-scheduler 调度（1.12+），自动给 Pod 加节点亲和性绑定到目标节点；**默认跳过 master 节点**（master 有 `NoSchedule` 污点），除非显式配 toleration。
4. **不看节点剩余资源**：即便节点资源紧张也要塞一个（目标是"全覆盖"不是"合理分配"），所以这类 Pod 要尽量轻量。

#### 和 Deployment 区别

| 维度 | DaemonSet | Deployment |
|------|-----------|-----------|
| 副本数 | = 节点数（自动） | 手动指定 replicas |
| 分布 | 每节点恰好 1 个 | 可能多个挤一个节点 |
| 跟随节点 | 新节点自动起 Pod | 不会 |
| 看资源 | 不看（硬塞） | 看（按资源调度） |
| 典型用途 | 节点级 Agent | 业务应用 |

#### 典型用途（"每个节点都要有"的东西）
- 网络插件（Calico / Flannel）
- 存储插件（CSI Provisioner）
- **日志采集**（Filebeat / Fluent Bit / Fluentd）
- **监控 Agent**（node-exporter / Datadog Agent）
- 安全 Agent（Falco）

#### 日志采集怎么实现（重点）

**先搞清：容器的日志去哪了？**
- 容器的 `stdout/stderr` 被容器运行时（containerd）捕获，写到**节点磁盘**：
  - `/var/log/pods/<命名空间>_<pod名>_<uid>/<容器名>/<重启序号>.log`
  - `/var/log/containers/*.log` 是指向上面的软链接
- 所以**节点上只要读 `/var/log` 就能拿到本节点所有容器的日志**，不用逐个进 Pod。

**方案：节点级 DaemonSet（K8s 官方推荐）**
- 每个节点跑一个 Filebeat / Fluent Bit
- 挂载节点的 `/var/log` 和 `/var/lib/containerd` 目录
- 读所有容器日志文件 -> 附上 Pod 元数据（哪个 namespace/pod/container）-> 发到 ES / Loki / Kafka

```yaml
spec:
  template:
    spec:
      containers:
        - name: filebeat
          image: filebeat:7
          volumeMounts:
            - {name: varlog, mountPath: /var/log, readOnly: true}
            - {name: containerd-logs, mountPath: /var/lib/containerd, readOnly: true}
      volumes:
        - name: varlog
          hostPath: {path: /var/log}            # 容器日志软链接
        - name: containerd-logs
          hostPath: {path: /var/lib/containerd} # 运行时原始日志
```

**三种日志方案对比**：

| 方案 | 资源开销 | 适用场景 |
|------|---------|---------|
| 节点级 DaemonSet | 低（每节点一个） | 大多数场景，**推荐** |
| Sidecar 容器 | 高（每 Pod 一个） | 日志需特殊处理、应用无法 stdout |
| 应用直推 | 无 agent | 自研应用直接发日志系统 |

#### 监控怎么实现（重点）

**节点级监控：node-exporter DaemonSet**
- 每个节点跑一个 `node-exporter`，暴露 `/metrics`（端口 9100）
- 采集**机器级指标**：CPU、内存、磁盘、网络、文件系统使用率
- Prometheus 通过 K8s 服务发现自动发现所有 node-exporter Pod 并抓取

**分层监控（理清各层用什么）**：

| 监控层级 | 采集器 | 部署方式 | 指标 |
|---------|--------|---------|------|
| 节点级 | node-exporter | **DaemonSet** | CPU/内存/磁盘/网络 |
| 容器级 | cAdvisor | kubelet 内置 | 容器 CPU/内存 |
| 集群对象级 | kube-state-metrics | Deployment | Pod/Deploy 状态数 |
| 指标聚合 | metrics-server | Deployment | HPA 弹性伸缩用 |

> 监控里 DaemonSet 专门负责"节点级"这一层，和日志采集一个套路：每节点一个 Agent，采集本节点数据上报。

#### 更新策略

| 策略 | 行为 |
|------|------|
| RollingUpdate（默认） | 滚动更新，`maxUnavailable` 控制并发 |
| OnDelete | 手动删除 Pod 才触发更新 |

### 1.6 节点亲和性（调度机制）

#### 是什么 + 通俗类比

Pod 由哪个 Node 来跑，是 **scheduler（调度器）** 决定的。节点亲和性就是让 Pod 表达"我想被调度到什么样的节点"，依据是**节点上的标签（labels）**。

> 类比调度器选节点像"相亲"分两步：**先初筛（Filter，硬性条件不满足直接淘汰）→ 再评分（Score，符合偏好的加分）**。节点亲和性就是 Pod 提的择偶要求。

#### 两类亲和性

| 类型 | 字段 | 行为 | 通俗理解 |
|------|------|------|---------|
| **硬亲和性（required）** | `requiredDuringSchedulingIgnoredDuringExecution` | 必须满足，否则 Pod 一直 `Pending` | 硬性要求，不满足宁可不调度 |
| **软亲和性（preferred）** | `preferredDuringSchedulingIgnoredDuringExecution` | 尽量满足，不满足也能调度到别的节点 | 偏好，能凑合 |

#### 实现原理（核心）

调度器选节点分两步走，两类亲和性分别作用在**不同阶段**：

```
Pod 待调度
   │
   ▼
1️⃣ Filter 过滤阶段  ← 硬亲和性（required）在这里生效
   │  遍历所有节点，标签不满足 matchExpressions 的直接淘汰
   ▼
2️⃣ Score 打分阶段  ← 软亲和性（preferred）在这里生效
   │  对幸存节点按 weight 加分（1-100），总分越高越优先
   ▼
选出得分最高的节点 → 由 kubelet 起容器
```

- **硬亲和性 → Filter 阶段**：是"过滤条件"，不满足的节点根本不进打分池。
- **软亲和性 → Score 阶段**：是"加分项"，每个 preference 带 `weight`（1~100），满足就加分、不满足不扣分。

#### 名字为什么这么长？（面试常问）

`requiredDuringSchedulingIgnoredDuringExecution` 拆开看：
- `DuringScheduling`：**调度阶段**生效（决定新 Pod 放哪）。
- `IgnoredDuringExecution`：**运行阶段忽略**——Pod 已经跑起来后，哪怕节点标签变了不再满足，也**不会**把 Pod 驱逐重新调度。

> 即"只管进门，不管后续变心"。这也是它和某些动态机制的区别。

#### 匹配操作符

| 操作符 | 含义 |
|------|------|
| `In` / `NotIn` | 标签值在 / 不在给定集合 |
| `Exists` / `DoesNotExist` | 该标签存在 / 不存在（不看值） |
| `Gt` / `Lt` | 数值大于 / 小于（如按 CPU 核数、内存大小筛） |

#### 逻辑关系（容易踩坑）

- 同一个 `nodeSelectorTerms` 内多个 `matchExpressions` 是 **AND（且）**。
- 多个 `nodeSelectorTerms` 之间是 **OR（或）**。

```yaml
nodeSelectorTerms:        # 多个 term 之间：OR
  - matchExpressions:     # 同一 term 内多个条件：AND
      - {key: disktype, operator: In, values: [ssd]}
      - {key: zone, operator: In, values: [east]}
```
上面含义：节点要 `disktype=ssd` **且** `zone=east` 才满足。

#### 完整 YAML 示例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      # 硬亲和性：必须调度到带 disktype=ssd 的节点
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values: ["ssd"]
      # 软亲和性：尽量调度到 east 机房（权重 100）
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          preference:
            matchExpressions:
              - key: zone
                operator: In
                values: ["east"]
  containers:
    - name: app
      image: nginx
```

#### 与相关机制对比（重点区分）

| 机制 | 视角 | 依据 | 一句话 |
|------|------|------|------|
| **nodeAffinity** | Pod 选 Node | 节点标签 | "我想去什么样的节点" |
| **nodeSelector** | Pod 选 Node | 节点标签 | 简陋版，只能精确匹配、无软约束，已被 nodeAffinity 取代 |
| **Taint / Toleration** | Node 排斥 Pod | 节点污点 | "我这台节点不欢迎 Pod"（除非 Pod 有 Toleration） |
| **Pod Affinity / Anti** | Pod 选 Pod | 已运行 Pod 的标签 | "我想和某些 Pod 放一起/不一起"（按 Pod 位置而非节点标签） |

> **记忆**：nodeAffinity 是"Pod 看节点标签挑节点"，Taint 是"节点反过来挑 Pod"，两者常配合使用。

### 1.7 资源管理：requests / limits / QoS

**一句话本质**：`requests` 管**调度**（能不能放这节点），`limits` 管**运行时**（最多用多少），底层都靠 Linux **Cgroups** 实现。

```yaml
resources:
  requests:
    cpu: 500m       # "我至少要 0.5 核，调度时给我留好"
    memory: 512Mi
  limits:
    cpu: 1000m      # "我最多用 1 核，超了就限流"
    memory: 1Gi     # "内存超 1Gi 就杀我"
```

#### 两个阶段各管一头

```
Pod 提交
  │
  ▼  调度阶段 ── requests 生效
  │   scheduler 看"节点上所有 Pod 的 requests 之和"是否 ≤ 节点可分配容量
  │   不够 -> 换节点 / Pending；够 -> 选这个节点
  ▼  运行阶段 ── limits 生效
      kubelet 给容器配 Cgroups 限制
      CPU 超 -> 限流 throttle（卡顿但不杀）
      内存超 -> OOM Kill（直接杀进程，容器重启）
```

> **关键**：requests 只是给调度器看的数字，不真正限制；limits 才是 Cgroups 写下去的硬限制。

#### CPU 限制原理（CFS quota/period）

底层是 Linux CFS（完全公平调度器）两个参数：

| Cgroups 参数 | 含义 | 默认 |
|---|---|---|
| `cpu.cfs_period_us` | 调度周期长度 | 100ms |
| `cpu.cfs_quota_us` | 周期内允许用的 CPU 时间 | 按 limit 算 |

- **limit=1000m（1核）**：每 100ms 周期可用 100ms CPU 时间
- **limit=500m（0.5核）**：每 100ms 只能跑 50ms -> "忙 50ms 闲 50ms"，占用率压在 50%
- 用完配额被**挂起限流（throttled）**，下个周期恢复

> 类比健身房限流：limit=500m 不是"只能用半台机器"，而是"每 10 秒只能练 5 秒，剩下 5 秒必须歇"。CPU 是**可压缩资源**--超限只变慢，**进程不会被杀**。应用莫名延迟飙高常因 CPU limit 太低被 throttle（静默限流，难排查）。

#### 内存限制原理（OOM Kill）

底层是 Cgroups 的 `memory.limit_in_bytes`：

- 容器内存用到 limit -> 内核 **OOM Killer** 直接 kill 进程 -> 容器退出 -> 按重启策略重启
- 内存是**不可压缩资源**--没法限流，只能杀

> 这就是内存超了应用会**重启**、CPU 超了只会**变慢**的本质区别。

#### 单位换算

| 写法 | 含义 |
|---|---|
| `1` 或 `1000m` | 1 个 CPU 核 |
| `500m` | 0.5 核（millicores 毫核） |
| `1Gi` | 1024 MiB（二进制） |
| `1G` | 1000 MB（十进制，别混用） |

> CPU 用核数/毫核，内存用 `Mi/Gi`。`m` 在 CPU 是千分之一核，在内存 `Mi` 是 2^20 字节，别搞混。

#### QoS 服务质量等级（高频考点）

requests 和 limits 怎么配，决定 Pod 的 **QoS 等级**，影响**节点资源紧张时谁先被驱逐**：

| QoS | 配置条件 | 被驱逐优先级 |
|---|---|---|
| **Guaranteed** | 所有容器 requests == limits（且都设了） | 最难被杀（最稳） |
| **Burstable** | 至少设了 requests，但不是 Guaranteed | 中等 |
| **BestEffort** | requests 和 limits 都没设 | **最先被杀** |

> 原理：kubelet 调整每个容器的 `oom_score_adj`，OOM Killer 优先杀 adj 高的。BestEffort 的 adj 最高，节点内存吃紧时最先被牺牲以保护 Guaranteed。

**生产建议**：核心服务用 Guaranteed（requests=limits），别留 BestEffort（裸奔）。

#### 常见坑

| 配置 | 后果 |
|---|---|
| 只设 limits 不设 requests | requests 默认等于 limits，调度按 limits 算，易调不上去 |
| 只设 requests 不设 limits | 无上限，可能吃光节点资源 |
| CPU limit 太低 | 应用变慢但**不报错**，难排查（throttle 静默） |
| 内存 limit 太低 | 频繁 OOM 重启 |

**记忆口诀**：requests 是"预订"（调度留座），limits 是"限流"（Cgroups 真限制）；CPU 超限只变慢（可压缩），内存超限直接杀（不可压缩）；两者相等成 Guaranteed（最难被驱逐）。

### 1.8 应用部署配置全景（运维要配哪些文件）

部署一个应用到 K8s，运维主要接触以下几类配置（应用部署类，不含集群搭建）：

| 配置类 | 典型文件/资源 | 作用 | 示例 |
|------|--------------|------|------|
| **资源清单** | `deployment.yaml`、`service.yaml` | 描述"部署什么"：镜像、副本数、端口、标签、调度 | `kubectl apply -f` 提交 |
| **配置注入** | ConfigMap / Secret | 把易变配置、敏感信息从镜像剥离，避免改配置重打镜像 | ConfigMap 存 `application.yml`；Secret 存数据库密码 |
| **权限控制** | ServiceAccount / Role / RoleBinding（RBAC） | 声明应用以什么身份运行、能操作哪些资源 | 应用要读集群信息需 SA + Role 授权 |
| **存储声明** | PVC / StorageClass | 有状态应用声明持久化存储 | 数据库要 10Gi，写 PVC 申请 |
| **流量入口** | Ingress | 七层路由，按域名/路径对外分发到 Service | `order.com -> order-svc` |

**注入方式**（ConfigMap/Secret 进容器的两种途径）：
- **环境变量**：把配置项映射为容器环境变量
- **挂载文件**：把配置当成文件挂载到容器目录（如直接挂成 `application.yml`）

**配置管理工具**：常配合 **Helm** 把这些 YAML 打包成模板，一份 `values.yaml` 参数化部署到 dev/test/prod 多环境。

> 核心记忆：**资源清单定部署形态、ConfigMap/Secret 注入配置、RBAC 管权限、PVC 管存储、Ingress 管入口**。

### 1.9 kubectl 常用命令与资源清单（简要）

**常用命令**：
```bash
kubectl get pods / svc / deploy / nodes -n <ns>   # 查看资源（-n 指定命名空间）
kubectl describe pod <name>                        # 查看详情（排查问题首选）
kubectl logs <pod> [-c <container>]                # 查看日志
kubectl exec -it <pod> -- sh                       # 进入容器
kubectl apply -f xxx.yaml                          # 声明式创建/更新
kubectl delete -f xxx.yaml                         # 删除资源
kubectl scale deploy <name> --replicas=5           # 手动扩缩容
kubectl rollout status / undo deployment <name>   # 查看升级状态 / 回滚
```

**资源清单（YAML）核心结构**（以 Deployment 为例）：
```yaml
apiVersion: apps/v1          # API 版本
kind: Deployment            # 资源类型
metadata:                   # 元数据（名字、标签、命名空间）
  name: order-service
spec:                       # 期望规格
  replicas: 3               # 期望副本数
  selector:                 # 选择器：管理哪些 Pod
    matchLabels:
      app: order
  template:                 # Pod 模板
    metadata:
      labels:
        app: order
    spec:
      containers:
        - name: order
          image: order:v1
          ports:
            - containerPort: 8080
```
> 四大要素：`apiVersion` / `kind` / `metadata` / `spec`，几乎所有资源清单都是这个骨架。

---

## 二、常见面试题

### Q1：K8s 是什么？和 Docker 是什么关系？

> K8s 是开源容器编排系统，负责在集群上自动化部署、扩缩容和管理容器化应用。Docker 负责把应用打包成镜像并运行成容器（构建+运行），K8s 负责管理成百上千个容器组成的集群（编排+调度），二者是互补关系。需注意：自 K8s v1.24 起移除了 dockershim，不再直接对接 Docker，而是通过 CRI 标准对接 Containerd、CRI-O 等运行时。

### Q2：K8s 架构由哪些组件组成？各自的作用？

> 分控制平面和工作节点。控制平面：**apiserver**（唯一入口，唯一写 etcd）、**etcd**（存集群状态）、**scheduler**（为 Pod 选节点）、**controller-manager**（跑各种控制器维持期望状态）。工作节点：**kubelet**（确保 Pod 按规约运行）、**kube-proxy**（维护 Service 网络规则）、**容器运行时**（跑容器）。所有组件都只和 apiserver 通信。

### Q3：什么是 Pod？为什么不直接调度容器？

> Pod 是 K8s 最小可部署计算单元。一个 Pod 内的容器共享网络（同 IP、同端口空间）和存储卷，但进程/文件系统隔离，且必跑在同一节点。需要 Pod 是因为遵循"一容器一主进程"的单一职责原则后，有些紧密协作的进程（如主容器 + sidecar）仍需共享网络/存储，Pod 把它们打包成一个逻辑单元统一调度，兼顾隔离与协作。

### Q4：Pod、ReplicaSet、Deployment 三者关系？

> 层层委托：**Deployment 管理 ReplicaSet，ReplicaSet 管理 Pod**。ReplicaSet 负责维持副本数；Deployment 在其上提供滚动更新和版本回滚能力——升级时新建一个 ReplicaSet，逐步把 Pod 从旧 RS 迁到新 RS，旧 RS 保留以便回滚。日常直接用 Deployment 即可，很少单独用 ReplicaSet。

### Q5：K8s 如何实现服务的自动扩缩容和滚动升级？

> 扩缩容：修改 Deployment 的 `replicas` 字段，控制器通过控制循环自动增减 Pod。滚动升级：Pod 模板变更即触发，默认 RollingUpdate 策略逐步起新停旧、新旧短暂共存不中断；用 `maxSurge`（向上取整）控制可超出的副本上限、`maxUnavailable`（向下取整）控制可不可用的副本上限来调节速率。回滚用 `kubectl rollout undo`。

### Q6：Service 有哪些类型？为什么要用 Service？

> Pod IP 随 Pod 销毁而变，客户端不能直连 Pod。Service 提供稳定 IP + DNS 名，背后负载均衡到一组 Pod。类型：**ClusterIP**（默认，集群内访问）、**NodePort**（节点开静态端口对外）、**LoadBalancer**（云 LB 对外）、**Headless**（不分配 VIP，DNS 返回所有 Pod IP，用于直连特定 Pod）。集群内可通过 Service DNS 名访问。

### Q7：Headless Service 有什么用？和普通 Service 区别？

> 普通 Service 有 ClusterIP，DNS 返回该 VIP，客户端请求被负载均衡到某个 Pod。Headless Service（`clusterIP: None`）不分配 VIP，DNS 直接返回所有后端 Pod IP，客户端可自己选连哪个 Pod。常配合 StatefulSet，让客户端按稳定身份（pod-0）直连特定实例，用于有状态应用如数据库主从。

### Q8：Pod、副本、Service 的 IP 和端口是怎么关联的？

> 涉及三类 IP：Pod IP（单个 Pod 网卡 IP，Pod 销毁即变）；ClusterIP（Service 虚拟 IP，Service 存活期内固定）；NodeIP（节点物理 IP）。**集群内访问**走 `ClusterIP:port`，kube-proxy 负载均衡落到 `PodIP:targetPort`（就两步）；**对外暴露**（NodePort/LoadBalancer）才多一个节点入口 `NodeIP:nodePort`，经 kube-proxy 转到 ClusterIP 再到 Pod。注意 nodePort 只在对外暴露时有，内部访问不经过它。其中 `port` 是 Service 门面端口、`targetPort` 是 Pod 容器真正监听的端口、`nodePort` 是节点对外开放端口。副本数 `replicas` 决定后端有几个独立 PodIP，Service 用 selector 圈定它们并负载均衡，Pod 增减客户端无感（只认 ClusterIP/Service 名）。

### Q9：StatefulSet 和 Deployment 区别？什么场景用？

> Deployment 管无状态应用，Pod 互换、名字随机、共享存储。StatefulSet 管有状态应用：Pod 有**稳定网络身份**（pod-0、pod-1，按序）、**独立持久存储**（每个 Pod 绑定自己的 PVC）、**有序**部署/扩缩/删除。典型场景：数据库、消息队列等需要稳定标识和独立存储的服务。

### Q10：Ingress 和 Service（LoadBalancer）有什么区别？

> LoadBalancer 类型的 Service 每个服务都会创建一个云厂商负载均衡器，成本高且只能四层。Ingress 是七层（HTTP/HTTPS）路由，一个 Ingress 控制器（如 Nginx Ingress）对外只暴露一个入口，按**域名/路径**把流量分发到多个 Service，更省资源、更灵活，是对外 Web 流量入口的主流方案。

### Q11：K8s 的声明式和命令式有什么区别？

> 命令式告诉系统"做什么"（启动 3 个 Pod），需要人盯结果；声明式只描述"期望状态"（3 个副本），系统通过控制循环持续让实际状态趋近期望状态。好处是幂等、可自愈——Pod 挂了自动补、配置改了自动收敛，运维只需声明意图，不用关心具体执行步骤。

### Q12：什么是节点亲和性？两类区别和实现原理是什么？

> 节点亲和性让 Pod 按**节点标签**表达"我想调度到什么样的节点"。分两类：**硬亲和性（required）**必须满足，否则 Pod 一直 Pending；**软亲和性（preferred）**尽量满足，每个偏好带 1~100 的 weight。实现上靠调度器的两阶段：硬亲和性在 **Filter 过滤阶段**淘汰不满足的节点，软亲和性在 **Score 打分阶段**给满足的节点加分。注意字段名里的 `IgnoredDuringExecution` 表示 Pod 运行后节点标签变了也不会驱逐已运行的 Pod。

### Q13：节点亲和性、Taint/Toleration、Pod 亲和性有什么区别？

> 三者都影响调度但视角不同。**nodeAffinity** 是 Pod 按节点标签挑节点（"我想去什么样的节点"）；**Taint/Toleration** 反过来，是节点设污点排斥 Pod，Pod 要有 Toleration 才能上（"节点挑 Pod"）；**Pod Affinity/Anti-Affinity** 依据的是**已运行的 Pod**而非节点标签（"我想和某些 Pod 放一起/不一起"）。日常常把 nodeAffinity 和 Taint 配合使用。

### Q14：requests 和 limits 的区别和原理是什么？

> requests 是"请求量"，调度器据此判断 Pod 能否放到某节点（节点上所有 Pod 的 requests 之和 ≤ 节点容量），它只是个数字给调度器看，不真正限制；limits 是"上限量"，kubelet 据此用 Cgroups 写下硬限制。CPU 超限被 CFS quota 限流（throttle，只变慢不杀，是可压缩资源）；内存超限触发 OOM Kill 直接杀进程（不可压缩资源）。所以内存超了应用会重启，CPU 超了只会变慢。

### Q15：什么是 QoS 等级？节点资源紧张时谁先被驱逐？

> QoS 由 requests/limits 的配置决定，分三级。**Guaranteed**：所有容器 requests==limits，最稳定、最难被驱逐，核心服务推荐；**Burstable**：至少设了 requests 但不是 Guaranteed，中等；**BestEffort**：两者都没设，最先被杀。原理是 kubelet 调整容器的 oom_score_adj，OOM Killer 优先杀 adj 高的，BestEffort 最高所以最先牺牲以保护 Guaranteed。生产上核心服务用 Guaranteed，避免 BestEffort 裸奔。

### Q16：DaemonSet 是什么？日志采集和监控怎么用它实现？

> DaemonSet 确保每个节点恰好运行一个 Pod 副本，新节点加入自动创建、节点移除自动回收，默认跳过 master。**日志采集**：容器 stdout/stderr 被运行时写到节点 `/var/log/pods` 和 `/var/log/containers`，每节点跑一个 Filebeat/Fluent Bit（DaemonSet）挂载 `/var/log` 即可采集本节点所有容器日志，发到 ES/Loki。**监控**：每节点跑一个 node-exporter（DaemonSet）暴露 `/metrics` 采集机器级指标，Prometheus 通过服务发现自动抓取。两者都是"每节点一个 Agent"的 DaemonSet 经典模式，资源开销低，是节点级数据采集的标准做法。

### Q17：部署应用时，运维需要配置哪几类文件？各自作用？

> 分两大类：**集群本身的管理类配置**（搭集群时一次性写）和**应用部署类配置**（每部署一个应用都要写）。运维部署应用时主要接触后者，常见有四类：
>
> 1. **资源清单 YAML**：描述"要部署什么"的核心文件，如 Deployment、Service、Ingress 等。含镜像版本、副本数、端口、标签、调度约束等。是运维最主要的工作产物，用 `kubectl apply -f` 提交。
> 2. **配置注入类（ConfigMap / Secret）**：把易变的配置和敏感信息从镜像里剥离出来，避免改个配置就重新打镜像。ConfigMap 存普通配置（如 `application.yml`、超时时间），Secret 存敏感数据（如数据库密码、证书，Base64 编码存储）。通过环境变量或挂载文件两种方式注入容器。
> 3. **权限控制类（RBAC：ServiceAccount / Role / RoleBinding）**：声明"这个应用以什么身份运行、能操作哪些资源"。生产环境尤其重要，避免应用拿过高权限。如一个应用要读集群信息就配 ServiceAccount + Role 授权。
> 4. **存储声明类（PVC / StorageClass）**：有状态应用声明持久化存储，如数据库要 10Gi 存储，写 PVC 申请；StorageClass 定义"用什么样的存储"（如 ssd、nfs），PVC 据此自动创建。
>
> 此外常配合 **Helm Chart**（把这些 YAML 打包成模板，用一份 values.yaml 参数化部署到多环境）。核心记忆：**资源清单定部署形态、ConfigMap/Secret 注入配置、RBAC 管权限、PVC 管存储**。

### Q18：什么是 Operator？为什么需要它？

> Operator 是一种**扩展 K8s 的模式**：把人类运维某个复杂应用的经验（如"数据库主从切换、备份恢复、滚动升级的步骤"）编码成一个**自定义控制器**，让 K8s 自动管理这个应用的全生命周期。
>
> **为什么需要**：K8s 内置控制器（Deployment 等）只懂"通用动作"（维持副本数、滚动更新），但复杂应用（如 MySQL、Kafka、Redis 集群）的运维需要"领域知识"：主从怎么选主、故障怎么切主、备份怎么做、版本怎么安全升级--这些步骤每个应用都不一样，内置控制器搞不定。Operator 就是把这些步骤写成代码，让控制器代替人去执行。
>
> **原理**：Operator = **CRD（自定义资源）+ 自定义控制器**。
> - CRD：定义一种新的资源类型，如 `MySQLCluster`，像 Deployment 一样能用 YAML 声明。
> - 控制器：盯着这个 CRD 资源的变化，跑你写的"运维逻辑"（控制循环），让实际状态趋近期望状态。
> - 用户只需声明 `MySQLCluster`（要 3 副本、版本 8.0），Operator 控制器自动完成建库、配主从、连健康检查等全部步骤。
>
> **本质**：声明式 API + 控制循环的模式，只是作用对象从内置的 Pod 变成了自定义的"应用整体"。常见例子：Prometheus Operator、Kafka Operator、Redis Operator。开发框架用最广的是 **Operator SDK / Kubebuilder**（Go 语言，基于 controller-runtime）。

**追问1：Operator 在哪一步执行？**
> Operator 控制器本身就是一个**常驻 Pod**（和你写的业务服务一样是普通进程），不是 K8s 内核的一部分。它通过 apiserver 的 watch 机制持续盯着 `MySQLCluster` 这类 CR 资源变化，在**常驻控制循环**中执行运维逻辑：watch 到用户提交的 CR -> 调谐（建 StatefulSet/Service/PVC、配主从、健康检查、备份、切主）-> 调 apiserver 创建普通资源 -> kubelet/kube-proxy 真正落地。所以它不是"某一步"，而是一直跑着、随时响应。

**追问2：有内置控制器吗？**
> **没有**。这是常见误解：K8s 内置控制器（Deployment/ReplicaSet/StatefulSet 控制器）只懂通用动作（维持副本数、滚动更新），**不认识** MySQL/Kafka/Redis，也不懂怎么给它们做主从切换。Operator 是**用户/厂商自己开发并部署**的扩展，K8s 只是提供了开发自定义控制器的能力（CRD + controller-runtime）。装上后对 K8s 来说它只是"集群里又多了一个应用"。一句话：**内置控制器管通用资源，Operator 是后天装的"插件"管特定应用。**

**追问3：怎么部署一个自定义 Operator？（以 MySQL Operator 为例）**
> 分两步，顺序固定：**先装 CRD，再部署控制器**。CRD 是"资源定义"（schema，告诉 apiserver 认识 `MySQLCluster`），控制器是"执行逻辑"（跑运维代码的 Pod），两者缺一不可但分开部署。
>
> **步骤1 - 安装 CRD**（集群级，装一次）：
> ```yaml
> apiVersion: apiextensions.k8s.io/v1
> kind: CustomResourceDefinition
> metadata:
>   name: mysqlclusters.example.com
> spec:
>   group: example.com
>   names: {kind: MySQLCluster, plural: mysqlclusters}
>   scope: Namespaced
>   versions:
>     - name: v1
>       served: true
>       storage: true
>       schema:
>         openAPIV3Schema:
>           type: object
>           properties:
>             spec:
>               type: object
>               properties:
>                 replicas: {type: integer}
>                 version:  {type: string}
> ```
> `kubectl apply -f crd.yaml` 后，apiserver 就认识 `MySQLCluster` 了。
>
> **步骤2 - 部署控制器 Pod**（用 Deployment 部署，需带 RBAC 权限的 ServiceAccount 才能操作集群资源）：
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata: {name: mysql-operator}
> spec:
>   replicas: 1                       # 控制器一般 1 副本（leader election 保证高可用）
>   selector: {matchLabels: {app: mysql-operator}}
>   template:
>     metadata: {labels: {app: mysql-operator}}
>     spec:
>       serviceAccountName: mysql-operator-sa   # 需 RBAC：能 watch CR、create StatefulSet/Service/PVC
>       containers:
>         - name: operator
>           image: example/mysql-operator:v1    # 你写好编译出的镜像
> ```
>
> **步骤3 - 用户使用**（像写 Deployment 一样声明）：
> ```yaml
> apiVersion: example.com/v1
> kind: MySQLCluster
> metadata: {name: my-db}
> spec:
>   replicas: 3        # 要 3 副本
>   version: "8.0"
> ```
> `kubectl apply -f my-mysql.yaml` 后，Operator 控制器 watch 到该 CR，自动建 StatefulSet、配主从、建 Service、挂 PVC--把"一个资深 DBA 的操作步骤"全自动化。生产中常用 **Helm/OLM** 一键打包安装这三步。

---

## 三、资料勘误与重点提醒

> 本文档参考知乎《深入Kubernetes：从零开始的进阶之路（上）》并做以下修正与补充：

1. **K8s 与 Docker 运行时关系（重要更正）**：原资料称"K8s 支持 Docker、Containerd、CRI-O 等容器类型"，表述易误解。准确说法：K8s 自 **v1.24（2022-05）移除 dockershim** 后，不再直接支持 Docker 作为运行时，而是统一通过 **CRI 接口**对接 Containerd / CRI-O。Docker 构建出的 OCI 镜像仍可被这些运行时运行——是"运行时"层面不再支持 Docker，不是"镜像"层面。

2. **补齐集群架构（原文重点缺失）**：原资料仅画了 Master/Worker 集群图，未讲解控制平面各组件职责。而 apiserver/etcd/scheduler/controller-manager 与 kubelet/kube-proxy 的职责是面试**最高频考点**，本文 1.2 节已补全，务必掌握"所有组件只和 apiserver 通信、apiserver 只和 etcd 通信"这一架构主线。

3. **补齐声明式 API 与控制循环**：原资料用"期望式声明"一带而过。这是 K8s 区别传统运维的根本设计思想（控制循环 Reconcile Loop），面试常问"K8s 如何实现自愈/自动收敛"，本文 1.3 节已展开。

4. **简化"为什么需要 Pod"论证**：原资料用生产者/消费者 + 共享内存 + 信号量论证，逻辑较绕。本质是"单一职责原则下，紧耦合的多容器需要共享网络/存储"——本文已用主容器 + sidecar 场景替代，更贴近实际。

5. **补充高频资源**：原资料（上篇）只讲 Pod/Label/RS/Service/Deployment。面试还常问 StatefulSet、DaemonSet、Ingress、ConfigMap/Secret、PV/PVC，本文 1.4 节已补充简介。

6. **保留原文正确要点**：原资料对 Service 四种类型、NodePort/LoadBalancer/Headless 区别、Deployment 滚动更新 `maxSurge` 向上取整 / `maxUnavailable` 向下取整的描述均准确，本文沿用。
