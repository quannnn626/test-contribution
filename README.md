# open-pay 支付模块开发技术文档

## 项目简介

open-pay 是开放服务平台的支付核心模块，为商户（如 vben 商城）提供统一的支付、账户、资金服务能力。

**技术栈**：Spring Boot 3.0.2 / MyBatis-Plus 3.5.5 / MySQL 8.0 / Redis / RabbitMQ / Java 17

---

## 第一步：环境准备

| 组件 | 版本 | 端口 | 用途 |
|------|------|------|------|
| JDK | 17+ | - | 运行环境 |
| MySQL | 8.0+ | 3307 | 数据持久化 |
| Redis | 7.x | 6379 | 分布式锁、回调幂等 |
| RabbitMQ | 3.x | 5672 | 支付成功消息通知 |

确保以上服务均已启动。

---

## 第二步：创建项目结构

```
open-platform/
├── pom.xml                  ← 父 POM，版本统一管控
├── open-common/             ← 公共模块（统一返回、异常、工具类）
├── open-pay/                ← 支付核心模块（当前开发重点）
└── sql/
    └── trade-service.sql    ← 完整建表脚本
```

**父 POM 关键配置**：

```xml
<groupId>com.boot</groupId>
<artifactId>open-platform</artifactId>
<version>1.0.0</version>
<packaging>pom</packaging>

<modules>
    <module>open-common</module>
    <module>open-pay</module>
</modules>

<properties>
    <java.version>17</java.version>
    <spring-boot.version>3.0.2</spring-boot.version>
    <mybatis-plus.version>3.5.5</mybatis-plus.version>
    <hutool.version>5.8.23</hutool.version>
</properties>
```

---

## 第三步：创建数据库

### 3.1 创建库

```sql
CREATE DATABASE IF NOT EXISTS `open-pay`
  DEFAULT CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci;
```

### 3.2 执行建表脚本

运行 [sql/trade-service.sql](sql/trade-service.sql)，一次性创建所有表并写入初始数据。

### 3.3 表清单

| 序号 | 表名 | 说明 | 所属模块 |
|------|------|------|----------|
| 1 | `pay_user` | 平台用户表 | open-pay |
| 2 | `pay_user_account` | 用户钱包账户表 | open-pay |
| 3 | `pay_merchant` | 商户信息表 | open-pay |
| 4 | `pay_merchant_account` | 商户资金账户表 | open-pay |
| 5 | `pay_payment_order` | 支付订单表 | open-pay |
| 6 | `pay_payment_channel` | 支付渠道表 | open-pay |
| 7 | `pay_account_flow` | 账户资金流水表 | open-pay |
| 8 | `pay_payment_notify` | 支付回调通知记录表 | open-pay |
| 9 | `pay_recharge_order` | 充值订单表（预留） | open-pay |
| 10 | `pay_refund_order` | 退款订单表（预留） | open-pay |
| 11 | `pay_api_log` | 接口调用日志表 | open-pay |

### 3.4 ER 关系

```
pay_user ──1:1── pay_user_account (user_id)
     │
     └── 1:N ── pay_payment_order (user_id)
                    │
pay_merchant ──1:1── pay_merchant_account (merchant_id)
     │              │
     └── 1:N ──────┤── pay_payment_order (merchant_id)
                    │
                    ├── pay_payment_notify (payment_no)
                    │
                    └── pay_account_flow (payment_no)
                              │
                              ├── account_type=1 → pay_user_account
                              └── account_type=2 → pay_merchant_account

pay_payment_channel ←── channel_id ── pay_payment_order
```

### 3.5 初始数据

支付渠道表 `pay_payment_channel` 需要预置 3 条数据：

| id | channel_code | channel_name |
|----|-------------|-------------|
| 1 | BALANCE | 余额支付 |
| 2 | ALIPAY | 支付宝 |
| 3 | WECHAT | 微信支付 |

---

## 第四步：配置数据源

在 `open-pay/src/main/resources/application.yml` 中：

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3307/open-pay?useUnicode=true&characterEncoding=utf8mb4&serverTimezone=Asia/Shanghai
    username: root
    password: ${DB_PASSWORD}
    driver-class-name: com.mysql.cj.jdbc.Driver
  redis:
    host: localhost
    port: 6379
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest

mybatis-plus:
  configuration:
    map-underscore-to-camel-case: true
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
  global-config:
    db-config:
      id-type: assign_id   # Snowflake 策略
