# Maven · Git · IDEA 三剑客

> Maven 是**项目构建工具**（管理依赖、编译、打包）；Git 是**版本控制工具**（管理代码历史、协作）；IDEA 是**代码编辑器**（写代码、运行、调试）。三者是你 Java 开发的标准工作流。

---

## 1️⃣ Maven（项目构建）

### 1-1 核心概念

```
Maven 解决的问题：
1. 依赖管理：不用手动下载 jar，Maven 自动下载
2. 项目结构统一：所有 Java 项目用同一套目录结构
3. 构建自动化：编译→测试→打包，一条命令搞定

Maven 坐标（dependency 定位）：
  groupId      → 公司/组织名（倒过来写域名） io.github.chenyouxin8
  artifactId   → 项目/模块名              chen-ai-agent
  version      → 版本号                   1.0.0
  → 唯一确定一个 jar 包

Maven 仓库（去哪找 jar）：
  中央仓库（https://repo.maven.apache.org/maven2）→ 官方维护，公开免费
  阿里云镜像（https://maven.aliyun.com/repository/public）→ 国内加速
  私有仓库 → 公司内部自己搭建的
```

### 1-2 pom.xml 骨架（你的 chen-ai-agent）

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <!-- ===== 父 POM：统一管理版本号 ===== -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.x</version>
        <relativePath/>
    </parent>

    <!-- ===== 项目基本信息 ===== -->
    <groupId>io.github.chenyouxin8</groupId>
    <artifactId>chen-ai-agent</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>           <!-- 默认 jar，web 项目可以 war -->
    <name>chen-ai-agent</name>
    <description>AI 恋爱助手</description>

    <!-- ===== properties：统一版本号变量 ===== -->
    <properties>
        <java.version>17</java.version>
        <spring-ai.version>1.0.0-M6</spring-ai.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <!-- ===== dependencies：项目依赖 ===== -->
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
            <optional>true</optional>    <!-- optional=true 表示不会传递依赖 -->
        </dependency>

        <!-- 测试（JUnit 5） -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>          <!-- 只在 src/test 下可用 -->
        </dependency>
    </dependencies>

    <!-- ===== dependencyManagement：版本锁定（大型项目） ===== -->
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

    <!-- ===== 构建配置 ===== -->
    <build>
        <!-- 项目名+版本号作为 jar 包名 -->
        <finalName>${project.artifactId}-${project.version}</finalName>

        <!-- 资源目录配置（src/main/resources） -->
        <resources>
            <resource>
                <directory>src/main/resources</directory>
                <filtering>true</filtering>   <!-- 开启变量替换（${xxx}） -->
            </resource>
        </resources>

        <!-- 插件 -->
        <plugins>
            <!-- Spring Boot Maven 插件：打可执行 jar -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <!-- 不打包 Lombok 源码（否则其他项目引用时会报错） -->
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>

            <!-- Maven 编译插件（指定 Java 版本） -->
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

            <!-- Maven Surefire：运行单元测试 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.2.2</version>
            </plugin>
        </plugins>
    </build>

    <!-- ===== 多环境 Profile ===== -->
    <profiles>
        <profile>
            <id>local</id>
            <activation><activeByDefault>true</activeByDefault></activation>
            <properties>
                <env>local</env>
            </properties>
        </profile>
        <profile>
            <id>prod</id>
            <properties>
                <env>prod</env>
            </properties>
        </profile>
    </profiles>
</project>
```

### 1-3 Maven 常用命令

```powershell
# ===== 项目根目录下执行 =====

# 编译（编译 src/main/java → target/classes）
mvn compile

# 运行测试（src/test/java）
mvn test

# 打包（编译 + 测试 + 打成 jar/war）
mvn package

# 清理（删除 target/ 目录）
mvn clean

# 清理 + 重新编译 + 测试 + 打包（完整构建）
mvn clean package

# 只下载依赖（不编译，用于检查依赖是否能解析）
mvn dependency:resolve

