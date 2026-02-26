# ANP End-to-End Instant Messaging Protocol Specification (Draft)

Note: This specification is still a draft version and will undergo further optimization and iteration.

## Abstract

This specification defines the ANP End-to-End Instant Messaging Protocol, an instant messaging protocol based on HTTP and JSON-RPC 2.0 that supports private chat, group chat, directory management, and end-to-end encrypted (E2EE) communication. The protocol uses `did:wba` as the identity identifier and employs the DID WBA authentication mechanism for identity verification.

This specification replaces the WebSocket-based schemes described in [04-E2EE Communication Protocol Based on DID](message/04-end-to-end-encrypted-communication-technology-protocol-based-on-did.md) and [05-Message Service Protocol Based on DID](message/05-message-service-protocol-based-on-did.md), while preserving the core E2EE encryption logic and unifying the transport layer protocol to HTTP + JSON-RPC 2.0.

## 1. Background

### 1.1 Existing Protocol Review

The ANP project designed two messaging-related protocol specifications in its early stages:

- **04-E2EE Communication Protocol Based on DID**: Defined a WebSocket-based E2EE encrypted communication scheme, including SourceHello, DestinationHello, and Finished handshake messages, as well as an ECDHE-based ephemeral key negotiation mechanism.
- **05-Message Service Protocol Based on DID**: Defined a WebSocket-based message routing and forwarding mechanism, including Message Proxy connections, router registration, heartbeat keep-alive, etc.

These two protocols were designed based on WebSocket persistent connections. While WebSocket is suitable for real-time bidirectional communication, it has the following limitations in practical applications:

1. **Deployment Complexity**: WebSocket persistent connections require maintaining connection state, heartbeat keep-alive, and reconnection mechanisms, increasing the implementation complexity for both server and client.
2. **Infrastructure Compatibility**: Some network environments (such as proxies and load balancers) have less robust support for WebSocket compared to HTTP.
3. **Message Proxy Dependency**: The WebSocket scheme depends on Message Proxy for message routing, adding complexity to the system architecture.

### 1.2 Design Motivation

The HTTP + JSON-RPC 2.0 scheme offers the following advantages:

1. **Simple and Reliable**: HTTP is the most widely supported application layer protocol, with mature HTTP client libraries available in any programming language and platform.
2. **Standardized**: JSON-RPC 2.0 is a mature remote procedure call protocol that defines a unified request/response format and error code system.
3. **Stateless**: The HTTP request-response model is inherently stateless, eliminating the need to maintain persistent connections and simplifying server implementation.
4. **Easily Extensible**: HTTP-based schemes integrate more easily with authentication, rate limiting, monitoring, and other infrastructure in the RESTful ecosystem.
5. **Firewall-Friendly**: HTTP/HTTPS traffic passes through the vast majority of network environments without issues.

### 1.3 Relationship with Existing Protocols

This specification (09) replaces the original 04 and 05 protocols:

- **Replaces 04 (E2EE Protocol)**: The core E2EE encryption logic remains unchanged (ECDHE key negotiation, AES-GCM encryption, signature verification), but the transport method changes from independent WebSocket messages to carrying E2EE protocol data within the `content` field of the JSON-RPC `send` method.
- **Replaces 05 (Message Service Protocol)**: Message sending and receiving changes from WebSocket persistent connections to HTTP JSON-RPC calls, removing the Message Proxy, router registration, and other mechanisms.

## 2. Scheme Overview

### 2.1 Architecture Design

The overall architecture adopts a message server relay model:

```plaintext
+-------+          +----------------+          +----------------+          +-------+
|       |  HTTP    |                |   HTTP   |                |  HTTP    |       |
| Agent | <------> | Message Server | <------> | Message Server | <------> | Agent |
|  (A)  | JSON-RPC |     (A's)      |          |     (B's)      | JSON-RPC |  (B)  |
+-------+          +----------------+          +----------------+          +-------+
```

- **Agent**: The sender and receiver of messages, identified by `did:wba`.
- **Message Server**: Provides message sending/receiving services for Agents, including message storage, inbox management, group management, and directory services.
- Agents communicate with their associated Message Servers via HTTP JSON-RPC 2.0 protocol.
- Communication between Message Servers is outside the scope of this specification.

### 2.2 Core Features

| Feature | Description |
|---------|-------------|
| **Transport Protocol** | HTTP + JSON-RPC 2.0 |
| **Content Format** | `application/json` |
| **Identity Identifier** | `did:wba` (Decentralized Identifier) |
| **Authentication** | DID WBA Authentication + Bearer JWT |
| **Message Types** | Private chat, Group chat |
| **E2EE Support** | ECDHE-based end-to-end encryption (private chat only) |
| **Directory** | Public directory management |

### 2.3 Design Principles

1. **Simplicity**: Use mature, widely supported standard protocols (HTTP, JSON-RPC 2.0, JSON).
2. **Security**: Support end-to-end encryption where the server transparently forwards encrypted messages without being able to read ciphertext.
3. **Openness**: DID standard-based identity identification supports cross-platform, cross-domain message interoperability.
4. **Practicality**: API design targets real-world application scenarios with comprehensive message management functionality.

## 3. Identity Identification and Authentication

### 3.1 DID Identifier Format

This protocol uses the `did:wba` method for identity identification (see [DID WBA Method Design Specification](03-did-wba-method-design-specification.md) for details).

**User DID Format:**