```

---

## 第五步：开发公共模块（open-common）

### 5.1 统一返回体

```java
@Data
public class R<T> {
    private int code;
    private String msg;
    private T data;

    public static <T> R<T> ok(T data) {
        R<T> r = new R<>();
        r.code = 200;
        r.msg = "success";
        r.data = data;
        return r;
    }

    public static <T> R<T> fail(String msg) {
        R<T> r = new R<>();
        r.code = 500;
        r.msg = msg;
        return r;
    }
}
```

### 5.2 全局异常处理

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public R<?> handleBusiness(BusinessException e) {
        return R.fail(e.getMessage());
    }

    @ExceptionHandler(Exception.class)
    public R<?> handleException(Exception e) {
        log.error("系统异常", e);
        return R.fail("系统繁忙，请稍后再试");
    }
}
```

### 5.3 工具类

- `IdUtil`（Hutool Snowflake）—— 生成全局唯一 ID
- `RedisUtil` —— Redis 操作封装
- `SignUtil` —— 商户签名验证（MD5/SHA256）

---

## 第六步：开发实体层（Domain）

在 `open-pay` 模块下创建实体类，每个实体对应一张数据库表，使用 MyBatis-Plus 注解：

```java
// 示例：支付订单实体
@Data
@TableName("pay_payment_order")
public class TradePaymentOrder {
    @TableId
    private Long id;
    private String paymentNo;
    private Long merchantId;
    private String orderNo;
    private Long userId;
    private BigDecimal amount;
    private Integer status;
    private LocalDateTime expireTime;
    private LocalDateTime payTime;
    private LocalDateTime createTime;
    private LocalDateTime updateTime;
    private String merchantPaymentNo;
    private Long channelId;
}
```

**11 个实体与表对应关系**：

| 实体类 | @TableName |
|--------|-----------|
| `TradeUser` | `pay_user` |
| `TradeUserAccount` | `pay_user_account` |
| `TradeMerchant` | `pay_merchant` |
| `TradeMerchantAccount` | `pay_merchant_account` |
| `TradePaymentOrder` | `pay_payment_order` |
| `TradePaymentChannel` | `pay_payment_channel` |
| `TradeAccountFlow` | `pay_account_flow` |
| `TradePaymentNotify` | `pay_payment_notify` |
| `TradeRechargeOrder` | `pay_recharge_order` |
| `TradeRefundOrder` | `pay_refund_order` |
| `TradeApiLog` | `pay_api_log` |

> 实体类名保留 `Trade` 前缀（Java 命名惯例），`@TableName` 注解映射到新的 `pay_` 表名。

---

## 第七步：开发 Mapper 层

继承 MyBatis-Plus 的 `BaseMapper<T>`，即可获得 CRUD 能力：

```java
@Mapper
public interface TradePaymentOrderMapper extends BaseMapper<TradePaymentOrder> {
    // 复杂查询可在此扩展
}
```

依次创建 11 张表对应的 Mapper 接口。

---

## 第八步：开发 Service 层（核心业务）

这是支付模块的核心，按业务流程分步实现。

### 8.1 商户校验

```java
// 验证 app_key + 签名，确认商户身份
TradeMerchant merchant = merchantService.getByAppKey(appKey);
if (!SignUtil.verify(params, merchant.getAppSecret())) {
    throw new BusinessException("签名验证失败");
}
```

### 8.2 创建支付订单

```
POST /pay/create

请求参数：
  - appKey      商户应用Key
  - sign        签名
  - orderNo     商城订单号
  - amount      支付金额
  - userId      用户ID
  - channelId   支付渠道 (1=余额)
  - notifyUrl   回调地址

处理逻辑：
  ① 校验商户签名
  ② 校验金额 > 0
  ③ 使用 Snowflake 生成 paymentNo
  ④ 写入 pay_payment_order（status=WAIT_PAY）
  ⑤ 返回 paymentNo
```