# 查看依赖树（排错时用：冲突/版本问题）
mvn dependency:tree

# 跳过测试打包（快速打包，不需要等测试）
mvn clean package -DskipTests

# 指定环境打包
mvn clean package -Plocal    # 使用 application-local.yml
mvn clean package -Pprod     # 使用 application-prod.yml

# 只运行单个测试类
mvn test -Dtest=FileOperationToolTest

# 查看有效 POM（解决后的完整 POM，解决继承和变量）
mvn help:effective-pom

# ===== mvnw（Maven Wrapper，你项目在用）=====
# 好处：不需要本地安装 Maven，仓库里有就能用
# 用法和 mvn 完全一样，只是命令换成 .\mvnw
.\mvnw.cmd clean compile          # Windows
./mvnw clean compile              # Mac/Linux

# 生成 Maven Wrapper（项目里没有 mvnw 时用）
mvn wrapper:wrapper
```

### 1-4 Maven 国内镜像配置（必须配！）

```xml
<!-- ~/.m2/settings.xml（用户目录下） -->
<!-- C:\Users\chenyouxin\.m2\settings.xml -->

<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
          http://maven.apache.org/xsd/settings-1.0.0.xsd">

    <mirrors>
        <!-- 阿里云镜像：国内下载依赖快10倍 -->
        <mirror>
            <id>aliyun</id>
            <name>Aliyun Maven Mirror</name>
            <url>https://maven.aliyun.com/repository/public</url>
            <mirrorOf>central</mirrorOf>   <!-- 镜像中央仓库 -->
        </mirror>
    </mirrors>

    <!-- JDK 版本默认配置（不依赖项目指定） -->
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

### 1-5 依赖冲突排查

```powershell
# 1. 查看依赖树，找出冲突来源
mvn dependency:tree

# 输出示例（chen-ai-agent）：
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
        <!-- 排除传递过来的某个依赖 -->
        <exclusion>
            <groupId>com.old</groupId>
            <artifactId>old-version-lib</artifactId>
        </exclusion>
    </exclusions>
</dependency>

# 3. 锁定版本（dependencyManagement）
# 在父 POM 的 dependencyManagement 中写死版本
# 子模块继承时用 groupId + artifactId 即可，不写 version
```

### 1-6 多模块项目结构

```
parent-project/                    ← 父项目（pom packaging）
├── pom.xml                        ← 父 pom，管理版本
├── module1/                       ← 子模块1
│   └── pom.xml                    ← 子 pom，parent 指向父
└── module2/                       ← 子模块2
    └── pom.xml

父 pom.xml 写法：
<project>
    <groupId>io.github.chenyouxin8</groupId>
    <artifactId>parent</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>     ← 关键：pom 类型表示父项目
    <modules>
        <module>module1</module>   ← 声明子模块
        <module>module2</module>
    </modules>
    <dependencyManagement>
        <!-- 统一版本管理 -->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson2</artifactId>
            <version>2.0.45</version>
        </dependency>
    </dependencyManagement>
</project>

子 pom.xml 写法：
<project>
    <parent>
        <groupId>io.github.chenyouxin8</groupId>
        <artifactId>parent</artifactId>
        <version>1.0.0</version>
    </parent>
    <artifactId>module1</artifactId>
    <!-- version 可以省略，继承父 POM -->
    <dependencies>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson2</artifactId>
            <!-- 不写 version，父 POM 管理 -->
        </dependency>
    </dependencies>
</project>
```

---

## 2️⃣ Git（版本控制）

### 2-1 Git 核心概念

```
Git 的三种状态：
  modified（已修改）  →  working directory（工作区）
  staged（已暂存）    →  staging area / index（暂存区）
  committed（已提交）  →  local repository（本地仓库）

文件流转：
  工作区 → git add → 暂存区 → git commit → 本地仓库
                                            ↓
  ←────────────── git checkout / git reset ←

四个区域：
  Working Directory（工作区）：你正在编辑的文件夹
  Staging Area（暂存区）：git add 后的文件快照
  Local Repository（本地仓库）：.git 目录，git commit 后的历史
  Remote Repository（远程仓库）：GitHub / Gitee / GitLab

Git 分支：
  main / master      →  主分支（生产代码）
  develop            →  开发分支
  feature/xxx        →  功能分支
  hotfix/xxx         →  热修复分支
```

