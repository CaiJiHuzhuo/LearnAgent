# 02 - MyBatis-Plus 实战

> 本章讲解 MyBatis-Plus 的核心功能、通用 CRUD、条件构造器、代码生成器及最佳实践。

## 目录

1. [MyBatis-Plus 概述](#一mybatis-plus-概述)
2. [通用 CRUD](#二通用-crud)
3. [条件构造器](#三条件构造器)
4. [分页查询](#四分页查询)
5. [代码生成器](#五代码生成器)
6. [常用功能](#六常用功能)
7. [最佳实践](#七最佳实践)
8. [大厂常见面试题](#八大厂常见面试题)

---

## 一、MyBatis-Plus 概述

### 1.1 什么是 MyBatis-Plus

MyBatis-Plus（MP）是 MyBatis 的增强工具，在 MyBatis 基础上只做增强不做改变：

| 特性 | 说明 |
|------|------|
| **通用 CRUD** | 内置通用 Mapper 和 Service，无需手写 SQL |
| **条件构造器** | 链式 API 构建查询条件 |
| **分页插件** | 物理分页，支持多种数据库 |
| **代码生成器** | 一键生成 Entity/Mapper/Service/Controller |
| **逻辑删除** | 自动处理删除标记 |
| **自动填充** | 创建时间、更新时间等自动填充 |
| **乐观锁** | 基于 version 字段的乐观锁 |

---

## 二、通用 CRUD

### 2.1 BaseMapper 核心方法

```java
public interface BaseMapper<T> extends Mapper<T> {
    // 插入
    int insert(T entity);

    // 删除
    int deleteById(Serializable id);
    int deleteBatchIds(Collection<?> idList);
    int delete(Wrapper<T> queryWrapper);

    // 更新
    int updateById(T entity);
    int update(T entity, Wrapper<T> updateWrapper);

    // 查询
    T selectById(Serializable id);
    List<T> selectBatchIds(Collection<?> idList);
    List<T> selectList(Wrapper<T> queryWrapper);
    Long selectCount(Wrapper<T> queryWrapper);
    IPage<T> selectPage(IPage<T> page, Wrapper<T> queryWrapper);
}
```

### 2.2 IService 核心方法

```java
public interface IService<T> {
    // 保存
    boolean save(T entity);
    boolean saveBatch(Collection<T> entityList);
    boolean saveOrUpdate(T entity);

    // 删除
    boolean removeById(Serializable id);
    boolean remove(Wrapper<T> queryWrapper);

    // 更新
    boolean updateById(T entity);
    boolean update(Wrapper<T> updateWrapper);

    // 查询
    T getById(Serializable id);
    List<T> list(Wrapper<T> queryWrapper);
    IPage<T> page(IPage<T> page, Wrapper<T> queryWrapper);
    long count(Wrapper<T> queryWrapper);
}
```

### 2.3 实体类注解

```java
@Data
@TableName("t_user")
public class User {

    @TableId(type = IdType.ASSIGN_ID)  // 雪花算法
    private Long id;

    @TableField("username")
    private String username;

    private String email;

    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;

    @TableLogic  // 逻辑删除
    private Integer deleted;

    @Version  // 乐观锁
    private Integer version;

    @TableField(exist = false)  // 非数据库字段
    private String roleNames;
}
```

---

## 三、条件构造器

### 3.1 QueryWrapper

```java
// 基本查询
QueryWrapper<User> wrapper = new QueryWrapper<>();
wrapper.eq("status", 1)
       .like("username", "张")
       .between("age", 18, 30)
       .orderByDesc("create_time");
List<User> users = userMapper.selectList(wrapper);

// 选择特定列
wrapper.select("id", "username", "email");
```

### 3.2 LambdaQueryWrapper（推荐）

```java
// Lambda 方式（避免硬编码字段名）
LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
wrapper.eq(User::getStatus, 1)
       .like(User::getUsername, "张")
       .between(User::getAge, 18, 30)
       .orderByDesc(User::getCreateTime);

// 条件构造（动态条件）
LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
wrapper.eq(StringUtils.isNotBlank(username), User::getUsername, username)
       .ge(minAge != null, User::getAge, minAge)
       .le(maxAge != null, User::getAge, maxAge);
```

### 3.3 UpdateWrapper

```java
// Lambda 更新
LambdaUpdateWrapper<User> wrapper = new LambdaUpdateWrapper<>();
wrapper.eq(User::getId, 1L)
       .set(User::getStatus, 0)
       .set(User::getUpdateTime, LocalDateTime.now());
userMapper.update(null, wrapper);
```

---

## 四、分页查询

### 4.1 配置分页插件

```java
@Configuration
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        // 分页插件
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        // 乐观锁插件
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());
        return interceptor;
    }
}
```

### 4.2 分页查询

```java
// 基本分页
Page<User> page = new Page<>(1, 10); // 第1页，每页10条
LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
wrapper.eq(User::getStatus, 1);

IPage<User> result = userMapper.selectPage(page, wrapper);
// result.getRecords()  - 数据列表
// result.getTotal()    - 总记录数
// result.getPages()    - 总页数
// result.getCurrent()  - 当前页
// result.getSize()     - 每页大小
```

### 4.3 自定义 SQL 分页

```java
// Mapper 接口
@Select("SELECT u.*, d.name AS deptName FROM t_user u LEFT JOIN t_dept d ON u.dept_id = d.id ${ew.customSqlSegment}")
IPage<UserVO> selectUserPage(IPage<UserVO> page, @Param(Constants.WRAPPER) Wrapper<UserVO> wrapper);
```

---

## 五、代码生成器

### 5.1 快速生成

```java
FastAutoGenerator.create("jdbc:mysql://localhost:3306/test", "root", "password")
    .globalConfig(builder -> {
        builder.author("developer")
               .outputDir(System.getProperty("user.dir") + "/src/main/java");
    })
    .packageConfig(builder -> {
        builder.parent("com.example")
               .moduleName("user")
               .entity("entity")
               .mapper("mapper")
               .service("service")
               .controller("controller");
    })
    .strategyConfig(builder -> {
        builder.addInclude("t_user", "t_role")
               .addTablePrefix("t_")
               .entityBuilder()
               .enableLombok()
               .enableTableFieldAnnotation()
               .logicDeleteColumnName("deleted")
               .versionColumnName("version");
    })
    .execute();
```

---

## 六、常用功能

### 6.1 自动填充

```java
@Component
public class MyMetaObjectHandler implements MetaObjectHandler {

    @Override
    public void insertFill(MetaObject metaObject) {
        this.strictInsertFill(metaObject, "createTime", LocalDateTime::now, LocalDateTime.class);
        this.strictInsertFill(metaObject, "updateTime", LocalDateTime::now, LocalDateTime.class);
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        this.strictUpdateFill(metaObject, "updateTime", LocalDateTime::now, LocalDateTime.class);
    }
}
```

### 6.2 逻辑删除

```yaml
# application.yml
mybatis-plus:
  global-config:
    db-config:
      logic-delete-field: deleted
      logic-delete-value: 1
      logic-not-delete-value: 0
```

配置后：
- `deleteById()` → `UPDATE t_user SET deleted = 1 WHERE id = ?`
- `selectList()` → `SELECT * FROM t_user WHERE deleted = 0`

### 6.3 乐观锁

```java
// 使用乐观锁：先查后改
User user = userMapper.selectById(1L);
user.setUsername("newName");
userMapper.updateById(user);
// 生成的 SQL：UPDATE t_user SET username='newName', version=2 WHERE id=1 AND version=1
```

### 6.4 枚举映射

```java
@Getter
@AllArgsConstructor
public enum StatusEnum implements IEnum<Integer> {
    DISABLED(0, "禁用"),
    ENABLED(1, "启用");

    @EnumValue
    private final Integer value;
    private final String desc;
}
```

---

## 七、最佳实践

### 7.1 项目规范

1. **实体类**：使用 `@TableName` 显式指定表名，`@TableId` 指定主键策略
2. **字段映射**：开启驼峰映射，减少 `@TableField` 使用
3. **条件构造器**：优先使用 `LambdaQueryWrapper`，避免硬编码字段名
4. **分页查询**：必须配置分页插件，避免全表扫描
5. **逻辑删除**：统一使用逻辑删除，配合唯一索引处理

### 7.2 性能优化

| 优化点 | 建议 |
|--------|------|
| 批量操作 | 使用 `saveBatch()` 而非循环 `save()` |
| 大数据查询 | 使用流式查询或分批查询 |
| N+1 问题 | 避免循环查询，使用 JOIN 或批量查询 |
| 选择字段 | 使用 `select()` 指定需要的字段 |

---

## 八、大厂常见面试题

### Q1：MyBatis-Plus 和 MyBatis 的关系？

**答：** MyBatis-Plus 是 MyBatis 的增强工具，只做增强不做改变。提供了通用 CRUD、条件构造器、分页插件、代码生成器等功能，减少了手写 SQL 的工作量。

### Q2：MyBatis-Plus 的主键生成策略有哪些？

**答：**
- `AUTO`：数据库自增
- `ASSIGN_ID`：雪花算法（默认，Long 类型）
- `ASSIGN_UUID`：UUID
- `INPUT`：手动输入
- `NONE`：无策略

### Q3：LambdaQueryWrapper 的优势？

**答：** 使用方法引用代替字符串字段名，编译期检查字段名正确性，避免硬编码导致的 bug，重构友好。

### Q4：MyBatis-Plus 的乐观锁实现原理？

**答：** 基于 `@Version` 注解和 `OptimisticLockerInnerInterceptor` 插件。更新时自动在 WHERE 条件中加上 `AND version = ?`，并将 version + 1。如果 version 不匹配则更新失败（返回影响行数 0）。

---

*下一篇：[03-ORM 常见面试题](./03-ORM常见面试题.md)*
