---
tags:
  - Java
  - 教程书
  - 第十二章
created: 2026-09-02
---

# 第 12 章 Lambda 与 Stream（JDK 8 函数式编程）

## 本章目标
- 理解 Lambda 是"一段可传递的代码"，会写常见 Lambda
- 认识函数式接口与方法引用
- 熟练用 Stream 对集合做过滤、映射、排序、分组、聚合

> 这是现代 Java（也是 Spring、面试）的必备写法，学会后处理集合代码量大幅减少。

---

## 12.1 从匿名内部类到 Lambda

回顾第 7 章：实现一个**只有一个抽象方法的接口（函数式接口）**，匿名内部类很啰嗦：

```java
Runnable r1 = new Runnable() {
    public void run() { System.out.println("hi"); }
};
```

Lambda 把"类对象样板"全部去掉，只保留**参数列表 -> 方法体**：

```java
Runnable r2 = () -> System.out.println("hi");
```

语法：`(参数) -> { 方法体 }`
- 无参：`() -> ...`
- 一个参数（类型可推断时，括号也能省）：`x -> x * x`
- 多参数：`(a, b) -> a + b`
- 方法体只有一句可省 `{}` 和 `return`；多句要 `{}` 且需要 return 时要写全。

---

## 12.2 四大内置函数式接口（`java.util.function`）

| 接口 | 抽象方法 | 含义 | 例子 |
|------|----------|------|------|
| `Predicate<T>` | `boolean test(T)` | 判断，返回布尔 | `s -> s.length() > 3` |
| `Function<T,R>` | `R apply(T)` | 转换 T→R | `s -> s.length()` |
| `Consumer<T>` | `void accept(T)` | 消费，无返回 | `s -> System.out.println(s)` |
| `Supplier<T>` | `T get()` | 供给，产出 T | `() -> new User()` |

还有 BiFunction、BiConsumer 等双参数版本。

---

## 12.3 方法引用 `::`（Lambda 的进一步简写）

当 Lambda 只是"调用一个已有方法"，可用方法引用：

```java
list.forEach(s -> System.out.println(s));
list.forEach(System.out::println);          // 对象::实例方法

list.stream().map(s -> s.length());
list.stream().map(String::length);           // 类::静态/实例方法

Supplier<List<String>> s = ArrayList::new;   // 构造器引用 ::new
```

四种形式：`对象::方法`、`类::静态方法`、`类::实例方法`、`类::new`。

---

## 12.4 Stream：集合的"声明式流水线"

Stream 不是集合，不存数据，而是对数据做**流式计算**。三步走：
**创建流 → 中间操作（可多个，惰性）→ 终端操作（触发执行）**。

准备数据：
```java
List<Student> students = List.of(
    new Student("张三", 18, 90),
    new Student("李四", 20, 55),
    new Student("王五", 19, 88)
);
```

### filter 过滤
```java
List<Student> passed = students.stream()
    .filter(stu -> stu.getScore() >= 60)   // 只留及格的
    .toList();                             // JDK16+ 收集为不可变 List
```

### map 映射（转换每个元素）
```java
List<String> names = students.stream()
    .map(Student::getName)                 // 取出所有名字
    .toList();
```

### sorted 排序
```java
List<Student> byScore = students.stream()
    .sorted(Comparator.comparingInt(Student::getScore).reversed()) // 分数降序
    .toList();
```

### distinct / limit / skip 去重、取前 N、跳过 N
```java
list.stream().distinct().limit(10).skip(2).toList();
```

### 聚合终端操作
```java
long n  = students.stream().filter(s -> s.getScore() >= 60).count(); // 个数
int sum = students.stream().mapToInt(Student::getScore).sum();       // 求和
double avg = students.stream().mapToInt(Student::getScore).average().orElse(0); // 平均
int max = students.stream().mapToInt(Student::getScore).max().orElse(0);        // 最大
boolean allPass = students.stream().allMatch(s -> s.getScore() >= 60); // 全部满足
boolean anyFail = students.stream().anyMatch(s -> s.getScore() < 60);  // 任一满足
```

### collect 收集为集合 / 拼接 / 分组（重点）
```java
import java.util.stream.Collectors;

List<String> list = stream.collect(Collectors.toList());
Set<String> set = stream.collect(Collectors.toSet());
String joined = students.stream().map(Student::getName).collect(Collectors.joining(",")); // 张三,李四,王五

// 分组：按是否及格分成两组
Map<Boolean, List<Student>> grouped = students.stream()
    .collect(Collectors.groupingBy(s -> s.getScore() >= 60));

// 分组统计人数
Map<Integer, Long> countByAge = students.stream()
    .collect(Collectors.groupingBy(Student::getAge, Collectors.counting()));
```

### 完整流水线示例
```java
// 及格学生的名字，按分数降序，拼成字符串
String result = students.stream()
    .filter(s -> s.getScore() >= 60)
    .sorted(Comparator.comparingInt(Student::getScore).reversed())
    .map(Student::getName)
    .collect(Collectors.joining(" -> "));
```

---

## 12.5 工程实践与踩坑
- Stream **中间操作是惰性的**，没有终端操作就不会执行。
- 一个 Stream 只能消费一次，重复用会抛 `IllegalStateException`。
- `toList()`（JDK16+）返回**不可变** List，要可变就用 `Collectors.toList()`。
- 别在 Stream 里写有副作用的操作（如在 forEach 里改外部共享变量），保持无状态。
- 简单循环更易读时不要硬套 Stream；复杂并行才考虑 `parallelStream()`（且注意线程安全）。

---

## 本章小结
- Lambda = `(参数) -> 方法体`，用于简化函数式接口的匿名实现。
- 记住 Predicate/Function/Consumer/Supplier 四大接口和 `::` 方法引用。
- Stream 三步走：创建 → 中间（filter/map/sorted/distinct）→ 终端（collect/count/分组/聚合）。
- groupingBy 分组、mapToInt 聚合、joining 拼接是高频操作。

## 动手练习
1. 给一个整数 List，用 Stream 求偶数的平方并去重后组成新 List。
2. 有一组学生，按班级分组并统计每组平均分。
3. 用 Stream 找出分数最高的学生。
4. 把 `List<String>` 中长度大于 3 的元素转大写、排序、用逗号拼成一个字符串。

## 延伸
- 速查复习：[[../面向对象OOP|面向对象 OOP]]（匿名内部类 vs Lambda）
- 集合基础：[[11-集合框架|第 11 章 集合框架]]
- 下一章：[[13-IO与文件|第 13 章 IO 与文件]]
- 返回目录：[[00-前言与学习地图|教程目录]]