### 2-2 你项目 Git 操作全流程

```powershell
# ===== 初始化（已有仓库，跳过）=====
cd D:\projectt\chen-ai-agent
git init                    # 初始化仓库（创建 .git）
git remote -v              # 查看远程仓库地址
git remote add origin https://github.com/chenyouxin8/ai-agent.git  # 添加远程

# ===== 克隆（从远程下载到本地）=====
git clone https://github.com/chenyouxin8/ai-agent.git
git clone https://github.com/chenyouxin8/ai-agent.git --depth 1   # 浅克隆，只下载最新版本

# ===== 查看状态 ======
git status                  # 查看哪些文件变了
git status -s              # 简洁模式
git diff                    # 查看工作区的具体改动（未暂存）
git diff --staged          # 查看暂存区的改动（已 add 未 commit）
git diff HEAD              # 对比工作区和最新 commit

# ===== 暂存与提交 ======
git add filename.txt       # 暂存单个文件
git add src/               # 暂存整个目录
git add .                  # 暂存所有改动（包括新文件，慎用！）
git add -p                 # 交互式暂存（逐块选择）
git reset filename.txt    # 取消暂存（文件还在，只是移出暂存区）
git reset HEAD filename.txt # 同上，HEAD 表示当前版本

git commit -m "feat: 新增用户登录功能"    # 提交（写清楚改动内容）
git commit -am "fix: 修复空指针异常"       # 暂存所有已跟踪文件并提交（不包含新文件）
git commit --amend         # 修改上一次的 commit 信息（还没 push 时用）
git commit --no-verify -m "..."  # 跳过 pre-commit 钩子

# ===== 查看历史 ======
git log                     # 完整提交历史
git log --oneline           # 每行一条，精简模式
git log --oneline -10       # 最近10条
git log --graph --oneline   # 分支图形化
git log -p filename.txt    # 查看某个文件的改动历史
git log --author="chen"     # 只看某个人的提交
git log --since="2024-01-01" --until="2024-12-31"  # 时间范围
git show <commit-id>       # 查看某次提交的具体内容
git blame filename.txt     # 逐行看是谁改的

# ===== 撤销与回退 ======
# 工作区撤销（还没 add）
git checkout -- filename.txt    # 丢弃工作区的改动（危险！）
git restore filename.txt       # 同上，Git 2.23+ 推荐用法

# 暂存区撤销（已经 add）
git reset HEAD filename.txt    # 移出暂存区

# 回退到某个版本（修改 HEAD 指针）
git reset --soft HEAD~1        # 回退1个 commit，保留改动在暂存区
git reset --mixed HEAD~1       # 回退1个 commit，保留改动在工作区（默认）
git reset --hard HEAD~1        # 回退1个 commit，丢弃所有改动（危险！）
git reset --hard abc1234       # 回退到某个 commit ID

# ===== 分支操作 ======
git branch                   # 查看本地分支（* 表示当前分支）
git branch -a                # 查看所有分支（包括远程）
git branch new-feature       # 创建新分支
git checkout new-feature     # 切换到新分支
git switch new-feature       # 同上，Git 2.23+ 推荐
git checkout -b new-feature  # 创建 + 切换 一步到位
git switch -c new-feature    # 同上

git branch -d new-feature    # 删除分支（已合并才允许）
git branch -D new-feature    # 强制删除分支
git branch -m old-name new-name  # 重命名分支

# ===== 合并分支 ======
git checkout main            # 先切回主分支
git merge new-feature        # 把 new-feature 合并进来
# 合并冲突时：
# 1. 打开冲突文件
# 2. 手动解决冲突（删掉 <<< === >>> 标记）
# 3. git add filename.txt
# 4. git commit -m "merge: 解决冲突"

# ===== Rebase（变基，整理提交历史）=====
# 场景：feature 分支基于 old main，想更新到 new main
git checkout feature
git rebase main
# 结果：feature 的提交"复制"到了 new main 上面，提交历史是一条直线
# ⚠️ 不要 rebase 已经 push 到远程的提交！

# ===== 暂存（临时保存工作区）=====
git stash                    # 暂存当前工作区（不提交）
git stash push -m "未完成的新功能"   # 暂存并加说明
git stash list               # 查看暂存列表
git stash pop                # 恢复暂存 + 删除暂存
git stash apply              # 恢复暂存（保留暂存记录）
git stash drop               # 删除某个暂存

# ===== 标签（标记版本）=====
git tag v1.0.0               # 创建轻量标签
git tag -a v1.0.0 -m "第一个正式版本"  # 创建附注标签
git tag                     # 查看所有标签
git show v1.0.0             # 查看标签详情
git push origin v1.0.0      # 推送标签到远程
git push origin --tags     # 推送所有标签
```

