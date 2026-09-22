## 远程调用
远程调用方式对比->Feign是什么->核心用法->工作原理->超时与重试->面试高频题
### 远程调用方式对比
```
// 服务间调用的三种方式：

// ① RestTemplate（手动）
@Autowired
private RestTemplate restTemplate;

public Stock getStock(Long id) {
    // 手动拼接 URL、手动转换结果
    return restTemplate.getForObject(
        "http://stock-service/api/stock/" + id, Stock.class);
}

// ② WebClient（响应式）
// 异步、链式，学习成本高

// ③ OpenFeign（声明式）✅ 推荐
@FeignClient(name = "stock-service")
public interface StockClient {
    @GetMapping("/api/stock/{id}")
    Stock getStock(@PathVariable("id") Long id);
}
// 像调用本地方法一样调用远程服务
```

| 对比      | RestTemplate | WebClient | OpenFeign |
| ------- | ------------ | --------- | --------- |
| **风格**  | 手动           | 响应式       | 声明式 ✅     |
| **代码量** | 多            | 中         | 少         |
| **异步**  | 支持           | ✅ 原生      | 同步为主      |
| **可读性** | 差            | 中         | 好         |
| **推荐**  | 简单场景         | WebFlux   | ✅ 微服务默认   |
### Feign是什么
OpenFeign：声明式HTTP客户端。定义一个接口，像调用本地方法一样调用远程服务，SpringCloud自动生成实现。
```
// ① 启用 Feign
@SpringBootApplication
@EnableFeignClients  // 扫描 @FeignClient 接口
public class OrderApplication { ... }

// ② 定义调用接口
@FeignClient(
    name = "stock-service",           // 服务名（注册中心）
    fallback = StockClientFallback.class  // 降级处理
)
public interface StockClient {

    @GetMapping("/api/stock/{id}")
    Stock getStock(@PathVariable("id") Long id);

    @PostMapping("/api/stock/deduct")
    Result deduct(@RequestBody DeductRequest request);

    @PutMapping("/api/stock/{id}")
    Stock update(@PathVariable("id") Long id, @RequestBody Stock stock);
}

// ③ 使用
@Service
public class OrderService {
    @Autowired
    private StockClient stockClient;

    public void createOrder() {
        Stock stock = stockClient.getStock(100L);  // 像本地调用
    }
}
```
### 核心配置
```
feign:
  client:
    config:
      default:                      # 默认配置
        connect-timeout: 5000       # 连接超时
        read-timeout: 5000          # 读取超时
        loggerLevel: basic          # 日志级别
      stock-service:                # 指定服务配置
        read-timeout: 10000
```
三种调用方式
```
// ① 路径参数
@GetMapping("/api/stock/{id}")
Stock getStock(@PathVariable("id") Long id);

// ② 查询参数
@GetMapping("/api/stock")
Stock findByCode(@RequestParam("code") String code);

// ③ 请求体
@PostMapping("/api/stock/deduct")
Result deduct(@RequestBody DeductRequest request);
```
拦截器
```
// 请求头传递（登录态、traceId）
@Configuration
public class FeignConfig {

    @Bean
    public RequestInterceptor requestInterceptor() {
        return requestTemplate -> {
            // 传递用户信息
            ServletRequestAttributes attrs =
                (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
            if (attrs != null) {
                String token = attrs.getRequest().getHeader("Authorization");
                requestTemplate.header("Authorization", token);
            }
            // 传递 traceId（链路追踪）
            requestTemplate.header("X-Trace-Id", TraceContext.getTraceId());
        };
    }
}
```
### 工作原理
Feign的实现原理
```
// ① @EnableFeignClients 扫描 @FeignClient 接口
// ② 为每个接口生成动态代理（JDK 代理）
// ③ 代理内部：
//    a. 解析方法上的注解（@GetMapping 等）
//    b. 组装 HTTP 请求（方法名、参数 → URL、Body）
//    c. 通过 LoadBalancer 选择实例
//    d. 发起 HTTP 调用
//    e. 解析响应为返回类型
```
核心流程
```
// 调用 stockClient.getStock(100L)
// ① FeignInvocationHandler.invoke（JDK 代理）
// ② MethodHandler.invoke（SynchronousMethodHandler）
// ③ 构建 Request（URL、Headers、Body）
// ④ LoadBalancer 选择实例（服务名 → 地址）
// ⑤ Client.execute（发 HTTP）
// ⑥ 解析 Response → Stock 对象
```
关键组件
```
// ① Contract：解析接口注解 → 方法元数据
// ② Encoder/Decoder：参数编码、响应解码（Jackson）
// ③ Client：HTTP 客户端（默认 JDK，可换 OkHttp）
// ④ LoadBalancerClient：负载均衡
// ⑤ RequestInterceptor：请求拦截器（传头）
```
### 超时与重试
超时配置
```
feign:
  client:
    config:
      default:
        connect-timeout: 3000     # 连接超时（默认 10s）
        read-timeout: 3000        # 读取超时（默认 60s）
```
重试机制
```
// Feign 默认不重试（防止重复请求）
// 需要时配置：

// ① Feign 重试（仅重试网络异常/超时）
@Bean
public Retryer feignRetryer() {
    return new Retryer.Default(100, 1000, 3);  // 间隔100ms，最多重试3次
}

// ② 注意：
// GET 等幂等请求可以重试
// POST 等非幂等请求要小心（重复提交！）
// 重试 + 接口幂等性配合使用
```
超时问题
```
// 场景：服务调用慢
// ① connect-timeout：连不上（网络问题）
// ② read-timeout：连上了但没返回（服务慢）
// ③ 默认：连接 10s、读取 60s（太长，要调小）

// 最佳实践：
// 连接 3s + 读取 5s（业务可容忍范围）
// 超时后走降级（fallback）或重试（幂等）
```
### 降级 fallback
```
// 服务不可用时，执行兜底逻辑

// ① 定义降级类
@Component
public class StockClientFallback implements StockClient {
    @Override
    public Stock getStock(Long id) {
        // 兜底：返回空/默认值
        return new Stock();
    }
}

// ② 指定降级
@FeignClient(name = "stock-service", fallback = StockClientFallback.class)
public interface StockClient { ... }

// ③ 开启降级（默认关闭）
feign:
  circuitbreaker:
    enabled: true
```
降级 vs 熔断
```
// fallback：调用失败/超时时的兜底（降级）
// 熔断：连续失败后，暂时不调用（直接走降级）
```
### 面试高频题
#### 题目1：Feign是什么？和RestTemplate区别
```
// 声明式 HTTP 客户端
// 像本地方法调用远程服务
// 代码量少、可读性好
```
#### 题目2：Feign的工作原理
```
// 接口 → 动态代理 → 解析注解 → 组装请求
// LoadBalancer 选实例 → 发 HTTP → 解析响应
```
#### 题目3：Feign是怎么传请求头
```
// RequestInterceptor 拦截器
// 传 Token、用户信息、traceId
```
#### 题目4：Feign怎么配置超时
```
// connect-timeout + read-timeout
// 连接 3s + 读取 5s 左右
```
#### 题目5：Feign会重试吗
```
// 默认不重试
// 自定义 Retryer，但非幂等请求要小心
```
#### 题目6：Feign降级怎么实现
```
// fallback 类 + @FeignClient(fallback=...)
// 配合熔断（circuitbreaker）
```
