# Week 8 — Enterprise Security & Identity

## Bird's eye view

- Enterprise app security rests on two pillars: **authentication** (who are you?) and **authorization** (what can you do?), controlled at the gateway level via **SSO** and propagated inward via **JWT**.
- The three control postures are **prevent** (threat modeling, encryption, authN/authZ), **detect** (firewalls, intrusion alerts), and **respond** (patching, backups, automated rebuild).
- **Credentials** split into user credentials (most vulnerable) and secrets (API keys, TLS certs, SSH keys); both require rotation, revocation, and scope limitation.
- **Data in transit** is secured with **HTTPS/TLS** (confidentiality + integrity + server identity); **mTLS** adds client identity for MS-to-MS calls.
- **Data at rest** is secured with salted password hashing, encryption-key separation, and GDPR-driven data minimization.
- **Zero trust** treats every process as potentially compromised; implicit trust is a deliberate trade-off, not a default.
- **OAuth 2.0** is the industry standard for delegated authorization; **OpenID Connect (OIDC)** layers identity (ID tokens) on top.
- **JWT** (JSON Web Token) is the dominant access-token format: a self-contained, signed, expirable claims envelope that microservices can verify without calling back to the gateway.

---

## Detailed notes

### 1. Principles of enterprise app security

Two foundational principles govern every design decision:

- **Principle of Least Privilege** — grant the minimum access level to individuals or services for the minimum time needed to accomplish the task. If credentials are compromised, the blast radius is as small as possible.
- **Defense in Depth** — layer multiple independent protections rather than relying on a single barrier. In microservice architectures (MSA) this is natural because each MS has limited responsibility and a separate security boundary.

#### 1.1 Control types

| Goal | Mechanism |
|---|---|
| **Prevent** | Threat modeling, secret storage, encryption at rest and in transit, authN, authZ |
| **Detect** | Intrusion alert systems, firewalls, anomaly monitoring |
| **Respond** | Automatic rebuild/patching, off-site backups, restorations, communication plans |

---

### 2. Credentials

A credential is anything that grants a person or program access to a restricted resource (database, system, user account, remote API, etc.). In an MSA there are many credentials to manage across many services.

**Two types:**
1. **User credentials** — the most vulnerable; username/password pairs or equivalent.
2. **Secrets** — pieces of information an MS needs to operate: API keys, TLS certificates, SSH keys, database passwords.

**Critical rule:** avoid "simplicity" — do not grant broader privileges to fewer credentials. Prefer many scoped credentials over one powerful one.

**Credential lifecycle management (applies to both):**
- **Rotation** — use tokens/keys with expiration dates; change them regularly rather than relying on username/password pairs.
- **Revocation** — ability to invalidate a credential immediately on compromise.
- **Scope limitation** — each credential grants access only to what is strictly needed.

#### 2.1 Secrets in depth

Examples of secrets: TLS certificates, SSH keys, public/private keys for APIs, database usernames and passwords.

Lifecycle of a secret:
1. **Creation** — how is it created and by whom?
2. **Distribution** — how does it reach the right place and only that place?
3. **Storage** — is it stored so that only authorized parties can access it?
4. **Monitoring** — do you know how and when this secret is used?
5. **Rotation (including revocation)** — can you change it without causing downtime?

Tools: **HashiCorp Vault**, **Spring Vault**, cloud providers (AWS Secrets Manager, Azure Key Vault) — they handle app-config synchronization and secret rotation.

#### 2.2 Limiting scope (diagram description)

The slide shows two diagrams side by side. On the left, all three Inventory instances share one credential (`User: Inventory, Password: 123`) against the Inventory database, which itself uses a read-only credential against Debezium. This is convenient but risky. On the right, each Inventory instance has its own credential (Inventory-A/123, Inventory-B/456, Inventory-C/789), so a breach of one credential exposes only one instance, not the whole DB tier.

---

### 3. Patching

Modern infrastructure has many layers, each requiring maintenance:

```
Your microservice
Container OS
Container
Kubernetes subsystems
VM OS (VMOS)
Virtual Machine (VM)
Hypervisor
Operating System
Underlying hardware
```

In a managed Kubernetes provider setup, the provider takes responsibility for hypervisor down to underlying hardware; you remain responsible for the container OS upward.

**Tools:**
- **Aqua** — analyzes Docker containers for vulnerabilities.
- **Snyk** / **GitHub Code Scanning** — integrate into CI/CD to scan library code for known vulnerabilities.

