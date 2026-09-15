---
tags:
  - Java
  - 教程书
  - 第八章
created: 2026-09-02
---

# 第 8 章 常用 API（String、包装类、日期、Math）

## 本章目标
- 吃透 String 不可变性、常量池、拼接选择
- 掌握自动装箱拆箱与 Integer 缓存
- 会用现代日期时间 API（LocalDate 等）
- 知道金额用 BigDecimal、常用 Math/Random

---

## 8.1 String：最常用的类

### 不可变（immutable）
String 底层字符内容**一旦创建不可修改**。所有"修改"方法（substring、replace、toUpperCase）都是**返回一个新字符串**，原串不变。

```java
String s = "abc";
s.toUpperCase();
System.out.println(s);   // 仍是 abc！要 s = s.toUpperCase() 才生效
```

不可变的好处：线程安全、可放进字符串常量池复用、可安全做 HashMap 的 key。

### 两种创建方式与常量池
```java
String a = "hello";              // 字面量：先去字符串常量池找，有就复用
String b = "hello";
String c = new String("hello");  // new：一定在堆新建对象
System.out.println(a == b);      // true，指向常量池同一个
System.out.println(a == c);      // false，地址不同
System.out.println(a.equals(c)); // true，内容相同（比内容永远用 equals！）
```

### 常用方法
```java
String s = "Hello,Java";
s.length();              // 长度 10
s.charAt(0);             // 'H'
s.substring(6);          // "Java"，从下标 6 到末尾
s.substring(0, 5);       // "Hello"，[0,5) 左闭右开
s.contains("Java");      // true
s.indexOf("J");          // 6，找不到返回 -1
s.split(",");            // 按逗号切分成数组 ["Hello","Java"]
s.replace("J", "j");     // 替换，返回新串
s.trim();                // 去首尾空白（JDK11 用 strip，对 Unicode 更好）
s.toUpperCase();         // 转大写
String.join("-", "2026","09","02"); // "2026-09-02"
```

### 字符串拼接怎么选（高频）
| 方式 | 适用 |
|------|------|
| `+` | 少量、简单拼接，编译器会优化 |
| `StringBuilder` | **循环里大量拼接（必用）**，可变、非线程安全、最快 |
| `StringBuffer` | 需要线程安全时（方法加了 synchronized，略慢） |

```java
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 100; i++) sb.append(i);  // 循环拼接不要用 +，会产生大量垃圾对象
String result = sb.toString();
```

---

## 8.2 包装类：把基本类型变成对象

集合的泛型不能放基本类型（`List<int>` 非法），需要对应的**包装类**：

| 基本 | byte | short | int | long | float | double | char | boolean |
|------|------|-------|-----|------|-------|--------|------|---------|
| 包装 | Byte | Short | **Integer** | Long | Float | Double | Character | Boolean |

### 自动装箱 / 拆箱
```java
Integer a = 10;     // 自动装箱：Integer.valueOf(10)
int b = a;          // 自动拆箱：a.intValue()
```

### Integer 缓存坑（面试必考）
```java
Integer x = 127, y = 127;
Integer m = 128, n = 128;
System.out.println(x == y);  // true！Integer 缓存了 -128~127
System.out.println(m == n);  // false，超出范围 new 了新对象
System.out.println(m.equals(n)); // true，比内容用 equals
```
结论：**包装类比较一律用 equals，别用 ==。**

### 字符串与数字互转
```java
int i = Integer.parseInt("123");          // 字符串 → int
String s = String.valueOf(123);           // 数字 → 字符串（推荐，null 安全）
Integer.valueOf("123");                  // 字符串 → Integer
```

---

## 8.3 日期时间 API

### 老 API 的问题（了解为什么要换）
`Date`、`SimpleDateFormat` 可变且**线程不安全**，月份从 0 开始反人类。新项目用 **JDK 8 的 `java.time`**（不可变、线程安全）。

