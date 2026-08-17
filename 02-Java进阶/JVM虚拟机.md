# JVM 虚拟机

> JVM 是 Java 面试最核心的模块之一，也是理解 Java 运行机制的关键。本笔记覆盖：内存结构、垃圾回收、类加载机制、字节码、JMM、性能调优、常见问题排查。

---

## 1️⃣ 代码模板

### 1-1 查看 JVM 内存使用

```java
public class MemoryDemo {
    public static void main(String[] args) {
        // 运行时数据区总览
        Runtime runtime = Runtime.getRuntime();
        long totalMemory = runtime.totalMemory();     // 堆总内存
        long freeMemory = runtime.freeMemory();       // 堆空闲内存
        long maxMemory = runtime.maxMemory();        // 堆最大内存（-Xmx）
        long usedMemory = totalMemory - freeMemory;  // 已用

        System.out.println("=== JVM 内存信息 ===");
        System.out.println("堆总内存:  " + (totalMemory / 1024 / 1024) + " MB");
        System.out.println("堆最大内存: " + (maxMemory / 1024 / 1024) + " MB");
        System.out.println("已用内存:  " + (usedMemory / 1024 / 1024) + " MB");
        System.out.println("空闲内存:  " + (freeMemory / 1024 / 1024) + " MB");

        // 手动 GC 测试
        System.gc();  // 建议 GC，但不保证立即执行
        System.out.println("GC 后空闲: " + (runtime.freeMemory() / 1024 / 1024) + " MB");
    }
}

// 运行时参数示例：
// java -Xms256m -Xmx512m -Xmn128m -XX:+UseG1GC MemoryDemo
// -Xms: 初始堆大小
// -Xmx: 最大堆大小
// -Xmn: 新生代大小
// -XX:+UseG1GC: 使用 G1 垃圾收集器
```

### 1-2 对象创建过程（字节码视角）

```java
public class ObjectCreate {
    public static void main(String[] args) {
        // 等价字节码步骤：
        // 1. new #2 <java/lang/Object>     → 分配内存
        // 2. dup                             → 复制操作数栈顶（供 init 调用）
        // 3. invokespecial #1 <java/lang/Object.<init>> → 调用构造方法
        // 4. astore_1                        → 存储到局部变量表
        Object obj = new Object();
    }
}

// 字节码分析工具：javap -c -p ObjectCreate.class
// javap -v ObjectCreate.class  → 更详细的常量池+行号表
```

### 1-3 四种引用类型代码示例

```java
import java.lang.ref.*;

// 强引用（默认，最常用）
Object strongRef = new Object();  // 永远不会 GC，除非手动置 null

// 软引用（内存不足时回收，可用于缓存）
SoftReference<byte[]> softRef = new SoftReference<>(new byte[1024 * 1024 * 10]);
// 内存不足时会被回收，可用于图片缓存
byte[] data = softRef.get();  // 可能返回 null

// 弱引用（下一次 GC 就回收）
WeakReference<Object> weakRef = new WeakReference<>(new Object());
System.gc();  // 弱引用对象立即变为 null
Object obj = weakRef.get();  // 几乎总是 null

// 虚引用（几乎无引用，随时可回收，配合队列做清理）
ReferenceQueue<Object> queue = new ReferenceQueue<>();
PhantomReference<Object> phantomRef = new PhantomReference<>(new Object(), queue);
// get() 永远返回 null，用于跟踪对象被回收的时机
Reference<? extends Object> ref = queue.poll();  // 对象回收时进入队列

// 实战：ThreadLocal 用的就是 WeakReference（key 弱引用，value 强引用）
// 实战：MyBatis 缓存用的 SoftReference（内存不足时自动清理）
```

### 1-4 类加载过程（验证代码）

```java
public class ClassLoadDemo {
    public static void main(String[] args) {
        // 主动使用才会触发类加载
        // new、读取/设置静态字段、调用静态方法
        // 子类加载时父类先加载

        System.out.println("=== 类加载器 ===");
        ClassLoader loader = ClassLoadDemo.class.getClassLoader();
        System.out.println("应用类加载器: " + loader);
        System.out.println("扩展类加载器: " + loader.getParent());
        System.out.println("启动类加载器: " + loader.getParent().getParent());  // null（由 C++ 实现）

        // 验证双亲委派
        System.out.println("\n=== 加载类测试 ===");
        try {
            // String 由启动类加载器加载，绕过了应用类加载器
            Class<?> cls = Class.forName("java.lang.String");
            System.out.println("String 类加载器: " + cls.getClassLoader());  // null
        } catch (ClassNotFoundException e) { e.printStackTrace(); }

        // 自定义类：由应用类加载器加载
        System.out.println("ClassLoadDemo 类加载器: " + this.getClass().getClassLoader());

        // 自己写的类加载器
        try {
            Class<?> myClass = new MyClassLoader().loadClass("com.example.MyClass");
            System.out.println("MyClass 类加载器: " + myClass.getClassLoader());
        } catch (ClassNotFoundException e) { e.printStackTrace(); }
    }
}

// 自定义类加载器
class MyClassLoader extends ClassLoader {
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        // 从指定路径加载 .class 文件
        String classPath = "D:/classes/" + name.replace('.', '/') + ".class";
        try (FileInputStream fis = new FileInputStream(classPath);
             FileChannel fc = fis.getChannel();
             ByteArrayOutputStream bos = new ByteArrayOutputStream()) {
            long size = fc.size();
            ByteBuffer bb = ByteBuffer.allocateDirect((int) size);
            fc.read(bb);
            bb.flip();
            byte[] bytes = bos.toByteArray();
            return defineClass(name, bb.array(), 0, bytes.length);
        } catch (IOException e) {
            throw new ClassNotFoundException(name, e);
        }
    }
}
```

