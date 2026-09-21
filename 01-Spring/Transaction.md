## 事务管理
事务是什么->声明式vs编程式->实现原理->传播行为->隔离级别->失效场景->面试高频题
### Spring事务是什么
两种事务方式
```
// ① 编程式事务（手动控制）
@Autowired
private TransactionTemplate transactionTemplate;

public void transfer() {
    transactionTemplate.execute(status -> {
        // 业务逻辑
        accountDao.decrease(1, 100);
        accountDao.increase(2, 100);
        return null;
    });
    // 抛异常自动回滚
}

// ② 声明式事务（推荐，注解）
@Transactional
public void transfer() {
    accountDao.decrease(1, 100);
    accountDao.increase(2, 100);
}
// 一行注解搞定，AOP 自动管理
```
声明式事务的好处
```
// ① 简单（一行注解）
// ② 非侵入（业务代码不含事务逻辑）
// ③ 基于 AOP 自动管理
// 推荐：声明式
```
### 实现原理
@Transactional的底层
```
// @Transactional → 事务拦截器 TransactionInterceptor
// 是一个 AOP 环绕通知

// 执行流程：
// ① 方法进入拦截器
// ② 开启事务（根据传播行为）
// ③ 执行业务方法
// ④ 正常返回 → 提交事务
// ⑤ 抛异常 → 判断是否回滚 → 回滚/提交
```
TransactionInterceptor的源码
```
// TransactionInterceptor.invoke
public Object invoke(MethodInvocation invocation) throws Throwable {
    // ① 获取事务属性（@Transactional 的配置）
    TransactionAttribute txAttr = ...;

    // ② 获取事务管理器
    PlatformTransactionManager tm = ...;

    // ③ 核心：执行事务
    return invokeWithinTransaction(invocation, txAttr, tm);
}

// invokeWithinTransaction
protected Object invokeWithinTransaction(...) {
    // ① 开启事务（根据传播行为）
    TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);

    // ② 执行目标方法
    Object retVal = invocation.proceedWithInvocation();

    // ③ 正常返回 → 提交事务
    commitTransactionAfterReturning(txInfo);
    return retVal;

    // ④ 异常 → 回滚
    catch (Throwable ex) {
        completeTransactionAfterThrowing(txInfo, ex);  // 判断是否回滚
        throw ex;
    }
}
```
回滚逻辑
```
// completeTransactionAfterThrowing：
// 判断这个异常是否需要回滚
protected void completeTransactionAfterThrowing(TransactionInfo txInfo, Throwable ex) {
    // 默认：RuntimeException 和 Error 回滚
    if (txInfo != null && txInfo.transactionAttribute != null) {
        if (txInfo.transactionAttribute.rollbackOn(ex)) {
            // 需要回滚
            txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
        } else {
            // 不需要回滚（受检异常默认不回滚！）→ 提交
            txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
        }
    }
}

// rollbackOn 的判断规则：
// ① RuntimeException → 回滚 ✅
// ② Error → 回滚 ✅
// ③ 受检异常（Exception）→ 默认不回滚！⚠️
//    除非 rollbackFor 指定
```
### 面试高频

> **Q:** "@Transactional 什么时候回滚？" 
> **A:** "默认：抛 RuntimeException 或 Error 时回滚；受检异常（Exception 的子类但非 RuntimeException）默认不回滚（会提交）。要回滚受检异常必须指定 rollbackFor = Exception.class。"
### 传播行为
```
// 传播行为：一个事务方法调用另一个事务方法时，事务怎么处理
// @Transactional(propagation = Propagation.XXX)
```