### 2-3 GitHub 协作（你和你的仓库）

```powershell
# ===== 远程操作 ======
git fetch origin             # 拉取远程最新状态（不合并）
git pull origin main        # 拉取 + 合并（相当于 fetch + merge）
git pull --rebase origin main  # 拉取 + 变基（历史更干净）
git push origin main         # 推送本地提交到远程
git push -u origin main     # 首次推送，设置上游分支
git push --force            # 强制推送（⚠️ 危险！只在自己分支用）
git push origin --delete old-branch  # 删除远程分支

# ===== 远程分支操作 ======
git checkout -b local-name origin/remote-name   # 拉取远程分支并切换
git push origin local-name:remote-name          # 推送本地分支为远程新名字

# ===== GitHub 常用操作 ======
# 1. Fork（fork 别人的仓库到自己 GitHub）
# 2. Clone 自己的 fork 到本地
# 3. 添加上游仓库
git remote add upstream https://github.com/original/repo.git
# 4. 同步上游更新
git fetch upstream
git merge upstream/main
# 5. 发起 Pull Request（PR）

# ===== SSH Key 配置（不用每次输入密码）=====
# 1. 生成 SSH Key
ssh-keygen -t ed25519 -C "your_email@example.com"
# 2. 查看公钥
cat ~/.ssh/id_ed25519.pub
# 3. 复制公钥，粘贴到 GitHub → Settings → SSH Keys
# 4. 测试连接
ssh -T git@github.com
```

### 2-4 .gitignore（重要！）

```gitignore
# ===== 你项目已有的 .gitignore =====

# 编译产物
target/
*.class
*.jar
*.war

# IDE
.idea/
*.iml
.vscode/

# 本地配置（包含 API Key！）
application-local.yml
application-local.properties
*.local.yml
*.local.properties
.env

# 日志
*.log
logs/

# 系统文件
.DS_Store
Thumbs.db

# Maven
.mvn/wrapper/maven-wrapper.jar

# 运行时生成的文件
data/
*.txt                       # 如果 data/chat-memory/*.txt 要忽略
```

```gitignore
# ===== .gitignore 高级用法 =====

# 忽略但保留目录
# （创建空 .gitkeep 文件在目录下）
# data/
# !data/.gitkeep

# 忽略所有 .log 文件
*.log

# 不忽略某文件（优先级高于上面的规则）
!error.log

# 忽略某目录下的某类文件
build/tmp/**/*.java

# 忽略已跟踪文件的改动（暂存但不提交）
git update-index --assume-unchanged filename.txt
git update-index --no-assume-unchanged filename.txt
```

### 2-5 提交信息规范（Conventional Commits）

