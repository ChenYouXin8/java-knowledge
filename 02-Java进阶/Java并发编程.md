# Java 并发编程

> Java 并发是面试最高频模块之一，也是实际开发必备技能。本笔记覆盖：线程基础、同步机制、JMM、线程池、并发工具类、CAS、常见并发问题。

---

## 1️⃣ 代码模板

### 1-1 创建线程的 4 种方式

```java
// 方式1：继承 Thread（不推荐，类已继承 Thread 无法再继承其他类）
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Thread running: " + Thread.currentThread().getName());
    }
}
new MyThread().start();  // 注意是 start()，不是 run()

// 方式2：实现 Runnable（最常用，解耦）
class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println("Runnable running: " + Thread.currentThread().getName());
    }
}
new Thread(new MyRunnable(), "MyRunnable").start();

// 方式3：Runnable Lambda（Java 8+，推荐，简写）
new Thread(() -> {
    System.out.println("Lambda Thread: " + Thread.currentThread().getName());
}, "LambdaThread").start();

// 方式4：实现 Callable + FutureTask（带返回值，可抛异常）
class MyCallable implements Callable<Integer> {
    @Override
    public Integer call() throws Exception {
        System.out.println("Callable running...");
        return 42;  // 有返回值
    }
}
FutureTask<Integer> future = new FutureTask<>(new MyCallable());
new Thread(future, "CallableThread").start();
try {
    Integer result = future.get();  // 阻塞等待结果
    System.out.println("Result: " + result);
} catch (InterruptedException | ExecutionException e) {
    e.printStackTrace();
}
```

### 1-2 线程常用方法速查

```java
Thread t = Thread.currentThread();  // 获取当前线程

// 基础属性
t.getName();              // 获取线程名
t.getId();                // 获取线程ID
t.getPriority();           // 获取优先级（1-10，默认5）
t.getState();             // 获取线程状态
t.isAlive();               // 是否存活（start 后未结束）

// 控制方法
t.start();                // 启动线程（调用 run()，不能重复调用）
t.run();                  // 直接调用 run()，不会启动新线程
t.join();                 // 等待线程结束
t.join(2000);             // 最多等 2000ms
t.sleep(1000);            // 休眠（不释放锁）
t.yield();                 // 礼让线程（提示调度器让出 CPU，不保证）
t.interrupt();            // 中断线程（设置中断标志）
t.isInterrupted();        // 检查是否被中断（不清除标志）
Thread.interrupted();     // 检查并清除中断标志

// 线程优先级
t.setPriority(Thread.MAX_PRIORITY);  // 10
t.setPriority(Thread.NORM_PRIORITY); // 5
t.setPriority(Thread.MIN_PRIORITY);  // 1
```

### 1-3 同步方法与同步块

```java
// 同步实例方法（锁对象是 this）
public synchronized void withdraw(int amount) {
    if (balance >= amount) {
        balance -= amount;
        System.out.println("取款成功，余额：" + balance);
    }
}

// 同步静态方法（锁对象是 class 对象）
public static synchronized void staticMethod() {
    System.out.println("静态同步方法");
}

// 同步代码块（锁对象是指定对象，推荐，粒度更细）
public void transfer(Account target, int amount) {
    synchronized (this) {  // 锁当前对象
        if (this.balance >= amount) {
            this.balance -= amount;
            target.balance += amount;
        }
    }
}

// 同步代码块（锁 class 对象，作用同同步静态方法）
public void staticBlock() {
    synchronized (Account.class) {
        System.out.println("锁住整个类");
    }
}

// 注意：静态方法加 synchronized 等价于锁 class 对象
```

### 1-4 Lock 锁模板（ReentrantLock）

```java
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.locks.Condition;

public class Counter {
    private int count = 0;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition condition = lock.newCondition();

    // 标准用法：try-finally 保证释放
    public void increment() {
        lock.lock();  // 加锁
        try {
            count++;
            System.out.println(Thread.currentThread().getName() + ": " + count);
            condition.signalAll();  // 唤醒等待线程
        } finally {
            lock.unlock();  // 必须放在 finally，抛异常也要释放
        }
    }

    public void decrement() {
        lock.lock();
        try {
            while (count == 0) {
                condition.await();  // 等待（会释放锁）
            }
            count--;
            System.out.println(Thread.currentThread().getName() + ": " + count);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    // ReentrantLock 特性：可中断锁
    public void interruptibleLock() {
        try {
            lock.lockInterruptibly();  // 可被 interrupt() 中断的等待
            // 业务逻辑
        } catch (InterruptedException e) {
            System.out.println("被中断");
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

### 1-5 线程池模板

```java
import java.util.concurrent.*;

