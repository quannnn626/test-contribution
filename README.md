# Open Platform 支付模块详细开发计划

> 本文档基于当前项目状态（已完成登录/注册/退出，已删除 `pay_user` 表，统一使用 `auth_user`），按照真实支付平台的开发节奏，逐阶段详细拆解开发任务。

---

## 当前项目基线

### 已完成

| 模块 | 内容 | 状态 |
|------|------|------|
| 认证模块 | 登录 / 注册 / 退出 / JWT Token 刷新 | ✅ 完成 |
| 用户体系 | 删除 `pay_user`，统一为 `auth_user` | ✅ 完成 |
| 基础框架 | open-common（Result、BusinessException、GlobalExceptionHandler） | ✅ 完成 |
| 实体/Mapper/Service | 全部 10 张表的空壳已生成 | ✅ 完成 |
| 支付渠道 | `pay_payment_channel` 已预置 BALANCE / ALIPAY / WECHAT 三条数据 | ✅ 完成 |

### 当前数据库（10 张表）

```
auth_user              ← 认证用户（已完成）
pay_user_account       ← 用户钱包账户
pay_merchant           ← 商户信息
pay_merchant_account   ← 商户资金账户
pay_payment_order      ← 支付订单
pay_payment_channel    ← 支付渠道（已预置数据）
pay_account_flow       ← 资金流水
pay_payment_notify     ← 支付回调通知
pay_recharge_order     ← 充值订单（预留）
pay_refund_order       ← 退款订单（预留）
pay_api_log            ← 接口调用日志
```

---

## 总体阶段划分

```
阶段一   ─ 商户管理模块         （3-4 天）
阶段二   ─ 开放 API 签名认证     （2-3 天）
阶段三   ─ 支付订单创建          （2-3 天）
阶段四   ─ 用户账户与余额        （3-4 天）
阶段五   ─ 支付执行引擎          （4-5 天）
阶段六   ─ 资金流水系统          （2-3 天）
阶段七   ─ 支付回调通知          （3-4 天）
阶段八   ─ 退款模块             （3-4 天）
阶段九   ─ 充值模块             （2-3 天）
阶段十   ─ 运营后台管理页面      （5-7 天）
阶段十一 ─ Redis 缓存与分布式锁  （2-3 天）
阶段十二 ─ RabbitMQ 异步消息     （2-3 天）
阶段十三 ─ 定时任务与对账        （2-3 天）
```

---

# 阶段一：商户管理模块

> **目标**：实现第三方商户的入驻、凭证管理、状态管控，让外部系统能够接入平台。

---

## 1.1 扩展 `pay_merchant` 表

### 为什么要扩展？

当前 `pay_merchant` 表只有最基础的字段（名称、appKey、appSecret、状态），无法满足企业级商户管理的诉求。真实支付平台（如支付宝、微信支付）的商户管理非常复杂，需要记录联系人、企业资质、密钥版本、授权有效期等信息。

### 扩展 SQL

```sql
-- 扩展 pay_merchant 表
ALTER TABLE pay_merchant
    ADD COLUMN contact_name    VARCHAR(64)   COMMENT '联系人姓名',
    ADD COLUMN contact_phone   VARCHAR(20)   COMMENT '联系人电话',
    ADD COLUMN contact_email   VARCHAR(128)  COMMENT '联系人邮箱',
    ADD COLUMN company_name    VARCHAR(128)  COMMENT '企业全称',
    ADD COLUMN business_license VARCHAR(64)  COMMENT '营业执照号',
    ADD COLUMN settle_type     INT DEFAULT 1 COMMENT '结算方式 1-T+1 2-T+0 3-周结 4-月结',
    ADD COLUMN settle_fee_rate DECIMAL(5,4) DEFAULT 0.0060 COMMENT '结算费率（如0.0060=0.6%）',
    ADD COLUMN expire_time     DATETIME      COMMENT '授权过期时间',
    ADD COLUMN secret_version  INT DEFAULT 1 COMMENT '密钥版本号',
    ADD COLUMN white_ip_list   TEXT          COMMENT 'IP白名单（JSON数组）',
    ADD COLUMN daily_limit     DECIMAL(12,2) COMMENT '单日交易限额',
    ADD COLUMN single_limit    DECIMAL(12,2) COMMENT '单笔交易限额',
    ADD COLUMN remark          VARCHAR(255)  COMMENT '备注',
    ADD COLUMN audit_status    INT DEFAULT 0 COMMENT '审核状态 0-待审核 1-已通过 2-已驳回',
    ADD COLUMN audit_remark    VARCHAR(255)  COMMENT '审核备注';
```

### 新增字段说明

| 字段 | 用途 | 为什么需要 |
|------|------|-----------|
| `contact_name/phone/email` | 商户联系人 | 出问题时能联系到人，企业级基本要求 |
| `company_name` | 企业全称 | 商户简称≠企业全称，财务对账需要 |
| `business_license` | 营业执照号 | 合规要求，商户实名认证 |
| `settle_type` | 结算方式 | 不同商户不同结算周期，T+1 和 T+0 的费率不同 |
| `settle_fee_rate` | 费率 | 平台盈利来源，每笔交易按费率抽成 |
| `expire_time` | 授权有效期 | 安全管控，过期自动拒绝请求 |
| `secret_version` | 密钥版本 | 支持密钥轮换（secret 泄露后可升级版本使旧密钥失效） |
| `white_ip_list` | IP 白名单 | 防止 appSecret 泄露后被其他 IP 恶意调用 |
| `daily_limit` | 单日限额 | 风控需求，防止商户被刷单 |
| `single_limit` | 单笔限额 | 风控需求，大额交易需要额外审核 |
| `audit_status` | 审核状态 | 商户入驻需要审核，不是谁都能接入 |

---

## 1.2 实现商户入驻功能

### 接口

```
POST /api/merchant/apply
```

### 流程

```
管理员提交商户入驻申请
    │
    ├── 企业信息（companyName, businessLicense）
    ├── 联系信息（contactName, contactPhone, contactEmail）
    ├── 结算配置（settleType, settleFeeRate）
    └── 风控配置（dailyLimit, singleLimit, whiteIpList）
        │
        ▼
    系统校验：
    ① 企业名称、营业执照号不能为空
    ② 联系人信息不能为空
    ③ 费率范围校验（0.1% ~ 5%）
    ④ 限额必须 > 0
        │
        ▼
    系统生成：
    ① merchant_no = "M" + 日期 + Snowflake（如 M20260806001）
    ② app_key = MD5(merchant_no + timestamp) 取前 16 位
    ③ app_secret = UUID 去横线（32 位随机字符串）
        │
        ▼
    写入 pay_merchant
    audit_status = 0（待审核）
        │
        ▼
    返回：
    {
      merchantNo,
      merchantName,
      appKey,
      appSecret,     ← 仅此一次明文展示，提示商户妥善保管
      auditStatus
    }
```

### 为什么要分开申请和审核？

- 商户入驻涉及资金结算，必须有审核环节
- 防止恶意注册、套现等风险
- appSecret 在审核通过后才正式生效
- 审核不通过时，appKey 和 appSecret 不会被激活

### 需要实现的类

| 层 | 类 | 位置 |
|------|------|------|
| DTO | `MerchantApplyDTO` | `dto/` |
| VO | `MerchantApplyVO` | `vo/` |
| Service | `PayMerchantService.apply()` | `service/impl/PayMerchantServiceImpl.java` |
| Controller | `MerchantController` | `controller/` |

---

## 1.3 实现商户审核功能

### 接口

```
POST /api/merchant/audit
```

### 流程

```
管理员审核
    │
    ├── 通过：audit_status = 1，商户状态 status = 1（启用）
    │
    └── 驳回：audit_status = 2，填写 audit_remark 说明原因
```

### 审核通过后触发

```
① 商户 status 变为 1（启用）
② 创建商户资金账户 → pay_merchant_account
   - account_no = "MA" + 日期 + Snowflake
   - merchant_id = 商户 ID
   - balance = 0.00
   - frozen_amount = 0.00
```