```
格式：<type>(<scope>): <description>

type（类型）：
  feat     → 新功能
  fix      → 修复 bug
  docs     → 文档改动
  style    → 格式/风格（不影响代码含义）
  refactor → 重构（不是新功能也不是 bug 修复）
  perf     → 性能优化
  test     → 测试相关
  chore    → 构建/工具/依赖更新

scope（范围，可选）：
  模块名，如: love, rag, agent, tools

examples：
  feat(love): 新增恋爱专家对话接口
  fix(terminal): 修复命令白名单校验逻辑
  docs: 更新 README 使用说明
  refactor(agent): 重构 ReAct Agent 思考循环
  chore(deps): 升级 Spring AI 到 1.0.0-M6

⚠️ 不要的提交信息：
  ❌ "update" / "fix bug" / "xxx" / "asdfgh"
  ✅ "feat: 新增登录功能" / "fix: 修复空指针异常"
```

---

## 3️⃣ IDEA（IntelliJ IDEA）

### 3-1 你项目的 IDEA 配置

```
D:\projectt\chen-ai-agent

推荐 IDEA 配置：
  IDEA 版本：2024.x（支持 Java 21）
  插件：
    - Lombok（自动生成 getter/setter/constructor）
    - Maven Helper（依赖冲突分析）
    - GitToolBox（Git 信息显示）
    - Translation（翻译，不会英语变量名也能看懂）
```

### 3-2 IDEA 常用快捷键（Windows）

```text
===== 导航与搜索 =====
Ctrl + N              搜索类名
Ctrl + Shift + N      搜索文件名
Ctrl + Alt + Shift + N 搜索符号（方法名/变量名）
Ctrl + F              当前文件查找
Ctrl + R              当前文件替换
Ctrl + Shift + F      全局搜索
Ctrl + Shift + R      全局替换

Ctrl + E              最近打开的文件
Ctrl + Tab            切换标签页
Ctrl + W              关闭当前标签
Ctrl + Shift + E      最近修改的文件

F7 / Shift + F7       下一个/上一个高亮

===== 代码编辑 =====
Alt + Enter            快速修复（最常用！）
Ctrl + D              复制当前行
Ctrl + Y              删除当前行
Ctrl + X              剪切当前行
Ctrl + /              注释/取消注释（//）
Ctrl + Shift + /      注释/取消注释（/**/）
Ctrl + Alt + L        格式化代码（自动缩进）
Ctrl + Alt + O        优化导入（删除未使用的 import）
Ctrl + Shift + U      大小写切换
Ctrl + Shift + V      剪切板历史

Alt + Shift + 上/下   移动当前行
Ctrl + Shift + 上/下  移动当前方法

Alt + Insert          生成代码（getter/setter/constructor/重写方法）
Ctrl + O              重写父类方法
Ctrl + I              实现接口方法

Ctrl + B              跳转到定义（变量/方法/类）
Ctrl + Alt + B        跳转到实现
Ctrl + U              跳转到父类
Ctrl + G              跳转到指定行
Ctrl + H              查看类结构（Hierarchy）

===== 重构 =====
Shift + F6            重命名（变量/方法/类，全局改名）
Ctrl + Alt + V        提取变量
Ctrl + Alt + M        提取方法
Ctrl + Alt + C        提取常量
Ctrl + Alt + P        提取参数

===== 运行与调试 =====
Shift + F10           运行当前程序
Shift + F9            调试当前程序
Ctrl + Shift + F10    运行选中的代码（测试方法）
F8                    单步跳过
F7                    单步进入
Shift + F8           跳出方法
Ctrl + Shift + F8    查看断点列表
Alt + F9             运行到光标位置

===== Maven =====
双击 pom.xml 的 lifecycle → 运行对应命令
右边 Maven 面板 → 点开 Lifecycle → 双击 compile / package / test

===== Git =====
Ctrl + K              提交（Commit）
Ctrl + Shift + K      推送（Push）
Ctrl + Alt + Z        撤销当前文件的改动（Revert）
Alt + `              Git 操作菜单（commit/push/pull/branch/merge）
```

### 3-3 IDEA Maven 面板使用

```
IDEA 右侧 Maven 面板：
chen-ai-agent → Lifecycle
├── clean         清理 target
├── validate      验证项目
├── compile       编译
├── test          运行测试
├── package       打包
└── install       安装到本地仓库（其他项目可以引用）