// 方式1：Executors 工厂方法（阿里的《Java开发手册》禁止用，有OOM风险）
ExecutorService executor1 = Executors.newFixedThreadPool(5);      // 固定线程数
ExecutorService executor2 = Executors.newSingleThreadExecutor();  // 单线程
ExecutorService executor3 = Executors.newCachedThreadPool();     // 可伸缩
ExecutorService executor4 = Executors.newScheduledThreadPool(3); // 定时任务

// 方式2：ThreadPoolExecutor（推荐，自己控制参数）
int corePoolSize = 5;        // 核心线程数（一直保留）
int maxPoolSize = 10;        // 最大线程数
long keepAliveTime = 60L;    // 空闲线程存活时间
TimeUnit unit = TimeUnit.SECONDS;
BlockingQueue<Runnable> workQueue = new LinkedBlockingQueue<>(100);  // 队列容量
ThreadFactory factory = r -> new Thread(r, "MyPool-Thread-" + new Random().nextInt());

ThreadPoolExecutor executor = new ThreadPoolExecutor(
    corePoolSize,
    maxPoolSize,
    keepAliveTime,
    unit,
    workQueue,
    factory,
    new ThreadPoolExecutor.AbortPolicy()  // 拒绝策略：抛RejectedExecutionException
);

// 提交任务
executor.execute(() -> {
    System.out.println("任务执行中：" + Thread.currentThread().getName());
});

// 提交带返回值任务
Future<Integer> future = executor.submit(() -> {
    return 1 + 2;
});
try {
    Integer result = future.get();  // 阻塞获取结果
    Integer result2 = future.get(5, TimeUnit.SECONDS);  // 最多等5秒
} catch (InterruptedException | ExecutionException | TimeoutException e) {
    e.printStackTrace();
}

// 关闭线程池
executor.shutdown();      // 等待任务完成，不再接受新任务
executor.shutdownNow();   // 立即停止，尝试中断正在执行的任务

// 线程池拒绝策略
// AbortPolicy（默认）：抛 RejectedExecutionException
// CallerRunsPolicy：由调用线程执行（任务不会被丢失）
// DiscardPolicy：直接丢弃，不抛异常
// DiscardOldestPolicy：丢弃队列中最老的任务，再重试
```

### 1-6 并发工具类模板

```java
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.Semaphore;
import java.util.concurrent.Exchanger;

// ===== CountDownLatch：倒计时门闩 =====
public void startMultiTasks() throws InterruptedException {
    CountDownLatch latch = new CountDownLatch(3);  // 计数3

    for (int i = 0; i < 3; i++) {
        new Thread(() -> {
            System.out.println("任务" + i + "开始");
            // 模拟工作
            try { Thread.sleep(1000); } catch (InterruptedException ignored) {}
            System.out.println("任务" + i + "完成");
            latch.countDown();  // 计数-1
        }).start();
    }

    latch.await();  // 等待计数归零
    System.out.println("所有任务完成！");
}

// ===== CyclicBarrier：循环栅栏（所有人都到齐再一起走） =====
public void cyclicBarrierDemo() {
    CyclicBarrier barrier = new CyclicBarrier(3, () -> {
        System.out.println("所有人都到了，发车！");
    });

    for (int i = 0; i < 3; i++) {
        final int userId = i;
        new Thread(() -> {
            try {
                System.out.println("用户" + userId + "到达");
                barrier.await();  // 等待其他人
                System.out.println("用户" + userId + "出发");
            } catch (Exception e) { e.printStackTrace(); }
        }).start();
    }
}

// ===== Semaphore：信号量（限流） =====
public void semaphoreDemo() {
    Semaphore semaphore = new Semaphore(2);  // 同时最多2个线程

    for (int i = 0; i < 5; i++) {
        new Thread(() -> {
            try {
                semaphore.acquire();  // 获取许可证（阻塞直到可用）
                System.out.println("线程" + Thread.currentThread().getName() + "开始工作");
                Thread.sleep(2000);
                System.out.println("线程" + Thread.currentThread().getName() + "结束工作");
            } catch (InterruptedException e) { e.printStackTrace(); }
            finally { semaphore.release(); }  // 释放许可证
        }).start();
    }
}

