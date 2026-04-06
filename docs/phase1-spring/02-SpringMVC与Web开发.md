# 02 - Spring MVC 与 Web 开发

> 本章深入讲解 Spring MVC 的核心流程、组件架构、常用注解与最佳实践。

## 目录

1. [Spring MVC 概述](#一spring-mvc-概述)
2. [核心流程（DispatcherServlet）](#二核心流程dispatcherservlet)
3. [常用注解](#三常用注解)
4. [参数绑定与数据校验](#四参数绑定与数据校验)
5. [异常处理](#五异常处理)
6. [拦截器](#六拦截器)
7. [RESTful API 设计](#七restful-api-设计)
8. [大厂常见面试题](#八大厂常见面试题)

---

## 一、Spring MVC 概述

### 1.1 MVC 架构

| 组件 | 职责 |
|------|------|
| **Model** | 数据模型，封装业务数据 |
| **View** | 视图层，负责数据展示 |
| **Controller** | 控制器，接收请求、调用 Service、返回视图/数据 |

### 1.2 Spring MVC 核心组件

| 组件 | 说明 |
|------|------|
| **DispatcherServlet** | 前端控制器，请求入口 |
| **HandlerMapping** | 请求映射器，根据 URL 找到 Handler |
| **HandlerAdapter** | 处理器适配器，执行 Handler |
| **ViewResolver** | 视图解析器，解析视图名称为具体视图 |
| **HandlerInterceptor** | 拦截器，请求前后处理 |
| **HandlerExceptionResolver** | 异常解析器 |

---

## 二、核心流程（DispatcherServlet）

### 2.1 请求处理流程（核心考点 ⭐⭐⭐）

```
1. 客户端发送 HTTP 请求
2. DispatcherServlet 接收请求
3. HandlerMapping 根据 URL 查找 Handler（Controller 方法）
   └── 返回 HandlerExecutionChain（Handler + 拦截器列表）
4. HandlerAdapter 执行 Handler
   ├── 参数解析（HandlerMethodArgumentResolver）
   ├── 调用 Controller 方法
   └── 返回值处理（HandlerMethodReturnValueHandler）
5. 如果返回 ModelAndView：
   ├── ViewResolver 解析视图
   └── View 渲染
6. 如果使用 @ResponseBody：
   └── HttpMessageConverter 序列化（如 Jackson → JSON）
7. 返回响应给客户端
```

### 2.2 DispatcherServlet 初始化

```
DispatcherServlet#onRefresh()
├── initMultipartResolver()        # 文件上传
├── initLocaleResolver()           # 国际化
├── initThemeResolver()            # 主题
├── initHandlerMappings()          # 请求映射
├── initHandlerAdapters()          # 处理器适配
├── initHandlerExceptionResolvers()# 异常处理
├── initRequestToViewNameTranslator()
├── initViewResolvers()            # 视图解析
└── initFlashMapManager()          # 重定向传参
```

---

## 三、常用注解

### 3.1 控制器注解

| 注解 | 说明 |
|------|------|
| `@Controller` | 标记控制器类，配合视图使用 |
| `@RestController` | = `@Controller` + `@ResponseBody`，RESTful 接口 |
| `@RequestMapping` | 类/方法级别的请求映射 |
| `@GetMapping` | GET 请求映射（Spring 4.3+） |
| `@PostMapping` | POST 请求映射 |
| `@PutMapping` | PUT 请求映射 |
| `@DeleteMapping` | DELETE 请求映射 |

### 3.2 参数注解

| 注解 | 说明 | 示例 |
|------|------|------|
| `@RequestParam` | 绑定查询参数 | `?name=Tom` |
| `@PathVariable` | 绑定路径变量 | `/users/{id}` |
| `@RequestBody` | 绑定请求体（JSON） | POST body |
| `@RequestHeader` | 绑定请求头 | `Authorization` |
| `@CookieValue` | 绑定 Cookie | `sessionId` |
| `@ModelAttribute` | 绑定表单数据到对象 | form 表单 |

### 3.3 示例

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping("/{id}")
    public Result<UserVO> getUser(@PathVariable Long id) {
        return Result.success(userService.getById(id));
    }

    @PostMapping
    public Result<Void> createUser(@RequestBody @Valid UserCreateDTO dto) {
        userService.create(dto);
        return Result.success();
    }

    @GetMapping
    public Result<PageResult<UserVO>> listUsers(
            @RequestParam(defaultValue = "1") Integer page,
            @RequestParam(defaultValue = "10") Integer size) {
        return Result.success(userService.list(page, size));
    }
}
```

---

## 四、参数绑定与数据校验

### 4.1 参数解析器

Spring MVC 通过 `HandlerMethodArgumentResolver` 解析方法参数：

| 解析器 | 处理注解 |
|--------|---------|
| RequestParamMethodArgumentResolver | `@RequestParam` |
| PathVariableMethodArgumentResolver | `@PathVariable` |
| RequestResponseBodyMethodProcessor | `@RequestBody` |
| ServletModelAttributeMethodProcessor | `@ModelAttribute` |

### 4.2 数据校验（JSR-303）

```java
public class UserCreateDTO {
    @NotBlank(message = "用户名不能为空")
    @Size(min = 2, max = 20, message = "用户名长度 2-20")
    private String username;

    @NotBlank(message = "密码不能为空")
    @Pattern(regexp = "^(?=.*[A-Za-z])(?=.*\\d)[A-Za-z\\d]{8,}$",
             message = "密码至少8位，包含字母和数字")
    private String password;

    @Email(message = "邮箱格式不正确")
    private String email;

    @Min(value = 0, message = "年龄不能小于0")
    @Max(value = 150, message = "年龄不能大于150")
    private Integer age;
}
```

### 4.3 分组校验

```java
// 定义分组
public interface Create {}
public interface Update {}

public class UserDTO {
    @Null(groups = Create.class)
    @NotNull(groups = Update.class)
    private Long id;

    @NotBlank(groups = {Create.class, Update.class})
    private String username;
}

// 使用分组
@PostMapping
public Result<Void> create(@RequestBody @Validated(Create.class) UserDTO dto) { ... }
```

---

## 五、异常处理

### 5.1 全局异常处理（推荐）

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<Void> handleValidation(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getFieldErrors().stream()
                .map(FieldError::getDefaultMessage)
                .collect(Collectors.joining("; "));
        return Result.fail(400, message);
    }

    @ExceptionHandler(BusinessException.class)
    public Result<Void> handleBusiness(BusinessException e) {
        return Result.fail(e.getCode(), e.getMessage());
    }

    @ExceptionHandler(Exception.class)
    public Result<Void> handleException(Exception e) {
        log.error("系统异常", e);
        return Result.fail(500, "系统内部错误");
    }
}
```

### 5.2 统一响应格式

```java
@Data
@AllArgsConstructor
public class Result<T> {
    private int code;
    private String message;
    private T data;

    public static <T> Result<T> success(T data) {
        return new Result<>(200, "success", data);
    }

    public static <T> Result<T> fail(int code, String message) {
        return new Result<>(code, message, null);
    }
}
```

---

## 六、拦截器

### 6.1 HandlerInterceptor 接口

```java
public class AuthInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                            HttpServletResponse response, Object handler) {
        // 请求处理前：权限校验
        String token = request.getHeader("Authorization");
        if (token == null || !tokenService.verify(token)) {
            response.setStatus(401);
            return false; // 中断请求
        }
        return true; // 继续执行
    }

    @Override
    public void postHandle(HttpServletRequest request,
                          HttpServletResponse response, Object handler,
                          ModelAndView modelAndView) {
        // 请求处理后，视图渲染前
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
                               HttpServletResponse response, Object handler,
                               Exception ex) {
        // 请求完成后（包括异常），清理资源
    }
}
```

### 6.2 拦截器 vs 过滤器

| 维度 | Filter（过滤器） | Interceptor（拦截器） |
|------|-----------------|---------------------|
| 规范 | Servlet 规范 | Spring MVC 规范 |
| 作用范围 | 所有请求（包括静态资源） | 仅 DispatcherServlet 处理的请求 |
| 能否获取 Handler | ❌ | ✅ 可以获取 Controller 方法 |
| IOC 容器 | 不能直接注入 Bean | 可以注入 Spring Bean |
| 执行顺序 | Filter → Interceptor → Controller |

---

## 七、RESTful API 设计

### 7.1 设计规范

| 操作 | HTTP 方法 | URI 示例 | 说明 |
|------|----------|---------|------|
| 查询列表 | GET | `/api/users` | 获取用户列表 |
| 查询详情 | GET | `/api/users/{id}` | 获取单个用户 |
| 创建 | POST | `/api/users` | 创建用户 |
| 全量更新 | PUT | `/api/users/{id}` | 更新全部字段 |
| 部分更新 | PATCH | `/api/users/{id}` | 更新部分字段 |
| 删除 | DELETE | `/api/users/{id}` | 删除用户 |

### 7.2 状态码规范

| 状态码 | 含义 | 场景 |
|--------|------|------|
| 200 | OK | 请求成功 |
| 201 | Created | 资源创建成功 |
| 204 | No Content | 删除成功 |
| 400 | Bad Request | 参数校验失败 |
| 401 | Unauthorized | 未认证 |
| 403 | Forbidden | 无权限 |
| 404 | Not Found | 资源不存在 |
| 500 | Internal Server Error | 服务器内部错误 |

---

## 八、大厂常见面试题

### Q1：Spring MVC 的请求处理流程？

**答：** 请求到达 DispatcherServlet → HandlerMapping 查找 Handler → HandlerAdapter 执行 Handler → 参数解析 → 调用 Controller 方法 → 返回值处理 → ViewResolver 解析视图/HttpMessageConverter 序列化 JSON → 返回响应。

### Q2：@Controller 和 @RestController 的区别？

**答：**
- `@RestController` = `@Controller` + `@ResponseBody`
- `@Controller` 返回视图名，由 ViewResolver 解析
- `@RestController` 直接返回数据，经 HttpMessageConverter 转换为 JSON

### Q3：拦截器和过滤器的区别？

**答：**
- Filter 是 Servlet 规范，作用于所有请求；Interceptor 是 Spring MVC 的，仅作用于 Controller 请求
- Interceptor 可以获取 Handler 信息，可以注入 Spring Bean
- 执行顺序：Filter → Interceptor → Controller

### Q4：Spring MVC 如何处理 JSON？

**答：**
- 使用 `@RequestBody` 接收 JSON，`@ResponseBody` 返回 JSON
- 底层通过 `HttpMessageConverter` 实现，默认使用 `MappingJackson2HttpMessageConverter`
- Spring Boot 自动配置了 Jackson，无需额外配置

### Q5：如何实现全局异常处理？

**答：** 使用 `@RestControllerAdvice` + `@ExceptionHandler`，可以统一处理各类异常，返回统一格式的错误响应。

---

*下一篇：[03-Spring Boot 核心](./03-SpringBoot核心.md)*
