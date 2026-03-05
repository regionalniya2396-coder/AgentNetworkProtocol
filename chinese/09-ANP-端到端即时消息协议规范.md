# ANP-端到端即时消息协议规范 (Draft)

备注：当前此规范仍是草案版本，会有进一步的优化与迭代。

## 摘要

本规范定义了 ANP 端到端即时消息协议，这是一个基于 JSON-RPC 2.0 的即时消息协议，支持私聊、群聊和端到端加密（E2EE）通信。协议使用 `did:wba` 作为身份标识，采用 DID WBA 认证机制进行身份验证。

本协议采用传输无关设计——JSON-RPC 2.0 定义消息格式和方法语义，可承载在 HTTP、WebSocket 或其他传输方式之上。本规范提供 HTTP 传输绑定作为默认参考实现。

E2EE 方案基于 HPKE（Hybrid Public Key Encryption，RFC 9180），采用密钥分离设计——签名密钥（ECDSA secp256r1）与密钥协商密钥（X25519）各司其职。私聊 E2EE 通过 HPKE 一步初始化会话密钥，后续消息使用对称密钥 + 链式 ratchet 加密；群聊 E2EE 基于 Sender Keys + epoch 轮转机制，通过 HPKE 对每个群成员独立分发 Sender Key。

本规范替代原有的 [04-基于DID的端到端加密通信技术协议](chinese/message/04-基于did的端到端加密通信技术协议.md) 和 [05-基于DID的消息服务协议](chinese/message/05-基于did的消息服务协议.md) 中的方案。

## 1. 背景

### 1.1 现有协议回顾

ANP 项目在早期阶段设计了两个消息相关的协议规范：

- **04-基于DID的端到端加密通信技术协议**：定义了基于 WebSocket 的 E2EE 加密通信方案，包括 SourceHello、DestinationHello、Finished 握手消息以及基于 ECDHE 的短期密钥协商机制。
- **05-基于DID的消息服务协议**：定义了基于 WebSocket 的消息路由和转发机制，包括 Message Proxy 连接、router 注册、心跳保活等。

原有方案存在以下问题：
1. **握手流程复杂**：ECDHE 方案需要三步交互式握手（SourceHello → DestinationHello → Finished），增加了延迟和实现复杂度。
2. **仅支持私聊 E2EE**：群聊场景下无法实现端到端加密。
3. **密钥用途混淆**：使用同一组 secp256r1 密钥同时承担签名和密钥协商两种用途。

### 1.2 设计动机

本规范在消息格式层和加密层两个维度进行升级：

**消息格式层：JSON-RPC 2.0（传输无关）**

1. **标准化**：JSON-RPC 2.0 是成熟的远程过程调用协议，定义了统一的请求/响应格式和错误码体系。
2. **传输无关**：JSON-RPC 2.0 仅定义消息格式和方法语义，不依赖特定传输协议。可承载在 HTTP、WebSocket、TCP 或其他传输方式之上，适应不同场景的需求。
3. **简单可靠**：JSON 格式在任何编程语言和平台都有成熟的支持库。
4. **易于扩展**：新增消息类型或 API 方法只需扩展方法名和参数定义，无需修改传输层。

**加密层：HPKE 替代 ECDHE**

1. **标准化安全**：HPKE 是 RFC 9180 标准化方案，经过严格的安全性证明。
2. **一步初始化**：HPKE 支持单向封装，将三步握手简化为一步 init，发起方无需等待对方回复即可开始加密通信。
3. **密钥分离**：X25519 专注密钥协商（HPKE KEM），ECDSA secp256r1 专注签名，职责清晰。
4. **群聊 E2EE**：HPKE 天然支持为多个接收方独立封装密钥，为群聊 Sender Key 分发提供基础。

### 1.3 与已有协议的关系

本规范替代原有的 04 和 05 协议：

- **替代 04（E2EE 协议）**：E2EE 加密方案从 ECDHE 交互式握手升级为 HPKE 单向封装 + 链式 ratchet，新增群聊 E2EE 支持。
- **替代 05（消息服务协议）**：消息收发协议从 WebSocket 长连接改为传输无关的 JSON-RPC 2.0 消息格式，去除了 Message Proxy、router 注册等机制。

## 2. 方案概述

### 2.1 架构设计

整体架构采用消息服务器中转模型：

```plaintext
+-------+          +----------------+          +----------------+          +-------+
|       | HTTP/WS/ |                |          |                | HTTP/WS/ |       |
| Agent | <------> | Message Server | <------> | Message Server | <------> | Agent |
|  (A)  | JSON-RPC |     (A's)      |          |     (B's)      | JSON-RPC |  (B)  |
+-------+          +----------------+          +----------------+          +-------+
```

- **Agent**：消息的发送者和接收者，通过 `did:wba` 进行身份标识。
- **Message Server**：为 Agent 提供消息收发服务，包括消息存储、收件箱管理和群组管理。
- Agent 与其所属的 Message Server 之间通过 JSON-RPC 2.0 协议通信，传输层可选用 HTTP、WebSocket 或其他方式。
- Message Server 之间的通信协议不在本规范的范围内。

### 2.2 核心特性

| 特性 | 说明 |
|------|------|
| **消息格式** | JSON-RPC 2.0 |
| **传输协议** | 传输无关（支持 HTTP、WebSocket 等，详见第 4 节） |
| **内容格式** | `application/json` |
| **身份标识** | `did:wba` (Decentralized Identifier) |
| **认证方式** | DID WBA 认证（详见 [DID:WBA 方法设计规范](03-did-wba方法规范.md)） |
| **消息类型** | 私聊、群聊 |
| **E2EE 支持** | 基于 HPKE 的端到端加密（私聊 + 群聊） |
| **E2EE 密码栈** | DHKEM(X25519, HKDF-SHA256) / HKDF-SHA256 / AES-128-GCM |
| **签名算法** | ECDSA secp256r1 (P-256) |

### 2.3 设计原则

1. **简单性**：使用成熟、广泛支持的标准协议（JSON-RPC 2.0、JSON、HPKE），传输层可灵活选择。
2. **安全性**：支持端到端加密，密钥分离设计，服务端透明转发加密消息，无法读取密文。
3. **开放性**：基于 DID 标准的身份标识，支持跨平台、跨域的消息互通。
4. **实用性**：API 设计面向实际应用场景，包含完整的消息管理功能。

## 3. 身份标识与认证

### 3.1 DID 标识格式

本协议使用 `did:wba` 方法进行身份标识（详见 [DID WBA 方法设计规范](03-did-wba方法规范.md)）。

**用户 DID 格式：**

```
did:wba:{domain}:user:{username}
```

示例：`did:wba:example.com:user:alice`

**群组 DID 格式：**

```
did:wba:{domain}:group:{group_id}
```

示例：`did:wba:example.com:group:group_dev`

当 DID 不存在时，服务端返回错误码 `-32002`（资源不存在）。

### 3.2 认证方式

本协议采用 ANP 统一的 DID WBA 认证机制进行身份验证。完整的认证流程和技术细节详见 [DID:WBA 方法设计规范](03-did-wba方法规范.md)。

## 4. JSON-RPC 2.0 消息格式与传输绑定

### 4.1 请求/响应格式

所有 API 采用 JSON-RPC 2.0 协议定义消息格式。JSON-RPC 2.0 本身是传输无关的，仅定义 JSON 消息的结构和方法语义。

**请求格式：**

```json
{
  "jsonrpc": "2.0",
  "method": "method_name",
  "params": { ... },
  "id": 1
}
```

**成功响应格式：**

```json
{
  "jsonrpc": "2.0",
  "result": { ... },
  "id": 1
}
```