### 1-5 垃圾回收代码演示

```java
public class GCDemo {
    // 对象何时被回收？——引用计数法（有缺陷，循环引用无法回收）
    // 主流算法：可达性分析（GC Roots）

    public static void main(String[] args) {
        // GC Roots 对象（永远不会被回收）
        // 1. 虚拟机栈中引用的对象
        // 2. 方法区中类静态属性引用的对象
        // 3. 方法区中常量引用的对象
        // 4. 本地方法栈中 JNI（Native 方法）引用的对象
        // 5. JVM 内部引用（Class 对象、异常对象）
        // 6. 同步锁（synchronized）持有的对象
        // 7. JVM 管理对象（JMXBean、绑定对象等）

        System.out.println("=== GC Roots 对象 ===");
        System.out.println("1. 虚拟机栈（本地变量表）中引用的对象");
        System.out.println("2. 方法区类静态属性引用的对象");
        System.out.println("3. 方法区常量引用的对象");
        System.out.println("4. 本地方法栈 JNI 引用的对象");
        System.out.println("5. JVM 内部引用（Class对象等）");
        System.out.println("6. synchronized 持有的对象");
    }
}

// 演示 finalize()（仅了解，已废弃，不推荐使用）
class FinalizeDemo {
    private static FinalizeDemo INSTANCE;

    @Override
    protected void finalize() throws Throwable {
        System.out.println("finalize() 被调用！");
        INSTANCE = this;  // 把自己赋值给静态引用，逃脱一次 GC
    }

    public static void main(String[] args) throws InterruptedException {
        INSTANCE = new FinalizeGuide();
        INSTANCE = null;  // 失去引用
        System.gc();
        Thread.sleep(100);  // 等 finalize 执行
        if (INSTANCE != null) {
            System.out.println("对象逃过一劫！");
        } else {
            System.out.println("对象被回收了");
        }
    }
}

//Minor GC / Major GC / Full GC 触发条件
//Minor GC（Young GC）：Eden 区满
//Major GC（Old GC）：Old 区满（CMS 术语，G1 尽量避免）
//Full GC：Old 区满 或 Metaspace 满 或 System.gc() 或 分配担保失败
```

### 1-6 常见垃圾回收器参数

```java
// 设置垃圾回收器的 JVM 参数示例

// Serial（单线程，最古老，适合桌面应用）
java -XX:+UseSerialGC -Xms512m -Xmx512m MyApp

// Parallel（多线程，默认，适合后台批处理）
java -XX:+UseParallelGC -Xms1g -Xmx1g -XX:+UseParallelOldGC MyApp

// CMS（并发标记清除，老年代，追求低停顿）
java -XX:+UseConcMarkSweepGC -XX:+UseParNewGC MyApp
// CMS 已废弃，JDK 14 移除

// G1（分区算法，JDK 9+ 默认，适合大堆，追求可控停顿） ★推荐
java -XX:+UseG1GC -Xms2g -Xmx2g -XX:MaxGCPauseMillis=200 MyApp

// ZGC（低延迟，JDK 11+，TB 级堆，停顿 < 1ms）
java -XX:+UseZGC -Xms4g -Xmx4g MyApp

// Shenandoah（低延迟，JDK 12+，停顿可控，与应用并发）
java -XX:+UseShenandoahGC -Xms2g -Xmx2g MyApp

// 常用 GC 日志参数
java -Xms512m -Xmx512m \
     -XX:+PrintGCDetails \           // 打印详细 GC 日志
     -XX:+PrintGCDateStamps \        // 打印时间戳
     -Xloggc:./gc.log \              // 输出到文件
     -XX:+HeapDumpOnOutOfMemoryError \ // OOM 时导出堆
     -XX:HeapDumpPath=./heapdump.hprof \
     MyApp

// 常用 GC 调优参数
-XX:NewSize=256m          // 新生代初始大小
-XX:MaxNewSize=512m       // 新生代最大大小
-XX:NewRatio=2            // 老年代/新生代比例（=2 表示老年代是新生代2倍）
-XX:SurvivorRatio=8      // Eden/Survivor 比例（=8 表示 Eden:From:To = 8:1:1）
-XX:MaxTenuringThreshold=15 // 对象进入老年代的年龄阈值（默认15）
-XX:+UseAdaptiveSizePolicy // 动态调整各区大小（JDK 8 默认开启）
```

### 1-7 常见 OOM 场景与复现

