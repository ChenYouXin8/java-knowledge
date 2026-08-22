---
tags:
  - MyBatis
  - ORM
  - 数据访问
  - 面试
created: 2026-08-22
---

# MyBatis 核心概念

> MyBatis 是 Java 持久层框架，用 XML 或注解将 Java 对象与 SQL 映射。企业 90% 的 Java 项目使用 MyBatis。

---

## 一、MyBatis 是什么

- **半自动 ORM**：SQL 自己写，结果集自动映射
- 对比 JPA/Hibernate（全自动 ORM，自动生成 SQL）
- 优势：SQL 灵活可控、容易优化、适合复杂查询
- 劣势：简单 CRUD 也要写 SQL（MyBatis-Plus 解决了这个问题）

| | MyBatis | JPA/Hibernate | JDBC |
|---|---|---|---|
| SQL 控制 | 手写，灵活 | 自动生成 | 纯手写 |
| 对象映射 | 自动 | 自动 | 手动 |
| 学习成本 | 中 | 高 | 低 |
| 企业使用 | 最多 | 中 | 少 |
| 适合场景 | 复杂业务查询 | 简单 CRUD | 底层定制 |

---

## 二、核心组件

### 2.1 三大核心对象

```
SqlSessionFactory  ← 全局唯一，读取 mybatis-config.xml
    ↓ 创建
SqlSession         ← 一次会话（请求级），执行 SQL、管理事务
    ↓ 获取
Mapper 接口代理     ← MyBatis 动态生成接口实现
```

### 2.2 执行流程

```
1. 读取 mybatis-config.xml → 构建 SqlSessionFactory
2. 打开 SqlSession
3. 获取 Mapper 接口代理
4. 执行 SQL（XML 中定义的）
5. 结果集自动映射为 Java 对象
6. 提交事务 → 关闭 SqlSession
```

### 2.3 配置文件结构

**mybatis-config.xml**（核心配置）：
```xml
<configuration>
    <!-- 数据库连接 -->
    <environments default="dev">
        <environment id="dev">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3306/mydb"/>
                <property name="username" value="root"/>
                <property name="password" value="123456"/>
            </dataSource>
        </environment>
    </environments>
    <!-- Mapper 映射文件 -->
    <mappers>
        <mapper resource="mapper/UserMapper.xml"/>
    </mappers>
</configuration>
```

> Spring Boot 整合后，这些配置写到 application.yml，不用手写 XML 配置文件。

---

## 三、Mapper XML 详解

### 3.1 基本结构

```xml
<mapper namespace="com.example.mapper.UserMapper">

    <!-- 查询 -->
    <select id="findById" resultType="User" parameterType="long">
        SELECT id, name, age FROM user WHERE id = #{id}
    </select>

    <!-- 新增 -->
    <insert id="insert" parameterType="User" useGeneratedKeys="true" keyProperty="id">
        INSERT INTO user (name, age) VALUES (#{name}, #{age})
    </insert>

    <!-- 修改 -->
    <update id="update" parameterType="User">
        UPDATE user SET name=#{name}, age=#{age} WHERE id=#{id}
    </update>

    <!-- 删除 -->
    <delete id="deleteById" parameterType="long">
        DELETE FROM user WHERE id = #{id}
    </delete>
</mapper>
```

### 3.2 #{} vs ${}

| | #{} | ${} |
|---|---|---|
| 底层 | PreparedStatement 预编译 | Statement 字符串拼接 |
| 安全 | **防 SQL 注入** | 有注入风险 |
| 用途 | 参数值传递 | 表名/列名传递 |

```xml
<!-- 安全：预编译占位 -->
<select id="findById">SELECT * FROM user WHERE id = #{id}</select>

<!-- 危险：直接拼接 -->
<select id="findByTable">SELECT * FROM ${tableName}</select>
```

> 面试必问：**为什么 #{} 能防 SQL 注入？** 因为底层用 PreparedStatement，参数用占位符 ? 传入，SQL 结构已固定，参数不会改变 SQL 语义。

### 3.3 resultMap 结果映射

当数据库列名和 Java 属性名不一致时：

```xml
<resultMap id="userResultMap" type="User">
    <id property="id" column="user_id"/>          <!-- 主键 -->
    <result property="name" column="user_name"/>  <!-- 普通列 -->
    <result property="age" column="user_age"/>
</resultMap>

<select id="findAll" resultMap="userResultMap">
    SELECT user_id, user_name, user_age FROM users
</select>
```

> 也可以用别名解决：`SELECT user_name AS name FROM users`，但 resultMap 更灵活。

---

## 四、动态 SQL（MyBatis 杀手锏）

### 4.1 常用动态标签

| 标签 | 作用 | 类似 |
|---|---|---|
| `<if>` | 条件判断 | Java if |
| `<where>` | 自动加 WHERE，去掉首部 AND/OR | — |
| `<choose>` | 多条件分支 | switch-case |
| `<foreach>` | 遍历集合 | for 循环 |
| `<set>` | 自动加 SET，去掉末尾逗号 | — |
| `<trim>` | 自定义前后缀裁剪 | — |

### 4.2 if + where 示例

```xml
<select id="search" resultType="User">
    SELECT * FROM user
    <where>
        <if test="name != null and name != ''">
            AND name LIKE CONCAT('%', #{name}, '%')
        </if>
        <if test="age != null">
            AND age = #{age}
        </if>
        <if test="status != null">
            AND status = #{status}
        </if>
    </where>
</select>
```

> `<where>` 会自动去掉第一个多余的 AND/OR，没有条件时不会生成 WHERE 子句。

### 4.3 foreach 批量操作

