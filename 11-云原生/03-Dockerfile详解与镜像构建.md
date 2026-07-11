# Dockerfile 详解与镜像构建

## 一、核心概念

### 1.1 Dockerfile 概述

**什么是 Dockerfile？**
- Dockerfile 是一个文本文件，包含构建 Docker 镜像所需的所有指令
- 每一条指令构建镜像的一层，多条指令组成完整镜像
- Docker 按顺序执行指令，支持缓存机制加速构建

**Dockerfile 基本结构：**
```dockerfile
# 基础镜像（第一条指令必须是 FROM）
FROM base_image:tag

# 元数据
MAINTAINER author <email>  # 已过时，使用 LABEL
LABEL version="1.0" description="这是一个示例镜像"

# 环境变量
ENV KEY=VALUE

# 工作目录
WORKDIR /app

# 复制文件
COPY src dest
ADD src dest

# 执行命令
RUN command

# 暴露端口
EXPOSE 8080

# 容器启动命令
CMD ["executable", "arg1", "arg2"]
```

### 1.2 Dockerfile 指令详解

#### FROM - 指定基础镜像
```dockerfile
# 格式
FROM [--platform=<platform>] <image>[:<tag>] [AS <name>]

# 示例
FROM nginx:alpine
FROM openjdk:11-jre-slim AS builder

# 特殊：scratch 为空镜像，用于构建极小镜像
FROM scratch
```

**注意：** FROM 必须是 Dockerfile 的第一条有效指令，可以有多个 FROM（多阶段构建）。

#### RUN - 执行命令
在构建阶段执行命令，创建新的镜像层。

```dockerfile
# shell 形式（默认 /bin/sh -c 执行）
RUN apt-get update && apt-get install -y nginx

# exec 形式（避免 shell 处理，直接调用可执行文件）
RUN ["apt-get", "update"]
RUN ["/bin/bash", "-c", "echo hello"]
```

**最佳实践：**
- 多条命令合并成一个 RUN，减少镜像层数
- 命令最后清理缓存：`rm -rf /var/lib/apt/lists/*`
- 复杂命令使用反斜杠换行，提高可读性

```dockerfile
# 反面教材：多个 RUN 产生多层
RUN apt-get update
RUN apt-get install -y nginx
RUN apt-get install -y curl

# 正确写法：合并 RUN，清理缓存
RUN apt-get update \
    && apt-get install -y \
        nginx \
        curl \
    && rm -rf /var/lib/apt/lists/*
```

#### CMD - 容器启动命令
指定容器启动时默认执行的命令，**只能有一个 CMD**。

```dockerfile
# exec 形式（推荐）
CMD ["nginx", "-g", "daemon off;"]

# shell 形式
CMD nginx -g 'daemon off;'

# 作为 ENTRYPOINT 的参数
CMD ["param1", "param2"]
```

**与 RUN 的区别：**
- RUN：构建时执行，产生镜像层
- CMD：容器启动时执行，不产生镜像层

**重要特性：**
- `docker run` 命令行参数会覆盖 CMD
  ```bash
  docker run my_image echo hello  # CMD 被覆盖
  ```

#### ENTRYPOINT - 入口点
配置容器启动时执行的命令，**不会被 docker run 的命令行参数覆盖**。

```dockerfile
# exec 形式（推荐）
ENTRYPOINT ["nginx", "-g", "daemon off;"]

# shell 形式
ENTRYPOINT nginx -g 'daemon off;'
```

**ENTRYPOINT + CMD 配合使用：**
- ENTRYPOINT 指定固定的执行程序
- CMD 指定可变的默认参数

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]
CMD ["--server.port=8080"]
```

运行时参数会追加到 CMD：
```bash
docker run my_app --server.port=9090
# 实际执行：java -jar app.jar --server.port=9090
```

#### EXPOSE - 声明端口
声明容器运行时监听的端口，**只是文档说明，不会自动映射端口**。

```dockerfile
EXPOSE 8080
EXPOSE 80/tcp
EXPOSE 53/udp
```

**注意：** 实际端口映射需要 `docker run -p` 或 `-P` 参数。

#### ENV - 设置环境变量
设置环境变量，在构建和运行时都有效。

```dockerfile
# 单个变量
ENV JAVA_HOME /usr/lib/jvm/java-11-openjdk-amd64