```java
// ===== OOM 1：堆溢出（Heap Space）=====
import java.util.*;
public class HeapOOM {
    public static void main(String[] args) {
        List<byte[]> list = new ArrayList<>();
        int i = 0;
        while (true) {
            list.add(new byte[1024 * 1024]);  // 每次分配 1MB
            System.out.println("分配第 " + (++i) + " MB");
        }
    }
}
// 运行：java -Xms100m -Xmx100m HeapOOM
// 现象：java.lang.OutOfMemoryError: Java heap space

// ===== OOM 2：栈溢出（Stack Overflow）=====
public class StackOverflow {
    private static int count = 0;

    public static void recursion() {
        count++;
        recursion();  // 无限递归
    }

    public static void main(String[] args) {
        try {
            recursion();
        } catch (StackOverflowError e) {
            System.out.println("递归层数: " + count);
            System.out.println("StackOverflowError: " + e.getMessage());
        }
    }
}
// 现象：java.lang.StackOverflowError

// ===== OOM 3：方法区溢出（Metaspace）=====
// 方法区存类元信息、字节码、常量池
// 常见原因：动态生成大量类（反射、CGlib、ASM）
// JDK 7 之前：永久代（-XX:PermSize -XX:MaxPermSize）
// JDK 8+：元空间（Metaspace，由本地内存而非堆内存分配）
// 运行：java -XX:MaxMetaspaceSize=64m MetaspaceOOM

// ===== OOM 4：直接内存溢出（Direct Memory）=====
// NIO 使用直接内存（-XX:MaxDirectMemorySize）
// ByteBuffer.allocateDirect() 分配
// 运行：java -XX:MaxDirectMemorySize=100m DirectMemoryOOM

// ===== OOM 5：创建线程过多（Native 内存）=====
public class ThreadOOM {
    public static void main(String[] args) {
        int count = 0;
        try {
            while (true) {
                new Thread(() -> {
                    try { Thread.sleep(10000); } catch (InterruptedException ignored) {}
                }).start();
                count++;
            }
        } catch (Error e) {
            System.out.println("创建线程数: " + count);
            System.out.println(e);
        }
    }
}
// 现象：native 内存不足，无法创建线程
```

### 1-8 JMM 可见性与有序性代码

```java
// ===== 可见性问题：volatile 解决 =====
public class VisibilityDemo {
    // 普通变量，多线程下可能不可见
    private /*volatile*/ boolean flag = false;

    // 线程A：写
    public void writer() {
        flag = true;  // 可能写入 CPU 缓存，不立即写回主存
        System.out.println("Writer set flag=true");
    }

    // 线程B：读
    public void reader() {
        while (!flag) {
            // 线程B 可能一直读到自己 CPU 缓存中的 false
        }
        System.out.println("Reader sees flag=true");
    }

    public static void main(String[] args) {
        VisibilityDemo demo = new VisibilityDemo();
        new Thread(demo::writer).start();
        new Thread(demo::reader).start();
        // 不加 volatile：reader 可能永远不会退出循环
    }
}

// ===== 有序性问题：指令重排 =====
public class ReorderDemo {
    private int a = 0;
    private boolean flag = false;

    // 线程A
    public void writer() {
        a = 1;           // 操作1
        flag = true;     // 操作2
        // 编译器/CPU 可能重排为：flag=true 先执行
    }

    // 线程B
    public void reader() {
        if (flag) {      // 操作3
            int r = a;   // 操作4
            // 若重排后 flag=true 但 a=0，r=0
        }
    }
    // 加 volatile 禁止重排，保证 happens-before
}

// ===== happens-before 规则演示 =====
public class HappensBeforeDemo {
    private int x = 0;
    private volatile int y = 0;

    // 规则：volatile 写 happens-before volatile 读
    public void write() {
        x = 10;          // 规则2：程序顺序规则（单线程）
        y = 20;          // 规则5：volatile 写
        // y=20 happens-before 后续的 y 读
    }

    public void read() {
        // y 读 happens-before 规则5
        int r1 = y;      // volatile 读
        int r2 = x;      // r2 可能仍为 0（不受 volatile 保护）
    }
}
```

### 1-9 OOP 对象头结构（理解锁原理）

```java
// ===== HotSpot 对象内存布局 =====
public class ObjectLayout {
    public static void main(String[] args) {
        // 使用 jol-core 分析对象布局
        // Maven: <dependency> <groupId>org.openjdk.jol</groupId> <artifactId>jol-core</artifactId> </dependency>

        // 普通对象布局
        System.out.println(ClassLayout.parseInstance(new Object()).toPrintable());
        // [OBJECT object internals:
        //  HEADER:       12 bytes  (对象头)
        //   mark word:   8 bytes
        //   klass word:  4 bytes (compressed)
        //  INSTANCE DATA: 0 bytes
        //  PADDING:      0 bytes
        //  OBJECT SHALLOW SIZE: 12 bytes
        //  HEAP INTERNALS: ...]

        // 带字段的对象
        System.out.println(ClassLayout.parseInstance(new Person()).toPrintable());
        // [OBJECT object internals:
        //  HEADER: 12 bytes  (对象头)
        //  INSTANCE DATA:
        //   int:  4 bytes    (age)
        //   bool: 1 byte     (male)
        //   3 bytes padding
        //  OBJECT SHALLOW SIZE: 20 bytes]

        // 数组对象
        System.out.println(ClassLayout.parseInstance(new int[8]).toPrintable());
    }
}

class Person {
    int age = 25;
    boolean male = true;
}

// 对象头结构（32位 JVM，64位会压缩）
// 无锁状态：25 bit 存分代年龄 + 2 bit 锁标志位 + 1 bit 固定0
// 偏向锁：23 bit 存线程ID + 2 bit 锁标志位 + epoch + 分代年龄
// 轻量级锁：30 bit 存栈中锁记录指针 + 2 bit 锁标志位
// 重量级锁：30 bit 存互斥量指针 + 2 bit 锁标志位（10）
// GC 标记：2 bit 锁标志位（11），配合 CMS 使用
```

---

## 2️⃣ 对比表格

### 2-1 JVM 内存区域对比

