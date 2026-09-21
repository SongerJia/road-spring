## Spring MVC
SpringMVC是什么->核心组件->DispatcherServlet流程->九大组件->拦截器->参数绑定与序列化->面试高频题
### SpringMVC是什么
Spring MVC：基于Servlet的web框架，核心是一个前端控制器DispatchServlet统一接收请求，分发给Controller处理。
Servlet是什么
Servlet是JavaEE规范里定义的服务端处理器，本质是一个接收Http请求，返回Http响应的Java接口。
```
public interface Servlet {
    void init(ServletConfig config);
    void service(ServletRequest req, ServletResponse res);  //最核心的方法，所有http请求，都会进入到这个方法
    void destroy();
}
//HTTP的专属版本：HTTPServlet
public abstract class HttpServlet extends GenericServlet {
    protected void doGet(HttpServletRequest req, HttpServletResponse resp);
    protected void doPost(HttpServletRequest req, HttpServletResponse resp);
    protected void doPut(...);
    protected void doDelete(...);
}
```
Servlet怎么被调用的
```
浏览器
  ↓ HTTP
Tomcat（Web 服务器）
  ↓
Connector（解析 HTTP）
  ↓
Container（Servlet 容器）
  ↓
DispatcherServlet（Spring MVC 的唯一 Servlet）
  ↓
Controller
```
Tomcat做了什么
```
Tomcat 本质是一个：实现了 Servlet 规范的容器
它负责：
- 监听端口（8080）
- 解析 HTTP 协议
- 创建 `HttpServletRequest / HttpServletResponse`
- 找到对应的 Servlet
- 调用 `service()`
```
Servlet的生命周期
```
1. 加载 Servlet 类
2. 实例化 Servlet（单例）
3. init()（一次）
4. service()（每次请求）
5. destroy()（关闭时）
```
DispatcherServlet流程
```
请求 → DispatcherServlet（前端控制器）
              ↓
       HandlerMapping（找 Controller）
              ↓
       HandlerAdapter（调 Controller）
              ↓
       Controller（业务逻辑）
              ↓
       ViewResolver（解析视图）
              ↓
       返回响应
```
### 核心组件
```
// DispatcherServlet 持有的九个关键组件：

// ① HandlerMapping：请求 → Controller 方法的映射
// ② HandlerAdapter：调用 Controller 方法（适配器）
// ③ HandlerExceptionResolver：异常处理
// ④ ViewResolver：视图解析（返回 JSON 时基本不用）
// ⑤ RequestToViewNameTranslator：请求 → 视图名
// ⑥ LocaleResolver：国际化
// ⑦ ThemeResolver：主题
// ⑧ MultipartResolver：文件上传
// ⑨ FlashMapManager：重定向参数
```
两大核心组件
```
// ① HandlerMapping（找方法）：
// 根据 URL 找到对应 Controller 方法（HandlerMethod）
// RequestMappingHandlerMapping：@RequestMapping 映射

// ② HandlerAdapter（调方法）：
// 调用 Controller 方法，参数绑定、返回值处理
// RequestMappingHandlerAdapter：调用 @RequestMapping 方法
```
### DispatcherServlet完整流程
```
一次请求的完整生命周期
① 客户端发送请求（GET /order/100）
    ↓
② DispatcherServlet 接收（doDispatch）
    ↓
③ HandlerMapping 找到 HandlerMethod
    （URL → Controller 方法 + 拦截器链）
    ↓
④ HandlerAdapter 准备调用
    （参数解析器、返回值处理器）
    ↓
⑤ 执行拦截器 preHandle（前置）
    ↓
⑥ 调用 Controller 方法（业务逻辑）
    ↓
⑦ 执行拦截器 postHandle（后置）
    ↓
⑧ 处理返回值（@ResponseBody → 序列化 JSON）
    ↓
⑨ 执行拦截器 afterCompletion（完成）
    ↓
⑩ 返回响应
```
doDispatch源码
```
// DispatcherServlet.doDispatch
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) {

    // ① 找到 Handler（Controller 方法 + 拦截器）
    mappedHandler = getHandler(processedRequest);
    // 找不到 → 404

    // ② 找到 HandlerAdapter
    HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

    // ③ 执行拦截器 preHandle
    if (!mappedHandler.applyPreHandle(processedRequest, response)) {
        return;  // 拦截器返回 false → 终止
    }

    // ④ 调用 Controller 方法（核心！）
    mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

    // ⑤ 执行拦截器 postHandle
    mappedHandler.applyPostHandle(processedRequest, response, mv);

    // ⑥ 处理视图/返回值，渲染响应
    processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);

    // ⑦ 执行拦截器 afterCompletion
    mappedHandler.triggerAfterCompletion(...);
}
```
### 请求完整链路
从URL到响应的第一步
```
// 请求：GET /api/order/100?type=vip

// ① 服务器（Tomcat）接收
// ② 找到 DispatcherServlet（/ 映射）
// ③ DispatcherServlet.doDispatch
//    a. HandlerMapping 匹配：
//       /api/order/{id} → OrderController.getOrder(id, type)
//    b. 组装拦截器链
//    c. HandlerAdapter（RequestMappingHandlerAdapter）
//       - 参数解析：HandlerMethodArgumentResolver
//         路径参数 @PathVariable → 100
//         查询参数 @RequestParam → vip
//    d. 调用方法：orderController.getOrder(100, "vip")
// ④ 返回 Order 对象
//    - 返回值处理器：HandlerMethodReturnValueHandler
//      @ResponseBody → MappingJackson2HttpMessageConverter
//      → JSON 序列化
// ⑤ 写出响应
```
拦截器执行顺序
```
// 多个拦截器的执行：
// ① 所有 preHandle 按顺序执行
// ② Controller 执行
// ③ 所有 postHandle 逆序执行
// ④ 所有 afterCompletion 逆序执行

// preHandle1 → preHandle2 → Controller → postHandle2 → postHandle1 → afterCompletion2 → afterCompletion1

// 只要有一个 preHandle 返回 false：
// 后续拦截器不执行，已执行的 afterCompletion 会逆序执行
```
自定义拦截器
```
@Component
public class LoginInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        // 登录校验
        String token = request.getHeader("Authorization");
        if (token == null) {
            response.setStatus(401);
            return false;  // 终止请求
        }
        return true;
    }

    @Override
    public void postHandle(...) {
        // Controller 之后
    }

    @Override
    public void afterCompletion(...) {
        // 请求完成（finally 类似）
    }
}

// 注册
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(loginInterceptor)
            .addPathPatterns("/api/**")          // 拦截路径
            .excludePathPatterns("/api/login");  // 排除路径
    }
}
```

