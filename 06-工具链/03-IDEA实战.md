# IDEA 实战

> IDEA（IntelliJ IDEA）是 Java 开发最专业的代码编辑器，核心功能：智能提示、代码补全、重构、调试、运行。

---

## 1️⃣ 你项目的 IDEA 配置

```
D:\projectt\chen-ai-agent

推荐配置：
  IDEA 版本：2024.x（支持 Java 21）
  推荐插件：
    - Lombok（自动生成 getter/setter）
    - Maven Helper（依赖冲突分析）
    - GitToolBox（Git 信息显示）
    - Translation（翻译）
```

---

## 2️⃣ 常用快捷键（Windows）

### 导航与搜索

| 快捷键 | 功能 |
|--------|------|
| `Ctrl + N` | 搜索类名 |
| `Ctrl + Shift + N` | 搜索文件名 |
| `Ctrl + Shift + F` | 全局搜索 |
| `Ctrl + F` | 当前文件查找 |
| `Ctrl + R` | 当前文件替换 |
| `Ctrl + E` | 最近打开的文件 |
| `Ctrl + Tab` | 切换标签页 |
| `Ctrl + B` | 跳转到定义 |
| `Ctrl + Alt + B` | 跳转到实现 |
| `Ctrl + G` | 跳转到指定行 |
| `Ctrl + H` | 查看类结构（Hierarchy） |

### 代码编辑

| 快捷键 | 功能 |
|--------|------|
| `Alt + Enter` | **快速修复（最常用！）** |
| `Ctrl + D` | 复制当前行 |
| `Ctrl + Y` | 删除当前行 |
| `Ctrl + /` | 注释/取消注释（//） |
| `Ctrl + Shift + /` | 注释/取消注释（/**/） |
| `Ctrl + Alt + L` | 格式化代码（自动缩进） |
| `Ctrl + Alt + O` | 优化导入（删除无用 import） |
| `Alt + Insert` | 生成代码（getter/setter/constructor） |
| `Ctrl + O` | 重写父类方法 |
| `Ctrl + I` | 实现接口方法 |
| `Alt + Shift + 上/下` | 移动当前行 |

### 重构

| 快捷键 | 功能 |
|--------|------|
| `Shift + F6` | 重命名（变量/方法/类，全局改名） |
| `Ctrl + Alt + V` | 提取变量 |
| `Ctrl + Alt + M` | 提取方法 |
| `Ctrl + Alt + C` | 提取常量 |
| `Ctrl + Alt + P` | 提取参数 |

### 运行与调试

| 快捷键 | 功能 |
|--------|------|
| `Shift + F10` | 运行当前程序 |
| `Shift + F9` | 调试当前程序 |
| `Ctrl + Shift + F10` | 运行选中代码（测试方法） |
| `F8` | 单步跳过 |
| `F7` | 单步进入 |
| `Shift + F8` | 跳出方法 |
| `Ctrl + Shift + F8` | 查看断点列表 |
| `Alt + F9` | 运行到光标位置 |

### Maven 和 Git

| 快捷键 | 功能 |
|--------|------|
| `Ctrl + K` | Git 提交（Commit） |
| `Ctrl + Shift + K` | Git 推送（Push） |
| `Ctrl + Alt + Z` | 撤销当前文件改动（Revert） |
| `Alt + 反引号` | Git 操作菜单 |

---

## 3️⃣ Maven 面板使用

```
IDEA 右侧 Maven 面板 → chen-ai-agent

Lifecycle：
├── clean         清理 target
├── compile       编译
├── test          运行测试
├── package       打包
└── install       安装到本地仓库

Plugins：
├── spring-boot:spring-boot     运行 Spring Boot
├── maven-compiler:compile      编译
├── maven-surefire:test        运行测试
└── dependency:tree            查看依赖树

Dependencies：树形展示所有依赖（可搜索/查看冲突）
```

---

