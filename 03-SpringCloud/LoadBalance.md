## 负载均衡
负载均衡是什么->服务端 vs 客户端->常见算法->Spring Cloud LoadBalance->与Nginx对比->面试高频题
### 负载均衡是什么？
负载均衡：把请求分发到多个服务器实例上，避免单个实例过载，提高可用性和吞吐量。
```
请求 ──→ 负载均衡器 ──→ 实例1（192.168.1.1:8080）
                  ├──→ 实例2（192.168.1.2:8080）
                  └──→ 实例3（192.168.1.3:8080）
```
为什么要负载均衡
```
// ① 分摊压力（多个实例分担）
// ② 高可用（实例挂了，流量转到其他）
// ③ 弹性扩缩容（加实例自动分担）
```
### 服务端 vs 客户端
服务端负载均衡
```
// 负载均衡器独立部署（中间件）
// 客户端 → LB → 实例

// 代表：Nginx（HTTP）、LVS、SLB（云）

// 特点：
// ① 客户端无感知
// ② 集中管理
// ③ 可能成为瓶颈（单点）
```
客户端负载均衡
```
// 负载均衡逻辑在客户端（调用方）
// 客户端从注册中心拿实例列表 → 自己选

// 代表：Spring Cloud LoadBalancer、Ribbon

// 特点：
// ① 无中心（不依赖 LB 组件）
// ② 客户端知道所有实例
// ③ 集成服务发现
```

|对比|服务端（Nginx）|客户端（LoadBalancer）|
|---|---|---|
|**位置**|独立组件|调用方内|
|**发现服务**|静态配置/动态|注册中心|
|**单点**|有（可集群）|无|
|**适用**|网关/对外|服务间调用|
|**Spring Cloud**|网关层|服务间|
### 常见算法
Round Robin 轮询
```
// 依次分发：1, 2, 3, 1, 2, 3...
// 最简单，不考虑服务器差异
// 适用：实例配置相同的场景
```
Random 随机
```
// 随机选一个实例
// 请求多时趋于均匀
```
Weighted 加权轮询
```
// 按权重分配：权重高的实例接更多请求
// 适用：配置不同的实例（8 核 vs 4 核）
// Nacos 控制台可以配权重
```
Least Connections 最少连接
```
// 选当前连接数最少的实例
// 适用：请求处理时间差异大的场景
// 避免长请求堆积
```
一致性hash
```
// 相同 key 的请求打到同一实例
// 适用：有状态服务（Session）、缓存
```
算法选择
```
// 无状态服务：轮询 / 随机（简单）
// 有差异实例：加权轮询
// 长请求/慢服务：最少连接
// 有状态：一致性哈希
```
### Spring Cloud LoadBalance
```
// Spring Cloud 的客户端负载均衡组件
// （Ribbon 已进入维护，LoadBalancer 是替代）

// 核心：@LoadBalanced 注解标记 RestTemplate
@Configuration
public class RestTemplateConfig {
    @Bean
    @LoadBalanced  // 开启负载均衡
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

// 使用（服务名调用，不用写 IP）：
@Service
public class OrderService {
    @Autowired
    private RestTemplate restTemplate;

    public String callStock() {
        // 服务名 stock-service 会被解析成实例地址
        String result = restTemplate.getForObject(
            "http://stock-service/api/stock", String.class);
        return result;
    }
}
```
原理
```
// @LoadBalanced 标记 RestTemplate
// 注入 LoadBalancerInterceptor（拦截器）
// 调用时拦截 "http://服务名/..." 请求
// ① 从注册中心获取服务实例列表
// ② 用负载均衡算法选一个
// ③ 替换服务名为真实地址
// ④ 发起真实调用
```
自定义负载均衡规则
```
// 自定义选择算法
@Configuration
public class LoadBalancerConfig {
    @Bean
    public ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(
            ObjectProvider<ServiceInstanceListSupplier> suppliers,
            Environment environment) {
        return new RandomLoadBalancer(
            suppliers, environment.getProperty("loadbalancer.client.name"));
    }
}
```
### OpenFeign的负载均衡
```
// OpenFeign 底层自动集成 LoadBalancer
// 声明式调用 + 自动负载均衡

@FeignClient(name = "stock-service")  // 服务名
public interface StockClient {
    @GetMapping("/api/stock/{id}")
    Stock getStock(@PathVariable("id") Long id);
}

// 调用时自动：
// ① 根据服务名查注册中心
// ② 负载均衡选实例
// ③ 发起 HTTP 调用
```
### 与Nginx对比
```
// 网关层（Nginx/Spring Cloud Gateway）：
// 对外统一入口：域名 → 网关 → 服务

// 服务间（LoadBalancer/Feign）：
// 内部服务调用：服务A → 服务B

// 两级负载均衡：
// ① 网关层：Nginx 负载均衡到服务实例
// ② 服务间：LoadBalancer 负载均衡

// 面试回答：
// "微服务有两层负载均衡：
// 对外用 Nginx/Gateway（服务端 LB），
// 服务间用 Spring Cloud LoadBalancer（客户端 LB）。"
```
### 面试高频题
#### 题目1：负载均衡算法有哪些？
```
// 轮询、随机、加权轮询、最少连接、一致性哈希
// 无状态用轮询/随机，有差异加权，有状态哈希
```
#### 题目2：客户端和服务端负载均衡区别
```
// 客户端：调用方自己选（LoadBalancer）
// 服务端：中间件分发（Nginx）
```
#### 题目3：@LoadBalance原理
```
// 拦截器把服务名解析成实例地址
// 注册中心 + 负载均衡算法
```
#### 题目4：Ribbon和LoadBalancer
```
// Ribbon 维护，LoadBalancer 是替代（Spring Cloud 2020+）
```
#### 题目5：负载均衡和注册中心的关系
```
// 注册中心提供实例列表
// 负载均衡从列表里选
// 注册中心是"数据源"，LB 是"选择器"
```