// ===== Exchanger：线程间交换数据 =====
public void exchangerDemo() {
    Exchanger<String> exchanger = new Exchanger<>();

    new Thread(() -> {
        try {
            String data = "数据A";
            System.out.println("线程A准备交换：" + data);
            String received = exchanger.exchange(data);
            System.out.println("线程A收到：" + received);
        } catch (InterruptedException e) { e.printStackTrace(); }
    }).start();

    new Thread(() -> {
        try {
            String data = "数据B";
            System.out.println("线程B准备交换：" + data);
            String received = exchanger.exchange(data);
            System.out.println("线程B收到：" + received);
        } catch (InterruptedException e) { e.printStackTrace(); }
    }).start();
}
```

### 1-7 生产者-消费者模式

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.locks.Condition;

// 使用 Lock + Condition 实现（比 wait/notify 更灵活）
public class ProducerConsumerWithLock {
    private static final int MAX = 10;
    private int count = 0;
    private final Lock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();   // 非满
    private final Condition notEmpty = lock.newCondition();  // 非空

    // 生产者
    public void produce() throws InterruptedException {
        lock.lock();
        try {
            while (count == MAX) {
                notFull.await();  // 满了，等消费
            }
            count++;
            System.out.println("生产后，数量：" + count);
            notEmpty.signalAll();  // 通知消费
        } finally {
            lock.unlock();
        }
    }

    // 消费者
    public void consume() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                notEmpty.await();  // 空了，等生产
            }
            count--;
            System.out.println("消费后，数量：" + count);
            notFull.signalAll();  // 通知生产
        } finally {
            lock.unlock();
        }
    }
}

// 使用 BlockingQueue 实现（最简洁，推荐）
public class ProducerConsumerWithQueue {
    private BlockingQueue<Integer> queue = new LinkedBlockingQueue<>(10);

    public void produce() throws InterruptedException {
        queue.put(1);  // 满则阻塞
        System.out.println("生产完成，当前队列大小：" + queue.size());
    }

    public void consume() throws InterruptedException {
        Integer item = queue.take();  // 空则阻塞
        System.out.println("消费了：" + item);
    }
}
```

### 1-8 volatile 模板

```java
// volatile 保证：1. 可见性 2. 禁止指令重排序
// 不保证原子性！

// 用法1：标志位（适合简单开关）
public class VolatileFlag {
    private volatile boolean running = true;  // volatile！

    public void stop() {
        running = false;  // 其他线程立即看到变化
    }

    public void run() {
        while (running) {
            // 业务逻辑
        }
    }
}

// 用法2：单例模式（双重检查锁）
public class Singleton {
    // 注意：instance 必须是 volatile！
    private static volatile Singleton instance;

    public static Singleton getInstance() {
        if (instance == null) {                // 第1次检查
            synchronized (Singleton.class) {
                if (instance == null) {        // 第2次检查（必须有）
                    instance = new Singleton();  // 分配内存、调用构造、赋值
                }
            }
        }
        return instance;
    }
}

// 不用 volatile 的 new Singleton() 可能重排序：
// 线程A: instance = allocate() → 线程B: if(instance!=null) → 使用未构造的对象！
// 加 volatile 后：禁止构造和赋值指令重排序
```

### 1-9 ThreadLocal 模板

```java
// ThreadLocal：每个线程独立的变量副本
public class ThreadLocalDemo {
    // ThreadLocal 实例
    private static final ThreadLocal<String> threadLocal =
        new ThreadLocal<>();

    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            threadLocal.set("线程1的数据");
            System.out.println("t1: " + threadLocal.get());  // 线程1的数据
        });

        Thread t2 = new Thread(() -> {
            threadLocal.set("线程2的数据");
            System.out.println("t2: " + threadLocal.get());  // 线程2的数据
        });

        t1.start(); t2.start();

        // 清理（防止内存泄漏）
        t1.join(); t2.join();
        threadLocal.remove();  // 手动清理！
    }
}

// ThreadLocal 在线程池中使用要注意清理
public class ThreadLocalInPool {
    private static final ThreadLocal<User> userThreadLocal = new ThreadLocal<>();

    public static void set(User user) {
        userThreadLocal.set(user);
    }

    public static User get() {
        return userThreadLocal.get();
    }

    // 拦截器/过滤器中：
    public void filter(HttpServletRequest request) {
        try {
            User user = parseToken(request);
            set(user);
            // 业务逻辑...
        } finally {
            userThreadLocal.remove();  // 必须在 finally 中清理！
        }
    }
}
```