| 区域 | 线程共享 | 存什么 | 大小 | 异常 |
|------|---------|--------|------|------|
| **程序计数器** | ❌ | 当前字节码行号（Native 则存 -1） | 很小（<1MB） | 无（唯一不 OOM 区域） |
| **虚拟机栈** | ❌ | 方法调用栈帧（局部变量表/操作数栈/动态链接/返回地址） | 可配置 `-Xss1m` | StackOverflowError（栈满）/ OOM（动态扩展失败） |
| **本地方法栈** | ❌ | Native 方法栈（同虚拟机栈，但由 C/C++ 实现） | 可配置 | StackOverflowError / OOM |
| **堆（Heap）** | ✅ | 对象实例 + 数组 | 可配置 `-Xms -Xmx`，通常最大 | OutOfMemoryError: Java heap space |
| **方法区（Method Area）** | ✅ | 类信息（类的字节码）+ 运行时常量池 + 静态变量 | JDK 7 永久代/JDK 8+ 元空间（本地内存） | OutOfMemoryError: Metaspace（JDK 8+） |
| **直接内存（Direct Memory）** | ✅ | NIO 直接缓冲区（ByteBuffer.allocateDirect） | `-XX:MaxDirectMemorySize` | OutOfMemoryError: Direct buffer memory |

### 2-2 堆内存分代对比

| 分代 | 位置 | 对象特点 | 垃圾回收算法 | GC 频率 |
|------|------|---------|------------|---------|
| **Eden 区** | 新生代 | 新创建的对象，大多数生命周期短 | 复制算法 | 最频繁（Minor GC） |
| **Survivor From (s0)** | 新生代 | Minor GC 后 From Survivor 区 | 复制算法 | 与 Eden 同步 |
| **Survivor To (s1)** | 新生代 | Minor GC 后的目标 Survivor 区 | 复制算法 | 与 Eden 同步 |
| **Old Generation（老年代）** | 老年代 | 长生命周期对象（默认 15 岁以上）、大对象直接进入 | 标记-清除 / 标记-整理 | 较少（Major/Full GC） |

### 2-3 垃圾回收算法对比

| 算法 | 核心思想 | 优点 | 缺点 | 适用场景 |
|------|---------|------|------|---------|
| **引用计数** | 对象被引用+1，引用失效-1，为0回收 | 实现简单 | 循环引用无法回收，计数开销大 | 几乎不用（Python 用，但处理了循环引用） |
| **标记-清除（Mark-Sweep）** | 先标记存活对象，再清除未标记对象 | 简单 | 产生内存碎片，效率随对象增加而下降 | 老年代（CMS 初始/重新标记阶段） |
| **复制算法（Copying）** | 分两半，每次把存活对象复制到另一半 | 无碎片，简单高效 | 浪费一半空间 | 新生代（Eden→Survivor） |
| **标记-整理（Mark-Compact）** | 标记后移动存活对象，清理边界外内存 | 无碎片，利用率高 | 移动成本高，有停顿 | 老年代（Serial Old / Parallel Old） |
| **分代收集（Generational）** | 新生代复制，老年代标记-整理 | 综合最优 | 参数调优复杂 | 主流（JVM 默认） |

### 2-4 常见垃圾回收器对比

| 回收器 | 版本 | 分代 | 线程 | 特点 | 停顿时间 | 吞吐量 |
|--------|------|------|------|------|---------|--------|
| **Serial** | 最老 | 新生代 | 单线程 | 简单，Client 模式默认 | 停顿长（~100ms） | 低 |
| **ParNew** | 较老 | 新生代 | 多线程（并行） | Serial 的多线程版，CMS 默认配合 | 较Serial更短 | 中 |
| **Parallel Scavenge** | 较老 | 新生代 | 多线程（并行） | 吞吐量优先，JDK 8 新生代默认 | 较Serial更短 | 高 |
| **Serial Old** | 最老 | 老年代 | 单线程 | Serial 老年代版 | 停顿长 | 低 |
| **Parallel Old** | 较老 | 老年代 | 多线程（并行） | Parallel Scavenge 老年代搭档 | 中 | 高 |
| **CMS** | 老 | 老年代 | 并发（与用户线程并行） | 追求低停顿，标记清除有碎片 | 短停顿（~200ms） | 中（CPU敏感） |
| **G1** | 新 ⭐ | 全堆（分区） | 并发+并行 | JDK 9+默认，可设停顿目标 | 可控停顿（-XX:MaxGCPauseMillis） | 高 |
| **ZGC** | 最新 | 全堆 | 并发 | TB级堆，<1ms停顿 | **<1ms** | 极高 |
| **Shenandoah** | 较新 | 全堆 | 并发 | 与应用并发，不依赖分代 | <10ms | 高 |

### 2-5 类加载器层级

| 加载器 | 加载什么 | 路径 |
|--------|---------|------|
| **启动类加载器（Bootstrap ClassLoader）** | Java 核心类库（`JAVA_HOME/lib`） | `%JAVA_HOME%/lib` |
| **扩展类加载器（Extension/Platform ClassLoader）** | `ext` 目录下的类（`JAVA_HOME/lib/ext`） | `%JAVA_HOME%/lib/ext` |
| **应用类加载器（App ClassLoader）** | classpath 下的类（我们自己写的类） | `-classpath` 或 `-cp` |
| **自定义类加载器** | 自定义路径/加密类 | 自定义 |