---

## 1.4 实现商户状态管理

### 接口

```
PUT  /api/merchant/{merchantNo}/enable     ← 启用商户
PUT  /api/merchant/{merchantNo}/disable    ← 禁用商户
GET  /api/merchant/{merchantNo}            ← 查询商户详情
GET  /api/merchant/list                    ← 分页查询商户列表
```

### 启用/禁用的影响范围

```
商户被禁用（status=0）后：
  → 所有 API 请求被拦截（签名认证阶段直接拒绝）
  → 该商户的 appKey 暂时失效
  → 不影响已完成的订单（只限制新请求）
  → 管理员可以重新启用
```

### 密钥轮换

```
PUT  /api/merchant/{merchantNo}/rotate-secret

流程：
① 生成新的 app_secret
② secret_version + 1
③ 旧 secret 保留 24 小时过渡期（存在 merchant_secret_history 表）
④ 24 小时后旧 secret 彻底失效
```

### 需要新增的表

```sql
-- 商户密钥历史表（支持密钥轮换的过渡期）
CREATE TABLE pay_merchant_secret_history (
    id            BIGINT NOT NULL COMMENT '主键',
    merchant_id   BIGINT NOT NULL COMMENT '商户ID',
    secret        VARCHAR(128) NOT NULL COMMENT '历史密钥',
    version       INT NOT NULL COMMENT '密钥版本号',
    expire_time   DATETIME NOT NULL COMMENT '失效时间',
    create_time   DATETIME DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    INDEX idx_merchant_id (merchant_id),
    INDEX idx_expire_time (expire_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='商户密钥历史表';
```

---

## 1.5 商户管理 ER 更新

```
pay_merchant ──1:1── pay_merchant_account (merchant_id)
     │
     └── 1:N ── pay_merchant_secret_history (merchant_id)  ← 新增
```

---

# 阶段二：开放 API 签名认证

> **目标**：让外部商户能够安全地调用平台 API，防止请求被篡改、重放、伪造。

---

## 2.1 设计签名认证体系

### 为什么不直接用 JWT？

- JWT 面向**平台内部用户**（admin 登录后台），需要登录态
- 开放 API 面向**第三方商户系统**，没有"登录"概念
- 商户系统是服务器-to-服务器调用，用 appKey + appSecret + 签名更合适
- 参考支付宝开放平台、微信支付商户平台的签名方案

### 签名流程

```
商户系统                                     Open Platform
   │                                              │
   ├── 1. 组装请求参数                              │
   ├── 2. 按 key 字典序排序                         │
   ├── 3. 拼接成 "key1=val1&key2=val2..."           │
   ├── 4. 末尾追加 appSecret                        │
   ├── 5. MD5/SHA256 生成 sign                      │
   ├── 6. 发请求（参数 + sign）                      │
   │                                              │
   │              ──── HTTP POST ────▶            │
   │                                              ├── 7. 根据 appKey 查商户
   │                                              ├── 8. 校验商户状态
   │                                              ├── 9. 校验 IP 白名单
   │                                              ├── 10. 校验授权有效期
   │                                              ├── 11. 重新计算 sign
   │                                              ├── 12. 比对 sign
   │                                              ├── 13. 校验 timestamp（防重放）
   │                                              ├── 14. 校验 nonce（防重复提交）
   │                                              │
   │              ◀──── Response ────             │
   │                                              │
```

### 公共请求参数（每个 API 都需要带）

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `appKey` | String | 是 | 商户应用 Key |
| `timestamp` | Long | 是 | 请求时间戳（毫秒），与服务端时间差 > 5 分钟则拒绝 |
| `nonce` | String | 是 | 随机字符串（32 位），5 分钟内不可重复 |
| `signType` | String | 否 | 签名算法，默认 MD5，可选 SHA256 |
| `sign` | String | 是 | 签名值 |
| `version` | String | 否 | API 版本号，默认 "1.0" |

---

## 2.2 实现签名工具类

### `SignUtil.java`（放在 `open-common` 模块）

```
核心方法：
├── generateSign(Map<String, Object> params, String secret, String signType)
│     → 参数排序 → 拼接 → 追加 secret → MD5/SHA256 → 返回 sign
│
├── verifySign(Map<String, Object> params, String secret, String sign)
│     → 重新生成 sign → 与传入的 sign 比对 → 返回 true/false
│
└── createNonce()
      → 生成 32 位随机字符串
```

### 签名算法细节

```
假设请求参数：
{
  "appKey": "ak_abc123",
  "timestamp": 1722938400000,
  "nonce": "a1b2c3d4e5f6",
  "orderNo": "ORD20260806001",
  "amount": 99.00
}

① 剔除 sign、signType 字段
② 按 key 字母序升序排列：
   amount=99.0&appKey=ak_abc123&nonce=a1b2c3d4e5f6&orderNo=ORD20260806001&timestamp=1722938400000

③ 末尾追加 appSecret：
   amount=99.0&appKey=ak_abc123&nonce=a1b2c3d4e5f6&orderNo=ORD20260806001&timestamp=1722938400000&secret=xxx

④ MD5 得到 sign：
   sign = MD5(上述字符串).toUpperCase()
```

---

## 2.3 实现签名验证拦截器

### `OpenApiAuthInterceptor.java`

这是一个 Spring MVC 拦截器，拦截所有 `/open/**` 路径的请求。

```
拦截流程：
① 从 request 中提取 appKey, timestamp, nonce, sign
② 根据 appKey 查询商户信息（含缓存）
③ 校验商户状态
④ 校验 IP 白名单（如果商户配置了）
⑤ 校验授权有效期（expire_time）
⑥ 校验 timestamp 与服务器时间差 ≤ 5 分钟
⑦ 校验 nonce 是否已被使用（Redis: nonce:{appKey}:{nonce}，TTL 5 分钟）
⑧ 重新计算 sign，比对
⑨ 通过 → 放行；失败 → 返回 401 "签名验证失败"
```

### 为什么需要 timestamp 校验？

防止**重放攻击**。如果没有时间窗口限制，攻击者截获一个合法请求后可以反复重放。

### 为什么需要 nonce 校验？

同一时间窗口内（5 分钟），timestamp 不会过期，攻击者可能重放同一个请求。nonce 保证每个请求在 5 分钟内只能使用一次。

### 需要实现的类

| 类 | 位置 | 说明 |
|------|------|------|
| `SignUtil` | `open-common/utils/` | 签名生成 & 验证 |
| `OpenApiAuthInterceptor` | `open-pay/.../handler/` | 签名验证拦截器 |
| `WebMvcConfig` | `open-pay/.../config/` | 注册拦截器（扩展已有） |
| `NonceValidator` | `open-pay/.../handler/` | nonce 防重放 |

---

## 2.4 请求日志记录

### 为什么要记录 API 日志？

- 排查问题：商户说"我调了你的接口但没收到返回"
- 计费对账：按调用量收费时需要精确统计
- 安全审计：追踪异常调用
- 性能分析：监控接口响应时间

### 接口日志表 `pay_api_log`（已存在，补充字段）

```sql
ALTER TABLE pay_api_log
    ADD COLUMN merchant_no  VARCHAR(64)  COMMENT '商户编号（冗余，方便查询）',
    ADD COLUMN sign_result  INT DEFAULT 0 COMMENT '验签结果 0-通过 1-失败',
    ADD COLUMN error_msg    VARCHAR(500) COMMENT '错误信息',
    ADD COLUMN user_agent   VARCHAR(255) COMMENT '请求UA',
    MODIFY COLUMN request_param LONGTEXT COMMENT '请求参数（改为LONGTEXT）',
    MODIFY COLUMN response_result LONGTEXT COMMENT '响应结果（改为LONGTEXT）';
```

### 日志记录方式

用 Spring AOP 切面，拦截 `@OpenApi` 注解的方法，自动记录请求/响应。

```java
@OpenApi("pay.create")  // ← 打上这个注解的接口自动记录日志
@PostMapping("/create")
public Result<String> create(@RequestBody CreatePayDTO dto) {
    ...
}
```