---

## 2️⃣ 对比表格

### 2-1 synchronized vs Lock

| 维度 | synchronized | ReentrantLock |
|------|-------------|--------------|
| 语法 | 关键字（自动释放） | API（try-finally 手动释放） |
| 加锁方式 | 方法/代码块自动加锁 | `lock.lock()` 手动加锁 |
| 释放方式 | 自动释放（代码块结束/异常） | `lock.unlock()` 手动释放，必须在 finally |
| 锁粒度 | 粗粒度（整个方法/代码块） | 细粒度（可精确控制） |
| 锁超时 | ❌ 不支持 | ✅ `tryLock(time, unit)` |
| 可中断 | ❌ 不支持 | ✅ `lockInterruptibly()` |
| 多条件 | ❌ 只有一个 wait/notify | ✅ `newCondition()` 多个条件队列 |
| 公平锁 | ❌ 非公平（默认） | ✅ 可选公平（`fair=true`） |
| 性能 | JDK 6+ 优化后差距很小 | 稍好（JDK 6+ synchronized 也优化了） |

### 2-2 wait/notify vs Condition

| 维度 | Object wait/notify | Condition |
|------|-------------------|----------|
| 关联锁 | 任意对象 | `Lock.newCondition()` |
| 调用方式 | `obj.wait()` | `condition.await()` |
| 通知方式 | `obj.notify()` / `notifyAll()` | `condition.signal()` / `signalAll()` |
| 条件队列 | 一个（每个对象一个） | 多个（可创建多个 Condition） |
| 典型场景 | 简单互斥 | 生产者-消费者多条件 |

### 2-3 sleep vs wait vs yield

| 方法 | 释放锁 | 阻塞状态 | 等待唤醒 | 典型用途 |
|------|--------|---------|---------|---------|
| `Thread.sleep()` | ❌ 不释放 | TIMED_WAITING | 时间到自动醒 | 暂停执行 |
| `Object.wait()` | ✅ 释放 | WAITING | notify/notifyAll | 等待条件 |
| `Thread.yield()` | ❌ 不释放 | RUNNABLE（重新竞争） | 不等待，直接让出 | 谦让（几乎不用） |

### 2-4 线程状态转换

| 状态 | 说明 | 触发条件 |
|------|------|---------|
| NEW | 新建，未start | `new Thread()` |
| RUNNABLE | 可运行（可能在等CPU） | `start()` |
| BLOCKED | 阻塞，等锁 | 等待 synchronized 锁 |
| WAITING | 无限等待 | `wait()` / `join()` / `LockSupport.park()` |
| TIMED_WAITING | 限时等待 | `sleep(n)` / `wait(n)` / `join(n)` / `await(n)` |
| TERMINATED | 结束 | 任务执行完毕 |

```
┌──────────────────────────────────────────────────────┐
│                                                       │
│  NEW ──→ RUNNABLE ──────────────────────────────────→ TERMINATED
│             │                                          ▲
│             │                                          │
│    ┌────────┴────────┐                               │
│    │                 │                               │
│  BLOCKED          WAITING ←──────────────────────┐  │
│  (等锁)      (wait/join/park)                   │  │
│    │                 │                           │  │
│    │                 └──────────────────────────→ TIMED_WAITING
│    │                                          │  (限时等待)
│    │ 获得锁        notify/signal              │
│    └──────────────┘                          │
│                                                    │
└────────────────────────────────────────────────────┘
```

### 2-5 线程池参数对比

| 参数 | 含义 | 典型值 |
|------|------|---------|
| `corePoolSize` | 核心线程数（始终保留） | CPU核数×2（IO密集型可更大） |
| `maxPoolSize` | 最大线程数 | CPU核数×2 + 1（IO密集型可更大） |
| `keepAliveTime` | 空闲线程存活时间 | 60s |
| `workQueue` | 任务队列 | `LinkedBlockingQueue`(有界) |
| `RejectedPolicy` | 拒绝策略 | `AbortPolicy`(抛异常) |

### 2-6 线程池执行流程