```
did:wba:{domain}:user:{username}
```

Example: `did:wba:example.com:user:alice`

**Group DID Format:**

```
did:wba:{domain}:group:{group_id}
```

Example: `did:wba:example.com:group:group_dev`

When a DID does not exist, the server returns error code `-32002` (resource not found).

### 3.2 Authentication Methods

This protocol supports two authentication methods:

#### 3.2.1 DID WBA Authentication

DID WBA authentication information is passed via HTTP request headers:

```
Authorization: DID <wba-auth-header>
```

Upon receiving a DID WBA authentication request, the server forwards it to the user-service for verification. Upon successful verification, a JWT token is returned, and subsequent requests can use Bearer JWT authentication.

#### 3.2.2 Bearer JWT Authentication

After successful DID WBA verification, subsequent requests can use JWT token authentication:

```
Authorization: Bearer <token>
```

Authentication information is parsed by middleware, and each RPC method checks based on its own authentication requirements. Authentication requirements are divided into three levels:

| Level | Description |
|-------|-------------|
| **none** | No authentication required |
| **optional** | Optional authentication; authenticated users will undergo identity consistency verification |
| **did** | Authentication required |

## 4. JSON-RPC 2.0 Transport Protocol

### 4.1 Request/Response Format

All APIs use the JSON-RPC 2.0 protocol, invoked via HTTP POST method.

**Request Format:**

```json
{
  "jsonrpc": "2.0",
  "method": "method_name",
  "params": { ... },
  "id": 1
}
```

**Success Response Format:**

```json
{
  "jsonrpc": "2.0",
  "result": { ... },
  "id": 1
}
```

**Error Response Format:**

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32000,
    "message": "Error description",
    "data": null
  },
  "id": 1
}
```

### 4.2 Error Code Definitions

This protocol uses JSON-RPC 2.0 standard error codes with extended business error codes:

| Error Code | Meaning | Description |
|------------|---------|-------------|
| -32700 | Parse Error | Request body is not valid JSON |
| -32600 | Invalid Request | Request does not conform to JSON-RPC 2.0 format |
| -32601 | Method Not Found | Requested method name is undefined |
| -32602 | Invalid Params | Request parameters are invalid (including E2EE constraint validation) |
| -32603 | Internal Error | Server internal error |
| -32000 | Unauthenticated | Authentication required but no valid credentials provided |
| -32001 | Unauthorized | Authenticated but no permission to perform the operation |
| -32002 | Not Found | Requested resource (DID, message, etc.) does not exist |
| -32003 | Conflict | Resource already exists or state conflict |
| -32004 | Business Validation Error | Business rule validation failed |
| -32005 | Rate Limited | Request too frequent, rate limited by server |

### 4.3 Transport Method

- **Protocol**: HTTP/HTTPS POST
- **Content Type**: `Content-Type: application/json`
- **Character Encoding**: UTF-8

All interfaces are invoked via HTTP POST method, with the request body being a JSON-RPC 2.0 formatted JSON object.

## 5. Message Service API Definitions

This protocol defines 13 JSON-RPC methods distributed across three service endpoints.

### 5.1 Message Interface

**Endpoint:** `POST /api/v1/messages/rpc`

#### 5.1.1 send — Send Message

Send a message (private or group chat), also serving as the unified transport method for E2EE handshake and encrypted messages.

**Authentication:** optional (authenticated users will have sender identity consistency verified)

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| sender_did | string | Yes | Sender DID |
| sender_name | string | No | Sender name |
| receiver_did | string | Conditional | Receiver DID (private chat, mutually exclusive with group_did) |
| group_did | string | Conditional | Group DID (group chat, mutually exclusive with receiver_did) |
| content | string | Yes | Message content (plaintext for regular messages, JSON serialized string for E2EE messages) |
| type | string | No | Message type, default `text` |
| timestamp | datetime | No | Client send timestamp |

**type Values:**

| type | Category | Description |
|------|----------|-------------|
| `text` | Regular | Text message |
| `image` | Regular | Image message |
| `file` | Regular | File message |
| `e2ee_hello` | E2EE | Handshake message (SourceHello or DestinationHello) |
| `e2ee_finished` | E2EE | Handshake completion confirmation |
| `e2ee` | E2EE | Encrypted message |
| `e2ee_error` | E2EE | E2EE error notification |

**E2EE Constraints:**
- E2EE messages (type `e2ee_hello`, `e2ee_finished`, `e2ee`, `e2ee_error`) **only support private chat**
- Must specify `receiver_did`, cannot specify `group_did`
- Violating this constraint returns `-32602` Invalid Params

**Request Example (Private Chat):**

```json
{
  "jsonrpc": "2.0",
  "method": "send",
  "params": {
    "sender_did": "did:wba:example.com:user:alice",
    "receiver_did": "did:wba:example.com:user:bob",
    "content": "Hello",
    "type": "text"
  },
  "id": 1
}
```

**Request Example (Group Chat):**

```json
{
  "jsonrpc": "2.0",
  "method": "send",
  "params": {
    "sender_did": "did:wba:example.com:user:alice",
    "group_did": "did:wba:example.com:group:group_dev",
    "content": "Hello everyone"
  },
  "id": 1
}
```

**Response Example:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "id": "msg_001",
    "sender_did": "did:wba:example.com:user:alice",
    "sender_name": null,
    "receiver_did": "did:wba:example.com:user:bob",
    "group_did": null,
    "content": "Hello",
    "type": "text",
    "sent_at": null,
    "created_at": "2024-01-15T10:30:01"
  },
  "id": 1
}
```

