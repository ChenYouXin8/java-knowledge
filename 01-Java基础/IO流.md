---
tags:
  - Java
  - 基础
  - IO
  - NIO
created: 2026-08-17
---

# IO 流

> 文件读写、网络传输、字节流 vs 字符流。**BIO / NIO / AIO** 三种模式。

---

## 1️⃣ 代码模板

### 1-1 文件读写模板（4种场景）

```java
// 场景1：按字节读文件（适合二进制文件）
try (FileInputStream fis = new FileInputStream("data.bin")) {
    int data;
    while ((data = fis.read()) != -1) {
        System.out.print((char) data);
    }
}

// 场景2：按字节数组读（推荐，大文件高效）
try (FileInputStream fis = new FileInputStream("data.bin")) {
    byte[] buf = new byte[8192];  // 8KB 缓冲区
    int len;
    while ((len = fis.read(buf)) != -1) {
        System.out.write(buf, 0, len);  // 写到标准输出
    }
}

// 场景3：按字符读文本文件
try (BufferedReader br = new BufferedReader(
        new InputStreamReader(new FileInputStream("test.txt"), "UTF-8"))) {
    String line;
    while ((line = br.readLine()) != null) {
        System.out.println(line);
    }
}

// 场景4：文件复制
try (InputStream in = new FileInputStream("source.jpg");
     OutputStream out = new FileOutputStream("dest.jpg")) {
    byte[] buf = new byte[8192];
    int len;
    while ((len = in.read(buf)) != -1) {
        out.write(buf, 0, len);
    }
}
```

### 1-2 序列化 / 反序列化模板

```java
// 实体类必须实现 Serializable
public class User implements Serializable {
    private static final long serialVersionUID = 1L;  // 版本号必须声明
    private String name;
    private transient int age;  // transient：不序列化

    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }
}

// 序列化（对象 → 文件）
public static void serialize(User user) throws Exception {
    try (ObjectOutputStream oos = new ObjectOutputStream(
            new FileOutputStream("user.bin"))) {
        oos.writeObject(user);
        System.out.println("序列化成功");
    }
}

// 反序列化（文件 → 对象）
public static User deserialize() throws Exception {
    try (ObjectInputStream ois = new ObjectInputStream(
            new FileInputStream("user.bin"))) {
        return (User) ois.readObject();
    }
}

// 注意：serialVersionUID 不一致会抛 InvalidClassException
```

### 1-3 NIO Channel + Buffer 模板

```java
// 文件复制（Channel 方式）
try (FileChannel in = new FileInputStream("source.txt").getChannel();
     FileChannel out = new FileOutputStream("dest.txt").getChannel()) {

    ByteBuffer buf = ByteBuffer.allocate(8192);

    while (in.read(buf) != -1) {
        buf.flip();  // 切换为读模式
        out.write(buf);
        buf.clear();  // 清空缓冲区，继续写
    }
}

// 内存映射文件（适合大文件随机读写）
try (RandomAccessFile raf = new RandomAccessFile("big.bin", "rw");
     FileChannel fc = raf.getChannel()) {

    MappedByteBuffer mbb = fc.map(FileChannel.MapMode.READ_WRITE, 0, 1024);
    mbb.put(0, (byte) 'A');    // 直接改内存
    byte b = mbb.get(0);       // 直接读内存
}
```

### 1-4 Properties / NIO Files 工具模板

```java
// Properties 配置文件读写
Properties props = new Properties();

// 加载配置文件
try (InputStream in = new FileInputStream("config.properties")) {
    props.load(in);
    String dbUrl = props.getProperty("db.url");
    String dbUser = props.getProperty("db.user");
    System.out.println(dbUrl + ", " + dbUser);
}

// 写入配置
props.setProperty("app.name", "chen-ai-agent");
props.setProperty("app.version", "1.0.0");
try (OutputStream out = new FileOutputStream("config.properties")) {
    props.store(out, "Application Config");
}

// NIO Files（Java 7+，最简洁）
List<String> lines = Files.readAllLines(Paths.get("test.txt"), StandardCharsets.UTF_8);
Files.writeString(Paths.get("out.txt"), "hello nio", StandardOpenOption.CREATE);
Files.copy(Paths.get("src.txt"), Paths.get("dst.txt"), StandardCopyOption.REPLACE_EXISTING);
Files.deleteIfExists(Paths.get("tmp.txt"));
```