**双亲委派流程：**
```
类加载请求
     ↓
应用类加载器 ──查▶ 扩展类加载器 ──查▶ 启动类加载器
     ↑                  ↑                  ↓
     │                  │            是否能加载？
     │                  ↓                    ↓
     │               是否能加载？           是 → 加载，返回
     ↓                  ↓                    ↓
     ↑                  否 ← ← ← ← ← 否 ← 否
     ↑                                       ↓
     ↑                                    返回（父加载器无法加载）
     ↑                                       ↓
     ↓                                    子加载器尝试加载
自身尝试加载
```

### 2-6 常见 OOM 类型对比

| OOM 类型 | 原因 | 解决思路 |
|---------|------|---------|
| **Java heap space** | 堆溢出（对象太多，内存泄漏） | 增加堆 `-Xmx`，排查内存泄漏 |
| **GC overhead limit exceeded** | GC 频繁但回收不了多少 | 同上，分析对象持有链 |
| **Metaspace** | 元空间溢出（类太多/类加载器泄漏） | 加大元空间 `-XX:MaxMetaspaceSize`，减少动态类生成 |
| **Unable to create new native thread** | 线程太多（native 内存耗尽） | 减少线程数/栈大小，分布式 |
| **Direct buffer memory** | NIO 直接内存耗尽 | 减小 `-XX:MaxDirectMemorySize`，查泄漏 |
| **StackOverflowError** | 栈溢出（递归太深/栈太大） | 检查递归，优化算法，增加 `-Xss` |

---

## 3️⃣ 图解结构

### 3-1 JVM 内存结构图

```
┌──────────────────────────────────────────────────────────────────────┐
│                        JVM 进程内存                                   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                        堆（Heap）✅线程共享                    │   │
│  │                                                               │   │
│  │  ┌───────────────┐ ┌────────────┐ ┌────────────────────────┐│   │
│  │  │   新生代       │ │            │ │     老年代（Old Gen）   ││   │
│  │  │               │ │            │ │                        ││   │
│  │  │ ┌───┬───┐    │ │ Survivor  │ │   大对象/长生命周期对象  ││   │
│  │  │ │Eden│ S0 │→│ │   区(s1)   │ │                        ││   │
│  │  │ └───┴───┘    │ │            │ │   ┌──────────────────┐ ││   │
│  │  │      ↑       │ │            │ │   │ 已分配对象区域     │ ││   │
│  │  │ Minor GC    │ │←─复制──────│ │   └──────────────────┘ ││   │
│  │  │ 存活对象→S1│ │  │            │ │                        ││   │
│  │  └───────────────┘ └────────────┘ └────────────────────────┘│   │
│  │  │←──── 新生代（Young Gen） ────→│←─── 老年代 ─────────────→│   │
│  │  │  -Xmn 控制新生代大小           │                          │   │
│  │  │  -XX:NewRatio=2 (老:新=2:1)  │                          │   │
│  │  └──────────────────────────────┘                           │   │
│  │                                                               │   │
│  │  ┌───────────────────────────────────────────────────────┐   │   │
│  │  │              运行时常量池（JDK 7 及之前含在方法区）      │   │   │
│  │  │  #1: Integer(100)  #2: String("hello")  #3: MethodRef │   │   │
│  │  └───────────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                   方法区（Method Area）✅线程共享              │   │
│  │                                                               │   │
│  │  ┌──────────────────┐  ┌─────────────────┐  ┌──────────────┐ │   │
│  │  │ 类信息（元数据）   │  │  字节码          │  │ 静态变量      │ │   │
│  │  │ - 类名           │  │  (方法区，不在堆) │  │ (JDK 7 移入堆)│ │   │
│  │  │ - 父类           │  │                  │  │ static obj   │ │   │
│  │  │ - 访问修饰符     │  │                  │  │              │ │   │
│  │  │ - 字段信息       │  │                  │  │              │ │   │
│  │  │ - 方法信息       │  │                  │  │              │ │   │
│  │  │ - 构造函数       │  │                  │  │              │ │   │
│  │  │ - 字节码         │  │                  │  │              │ │   │
│  │  └──────────────────┘  └─────────────────┘  └──────────────┘ │   │
│  │  JDK 7：永久代（PermGen）— 固定大小（易 OOM）                  │   │
│  │  JDK 8+：元空间（Metaspace）— 本地内存，动态扩展               │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │              直接内存（Direct Memory）✅线程共享               │   │
│  │           NIO 的 ByteBuffer.allocateDirect() 分配            │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                   本地方法栈 ❌线程私有                        │   │
│  │                   Native 方法（JNI）调用                       │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                   虚拟机栈 ❌线程私有                           │   │
│  │                                                               │   │
│  │  ┌────────┐  ┌────────┐  ┌────────┐                        │   │
│  │  │栈帧    │  │栈帧     │  │栈帧     │                        │   │
│  │  │main() │  │m1()    │  │m2()    │   ← 深度调用方向        │   │
│  │  │局部变量│  │局部变量 │  │局部变量 │                        │   │
│  │  │操作数栈│  │操作数栈 │  │操作数栈 │                        │   │
│  │  │动态链接│  │动态链接 │  │动态链接 │                        │   │
│  │  │返回地址│  │返回地址 │  │返回地址 │                        │   │
│  │  └────────┘  └────────┘  └────────┘                        │   │
│  │  -Xss1m（默认1MB，每线程一个栈）                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                   程序计数器 ❌线程私有                        │   │
│  │   每个线程一个，存当前字节码行号，Native 方法存 -1             │   │
│  │   唯一不 OOM 的区域                                            │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

### 3-2 对象创建过程

```
new Object() 执行流程：