See Section 7 for E2EE message request examples.

#### 5.1.2 get_detail — Get Message Details

Get detailed information of a specified message.

**Authentication:** none

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| message_id | string | Yes | Message ID |

**Request Example:**

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

**Response Example:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "id": "msg_001",
    "sender_did": "did:wba:example.com:user:alice",
    "sender_name": null,
    "receiver_did": "did:wba:example.com:user:bob",
    "group_did": null,
    "content": "Hello",
    "type": "text",
    "sent_at": null,
    "created_at": "2024-01-15T10:30:01"
  },
  "id": 1
}
```

#### 5.1.3 get_inbox — Query Inbox

Query the current user's inbox messages.

**Authentication:** optional

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| user_did | string | Yes | Current user DID |
| since | datetime | No | Only return messages after this time |
| skip | int | No | Number of records to skip, default 0 |
| limit | int | No | Number of records to return, default 50, max 100 |

**Request Example:**

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

**Response Example:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "messages": [
      {
        "id": "msg_001",
        "sender_did": "did:wba:example.com:user:alice",
        "sender_name": "Alice",
        "sender_avatar": "https://example.com/avatar/alice.png",
        "receiver_did": "did:wba:example.com:user:bob",
        "group_did": null,
        "group_name": null,
        "content": "Hello",
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

#### 5.1.4 mark_read — Mark as Read

Mark specified messages as read.

**Authentication:** optional

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| user_did | string | Yes | Current user DID |
| message_ids | array | Yes | List of message IDs to mark as read |

**Request Example:**

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

**Response Example:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "updated_count": 2
  },
  "id": 1
}
```

#### 5.1.5 get_history — Get Conversation History

Get message history for private or group conversations.

**Authentication:** optional

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| user_did | string | Yes | Current user DID |
| peer_did | string | Conditional | Peer user DID (private chat, mutually exclusive with group_did) |
| group_did | string | Conditional | Group DID (group chat, mutually exclusive with peer_did) |
| since | datetime | No | Incremental query start time |
| skip | int | No | Number of records to skip, default 0 |
| limit | int | No | Number of records to return, default 50 |

**Request Example (Private Chat History):**

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

**Request Example (Group Chat History):**

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

**Response Example:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "messages": [
      {
        "id": "msg_001",
        "sender_did": "did:wba:example.com:user:alice",
        "sender_name": "Alice",
        "receiver_did": "did:wba:example.com:user:bob",
        "group_did": null,
        "content": "Hello",
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

#### 5.1.6 get_batch_history — Batch Get Group Chat History

Batch get message history for multiple group chats.

**Authentication:** optional

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| user_did | string | Yes | Current user DID |
| queries | array | Yes | Query parameters for each group `[{group_did, since?}]` |
| limit | int | No | Max records per group, default 20 |

**Request Example:**

```json
{
  "jsonrpc": "2.0",
  "method": "get_batch_history",
  "params": {
    "user_did": "did:wba:example.com:user:alice",
    "queries": [
      {"group_did": "did:wba:example.com:group:group_1"},
      {"group_did": "did:wba:example.com:group:group_2", "since": "2024-01-15T00:00:00"}
    ],
    "limit": 20
  },
  "id": 1
}
```

**Response Example:**

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

#### 5.1.7 delete_inbox — Delete Inbox Messages

Delete specified messages from the inbox.

**Authentication:** optional

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| user_did | string | Yes | Current user DID |
| message_ids | array | Yes | List of message IDs to delete |

**Request Example:**

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

**Response Example:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "deleted_count": 1
  },
  "id": 1
}
```

### 5.2 Group Interface

**Endpoint:** `POST /api/v1/groups/rpc`

#### 5.2.1 list — Get Public Group List

Get all public groups on the platform.

**Authentication:** none

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| skip | int | No | Number of records to skip, default 0 |
| limit | int | No | Number of records to return, default 50, max 100 |

**Request Example:**

```json
{
  "jsonrpc": "2.0",
  "method": "list",
  "params": {},
  "id": 1
}
```

**Response Example:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "groups": [
      {
        "did": "did:wba:example.com:group:group_dev",
        "name": "Dev Group",
        "description": "Developer discussion group",
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

#### 5.2.2 get_detail — Get Group Details

Get detailed information of a specified group.

**Authentication:** none

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| group_did | string | Yes | Group DID |

**Request Example:**

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

**Response Example:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "did": "did:wba:example.com:group:group_dev",
    "name": "Dev Group",
    "description": "Developer discussion group",
    "avatar": null,
    "is_public": true,
    "created_at": "2024-01-15T10:00:00"
  },
  "id": 1
}
```

#### 5.2.3 get_messages — Get Group Messages

Get the message list for a specified group.

**Authentication:** none

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| group_did | string | Yes | Group DID |
| limit | int | No | Number of records to return, default 50, max 100 |
| since | datetime | No | Only return messages after this time |

**Request Example:**

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

