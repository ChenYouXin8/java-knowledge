# Maven 实战

> Maven 是 Java 项目的**构建工具**，核心职责：管理依赖（自动下载 jar）、统一项目结构、自动化构建（编译→测试→打包）。

---

## 1️⃣ 核心概念

```
Maven 解决的问题：
1. 依赖管理：不用手动下载 jar，Maven 自动下载
2. 项目结构统一：所有 Java 项目用同一套目录结构
3. 构建自动化：编译→测试→打包，一条命令搞定

Maven 坐标（定位 dependency）：
  groupId      → 公司/组织名（倒写域名） io.github.chenyouxin8
  artifactId   → 项目/模块名              chen-ai-agent
  version      → 版本号                   1.0.0
  → 三者唯一确定一个 jar 包

Maven 仓库（去哪找 jar）：
  中央仓库     → 官方，公开免费
  阿里云镜像   → 国内加速，必配！
  私有仓库     → 公司内部自己搭建
```

---

## 2️⃣ pom.xml 骨架（对照你的 chen-ai-agent）

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <!-- 父 POM：统一管理版本号 -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.x</version>
        <relativePath/>
    </parent>

    <!-- 项目基本信息 -->
    <groupId>io.github.chenyouxin8</groupId>
    <artifactId>chen-ai-agent</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>
    <name>chen-ai-agent</name>
    <description>AI 恋爱助手</description>

    <!-- properties：统一版本号变量 -->
    <properties>
        <java.version>17</java.version>
        <spring-ai.version>1.0.0-M6</spring-ai.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <!-- dependencies：项目依赖 -->
    <dependencies>
        <!-- Spring Boot Web（内嵌 Tomcat） -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Spring AI 阿里云（通义千问） -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-alibaba-starter</artifactId>
        </dependency>

        <!-- Lombok（自动生成 getter/setter） -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>    <!-- optional=true 不会传递依赖 -->
        </dependency>

        <!-- 测试（JUnit 5） -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>          <!-- 只在 src/test 下可用 -->
        </dependency>
    </dependencies>

    <!-- dependencyManagement：版本锁定（大型项目） -->
    <!-- 子 module 不写 version，统一从父 POM 继承 -->
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>com.example</groupId>
                <artifactId>my-lib</artifactId>
                <version>1.0.0</version>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <!-- 构建配置 -->
    <build>
        <finalName>${project.artifactId}-${project.version}</finalName>

        <!-- 插件 -->
        <plugins>
            <!-- Spring Boot Maven 插件：打可执行 jar -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>

            <!-- 编译插件 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.11.0</version>
                <configuration>
                    <source>17</source>
                    <target>17</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>

            <!-- 运行测试 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.2.2</version>
            </plugin>
        </plugins>
    </build>

    <!-- 多环境 Profile -->
    <profiles>
        <profile>
            <id>local</id>
            <activation><activeByDefault>true</activeByDefault></activation>
            <properties><env>local</env></properties>
        </profile>
        <profile>
            <id>prod</id>
            <properties><env>prod</env></properties>
        </profile>
    </profiles>
</project>
```

---

## 3️⃣ 常用命令

```powershell
# 编译
mvn compile

# 运行测试
mvn test

# 打包（编译+测试+打成 jar）
mvn package

# 清理
mvn clean

# 完整构建
mvn clean package

# 只下载依赖（检查能否解析）
mvn dependency:resolve

# 查看依赖树（排错用）
mvn dependency:tree

# 跳过测试打包
mvn clean package -DskipTests

# 指定环境打包
mvn clean package -Plocal
mvn clean package -Pprod

# 单个测试类
mvn test -Dtest=FileOperationToolTest

# 查看有效 POM（解决继承和变量后的完整配置）
mvn help:effective-pom

# mvnw（Maven Wrapper，你项目在用）
# 好处：不需要本地安装 Maven
.\mvnw.cmd clean compile          # Windows
./mvnw clean compile              # Mac/Linux

# 生成 Maven Wrapper
mvn wrapper:wrapper
```

---

## 4️⃣ 国内镜像配置（必须配！）

```xml
<!-- C:\Users\chenyouxin\.m2\settings.xml -->

<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
          http://maven.apache.org/xsd/settings-1.0.0.xsd">

    <mirrors>
        <!-- 阿里云镜像：国内下载快10倍 -->
        <mirror>
            <id>aliyun</id>
            <name>Aliyun Maven Mirror</name>
            <url>https://maven.aliyun.com/repository/public</url>
            <mirrorOf>central</mirrorOf>
        </mirror>
    </mirrors>

    <!-- JDK 17 默认配置 -->
    <activeProfiles>
        <activeProfile>jdk-17</activeProfile>
    </activeProfiles>

    <profiles>
        <profile>
            <id>jdk-17</id>
            <activation>
                <activeByDefault>true</activeByDefault>
                <jdk>17</jdk>
            </activation>
            <properties>
                <maven.compiler.source>17</maven.compiler.source>
                <maven.compiler.target>17</maven.compiler.target>
                <maven.compiler.compilerVersion>17</maven.compiler.compilerVersion>
            </properties>
        </profile>
    </profiles>