---

### 4. Backups and rebuilds

- Databases and app logs are valuable; always back them up to a **secure, off-site location** isolated from the app environment.
- Avoid "dark" backups — periodically **test restoration** (e.g., restore to a load-test database) to confirm they are actually restorable.
- Automate regular rebuild and restart of microservices to prevent persistence of malicious code that may have been installed.

---

### 5. Zero trust vs. implicit trust

In MSA, many entities interact: users, front-ends, back-end microservices, MS-to-MS calls. The question is whether to trust every process on the internal network (implicit trust / perimeter model) or assume the environment is already compromised (zero trust).

- **Implicit trust** — tolerable in low-criticality apps; simpler to implement but higher risk.
- **Zero trust** — operate as if every request could come from an attacker; verify every interaction regardless of origin. Requires mTLS, token-based authN/authZ on internal calls, and strict network policies.

The right choice depends on the criticality of the application.

---

### 6. Data in transit — four concerns

The four questions to answer for any MS-to-MS or client-to-server communication:

| Concern | Question |
|---|---|
| **Visibility (confidentiality)** | Can anyone see the data being transmitted? |
| **Manipulation (integrity/tampering)** | Can anyone change the data in transit? |
| **Client identity** | Is the client who it claims to be? |
| **Server identity** | Is the server who it claims to be? |

#### 6.1 Server identity

- **HTTPS (HTTP over TLS)** allows the client to verify the server's certificate against a trusted Certificate Authority (CA), ensuring it is communicating with the legitimate server.

#### 6.2 Client identity

- The client can be asked to include identifying information in the request: a secret, or a **client certificate** to sign the request.
- **Mutual TLS (mTLS)** — certificates in both directions. The server also authenticates the client's certificate. This is the standard for secure MS-to-MS communication, often managed by **service meshes** (e.g., Istio, Linkerd).

#### 6.3 Data visibility and tampering

- **HTTPS/TLS** encrypts transmitted data (confidentiality) and prevents tampering (integrity via MAC).
- Downside: encrypted payloads cannot be cached by reverse proxies.
- If data must be sent in plaintext but needs tamper protection, transmit an **HMAC** of the data alongside it; the server re-computes and verifies.

---

### 7. Data at rest

- Use well-known, vetted encryption algorithms; subscribe to security mailing lists for vulnerability patches.
- **Password hashing:** always use salts. Reference: crackstation.net/hashing-security. If a password is brute-forced, the salt prevents cracking other passwords with the same hash. Use adaptive algorithms: **bcrypt**, **argon2**, **scrypt**.
- **Data minimization:** store only essential data (GDPR obligation for personal data).
- **Key separation:** store encryption/decryption keys on systems other than where the data lives.
- **Early encryption:** encrypt data at the first point it is seen (e.g., the frontend) and decrypt only on demand.

---

### 8. Authentication and authorization

**Authentication (authN)** — "Who am I?" Verifies identity.
**Authorization (authZ)** — "What can I do?" Grants or denies access to resources based on verified identity.

#### 8.1 SSO and Identity Providers

Asking for username/password on every service call is unusable. The solution is **Single Sign-On (SSO):**

- Implemented at the **API Gateway** level in MSA.
- The gateway redirects unauthenticated requests to an **Identity Provider (IdP)** such as:
  - External providers: Google, GitHub.
  - Enterprise IdPs: **Keycloak** (open-source), or directory services like **LDAP / Active Directory**.
- Once authenticated, the gateway passes identity information (e.g., roles) to downstream microservices via HTTP headers.
- **Multi-Factor Authentication (MFA)** adds a second factor (OTP, hardware key) to strengthen authN.

**SSO Gateway flow (diagram description):**
The browser hits the SSO gateway. If the user is not authenticated, the gateway redirects to the Identity Provider. The IdP authenticates the user and redirects back. The gateway then allows calls to downstream microservices (Catalog, Customer, Recommendation), forwarding identity information in headers.

#### 8.2 Authorization granularity

- The gateway handles **coarse-grained authZ**: assign roles/groups to users; allow or deny access to endpoints based on role.
- **Fine-grained authZ** (within a resource) is left to the MS's discretion.
- Avoid overly specific roles per MS (e.g., `OPERATOR_ALLOWED_TO_REFUND_MORE_THAN_100_EUROS`) — too complex to maintain.
- The MS uses **JWT** to carry the claims it needs to make fine-grained decisions locally.

#### 8.3 Centralized vs. decentralized authorization

