---
tags:
  - Nginx
  - Linux
  - 部署
  - DevOps
  - 面试
created: 2026-08-22
---

# Nginx 与 Linux 部署

> Nginx 是企业最常用的反向代理和 Web 服务器。Linux 是 Java 服务部署的标准环境。

---

## 一、Nginx 核心概念

### 1.1 正向代理 vs 反向代理

| | 正向代理 | 反向代理 |
|---|---|---|
| 代理对象 | 客户端 | 服务端 |
| 客户端是否知道 | 知道（配代理地址） | 不知道（以为直接访问） |
| 典型场景 | 翻墙、公司出口 | 负载均衡、网关 |

### 1.2 Nginx 三大用途

```
1. 静态资源服务器   → 前端页面 / 图片 / 文件
2. 反向代理 + 负载均衡 → 转发请求到后端多个实例
3. API 网关          → 统一入口 / 跨域处理 / HTTPS 证书
```

---

## 二、Nginx 配置

### 2.1 核心配置结构

```nginx
# nginx.conf
events {
    worker_connections 1024;
}

http {
    # 上游服务器（负载均衡目标）
    upstream backend {
        server 192.168.1.10:8080 weight=1;
        server 192.168.1.11:8080 weight=1;
        server 192.168.1.12:8080 weight=1 backup;  # 备用
    }

    server {
        listen 80;
        server_name api.example.com;

        # 前端静态资源
        location / {
            root /usr/share/nginx/html;
            try_files $uri $uri/ /index.html;  # Vue/React 路由
        }

        # API 反向代理
        location /api/ {
            proxy_pass http://backend/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        # 文件上传大小
        client_max_body_size 10m;

        # 日志
        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;
    }
}
```

### 2.2 负载均衡策略

| 策略 | 配置 | 说明 |
|---|---|---|
| 轮询（默认） | `server addr:port;` | 按顺序轮流 |
| 权重 | `server addr:port weight=3;` | 权重高的多分 |
| IP Hash | `ip_hash;` | 同 IP 固定到同一台 |
| 最少连接 | `least_conn;` | 分给连接最少的 |

### 2.3 常用 location 匹配

```nginx
location = /exact { }          # 精确匹配
location /api/ { }              # 前缀匹配
location ~ \.php$ { }           # 正则匹配（区分大小写）
location ~* \.jpg$ { }          # 正则匹配（不区分大小写）
location / { }                  # 默认匹配（兜底）
```

---

## 三、Nginx 常用命令

```bash
nginx                    # 启动
nginx -s reload          # 重载配置（不中断服务）
nginx -s stop            # 停止
nginx -t                 # 检查配置语法
nginx -T                 # 查看完整配置
```

---

## 四、Linux 常用命令

### 4.1 文件操作

| 命令 | 作用 | 示例 |
|---|---|---|
| `ls -la` | 查看文件 | `ls -la /home` |
| `cd` | 切换目录 | `cd /opt/app` |
| `cp` | 复制 | `cp app.jar /backup/` |
| `mv` | 移动/重命名 | `mv old.jar new.jar` |
| `rm -rf` | 递归删除 | `rm -rf /tmp/test` |
| `mkdir -p` | 创建多级目录 | `mkdir -p a/b/c` |
| `chmod` | 改权限 | `chmod +x deploy.sh` |
| `find` | 查找文件 | `find / -name "*.log"` |
| `tar` | 压缩解压 | `tar -zxvf app.tar.gz` |

### 4.2 文本处理

```bash
grep "ERROR" app.log              # 搜索关键词
grep -n "ERROR" app.log            # 显示行号
grep -c "ERROR" app.log            # 统计出现次数
tail -f app.log                    # 实时查看日志末尾
tail -100f app.log                 # 查看最后 100 行
head -20 app.log                   # 查看开头 20 行
cat file1 file2 > merged           # 合并文件
less app.log                       # 分页查看（q 退出，/ 搜索）
```

### 4.3 进程与端口

```bash
ps -ef | grep java                 # 查看 Java 进程
netstat -tlnp                      # 查看监听端口
lsof -i:8080                       # 查看 8080 端口占用
kill -9 12345                      # 强杀进程
top                                 # 实时系统资源
df -h                               # 磁盘使用
free -h                             # 内存使用
```

### 4.4 Java 部署常用

```bash
# 后台运行 Java 服务
nohup java -jar -Xms512m -Xmx512m app.jar > app.log 2>&1 &

# 查看进程
ps -ef | grep app.jar

# 停止服务
kill -9 $(ps -ef | grep app.jar | grep -v grep | awk '{print $2}')

# 查看实时日志
tail -f app.log
```

---

## 五、企业部署流程

### 5.1 典型 CI/CD 流程

```
开发提交代码 → Git Push
    ↓
CI（GitHub Actions / Jenkins）
    ↓ Maven 打包
    ↓ Docker 构建镜像
    ↓ 推送到镜像仓库
    ↓
CD（部署）
    ↓ 服务器拉取新镜像
    ↓ docker-compose up -d
    ↓ Nginx 负载均衡
    ↓ 健康检查
    ↓ 完成
```

### 5.2 常用端口约定

| 服务 | 端口 |
|---|---|
| HTTP | 80 |
| HTTPS | 443 |
| MySQL | 3306 |
| Redis | 6379 |
| RabbitMQ | 5672（AMQP）/ 15672（管理界面） |
| Kafka | 9092 |
| Spring Boot | 8080 |
| Nginx | 80 / 443 |

---

## 六、面试高频题

**Q1: Nginx 负载均衡有哪些策略？**
> 轮询（默认）、权重（weight）、IP Hash（会话保持）、最少连接（least_conn）。

**Q2: Nginx 反向代理的作用？**
> 隐藏后端服务、负载均衡、SSL 终结、静态资源缓存、跨域处理。

**Q3: Java 服务怎么部署到 Linux？**
> 1. Maven 打包 jar；2. 上传到服务器；3. nohup java -jar 启动；4. 或用 Docker 容器化部署。

**Q4: 怎么查看线上 Java 服务日志？**
> `tail -f app.log` 实时查看；`grep "ERROR" app.log` 过滤错误；`less app.log` 分页搜索。

**Q5: 端口被占用怎么办？**
> `lsof -i:8080` 或 `netstat -tlnp | grep 8080` 查看占用进程，`kill -9 PID` 终止。

---

## 🔗 相关笔记

- [[Docker基础|Docker 基础]] — 容器化部署
- [[../06-工具链/02-Git实战|Git 实战]] — CI/CD 源头
- [[../06-工具链/01-Maven实战|Maven 实战]] — 打包构建
- [[../04-Spring生态/SpringBoot-基础与配置|Spring Boot 基础与配置]] — 应用部署
- [[../07-面试与项目/Java学习情况分析与实习冲刺计划|学习计划]] — 第二层会用+懂流程