```java
@Transactional
public String createPayment(CreatePaymentRequest req) {
    // 1. 校验商户
    TradeMerchant merchant = validateMerchant(req.getAppKey(), req.getSign());

    // 2. 幂等检查：同商户+同商城订单号去重
    TradePaymentOrder exist = paymentOrderMapper.selectOne(
        new LambdaQueryWrapper<TradePaymentOrder>()
            .eq(TradePaymentOrder::getMerchantId, merchant.getId())
            .eq(TradePaymentOrder::getMerchantPaymentNo, req.getOrderNo())
    );
    if (exist != null) return exist.getPaymentNo();

    // 3. 创建订单
    TradePaymentOrder order = new TradePaymentOrder();
    order.setId(IdUtil.getSnowflakeNextId());
    order.setPaymentNo(generatePaymentNo());
    order.setMerchantId(merchant.getId());
    order.setOrderNo(req.getOrderNo());
    order.setMerchantPaymentNo(req.getOrderNo());
    order.setUserId(req.getUserId());
    order.setAmount(req.getAmount());
    order.setChannelId(req.getChannelId());
    order.setStatus(PaymentStatus.WAIT_PAY.getCode());
    order.setExpireTime(LocalDateTime.now().plusMinutes(30));
    paymentOrderMapper.insert(order);

    return order.getPaymentNo();
}
```

### 8.3 执行支付（核心流程）

```
POST /pay/execute

请求参数：
  - paymentNo   支付流水号
  - userId      用户ID

处理流程：
  
  ① Redisson 分布式锁: pay:lock:{paymentNo}
  ② 查询支付订单，校验状态=WAIT_PAY
  ③ 查询用户账户余额
  ④ 校验余额 >= 订单金额
  ⑤ 扣减用户余额（version 乐观锁）
  ⑥ 增加商户余额（version 乐观锁）
  ⑦ 写入 pay_account_flow × 2 条（用户支出 + 商户收入）
  ⑧ 更新支付订单 status=SUCCESS, pay_time=now
  ⑨ 发送 MQ: pay.success
  ⑩ POST 回调商户 notifyUrl
      ├── 成功 → 结束
      └── 失败 → 写入 pay_payment_notify，定时任务重试
```

```java
@Transactional
public void executePayment(String paymentNo, Long userId) {
    // 1. 分布式锁
    RLock lock = redissonClient.getLock("pay:lock:" + paymentNo);
    try {
        if (!lock.tryLock(10, 30, TimeUnit.SECONDS)) {
            throw new BusinessException("请勿重复支付");
        }

        // 2. 查询订单
        TradePaymentOrder order = paymentOrderMapper.selectOne(
            new LambdaQueryWrapper<TradePaymentOrder>()
                .eq(TradePaymentOrder::getPaymentNo, paymentNo)
        );
        if (order == null) throw new BusinessException("订单不存在");
        if (order.getStatus() != PaymentStatus.WAIT_PAY.getCode()) {
            throw new BusinessException("订单状态异常");
        }

        // 3. 查用户账户
        TradeUserAccount userAccount = userAccountMapper.selectOne(
            new LambdaQueryWrapper<TradeUserAccount>()
                .eq(TradeUserAccount::getUserId, userId)
        );
        if (userAccount.getBalance().compareTo(order.getAmount()) < 0) {
            throw new BusinessException("余额不足");
        }

        // 4. 查商户账户
        TradeMerchantAccount merchantAccount = merchantAccountMapper.selectOne(
            new LambdaQueryWrapper<TradeMerchantAccount>()
                .eq(TradeMerchantAccount::getMerchantId, order.getMerchantId())
        );

        // 5. 扣减用户余额（乐观锁）
        BigDecimal beforeUser = userAccount.getBalance();
        BigDecimal afterUser = beforeUser.subtract(order.getAmount());
        int rows = userAccountMapper.update(null,
            new LambdaUpdateWrapper<TradeUserAccount>()
                .eq(TradeUserAccount::getUserId, userId)
                .eq(TradeUserAccount::getVersion, userAccount.getVersion())
                .set(TradeUserAccount::getBalance, afterUser)
                .set(TradeUserAccount::getVersion, userAccount.getVersion() + 1)
        );
        if (rows == 0) throw new BusinessException("账户并发冲突，请重试");

        // 6. 增加商户余额（乐观锁）
        BigDecimal beforeMerchant = merchantAccount.getBalance();
        BigDecimal afterMerchant = beforeMerchant.add(order.getAmount());
        rows = merchantAccountMapper.update(null,
            new LambdaUpdateWrapper<TradeMerchantAccount>()
                .eq(TradeMerchantAccount::getMerchantId, order.getMerchantId())
                .eq(TradeMerchantAccount::getVersion, merchantAccount.getVersion())
                .set(TradeMerchantAccount::getBalance, afterMerchant)
                .set(TradeMerchantAccount::getVersion, merchantAccount.getVersion() + 1)
        );
        if (rows == 0) throw new BusinessException("商户账户并发冲突，请重试");

        // 7. 写入资金流水
        saveAccountFlow(paymentNo, 1, userAccount.getId(), -order.getAmount(),
            beforeUser, afterUser, "用户支付");
        saveAccountFlow(paymentNo, 2, merchantAccount.getId(), order.getAmount(),
            beforeMerchant, afterMerchant, "商户收款");

        // 8. 更新订单状态
        order.setStatus(PaymentStatus.SUCCESS.getCode());
        order.setPayTime(LocalDateTime.now());
        paymentOrderMapper.updateById(order);

        // 9. 发送 MQ
        rabbitTemplate.convertAndSend("pay.success", paymentNo);

        // 10. 回调商户（异步）
        notifyMerchant(order);

    } finally {
        if (lock.isHeldByCurrentThread()) {
            lock.unlock();
        }
    }
}
```

