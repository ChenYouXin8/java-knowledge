---
tags:
  - Java
  - 进阶
  - 并发
  - 线程
  - 面试
created: 2026-08-17
---

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

---

## 🔗 相关笔记

- [[../01-Java基础/集合框架|集合框架]]
- [[JVM虚拟机|JVM 虚拟机]]

