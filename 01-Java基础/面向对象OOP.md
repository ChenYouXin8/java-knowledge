<!-- 笔记标题：面向对象OOP -->

> 4 大特性：封装、继承、多态、抽象。这是 Java 程序员的基本功。

---

## 1️⃣ 代码模板

### 1-1 类的标准写法（Java Bean 模板）

```java
public class User {
    // ① 字段（私有化）
    private Long id;
    private String name;
    private int age;
    private String email;

    // ② 无参构造（框架要求）
    public User() {}

    // ③ 全参构造
    public User(Long id, String name, int age, String email) {
        this.id = id;
        this.name = name;
        this.age = age;
        this.email = email;
    }

    // ④ Getter & Setter（IDE 自动生成）
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public int getAge() { return age; }
    public void setAge(int age) {
        if (age < 0 || age > 150) throw new IllegalArgumentException("年龄非法");
        this.age = age;
    }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }

    // ⑤ toString（调试用）
    @Override
    public String toString() {
        return "User{id=" + id + ", name='" + name + "', age=" + age + "}";
    }

    // ⑥ equals & hashCode（集合用）
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        User user = (User) o;
        return Objects.equals(id, user.id);
    }
    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}
```

### 1-2 继承的标准写法

```java
// 父类
public class Animal {
    protected String name;

    public Animal() {}  // 子类构造必须调用 super()

    public Animal(String name) {
        this.name = name;
    }

    public void eat() {
        System.out.println(name + " is eating");
    }

    public void sleep() {
        System.out.println(name + " is sleeping");
    }
}

// 子类
public class Dog extends Animal {
    private String breed;  // 独有字段

    public Dog() {
        super();  // 隐式调用，写不写都行
    }

    public Dog(String name, String breed) {
        super(name);  // 显式调用父类有参构造
        this.breed = breed;
    }

    @Override  // 重写父类方法
    public void eat() {
        System.out.println(name + " 正在吃狗粮");
    }

    public void bark() {
        System.out.println(name + " 汪汪叫");
    }
}

// 使用
Dog dog = new Dog("旺财", "金毛");
dog.eat();    // 旺财 正在吃狗粮（调用子类版本）
dog.sleep();  // 旺财 is sleeping（继承父类）
dog.bark();   // 旺财 汪汪叫（子类独有）
```

### 1-3 接口实现的标准写法

```java
// 接口：定义能力
public interface Flyable {
    int MAX_HEIGHT = 1000;  // 隐式 public static final

    void fly();  // 隐式 public abstract

    // Java 8+：默认方法
    default void land() {
        System.out.println("安全降落");
    }
}

// 接口：定义另一个能力
public interface Swimmable {
    void swim();
}

// 类实现多个接口
public class Duck extends Animal implements Flyable, Swimmable {
    private String name;

    public Duck(String name) {
        super(name);
    }

    @Override
    public void fly() {
        System.out.println(name + " 展翅高飞");
    }

    @Override
    public void swim() {
        System.out.println(name + " 浮在水面上");
    }

    @Override
    public void eat() {  // 继承 Animal，也需要重写
        System.out.println(name + " 吃虫子");
    }
}

// 多态调用
Flyable f = new Duck("唐老鸭");
f.fly();      // 唐老鸭 展翅高飞
f.land();     // 安全降落（默认方法）
```

### 1-4 匿名内部类 vs Lambda（函数式接口）

```java
// 场景：排序时传入比较器

// 方式1：匿名内部类（Java 7 及以前）
Collections.sort(list, new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return a.length() - b.length();
    }
});

// 方式2：Lambda 表达式（Java 8+，推荐）
Collections.sort(list, (a, b) -> a.length() - b.length());

// Lambda 的几种简化形式
list.sort((a, b) -> a.length() - b.length());        // 标准
list.sort(Comparator.comparingInt(String::length));   // 方法引用
list.sort(Comparator.comparingInt(s -> s.length()));   // 方法引用简化
```

### 1-5 工厂模式（封装对象创建）

```java
// 不用工厂：直接 new
Connection conn = new MySQLConnection();

// 用工厂：统一创建
public interface ConnectionFactory {
    Connection create();
}

public class MySQLFactory implements ConnectionFactory {
    @Override
    public Connection create() {
        return new MySQLConnection();
    }
}

// 使用
ConnectionFactory factory = new MySQLFactory();
Connection conn = factory.create();  // 多态：只依赖接口，不依赖实现
```