# 多个变量
ENV APP_HOME=/app \
    APP_PORT=8080 \
    JAVA_OPTS="-Xmx512m -Xms256m"
```

使用环境变量：
```dockerfile
RUN echo $APP_HOME
WORKDIR $APP_HOME
```

构建时也可以传入：
```bash
docker build --build-arg JAVA_OPTS="-Xmx1g" .
```

#### ARG - 构建参数
仅在构建过程中有效的变量，**运行时不存在**。

```dockerfile
# 定义构建参数
ARG VERSION=latest
ARG USER

# 使用
FROM myapp:${VERSION}
RUN echo "Building for user: ${USER}"
```

构建时传入：
```bash
docker build --build-arg VERSION=1.0 --build-arg USER=admin .
```

**ARG vs ENV 区别：**

| 特性 | ARG | ENV |
|------|-----|-----|
| 构建时可用 | ✅ | ✅ |
| 运行时可用 | ❌ | ✅ |
| docker build 传入 | ✅ | ❌ |
| docker run 传入 | ❌ | ✅ |

#### COPY - 复制文件
将宿主机文件复制到镜像中。

```dockerfile
# 基本格式
COPY src dest

# 支持通配符
COPY *.jar /app/
COPY config.? /app/

# 改变所有者和组（--chown）
COPY --chown=user:group src dest