### 现代 API（必会）
```java
import java.time.*;

LocalDate date = LocalDate.now();          // 今天（只有日期）
LocalTime time = LocalTime.now();          // 当前时间
LocalDateTime dt = LocalDateTime.now();    // 日期+时间

LocalDate d = LocalDate.of(2026, 9, 2);
d.plusDays(7);                             // 加 7 天（返回新对象，原对象不变）
d.minusMonths(1);                          // 减 1 月
d.getDayOfWeek();                          // 星期几
d.isBefore(LocalDate.of(2027,1,1));        // 比较

// 格式化与解析
DateTimeFormatter fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd");
String text = dt.format(fmt);                       // 格式化为字符串
LocalDate parsed = LocalDate.parse("2026-09-02", fmt); // 字符串解析为日期
```

计算时间差用 `Duration`（时分秒）/ `Period`（年月日）；需要时区用 `ZonedDateTime`；时间戳用 `Instant`。

---

## 8.4 Math、Random、BigDecimal

```java
Math.abs(-5);      // 5 绝对值
Math.max(3, 9);    // 9
Math.min(3, 9);    // 3
Math.pow(2, 10);   // 1024.0
Math.sqrt(16);     // 4.0
Math.round(3.6);   // 4 四舍五入（long）
Math.floor(3.9);   // 3.0 向下取整；ceil 向上取整
Math.random();     // [0.0, 1.0) 随机小数

Random r = new Random();
r.nextInt(100);    // [0,100) 随机整数
```

### BigDecimal：金额计算专用
```java
import java.math.BigDecimal;
BigDecimal a = new BigDecimal("0.1");   // 用字符串构造，别用 double 构造
BigDecimal b = new BigDecimal("0.2");
a.add(b);            // 加
a.subtract(b);       // 减
a.multiply(b);       // 乘
a.divide(b, 2, RoundingMode.HALF_UP);   // 除：必须指定精度和舍入模式，否则可能抛异常
```
> 铁律：**钱用 BigDecimal，构造用字符串，除法指定精度与舍入模式。**

---

## 8.5 Objects 工具类（判空很有用）
```java
Objects.equals(a, b);   // 避免 a 为 null 时 a.equals 空指针
Objects.requireNonNull(x, "x 不能为 null");  // 为空直接抛异常，做参数校验
```

---

## 8.6 工程实践与踩坑
- 比字符串/包装类内容永远用 `equals`，常量放前面：`"ok".equals(x)` 防空指针。
- 循环拼接字符串用 StringBuilder，别用 `+`。
- 金额、精确计算用 BigDecimal（字符串构造）。
- 新项目日期用 java.time，别再用 Date/SimpleDateFormat。
- 字符串判空用工具库（如 `str == null || str.isEmpty()`，或 Apache Commons 的 StringUtils）。

---

## 本章小结
- String 不可变，比内容用 equals，循环拼接用 StringBuilder。
- 包装类自动装箱拆箱，Integer 有 -128~127 缓存，比较用 equals。
- 现代日期用 LocalDate/LocalDateTime/DateTimeFormatter，不可变且线程安全。
- 金额用 BigDecimal，Math/Random 掌握常用方法。

## 动手练习
1. 统计一个字符串中每个字符出现的次数（提示：用 HashMap，第 11 章）。
2. 验证 Integer 缓存：分别用 == 和 equals 比较 127、128 两组值并解释。
3. 用 StringBuilder 把数组 `[1,2,3]` 拼成 `"1,2,3"`。
4. 用 LocalDate 计算你出生到今天一共多少天；用 BigDecimal 算 0.1+0.2。

## 延伸
- 速查复习：[[../String与包装类|String 与包装类]]（拼接方式对比 / 常量池图 / 包装类速查）
- 下一章：[[09-异常处理|第 9 章 异常处理]]
- 返回目录：[[00-前言与学习地图|教程目录]]