---

## 2️⃣ 对比表格

### 2-1 重载 vs 重写

| 维度 | 重载（Overload） | 重写（Override） |
|------|-----------------|----------------|
| 位置 | 同一个类 | 子类 vs 父类 |
| 方法名 | 必须相同 | 必须相同 |
| 参数列表 | 必须不同（个数/类型/顺序） | 必须相同 |
| 返回类型 | 可以不同 | 相同或子类类型 |
| 访问修饰符 | 可以不同 | 不能比父类更严格 |
| 异常 | 无限制 | 不能抛出新异常或更宽的异常 |
| 关键字 | — | `@Override`（建议写） |
| 发生时机 | 编译时（静态绑定） | 运行时（动态绑定） |

### 2-2 抽象类 vs 接口

| 特性 | 抽象类 | 接口 |
|------|--------|------|
| 关键字 | `abstract class` | `interface` |
| 字段 | 普通字段 | 默认 `public static final`（常量） |
| 方法 | 抽象方法 + 普通方法均可 | 默认 `public abstract`（抽象）|
| Java 8+ | — | 可有 `default` / `static` 方法 |
| Java 9+ | — | 可有 `private` 方法 |
| 构造方法 | ✅ 有 | ❌ 无 |
| 继承 | 单继承 | 多实现（`implements A, B`） |
| 适用场景 | "is-a"（是什么）共性抽象 | "has-a"（有什么能力）能力组合 |

### 2-3 this / super / static / final 对比

| 关键字 | 指向 | 能用在哪里 | 常见用途 |
|--------|------|----------|---------|
| `this` | 当前对象 | 实例方法/构造方法 | 区分字段与参数、构造方法间调用 |
| `super` | 父类对象 | 实例方法/构造方法 | 调用父类方法/构造 |
| `static` | 类本身（非对象） | 方法/字段/静态块 | 工具方法、类级别共享数据 |
| `final` | — | 类/方法/字段 | 类不可继承、方法不可重写、字段不可修改 |

### 2-4 访问修饰符权限

| 修饰符 | 本类 | 同包 | 子类 | 任意位置 |
|--------|------|------|------|---------|
| `private` | ✅ | ❌ | ❌ | ❌ |
| （默认） | ✅ | ✅ | ❌ | ❌ |
| `protected` | ✅ | ✅ | ✅ | ❌ |
| `public` | ✅ | ✅ | ✅ | ✅ |

---

## 3️⃣ 图解结构

### 3-1 OOP 四大特性关系图

```
┌──────────────────────────────────────────────┐
│                   面向对象 OOP               │
└─────────────────────┬────────────────────────┘
                      │
    ┌─────────┬───────┴───────┬─────────┐
    │         │               │         │
 ┌──▼───┐ ┌──▼───┐     ┌──────▼────┐ ┌──▼────┐
 │ 封装  │ │ 继承  │     │   多态    │ │ 抽象  │
 │Encaps.│ │Inherit│     │Polymorph.│ │Abstract│
 └──┬───┘ └──┬───┘     └─────┬─────┘ └───┬───┘
    │         │                │           │
 把数据藏│子类复用│父类引用指向子 │抽象方法强制
 在内部  │父类代码│类对象执行子类 │子类实现
    │         │   方法           │           │
 ┌──▼─────────▼───────┐  ┌─────▼──────┐ ┌──▼────────┐
 │  private + getter  │  │ Animal a   │ │ abstract  │
 │  /setter           │  │ = new Dog()│ │ class/    │
 │  保护数据安全       │  │ a.eat()    │ │ interface │
 └────────────────────┘  └────────────┘ └───────────┘
```

### 3-2 类初始化顺序

```
执行顺序（子类加载时）：

  ① 父类静态字段初始化
  ② 父类静态代码块
  ③ 子类静态字段初始化
  ④ 子类静态代码块
  ─────────────────────
  ⑤ 父类实例字段初始化
  ⑥ 父类构造方法
  ⑦ 子类实例字段初始化
  ⑧ 子类构造方法
```

### 3-3 多态执行流程

```
Animal a = new Dog();

编译时（左边）：      Animal a  ──→ 编译检查只看 Animal 类
运行时（右边）：         new Dog()  ──→ 实际创建 Dog 对象

调用 a.eat() 时：
  1. 查 Animal 类有没有 eat()  ─→ 有，编译通过
  2. 运行时 new Dog()  ─→ 实际执行 Dog 重写的 eat()
```