**错误响应格式：**

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32000,
    "message": "错误描述",
    "data": null
  },
  "id": 1
}
```

### 4.2 错误码定义

本协议使用 JSON-RPC 2.0 标准错误码，并扩展了业务错误码：

| 错误码 | 含义 | 说明 |
|--------|------|------|
| -32700 | JSON 解析错误 | 请求体不是合法的 JSON |
| -32600 | 无效请求 | 请求不符合 JSON-RPC 2.0 格式 |
| -32601 | 方法不存在 | 请求的方法名未定义 |
| -32602 | 参数校验失败 | 请求参数不合法（包括 E2EE 约束校验） |
| -32603 | 内部错误 | 服务端内部错误 |
| -32000 | 未认证 | 需要认证但未提供有效凭证 |
| -32001 | 无权限 | 已认证但无权执行该操作 |
| -32002 | 资源不存在 | 请求的资源（DID、消息等）不存在 |
| -32003 | 资源冲突 | 资源已存在或状态冲突（包括 client_msg_id 超过幂等窗口后的重复提交） |
| -32004 | 业务校验错误 | 业务规则校验未通过 |
| -32005 | 限流 | 请求过于频繁，被服务端限流 |

### 4.3 传输绑定

本协议的 JSON-RPC 2.0 消息可承载在不同的传输方式之上。实现方**必须**至少支持一种传输绑定，**推荐**支持 HTTP 绑定。

#### 4.3.1 HTTP 传输绑定（默认）

- **协议**：HTTP/HTTPS POST
- **内容类型**：`Content-Type: application/json`
- **字符编码**：UTF-8
- **端点**：JSON-RPC 请求发送到对应的服务端点路径（如 `/api/v1/messages/rpc`）

所有接口通过 HTTP POST 方法调用，请求体为 JSON-RPC 2.0 格式的 JSON 对象，响应体为 JSON-RPC 2.0 格式的结果或错误。

HTTP 绑定适用于大多数场景，尤其适合无状态、防火墙友好的环境。

#### 4.3.2 WebSocket 传输绑定

- **协议**：WebSocket (ws/wss)
- **端点路径**：`/message/ws`（在消息服务前缀下）
- **内容格式**：每条 WebSocket 消息为一个完整的 JSON-RPC 2.0 JSON 对象
- **字符编码**：UTF-8
- **连接管理**：客户端建立 WebSocket 连接后，可在同一连接上发送和接收多条 JSON-RPC 消息。服务端支持同一用户的多个并发连接（如多设备场景）。

WebSocket 绑定适用于需要实时推送的场景（如消息即时通知），服务端可主动向客户端推送新消息，无需客户端轮询收件箱。

##### 认证

WebSocket 连接**必须**在握手阶段完成认证。实现方**必须**至少支持以下认证方式之一：

| 方式 | 机制 | 说明 |
|------|------|------|
| Bearer JWT | `Authorization: Bearer <jwt_token>` 请求头 | **推荐**。服务端使用预配置公钥本地验证 JWT（RS256）。 |
| DID WBA | `Authorization: DID-WBA <signature>` 请求头 | 服务端将认证头转发到 DID 认证端点进行验证。 |
| JWT Query Parameter | `?token=<jwt_token>` 查询参数 | 用于不支持自定义请求头的环境（如浏览器 WebSocket API）。验证逻辑与 Bearer JWT 一致。 |

认证失败时，服务端**必须**以关闭码 `4001`、原因 `Unauthorized` 关闭 WebSocket 连接。

认证成功后，服务端将已认证的 DID 解析为内部用户标识符。后续 `send` 请求中如未显式指定 `sender_did`，将自动注入已认证的 DID。

##### 客户端 → 服务端：请求

客户端通过 WebSocket 连接发送 JSON-RPC 2.0 请求。支持以下方法：

**`send` — 发送消息**

`send` 方法使用与[第 6.1.1 节](#611-send--发送消息)相同的参数定义，WebSocket 下有以下特殊行为：

- `sender_did` 未提供时自动从已认证身份注入。
- 响应为标准 JSON-RPC 2.0 响应，`id` 与请求一致。
- 通过 WebSocket 发送的消息会触发与 HTTP 发送相同的推送通知流水线。

**`ping` — 应用层心跳**

```json
{"jsonrpc": "2.0", "method": "ping"}
```

服务端响应：

```json
{"jsonrpc": "2.0", "method": "pong"}
```

客户端**应当**定期发送 `ping` 消息以保持连接活跃。心跳间隔**必须**小于服务端空闲超时时间（默认 300 秒）。推荐间隔为 120 秒。

##### 服务端 → 客户端：推送通知

当新消息投递到某用户时（无论通过 HTTP 还是 WebSocket 发送），服务端向该用户所有活跃的 WebSocket 连接推送 JSON-RPC 2.0 **Notification**（无 `id` 字段）：

```json
{
  "jsonrpc": "2.0",
  "method": "new_message",
  "params": {
    "id": "msg-uuid-001",
    "server_seq": 42,
    "sender_did": "did:wba:example.com:user:alice",
    "sender_name": "Alice",
    "sender_avatar": "https://example.com/avatar.jpg",
    "receiver_did": "did:wba:example.com:user:bob",
    "group_did": null,
    "group_name": null,
    "content": "Hello!",
    "type": "text",
    "sent_at": "2024-01-15T10:30:00Z"
  }
}
```

**推送通知 params 字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | string | 消息 ID |
| server_seq | integer | 服务端分配的会话内单调递增序号 |
| sender_did | string | 发送者 DID |
| sender_name | string \| null | 发送者显示名称 |
| sender_avatar | string \| null | 发送者头像 URL |
| receiver_did | string \| null | 接收者 DID（仅私聊，群聊为 null） |
| group_did | string \| null | 群组 DID（仅群聊，私聊为 null） |
| group_name | string \| null | 群组名称（仅群聊） |
| content | string | 消息内容（明文或 E2EE JSON 字符串） |
| type | string | 消息类型（与 `send` 方法的 type 取值一致） |
| sent_at | string \| null | 客户端发送时间戳（ISO 8601） |

推送通知仅投递到拥有活跃 WebSocket 连接的用户。离线消息通过 HTTP `get_inbox` 方法拉取。

##### 连接生命周期

| 事件 | 行为 |
|------|------|
| 空闲超时 | 在空闲超时周期内（默认 300 秒）未收到客户端任何消息，服务端以关闭码 `1000`、原因 `Idle timeout` 关闭连接。 |
| 客户端断连 | 服务端将该连接从活跃连接池中移除。 |
| 服务端关闭 | 优雅关闭所有活跃连接。 |
| 重连 | 客户端**应当**实现指数退避重连策略。重连成功后，客户端**应当**调用 `get_inbox` 拉取断连期间遗漏的消息。 |

#### 4.3.3 其他传输绑定

JSON-RPC 2.0 消息亦可承载在 TCP、QUIC、消息队列等其他传输方式之上。实现方应确保：

- 消息边界清晰（每条 JSON-RPC 消息可被完整解析）
- 字符编码为 UTF-8
- 传输层提供可靠传递保证（或在应用层处理重传）

## 5. 智能体描述协议集成

本节描述如何在智能体描述（Agent Description, AD）文档中声明 E2EE 即时消息能力，使其他智能体能够自动发现并与 E2EE IM 服务进行交互。ANP E2EE IM 协议作为内置协议类型纳入智能体描述协议框架（完整的 AD 规范详见[智能体描述协议规范](07-ANP-智能体描述协议规范.md)）。

### 5.1 在 AD 文档中声明 E2EE IM 支持

支持 ANP E2EE IM 协议的智能体应在其 AD 文档的 `interfaces` 数组中包含相应条目。由于 E2EE IM 协议是 ANP 内置协议，接口描述使用 `content` 字段进行内联端点和能力声明，而非通过 `url` 引用外部文件。

**AD 文档示例：**

```json
{
  "protocolType": "ANP",
  "protocolVersion": "1.0.0",
  "type": "AgentDescription",
  "name": "示例聊天智能体",
  "did": "did:wba:example.com:user:alice",
  "interfaces": [
    {
      "type": "StructuredInterface",
      "protocol": "ANP-E2EE-IM",
      "version": "1.0",
      "description": "ANP E2EE 即时消息协议，支持私聊、群聊和端到端加密通信。",
      "content": {
        "endpoints": [
          {
            "path": "/api/v1/messages/rpc",
            "description": "消息服务端点，用于发送、接收和管理消息，包括 E2EE 加密消息"
          },
          {
            "path": "/api/v1/groups/rpc",
            "description": "群组服务端点，用于群组管理和群组消息"
          },
          {
            "path": "/message/ws",
            "transport": "websocket",
            "description": "WebSocket 端点，用于实时消息推送和双向通信"
          }
        ],
        "e2ee": {
          "hpke_suite": "DHKEM-X25519-HKDF-SHA256/HKDF-SHA256/AES-128-GCM",
          "signing_algorithm": "EcdsaSecp256r1Signature2019",
          "features": ["private_chat_e2ee", "group_chat_e2ee"],
          "description": "E2EE 能力：HPKE 密钥封装 + 链式 ratchet（私聊），Sender Keys + epoch 轮转（群聊）"
        }
      }
    }
  ]
}
```

### 5.2 字段说明

**接口级别字段：**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| type | string | 是 | 固定为 `"StructuredInterface"` |
| protocol | string | 是 | 固定为 `"ANP-E2EE-IM"`，标识本协议 |
| version | string | 是 | 协议版本号（如 `"1.0"`） |
| description | string | 否 | 接口的可读描述 |
| content | object | 是 | 内联描述端点和 E2EE 能力 |

**content.endpoints 元素：**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| path | string | 是 | API 端点路径（如 `/api/v1/messages/rpc`） |
| description | string | 否 | 端点功能描述 |

**content.e2ee 字段：**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| hpke_suite | string | 是 | HPKE 密码套件标识 |
| signing_algorithm | string | 是 | 签名算法（如 `"EcdsaSecp256r1Signature2019"`） |
| features | array | 是 | 支持的 E2EE 特性列表（`"private_chat_e2ee"` / `"group_chat_e2ee"`） |
| description | string | 否 | E2EE 能力的可读描述 |

## 6. 消息服务 API 定义

本协议定义了 10 个 JSON-RPC 方法，分布在两个服务端点上。

### 6.1 消息接口

**端点：** `POST /api/v1/messages/rpc`

#### 6.1.1 send — 发送消息

发送消息（私聊或群聊），同时也是 E2EE 加密消息的统一传输方法。

**认证：** optional（已认证用户会校验 sender 身份一致性）

**参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| sender_did | string | 是 | 发送者 DID |
| sender_name | string | 否 | 发送者名称 |
| receiver_did | string | 条件 | 接收者 DID（私聊，与 group_did 互斥） |
| group_did | string | 条件 | 群组 DID（群发，与 receiver_did 互斥） |
| content | string | 是 | 消息内容（普通消息为明文，E2EE 消息为 JSON 序列化字符串） |
| type | string | 否 | 消息类型，默认 `text` |
| timestamp | datetime | 否 | 客户端发送时间戳 |
| client_msg_id | string | 是 | 客户端生成的全局唯一消息 ID（UUIDv4/UUIDv7/ULID），用于幂等去重 |

**type 可选值：**

| type | 类别 | 说明 |
|------|------|------|
| `text` | 普通 | 文本消息 |
| `image` | 普通 | 图片消息 |
| `file` | 普通 | 文件消息 |
| `e2ee_init` | 私聊 E2EE | 会话初始化（HPKE 封装 session seed） |
| `e2ee_msg` | 私聊 E2EE | 加密消息（对称 AEAD 密文） |
| `e2ee_rekey` | 私聊 E2EE | 会话重置/换钥 |
| `e2ee_error` | E2EE | E2EE 错误通知 |
| `group_e2ee_key` | 群聊 E2EE | Sender Key 分发 |
| `group_e2ee_msg` | 群聊 E2EE | 群密文消息 |
| `group_epoch_advance` | 群聊 E2EE | 群纪元变化通知 |

**E2EE 约束：**
- 私聊 E2EE 消息（type 为 `e2ee_init`、`e2ee_msg`、`e2ee_rekey`、`e2ee_error`）必须指定 `receiver_did`，不能指定 `group_did`
- 群聊 E2EE 消息（type 为 `group_e2ee_msg`、`group_epoch_advance`）必须指定 `group_did`，不能指定 `receiver_did`
- `group_e2ee_key`（Sender Key 分发）需同时指定 `receiver_did`（接收该 Sender Key 的成员）和 `group_did`（所属群组），这是唯一同时指定两者的情况
- 违反约束将返回 `-32602` 参数校验失败

**请求示例（私聊）：**

```json
{
  "jsonrpc": "2.0",
  "method": "send",
  "params": {
    "sender_did": "did:wba:example.com:user:alice",
    "receiver_did": "did:wba:example.com:user:bob",
    "content": "你好",
    "type": "text",
    "client_msg_id": "01961e2c-45a0-7c8b-a4e3-f0a1b2c3d4e5"
  },
  "id": 1
}
```

**请求示例（群聊）：**

```json
{
  "jsonrpc": "2.0",
  "method": "send",
  "params": {
    "sender_did": "did:wba:example.com:user:alice",
    "group_did": "did:wba:example.com:group:group_dev",
    "content": "大家好",
    "client_msg_id": "01961e2c-55b1-7d9c-b5f4-a1b2c3d4e5f6"
  },
  "id": 1
}
```

**响应示例：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "id": "msg_001",
    "client_msg_id": "01961e2c-45a0-7c8b-a4e3-f0a1b2c3d4e5",
    "server_seq": 42,
    "sender_did": "did:wba:example.com:user:alice",
    "sender_name": null,
    "receiver_did": "did:wba:example.com:user:bob",
    "group_did": null,
    "content": "你好",
    "type": "text",
    "sent_at": null,
    "created_at": "2024-01-15T10:30:01"
  },
  "id": 1
}
```

