## 嵌入式容器
什么是嵌入式容器->三种容器对比->启动流程->端口与配置->替换容器->面试高频题
### 什么是嵌入式容器
嵌入式容器：把Tomcat/Jetty/Undertow内嵌到SpringBoot应用里，Java-jar直接启动，无需外部安装容器。
```
// 传统：外部 Tomcat
// ① 安装 Tomcat
// ② 打包 war 部署到 Tomcat webapps
// ③ 启动 Tomcat

// Spring Boot：嵌入式
// ① java -jar app.jar
// ② Tomcat 在 jar 内部自动启动
// ③ 更简单、更适合微服务/容器化
```
### 三种容器对比

| 对比     | Tomcat | Jetty | Undertow |
| ------ | ------ | ----- | -------- |
| **默认** | ✅ 默认   | 可选    | 可选       |
| **内存** | 中      | 低     | 最低       |
| **性能** | 成熟稳定   | 一般    | 高并发好     |
| **生态** | 最广     | 一般    | 一般       |
| **适用** | 通用     | 嵌入式轻量 | 高并发      |
换容器配置
```
<!-- 排除默认 Tomcat -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<!-- 换成 Undertow -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```
### 启动流程
嵌入式容器什么时候启动
```
// 回顾 Spring Boot 启动流程：
// refreshContext() 的 refresh() 中
// → onRefresh() 钩子 → ServletWebServerApplicationContext.onRefresh()
// → createWebServer() → 创建并启动嵌入式容器
```
源码流程
```
// ServletWebServerApplicationContext.onRefresh
@Override
protected void onRefresh() {
    super.onRefresh();
    try {
        // 创建 Web 服务器（Tomcat）
        createWebServer();
    } catch (Throwable ex) {
        throw new ApplicationContextException("Unable to start web server", ex);
    }
}

// createWebServer：
private void createWebServer() {
    // ① 获取 WebServerFactory（TomcatServletWebServerFactory）
    ServletWebServerFactory factory = getWebServerFactory();

    // ② 创建 WebServer（Tomcat）
    this.webServer = factory.getWebServer(getSelfInitializer());

    // ③ 启动
    this.webServer.start();
}
```
Tomcat工厂创建流程
```
// TomcatServletWebServerFactory.getWebServer
public WebServer getWebServer(ServletContextInitializer... initializers) {
    // ① 创建 Tomcat 实例
    Tomcat tomcat = new Tomcat();

    // ② 配置端口
    if (getPort() > 0) {
        tomcat.setPort(getPort());
    }

    // ③ 添加 Connector（连接器）
    // 处理 HTTP 连接

    // ④ 准备 Web 应用上下文
    // addWebapp → StandardContext

    // ⑤ 注册 DispatcherServlet
    // 把 Spring MVC 的 DispatcherServlet 挂到 Tomcat

    // ⑥ 初始化并启动
    tomcat.getConnector();
    tomcat.start();

    return new TomcatWebServer(tomcat);
}
```
关键点
```
// ① 容器启动在 onRefresh（Spring refresh 的第 9 步）
// ② 此时 Bean 还没全部创建完（实例化单例在第 11 步）
// ③ DispatcherServlet 是后注册的（ServletWebServerInitializedEvent 后）
// ④ 端口监听在 refresh 完成后真正就绪
```
### 端口与配置
```
server:
  port: 8080                    # 端口
  address: 0.0.0.0              # 绑定地址
  servlet:
    context-path: /api          # 上下文路径（访问前缀）
    encoding:
      charset: UTF-8
  tomcat:
    max-threads: 200            # 最大线程数
    min-spare-threads: 10       # 最小空闲线程
    max-connections: 10000      # 最大连接数
    accept-count: 100           # 等待队列
    uri-encoding: UTF-8
  shutdown: graceful            # 优雅停机（2.3+）
```
随机端口
```
server:
  port: 0     # 随机端口

// 获取实际端口：
@Autowired
private ServletWebServerApplicationContext context;

int port = context.getWebServer().getPort();
```
### Tomcat的调优参数
| 参数                     | 说明     | 建议      |
| ---------------------- | ------ | ------- |
| **max-threads**        | 最大工作线程 | 200~500 |
| **min-spare-threads**  | 最小空闲线程 | 10~20   |
| **max-connections**    | 最大连接数  | 10000   |
| **accept-count**       | 等待队列长度 | 100~200 |
| **connection-timeout** | 连接超时   | 20000ms |
```
server:
  tomcat:
    max-threads: 500
    min-spare-threads: 50
    max-connections: 20000
    accept-count: 200
    connection-timeout: 20000
```
优化思路
```
// ① max-threads 不是越大越好（线程多了上下文切换）
// ② 根据 CPU 核数和 IO 类型调整
// ③ 高并发 + 短请求：线程数可以大些
// ④ 长连接：连接数和线程数权衡
// ⑤ 配合线程池监控调整
```
### 面试高频题
#### 题目1：嵌入式容器是怎么启动的
```
// refresh 的 onRefresh 阶段
// createWebServer → Tomcat 工厂创建
// TomcatServletWebServerFactory.getWebServer
```
#### 题目2：怎么换容器
```
// 排除 starter-tomcat + 加 starter-jetty/undertow
```
#### 题目3：Tomcat和Undertow选哪个
```
// 通用选 Tomcat（默认、生态广）
// 高并发轻量选 Undertow（内存小）
```
#### 题目4：端口怎么配
```
// server.port，0 是随机端口
```
#### 题目5：优雅停机是什么
```
// server.shutdown=graceful
// 停止时先停止接收新请求，处理完存量再退出
```
#### 题目6：context-path是什么
```
// 访问前缀：server.servlet.context-path=/api
// 所有请求带 /api 前缀
```
