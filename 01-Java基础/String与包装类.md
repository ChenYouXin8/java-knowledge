<!-- 笔记标题：String与包装类 -->

> String 是 Java 最高频的类，面试必考。包装类是基本类型和对象类型的桥梁。

---

## 1️⃣ 代码模板

### 1-1 String 拼接的 4 种方式

```java
// 方式1：直接 + 拼接（编译器优化为 StringBuilder，推荐用于少量拼接）
String a = "Hello" + " " + "World";  // 编译时优化，无性能问题

// 方式2：StringBuilder（循环内拼接，推荐）
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 100; i++) {
    sb.append(i).append(",");
}
String result = sb.toString();

// 方式3：StringJoiner（按分隔符拼接数组）
StringJoiner sj = new StringJoiner(", ", "[", "]");
for (String s : new String[]{"a", "b", "c"}) {
    sj.add(s);
}
System.out.println(sj.toString());  // [a, b, c]

// 方式4：String.join()（最简洁，按分隔符合并数组）
String[] arr = {"a", "b", "c"};
String joined = String.join(",", arr);  // "a,b,c"
```

### 1-2 String 常用操作模板

```java
String s = "  Hello World!  ";

// 判空与长度
s.isEmpty();           // false（非空）
s.isBlank();          // false（含空格）
s.length();           // 16（含前后空格）

// 查找与判断
s.indexOf("World");   // 7（首次出现位置）
s.lastIndexOf("l");   // 11（最后出现位置）
s.contains("World");  // true
s.startsWith("  H");  // true
s.endsWith("!  ");    // true

// 截取
s.trim();             // "Hello World!"（去首尾空格）
s.strip();           // "Hello World!"（Unicode 空白，比 trim 更强）
s.substring(2, 7);   // "Hello"（[2,7)）
s.split(" ");        // ["", "", "Hello", "World!", ""]

// 转换大小写
"abc".toUpperCase();  // "ABC"
"HELLO".toLowerCase(); // "hello"

// 替换
s.replace("World", "Java");      // "  Hello Java!  "
s.replaceAll("\\s+", "-");       // 正则替换空格为 "-"
s.replaceFirst("\\d+", "#");     // 只替换第一个匹配的

// 格式化
String name = "Tom";
int age = 18;
String formatted = String.format("姓名=%s, 年龄=%d", name, age);
String path = String.format("D:\\Users\\%s\\Documents", name);

// 转换
String.valueOf(123);        // "123"（最安全，null 不抛异常）
Integer.toString(123);      // "123"
"123".toCharArray();        // ['1','2','3']
"hello".getBytes();         // UTF-8 字节数组
new String(bytes, "UTF-8"); // 字节数组转字符串
```

### 1-3 Integer 缓存与比较模板

```java
// ⚠️ Integer 缓存范围：-128 ~ 127
Integer a = 127;
Integer b = 127;
System.out.println(a == b);  // true（缓存命中）

Integer c = 128;
Integer d = 128;
System.out.println(c == d);  // false（缓存范围外）

// ✅ 正确比较方式：永远用 equals
System.out.println(a.equals(b));  // true
System.out.println(c.equals(d));  // true

// ✅ 整数运算时自动拆箱，结果用 ==
System.out.println(Integer.valueOf(127) == 127);  // true（自动拆箱）

// ✅ 集合中用 Integer 也用 equals 比较
List<Integer> list = Arrays.asList(127, 128);
System.out.println(list.contains(127));  // true
System.out.println(list.get(0) == 127);  // true（自动拆箱）

// ✅ 自动装箱细节
Integer x = 127;        // Integer.valueOf(127)，命中缓存，返回同一对象
Integer y = new Integer(127);  // 永远创建新对象，❌ 不推荐
```

### 1-4 字符串反转 3 种写法

```java
// 方式1：StringBuilder（最推荐）
String reversed = new StringBuilder("hello").reverse().toString();

// 方式2：char 数组
public static String reverse2(String s) {
    char[] chars = s.toCharArray();
    for (int i = 0, j = chars.length - 1; i < j; i++, j--) {
        char temp = chars[i];
        chars[i] = chars[j];
        chars[j] = temp;
    }
    return new String(chars);
}

// 方式3：Stream（Java 8+）
String reversed3 = new StringBuilder("hello")
    .reverse()
    .toString();
```

---

## 2️⃣ 对比表格

### 2-1 String / StringBuilder / StringBuffer