**响应 result 字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | string | 服务端分配的消息 ID |
| client_msg_id | string | 客户端提交的幂等键（原样回显） |
| server_seq | integer | 服务端分配的会话内单调递增序号，用于消息排序和 gap 检测 |
| sender_did | string | 发送者 DID |
| sender_name | string \| null | 发送者名称 |
| receiver_did | string \| null | 接收者 DID（私聊） |
| group_did | string \| null | 群组 DID（群聊） |
| content | string | 消息内容 |
| type | string | 消息类型 |
| sent_at | string \| null | 客户端发送时间戳 |
| created_at | string | 服务端创建时间戳 |

E2EE 消息的请求示例详见第 8 节。

##### 6.1.1.1 幂等与排序机制

**client_msg_id 幂等规则：**

- 客户端**必须**为每条 `send` 请求生成全局唯一的 `client_msg_id`（推荐 UUIDv7 或 ULID，兼容 UUIDv4）。
- 服务端以 `(sender_did, client_msg_id)` 作为幂等键。若收到重复的幂等键，服务端**必须**返回首次处理的结果（相同的 `id`、`server_seq` 和 `created_at`），而非创建新消息。
- 幂等窗口**建议**不小于 24 小时。超过幂等窗口后的重复 `client_msg_id` 提交，服务端**可以**返回 `-32003`（资源冲突）错误。
- `client_msg_id` 仅用于传输层幂等去重，与 E2EE 层的 `seq` 防重放机制相互独立、互为补充。

**server_seq 排序机制：**

- `server_seq` 由服务端在消息成功持久化时分配，从 1 开始单调递增（每个会话独立计数）。
- 维度定义：
  - **私聊**：`server_seq` 在 `(min(A_did, B_did), max(A_did, B_did))` 维度内递增，即同一对私聊参与者共享一个序号空间。
  - **群聊**：`server_seq` 在 `group_did` 维度内递增，即同一群组内所有消息共享一个序号空间。
- `server_seq` 保证单调递增，但**允许存在 gap**（例如因数据库分配策略导致的编号间隔），**不允许回退**。
- 所有查询接口返回的消息列表**必须**按 `server_seq` 升序排列。

**客户端 gap 检测与补拉：**

- 客户端**应**记录每个会话的已接收最大 `server_seq`。
- 当通过 WebSocket 推送或查询接口收到的消息 `server_seq` 与本地最大值不连续（即存在 gap）时，客户端**应**调用 `get_history(since_seq=<本地最大 server_seq>)` 补拉缺失的消息。
- 补拉到的消息按 `server_seq` 升序插入本地消息列表。

**server_seq 与 E2EE seq 关系对比：**

| | server_seq | E2EE seq |
|------|------|------|
| **分配方** | Message Server | 发送端客户端 |
| **维度** | 私聊=DID 对，群聊=group_did | 私聊=session_id，群聊=(sender_did, epoch, sender_key_id) |
| **用途** | 消息排序、gap 检测、增量拉取 | 密钥派生、防重放 |
| **可见性** | 明文，所有参与方和服务端可见 | E2EE content 内部，仅端到端可见 |
| **覆盖类型** | 所有消息类型（明文 + E2EE） | 仅 E2EE 加密消息（e2ee_msg, group_e2ee_msg） |

#### 6.1.2 get_detail — 获取消息详情

获取指定消息的详细信息。

**认证：** none

**参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| message_id | string | 是 | 消息 ID |

**请求示例：**

```json
{
  "jsonrpc": "2.0",
  "method": "get_detail",
  "params": {
    "message_id": "msg_001"
  },
  "id": 1
}
```

**响应示例：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "id": "msg_001",
    "server_seq": 42,
    "sender_did": "did:wba:example.com:user:alice",
    "sender_name": null,
    "receiver_did": "did:wba:example.com:user:bob",
    "group_did": null,
    "content": "你好",
    "type": "text",
    "sent_at": null,
    "created_at": "2024-01-15T10:30:01"
  },
  "id": 1
}
```

#### 6.1.3 get_inbox — 查询收件箱

查询当前用户的收件箱消息。

**认证：** optional

**参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| user_did | string | 是 | 当前用户 DID |
| since | datetime | 否 | 只返回此时间之后的消息 |
| skip | int | 否 | 跳过条数，默认 0 |
| limit | int | 否 | 返回条数，默认 50，最大 100 |

**请求示例：**

```json
{
  "jsonrpc": "2.0",
  "method": "get_inbox",
  "params": {
    "user_did": "did:wba:example.com:user:bob"
  },
  "id": 1
}
```

**响应示例：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "messages": [
      {
        "id": "msg_001",
        "server_seq": 42,
        "sender_did": "did:wba:example.com:user:alice",
        "sender_name": "Alice",
        "sender_avatar": "https://example.com/avatar/alice.png",
        "receiver_did": "did:wba:example.com:user:bob",
        "group_did": null,
        "group_name": null,
        "content": "你好",
        "type": "text",
        "sent_at": null,
        "created_at": "2024-01-15T10:30:00",
        "is_read": false
      }
    ],
    "total": 1,
    "has_more": false
  },
  "id": 1
}
```

#### 6.1.4 mark_read — 标记已读

标记指定消息为已读。

**认证：** optional

**参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| user_did | string | 是 | 当前用户 DID |
| message_ids | array | 是 | 要标记为已读的消息 ID 列表 |

**请求示例：**

```json
{
  "jsonrpc": "2.0",
  "method": "mark_read",
  "params": {
    "user_did": "did:wba:example.com:user:bob",
    "message_ids": ["msg_001", "msg_002"]
  },
  "id": 1
}
```

**响应示例：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "updated_count": 2
  },
  "id": 1
}
```

#### 6.1.5 get_history — 获取会话历史

获取私聊或群聊的消息历史记录。

**认证：** optional

**参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| user_did | string | 是 | 当前用户 DID |
| peer_did | string | 条件 | 对方用户 DID（私聊，与 group_did 互斥） |
| group_did | string | 条件 | 群组 DID（群聊，与 peer_did 互斥） |
| since | datetime | 否 | 增量查询起始时间 |
| since_seq | integer | 否 | 只返回 server_seq 大于此值的消息。与 since 互斥，若同时提供以 since_seq 为准 |
| skip | int | 否 | 跳过条数，默认 0 |
| limit | int | 否 | 返回条数，默认 50 |

**请求示例（私聊历史）：**

```json
{
  "jsonrpc": "2.0",
  "method": "get_history",
  "params": {
    "user_did": "did:wba:example.com:user:alice",
    "peer_did": "did:wba:example.com:user:bob"
  },
  "id": 1
}
```

**请求示例（群聊历史）：**

```json
{
  "jsonrpc": "2.0",
  "method": "get_history",
  "params": {
    "user_did": "did:wba:example.com:user:alice",
    "group_did": "did:wba:example.com:group:group_dev"
  },
  "id": 1
}
```

**响应示例：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "messages": [
      {
        "id": "msg_001",
        "server_seq": 42,
        "sender_did": "did:wba:example.com:user:alice",
        "sender_name": "Alice",
        "receiver_did": "did:wba:example.com:user:bob",
        "group_did": null,
        "content": "你好",
        "type": "text",
        "sent_at": null,
        "created_at": "2024-01-15T10:30:00"
      }
    ],
    "total": 1
  },
  "id": 1
}
```

**说明：** 返回的消息列表**必须**按 `server_seq` 升序排列。

#### 6.1.6 get_batch_history — 批量获取群聊历史

批量获取多个群聊的消息历史。

**认证：** optional

**参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| user_did | string | 是 | 当前用户 DID |
| queries | array | 是 | 各群查询参数 `[{group_did, since?, since_seq?}]` |
| limit | int | 否 | 每个群返回条数上限，默认 20 |

**请求示例：**

```json
{
  "jsonrpc": "2.0",
  "method": "get_batch_history",
  "params": {
    "user_did": "did:wba:example.com:user:alice",
    "queries": [
      {"group_did": "did:wba:example.com:group:group_1"},
      {"group_did": "did:wba:example.com:group:group_2", "since": "2024-01-15T00:00:00"},
      {"group_did": "did:wba:example.com:group:group_3", "since_seq": 100}
    ],
    "limit": 20
  },
  "id": 1
}
```

**响应示例：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "groups": [
      {
        "group_id": "group_1",
        "messages": [...],
        "total": 5
      },
      {
        "group_id": "group_2",
        "messages": [...],
        "total": 3
      }
    ]
  },
  "id": 1
}
```

**说明：** 每个 group 的 `messages` 数组中的消息对象包含 `server_seq` 字段，且**必须**按 `server_seq` 升序排列。`queries` 中的 `since_seq` 与 `since` 互斥，若同时提供以 `since_seq` 为准。

#### 6.1.7 delete_inbox — 删除收件箱消息

删除收件箱中的指定消息。

**认证：** optional

**参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| user_did | string | 是 | 当前用户 DID |
| message_ids | array | 是 | 要删除的消息 ID 列表 |

**请求示例：**

```json
{
  "jsonrpc": "2.0",
  "method": "delete_inbox",
  "params": {
    "user_did": "did:wba:example.com:user:bob",
    "message_ids": ["msg_001"]
  },
  "id": 1
}
```

**响应示例：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "deleted_count": 1
  },
  "id": 1
}
```

### 6.2 群组接口

**端点：** `POST /api/v1/groups/rpc`

#### 6.2.1 list — 获取公开群组列表

获取平台上所有公开的群组。

**认证：** none

**参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| skip | int | 否 | 跳过条数，默认 0 |
| limit | int | 否 | 返回条数，默认 50，最大 100 |

**请求示例：**

```json
{
  "jsonrpc": "2.0",
  "method": "list",
  "params": {},
  "id": 1
}
```

**响应示例：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "groups": [
      {
        "did": "did:wba:example.com:group:group_dev",
        "name": "开发群",
        "description": "开发者交流群",
        "avatar": null,
        "is_public": true,
        "created_at": "2024-01-15T10:00:00"
      }
    ],
    "total": 1
  },
  "id": 1
}
```

#### 6.2.2 get_detail — 获取群组详情

获取指定群组的详细信息。

**认证：** none

**参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| group_did | string | 是 | 群组 DID |

**请求示例：**

```json
{
  "jsonrpc": "2.0",
  "method": "get_detail",
  "params": {
    "group_did": "did:wba:example.com:group:group_dev"
  },
  "id": 1
}
```

**响应示例：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "did": "did:wba:example.com:group:group_dev",
    "name": "开发群",
    "description": "开发者交流群",
    "avatar": null,
    "is_public": true,
    "created_at": "2024-01-15T10:00:00"
  },
  "id": 1
}
```

