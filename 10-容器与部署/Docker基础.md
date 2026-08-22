---
tags:
  - Docker
  - 容器
  - 部署
  - DevOps
  - 面试
created: 2026-08-22
---

# Docker 基础

> Docker 是容器化技术，把应用和依赖打包成一个可移植的镜像，一次构建到处运行。

---

## 一、Docker 是什么

### 1.1 容器 vs 虚拟机

| | Docker 容器 | 虚拟机 |
|---|---|---|
| 隔离级别 | 进程级隔离 | 操作系统级隔离 |
| 启动速度 | 秒级 | 分钟级 |
| 资源占用 | MB 级 | GB 级 |
| 镜像大小 | 通常 10-500MB | 几个 GB |
| 运行性能 | 接近原生 | 有虚拟化开销 |
| 适用场景 | 微服务部署 | 强隔离环境 |

### 1.2 核心概念

| 概念 | 说明 | 类比 |
|---|---|---|
| **镜像 (Image)** | 只读模板，包含应用+依赖 | 面向对象的类 |
| **容器 (Container)** | 镜像的运行实例 | 面向对象的对象 |
| **Dockerfile** | 构建镜像的脚本 | 配方 |
| **仓库 (Registry)** | 存放镜像 | 应用商店 |
| **数据卷 (Volume)** | 持久化存储 | 外接硬盘 |
| **网络 (Network)** | 容器间通信 | 局域网 |

---

## 二、常用命令

### 2.1 镜像命令

```bash
docker pull mysql:8.0              # 拉取镜像
docker images                       # 查看本地镜像
docker rmi mysql:8.0                # 删除镜像
docker build -t myapp:1.0 .         # 构建镜像
docker tag myapp:1.0 myapp:latest  # 打标签
```

### 2.2 容器命令

```bash
docker run -d \                     # 后台运行
  --name my-mysql \                 # 容器名
  -p 3306:3306 \                    # 端口映射 宿主机:容器
  -e MYSQL_ROOT_PASSWORD=123456 \   # 环境变量
  -v /data/mysql:/var/lib/mysql \   # 数据卷挂载
  --network my-net \                # 加入网络
  mysql:8.0                         # 镜像

docker ps                           # 查看运行中容器
docker ps -a                        # 查看所有容器
docker stop my-mysql                # 停止容器
docker start my-mysql               # 启动容器
docker restart my-mysql             # 重启容器
docker rm my-mysql                  # 删除容器
docker logs my-mysql                # 查看日志
docker exec -it my-mysql bash       # 进入容器
docker cp my-mysql:/tmp/a.txt .     # 从容器复制文件到宿主机
```

### 2.3 速查表

| 操作 | 命令 |
|---|---|
| 拉取镜像 | `docker pull <image>:<tag>` |
| 查看镜像 | `docker images` |
| 运行容器 | `docker run -d --name <name> -p <port>:<port> <image>` |
| 查看容器 | `docker ps` / `docker ps -a` |
| 进入容器 | `docker exec -it <name> bash` |
| 查看日志 | `docker logs -f --tail 100 <name>` |
| 停止/启动 | `docker stop/start <name>` |
| 删除容器 | `docker rm <name>` |
| 构建镜像 | `docker build -t <name>:<tag> .` |
| 清理悬空镜像 | `docker image prune` |

---

## 三、Dockerfile

### 3.1 常用指令

| 指令 | 作用 | 示例 |
|---|---|---|
| `FROM` | 基础镜像 | `FROM openjdk:17-slim` |
| `WORKDIR` | 工作目录 | `WORKDIR /app` |
| `COPY` | 复制文件 | `COPY target/app.jar app.jar` |
| `ADD` | 复制+解压 | `ADD xxx.tar.gz /opt/` |
| `RUN` | 构建时执行 | `RUN apt-get install -y vim` |
| `ENV` | 环境变量 | `ENV TZ=Asia/Shanghai` |
| `EXPOSE` | 声明端口 | `EXPOSE 8080` |
| `CMD` | 启动命令 | `CMD ["java","-jar","app.jar"]` |
| `ENTRYPOINT` | 启动入口 | `ENTRYPOINT ["java","-jar"]` |