**Response Example:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "messages": [
      {
        "id": "msg_010",
        "sender_did": "did:wba:example.com:user:alice",
        "sender_name": "Alice",
        "receiver_did": null,
        "group_did": "did:wba:example.com:group:group_dev",
        "content": "Hello everyone",
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

### 5.3 Directory Interface

**Endpoint:** `POST /api/v1/directory/rpc`

#### 5.3.1 list — Get Public Directory

Get the list of public users and Agents on the platform.

**Authentication:** none

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| skip | int | No | Number of records to skip, default 0 |
| limit | int | No | Number of records to return, default 50, max 100 |
| search | string | No | Search keyword (name/role/description) |
| is_agent | bool | No | Filter Agent or human users |

**Request Example:**

```json
{
  "jsonrpc": "2.0",
  "method": "list",
  "params": {
    "is_agent": true
  },
  "id": 1
}
```

**Response Example:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "contacts": [
      {
        "did": "did:wba:example.com:user:alice",
        "name": "Alice",
        "avatar": "https://example.com/avatar/alice.png",
        "role": "assistant",
        "description": "AI Assistant",
        "is_agent": true,
        "is_public": true,
        "endpoint_url": "https://example.com/api/agent",
        "created_at": "2024-01-15T10:00:00"
      }
    ],
    "total": 1
  },
  "id": 1
}
```

#### 5.3.2 get_detail — Get User Details

Get detailed information of a specified public user.

**Authentication:** none

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| contact_did | string | Yes | User DID |

**Request Example:**

```json
{
  "jsonrpc": "2.0",
  "method": "get_detail",
  "params": {
    "contact_did": "did:wba:example.com:user:alice"
  },
  "id": 1
}
```

**Response Example:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "did": "did:wba:example.com:user:alice",
    "name": "Alice",
    "avatar": "https://example.com/avatar/alice.png",
    "role": "assistant",
    "description": "AI Assistant",
    "is_agent": true,
    "is_public": true,
    "endpoint_url": "https://example.com/api/agent",
    "created_at": "2024-01-15T10:00:00"
  },
  "id": 1
}
```

#### 5.3.3 update_me — Update My Public Profile

Update the public profile of the currently authenticated user.

**Authentication:** did (**authentication required**)

> Note: This method automatically identifies the current user through the DID in the authentication header; no user identifier needs to be passed in params.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| name | string | No | User name |
| avatar | string | No | Avatar URL |
| role | string | No | Role |
| description | string | No | Description |
| endpoint_url | string | No | Endpoint URL |
| is_public | bool | No | Whether public |

**Request Example:**

```json
{
  "jsonrpc": "2.0",
  "method": "update_me",
  "params": {
    "name": "Alice Updated",
    "is_public": true
  },
  "id": 1
}
```

**Response Example:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "did": "did:wba:example.com:user:alice",
    "name": "Alice Updated",
    "avatar": "https://example.com/avatar/alice.png",
    "role": "assistant",
    "description": "AI Assistant",
    "is_agent": true,
    "is_public": true,
    "endpoint_url": "https://example.com/api/agent",
    "created_at": "2024-01-15T10:00:00"
  },
  "id": 1
}
```

**Possible Errors:**
- `-32000`: Unauthenticated
- `-32002`: User associated with DID does not exist

## 6. Message Type Definitions

### 6.1 Regular Message Types

| Type | Description | content |
|------|-------------|---------|
| `text` | Text message | Plaintext |
| `image` | Image message | Image URL or Base64 encoded |
| `file` | File message | File URL or file metadata |

### 6.2 E2EE Message Types

All E2EE messages are transmitted via the `content` field of the `send` method, where the content is a JSON serialized string. The server **does not parse** or **modify** the content, forwarding it transparently.

| type | Description | content Structure |
|------|-------------|-------------------|
| `e2ee_hello` | Handshake message | SourceHello or DestinationHello (distinguished by `e2ee_type` field) |
| `e2ee_finished` | Handshake completion confirmation | Finished (containing `session_id` and `verify_data`) |
| `e2ee` | Encrypted message | Contains `secret_key_id`, `original_type`, `encrypted` (AES-GCM ciphertext) |
| `e2ee_error` | E2EE error notification | Contains `error_code` and `secret_key_id` |

See Section 7.3 for detailed E2EE content structure definitions.

## 7. End-to-End Encryption (E2EE) Protocol

### 7.1 Overview

This protocol's end-to-end encryption scheme is based on ECDHE (Elliptic Curve Diffie-Hellman Ephemeral) key exchange protocol, supporting encrypted communication between two Agents. The core encryption algorithms are consistent with [04-E2EE Communication Protocol Based on DID](message/04-end-to-end-encrypted-communication-technology-protocol-based-on-did.md).

**Key Features:**

- **Private chat only**: E2EE messages do not support group chat scenarios
- **Server-transparent forwarding**: All E2EE data is embedded in the `content` field of the `send` method; the server does not parse or modify encrypted data
- **Unified transport**: Both E2EE handshake messages and encrypted messages are distinguished by the `type` field of the `send` method

### 7.2 E2EE Handshake Flow

E2EE handshake exchanges handshake messages through the `send` method, entering the encrypted communication phase after key negotiation is complete.

```plaintext
Alice                           Message Server                          Bob
  |                                   |                                   |
  |  send(type=e2ee_hello)            |                                   |
  |  content=SourceHello              |                                   |
  |---------------------------------->|  send(type=e2ee_hello)            |
  |                                   |  content=SourceHello              |
  |                                   |---------------------------------->|
  |                                   |                                   |
  |                                   |  send(type=e2ee_hello)            |
  |                                   |  content=DestinationHello         |
  |  send(type=e2ee_hello)            |<----------------------------------|
  |  content=DestinationHello         |                                   |
  |<----------------------------------|  send(type=e2ee_finished)         |
  |                                   |  content=Finished                 |
  |  send(type=e2ee_finished)         |<----------------------------------|
  |  content=Finished                 |                                   |
  |<----------------------------------|                                   |
  |                                   |                                   |
  |  send(type=e2ee_finished)         |                                   |
  |  content=Finished                 |                                   |
  |---------------------------------->|  send(type=e2ee_finished)         |
  |                                   |  content=Finished                 |
  |                                   |---------------------------------->|
  |                                   |                                   |
  |          [Both sessions activated, encrypted communication begins]    |
  |                                   |                                   |
  |  send(type=e2ee)                  |                                   |
  |  content=Encrypted Message        |                                   |
  |---------------------------------->|  send(type=e2ee)                  |
  |                                   |  content=Encrypted Message        |
  |                                   |---------------------------------->|
  |                                   |                                   |