| 传播行为              | 说明              | 场景      |
| ----------------- | --------------- | ------- |
| **REQUIRED**（默认）  | 有事务加入，没有则新建     | ✅ 最常用   |
| **REQUIRES_NEW**  | 无论如何新建独立事务      | 日志、独立操作 |
| **SUPPORTS**      | 有事务加入，没有则以非事务执行 | 查询      |
| **NOT_SUPPORTED** | 挂起当前事务，以非事务执行   | 大查询     |
| **MANDATORY**     | 必须有事务，否则报错      | 强依赖事务   |
| **NEVER**         | 必须没有事务，否则报错     | 反操作     |
| **NESTED**        | 嵌套事务（保存点，可部分回滚） | 复杂场景    |
REQUIRED VS REQUIRED_NEW
```
// REQUIRED（默认）：加入外部事务
public void outer() {
    inner();  // 加入 outer 的事务
    // 若 inner 抛异常 → 整个事务回滚（outer 也回滚）
}

// REQUIRES_NEW：独立新事务
public void outer() {
    inner();  // 新建独立事务（挂起 outer 的事务）
    // inner 抛异常 → 只回滚 inner
    // outer 的事务不受影响（可以继续）
}

// 例子：
@Transactional
public void createOrder() {
    orderDao.insert();
    try {
        logService.saveLog();  // REQUIRES_NEW：日志独立事务
    } catch (Exception e) {
        // 日志失败不影响订单！
    }
}
// 如果 saveLog 是 REQUIRED：日志失败 → 整个订单也回滚 ❌
// 如果 saveLog 是 REQUIRES_NEW：日志失败只回滚日志 ✅
```
### 隔离级别
和数据库的隔离级别对应
```
// @Transactional(isolation = Isolation.XXX)

// Spring 的隔离级别（对应 MySQL）：
// DEFAULT          → 用数据库默认（MySQL = REPEATABLE READ）
// READ_UNCOMMITTED → 读未提交（脏读）
// READ_COMMITTED   → 读已提交（解决脏读）
// REPEATABLE_READ  → 可重复读（解决不可重复读）
// SERIALIZABLE     → 串行化（解决幻读，最安全最慢）
```
注意点
```
// ① DEFAULT 最常用（用数据库默认，MySQL 是 RR）
// ② 隔离级别越高，并发越低
// ③ 事务里读的隔离由数据库级别决定
// ④ Spring 只是把配置传给数据库，会判断与数据库的隔离级别是否一致，不一致要设置当前事务的隔离级别
```
### 事务失效的场景
自调用
```
@Service
public class OrderService {

    @Transactional
    public void createOrder() {
        // 事务生效（外部调用）
    }

    public void process() {
        this.createOrder();  // ❌ 自调用！事务失效！
        // this 是原始对象，不是代理对象
        // AOP 拦截不到
    }
}

// 解决：
// ① 注入自己（代理对象）
// ② AopContext.currentProxy()
// ③ 拆到另一个 Service
```
方法非public
```
@Transactional
private void createOrder() {}  // ❌ private → 事务不生效！
// AOP 代理无法拦截 private 方法（CGLIB 也不能覆写 private）

// 解决：必须是 public
```
异常被catch吞掉
```
@Transactional
public void createOrder() {
    try {
        orderDao.insert();
        // 发生异常
    } catch (Exception e) {
        // ❌ 异常被捕获，没抛出
        // 事务不知道有异常 → 正常提交！不回滚！  回滚判定只发生在@Transactional方法返回调用方的那一刻
    }
}

// 解决：catch 后重新抛出 / 手动回滚
// TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()
```
抛的是受检异常
```
@Transactional
public void createOrder() throws Exception {
    throw new Exception("业务异常");  // ❌ 受检异常，默认不回滚！
}

// 解决：@Transactional(rollbackFor = Exception.class)
```
非事务方法调用
```
// 通过 new 出来的对象调用 → 不是 Spring 管理的 Bean → 无代理
```
数据库不支持事务
```
// MyISAM 引擎不支持事务 → @Transactional 无效
// 确保用 InnoDB
```
事务方法内部调用异步
```
@Transactional
public void createOrder() {
    asyncService.sendLog();  // 异步线程 → 新线程没有事务上下文！
    // 异步方法的事务独立（或不生效）
}
```

|场景|原因|解决|
|---|---|---|
|自调用|this 不是代理|注入代理|
|private 方法|无法代理|public|
|catch 吞异常|事务不知道|重抛/setRollbackOnly|
|受检异常|默认不回滚|rollbackFor|
|非 Spring Bean|无代理|注入 Bean|
|引擎不支持|MyISAM|InnoDB|
### 面试高频题
#### 题目1：Spring事务的实现原理
```
// AOP + TransactionInterceptor（环绕通知）
// 开启事务 → 执行方法 → 提交/回滚
```
#### 题目2：默认什么时候回滚
```
// RuntimeException 和 Error
// 受检异常默认不回滚（rollbackFor 指定）
```
#### 题目3：传播行为有哪些
```
// 7 种，重点 REQUIRED 和 REQUIRES_NEW
```
#### 题目4：REQUIRED_NEW有什么用
```
// 独立事务：失败不影响外层
// 日志、审计、独立操作
```
#### 题目5：事务失效的场景
```
// 自调用、private、catch 吞异常、受检异常
```
#### 题目6：事务隔离级别
```
// 对应数据库隔离级别，DEFAULT 用数据库默认
```