#### 6.2.3 get_messages — 获取群组消息

获取指定群组的消息列表。

**认证：** none

**参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| group_did | string | 是 | 群组 DID |
| limit | int | 否 | 返回条数，默认 50，最大 100 |
| since | datetime | 否 | 只返回此时间之后的消息 |
| since_seq | integer | 否 | 只返回 server_seq 大于此值的消息。与 since 互斥，若同时提供以 since_seq 为准 |

**请求示例：**

```json
{
  "jsonrpc": "2.0",
  "method": "get_messages",
  "params": {
    "group_did": "did:wba:example.com:group:group_dev",
    "limit": 20
  },
  "id": 1
}
```

**响应示例：**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "messages": [
      {
        "id": "msg_010",
        "server_seq": 42,
        "sender_did": "did:wba:example.com:user:alice",
        "sender_name": "Alice",
        "receiver_did": null,
        "group_did": "did:wba:example.com:group:group_dev",
        "content": "大家好",
        "type": "text",
        "sent_at": null,
        "created_at": "2024-01-15T10:30:00"
      }
    ],
    "total": 1
  },
  "id": 1
}
```

**说明：** 返回的消息列表**必须**按 `server_seq` 升序排列。

## 7. 消息类型定义

### 7.1 普通消息类型

| 类型 | 说明 | content 内容 |
|------|------|-------------|
| `text` | 文本消息 | 明文文本 |
| `image` | 图片消息 | 图片 URL 或 Base64 编码 |
| `file` | 文件消息 | 文件 URL 或文件元数据 |

### 7.2 私聊 E2EE 消息类型

所有 E2EE 消息通过 `send` 方法的 `content` 字段传输，content 内容为 JSON 序列化后的字符串。服务端**不解析**也**不修改** content 内容，直接透明转发。

| type | 说明 | content 结构 |
|------|------|-------------|
| `e2ee_init` | 会话初始化 | 含 HPKE 封装数据（`enc`、`encrypted_seed`）和签名 `proof` |
| `e2ee_msg` | 加密消息 | 含 `session_id`、`seq`、`original_type` 和 AES-GCM 密文 |
| `e2ee_rekey` | 会话重置/换钥 | 结构与 `e2ee_init` 相同，语义为重新建立会话密钥 |
| `e2ee_error` | E2EE 错误通知 | 含 `error_code` 和 `session_id` |

E2EE content 结构的详细定义见第 8.4 节。

### 7.3 群聊 E2EE 消息类型

群聊 E2EE 消息同样通过 `send` 方法的 `content` 字段传输，服务端透明转发。

| type | 说明 | content 结构 |
|------|------|-------------|
| `group_e2ee_key` | Sender Key 分发 | 含 HPKE 封装的 Sender Key、`epoch`、`sender_key_id` 和签名 |
| `group_e2ee_msg` | 群密文消息 | 含 `epoch`、`sender_key_id`、`seq`、AES-GCM 密文 |
| `group_epoch_advance` | 群纪元变化通知 | 含 `new_epoch`、`reason`、成员变更列表和管理员签名 |

群聊 E2EE content 结构的详细定义见第 8.6 节。

## 8. 端到端加密（E2EE）协议

### 8.1 概述

本协议的端到端加密方案基于 HPKE（Hybrid Public Key Encryption，RFC 9180），支持私聊和群聊两种场景的加密通信。

**核心设计：**

- **私聊 E2EE**：发起方使用接收方 DID 文档中的 X25519 `keyAgreement` 公钥，通过 HPKE 封装一个随机 session seed（一步完成，无需交互式握手）；后续消息使用从 seed 派生的对称会话密钥 + 链式 ratchet 加密。
- **群聊 E2EE**：每个发送者生成 Sender Key，通过 HPKE 对每个群成员独立封装分发；群消息使用 Sender Key 的链式 ratchet 加密（一次加密，所有持有 Sender Key 的成员均可解密）。成员变更时递增 epoch，触发新一轮 Sender Key 分发。

**关键特性：**

- **服务端透明转发**：所有 E2EE 数据嵌入 `send` 方法的 `content` 字段，服务端不解析或修改加密数据
- **统一传输**：所有 E2EE 消息都通过 `send` 方法的 `type` 字段区分类型
- **一步初始化**：私聊会话建立只需一条 `e2ee_init` 消息，无需交互式握手

### 8.2 密码学基础

#### 8.2.1 密钥分离

本协议严格分离签名密钥和密钥协商密钥：

| 用途 | 算法 | DID 文档字段 | 说明 |
|------|------|-------------|------|
| 签名 / 身份验证 | ECDSA secp256r1 | `authentication` / `assertionMethod` | 用于 proof 签名、DID WBA 认证 |
| 密钥协商 | X25519 | `keyAgreement` | 用于 HPKE KEM 封装/解封装 |

签名密钥不参与密钥协商，密钥协商密钥不参与签名。这种分离设计确保单一密钥的泄露不会同时影响身份认证和通信机密性。

#### 8.2.2 HPKE 密码栈

本协议使用以下固定的 HPKE 密码栈：

| 组件 | 选择 | RFC 9180 标识 |
|------|------|--------------|
| KEM | DHKEM(X25519, HKDF-SHA256) | 0x0020 |
| KDF | HKDF-SHA256 | 0x0001 |
| AEAD | AES-128-GCM | 0x0001 |

完整套件标识字符串：`DHKEM-X25519-HKDF-SHA256/HKDF-SHA256/AES-128-GCM`

HPKE 操作模式：**Base 模式**（即 `mode_base`，不绑定发送方身份到 HPKE 层，发送方身份通过 proof 签名保证）。

#### 8.2.3 DID 文档中的 keyAgreement 引用

HPKE 封装时所需的接收方 X25519 公钥，从接收方 DID 文档的 `keyAgreement` 字段获取。DID:WBA 规范已定义该字段为可选字段，类型 `X25519KeyAgreementKey2019`，公钥格式 `publicKeyMultibase`。

**公钥获取步骤：**

1. 解析接收方 DID，获取 DID 文档
2. 从 `keyAgreement` 数组中取出 `X25519KeyAgreementKey2019` 类型条目
3. 解码 `publicKeyMultibase` 字段得到 X25519 公钥原始字节（32 字节）

**DID 文档相关片段示例：**

```json
{
  "id": "did:wba:example.com:user:alice",
  "verificationMethod": [
    {
      "id": "did:wba:example.com:user:alice#keys-1",
      "type": "EcdsaSecp256r1VerificationKey2019",
      "controller": "did:wba:example.com:user:alice",
      "publicKeyJwk": {
        "crv": "P-256",
        "kty": "EC",
        "x": "...",
        "y": "..."
      }
    },
    {
      "id": "did:wba:example.com:user:alice#key-x25519-1",
      "type": "X25519KeyAgreementKey2019",
      "controller": "did:wba:example.com:user:alice",
      "publicKeyMultibase": "z9hFgmPVfmBZwRvFEyniQDBkz9LmV7gDEqytWyGZLmDXE"
    }
  ],
  "authentication": [
    "did:wba:example.com:user:alice#keys-1"
  ],
  "keyAgreement": [
    "did:wba:example.com:user:alice#key-x25519-1"
  ]
}
```

如果对方 DID 文档中没有 `keyAgreement` 字段或没有 `X25519KeyAgreementKey2019` 类型条目，则表示该智能体不支持 E2EE。

### 8.3 私聊 E2EE 协议

#### 8.3.1 会话模型

每个私聊 E2EE 会话维护以下状态：

| 状态变量 | 类型 | 说明 |
|----------|------|------|
| session_id | string | 会话标识，32 个随机 hex 字符 |
| root_seed | bytes(32) | 来自 HPKE 解密的 session seed |
| send_chain_key | bytes(32) | 发送方向的链密钥 |
| recv_chain_key | bytes(32) | 接收方向的链密钥 |
| send_seq | uint64 | 发送序号，从 0 开始递增 |
| recv_seq | uint64 | 接收序号，从 0 开始递增 |
| expires_at | datetime | 会话过期时间 |

**方向确定规则：** 比较发送方和接收方的 DID 字符串（UTF-8 字节序），字典序较小的一方为 `initiator`。`initiator` 使用 label `"anp-e2ee-init"` 派生 send_chain_key、`"anp-e2ee-resp"` 派生 recv_chain_key；另一方反之。这样保证双方对于 send/recv 密钥链的分配完全一致，无论由谁发起 `e2ee_init`。

#### 8.3.2 会话初始化流程

```plaintext
Alice                           Message Server                          Bob
  |                                   |                                   |
  |  [1. 获取 Bob DID 文档]            |                                   |
  |  [2. 取 keyAgreement X25519 公钥]  |                                   |
  |  [3. 生成 root_seed (32 bytes)]    |                                   |
  |  [4. HPKE.Seal(bob_pk,            |                                   |
  |       root_seed) -> (enc, ct)]     |                                   |
  |  [5. 签名 proof]                   |                                   |
  |                                   |                                   |
  |  send(type=e2ee_init)             |                                   |
  |  content={session_id, enc, ct,    |                                   |
  |           proof, ...}             |                                   |
  |---------------------------------->|  send(type=e2ee_init)             |
  |                                   |---------------------------------->|
  |                                   |                                   |
  |                                   |  [6. HPKE.Open(enc, ct)           |
  |                                   |       -> root_seed]               |
  |                                   |  [7. 验证 proof 签名]             |
  |                                   |  [8. 派生 send/recv chain keys]   |
  |                                   |                                   |
  |  [Alice 也派生 send/recv chain keys，会话即刻激活]                       |
  |                                   |                                   |
  |  send(type=e2ee_msg, seq=0)       |                                   |
  |  content=加密消息                  |                                   |
  |---------------------------------->|  send(type=e2ee_msg)              |
  |                                   |---------------------------------->|
  |                                   |                                   |