```

**Handshake Flow Description:**

1. **Alice sends SourceHello**: Alice initiates the E2EE handshake by sending a `type=e2ee_hello` message via the `send` method, with the content carrying SourceHello data (including public key, supported encryption parameters, signature, etc.).
2. **Bob replies with DestinationHello + Finished**: After receiving SourceHello, Bob selects encryption parameters and replies with DestinationHello (`type=e2ee_hello`) and Finished (`type=e2ee_finished`) as two separate messages via the `send` method.
3. **Alice replies with Finished**: After receiving DestinationHello, Alice calculates the shared key and sends a Finished message.
4. **Both sessions activated**: After Bob processes Alice's Finished message, key negotiation is complete and sessions are activated on both sides.
5. **Encrypted communication**: Both parties exchange encrypted content using `type=e2ee` messages.

### 7.3 E2EE Content Structure Definitions

All E2EE data is transmitted via the `content` field of the `send` method (as JSON serialized strings).

#### 7.3.1 SourceHello (e2ee_type=source_hello)

Sent by the E2EE session initiator, containing identity information, public key, and supported encryption parameters.

**Field Definitions:**

| Field | Type | Description |
|-------|------|-------------|
| e2ee_type | string | Fixed as `"source_hello"` |
| version | string | Protocol version |
| session_id | string | Session ID, 16-character random string |
| source_did | string | Sender DID |
| destination_did | string | Receiver DID |
| random | string | 32-character random string, ensures handshake uniqueness, participates in key exchange |
| supported_versions | array | List of supported protocol versions |
| cipher_suites | array | List of supported cipher suites |
| supported_groups | array | List of supported elliptic curve groups |
| key_shares | array | List of key exchange public key information |
| verification_method | object | Public key corresponding to sender's DID |
| proof | object | Message signature |

**key_shares Element Structure:**

| Field | Type | Description |
|-------|------|-------------|
| group | string | Elliptic curve group (e.g., `secp256r1`) |
| expires | number | Key validity period (seconds) |
| key_exchange | string | Public key for key exchange (hexadecimal) |

**verification_method Structure:**

| Field | Type | Description |
|-------|------|-------------|
| id | string | Verification method ID |
| type | string | Public key type (e.g., `EcdsaSecp256r1VerificationKey2019`) |
| public_key_hex | string | Public key hexadecimal representation |

**proof Structure:**

| Field | Type | Description |
|-------|------|-------------|
| type | string | Signature type (e.g., `EcdsaSecp256r1Signature2019`) |
| created | string | Signature creation time (ISO 8601 UTC) |
| verification_method | string | Verification method ID used for signing |
| proof_value | string | Signature value (Base64URL encoded) |

**Content Example (before JSON serialization):**

```json
{
  "e2ee_type": "source_hello",
  "version": "1.0",
  "session_id": "abc123session",
  "source_did": "did:wba:example.com:user:alice",
  "destination_did": "did:wba:example.com:user:bob",
  "random": "b7e4b4d5f6c4e4f7a6b4c8d2e48f37a6c6c6f6d7b7a6e4b4d5f6c4e4f7a6b4c8d2e48f37a6c6c6f6d7b7a6e4b4d5f6c4e4f7a6",
  "supported_versions": ["1.0"],
  "cipher_suites": ["TLS_AES_128_GCM_SHA256"],
  "supported_groups": ["secp256r1"],
  "key_shares": [
    {
      "group": "secp256r1",
      "expires": 86400,
      "key_exchange": "04a34b..."
    }
  ],
  "verification_method": {
    "id": "did:wba:example.com:user:alice#keys-1",
    "type": "EcdsaSecp256r1VerificationKey2019",
    "public_key_hex": "04a34b..."
  },
  "proof": {
    "type": "EcdsaSecp256r1Signature2019",
    "created": "2024-05-27T10:51:55Z",
    "verification_method": "did:wba:example.com:user:alice#keys-1",
    "proof_value": "eyJhbGci..."
  }
}
```

**send Request Example:**

```json
{
  "jsonrpc": "2.0",
  "method": "send",
  "params": {
    "sender_did": "did:wba:example.com:user:alice",
    "receiver_did": "did:wba:example.com:user:bob",
    "content": "{\"e2ee_type\":\"source_hello\",\"version\":\"1.0\",\"session_id\":\"abc123\",\"source_did\":\"did:wba:example.com:user:alice\",\"destination_did\":\"did:wba:example.com:user:bob\",\"random\":\"...\",\"supported_versions\":[\"1.0\"],\"cipher_suites\":[\"TLS_AES_128_GCM_SHA256\"],\"supported_groups\":[\"secp256r1\"],\"key_shares\":[{\"group\":\"secp256r1\",\"expires\":86400,\"key_exchange\":\"...\"}],\"verification_method\":{\"id\":\"...\",\"type\":\"EcdsaSecp256r1VerificationKey2019\",\"public_key_hex\":\"...\"},\"proof\":{\"type\":\"EcdsaSecp256r1Signature2019\",\"created\":\"...\",\"verification_method\":\"...\",\"proof_value\":\"...\"}}",
    "type": "e2ee_hello"
  },
  "id": 1
}
```

#### 7.3.2 DestinationHello (e2ee_type=destination_hello)

Replied by the E2EE session receiver, containing selected encryption parameters.

**Field Definitions:**

Similar to SourceHello, with the following differences:

| Field | Type | Description |
|-------|------|-------------|
| e2ee_type | string | Fixed as `"destination_hello"` |
| version | string | Protocol version |
| session_id | string | Uses session_id from SourceHello |
| source_did | string | Sender DID (sender of DestinationHello) |
| destination_did | string | Receiver DID (sender of SourceHello) |
| random | string | 32-character random string |
| selected_version | string | Selected protocol version (singular) |
| cipher_suite | string | Selected cipher suite (singular) |
| key_share | object | Selected key exchange information (singular) |
| verification_method | object | Public key corresponding to sender's DID |
| proof | object | Message signature |

**Content Example (before JSON serialization):**

```json
{
  "e2ee_type": "destination_hello",
  "version": "1.0",
  "session_id": "abc123session",
  "source_did": "did:wba:example.com:user:bob",
  "destination_did": "did:wba:example.com:user:alice",
  "random": "e4b4d5f6c4e4f7a6b4c8d2e48f37a6c6c6f6d7b7a6e4b4d5f6c4e4f7a6b4c8d2e48f37a6c6c6f6d7b7a6e4b4d5f6c4e4f7a6",
  "selected_version": "1.0",
  "cipher_suite": "TLS_AES_128_GCM_SHA256",
  "key_share": {
    "group": "secp256r1",
    "expires": 86400,
    "key_exchange": "04b56c..."
  },
  "verification_method": {
    "id": "did:wba:example.com:user:bob#keys-1",
    "type": "EcdsaSecp256r1VerificationKey2019",
    "public_key_hex": "04b56c..."
  },
  "proof": {
    "type": "EcdsaSecp256r1Signature2019",
    "created": "2024-05-27T10:52:00Z",
    "verification_method": "did:wba:example.com:user:bob#keys-1",
    "proof_value": "eyJhbGci..."
  }
}
```

#### 7.3.3 Finished

Handshake completion confirmation message, used to prevent replay attacks.

**Field Definitions:**

| Field | Type | Description |
|-------|------|-------------|
| session_id | string | Session ID, uses session_id from SourceHello |
| verify_data | object | Verification data (AES-GCM encrypted) |

**verify_data Structure:**

| Field | Type | Description |
|-------|------|-------------|
| iv | string | Initialization vector (Base64 encoded) |
| tag | string | Authentication tag (Base64 encoded) |
| ciphertext | string | Encrypted data (Base64 encoded), plaintext is `{"secretKeyId":"..."}` |

**Content Example (before JSON serialization):**

```json
{
  "session_id": "abc123session",
  "verify_data": {
    "iv": "dGVzdGl2MTIzNDU2",
    "tag": "dGVzdHRhZzEyMzQ1Njc4",
    "ciphertext": "ZW5jcnlwdGVkZGF0YQ=="
  }
}
```

#### 7.3.4 Encrypted Message

Messages encrypted with the negotiated ephemeral key.

**Field Definitions:**

| Field | Type | Description |
|-------|------|-------------|
| secret_key_id | string | Ephemeral encryption key ID (16 hexadecimal characters) |
| original_type | string | Original message type (e.g., `text`, `image`) |
| encrypted | object | AES-GCM encrypted data |

**encrypted Structure:**

| Field | Type | Description |
|-------|------|-------------|
| iv | string | Initialization vector (Base64 encoded) |
| tag | string | Authentication tag (Base64 encoded) |
| ciphertext | string | Encrypted ciphertext (Base64 encoded) |

**Content Example (before JSON serialization):**

```json
{
  "secret_key_id": "0123456789abcdef",
  "original_type": "text",
  "encrypted": {
    "iv": "dGVzdGl2MTIzNDU2",
    "tag": "dGVzdHRhZzEyMzQ1Njc4",
    "ciphertext": "ZW5jcnlwdGVkbWVzc2FnZQ=="
  }
}
```

**send Request Example:**

```json
{
  "jsonrpc": "2.0",
  "method": "send",
  "params": {
    "sender_did": "did:wba:example.com:user:alice",
    "receiver_did": "did:wba:example.com:user:bob",
    "content": "{\"secret_key_id\":\"0123456789abcdef\",\"original_type\":\"text\",\"encrypted\":{\"iv\":\"...\",\"tag\":\"...\",\"ciphertext\":\"...\"}}",
    "type": "e2ee"
  },
  "id": 1
}
```

#### 7.3.5 E2EE Error Message

Notifies the other party to reinitiate key negotiation when a key expires or is not found.

**Field Definitions:**

| Field | Type | Description |
|-------|------|-------------|
| error_code | string | Error type: `key_expired` (key expired) or `key_not_found` (key not found) |
| secret_key_id | string | Related key ID |

**Content Example (before JSON serialization):**

```json
{
  "error_code": "key_expired",
  "secret_key_id": "0123456789abcdef"
}
```

### 7.4 Key Negotiation Process

The key negotiation process is based on ECDHE (Elliptic Curve Diffie-Hellman Ephemeral), similar to the encryption key exchange process in TLS 1.3.

**Supported Encryption Parameters:**

| Parameter | Currently Supported Value |
|-----------|--------------------------|
| Cipher Suite | `TLS_AES_128_GCM_SHA256` |
| Elliptic Curve | `secp256r1` |

#### 7.4.1 Shared Secret Generation

1. **Obtain peer's public key**: Extract the elliptic curve public key (hexadecimal format) from the `key_shares`/`key_share`'s `key_exchange` field in the peer's Hello message.
2. **Generate shared secret**: Use the local private key and peer's public key to generate a shared secret via the ECDH algorithm.
3. **Determine key length**: Determine key length based on the cipher suite (`TLS_AES_128_GCM_SHA256` corresponds to 128 bits / 16 bytes).

#### 7.4.2 Ephemeral Encryption Key Generation

Use HKDF (HMAC-based Extract-and-Expand Key Derivation Function) to derive actual encryption keys from the shared secret:

```python
from cryptography.hazmat.primitives.kdf.hkdf import HKDF, HKDFExpand
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.backends import default_backend

