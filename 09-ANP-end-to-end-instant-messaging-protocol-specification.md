# ANP End-to-End Instant Messaging Protocol Specification (Draft)

Note: This specification is still a draft version and will undergo further optimization and iteration.

## Abstract

This specification defines the ANP End-to-End Instant Messaging Protocol, an instant messaging protocol based on JSON-RPC 2.0 that supports private chat, group chat, and end-to-end encrypted (E2EE) communication. The protocol uses `did:wba` as the identity identifier and employs the DID WBA authentication mechanism for identity verification.

The protocol adopts a transport-agnostic design — JSON-RPC 2.0 defines the message format and method semantics, which can be carried over HTTP, WebSocket, or other transport methods. This specification provides an HTTP transport binding as the default reference implementation.

The E2EE scheme is based on HPKE (Hybrid Public Key Encryption, RFC 9180), adopting a key separation design — signing keys (ECDSA secp256r1) and key agreement keys (X25519) serve distinct purposes. Private chat E2EE initializes session keys in one step via HPKE, with subsequent messages encrypted using symmetric keys + chain ratchet; group chat E2EE is based on Sender Keys + epoch rotation mechanism, distributing Sender Keys independently to each group member via HPKE.

This specification replaces the schemes described in [04-E2EE Communication Protocol Based on DID](message/04-end-to-end-encrypted-communication-technology-protocol-based-on-did.md) and [05-Message Service Protocol Based on DID](message/05-message-service-protocol-based-on-did.md).

## 1. Background

### 1.1 Existing Protocol Review

The ANP project designed two messaging-related protocol specifications in its early stages:

- **04-E2EE Communication Protocol Based on DID**: Defined a WebSocket-based E2EE encrypted communication scheme, including SourceHello, DestinationHello, and Finished handshake messages, as well as an ECDHE-based ephemeral key negotiation mechanism.
- **05-Message Service Protocol Based on DID**: Defined a WebSocket-based message routing and forwarding mechanism, including Message Proxy connections, router registration, heartbeat keep-alive, etc.

The existing scheme has the following issues:
1. **Complex Handshake Process**: The ECDHE scheme requires a three-step interactive handshake (SourceHello → DestinationHello → Finished), increasing latency and implementation complexity.
2. **Private Chat E2EE Only**: End-to-end encryption cannot be achieved in group chat scenarios.
3. **Key Purpose Confusion**: The same set of secp256r1 keys is used for both signing and key agreement purposes.

### 1.2 Design Motivation

This specification upgrades both the message format layer and the encryption layer:

**Message Format Layer: JSON-RPC 2.0 (Transport-Agnostic)**

1. **Standardized**: JSON-RPC 2.0 is a mature remote procedure call protocol that defines a unified request/response format and error code system.
2. **Transport-Agnostic**: JSON-RPC 2.0 defines only the message format and method semantics, without depending on any specific transport protocol. It can be carried over HTTP, WebSocket, TCP, or other transport methods, adapting to different scenario requirements.
3. **Simple and Reliable**: JSON format has mature support libraries in any programming language and platform.
4. **Easily Extensible**: Adding new message types or API methods only requires extending method names and parameter definitions, without modifying the transport layer.

**Encryption Layer: HPKE Replaces ECDHE**

1. **Standardized Security**: HPKE is an RFC 9180 standardized scheme with rigorous security proofs.
2. **One-Step Initialization**: HPKE supports unidirectional encapsulation, simplifying the three-step handshake to a one-step init — the initiator can begin encrypted communication without waiting for the other party's reply.
3. **Key Separation**: X25519 focuses on key agreement (HPKE KEM), ECDSA secp256r1 focuses on signing, with clear separation of responsibilities.
4. **Group Chat E2EE**: HPKE naturally supports independent key encapsulation for multiple recipients, providing the foundation for group chat Sender Key distribution.

### 1.3 Relationship with Existing Protocols

This specification replaces the original 04 and 05 protocols:

- **Replaces 04 (E2EE Protocol)**: The E2EE encryption scheme is upgraded from ECDHE interactive handshake to HPKE unidirectional encapsulation + chain ratchet, with new group chat E2EE support.
- **Replaces 05 (Message Service Protocol)**: The messaging protocol changes from WebSocket persistent connections to a transport-agnostic JSON-RPC 2.0 message format, removing the Message Proxy, router registration, and other mechanisms.

## 2. Scheme Overview

### 2.1 Architecture Design

The overall architecture adopts a message server relay model:

```plaintext
+-------+          +----------------+          +----------------+          +-------+
|       | HTTP/WS/ |                |          |                | HTTP/WS/ |       |
| Agent | <------> | Message Server | <------> | Message Server | <------> | Agent |
|  (A)  | JSON-RPC |     (A's)      |          |     (B's)      | JSON-RPC |  (B)  |
+-------+          +----------------+          +----------------+          +-------+
```

- **Agent**: The sender and receiver of messages, identified by `did:wba`.
- **Message Server**: Provides message sending/receiving services for Agents, including message storage, inbox management, and group management.
- Agents communicate with their associated Message Servers via JSON-RPC 2.0 protocol; the transport layer can use HTTP, WebSocket, or other methods.
- Communication between Message Servers is outside the scope of this specification.

### 2.2 Core Features

| Feature | Description |
|---------|-------------|
| **Message Format** | JSON-RPC 2.0 |
| **Transport Protocol** | Transport-agnostic (supports HTTP, WebSocket, etc., see Section 4) |
| **Content Format** | `application/json` |
| **Identity Identifier** | `did:wba` (Decentralized Identifier) |
| **Authentication** | DID WBA Authentication (see [DID:WBA Method Design Specification](03-did-wba-method-design-specification.md)) |
| **Message Types** | Private chat, Group chat |
| **E2EE Support** | HPKE-based end-to-end encryption (private chat + group chat) |
| **E2EE Crypto Stack** | DHKEM(X25519, HKDF-SHA256) / HKDF-SHA256 / AES-128-GCM |
| **Signing Algorithm** | ECDSA secp256r1 (P-256) |

### 2.3 Design Principles

1. **Simplicity**: Use mature, widely supported standard protocols (JSON-RPC 2.0, JSON, HPKE), with flexible transport layer choices.
2. **Security**: Support end-to-end encryption with key separation design where the server transparently forwards encrypted messages without being able to read ciphertext.
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

### 3.2 Authentication

This protocol adopts the ANP unified DID WBA authentication mechanism for identity verification. For the complete authentication flow and technical details, see [DID:WBA Method Design Specification](03-did-wba-method-design-specification.md).

## 4. JSON-RPC 2.0 Message Format and Transport Bindings

### 4.1 Request/Response Format

All APIs use the JSON-RPC 2.0 protocol to define message format. JSON-RPC 2.0 itself is transport-agnostic, defining only the structure of JSON messages and method semantics.

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
| -32003 | Conflict | Resource already exists or state conflict (including duplicate client_msg_id submissions beyond the idempotency window) |
| -32004 | Business Validation Error | Business rule validation failed |
| -32005 | Rate Limited | Request too frequent, rate limited by server |

### 4.3 Transport Bindings

The JSON-RPC 2.0 messages of this protocol can be carried over different transport methods. Implementations **MUST** support at least one transport binding; the HTTP binding is **RECOMMENDED**.

#### 4.3.1 HTTP Transport Binding (Default)

- **Protocol**: HTTP/HTTPS POST
- **Content Type**: `Content-Type: application/json`
- **Character Encoding**: UTF-8
- **Endpoints**: JSON-RPC requests are sent to the corresponding service endpoint paths (e.g., `/api/v1/messages/rpc`)

All interfaces are invoked via HTTP POST method, with the request body being a JSON-RPC 2.0 formatted JSON object and the response body being a JSON-RPC 2.0 formatted result or error.

HTTP binding is suitable for most scenarios, especially for stateless, firewall-friendly environments.

#### 4.3.2 WebSocket Transport Binding

- **Protocol**: WebSocket (ws/wss)
- **Endpoint Path**: `/message/ws` (under the message service prefix)
- **Content Format**: Each WebSocket message is a complete JSON-RPC 2.0 JSON object
- **Character Encoding**: UTF-8
- **Connection Management**: After the client establishes a WebSocket connection, multiple JSON-RPC messages can be sent and received over the same connection. The server supports multiple concurrent connections per user (e.g., multi-device scenarios).

WebSocket binding is suitable for scenarios requiring real-time push (e.g., instant message notifications), where the server can proactively push new messages to clients without requiring client-side inbox polling.