---

# 阶段三：支付订单模块

> **目标**：商户系统能够通过 API 创建支付订单，生成支付单号，为后续支付执行做准备。

---

## 3.1 扩展 `pay_payment_order` 表

### 扩展 SQL

```sql
ALTER TABLE pay_payment_order
    ADD COLUMN client_ip      VARCHAR(64)   COMMENT '客户端/服务器IP',
    ADD COLUMN subject        VARCHAR(128)  COMMENT '商品标题（展示用）',
    ADD COLUMN description    VARCHAR(255)  COMMENT '订单描述',
    ADD COLUMN notify_url     VARCHAR(255)  COMMENT '订单级别回调地址（覆盖商户默认回调地址）',
    ADD COLUMN return_url     VARCHAR(255)  COMMENT '支付完成跳转地址',
    ADD COLUMN attach         TEXT          COMMENT '附加数据（透传，原样返回给商户）',
    ADD COLUMN timeout_expire DATETIME      COMMENT '订单超时自动关闭时间',
    ADD COLUMN close_time     DATETIME      COMMENT '关单时间',
    ADD COLUMN close_reason   VARCHAR(255)  COMMENT '关单原因',
    ADD COLUMN fee_amount     DECIMAL(12,2) DEFAULT 0.00 COMMENT '手续费金额',
    ADD COLUMN settle_amount  DECIMAL(12,2) DEFAULT 0.00 COMMENT '结算金额（amount - fee_amount）',
    ADD COLUMN settle_status  INT DEFAULT 0 COMMENT '结算状态 0-未结算 1-已结算',
    ADD COLUMN settle_time    DATETIME      COMMENT '结算时间',
    ADD COLUMN sign           VARCHAR(128)  COMMENT '订单签名（防篡改）';
```

### 新增字段说明

| 字段 | 用途 | 为什么需要 |
|------|------|-----------|
| `client_ip` | 请求来源 IP | 风控、安全审计 |
| `subject` / `description` | 商品信息 | 展示给用户看"你买了什么"，对账也需要 |
| `notify_url` | 订单级回调地址 | 不同订单可能需要回调到不同地址 |
| `return_url` | 支付完成跳转 | Web 支付场景，支付完跳回商户页面 |
| `attach` | 附加数据 | 商户自定义参数，原样透传，非常常用的需求 |
| `timeout_expire` | 超时关单时间 | 订单创建后 N 分钟不支付自动关闭，释放资源 |
| `close_time/reason` | 关单信息 | 审计追溯 |
| `fee_amount` | 手续费 | 平台盈利，从订单金额中扣除 |
| `settle_amount` | 结算金额 | = amount - fee_amount，实际打给商户的钱 |
| `sign` | 订单签名 | 用平台密钥对关键字段签名，防内部篡改 |

---

## 3.2 实现创建支付订单接口

### 接口

```
POST /open/pay/create
```

### 完整流程

```
商户系统发起请求
    │
    ├── Header: appKey, timestamp, nonce, sign
    └── Body:
        {
          "orderNo": "ORD20260806001",        // 商户自己的订单号
          "amount": 99.00,                     // 支付金额
          "subject": "iPhone 15 手机壳",        // 商品标题
          "description": "透明防摔手机壳 x1",    // 商品描述
          "channelCode": "BALANCE",            // 支付渠道（BALANCE/ALIPAY/WECHAT）
          "notifyUrl": "https://shop.com/cb",  // 回调地址（可选，不填用商户默认）
          "returnUrl": "https://shop.com/ok",  // 支付完成跳转
          "attach": "{\"tableId\":8}",         // 附加数据透传
          "expireMinutes": 30                  // 订单过期分钟数（默认30）
        }
        │
        ▼
    签权拦截器已经验证了签名（阶段二的工作）
        │
        ▼
    PayOrderService.createPayment():
        │
        ① 从拦截器写入的上下文获取 merchantId
        │
        ② 幂等检查：
           查询 uk_merchant_payment (merchant_id + orderNo)
           如果已存在 → 直接返回已有的 paymentNo
           原因：网络超时后商户重试，不能创建重复订单
        │
        ③ 参数校验：
           - amount > 0
           - amount ≤ 商户单笔限额 (single_limit)
           - 商户今日累计交易额 + amount ≤ 日限额 (daily_limit)
           - channelCode 对应的渠道是否启用
        │
        ④ 生成 paymentNo：
           格式：PAY + yyyyMMdd + 10位Snowflake
           示例：PAY202608061234567890
        │
        ⑤ 计算手续费：
           fee_amount = amount × settle_fee_rate（如 100 × 0.6% = 0.60）
           settle_amount = amount - fee_amount
        │
        ⑥ 组装订单对象：
           PayPaymentOrder
             .paymentNo = paymentNo
             .merchantId = merchantId
             .orderNo = orderNo
             .merchantPaymentNo = orderNo（同 orderNo）
             .userId = null（执行支付时再绑定）
             .amount = 99.00
             .feeAmount = 0.60
             .settleAmount = 98.40
             .status = WAIT_PAY（0）
             .channelId = 渠道ID
             .clientIp = 请求IP
             .subject = subject
             .description = description
             .notifyUrl = notifyUrl（有则用，无则用商户默认）
             .returnUrl = returnUrl
             .attach = attach
             .expireTime = now + expireMinutes
             .timeoutExpire = now + expireMinutes
             .sign = 对关键字段做平台签名
        │
        ⑦ 写入 pay_payment_order
        │
        ⑧ 返回：
           {
             "paymentNo": "PAY202608061234567890",
             "amount": 99.00,
             "status": "WAIT_PAY",
             "expireTime": "2026-08-06T12:00:00"
           }
```

### 需要实现的类

| 类 | 说明 |
|------|------|
| `CreatePayDTO` | 创建支付请求 DTO |
| `CreatePayVO` | 创建支付返回 VO |
| `PayOrderController` | 支付接口 Controller |
| `PayPaymentOrderServiceImpl.createPayment()` | 核心业务逻辑 |

---

## 3.3 实现支付状态枚举

```java
public enum PayStatusEnum {
    WAIT_PAY(0, "待支付"),    // 订单已创建，等待用户支付
    PAYING(1, "支付中"),      // 用户已发起支付，处理中
    SUCCESS(2, "支付成功"),   // 支付完成
    FAILED(3, "支付失败"),    // 支付失败
    CLOSED(4, "已关闭"),      // 超时关闭或主动关闭
    REFUNDING(5, "退款中"),   // 已发起退款，处理中（阶段八）
    REFUNDED(6, "已退款");    // 退款完成（阶段八）

    // 注意：原表状态只有 0-4，阶段八扩展 5-6
}
```

---

## 3.4 实现支付订单查询

### 接口

```
GET  /open/pay/query/{paymentNo}          ← 通过支付单号查询
GET  /open/pay/query-by-order/{orderNo}   ← 通过商户订单号查询
```

### 查询逻辑

```
① 验签（拦截器自动完成）
② 查 pay_payment_order
③ 校验：该订单属于当前商户（防止商户A查商户B的订单）
④ 返回订单详情
```

---

## 3.5 实现订单超时自动关闭（定时任务）

### 为什么需要？

用户创建了订单但长时间不支付，订单一直处于 WAIT_PAY 状态会占用资源，也影响商户对账。

### 定时任务

```java
@Scheduled(fixedDelay = 60_000)  // 每分钟执行
public void closeExpiredOrders() {
    // 查询 status=WAIT_PAY 且 timeout_expire < now 的订单
    // 批量更新 status=CLOSED, close_reason="超时关闭", close_time=now
}
```

---

# 阶段四：用户账户与余额

> **目标**：为平台用户建立钱包账户，支持余额查询、账户管理。删除 `pay_user` 后，账户通过 `user_id` 关联 `auth_user`。

---

## 4.1 扩展 `pay_user_account` 表

### 扩展 SQL