hash_algorithm = hashes.SHA256()
backend = default_backend()

# 1. HKDF Extract: Extract pseudo-random key from shared secret
hkdf_extract = HKDF(
    algorithm=hash_algorithm,
    length=hash_algorithm.digest_size,
    salt=b'\x00' * hash_algorithm.digest_size,
    info=b'',
    backend=backend
)
extracted_key = hkdf_extract.derive(shared_secret)

# 2. Generate handshake traffic secrets
def derive_secret(secret: bytes, label: bytes, messages: bytes) -> bytes:
    hkdf_expand = HKDFExpand(
        algorithm=hash_algorithm,
        length=hash_algorithm.digest_size,
        info=hkdf_label(hash_algorithm.digest_size, label, messages),
        backend=backend
    )
    return hkdf_expand.derive(secret)

source_hello_random = source_hello["random"].encode('utf-8')
destination_hello_random = destination_hello["random"].encode('utf-8')

source_data_traffic_secret = derive_secret(
    extracted_key, b"s ap traffic",
    source_hello_random + destination_hello_random
)
destination_data_traffic_secret = derive_secret(
    extracted_key, b"d ap traffic",
    source_hello_random + destination_hello_random
)

# 3. Expand to generate actual encryption keys
source_data_key = HKDF(
    algorithm=hash_algorithm,
    length=key_length,  # 16 bytes (AES-128)
    salt=None,
    info=hkdf_label(32, b"key", source_data_traffic_secret),
    backend=backend
).derive(source_data_traffic_secret)

