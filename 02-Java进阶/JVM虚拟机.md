---
tags:
  - Java
  - 进阶
  - JVM
  - GC
  - 面试
created: 2026-08-17
---

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

---

## 🔗 相关笔记

- [[Java并发编程|Java 并发编程]]
- [[../01-Java基础/反射|反射]]