| 特性 | String | StringBuilder | StringBuffer |
|------|--------|-------------|------------|
| 可变性 | ❌ 不可变 | ✅ 可变 | ✅ 可变 |
| 线程安全 | ✅ 安全（不可变） | ❌ 不安全 | ✅ 安全（synchronized） |
| 性能 | 拼接慢（每次new对象） | 最快 | 较慢（加锁） |
| 使用场景 | 字符串常量 | **单线程**字符串拼接 | 多线程字符串拼接 |
| 底层 | final char[] / byte[] | char[] | char[] |

### 2-2 String 两种创建方式对比

| 方式 | 代码 | 创建位置 | 是否共享 |
|------|------|---------|---------|
| 字面量 | `String s = "abc"` | **字符串常量池** | 相同字面量共享 |
| new | `String s = new String("abc")` | **堆内存** | 每次 new 新建对象 |

```java
String a = "abc";
String b = "abc";
System.out.println(a == b);  // true，共享常量池

String c = new String("abc");
String d = new String("abc");
System.out.println(c == d);  // false，堆里两个对象

System.out.println(a == c);  // false，池 vs 堆
System.out.println(a.equals(c));  // true，内容相同
```

### 2-3 基本类型 vs 包装类

| 维度 | 基本类型 | 包装类 |
|------|---------|--------|
| 默认值 | 0/false | null |
| 存储位置 | 栈（局部变量） | 堆（对象） |
| 是否为对象 | ❌ | ✅（可调用方法） |
| 泛型支持 | ❌ | ✅ |
| 集合兼容 | ❌ | ✅ |
| 自动装箱 | — | Java 5+ 自动 |
| 自动拆箱 | Java 5+ 自动 | — |
| 性能 | 快（无对象开销） | 慢（需堆内存） |

### 2-4 String 不可变的原因与好处

| 不可变原因 | 说明 |
|-----------|------|
| `final class` | 类不可被继承，防止子类破坏 |
| `private final` 字段 | 字段初始化后不可修改 |
| 无 setter | String 没有提供修改内容的方法 |

| 不可变好处 | 说明 |
|-----------|------|
| 线程安全 | 无需同步，可共享 |
| HashMap key | hashCode 固定，可缓存 |
| 安全性 | 数据库密码、URL 参数不可被篡改 |
| 字符串常量池 | 相同字面量共享内存 |

---

## 3️⃣ 图解结构

### 3-1 String 内存结构（JDK 9+）

```
JDK 8 及之前：
┌─────────────────────────────────┐
│ String (class)                   │
│ private final char[] value;      │
│                                  │
│ "abc" 在堆里的结构：              │
│ ┌─────────────────────┐         │
│ │ value: ['a','b','c'] │         │
│ └─────────────────────┘         │
└─────────────────────────────────┘

JDK 9+（压缩字符串）：
┌─────────────────────────────────┐
│ String (class)                   │
│ private final byte[] value;      │
│ private final coder;  // LATIN1  │
│           或 UTF16               │
│                                  │
│ ASCII 字符用 1 字节（LATIN1）    │
│ 中文用 2 字节（UTF16）           │
│ → 节省内存！                     │
└─────────────────────────────────┘
```

### 3-2 字符串常量池

```
字符串常量池（堆的独立区域）

第一次：
String a = "hello";  → 先在池里找"hello"
                       → 找不到，创建并放入池
                       → a 指向池中对象

第二次：
String b = "hello";  → 在池里找到"hello"
                       → b 也指向同一个对象
                       → a == b 为 true

new 方式：
String c = new String("hello");  → 堆里新建对象
                                   → c 指向堆对象（不是池）
                                   → a == c 为 false
```

### 3-3 自动装箱与拆箱流程

```
装箱（基本类型 → 包装类）：
  int i = 100;
         ↓  Integer.valueOf(i)
  Integer I = 100;

拆箱（包装类 → 基本类型）：
  Integer I = 100;
         ↓  I.intValue()
  int i = I;
         ↓ 或自动拆箱（编译时自动插入 .intValue()）
  i = I;
```

---

## 4️⃣ 速查清单

### 4-1 String 常用方法速查