---

## 2️⃣ 对比表格

### 2-1 BIO / NIO / AIO 对比

| 维度 | BIO（同步阻塞） | NIO（同步非阻塞） | AIO（异步非阻塞） |
|------|--------------|----------------|----------------|
| 线程模型 | 1 连接 1 线程 | 1 线程多连接 | 回调通知 |
| 核心 API | Stream | Channel + Buffer + Selector | AsynchronousChannelGroup |
| 适合场景 | 连接少、低并发 | 高并发、短连接 | 高并发、长连接 |
| 复杂度 | 简单 | 中等 | 较复杂 |
| Java 版本 | 1.0 | 1.4 | 1.7 |
| 阻塞 | 全部阻塞 | 非阻塞（Selector） | 回调通知 |

### 2-2 字节流 vs 字符流

| 维度 | 字节流 | 字符流 |
|------|--------|--------|
| 父类 | `InputStream`/`OutputStream` | `Reader`/`Writer` |
| 处理单位 | 字节（byte） | 字符（char，自动编码转换） |
| 适用场景 | 二进制文件（图片/音频/压缩包） | 文本文件 |
| 编码处理 | 无 | 自动按编码转换 |
| 性能 | 较快 | 缓冲流更优 |
| 典型类 | `FileInputStream`/`BufferedInputStream` | `FileReader`/`BufferedReader` |

### 2-3 缓冲流 vs 非缓冲流

| 维度 | 非缓冲流 | 缓冲流 |
|------|---------|--------|
| 每次读写 | 直接操作底层 | 先放缓冲区，满或 flush 才 IO |
| 性能 | 差（频繁磁盘访问） | 好（批量 IO） |
| `read()` | 每次读 1 字节 | 每次读一批（默认 8192 字节）|
| 需手动 flush | 否（close 自动） | `flush()` 后立即写出 |
| 推荐场景 | 偶尔读写 | **日常开发首选** |

---

## 3️⃣ 图解结构

### 3-1 IO 体系架构

```
                    ┌──────────────────────┐
                    │     InputStream       │
                    │     OutputStream      │
                    │      (抽象类)         │
                    └──────────┬───────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                       │
┌───────▼───────┐     ┌───────▼───────┐      ┌───────▼────────┐
│ FileInput/    │     │ FilterInput/  │      │ DataInput/     │
│ OutputStream  │     │ FilterOutput  │      │ DataOutputStream│
│ (文件节点)    │     │ (装饰器)       │      │ (数据流)        │
└───────────────┘     └───────┬───────┘      └────────────────┘
        │                      │
        │           ┌──────────┼──────────┐
        │           │                     │
        │    ┌──────▼──────┐    ┌───────▼───────┐
        │    │ BufferedInput/│    │  GZIPInput/   │
        │    │ BufferedOutput│    │  ZipInputStream│
        │    │ (缓冲装饰器)  │    │ (压缩流)       │
        │    └──────────────┘    └────────────────┘
        │
        └──────────┐
                   │
         ┌─────────▼─────────┐
         │  ObjectInput/     │
         │  ObjectOutput     │
         │  (对象序列化流)   │
         └───────────────────┘
```

### 3-2 NIO 三组件关系

```
┌─────────────────────────────────────────────────┐
│                    Selector                      │
│             （多路复用器，单线程监听多个 Channel） │
└─────────────────────────────────────────────────┘
            ▲ 监听             ▲ 监听
            │                  │
    ┌───────┴───────┐  ┌──────┴──────┐
    │   Channel     │  │   Channel   │
    │  (通道/连接)   │  │  (另一个连接) │
    └───────┬───────┘  └──────┬──────┘
            │ 读写数据           │ 读写数据
    ┌───────▼───────┐  ┌───────▼──────┐
    │    Buffer     │  │    Buffer    │
    │ (内存缓冲区)   │  │ (内存缓冲区)  │
    │               │  │               │
    │ position      │  │ position     │
    │ limit         │  │ limit        │
    │ capacity      │  │ capacity     │
    └───────────────┘  └──────────────┘
```