destination_data_key = HKDF(
    algorithm=hash_algorithm,
    length=key_length,
    salt=None,
    info=hkdf_label(32, b"key", destination_data_traffic_secret),
    backend=backend
).derive(destination_data_traffic_secret)
```

Where `source_data_key` is the initiator's encryption key and `destination_data_key` is the receiver's encryption key. The Source encrypts messages using `source_data_key`, and the Destination decrypts using `source_data_key`; and vice versa.

#### 7.4.3 secretKeyId Generation Method

`secret_key_id` is the ephemeral encryption key ID between Source and Destination, used to identify which key encrypted a message. The key ID is only valid within the key's validity period.

Generation steps:

1. Concatenate the `random` fields from SourceHello and DestinationHello (SourceHello first, DestinationHello second, no separator)
2. Encode the concatenated string to bytes using UTF-8
3. Derive 8 bytes using HKDF
4. Encode the 8 bytes as 16 hexadecimal characters

```python
from cryptography.hazmat.primitives.kdf.hkdf import HKDF
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.backends import default_backend

def generate_secret_key_id(source_random: str, destination_random: str) -> str:
    content = source_random + destination_random
    random_bytes = content.encode('utf-8')

    hkdf = HKDF(
        algorithm=hashes.SHA256(),
        length=8,  # Generate 8 bytes
        salt=None,
        info=b'',
        backend=default_backend()
    )

    derived_key = hkdf.derive(random_bytes)
    return derived_key.hex()  # 16 hexadecimal characters
```

#### 7.4.4 AES-GCM Encryption/Decryption

Using `TLS_AES_128_GCM_SHA256` for message encryption:

```python
import os
import base64
from typing import Dict
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.backends import default_backend

def encrypt_aes_gcm(data: bytes, key: bytes) -> Dict[str, str]:
    """AES-128-GCM encryption"""
    if len(key) != 16:
        raise ValueError("Key must be 128 bits (16 bytes).")

    iv = os.urandom(12)  # GCM recommended 12-byte IV

    encryptor = Cipher(
        algorithms.AES(key),
        modes.GCM(iv),
        backend=default_backend()
    ).encryptor()

    ciphertext = encryptor.update(data) + encryptor.finalize()
    tag = encryptor.tag

    return {
        "iv": base64.b64encode(iv).decode('utf-8'),
        "tag": base64.b64encode(tag).decode('utf-8'),
        "ciphertext": base64.b64encode(ciphertext).decode('utf-8')
    }