##### Authentication

WebSocket connections **MUST** be authenticated during the handshake phase. Implementations **MUST** support at least one of the following authentication methods:

| Method | Mechanism | Description |
|--------|-----------|-------------|
| Bearer JWT | `Authorization: Bearer <jwt_token>` header | **Recommended**. The server verifies the JWT locally using the pre-configured public key (RS256). |
| DID WBA | `Authorization: DID-WBA <signature>` header | The server forwards the authorization header to the DID authentication endpoint for verification. |
| JWT Query Parameter | `?token=<jwt_token>` query parameter | For environments where custom headers are not supported (e.g., browser WebSocket API). Verification is identical to Bearer JWT. |

If authentication fails, the server **MUST** close the WebSocket connection with close code `4001` and reason `Unauthorized`.

Upon successful authentication, the server resolves the authenticated DID to an internal user identifier. The authenticated DID is automatically injected as `sender_did` for subsequent `send` requests if not explicitly provided by the client.

##### Client → Server: Requests

Clients send JSON-RPC 2.0 requests over the WebSocket connection. The supported methods are:

**`send` — Send Message**

The `send` method uses the same parameters as defined in [Section 6.1.1](#611-send--send-message), with the following WebSocket-specific behaviors:

- `sender_did` is automatically injected from the authenticated identity if not provided.
- The response is a standard JSON-RPC 2.0 response with the same `id` as the request.
- Messages sent via WebSocket trigger the same push notification pipeline as HTTP-sent messages.

**`ping` — Application-Layer Heartbeat**

```json
{"jsonrpc": "2.0", "method": "ping"}
```

The server responds with:

```json
{"jsonrpc": "2.0", "method": "pong"}
```

Clients **SHOULD** send periodic `ping` messages to keep the connection alive. The ping interval **MUST** be shorter than the server's idle timeout (default: 300 seconds). A recommended interval is 120 seconds.

##### Server → Client: Push Notifications

When a new message is delivered to a user (via any transport — HTTP or WebSocket), the server pushes a JSON-RPC 2.0 **Notification** (no `id` field) to all of the user's active WebSocket connections:

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

**Notification params fields:**

| Field | Type | Description |
|-------|------|-------------|
| id | string | Message ID |
| server_seq | integer | Server-assigned monotonically increasing sequence number within the conversation |
| sender_did | string | Sender DID |
| sender_name | string \| null | Sender display name |
| sender_avatar | string \| null | Sender avatar URL |
| receiver_did | string \| null | Receiver DID (private chat only, null for group chat) |
| group_did | string \| null | Group DID (group chat only, null for private chat) |
| group_name | string \| null | Group name (group chat only) |
| content | string | Message content (plaintext or E2EE JSON string) |
| type | string | Message type (same values as `send` method) |
| sent_at | string \| null | Client send timestamp (ISO 8601) |

Push notifications are delivered only to users with active WebSocket connections. Offline messages are retrieved via the HTTP `get_inbox` method.

##### Connection Lifecycle

| Event | Behavior |
|-------|----------|
| Idle timeout | If no messages are received from the client within the idle timeout period (default: 300 seconds), the server closes the connection with code `1000` and reason `Idle timeout`. |
| Client disconnect | The server removes the connection from the active connection pool. |
| Server shutdown | All active connections are closed gracefully. |
| Reconnection | Clients **SHOULD** implement reconnection with exponential backoff. After reconnecting, clients **SHOULD** call `get_inbox` to retrieve any messages missed during the disconnection period. |

#### 4.3.3 Other Transport Bindings

JSON-RPC 2.0 messages can also be carried over TCP, QUIC, message queues, or other transport methods. Implementations should ensure:

- Clear message boundaries (each JSON-RPC message can be fully parsed)
- Character encoding is UTF-8
- The transport layer provides reliable delivery guarantees (or handles retransmission at the application layer)

## 5. Agent Description Protocol Integration

This section describes how to declare E2EE instant messaging capabilities in an Agent Description (AD) document, enabling other agents to automatically discover and interact with the E2EE IM service. The ANP E2EE IM protocol is treated as a built-in protocol type within the Agent Description Protocol framework (see [Agent Description Protocol Specification](07-anp-agent-description-protocol-specification.md) for the full AD specification).

### 5.1 Declaring E2EE IM Support in the AD Document

An agent that supports the ANP E2EE IM protocol should include an entry in the `interfaces` array of its AD document. Since the E2EE IM protocol is a built-in ANP protocol, the interface description uses the `content` field for inline endpoint and capability declaration rather than referencing an external file via `url`.

**Example AD Document:**

```json
{
  "protocolType": "ANP",
  "protocolVersion": "1.0.0",
  "type": "AgentDescription",
  "name": "Example Chat Agent",
  "did": "did:wba:example.com:user:alice",
  "interfaces": [
    {
      "type": "StructuredInterface",
      "protocol": "ANP-E2EE-IM",
      "version": "1.0",
      "description": "ANP E2EE instant messaging protocol supporting private chat, group chat, and end-to-end encrypted communication.",
      "content": {
        "endpoints": [
          {
            "path": "/api/v1/messages/rpc",
            "description": "Message service endpoint for sending, receiving, and managing messages including E2EE encrypted messages"
          },
          {
            "path": "/api/v1/groups/rpc",
            "description": "Group service endpoint for group management and group messages"
          },
          {
            "path": "/message/ws",
            "transport": "websocket",
            "description": "WebSocket endpoint for real-time message push and bidirectional communication"
          }
        ],
        "e2ee": {
          "hpke_suite": "DHKEM-X25519-HKDF-SHA256/HKDF-SHA256/AES-128-GCM",
          "signing_algorithm": "EcdsaSecp256r1Signature2019",
          "features": ["private_chat_e2ee", "group_chat_e2ee"],
          "description": "E2EE capabilities: HPKE key encapsulation + chain ratchet (private chat), Sender Keys + epoch rotation (group chat)"
        }
      }
    }
  ]
}
```

### 5.2 Field Descriptions

**Interface-level Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| type | string | Yes | Fixed as `"StructuredInterface"` |
| protocol | string | Yes | Fixed as `"ANP-E2EE-IM"` to identify this protocol |
| version | string | Yes | Protocol version (e.g., `"1.0"`) |
| description | string | No | Human-readable description of the interface |
| content | object | Yes | Inline description of endpoints and E2EE capabilities |

**content.endpoints Element:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| path | string | Yes | API endpoint path (e.g., `/api/v1/messages/rpc`) |
| description | string | No | Description of the endpoint's functionality |

**content.e2ee Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| hpke_suite | string | Yes | HPKE cipher suite identifier |
| signing_algorithm | string | Yes | Signing algorithm (e.g., `"EcdsaSecp256r1Signature2019"`) |
| features | array | Yes | List of supported E2EE features (`"private_chat_e2ee"` / `"group_chat_e2ee"`) |
| description | string | No | Human-readable summary of E2EE capabilities |

## 6. Message Service API Definitions

This protocol defines 10 JSON-RPC methods distributed across two service endpoints.

### 6.1 Message Interface

**Endpoint:** `POST /api/v1/messages/rpc`

#### 6.1.1 send — Send Message

Send a message (private or group chat), also serving as the unified transport method for E2EE encrypted messages.

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
| client_msg_id | string | Yes | Client-generated globally unique message ID (UUIDv4/UUIDv7/ULID), used for idempotent deduplication |

**type Values:**

| type | Category | Description |
|------|----------|-------------|
| `text` | Regular | Text message |
| `image` | Regular | Image message |
| `file` | Regular | File message |
| `e2ee_init` | Private E2EE | Session initialization (HPKE encapsulated session seed) |
| `e2ee_msg` | Private E2EE | Encrypted message (symmetric AEAD ciphertext) |
| `e2ee_rekey` | Private E2EE | Session reset/rekey |
| `e2ee_error` | E2EE | E2EE error notification |
| `group_e2ee_key` | Group E2EE | Sender Key distribution |
| `group_e2ee_msg` | Group E2EE | Group ciphertext message |
| `group_epoch_advance` | Group E2EE | Group epoch change notification |

**E2EE Constraints:**
- Private chat E2EE messages (type `e2ee_init`, `e2ee_msg`, `e2ee_rekey`, `e2ee_error`) must specify `receiver_did` and cannot specify `group_did`
- Group chat E2EE messages (type `group_e2ee_msg`, `group_epoch_advance`) must specify `group_did` and cannot specify `receiver_did`
- `group_e2ee_key` (Sender Key distribution) requires both `receiver_did` (the member receiving the Sender Key) and `group_did` (the associated group) — this is the only case where both are specified simultaneously
- Violating these constraints returns `-32602` Invalid Params

**Request Example (Private Chat):**

```json
{
  "jsonrpc": "2.0",
  "method": "send",
  "params": {
    "sender_did": "did:wba:example.com:user:alice",
    "receiver_did": "did:wba:example.com:user:bob",
    "content": "Hello",
    "type": "text",
    "client_msg_id": "01961e2c-45a0-7c8b-a4e3-f0a1b2c3d4e5"
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
    "content": "Hello everyone",
    "client_msg_id": "01961e2c-55b1-7d9c-b5f4-a1b2c3d4e5f6"
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
    "client_msg_id": "01961e2c-45a0-7c8b-a4e3-f0a1b2c3d4e5",
    "server_seq": 42,
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

**Response result Field Description:**

| Field | Type | Description |
|-------|------|-------------|
| id | string | Server-assigned message ID |
| client_msg_id | string | Client-submitted idempotency key (echoed back as-is) |
| server_seq | integer | Server-assigned monotonically increasing sequence number within the conversation, used for message ordering and gap detection |
| sender_did | string | Sender DID |
| sender_name | string \| null | Sender name |
| receiver_did | string \| null | Receiver DID (private chat) |
| group_did | string \| null | Group DID (group chat) |
| content | string | Message content |
| type | string | Message type |
| sent_at | string \| null | Client send timestamp |
| created_at | string | Server creation timestamp |

See Section 8 for E2EE message request examples.

##### 6.1.1.1 Idempotency and Ordering Mechanism

**client_msg_id Idempotency Rules:**

- Clients **MUST** generate a globally unique `client_msg_id` for every `send` request (UUIDv7 or ULID recommended, UUIDv4 compatible).
- The server uses `(sender_did, client_msg_id)` as the idempotency key. If a duplicate idempotency key is received, the server **MUST** return the result from the first processing (same `id`, `server_seq`, and `created_at`) instead of creating a new message.
- The idempotency window **SHOULD** be at least 24 hours. For duplicate `client_msg_id` submissions beyond the idempotency window, the server **MAY** return `-32003` (Conflict) error.
- `client_msg_id` is used solely for transport-layer idempotent deduplication and is independent of and complementary to the E2EE layer's `seq` anti-replay mechanism.

**server_seq Ordering Mechanism:**

- `server_seq` is assigned by the server when the message is successfully persisted, starting from 1 and monotonically increasing (independently counted per conversation).
- Dimension definitions:
  - **Private chat**: `server_seq` increments within the `(min(A_did, B_did), max(A_did, B_did))` dimension, meaning the same pair of private chat participants share one sequence number space.
  - **Group chat**: `server_seq` increments within the `group_did` dimension, meaning all messages within the same group share one sequence number space.
- `server_seq` guarantees monotonic increase but **allows gaps** (e.g., due to database allocation strategies), and **does not allow regression**.
- All query interfaces **MUST** return message lists sorted by `server_seq` in ascending order.

**Client Gap Detection and Backfill:**

- Clients **SHOULD** track the maximum received `server_seq` for each conversation.
- When a message received via WebSocket push or query interface has a `server_seq` that is not contiguous with the local maximum (i.e., a gap exists), the client **SHOULD** call `get_history(since_seq=<local max server_seq>)` to backfill missing messages.
- Backfilled messages are inserted into the local message list in ascending `server_seq` order.

**server_seq vs E2EE seq Comparison:**

| | server_seq | E2EE seq |
|------|------|------|
| **Assigned by** | Message Server | Sending client |
| **Dimension** | Private=DID pair, Group=group_did | Private=session_id, Group=(sender_did, epoch, sender_key_id) |
| **Purpose** | Message ordering, gap detection, incremental fetch | Key derivation, anti-replay |
| **Visibility** | Plaintext, visible to all participants and server | Inside E2EE content, end-to-end only |
| **Coverage** | All message types (plaintext + E2EE) | E2EE encrypted messages only (e2ee_msg, group_e2ee_msg) |

#### 6.1.2 get_detail — Get Message Details

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
    "server_seq": 42,
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

#### 6.1.3 get_inbox — Query Inbox

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
        "server_seq": 42,
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

#### 6.1.4 mark_read — Mark as Read

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

#### 6.1.5 get_history — Get Conversation History

Get message history for private or group conversations.

**Authentication:** optional

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| user_did | string | Yes | Current user DID |
| peer_did | string | Conditional | Peer user DID (private chat, mutually exclusive with group_did) |
| group_did | string | Conditional | Group DID (group chat, mutually exclusive with peer_did) |
| since | datetime | No | Incremental query start time |
| since_seq | integer | No | Only return messages with server_seq greater than this value. Mutually exclusive with since; if both are provided, since_seq takes precedence |
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
        "server_seq": 42,
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

**Note:** The returned message list **MUST** be sorted by `server_seq` in ascending order.

#### 6.1.6 get_batch_history — Batch Get Group Chat History

Batch get message history for multiple group chats.

**Authentication:** optional

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| user_did | string | Yes | Current user DID |
| queries | array | Yes | Query parameters for each group `[{group_did, since?, since_seq?}]` |
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
      {"group_did": "did:wba:example.com:group:group_2", "since": "2024-01-15T00:00:00"},
      {"group_did": "did:wba:example.com:group:group_3", "since_seq": 100}
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

**Note:** Each group's `messages` array contains message objects with a `server_seq` field, and **MUST** be sorted by `server_seq` in ascending order. `since_seq` and `since` within `queries` are mutually exclusive; if both are provided, `since_seq` takes precedence.

#### 6.1.7 delete_inbox — Delete Inbox Messages

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

### 6.2 Group Interface

**Endpoint:** `POST /api/v1/groups/rpc`

#### 6.2.1 list — Get Public Group List

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

#### 6.2.2 get_detail — Get Group Details

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

#### 6.2.3 get_messages — Get Group Messages

Get the message list for a specified group.

**Authentication:** none

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| group_did | string | Yes | Group DID |
| limit | int | No | Number of records to return, default 50, max 100 |
| since | datetime | No | Only return messages after this time |
| since_seq | integer | No | Only return messages with server_seq greater than this value. Mutually exclusive with since; if both are provided, since_seq takes precedence |

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
        "server_seq": 42,
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

**Note:** The returned message list **MUST** be sorted by `server_seq` in ascending order.

## 7. Message Type Definitions

### 7.1 Regular Message Types

| Type | Description | content |
|------|-------------|---------|
| `text` | Text message | Plaintext |
| `image` | Image message | Image URL or Base64 encoded |
| `file` | File message | File URL or file metadata |

### 7.2 Private Chat E2EE Message Types

All E2EE messages are transmitted via the `content` field of the `send` method, where the content is a JSON serialized string. The server **does not parse** or **modify** the content, forwarding it transparently.

| type | Description | content Structure |
|------|-------------|-------------------|
| `e2ee_init` | Session initialization | Contains HPKE encapsulated data (`enc`, `encrypted_seed`) and signature `proof` |
| `e2ee_msg` | Encrypted message | Contains `session_id`, `seq`, `original_type` and AES-GCM ciphertext |
| `e2ee_rekey` | Session reset/rekey | Same structure as `e2ee_init`, semantically re-establishes session keys |
| `e2ee_error` | E2EE error notification | Contains `error_code` and `session_id` |

See Section 8.4 for detailed E2EE content structure definitions.

### 7.3 Group Chat E2EE Message Types

Group chat E2EE messages are also transmitted via the `content` field of the `send` method, with the server forwarding them transparently.

| type | Description | content Structure |
|------|-------------|-------------------|
| `group_e2ee_key` | Sender Key distribution | Contains HPKE encapsulated Sender Key, `epoch`, `sender_key_id` and signature |
| `group_e2ee_msg` | Group ciphertext message | Contains `epoch`, `sender_key_id`, `seq`, AES-GCM ciphertext |
| `group_epoch_advance` | Group epoch change notification | Contains `new_epoch`, `reason`, member change lists and admin signature |

See Section 8.6 for detailed group chat E2EE content structure definitions.

## 8. End-to-End Encryption (E2EE) Protocol

### 8.1 Overview

This protocol's end-to-end encryption scheme is based on HPKE (Hybrid Public Key Encryption, RFC 9180), supporting encrypted communication in both private chat and group chat scenarios.

**Core Design:**

- **Private Chat E2EE**: The initiator uses the recipient's X25519 `keyAgreement` public key from their DID document to encapsulate a random session seed via HPKE (completed in one step, no interactive handshake required); subsequent messages are encrypted using symmetric session keys derived from the seed + chain ratchet.
- **Group Chat E2EE**: Each sender generates a Sender Key and distributes it independently to each group member via HPKE encapsulation; group messages are encrypted using the Sender Key's chain ratchet (encrypted once, decryptable by all members holding the Sender Key). When membership changes, the epoch is incremented, triggering a new round of Sender Key distribution.

**Key Features:**

- **Server-transparent forwarding**: All E2EE data is embedded in the `content` field of the `send` method; the server does not parse or modify encrypted data
- **Unified transport**: All E2EE messages are distinguished by the `type` field of the `send` method
- **One-step initialization**: Private chat session establishment requires only a single `e2ee_init` message, with no interactive handshake needed

### 8.2 Cryptographic Foundations

#### 8.2.1 Key Separation

This protocol strictly separates signing keys and key agreement keys:

| Purpose | Algorithm | DID Document Field | Description |
|---------|-----------|-------------------|-------------|
| Signing / Identity verification | ECDSA secp256r1 | `authentication` / `assertionMethod` | Used for proof signatures, DID WBA authentication |
| Key agreement | X25519 | `keyAgreement` | Used for HPKE KEM encapsulation/decapsulation |

Signing keys do not participate in key agreement, and key agreement keys do not participate in signing. This separation design ensures that the compromise of a single key does not simultaneously affect both identity authentication and communication confidentiality.

#### 8.2.2 HPKE Crypto Stack

This protocol uses the following fixed HPKE crypto stack:

| Component | Selection | RFC 9180 Identifier |
|-----------|-----------|-------------------|
| KEM | DHKEM(X25519, HKDF-SHA256) | 0x0020 |
| KDF | HKDF-SHA256 | 0x0001 |
| AEAD | AES-128-GCM | 0x0001 |

Full suite identifier string: `DHKEM-X25519-HKDF-SHA256/HKDF-SHA256/AES-128-GCM`

HPKE operation mode: **Base mode** (i.e., `mode_base`, sender identity is not bound at the HPKE layer — sender identity is guaranteed through proof signatures).

#### 8.2.3 keyAgreement Reference in DID Documents

The recipient's X25519 public key required for HPKE encapsulation is obtained from the `keyAgreement` field of the recipient's DID document. The DID:WBA specification defines this field as optional, with type `X25519KeyAgreementKey2019` and public key format `publicKeyMultibase`.

**Public Key Retrieval Steps:**

1. Resolve the recipient's DID to obtain the DID document
2. Extract the `X25519KeyAgreementKey2019` type entry from the `keyAgreement` array
3. Decode the `publicKeyMultibase` field to obtain the raw X25519 public key bytes (32 bytes)

**DID Document Snippet Example:**

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

If the other party's DID document does not contain a `keyAgreement` field or does not have an `X25519KeyAgreementKey2019` type entry, it indicates that the agent does not support E2EE.

### 8.3 Private Chat E2EE Protocol

#### 8.3.1 Session Model

Each private chat E2EE session maintains the following state:

| State Variable | Type | Description |
|---------------|------|-------------|
| session_id | string | Session identifier, 32 random hex characters |
| root_seed | bytes(32) | Session seed from HPKE decryption |
| send_chain_key | bytes(32) | Chain key for the send direction |
| recv_chain_key | bytes(32) | Chain key for the receive direction |
| send_seq | uint64 | Send sequence number, starting from 0 |
| recv_seq | uint64 | Receive sequence number, starting from 0 |
| expires_at | datetime | Session expiration time |

**Direction Determination Rule:** Compare the DID strings of the sender and receiver (UTF-8 byte order); the party with the lexicographically smaller DID is the `initiator`. The `initiator` uses label `"anp-e2ee-init"` to derive send_chain_key and `"anp-e2ee-resp"` to derive recv_chain_key; the other party does the reverse. This ensures that both parties have a completely consistent assignment of send/recv key chains, regardless of who initiates the `e2ee_init`.

#### 8.3.2 Session Initialization Flow

```plaintext
Alice                           Message Server                          Bob
  |                                   |                                   |
  |  [1. Get Bob's DID document]      |                                   |
  |  [2. Extract keyAgreement         |                                   |
  |      X25519 public key]           |                                   |
  |  [3. Generate root_seed           |                                   |
  |      (32 bytes)]                  |                                   |
  |  [4. HPKE.Seal(bob_pk,           |                                   |
  |       root_seed) -> (enc, ct)]    |                                   |
  |  [5. Sign proof]                  |                                   |
  |                                   |                                   |
  |  send(type=e2ee_init)             |                                   |
  |  content={session_id, enc, ct,    |                                   |
  |           proof, ...}             |                                   |
  |---------------------------------->|  send(type=e2ee_init)             |
  |                                   |---------------------------------->|
  |                                   |                                   |
  |                                   |  [6. HPKE.Open(enc, ct)           |
  |                                   |       -> root_seed]               |
  |                                   |  [7. Verify proof signature]      |
  |                                   |  [8. Derive send/recv             |
  |                                   |      chain keys]                  |
  |                                   |                                   |
  |  [Alice also derives send/recv chain keys, session activated]         |
  |                                   |                                   |
  |  send(type=e2ee_msg, seq=0)       |                                   |
  |  content=encrypted message        |                                   |
  |---------------------------------->|  send(type=e2ee_msg)              |
  |                                   |---------------------------------->|
  |                                   |                                   |
```

**Flow Description:**

1. Alice resolves Bob's DID document and obtains Bob's X25519 public key from `keyAgreement`.
2. Alice generates a 32-byte random `root_seed` (**MUST** use a cryptographically secure random number generator, CSPRNG) and `session_id`.
3. Alice uses HPKE Base mode `Seal` operation to encapsulate `root_seed`:
   - Recipient public key: Bob's X25519 public key
   - Plaintext: `root_seed` (32 bytes)
   - AAD: UTF-8 encoding of `session_id`
   - Output: `enc` (32-byte X25519 ephemeral public key, Base64 encoded) and `encrypted_seed` (AEAD ciphertext, Base64 encoded)
4. Alice signs the entire content with a proof signature (using the signing key from her DID document's `authentication`).
5. Alice sends the `type=e2ee_init` message via the `send` method. After sending, Alice can immediately send `e2ee_msg` encrypted messages without waiting for Bob's reply.
6. Upon receiving the message, Bob uses his X25519 private key and `enc` to perform HPKE `Open`, recovering `root_seed`.
7. Bob obtains Alice's signing public key from her DID document and verifies the proof signature.
8. Both parties derive send/recv chain keys from `root_seed` (see 8.3.3), and the session is activated.

**Message Delivery Order Requirements:**

The Message Server **SHOULD** make best efforts to guarantee that messages from the same sender to the same receiver are delivered in sending order (FIFO order), but absolute FIFO delivery is not always achievable in distributed systems. The protocol provides **detectable and recoverable** ordering guarantees through `server_seq`:

- Receivers detect whether the arrival order of messages meets expectations via `server_seq`. If a `server_seq` gap appears, the receiver **SHOULD** backfill missing messages via `get_history(since_seq=...)`.
- The `e2ee_init` message **SHOULD** be delivered to the receiver before subsequent `e2ee_msg` messages. If the receiver receives an `e2ee_msg` first and cannot find the corresponding session state for the `session_id`, it **MUST** buffer the message (limit 64 messages, timeout 30 seconds) and wait for the `e2ee_init` to arrive. After the buffer timeout, it **SHOULD** discard the message and return an `e2ee_error` (error_code: `session_not_found`).
- The same applies to `e2ee_rekey` and subsequent `e2ee_msg` — `e2ee_rekey` **SHOULD** be delivered before any `e2ee_msg` using the new session_id.
- For group chat, `group_e2ee_key` **SHOULD** be delivered to the corresponding receiver before the same sender's `group_e2ee_msg`. See Section 8.5.3 for the missing key buffering strategy.

**E2EE Message Serial Sending Constraint:**

Within the same E2EE session, the sender **MUST** wait for the response of the previous `send` request before sending the next E2EE-type message (`e2ee_init`, `e2ee_msg`, `e2ee_rekey`, `group_e2ee_key`, `group_e2ee_msg`). This constraint ensures that the `server_seq` assignment order is consistent with the sender's E2EE `seq` order, allowing the receiver to infer E2EE `seq` ordering from `server_seq` ordering.

- This constraint **applies only to E2EE-type messages**. Regular plaintext messages (`text`, `image`, `file`) can be sent concurrently.
- E2EE messages across different sessions (different `session_id` or different `group_did`) can be sent concurrently.

#### 8.3.3 Chain Ratchet Key Derivation

**Initial Derivation (generating directional chain keys from root_seed):**

The `root_seed` is a 32-byte random number generated by the sender using a cryptographically secure random number generator (CSPRNG), transmitted via HPKE encryption. Since `root_seed` already has sufficient entropy (256-bit random), it can be used directly as the PRK (Pseudo-Random Key) input for HKDF-Expand, without the need for an HKDF-Extract step.

```python
from cryptography.hazmat.primitives.kdf.hkdf import HKDFExpand
from cryptography.hazmat.primitives import hashes

def derive_chain_keys(root_seed: bytes) -> tuple[bytes, bytes]:
    """Derive two directional initial chain keys from root_seed"""
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

**Direction Assignment:**
- The party with the lexicographically smaller DID (initiator): `send_chain_key = init_chain_key`, `recv_chain_key = resp_chain_key`
- The party with the lexicographically larger DID (responder): `send_chain_key = resp_chain_key`, `recv_chain_key = init_chain_key`

**Per-Message Key Derivation:**

```python
import hmac
import hashlib

def derive_message_key(chain_key: bytes, seq: int) -> tuple[bytes, bytes, bytes]:
    """Derive message key, returns (enc_key, nonce, new_chain_key)"""
    seq_bytes = seq.to_bytes(8, 'big')

    # Derive message key
    msg_key = hmac.new(chain_key, b"msg" + seq_bytes, hashlib.sha256).digest()
    # Update chain key
    new_chain_key = hmac.new(chain_key, b"ck", hashlib.sha256).digest()

    # Derive AES encryption key and nonce from msg_key
    enc_key = hmac.new(msg_key, b"key", hashlib.sha256).digest()[:16]  # AES-128
    nonce = hmac.new(msg_key, b"nonce", hashlib.sha256).digest()[:12]  # GCM nonce

    return enc_key, nonce, new_chain_key
```

Each message uses an independent `enc_key` and `nonce`, which are discarded after use. The chain key is updated to a new value after each message (one-way update — the old chain_key cannot be derived from the new chain_key, achieving message-level forward secrecy).

#### 8.3.4 Message Encryption/Decryption

**Encryption Flow:**

1. Call `derive_message_key(send_chain_key, send_seq)` to obtain `enc_key`, `nonce`, `new_chain_key`
2. Encrypt the plaintext using AES-128-GCM, AAD = UTF-8 encoding of `session_id`
3. Concatenate the ciphertext and GCM tag, Base64 encode them, and write to the `ciphertext` field
4. Update `send_chain_key = new_chain_key`, `send_seq += 1`

**Decryption Flow:**

1. Verify that the `seq` in the message is within an acceptable range (see out-of-order tolerance below)
2. Call `derive_message_key(recv_chain_key, recv_seq)` to obtain `enc_key`, `nonce`, `new_chain_key`
3. Base64 decode `ciphertext`, separate the ciphertext and GCM tag, decrypt using AES-128-GCM
4. Update `recv_chain_key = new_chain_key`, `recv_seq += 1`

**Sequence Numbers and Out-of-Order Tolerance:**

This specification defines two seq validation strategies; implementations **MUST** support at least one:

- **Strict mode (default)**: Requires `seq == recv_seq` (strictly incrementing), rejecting any out-of-order messages. This mode relies on the Message Server guaranteeing FIFO delivery order in the same direction (see 8.3.2 Message Delivery Order Requirements).
- **Window mode (recommended)**: Allows `recv_seq <= seq < recv_seq + MAX_SKIP` (recommended `MAX_SKIP = 256`). If `seq > recv_seq`, the receiver needs to "fast-forward" the intermediate chain keys from `recv_seq` to `seq - 1` (by consecutively calling `derive_message_key` to advance chain_key to the target position), and **SHOULD** cache the skipped msg_keys for later decryption of out-of-order messages that arrive subsequently. Cached msg_keys **SHOULD** have a time-to-live (recommended no more than 300 seconds), after which they are destroyed.

Regardless of which mode is used, previously used `seq` values **MUST** be rejected for duplicate decryption (anti-replay).

**AES-128-GCM Encryption Example:**

```python
import os
import base64
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

def encrypt_message(plaintext: bytes, enc_key: bytes, nonce: bytes,
                    session_id: str) -> str:
    """AES-128-GCM encryption, returns Base64-encoded ciphertext (including tag)"""
    aad = session_id.encode('utf-8')
    aesgcm = AESGCM(enc_key)
    ct_with_tag = aesgcm.encrypt(nonce, plaintext, aad)
    return base64.b64encode(ct_with_tag).decode('utf-8')

def decrypt_message(ciphertext_b64: str, enc_key: bytes, nonce: bytes,
                    session_id: str) -> bytes:
    """AES-128-GCM decryption"""
    aad = session_id.encode('utf-8')
    ct_with_tag = base64.b64decode(ciphertext_b64)
    aesgcm = AESGCM(enc_key)
    return aesgcm.decrypt(nonce, ct_with_tag, aad)
```

#### 8.3.5 Session Rebuild (Rekey)

Either party can send an `e2ee_rekey` message to rebuild session keys in the following situations:

- Session key expired (past `expires_at`)
- Security event (suspected key compromise)
- Device change or state loss
- Proactive key rotation

The structure of `e2ee_rekey` is identical to `e2ee_init`. Upon receiving `e2ee_rekey`:

1. Perform the same HPKE decapsulation and key derivation as `e2ee_init`
2. Use the new `session_id` and `root_seed`
3. Reset `send_seq` and `recv_seq` to 0
4. Destroy the old session state

#### 8.3.6 Error Handling

When E2EE communication encounters anomalies, the other party is notified via `e2ee_error` type messages.

| error_code | Description | Recommended Action |
|------------|-------------|-------------------|
| `session_not_found` | Session does not exist or has been destroyed | Initiator resends `e2ee_init` |
| `session_expired` | Session has expired | Initiator sends `e2ee_rekey` |
| `decryption_failed` | Decryption failed (key desync, etc.) | Initiator sends `e2ee_rekey` |
| `invalid_seq` | Sequence number mismatch | Initiator sends `e2ee_rekey` |
| `unsupported_suite` | Unsupported HPKE cipher suite | Check the other party's capability declaration in AD document |
| `no_key_agreement` | Other party's DID document has no keyAgreement | Fall back to plaintext communication or abort |
| `sender_key_not_found` | Sender Key not received for the corresponding sender in group chat, buffer timeout | Sender redistributes `group_e2ee_key` |

### 8.4 Private Chat E2EE Content Structure Definitions

All E2EE data is transmitted via the `content` field of the `send` method (as JSON serialized strings).

#### 8.4.1 e2ee_init — Session Initialization

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Session ID, 32 random hex characters |
| hpke_suite | string | Yes | HPKE cipher suite identifier |
| sender_did | string | Yes | Sender DID |
| recipient_did | string | Yes | Recipient DID |
| recipient_key_id | string | Yes | ID of the keyAgreement entry in the recipient's DID document |
| enc | string | Yes | HPKE encapsulated ciphertext (Base64 encoded, 32-byte X25519 ephemeral public key) |
| encrypted_seed | string | Yes | HPKE AEAD encrypted root_seed (Base64 encoded, including GCM tag) |
| expires | integer | No | Session validity period (seconds), default 86400 |
| proof | object | Yes | Sender signature |

**proof Structure (applicable to all E2EE messages requiring signatures):**

| Field | Type | Description |
|-------|------|-------------|
| type | string | Signature type (`"EcdsaSecp256r1Signature2019"`) |
| created | string | Signature creation time (ISO 8601 UTC) |
| verification_method | string | Verification method ID used for signing (entry from DID document `authentication`) |
| proof_value | string | Signature value (Base64URL encoded) |

**Content Example (before JSON serialization):**

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

**send Request Example:**

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

#### 8.4.2 e2ee_msg — Encrypted Message

`e2ee_msg` does not include a `proof` signature field. The reason: the authenticity of `e2ee_msg` is guaranteed by the session key itself — only the party holding the correct chain_key for the corresponding `session_id` can generate valid ciphertext (AES-128-GCM provides authenticated encryption), and the chain_key is derived from the root_seed authenticated via proof signature in `e2ee_init`. Therefore, successful decryption itself verifies the sender's identity. Additionally, omitting per-message signatures avoids the computational overhead of ECDSA signing.

**Trust Prerequisite**: When processing `e2ee_msg`, the receiver **MUST** verify that the `sender_did` in the `send` method matches the peer DID bound to the session identified by `session_id`. If they don't match, the receiver **MUST** reject the message. This ensures that even if the Message Server tampers with the outer `sender_did` field, messages cannot be incorrectly attributed to other sessions.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Session ID |
| seq | integer | Yes | Message sequence number (starting from 0, strictly incrementing) |
| original_type | string | Yes | Original message type (`text` / `image` / `file`) |
| ciphertext | string | Yes | AES-128-GCM ciphertext + tag (Base64 encoded) |

**Content Example:**

```json
{
  "session_id": "a1b2c3d4e5f60718a1b2c3d4e5f60718",
  "seq": 0,
  "original_type": "text",
  "ciphertext": "ZW5jcnlwdGVkIG1lc3NhZ2UgY29udGVudA=="
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
    "content": "{\"session_id\":\"a1b2c3d4e5f60718a1b2c3d4e5f60718\",\"seq\":0,\"original_type\":\"text\",\"ciphertext\":\"ZW5jcnlwdGVkIG1lc3NhZ2UgY29udGVudA==\"}",
    "type": "e2ee_msg",
    "client_msg_id": "01961e2c-71d3-7fbd-d7b6-c3d4e5f60718"
  },
  "id": 1
}
```

**Note:** The nonce is deterministically derived from the chain ratchet (not randomly generated), and the GCM tag is appended to the ciphertext. Therefore, the `ciphertext` field alone can carry the complete AEAD output, without the need for separate `iv` and `tag` fields.

#### 8.4.3 e2ee_rekey — Session Rebuild

The structure is identical to `e2ee_init`; all field definitions are in Section 8.4.1.

Semantic difference: `e2ee_rekey` indicates rebuilding an existing session key. The receiver should destroy the old session state and establish a new session using the new `session_id` and `root_seed`.

#### 8.4.4 e2ee_error — Error Notification

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| error_code | string | Yes | Error code (see error code table in 8.3.6) |
| session_id | string | No | Associated session ID |
| failed_msg_id | string | No | The message ID that failed to decrypt |
| message | string | No | Human-readable error description |

**Content Example:**

```json
{
  "error_code": "session_not_found",
  "session_id": "a1b2c3d4e5f60718a1b2c3d4e5f60718",
  "failed_msg_id": "msg_20260305_abc123",
  "message": "E2EE session not found, please re-initialize"
}
```

### 8.5 Group Chat E2EE Protocol

#### 8.5.1 Overview and Core Concepts

Group chat E2EE adopts the **Sender Keys** scheme: each group member, as a sender, holds their own symmetric Sender Key and distributes it to all other group members via HPKE. Group messages are encrypted using the sender's Sender Key chain ratchet, and all members holding that Sender Key can decrypt.

Core concepts:

| Concept | Type | Description |
|---------|------|-------------|
| **epoch** | integer | Group epoch, non-negative integer starting from 0, incremented on membership changes |
| **sender_key_id** | string | Sender Key identifier, suggested format `{sender_did_short_hash}:{epoch}` |
| **sender_chain_key** | bytes(32) | 32-byte random symmetric key, used for chain ratchet to derive each group message's encryption key |
| **sender_seq** | uint64 | Sender's message sequence number within the current epoch |

#### 8.5.2 Sender Key Model

Each group member generates a `sender_chain_key` (32 bytes random) for each epoch. The chain ratchet for this key is similar to private chat but uses different label prefixes for differentiation:

```python
def derive_group_message_key(sender_chain_key: bytes, seq: int) -> tuple[bytes, bytes, bytes]:
    """Derive group message key"""
    seq_bytes = seq.to_bytes(8, 'big')

    msg_key = hmac.new(sender_chain_key, b"gmsg" + seq_bytes, hashlib.sha256).digest()
    new_chain_key = hmac.new(sender_chain_key, b"gck", hashlib.sha256).digest()

    enc_key = hmac.new(msg_key, b"key", hashlib.sha256).digest()[:16]
    nonce = hmac.new(msg_key, b"nonce", hashlib.sha256).digest()[:12]

    return enc_key, nonce, new_chain_key
```

#### 8.5.3 Sender Key Distribution Flow

When a member speaks for the first time in an epoch (or after epoch rotation), they must first distribute their Sender Key to all other group members:

```plaintext
Alice (sender)                  Message Server           Bob (member), Carol (member)
  |                                   |                              |
  |  [Generate sender_chain_key]      |                              |
  |                                   |                              |
  |  [Get Bob's X25519 public key]    |                              |
  |  [HPKE.Seal(bob_pk,              |                              |
  |   sender_chain_key)]             |                              |
  |                                   |                              |
  |  send(type=group_e2ee_key,        |                              |
  |   receiver_did=bob,               |                              |
  |   group_did=group_dev)            |                              |
  |---------------------------------->|  Forward to Bob              |
  |                                   |----------------------------->|
  |                                   |                              |
  |  [Get Carol's X25519 public key]  |                              |
  |  [HPKE.Seal(carol_pk,            |                              |
  |   sender_chain_key)]             |                              |
  |                                   |                              |
  |  send(type=group_e2ee_key,        |                              |
  |   receiver_did=carol,             |                              |
  |   group_did=group_dev)            |                              |
  |---------------------------------->|  Forward to Carol            |
  |                                   |----------------------------->|
  |                                   |                              |
  |  [Distribution complete,          |                              |
  |   begin sending group ciphertext] |                              |
  |                                   |                              |
  |  send(type=group_e2ee_msg,        |                              |
  |   group_did=group_dev)            |                              |
  |---------------------------------->|  Forward to all members      |
  |                                   |----------------------------->|
```

The sender performs HPKE encapsulation separately for **each other member** in the group, sending independent distribution messages via `send(type=group_e2ee_key)`. The `enc` and `encrypted_sender_key` in each message differ because the recipients are different.

**Sender Key Replay Rejection:** Receivers **MUST** reject `group_e2ee_key` messages with an already existing `(sender_did, epoch, sender_key_id)` combination. If a duplicate combination is received, it **MUST** be discarded, and the existing Sender Key state must not be overwritten. This prevents attackers from rolling back a receiver's Sender Key state to a known value by replaying old `group_e2ee_key` messages.

**Missing Sender Key Buffering Strategy:**

When a receiver receives a `group_e2ee_msg` but has not yet received the corresponding sender's `group_e2ee_key` (i.e., no local Sender Key exists for that `(sender_did, epoch, sender_key_id)`), it **MUST** follow this strategy:

1. **Buffer the message**: The receiver **MUST** buffer the message in a local waiting queue rather than immediately discarding it.
   - Buffer limit: Up to **64 messages** per `(sender_did, epoch, sender_key_id)` dimension.
   - Buffer timeout: Maximum wait time of **30 seconds** per message.

2. **Timeout handling**: After the buffer timeout, the receiver **SHOULD** discard the message and send an `e2ee_error` (error_code: `sender_key_not_found`) to the sender, notifying them to redistribute the Sender Key.

3. **Processing upon key arrival**: When the corresponding `group_e2ee_key` arrives, the receiver **MUST** process all buffered messages in ascending `server_seq` order.

4. **Post-timeout backfill**: After buffer timeout and discard, the receiver **MAY** backfill missing messages via `get_history(since_seq=<local max server_seq>)` and reprocess them after the Sender Key arrives.

5. **Message Server responsibility**: The Message Server **SHOULD** make best efforts to deliver `group_e2ee_key` messages before the same sender's `group_e2ee_msg` messages to the corresponding receiver.

#### 8.5.4 Group Message Encryption/Decryption

**Encryption (Sender):**

1. Use the sender's own `sender_chain_key` for the current epoch
2. Call `derive_group_message_key(sender_chain_key, sender_seq)` to get `enc_key`, `nonce`, `new_chain_key`
3. Encrypt with AES-128-GCM, AAD = UTF-8 encoding of `group_did + ":" + str(epoch)`
4. Update `sender_chain_key = new_chain_key`, `sender_seq += 1`
5. Send via `send(type=group_e2ee_msg, group_did=...)`, the server forwards to all group members

**Decryption (Receiver):**

1. Look up the locally stored corresponding `sender_chain_key` based on the message's `sender_did` + `epoch` + `sender_key_id`
2. Verify that `seq` is within the acceptable range (see group chat sequence number rules below)
3. Derive keys, decrypt
4. Update the corresponding sender's `recv_seq` and `sender_chain_key`

**Group Chat Sequence Numbers and Out-of-Order Tolerance:**

In group chat, each `(sender_did, epoch, sender_key_id)` dimension maintains an independent `recv_seq`. The sequence number validation strategy is the same as private chat (see 8.3.4):
- **Strict mode**: Requires `seq == recv_seq`
- **Window mode (recommended)**: Allows `recv_seq <= seq < recv_seq + MAX_SKIP`, skipped msg_keys can be cached

Regardless of mode, previously used `seq` values **MUST** be rejected for duplicate decryption (anti-replay).

#### 8.5.5 Epoch Rotation (Membership Changes)

When group membership changes, the epoch is incremented to ensure forward and backward secrecy:

**Admin Identity and Trust Anchor:**

The sender of `group_epoch_advance` messages **MUST** have group admin privileges. Group admin identity is defined and maintained by the Message Server's group management API (see Section 6 for group management interfaces). Specifically:
- The group creator is the default group admin.
- The Message Server **SHOULD** verify whether the sender is a group admin before forwarding `group_epoch_advance` messages. `group_epoch_advance` messages sent by non-admins **SHOULD** be rejected by the Message Server.
- When processing `group_epoch_advance`, receivers verify the signer's identity via `proof.verification_method` and **SHOULD** check whether the signer is a known group admin (the admin list can be obtained via the group management API or cached from prior group metadata messages).
- Clients **SHOULD NOT** rely entirely on the Message Server's admin validation — clients **MUST** independently verify the `proof` signature to ensure that even if the Message Server is compromised or behaves maliciously, epoch changes cannot be forged.

**Rotation Flow:**

1. The group admin (or server) sends a `group_epoch_advance` message announcing the new epoch and reason for change
2. All members remaining in the group transition to the new epoch, with old epoch Sender Keys handled as follows:
   - Old epoch `sender_chain_key` state **MUST** be marked as read-only, and sending new messages with old epoch keys is **PROHIBITED**
   - Old epoch `sender_chain_key` **SHOULD** be retained (in read-only mode) to decrypt delayed old epoch messages
   - The retention period for old epoch keys is **RECOMMENDED** not to exceed 3600 seconds (1 hour); they **SHOULD** be destroyed after expiration
3. Each member regenerates `sender_chain_key` for the new epoch
4. Each member distributes the new Sender Key to all other current group members via `group_e2ee_key`

**Security Guarantees:**

- **Forward secrecy (member removal)**: Removed members cannot obtain new epoch Sender Keys and cannot decrypt future messages
- **Backward secrecy (member addition)**: Newly added members only obtain current epoch Sender Keys and cannot decrypt historical messages

**Epoch Increment Trigger Conditions:**

| Trigger Event | reason Field Value | Required Action |
|---------------|-------------------|-----------------|
| Member added | `member_added` | epoch+1, **all** existing members **MUST** regenerate Sender Keys and redistribute to all members (including new members) |
| Member left or removed | `member_removed` | epoch+1, **all** remaining members **MUST** regenerate Sender Keys and redistribute to all members |
| Proactive key rotation | `key_rotation` | epoch+1, **all** members **MUST** regenerate Sender Keys and redistribute |

**Important**: Regardless of the reason triggering the epoch increment, all members **MUST** generate a completely new `sender_chain_key` (old values must not be reused) and redistribute via `group_e2ee_key` to all other current group members. This ensures:
- Removed members cannot use old keys to decrypt new epoch messages (forward secrecy)
- Newly added members cannot use new keys to decrypt old epoch messages (backward secrecy)

### 8.6 Group Chat E2EE Content Structure Definitions

#### 8.6.1 group_e2ee_key — Sender Key Distribution

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| group_did | string | Yes | Group DID |
| epoch | integer | Yes | Group epoch |
| sender_did | string | Yes | DID of the Sender Key owner |
| sender_key_id | string | Yes | Sender Key identifier |
| recipient_key_id | string | Yes | ID of the keyAgreement entry in the recipient's DID document |
| hpke_suite | string | Yes | HPKE cipher suite identifier |
| enc | string | Yes | HPKE encapsulated ciphertext (Base64 encoded) |
| encrypted_sender_key | string | Yes | HPKE AEAD encrypted sender_chain_key (Base64 encoded, including tag) |
| expires | integer | No | Sender Key validity period (seconds), default 86400 |
| proof | object | Yes | Sender signature |

AAD for HPKE encryption of `sender_chain_key` = UTF-8 encoding of `group_did + ":" + str(epoch) + ":" + sender_key_id`.

**Content Example:**

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

**send Request Example:**

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

#### 8.6.2 group_e2ee_msg — Group Ciphertext Message

Similar to `e2ee_msg`, `group_e2ee_msg` does not include a `proof` signature field. Authenticity is guaranteed by `sender_chain_key` — only the party holding the correct Sender Key can generate valid ciphertext, and the Sender Key comes from the proof-signed `group_e2ee_key` distribution message.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| group_did | string | Yes | Group DID |
| epoch | integer | Yes | Group epoch |
| sender_did | string | Yes | Sender DID |
| sender_key_id | string | Yes | Sender Key identifier |
| seq | integer | Yes | Sender's message sequence number within the current epoch |
| original_type | string | Yes | Original message type (`text` / `image` / `file`) |
| ciphertext | string | Yes | AES-128-GCM ciphertext + tag (Base64 encoded) |

**Content Example:**

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

**send Request Example:**

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

#### 8.6.3 group_epoch_advance — Group Epoch Change Notification

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| group_did | string | Yes | Group DID |
| new_epoch | integer | Yes | New epoch value |
| reason | string | Yes | Change reason: `member_added` / `member_removed` / `key_rotation` |
| members_added | array | No | List of newly added member DIDs |
| members_removed | array | No | List of removed member DIDs |
| proof | object | Yes | Group admin signature |

**Content Example:**

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

### 8.7 Signature and Verification

#### 8.7.1 proof_value Generation Process

E2EE messages requiring signatures include: `e2ee_init`, `e2ee_rekey`, `group_e2ee_key`, `group_epoch_advance`.

**Generation Steps:**

1. Construct all fields of the message, excluding the `proof_value` field in `proof`
2. Canonicalize the JSON using JCS (JSON Canonicalization Scheme, RFC 8785)
3. Encode the canonicalized JSON string to UTF-8 bytes
4. Sign using ECDSA + SHA-256 algorithm (using the signing key from the DID document's `authentication`)
5. Base64URL encode the signature value and write to the `proof_value` field

```python
import json
import base64
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import ec

# 1. Construct message, excluding proof_value field
msg = {
    "session_id": "a1b2c3d4e5f60718a1b2c3d4e5f60718",
    "hpke_suite": "DHKEM-X25519-HKDF-SHA256/HKDF-SHA256/AES-128-GCM",
    # ... other fields
    "proof": {
        "type": "EcdsaSecp256r1Signature2019",
        "created": "2026-03-01T10:30:00Z",
        "verification_method": "did:wba:example.com:user:alice#keys-1"
        # proof_value excluded
    }
}

# 2. JCS canonicalization (RFC 8785)
# JCS canonicalization uses deterministic JSON serialization,
# ensuring the same object always produces the same byte representation
msg_canonical = jcs_canonicalize(msg)  # Canonicalize per RFC 8785

# 3. Encode to UTF-8 bytes
msg_bytes = msg_canonical.encode('utf-8')

# 4. Sign using ECDSA + SHA-256
signature = private_key.sign(msg_bytes, ec.ECDSA(hashes.SHA256()))

# 5. Base64URL encode
msg["proof"]["proof_value"] = base64.urlsafe_b64encode(signature).decode('utf-8')
```

#### 8.7.2 Signature Verification Process

1. **Extract signature**: Extract `proof.proof_value` from the message, Base64URL decode to get signature bytes
2. **Reconstruct data to verify**: Remove the `proof.proof_value` field from the message, canonicalize the remaining JSON using JCS
3. **Obtain public key**: Based on the ID referenced by `proof.verification_method`, obtain the corresponding signing public key from the sender's DID document
4. **Verify signature**: Use the public key and ECDSA + SHA-256 to verify the signature
5. **Verify timestamp**: Check whether the `proof.created` time is within a reasonable range (recommended ±300-second window, allowing up to 5 minutes of clock skew). Messages outside the window **SHOULD** be rejected.

## 9. Security Considerations

### 9.1 Transport Security

All API endpoints should use the HTTPS protocol to ensure encrypted transmission and integrity of data during communication.

### 9.2 Identity Authentication Security

Identity authentication follows the ANP DID WBA authentication mechanism. For detailed security considerations regarding DID-based authentication, see [DID:WBA Method Design Specification](03-did-wba-method-design-specification.md). When authenticated users send messages, the server verifies the consistency between `sender_did` and the authenticated identity.

### 9.3 Private Chat E2EE Security

- **Server-transparent forwarding**: All E2EE content is opaque to the server; the server cannot read root_seed or session keys.
- **Session initialization security**: HPKE Base mode uses ephemeral Diffie-Hellman key encapsulation. A passive third-party attacker cannot recover `root_seed` from `enc` without knowing the recipient's X25519 private key. However, note that **HPKE Base mode does not provide forward secrecy against compromise of the recipient's static key** — if the recipient's X25519 long-term private key is compromised in the future, an attacker can combine previously intercepted `enc` and `encrypted_seed` to recover `root_seed`, and subsequently derive all message keys for that session.

  | Compromise Scenario | Can Historical root_seed Be Recovered? | Description |
  |--------------------|---------------------------------------|-------------|
  | Sender's ephemeral private key compromised | No | HPKE ephemeral key pair is only used during Seal and not persisted |
  | Recipient's X25519 static private key compromised | **Yes** | Can combine intercepted `enc` to perform HPKE.Open and recover root_seed |
  | Both parties' static private keys compromised | **Yes** | Same as above; the recipient's private key alone suffices for decryption |

  It is recommended to periodically rotate X25519 keys (update keyAgreement in the DID document) to reduce the impact window of key compromise, and to rebuild session keys via `e2ee_rekey`.
- **Message-level forward secrecy**: The chain ratchet ensures each message uses an independent msg_key, and chain_key is updated in a one-way manner — the old chain_key and msg_key cannot be derived from the current chain_key. Assuming root_seed and chain_key have been securely deleted, even if a subsequent chain_key is compromised, historical messages cannot be recovered.
- **Key separation**: Signing keys (ECDSA secp256r1) and key agreement keys (X25519) are physically separated; compromise of one does not affect the security of the other.
- **Authenticity**: `e2ee_init` / `e2ee_rekey` carry the sender's proof signature, allowing the receiver to verify the initiator's identity.
- **Key compromise risk**: Since `e2ee_msg` does not carry per-message signatures (authenticity relies on symmetric session keys), if `chain_key` or `root_seed` is compromised, an attacker can forge subsequent encrypted messages within that session. Therefore, `chain_key` **MUST** be updated immediately after use and old values destroyed, and `root_seed` **SHOULD** be destroyed immediately after deriving chain keys.

### 9.4 Group Chat E2EE Security

- **Forward secrecy (member removal)**: After a member is removed, the epoch is incremented and all remaining members regenerate Sender Keys. The removed member cannot obtain new epoch Sender Keys and cannot decrypt future messages.
- **Backward secrecy (member addition)**: Newly added members only obtain current epoch Sender Keys and cannot decrypt historical messages.
- **Independent Sender Key distribution**: Each member's Sender Key is independently encapsulated via HPKE for each recipient; compromise of a single recipient's key does not affect other recipients.
- **Epoch signature**: `group_epoch_advance` messages carry admin signatures, preventing forged epoch changes.
- **Sender Key compromise risk**: Under the Sender Key model, all group members hold each sender's `sender_chain_key`. If a member's device is compromised causing their `sender_chain_key` to leak, an attacker can forge that sender's subsequent group messages within the current epoch, until epoch rotation. This is a known limitation of the Sender Key scheme, which can be mitigated by shortening epoch rotation cycles (e.g., periodic `key_rotation`).

### 9.5 Key Management Security

- X25519 key agreement keys **SHOULD** be periodically rotated (update keyAgreement in the DID document). After rotation, `e2ee_rekey` **SHOULD** be proactively sent to active session peers to rebuild sessions with the new key.
- Session keys (root_seed, sender_chain_key) should only exist in both parties' memory and should not be persisted to disk.
- The chain ratchet's chain_key **MUST** be updated immediately after use, and old values **MUST** be destroyed immediately.

### 9.6 Replay Attack Prevention

- **Sequence number anti-replay**: The `seq` field in `e2ee_msg` and `group_e2ee_msg` is strictly incrementing; receivers reject any message with a previously used seq value (regardless of strict or window mode).
- **session_id uniqueness**: Each `e2ee_init` / `e2ee_rekey` generates a new random session_id.
- **epoch + sender_key_id uniqueness**: In group chat, the epoch + sender_key_id combination uniquely identifies a Sender Key lifecycle.
- **proof timestamp**: The `created` timestamp in signatures is used to detect expired messages.
- **Complementary relationship between client_msg_id idempotency and E2EE seq anti-replay**: `client_msg_id` prevents duplicate message delivery caused by HTTP retries at the transport layer (deduplication performed by the Message Server), while E2EE `seq` prevents attackers from replaying captured ciphertext messages at the end-to-end layer (verified by the receiving client). The two mechanisms operate independently at different layers, collectively ensuring message uniqueness. Even if the Message Server is compromised or the `client_msg_id` idempotency check is bypassed, the E2EE `seq` anti-replay protection remains effective.

## 10. Limitations

### 10.1 O(N) Distribution Cost for Group Chat E2EE

Sender Key distribution requires independent HPKE encapsulation for each group member, resulting in significant overhead when the group member count N is large. Future consideration may be given to MLS (RFC 9420) tree-based key distribution schemes for optimization.

### 10.2 Single Device Limitation

In this version of the specification, a DID **MUST NOT** actively use E2EE functionality on multiple devices simultaneously. Concurrent multi-device usage would cause ratchet state conflicts, group chat Sender Key confusion, and session state loss. Multi-device key synchronization mechanisms will be defined in future versions.

### 10.3 Large File Transfer Efficiency

Message content is transmitted via the `content` field in JSON, which is not efficient for large files (such as videos). It is recommended to transmit the file's URL and decryption key in the `content`, and the receiver downloads the file separately via HTTPS or other protocols.

## 11. Summary and Future Directions

This specification defines an instant messaging protocol based on HTTP + JSON-RPC 2.0, using `did:wba` as the identity identifier, supporting end-to-end encrypted communication for both private chat and group chat. The E2EE scheme is based on HPKE (RFC 9180), adopting a key separation design — private chat achieves session encryption through one-step HPKE initialization + chain ratchet, while group chat achieves group encryption through Sender Keys + epoch rotation.

Through integration with the Agent Description Protocol, agents can declare their E2EE IM capabilities in a standardized manner, enabling automatic discovery and interoperability.

Future work directions include:

1. **MLS Protocol Integration**: Exploring MLS (RFC 9420) tree-based key management schemes to optimize Sender Key distribution efficiency for large groups
2. **Multi-Device Key Synchronization**: Defining key synchronization and session management mechanisms for the same DID across multiple devices
3. **Offline Messages**: Improving push and synchronization mechanisms for offline messages
4. **Message Type Extension**: Supporting richer message types (such as audio/video call signaling)
5. **Cross-server Interoperability**: Defining federation protocols between Message Servers

## References

- [RFC 9180] Hybrid Public Key Encryption (HPKE)
- [RFC 8785] JSON Canonicalization Scheme (JCS)
- [RFC 7159] The JavaScript Object Notation (JSON) Data Interchange Format
- [JSON-RPC 2.0 Specification](https://www.jsonrpc.org/specification)
- [W3C DID Core Specification](https://www.w3.org/TR/did-core/)
- [ANP Technical White Paper](01-agentnetworkprotocol-technical-white-paper.md)
- [DID:WBA Method Design Specification](03-did-wba-method-design-specification.md)
- [Agent Description Protocol Specification](07-anp-agent-description-protocol-specification.md)

## Copyright Notice

Copyright (c) 2024 GaoWei Chang
This document is released under the [MIT License](LICENSE). You are free to use and modify it, but you must retain this copyright notice.
