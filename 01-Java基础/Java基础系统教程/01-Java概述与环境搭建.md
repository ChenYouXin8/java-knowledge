---
tags:
  - Java
  - 教程书
  - 第一章
created: 2026-09-02
---

# 第 1 章 Java 概述与环境搭建

## 本章目标
- 说清 Java 是什么、JVM/JDK/JRE 的区别
- 在自己电脑上装好 JDK 并配好环境变量
- 用命令行和 IDEA 各跑通第一个程序
- 理解 Java 程序"一次编写，到处运行"的原理

---

## 1.1 Java 是什么，能做什么

Java 是一门 1995 年由 Sun 公司（后被 Oracle 收购）推出的、**面向对象**的编程语言，长期占据后端开发主流。

它主要用来做：
- **后端服务 / Web 系统**：银行、电商、企业管理系统（配合 Spring Boot，正是你后面的主战场）
- **安卓开发**：安卓 App 早期主力语言（现在与 Kotlin 并存）
- **大数据**：Hadoop、Spark、Flink 都跑在 JVM 上
- **中间件 / 分布式系统**：消息队列、微服务

Java 最大的特点是 **"一次编写，到处运行"（Write Once, Run Anywhere）**，靠的就是下一节的 JVM。

> 版本说明：Java 8 是国内存量最广的版本；**17 和 21 是当前 LTS（长期支持）版本**，新项目建议直接用 21。本书代码在 17/21 下均可运行。

---

## 1.2 JVM、JDK、JRE 到底什么关系（必背）

这是面试第一题，也是理解 Java 的钥匙。

```
JDK（Java 开发工具包，开发者装这个）
├── 开发工具：javac(编译器)、java(运行器)、javadoc、调试工具 jdb ...
└── JRE（Java 运行环境，只想"运行"程序的机器装这个）
    ├── 核心类库（rt.jar / java.base 模块：String、集合等现成的类）
    └── JVM（Java 虚拟机，真正执行字节码的程序）
```

一句话记忆：
- **JVM**：虚拟机，负责把 `.class` 字节码翻译成当前操作系统能执行的机器码。**跨平台靠它**。
- **JRE** = JVM + 运行所需的核心类库。只运行不开发，装它即可。
- **JDK** = JRE + 编译/调试等开发工具。**我们写代码，装 JDK。**

> 从 JDK 11 开始，Oracle 不再单独提供 JRE，理解为"JDK 已包含运行能力"即可。

为什么能跨平台？因为 `.java` 源码被编译成与操作系统无关的**字节码 `.class`**，再由"各平台各自实现的 JVM"去解释/编译执行。Windows 的 JVM、Linux 的 JVM 都认识同一份字节码。

---

## 1.3 安装 JDK 并配置环境变量（Windows）

1. 下载 JDK 21（推荐 **Eclipse Temurin / 亚马逊 Corretto / Oracle JDK** 任一发行版），默认安装，记住安装路径，例如 `C:\Program Files\Java\jdk-21`。
2. 配置环境变量（此电脑 → 属性 → 高级系统设置 → 环境变量）：
   - 新建**系统变量** `JAVA_HOME`，值为 JDK 安装目录（**不要带 bin**）。
   - 编辑**系统变量 `Path`**，新增一行 `%JAVA_HOME%\bin`。
3. 验证：打开**新的**命令行窗口（旧窗口读不到新变量），输入：

```bash
java -version
javac -version
```

都能打印出版本号（如 `openjdk version "21.x.x"`）即成功。

> 踩坑：`java` 能用但 `javac` 提示"不是内部命令"，99% 是 Path 没配对或没重开命令行窗口。

---

## 1.4 第一个程序 HelloWorld

新建一个纯文本文件，命名为 `HelloWorld.java`（**文件名必须和 public 类名完全一致，大小写敏感**）：

```java
public class HelloWorld {
    // main 方法是程序入口，固定写法
    public static void main(String[] args) {
        System.out.println("Hello, Java!");
    }
}
```

在该文件所在目录打开命令行，分两步：

```bash
javac HelloWorld.java     # 1. 编译：生成 HelloWorld.class 字节码
java HelloWorld           # 2. 运行：注意不要写 .class 后缀
```

运行结果：

```
Hello, Java!
```

逐行解释：
- `public class HelloWorld`：定义一个公共类，类名 HelloWorld。
- `public static void main(String[] args)`：**主方法，程序的唯一入口**，签名必须长这样。
- `System.out.println(...)`：向控制台打印一行并换行（`print` 不换行）。
- 每条语句以**英文分号 `;`** 结尾。

---

## 1.5 一个 Java 程序是怎么跑起来的（流程图）

```
HelloWorld.java  --javac 编译-->  HelloWorld.class(字节码)
                                        |
                            交给 JVM：类加载 → 校验 → 解释/JIT 编译
                                        |
                                   操作系统执行 → 控制台输出
```

关键点：`.java` 给人看，`.class` 给 JVM 看。你真正交付/部署的通常是编译后的 class 或打成的 jar 包。

---

## 1.6 用 IDEA 创建第一个项目（实际工作都用它）

命令行是为了理解原理，真实开发用 IntelliJ IDEA：

1. New Project → 选 JDK 21 → 选构建系统（初学先选 `普通 Java / 不选 Maven`，后面工具链章节再学 Maven）。
2. 在 `src` 目录右键 New → Java Class，输入 `HelloWorld`。
3. 写好 main 方法，点击方法左侧绿色三角 ▶ → Run。
4. 下方 Run 窗口看到输出即成功。

IDEA 快捷键先记两个：`psvm` + Tab 自动生成 main 方法；`sout` + Tab 自动生成打印语句。

---

## 1.7 工程实践与踩坑（借鉴 Effective Java 的思路）

- **一个文件只写一个 public 类，且文件名 = public 类名**，否则编译报错。
- Java **大小写敏感**：`System` 写成 `system` 直接报错。
- 所有括号、分号都用**英文半角**；中文标点是新手最常见报错来源。
- 代码缩进用 4 个空格，从第一天就养成规范习惯。

---

## 本章小结
- JDK ⊃ JRE ⊃ JVM；开发装 JDK，跨平台靠 JVM 执行字节码。
- Java 流程：`.java --javac--> .class --java(JVM)--> 结果`。
- 文件名与 public 类名一致，main 方法是固定入口。

## 动手练习
1. 装好 JDK，命令行打印出 `java -version`。
2. 输出三行：你的名字、你为什么学 Java、今天日期。
3. 故意把类名改成和文件名不一致，观察编译报错，再改回来（记住这个错误）。

## 延伸
- 速查复习：[[../基础语法|基础语法]]
- 工具进阶：[[../../06-工具链/03-IDEA实战|IDEA 实战]]、[[../../06-工具链/01-Maven实战|Maven 实战]]
- 返回目录：[[00-前言与学习地图|《Java 基础系统教程》目录]]
