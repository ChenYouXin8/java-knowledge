---
tags:
  - MyBatis-Plus
  - ORM
  - 数据访问
  - 面试
created: 2026-08-22
---

# MyBatis-Plus 实战

> MyBatis-Plus（MP）是 MyBatis 的增强工具，**不修改 MyBatis 本体**，只做增强。省去简单 CRUD 的 XML 编写，企业开发几乎必配。

---

## 一、快速开始

### 1.1 Spring Boot 整合

**依赖**：
```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.5.5</version>
</dependency>
```

> 引入 MP 后**不要**再引入 mybatis-spring-boot-starter，否则冲突。

**配置**：
```yaml
# application.yml
mybatis-plus:
  configuration:
    map-underscore-to-camel-case: true  # 下划线转驼峰
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl  # 打印 SQL
  global-config:
    db-config:
      id-type: assign_id        # 雪花 ID（默认）
      logic-delete-field: deleted  # 逻辑删除字段
      logic-delete-value: 1
      logic-not-delete-value: 0
```

### 1.2 实体类

```java
@Data
@TableName("user")          // 指定表名
public class User {
    @TableId(type = IdType.ASSIGN_ID)  // 雪花 ID
    private Long id;
    private String name;
    private Integer age;
    @TableField("create_time")        // 字段名映射
    private LocalDateTime createTime;
    @TableField(exist = false)        // 非数据库字段
    private String extra;
    @Version                            // 乐观锁
    private Integer version;
    @TableLogic                          // 逻辑删除
    private Integer deleted;
}
```

---

## 二、BaseMapper 通用 CRUD

继承 BaseMapper 直接获得全套 CRUD，零 XML：

```java
public interface UserMapper extends BaseMapper<User> {
    // 自定义复杂查询仍可写 XML
}
```

### 2.1 常用方法

```java
@Autowired
private UserMapper userMapper;

// 增
userMapper.insert(user);                          // 插入

// 删
userMapper.deleteById(1L);                        // 按 ID 删
userMapper.deleteByIds(Arrays.asList(1L, 2L));    // 批量删
userMapper.deleteByMap(map);                      // 按条件删

// 改
userMapper.updateById(user);                      // 按 ID 改

// 查
userMapper.selectById(1L);                        // 按 ID 查
userMapper.selectByIds(Arrays.asList(1L, 2L));    // 批量查
userMapper.selectList(null);                      // 查全部
userMapper.selectCount(null);                     // 计数
userMapper.selectList(QueryWrapper);              // 条件查
```

---

## 三、条件构造器（核心）

### 3.1 QueryWrapper

```java
QueryWrapper<User> wrapper = new QueryWrapper<>();
wrapper.eq("name", "张三")          // name = '张三'
       .ne("status", 0)             // status != 0
       .gt("age", 18)               // age > 18
       .ge("age", 18)               // age >= 18
       .lt("age", 60)               // age < 60
       .like("name", "张")          // name LIKE '%张%'
       .likeLeft("name", "三")       // name LIKE '%三'
       .likeRight("name", "张")      // name LIKE '张%'
       .between("age", 18, 30)      // age BETWEEN 18 AND 30
       .in("status", 1, 2, 3)       // status IN (1,2,3)
       .isNull("email")             // email IS NULL
       .orderByDesc("create_time")  // ORDER BY create_time DESC
       .last("LIMIT 10");           // 末尾拼接 SQL

List<User> users = userMapper.selectList(wrapper);
```

### 3.2 LambdaQueryWrapper（推荐）

避免硬编码列名，编译期检查：

```java
LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
wrapper.eq(User::getName, "张三")
       .gt(User::getAge, 18)
       .like(User::getName, "张")
       .orderByDesc(User::getCreateTime);

List<User> users = userMapper.selectList(wrapper);
```

### 3.3 条件构造器速查表

| 方法 | SQL | 说明 |
|---|---|---|
| eq | = | 等于 |
| ne | <> | 不等于 |
| gt | > | 大于 |
| ge | >= | 大于等于 |
| lt | < | 小于 |
| le | <= | 小于等于 |
| like | LIKE '%x%' | 模糊 |
| likeLeft | LIKE '%x' | 左模糊 |
| likeRight | LIKE 'x%' | 右模糊 |
| between | BETWEEN | 范围 |
| in | IN | 集合 |
| isNull | IS NULL | 空值 |
| orderByDesc | ORDER BY DESC | 降序 |
| groupBy | GROUP BY | 分组 |
| having | HAVING | 聚合过滤 |
| last | 末尾拼接 | 加 LIMIT 等 |

