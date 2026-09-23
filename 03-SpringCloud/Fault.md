## 容错
容错是什么->超时控制->重试机制->幂等性->隔离策略->面试高频题
### 容错是什么
服务容错：通过超时、重试、幂等、隔离等手段，让系统在部分组件故障时仍能正常工作或优雅降级。
```
容错的四板斧：
① 超时控制：不让调用无限等待
② 重试机制：网络抖动时重试
③ 幂等性：重复执行结果一致（配合重试）
④ 服务隔离：故障不扩散
```
### 超时控制
为什么需要超时
```
// 没有超时的问题：
// 下游服务挂了/网络断 → 请求一直挂起
// 线程被占住 → 线程池耗尽 → 自己也被拖垮

// 超时：约定时间没响应 → 放弃（走降级/报错）
```
各层超时配置
```
# ① 网关超时
spring:
  cloud:
    gateway:
      httpclient:
        connect-timeout: 3000
        response-timeout: 5000

# ② Feign 超时
feign:
  client:
    config:
      default:
        connect-timeout: 3000
        read-timeout: 5000

# ③ 数据库超时
spring:
  datasource:
    hikari:
      connection-timeout: 30000
```
超时层级
```
// 超时要"层层递减"：
// 外层超时 > 内层超时（留给内层处理的时间）

// 网关 5s > 服务A 4s > 服务B 3s > 数据库 2s
// 这样内层先超时，外层有兜底
```
### 重试机制
重试的场景
```
// 网络抖动（瞬时故障）→ 重试能解决
// 下游短暂不可用 → 重试能解决
// 下游永久故障 → 重试无用（反而加重负载）

// 判断：只对"瞬时故障"重试
```
重试的配置
```
# Feign 重试
feign:
  client:
    config:
      default:
        connect-timeout: 3000
        read-timeout: 5000
# 自定义重试（默认不重试）
```
```
// 手动重试（Spring Retry）
@Configuration
public class RetryConfig {

    @Bean
    public RetryTemplate retryTemplate() {
        RetryTemplate template = new RetryTemplate();

        // 最多重试 3 次，间隔 1s
        SimpleRetryPolicy retryPolicy = new SimpleRetryPolicy(3);
        FixedBackOffPolicy backOffPolicy = new FixedBackOffPolicy();
        backOffPolicy.setBackOffPeriod(1000);

        template.setRetryPolicy(retryPolicy);
        template.setBackOffPolicy(backOffPolicy);
        return template;
    }
}

// 使用
public Stock getStock(Long id) {
    return retryTemplate.execute(context -> {
        return stockClient.getStock(id);  // 失败自动重试
    });
}
```
重试的重点
```
// ① 只重试"幂等"操作（GET、查询）
// ② 写操作重试要配合幂等性（否则重复扣款！）
// ③ 重试要有间隔（backoff），不要疯狂重试
// ④ 重试次数有限（3 次左右）
// ⑤ 重试 + 熔断配合（熔断期间不重试）
```
### 幂等性
什么是幂等
```
// 幂等：同一个操作执行多次，结果和一次一样

// 幂等：GET 查询、PUT 覆盖、DELETE（按条件）
// 非幂等：POST 创建、扣款、减库存

// 场景：
// 网络超时 → 客户端重试 → 服务端执行了两次！
// 扣款扣了两次 → 需要幂等
```
幂等方案
```
// ① 唯一 ID（token）方案
// 前端/调用方生成唯一请求 ID
// 服务端记录已处理的 ID，重复则直接返回

// ② 数据库唯一约束
// 订单号唯一索引 → 重复插入报错 → 捕获返回成功

// ③ 状态机
// 订单状态：待支付 → 已支付 → 已发货
// 已支付再触发支付 → 拒绝（状态不符）

// ④ 乐观锁（version）
// UPDATE ... WHERE version = 1
// 更新失败说明已处理

// ⑤ 分布式锁
// 相同请求 ID 加锁，只处理一次
```
唯一ID方案
```
// ① 调用方生成请求 ID
String requestId = UUID.randomUUID().toString();

// ② 服务端先查是否已处理
public Result createOrder(OrderRequest req, String requestId) {
    // 幂等检查
    if (idempotentService.isProcessed(requestId)) {
        return Result.success(已处理的结果);  // 直接返回
    }

    // 加分布式锁（防并发）
    boolean locked = lock.tryLock("idempotent:" + requestId, 5, TimeUnit.SECONDS);
    if (locked) {
        try {
            // 再次检查（double check）
            if (idempotentService.isProcessed(requestId)) {
                return Result.success(...);
            }
            // 执行业务
            Order order = orderService.create(req);
            // 标记已处理（Redis 带过期时间）
            idempotentService.markProcessed(requestId, order.getId());
            return Result.success(order);
        } finally {
            lock.unlock();
        }
    }
    // 没拿到锁：另一个请求在处理
    return Result.fail("处理中，请稍后");
}
```
幂等 vs 重试的关系
```
// 重试的前提是幂等！
// 重试 = 客户端行为（多发一次）
// 幂等 = 服务端保障（多执行无害）
// 两者配合：重试才安全
```
### 服务隔离
隔离方式
```
// ① 线程池隔离：
// 每个服务一个线程池
// 一个服务慢 → 只占自己的线程池
// 不拖垮其他调用

// ② 信号量隔离：
// 限制并发数（轻量）

// ③ 舱壁模式（Bulkhead）：
// 像船舱隔间，一个进水不沉整船
// Sentinel 的线程数限流就是
```
例子
```
// 服务A 同时调服务B、服务C
// 线程池隔离：
// A→B 的线程池：10 个
// A→C 的线程池：10 个
// B 挂了 → 只占 A→B 的池 → C 不受影响 ✅
```
### 面试高频题
#### 题目1：服务容错有哪些手段
```
// 超时、重试、幂等、熔断、降级、限流、隔离
```
#### 题目2：超时为什么要重重递减
```
// 外层 > 内层，内层先超时，外层兜底
```
#### 题目3：重试要注意什么
```
// 只重试幂等操作、有间隔、次数有限
```
#### 题目4：什么是幂等，怎么实现
```
// 重复执行结果一致
// 唯一 ID、唯一约束、状态机、乐观锁、分布式锁
```
#### 题目5：幂等和重试的关系
```
// 重试的前提是幂等
// 重试多发一次，幂等保证无害
```
#### 题目6：服务隔离怎么做
```
// 线程池隔离、信号量、舱壁模式
```