```sql
ALTER TABLE pay_user_account
    ADD COLUMN total_income   DECIMAL(12,2) DEFAULT 0.00 COMMENT '累计收入',
    ADD COLUMN total_expense  DECIMAL(12,2) DEFAULT 0.00 COMMENT '累计支出',
    ADD COLUMN pay_password   VARCHAR(128)  COMMENT '支付密码（BCrypt加密）',
    ADD COLUMN real_name      VARCHAR(32)   COMMENT '实名认证姓名',
    ADD COLUMN id_card        VARCHAR(18)   COMMENT '实名认证身份证号',
    ADD COLUMN real_name_auth INT DEFAULT 0 COMMENT '实名认证状态 0-未认证 1-已认证',
    ADD COLUMN daily_limit    DECIMAL(12,2) DEFAULT 50000.00 COMMENT '单日支付限额',
    ADD COLUMN daily_used     DECIMAL(12,2) DEFAULT 0.00 COMMENT '今日已支付金额',
    ADD COLUMN daily_date     DATE          COMMENT '限额日期（用于重置日限额）',
    MODIFY COLUMN user_id BIGINT NOT NULL COMMENT '关联 auth_user.id（不再关联 pay_user）';
```

### 新增字段说明

| 字段 | 用途 | 为什么需要 |
|------|------|-----------|
| `total_income/expense` | 累计收支 | 不用每次 sum 流水表即可快速展示用户总览 |
| `pay_password` | 支付密码 | 支付时需要二次确认，6 位数字 BCrypt 加密 |
| `real_name/id_card` | 实名信息 | 合规要求，央行规定支付账户必须实名 |
| `real_name_auth` | 实名状态 | 未实名用户限制支付额度 |
| `daily_limit/used/date` | 日限额 | 用户侧风控，防止账户被盗刷 |

---

## 4.2 实现账户自动创建

### 时机

用户注册成功后（`AuthService.register()`），自动创建钱包账户。

### 改造 AuthServiceImpl.register()

```
现有流程：
  register(dto) {
    ① 校验 username/phone/email 唯一性
    ② BCrypt 加密密码
    ③ 生成 user_no
    ④ 写入 auth_user
    ⑤ 返回
  }

改造后：
  register(dto) {
    ① 校验唯一性
    ② 加密密码
    ③ 生成 user_no
    ④ 写入 auth_user
    ⑤ 创建钱包账户：              ← 新增
         account_no = "UA" + 日期 + Snowflake
         user_id = auth_user.id
         balance = 0.00
         frozen_amount = 0.00
         status = 1
         daily_limit = 50000.00
         total_income = 0.00
         total_expense = 0.00
         写入 pay_user_account
    ⑥ 返回
  }
```

---

## 4.3 实现账户查询接口

### 接口（平台内部，非开放 API）

```
GET  /api/account/my               ← 当前登录用户查自己的账户
GET  /api/account/{accountNo}      ← 管理员查询指定账户
```

### 返回内容

```
{
  "accountNo": "UA20260806001",
  "balance": 1500.00,
  "frozenAmount": 0.00,
  "availableBalance": 1500.00,     ← balance - frozenAmount
  "totalIncome": 5000.00,
  "totalExpense": 3500.00,
  "realNameAuth": true,
  "status": 1
}
```

---

## 4.4 实现实名认证

### 接口

```
POST /api/account/real-name-auth
```

### 流程

```
① 校验支付密码（首次设置时需要先设置支付密码）
② 提交姓名 + 身份证号
③ 调用第三方实名认证接口（开发阶段用正则校验模拟）
④ 更新 pay_user_account.real_name, id_card, real_name_auth=1
```

### 为什么要实名认证？

- 央行《非银行支付机构网络支付业务管理办法》规定
- 未实名用户单日支付限额 1000 元
- 实名用户单日限额可提升至 50000 元

---

# 阶段五：支付执行引擎

> **目标**：实现支付的核心扣款逻辑——从用户账户扣钱，给商户账户加钱。这是整个支付系统最核心的链路，必须保证资金安全。

---

## 5.1 支付安全设计

### 为什么支付要设计得这么复杂？

```
资金安全 = 支付系统的生命线

如果发生以下问题，平台就要赔钱：
  ✗ 重复扣款（用户被扣两次钱）
  ✗ 少扣多付（扣了100却给商户加了200）
  ✗ 余额负数（-100元还能继续支付）
  ✗ 并发冲突（两个线程同时扣同一笔余额）
  ✗ 部分成功（扣了用户钱但没给商户加上）
```

### 安全措施一览

| 措施 | 技术 | 防护场景 |
|------|------|---------|
| 分布式锁 | Redisson | 同一笔订单不能并发执行 |
| 乐观锁 | version 字段 | 同一账户不能并发修改余额 |
| 数据库事务 | `@Transactional` | 扣款和加款要么全成功要么全回滚 |
| 状态机校验 | 枚举状态流转 | 已支付的订单不能再次支付 |
| 余额校验 | BigDecimal 比较 | 余额不足拒绝支付 |
| 幂等校验 | 唯一索引 + 状态 | 同一个订单号不能创建两次支付 |

---

## 5.2 扩展 `pay_merchant_account` 表

```sql
ALTER TABLE pay_merchant_account
    ADD COLUMN total_income   DECIMAL(12,2) DEFAULT 0.00 COMMENT '累计收入',
    ADD COLUMN total_expense  DECIMAL(12,2) DEFAULT 0.00 COMMENT '累计支出（退款）',
    ADD COLUMN total_fee      DECIMAL(12,2) DEFAULT 0.00 COMMENT '累计手续费支出';
```

---

## 5.3 实现支付执行接口

### 接口

```
POST /open/pay/execute
```

### 请求参数

```json
{
  "paymentNo": "PAY202608061234567890",
  "userId": 2084908504743415809,
  "payPassword": "123456"       // 支付密码（余额支付时需要）
}
```

### 完整流程（核心！）