```
提交任务 → 核心线程数未满？ ──是──→ 分配核心线程执行
              │
             否
              ↓
         队列未满？ ──是──→ 进入队列等待
              │
             否
              ↓
      最大线程数未满？ ──是──→ 创建临时线程执行
              │
             否
              ↓
         执行拒绝策略（抛异常/丢弃/调用者执行）
```

---

## 3️⃣ 图解结构

### 3-1 synchronized 锁的是什么

```
synchronized(对象) {
    // 代码块
}

锁住的是：对象头中的 Mark Word（锁标志位）

┌─────────────────────────────────────────────────────┐
│              对象在内存中的结构                      │
├─────────────────────────────────────────────────────┤
│  对象头（Header）                                    │
│  ├── Mark Word（锁、GC状态、哈希码、类型指针）      │
│  └── Klass Pointer（指向类元数据的指针）              │
├─────────────────────────────────────────────────────┤
│  实例数据（Instance Data）                           │
│  ├── 字段值                                         │
│  └── ...                                            │
├─────────────────────────────────────────────────────┤
│  对齐填充（Padding）                                │
└─────────────────────────────────────────────────────┘

Mark Word 锁标志位变化：
无锁 → 偏向锁 → 轻量级锁 → 重量级锁
    （单个线程） （竞争加剧） （重量竞争）
```

### 3-2 volatile 保证可见性原理

```
不用 volatile（可见性问题）：

  线程A（主线程）              线程B（计算线程）
  ┌─────────────┐              ┌─────────────┐
  │ running=true│              │ while(!running)
  │             │              │   ↓
  │ (写缓存/CPU) │              │ 一直循环！不退出！
  └─────────────┘              └─────────────┘
            ↓写入主存                   ↑一直读缓存
          ┌──────────┐               ┌──────────┐
          │  主 存   │               │ CPU缓存  │
          │          │ ← ← ← ← ← ←← │ (旧值)  │
          └──────────┘  不可见       └──────────┘

用 volatile 后：

  线程A                      线程B
  running=true  →  强制写入  →  缓存失效，重新读主存
  (volatile写)     主存       (MESI协议)
```

### 3-3 CAS 原理

```
Compare And Swap（比较并交换）

public class CASDemo {
    // AtomicInteger 内部就是 CAS
    private AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        // do-while 循环，直到成功
        int old, newVal;
        do {
            old = count.get();      // 读取当前值
            newVal = old + 1;       // 计算新值
        } while (!count.compareAndSet(old, newVal)); // CAS
        // 如果 old 和当前值一致，说明没人改过，交换成功
        // 如果不一致，说明有人改过，重新读再试
    }
}

CAS 底层：CPU 提供原子指令（lock cmpxchg）
  操作系统层面：MESI 缓存一致性协议保证可见性
```

### 3-4 线程池内部结构

```
                    ┌──────────────────────────────┐
                    │      ThreadPoolExecutor       │
                    │                               │
                    │  ┌────────────────────────┐  │
                    │  │ 核心线程池 (corePool)  │  │
                    │  │ Thread-0              │  │
                    │  │ Thread-1              │  │
                    │  │ ...                  │  │
                    │  └────────────────────────┘  │
                    │                               │
  submit(task) ────→│  ┌────────────────────────┐  │
                    │  │ 任务队列                │  │
                    │  │ [task1][task2][task3]  │  │
                    │  │ (LinkedBlockingQueue) │  │
                    │  └────────────────────────┘  │
                    │                               │
                    │  ┌────────────────────────┐  │
                    │  │ 最大线程池 (maxPool)   │  │
                    │  │ Thread-3 (临时线程)    │  │
                    │  │ Thread-4 (临时线程)    │  │
                    │  └────────────────────────┘  │
                    └──────────────────────────────┘

拒绝策略触发：队列满 + 最大线程都在忙
```

### 3-5 死锁产生条件

```
死锁 = 4 个必要条件同时满足：

┌─────────────────────────────────────────────────────┐
│  ① 互斥条件                                         │
│     资源只能被一个线程占用                           │
│     示例：synchronized(obj) 锁对象只能被一个线程持有  │
├─────────────────────────────────────────────────────┤
│  ② 请求并持有                                        │
│     持有已有资源，同时请求其他资源                   │
│     示例：线程A持有lock1，还想获取lock2              │
├─────────────────────────────────────────────────────┤
│  ③ 不可抢占                                         │
│     资源不能被强制从线程中剥夺                        │
│     示例：synchronized 锁不能抢，只能等释放          │
├─────────────────────────────────────────────────────┤
│  ④ 循环等待                                         │
│     形成循环链：A等B，B等C，C等A                    │
│     示例：线程A等线程B的锁，线程B等线程A的锁         │
└─────────────────────────────────────────────────────┘

如何破坏死锁（让任意一个条件不成立即可）：
  - 破坏②：一次性申请所有资源，不边持边等
  - 破坏③：尝试获取锁，超时放弃
  - 破坏④：按固定顺序获取锁
```

