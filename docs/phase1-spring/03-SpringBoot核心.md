# 03 - Spring Boot 核心

> 本章深入讲解 Spring Boot 的自动配置原理、Starter 机制、核心注解及实战最佳实践。

## 目录

1. [Spring Boot 概述](#一spring-boot-概述)
2. [自动配置原理](#二自动配置原理)
3. [Starter 机制](#三starter-机制)
4. [核心注解](#四核心注解)
5. [配置管理](#五配置管理)
6. [内嵌容器](#六内嵌容器)
7. [Actuator 监控](#七actuator-监控)
8. [大厂常见面试题](#八大厂常见面试题)

---

## 一、Spring Boot 概述

### 1.1 Spring Boot 解决了什么问题

| 传统 Spring 痛点 | Spring Boot 解决方案 |
|-----------------|-------------------|
| 大量 XML 配置 | 自动配置 + 注解驱动 |
| 依赖管理复杂 | Starter 依赖管理 |
| 部署需要外部容器 | 内嵌 Tomcat/Jetty |
| 环境配置繁琐 | 多环境 Profile 支持 |

### 1.2 核心特性

- **约定优于配置**：提供合理的默认值
- **自动配置**：根据依赖自动配置 Bean
- **Starter 依赖**：一站式依赖管理
- **内嵌容器**：无需部署 WAR 包
- **Actuator**：生产级监控端点

---

## 二、自动配置原理

### 2.1 @SpringBootApplication（核心考点 ⭐⭐⭐）

```java
@SpringBootApplication
// 等价于以下三个注解的组合：
@SpringBootConfiguration     // 标识为配置类（= @Configuration）
@EnableAutoConfiguration     // 开启自动配置（核心）
@ComponentScan               // 组件扫描（默认扫描主类所在包及子包）
```

### 2.2 自动配置加载流程

```
1. @EnableAutoConfiguration
   └── @Import(AutoConfigurationImportSelector.class)

2. AutoConfigurationImportSelector#selectImports()
   └── 读取 META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
       （Spring Boot 2.7+ 新文件，旧版为 META-INF/spring.factories）

3. 加载所有自动配置类（如 DataSourceAutoConfiguration）

4. 条件注解过滤（@ConditionalOnXxx）
   ├── 满足条件 → 注册 Bean
   └── 不满足 → 跳过

5. 用户自定义 Bean 优先（@ConditionalOnMissingBean）
```

### 2.3 核心条件注解

| 条件注解 | 说明 |
|---------|------|
| `@ConditionalOnClass` | classpath 下存在指定类时生效 |
| `@ConditionalOnMissingClass` | classpath 下不存在指定类时生效 |
| `@ConditionalOnBean` | 容器中存在指定 Bean 时生效 |
| `@ConditionalOnMissingBean` | 容器中不存在指定 Bean 时生效 |
| `@ConditionalOnProperty` | 配置属性满足条件时生效 |
| `@ConditionalOnWebApplication` | Web 环境下生效 |

### 2.4 自动配置示例分析

以 `DataSourceAutoConfiguration` 为例：

```java
@AutoConfiguration(before = SqlInitializationAutoConfiguration.class)
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {

    @Configuration
    @ConditionalOnMissingBean(DataSource.class)
    @Import({ DataSourceConfiguration.Hikari.class,
              DataSourceConfiguration.Tomcat.class })
    static class PooledDataSourceConfiguration { }
}
```

**解读：**
1. classpath 有 `DataSource` 类（引入了 JDBC 依赖）→ 生效
2. 没有用户自定义的 `DataSource` Bean → 自动创建连接池
3. 默认使用 HikariCP（Spring Boot 2.x 默认）

---

## 三、Starter 机制

### 3.1 什么是 Starter

Starter 是一组依赖的集合 + 自动配置，引入一个 Starter 即可获得完整功能。

### 3.2 常用 Starter

| Starter | 功能 |
|---------|------|
| `spring-boot-starter-web` | Web 开发（内嵌 Tomcat + Spring MVC） |
| `spring-boot-starter-data-jpa` | JPA 持久层 |
| `spring-boot-starter-data-redis` | Redis 客户端 |
| `spring-boot-starter-security` | Spring Security 安全框架 |
| `spring-boot-starter-actuator` | 监控端点 |
| `spring-boot-starter-test` | 测试依赖集合 |

### 3.3 自定义 Starter

```
my-starter/
├── my-spring-boot-starter/            # Starter 模块（只有 pom，聚合依赖）
│   └── pom.xml
└── my-spring-boot-autoconfigure/      # 自动配置模块
    ├── src/main/java/
    │   └── com/example/
    │       ├── MyAutoConfiguration.java
    │       ├── MyProperties.java
    │       └── MyService.java
    └── src/main/resources/
        └── META-INF/
            └── spring/
                └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

```java
// 属性配置
@ConfigurationProperties(prefix = "my.service")
public class MyProperties {
    private String name = "default";
    private boolean enabled = true;
    // getter/setter
}

// 自动配置类
@AutoConfiguration
@ConditionalOnClass(MyService.class)
@EnableConfigurationProperties(MyProperties.class)
public class MyAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public MyService myService(MyProperties properties) {
        return new MyService(properties.getName());
    }
}
```

---

## 四、核心注解

| 注解 | 作用 |
|------|------|
| `@SpringBootApplication` | 主启动类，组合了三个核心注解 |
| `@ConfigurationProperties` | 将配置文件属性绑定到 Bean |
| `@EnableConfigurationProperties` | 激活 @ConfigurationProperties |
| `@ConditionalOnXxx` | 条件化 Bean 注册 |
| `@AutoConfiguration` | 标记自动配置类（Spring Boot 2.7+） |
| `@Import` | 导入其他配置类 |
| `@Profile` | 指定环境生效 |

---

## 五、配置管理

### 5.1 配置文件优先级（由高到低）

```
1. 命令行参数 --server.port=8081
2. SPRING_APPLICATION_JSON 环境变量
3. ServletConfig/ServletContext 参数
4. JNDI 属性
5. Java 系统属性 -Dserver.port=8081
6. OS 环境变量
7. application-{profile}.yml
8. application.yml
9. @PropertySource 指定的外部配置
10. 默认属性 SpringApplication.setDefaultProperties()
```

### 5.2 多环境配置

```yaml
# application.yml
spring:
  profiles:
    active: dev

---
# application-dev.yml
server:
  port: 8080
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/dev_db

---
# application-prod.yml
server:
  port: 8080
spring:
  datasource:
    url: jdbc:mysql://prod-host:3306/prod_db
```

### 5.3 配置绑定

```java
@ConfigurationProperties(prefix = "app.mail")
@Validated
public class MailProperties {

    @NotBlank
    private String host;
    private int port = 25;
    private String username;
    private String password;
    private Map<String, String> properties = new HashMap<>();
    // getter/setter
}
```

---

## 六、内嵌容器

### 6.1 支持的容器

| 容器 | Starter | 特点 |
|------|---------|------|
| **Tomcat** | 默认 | 使用最广泛 |
| **Jetty** | `spring-boot-starter-jetty` | 轻量，适合长连接 |
| **Undertow** | `spring-boot-starter-undertow` | 高性能，非阻塞 |

### 6.2 启动流程

```
1. SpringApplication.run()
2. 创建 ApplicationContext（Web 环境创建 ServletWebServerApplicationContext）
3. refresh() 阶段调用 onRefresh()
4. 创建 WebServer（如 TomcatWebServer）
5. 启动内嵌容器
```

---

## 七、Actuator 监控

### 7.1 常用端点

| 端点 | 说明 |
|------|------|
| `/actuator/health` | 健康检查 |
| `/actuator/info` | 应用信息 |
| `/actuator/metrics` | 指标数据 |
| `/actuator/env` | 环境变量 |
| `/actuator/beans` | 所有 Bean 列表 |
| `/actuator/mappings` | 请求映射 |
| `/actuator/threaddump` | 线程 Dump |
| `/actuator/heapdump` | 堆 Dump |

### 7.2 自定义健康检查

```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {

    @Override
    public Health health() {
        try {
            // 检查数据库连接
            checkDatabase();
            return Health.up()
                    .withDetail("database", "Available")
                    .build();
        } catch (Exception e) {
            return Health.down()
                    .withDetail("error", e.getMessage())
                    .build();
        }
    }
}
```

---

## 八、大厂常见面试题

### Q1：Spring Boot 自动配置原理？

**答：** `@SpringBootApplication` 中的 `@EnableAutoConfiguration` 通过 `AutoConfigurationImportSelector` 读取 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件，加载所有自动配置类，再通过 `@ConditionalOnXxx` 条件注解过滤，最终满足条件的配置类生效，注册 Bean。用户自定义 Bean 优先于自动配置（`@ConditionalOnMissingBean`）。

### Q2：Spring Boot Starter 的作用和原理？

**答：** Starter 是一组依赖 + 自动配置的打包。引入 Starter 后，Spring Boot 自动扫描其自动配置类，根据条件注解决定是否生效。自定义 Starter 需要创建 `autoconfigure` 模块和 `starter` 模块，在 `AutoConfiguration.imports` 中注册配置类。

### Q3：Spring Boot 配置文件的加载优先级？

**答：** 命令行参数 > 系统属性 > 环境变量 > `application-{profile}.yml` > `application.yml`。同时 jar 包外的配置优先于 jar 包内。

### Q4：如何自定义一个 Starter？

**答：** 创建 `autoconfigure` 模块，编写自动配置类（`@AutoConfiguration` + 条件注解），定义属性类（`@ConfigurationProperties`），在 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 注册配置类。创建 `starter` 模块引入 `autoconfigure` 和必要依赖。

### Q5：Spring Boot 启动流程？

**答：**
1. `SpringApplication.run()` 创建 SpringApplication 实例
2. 从 `spring.factories` 加载 `ApplicationContextInitializer` 和 `ApplicationListener`
3. 创建 Environment，加载配置
4. 创建 ApplicationContext（Web 环境用 `ServletWebServerApplicationContext`）
5. 调用 `refresh()` → 执行 BeanFactory 后处理器 → 注册 Bean → 自动配置生效
6. 启动内嵌 Web 容器
7. 发布 `ApplicationStartedEvent` 和 `ApplicationReadyEvent`

---

*下一篇：[04-Spring Cloud 微服务](./04-SpringCloud微服务.md)*