```
PayOrderService.executePayment(paymentNo, userId, payPassword):
    │
    ├── 【阶段一：前置校验】
    │   ├── ① 验证支付密码（余额支付）
    │   │     BCrypt.matches(payPassword, 数据库加密密码)
    │   │     失败 → "支付密码错误"
    │   │
    │   ├── ② 获取分布式锁: pay:lock:{paymentNo}
    │   │     lock.tryLock(5, 30, SECONDS)
    │   │     失败 → "订单处理中，请勿重复操作"
    │   │
    │   └── ③ 查询支付订单
    │         不存在 → "订单不存在"
    │         status != WAIT_PAY → "订单状态异常（当前：xxx）"
    │         expireTime < now → "订单已过期"
    │
    ├── 【阶段二：账户校验】
    │   ├── ④ 查询用户账户
    │   │     pay_user_account WHERE user_id = userId
    │   │     不存在 → "用户账户不存在"
    │   │     status = 0 → "账户已被冻结"
    │   │
    │   ├── ⑤ 查询商户账户
    │   │     pay_merchant_account WHERE merchant_id = order.merchantId
    │   │     不存在 → "商户账户异常"
    │   │     status != 1 → "商户账户不可用"
    │   │
    │   ├── ⑥ 余额校验
    │   │     userAccount.availableBalance < order.amount → "余额不足"
    │   │     (availableBalance = balance - frozenAmount)
    │   │
    │   └── ⑦ 用户日限额校验
    │         (dailyUsed + amount) > dailyLimit → "超出单日支付限额"
    │
    ├── 【阶段三：资金操作（事务内）】
    │   │
    │   ├── ⑧ 更新订单状态为 PAYING
    │   │     UPDATE pay_payment_order
    │   │     SET status = 1, user_id = userId, pay_time = now
    │   │     WHERE payment_no = paymentNo AND status = 0
    │   │     affected_rows = 0 → "订单状态已变更"
    │   │
    │   ├── ⑨ 扣减用户余额（乐观锁）
    │   │     UPDATE pay_user_account
    │   │     SET balance = balance - amount,
    │   │         version = version + 1,
    │   │         total_expense = total_expense + amount,
    │   │         daily_used = daily_used + amount,
    │   │         daily_date = CURDATE()
    │   │     WHERE user_id = userId
    │   │       AND version = #{oldVersion}
    │   │       AND balance >= #{amount}     ← 二次兜底
    │   │     affected_rows = 0 → 回滚 "账户余额变动，请重试"
    │   │
    │   ├── ⑩ 增加商户余额（乐观锁）
    │   │     UPDATE pay_merchant_account
    │   │     SET balance = balance + settle_amount,
    │   │         version = version + 1,
    │   │         total_income = total_income + settle_amount
    │   │     WHERE merchant_id = #{merchantId}
    │   │       AND version = #{oldVersion}
    │   │     affected_rows = 0 → 回滚 "商户账户变动，请重试"
    │   │
    │   ├── ⑪ 写入资金流水 ×2 条
    │   │     流水1：用户支出
    │   │       flow_no = FLW + Snowflake
    │   │       account_type = 1（用户）
    │   │       account_id = userAccount.id
    │   │       payment_no = paymentNo
    │   │       flow_type = 1（支出）
    │   │       amount = -order.amount
    │   │       before_balance = userOldBalance
    │   │       after_balance = userNewBalance
    │   │       remark = "支付消费-" + order.subject
    │   │
    │   │     流水2：商户收入
    │   │       flow_no = FLW + Snowflake
    │   │       account_type = 2（商户）
    │   │       account_id = merchantAccount.id
    │   │       payment_no = paymentNo
    │   │       flow_type = 2（收入）
    │   │       amount = order.settle_amount
    │   │       before_balance = merchantOldBalance
    │   │       after_balance = merchantNewBalance
    │   │       remark = "收款-" + order.subject
    │   │
    │   ├── ⑫ 写入手续费流水（如果 fee > 0）
    │   │     流水3：平台手续费收入
    │   │       flow_type = 3（手续费）
    │   │       amount = order.fee_amount
    │   │       remark = "平台手续费"
    │   │       → 待平台账户创建后使用（阶段五后面会扩展）
    │   │
    │   └── ⑬ 更新订单状态为 SUCCESS
    │         UPDATE pay_payment_order
    │         SET status = 2, pay_time = now
    │         WHERE payment_no = paymentNo
    │
    ├── 【阶段四：事务提交后】
    │   ├── ⑭ 释放分布式锁
    │   ├── ⑮ 发送 MQ 消息（阶段十二）: pay.success
    │   └── ⑯ 触发回调通知（阶段七）
    │          → 立即尝试一次回调
    │          → 失败则写入 pay_payment_notify 等待重试
    │
    └── 返回结果
```

### @Transactional 事务边界

```
整个【阶段三】在 @Transactional 内：
  - 扣用户钱 + 加商户钱 + 写流水 + 更新订单 = 一个事务
  - 任何一步失败，全部回滚
  - 保证资金不会"消失"或"凭空产生"

【阶段四】在事务外部：
  - MQ 发送和回调通知不在事务内
  - 因为事务提交后才算支付真正成功
  - 如果通知失败，事务已经提交了，通过重试弥补
```

---

## 5.4 订单状态机

```
                 创建订单
                    │
                    ▼
              ┌──────────┐
              │ WAIT_PAY │──────── 超时/主动取消 ──▶ ┌──────────┐
              │  (0)     │                          │  CLOSED  │
              └────┬─────┘                          │   (4)    │
                   │                                └──────────┘
                   │ 发起支付
                   ▼
              ┌──────────┐
              │  PAYING  │
              │   (1)    │
              └────┬─────┘
                   │
              ┌────┴────┐
              │         │
          成功 ▼         ▼ 失败
    ┌──────────┐   ┌──────────┐
    │ SUCCESS  │   │  FAILED  │
    │   (2)    │   │   (3)    │
    └────┬─────┘   └──────────┘
         │
         │ 发起退款
         ▼
    ┌──────────┐
    │ REFUNDING │
    │   (5)    │
    └────┬─────┘
         │
    ┌────┴────┐
    │         │
成功 ▼         ▼ 失败
┌──────────┐ ┌──────────┐
│ REFUNDED │ │ 回退到   │
│   (6)    │ │ SUCCESS(2)│
└──────────┘ └──────────┘
```

---

## 5.5 支付失败处理

### 哪些情况支付会失败？

| 失败原因 | 处理方式 |
|---------|---------|
| 余额不足 | 直接返回错误，不重试 |
| 支付密码错误 | 返回错误，记录失败次数（连续 5 次锁账户） |
| 乐观锁冲突 | 重试 3 次（短暂等待 + 重新读取版本号） |
| 分布式锁获取失败 | 返回"订单处理中" |
| 订单状态异常 | 返回当前状态 |
| 数据库异常 | 回滚，返回"系统异常" |

### 乐观锁重试机制

```java
// 乐观锁冲突重试（最多3次）
for (int i = 0; i < 3; i++) {
    // 重新查最新数据
    PayUserAccount latest = userAccountMapper.selectById(userAccount.getId());
    int rows = userAccountMapper.updateByVersion(latest, amount);
    if (rows > 0) break;
    if (i == 2) throw new BusinessException("账户变动频繁，请稍后重试");
    Thread.sleep(50);  // 短暂等待，让其他事务完成
}
```

---

# 阶段六：资金流水系统

> **目标**：所有资金变动必须记录可追溯的流水，支持对账、审计、争议处理。

---

## 6.1 完善 `pay_account_flow` 表

### 表已存在，补充唯一索引和完善枚举

```sql
-- 流水类型说明（通过注释约束，字段 flow_type 已存在）
-- 1: 支出（用户支付）
-- 2: 收入（商户收款）
-- 3: 手续费（平台抽成）
-- 4: 退款支出（商户退款给用户）
-- 5: 退款收入（用户收到退款）
-- 6: 充值（用户充值入账）
-- 7: 冻结（资金冻结）
-- 8: 解冻（资金解冻）
-- 9: 调整（人工调账，超级管理员权限）

-- 新增索引
ALTER TABLE pay_account_flow
    ADD INDEX idx_account_time (account_id, create_time),
    ADD INDEX idx_flow_type (flow_type),
    ADD INDEX idx_account_type_time (account_type, create_time);
```

---

## 6.2 实现流水查询接口

### 接口（平台内部管理后台用）

```
GET /api/flow/list                    ← 分页查询所有流水
GET /api/flow/list-by-account/{accountNo}  ← 按账户查询流水
GET /api/flow/list-by-payment/{paymentNo}  ← 按支付单号查询流水
GET /api/flow/daily-summary           ← 日汇总报表
```

### 流水记录时机（强制规则）

```
涉及余额变化的操作，必须先写流水再改余额（或同一个事务内完成）：

✓ 支付执行 → 写 2 条流水（用户支出 + 商户收入）+ 1 条手续费
✓ 退款执行 → 写 2 条流水（商户支出 + 用户收入）
✓ 充值到账 → 写 1 条流水（用户充值入账）
✓ 人工调账 → 写 2 条流水（双方调整）+ 审计日志
✗ 不允许：改了余额但不写流水
```

---

# 阶段七：支付回调通知

> **目标**：支付成功后及时通知商户系统，通知失败时自动重试，保证通知必达。

---

## 7.1 完善 `pay_payment_notify` 表

```sql
ALTER TABLE pay_payment_notify
    ADD COLUMN notify_type    INT DEFAULT 1 COMMENT '通知类型 1-支付成功 2-退款成功 3-退款失败',
    ADD COLUMN max_retry      INT DEFAULT 10 COMMENT '最大重试次数',
    ADD COLUMN last_error     TEXT COMMENT '最后一次错误信息',
    MODIFY COLUMN notify_status INT DEFAULT 0 COMMENT '0待通知 1成功 2失败(达上限)';
```

---

## 7.2 回调通知流程