| 方法 | 说明 | 示例 |
|------|------|------|
| `length()` | 字符数 | `"abc".length()` → 3 |
| `charAt(i)` | 指定位置字符 | `"abc".charAt(1)` → 'b' |
| `indexOf(s)` | 首次出现位置 | `"ababa".indexOf("b")` → 1 |
| `contains(s)` | 是否包含 | `"abc".contains("ab")` → true |
| `startsWith(s)` | 是否以 s 开头 | `"abc".startsWith("ab")` → true |
| `substring(s,e)` | 截取 [s, e) | `"abcde".substring(1,3)` → "bc" |
| `trim()` | 去首尾空格 | `" abc ".trim()` → "abc" |
| `strip()` | 去 Unicode 空白 | `" abc".strip()` → "abc" |
| `split(regex)` | 按正则分割 | `"a,b".split(",")` → ["a","b"] |
| `replace(s,t)` | 替换字符串 | `"abab".replace("a","c")` → "cbcb" |
| `replaceAll(r,s)` | 按正则替换 | `"a1b2".replaceAll("\\d","#")` → "a#b#" |
| `toUpperCase()` | 转大写 | `"abc".toUpperCase()` → "ABC" |
| `toLowerCase()` | 转小写 | `"ABC".toLowerCase()` → "abc" |
| `isEmpty()` | 长度为 0 | `"".isEmpty()` → true |
| `isBlank()` | 全空白 | `"   ".isBlank()` → true |
| `valueOf(x)` | 转字符串 | `String.valueOf(123)` → "123" |
| `getBytes()` | 转字节数组 | `"中".getBytes("UTF-8")` |
| `intern()` | 放入常量池 | `"abc".intern()` |

### 4-2 包装类速查

| 基本类型 | 包装类 | 缓存范围 | 主要方法 |
|---------|--------|---------|---------|
| `byte` | `Byte` | -128~127 | `byteValue()` |
| `short` | `Short` | -128~127 | `shortValue()` |
| `int` | `Integer` | -128~127 | `parseInt()`, `valueOf()`, `toBinaryString()` |
| `long` | `Long` | -128~127 | `longValue()`, `toHexString()` |
| `float` | `Float` | — | `parseFloat()`, `isNaN()` |
| `double` | `Double` | — | `parseDouble()`, `isNaN()`, `isInfinite()` |
| `char` | `Character` | 0~127 | `isDigit()`, `isLetter()`, `toUpperCase()` |
| `boolean` | `Boolean` | true/false | `parseBoolean()` |

---

## 5️⃣ 场景选择器

### 5-1 字符串拼接场景选择

```
拼接数量少（≤3段）？
         │
    ┌────┴────┐
    │          │
   Yes         No（循环内拼接）
    │          │
  直接用 +    串数是否固定？
    │            │
  最简单         ├─ 是 → StringBuilder.append()
  性能无差       │
  编译器自动优化  └─ 否（数组/列表转字符串）
                   │
               String.join()
               (最简洁)
```

### 5-2 基本类型选择场景

```
需要精确数值运算？
         │
    ┌────┴────┐
    │          │
  是          否（只是存储/比较）
    │          │
 用 BigDecimal  用哪种？
 用 BigInteger    │
   (金融/科学)   需要大整数？
                  │
              ┌───┴────┐
              │         │
             是          否
              │         │
        long (±9×10¹⁸)  int (±21亿）
              │         溢出怎么办？
              │           │
              │      ┌────┴────┐
              │     是          否
              │      │          │
              │    long       int
              │    (够用)    (不够用)
              │                │
              └──────────────┘
```

---

## ❓ 常见面试题

**Q1: `String s = new String("abc")` 创建了几个对象？**
> 1 或 2 个。"abc" 先在常量池找，无则创建 1 个；`new String()` 在堆里必创建 1 个。

**Q2: `intern()` 是什么？**
> 把字符串放入常量池并返回引用。JDK 7+ 常量池在堆里。

**Q3: 为什么 String 要设计成 final 类？**
> 防止子类重写破坏语义；保证 hashCode 固定（HashMap key）；保证线程安全。

**Q4: `Integer` 缓存范围为什么是 -128 ~ 127？**
> 小整数最常用，缓存可减少对象创建。可通过 `-XX:AutoBoxCacheMax` 调整上限。

**Q5: `"abc" + 123` 执行过程是怎样的？**
> 编译器优化为 `new StringBuilder().append("abc").append(123).toString()`，最终生成一个新字符串 "abc123"。

## 📊 学习状态

- [x] String 不可变原理
- [x] StringBuilder / StringBuffer
- [x] 包装类装箱拆箱
- [x] `==` vs `equals()`
- [x] 代码模板 / 速查清单 / 场景选择器

## 🐛 踩坑记录

- 循环字符串拼接用 `StringBuilder`，别用 `String +=`
- 包装类比较用 `.equals()`
- `String.split()` 参数是正则，`.` 要写 `\\.`
- `intern()` 在 JDK 7+ 行为变了（常量池移入堆）
- `new String("abc")` 每次创建新堆对象，要避免