```

**流程说明：**

1. Alice 解析 Bob 的 DID 文档，从 `keyAgreement` 中获取 Bob 的 X25519 公钥。
2. Alice 生成 32 字节随机 `root_seed`（**必须**使用密码学安全随机数生成器 CSPRNG 生成）和 `session_id`。
3. Alice 使用 HPKE Base 模式的 `Seal` 操作封装 `root_seed`：
   - 接收方公钥：Bob 的 X25519 公钥
   - 明文：`root_seed`（32 字节）
   - AAD：`session_id` 的 UTF-8 编码
   - 输出：`enc`（32 字节 X25519 临时公钥，Base64 编码）和 `encrypted_seed`（AEAD 密文，Base64 编码）
4. Alice 对整个 content 做 proof 签名（使用自己 DID 文档 `authentication` 中的签名密钥）。
5. Alice 通过 `send` 方法发送 `type=e2ee_init` 消息。Alice 发送后即可直接发送 `e2ee_msg` 加密消息，无需等待 Bob 回复。
6. Bob 收到后，使用自己的 X25519 私钥和 `enc` 执行 HPKE `Open`，恢复 `root_seed`。
7. Bob 从 Alice 的 DID 文档获取签名公钥，验证 proof 签名。
8. 双方各自从 `root_seed` 派生 send/recv chain key（参见 8.3.3），会话激活。

**消息送达顺序要求：**

Message Server **应**（SHOULD）尽力保证同一发送方到同一接收方的消息按发送顺序送达（FIFO 顺序），但在分布式系统中绝对的 FIFO 送达不总是可行。协议通过 `server_seq` 提供**可检测、可恢复**的顺序保证：

- 接收方通过 `server_seq` 检测消息的到达顺序是否符合预期。若 `server_seq` 出现 gap，接收方**应**通过 `get_history(since_seq=...)` 补拉缺失消息。
- `e2ee_init` 消息**应**在后续 `e2ee_msg` 消息之前送达接收方。如果接收方先收到 `e2ee_msg` 而找不到对应 `session_id` 的会话状态，**必须**缓存该消息（上限 64 条，超时 30 秒）并等待 `e2ee_init` 到达。缓存超时后**应**丢弃并返回 `e2ee_error`（error_code: `session_not_found`）。
- `e2ee_rekey` 与后续 `e2ee_msg` 同理，`e2ee_rekey` **应**在使用新 session_id 的 `e2ee_msg` 之前送达。
- 对于群聊，`group_e2ee_key` **应**在同一发送方的 `group_e2ee_msg` 之前送达给对应接收方。缺 key 时的缓存策略参见 8.5.3 节。

**E2EE 消息串行发送约束：**

同一 E2EE 会话中，发送端**必须**等待上一条 `send` 请求的响应返回后，才能发送下一条 E2EE 类型消息（`e2ee_init`、`e2ee_msg`、`e2ee_rekey`、`group_e2ee_key`、`group_e2ee_msg`）。此约束确保 `server_seq` 的分配顺序与发送端的 E2EE `seq` 顺序一致，使接收端可以通过 `server_seq` 的顺序推断 E2EE `seq` 的顺序。

- 此约束**仅适用于 E2EE 类型消息**。普通明文消息（`text`、`image`、`file`）可以并发发送。
- 不同会话之间（不同的 `session_id` 或不同的 `group_did`）的 E2EE 消息可以并发发送。

#### 8.3.3 链式 ratchet 密钥派生

**初始派生（从 root_seed 生成方向链密钥）：**

`root_seed` 由发送方使用密码学安全随机数生成器（CSPRNG）生成的 32 字节随机数，通过 HPKE 加密传输。由于 `root_seed` 本身已具有充分的熵（256 bit 随机），可直接作为 HKDF-Expand 的 PRK（Pseudo-Random Key）输入，无需再经过 HKDF-Extract 步骤。

```python
from cryptography.hazmat.primitives.kdf.hkdf import HKDFExpand
from cryptography.hazmat.primitives import hashes

def derive_chain_keys(root_seed: bytes) -> tuple[bytes, bytes]:
    """从 root_seed 派生两个方向的初始链密钥"""
    init_chain_key = HKDFExpand(
        algorithm=hashes.SHA256(),
        length=32,
        info=b"anp-e2ee-init"
    ).derive(root_seed)

    resp_chain_key = HKDFExpand(
        algorithm=hashes.SHA256(),
        length=32,
        info=b"anp-e2ee-resp"
    ).derive(root_seed)

    return init_chain_key, resp_chain_key
```

**方向分配：**
- DID 字典序较小方（initiator）：`send_chain_key = init_chain_key`，`recv_chain_key = resp_chain_key`
- DID 字典序较大方（responder）：`send_chain_key = resp_chain_key`，`recv_chain_key = init_chain_key`

**每条消息的密钥派生：**

```python
import hmac
import hashlib

def derive_message_key(chain_key: bytes, seq: int) -> tuple[bytes, bytes, bytes]:
    """派生消息密钥，返回 (enc_key, nonce, new_chain_key)"""
    seq_bytes = seq.to_bytes(8, 'big')

    # 派生消息密钥
    msg_key = hmac.new(chain_key, b"msg" + seq_bytes, hashlib.sha256).digest()
    # 更新链密钥
    new_chain_key = hmac.new(chain_key, b"ck", hashlib.sha256).digest()

    # 从 msg_key 派生 AES 加密密钥和 nonce
    enc_key = hmac.new(msg_key, b"key", hashlib.sha256).digest()[:16]  # AES-128
    nonce = hmac.new(msg_key, b"nonce", hashlib.sha256).digest()[:12]  # GCM nonce

    return enc_key, nonce, new_chain_key
```

每条消息使用独立的 `enc_key` 和 `nonce`，用后丢弃。链密钥在每条消息后更新为新值（单向更新，无法从新 chain_key 反推旧 chain_key，实现消息级前向安全）。

#### 8.3.4 消息加密/解密

**加密流程：**

1. 调用 `derive_message_key(send_chain_key, send_seq)` 获取 `enc_key`、`nonce`、`new_chain_key`
2. 使用 AES-128-GCM 加密明文，AAD = `session_id` 的 UTF-8 编码
3. 将密文和 GCM tag 拼接后 Base64 编码，写入 `ciphertext` 字段
4. 更新 `send_chain_key = new_chain_key`，`send_seq += 1`

**解密流程：**

1. 验证消息中的 `seq` 是否在可接受范围内（见下方乱序容忍说明）
2. 调用 `derive_message_key(recv_chain_key, recv_seq)` 获取 `enc_key`、`nonce`、`new_chain_key`
3. Base64 解码 `ciphertext`，分离密文和 GCM tag，使用 AES-128-GCM 解密
4. 更新 `recv_chain_key = new_chain_key`，`recv_seq += 1`

**序号与乱序容忍：**

本规范定义两种 seq 验证策略，实现方**必须**至少支持其一：

- **严格模式（默认）**：要求 `seq == recv_seq`（严格递增），拒绝任何乱序消息。此模式依赖 Message Server 保证同一方向的 FIFO 送达顺序（参见 8.3.2 消息送达顺序要求）。
- **窗口模式（推荐）**：允许 `recv_seq <= seq < recv_seq + MAX_SKIP`（建议 `MAX_SKIP = 256`）。若 `seq > recv_seq`，接收方需要对 `recv_seq` 到 `seq - 1` 的中间链密钥执行"快进"（连续调用 `derive_message_key` 推进 chain_key 到目标位置），并**应**将跳过的 msg_key 缓存以便后续乱序消息到达时解密。缓存的 msg_key **应**设定有效期（建议不超过 300 秒），过期后销毁。

无论采用哪种模式，已使用过的 `seq` **必须**拒绝重复解密（防重放）。

**AES-128-GCM 加密示例：**

```python
import os
import base64
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

def encrypt_message(plaintext: bytes, enc_key: bytes, nonce: bytes,
                    session_id: str) -> str:
    """AES-128-GCM 加密，返回 Base64 编码的 ciphertext（含 tag）"""
    aad = session_id.encode('utf-8')
    aesgcm = AESGCM(enc_key)
    ct_with_tag = aesgcm.encrypt(nonce, plaintext, aad)
    return base64.b64encode(ct_with_tag).decode('utf-8')

def decrypt_message(ciphertext_b64: str, enc_key: bytes, nonce: bytes,
                    session_id: str) -> bytes:
    """AES-128-GCM 解密"""
    aad = session_id.encode('utf-8')
    ct_with_tag = base64.b64decode(ciphertext_b64)
    aesgcm = AESGCM(enc_key)
    return aesgcm.decrypt(nonce, ct_with_tag, aad)
