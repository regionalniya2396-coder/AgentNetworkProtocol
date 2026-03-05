# did:wba Method Specification(V0.1)

## Abstract

The wba DID method is a web-based decentralized identifier (DID) specification designed to meet the requirements of cross-platform identity authentication and intelligent agent communication. This method extends and optimizes the did:web specification, named as did:wba, maintaining compatibility while enhancing adaptability for agent scenarios. In this specification, we design a process based on the did:wba method and HTTP protocol that enables servers to quickly verify client identities from other platforms without increasing interaction frequency.

## 1. Introduction 

### 1.1 Preface

The wba DID method specification complies with the requirements specified in Decentralized Identifiers V1.0 [[DID-CORE](https://www.w3.org/TR/did-core/)].

This specification builds upon the did:web method specification by adding DID document constraints, cross-platform identity authentication processes, and agent description service specifications, proposing a new method name did:wba (Web-Based Agent).

Considering that the did:web method specification is still a draft and may undergo changes unsuitable for agent communication scenarios, and that reaching consensus with the original authors on specification modifications is a long-term process, we have decided to use a new method name.

We do not rule out the possibility of merging the did:wba specification into the did:web specification in the future, and we will work towards achieving this goal.

The did:wba method references the did:web method specification at [https://w3c-ccg.github.io/did-method-web](https://w3c-ccg.github.io/did-method-web), version dated July 31, 2024. For management purposes, we have backed up a copy of the did:web method specification currently used by did:wba: [did:web Method Specification](/references/did_web%20Method%20Specification.html).

### 1.2 Design Principles

In designing the did:wba method, our core principle is to fully utilize existing mature technologies and robust Web infrastructure while achieving decentralization. Using did:wba enables email-like functionality where platforms can implement their account systems in a centralized manner while maintaining interoperability between platforms.

Furthermore, various types of identifier systems can add DID support, bridging interoperability between centralized, federated, and decentralized identifier systems. This means existing centralized identifier systems do not require complete restructuring; they can achieve cross-system interoperability by simply creating DIDs on top of their existing infrastructure, significantly reducing technical implementation complexity.
## 2. WBA DID Method Specification

### 2.1 Method Name
The method name string used to identify this DID method is: wba. DIDs using this method MUST begin with the following prefix: did:wba. According to the DID specification, this string MUST be lowercase. The remainder of the DID (after the prefix) is specified below.

### 2.2 Method-Specific Identifiers 
The method-specific identifier is a fully qualified domain name protected by TLS/SSL certificates, optionally including a path to the DID document. The formal rules describing valid domain name syntax are specified in [(RFC1035)](https://www.rfc-editor.org/rfc/rfc1035), [(RFC1123)](https://www.rfc-editor.org/rfc/rfc1123), and [(RFC2181)](https://www.rfc-editor.org/rfc/rfc2181).

The method-specific identifier MUST match the common name used in the SSL/TLS certificate and MUST NOT contain IP addresses. Port numbers may be included but colons must be percent-encoded to prevent conflicts with paths. Directories and subdirectories may be optionally included, using colons instead of forward slashes as separators.

wba-did = "did:wba:" domain-name
wba-did = "did:wba:" domain-name * (":" path)

```plaintext
example: wba method DID example

did:wba:example.com

did:wba:example.com:user:alice

did:wba:example.com%3A3000
```

### 2.4 Key Material and Document Processing

Due to how most web servers render content, specific did:wba documents are likely to be served with the media type application/json. When retrieving a document named did.json, the following processing rules should be followed:

1. If @context exists at the root of the JSON document, the document should be processed according to JSON-LD rules. If processing fails or cannot be performed, the document should be rejected as a did:wba document.

2. If @context exists at the root and the document is successfully processed as JSON-LD, and includes the context `https://www.w3.org/ns/did/v1`, it may be further processed as a DID document according to [[did-core specification section 6.3.2](https://www.w3.org/TR/did-core/#consumption-0)].

3. If @context does not exist, the document should be processed as a DID according to the normal JSON rules specified in [[did-core specification section 6.2.2](https://www.w3.org/TR/did-core/#consumption)].

4. When DID URLs appear in a did:wba document, they MUST be absolute URLs.

> Note: This includes URLs in embedded key material and other metadata, which prevents key confusion attacks.

### 2.5 DID Document Specification

Apart from the DID Core specification, most other specifications are still in draft stage. This section will demonstrate a subset of a DID document used for authentication. To enhance interoperability between systems, all fields marked as required MUST be supported by all systems; fields marked as optional MAY be selectively supported. Fields defined in other standards not listed here MAY be selectively supported.

**Example DID Document:**

```json
{
    "@context": [
      "https://www.w3.org/ns/did/v1",
      "https://w3id.org/security/suites/jws-2020/v1",
      "https://w3id.org/security/suites/secp256k1-2019/v1",
      "https://w3id.org/security/suites/ed25519-2020/v1",
      "https://w3id.org/security/suites/x25519-2019/v1"
    ],
    "id": "did:wba:example.com%3A8800:user:alice",
    "verificationMethod": [
      {
        "id": "did:wba:example.com%3A8800:user:alice#WjKgJV7VRw3hmgU6--4v15c0Aewbcvat1BsRFTIqa5Q",
        "type": "EcdsaSecp256k1VerificationKey2019",
        "controller": "did:wba:example.com%3A8800:user:alice",
        "publicKeyJwk": {
          "crv": "secp256k1",
          "x": "NtngWpJUr-rlNNbs0u-Aa8e16OwSJu6UiFf0Rdo1oJ4",
          "y": "qN1jKupJlFsPFc1UkWinqljv4YE0mq_Ickwnjgasvmo",
          "kty": "EC",
          "kid": "WjKgJV7VRw3hmgU6--4v15c0Aewbcvat1BsRFTIqa5Q"
        }
      },
      {
        "id": "did:wba:example.com%3A8800:user:alice#keys-p256-1",
        "type": "EcdsaSecp256r1VerificationKey2019",
        "controller": "did:wba:example.com%3A8800:user:alice",
        "publicKeyJwk": {
          "crv": "P-256",
          "x": "f83OJ3D2xF1Bg8vub9tLe1gHMzV76e8Tus9uPHvRVEU",
          "y": "x_FEzRu9m36HLN_tue659LNpXW6pCyStikYjKIWI5a0",
          "kty": "EC",
          "kid": "keys-p256-1"
        }
      },
      {
        "id": "did:wba:example.com%3A8800:user:alice#key-x25519-1",
        "type": "X25519KeyAgreementKey2019",
        "controller": "did:wba:example.com%3A8800:user:alice",
        "publicKeyMultibase": "z9hFgmPVfmBZwRvFEyniQDBkz9LmV7gDEqytWyGZLmDXE"
      }
    ],
    "authentication": [
      "did:wba:example.com%3A8800:user:alice#WjKgJV7VRw3hmgU6--4v15c0Aewbcvat1BsRFTIqa5Q",
      "did:wba:example.com%3A8800:user:alice#keys-p256-1",
      {
        "id": "did:wba:example.com%3A8800:user:alice#key-1",
        "type": "Ed25519VerificationKey2020",
        "controller": "did:wba:example.com%3A8800:user:alice",
        "publicKeyMultibase": "zH3C2AVvLMv6gmMNam3uVAjZpfkcJCwDwnZn6z3wXmqPV"
      }
    ],
    "keyAgreement": [
      "did:wba:example.com%3A8800:user:alice#key-x25519-1"
    ],
    "humanAuthorization": [
      "did:wba:example.com%3A8800:user:alice#WjKgJV7VRw3hmgU6--4v15c0Aewbcvat1BsRFTIqa5Q",
      {
        "id": "did:wba:example.com%3A8800:user:alice#key-3",
        "type": "Ed25519VerificationKey2020",
        "controller": "did:wba:example.com%3A8800:user:alice",
        "publicKeyMultibase": "z9XK2BVwLNv6gmMNbm4uVAjZpfkcJDwDwnZn6z3wweKLo"
      }
    ],
    "service": [
      {
        "id": "did:wba:example.com%3A8800:user:alice#agent-description",
        "type": "AgentDescription",
        "serviceEndpoint": "https://agent-network-protocol.com/agents/example/ad.json"
      },
      {
        "id": "did:wba:example.com%3A8800:user:alice#handle",
        "type": "HandleService",
        "serviceEndpoint": "https://example.com/.well-known/handle/alice"
      }
    ]
}
```

**Field Descriptions:**

- **@context**: Required field. The JSON-LD context defines the semantics and data model used in the DID document, ensuring document comprehensibility and interoperability. `https://www.w3.org/ns/did/v1` is required. Others can be added as needed.

- **id**: Required field. Cannot contain IP addresses but may include ports. When ports are included, colons must be encoded as %3A. Paths are separated using colons.

- **verificationMethod**: Required field. Contains an array of verification methods that define public key information for verifying the DID subject. For scenarios requiring end-to-end encryption (E2EE) communication, `verificationMethod` SHOULD include both signing keys and key agreement keys to achieve key separation. Signing keys (e.g., EcdsaSecp256r1VerificationKey2019) are used for identity authentication and E2EE proof signatures; key agreement keys (e.g., X25519KeyAgreementKey2019) are used for HPKE key encapsulation. Each type of key serves its own purpose, so that the compromise of a single key does not simultaneously affect both identity authentication and communication confidentiality.
  - **Sub-fields**:
    - **id**: Unique identifier for the verification method
    - **type**: Type of verification method
    - **controller**: DID that controls this verification method
    - **publicKeyJwk**: Public key information in JSON Web Key format

- **authentication**: Required field. Lists verification methods used for authentication, which can be strings or objects.
  - **Sub-fields**:
    - **id**: Unique identifier for the verification method
    - **type**: Type of verification method
    - **controller**: DID that controls this verification method
    - **publicKeyMultibase**: Public key information in Multibase format

- **keyAgreement**: Optional field. Defines public key information used for key agreement between DIDs for encrypted communication. Verification methods typically use key agreement algorithms suitable for key exchange, such as X25519KeyAgreementKey2019. `keyAgreement` can be either a string reference (pointing to an entry in `verificationMethod`) or an embedded object. For end-to-end encryption (E2EE) scenarios (see [09-ANP E2EE Instant Messaging Protocol Specification](/09-anp-e2ee-instant-messaging-protocol-specification.md)), this field is used for HPKE key encapsulation and SHOULD contain an entry of type `X25519KeyAgreementKey2019`. If a DID document has no `keyAgreement` field or no X25519 type entry, it indicates that the agent does not support E2EE.
  - **Sub-fields**:
    - **id**: Unique identifier for the key agreement method
    - **type**: Type of key agreement method
    - **controller**: DID that controls this key agreement method
    - **publicKeyMultibase**: Public key information in Multibase format

- **humanAuthorization**: Optional field that defines the public key information used for human authorization. The corresponding private key is used only when human authorization is required, for critical identity verification scenarios.
  - **Sub-fields**:
    - **id**: Unique identifier for the human authorization method.
    - **type**: Type of the human authorization method.
    - **controller**: DID that controls this human authorization method.
    - **publicKeyMultibase**: Public key information in Multibase format.

- **service**: Optional field that defines a list of services associated with the DID subject.
  - **id**: Unique identifier for the service.
  - **type**: Type of service. The following types are currently supported:
    - `AgentDescription`: Agent description service. The `serviceEndpoint` points to a document that follows the [ANP-Agent Description Protocol Specification](/07-anp-agent-description-protocol-specification.md).
    - `HandleService`: Handle binding service, used for WNS (WBA Name Space) bidirectional binding verification. The `serviceEndpoint` points to the Handle Resolution Endpoint (e.g., `https://example.com/.well-known/handle/alice`). By declaring a HandleService in the DID Document, the DID holder confirms their binding relationship with the specified Handle, allowing verifiers to perform bidirectional verification and prevent Handle Providers from unilaterally tampering with mappings. See [04-ANP-DID:WBA Name Space Specification](04-anp-did-wba-name-space-specification.md) for details.
  - **serviceEndpoint**: URL endpoint of the service.

> Note:
> 1. Public key information currently supports two formats: publicKeyJwk and publicKeyMultibase. For details, see [https://www.w3.org/TR/did-extensions-properties/#verification-method-properties](https://www.w3.org/TR/did-extensions-properties/#verification-method-properties).
> 2. The definitions of verification method types can be found at [https://www.w3.org/TR/did-extensions-properties/#verification-method-types](https://www.w3.org/TR/did-extensions-properties/#verification-method-types). The currently supported types are: EcdsaSecp256k1VerificationKey2019, EcdsaSecp256r1VerificationKey2019, Ed25519VerificationKey2018, Ed25519VerificationKey2020, X25519KeyAgreementKey2019. Among these, EcdsaSecp256r1VerificationKey2019 is used for signature verification in end-to-end encryption scenarios (see [09-ANP E2EE Instant Messaging Protocol Specification](/09-anp-e2ee-instant-messaging-protocol-specification.md)).
> 3. AgentDescription is a newly added service type to support the discovery of agent description documents.
> 4. HandleService is a newly added service type to support bidirectional binding verification between WNS (WBA Name Space) Handles and DIDs. By adding a HandleService entry in the DID Document's service section, the DID holder declares their associated Handle, allowing verifiers to check the consistency of mappings in both directions: Handle → DID (via Handle Provider) and DID → Handle (via DID Document). See [04-ANP-DID:WBA Name Space Specification](04-anp-did-wba-name-space-specification.md) for details.
> 5. For scenarios requiring end-to-end encrypted communication, a key separation design is recommended: signing keys (ECDSA secp256r1 or secp256k1) and key agreement keys (X25519) should be managed separately. Signing keys should not participate in key agreement, and key agreement keys should not participate in signing. For detailed E2EE protocol design, see [09-ANP E2EE Instant Messaging Protocol Specification](/09-anp-e2ee-instant-messaging-protocol-specification.md).

### 2.5 DID Method Operations

#### 2.5.1 Create (Register)

The did:wba method specification does not specify specific HTTP API operations, but rather leaves the programmatic registration and management to each implementation based on their Web environment requirements.

Creating a DID requires the following steps:

1. Apply for a domain name with a domain name registrar
2. Store the location and IP address of the hosted service in the DNS query service
3. Create a DID document JSON-LD file containing the appropriate key pair and store the did.json file in the well-known URL on behalf of the entire domain name. If multiple DIDs need to be parsed under the same domain name, store them in the specified path.

For example, for the domain name example.com, the did.json will be available at the following URL:

```plaintext
Example: Create a DID
did:wba:example.com
 -> https://example.com/.well-known/did.json
```

If a specified optional path is used instead of the bare domain name, the did.json will be available at the specified path:

```plaintext
Example 5: Create a DID using an optional path
did:wba:example.com:user:alice
 -> https://example.com/user/alice/did.json
```

If a specified port is used on the domain name, the colons between the host and port must be percent-encoded to prevent conflicts with paths.

```plaintext
Example 6: Create a DID using an optional path and port
did:wba:example.com%3A3000:user:alice
 -> https://example.com:3000/user/alice/did.json
```

#### 2.5.2 Read (Parse)

The following steps must be performed to parse a Web DID from a DID document:

- Replace the ":" in the method-specific identifier with "/" to obtain the fully qualified domain name and optional path.
- If the domain name contains a port, decode the port.
- Generate an HTTPS URL by prefixing the expected location of the DID document with `https://`.
- If the URL does not specify a path, append `/.well-known`.
- Append `/did.json` to complete the URL.
- Use a proxy that can successfully negotiate a secure HTTPS connection to execute an HTTP GET request to the URL. The proxy enforces the security and privacy requirements described in [2.6 Security and Privacy Considerations](https://w3c-ccg.github.io/did-method-web/#security-and-privacy-considerations).
- Verify that the ID of the parsed DID document matches the Web DID being parsed.
- During the HTTP GET request, the client should use [[RFC8484](https://w3c-ccg.github.io/did-method-web/#bib-rfc8484)] to prevent tracking the identity being parsed.

#### 2.5.3 Update

To update the DID document, the DID corresponding did.json file must be updated. Note that the DID itself will remain unchanged, but the content of the DID document can be changed, such as adding new verification keys or service endpoints.

> Note:
> Using version control systems such as git and continuous integration systems such as GitHub Actions to manage the update of the DID document can provide support for identity verification and auditing history.

> Note: HTTP API
> The update process does not specify specific HTTP API, but rather leaves the programmatic registration and management to each implementation based on their needs.

#### 2.5.4 Deactivate (Revoke)

To delete the DID document, the did.json file must be removed, or it must no longer be publicly available for other reasons.

### 2.6 Security and Privacy Considerations

Security and privacy considerations are based on [[did:web method specification section 2.6](https://w3c-ccg.github.io/did-method-web/#security-and-privacy-considerations)].

## 3. Cross-Platform Identity Authentication Based on did:wba Method and HTTP Protocol

When a client makes a request to a service on different platforms, the client can use the domain name combined with TLS to authenticate the service. The service then verifies the identity of the client based on the verification methods in the client's DID document.

The client can include the DID and signature in the HTTP header during the first HTTP request. Without increasing the number of interactions, the service can quickly verify the identity of the client. After the initial verification is successful, the service can return a access token to the client. The client can then carry the access token in subsequent requests, and the service does not need to verify the client's identity each time, but only needs to verify the access token.

```mermaid
sequenceDiagram
    participant Agent A Client
    participant Agent B Server 
    participant Agent A DID Sever

    Note over Agent A Client,Agent B Server: First Request

    Agent A Client->>Agent B Server: HTTP Request: DID,Signature
    Agent B Server->>Agent A DID Sever: Get DID Document
    Agent A DID Sever->>Agent B Server: DID Document

    Note over Agent B Server: Authentication

    Agent B Server->>Agent A Client: HTTP Response: access token

    Note over Agent A Client, Agent B Server: Subsequent Requests

    Agent A Client->>Agent B Server: HTTP Request: access token
    Agent B Server->>Agent A Client: HTTP Response
```


### 3.1 Initial Request

When the client first makes an HTTP request to the service, it needs to authenticate according to the following method.

#### 3.1.1 Request Header Format

The client sends the following information through the `Authorization` header field to the service:
- **DIDWba**: Indicates the use of the did:wba protocol
- **did**: The did identifier of the client, used for identity verification.
- **nonce**: A randomly generated string used to prevent replay attacks. It must be unique for each request. We recommend using a 16-byte random string.
- **timestamp**: The time when the request is initiated, usually in UTC format using ISO 8601, accurate to seconds.
- **verification_method**: Identifies the verification method used in the signature, which is the DID fragment of the verification method in the DID document. For example, for the verification method "did:wba:example.com%3A8800:user:alice#key-1", the verification method's DID fragment is "key-1".
- **signature**: Sign the `nonce`, `timestamp`, service domain, and client DID. For ECDSA signatures, use the R|S format. It includes the following fields:
  - `nonce`
  - `timestamp`
  - `aud` (the domain name of the service)
  - `did` (the DID of the client)

Client request example:

```plaintext
Authorization: DIDWba did="did:wba:example.com%3A8800:user:alice", nonce="abc123", timestamp="2024-12-05T12:34:56Z", verification_method="key-1", signature="base64url(signature_of_nonce_timestamp_aud_did)"
```

#### 3.1.2 Signature Generation Process

1. The client generates a string containing the following information:

```json
{ 
  "nonce": "abc123", 
  "timestamp": "2024-12-05T12:34:56Z", 
  "aud": "example.com",
  "did": "did:wba:example.com:user:alice"
}
```

2. Use [JCS(JSON Canonicalization Scheme)](https://www.rfc-editor.org/rfc/rfc8785) to normalize the JSON string, generating a normalized string.

3. Use the SHA-256 algorithm to hash the normalized string, generating a hash value.

4. Use the client's private key to sign the hash value, generating a signature value `signature`, and encode it in URL-safe Base64.

5. Construct the `Authorization` header in the above format and send it to the service.

### 3.2 Service Verification

#### 3.2.1 Verify Request Header

After receiving the client's request, the service performs the following verification:

1. **Verify Timestamp**: Check if the timestamp in the request is within a reasonable time range. The recommended time range is 1 minute. If the timestamp is out of range, the request is considered expired, and the service returns `401 Unauthorized` with a authentication challenge.

2. **Verify Nonce**: Check if the `nonce` in the request has been used or exists. If the `nonce` has been used or exists, it is considered a replay attack, and the service returns `401 Unauthorized` with a authentication challenge.

3. **Verify DID Permissions**: Verify if the DID in the request has the permission to access the resources of the service. If not, the service returns `403 Forbidden`.

4. **Verify Signature**:

- Read the DID document based on the client's DID.
- Find the corresponding verification method in the DID document based on the `verification_method` in the request.
- Use the public key of the verification method to verify the signature in the request.

5. **Verification Result**: If the signature verification is successful, the request passes verification; otherwise, the service returns `401 Unauthorized` with a authentication challenge.

#### 3.2.2 Signature Verification Process

1. **Extract Information**: Extract `nonce`, `timestamp`, `aud`, `did`, and `verification_method` from the `Authorization` header.

2. **Build Verification String**: Construct a JSON string identical to the one constructed by the client:

```json
{ 
    "nonce": "abc123", 
    "timestamp": "2024-12-05T12:34:56Z", 
    "aud": "example.com",
    "did": "did:wba:example.com:user:alice"
}
```

3. **Normalize String**: Use [JCS(JSON Canonicalization Scheme)](https://www.rfc-editor.org/rfc/rfc8785) to normalize the JSON string, generating a normalized string.

4. **Generate Hash Value**: Use the SHA-256 algorithm to hash the normalized string, generating a hash value.

5. **Get Public Key**: Obtain the corresponding public key from the DID document based on `did` and `verification_method`.

6. **Verify Signature**: Use the obtained public key to verify the `signature`, ensuring that it is generated by the corresponding private key.

#### 3.2.3 Authentication Success Return Access Token

After the service successfully verifies the client's identity, it can return a access token in the response. The access token is recommended to be in JWT (JSON Web Token) format. The client can then carry the access token in subsequent requests, and the service does not need to verify the client's identity each time, but only needs to verify the access token.

The following generation process is not required by the specification, but is provided for reference. Implementers can define and implement it as needed.

JWT generation method reference [RFC7519](https://www.rfc-editor.org/rfc/rfc7519).

1. **Generate Access Token**

Assuming the service uses **JWT (JSON Web Token)** as the access token format, JWT typically contains the following fields:

- **header**: Specifies the signing algorithm
- **payload**: Stores user-related information
- **signature**: Signs the `header` and `payload` to ensure their integrity

The payload can include the following fields (other fields can be added as needed):
```json
{
  "sub": "did:wba:example.com:user:alice",  // User DID 
  "iat": "2024-12-05T12:34:56Z",            // Issued time
  "exp": "2024-12-06T12:34:56Z",            // Expiration time
}
```

Implementers can add other security measures in the payload, such as using scope or binding IP addresses.

2. **Return Access Token**
The generated header, payload, and signature are concatenated and URL-safe Base64 encoded to form the final access token. Then, the access token is returned through the Authorization header:

```plaintext
Authorization: Bearer <access_token>
```

3. **Client Send Access Token**
The client sends the access token through the Authorization header field to the service:

```plaintext
Authorization: Bearer <access_token>
```

4. **Service Verify Access Token**
After receiving the client's request, the service extracts the access token from the Authorization header and verifies it, including verifying the signature, verifying the expiration time, and verifying the fields in the payload. The verification method is based on [RFC7519](https://www.rfc-editor.org/rfc/rfc7519).

#### 3.2.4 Error Handling

#### 3.2.4.1 401 Response

When the server fails to verify the signature and requires the client to reinitiate the request, it should return a 401 response.

Additionally, if the server doesn't support recording client request Nonces, or requires clients to always use server-generated Nonces for signing, it may return a 401 response with an authentication challenge containing a Nonce for each initial client request. However, this increases the number of client requests, and implementers can choose whether to use this approach.

Error information is returned through the `WWW-Authenticate` header field, for example:

```plaintext
WWW-Authenticate: Bearer error="invalid_nonce", error_description="Nonce has already been used. Please provide a new nonce.", nonce="xyz987"
```

Contains the following fields:
- **error**: Required field, error type, containing the following string values:
  - **invalid_request**: Request format error, missing required fields, or contains unsupported parameters.
  - **invalid_nonce**: Nonce has already been used.
  - **invalid_timestamp**: Timestamp is out of range.
  - **invalid_did**: DID format error, or unable to find corresponding DID document.
  - **invalid_signature**: Signature verification failed.
  - **invalid_verification_method**: Unable to find corresponding public key based on verification method.
  - **invalid_access_token**: Access token verification failed.
  - **forbidden_did**: DID lacks permission to access server resources.
- **error_description**: Optional field, error description.
- **nonce**: Optional field, server-generated random string. If present, the client must use this Nonce to regenerate the signature and reinitiate the request.

When the client receives a 401 response, if the response contains a Nonce, the client must use the server's Nonce to regenerate the signature and reinitiate the request. If the response doesn't contain a Nonce, the client must use a client-generated Nonce to regenerate the signature and reinitiate the request.

It's important to note that both client and server implementations should limit the number of retry attempts to prevent infinite loops.

#### 3.2.4.2 403 Response

When server authentication succeeds but the DID lacks permission to access server resources, a 403 response should be returned.

## 4. Cross-Platform Identity Authentication Process Based on did:wba Method and JSON Format Data

In the previous chapter, we introduced the cross-platform identity authentication process based on the did:wba method and HTTP protocol. However, identity authentication using the did:wba method is transport protocol agnostic. Here we define a cross-platform identity authentication process based on the did:wba method and JSON format data, which can be used in scenarios where JSON format is used for communication. This specification only describes the process of using JSON for identity authentication, and how to pass JSON is left to the implementer to decide.

Theoretically, protocols based on other data formats can also add support for the did:wba method.

The overall process is as follows:

```mermaid
sequenceDiagram
    participant Agent A Client
    participant Agent B Server 
    participant Agent A DID Sever

    Note over Agent A Client,Agent B Server: Initial Request

    Agent A Client->>Agent B Server: Authentication Request Info: DID,Signature
    Agent B Server->>Agent A DID Sever: Get DID Document
    Agent A DID Sever->>Agent B Server: DID Document

    Note over Agent B Server: Authentication

    Agent B Server->>Agent A Client: Authentication Response Info: access token

    Note over Agent A Client, Agent B Server: Subsequent Requests

    Agent A Client->>Agent B Server: Request: access token
    Agent B Server->>Agent A Client: Response
```

### 4.1 Initial Request

When the client first initiates a request to the server, authentication needs to be performed according to the following method.

#### 4.1.1 Authentication Information Data Format

The client needs to send the following information to the server:
- **did**: The client's DID identifier in the request, used for identity verification.
- **nonce**: A randomly generated string used to prevent replay attacks. Must be unique for each request. A 16-byte random string is recommended.
- **timestamp**: The time when the request is initiated, typically in ISO 8601 UTC format, accurate to seconds.
- **verification_method**: Identifies the verification method used in the signature, which is the DID fragment of the verification method in the DID document. For example, for the verification method ID "did:wba:example.com%3A8800:user:alice#key-1", the verification method's DID fragment is "key-1".
- **signature**: Signs the `nonce`, `timestamp`, server domain, and client DID. For ECDSA signatures, uses R|S format. Includes the following fields:
  - `nonce`: A randomly generated string
  - `timestamp`: The time when the request is initiated
  - `aud`: The server's domain name (note: the domain name does not include the port)
  - `did`: The client's DID

Client request example:

```json
{
  "did": "did:wba:example.com%3A8800:user:alice",
  "nonce": "abc123",
  "timestamp": "2024-12-05T12:34:56Z",
  "verification_method": "key-1",
  "signature": "base64url(signature_of_nonce_timestamp_aud_did)"
}
```

The authentication information can be sent in a separate request or together with the business request data.

#### 4.1.2 Signature Generation Process

Same as [Section 3.1.2 Signature Generation Process](#312-signature-generation-process).

### 4.2 Server Verification

#### 4.2.1 Verify Authentication Request

The verification process is the same as [Section 3.2.1 Verify Request Header](#321-verify-request-header). The difference is that the did, nonce, timestamp, verification_method, and signature fields need to be extracted from the request data.

After verification passes, the server can return a access token, and the client carries the access token in subsequent requests. The server doesn't need to verify the client's identity each time but only needs to verify the access token.

The access token generation method is the same as [Section 3.2.4 Return Access Token After Successful Authentication](#324-authentication-success-return-access-token).

Response JSON example:

```json
{
  "code": 200,
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c"
}
```

Field descriptions:
- **code**: Status code, using HTTP status codes.
- **access_token**: Access token returned after successful authentication.

When the client receives a 200 response, it can carry the access token in subsequent requests.

Subsequent request example:

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c"
}
```

#### 4.2.2 Error Handling

Error handling follows the same principles as [Section 3.2.4 Error Handling](#324-error-handling).

Example of 401 response in JSON format:

```json
{
  "code": 401,
  "error": "invalid_nonce",
  "error_description": "Nonce has already been used. Please provide a new nonce.",
  "nonce": "1234567890"
}
```

Example of 403 response in JSON format:

```json
{
  "code": 403,
  "error": "forbidden_did",
  "error_description": "did not have permission to access the resource."
}
```

## 5 Distinguishing Between Human Authorization and Agent Automatic Authorization

For requests that are not very important, the user agent can authorize automatically. For example, when accessing a hotel agent and reading hotel information, no human manual confirmation is needed, and the user agent can initiate the request on behalf of the human.

For important requests, such as booking a hotel room, the hotel agent may require human manual confirmation. When the user agent initiates a booking request, it needs to sign using the method defined in humanAuthorization. At this time, the user agent needs to request authorization from a human for manual confirmation before proceeding with the booking request.

The signature method is the same as [Section 3.1 Initial Request](#31-initial-request).

Developers of user agents need to securely keep the private key of humanAuthorization and implement access control, such as allowing the use of humanAuthorization for signing only after biometric verification (fingerprint, facial recognition, etc.) of the user.

An agent can define the authorization type of the document or interface in the agent description document. By default, all ordinary authorizations are sufficient. If a request requires human manual authorization, it must be explicitly defined in the document (refer to the agent description specification for the definition method).

## 6 Privacy Protection Strategy

Privacy protection is crucial in decentralized networks. For instance, malicious software might use a user's DID to record and track their behavior, leading to privacy breaches.

Therefore, we recommend that DID providers adopt a multi-DID strategy, where multiple DIDs are generated for a single user. Each DID has different roles and permissions, using different key pairs, to achieve privacy protection and fine-grained access control.

For example, a main DID can be generated for the user, which generally remains unchanged and is used for maintaining social relationships. Additionally, a series of sub-DIDs can be generated for scenarios such as shopping, ordering takeout, and booking tickets. These sub-DIDs are subordinate to the main DID and can periodically deactivate expired DIDs and apply for new ones to enhance privacy and security protection.

## 7 Security Considerations

Implementers need to consider the following security issues when implementing:

1. Key Management
- The private key corresponding to the DID **must** be properly kept and never leaked. Additionally, a regular refresh mechanism for private keys **should** be established.
- Users **should** generate multiple DIDs, each with different roles and permissions, using different key pairs to achieve fine-grained access control.

2. Anti-Attack Measures
- The server **must** keep a record of the Nonce in requests to prevent replay attacks.
- The server **must** check the timestamp in requests to prevent time rollback attacks. Generally, the server's nonce cache duration **should** be longer than the timestamp expiration duration.
- When generating a Nonce, **must** use a secure random number generator provided by the operating system to meet modern cryptographic security standards and norms. For example, you can use a module like Python's `secrets` to generate secure random numbers.

3. Transport Security
- When retrieving DID documents, the server **should** use the DNS-over-HTTPS (DoH) protocol to enhance security.
- The transport protocol **must** use HTTPS, and clients **must** strictly verify whether the CA certificate of the other party is trustworthy.

4. Token Security
- Both client and server **must** properly keep the Access Token and **must** set a reasonable expiration time.
- **Should** include additional security information in the Access Token, such as client IP binding, User-Agent binding, etc., to prevent token abuse.


## 8. Use Cases

1. Use Case 1: User Accessing Files on Other Websites via Intelligent Assistant

Alice has stored a file on example.com and wants to access it through her intelligent assistant. First, Alice creates a did:wba-based DID on her intelligent assistant, logs into example.com, associates this DID with her account, and grants file access permissions to the DID. After setup, the intelligent assistant can log into example.com using this DID. Upon successful authentication, example.com allows the intelligent assistant to access Alice's stored files. This DID can also be configured on other websites to enable the intelligent assistant to access files across different platforms.

2. Use Case 2: User Calling Third-party Service APIs via Intelligent Assistant

Alice wants to call APIs of a third-party service named "example" through her intelligent assistant. First, Alice creates a did:wba-based DID on her intelligent assistant and uses it to subscribe to services on the example platform. The example service authenticates the DID, confirms Alice as the purchaser, and records her DID. After authentication, Alice can use her intelligent assistant to call example service APIs using this DID.

> Note: While client-to-server authentication is not illustrated in the current use cases, this process can still function effectively.

## 9. Summary

This specification builds upon the did:web method specification by adding DID document constraints, cross-platform authentication processes, and agent description services. It proposes a new method name did:wba (Web-Based Agent). We designed a cross-platform authentication process based on the did:wba method and HTTP protocol, and provided detailed implementation methods.

In the future, we will further improve the did:wba method by adding agent capability and protocol description service endpoints, bidirectional authentication processes between agents, and more.

## References

1. **DID-CORE**. Decentralized Identifiers (DIDs) v1.0. Manu Sporny; Amy Guy; Markus Sabadello; Drummond Reed. W3C. 19 July 2022. W3C Recommendation. Retrieved from [https://www.w3.org/TR/did-core/](https://www.w3.org/TR/did-core/)

2. **did:web**. Retrieved from [https://w3c-ccg.github.io/did-method-web/](https://w3c-ccg.github.io/did-method-web/)

3. **JSON Canonicalization Scheme (JCS)**. Retrieved from [https://www.rfc-editor.org/rfc/rfc8785](https://www.rfc-editor.org/rfc/rfc8785)

4. **RFC 1035**. Domain names - implementation and specification. P. Mockapetris. IETF. November 1987. Internet Standard. Retrieved from [https://www.rfc-editor.org/rfc/rfc1035](https://www.rfc-editor.org/rfc/rfc1035)

5. **RFC 1123**. Requirements for Internet Hosts - Application and Support. R. Braden, Ed. IETF. October 1989. Internet Standard. Retrieved from [https://www.rfc-editor.org/rfc/rfc1123](https://www.rfc-editor.org/rfc/rfc1123)

6. **RFC 2119**. Key words for use in RFCs to Indicate Requirement Levels. S. Bradner. IETF. March 1997. Best Current Practice. Retrieved from [https://www.rfc-editor.org/rfc/rfc2119](https://www.rfc-editor.org/rfc/rfc2119)

7. **RFC 2181**. Clarifications to the DNS Specification. R. Elz; R. Bush. IETF. July 1997. Proposed Standard. Retrieved from [https://www.rfc-editor.org/rfc/rfc2181](https://www.rfc-editor.org/rfc/rfc2181)

8. **RFC 8174**. Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words. B. Leiba. IETF. May 2017. Best Current Practice. Retrieved from [https://www.rfc-editor.org/rfc/rfc8174](https://www.rfc-editor.org/rfc/rfc8174)

9. **RFC 8484**. DNS Queries over HTTPS (DoH). P. Hoffman; P. McManus. IETF. October 2018. Proposed Standard. Retrieved from [https://www.rfc-editor.org/rfc/rfc8484](https://www.rfc-editor.org/rfc/rfc8484)

10. **DID Use Cases**. Decentralized Identifier Use Cases. Joe Andrieu; Kim Hamilton Duffy; Ryan Grant; Adrian Gropper. W3C. 24 June 2021. W3C Note. Retrieved from [https://www.w3.org/TR/did-use-cases/](https://www.w3.org/TR/did-use-cases/)

11. **DID Extensions**. Decentralized Identifier Extensions. Orie Steele; Manu Sporny. W3C. 24 June 2021. W3C Note. Retrieved from [https://www.w3.org/TR/did-extensions/](https://www.w3.org/TR/did-extensions/)

12. **DID Extension Properties**. Decentralized Identifier Extension Properties. Orie Steele; Manu Sporny. W3C. 24 June 2021. W3C Note. Retrieved from [https://www.w3.org/TR/did-extensions-properties/](https://www.w3.org/TR/did-extensions-properties/)

13. **DID Extension Methods**. Decentralized Identifier Extension Methods. Orie Steele; Manu Sporny. W3C. 24 June 2021. W3C Note. Retrieved from [https://www.w3.org/TR/did-extensions-methods/](https://www.w3.org/TR/did-extensions-methods/)

14. **DID Extension Resolution**. Decentralized Identifier Extension Resolution. Orie Steele; Manu Sporny. W3C. 24 June 2021. W3C Note. Retrieved from [https://www.w3.org/TR/did-extensions-resolution/](https://www.w3.org/TR/did-extensions-resolution/)


15. **Controller Document**. Controller Document. Manu Sporny; Markus Sabadello. W3C. 24 June 2021. W3C Note. Retrieved from [https://www.w3.org/TR/controller-document/](https://www.w3.org/TR/controller-document/)

## Copyright Notice
Copyright (c) 2024 GaoWei Chang  
This file is released under the [MIT License](./LICENSE). You are free to use and modify it, but you must retain this copyright notice.