chen-ai-agent → Plugins
├── spring-boot:spring-boot     运行 Spring Boot
├── maven-compiler:compile      编译
├── maven-surefire:test          运行测试
└── dependency:tree             查看依赖树

chen-ai-agent → Dependencies
└── 树形展示所有依赖（可以搜索、展开、查看冲突）
```

### 3-4 IDEA 运行 Spring Boot 项目

```text
方式1：右键启动类 → Run
方式2：IDEA Maven 面板 → Plugins → spring-boot → spring-boot:run
方式3：Terminal 里运行
  cd D:\projectt\chen-ai-agent
  .\mvnw.cmd spring-boot:run

打断点：代码行号左边点一下（红色圆点）
调试：Shift + F9，然后 F7/F8 逐行走
```

### 3-5 IDEA 配置（JVM / JDK / 编码）

```
File → Project Structure → Project
  Project SDK:    JDK 21  ← 设置项目 JDK 版本
  Project language level: 17
  Project compiler output: 任意，如 D:\projectt\chen-ai-agent\out

File → Settings → Build, Execution, Deployment → Build Tools → Maven
  Maven home:     C:\Users\chenyouxin\.m2   ← 本地 Maven 仓库
  Settings file:   C:\Users\chenyouxin\.m2\settings.xml
  Local repository: C:\Users\chenyouxin\.m2\repository

File → Settings → Editor → File Encodings
  Global encoding:     UTF-8
  Project encoding:    UTF-8
  Properties files:    UTF-8
  ✓ ✅ Transparent native-to-ascii conversion（中文不乱码）

File → Settings → Build, Execution, Deployment → Compiler → Java Compiler
  Target bytecode version: 17

File → Settings → Editor → General → Auto Import
  ✓ Add unambiguous imports on the fly
  ✓ Optimize imports on the fly
```

### 3-6 IDEA 调试技巧

```java
// ===== 条件断点 =====
在断点上右键 → Condition
填写条件表达式，如：userId > 100
这样只有条件满足时才会停下

// ===== 变量监视 =====
在 Variables 窗口右键 → Add to Watches
把变量加到 Watch 窗口，持续监视

// ===== 方法断点 =====
在方法入口行号上打断点
右键断点 → Method Breakpoint
可以捕获方法的入参和返回值

// ===== 求值表达式（Evaluate）=====
Alt + F8 打开 Evaluate 表达式窗口
可以输入任意 Java 代码实时查看结果

// ===== 回退到上一步 =====
调试时，右键变量 → Set Value
改掉变量的值，然后重新走一遍逻辑
```

---

## 4️⃣ 对比表格

### 4-1 Maven 命令对应 IDEA 操作

| Maven 命令 | IDEA 操作 | 场景 |
|-----------|---------|------|
| `mvn compile` | IDEA 自动编译 / Ctrl+Shift+F9 | 编译检查错误 |
| `mvn test` | 运行测试类旁边绿色按钮 | 跑单元测试 |
| `mvn package` | Maven 面板 → Lifecycle → package | 打包 jar |
| `mvn clean` | Maven 面板 → Lifecycle → clean | 清理旧文件 |
| `mvn dependency:tree` | Maven 面板 → Dependencies | 查看/排查依赖 |
| `mvn help:effective-pom` | Maven 面板 → pom.xml 右键 → Show Effective POM | 看完整配置 |

### 4-2 Git 三棵树

| 区域 | 命令 | 说明 |
|------|------|------|
| 工作区（Working Tree） | `git status` | 你正在编辑的 |
| 暂存区（Staging） | `git add` / `git reset` | 准备提交的文件快照 |
| 仓库（Repository） | `git commit` | 已提交的版本历史 |

### 4-3 Git 工作流（Git Flow）

```
main（生产） ───────────────────────────────────────────▶
                ↑                           ↑
                │      ↑_______________↑   │
                │ merge│               │   │ merge
                │      ↓               │   │