```
支付成功（事务已提交）
    │
    ▼
创建回调通知记录
    notify_status = 0（待通知）
    next_retry_time = now（立即执行）
    retry_count = 0
    │
    ▼
立即尝试第一次通知：
    POST {notify_url}
    Body: {
      "notifyType": "PAY_SUCCESS",
      "paymentNo": "PAY20260806...",
      "orderNo": "ORD20260806001",
      "tradeStatus": "SUCCESS",
      "amount": 99.00,
      "payTime": "2026-08-06 10:30:00",
      "attach": "{\"tableId\":8}",
      "sign": "平台的回调签名"
    }
    Timeout: 10s
    │
    ├── 成功（HTTP 200 + 商户返回 "SUCCESS"）
    │     → 更新 notify_status = 1
    │     → 结束
    │
    └── 失败（超时 / HTTP 非200 / 返回非 "SUCCESS"）
          → retry_count + 1
          → 计算下次重试时间
          → 更新 next_retry_time
```

---

## 7.3 重试策略

### 退避算法（指数递增）

```
第 1 次失败 → 1 分钟后重试
第 2 次失败 → 2 分钟后重试
第 3 次失败 → 5 分钟后重试
第 4 次失败 → 10 分钟后重试
第 5 次失败 → 30 分钟后重试
第 6 次失败 → 1 小时后重试
第 7 次失败 → 2 小时后重试
第 8 次失败 → 6 小时后重试
第 9 次失败 → 12 小时后重试
第 10 次失败 → 标记 notify_status=2（失败达上限），人工介入

公式：delay = min(2^retryCount 分钟, 720 分钟)  // 上限 12 小时
```

### 定时任务

```java
@Scheduled(fixedDelay = 30_000)  // 每 30 秒扫描一次
public void retryNotify() {
    // 查询 notify_status=0 且 next_retry_time <= now 的记录
    // 按 next_retry_time ASC 排序，每次最多处理 100 条
    // 逐条执行 HTTP POST 回调
    // 更新结果
}
```

---

## 7.4 回调安全性

### 商户如何验证回调是真的来自平台？

平台回调时也携带签名：

```
回调签名算法：
① 回调参数按 key 字典序排序
② 拼接成 key=value 字符串
③ 末尾追加 appSecret
④ SHA256 生成 sign
⑤ 商户用同样的方式计算 sign，比对
```

商户验证流程：
```
收到回调 → 根据 paymentNo 反查平台确认 → 验证 sign → 更新订单状态
```

---

# 阶段八：退款模块

> **目标**：商户可以发起退款，资金从商户账户退回用户账户。

---

## 8.1 完善 `pay_refund_order` 表

```sql
ALTER TABLE pay_refund_order
    ADD COLUMN merchant_refund_no VARCHAR(64) COMMENT '商户退款单号（幂等用）',
    ADD COLUMN refund_type    INT DEFAULT 1 COMMENT '退款类型 1-全额 2-部分',
    ADD COLUMN apply_amount   DECIMAL(12,2) COMMENT '申请退款金额',
    ADD COLUMN actual_amount  DECIMAL(12,2) COMMENT '实际退款金额',
    ADD COLUMN fee_refund     DECIMAL(12,2) DEFAULT 0.00 COMMENT '退还手续费',
    ADD COLUMN refund_channel INT DEFAULT 1 COMMENT '退款渠道 1-原路退回',
    ADD COLUMN apply_time     DATETIME COMMENT '申请时间',
    ADD COLUMN audit_status   INT DEFAULT 0 COMMENT '审核状态 0-待审核 1-通过 2-驳回',
    ADD COLUMN auditor_id     BIGINT COMMENT '审核人ID',
    ADD COLUMN audit_time     DATETIME COMMENT '审核时间',
    ADD COLUMN fail_reason    VARCHAR(255) COMMENT '退款失败原因',
    ADD COLUMN notify_url     VARCHAR(255) COMMENT '退款回调地址',
    ADD UNIQUE INDEX uk_merchant_refund (merchant_id, merchant_refund_no);
```

---

## 8.2 退款流程

```
商户发起退款申请
    │
    POST /open/pay/refund
    {
      "paymentNo": "PAY20260806...",
      "merchantRefundNo": "REF20260806001",
      "refundAmount": 99.00,          // 退款金额（不能超过原订单金额）
      "refundReason": "用户取消订单",
      "refundType": 1                  // 1-全额 2-部分
    }
    │
    ▼
【第一阶段：校验】
    ① 验签（拦截器自动完成）
    ② 幂等：查 merchant_id + merchant_refund_no 是否已存在
    ③ 查原支付订单
        - 必须存在
        - status 必须是 SUCCESS（已支付才能退）
        - 未过期（支付完成后 90 天内可退）
    ④ 退款金额校验
        - refundAmount ≤ 原订单金额
        - refundAmount ≤ 原订单金额 - 已退款金额（支持部分退款多次退）
        - refundAmount > 0
    ⑤ 查商户账户余额
        - balance >= refundAmount（商户有钱才能退）
    │
    ▼
【第二阶段：执行退款（事务内）】
    ⑥ 创建退款单
        refund_no = RFD + yyyyMMdd + Snowflake
        status = 处理中
    ⑦ 扣减商户余额
        UPDATE pay_merchant_account
        SET balance = balance - refundAmount,
            version = version + 1
        WHERE merchant_id = #{id} AND version = #{ver}
    ⑧ 增加用户余额
        UPDATE pay_user_account
        SET balance = balance + refundAmount,
            version = version + 1
        WHERE user_id = #{id} AND version = #{ver}
    ⑨ 写 2 条退款流水
        流水1：商户退款支出（flow_type=4）
        流水2：用户退款收入（flow_type=5）
    ⑩ 更新退款单状态 = 成功
    ⑪ 如果全额退款，更新原支付订单状态 = REFUNDED
    │
    ▼
【第三阶段：通知】
    ⑫ 退款回调通知商户（复用阶段七的通知机制）
```

---

## 8.3 退款规则

| 规则 | 说明 |
|------|------|
| 退款期限 | 支付成功后 90 天内可退款 |
| 部分退款 | 支持多次部分退款，累计金额 ≤ 原订单金额 |
| 手续费退还 | 全额退款退还全部手续费，部分退款按比例退还 |
| 余额不足 | 商户余额不足时退款失败，提示"商户余额不足，请充值后重试" |
| 退款审核 | 超过一定金额（如 10000 元）需要人工审核 |

---

## 8.4 退款审核（大额）

```java
// 超过 10000 元的退款需要审核
if (refundAmount.compareTo(new BigDecimal("10000")) > 0) {
    refundOrder.setAuditStatus(0);  // 待审核
    // 发送审核通知给管理员
} else {
    refundOrder.setAuditStatus(1);  // 自动通过
    executeRefund(refundOrder);     // 直接执行
}
```

---

# 阶段九：充值模块

> **目标**：用户可以向自己的钱包账户充值，为余额支付准备资金。

---

## 9.1 完善 `pay_recharge_order` 表

```sql
ALTER TABLE pay_recharge_order
    ADD COLUMN channel_id     BIGINT COMMENT '充值渠道（关联pay_payment_channel）',
    ADD COLUMN recharge_way   INT DEFAULT 1 COMMENT '充值方式 1-银行卡 2-支付宝 3-微信',
    ADD COLUMN bank_name      VARCHAR(64) COMMENT '银行名称（银行卡充值时）',
    ADD COLUMN card_no_tail   VARCHAR(4) COMMENT '银行卡尾号',
    ADD COLUMN arrival_amount DECIMAL(12,2) COMMENT '实际到账金额（可能有手续费）',
    ADD COLUMN fee_amount     DECIMAL(12,2) DEFAULT 0.00 COMMENT '充值手续费',
    ADD COLUMN finish_time    DATETIME COMMENT '到账时间';
```

---

## 9.2 充值流程

