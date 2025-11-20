# ANP-智能体支付协议规范 (AP2)

## 摘要

本规范定义了智能体支付协议(Agent Payment Protocol, AP2),这是一个用于智能体之间进行支付和交易交互的标准化协议。AP2是基于ANP(Agent Network Protocol)的应用层协议,旨在实现智能体之间安全、高效的点对点支付交易。

**AP2原始协议**: 由Google于2025年9月发布的开放协议,旨在让AI代理能够安全地代表用户完成支付。官方网站: [https://ap2-protocol.org/](https://ap2-protocol.org/)

**本规范的定位**: 本文档是AP2协议在ANP框架下的改造和扩展版本,针对去中心化智能体网络场景进行了优化和适配。

协议的核心内容包括:
1. 定义了支付场景中的四个核心角色:购物者智能体、商户智能体、凭证提供方、支付处理方
2. 定义了三种关键凭证类型:购物车授权(CartMandate)、支付授权(PaymentMandate)、交付收据(DeliveryReceipt)
3. 使用JWT/JWS标准进行授权签名,确保交易的完整性和安全性
4. 支持多种支付方式,包括二维码支付(支付宝、微信支付)
5. 与ANP的DID身份认证机制深度集成

本规范旨在为智能体网络提供标准化的支付交互方案,支持商品交易、服务采购等多种交易场景。

## 1. 概述

### 1.1 背景

随着智能体网络的发展,智能体需要代表用户完成各种商业交易和支付操作。传统的支付系统主要是为人机交互设计的,缺乏对智能体之间支付场景的原生支持。AP2协议正是为了填补这一空白,为智能体交互提供专门的支付解决方案。

### 1.2 设计原则

- **安全性**:使用密码学签名(JWS/JWT)确保交易内容的完整性和不可抵赖性
- **隐私保护**:支持选择性披露,最小化不必要的信息泄露
- **标准化**:基于现有标准(JWT、Payment Request API等)
- **可扩展性**:支持多种支付方式和未来的协议扩展
- **用户主权**:支付操作需要用户明确授权,确保用户对资产的控制权

### 1.3 与Google AP2的关系及主要改造

本规范基于Google AP2协议(官方网站: [https://ap2-protocol.org/](https://ap2-protocol.org/)),并针对ANP去中心化智能体网络场景进行了重要改造:

#### 1.3.1 Google AP2原始设计

Google AP2使用三种Mandate类型:
- **IntentMandate**: 用于"人不在场"(Human-Not-Present)场景,代理基于用户预先授权自主完成交易
- **CartMandate**: 用于"人在场"(Human-Present)场景,用户实时确认购物车并授权支付
- **PaymentMandate**: 为支付网络提供代理交易的可见性信号

这些Mandate都基于W3C可验证凭证(Verifiable Credentials)标准。

#### 1.3.2 ANP/AP2的主要改造

我们在ANP框架下对AP2进行了以下关键改造:

| 改造项 | Google AP2 | ANP/AP2 | 改造原因 |
|--------|------------|---------|----------|
| **身份认证** | 依赖现有支付网络的身份体系 | 使用DID:WBA去中心化身份 | 实现跨平台、去中心化的智能体身份认证 |
| **通信协议** | 基于A2A(Agent-to-Agent)和MCP协议扩展 | 基于ANP元协议和消息格式 | 统一智能体通信标准,支持更丰富的协商能力 |
| **凭证格式** | 严格遵循W3C VC规范 | 使用JWT/JWS作为轻量级实现 | 简化实现复杂度,同时保持密码学安全性 |
| **支付方式** | 主要面向信用卡/借记卡(Pull支付) | 优先支持二维码支付(支付宝/微信) | 适配中国及亚洲市场的主流支付方式 |
| **IntentMandate** | 支持预授权的自主代理购物 | **暂不支持**(规划中) | 聚焦最小可行实现(M1),后续版本扩展 |
| **角色分离** | 强调四方模型(用户、商户、支付处理方、发卡行) | 支持角色合并的简化实现 | M1阶段允许购物者智能体集成CP和PP功能 |
| **智能体发现** | 未明确定义 | 通过ANP智能体描述协议(ADP)声明能力 | 实现智能体能力的自动发现和协商 |

#### 1.3.3 保留的核心特性

- ✅ 使用密码学签名确保交易不可抵赖
- ✅ 支持用户明确授权(User Authorization)
- ✅ 商户对购物车内容的签名保证(Merchant Authorization)
- ✅ 可审计的交易凭证链
- ✅ 隐私保护和最小信息披露原则

#### 1.3.4 未来演进方向

- [ ] 在M2版本中引入IntentMandate支持"人不在场"场景
- [ ] 支持SD-JWT(Selective Disclosure JWT)实现更细粒度的隐私保护
- [ ] 与W3C VC标准对齐,支持完整的可验证凭证互操作
- [ ] 扩展支付方式:加密货币、央行数字货币(CBDC)等

### 1.4 协议定位

AP2是ANP中的应用层协议,构建在以下基础之上:
- **身份层**:使用did:wba进行智能体身份认证
- **元协议层**:使用ANP消息格式进行智能体通信
- **智能体描述**:在智能体描述文档中声明AP2支持

## 2. 核心概念

### 2.1 角色定义

AP2定义了四个核心角色:

#### 2.1.1 购物者智能体 (Shopper Agent, SA)
- 代表买方/消费者
- 负责与用户交互,展示订单信息,获取支付授权
- 发送购买请求和支付授权

#### 2.1.2 商户智能体 (Merchant Agent, MA)
- 代表卖方/服务提供方
- 负责生成购物车信息,创建订单,发放交付收据
- 接收支付授权并处理交易

#### 2.1.3 凭证提供方 (Credentials Provider, CP)
- 提供用户身份凭证
- 负责用户身份认证和授权签名生成
- 确保支付操作得到用户明确授权

#### 2.1.4 支付处理方 (Payment Processor, PP)
- 处理实际的支付操作
- 负责对接支付渠道(支付宝、微信支付、银行卡等)
- 返回支付结果

**注意**:在最小实现(M1)中,购物者智能体可以集成CP和PP的功能。

### 2.2 核心凭证

#### 2.2.1 CartMandate(购物车授权)
- 由商户智能体生成
- 包含订单详情、商品信息、总金额、支付方式
- 由商户智能体签名,确保订单信息完整性
- 方向:MA → SA

#### 2.2.2 PaymentMandate(支付授权)
- 由购物者智能体生成
- 包含用户对特定订单的支付授权
- 由用户私钥签名,确保授权真实性
- 方向:SA → MA

#### 2.2.3 DeliveryReceipt(交付收据)
- 由商户智能体生成
- 确认订单完成、发货状态或服务提供
- 用于交易记录和纠纷解决
- 方向:MA → SA

### 2.3 交易流程

ANP/AP2协议定义了完整的智能体支付交易流程,包含购物车创建、用户授权、支付处理和交付确认四个主要阶段。

#### 2.3.1 完整交易流程图

```mermaid
sequenceDiagram
    participant User as 用户
    participant SA as 购物者智能体<br/>(Shopper Agent)
    participant MA as 商户智能体<br/>(Merchant Agent)
    participant PP as 支付处理方<br/>(Payment Processor)

    Note over User,PP: 阶段1: 购物车创建
    User->>SA: 1. 发起购买请求<br/>(选择商品、数量、配置)
    SA->>MA: 2. create_cart_mandate<br/>(商品SKU、数量、收货地址)
    Note over MA: 3. 生成订单<br/>计算总价<br/>生成支付二维码
    Note over MA: 4. 计算cart_hash<br/>使用私钥签名生成<br/>merchant_authorization
    MA->>SA: 5. CartMandate<br/>(订单详情 + 二维码 + 商户签名)

    Note over User,PP: 阶段2: 用户授权
    SA->>User: 6. 展示订单信息<br/>(商品、价格、支付二维码)
    User->>User: 7. 扫码支付<br/>(支付宝/微信)
    Note over User: 8. 完成第三方支付<br/>(获得支付凭证)
    User->>SA: 9. 授权支付<br/>(提供支付凭证)

    Note over User,PP: 阶段3: 支付处理
    Note over SA: 10. 计算transaction_data<br/>(cart_hash + pmt_hash)
    Note over SA: 11. 使用用户私钥签名<br/>生成user_authorization
    SA->>MA: 12. send_payment_mandate<br/>(PaymentMandate + 用户授权签名)

    Note over MA: 13. 验证签名<br/>- 验证merchant_authorization<br/>- 验证user_authorization<br/>- 校验cart_hash和pmt_hash
    MA->>PP: 14. 请求支付确认<br/>(支付凭证)
    PP->>PP: 15. 验证支付状态
    PP->>MA: 16. 支付结果<br/>(成功/失败)

    Note over User,PP: 阶段4: 交付确认
    alt 支付成功
        Note over MA: 17. 安排发货/服务提供
        MA->>SA: 18. 返回交易成功<br/>(transaction_id)
        SA->>User: 19. 通知支付成功<br/>显示订单状态
        Note over MA: 20. (可选)发送DeliveryReceipt
        MA->>SA: 21. DeliveryReceipt<br/>(发货信息/服务凭证)
    else 支付失败
        MA->>SA: 18. 返回交易失败<br/>(错误信息)
        SA->>User: 19. 通知支付失败<br/>显示失败原因
    end
```

#### 2.3.2 流程说明

**阶段1: 购物车创建** (步骤1-5)
- 用户通过购物者智能体选择商品并提交订单
- 购物者智能体向商户智能体发送购物车创建请求
- 商户智能体生成CartMandate,包含订单详情和支付二维码
- 商户使用其私钥对购物车内容签名(merchant_authorization)

**阶段2: 用户授权** (步骤6-9)
- 购物者智能体向用户展示订单信息和支付二维码
- 用户通过第三方支付平台(支付宝/微信)完成扫码支付
- 用户获得支付凭证后,向购物者智能体授权交易

**阶段3: 支付处理** (步骤10-16)
- 购物者智能体生成PaymentMandate
- 使用用户私钥对交易数据签名(user_authorization)
- 商户智能体验证所有签名和哈希值
- 商户通过支付处理方确认支付状态

**阶段4: 交付确认** (步骤17-21)
- 支付成功后,商户安排发货或提供服务
- 商户可选择性发送DeliveryReceipt作为交付凭证
- 用户收到交易完成通知

#### 2.3.3 关键安全机制

| 安全机制 | 实现方式 | 作用 |
|---------|---------|------|
| **数据完整性** | cart_hash = SHA-256(JCS(contents)) | 确保购物车内容不被篡改 |
| **商户认证** | merchant_authorization (RS256/ES256K签名) | 证明订单由合法商户生成 |
| **用户授权** | user_authorization (包含transaction_data) | 证明用户明确授权该交易 |
| **防重放攻击** | jti(JWT ID)全局唯一标识符 | 防止授权凭证被重复使用 |
| **时间限制** | iat/exp字段控制有效期 | 限制凭证的时间窗口(建议15分钟) |
| **身份绑定** | cnf字段绑定持有者DID | 确保凭证只能由特定智能体使用 |

## 3. 智能体描述协议集成

### 3.1 在AD文档中声明AP2支持

支持AP2协议的智能体应在其智能体描述文档中声明:

```json
{
  "protocolType": "ANP",
  "protocolVersion": "1.0.0",
  "type": "AgentDescription",
  "name": "Grand Hotel Assistant",
  "did": "did:wba:grand-hotel.com:service:hotel-assistant",
  "interfaces": [
    {
      "type": "NaturalLanguageInterface",
      "protocol": "AP2/ANP",
      "version": "0.0.1",
      "url": "https://grand-hotel.com/api/ap2.json",
      "description": "基于ANP协议的AP2协议实现,用于智能体之间的支付和交易"
    }
  ]
}
```

### 3.2 AP2接口描述

`ap2.json`文件描述了智能体支持的角色和端点:

```json
{
  "ap2/anp": "0.0.1",
  "roles": {
    "merchant": {
      "description": "商户智能体 - 生成购物车、创建二维码订单、发放交付收据",
      "endpoints": {
        "create_cart_mandate": "/ap2/merchant/create_cart_mandate",
        "send_payment_mandate": "/ap2/merchant/send_payment_mandate"
      }
    },
    "shopper": {
      "description": "购物者智能体 - 处理用户交互、PIN验证、二维码展示",
      "endpoints": {
        "receive_delivery_receipt": "/ap2/shopper/receive_delivery_receipt"
      }
    }
  }
}
```

### 3.3 内联接口描述

或者,可以直接在AD文档中详细描述接口信息:

```json
{
  "type": "NaturalLanguageInterface",
  "protocol": "AP2/ANP",
  "version": "0.0.1",
  "description": "AP2协议实现,支持支付和交易操作",
  "content": {
    "roles": {
      "merchant": {
        "description": "商户智能体 - 酒店预订服务提供方",
        "create_cart_mandate": {
          "description": "创建酒店预订的购物车授权",
          "endpoint": "/ap2/merchant/create_cart_mandate",
          "item_schema": {
            "type": "object",
            "properties": {
              "hotel_id": {"type": "integer", "description": "酒店ID"},
              "rate_plan_id": {"type": "string", "description": "房价计划ID"},
              "room_num": {"type": "integer", "description": "房间数量"},
              "check_in_date": {"type": "string", "format": "date", "description": "入住日期(YYYY-MM-DD)"},
              "check_out_date": {"type": "string", "format": "date", "description": "离店日期(YYYY-MM-DD)"}
            },
            "required": ["hotel_id", "rate_plan_id", "room_num", "check_in_date", "check_out_date"]
          }
        }
      }
    }
  }
}
```

## 4. 凭证定义

### 4.1 CartMandate(购物车授权)

**方向**: MA → SA

**完整消息结构**:

```json
{
  "contents": {
    "id": "cart_shoes_123",
    "user_signature_required": false,
    "payment_request": {
      "method_data": [
        {
          "supported_methods": "QR_CODE",
          "data": {
            "channel": "ALIPAY",
            "qr_url": "https://pay.example.com/qrcode/abc123",
            "out_trade_no": "order_20250117_123456",
            "expires_at": "2025-01-17T09:15:00Z"
          }
        },
        {
          "supported_methods": "QR_CODE",
          "data": {
            "channel": "WECHAT",
            "qr_url": "https://pay.example.com/qrcode/abc123",
            "out_trade_no": "order_20250117_123456",
            "expires_at": "2025-01-17T09:15:00Z"
          }
        }
      ],
      "details": {
        "id": "order_shoes_123",
        "displayItems": [
          {
            "id": "sku-id-123",
            "sku": "Nike-Air-Max-90",
            "label": "Nike Air Max 90",
            "quantity": 1,
            "options": {"color": "red", "size": "42"},
            "amount": {"currency": "CNY", "value": 120.0},
            "pending": null,
            "remark": "请尽快发货"
          }
        ],
        "shipping_address": {
          "recipient_name": "张三",
          "phone": "13800138000",
          "region": "北京市",
          "city": "北京市",
          "address_line": "朝阳区某某街道123号",
          "postal_code": "100000"
        },
        "shipping_options": null,
        "modifiers": null,
        "total": {
          "label": "Total",
          "amount": {"currency": "CNY", "value": 120.0},
          "pending": null
        }
      },
      "options": {
        "requestPayerName": false,
        "requestPayerEmail": false,
        "requestPayerPhone": false,
        "requestShipping": true,
        "shippingType": null
      }
    }
  },
  "merchant_authorization": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "timestamp": "2025-08-26T19:36:36.377022Z"
}
```

**关键点**:
- `contents` 包含购物车内容、支付请求和二维码信息
- `merchant_authorization` 是对 `cart_hash` 的 JWS 签名(RS256 或 ES256K)
- `cart_hash = b64url(sha256(JCS(contents)))`

### 4.2 商户授权凭证 (Merchant Authorization)

#### 4.2.1 概述

`merchant_authorization` 字段是商户对购物车内容 (`CartContents`) 的**短期数字签名授权凭证**,用于保证购物车内容的真实性与完整性。

该字段取代旧版的 `merchant_signature`,并采用符合 JOSE/JWT 标准的 **JSON Web Signature (JWS)** 容器格式。

#### 4.2.2 数据类型

- **类型**: base64url 编码的紧凑 JWS 字符串(`header.payload.signature`)
- **算法**: `RS256` 或 `ES256K`
- **字段**: `CartMandate.merchant_authorization`

#### 4.2.3 Header 格式

```json
{
  "alg": "RS256",
  "kid": "MA-key-001",
  "typ": "JWT"
}
```

或:

```json
{
  "alg": "ES256K",
  "kid": "MA-es256k-key-001",
  "typ": "JWT"
}
```

#### 4.2.4 Payload 格式

```json
{
  "iss": "did:wba:a.com:MA",
  "sub": "did:wba:a.com:MA",
  "aud": "did:wba:a.com:TA",
  "iat": 1730000000,
  "exp": 1730000900,
  "jti": "uuid",
  "cart_hash": "<b64url>",
  "cnf": {"kid": "did:wba:a.com:TA#keys-1"},
  "extensions": ["anp.ap2.qr.v1", "anp.human_presence.v1"]
}
```

**字段说明**:
- `iss`: 签发者(商户智能体 DID)
- `sub`: 主体(可与 iss 相同)
- `aud`: 受众(交易智能体或支付处理方)
- `iat`: 签发时间(秒)
- `exp`: 过期时间(建议 15 分钟)
- `jti`: 全局唯一标识符(防重放攻击)
- `cart_hash`: 对 CartMandate.contents 的哈希
- `cnf`: (推荐)持有者绑定信息
- `extensions`: 支持的协议扩展

#### 4.2.5 cart_hash 计算规则

```text
cart_hash = Base64URL(SHA-256(JCS(CartMandate.contents)))
```

- 使用 [RFC 8785 JSON Canonicalization Scheme (JCS)](https://datatracker.ietf.org/doc/rfc8785/) 对 `CartMandate.contents` 进行规范化
- 对规范化后的 UTF-8 字节执行 `SHA-256` 哈希
- 将结果 Base64URL 编码(去掉"="填充)

#### 4.2.6 签名生成流程(商户端 MA)

1. 计算 `cart_hash`
2. 构造 JWT Payload(含 iss/sub/aud/iat/exp/jti/cart_hash/cnf/extensions)
3. 构造 Header(alg=RS256 或 alg=ES256K, kid=<商户公钥标识>)
4. 用商户私钥对 payload 进行签名,生成紧凑 JWS
5. 将生成的 JWS 作为 `merchant_authorization` 写入 `CartMandate` 对象

#### 4.2.7 验签流程(交易端 TA)

1. 对 `CartMandate.contents` 重新计算 `cart_hash'`
2. 解析 `merchant_authorization`:
   - 提取 Header → `kid`
   - 通过 DID 文档或注册表获取 MA 的公钥
   - 验证 JWS 签名(RS256 或 ES256K,与 Header 匹配)
3. 校验声明:
   - `iss/aud/iat/exp/jti` 均符合规范
   - 当前时间在 `[iat, exp]` 内
   - `jti` 未被重复使用
4. 校验数据绑定:
   - `payload.cart_hash == cart_hash'`,否则拒绝
5. 识别扩展:
   - 如存在 `cnf`,可用于后续持有者验证

#### 4.2.8 参考实现(Python)

```python
import json, base64, hashlib, uuid, time
import jwt  # pip install pyjwt

def jcs_canonicalize(obj):
    return json.dumps(obj, ensure_ascii=False, separators=(",", ":"), sort_keys=True)

def b64url_no_pad(b: bytes) -> str:
    return base64.urlsafe_b64encode(b).decode("ascii").rstrip("=")

def compute_cart_hash(contents: dict) -> str:
    canon = jcs_canonicalize(contents)
    digest = hashlib.sha256(canon.encode("utf-8")).digest()
    return b64url_no_pad(digest)

def sign_merchant_authorization(contents: dict, ma_private_pem: str, kid: str,
                                iss: str, aud: str, cnf: dict = None,
                                ttl_seconds: int = 900) -> str:
    now = int(time.time())
    payload = {
        "iss": iss,
        "sub": iss,
        "aud": aud,
        "iat": now,
        "exp": now + ttl_seconds,
        "jti": str(uuid.uuid4()),
        "cart_hash": compute_cart_hash(contents),
        "extensions": ["anp.ap2.qr.v1", "anp.human_presence.v1"]
    }
    if cnf:
        payload["cnf"] = cnf
    headers = {"alg": "RS256", "kid": kid, "typ": "JWT"}
    return jwt.encode(payload, ma_private_pem, algorithm="RS256", headers=headers)
```

#### 4.2.9 校验清单

| 校验项 | 要求 |
|--------|------|
| 签名算法 | RS256 或 ES256K(需与 Header.alg 一致) |
| 时间窗 | `iat ≤ now ≤ exp`,有效期 ≤ 15 分钟 |
| 重放防护 | `jti` 全局唯一 |
| 签发者与受众 | `iss=MA`,`aud=TA`(或 MPP) |
| 数据一致性 | `payload.cart_hash == computed_cart_hash` |
| DID 解析 | 通过 `kid` → DID 文档解析公钥 |
| 兼容扩展 | 支持解析 `cnf`、`extensions` 字段 |

### 4.3 PaymentMandate(支付授权)

**方向**: SA → MA

**消息结构**:

```json
{
  "payment_mandate_contents": {
    "payment_mandate_id": "pm_12345",
    "payment_details_id": "order_shoes_123",
    "payment_details_total": {
      "label": "Total",
      "amount": {"currency": "CNY", "value": 120.0},
      "pending": null,
      "refund_period": 30
    },
    "payment_response": {
      "request_id": "order_shoes_123",
      "method_name": "QR_CODE",
      "details": {
        "channel": "ALIPAY",
        "out_trade_no": "order_20250117_123456"
      },
      "shipping_address": null,
      "shipping_option": null,
      "payer_name": null,
      "payer_email": null,
      "payer_phone": null
    },
    "merchant_agent": "MerchantAgent",
    "timestamp": "2025-08-26T19:36:36.377022Z"
  },
  "user_authorization": "eyJhbGciOiJFUzI1NksiLCJraWQiOiJkaWQ6ZXhhbXBsZ..."
}
```

**字段说明**:
- `payment_details_id`: cart mandate中Payment request的detail的ID
- `user_authorization`: 用户对交易的授权签名

### 4.4 用户授权 (User Authorization)

用户授权的生成方式参考商户授权凭证,不同点如下:

**Payload 使用 `transaction_data` 代替 `cart_hash`**:

```json
{
  "iss": "did:wba:a.com:user",
  "sub": "did:wba:a.com:user",
  "aud": "did:wba:a.com:MA",
  "iat": 1730000000,
  "exp": 1730000900,
  "jti": "uuid",
  "transaction_data": [
    "<cart_hash>",
    "<pmt_hash>"
  ],
  "cnf": {"kid": "did:wba:a.com:MA#keys-1"}
}
```

**transaction_data 计算**:
```text
cart_hash = Base64URL(SHA-256(JCS(CartMandate.contents)))
pmt_hash = Base64URL(SHA-256(JCS(PaymentMandate.payment_mandate_contents)))
```

## 5. 消息定义

### 5.1 create_cart_mandate

**方向**: Shopper (SA) → Merchant (MA)

**API 路径**: `POST /ap2/merchant/create_cart_mandate`

**请求消息结构**:

```json
{
  "messageId": "cart-request-001",
  "from": "did:wba:a.com:shopper",
  "to": "did:wba:a.com:merchant",
  "data": {
    "cart_mandate_id": "cart-mandate-id-123",
    "items": [
      {
        "id": "sku-id-123",
        "sku": "Nike-Air-Max-90",
        "quantity": 1,
        "options": {"color": "red", "size": "42"},
        "remark": "请尽快发货"
      }
    ],
    "shipping_address": {
      "recipient_name": "张三",
      "phone": "13800138000",
      "region": "北京市",
      "city": "北京市",
      "address_line": "朝阳区某某街道123号",
      "postal_code": "100000"
    },
    "remark": "请尽快发货"
  }
}
```

**响应消息结构**(返回 CartMandate):

```json
{
  "messageId": "cart-response-001",
  "from": "did:wba:a.com:merchant",
  "to": "did:wba:a.com:shopper",
  "data": {
    "contents": {
      "id": "cart-mandate-id-123",
      "user_signature_required": false,
      "payment_request": {
        "method_data": [
          {
            "supported_methods": "QR_CODE",
            "data": {
              "channel": "ALIPAY",
              "qr_url": "https://pay.example.com/qrcode/abc123",
              "out_trade_no": "order_20250117_123456",
              "expires_at": "2025-01-17T09:15:00Z"
            }
          }
        ],
        "details": {
          "id": "order_shoes_123",
          "displayItems": [
            {
              "id": "sku-id-123",
              "sku": "Nike-Air-Max-90",
              "label": "Nike Air Max 90",
              "quantity": 1,
              "options": {"color": "red", "size": "42"},
              "amount": {"currency": "CNY", "value": 120.0},
              "pending": null
            }
          ],
          "total": {
            "label": "Total",
            "amount": {"currency": "CNY", "value": 120.0},
            "pending": null
          }
        }
      }
    },
    "merchant_authorization": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
    "timestamp": "2025-01-17T09:00:01Z"
  }
}
```

### 5.2 send_payment_mandate

**方向**: Shopper (SA) → Merchant (MA)

**API 路径**: `POST /ap2/merchant/send_payment_mandate`

**请求消息结构**:

```json
{
  "messageId": "payment-mandate-001",
  "from": "did:wba:a.com:shopper",
  "to": "did:wba:a.com:merchant",
  "data": {
    "payment_mandate_contents": {
      "payment_mandate_id": "pm_12345",
      "payment_details_id": "order_shoes_123",
      "payment_details_total": {
        "label": "Total",
        "amount": {"currency": "CNY", "value": 120.0},
        "pending": null,
        "refund_period": 30
      },
      "payment_response": {
        "request_id": "order_shoes_123",
        "method_name": "QR_CODE",
        "details": {
          "channel": "ALIPAY",
          "out_trade_no": "order_20250117_123456"
        },
        "shipping_address": null,
        "shipping_option": null,
        "payer_name": null,
        "payer_email": null,
        "payer_phone": null
      },
      "merchant_agent": "MerchantAgent",
      "timestamp": "2025-01-17T09:05:00Z"
    },
    "user_authorization": "eyJhbGciOiJFUzI1NksiLCJraWQiOiJkaWQ6ZXhhbXBsZ..."
  }
}
```

**响应消息结构**:

```json
{
  "messageId": "payment-response-001",
  "from": "did:wba:a.com:merchant",
  "to": "did:wba:a.com:shopper",
  "data": {
    "status": "success",
    "payment_mandate_id": "pm_12345",
    "transaction_id": "txn_67890",
    "message": "支付处理成功"
  }
}
```

## 6. 消息流转顺序

1. **SA 请求** → MA: `create_cart_mandate`(购物车创建请求)
2. **MA 返回** → SA: `CartMandate`(购物车授权 + 二维码)在HTTP响应中
3. **SA 请求** → MA: `send_payment_mandate`(支付授权)
4. **MA 返回** → SA: 支付处理结果

## 7. 安全性考虑

### 7.1 签名验证

所有关键凭证(CartMandate、PaymentMandate)都必须进行签名和验证:
- 商户智能体使用其私钥对CartMandate签名
- 用户使用其私钥对PaymentMandate签名
- 接收方必须通过DID文档中的公钥验证签名

### 7.2 时间戳和过期时间

- 所有凭证都必须包含时间戳
- CartMandate应包含二维码过期时间(expires_at)
- JWT令牌应设置合理的过期时间(建议15分钟)

### 7.3 重放攻击防护

- 使用`jti`(JWT ID)确保每个授权凭证全局唯一
- 接收方应维护已使用的`jti`列表,防止重放攻击

### 7.4 HTTPS传输

- 所有API端点必须使用HTTPS协议
- 确保通信过程中敏感信息的加密传输

### 7.5 用户授权

- 支付操作必须获得用户明确授权
- 用户私钥应安全存储,不离开用户设备
- 支持硬件安全模块或安全飞地进行密钥存储

## 8. 扩展机制

### 8.1 支付方式扩展

AP2支持通过`supported_methods`字段扩展新的支付方式:
- `QR_CODE`: 二维码支付
- `CARD`: 银行卡支付
- `CRYPTO`: 加密货币支付
- 自定义支付方式

### 8.2 协议扩展字段

JWT Payload支持`extensions`字段声明协议扩展:
```json
{
  "extensions": [
    "anp.ap2.qr.v1",
    "anp.human_presence.v1",
    "anp.ap2.subscription.v1"
  ]
}
```

## 9. 实现建议

### 9.1 最小实现(M1)

最小实现应包括:
- 支持二维码支付方式(支付宝或微信支付)
- 实现CartMandate和PaymentMandate
- 基本的签名生成和验证
- DID:WBA身份认证

### 9.2 完整实现

完整实现还应包括:
- 支持多种支付方式
- DeliveryReceipt实现
- 角色分离(SA、MA、CP、PP)
- 退款和争议处理机制

### 9.3 库和工具推荐

- JWT处理: PyJWT (Python)、jsonwebtoken (JavaScript)
- JSON规范化: jcs (多语言实现)
- DID解析: Universal Resolver
- 密码学操作: cryptography (Python)、Web Crypto API (JavaScript)

## 10. 示例场景

### 10.1 电商购物场景

1. 用户通过购物者智能体浏览商品
2. 用户选择商品并提交订单
3. 购物者智能体向商户智能体发送create_cart_mandate请求
4. 商户智能体返回CartMandate,包含支付二维码
5. 购物者智能体向用户展示订单信息和二维码
6. 用户扫码完成支付
7. 购物者智能体生成PaymentMandate并发送给商户智能体
8. 商户智能体验证支付并安排发货

### 10.2 服务预订场景(酒店预订)

1. 用户通过购物者智能体搜索酒店
2. 用户选择酒店和房型
3. 购物者智能体向酒店商户智能体发送预订请求
4. 酒店商户智能体返回预订CartMandate,包含总费用
5. 用户确认预订信息并授权支付
6. 购物者智能体发送PaymentMandate给酒店商户智能体
7. 酒店商户智能体确认支付并生成预订确认

## 11. 参考文献

- [RFC 7519] JSON Web Token (JWT)
- [RFC 7515] JSON Web Signature (JWS)
- [RFC 8785] JSON Canonicalization Scheme (JCS)
- [W3C Payment Request API](https://www.w3.org/TR/payment-request/)
- [ANP 技术白皮书](../../01-agentnetworkprotocol-technical-white-paper.md)
- [ANP 智能体描述协议](../../07-anp-agent-description-protocol-specification.md)
- [DID:WBA 方法规范](../../03-did-wba-method-design-specification.md)

## 版权声明

Copyright (c) 2024 GaoWei Chang
本文件依据 [MIT 许可证](../../LICENSE) 发布,您可以自由使用和修改,但必须保留本版权声明。