### 3.2 Spring Boot Dockerfile

```dockerfile
# 构建阶段
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

# 运行阶段（更小镜像）
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /build/target/*.jar app.jar
ENV TZ=Asia/Shanghai
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

> 多阶段构建：builder 阶段用 Maven 编译，运行阶段只保留 JRE，镜像从 800MB 缩到 ~200MB。

### 3.3 构建和运行

```bash
# 构建镜像
docker build -t myapp:1.0 .

# 运行
docker run -d --name myapp -p 8080:8080 myapp:1.0
```

---

## 四、数据卷与网络

### 4.1 数据卷（持久化）

```bash
# 命名卷（推荐）
docker volume create mydata
docker run -v mydata:/var/lib/mysql mysql:8.0

# 绑定挂载（开发调试）
docker run -v /host/path:/container/path nginx

# 临时卷
docker run --tmpfs /tmp nginx
```

### 4.2 网络

| 网络类型 | 说明 |
|---|---|
| bridge（默认） | 容器间通过 IP 通信 |
| host | 容器直接用宿主机网络 |
| none | 无网络 |
| 自定义网络 | 容器间可用容器名通信 |

```bash
# 创建自定义网络
docker network create my-net

# 容器加入网络后可用容器名互访
docker run -d --name app --network my-net myapp
docker run -d --name db --network my-net mysql
# app 容器内可直接 ping db（容器名即域名）
```

---

## 五、Docker Compose

### 5.1 多容器编排

```yaml
# docker-compose.yml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    depends_on:
      - mysql
      - redis
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/mydb
      SPRING_REDIS_HOST: redis
    networks:
      - my-net

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: "123456"
      MYSQL_DATABASE: mydb
    volumes:
      - mysql-data:/var/lib/mysql
    ports:
      - "3306:3306"
    networks:
      - my-net

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    networks:
      - my-net

volumes:
  mysql-data:

networks:
  my-net:
```

### 5.2 常用命令

```bash
docker-compose up -d        # 启动全部服务
docker-compose down         # 停止并删除
docker-compose logs -f app  # 查看日志
docker-compose restart app  # 重启单个服务
docker-compose ps           # 查看状态
```

---

## 六、面试高频题

**Q1: Docker 和虚拟机的区别？**
> 容器是进程级隔离，共享宿主机内核，启动快、占用少。虚拟机是操作系统级隔离，有完整内核，隔离强但重。

**Q2: Dockerfile 中 CMD 和 ENTRYPOINT 的区别？**
> CMD 可以被 docker run 后面的参数覆盖。ENTRYPOINT 不会被覆盖，docker run 后的参数会追加到 ENTRYPOINT 后面。推荐用 ENTRYPOINT + CMD 组合。

**Q3: Docker 数据怎么持久化？**
> 数据卷（Volume）和绑定挂载（Bind Mount）。生产推荐命名卷，开发可用绑定挂载。

**Q4: Docker 多阶段构建的好处？**
> 编译阶段用大镜像（含 Maven/JDK），运行阶段只保留 JRE，大幅减小最终镜像体积。

**Q5: Docker 容器间怎么通信？**
> 创建自定义网络，容器加入后可用容器名作为域名互访。默认 bridge 网络只能用 IP 通信。

---

## 🔗 相关笔记

- [[Nginx与Linux部署|Nginx 与 Linux 部署]] — 反向代理与常用命令
- [[../04-Spring生态/SpringBoot-基础与配置|Spring Boot 基础与配置]] — 打包部署
- [[../06-工具链/01-Maven实战|Maven 实战]] — 构建打包
- [[../07-面试与项目/Java学习情况分析与实习冲刺计划|学习计划]] — 第二层会用+懂流程