> **模块边界**：Dubbo / gRPC 等 RPC 框架 → 中间件模块；消息中间件本体（Kafka / RocketMQ）→ 中间件模块；Spring Security 细节 → Spring 模块。本模块覆盖微服务治理的全部横切能力。

**① Spring Cloud 总览**

1. 是什么、微服务治理全景
2. 版本对应（年份版本 + 与 Boot 强绑定）
3. Spring Cloud vs Spring Cloud Alibaba
4. 微服务架构总览与拆分原则（DDD、限界上下文）
5. 组件选型现状与 Netflix 组件退役（Eureka / Ribbon / Hystrix / Zuul）

**② 服务注册与发现** 6. 注册中心的作用 7. Eureka 8. Nacos（含集群与持久化） 9. Consul / Zookeeper 10. 服务注册、心跳与健康检查、临时实例 vs 持久实例 11. AP vs CP

**③ 配置中心** 12. Spring Cloud Config 13. Nacos Config 14. 配置动态刷新（@RefreshScope 机制与限制）与不刷新排查 15. Spring Cloud Bus（配置刷新广播） 16. 命名空间与分组、配置优先级与灰度

**④ 负载均衡** 17. Ribbon 18. Spring Cloud LoadBalancer 19. 负载均衡策略（轮询 / 随机 / 权重 / 一致性哈希 / 最小活跃数）

**⑤ 远程调用** 20. RestTemplate 21. OpenFeign 与原理（动态代理） 22. Feign 的拦截器与编解码、日志级别 23. Feign 的超时与重试、与 Sentinel / Resilience4j 的整合

**⑥ 服务容错** 24. Hystrix 25. Sentinel：流控模式（直接 / 关联 / 链路）、流控效果（快速失败 / Warm Up / 排队等待）、熔断策略（慢调用比例 / 异常比例 / 异常数）、规则持久化 26. Resilience4j 27. 限流算法（计数器 / 滑动窗口 / 漏桶 / 令牌桶） 28. 舱壁隔离、超时与重试

**⑦ 网关** 29. Spring Cloud Gateway（基于 WebFlux，不可与 MVC 混用） 30. 断言与过滤器 31. 网关限流与鉴权 32. 动态路由与灰度发布

**⑧ 链路追踪与可观测** 33. Sleuth + Zipkin 及其被 Micrometer Tracing / OpenTelemetry 取代 34. TraceId / SpanId 与日志关联、采样率 35. Micrometer + Prometheus + Grafana 监控与告警

**⑨ 消息驱动** 36. Spring Cloud Stream 与 Binder

**⑩ 分布式事务** 37. Seata：TC / TM / RM 三角色与 AT 模式（全局锁 + undo log） 38. 分布式事务方案对比（2PC / 3PC / TCC / SAGA / 本地消息表 / 可靠消息最终一致）

**⑪ 安全与交付** 39. OAuth2 / JWT 与网关鉴权 40. 蓝绿部署与灰度发布 41. 服务的优雅上下线与滚动发布

**⑫ 常见坑与落地规范** 42. 常见坑清单（版本不对齐 / Gateway 与 MVC 冲突 / Feign 首次超时 / Sentinel 规则丢失 / Nacos 配置不刷新 / 循环调用） 43. 微服务落地规范（接口契约、超时与重试的统一约定、错误码规范、链路透传） 44. 边界：接口幂等与防重 → 分布式模块 45. 边界：Dubbo / gRPC → 中间件模块 46. 边界：Kafka / RocketMQ 本体 → 中间件模块