---

## 4️⃣ 速查清单

### 4-1 synchronized 速查

| 用法 | 锁对象 | 锁住范围 |
|------|--------|---------|
| `synchronized void m()` | `this`（当前对象） | 整个方法 |
| `synchronized static void m()` | `类.class` | 整个静态方法 |
| `synchronized (this)` | `this` | 代码块 |
| `synchronized (obj)` | 指定对象 | 代码块 |
| `synchronized (Class.class)` | `类.class` | 代码块 |

### 4-2 ReentrantLock 速查

```java
ReentrantLock lock = new ReentrantLock(boolean fair);  // fair=true 公平锁

lock.lock();              // 加锁（阻塞）
lock.tryLock();          // 尝试获取（非阻塞）
lock.tryLock(5, TimeUnit.SECONDS);  // 带超时
lock.lockInterruptibly(); // 可中断锁
lock.unlock();           // 解锁（必须在 finally）

Condition c = lock.newCondition();
c.await();               // 等待（相当于 wait）
c.signal();             // 唤醒一个（相当于 notify）
c.signalAll();          // 唤醒全部（相当于 notifyAll）
```

### 4-3 BlockingQueue 速查

| 方法 | 队列满/空时 | 队列满/空时 |
|------|------------|------------|
| `add()` | 抛异常 | 抛异常 |
| `offer()` | 返回 false | 返回 false |
| `put()` | **阻塞**等待 | **阻塞**等待 |
| `offer(timeout)` | 等待超时后返回 false | 等待超时后返回 false |

| 方法 | 描述 |
|------|------|
| `take()` | 取走头部，若空则阻塞 |
| `poll(timeout)` | 取走头部，超时返回 null |

**常用实现：**
- `ArrayBlockingQueue(int capacity)` — 有界，数组
- `LinkedBlockingQueue(int capacity)` — 有界/无界，链表（默认无界Integer.MAX_VALUE）
- `PriorityBlockingQueue` — 无界，按优先级
- `SynchronousQueue` — 不存储，put 必须等 take

### 4-4 并发工具类速查

| 工具类 | 作用 | 关键方法 |
|--------|------|---------|
| `CountDownLatch` | 倒计时闩 | `countDown()`、`await()` |
| `CyclicBarrier` | 循环栅栏 | `await()` |
| `Semaphore` | 限流 | `acquire()`、`release()` |
| `Exchanger` | 线程间交换 | `exchange()` |
| `CompletableFuture` | 异步编排 | `thenApply()`、`thenCompose()`、`join()` |

### 4-5 JUC 原子类速查

| 类 | 对应类型 | 操作 |
|------|---------|------|
| `AtomicInteger` | int | `incrementAndGet()`、`getAndIncrement()` |
| `AtomicLong` | long | `addAndGet(delta)` |
| `AtomicBoolean` | boolean | `compareAndSet(expected, new)` |
| `AtomicReference<V>` | 引用类型 | `compareAndSet(expected, new)` |
| `AtomicIntegerArray` | int[] | 数组元素的原子操作 |
| `LongAdder` | long（高频累加） | `increment()`、`sum()`，比 AtomicLong 性能好 |

---

## 5️⃣ 场景选择器

### 5-1 该用哪种同步方式？

```
需要同步的原因是什么？
         │
    ┌────┴──────────────┐
    │                   │
  简单互斥             需要灵活控制
  （方法/代码块）        │
    │                ┌──┴──────────────┐
  synchronized      需要公平锁？      需要多条件？
    │                │                 │
   是               ┌┴──┐            ReentrantLock
                    │是 │否          newCondition()
                   ┌┴───┴┐
                   │公平 │非公平
                   │     │
             ReentrantLock   ReentrantLock
             (fair=true)   (fair=false)
```

### 5-2 该用哪种线程池？