```

#### 8.3.5 会话重建（rekey）

当出现以下情况时，任一方可发送 `e2ee_rekey` 消息重建会话密钥：

- 会话密钥过期（超过 `expires_at`）
- 安全事件（怀疑密钥泄露）
- 设备更换或状态丢失
- 主动密钥轮换

`e2ee_rekey` 的结构与 `e2ee_init` 完全相同。收到 `e2ee_rekey` 后：

1. 执行与 `e2ee_init` 相同的 HPKE 解封装和密钥派生
2. 使用新的 `session_id` 和 `root_seed`
3. 重置 `send_seq` 和 `recv_seq` 为 0
4. 旧会话状态销毁

#### 8.3.6 错误处理

当 E2EE 通信发生异常时，通过 `e2ee_error` 类型消息通知对方。

| error_code | 说明 | 建议操作 |
|------------|------|----------|
| `session_not_found` | 会话不存在或已销毁 | 发起方重新发送 `e2ee_init` |
| `session_expired` | 会话已过期 | 发起方发送 `e2ee_rekey` |
| `decryption_failed` | 解密失败（密钥不同步等） | 发起方发送 `e2ee_rekey` |
| `invalid_seq` | 序号不匹配 | 发起方发送 `e2ee_rekey` |
| `unsupported_suite` | 不支持的 HPKE 密码套件 | 检查对方 AD 文档中的能力声明 |
| `no_key_agreement` | 对方 DID 文档无 keyAgreement | 退回明文通信或放弃 |
| `sender_key_not_found` | 群聊中未收到对应发送方的 Sender Key，缓存超时 | 发送方重新分发 `group_e2ee_key` |

### 8.4 私聊 E2EE content 结构定义

所有 E2EE 数据都通过 `send` 方法的 `content` 字段传输（JSON 序列化字符串）。

#### 8.4.1 e2ee_init — 会话初始化

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| session_id | string | 是 | 会话 ID，32 个随机 hex 字符 |
| hpke_suite | string | 是 | HPKE 密码套件标识 |
| sender_did | string | 是 | 发送方 DID |
| recipient_did | string | 是 | 接收方 DID |
| recipient_key_id | string | 是 | 接收方 DID 文档中 keyAgreement 的 id |
| enc | string | 是 | HPKE 封装密文（Base64 编码，32 字节 X25519 临时公钥） |
| encrypted_seed | string | 是 | HPKE AEAD 加密的 root_seed（Base64 编码，含 GCM tag） |
| expires | integer | 否 | 会话有效期（秒），默认 86400 |
| proof | object | 是 | 发送方签名 |

**proof 结构（适用于所有需要签名的 E2EE 消息）：**

| 字段 | 类型 | 说明 |
|------|------|------|
| type | string | 签名类型（`"EcdsaSecp256r1Signature2019"`） |
| created | string | 签名创建时间（ISO 8601 UTC） |
| verification_method | string | 签名使用的验证方法 ID（DID 文档 `authentication` 中的条目） |
| proof_value | string | 签名值（Base64URL 编码） |

**content 示例（JSON 序列化前）：**

```json
{
  "session_id": "a1b2c3d4e5f60718a1b2c3d4e5f60718",
  "hpke_suite": "DHKEM-X25519-HKDF-SHA256/HKDF-SHA256/AES-128-GCM",
  "sender_did": "did:wba:example.com:user:alice",
  "recipient_did": "did:wba:example.com:user:bob",
  "recipient_key_id": "did:wba:example.com:user:bob#key-x25519-1",
  "enc": "dGhpcyBpcyBhIDMyLWJ5dGUgZXBoZW1lcmFsIHB1YmxpYyBrZXk=",
  "encrypted_seed": "ZW5jcnlwdGVkIHNlZWQgZGF0YSB3aXRoIEdDTSB0YWc=",
  "expires": 86400,
  "proof": {
    "type": "EcdsaSecp256r1Signature2019",
    "created": "2026-03-01T10:30:00Z",
    "verification_method": "did:wba:example.com:user:alice#keys-1",
    "proof_value": "MEUCIQDx..."
  }
}
```

**send 请求示例：**

```json
{
  "jsonrpc": "2.0",
  "method": "send",
  "params": {
    "sender_did": "did:wba:example.com:user:alice",
    "receiver_did": "did:wba:example.com:user:bob",
    "content": "{\"session_id\":\"a1b2c3d4e5f60718a1b2c3d4e5f60718\",\"hpke_suite\":\"DHKEM-X25519-HKDF-SHA256/HKDF-SHA256/AES-128-GCM\",\"sender_did\":\"did:wba:example.com:user:alice\",\"recipient_did\":\"did:wba:example.com:user:bob\",\"recipient_key_id\":\"did:wba:example.com:user:bob#key-x25519-1\",\"enc\":\"...\",\"encrypted_seed\":\"...\",\"expires\":86400,\"proof\":{\"type\":\"EcdsaSecp256r1Signature2019\",\"created\":\"2026-03-01T10:30:00Z\",\"verification_method\":\"did:wba:example.com:user:alice#keys-1\",\"proof_value\":\"...\"}}",
    "type": "e2ee_init",
    "client_msg_id": "01961e2c-60c2-7eac-c6a5-b2c3d4e5f607"
  },
  "id": 1
}
```

#### 8.4.2 e2ee_msg — 加密消息

`e2ee_msg` 不包含 `proof` 签名字段。原因：`e2ee_msg` 的认证性由会话密钥本身保证——只有持有正确 `session_id` 对应的 chain_key 的一方才能生成有效密文（AES-128-GCM 提供认证加密），而 chain_key 来自 `e2ee_init` 中经 proof 签名认证的 root_seed。因此，成功解密本身即可验证发送方身份。此外，省略逐条消息签名避免了 ECDSA 签名的计算开销。

**信任前提**：接收方在处理 `e2ee_msg` 时，**必须**验证 `send` 方法中的 `sender_did` 与该 `session_id` 所绑定的会话对端 DID 一致。若不匹配，**必须**拒绝处理该消息。这确保即使 Message Server 篡改了外层 `sender_did` 字段，也不会导致消息被错误归属到其他会话。

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| session_id | string | 是 | 会话 ID |
| seq | integer | 是 | 消息序号（从 0 开始，严格递增） |
| original_type | string | 是 | 原始消息类型（`text` / `image` / `file`） |
| ciphertext | string | 是 | AES-128-GCM 密文 + tag（Base64 编码） |

**content 示例：**

```json
{
  "session_id": "a1b2c3d4e5f60718a1b2c3d4e5f60718",
  "seq": 0,
  "original_type": "text",
  "ciphertext": "ZW5jcnlwdGVkIG1lc3NhZ2UgY29udGVudA=="
}
```

**send 请求示例：**

```json
{
  "jsonrpc": "2.0",
  "method": "send",
  "params": {
    "sender_did": "did:wba:example.com:user:alice",
    "receiver_did": "did:wba:example.com:user:bob",
    "content": "{\"session_id\":\"a1b2c3d4e5f60718a1b2c3d4e5f60718\",\"seq\":0,\"original_type\":\"text\",\"ciphertext\":\"ZW5jcnlwdGVkIG1lc3NhZ2UgY29udGVudA==\"}",
    "type": "e2ee_msg",
    "client_msg_id": "01961e2c-71d3-7fbd-d7b6-c3d4e5f60718"
  },
  "id": 1
}
```

**说明：** nonce 从链式 ratchet 确定性派生（而非随机生成），GCM tag 附在密文末尾，因此 `ciphertext` 单个字段即可承载完整的 AEAD 输出，无需单独的 `iv` 和 `tag` 字段。

#### 8.4.3 e2ee_rekey — 会话重建

结构与 `e2ee_init` 完全相同，所有字段定义参见 8.4.1。

语义区别：`e2ee_rekey` 表示重建已有的会话密钥，接收方应销毁旧会话状态，使用新的 `session_id` 和 `root_seed` 建立新会话。

#### 8.4.4 e2ee_error — 错误通知

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| error_code | string | 是 | 错误码（见 8.3.6 错误码表） |
| session_id | string | 否 | 关联的会话 ID |
| failed_msg_id | string | 否 | 解密失败的消息 ID |
| message | string | 否 | 可读错误描述 |

**content 示例：**

```json
{
  "error_code": "session_not_found",
  "session_id": "a1b2c3d4e5f60718a1b2c3d4e5f60718",
  "failed_msg_id": "msg_20260305_abc123",
  "message": "E2EE session not found, please re-initialize"
}
```

### 8.5 群聊 E2EE 协议

#### 8.5.1 概述与核心概念

群聊 E2EE 采用 **Sender Keys** 方案：每个群成员作为发送者时，持有自己的对称 Sender Key，并将其通过 HPKE 分发给所有其他群成员。群消息使用发送者的 Sender Key 链式 ratchet 加密，所有持有该 Sender Key 的成员均可解密。

核心概念：

| 概念 | 类型 | 说明 |
|------|------|------|
| **epoch** | integer | 群纪元，非负整数，从 0 开始，成员变更时递增 |
| **sender_key_id** | string | Sender Key 标识，格式建议 `{sender_did_short_hash}:{epoch}` |
| **sender_chain_key** | bytes(32) | 32 字节随机对称密钥，用于链式 ratchet 派生每条群消息的加密密钥 |
| **sender_seq** | uint64 | 发送者在当前 epoch 内的消息序号 |

#### 8.5.2 Sender Key 模型

每个群成员在每个 epoch 生成一个 `sender_chain_key`（32 字节随机数）。该密钥的链式 ratchet 与私聊类似，但使用不同的 label 前缀以区分：

```python
def derive_group_message_key(sender_chain_key: bytes, seq: int) -> tuple[bytes, bytes, bytes]:
    """派生群消息密钥"""
    seq_bytes = seq.to_bytes(8, 'big')

    msg_key = hmac.new(sender_chain_key, b"gmsg" + seq_bytes, hashlib.sha256).digest()
    new_chain_key = hmac.new(sender_chain_key, b"gck", hashlib.sha256).digest()

    enc_key = hmac.new(msg_key, b"key", hashlib.sha256).digest()[:16]
    nonce = hmac.new(msg_key, b"nonce", hashlib.sha256).digest()[:12]

    return enc_key, nonce, new_chain_key
```

#### 8.5.3 Sender Key 分发流程

当成员首次在某个 epoch 发言（或 epoch 轮转后），需要先将 Sender Key 分发给所有其他群成员：

```plaintext
Alice (sender)                  Message Server           Bob (member), Carol (member)
  |                                   |                              |
  |  [生成 sender_chain_key]           |                              |
  |                                   |                              |
  |  [获取 Bob 的 X25519 公钥]         |                              |
  |  [HPKE.Seal(bob_pk,               |                              |
  |   sender_chain_key)]              |                              |
  |                                   |                              |
  |  send(type=group_e2ee_key,        |                              |
  |   receiver_did=bob,               |                              |
  |   group_did=group_dev)            |                              |
  |---------------------------------->|  转发给 Bob                   |
  |                                   |----------------------------->|
  |                                   |                              |
  |  [获取 Carol 的 X25519 公钥]       |                              |
  |  [HPKE.Seal(carol_pk,             |                              |
  |   sender_chain_key)]              |                              |
  |                                   |                              |
  |  send(type=group_e2ee_key,        |                              |
  |   receiver_did=carol,             |                              |
  |   group_did=group_dev)            |                              |
  |---------------------------------->|  转发给 Carol                 |
  |                                   |----------------------------->|
  |                                   |                              |
  |  [分发完成后开始发送群密文消息]       |                              |
  |                                   |                              |
  |  send(type=group_e2ee_msg,        |                              |
  |   group_did=group_dev)            |                              |
  |---------------------------------->|  转发给所有群成员              |
  |                                   |----------------------------->|
