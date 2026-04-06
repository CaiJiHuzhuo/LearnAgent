# 03 - ORM 常见面试题汇总

> 本章汇总 MyBatis / MyBatis-Plus 方向大厂高频面试题。

## 目录

1. [MyBatis 核心原理](#一mybatis-核心原理)
2. [MyBatis 映射与 SQL](#二mybatis-映射与-sql)
3. [MyBatis 缓存](#三mybatis-缓存)
4. [MyBatis-Plus](#四mybatis-plus)
5. [综合与对比](#五综合与对比)

---

## 一、MyBatis 核心原理

### Q1：MyBatis 的执行流程？

**答：**
1. 加载配置（mybatis-config.xml / application.yml），构建 `SqlSessionFactory`
2. 获取 `SqlSession`
3. 调用 Mapper 接口方法 → `MapperProxy`（JDK 动态代理）拦截
4. 根据 接口全限定名 + 方法名 定位 `MappedStatement`
5. `Executor` 执行：查缓存 → 创建 Statement → 设置参数 → 执行 SQL → 处理结果集
6. 返回结果

### Q2：Mapper 接口没有实现类，为什么能工作？

**答：**
- MyBatis 使用 **JDK 动态代理** 为 Mapper 接口创建代理对象（`MapperProxy`）
- `MapperProxy` 实现 `InvocationHandler`，在 `invoke()` 方法中：
  - 根据接口名 + 方法名定位到 XML 中的 SQL（`MappedStatement`）
  - 通过 `SqlSession` 执行 SQL 并返回结果
- Spring 集成时，`MapperFactoryBean` 负责将代理对象注册到容器

### Q3：MyBatis 的插件（拦截器）原理？

**答：**
- 基于 **JDK 动态代理 + 责任链模式**
- 可拦截四大对象：Executor、StatementHandler、ParameterHandler、ResultSetHandler
- 通过 `@Intercepts` + `@Signature` 注解指定拦截目标
- 典型应用：分页插件、SQL 打印、审计字段填充、数据权限过滤

---

## 二、MyBatis 映射与 SQL

### Q4：#{} 和 ${} 的区别？

**答：**

| 特性 | `#{}` | `${}` |
|------|-------|-------|
| 处理方式 | 预编译（PreparedStatement） | 字符串直接替换 |
| SQL 注入 | ✅ 安全（参数化） | ❌ 有注入风险 |
| 性能 | 可复用预编译 SQL | 每次重新编译 |
| 使用场景 | 参数值 | 动态表名/列名/ORDER BY |

**原则**：参数值必须用 `#{}`，只有表名/列名等无法参数化的地方才用 `${}`。

### Q5：MyBatis 如何处理一对一、一对多映射？

**答：**
- **一对一**：`<association>` 标签，支持嵌套结果映射和嵌套查询
- **一对多**：`<collection>` 标签
- 嵌套结果（一条 SQL JOIN）性能更好；嵌套查询（N+1）更灵活但有性能问题

```xml
<resultMap id="orderMap" type="Order">
    <association property="user" javaType="User" .../>
    <collection property="items" ofType="OrderItem" .../>
</resultMap>
```

### Q6：MyBatis 动态 SQL 有哪些标签？

**答：**
- `<if>`：条件判断
- `<choose>/<when>/<otherwise>`：多条件分支
- `<where>`：自动处理 WHERE 和多余的 AND/OR
- `<set>`：自动处理 SET 和多余的逗号
- `<foreach>`：遍历集合（IN 查询、批量插入）
- `<trim>`：自定义前后缀处理
- `<sql>/<include>`：SQL 片段复用

### Q7：MyBatis 如何实现批量操作？

**答：**
1. **foreach 标签**：在 XML 中用 `<foreach>` 拼接 VALUES
2. **Batch 模式**：`SqlSession` 使用 `ExecutorType.BATCH`
3. **MyBatis-Plus**：`saveBatch()` 方法（底层分批执行）

```xml
<insert id="batchInsert">
    INSERT INTO t_user (username, email) VALUES
    <foreach collection="list" item="u" separator=",">
        (#{u.username}, #{u.email})
    </foreach>
</insert>
```

---

## 三、MyBatis 缓存

### Q8：MyBatis 的一级缓存和二级缓存？

**答：**

| 缓存 | 级别 | 范围 | 默认状态 |
|------|------|------|---------|
| 一级缓存 | SqlSession | 同一 SqlSession 内共享 | 默认开启 |
| 二级缓存 | Mapper（namespace） | 跨 SqlSession 共享 | 需手动开启 |

- **查询顺序**：二级缓存 → 一级缓存 → 数据库
- **一级缓存失效**：不同 SqlSession、执行增删改、手动 clearCache
- **二级缓存问题**：多表关联更新不及时、分布式环境不一致

### Q9：为什么不建议在生产环境使用 MyBatis 二级缓存？

**答：**
1. **多表关联**：A 表的 Mapper 缓存了关联 B 表的数据，B 表更新后 A 的缓存不会失效
2. **分布式环境**：各节点的缓存相互独立，数据不一致
3. **粒度粗**：以 namespace 为单位，任何增删改都会清空整个 namespace 缓存
4. **推荐**：使用 Redis 作为统一的分布式缓存层

---

## 四、MyBatis-Plus

### Q10：MyBatis-Plus 和 MyBatis 的关系？

**答：**
- MyBatis-Plus 是 MyBatis 的**增强工具**，只做增强不做改变
- 提供通用 CRUD（BaseMapper/IService）、条件构造器、分页插件、代码生成器等
- 减少手写 SQL 的重复工作，复杂 SQL 仍可使用原生 MyBatis XML

### Q11：MyBatis-Plus 的主键生成策略？

**答：**
| 策略 | 说明 |
|------|------|
| `AUTO` | 数据库自增 |
| `ASSIGN_ID` | 雪花算法（默认，生成 Long 型 ID） |
| `ASSIGN_UUID` | UUID |
| `INPUT` | 手动输入 |

推荐使用雪花算法（`ASSIGN_ID`），兼顾有序性和分布式唯一性。

### Q12：MyBatis-Plus 的乐观锁实现？

**答：**
- 实体类字段加 `@Version` 注解
- 配置 `OptimisticLockerInnerInterceptor` 插件
- 更新时 SQL 自动加 `WHERE version = oldVersion`，并 `SET version = oldVersion + 1`
- 如果 version 不匹配，更新影响行数为 0，需业务层处理冲突

### Q13：MyBatis-Plus 的逻辑删除如何实现？

**答：**
- 实体类字段加 `@TableLogic` 注解
- 配置全局逻辑删除值（`logic-delete-value` / `logic-not-delete-value`）
- 删除操作自动变为 `UPDATE SET deleted = 1`
- 查询操作自动附加 `WHERE deleted = 0`

---

## 五、综合与对比

### Q14：MyBatis 和 Hibernate 的区别？

**答：**

| 维度 | MyBatis | Hibernate |
|------|---------|-----------|
| SQL 控制 | 手写 SQL，完全控制 | 自动生成 SQL |
| 学习成本 | 低 | 高（HQL/Criteria） |
| 灵活性 | 高，复杂查询友好 | 低，复杂查询需原生 SQL |
| 缓存 | 一级/二级缓存 | 一级/二级/查询缓存 |
| 性能 | SQL 可精确优化 | 生成的 SQL 可能不最优 |
| 跨数据库 | 需要适配 | HQL 天然跨数据库 |
| 国内使用 | 互联网公司主流 | 传统企业/外企 |

### Q15：ORM 框架的 N+1 问题是什么？如何解决？

**答：**
- **N+1 问题**：查询 1 次获取主表数据，再 N 次查询关联表数据
- 例如：查 10 个订单，每个订单再查 1 次用户，共 11 次查询

**解决方案：**
1. **JOIN 查询**：一条 SQL 关联查询（嵌套结果映射）
2. **批量查询**：先查所有订单，再 IN 查询所有关联用户
3. **MyBatis-Plus**：使用 `selectBatchIds()` 批量查询

### Q16：如何防止 SQL 注入？

**答：**
1. 使用 `#{}` 预编译参数（最核心）
2. `${}` 场景做好参数校验（白名单校验表名/列名）
3. 使用 MyBatis-Plus 的条件构造器（自动参数化）
4. 最小化数据库账号权限

---

*本模块完结。推荐学习路径：MyBatis 核心 → MyBatis-Plus 实战 → 面试题汇总*