```xml
<!-- 批量查询 -->
<select id="findByIds" resultType="User">
    SELECT * FROM user WHERE id IN
    <foreach collection="ids" item="id" open="(" separator="," close=")">
        #{id}
    </foreach>
</select>

<!-- 批量插入 -->
<insert id="batchInsert" parameterType="list">
    INSERT INTO user (name, age) VALUES
    <foreach collection="list" item="u" separator=",">
        (#{u.name}, #{u.age})
    </foreach>
</insert>
```

### 4.4 choose 分支

```xml
<select id="findByCondition" resultType="User">
    SELECT * FROM user
    <where>
        <choose>
            <when test="id != null">
                AND id = #{id}
            </when>
            <when test="name != null">
                AND name = #{name}
            </when>
            <otherwise>
                AND status = 1
            </otherwise>
        </choose>
    </where>
</select>
```

---

## 五、关联查询（一对多 / 多对一）

### 5.1 多对一（association）

一个用户属于一个部门：

```xml
<resultMap id="userWithDept" type="User">
    <id property="id" column="id"/>
    <result property="name" column="name"/>
    <!-- 关联单个对象 -->
    <association property="dept" javaType="Dept">
        <id property="id" column="dept_id"/>
        <result property="name" column="dept_name"/>
    </association>
</resultMap>

<select id="findUserWithDept" resultMap="userWithDept">
    SELECT u.*, d.id AS dept_id, d.name AS dept_name
    FROM user u LEFT JOIN dept d ON u.dept_id = d.id
</select>
```

### 5.2 一对多（collection）

一个部门有多个用户：

```xml
<resultMap id="deptWithUsers" type="Dept">
    <id property="id" column="id"/>
    <result property="name" column="name"/>
    <!-- 关联集合 -->
    <collection property="users" ofType="User">
        <id property="id" column="user_id"/>
        <result property="name" column="user_name"/>
    </collection>
</resultMap>
```

---

## 六、缓存机制

### 6.1 一级缓存（默认开启）

- **作用域**：SqlSession 级别
- 同一个 SqlSession 内，相同查询只查一次数据库
- 增删改操作会清空一级缓存
- Spring Boot 中每次请求一个 SqlSession，方法结束就关闭

### 6.2 二级缓存（需手动开启）

- **作用域**：Mapper 级别（跨 SqlSession 共享）
- 实体类必须实现 Serializable
- 配置：`<cache/>` 在 Mapper XML 中

```xml
<!-- UserMapper.xml -->
<mapper namespace="com.example.mapper.UserMapper">
    <cache/>
    <!-- 或带参数 -->
    <cache eviction="LRU" flushInterval="60000" size="512" readOnly="true"/>
</mapper>
```

| | 一级缓存 | 二级缓存 |
|---|---|---|
| 作用域 | SqlSession | Mapper（跨 Session） |
| 默认 | 开启 | 关闭 |
| 清空 | 增删改 | 增删改 |
| 分布式 | 不支持 | 不支持（需第三方） |

> 企业开发中通常用 Redis 替代 MyBatis 二级缓存。

---

## 七、注解开发

简单 CRUD 可以用注解替代 XML：

```java
public interface UserMapper {
    @Select("SELECT * FROM user WHERE id = #{id}")
    User findById(Long id);

    @Insert("INSERT INTO user(name, age) VALUES(#{name}, #{age})")
    @Options(useGeneratedKeys = true, keyProperty = "id")
    int insert(User user);

    @Update("UPDATE user SET name=#{name}, age=#{age} WHERE id=#{id}")
    int update(User user);

    @Delete("DELETE FROM user WHERE id = #{id}")
    int deleteById(Long id);
}
```

> 注解适合简单 SQL，动态 SQL 和复杂映射仍推荐 XML。

---

## 八、面试高频题

**Q1: #{} 和 ${} 的区别？**
> #{} 是预编译参数（PreparedStatement，防注入），${} 是字符串拼接（有注入风险）。#{} 用于传值，${} 用于传表名/列名。

**Q2: MyBatis 的一级缓存和二级缓存？**
> 一级缓存是 SqlSession 级别，默认开启，相同查询走缓存，增删改清空缓存。二级缓存是 Mapper 级别，需手动开启，跨 SqlSession 共享。企业中通常用 Redis 替代二级缓存。

**Q3: MyBatis 动态 SQL 有哪些标签？**
> if、where、choose、foreach、set、trim。where 自动加 WHERE 并去掉首部 AND/OR，foreach 用于 IN 查询和批量插入。

**Q4: MyBatis 和 JPA 的区别？**
> MyBatis 半自动 ORM，SQL 手写、灵活可控，适合复杂查询。JPA 全自动 ORM，自动生成 SQL，开发快但优化难。企业中 MyBatis 更主流。

**Q5: resultMap 和 resultType 的区别？**
> resultType 自动映射（列名=属性名时用），resultMap 手动映射（列名≠属性名或关联查询时用）。

**Q6: MyBatis 怎么实现分页？**
> 1. 物理分页：用 PageHelper 插件或手写 SQL LIMIT，查一次只返回一页数据。2. 逻辑分页：RowBounds，查出全部再截取（不推荐）。

---

## 🔗 相关笔记

- [[MyBatis-Plus实战|MyBatis-Plus 实战]] — 增强版 CRUD / 条件构造器 / 分页
- [[../03-Database/MySQL数据库|MySQL 数据库]] — 底层数据库
- [[../04-Spring生态/SpringBoot-基础与配置|Spring Boot 基础与配置]] — 整合 MyBatis
- [[../04-Spring生态/SpringBoot-数据访问|Spring Boot 数据访问]] — Spring Boot 整合 MyBatis 配置
- [[../07-面试与项目/Java学习情况分析与实习冲刺计划|学习计划]] — 第二层会用+懂流程