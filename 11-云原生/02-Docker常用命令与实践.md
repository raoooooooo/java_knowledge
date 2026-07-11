# Docker 常用命令与实践

## 一、核心概念

### 1.1 镜像相关命令

#### 镜像查询与信息
```bash
# 列出本地镜像
docker images [OPTIONS] [REPOSITORY[:TAG]]
  -a, --all             显示所有镜像（默认隐藏中间镜像）
  -q, --quiet           只显示镜像 ID
  --format              格式化输出

# 查看镜像详细信息
docker inspect [OPTIONS] NAME|ID [NAME|ID...]
  -f, --format          使用 Go 模板格式化输出

# 查看镜像历史（各层信息）
docker history [OPTIONS] IMAGE
  --no-trunc            不截断输出
  -q, --quiet           只显示镜像 ID
```

#### 镜像拉取与推送
```bash
# 从仓库拉取镜像
docker pull [OPTIONS] NAME[:TAG|@DIGEST]
  -a, --all-tags        拉取仓库所有标签的镜像
  --platform            指定平台（如 linux/amd64）

# 推送镜像到仓库
docker push [OPTIONS] NAME[:TAG]
  -a, --all-tags        推送所有标签

# 搜索 Docker Hub 上的镜像
docker search [OPTIONS] TERM
  --limit int           结果数量限制（默认 25）
  --filter=stars=100   筛选星数 >= 100 的镜像
```

#### 镜像构建
```bash
# 从 Dockerfile 构建镜像
docker build [OPTIONS] PATH | URL | -
  -t, --tag name:tag   镜像名称和标签
  -f, --file           指定 Dockerfile 路径
  --build-arg KEY=VAL  传递构建参数
  --no-cache           不使用构建缓存
  --platform           指定目标平台
  -q, --quiet          静默模式
```

#### 镜像删除与清理
```bash
# 删除镜像
docker rmi [OPTIONS] IMAGE [IMAGE...]
  -f, --force          强制删除（即使有容器在使用）

# 清理未使用的镜像/容器/卷/网络
docker system prune [OPTIONS]
  -a, --all            删除所有未使用的（不仅是 dangling）
  -f, --force          不提示确认
  --volumes            同时清理未使用的卷

# 仅清理 dangling 镜像
docker image prune
```

#### 镜像保存与加载
```bash
# 保存镜像为 tar 文件
docker save [OPTIONS] IMAGE [IMAGE...]
  -o, --output file    输出到文件而非 stdout

# 从 tar 文件加载镜像
docker load [OPTIONS]
  -i, --input file     从文件读取而非 stdin

# 示例
docker save -o nginx.tar nginx:alpine
docker load -i nginx.tar
```

### 1.2 容器相关命令

#### 容器启动
```bash
# 创建并启动容器
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
  
  # 基础选项
  --name name           容器名称
  -d, --detach          后台运行
  -i, --interactive     保持标准输入打开
  -t, --tty             分配伪终端
  --rm                  容器退出时自动删除
  
  # 端口映射
  -p, --publish hostPort:containerPort  端口映射（指定主机端口）
  -P, --publish-all     随机映射所有暴露的端口
  
  # 环境变量
  -e, --env KEY=VAL     设置环境变量
  --env-file file       从文件加载环境变量
  
  # 卷挂载
  -v, --volume hostPath:containerPath:mode  挂载卷
  --mount type=bind,source=...,target=...    更灵活的挂载
  
  # 资源限制
  --cpus number         CPU 核心限制
  -m, --memory bytes    内存限制
  --memory-swap bytes   交换内存限制
  
  # 网络
  --network name        连接到指定网络
  
  # 重启策略
  --restart policy      no|on-failure|unless-stopped|always

# 示例
docker run -d -p 8080:80 --name mynginx -v /data:/usr/share/nginx/html nginx
```

#### 容器生命周期管理
```bash
# 启动已存在的容器
docker start [OPTIONS] CONTAINER [CONTAINER...]
  -i, --interactive     交互式模式
  -a, --attach          附加到容器

# 停止运行中的容器
docker stop [OPTIONS] CONTAINER [CONTAINER...]
  -t, --time seconds    超时时间（默认 10s）

# 强制停止（发送 SIGKILL）
docker kill [OPTIONS] CONTAINER [CONTAINER...]

# 重启容器
docker restart [OPTIONS] CONTAINER [CONTAINER...]

# 暂停容器
docker pause CONTAINER [CONTAINER...]

# 恢复暂停的容器
docker unpause CONTAINER [CONTAINER...]
```