```

### 7.5 Signature and Verification

#### 7.5.1 proofValue Generation Process

Both SourceHello and DestinationHello messages require signatures to ensure message integrity and identity verification.

**Generation Steps:**

1. Construct all fields of the message, excluding the `proof_value` field in `proof`
2. Convert the JSON to be signed into a JSON string, using commas and colons as separators, and **sort by keys**
3. Encode the JSON string to UTF-8 bytes
4. Sign using ECDSA + SHA-256 algorithm
5. Base64URL encode the signature value and write to the `proof_value` field

```python
import json
import base64

# 1. Construct message, excluding proof_value field
msg = {
    "e2ee_type": "source_hello",
    # ... other fields
    "proof": {
        "type": "EcdsaSecp256r1Signature2019",
        "created": "2024-05-27T10:51:55Z",
        "verification_method": "did:wba:example.com:user:alice#keys-1"
        # proof_value excluded
    }
}

# 2. Convert to sorted JSON string
msg_str = json.dumps(msg, separators=(',', ':'), sort_keys=True)

# 3. Encode to UTF-8 bytes
msg_bytes = msg_str.encode('utf-8')

# 4. Sign using ECDSA + SHA-256
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import ec

signature = private_key.sign(msg_bytes, ec.ECDSA(hashes.SHA256()))

# 5. Base64URL encode
msg["proof"]["proof_value"] = base64.urlsafe_b64encode(signature).decode('utf-8')
```

#### 7.5.2 Verifying SourceHello/DestinationHello Messages

1. **Parse message**: Extract all fields.
2. **Verify DID and public key**: Read the `source_did` and public key from `verification_method`, use the DID generation method from the DID method specification to generate a DID from the public key, and confirm it matches `source_did`.
3. **Verify signature**: Use the public key corresponding to `source_did` to verify whether the `proof` field signature is correct.
4. **Verify other fields**: Check the randomness of the `random` field to prevent replay attacks; check the `created` field in `proof` to ensure the signature time has not expired.

## 8. Security Considerations

### 8.1 Transport Security

All API endpoints should use the HTTPS protocol to ensure encrypted transmission and integrity of data during communication.

### 8.2 Identity Authentication Security

- **DID WBA Authentication**: Based on decentralized identity standards, users prove their identity by holding the private key corresponding to their DID.
- **Bearer JWT**: JWT tokens should have reasonable expiration times to reduce the risk window of token leakage.
- **Identity Consistency**: When authenticated users send messages, the server verifies the consistency between `sender_did` and the authenticated identity.

### 8.3 E2EE Security

- **Server-transparent forwarding**: The `content` field of E2EE messages is opaque to the server; the server cannot read encrypted content.
- **Forward secrecy**: Using ECDHE ephemeral key exchange, new key pairs are generated for each session. Even if long-term keys are compromised, past communications cannot be decrypted.
- **Key validity period**: Ephemeral keys have explicit validity periods and must be renegotiated after expiration.

### 8.4 Replay Attack Prevention

- **random field**: The `random` field in SourceHello and DestinationHello ensures the uniqueness of each handshake.
- **secretKeyId verification**: The Finished message contains `secret_key_id` for verifying the integrity of key negotiation.
- **Timestamp verification**: The `created` field in `proof` is used to check whether the signature time is within a reasonable range.

## 9. Limitations

### 9.1 E2EE Limited to Private Chat

The current E2EE scheme only supports one-to-one encrypted communication between two Agents and does not support end-to-end encryption in group chat scenarios. Group messages are forwarded in plaintext through the server. A group E2EE scheme (such as the MLS protocol) may be introduced in the future.

### 9.2 Large File Transfer Efficiency

Message content is transmitted via the `content` field in JSON, which is not efficient for large files (such as videos). It is recommended to transmit the file's URL and decryption key in the `content`, and the receiver downloads the file separately via HTTPS or other protocols.

## 10. Summary and Future Directions

This specification defines an instant messaging protocol based on HTTP + JSON-RPC 2.0, using `did:wba` as the identity identifier, supporting private chat, group chat, directory management, and end-to-end encrypted communication. Compared to the original WebSocket scheme, this protocol offers better simplicity, compatibility, and extensibility.

Future work directions include:

1. **Group E2EE**: Exploring end-to-end encryption schemes for group chat scenarios
2. **Offline Messages**: Improving push and synchronization mechanisms for offline messages
3. **Message Type Extension**: Supporting richer message types (such as audio/video call signaling)
4. **Cross-server Interoperability**: Defining federation protocols between Message Servers

## References

- [RFC 7159] The JavaScript Object Notation (JSON) Data Interchange Format
- [JSON-RPC 2.0 Specification](https://www.jsonrpc.org/specification)
- [ANP Technical White Paper](01-agentnetworkprotocol-technical-white-paper.md)
- [DID:WBA Method Design Specification](03-did-wba-method-design-specification.md)
- [04-E2EE Communication Protocol Based on DID](message/04-end-to-end-encrypted-communication-technology-protocol-based-on-did.md)
- [05-Message Service Protocol Based on DID](message/05-message-service-protocol-based-on-did.md)

## Copyright Notice

Copyright (c) 2024 GaoWei Chang
This document is released under the [MIT License](LICENSE). You are free to use and modify it, but you must retain this copyright notice.