## 4️⃣ 运行 Spring Boot 项目

```
方式1：右键启动类 ChenAiAgentApplication → Run
方式2：Maven 面板 → Plugins → spring-boot → spring-boot:run
方式3：Terminal 运行
  cd D:\projectt\chen-ai-agent
  .\mvnw.cmd spring-boot:run

打断点：代码行号左边点一下（红色圆点）
调试：Shift + F9，然后 F7/F8 逐行
```

---

## 5️⃣ IDEA 配置

```
File → Project Structure → Project
  Project SDK:          JDK 21
  Project language level: 17
  Project compiler output: D:\projectt\chen-ai-agent\out

File → Settings → Build, Execution, Deployment → Build Tools → Maven
  Maven home:     C:\Users\chenyouxin\.m2
  Settings file:   C:\Users\chenyouxin\.m2\settings.xml
  Local repository: C:\Users\chenyouxin\.m2\repository

File → Settings → Editor → File Encodings
  Global encoding:     UTF-8
  Project encoding:    UTF-8
  Properties files:    UTF-8
  ✅ Transparent native-to-ascii conversion（中文不乱码）

File → Settings → Build, Execution, Deployment → Compiler → Java Compiler
  Target bytecode version: 17

File → Settings → Editor → General → Auto Import
  ✅ Add unambiguous imports on the fly
  ✅ Optimize imports on the fly
```

---

## 6️⃣ 调试技巧

```java
// 条件断点：在断点上右键 → Condition
// 填写条件表达式，如：userId > 100
// 这样只有条件满足时才会停下

// 变量监视：Variables 窗口 → 右键 → Add to Watches
// 把变量加到 Watch 窗口，持续监视

// 方法断点：在方法入口打断点 → 右键 → Method Breakpoint
// 捕获方法入参和返回值

// 求值表达式：Alt + F8
// 输入任意 Java 代码，实时查看结果

// 回退变量值：调试时右键变量 → Set Value
// 改掉变量的值，然后重新走一遍逻辑
```

---

## 7️⃣ 与 VS Code 对比

| 维度 | IDEA | VS Code |
|------|------|---------|
| 定位 | Java 专业 IDE | 通用代码编辑器 |
| 智能提示 | 🔥 强（懂 Java 语义） | 一般（靠插件） |
| 重构 | 🔥 内置强大 | 依赖插件 |
| 调试 | 🔥 功能完整 | 依赖插件 |
| 启动速度 | 慢（功能多） | 快 |
| 内存占用 | 高（~1GB） | 低（~200MB） |
| 价格 | 社区版免费 | 免费 |
| Java 项目 | **强烈推荐** | 可用但不推荐 |
| 多语言 | 一般 | **强烈推荐** |

---

## 8️⃣ 速查清单

```text
# 最常用 10 个快捷键
Alt + Enter        快速修复
Ctrl + D           复制行
Ctrl + Y           删除行
Ctrl + /           注释
Ctrl + F           查找
Ctrl + B           跳转定义
Ctrl + E           最近文件
Ctrl + K           Git 提交
Ctrl + Shift + F10 运行
Ctrl + Alt + L     格式化

# 快速打开
Ctrl + Shift + A   搜索 IDEA 命令（不知道快捷键时用）
Ctrl + N           搜索类
双击 Shift          搜索所有（类/文件/符号）
```

---

## ❓ 面试题

**Q: IDEA 里如何排查 Maven 依赖冲突？**
> Maven Helper 插件（Dependencies 面板，红色的是冲突）；或 Terminal 执行 `mvn dependency:tree -Dverbose`（显示被排除的版本）；右键冲突依赖 → Exclude 排除。

**Q: IDEA Lombok 插件没装会有什么现象？**
> 代码会报红（找不到 getter/setter/constructor），但编译能通过。解决：安装 Lombok 插件并启用 Annotation Processing（Settings → Build → Compiler → Annotation Processors → Enable）。