|对比|Filter（过滤器）|Interceptor（拦截器）|
|---|---|---|
|**层级**|Servlet 层|Spring MVC 层|
|**时机**|请求进入 Servlet 前|Controller 调用前后|
|**能拿到**|Request/Response|Controller 方法|
|**依赖**|无（Servlet 规范）|Spring 容器|
|**使用**|编码、跨域|登录、权限、日志|
### 参数绑定与序列化
常用注解
```
@RestController
public class OrderController {

    // 路径参数
    @GetMapping("/order/{id}")
    public Order get(@PathVariable Long id) {}

    // 查询参数
    @GetMapping("/orders")
    public List<Order> list(@RequestParam(defaultValue = "1") int page) {}

    // 请求体 JSON → 对象（Jackson）
    @PostMapping("/order")
    public Order create(@RequestBody Order order) {}

    // 请求头
    @GetMapping("/header")
    public String header(@RequestHeader("token") String token) {}

    // 表单参数
    @PostMapping("/form")
    public String form(@RequestParam String name) {}

    // 对象绑定
    @PostMapping("/obj")
    public String obj(User user) {}  // 表单字段自动绑定到对象
}
```
JSON序列化
```
// @ResponseBody → 返回 JSON
// 原理：MappingJackson2HttpMessageConverter
// ① 读请求体：JSON → Java 对象（@RequestBody）
// ② 写响应：Java 对象 → JSON（@ResponseBody）

// 时间格式：
@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
private LocalDateTime createTime;

// 忽略字段：
@JsonIgnore
private String password;

// 属性名映射：
@JsonProperty("user_name")
private String userName;

// 空值处理：
@JsonInclude(JsonInclude.Include.NON_NULL)
private String remark;  // null 不序列化
```
### 异常处理
全局异常处理器
```
@RestControllerAdvice
public class GlobalExceptionHandler {

    // 业务异常
    @ExceptionHandler(BusinessException.class)
    public Result<?> handleBusiness(BusinessException e) {
        return Result.fail(e.getCode(), e.getMessage());
    }

    // 参数校验异常
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<?> handleValid(MethodArgumentNotValidException e) {
        String msg = e.getBindingResult().getFieldError().getDefaultMessage();
        return Result.fail(400, msg);
    }

    // 兜底异常
    @ExceptionHandler(Exception.class)
    public Result<?> handleException(Exception e) {
        log.error("未知异常", e);
        return Result.fail(500, "系统繁忙");
    }
}

// @RestControllerAdvice = @ControllerAdvice + @ResponseBody
// 原理：HandlerExceptionResolver → ExceptionHandlerExceptionResolver
```
### 面试高频题
#### 题目1：Spring MVC处理请求的流程
```
// DispatcherServlet 接收
// HandlerMapping 找方法
// HandlerAdapter 调用
// 参数绑定、返回值处理
// 拦截器前中后
```
#### 题目2：DispatcherServlet是什么
```
// 前端控制器（Front Controller）
// 统一接收所有请求，分发处理
```
#### 题目3：拦截器和过滤器的区别
```
// Filter：Servlet 层，请求前后
// Interceptor：MVC 层，Controller 前后
```
#### 题目4：@RequestBody怎么把json转对象
```
// Jackson（MappingJackson2HttpMessageConverter）
// JSON → Java 对象（反射）
```
#### 题目5：全局异常怎么处理
```
// @RestControllerAdvice + @ExceptionHandler
// HandlerExceptionResolver
```
#### 题目6：参数绑定流程
```
// HandlerMethodArgumentResolver
// @PathVariable/@RequestParam/@RequestBody 各自解析
```