#### 容器查询
```bash
# 列出容器
docker ps [OPTIONS]
  -a, --all             显示所有容器（包括停止的）
  -q, --quiet           只显示容器 ID
  -s, --size            显示文件大小
  -n, --last int        显示最近 N 个容器
  --filter key=value    过滤（如 status=running）

# 查看容器详细信息
docker inspect CONTAINER

# 查看容器资源使用情况
docker stats [OPTIONS] [CONTAINER...]
  --no-stream           不持续监控，只输出一次
  -a, --all             显示所有容器（不仅是运行的）

# 查看容器内进程
docker top CONTAINER [ps OPTIONS]

# 查看容器日志
docker logs [OPTIONS] CONTAINER
  -f, --follow          持续跟踪日志
  --tail N              显示最后 N 行
  -t, --timestamps      显示时间戳
  --since time          显示指定时间之后的日志
```

#### 进入容器
```bash
# 在运行中的容器执行命令
docker exec [OPTIONS] CONTAINER COMMAND [ARG...]
  -i, --interactive     保持标准输入
  -t, --tty             分配伪终端
  -d, --detach          后台执行

# 最常用：进入容器交互式 Shell
docker exec -it container_name /bin/bash
docker exec -it container_name /bin/sh  # Alpine 镜像

# 附加到容器（不推荐，会影响容器）
docker attach [OPTIONS] CONTAINER
```

#### 容器删除
```bash
# 删除容器
docker rm [OPTIONS] CONTAINER [CONTAINER...]
  -f, --force          强制删除（即使在运行）
  -v, --volumes        删除关联的卷

# 删除所有停止的容器
docker container prune
```

#### 容器与镜像转换
```bash
# 将容器保存为镜像
docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]
  -a, --author         作者信息
  -m, --message        提交说明
  -c, --change         应用 Dockerfile 指令

# 将容器文件系统导出为 tar
docker export [OPTIONS] CONTAINER
  -o, --output file    输出文件

# 从 tar 导入为镜像
docker import [OPTIONS] file|URL|- [REPOSITORY[:TAG]]

# 注意：export 导出容器文件系统，save 保存完整镜像（含历史）
```

### 1.3 网络相关命令

```bash
# 列出网络
docker network ls

# 查看网络详情
docker network inspect NETWORK

# 创建网络
docker network create [OPTIONS] NETWORK
  --driver driver       网络驱动（bridge/host/overlay/macvlan/none）
  --subme t CIDR        子网
  --gateway IP          网关
  --opt key=value       驱动选项

# 连接容器到网络
docker network connect NETWORK CONTAINER

# 断开容器从网络
docker network disconnect NETWORK CONTAINER

# 删除未使用的网络
docker network prune
```

### 1.4 数据卷（Volume）相关命令

```bash
# 列出卷
docker volume ls

# 查看卷详情
docker volume inspect VOLUME

# 创建卷
docker volume create [OPTIONS] [VOLUME]
  --driver driver       卷驱动
  --opt key=value       驱动选项

# 删除卷
docker volume rm VOLUME

# 删除未使用的卷
docker volume prune
```

---

## 二、常见面试题

### Q1：docker run 命令执行的完整流程是什么？

**核心回答：**
> docker run 执行流程：
> 1. **检查镜像**：本地是否存在指定镜像，不存在则从仓库拉取
> 2. **创建容器**：基于镜像创建容器，分配文件系统、网络、资源
> 3. **分配网络**：创建网络命名空间，分配 IP，连接到网桥
> 4. **启动容器**：执行启动命令，初始化进程
> 5. **附加终端**（如果 -it）：连接标准输入输出，分配伪终端
> 6. **健康检查**（如果配置）：开始监控容器健康状态

### Q2：docker attach 和 docker exec 有什么区别？

**核心回答：**
> - **docker attach**：附加到容器的 1 号进程（PID 1）的标准输入输出。如果容器启动命令不是交互式的，attach 进去可能无法输入；exit 会导致容器退出。**不推荐日常使用**。
> - **docker exec**：在容器内启动一个新的进程执行命令，不会影响 PID 1。可以指定 -it 进入交互式 Shell，exit 不会导致容器退出。**这是日常进入容器的标准方式**。

### Q3：如何排查 Docker 容器启动失败的问题？

