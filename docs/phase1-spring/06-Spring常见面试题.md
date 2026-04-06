# 06 - Spring 常见面试题汇总

> 本章汇总 Spring 方向大厂高频面试题，涵盖 IoC/AOP、Spring MVC、Spring Boot、Spring Cloud、事务管理等核心考点。

## 目录

1. [Spring Core（IoC/AOP）](#一spring-coreiocaop)
2. [Spring MVC](#二spring-mvc)
3. [Spring Boot](#三spring-boot)
4. [Spring Cloud](#四spring-cloud)
5. [Spring 事务](#五spring-事务)
6. [综合与进阶](#六综合与进阶)

---

## 一、Spring Core（IoC/AOP）

### Q1：什么是 Spring IoC？和 DI 的关系？

**答：**
- **IoC（控制反转）**：将对象的创建和依赖管理交给 Spring 容器，而非程序自身控制
- **DI（依赖注入）**：IoC 的具体实现方式，通过构造器、Setter 或字段注入依赖
- IoC 是设计思想，DI 是实现手段

### Q2：Spring Bean 的生命周期？

**答：**
1. **实例化**：通过构造器或工厂方法创建 Bean 实例
2. **属性填充**：依赖注入（@Autowired 等）
3. **Aware 接口回调**：BeanNameAware、ApplicationContextAware 等
4. **BeanPostProcessor 前置处理**：`postProcessBeforeInitialization`（@PostConstruct 在此执行）
5. **初始化**：`InitializingBean#afterPropertiesSet()` → 自定义 init-method
6. **BeanPostProcessor 后置处理**：`postProcessAfterInitialization`（AOP 代理在此生成）
7. **使用**
8. **销毁**：@PreDestroy → `DisposableBean#destroy()` → 自定义 destroy-method

### Q3：Spring 中 Bean 的作用域有哪些？

**答：**
- **singleton**（默认）：容器中只有一个实例
- **prototype**：每次获取创建新实例
- **request**：每个 HTTP 请求一个实例
- **session**：每个 HTTP Session 一个实例
- **application**：ServletContext 生命周期

### Q4：Spring AOP 的实现原理？

**答：**
- Spring AOP 基于**动态代理**实现
- 有接口：默认使用 **JDK 动态代理**（基于 `Proxy + InvocationHandler`）
- 无接口：使用 **CGLIB 代理**（基于字节码生成子类）
- Spring Boot 默认全部使用 CGLIB（`proxyTargetClass=true`）
- 在 `BeanPostProcessor#postProcessAfterInitialization` 阶段创建代理

### Q5：Spring 如何解决循环依赖？

**答：**
- 通过**三级缓存**解决 Singleton + Setter/字段注入的循环依赖
- 一级缓存 `singletonObjects`：完整 Bean
- 二级缓存 `earlySingletonObjects`：提前暴露的半成品 Bean
- 三级缓存 `singletonFactories`：Bean 的 ObjectFactory（延迟创建代理）
- **无法解决**：构造器注入循环依赖、Prototype 循环依赖

### Q6：@Autowired 和 @Resource 的区别？

**答：**
| 维度 | @Autowired | @Resource |
|------|-----------|-----------|
| 来源 | Spring | JSR-250（JDK） |
| 注入方式 | 默认 byType | 默认 byName |
| 多个候选 | 配合 @Qualifier | name 属性指定 |
| required | 支持 `required=false` | 不支持 |

---

## 二、Spring MVC

### Q7：Spring MVC 的请求处理流程？

**答：**
1. 请求到达 **DispatcherServlet**
2. **HandlerMapping** 根据 URL 查找 Handler（返回 HandlerExecutionChain）
3. **HandlerAdapter** 执行 Handler（参数解析 → 调用 Controller → 返回值处理）
4. 返回 ModelAndView 或通过 `@ResponseBody` + `HttpMessageConverter` 转 JSON
5. **ViewResolver** 解析视图并渲染

### Q8：@Controller 和 @RestController 的区别？

**答：**
- `@RestController` = `@Controller` + `@ResponseBody`
- `@Controller` 返回视图名，`@RestController` 直接返回数据（JSON）

### Q9：拦截器和过滤器的区别？

**答：**
- **Filter**：Servlet 规范，作用于所有请求，不能获取 Handler 信息
- **Interceptor**：Spring MVC 规范，仅作用于 Controller 请求，可获取 Handler，可注入 Bean
- 执行顺序：Filter → Interceptor → Controller

---

## 三、Spring Boot

### Q10：Spring Boot 自动配置原理？

**答：**
1. `@SpringBootApplication` 包含 `@EnableAutoConfiguration`
2. 通过 `AutoConfigurationImportSelector` 读取 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
3. 加载所有自动配置类
4. `@ConditionalOnXxx` 条件注解过滤，满足条件的生效
5. 用户自定义 Bean 优先（`@ConditionalOnMissingBean`）

### Q11：Spring Boot Starter 的原理？

**答：**
- Starter = 依赖集合 + 自动配置
- 引入 Starter 自动触发自动配置机制
- 自定义 Starter 需要 `autoconfigure` 模块（配置类 + 条件注解）和 `starter` 模块（聚合依赖）

### Q12：Spring Boot 的启动流程？

**答：**
1. `SpringApplication.run()` 创建 SpringApplication
2. 加载 `ApplicationContextInitializer` 和 `ApplicationListener`
3. 创建 Environment，加载配置文件
4. 创建 ApplicationContext
5. `refresh()` → BeanFactory 后处理 → Bean 注册 → 自动配置生效
6. 启动内嵌 Web 容器
7. 发布 `ApplicationReadyEvent`

---

## 四、Spring Cloud

### Q13：微服务的核心组件有哪些？

**答：**
| 组件 | 当前主流方案 | 作用 |
|------|------------|------|
| 注册中心 | Nacos | 服务注册与发现 |
| 远程调用 | OpenFeign | 声明式 HTTP 调用 |
| 负载均衡 | LoadBalancer | 客户端负载均衡 |
| 熔断限流 | Sentinel | 服务保护 |
| 网关 | Spring Cloud Gateway | 路由转发、统一鉴权 |
| 配置中心 | Nacos Config | 集中配置管理 |

### Q14：Nacos 和 Eureka 的区别？

**答：**
- Nacos 支持 CP/AP 可切换，Eureka 只有 AP
- Nacos 同时支持注册中心和配置中心
- Nacos 支持主动健康检查和服务变更实时推送
- Eureka 已停止维护，Nacos 是当前主流

### Q15：Sentinel 的限流原理？

**答：**
- 基于**滑动窗口**统计 QPS
- 通过**责任链模式**（ProcessorSlotChain）处理规则
- 核心 Slot：FlowSlot（流控）、DegradeSlot（降级）、SystemSlot（系统保护）
- 支持 QPS 限流、线程数限流、热点参数限流

---

## 五、Spring 事务

### Q16：Spring 事务的传播行为？

**答：** 7 种传播行为，最常用的三种：
- **REQUIRED**（默认）：有事务加入，无则新建
- **REQUIRES_NEW**：总是新建独立事务
- **NESTED**：嵌套事务（保存点），外部回滚内部也回滚

### Q17：@Transactional 事务失效的场景？

**答：**
1. 方法非 public（AOP 限制）
2. 自调用（绕过代理对象）
3. 异常被 catch 吞掉
4. 抛出 checked 异常未指定 `rollbackFor`
5. 非 Spring 管理的类
6. 数据库引擎不支持事务
7. 多线程中新线程不在同一事务中

### Q18：Spring 事务的实现原理？

**答：**
基于 AOP 实现。`TransactionInterceptor` 拦截 `@Transactional` 方法：
1. 获取 `PlatformTransactionManager`
2. 根据传播行为决定是否开启事务
3. 关闭自动提交（`setAutoCommit(false)`）
4. 执行目标方法
5. 正常 → `commit()`，异常 → `rollback()`

---

## 六、综合与进阶

### Q19：Spring、Spring MVC、Spring Boot 的关系？

**答：**
- **Spring**：核心框架，提供 IoC 和 AOP
- **Spring MVC**：Spring 的 Web 模块，实现 MVC 架构
- **Spring Boot**：简化 Spring 应用开发，提供自动配置、Starter、内嵌容器

Spring Boot > Spring MVC > Spring（包含关系）

### Q20：Spring 中用到了哪些设计模式？

**答：**
| 设计模式 | 应用场景 |
|---------|---------|
| **工厂模式** | BeanFactory 创建 Bean |
| **单例模式** | Bean 默认 singleton |
| **代理模式** | AOP 动态代理 |
| **模板方法** | JdbcTemplate、RestTemplate |
| **观察者模式** | ApplicationEvent 事件机制 |
| **适配器模式** | HandlerAdapter |
| **策略模式** | Resource 接口的不同实现 |
| **责任链模式** | Interceptor 链 |

### Q21：BeanFactory 和 FactoryBean 的区别？

**答：**
- **BeanFactory**：Spring IoC 容器的顶层接口，负责管理 Bean
- **FactoryBean**：一个特殊的 Bean，用于创建复杂对象，`getObject()` 返回实际 Bean
- 从容器获取 FactoryBean 本身需要加 `&` 前缀：`getBean("&myFactory")`

### Q22：@Component 和 @Bean 的区别？

**答：**
| 维度 | @Component | @Bean |
|------|-----------|-------|
| 作用位置 | 类上 | 方法上（配置类中） |
| 注册方式 | 组件扫描自动注册 | 手动声明 |
| 适用场景 | 自己编写的类 | 第三方库的类 |
| 自定义程度 | 低 | 高（可自定义创建逻辑） |

### Q23：如何解决 Bean 注入冲突（多个实现）？

**答：**
1. `@Qualifier("beanName")` 指定 Bean 名称
2. `@Primary` 标记优先注入的 Bean
3. 使用 `@Resource(name = "beanName")` 按名称注入
4. 注入 `List<Interface>` 或 `Map<String, Interface>` 获取所有实现

---

*本模块完结。推荐学习路径：IoC/AOP → Spring MVC → Spring Boot → Spring 事务 → Spring Cloud*