```

发送者对群内**每个其他成员**分别执行 HPKE 封装，通过 `send(type=group_e2ee_key)` 发送独立的分发消息。每条消息的 `enc` 和 `encrypted_sender_key` 因接收者不同而不同。

**Sender Key 重放拒绝：** 接收方**必须**拒绝已存在的 `(sender_did, epoch, sender_key_id)` 组合的 `group_e2ee_key` 消息。若收到重复的组合，**必须**丢弃该消息，不得覆盖已有的 Sender Key 状态。这防止攻击者通过重放旧的 `group_e2ee_key` 消息将接收方的 Sender Key 状态回滚到已知值。

**缺 Sender Key 缓存策略：**

当接收方收到 `group_e2ee_msg` 但尚未收到对应发送方的 `group_e2ee_key`（即本地不存在该 `(sender_did, epoch, sender_key_id)` 的 Sender Key）时，**必须**按以下策略处理：

1. **缓存消息**：接收方**必须**将该消息缓存到本地等待队列，而非立即丢弃。
   - 缓存上限：每个 `(sender_did, epoch, sender_key_id)` 维度最多缓存 **64 条**消息。
   - 缓存超时：单条消息的最大缓存等待时间为 **30 秒**。

2. **超时处理**：缓存超时后，接收方**应**丢弃该消息，并向发送方发送 `e2ee_error`（error_code: `sender_key_not_found`），通知发送方重新分发 Sender Key。

3. **Key 到达后处理**：当对应的 `group_e2ee_key` 到达后，接收方**必须**按 `server_seq` 升序处理所有缓存的消息。

4. **超时后补拉**：缓存超时丢弃后，接收方**可以**（MAY）通过 `get_history(since_seq=<本地最大 server_seq>)` 补拉缺失的消息，待 Sender Key 到达后重新处理。

5. **Message Server 职责**：Message Server **应**（SHOULD）尽力保证 `group_e2ee_key` 消息在同一发送方的 `group_e2ee_msg` 消息之前送达给对应接收方。

#### 8.5.4 群消息加密/解密

**加密（发送者）：**

1. 使用自己当前 epoch 的 `sender_chain_key`
2. `derive_group_message_key(sender_chain_key, sender_seq)` 得到 `enc_key`、`nonce`、`new_chain_key`
3. AES-128-GCM 加密，AAD = `group_did + ":" + str(epoch)` 的 UTF-8 编码
4. 更新 `sender_chain_key = new_chain_key`，`sender_seq += 1`
5. 通过 `send(type=group_e2ee_msg, group_did=...)` 发送，服务端转发给所有群成员

**解密（接收者）：**

1. 根据消息中的 `sender_did` + `epoch` + `sender_key_id` 查找本地存储的对应 `sender_chain_key`
2. 验证 `seq` 是否在可接受范围内（见下方群聊序号规则）
3. 派生密钥，解密
4. 更新对应 sender 的 `recv_seq` 和 `sender_chain_key`

**群聊序号与乱序容忍：**

群聊中，每个 `(sender_did, epoch, sender_key_id)` 维度维护独立的 `recv_seq`。序号验证策略与私聊一致（参见 8.3.4）：
- **严格模式**：要求 `seq == recv_seq`
- **窗口模式（推荐）**：允许 `recv_seq <= seq < recv_seq + MAX_SKIP`，跳过的 msg_key 可缓存

无论哪种模式，已使用过的 `seq` **必须**拒绝重复解密（防重放）。

#### 8.5.5 Epoch 轮转（成员变更）

当群组成员发生变更时，epoch 递增以保证前向和后向安全：

**管理员身份与信任锚：**

`group_epoch_advance` 消息的发送者**必须**具有群管理员权限。群管理员的身份由 Message Server 的群组管理 API 定义和维护（参见第 6 节群组管理相关接口）。具体而言：
- 群创建者默认为群管理员。
- Message Server 在转发 `group_epoch_advance` 消息前，**应**验证发送者是否为群管理员。非管理员发送的 `group_epoch_advance` 消息**应**被 Message Server 拒绝。
- 接收方在处理 `group_epoch_advance` 时，通过 `proof.verification_method` 验证签名者身份，并**应**检查该签名者是否为已知的群管理员（管理员列表可通过群组管理 API 获取或由先前的群元数据消息缓存）。
- 客户端**不应**完全依赖 Message Server 的管理员校验——客户端**必须**独立验证 `proof` 签名，确保即使 Message Server 被攻破或存在恶意行为，也无法伪造 epoch 变更。

**轮转流程：**

1. 群管理员（或服务端）发送 `group_epoch_advance` 消息，通告新 epoch 和变更原因
2. 所有留在群内的成员转入新 epoch，旧 epoch 的 Sender Key 按以下规则处理：
   - **必须**将旧 epoch 的 `sender_chain_key` 状态标记为只读，**禁止**使用旧 epoch 密钥发送新消息
   - **应**保留旧 epoch 的 `sender_chain_key`（只读模式），以便解密延迟到达的旧 epoch 消息
   - 旧 epoch 密钥的保留时间**建议**不超过 3600 秒（1 小时），过期后**应**销毁
3. 每个成员为新 epoch 重新生成 `sender_chain_key`
4. 每个成员将新的 Sender Key 通过 `group_e2ee_key` 分发给其他所有当前群成员

**安全保证：**

- **前向安全（成员移除）**：被移除的成员无法获取新 epoch 的 Sender Keys，无法解密未来消息
- **后向安全（成员加入）**：新加入的成员仅获得当前 epoch 的 Sender Keys，无法解密历史消息

**epoch 递增触发条件：**

| 触发事件 | reason 字段值 | 必需动作 |
|----------|--------------|---------|
| 成员加入 | `member_added` | epoch+1，**所有**现有成员**必须**重新生成 Sender Key 并向全体成员（含新成员）重新分发 |
| 成员退出或被移除 | `member_removed` | epoch+1，**所有**留在群内的成员**必须**重新生成 Sender Key 并向全体成员重新分发 |
| 主动密钥轮换 | `key_rotation` | epoch+1，**所有**成员**必须**重新生成 Sender Key 并重新分发 |

**重要**：无论何种原因触发 epoch 递增，所有成员**必须**生成全新的 `sender_chain_key`（不得复用旧值），并通过 `group_e2ee_key` 向所有其他当前群成员重新分发。这确保：
- 被移除成员无法使用旧密钥解密新 epoch 的消息（前向安全）
- 新加入成员无法使用新密钥解密旧 epoch 的消息（后向安全）

### 8.6 群聊 E2EE content 结构定义

#### 8.6.1 group_e2ee_key — Sender Key 分发

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| group_did | string | 是 | 群组 DID |
| epoch | integer | 是 | 群纪元 |
| sender_did | string | 是 | Sender Key 所有者的 DID |
| sender_key_id | string | 是 | Sender Key 标识 |
| recipient_key_id | string | 是 | 接收方 DID 文档中 keyAgreement 的 id |
| hpke_suite | string | 是 | HPKE 密码套件标识 |
| enc | string | 是 | HPKE 封装密文（Base64 编码） |
| encrypted_sender_key | string | 是 | HPKE AEAD 加密的 sender_chain_key（Base64 编码，含 tag） |
| expires | integer | 否 | Sender Key 有效期（秒），默认 86400 |
| proof | object | 是 | 发送方签名 |

HPKE 加密 `sender_chain_key` 时的 AAD = `group_did + ":" + str(epoch) + ":" + sender_key_id` 的 UTF-8 编码。

**content 示例：**

```json
{
  "group_did": "did:wba:example.com:group:group_dev",
  "epoch": 3,
  "sender_did": "did:wba:example.com:user:alice",
  "sender_key_id": "a1b2c3:3",
  "recipient_key_id": "did:wba:example.com:user:bob#key-x25519-1",
  "hpke_suite": "DHKEM-X25519-HKDF-SHA256/HKDF-SHA256/AES-128-GCM",
  "enc": "dGhpcyBpcyBhIDMyLWJ5dGUgZXBoZW1lcmFsIHB1YmxpYyBrZXk=",
  "encrypted_sender_key": "ZW5jcnlwdGVkIHNlbmRlciBrZXkgZGF0YQ==",
  "expires": 86400,
  "proof": {
    "type": "EcdsaSecp256r1Signature2019",
    "created": "2026-03-01T10:30:00Z",
    "verification_method": "did:wba:example.com:user:alice#keys-1",
    "proof_value": "MEUCIQDx..."
  }
}
```

**send 请求示例：**

```json
{
  "jsonrpc": "2.0",
  "method": "send",
  "params": {
    "sender_did": "did:wba:example.com:user:alice",
    "receiver_did": "did:wba:example.com:user:bob",
    "group_did": "did:wba:example.com:group:group_dev",
    "content": "{\"group_did\":\"did:wba:example.com:group:group_dev\",\"epoch\":3,\"sender_did\":\"did:wba:example.com:user:alice\",\"sender_key_id\":\"a1b2c3:3\",\"recipient_key_id\":\"did:wba:example.com:user:bob#key-x25519-1\",\"hpke_suite\":\"DHKEM-X25519-HKDF-SHA256/HKDF-SHA256/AES-128-GCM\",\"enc\":\"...\",\"encrypted_sender_key\":\"...\",\"expires\":86400,\"proof\":{...}}",
    "type": "group_e2ee_key",
    "client_msg_id": "01961e2c-82e4-70ce-e8c7-d4e5f6071829"
  },
  "id": 1
}
```

#### 8.6.2 group_e2ee_msg — 群密文消息

与 `e2ee_msg` 类似，`group_e2ee_msg` 不包含 `proof` 签名字段。认证性由 `sender_chain_key` 保证——只有持有正确 Sender Key 的一方才能生成有效密文，而 Sender Key 来自经 proof 签名认证的 `group_e2ee_key` 分发消息。

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| group_did | string | 是 | 群组 DID |
| epoch | integer | 是 | 群纪元 |
| sender_did | string | 是 | 发送者 DID |
| sender_key_id | string | 是 | Sender Key 标识 |
| seq | integer | 是 | 该 sender 在当前 epoch 内的消息序号 |
| original_type | string | 是 | 原始消息类型（`text` / `image` / `file`） |
| ciphertext | string | 是 | AES-128-GCM 密文 + tag（Base64 编码） |

**content 示例：**

```json
{
  "group_did": "did:wba:example.com:group:group_dev",
  "epoch": 3,
  "sender_did": "did:wba:example.com:user:alice",
  "sender_key_id": "a1b2c3:3",
  "seq": 0,
  "original_type": "text",
  "ciphertext": "ZW5jcnlwdGVkIGdyb3VwIG1lc3NhZ2U="
}
```

**send 请求示例：**

```json
{
  "jsonrpc": "2.0",
  "method": "send",
  "params": {
    "sender_did": "did:wba:example.com:user:alice",
    "group_did": "did:wba:example.com:group:group_dev",
    "content": "{\"group_did\":\"did:wba:example.com:group:group_dev\",\"epoch\":3,\"sender_did\":\"did:wba:example.com:user:alice\",\"sender_key_id\":\"a1b2c3:3\",\"seq\":0,\"original_type\":\"text\",\"ciphertext\":\"ZW5jcnlwdGVkIGdyb3VwIG1lc3NhZ2U=\"}",
    "type": "group_e2ee_msg",
    "client_msg_id": "01961e2c-93f5-71df-f9d8-e5f607182930"
  },
  "id": 1
}
```

#### 8.6.3 group_epoch_advance — 群纪元变化通知

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| group_did | string | 是 | 群组 DID |
| new_epoch | integer | 是 | 新的 epoch 值 |
| reason | string | 是 | 变更原因：`member_added` / `member_removed` / `key_rotation` |
| members_added | array | 否 | 新加入的成员 DID 列表 |
| members_removed | array | 否 | 被移除的成员 DID 列表 |
| proof | object | 是 | 群管理员签名 |

**content 示例：**

```json
{
  "group_did": "did:wba:example.com:group:group_dev",
  "new_epoch": 4,
  "reason": "member_removed",
  "members_added": [],
  "members_removed": ["did:wba:example.com:user:charlie"],
  "proof": {
    "type": "EcdsaSecp256r1Signature2019",
    "created": "2026-03-01T11:00:00Z",
    "verification_method": "did:wba:example.com:user:admin#keys-1",
    "proof_value": "MEUCIQDx..."
  }
}
```

### 8.7 签名与验证

#### 8.7.1 proof_value 生成过程

需要签名的 E2EE 消息包括：`e2ee_init`、`e2ee_rekey`、`group_e2ee_key`、`group_epoch_advance`。

**生成步骤：**

1. 构造消息的所有字段，`proof` 中的 `proof_value` 字段除外
2. 使用 JCS（JSON Canonicalization Scheme, RFC 8785）对 JSON 进行规范化
3. 将规范化后的 JSON 字符串编码为 UTF-8 字节
4. 使用 ECDSA + SHA-256 算法签名（使用 DID 文档 `authentication` 中的签名密钥）
5. 将签名值 Base64URL 编码后写入 `proof_value` 字段

```python
import json
import base64
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import ec