hotfix/xxx ────▶   develop ←────────────────┘
                     ↑     ↑           │
                     │     │     ↑_____│____↑
                     │     │     │           |
                feature/  refactor/   bugfix/
                login      optimize    sidebar
（从 develop 创建）  （从 develop 创建）  （从 develop 创建）
合并回 develop 后
删除 feature 分支
```

### 4-4 IDEA 与 VS Code 对比

| 维度 | IDEA | VS Code |
|------|------|---------|
| 定位 | Java 专业 IDE | 通用代码编辑器 |
| 智能提示 | 🔥 强（懂 Java 语义） | 一般（靠插件） |
| 重构 | 🔥 内置，强大 | 依赖插件 |
| 调试 | 🔥 功能完整 | 依赖插件 |
| 启动速度 | 慢（功能多） | 快 |
| 内存占用 | 高（~1GB） | 低（~200MB） |
| 价格 | 社区版免费/终极版收费 | 免费 |
| Java 项目 | **强烈推荐 IDEA** | 可用但不推荐 |
| 多语言 | 一般 | **强烈推荐** |

---

## 5️⃣ 速查清单

### 5-1 Maven 速查

```powershell
# 常用命令
mvn clean compile                        # 清理+编译
mvn test                                 # 运行测试
mvn package -DskipTests                  # 打包（跳过测试）
mvn dependency:tree > tree.txt           # 导出依赖树
mvn dependency:analyze                   # 分析未使用/可传递依赖
mvn spring-boot:run                      # 运行 Spring Boot
mvn help:effective-pom > pom.xml         # 查看有效 POM

# 常见报错
# Could not resolve dependencies
#   → 检查网络 + 阿里云镜像是否配置 + dependency 坐标是否正确
# Maven source 1.5 no supported
#   → maven-compiler-plugin 的 source/target 改成 17
# Unable to access jar
#   → jar 打包可能有问题，删 target/ 重新 mvn clean package
```

### 5-2 Git 速查

```powershell
# 日常流程（你项目的标准操作）
git status                               # 查看改动
git add .                                # 暂存所有
git commit -m "feat: xxx"               # 提交
git push origin main                     # 推送

# 解决冲突
git fetch origin                         # 拉取远程最新
git merge origin/main                     # 合并（如果有冲突）
# 手动解决 → git add → git commit → git push

# 紧急修复（hotfix）
git checkout -b hotfix/xxx main          # 从 main 创建热修分支
# 修复 → git add → git commit → git push
# 合并回 main：git checkout main → git merge hotfix/xxx → git push

# 查看远程仓库
git remote -v                            # 查看地址
git remote set-url origin NEW_URL        # 修改远程地址
```

### 5-3 IDEA 速查

```text
# 最常用 10 个快捷键
Alt + Enter        快速修复
Ctrl + D           复制行
Ctrl + Y           删除行
Ctrl + /           注释
Ctrl + F           查找
Ctrl + R           替换
Ctrl + B           跳转定义
Ctrl + E           最近文件
Ctrl + K           Git 提交
Ctrl + Shift + F10 运行