---

## 四、分页插件

### 4.1 配置

```java
@Configuration
public class MybatisPlusConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(
            new PaginationInnerInterceptor(DbType.MYSQL)
        );
        return interceptor;
    }
}
```

### 4.2 使用

```java
Page<User> page = new Page<>(1, 10);  // 第1页，每页10条
userMapper.selectPage(page, null);

page.getRecords();   // 当前页数据
page.getTotal();     // 总条数
page.getPages();     // 总页数
page.getCurrent();   // 当前页码
page.getSize();       // 每页条数
```

---

## 五、高级特性

### 5.1 自动填充

```java
@Component
public class MetaObjectHandler implements com.baomidou.mybatisplus.core.handlers.MetaObjectHandler {
    @Override
    public void insertFill(MetaObject metaObject) {
        this.strictInsertFill(metaObject, "createTime", LocalDateTime.class, LocalDateTime.now());
        this.strictInsertFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        this.strictUpdateFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
    }
}
```

```java
@TableField(fill = FieldFill.INSERT)
private LocalDateTime createTime;

@TableField(fill = FieldFill.INSERT_UPDATE)
private LocalDateTime updateTime;
```

### 5.2 逻辑删除

配置后，DELETE 变成 UPDATE：

```yaml
mybatis-plus:
  global-config:
    db-config:
      logic-delete-field: deleted
      logic-delete-value: 1
      logic-not-delete-value: 0
```

```java
userMapper.deleteById(1L);
// 实际执行: UPDATE user SET deleted=1 WHERE id=1

// 查询自动过滤已删除
userMapper.selectList(null);
// 实际执行: SELECT * FROM user WHERE deleted=0
```

### 5.3 乐观锁

```java
@Version
private Integer version;
```

```java
// 更新时自动检查版本
User user = userMapper.selectById(1L);
user.setName("新名字");
userMapper.updateById(user);
// 实际执行: UPDATE user SET name='新名字', version=version+1
//           WHERE id=1 AND version=原版本号
```

### 5.4 代码生成器

```java
AutoGenerator generator = new AutoGenerator(dataSource);
generator.strategy()
    .entityBuilder()
    .enableLombok()
    .enableTableFieldAnnotation()
    .build();
generator.execute();
```

---

## 六、IService 业务层封装

```java
public interface UserService extends IService<User> {
    // 自定义业务方法
}

@Service
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService {
    // 继承全套业务方法
}
```

```java
userService.save(user);                    // 插入
userService.saveBatch(userList);          // 批量插入
userService.updateById(user);             // 更新
userService.removeById(1L);                // 删除
userService.getById(1L);                   // 查询
userService.list();                        // 查全部
userService.page(new Page<>(1, 10));      // 分页
userService.lambdaQuery()
    .eq(User::getName, "张三")
    .list();                               // 链式条件查询
```

---

## 七、面试高频题

**Q1: MyBatis-Plus 和 MyBatis 的关系？**
> MP 是 MyBatis 的增强工具，不修改 MyBatis 本体。提供通用 CRUD、条件构造器、分页插件、代码生成器等。引入 MP 仍可写 XML。

**Q2: QueryWrapper 和 LambdaQueryWrapper 的区别？**
> QueryWrapper 用字符串列名，LambdaQueryWrapper 用方法引用（User::getName），编译期检查避免写错列名，推荐后者。

**Q3: MyBatis-Plus 分页原理？**
> PaginationInnerInterceptor 拦截 SQL，自动追加 LIMIT 和 COUNT 查询。先查总数判断是否溢出，再执行分页 SQL。

**Q4: 逻辑删除和物理删除？**
> 物理删除 DELETE FROM 直接删行。逻辑删除 UPDATE SET deleted=1 标记删除，查询自动过滤。MP 配置后自动处理。

**Q5: MyBatis-Plus 怎么实现乐观锁？**
> 实体类加 @Version 字段，配置 OptimisticLockerInnerInterceptor。更新时自动在 WHERE 条件加 version 检查并 +1。

---

## 🔗 相关笔记

- [[MyBatis核心概念|MyBatis 核心概念]] — 动态 SQL / 关联查询 / 缓存
- [[../03-Database/MySQL数据库|MySQL 数据库]] — 底层数据库
- [[../04-Spring生态/SpringBoot-数据访问|Spring Boot 数据访问]] — Spring Boot 整合配置
- [[../07-面试与项目/Java学习情况分析与实习冲刺计划|学习计划]] — 第二层会用+懂流程