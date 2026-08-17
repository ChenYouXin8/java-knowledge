---
tags:
  - Java
  - 进阶
  - 并发
  - 面试
  - 速查
created: 2026-08-17
---

# Java并发编程 — 速查与面试题

> 从 [[Java并发编程]] 拆分而来，包含对比表格、图解、速查清单、面试题和踩坑记录。

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

---

## 🔗 相关笔记

- [[Java并发编程]]
