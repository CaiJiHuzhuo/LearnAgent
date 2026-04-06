# 01 - Spring IoC 与 AOP

> 本章深入讲解 Spring 框架两大核心：**IoC（控制反转）** 和 **AOP（面向切面编程）**，覆盖原理、源码与面试高频考点。

## 目录

1. [IoC 容器概述](#一ioc-容器概述)
2. [Bean 生命周期](#二bean-生命周期)
3. [依赖注入方式](#三依赖注入方式)
4. [Bean 作用域](#四bean-作用域)
5. [AOP 核心概念](#五aop-核心概念)
6. [AOP 实现原理](#六aop-实现原理)
7. [Spring 循环依赖与三级缓存](#七spring-循环依赖与三级缓存)
8. [大厂常见面试题](#八大厂常见面试题)

---

## 一、IoC 容器概述

### 1.1 什么是 IoC

**IoC（Inversion of Control，控制反转）**：将对象的创建和依赖关系的管理交给 Spring 容器，而非程序员手动 new 对象。

| 传统方式 | IoC 方式 |
|---------|---------|
| `UserService service = new UserServiceImpl()` | 容器自动创建并注入 |
| 调用方控制依赖的创建 | 容器控制依赖的创建（控制反转） |
| 强耦合 | 松耦合 |

### 1.2 IoC 容器类型

| 容器 | 说明 |
|------|------|
| **BeanFactory** | IoC 容器最基础接口，懒加载 Bean |
| **ApplicationContext** | BeanFactory 的子接口，功能更丰富（国际化、事件机制、AOP 集成等），启动时预加载单例 Bean |

常用 ApplicationContext 实现：

```
ApplicationContext
├── ClassPathXmlApplicationContext     # 类路径下 XML 配置
├── FileSystemXmlApplicationContext    # 文件系统 XML 配置
├── AnnotationConfigApplicationContext # 注解配置
└── GenericWebApplicationContext       # Web 环境
```

### 1.3 BeanDefinition

Spring 用 **BeanDefinition** 来描述一个 Bean 的元信息：

| 属性 | 说明 |
|------|------|
| beanClassName | Bean 的全限定类名 |
| scope | 作用域（singleton/prototype） |
| lazyInit | 是否懒加载 |
| dependsOn | 依赖的其他 Bean |
| initMethodName | 初始化方法 |
| destroyMethodName | 销毁方法 |
| constructorArgumentValues | 构造器参数 |
| propertyValues | 属性值 |

---

## 二、Bean 生命周期

### 2.1 完整生命周期（核心考点 ⭐⭐⭐）

```
1. 实例化（Instantiation）
   └── 通过构造器或工厂方法创建 Bean 实例

2. 属性填充（Populate Properties）
   └── 依赖注入（@Autowired、@Value 等）

3. Aware 接口回调
   ├── BeanNameAware#setBeanName()
   ├── BeanFactoryAware#setBeanFactory()
   └── ApplicationContextAware#setApplicationContext()

4. BeanPostProcessor#postProcessBeforeInitialization()
   └── 前置处理（如 @PostConstruct 在此执行）

5. 初始化
   ├── InitializingBean#afterPropertiesSet()
   └── 自定义 init-method

6. BeanPostProcessor#postProcessAfterInitialization()
   └── 后置处理（AOP 代理在此生成）

7. 使用 Bean

8. 销毁
   ├── DisposableBean#destroy()
   └── 自定义 destroy-method
```

### 2.2 核心扩展点

| 扩展点 | 作用 | 典型应用 |
|--------|------|---------|
| **BeanPostProcessor** | 在 Bean 初始化前后做增强 | AOP 代理、`@Autowired` 注入 |
| **BeanFactoryPostProcessor** | 在 Bean 实例化前修改 BeanDefinition | `PropertyPlaceholderConfigurer` |
| **InstantiationAwareBeanPostProcessor** | 控制 Bean 实例化过程 | AOP 提前代理 |

### 2.3 代码示例

```java
@Component
public class MyBean implements InitializingBean, DisposableBean,
        BeanNameAware, ApplicationContextAware {

    @Value("${app.name:demo}")
    private String appName;

    @Override
    public void setBeanName(String name) {
        System.out.println("1. BeanNameAware: " + name);
    }

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        System.out.println("2. ApplicationContextAware");
    }

    @PostConstruct
    public void postConstruct() {
        System.out.println("3. @PostConstruct");
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("4. InitializingBean#afterPropertiesSet");
    }

    @Override
    public void destroy() {
        System.out.println("5. DisposableBean#destroy");
    }

    @PreDestroy
    public void preDestroy() {
        System.out.println("6. @PreDestroy");
    }
}
```

---

## 三、依赖注入方式

### 3.1 三种注入方式对比

| 方式 | 注解/配置 | 推荐度 | 说明 |
|------|----------|--------|------|
| **构造器注入** | `@Autowired` 在构造器上 | ⭐⭐⭐ | Spring 官方推荐，保证不可变性和完整初始化 |
| **Setter 注入** | `@Autowired` 在 setter 上 | ⭐⭐ | 可选依赖场景 |
| **字段注入** | `@Autowired` 在字段上 | ⭐ | 简洁但不利于测试 |

### 3.2 @Autowired vs @Resource

| 特性 | @Autowired | @Resource |
|------|-----------|-----------|
| 来源 | Spring 框架 | JSR-250（JDK 标准） |
| 注入策略 | 默认 **byType**，可配合 `@Qualifier` | 默认 **byName**，找不到再 byType |
| required 属性 | `@Autowired(required = false)` | 无 |
| 推荐场景 | Spring 项目 | 需要跨框架兼容 |

### 3.3 构造器注入示例（推荐写法）

```java
@Service
public class OrderService {

    private final UserService userService;
    private final ProductService productService;

    // Spring 4.3+ 单构造器可省略 @Autowired
    public OrderService(UserService userService, ProductService productService) {
        this.userService = userService;
        this.productService = productService;
    }
}
```

---

## 四、Bean 作用域

| 作用域 | 说明 | 适用 |
|--------|------|------|
| **singleton** | 默认，容器中只有一个实例 | 无状态 Service/DAO |
| **prototype** | 每次 getBean 创建新实例 | 有状态 Bean |
| **request** | 每个 HTTP 请求一个实例 | Web 应用 |
| **session** | 每个 HTTP Session 一个实例 | Web 应用 |
| **application** | ServletContext 生命周期 | Web 应用 |

### Singleton 中注入 Prototype 问题

Singleton Bean 中注入的 Prototype Bean 始终是同一个实例。

**解决方案：**

```java
// 方案一：@Lookup 方法注入
@Component
public abstract class SingletonBean {
    @Lookup
    public abstract PrototypeBean getPrototypeBean();
}

// 方案二：ObjectFactory / ObjectProvider
@Component
public class SingletonBean {
    @Autowired
    private ObjectProvider<PrototypeBean> prototypeBeanProvider;

    public void doSomething() {
        PrototypeBean bean = prototypeBeanProvider.getObject(); // 每次获取新实例
    }
}
```

---

## 五、AOP 核心概念

### 5.1 什么是 AOP

**AOP（Aspect-Oriented Programming，面向切面编程）**：将横切关注点（日志、事务、权限等）从业务代码中分离出来，实现模块化。

### 5.2 核心术语

| 术语 | 英文 | 说明 |
|------|------|------|
| **切面** | Aspect | 横切关注点的模块化（如日志切面、事务切面） |
| **连接点** | Join Point | 程序执行中的某个点（方法调用、异常抛出等） |
| **切入点** | Pointcut | 匹配连接点的表达式，决定在哪些方法上织入 |
| **通知** | Advice | 在切入点上执行的具体操作 |
| **织入** | Weaving | 将切面应用到目标对象的过程 |
| **目标对象** | Target | 被代理的原始对象 |
| **代理** | Proxy | 织入切面后生成的代理对象 |

### 5.3 五种通知类型

| 通知类型 | 注解 | 执行时机 |
|---------|------|---------|
| 前置通知 | `@Before` | 目标方法执行前 |
| 后置通知 | `@AfterReturning` | 目标方法正常返回后 |
| 异常通知 | `@AfterThrowing` | 目标方法抛出异常后 |
| 最终通知 | `@After` | 目标方法执行后（无论是否异常） |
| 环绕通知 | `@Around` | 包裹目标方法，可控制是否执行 |

**执行顺序：**

```
@Around (前半部分)
  └── @Before
        └── 目标方法
              ├── 正常返回 → @AfterReturning → @After → @Around (后半部分)
              └── 异常抛出 → @AfterThrowing → @After
```

### 5.4 切入点表达式

```java
// 匹配 com.example.service 包下所有类的所有方法
@Pointcut("execution(* com.example.service.*.*(..))")
public void serviceLayer() {}

// 匹配带有 @Transactional 注解的方法
@Pointcut("@annotation(org.springframework.transaction.annotation.Transactional)")
public void transactionalMethod() {}

// 组合表达式
@Pointcut("serviceLayer() && !transactionalMethod()")
public void nonTransactionalService() {}
```

---

## 六、AOP 实现原理

### 6.1 两种代理方式（核心考点 ⭐⭐⭐）

| 代理方式 | JDK 动态代理 | CGLIB 代理 |
|---------|------------|-----------|
| 原理 | 基于接口，`java.lang.reflect.Proxy` | 基于继承，生成目标类的子类 |
| 要求 | 目标类必须实现接口 | 目标类不能是 final |
| 性能 | JDK 8+ 性能接近 CGLIB | 生成字节码，调用快 |
| Spring 默认 | 有接口用 JDK 代理 | 无接口用 CGLIB |
| Spring Boot 默认 | - | **默认使用 CGLIB**（`proxyTargetClass=true`） |

### 6.2 JDK 动态代理原理

```java
// JDK 动态代理核心：InvocationHandler
public class JdkProxyDemo implements InvocationHandler {
    private Object target;

    public JdkProxyDemo(Object target) {
        this.target = target;
    }

    public Object getProxy() {
        return Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            target.getClass().getInterfaces(),
            this
        );
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("Before: " + method.getName());
        Object result = method.invoke(target, args);
        System.out.println("After: " + method.getName());
        return result;
    }
}
```

### 6.3 CGLIB 代理原理

```java
// CGLIB 核心：MethodInterceptor
public class CglibProxyDemo implements MethodInterceptor {

    public Object getProxy(Class<?> clazz) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(clazz);
        enhancer.setCallback(this);
        return enhancer.create();
    }

    @Override
    public Object intercept(Object obj, Method method, Object[] args,
                           MethodProxy proxy) throws Throwable {
        System.out.println("Before: " + method.getName());
        Object result = proxy.invokeSuper(obj, args);
        System.out.println("After: " + method.getName());
        return result;
    }
}
```

### 6.4 AOP 代理创建流程

```
1. Spring 容器启动，注册 AnnotationAwareAspectJAutoProxyCreator（BeanPostProcessor）
2. Bean 实例化后，postProcessAfterInitialization() 检查是否需要代理
3. 收集所有 Advisor（切面+切入点），匹配当前 Bean
4. 有匹配的 Advisor → 创建代理对象
   ├── 有接口 → JDK 动态代理（默认）
   └── 无接口 → CGLIB 代理
5. 返回代理对象放入容器
```

---

## 七、Spring 循环依赖与三级缓存

### 7.1 什么是循环依赖

```
A → 依赖 B → 依赖 A（循环）
```

### 7.2 三级缓存（核心考点 ⭐⭐⭐）

| 缓存 | 名称 | 存储内容 |
|------|------|---------|
| **一级缓存** | singletonObjects | 完全初始化好的 Bean（成品） |
| **二级缓存** | earlySingletonObjects | 提前暴露的 Bean（半成品，可能已代理） |
| **三级缓存** | singletonFactories | Bean 的 ObjectFactory（lambda 工厂） |

### 7.3 解决流程

```
1. 创建 A → 实例化 A → 将 A 的 ObjectFactory 放入三级缓存
2. A 属性填充 → 发现依赖 B → 去创建 B
3. 创建 B → 实例化 B → 将 B 的 ObjectFactory 放入三级缓存
4. B 属性填充 → 发现依赖 A → 从三级缓存获取 A 的 ObjectFactory
   → 调用 getObject() 获得 A（可能是代理对象）→ 放入二级缓存
5. B 初始化完成 → 放入一级缓存
6. A 属性填充 B 完成 → A 初始化完成 → 放入一级缓存
```

### 7.4 为什么需要三级缓存？

**二级缓存不够吗？**

如果没有 AOP，两级缓存就够了。但有 AOP 时：
- 三级缓存的 ObjectFactory 可以在需要时才生成代理对象
- 保证了代理对象的创建时机在 `postProcessAfterInitialization` 之后
- 如果提前暴露的是原始对象，最终放入容器的却是代理对象，就会导致不一致

### 7.5 哪些循环依赖无法解决？

| 场景 | 能否解决 | 原因 |
|------|---------|------|
| Singleton + Setter 注入 | ✅ 可以 | 三级缓存解决 |
| Singleton + 构造器注入 | ❌ 不行 | 实例化阶段就需要依赖，无法提前暴露 |
| Prototype 循环依赖 | ❌ 不行 | Prototype 不使用缓存 |

**构造器循环依赖解决方案：** 使用 `@Lazy` 注解延迟加载。

```java
@Service
public class A {
    public A(@Lazy B b) { // 注入代理对象，使用时才真正初始化
        this.b = b;
    }
}
```

---

## 八、大厂常见面试题

### Q1：Spring IoC 的理解？BeanFactory 和 ApplicationContext 的区别？

**答：**
- IoC 是控制反转，将对象创建和依赖管理交给 Spring 容器
- BeanFactory 是最基础的 IoC 容器接口，懒加载
- ApplicationContext 继承 BeanFactory，增加了 AOP 集成、事件机制、国际化、资源加载等功能，启动时预加载单例 Bean

### Q2：Bean 的生命周期？（说出核心步骤）

**答：** 实例化 → 属性填充 → Aware 回调 → BeanPostProcessor 前置处理 → 初始化（@PostConstruct → InitializingBean → init-method）→ BeanPostProcessor 后置处理（AOP 代理在此生成）→ 使用 → 销毁（@PreDestroy → DisposableBean → destroy-method）

### Q3：Spring AOP 的实现原理？JDK 代理和 CGLIB 的区别？

**答：**
- Spring AOP 基于动态代理实现
- **JDK 动态代理**：基于接口，使用 `Proxy + InvocationHandler`
- **CGLIB 代理**：基于继承，通过字节码技术生成子类
- Spring 默认有接口用 JDK 代理，无接口用 CGLIB；Spring Boot 默认全部用 CGLIB

### Q4：Spring 如何解决循环依赖？为什么需要三级缓存？

**答：**
- 通过三级缓存解决 Singleton + Setter 注入的循环依赖
- 一级缓存放成品 Bean，二级缓存放半成品，三级缓存放 ObjectFactory
- 三级缓存的作用是延迟创建 AOP 代理对象，保证代理对象创建时机正确
- 构造器注入和 Prototype 的循环依赖无法解决

### Q5：@Autowired 的实现原理？

**答：**
- 由 `AutowiredAnnotationBeanPostProcessor` 处理
- 在属性填充阶段，通过 `postProcessProperties()` 扫描字段/方法上的 `@Autowired`
- 默认 byType 查找，找到多个用 `@Qualifier` 或字段名匹配
- `required=true`（默认）时找不到会抛异常

### Q6：@Configuration 和 @Component 的区别？

**答：**
- `@Configuration` 类会被 CGLIB 增强，保证 `@Bean` 方法之间的调用返回同一个单例
- `@Component` 类不会增强，`@Bean` 方法互相调用会创建新对象
- `@Configuration` 的 Full 模式 vs `@Component` 的 Lite 模式

---

*下一篇：[02-Spring MVC 与 Web 开发](./02-SpringMVC与Web开发.md)*