### 8.4 查询支付状态

```
GET /pay/query/{paymentNo}

返回：{ paymentNo, tradeNo, tradeStatus, amount }
```

```java
public PaymentResult queryPayment(String paymentNo) {
    TradePaymentOrder order = paymentOrderMapper.selectOne(
        new LambdaQueryWrapper<TradePaymentOrder>()
            .eq(TradePaymentOrder::getPaymentNo, paymentNo)
    );
    if (order == null) throw new BusinessException("订单不存在");

    PaymentResult result = new PaymentResult();
    result.setPaymentNo(order.getPaymentNo());
    result.setTradeNo(order.getOrderNo());
    result.setTradeStatus(order.getStatus());
    result.setAmount(order.getAmount());
    return result;
}
```

### 8.5 支付状态枚举

| 状态码 | 枚举值 | 说明 |
|--------|--------|------|
| 0 | WAIT_PAY | 待支付 |
| 1 | PAYING | 支付中 |
| 2 | SUCCESS | 支付成功 |
| 3 | FAILED | 支付失败 |
| 4 | CLOSED | 已关闭 |

---

## 第九步：开发 Controller 层

```java
@RestController
@RequestMapping("/pay")
public class PayController {

    @Autowired
    private PayService payService;

    @PostMapping("/create")
    public R<String> create(@RequestBody CreatePaymentRequest req) {
        String paymentNo = payService.createPayment(req);
        return R.ok(paymentNo);
    }

    @PostMapping("/execute")
    public R<Void> execute(@RequestBody ExecutePaymentRequest req) {
        payService.executePayment(req.getPaymentNo(), req.getUserId());
        return R.ok(null);
    }

    @GetMapping("/query/{paymentNo}")
    public R<PaymentResult> query(@PathVariable String paymentNo) {
        return R.ok(payService.queryPayment(paymentNo));
    }
}
```

---

## 第十步：接入 Redis

### 10.1 分布式锁（防重复支付）

```
Key:  pay:lock:{paymentNo}
TTL: 30s
工具: Redisson
```

每次执行支付前获取锁，同一笔 paymentNo 只能有一个线程执行扣款。

### 10.2 回调幂等

```
Key:  pay:callback:{paymentNo}
TTL: 24h
```

回调成功后写入 Redis，重试时先检查，避免重复通知。

---

## 第十一步：接入 RabbitMQ

### 11.1 发送方（open-pay）

支付成功后发送：

```
Exchange:  pay.exchange
Queue:     pay.success.queue
RoutingKey:pay.success
消息体:    { "paymentNo": "xxx", "orderNo": "xxx", "status": 2 }
```

### 11.2 消费方（商城 vben-admin）

商城监听 `pay.success.queue`，收到消息后更新订单状态。

---

## 第十二步：开发回调通知

### 12.1 同步回调

支付完成后，立即 POST 请求 `pay_merchant.notify_url`：

```json
{
  "paymentNo": "PAY20260803...",
  "tradeNo": "商城订单号",
  "tradeStatus": 2,
  "amount": 99.00
}
```

### 12.2 重试机制

回调失败时写入 `pay_payment_notify`：

```java
TradePaymentNotify notify = new TradePaymentNotify();
notify.setId(IdUtil.getSnowflakeNextId());
notify.setPaymentNo(paymentNo);
notify.setMerchantId(merchantId);
notify.setNotifyUrl(merchant.getNotifyUrl());
notify.setNotifyStatus(0);
notify.setRetryCount(0);
notify.setNextRetryTime(LocalDateTime.now().plusMinutes(1));
paymentNotifyMapper.insert(notify);
```

### 12.3 定时重试任务

