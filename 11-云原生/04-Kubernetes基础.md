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

### 1.5 kubectl 常用命令与资源清单（简要）

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

### Q8：StatefulSet 和 Deployment 区别？什么场景用？

> Deployment 管无状态应用，Pod 互换、名字随机、共享存储。StatefulSet 管有状态应用：Pod 有**稳定网络身份**（pod-0、pod-1，按序）、**独立持久存储**（每个 Pod 绑定自己的 PVC）、**有序**部署/扩缩/删除。典型场景：数据库、消息队列等需要稳定标识和独立存储的服务。

### Q9：Ingress 和 Service（LoadBalancer）有什么区别？

> LoadBalancer 类型的 Service 每个服务都会创建一个云厂商负载均衡器，成本高且只能四层。Ingress 是七层（HTTP/HTTPS）路由，一个 Ingress 控制器（如 Nginx Ingress）对外只暴露一个入口，按**域名/路径**把流量分发到多个 Service，更省资源、更灵活，是对外 Web 流量入口的主流方案。

### Q10：K8s 的声明式和命令式有什么区别？

> 命令式告诉系统"做什么"（启动 3 个 Pod），需要人盯结果；声明式只描述"期望状态"（3 个副本），系统通过控制循环持续让实际状态趋近期望状态。好处是幂等、可自愈——Pod 挂了自动补、配置改了自动收敛，运维只需声明意图，不用关心具体执行步骤。

---

## 三、资料勘误与重点提醒

> 本文档参考知乎《深入Kubernetes：从零开始的进阶之路（上）》并做以下修正与补充：

1. **K8s 与 Docker 运行时关系（重要更正）**：原资料称"K8s 支持 Docker、Containerd、CRI-O 等容器类型"，表述易误解。准确说法：K8s 自 **v1.24（2022-05）移除 dockershim** 后，不再直接支持 Docker 作为运行时，而是统一通过 **CRI 接口**对接 Containerd / CRI-O。Docker 构建出的 OCI 镜像仍可被这些运行时运行——是"运行时"层面不再支持 Docker，不是"镜像"层面。

2. **补齐集群架构（原文重点缺失）**：原资料仅画了 Master/Worker 集群图，未讲解控制平面各组件职责。而 apiserver/etcd/scheduler/controller-manager 与 kubelet/kube-proxy 的职责是面试**最高频考点**，本文 1.2 节已补全，务必掌握"所有组件只和 apiserver 通信、apiserver 只和 etcd 通信"这一架构主线。

3. **补齐声明式 API 与控制循环**：原资料用"期望式声明"一带而过。这是 K8s 区别传统运维的根本设计思想（控制循环 Reconcile Loop），面试常问"K8s 如何实现自愈/自动收敛"，本文 1.3 节已展开。

4. **简化"为什么需要 Pod"论证**：原资料用生产者/消费者 + 共享内存 + 信号量论证，逻辑较绕。本质是"单一职责原则下，紧耦合的多容器需要共享网络/存储"——本文已用主容器 + sidecar 场景替代，更贴近实际。

5. **补充高频资源**：原资料（上篇）只讲 Pod/Label/RS/Service/Deployment。面试还常问 StatefulSet、DaemonSet、Ingress、ConfigMap/Secret、PV/PVC，本文 1.4 节已补充简介。

6. **保留原文正确要点**：原资料对 Service 四种类型、NodePort/LoadBalancer/Headless 区别、Deployment 滚动更新 `maxSurge` 向上取整 / `maxUnavailable` 向下取整的描述均准确，本文沿用。