| Model | Description | Trade-off |
|---|---|---|
| **Centralized (gateway)** | Gateway verifies all authZ before forwarding | Simple to implement; changing MS authZ requires changing gateway code |
| **Decentralized (MS)** | Each MS decides based on claims in the JWT | More flexible; no gateway coupling; MS must validate JWT itself |

The most common approach: gateway handles coarse authZ, MS handles fine authZ using JWT claims.

---

### 9. JSON Web Tokens (JWT)

JWT is a compact, URL-safe string that encodes multiple "claims" about a user or entity. It can be signed (to prevent tampering) and optionally encrypted.

#### 9.1 Structure: Header.Payload.Signature

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjMiLCJuYW1lIjoiU2FtIE5ld21hbiIsImV4cCI6MTYwNjc0MTczNiwiZ3JvdXBzIjpbImFkbWluIiwiYmV0YSJdfQ.
<signature>
```

Three parts, each Base64URL-encoded and separated by dots:

| Part | Content |
|---|---|
| **Header** | Metadata: algorithm (e.g., `HS256`), token type (`JWT`) |
| **Payload** | Claims: the actual data |
| **Signature** | HMAC or RSA/ECDSA signature — prevents tampering; computed from header + payload + server secret |

#### 9.2 Claims (payload example)

```json
{
  "sub": "123",
  "name": "Sam Newman",
  "exp": 1606741736,
  "groups": ["admin", "beta"]
}
```

- `sub` — subject identifier (standard public claim).
- `exp` — expiration time as Unix timestamp (standard public claim).
- `groups` — custom private claim used for authZ decisions.

#### 9.3 Usage

The client includes the JWT in every HTTP request:

```
Authorization: Bearer <token>
```

The receiving microservice validates the signature locally (no round-trip to the IdP) and reads the claims to make authZ decisions.

#### 9.4 JWT in an MS architecture (flow description)

1. Browser authenticates and receives an OAuth token from the SSO gateway.
2. SSO gateway issues a JWT.
3. The BFF (Backend for Frontend) / Web Shop forwards the JWT to downstream services (Shipping, Order).
4. Each MS validates the JWT's signature using the shared secret or the IdP's public key.
5. JWT can serve as the access token in OAuth 2.0 flows.

---

### 10. Identity Management

**Identity management** involves creating, maintaining, and managing digital identities within an organization or system, ensuring each individual or entity has a unique identity that can be authenticated and authorized.

- **Authentication** uses these identities to verify users.
- **Authorization** uses verified identities and their attributes to determine access rights.
- **Access Control** enforces rights based on policies.

#### 10.1 Kinds of Identity Management Systems

| Type | Characteristics | Example |
|---|---|---|
| **Centralized** | Single authority; unified control; easier management; single point of failure | University IT managing all logins for email, library, course access |
| **Federated** | Multiple trusted domains; shared authentication; improved scalability; requires trust agreements | Company allowing employees to use corporate credentials for third-party services |
| **Decentralized** | No central authority; user-controlled data; enhanced privacy; complex management | Blockchain-based Self-Sovereign Identity (SSI) |

OAuth 2.0 and OpenID Connect are the technical framework for **federated** identity management.

---

### 11. OAuth 2.0

#### 11.1 What is OAuth 2.0?

**OAuth** stands for **Open Authorization**. It is a standard that allows an application to access a protected resource hosted by another application on behalf of a user, without ever sharing the user's credentials.

- Replaced OAuth 1 in 2012; became the industry standard for online authorization.
- **OAuth 2 is an authorization protocol, NOT an authentication protocol.** (Authentication is handled by OpenID Connect built on top of it.)

**The CERN analogy:** You are an engineer; an intern needs to retrieve a USB key from your office. Rather than giving the intern your badge (credentials), you send them to reception (Authorization Server). Reception verifies both parties and issues a temporary short-term badge (access token) scoped to the specific office (Protected Resource) for a limited time.

**OAuth 2.0 entities:**

| Entity | Role |
|---|---|
| **Protected Resource (PR)** | The API endpoint being accessed |
| **Resource Owner (RO)** | The user or system that owns and can grant access |
| **Resource Server (RS)** | Hosts the API; validates access tokens and returns resources |
| **Authorization Server (AS)** | Receives token requests; authenticates RO; issues tokens. Exposes: (1) authorization endpoint (user-facing), (2) token endpoint (machine-to-machine) |
| **Client** | The application requesting access; must possess a valid access token |

#### 11.2 Access tokens and scope

- OAuth 2.0 uses **access tokens** — data representing authorization to access a resource and perform an action on behalf of the end-user.
- OAuth 2.0 does not mandate a specific token format; in practice **JWT** is used because it allows embedding data (e.g., expiration) in the token itself.
- **Scope** specifies exactly which resources and actions the client is requesting. Presented to the user on a consent screen; the issued token is limited to consented scopes.

#### 11.3 Authorization Code flow (standard flow)

The AS cannot directly return an access token after consent — an untrusted channel (browser redirect) is involved. Instead:

1. Client (browser) sends authorization request to AS including `client_id`, `redirect_uri`, `scope`, `state`.
2. AS authenticates the RO and shows a consent screen.
3. AS returns a **one-time authorization code** via the front-channel (redirect to `redirect_uri`).
4. Client makes a **back-channel** POST request to the token endpoint with the authorization code + `client_id` + `client_secret`.
5. AS returns an **access token** (and optionally a **refresh token**).
6. Client uses the access token to call the Resource Server.
7. RS validates the token and returns the protected resource.

#### 11.4 Refresh tokens

- Issued alongside the access token.
- Access tokens are short-lived (~5 minutes); refresh tokens are long-lived (30 minutes to several days).
- When the access token expires, the client POSTs the refresh token to the token endpoint to get a new access token without involving the user.
- Best practice: issue a new refresh token each time a refresh is used (**refresh token rotation**).

#### 11.5 OAuth 2.0 Grant Types

| Grant Type | Use case | Notes |
|---|---|---|
| **Authorization Code** | Server-side web apps | Standard flow; most secure for confidential clients |
| **Authorization Code + PKCE** | SPAs, mobile apps | Recommended for public clients that cannot store a client secret |
| **Implicit** (deprecated) | Was used for SPAs | AS returns token directly in redirect; risk of token leakage |
| **Resource Owner Credentials** | First-party apps with full trust | Client collects and sends user's credentials directly; less secure |
| **Client Credentials** | MS-to-MS / non-interactive | Client ID and secret used directly; no user involved |
| **Device Authorization** | Smart TVs, constrained input devices | Out-of-band user authorization |

#### 11.6 PKCE (Proof Key for Code Exchange)

PKCE is an extension to the Authorization Code flow, now recommended for all public clients (SPAs, mobile) and increasingly for server-side apps too. It prevents CSRF attacks and authorization code injection attacks.

**How PKCE works:**

Step 1 — Authorization Request:
- Client generates a **code verifier**: random string 43–128 characters.
- Client generates a **code challenge**: `BASE64URL(SHA256(code_verifier))`.
- Client sends `code_challenge` and `code_challenge_method=S256` in the authorization request.
- AS stores the code challenge alongside the returned authorization code.

Step 2 — Token Request:
- Client sends the authorization code + the original **code verifier** (not the challenge) to the token endpoint.
- AS recomputes `SHA256(code_verifier)` and compares with the stored code challenge.
- If they match, the token is issued. If not, responds with `invalid_grant`.

```
# Authorization Request (GET)
https://as/authorize
  ?response_type=code
  &client_id=CID-XXX
  &redirect_uri=https://client/callback
  &scope=scope1+scope2
  &state=AS-YYY
  &code_challenge_method=S256
  &code_challenge=ZZZZ