**排查步骤：**
1. **查看容器状态**：`docker ps -a` 看容器是否存在
2. **查看容器日志**：`docker logs container_name` 看启动日志
3. **查看容器详情**：`docker inspect container_name` 看配置是否正确
4. **手动启动测试**：去掉 -d 参数前台启动看输出
5. **进入容器调试**：`docker run --rm -it --entrypoint /bin/sh image_name`
6. **检查宿主机资源**：磁盘空间、内存、端口占用
7. **检查 SELinux/AppArmor**：安全策略可能阻止容器运行

### Q4：Dockerfile 中的 COPY 和 ADD 有什么区别？

**核心回答：**
> - **COPY**：只支持从宿主机复制文件到镜像，简单直接
> - **ADD**：除了复制，还支持：
>   1. 从 URL 下载文件（但不推荐，层会变大）
>   2. 自动解压 tar 文件（会破坏镜像层缓存）
> 
> **最佳实践**：优先使用 COPY，ADD 只在需要自动解压时使用。

### Q5：如何限制 Docker 容器的资源使用？

**常用限制：**
```bash
# CPU 限制
docker run --cpus=2          # 最多使用 2 个 CPU 核心
docker run --cpuset-cpus=0-1 # 绑定到 CPU 0 和 1

# 内存限制
docker run -m 512m           # 内存限制 512MB
docker run --memory-swap=1g  # 内存 + swap 总共 1GB

# IO 限制
docker run --blkio-weight=500 # 块设备 IO 权重（10-1000）
```

### Q6：docker save 和 docker export 有什么区别？

| 特性 | docker save | docker export |
|------|------------|--------------|
| 操作对象 | 镜像（image） | 容器（container） |
| 包含内容 | 完整镜像（所有层 + 历史 + 元数据） | 仅容器文件系统快照 |
| 体积 | 较大（包含所有层） | 较小（扁平化） |
| 导入后 | 保留镜像历史和标签 | 变成单层镜像，丢失历史 |
| 用途 | 镜像分发、备份 | 容器快照、迁移 |

### Q7：如何清理 Docker 占用的磁盘空间？

**清理命令：**
```bash
# 一键清理所有未使用的资源
docker system prune -a --volumes

# 分别清理
docker image prune -a    # 清理未使用的镜像
docker container prune   # 清理停止的容器
docker volume prune      # 清理未使用的卷
docker network prune     # 清理未使用的网络

# 查看磁盘使用情况
docker system df
docker system df -v      # 详细视图
```

### Q8：如何实现容器和宿主机的时间同步？

**方法：**
1. **挂载 /etc/localtime**（推荐简单方式）：
   ```bash
   docker run -v /etc/localtime:/etc/localtime:ro ...
   ```

2. **设置 TZ 环境变量**（更灵活）：
   ```bash
   docker run -e TZ=Asia/Shanghai ...
   ```

3. **Dockerfile 中设置时区**：
   ```dockerfile
   ENV TZ=Asia/Shanghai
   RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
   ```

### Q9：容器退出码有哪些常见含义？

| 退出码 | 含义 |
|--------|------|
| 0 | 正常退出 |
| 1 | 应用错误（如除零、数组越界） |
| 2 | 误用 shell 内置命令 |
| 125 | Docker 自身错误（如执行命令错误） |
| 126 | 命令无法执行（权限问题、不是可执行文件） |
| 127 | 命令找不到 |
| 128 + N | 收到信号 N 退出<br>130 = Ctrl+C (SIGINT)<br>137 = SIGKILL (被强制杀死)<br>143 = SIGTERM (正常停止) |

### Q10：如何将宿主机文件复制到容器内？

```bash
# 宿主机 → 容器
docker cp /host/path container_name:/container/path

# 容器 → 宿主机
docker cp container_name:/container/path /host/path

# 也可以使用 tar 管道
tar -cf - /host/path | docker exec -i container_name tar -xf - -C /container
```

---

## 三、生产环境最佳实践

1. **容器应该是无状态的**，数据持久化使用 Volume
2. **一个容器只跑一个进程**，职责单一
3. **不要在容器内存日志**，日志输出到 stdout/stderr，由宿主机统一收集
4. **不要用 latest 标签**，指定具体版本号
5. **健康检查必须配置**，`HEALTHCHECK` 指令
6. **不要以 root 用户运行**，Dockerfile 中创建普通用户
7. **资源限制必须设置**，防止容器耗尽宿主机资源
8. **使用 .dockerignore**，排除不必要的文件，减小镜像体积
9. **多阶段构建**，减小最终镜像大小
10. **定期清理**，防止磁盘空间耗尽
