## 消息驱动
为什么需要->核心概念->基本使用->消息模型->消费组与分区->与原生MQ对比->面试高频题
### 为什么需要
消息驱动场景
```
// 微服务间解耦 + 异步：
// ① 下单后发通知（不用等）
// ② 订单状态变更 → 通知库存/物流（解耦）
// ③ 高峰削峰（秒杀请求进队列）
// ④ 事件驱动（订单创建 → 其他服务响应）
```
问题：MQ生态碎片化
```
// 不同项目用不同 MQ：
// RabbitMQ、Kafka、RocketMQ
// 每个 MQ 的 API 不同：
// 用 RabbitMQ 写一套，换 Kafka 要重写！

// Spring Cloud Stream：统一编程模型
// 屏蔽底层 MQ 差异，一套代码切换 MQ
```
### 核心概念
三大概念 Binder绑定器
```
Spring Cloud Stream：
┌────────────────────────────────┐
│ 输入通道（Input）/ 输出通道（Output）│  ← 应用视角（消息的收发）
└────────────────────────────────┘
              ↓ 绑定（Binder）
┌────────────────────────────────┐
│        Binder（绑定器）           │  ← 屏蔽 MQ 差异
│   RabbitBinder / KafkaBinder    │
└────────────────────────────────┘
              ↓
        RabbitMQ / Kafka / RocketMQ（底层）
```

|概念|说明|
|---|---|
|**Binder**|绑定器：连接 MQ 的适配层|
|**Channel**|通道：Input（收）/ Output（发）|
|**Message**|消息（消息头 + 消息体）|
### 基本使用
```
<!-- 以 RocketMQ 为例（或 kafka / rabbit） -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rocketmq</artifactId>
</dependency>
```
定义消息通道
```
public interface OrderSink {
    String INPUT = "order-input";
    String OUTPUT = "order-output";

    @Input(INPUT)
    SubscribableChannel input();

    @Output(OUTPUT)
    MessageChannel output();
}

// 启用
@SpringBootApplication
@EnableBinding(OrderSink.class)
public class OrderApplication { ... }
```
发送消息
```
@Service
public class OrderProducer {

    @Autowired
    @Qualifier(OrderSink.OUTPUT)
    private MessageChannel output;

    public void orderCreated(Long orderId) {
        // 构建消息
        Message<String> message = MessageBuilder
            .withPayload("order:" + orderId)      // 消息体
            .setHeader("contentType", "application/json")
            .build();
        output.send(message);  // 发送
    }
}
```
接收消息
```
@Component
public class OrderConsumer {

    @StreamListener(OrderSink.INPUT)  // 监听输入通道
    public void onOrderCreated(String message) {
        System.out.println("收到订单通知: " + message);
        // 业务处理：发短信、更新统计
    }
}
```
### 消息模型
消息结构
```
// Message<T>：
// ① 消息头（headers）：消息 ID、类型、业务头
// ② 消息体（payload）：业务数据
// ③ 消息元数据

Message<OrderEvent> msg = MessageBuilder
    .withPayload(orderEvent)   // 业务对象
    .setHeader("traceId", "abc")  // 自定义头（链路追踪）
    .build();
```
消息转换
```
// 消息体自动序列化（JSON）
// 接收时反序列化：
@StreamListener(OrderSink.INPUT)
public void onOrder(OrderEvent event) {  // 自动转对象
    // ...
}
```
### 消费组与分区
消费组
```
// 问题：
// 订单服务部署 3 个实例，都监听同一个队列
// 一条消息 → 3 个实例都消费 → 重复处理！

// 解决：消费组
// 同一组的实例共享消费（一条消息只被一个实例处理）
// 不同组的实例各自消费（广播给多组）

// 配置：
spring:
  cloud:
    stream:
      bindings:
        order-input:
          group: order-group   # 同组实例共享消费

订单服务 3 个实例（同组 order-group）：
消息 1 → 实例1 处理 ✅
消息 2 → 实例2 处理 ✅
消息 3 → 实例3 处理 ✅
（每条消息只处理一次，负载均衡）
```
分区 Partition
```
// 分区：保证"同一个 key 的消息"到同一个实例
// 例：同一个订单的所有消息 → 同一个实例（顺序处理）

spring:
  cloud:
    stream:
      bindings:
        order-input:
          group: order-group
          consumer:
            partitioned: true     # 分区消费
      instance-count: 3           # 实例数
      instance-index: 0           # 当前实例索引
```
```
// 发送时指定分区 key：
Message<String> message = MessageBuilder
    .withPayload("order:" + orderId)
    .setHeader("partitionKey", orderId)  // 按订单 ID 分区
    .build();
// 同一个订单的消息 → 同一个实例 → 有序处理
```
为什么需要分区
```
// 场景：订单状态流转（创建 → 支付 → 发货）
// 同一个订单的消息如果被不同实例处理 → 状态错乱！
// 分区保证同 key 消息有序
```
### 与原生MQ对比
| 对比        | Spring Cloud Stream | 原生 MQ API  |
| --------- | ------------------- | ---------- |
| **统一性**   | ✅ 一套 API            | 各 MQ 不同    |
| **切换 MQ** | 改配置即可               | 重写代码       |
| **学习成本**  | 中（概念多）              | 低          |
| **灵活性**   | 受框架限制               | 最强         |
| **适用**    | 微服务统一消息             | 深入使用 MQ 特性 |
什么时候用Stream
```
// ① 微服务统一用消息（团队规范）
// ② 可能切换 MQ（不想绑定）
// ③ 简单收发场景

// 什么时候用原生
// ① 深度使用 MQ 特性（Kafka 精确一次语义）
// ② 复杂流处理
// ③ 团队已熟练原生 API
```
### 面试高频题
#### 题目1：为什么用消息中间件
```
// 解耦、异步、削峰、事件驱动
```
#### 题目2：Spring cloud stream是什么
```
// 统一消息编程模型
// Binder 屏蔽 MQ 差异
// 一套代码切换 RabbitMQ/Kafka/RocketMQ
```
### 题目3：消费组是什么
```
// 同组实例共享消费（一条消息一个实例处理）
// 防止重复消费
```
#### 题目4：分区是什么
```
// 同 key 消息到同一实例
// 保证消息有序
```
#### 题目5：怎么防止重复消费
```
// ① 消费组（同组不重复）
// ② 业务幂等（MQ 重试时重复）
// ③ 消息去重（记录已处理消息 ID）
```
#### 题目6：消息丢失怎么办
```
// ① 生产者：确认机制（ack）
// ② 中间件：持久化
// ③ 消费者：手动 ack（处理完再确认）
```