第1步：检查类是否已加载
        │
        ↓
    类已加载？ ──否──→ 执行类加载（加载→验证→准备→解析→初始化）
        │
        是
        ↓
第2步：分配内存
        │
    指针碰撞（内存规整）
        │  ←─────── 分配完成后指针移动
        ↓
    空闲列表（内存碎片）
        │
    线程安全：CAS + 失败重试  或  TLAB（本地线程分配缓冲）
        │
        ↓
第3步：初始化为零值
        int → 0, boolean → false, 引用 → null
        ↓
第4步：设置对象头（Mark Word + Klass Pointer）
        ↓
第5步：执行 <init> 构造方法
        ↓
    对象创建完成，可被使用
```

### 3-3 垃圾回收算法示意图

```
【复制算法 - 新生代】

Eden                    Survivor(s1)
┌──────────────┐         ┌─────────┐
│ 对象A (存活) │
│ 对象B (存活) │ ──复制──▶│ 对象A   │
│ 对象C (死亡) │         │ 对象B   │   丢弃 Eden + s0 中死亡对象
│ 对象D (死亡) │         └─────────┘
└──────────────┘

特点：浪费 50% 空间，但效率高（复制成本 < 整理成本）
实际比例：Eden : s0 : s1 = 8 : 1 : 1


【标记-整理 - 老年代】

老年代内存
┌─────────────────────────────┐
│ 对象A │ 对象B │     │ 对象D │  ← 有碎片
│ 对象C │ 已删除│ 对象E│ 已删除│
└─────────────────────────────┘
        ↓ 标记-整理
┌─────────────────────────────┐
│ 对象A │ 对象B │ 对象C │ 对象E │  ← 无碎片，向一端移动
└─────────────────────────────┘
```

### 3-4 G1 收集器分区

```
G1 将堆划分为多个大小相等的 Region（1MB~32MB）：

┌────────┬────────┬────────┬────────┐
│ Eden   │ Eden   │ Survivor│  Old   │  ← 动态分配，比例不固定
│  (新)  │  (新)  │   (S)  │        │
├────────┼────────┼────────┼────────┤
│  Old   │  Old   │  Old   │ Eden   │
│        │        │        │  (新)  │
├────────┼────────┼────────┼────────┤
│ Humongous│ Humongous│  Survivor│  Old   │  ← H：大对象（> 50% Region）
│  (H区)  │   (H区)  │    (S)   │        │
├────────┼────────┼────────┼────────┤
│        │ Free   │  Old   │  Old   │
│        │  (空)  │        │        │
└────────┴────────┴────────┴────────┘

G1 特点：
- 每个 Region 可以是 Eden/Survivor/Old/Humongous/Free
- 设置停顿目标：-XX:MaxGCPauseMillis=200
- G1 会优先回收回收价值最高的 Region（Mixed GC）
```

### 3-5 CMS 收集过程

```
CMS = Concurrent Mark Sweep（并发标记清除）

阶段1：初始标记（STW）— 标记 GC Roots 直接引用的对象（非常快）
         ↓ 短暂停顿
阶段2：并发标记 — 与用户线程并发，从 GC Roots 遍历引用链（慢但不停顿）
         ↓ 与应用并发
阶段3：重新标记（STW）— 修正并发期间产生的变化（比初始标记长，但比 Full GC 短）
         ↓ 短暂停顿
阶段4：并发清除 — 与用户线程并发，清除死亡对象（不移动存活对象）
         ↓ 与应用并发

⚠️ CMS 问题：
- 并发阶段与应用争抢 CPU
- 无法处理"并发更新浮动垃圾"
- 标记清除产生内存碎片，导致 Full GC
- 预留空间不够时触发 "Concurrent Mode Failure"，退化为 Serial Old
```

---

## 4️⃣ 速查清单

### 4-1 虚拟机参数速查

```bash
# 堆内存参数
-Xms256m           # 初始堆大小
-Xmx512m           # 最大堆大小
-Xmn128m           # 新生代大小
-Xss1m             # 线程栈大小（JDK 8 默认 1MB）

# 新生代参数
-XX:NewRatio=2     # 老年代/新生代比例（=2，老:新=2:1）
-XX:SurvivorRatio=8 # Eden/Survivor 比例（=8，Eden:From:To=8:1:1）
-XX:MaxTenuringThreshold=15 # 对象进入老年代的年龄阈值

# 元空间参数
-XX:MetaspaceSize=128m  # 元空间初始大小
-XX:MaxMetaspaceSize=256m # 元空间最大（JDK 8+）
# JDK 7 及之前：
-XX:PermSize=128m       # 永久代初始大小
-XX:MaxPermSize=256m   # 永久代最大大小

# GC 日志
-verbose:gc            # 打印 GC 日志
-XX:+PrintGCDetails    # 打印详细 GC 日志
-Xloggc:gc.log         # GC 日志写入文件
-XX:+HeapDumpOnOutOfMemoryError # OOM 时导出堆
-XX:HeapDumpPath=./heap.hprof # 堆转储文件路径

# G1 专用
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200    # 最大停顿时间目标
-XX:G1HeapRegionSize=4m     # Region 大小（1/2/4/8/16/32 MB）
-XX:InitiatingHeapOccupancyPercent=45 # 触发 Mixed GC 的堆占用比例