# Authorization Response (redirect)
GET https://client/callback?code=AC-XXX&state=AS-YYY

# Access Token Request (POST)
POST https://as/token
  grant_type=authorization_code
  &code=AC-XXX
  &redirect_uri=https://client/callback
  &code_verifier=AAA

# Access Token Response (HTTP 201 JSON)
{ "access_token": "AT-XXX", "refresh_token": "RT-YYY" }
```

After token expiry, the client retries with the expired token (gets 403 `Bearer error="invalid_token"`), then POSTs the refresh token to get a new access token and refresh token.

#### 11.7 Other OAuth 2.0 considerations

- The **`state` parameter** serves dual purposes: (1) session key for redirect routing after authZ; (2) CSRF protection when it contains a per-request random value.
- Token verification by the RS: OAuth 2.0 does not specify how; in practice the RS and AS share the token (same system), or the RS calls a protected token-introspection API.
- Client registration: clients register via a web page or the **Dynamic Client Registration** spec to obtain `client_id` and `client_secret`, specifying `redirect_uris` and whether the client is confidential or public.
- Client authentication at the AS (in the AC flow): HTTP Basic Auth (`client_id` as username, `client_secret` as password) or POST body with both values.

---

### 12. OpenID Connect (OIDC)

OAuth 2.0 handles authorization but not identity. **OpenID Connect** is a protocol built on top of OAuth 2.0 that adds identity:

- Specifies how **ID tokens** (in addition to access tokens) should be distributed.
- The key addition to the authorization request is `scope=openid`. If `openid` is not in the scope, a standard OAuth 2.0 flow executes (no ID token).
- Adds `id_token` as a valid value for `response_type` (alongside `code` and `token`).

**OIDC flow with `response_type=code` and `scope` including `openid`:**

1. Client sends authentication + authorization request to Authorization Endpoint.
2. User authenticates and authorizes.
3. Server returns authorization code.
4. Client sends token request to Token Endpoint.
5. Server returns both an **ID Token** (identity, who the user is) and an **Access Token** (authorization, what they can do).

**Federated identity summary:**

- **Identity Providers (IdPs)** manage user identities and handle authentication. OIDC allows IdPs to authenticate users and provide identity to relying parties.
- **Service Providers (SPs)** trust IdPs for authN and use OAuth 2.0 for authZ. This enables accessing multiple services with a single set of credentials (SSO).
- **Trust relationships** between IdPs and SPs are the foundation of federated identity. OAuth 2.0 and OIDC provide the technical framework for establishing and managing these trust relationships securely.

---

## Key terms

| Term | Definition |
|---|---|
| **Authentication (authN)** | Verifying the identity of a user or system ("who are you?") |
| **Authorization (authZ)** | Determining what an authenticated identity is allowed to do ("what can you do?") |
| **Identity Provider (IdP)** | A system that manages and authenticates user identities (e.g., Keycloak, Google, Active Directory) |
| **Service Provider (SP)** | A system that trusts an IdP for authN and uses OAuth 2.0 for authZ |
| **Single Sign-On (SSO)** | One authentication event grants access to multiple services |
| **Federation** | Multiple trusted organizations sharing authentication across domains |
| **JWT (JSON Web Token)** | Compact signed token encoding claims; format: Header.Payload.Signature |
| **Claim** | A key-value assertion in a JWT payload (e.g., `sub`, `exp`, `groups`) |
| **OAuth 2.0** | Delegated authorization framework; enables a client to access a resource on behalf of a user |
| **Access Token** | Short-lived token authorizing access to a specific resource; typically JWT |
| **Refresh Token** | Long-lived token used to obtain new access tokens without user re-authentication |
| **Authorization Code** | One-time code issued by AS to the client via front-channel; exchanged for tokens via back-channel |
| **PKCE** | Proof Key for Code Exchange; CSRF-resistant extension to Auth Code flow using code verifier/challenge |
| **OpenID Connect (OIDC)** | Identity layer on top of OAuth 2.0; adds ID tokens specifying who the user is |
| **ID Token** | JWT issued by OIDC containing user identity information |
| **Client Credentials Grant** | OAuth 2.0 grant for MS-to-MS non-interactive authorization |
| **Resource Owner** | The user or entity that owns and can grant access to a protected resource |
| **Authorization Server (AS)** | Issues tokens after authenticating the resource owner and obtaining consent |
| **Resource Server (RS)** | Hosts the protected API; validates access tokens |
| **Scope** | Declared set of permissions requested by the client; shown to user on consent screen |
| **mTLS (Mutual TLS)** | TLS with certificate verification in both directions; used for MS-to-MS identity |
| **HMAC** | Hash-based Message Authentication Code; used to verify data integrity without full encryption |
| **Secret** | Sensitive information an MS needs to operate (API key, cert, password); requires lifecycle management |
| **Least Privilege** | Security principle: grant only the minimum access needed |
| **Defense in Depth** | Security principle: use multiple independent layers of protection |
| **Zero Trust** | Security model: assume the environment is hostile; verify every request regardless of origin |
| **Implicit Trust** | Perimeter security model: trust everything inside the network boundary |
| **RBAC** | Role-Based Access Control: authZ decisions based on user roles |
| **ABAC** | Attribute-Based Access Control: authZ decisions based on multiple attributes (role, time, location) |
| **MFA** | Multi-Factor Authentication: requires two or more verification factors |
| **code verifier** | Per-request random string (43–128 chars) generated by client in PKCE |
| **code challenge** | SHA256 hash of code verifier, sent in authorization request in PKCE |
| **Implicit Grant** | Deprecated OAuth 2.0 flow; AS returns token directly in redirect (token leakage risk) |
| **state parameter** | Random value in OAuth request; used for CSRF protection and redirect routing |
| **Token rotation** | Issuing a new refresh token each time the previous one is used |
| **Salt** | Random value added to a password before hashing; prevents rainbow table attacks |

---

## Exam targets

1. **Distinguish authN from authZ** — give precise definitions and explain how SSO at the gateway handles authN while JWT enables fine-grained authZ inside the MS.
2. **Explain the SSO gateway pattern** — draw: Browser → SSO Gateway → IdP (redirect for unauthenticated) → downstream MSes (with JWT in headers). State what the gateway checks vs. what each MS checks.
3. **Decode a JWT** — given a base64url-encoded token, identify the three parts (header, payload, signature); name the standard claims (`sub`, `exp`) and explain how the signature prevents tampering.
4. **Walk through the OAuth 2.0 Authorization Code flow** — all five steps (authZ request, consent, code return, back-channel token exchange, RS call); identify front-channel vs. back-channel.
5. **Explain PKCE** — why it is needed (public clients cannot store `client_secret`; prevents code injection and CSRF); how `code_verifier` and `code_challenge` work; what happens on mismatch.
6. **Compare the five OAuth 2.0 grant types** — use case, security level, and whether a user is involved.
7. **Explain how OIDC extends OAuth 2.0** — what `scope=openid` triggers; the difference between an ID token (identity) and an access token (authorization).
8. **Compare centralized, federated, and decentralized identity management** — give one concrete example of each; state why OAuth 2.0 + OIDC are specifically federated-identity tools.
9. **Explain the four concerns for data in transit** — visibility, manipulation, server identity, client identity — and the mechanism that addresses each (TLS, HMAC, HTTPS cert verification, mTLS).
10. **Describe the secrets lifecycle** — creation, distribution, storage, monitoring, rotation/revocation; name one tool (Vault) and explain why scope limitation matters with the two-diagram example.
11. **Compare zero trust vs. implicit trust** — when each is appropriate; what technical controls zero trust requires (mTLS, token authN on internal calls).
12. **Explain refresh tokens** — why access tokens are short-lived; how a client uses a refresh token after expiry; what refresh token rotation achieves.

---

## Pitfalls

- **OAuth 2.0 is NOT an authentication protocol.** It is for authorization (delegated access). Authentication of the user is done by the IdP; OIDC (not OAuth 2.0 alone) is the authentication layer. A common exam trap is saying "OAuth 2.0 authenticates the user."
- **JWT's signature does not encrypt the payload.** Anyone with the token can base64url-decode the payload and read the claims. Do not put secrets in JWT claims unless the token is also encrypted (JWE). Only the signature prevents tampering.
- **The Implicit Grant is deprecated.** The reason: the access token is returned in the URL fragment (front-channel / browser redirect), where it can be captured. Always use Authorization Code (+ PKCE) instead.
- **PKCE does not replace `client_secret` for confidential clients.** It is an additional protection layer. Confidential clients still send `client_id` + `client_secret` at the token endpoint; they also add PKCE on top.
- **The `state` parameter is the developer's responsibility.** OAuth 2.0 defines it but does not enforce its use for CSRF. If you omit a random, per-request state, you are vulnerable to CSRF on the redirect.
- **A refresh token is not an access token.** It cannot be used directly to access a resource. It is only presented to the AS token endpoint to get a new access token.
- **Centralized authZ at the gateway creates coupling.** Every time an MS adds a new authZ rule, the gateway must change. Decentralized (JWT-based) authZ avoids this but requires each MS to validate JWT signatures.
- **Secondary NameNode for HDFS vs. IdP — unrelated.** Do not confuse enterprise identity concepts with distributed-storage checkpointing concepts.
- **mTLS ≠ HTTPS.** Standard HTTPS only authenticates the server. mTLS authenticates both server and client via certificates in both directions. Service meshes (Istio, Linkerd) typically manage mTLS transparently between microservices.
- **Salted hashing ≠ encryption.** Hashing is one-way; you cannot recover the original password. Use bcrypt/argon2/scrypt (adaptive, slow), not SHA-256 or MD5 (fast, GPU-crackable) for passwords.
- **Authorization code is one-time use.** If an attacker intercepts and uses it before the legitimate client, the legitimate exchange will fail. PKCE further hardens this by requiring the code verifier that only the legitimate client knows.