```java
@Scheduled(fixedDelay = 60_000)  // 每分钟执行
public void retryNotify() {
    List<TradePaymentNotify> list = paymentNotifyMapper.selectList(
        new LambdaQueryWrapper<TradePaymentNotify>()
            .eq(TradePaymentNotify::getNotifyStatus, 0)
            .le(TradePaymentNotify::getNextRetryTime, LocalDateTime.now())
    );
    for (TradePaymentNotify notify : list) {
        boolean success = doNotify(notify);
        if (success) {
            notify.setNotifyStatus(1);
        } else {
            notify.setRetryCount(notify.getRetryCount() + 1);
            notify.setNextRetryTime(calcNextRetry(notify.getRetryCount()));
        }
        paymentNotifyMapper.updateById(notify);
    }
}
```

重试策略：1min → 5min → 15min → 30min → 1h，最多 10 次。

---

## 第十三步：开发后台管理

基于 Vben Admin 开发管理页面：

| 页面 | 功能 | 主要字段 |
|------|------|---------|
| 支付订单管理 | 列表、搜索、详情 | paymentNo, orderNo, 用户, 金额, 状态, 支付时间 |
| 用户账户管理 | 列表、启停 | userNo, 账户编号, 当前余额, 状态 |
| 商户管理 | 列表、新增 | merchantNo, 商户名, appKey, 余额, 状态 |
| 资金流水管理 | 列表、筛选 | flowNo, 类型, 金额, 关联单号, 时间 |
| 接口日志 | 列表 | apiName, 耗时, 请求参数, 响应结果 |

---

## 第十四步：测试验证

### 14.1 单元测试

```java
@SpringBootTest
class PayServiceTest {

    @Autowired
    private PayService payService;

    @Test
    void testCreateAndExecute() {
        // 1. 创建支付订单
        CreatePaymentRequest req = new CreatePaymentRequest();
        req.setAppKey("test_app_key");
        req.setOrderNo("ORDER20260803001");
        req.setAmount(new BigDecimal("99.00"));
        req.setUserId(1L);
        req.setChannelId(1L);
        String paymentNo = payService.createPayment(req);

        // 2. 执行支付
        payService.executePayment(paymentNo, 1L);

        // 3. 查询状态
        PaymentResult result = payService.queryPayment(paymentNo);
        assertEquals(PaymentStatus.SUCCESS.getCode(), result.getTradeStatus());
    }
}
```

### 14.2 接口测试

```bash
# 创建支付
curl -X POST http://localhost:8080/pay/create \
  -H "Content-Type: application/json" \
  -d '{"appKey":"xxx","sign":"xxx","orderNo":"O001","amount":99.00,"userId":1,"channelId":1}'

# 执行支付
curl -X POST http://localhost:8080/pay/execute \
  -H "Content-Type: application/json" \
  -d '{"paymentNo":"PAY20260803...","userId":1}'

# 查询状态
curl http://localhost:8080/pay/query/PAY20260803...
```

### 14.3 验证清单

- [ ] 创建订单成功，status=WAIT_PAY
- [ ] 余额不足时支付失败，不扣款
- [ ] 重复执行同一 paymentNo 被分布式锁拦截
- [ ] 支付成功后用户余额减少、商户余额增加
- [ ] 资金流水表写入两条记录，金额正确
- [ ] 回调成功时商城收到通知
- [ ] 回调失败时写入重试表，定时任务重试成功

---

## 关键技术决策

| 决策点 | 方案 | 说明 |
|--------|------|------|
| ID 策略 | Snowflake（Hutool IdUtil） | 分布式唯一，非数据库自增 |
| 分布式锁 | Redisson | 防重复支付 |
| 乐观锁 | account 表 version 字段 | 防并发余额覆盖 |
| 订单幂等 | uk_merchant_payment 唯一索引 | 同商城同订单号去重 |
| 回调幂等 | Redis + DB 唯一约束 | 避免重复通知 |
| 用户独立 | 平台维护自己的 pay_user | 不与商城用户耦合 |

---

## 外部对接

| 对接方 | 地址 | 说明 |
|--------|------|------|
| 商城后端 | `localhost:9000/api` | 通过 API 调用平台服务 |
| 商城回调 | `POST /mall/payment/paymentCallback` | 支付成功通知（可无认证） |

回调参数格式：

```json
{
  "paymentNo": "平台支付单号",
  "tradeNo": "商城订单号",
  "tradeStatus": 2,
  "amount": 99.00
}
```
