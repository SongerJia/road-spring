## 链路追踪
为什么需要链路追踪->核心概念->Sleuth使用->采样与传播->Zipkin展示->面试高频题
### 为什么需要链路追踪
```
// 一个请求经过多个服务：
// 客户端 → 网关 → 订单服务 → 库存服务 → 支付服务

// 问题：
// ① 请求慢了，慢在哪个服务？
// ② 请求失败了，哪个环节出错？
// ③ 日志分散在各个服务，怎么串起来？

// 解决：链路追踪（分布式追踪）
// 记录一次请求经过的所有服务，串成一条"链路"
```
### 核心概念
Trace和Span
```
// ① Trace（链路）：一次请求的完整路径（一棵树）
// ② Span（跨度）：链路中的一个环节（一个服务调用）

// 一个 Trace 由多个 Span 组成：
// Trace（traceId=abc123）
//  ├── Span1：网关（parent 无）
//  │    └── Span2：订单服务（parent=Span1）
//  │         └── Span3：库存服务（parent=Span2）
//  │         └── Span4：支付服务（parent=Span2）
//  └── 所有 Span 共享同一个 traceId
```
三个关键id

|ID|说明|作用|
|---|---|---|
|**traceId**|整条链路的唯一 ID|串联所有服务日志|
|**spanId**|每个环节的 ID|标识单个服务调用|
|**parentId**|父环节的 ID|构建调用树|
```
// 传递方式：请求头
// X-B3-TraceId: abc123
// X-B3-SpanId: span2
// X-B3-ParentSpanId: span1
```
### Sleuth使用
```
// Spring Cloud Sleuth：链路追踪客户端
// 自动生成 traceId/spanId，通过请求头传递
// 配合 Zipkin 展示

// 注意：Sleuth 已停止新特性（官方转向 Micrometer Tracing）
// 但面试还是常问 Sleuth + Zipkin
```
接入
```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>

<!-- 或新方案 -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
```
自动完成的内容
```
// ① 每个请求生成 traceId + spanId
// ② 通过 HTTP 请求头传递（Feign/RestTemplate 自动）
// ③ 日志自动带 traceId

// 日志效果（加了 traceId）：
// 2026-09-22 10:00:00 [order-service,abc123,span2] 处理订单
// 2026-09-22 10:00:01 [stock-service,abc123,span3] 扣库存
// ↑ traceId 相同 → grep abc123 就能串起整个链路
```
### 采样与传播
采样率
```yaml
spring:
  sleuth:
    sampler:
      probability: 0.1   # 采样率 10%（默认）
```
```
// 为什么采样？
// 全量采集：数据量大、存储开销大
// 采样 10%：统计意义足够，成本低

// 生产建议：
// 默认 0.1（10%）即可
// 排障时可以临时调高
```
传播 Propagation
```
// traceId 怎么跨服务传递：
// ① HTTP：请求头（X-B3-TraceId）
// ② MQ：消息头
// ③ 线程池：ThreadLocal（注意线程切换）

// 注意：
// 异步线程会丢 traceId（线程池的坑）
// 解决：HystrixContext / 手动传递
```
### Zipkin展示
```
// Zipkin：链路追踪的"可视化展示平台"
// Sleuth 采集 → 上报 Zipkin → 图形化查看链路
```
接入
```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-sleuth-zipkin</artifactId>
</dependency>
```
```
spring:
  zipkin:
    base-url: http://zipkin-server:9411
  sleuth:
    sampler:
      probability: 0.1
```
能看到什么
```
// ① 链路拓扑：请求经过哪些服务
// ② 每个服务的耗时（定位慢在哪）
// ③ 每段调用的状态（成功/失败）
// ④ 按 traceId 搜索具体链路

// 排查场景：
// 订单接口慢 → Zipkin 看链路
// → 发现库存服务耗时 2s → 定位问题
```
### 高频面试题
#### 题目1：链路追踪解决什么问题
```
// 一次请求跨多服务，定位慢/错误在哪
// traceId 串联日志
```
#### 题目2：Trace和Span
```
// Trace：整条链路
// Span：链路上的一个环节
// 共享 traceId，各自 spanId
```
#### 题目3：TraceId怎么传递
```
// HTTP 请求头（X-B3-TraceId）
// MQ 消息头、线程上下文
```
#### 题目4：为什么采样
```
// 全量采集开销大
// 10% 采样够用（统计意义）
```
#### 题目5：异步线程会丢traceId吗
```
// 会！线程切换丢 ThreadLocal
// 需要手动传递（装饰器/包装）
```
#### 题目6：怎么定位慢接口
```
// Zipkin 看链路 → 每段耗时 → 定位慢的服务
```