# 1. 构造消息，排除 proof_value 字段
msg = {
    "session_id": "a1b2c3d4e5f60718a1b2c3d4e5f60718",
    "hpke_suite": "DHKEM-X25519-HKDF-SHA256/HKDF-SHA256/AES-128-GCM",
    # ... 其他字段
    "proof": {
        "type": "EcdsaSecp256r1Signature2019",
        "created": "2026-03-01T10:30:00Z",
        "verification_method": "did:wba:example.com:user:alice#keys-1"
        # proof_value 除外
    }
}

# 2. JCS 规范化（RFC 8785）
# JCS 规范化使用确定性的 JSON 序列化，确保相同对象始终产生相同字节表示
msg_canonical = jcs_canonicalize(msg)  # 按 RFC 8785 规范化

# 3. 编码为 UTF-8 字节
msg_bytes = msg_canonical.encode('utf-8')

# 4. 使用 ECDSA + SHA-256 签名
signature = private_key.sign(msg_bytes, ec.ECDSA(hashes.SHA256()))

# 5. Base64URL 编码
msg["proof"]["proof_value"] = base64.urlsafe_b64encode(signature).decode('utf-8')
```

#### 8.7.2 签名验证过程

1. **提取签名**：从消息中取出 `proof.proof_value`，Base64URL 解码得到签名字节
2. **重构待验证数据**：从消息中去除 `proof.proof_value` 字段，对剩余 JSON 做 JCS 规范化
3. **获取公钥**：根据 `proof.verification_method` 引用的 ID，从发送方 DID 文档中获取对应的签名公钥
4. **验证签名**：使用公钥和 ECDSA + SHA-256 验证签名
5. **验证时间戳**：检查 `proof.created` 时间是否在合理范围内（建议 ±300 秒窗口，即允许最多 5 分钟的时钟偏差）。超出窗口的消息**应**被拒绝。

## 9. 安全性考虑

### 9.1 传输安全

所有 API 端点应使用 HTTPS 协议，确保通信过程中数据的加密传输和完整性。

### 9.2 身份认证安全

身份认证遵循 ANP DID WBA 认证机制。有关基于 DID 的身份认证安全详细说明，请参见 [DID:WBA 方法设计规范](03-did-wba方法规范.md)。已认证用户发送消息时，服务端校验 `sender_did` 与认证身份的一致性。

### 9.3 私聊 E2EE 安全

- **服务端透明转发**：所有 E2EE content 对服务端不透明，服务端无法读取 root_seed 或会话密钥。
- **会话初始化安全**：HPKE Base 模式使用临时 Diffie-Hellman 密钥封装，第三方被动攻击者在不知道接收方 X25519 私钥的情况下，无法从 `enc` 恢复 `root_seed`。但需要注意：**HPKE Base 模式不提供对接收方静态密钥泄露的前向安全保护**——若接收方的 X25519 长期私钥在未来被泄露，攻击者可结合先前截获的 `enc` 和 `encrypted_seed` 恢复 `root_seed`，进而推导该会话的所有消息密钥。

  | 泄露场景 | 能否恢复历史 root_seed | 说明 |
  |---------|---------------------|------|
  | 发送方临时私钥泄露 | 否 | HPKE 临时密钥对仅在 Seal 时使用，不持久化 |
  | 接收方 X25519 静态私钥泄露 | **是** | 可结合截获的 `enc` 执行 HPKE.Open 恢复 root_seed |
  | 双方静态私钥均泄露 | **是** | 同上，接收方私钥即可解密 |

  建议定期轮换 X25519 密钥（更新 DID 文档中的 keyAgreement）以缩小密钥泄露的影响窗口，并通过 `e2ee_rekey` 重建会话密钥。
- **消息级前向安全**：链式 ratchet 确保每条消息使用独立的 msg_key，且 chain_key 单向更新，无法从当前 chain_key 反推已丢弃的旧 chain_key 和 msg_key。在 root_seed 和 chain_key 已被安全删除的前提下，即使后续 chain_key 泄露也无法恢复历史消息。
- **密钥分离**：签名密钥（ECDSA secp256r1）和密钥协商密钥（X25519）物理分离，泄露其中一个不影响另一个的安全性。
- **认证性**：`e2ee_init` / `e2ee_rekey` 携带发送方的 proof 签名，接收方可验证发起者的身份。
- **密钥泄露风险**：由于 `e2ee_msg` 不携带逐条签名（认证性依赖对称会话密钥），若 `chain_key` 或 `root_seed` 泄露，攻击者可伪造该会话内后续的加密消息。因此，`chain_key` **必须**在使用后立即更新并销毁旧值，`root_seed` 在派生 chain_key 后**应**立即销毁。

### 9.4 群聊 E2EE 安全

- **前向安全（成员移除）**：成员被移除后，epoch 递增，所有留在群内的成员重新生成 Sender Keys。被移除成员无法获取新 epoch 的 Sender Keys，无法解密未来消息。
- **后向安全（成员加入）**：新加入的成员仅获得当前 epoch 的 Sender Keys，无法解密历史消息。
- **Sender Key 独立分发**：每个成员的 Sender Key 通过 HPKE 独立封装给每个接收者，单个接收者的密钥泄露不影响其他接收者。
- **epoch 签名**：`group_epoch_advance` 消息携带管理员签名，防止伪造 epoch 变更。
- **Sender Key 泄露风险**：Sender Key 模型下，所有群成员都持有每个发送者的 `sender_chain_key`。若某成员设备被攻破导致其 `sender_chain_key` 泄露，攻击者可伪造该发送者在当前 epoch 内的后续群消息，直到 epoch 轮转。这是 Sender Key 方案的已知限制，可通过缩短 epoch 轮换周期（如定期 `key_rotation`）来缩小影响窗口。

### 9.5 密钥管理安全

- X25519 密钥协商密钥**应**（SHOULD）定期轮换（更新 DID 文档中的 keyAgreement），轮换后**应**主动向活跃会话的对端发送 `e2ee_rekey` 以使用新密钥重建会话。
- 会话密钥（root_seed、sender_chain_key）仅存在于双方内存中，不应持久化到磁盘。
- 链式 ratchet 的 chain_key **必须**在使用后立即更新，旧值**必须**立即销毁。

### 9.6 防重放攻击

- **序号防重放**：`e2ee_msg` 和 `group_e2ee_msg` 的 `seq` 字段严格递增，接收方拒绝任何已使用过的 seq 值的消息（无论严格模式还是窗口模式）。
- **session_id 唯一性**：每次 `e2ee_init` / `e2ee_rekey` 生成新的随机 session_id。
- **epoch + sender_key_id 唯一性**：群聊中 epoch + sender_key_id 组合唯一标识一个 Sender Key 生命周期。
- **proof 时间戳**：签名中的 `created` 时间戳用于检测过期消息。
- **client_msg_id 幂等与 E2EE seq 防重放的互补关系**：`client_msg_id` 在传输层防止 HTTP 重试导致的消息重复投递（由 Message Server 执行去重），而 E2EE `seq` 在端到端层防止攻击者重放已截获的密文消息（由接收端客户端验证）。两者在不同层级独立工作，共同确保消息的唯一性。即使 Message Server 被攻破或绕过了 `client_msg_id` 幂等检查，E2EE `seq` 防重放仍然有效。

## 10. 局限性

### 10.1 群聊 E2EE 的 O(N) 分发成本

Sender Key 分发时需要对每个群成员独立执行 HPKE 封装，群成员数 N 较大时开销显著。未来可考虑 MLS (RFC 9420) 的树形密钥分发方案优化。

### 10.2 单设备限制

在本规范版本中，一个 DID **不得**同时在多个设备上活跃使用 E2EE 功能。多设备并发会导致 ratchet 状态冲突、群聊 Sender Key 混乱和会话状态丢失。多设备密钥同步机制将在未来版本中定义。

### 10.3 大文件传输效率

消息内容通过 JSON 的 `content` 字段传输，对于大文件（如视频）的传输效率不高。建议在 `content` 中传递文件的 URL 和解密密钥，接收者通过 HTTPS 等协议单独下载文件。

## 11. 总结与展望

本规范定义了基于 HTTP + JSON-RPC 2.0 的即时消息协议，以 `did:wba` 为身份标识，支持私聊和群聊的端到端加密通信。E2EE 方案基于 HPKE (RFC 9180)，采用密钥分离设计，私聊通过一步 HPKE 初始化 + 链式 ratchet 实现会话加密，群聊通过 Sender Keys + epoch 轮转实现群体加密。

通过与智能体描述协议的集成，智能体可以以标准化方式声明其 E2EE IM 能力，实现自动发现和互操作。

未来的工作方向包括：

1. **MLS 协议集成**：探索 MLS (RFC 9420) 的树形密钥管理方案，优化大群组的 Sender Key 分发效率
2. **多设备密钥同步**：定义同一 DID 跨多个设备的密钥同步和会话管理机制
3. **离线消息**：完善离线消息的推送和同步机制
4. **消息类型扩展**：支持更丰富的消息类型（如音视频通话信令）
5. **跨服务器互通**：定义 Message Server 之间的联邦协议

## 参考文献

- [RFC 9180] Hybrid Public Key Encryption (HPKE)
- [RFC 8785] JSON Canonicalization Scheme (JCS)
- [RFC 7159] The JavaScript Object Notation (JSON) Data Interchange Format
- [JSON-RPC 2.0 Specification](https://www.jsonrpc.org/specification)
- [W3C DID Core Specification](https://www.w3.org/TR/did-core/)
- [ANP 技术白皮书](01-AgentNetworkProtocol技术白皮书.md)
- [DID:WBA 方法设计规范](03-did-wba方法规范.md)
- [智能体描述协议规范](07-ANP-智能体描述协议规范.md)

## 版权声明

Copyright (c) 2024 GaoWei Chang
本文件依据 [MIT 许可证](LICENSE) 发布，您可以自由使用和修改，但必须保留本版权声明。