# 其他
-XX:+PrintCommandLineFlags  # 打印命令行参数
-XX:+PrintFlagsFinal       # 打印所有最终参数
```

### 4-2 垃圾回收器组合速查

| 新生代 | 老年代 | 组合效果 |
|--------|--------|---------|
| Serial | Serial Old | 最古老，Client 模式默认 |
| ParNew | CMS | CMS 标配（CMS 不支持新生代 Parallel Scavenge） |
| Parallel Scavenge | Parallel Old | 吞吐量优先，JDK 8 默认 |
| G1 | G1 | JDK 9+ 全堆默认，分区收集 |
| ZGC | ZGC | 全堆，低延迟 |

### 4-3 对象进入老年代的时机速查

| 条件 | 说明 |
|------|------|
| 对象年龄 ≥ MaxTenuringThreshold | 默认 15 岁（对象头存 4 bit，最大 15） |
| Survivor 区相同年龄对象总和 > Survivor 区 50% | 该年龄以上全部进入老年代 |
| 大对象（> PretenureSizeThreshold） | 直接分配在老年代 |
| Minor GC 后 Survivor 区放不下 | 直接进入老年代（空间担保） |

### 4-4 类加载过程速查

| 阶段 | 做了什么 | 关键行为 |
|------|---------|---------|
| **加载（Loading）** | 读取 .class 字节码，生成 Class 对象 | 类加载器完成，双亲委派 |
| **验证（Verification）** | 验证字节码格式/语义/权限 | 文件头魔数 CAFEBABE |
| **准备（Preparation）** | 为静态变量分配内存，赋零值 | static int a = 10 → a=0（非10） |
| **解析（Resolution）** | 符号引用 → 直接引用（内存地址） | 类/方法/字段引用解析 |
| **初始化（Initialization）** | 执行 `<clinit>` 赋值、静态代码块 | **真正执行赋值** |

### 4-5 诊断命令速查

```bash
# 查看 JVM 进程
jps -l                    # 列出所有 Java 进程

# 查看进程 GC 情况（实时）
jstat -gc <pid> 1000      # 每 1 秒打印一次 GC 统计
jstat -gcutil <pid> 1000 # 打印 GC 使用率百分比

# 查看进程 GC 详细信息
jstat -gc <pid>
# 输出示例：
# S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU
# 43008.0 43008.0  0.0    0.0   344064.0  245760.0  860160.0   41943.0   45440.0  43840.0

# 导出堆转储文件（jmap）
jmap -dump:format=b,file=heap.hprof <pid>   # Full GC 后导出
jmap -heap <pid>                             # 查看堆配置和使用情况

# 查看线程堆栈（jstack）
jstack <pid>              # 打印线程堆栈
jstack -l <pid>           # 打印锁信息
# 查找死锁：jstack -l <pid> | findstr "Deadlock"

# 反编译字节码（javap）
javap -c MyClass.class     # 字节码
javap -p MyClass.class     # 私有成员
javap -v MyClass.class     # 详细（常量池+行号表）

# 远程连接（jmx）
java -Dcom.sun.management.jmxremote \
     -Dcom.sun.management.jmxremote.port=9010 \
     -Dcom.sun.management.jmxremote.authenticate=false \
     -Dcom.sun.management.jmxremote.ssl=false \
     MyApp

# Arthas（更强大的诊断工具）
# java -jar arthas-boot.jar
# dashboard          # 查看系统状态/线程/GC
# heapdump           # 导出堆
# jad <类名>         # 反编译类
# watch <类> <方法>  # 观察方法调用
# thread -b          # 找出死锁
```

---

## 5️⃣ 场景选择器

### 5-1 该用什么垃圾回收器？

```
业务场景是什么？
         │
    ┌────┴──────────────────────────────────┐
    │                                       │
  小内存（< 4GB）                        大内存（> 4GB）
  追求稳定低延迟                          追求高吞吐
  单个应用                                分布式/微服务
    │                                       │
    │                                    ┌──┴───────────────┐
    │                                    │                   │
    │                               停顿 < 10ms?          更低延迟？
    │                                    │                    │
    │                               G1 (JDK 9+)          ZGC（JDK 11+）
    │                               (-XX:MaxGCPauseMillis)  (超大堆)
    │                                   │
    └──────────────────────────────────→ ZGC（JDK 13+）
                                          或 Shenandoah（JDK 12+）
```

### 5-2 OOM 了怎么排查？

```
发生 OOM，JVM 已崩溃？
         │
    ┌────┴──────────────────────┐
    │                           │
  是（进程退出）               否（服务仍运行）
    │                           │
  查看 gc.log                  jmap -heap <pid>
  找 dump 文件                  jstat -gcutil <pid>
  导入 MAT 分析                  Arthas dashboard
                                heapdump 导出分析
    │
    ↓
  分析对象类型？
         │
    ┌────┴──────────────────────────┐
    │                               │
  char[] / byte[] 为主            其他对象为主
  → 字符串/缓存问题              → 业务对象泄漏
  → Arthas: sc -d 类名             → 检查集合/静态引用
    │
    ↓
  引用链分析（Retained Heap 排序）