---

## 4️⃣ 速查清单

### 4-1 常用 IO 类速查

| 类 | 用途 | 是否缓冲 |
|------|------|---------|
| `FileInputStream` | 按字节读文件 | ❌ |
| `FileOutputStream` | 按字节写文件 | ❌ |
| `FileReader` | 按字符读文本 | ❌ |
| `FileWriter` | 按字符写文本 | ❌ |
| `BufferedReader` | 按行读文本（`readLine()`） | ✅ |
| `BufferedWriter` | 写文本（`newLine()`） | ✅ |
| `BufferedInputStream` | 字节缓冲读 | ✅ |
| `ObjectInputStream` | 反序列化 | ❌ |
| `ObjectOutputStream` | 序列化 | ❌ |
| `DataInputStream` | 读写基本类型/UTF | ❌ |
| `InputStreamReader` | 字节→字符转换桥 | ❌ |

### 4-2 NIO Buffer 状态指针速查

| 指针 | 含义 |
|------|------|
| `position` | 当前读写位置 |
| `limit` | 有效数据边界（写完/读完切换时设置） |
| `capacity` | 缓冲区总容量 |
| `flip()` | `limit=position, position=0`，切换读写 |
| `clear()` | `position=0, limit=capacity`，清空缓冲区 |
| `compact()` | 压缩：把未读数据移到开头，继续写 |

---

## 5️⃣ 场景选择器

### 5-1 该用哪种 IO 类？

```
读取的文件类型？
         │
    ┌────┴─────┐
    │           │
  文本文件    二进制文件
    │           │
  字符流？    字节流
    │           │
  Buffered  ┌──┴────────┐
  Reader   对象流        普通字节流
  (readLine)  │         │
    │    implements   ┌─┴────────┐
    │    Serializable  │          │
    │         │       普通数据    │
    │         │          │        │
    │    ObjectInput  DataInput
    │    Stream       Stream
```

### 5-2 BIO / NIO / AIO 选择

```
并发量多少？
         │
    ┌────┴──────┐
    │            │
  低（<1000）    高（>1000）
    │            │
  BIO            连接类型？
  (简单)           │
              ┌───┴────────┐
              │             │
           短连接          长连接
              │             │
           NIO             AIO
         (Reactor)       (Proactor)
```

---

## ❓ 常见面试题

**Q1: BIO / NIO / AIO 的区别？**
> 详见上方对比表格 2-1。

**Q2: `flush()` 什么时候必须调用？**
> `BufferedWriter`/`BufferedOutputStream` 先写缓冲区，**close() 会自动 flush**，只写不关时要手动调。

**Q3: 为什么序列化要定义 `serialVersionUID`？**
> JVM 用它判断类的版本一致性。不定义会由类结构自动生成，修改类后 UID 变化导致反序列化失败。

**Q4: `transient` 字段有什么特点？**
> 序列化时被忽略，如密码、敏感信息不应序列化。

**Q5: 装饰器模式和继承的区别？**
> 装饰器运行时动态组合，继承编译时静态决定。`BufferedReader` 包装 `FileReader` 就是装饰器。

## 📊 学习状态

- [x] 字节流 / 字符流
- [x] 缓冲流
- [x] 转换流
- [x] 对象流（序列化）
- [x] NIO（Channel/Buffer/Selector）
- [x] 代码模板 / 速查清单

## 🐛 踩坑记录

- `FileWriter` 写完要 `flush()` 或 `close()`，否则内容丢失
- 字符流读中文要指定编码，否则乱码
- 读写对象要保证序列化版本号一致
- `BufferedReader.readLine()` 不包含换行符，需手动加
- `InputStreamReader` 是字节到字符的转换桥，底层还是要套 `FileInputStream`


---

## 🔗 相关笔记

- [[异常处理|异常处理]]
- [[集合框架|集合框架]]