</settings>
```

---

## 5️⃣ 依赖冲突排查

```powershell
# 1. 查看依赖树
mvn dependency:tree

# 输出示例：
# [INFO] +- org.springframework.ai:spring-ai-alibaba-starter:jar:1.0.0-M6
# [INFO] |  +- org.springframework.ai:spring-ai-core:jar:1.0.0-M6
# [INFO] |  |  +- com.squareup.okhttp3:okhttp:jar:4.12.0        ← 间接依赖
# [INFO] |  +- com.alibaba:fastjson2:jar:2.0.45                ← 间接依赖

# 2. 排除冲突依赖（exclusions）
<dependency>
    <groupId>com.example</groupId>
    <artifactId>bad-lib</artifactId>
    <version>1.0.0</version>
    <exclusions>
        <exclusion>
            <groupId>com.old</groupId>
            <artifactId>old-version-lib</artifactId>
        </exclusion>
    </exclusions>
</dependency>

# 3. 锁定版本（dependencyManagement）
# 在父 POM 写死版本，子模块不写 version
```

---

## 6️⃣ 多模块项目结构

```
parent-project/                    ← 父项目（pom packaging）
├── pom.xml                        ← 父 pom，管理版本
├── module1/
│   └── pom.xml                    ← 子 pom，parent 指向父
└── module2/
    └── pom.xml

父 pom.xml：
<project>
    <groupId>io.github.chenyouxin8</groupId>
    <artifactId>parent</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>     ← 关键：pom 类型
    <modules>
        <module>module1</module>
        <module>module2</module>
    </modules>
    <dependencyManagement>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson2</artifactId>
            <version>2.0.45</version>
        </dependency>
    </dependencyManagement>
</project>

子 pom.xml：
<project>
    <parent>
        <groupId>io.github.chenyouxin8</groupId>
        <artifactId>parent</artifactId>
        <version>1.0.0</version>
    </parent>
    <artifactId>module1</artifactId>
    <!-- version 省略，继承父 POM -->
    <dependencies>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson2</artifactId>
            <!-- 不写 version -->
        </dependency>
    </dependencies>
</project>
```

---

## 7️⃣ scope 速查

| scope | 编译 | 运行 | 测试 | 典型场景 |
|-------|------|------|------|---------|
| `compile`（默认） | ✅ | ✅ | ✅ | 常规依赖 |
| `provided` | ✅ | ❌ | ✅ | JDK/容器提供，如 servlet-api |
| `runtime` | ❌ | ✅ | ✅ | JDBC 驱动 |
| `test` | ❌ | ❌ | ✅ | JUnit，仅测试代码 |
| `system` | ✅ | ❌ | ✅ | 系统路径（不推荐） |

---

## 8️⃣ 速查清单

```powershell
# 日常三件套
mvn clean compile                        # 清理+编译
mvn test                                 # 运行测试
mvn package -DskipTests                  # 打包（跳过测试）

# 排错
mvn dependency:tree                       # 查看依赖冲突
mvn dependency:resolve                   # 检查依赖能否解析
mvn spring-boot:run                     # 运行 Spring Boot

# 报错处理
# Could not resolve dependencies → 检查网络 + 镜像 + 坐标
# Maven source 1.5 no supported → source/target 改成 17
# Unable to access jar → 删 target/ 重新 mvn clean package
```

---

## ❓ 面试题

**Q: Maven 的依赖传递是什么？**
> A 依赖 B，B 依赖 C，则 A 自动获得 C（传递依赖）。`<optional>true</optional>` 阻止传递，`<exclusions>` 排除某个传递依赖。依赖冲突时用"最短路径优先"原则选择版本。

**Q: Maven 的聚合和继承是什么？**
> 聚合（`<modules>`）：父项目 `packaging=pom`，管理多子模块，一次命令构建全部。继承（子 POM 的 `<parent>`）：子模块继承父 POM 的 `<dependencyManagement>` 版本管理，避免版本不一致。

**Q: IDEA 里如何排查 Maven 依赖冲突？**
> Maven Helper 插件（Dependencies 面板，红色的是冲突）；或 `mvn dependency:tree -Dverbose`（显示被排除的版本）；右键冲突依赖 → Exclude 排除。