# 快速打开
Ctrl + Shift + A   搜索 IDEA 命令（不知道快捷键时用）
Ctrl + N           搜索类
双击 Shift          搜索所有（类/文件/符号）
```

---

## ❓ 常见面试题

**Q1: Maven 的依赖传递是什么？**
> 如果 A 依赖 B，B 依赖 C，那么 A 会自动获得 C（传递依赖）。可以用 `<optional>true</optional>` 阻止传递，用 `<exclusions>` 排除某个传递依赖。依赖冲突时，Maven 用"最短路径优先"原则选择版本：路径短的优先。

**Q2: Git 的 merge 和 rebase 区别？**
> `git merge` 把两个分支的提交历史合并，保留真实分叉历史，适合协作分支。`git rebase` 把当前分支的提交"复制"到目标分支之后，历史变成一条直线，更干净，但会改变提交历史。规则：**已经 push 到远程的分支不要 rebase**。

**Q3: Git 怎么撤销已经 push 的提交？**
> `git revert HEAD`（安全撤销，创建新提交来反向之前的改动，不会改变历史）。或者 `git reset --hard HEAD~n` + `git push --force`（危险，改历史）。

**Q4: Maven 的 scope 有哪些？**
> `compile`（默认，编译/运行/测试都有效）、`provided`（JDK/容器提供，编译时用，打包时不包含，如 servlet-api）、`runtime`（运行/测试用，编译时不需要，如 JDBC 驱动）、`test`（只在测试时有效，如 JUnit）、`system`（系统路径，不推荐）。

**Q5: Git 如何解决合并冲突？**
> 1. `git pull` 时报 CONFLICT；2. 打开冲突文件，搜索 `<<<<<<< HEAD` 标记；3. 手动保留想要的代码，删掉 `=======` 等标记；4. `git add` 暂存；5. `git commit` 完成合并。

**Q6: Maven 的聚合和继承是什么？**
> 聚合（`<modules><module>child</module></modules>`）：父项目用 `packaging=pom`，管理多个子模块，一次命令构建全部。继承（子 POM 的 `<parent>`）：子模块继承父 POM 的 `<dependencyManagement>` 版本管理，避免版本不一致。

**Q7: IDEA 里如何排查 Maven 依赖冲突？**
> Maven Helper 插件（点开 pom.xml → Dependencies → 红色的是冲突）；或者 Terminal 执行 `mvn dependency:tree -Dverbose`（verbose 会显示被排除的版本）；右键冲突依赖 → Exclude 排除。

**Q8: Git 的 HEAD 是什么？**
> HEAD 是指向当前分支最新提交的指针（书签）。`HEAD~1` = 当前提交的父提交，`HEAD~2` = 祖父提交。切换分支时 HEAD 跟着移动，`git reset` 实际上是移动 HEAD 指针。

## 📊 学习状态

- [x] Maven pom.xml 结构与依赖管理
- [x] Maven 常用命令与多环境
- [x] Maven 国内镜像配置
- [x] 依赖冲突排查
- [x] Git 三种状态与四个区域
- [x] Git 提交/分支/合并/回退
- [x] GitHub SSH Key 配置
- [x] .gitignore 规则
- [x] Conventional Commits 提交规范
- [x] IDEA 常用快捷键
- [x] IDEA Maven 面板使用
- [x] IDEA 配置（JDK/编码/Maven）
- [ ] Git Flow 团队协作流程

## 🐛 踩坑记录

- **Maven 下载慢**：没配阿里云镜像，第一次构建可能等很久；配好后删 `.m2/repository` 重下
- **git add . 误提交**：加了不该加的文件（target/、*.local.yml）；提交前 `git status` 先看一眼
- **.gitignore 不生效**：文件已经被 git track 后再写入 .gitignore 无效；需 `git rm --cached filename` 清除跟踪
- **IDEA 编译版本不对**：pom.xml 设为 Java 17 但 IDEA SDK 是 21；File → Project Structure → SDK 保持一致
- **git commit 信息随便写**：不规范的 commit 信息在协作时会被嫌弃；用 Conventional Commits 格式
- **git push --force**：强推会覆盖远程历史；只在个人分支用，不要在 main/develop 上用
- **IDEA Lombok 插件**：装了 Lombok 依赖但没装插件，代码会报红（找不到 getter/setter）；装 Lombok 插件并启用 Annotation Processing
- **git merge 产生冲突**：手动解决后一定要删掉 `<<<<<<<` 等标记，否则文件还是冲突状态
- **Maven 仓库路径中文**：`.m2` 放在中文路径下可能出问题；移到纯英文路径
- **git reset --hard 丢失代码**：用前确认工作区已提交或 stash；建议用 `git stash` 而不是硬回退
