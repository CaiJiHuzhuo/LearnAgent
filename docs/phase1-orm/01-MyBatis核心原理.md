# 01 - MyBatis 核心原理

> 本章深入讲解 MyBatis 框架的核心架构、工作原理、映射机制、缓存体系等知识点。

## 目录

1. [MyBatis 概述](#一mybatis-概述)
2. [核心架构](#二核心架构)
3. [SQL 映射](#三sql-映射)
4. [动态 SQL](#四动态-sql)
5. [缓存机制](#五缓存机制)
6. [插件机制](#六插件机制)
7. [与 Spring 集成](#七与-spring-集成)
8. [大厂常见面试题](#八大厂常见面试题)

---

## 一、MyBatis 概述

### 1.1 MyBatis vs JPA/Hibernate

| 维度 | MyBatis | JPA/Hibernate |
|------|---------|--------------|
| 定位 | 半自动 ORM（SQL 手写） | 全自动 ORM（SQL 自动生成） |
| SQL 控制 | 完全控制 SQL | 框架生成 SQL |
| 学习曲线 | 低 | 高 |
| 复杂查询 | 擅长 | 需 HQL/原生 SQL |
| 缓存 | 一级/二级缓存 | 一级/二级缓存 + 查询缓存 |
| 国内使用 | 主流（互联网公司） | 较少（传统企业） |

### 1.2 核心优势

- **SQL 与代码分离**：XML 中编写 SQL，便于维护和优化
- **灵活的映射**：支持复杂结果集映射
- **动态 SQL**：强大的条件拼装能力
- **插件扩展**：拦截器机制，支持分页、审计等

---

## 二、核心架构

### 2.1 核心组件

```
MyBatis 架构
├── SqlSessionFactoryBuilder  # 构建 SqlSessionFactory
├── SqlSessionFactory          # 创建 SqlSession（单例）
├── SqlSession                 # 执行 SQL 的会话
├── Executor                   # SQL 执行器
│   ├── SimpleExecutor         # 简单执行器（默认）
│   ├── ReuseExecutor          # 重用 Statement
│   └── BatchExecutor          # 批量执行
├── MappedStatement            # 封装 SQL 语句信息
├── ParameterHandler           # 参数处理器
├── ResultSetHandler           # 结果集处理器
├── StatementHandler           # Statement 处理器
└── TypeHandler                # 类型转换器
```

### 2.2 SQL 执行流程（核心考点 ⭐⭐⭐）

```
1. 调用 Mapper 接口方法
2. MapperProxy 拦截（JDK 动态代理）
3. 根据 Mapper 接口 + 方法名定位 MappedStatement
4. Executor 执行
   ├── 查询一级缓存（SqlSession 级别）
   ├── 缓存未命中 → 查询数据库
   │   ├── StatementHandler 创建 Statement
   │   ├── ParameterHandler 设置参数
   │   ├── 执行 SQL
   │   └── ResultSetHandler 处理结果集
   └── 结果放入一级缓存
5. 返回结果
```

### 2.3 Mapper 接口的工作原理

```
Q: Mapper 接口没有实现类，为什么能调用？

A: MyBatis 使用 JDK 动态代理：
1. SqlSession.getMapper(UserMapper.class)
2. MapperProxyFactory 创建 MapperProxy（实现 InvocationHandler）
3. 调用 Mapper 方法时，MapperProxy#invoke() 拦截
4. 根据 接口全限定名 + 方法名 定位到 XML 中的 SQL
5. 执行 SQL 并返回结果
```

---

## 三、SQL 映射

### 3.1 结果映射

```xml
<!-- 简单映射 -->
<select id="selectById" resultType="com.example.entity.User">
    SELECT id, username, email, create_time AS createTime
    FROM t_user WHERE id = #{id}
</select>

<!-- 复杂映射（resultMap） -->
<resultMap id="userResultMap" type="User">
    <id property="id" column="id"/>
    <result property="username" column="username"/>
    <result property="createTime" column="create_time"/>
    <!-- 一对一 -->
    <association property="department" javaType="Department">
        <id property="id" column="dept_id"/>
        <result property="name" column="dept_name"/>
    </association>
    <!-- 一对多 -->
    <collection property="roles" ofType="Role">
        <id property="id" column="role_id"/>
        <result property="name" column="role_name"/>
    </collection>
</resultMap>
```

### 3.2 #{} 和 ${} 的区别（核心考点 ⭐⭐⭐）

| 特性 | `#{}` | `${}` |
|------|-------|-------|
| 处理方式 | 预编译（PreparedStatement） | 字符串替换 |
| SQL 注入 | ✅ 安全 | ❌ 有风险 |
| 性能 | 可复用预编译 SQL | 每次重新编译 |
| 使用场景 | 参数值 | 表名、列名、ORDER BY |

```xml
<!-- ✅ 安全：参数用 #{} -->
<select id="selectByName" resultType="User">
    SELECT * FROM t_user WHERE username = #{username}
</select>

<!-- ⚠️ 动态表名/列名用 ${} -->
<select id="selectAll" resultType="User">
    SELECT * FROM ${tableName} ORDER BY ${orderBy}
</select>
```

---

## 四、动态 SQL

### 4.1 常用标签

| 标签 | 说明 |
|------|------|
| `<if>` | 条件判断 |
| `<choose>/<when>/<otherwise>` | 多条件分支（类似 switch） |
| `<where>` | 自动处理 WHERE 和多余的 AND/OR |
| `<set>` | 自动处理 SET 和多余的逗号 |
| `<foreach>` | 遍历集合 |
| `<trim>` | 自定义前缀/后缀处理 |
| `<sql>/<include>` | SQL 片段复用 |

### 4.2 示例

```xml
<!-- 动态查询 -->
<select id="selectByCondition" resultType="User">
    SELECT * FROM t_user
    <where>
        <if test="username != null and username != ''">
            AND username LIKE CONCAT('%', #{username}, '%')
        </if>
        <if test="email != null">
            AND email = #{email}
        </if>
        <if test="status != null">
            AND status = #{status}
        </if>
    </where>
    ORDER BY create_time DESC
</select>

<!-- 批量插入 -->
<insert id="batchInsert" parameterType="list">
    INSERT INTO t_user (username, email, status)
    VALUES
    <foreach collection="list" item="user" separator=",">
        (#{user.username}, #{user.email}, #{user.status})
    </foreach>
</insert>

<!-- 批量查询 -->
<select id="selectByIds" resultType="User">
    SELECT * FROM t_user WHERE id IN
    <foreach collection="ids" item="id" open="(" separator="," close=")">
        #{id}
    </foreach>
</select>
```

---

## 五、缓存机制

### 5.1 一级缓存（SqlSession 级别）

| 特性 | 说明 |
|------|------|
| 范围 | SqlSession 级别（默认开启） |
| 生命周期 | 随 SqlSession 创建和关闭 |
| 失效条件 | 执行 update/insert/delete、手动 clearCache()、不同 SqlSession |
| Spring 集成 | 每次请求新建 SqlSession，一级缓存基本失效 |

### 5.2 二级缓存（Mapper 级别）

| 特性 | 说明 |
|------|------|
| 范围 | Mapper（namespace）级别 |
| 生命周期 | 随 SqlSessionFactory |
| 开启方式 | `<cache/>` 标签或 `@CacheNamespace` |
| 序列化 | 缓存对象需实现 Serializable |

```xml
<!-- 开启二级缓存 -->
<mapper namespace="com.example.mapper.UserMapper">
    <cache eviction="LRU" flushInterval="60000"
           size="1024" readOnly="true"/>
    ...
</mapper>
```

### 5.3 缓存查询顺序

```
1. 查二级缓存（Mapper 级别）
2. 查一级缓存（SqlSession 级别）
3. 查数据库
4. 结果放入一级缓存
5. SqlSession 关闭时，一级缓存数据写入二级缓存
```

### 5.4 生产建议

> ⚠️ **实际开发中很少使用 MyBatis 二级缓存**，原因：
> - 多表关联查询时缓存更新不及时
> - 分布式环境下各节点缓存不一致
> - 推荐使用 **Redis** 作为分布式缓存

---

## 六、插件机制

### 6.1 拦截器原理

MyBatis 允许在以下四个核心对象的方法执行时拦截：

| 对象 | 可拦截方法 |
|------|-----------|
| **Executor** | update、query、commit、rollback |
| **StatementHandler** | prepare、parameterize、batch、update、query |
| **ParameterHandler** | getParameterObject、setParameters |
| **ResultSetHandler** | handleResultSets、handleOutputParameters |

### 6.2 分页插件示例

```java
@Intercepts({
    @Signature(type = Executor.class, method = "query",
        args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class})
})
public class PageInterceptor implements Interceptor {

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        // 1. 获取原始 SQL
        // 2. 改写为 COUNT 查询，获取总数
        // 3. 改写为分页 SQL（LIMIT offset, size）
        // 4. 执行分页查询
        return invocation.proceed();
    }
}
```

---

## 七、与 Spring 集成

### 7.1 Spring Boot 集成配置

```yaml
mybatis:
  mapper-locations: classpath:mapper/**/*.xml
  type-aliases-package: com.example.entity
  configuration:
    map-underscore-to-camel-case: true   # 驼峰映射
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl  # SQL 日志
```

### 7.2 SqlSessionTemplate

Spring 集成使用 `SqlSessionTemplate` 代替原生 `SqlSession`：
- 线程安全（内部每次操作获取新的 SqlSession）
- 自动管理事务（委托给 Spring 事务管理器）
- 自动关闭 SqlSession

---

## 八、大厂常见面试题

### Q1：MyBatis 的执行流程？

**答：**
1. 加载配置文件，构建 `SqlSessionFactory`
2. 获取 `SqlSession`
3. 调用 Mapper 接口方法，`MapperProxy`（JDK 动态代理）拦截
4. 根据接口全限定名 + 方法名定位 `MappedStatement`
5. `Executor` 执行：查缓存 → 未命中查数据库 → `StatementHandler` 创建 Statement → `ParameterHandler` 设参数 → 执行 SQL → `ResultSetHandler` 处理结果

### Q2：#{} 和 ${} 的区别？

**答：**
- `#{}` 预编译参数，防 SQL 注入，安全
- `${}` 字符串替换，有注入风险，用于动态表名/列名
- 参数值必须用 `#{}`，表名/排序字段用 `${}`

### Q3：MyBatis 的缓存机制？

**答：**
- **一级缓存**：SqlSession 级别，默认开启，同一 SqlSession 内有效
- **二级缓存**：Mapper 级别，需手动开启，跨 SqlSession 有效
- 查询顺序：二级缓存 → 一级缓存 → 数据库
- 生产环境推荐用 Redis 替代二级缓存

### Q4：MyBatis 的插件（拦截器）原理？

**答：**
- 基于 JDK 动态代理，对 Executor、StatementHandler、ParameterHandler、ResultSetHandler 进行拦截
- 通过 `@Intercepts` + `@Signature` 指定拦截目标
- 典型应用：分页插件（PageHelper）、SQL 打印、审计字段自动填充

### Q5：Mapper 接口没有实现类，为什么能工作？

**答：**
MyBatis 使用 JDK 动态代理为 Mapper 接口创建代理对象（MapperProxy）。调用方法时，MapperProxy 的 invoke() 方法根据接口全限定名 + 方法名定位到 XML 中对应的 SQL，通过 SqlSession 执行并返回结果。

---

*下一篇：[02-MyBatis-Plus 实战](./02-MyBatis-Plus实战.md)*