```

### 5-3 频繁 Full GC 怎么排查？

```
Full GC 频繁（CMS / G1 / ZGC 都有）？
         │
    ┌────┴──────────────────────────────┐
    │                                   │
  老年代快速填满                       元空间快速增长
  （Old 区 GC 频繁）                   （Metaspace OOM）
    │                                   │
    │                               检查是否动态生成大量类
    │                               (CGlib/ASM/反射/类加载器)
    ↓                                   ↓
  是否大对象直接进老年代？            ↑ 是 → 优化类加载
  Survivor 区够不够？                 否 → 加大 Metaspace
  是否有内存泄漏？                   ↑ 否 → 检查是否泄漏
    │
    ↓
  Minor GC 后对象进入老年代太多？
         │
    ┌────┴────────────────┐
    │                     │
  Survivor 区太小        分配担保失败
    │                     │
    │                  检查对象年龄分布
    ↓                  jstat -gc <pid>
  调 SurvivorRatio        jstat -gcnew <pid>
```

---

## ❓ 常见面试题

**Q1: 对象的内存布局是怎样的？**
> 对象头（Mark Word 8字节 + Klass Pointer 4/8字节）+ 实例数据 + 对齐填充（8字节倍数）。Mark Word 存哈希码、GC 分代年龄、锁标志位。数组还有额外 4 字节存数组长度。

**Q2: 对象一定分配在堆上吗？**
> 不一定。JIT 逃逸分析后发现对象不会逃逸出方法（比如只返回给调用者，或没有逃逸），则可以在**栈上分配**（栈上分配=随栈帧出栈自动回收，不需要 GC），或**标量替换**（拆解为基本类型直接存在栈帧中）。

**Q3: 对象引用有哪些类型？**
> 强引用（new）、软引用（SoftReference，内存不足时回收）、弱引用（WeakReference，下次 GC 必回收）、虚引用（PhantomReference，几乎无引用，用于追踪回收时机，配合 ReferenceQueue）。

**Q4: 讲一下双亲委派模型？**
> 类加载请求逐级向上委托：应用加载器→扩展加载器→启动加载器。只有父加载器无法加载时，子加载器才自己加载。好处：保证类的唯一性和安全性（防止自定义 String 类覆盖核心类）。

**Q5: G1 和 CMS 的区别？**
> CMS 是老年代收集器，标记-清除算法，有碎片，并发阶段与应用并发，停顿短但 CPU 敏感。G1 是全堆收集器，分区算法（将堆分成多个 Region），可设置停顿目标（MaxGCPauseMillis），优先回收价值最高的 Region。JDK 9+ G1 成为默认收集器。

**Q6: Full GC 的触发条件有哪些？**
> ① 老年代空间不足；② 元空间/永久代满；③ System.gc()（不一定立即触发，JVM 自己决定）；④ 空间担保失败（Minor GC 前，老年代最大可用连续空间 < 新生代对象总空间）；⑤ CMS GC 时浮动垃圾导致 Concurrent Mode Failure。

**Q7: JIT 编译器做了什么优化？**
> JIT（Just-In-Time）是运行时编译器，将热点字节码（执行频率高的代码）编译为本地机器码（字节码一次编译，后续直接运行）。优化：方法内联（将小方法展开消除调用开销）、逃逸分析（栈上分配/标量替换/锁消除）、热点探测（基于采样或计数器判断热点）。

**Q8: ZGC 和 G1 的根本区别？**
> G1 仍然有 STW（Stop The World）阶段（初始标记、重新标记），且内存整理时需要移动对象。ZGC 通过着色指针和读屏障，在整个 GC 周期内（标记、重定位）**全部并发**，STW 停顿时间 < 1ms，且不产生内存碎片。代价是更高的 CPU 占用。

## 📊 学习状态

- [x] JVM 内存结构（5 大区域）
- [x] 堆分代（Eden / Survivor / Old）
- [x] 对象创建过程
- [x] 对象头结构（Mark Word）
- [x] 4 种引用类型
- [x] 垃圾回收算法（标记-清除/复制/标记-整理）
- [x] 常见垃圾回收器（Serial/Parallel/CMS/G1/ZGC）
- [x] 类加载机制（双亲委派）
- [x] 类加载过程（加载→验证→准备→解析→初始化）
- [x] JVM 常用参数
- [x] OOM 类型与排查
- [ ] JIT 编译器优化细节
- [ ] 常见调优场景实操
- [ ] Arthas 工具实战

## 🐛 踩坑记录

- `static int a = 10` 在**准备阶段赋零值**，在**初始化阶段才赋 10**，容易混淆
- **对象头 Mark Word** 在不同锁状态下内容不同：偏向锁/轻量级锁/重量级锁状态下内容都变了
- **String.intern()**：JDK 7+ 会把字符串放入常量池，JDK 6 则是把引用放入永久代
- **JDK 8 永久代替换为元空间**：静态变量移入堆中，但类元信息在元空间
- **jstat -gcutil** 输出 S0/S1 同时为 0 不代表 Survivor 区没东西，可能是对象还在 From Survivor 等待年龄达标
- **CMS 预留空间**：CMS 并发收集期间会产生浮动垃圾，若老年代满了触发 Concurrent Mode Failure，会退化为 Serial Old，产生长停顿
- **G1 不是严格分代**：G1 的 Region 可以是 Eden/Survivor/Old，比例是动态的，设置了 MaxGCPauseMillis 后 G1 会自动调整
- **JIT 内联**：只有热点方法（调用频率高）才会被 JIT 编译，冷代码不走 JIT
- **Minor GC 后对象进入老年代**：如果 Minor GC 后 Survivor 区放不下，对象会直接进入老年代（空间担保机制）
- **内存泄漏 vs 内存溢出**：内存泄漏（对象无法被 GC root 引用但代码持有）是导致内存溢出的常见原因