```
用户发起充值
    │
    POST /api/recharge/create
    {
      "amount": 500.00,
      "rechargeWay": 1,          // 1-银行卡 2-支付宝 3-微信
      "channelCode": "ALIPAY"    // 用什么渠道充值
    }
    │
    ▼
① 生成充值单
    recharge_no = RCG + yyyyMMdd + Snowflake
    status = WAIT_PAY
② 返回充值单号 + 跳转链接（模拟，实际对接第三方支付）
    │
    ▼
用户完成外部支付（模拟）
    │
    POST /api/recharge/callback
    {
      "rechargeNo": "RCG20260806...",
      "payStatus": "SUCCESS"
    }
    │
    ▼
③ 充值到账
    获取锁: recharge:lock:{rechargeNo}
    更新充值单 status = SUCCESS
    增加用户余额:
        UPDATE pay_user_account
        SET balance = balance + amount,
            total_income = total_income + amount
        WHERE user_id = #{userId}
    写入充值流水（flow_type=6）
```

---

# 阶段十：运营后台管理页面

> **目标**：开发 Vue 3 + Element Plus 后台管理页面，让运营人员可视化管理整个支付平台。

---

## 10.1 需要开发的页面清单

### 支付管理

| 页面 | 路由 | 功能 |
|------|------|------|
| 支付订单列表 | `/pay/order` | 分页列表、搜索（商户/单号/状态/日期）、查看详情、手动关单 |
| 支付订单详情 | `/pay/order/:paymentNo` | 订单信息、流水记录、回调记录、退款记录 |

### 退款管理

| 页面 | 路由 | 功能 |
|------|------|------|
| 退款订单列表 | `/pay/refund` | 分页列表、搜索、审核（通过/驳回）、查看详情 |
| 退款订单详情 | `/pay/refund/:refundNo` | 退款信息、关联支付订单、流水记录 |

### 资金管理

| 页面 | 路由 | 功能 |
|------|------|------|
| 用户账户列表 | `/account/user` | 分页列表、搜索、启停、查看流水、调整余额(超管) |
| 商户账户列表 | `/account/merchant` | 查余额、查流水、调整余额(超管) |
| 资金流水列表 | `/account/flow` | 分页列表、按类型/账户/日期筛选 |
| 日汇总报表 | `/account/report` | 今日交易额、手续费、退款统计 |

### 商户管理

| 页面 | 路由 | 功能 |
|------|------|------|
| 商户列表 | `/merchant/list` | 分页列表、搜索、查看详情 |
| 新建商户 | `/merchant/create` | 填写企业信息、联系人、结算配置 |
| 商户详情 | `/merchant/:id` | 基本信息、账户信息、交易统计、密钥管理 |
| 商户审核 | `/merchant/audit` | 待审核列表、通过/驳回 |

### 系统管理

| 页面 | 路由 | 功能 |
|------|------|------|
| 接口日志 | `/system/api-log` | 查看接口调用记录、耗时分析 |
| 回调通知管理 | `/system/notify` | 查看回调记录、手动重试 |
| 支付渠道管理 | `/system/channel` | 渠道启停、配置 |

---

## 10.2 前端开发步骤

每个页面的开发模式遵循项目现有的 CrudSchema 模式：

```
① 在 src/api/ 下创建对应 API 文件
② 在 src/views/ 下创建页面目录和文件
③ 使用 useCrudSchemas.ts 定义表格列和搜索条件
④ 使用 Table 和 Search 组件搭建页面
⑤ 使用 Dialog 组件处理新增/编辑/详情弹窗
```

参考现有页面结构：
- [src/api/](open-platform-admin/src/api/) — API 接口层
- [src/views/Authorization/User/](open-platform-admin/src/views/Authorization/User/) — 用户管理参考
- [src/components/Table/](open-platform-admin/src/components/Table/) — 表格组件
- [src/components/Search/](open-platform-admin/src/components/Search/) — 搜索组件

---

# 阶段十一：Redis 缓存与分布式锁

> **目标**：引入 Redis 提升性能、保证并发安全。

---

## 11.1 商户信息缓存

### 缓存策略

```
Key:   pay:merchant:{appKey}
Value: JSON 序列化的商户信息
TTL:   30 分钟
更新:  商户信息修改时主动删除缓存
```

### 读取流程

```
请求到达 → 拦截器验签
    │
    ├── 查 Redis: pay:merchant:{appKey}
    │   ├── 命中 → 直接使用
    │   └── 未命中 → 查 DB → 写入 Redis → 使用
    │
    └── 校验状态、IP、有效期...
```

### 缓存更新

```java
// 商户信息更新时
public void updateMerchant(PayMerchant merchant) {
    merchantMapper.updateById(merchant);
    // 删除缓存，下次读取时重新加载
    redisTemplate.delete("pay:merchant:" + merchant.getAppKey());
}
```

---

## 11.2 支付防重缓存

```
Key:   pay:create:{merchantId}:{orderNo}
Value: paymentNo
TTL:   1 小时（覆盖订单有效期）
用途:  创建支付订单时先查此缓存，如果存在则直接返回已有 paymentNo
```

### 说明

虽然数据库有 `uk_merchant_payment` 唯一索引做兜底，但 Redis 可以更快地拦截重复请求，减少数据库压力。

---

## 11.3 nonce 防重放缓存

```
Key:   pay:nonce:{appKey}:{nonce}
Value: "1"
TTL:   5 分钟（与 timestamp 时间窗口一致）
```

---

## 11.4 支付密码错误计数

```
Key:   pay:pwd-error:{userId}
Value: 错误次数（整数）
TTL:   30 分钟
规则:  连续 5 次错误，锁定账户 30 分钟
```

```java
public void validatePayPassword(Long userId, String password) {
    String key = "pay:pwd-error:" + userId;
    Integer count = (Integer) redisTemplate.opsForValue().get(key);
    if (count != null && count >= 5) {
        throw new BusinessException("密码错误次数过多，请30分钟后再试");
    }
    if (!BCrypt.matches(password, dbPassword)) {
        redisTemplate.opsForValue().increment(key, 1);
        redisTemplate.expire(key, 30, TimeUnit.MINUTES);
        throw new BusinessException("支付密码错误");
    }
    // 验证成功，清除错误计数
    redisTemplate.delete(key);
}
```

---

## 11.5 分布式锁汇总

| Key 模式 | 用途 | TTL |
|---------|------|-----|
| `pay:lock:execute:{paymentNo}` | 支付执行防重 | 30s |
| `pay:lock:refund:{refundNo}` | 退款执行防重 | 30s |
| `pay:lock:recharge:{rechargeNo}` | 充值到账防重 | 30s |
| `pay:lock:account:{accountId}` | 账户操作锁 | 10s |

---

# 阶段十二：RabbitMQ 异步消息

> **目标**：支付成功、退款成功等事件通过 MQ 异步通知，解耦支付服务与通知服务。

---

## 12.1 消息队列设计

### Exchange / Queue / Routing Key

```
Exchange:  pay.event.exchange（Topic 类型）

Queue:     pay.success.queue
Routing:   pay.event.success
用途:      支付成功 → 消费者发送回调通知

Queue:     pay.refund.queue
Routing:   pay.event.refund
用途:      退款成功 → 消费者发送退款通知

Queue:     pay.flow.queue
Routing:   pay.event.flow
用途:      流水产生 → 消费者做统计/风控

Queue:     pay.recharge.queue
Routing:   pay.event.recharge
用途:      充值成功 → 消费者通知用户
```

---

## 12.2 消息体格式

```json
{
  "eventType": "PAY_SUCCESS",
  "eventId": "EVT202608061234567890",
  "eventTime": "2026-08-06T10:30:00",
  "data": {
    "paymentNo": "PAY20260806...",
    "orderNo": "ORD20260806001",
    "merchantId": 123456,
    "userId": 987654,
    "amount": 99.00,
    "feeAmount": 0.60,
    "settleAmount": 98.40,
    "channelCode": "BALANCE",
    "payTime": "2026-08-06T10:30:00"
  }
}
```

---

## 12.3 改造支付执行流程

```
原流程（同步通知）：
  支付成功 → 立即回调商户

改造后（MQ 异步）：
  支付成功 → 发送 MQ 消息 → 返回成功给调用方
                            │
                            └── 消费者异步处理回调通知
```

### 好处

- 支付接口响应更快（不等回调结果）
- 回调失败不影响支付接口
- 消费者可以独立扩容
- 解耦：支付服务不需要知道通知的细节