# 多阶段构建：从其他阶段复制
COPY --from=builder /app/target/*.jar /app/
```

**注意事项：**
- src 必须是构建上下文中的路径，不能是绝对路径
- 目录复制时，src 结尾的 / 很重要：
  - `COPY dir /app` → 将 dir 目录内容复制到 /app
  - `COPY dir/ /app` → 同样效果

#### ADD - 增强版复制
除了 COPY 的功能，还支持：
1. 自动解压 tar 文件（tar/gzip/bzip2/xz）
2. 支持 URL 下载文件

```dockerfile
# 解压 tar 文件到 /app
ADD archive.tar.gz /app/

# 从 URL 下载
ADD https://example.com/file.zip /tmp/
```

**最佳实践：**
- 优先使用 COPY，ADD 只在需要自动解压时使用
- 不要用 ADD 下载文件（会增加镜像层且无法删除），应该用 `RUN curl/wget`

#### WORKDIR - 设置工作目录
设置后续 RUN/CMD/ENTRYPOINT/COPY/ADD 指令的工作目录。

```dockerfile
WORKDIR /app

# 可以多次设置，相对路径会拼接
WORKDIR src
# 当前目录：/app/src
```

**注意：** WORKDIR 会自动创建不存在的目录。

#### VOLUME - 创建挂载点
在容器内创建一个目录，用于挂载宿主机卷或其他容器的卷。

```dockerfile
VOLUME ["/data", "/logs"]
VOLUME /data
```

**注意：**
- VOLUME 只是声明，实际挂载需要 `docker run -v`
- VOLUME 之后对该目录的修改都不会保留在镜像中

#### USER - 指定用户
指定后续 RUN/CMD/ENTRYPOINT 指令执行时的用户。

```dockerfile
# 指定用户
RUN useradd -m appuser
USER appuser

# 也可以指定 UID:GID
USER 1000:1000
```

**最佳实践：**
- 不要以 root 用户运行应用
- Dockerfile 中创建普通用户，最后切换到该用户

#### HEALTHCHECK - 健康检查
检查容器健康状态。

```dockerfile
# 格式
HEALTHCHECK [OPTIONS] CMD command

# 示例
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

# 禁用健康检查（继承基础镜像时）
HEALTHCHECK NONE
```

**选项说明：**
- `--interval`：检查间隔，默认 30s
- `--timeout`：超时时间，默认 30s
- `--start-period`：启动宽限期，默认 0s
- `--retries`：重试次数，默认 3 次

**健康状态：**
- starting：初始状态
- healthy：检查成功
- unhealthy：连续失败达到重试次数

#### ONBUILD - 构建触发器
当该镜像作为其他镜像的基础镜像时，触发执行的指令。

```dockerfile
ONBUILD COPY . /app
ONBUILD RUN mvn package
```

**用途：** 制作通用构建基础镜像，子镜像自动执行构建流程。

### 1.3 多阶段构建（Multi-stage Build）

**为什么需要多阶段构建？**
- 构建环境通常需要很多工具（编译器、依赖库）
- 运行环境只需要编译后的产物
- 多阶段构建可以分离构建和运行环境，减小最终镜像大小

**示例：Java 应用构建**

```dockerfile
# 阶段 1：构建阶段（包含完整 JDK 和 Maven）
FROM maven:3.8-openjdk-11 AS builder

WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline

COPY src ./src
RUN mvn package -DskipTests

# 阶段 2：运行阶段（只有 JRE）
FROM openjdk:11-jre-slim

WORKDIR /app
COPY --from=builder /app/target/myapp.jar .

EXPOSE 8080
CMD ["java", "-jar", "myapp.jar"]
```

**示例：Go 应用构建**
```dockerfile
# 构建阶段
FROM golang:1.18 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o myapp .

# 运行阶段（空镜像）
FROM scratch
COPY --from=builder /app/myapp /
EXPOSE 8080
CMD ["/myapp"]
```

### 1.4 .dockerignore 文件

排除不需要复制到镜像中的文件，减小构建上下文大小。

```
# .dockerignore 示例

# 版本控制
.git
.gitignore

# 构建产物
target/
dist/
*.jar
*.war

# 临时文件
*.log
npm-debug.log
tmp/

# IDE
.idea/
.vscode/
*.iml

# 操作系统
.DS_Store
Thumbs.db

# 反向模式（不排除）
!config/prod.yml
```

---

## 二、常见面试题

### Q1：Dockerfile 中的 RUN、CMD、ENTRYPOINT 有什么区别？

**核心回答：**
> - **RUN**：构建镜像时执行的命令，会产生新的镜像层，常用于安装软件、编译代码
> - **CMD**：容器启动时默认执行的命令，可以被 `docker run` 的命令行参数覆盖，一个 Dockerfile 只有最后一个 CMD 有效
> - **ENTRYPOINT**：容器启动时执行的入口命令，不会被命令行参数覆盖，参数会追加到命令后，常用于配置固定的启动程序
> 
> 最佳实践：ENTRYPOINT 指定固定程序，CMD 指定默认参数。

### Q2：COPY 和 ADD 有什么区别？

**核心回答：**
> - **COPY**：单纯将宿主机文件复制到镜像中，功能明确，推荐优先使用
> - **ADD**：除了 COPY 的功能，还支持自动解压 tar 文件和从 URL 下载文件，但会破坏缓存且可能引入安全问题
> 
> **最佳实践**：除非需要自动解压 tar 文件，否则始终使用 COPY。

### Q3：如何优化 Docker 镜像大小？

**优化方法：**
1. **使用轻量级基础镜像**：Alpine > slim > 完整版
2. **多阶段构建**：分离构建环境和运行环境
3. **合并 RUN 指令**：减少镜像层数
4. **清理缓存**：安装软件后清理包管理缓存
5. **.dockerignore**：排除不必要的文件
6. **避免安装调试工具**：curl、vim 等只在需要时安装
7. **压缩镜像层**：使用 docker-squash 等工具

**前后对比示例：**
```dockerfile
# 优化前（> 1GB）
FROM ubuntu
RUN apt-get update
RUN apt-get install -y openjdk-11-jdk maven
COPY . .
RUN mvn package
CMD java -jar target/app.jar

# 优化后（< 200MB）
FROM maven:3.8-openjdk-11 AS builder
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

FROM openjdk:11-jre-slim
COPY --from=builder /app/target/app.jar .
CMD ["java", "-jar", "app.jar"]
```

### Q4：Docker 构建缓存的原理是什么？如何利用缓存？

**缓存原理：**
> Docker 按顺序检查每条指令：
> 1. 如果指令和镜像层不匹配，缓存失效，重新执行
> 2. 如果指令匹配，继续检查下一条
> 3. COPY/ADD 会检查文件内容的 checksum
> 4. RUN 只检查命令字符串是否相同
> 5. 一旦某一层缓存失效，后续所有层都重新构建

**利用缓存的技巧：**
1. **将不变的指令放在前面**：基础镜像、环境变量
2. **将经常变化的指令放在后面**：代码复制
3. **分层复制依赖**：先复制 pom.xml/package.json，下载依赖，再复制源代码
   ```dockerfile
   COPY pom.xml .
   RUN mvn dependency:go-offline
   COPY src ./src
   RUN mvn package
   ```
4. **避免破坏缓存**：不要在 RUN 中使用会变化的命令（如 `apt-get update` 单独一行）

### Q5：什么是构建上下文（build context）？为什么重要？

**核心回答：**
> `docker build PATH` 中的 PATH 就是构建上下文。Docker 客户端会将该目录下的所有文件打包发送给 Docker Daemon。
> 
> **重要性：**
> - 上下文过大（包含 node_modules、target 等）会导致构建缓慢
> - COPY 指令只能复制上下文中的文件
> - .dockerignore 可以排除不需要的文件，减小上下文大小
> 
> **最佳实践：** Dockerfile 放在项目根目录，配置好 .dockerignore。

### Q6：容器启动时如何传递环境变量？有哪些方式？

**方式：**

1. **Dockerfile 中设置默认值**：
   ```dockerfile
   ENV DB_HOST=localhost DB_PORT=3306
   ```

2. **docker run 命令行传入**：
   ```bash
   docker run -e DB_HOST=mysql -e DB_PASS=secret myapp
   ```

3. **从环境变量文件加载**：
   ```bash
   docker run --env-file .env myapp
   ```

4. **Docker Compose 中配置**：
   ```yaml
   environment:
     - DB_HOST=mysql
     - DB_PASS=${DB_PASS}  # 从宿主机环境变量读取
   env_file:
     - .env
   ```

### Q7：为什么不推荐在 Docker 容器中运行 sshd？

**原因：**
1. **违背容器理念**：一个容器应该只跑一个进程
2. **增加攻击面**：ssh 是额外的安全风险
3. **管理复杂**：需要管理 ssh 密钥、权限
4. **不需要**：`docker exec` 可以完全替代 ssh 进入容器

**正确做法：** 需要调试时使用 `docker exec -it container /bin/bash`。

### Q8：如何查看镜像构建历史？如何调试 Dockerfile 构建失败？

**查看构建历史：**
```bash
docker history image_name
docker history --no-trunc image_name  # 不截断命令
```

**调试构建失败：**
1. **看错误输出**：哪一步失败，错误信息是什么
2. **利用失败的中间镜像**：
   ```bash
   # 假设第 5 步失败，进入第 4 步成功的镜像
   docker run -it <中间镜像ID> /bin/bash
   # 手动执行失败的命令，看具体错误
   ```
3. **构建时不删除中间容器**：
   ```bash
   docker build --rm=false .
   ```

### Q9：什么是 Docker 的 1750 MB 镜像大小限制？

**说明：**
- 这不是 Docker 的硬限制，是 AUFS 文件系统的限制
- OverlayFS 没有这个限制
- 现代 Docker 默认使用 OverlayFS，基本不需要担心

**避免过大镜像的方法：**
- 多阶段构建
- 不要在镜像中存数据
- 合理清理缓存和日志

### Q10：健康检查（HEALTHCHECK）的原理是什么？有什么用？

**原理：**
> Docker 定期在容器内执行 HEALTHCHECK 指定的命令：
> - 退出码 0 = 健康
> - 退出码 1 = 不健康
> - 退出码 2 = 保留（不使用）
> 
> 连续多次失败后，容器状态变为 unhealthy。

**作用：**
- 服务状态感知（启动成功、运行正常）
- 编排系统（Swarm/K8s）自动替换不健康容器
- 配合滚动更新，确保新版本服务正常才继续

**示例：**
```dockerfile
HEALTHCHECK --interval=10s --timeout=3s --retries=3 \
    CMD wget -q --spider http://localhost:8080/actuator/health || exit 1
```

---

## 三、Dockerfile 最佳实践总结

1. **使用官方镜像作为基础镜像**：安全、维护好
2. **优先使用 Alpine 镜像**：体积小，安全
3. **一个容器只跑一个进程**：职责单一，易于管理
4. **最小化镜像层数**：合并相关的 RUN 指令
5. **合理利用构建缓存**：不变的放前面，变化的放后面
6. **多阶段构建**：分离构建和运行环境
7. **不要用 latest 标签**：指定具体版本，可复现
8. **不要以 root 用户运行**：创建普通用户，切换权限
9. **配置健康检查**：便于监控和自动恢复
10. **合理设置 WORKDIR**：避免大量 cd 命令
11. **使用 .dockerignore**：减小构建上下文
12. **清理缓存**：RUN 最后清理包管理缓存
13. **注释和元数据**：LABEL 添加版本、维护者等信息
14. **日志输出到 stdout/stderr**：不要在容器内写日志文件