```
任务类型是什么？
         │
    ┌────┴──────────────┐
    │                   │
  CPU 密集型           IO 密集型
  (计算/逻辑)           (等待网络/IO)
    │                   │
  线程数 =             线程数 =
  CPU核数 + 1          CPU核数 × 2
  (或 CPU核数×CPU利用率)  （或更大，建议测试调优）
    │
最大线程数 ≈ 核心线程数
队列选有界（防止OOM）
    │
推荐：newFixedThreadPool
      或 ThreadPoolExecutor(核心=max=CPU+1)
```

```
任务特性是什么？
         │
    ┌────┴──────────────────┐
    │                       │
  短期异步任务            定时/周期任务
    │                       │
  ThreadPoolExecutor      ScheduledThreadPoolExecutor
  (核心数可伸缩)           (newScheduledThreadPool)
```

### 5-3 该用哪个并发工具？

```
业务场景是什么？
         │
    ┌────┴──────────────────────────────────┐
    │                                       │
  等 N 个任务都完成              多线程同时工作
  再继续？                       要协调？
    │                                │
CountDownLatch                  ┌──┴─────────────┐
                               │                 │
                           限流（同时N个）      多人集合点
                           资源？               再一起走？
                             │                   │
                       Semaphore           CyclicBarrier
                       (acquire/release)   (await)
```

---

## ❓ 常见面试题

**Q1: synchronized 锁的到底是什么？**
> 锁的是对象头中的 Mark Word。加在实例方法上锁 `this` 对象；加在静态方法上锁 `Class` 对象；加在代码块上锁括号里的对象。

**Q2: synchronized 和 ReentrantLock 的区别？**
> 详见上方对比表格 2-1。核心区别：ReentrantLock 支持公平/非公平、可超时、可中断、多条件队列。

**Q3: volatile 和 synchronized 的区别？**
> volatile 不保证原子性，只保证**可见性**和**禁止指令重排序**。synchronized 保证**原子性 + 可见性**。

**Q4: ThreadLocal 内存泄漏是怎么回事？**
> ThreadLocalMap 的 Entry 继承 WeakReference（弱引用 key），但 value 是强引用。线程复用时如果忘了 remove()，value 不会被回收导致泄漏。**用完必须 remove()**。

**Q5: 线程池大小怎么设置？**
> CPU 密集型：`核心线程数 = CPU核数 + 1`；IO 密集型：`核心线程数 = CPU核数 × CPU利用率 × (1 + 平均等待时间/平均工作时间)`。实际需要压测调优。

**Q6: 如何排查死锁？**
> JStack 命令：`jstack <pid>`，会直接报告 Deadlock found。其他线程分析工具（JConsole、VisualVM）也能看到。

**Q7: happens-before 是什么？**
> Java 内存模型规定的 8 条规则：**程序顺序规则**（单线程内按代码顺序）、**volatile 规则**、**synchronized 规则**（unlock 先于 lock）、**线程启动规则**（start 先于 run）、**线程终止规则**（join 返回先于其他线程看到结果）、**传递性**、**中断规则**、**终结规则**。

## 📊 学习状态

- [x] 创建线程（Thread/Runnable/Callable）
- [x] synchronized（方法/代码块）
- [x] Lock（ReentrantLock）
- [x] wait/notify/notifyAll
- [x] volatile
- [x] ThreadLocal
- [x] JMM & happens-before
- [x] 线程池（ThreadPoolExecutor）
- [x] 并发工具类
- [x] CAS 原理
- [ ] 常见并发问题（死锁、活锁、饥饿）
- [ ] 生产者-消费者模式

## 🐛 踩坑记录

- `Thread.sleep()` **不释放锁**
- `wait()` **必须**在 `synchronized` 中调用，否则抛 `IllegalMonitorStateException`
- `Thread.start()` 只能调用**一次**，多次调用抛 `IllegalThreadStateException`
- `volatile` **不保证原子性**（i++ 这种复合操作不是原子的）
- `ThreadLocal` **用完必须 remove()**，否则内存泄漏
- `ReentrantLock` **必须在 finally 中 unlock()**，否则异常时不释放
- 线程池 **拒绝策略**要根据业务选，线上别用 `DiscardPolicy`（丢任务无感知）
- `CountDownLatch` 只能使用**一次**，`CyclicBarrier` 可重用
- `CompletableFuture` 默认用 **ForkJoinPool.commonPool()**，适合异步任务链编排