---

# 阶段十三：定时任务与对账

> **目标**：通过定时任务完成超时关单、回调重试、日终对账等自动化运维。

---

## 13.1 定时任务清单

| 任务 | 频率 | 说明 |
|------|------|------|
| `closeExpiredOrders` | 每分钟 | 关闭超时未支付的订单 |
| `retryNotify` | 每 30 秒 | 重试失败的回调通知 |
| `resetDailyLimit` | 每天 00:00 | 重置用户和商户的日累计限额 |
| `dailyReconciliation` | 每天 02:00 | 日终对账：流水汇总 vs 账户余额变动 |
| `cleanExpiredNonce` | 每小时 | 清理过期的 nonce 记录（Redis 自动过期，这个是 DB 层面的兜底） |
| `checkSecretExpire` | 每小时 | 检查密钥历史表中已过期的密钥，彻底删除 |

---

## 13.2 日终对账

### 为什么要对账？

```
钱的左边 = 钱的右边

用户侧： 所有用户余额变化之和 = 所有流水之和
商户侧： 所有商户余额变化之和 = 所有流水之和
平台侧： 手续费收入 = 所有手续费流水之和
```

### 对账流程

```
每天凌晨 2:00 执行：

① 汇总昨日所有资金流水：
   SELECT account_type, flow_type, SUM(amount)
   FROM pay_account_flow
   WHERE DATE(create_time) = YESTERDAY
   GROUP BY account_type, flow_type

② 计算昨日账户余额净变动：
   用户余额净变动 = SUM(用户昨日最终余额 - 用户昨日初始余额)
   商户余额净变动 = SUM(商户昨日最终余额 - 商户昨日初始余额)

③ 比对：
   用户余额净变动 = 用户流水净变动 ？
   商户余额净变动 = 商户流水净变动 ？
   总流水支出 = 总流水收入 + 总手续费 ？

④ 不一致 → 生成对账差异报告 → 人工排查
```

### 新增对账表

```sql
CREATE TABLE pay_daily_reconciliation (
    id            BIGINT NOT NULL COMMENT '主键',
    recon_date    DATE NOT NULL COMMENT '对账日期',
    recon_type    INT NOT NULL COMMENT '对账类型 1-用户 2-商户 3-平台',
    flow_amount   DECIMAL(12,2) COMMENT '流水汇总金额',
    account_amount DECIMAL(12,2) COMMENT '账户变动金额',
    diff_amount   DECIMAL(12,2) COMMENT '差异金额',
    status        INT DEFAULT 0 COMMENT '0-一致 1-不一致 2-已处理',
    detail        TEXT COMMENT '详细信息JSON',
    create_time   DATETIME DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    UNIQUE INDEX uk_date_type (recon_date, recon_type)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='日终对账表';
```

---

# 附录 A：完整 ER 图（最终态）

```
auth_user
  │
  ├── 1:1 ── pay_user_account (user_id)
  │              │
  │              ├── 1:N ── pay_account_flow (account_id, account_type=1)
  │              │
  │              └── 1:N ── pay_recharge_order (user_id)
  │
  └── 1:N ── pay_payment_order (user_id)
                 │
                 ├── N:1 ── pay_merchant (merchant_id)
                 │              │
                 │              ├── 1:1 ── pay_merchant_account (merchant_id)
                 │              │              │
                 │              │              └── 1:N ── pay_account_flow (account_id, account_type=2)
                 │              │
                 │              └── 1:N ── pay_merchant_secret_history (merchant_id)
                 │
                 ├── N:1 ── pay_payment_channel (channel_id)
                 │
                 ├── 1:N ── pay_payment_notify (payment_no)
                 │
                 ├── 1:N ── pay_account_flow (payment_no)
                 │
                 └── 1:N ── pay_refund_order (payment_no)

pay_api_log                                  ← 独立，记录所有 API 调用

pay_daily_reconciliation                     ← 独立，日终对账
```

---

# 附录 B：接口全景图（最终态）

## 开放 API（商户调用，需签名认证）

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/open/pay/create` | 创建支付订单 |
| POST | `/open/pay/execute` | 执行支付 |
| GET | `/open/pay/query/{paymentNo}` | 查询支付订单 |
| GET | `/open/pay/query-by-order/{orderNo}` | 按商户订单号查询 |
| POST | `/open/pay/close` | 关闭订单 |
| POST | `/open/pay/refund` | 发起退款 |
| GET | `/open/refund/query/{refundNo}` | 查询退款订单 |

## 平台内部 API（JWT 认证，给管理后台用）

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/auth/login` | 登录 ✅ |
| POST | `/api/auth/register` | 注册 ✅ |
| POST | `/api/auth/logout` | 退出 ✅ |
| POST | `/api/auth/refresh` | 刷新 Token ✅ |
| POST | `/api/merchant/apply` | 商户入驻申请 |
| POST | `/api/merchant/audit` | 商户审核 |
| PUT | `/api/merchant/{no}/enable` | 启用商户 |
| PUT | `/api/merchant/{no}/disable` | 禁用商户 |
| PUT | `/api/merchant/{no}/rotate-secret` | 密钥轮换 |
| GET | `/api/merchant/list` | 商户列表 |
| GET | `/api/merchant/{no}` | 商户详情 |
| GET | `/api/account/my` | 我的账户 |
| GET | `/api/account/{accountNo}` | 账户详情 |
| POST | `/api/account/real-name-auth` | 实名认证 |
| POST | `/api/recharge/create` | 创建充值单 |
| GET | `/api/flow/list` | 流水列表 |
| GET | `/api/flow/daily-summary` | 日汇总 |
| GET | `/api/pay/order/list` | 支付订单列表 |
| GET | `/api/refund/list` | 退款订单列表 |
| POST | `/api/refund/audit` | 退款审核 |

---

# 附录 C：开发顺序依赖图

```
阶段一（商户管理）
    │
    ▼
阶段二（签名认证）          ← 依赖商户数据
    │
    ▼
阶段三（支付订单创建）       ← 依赖签名认证通过
    │
    ├──────────────────────────────┐
    ▼                              ▼
阶段四（用户账户）          阶段五（支付执行）
    │                              │
    └──────────┬───────────────────┘
               ▼
         阶段六（资金流水）    ← 依赖支付执行产生流水
               │
               ▼
         阶段七（回调通知）    ← 依赖支付成功事件
               │
               ▼
         阶段八（退款模块）    ← 依赖支付订单 + 账户 + 流水
               │
               ▼
         阶段九（充值模块）    ← 依赖用户账户
               │
               ▼
    阶段十（运营后台页面）     ← 依赖所有后端 API
               │
               ▼
    阶段十一/十二/十三          ← 横向优化：缓存 + MQ + 定时任务
    （可并行开发）
```

---

# 附录 D：与原计划的对比

| 维度 | 原计划 | 本计划 |
|------|--------|--------|
| 阶段数 | 10 个阶段 | 13 个阶段 |
| 商户管理 | 简单的 CRUD | 入驻申请 + 审核 + 密钥轮换 + IP白名单 + 费率 + 限额 |
| 签名认证 | 基础 MD5 | timestamp + nonce 防重放 + IP 校验 + 授权过期 |
| 支付订单 | 创建 + 模拟支付 | 完整状态机 + 手续费 + 结算 + 超时关单 |
| 用户账户 | 简单余额 | 实名认证 + 支付密码 + 日限额 + 累计统计 |
| 支付执行 | 一个方法 | 四阶段流程 + 乐观锁重试 + 分布式锁 + 完整事务 |
| 资金流水 | 两条流水 | 流水分类型 + 手续费流水 + 对账报表 |
| 回调通知 | 简单 POST | 指数退避重试 + 回调签名 |
| 退款 | 预留 | 完整退款流程 + 部分退款 + 大额审核 + 手续费退还 |
| 充值 | 预留 | 完整充值流程 |
| 后台管理 | 未详细设计 | 10+ 页面详细规划 |
| 新增表 | 0 | 3 张（secret_history + reconciliation + 扩展现有表） |