---

## 4️⃣ 速查清单

### 4-1 Object 常用方法

| 方法 | 返回值 | 说明 |
|------|--------|------|
| `equals(Object o)` | boolean | 比较对象相等性（需重写） |
| `hashCode()` | int | 哈希值（需与 equals 同步重写） |
| `toString()` | String | 返回 "类名@哈希值"（建议重写） |
| `getClass()` | Class<?> | 获取运行时类对象 |
| `clone()` | Object | 克隆对象（需实现 Cloneable） |
| `finalize()` | void | GC 前调用（已废弃，不推荐用） |

### 4-2 常用设计模式（OOP 实现）

| 模式 | 核心思想 | 代码模板位置 |
|------|---------|------------|
| 单例模式 | 类只有一个实例 | 见下方速查 |
| 工厂模式 | 统一创建对象 | 详见上方 1-5 |
| 模板方法 | 父类定义流程骨架，子类实现细节 | — |
| 策略模式 | 定义一系列算法，运行时切换 | — |

### 4-3 单例模式 3 种写法

```java
// 饿汉式（线程安全，但类加载时就创建，可能浪费）
public class Singleton1 {
    private static final Singleton1 INSTANCE = new Singleton1();
    private Singleton1() {}
    public static Singleton1 getInstance() { return INSTANCE; }
}

// 懒汉式（线程不安全）
public class Singleton2 {
    private static Singleton2 instance;
    private Singleton2() {}
    public static Singleton2 getInstance() {
        if (instance == null) {
            instance = new Singleton2();  // 多线程下可能创建多个
        }
        return instance;
    }
}

// 双检锁（线程安全，推荐）
public class Singleton3 {
    private static volatile Singleton3 instance;
    private Singleton3() {}
    public static Singleton3 getInstance() {
        if (instance == null) {
            synchronized (Singleton3.class) {
                if (instance == null) {
                    instance = new Singleton3();
                }
            }
        }
        return instance;
    }
}
```

---

## 5️⃣ 场景选择器

### 5-1 该用抽象类还是接口？

```
需求是"是什么"（共性）？
         │
    ┌────┴────┐
    │         │
   Yes        No
    │         │
  抽象类    有多种能力/行为？
    │         │
  要复用   ┌──┴──────────┐
  代码？    │              │
    │      Yes             No
 ┌──┴──┐   │              │
 Yes    No  │              │
  │      │  │           普通类
  │      │  │
  │   接 口  接口
  │   抽象方法
抽象类  定义行为
有共有  但无共有
代码    实现
  │      │
  ▼      ▼
抽象类  接口
```

### 5-2 该用继承还是组合？

```
需要复用父类的行为（方法）吗？
         │
    ┌────┴────┐
    │         │
   Yes        No
    │         │
继承 is-a   普通类
    │         │
子类是   ──→ 用组合 has-a
父类吗？      （把另一个对象作为字段）
    │         │
  ┌──┴───┐   ┌──┴───┐
 Yes      No  │      │
  │        │  │    需要多态？
继承   接 口  组合   │
    │      │    │      │
    ▼      ▼    ▼      ▼
  继承  implements 组合   普通引用
```

---

## ❓ 常见面试题

**Q1: 重载（Overload）和重写（Override）的区别？**
> 详见上方对比表格 2-1。
**Q2: 抽象类能 `new` 吗？接口能 `new` 吗？**
> 都不能直接 `new`，但可以用匿名内部类实现。
**Q3: `this` 和 `super` 的区别？**
> `this` 指向当前对象，`super` 指向父类。构造方法里 `this()` 调用本类其他构造，`super()` 调用父类构造，**且 `this()` 和 `super()` 不能同时出现**。

## 📊 学习状态

- [x] 类与对象
- [x] 封装
- [x] 继承
- [x] 多态
- [x] 抽象类与接口
- [x] 代码模板
- [x] 设计模式

## 🐛 踩坑记录

- 重写时**访问修饰符不能更严格**（如父类 `public`，子类不能 `protected`）
- 抽象类**可以有构造方法**（给子类用）
- 接口的字段默认 `public static final`，必须在声明时初始化
- `private` 方法不能被重写
- `static` 方法不能被重写（隐藏）
- 构造方法里 `this()` 和 `super()` 只能写一个，且必须是第一句
