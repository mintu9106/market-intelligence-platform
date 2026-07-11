# Phase 4 — API Specification Design
## AI-Powered Indian Stock Market Intelligence SaaS Platform

> **Status**: Draft — Pending User Approval
> **Version**: 1.0
> **Depends On**: Phase 3C v1.5 (Physical Database Schema) — Approved & Frozen
> **Deployment Context**: Private single-user production. Interfaces designed for future multi-tenant SaaS expansion.
> **Next Phase**: Phase 5 — Backend Service Architecture

---

## Table of Contents

1. [API Architecture Overview](#1-api-architecture-overview)
2. [API Versioning Strategy](#2-api-versioning-strategy)
3. [Authentication & Authorization](#3-authentication--authorization)
4. [JWT Lifecycle & Refresh Token Strategy](#4-jwt-lifecycle--refresh-token-strategy)
5. [Standard Request & Response Formats](#5-standard-request--response-formats)
6. [Error Handling Specification](#6-error-handling-specification)
7. [Validation Standards](#7-validation-standards)
8. [Idempotency Strategy](#8-idempotency-strategy)
9. [Rate Limiting Strategy](#9-rate-limiting-strategy)
10. [OpenAPI 3.1 Base Configuration](#10-openapi-31-base-configuration)
11. [Identity & Auth APIs](#11-identity--auth-apis)
12. [User & Profile APIs](#12-user--profile-apis)
13. [Subscription & Billing APIs](#13-subscription--billing-apis)
14. [Market Data APIs](#14-market-data-apis)
15. [Portfolio APIs](#15-portfolio-apis)
16. [Watchlist APIs](#16-watchlist-apis)
17. [Scanner APIs](#17-scanner-apis)
18. [AI Signal APIs](#18-ai-signal-apis)
19. [AI Prediction & Explainability APIs](#19-ai-prediction--explainability-apis)
20. [AI Chat APIs](#20-ai-chat-apis)
21. [Alert & Notification APIs](#21-alert--notification-apis)
22. [Broker Integration APIs](#22-broker-integration-apis)
23. [Backtest APIs](#23-backtest-apis)
24. [Webhook Architecture](#24-webhook-architecture)
25. [WebSocket Contracts](#25-websocket-contracts)
26. [Background Job APIs](#26-background-job-apis)
27. [Health, Monitoring & Admin APIs](#27-health-monitoring--admin-apis)
28. [Future Roadmap](#28-future-roadmap)

---

## 1. API Architecture Overview

### 1.1 Architectural Principles

The API layer follows **REST over HTTP/1.1 and HTTP/2**, with **WebSocket** for real-time streaming. All design decisions prioritise forward-compatibility with multi-tenant SaaS without over-engineering for the private v1.0 deployment.

| Principle | Decision | Rationale |
|---|---|---|
| Transport | HTTPS only (TLS 1.3+) | Security non-negotiable |
| Protocol | REST + WebSocket | REST for CRUD, WS for real-time streams |
| API Style | Resource-oriented URLs | Intuitive, cacheable, tooling-friendly |
| Data Format | JSON (UTF-8) | Universal, human-readable |
| Auth | JWT Bearer + Refresh Token | Stateless, scalable |
| Versioning | URL path prefix (`/api/v1`) | Explicit, easy to route |
| Idempotency | `Idempotency-Key` header | Prevents double-writes on retries |
| Pagination | Cursor-based | Stable under concurrent writes |
| Observability | `X-Request-ID`, `X-Trace-ID` headers | End-to-end tracing |

### 1.2 URL Namespace Design

```
Base URL (Private):   https://api.yourdomain.com
Base URL (SaaS-ready): https://{tenant}.yourdomain.com  OR  https://api.yourdomain.com/api/v1

Path structure:
  /api/v{N}/{resource}
  /api/v{N}/{resource}/{id}
  /api/v{N}/{resource}/{id}/{sub-resource}

WebSocket:
  wss://api.yourdomain.com/ws/v1/{stream}

Health:
  /health
  /health/live
  /health/ready
```

### 1.3 API Gateway Responsibilities

For v1.0 private deployment, the API gateway responsibilities are handled by the FastAPI application itself. When scaling to SaaS, an external gateway (Kong, AWS API Gateway, or Nginx) is inserted before the application layer.

**Current (v1.0)**:
```
Client → [TLS Termination / Nginx] → [FastAPI App] → [PostgreSQL / Redis / Kafka]
```

**Future SaaS**:
```
Client → [CDN] → [API Gateway] → [Auth Middleware] → [Microservices] → [Databases]
```

### 1.4 Request Lifecycle

```
1. Client sends HTTPS request with Bearer token + optional Idempotency-Key
2. TLS termination at reverse proxy
3. Auth middleware validates JWT:
   a. Verify signature, expiry, and issuer
   b. Extract tenant_id, user_id, roles from claims
   c. Set app.current_tenant_id on the DB connection (RLS activation)
4. Rate limit check (Redis sliding window counter)
5. Idempotency check (if Idempotency-Key present)
6. Request validation (Pydantic schema)
7. Business logic (Service layer)
8. Response serialisation
9. Audit log written (async, via outbox)
10. Response returned with X-Request-ID header
```

---

## 2. API Versioning Strategy

### 2.1 Version Scheme

- **Current version**: `v1`
- **Version in URL path**: `/api/v1/...`
- **Version in OpenAPI**: `info.version: "1.0.0"` (semver)
- **Deprecation header**: `Sunset: <RFC 7231 date>` and `Deprecation: true`

### 2.2 Versioning Rules

| Rule | Detail |
|---|---|
| Breaking changes require new version | `v1 → v2` when response schema changes incompatibly |
| Additive changes are non-breaking | New optional fields can be added to v1 without bumping |
| Old versions supported for 12 months | After deprecation notice, v_old served for 12 months |
| Version negotiation | URL path only — no `Accept: application/vnd.api+json;version=1` complexity |
| Internal headers | `X-API-Version: 1.0.3` returned in all responses for audit |

### 2.3 Supported Versions Table

| Version | Status | Sunset Date |
|---|---|---|
| v1 | **Active** | TBD |

---

## 3. Authentication & Authorization

### 3.1 Authentication Methods

| Method | Used For | Details |
|---|---|---|
| JWT Bearer | All REST API calls | Short-lived access token (15 min) |
| Refresh Token | Token renewal | Long-lived (30 days), httpOnly cookie |
| API Key | Machine-to-machine, webhooks | SHA-256 hashed, stored in `api_keys` table |
| WebSocket JWT | WS handshake | Token passed as `?token=` query param on initial upgrade |

### 3.2 RBAC Authorization Model

Authorization follows the RBAC model defined in Phase 3A. Every API endpoint declares its required permission code.

**Role → Permission Mapping (default)**:

| Role | Example Permissions |
|---|---|
| `ADMIN` | All permissions |
| `ANALYST` | `signal:read`, `backtest:execute`, `scanner:create`, `portfolio:read` |
| `PREMIUM_USER` | `signal:read`, `alert_rule:create`, `portfolio:manage`, `watchlist:manage` |
| `FREE_USER` | `signal:read` (limited), `watchlist:read` |

**Permission Code Convention**: `{resource}:{action}`
Examples: `portfolio:create`, `signal:read`, `backtest:execute`, `broker:connect`, `admin:manage`

### 3.3 Authorization Header

```
Authorization: Bearer <access_token>
```

For API Key authentication:
```
X-API-Key: <raw_api_key>
```
The raw key is SHA-256 hashed by the server and looked up in `api_keys.key_hash`.

### 3.4 Multi-Tenant Context Injection

For v1.0 (private, single-user), `tenant_id` is resolved from the JWT `tenant_id` claim. For future SaaS, subdomain-based tenant resolution is added at the gateway layer before JWT validation.

```
JWT Claim: { "tenant_id": "uuid", "user_id": "uuid", "roles": ["ADMIN"], "permissions": [...] }
         ↓
Middleware: SET LOCAL app.current_tenant_id = claim.tenant_id;
         ↓
PostgreSQL RLS policies automatically apply
```

---

## 4. JWT Lifecycle & Refresh Token Strategy

### 4.1 Token Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     Access Token (JWT)                   │
│  Algorithm : RS256 (asymmetric — separate sign/verify)  │
│  Expiry    : 15 minutes                                  │
│  Transport : Authorization: Bearer header               │
│  Claims    : tenant_id, user_id, roles, permissions,    │
│              iat, exp, iss, jti (token ID for revocation)│
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                  Refresh Token                           │
│  Format    : Opaque random string (32 bytes, hex)       │
│  Expiry    : 30 days                                     │
│  Storage   : httpOnly, Secure, SameSite=Strict cookie   │
│  DB Record : sessions table (token_hash = SHA-256)      │
│  Rotation  : Issued fresh on every use (single-use)     │
└─────────────────────────────────────────────────────────┘
```

### 4.2 Token Lifecycle

```
POST /api/v1/auth/login
  → Returns: access_token (JSON body) + refresh_token (httpOnly cookie)

Client stores access_token in memory only (NOT localStorage).

On access_token expiry (HTTP 401):
  POST /api/v1/auth/refresh  (cookie sent automatically)
  → Returns: new access_token (body) + rotated refresh_token (cookie)
  → Old refresh_token is immediately invalidated in sessions table

On logout:
  POST /api/v1/auth/logout
  → Deletes session row from sessions table
  → Clears httpOnly cookie
  → Invalidates access_token via JTI blocklist in Redis
     Key: jti_blocklist:{jti}  TTL: remaining access_token lifetime
```

### 4.3 Token Security Rules

| Rule | Implementation |
|---|---|
| RS256 signing | Private key in secrets manager, public key distributed |
| JTI revocation | Redis blocklist checked on every authenticated request |
| Refresh token rotation | Single-use: new token on every refresh, old immediately deleted |
| Refresh token binding | Bound to `user_agent` + `ip_address` — mismatch triggers re-auth |
| MFA enforcement | If `mfa_enabled = true`, token issued with `mfa_verified: false` claim; restricted to MFA endpoints only |
| Absolute session limit | Refresh token max lifetime = 30 days regardless of activity |

---

## 5. Standard Request & Response Formats

### 5.1 Request Conventions

| Convention | Specification |
|---|---|
| Content-Type | `application/json` for all POST/PUT/PATCH |
| Date format | ISO 8601 with timezone: `2025-01-15T09:15:00+05:30` |
| Currency amounts | Integer cents/paise (never floating point): `150000` = ₹1500.00 |
| UUIDs | Lowercase hyphenated: `550e8400-e29b-41d4-a716-446655440000` |
| Booleans | JSON native `true`/`false` (not `"true"`/`"1"`) |
| Pagination | Cursor-based via `cursor` query param |
| Sorting | `sort_by=created_at&order=desc` |
| Filtering | `status=ACTIVE&type=EQUITY` (query params) |

### 5.2 Successful Response Envelope

**Single resource**:
```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "My Portfolio",
    "balance_cents": 100000000,
    "created_at": "2025-01-15T09:15:00+05:30"
  },
  "meta": {
    "request_id": "req_01HX...",
    "api_version": "1.0.0"
  }
}
```

**Paginated list**:
```json
{
  "success": true,
  "data": [ ... ],
  "pagination": {
    "cursor": "eyJpZCI6IjU1MGU4...",
    "next_cursor": "eyJpZCI6Ijc3MWU4...",
    "has_more": true,
    "total_count": 243,
    "page_size": 50
  },
  "meta": {
    "request_id": "req_01HX...",
    "api_version": "1.0.0"
  }
}
```

**No content (DELETE / action endpoints)**:
```json
{
  "success": true,
  "data": null,
  "meta": {
    "request_id": "req_01HX...",
    "api_version": "1.0.0"
  }
}
```

### 5.3 Standard Response Headers

| Header | Value | Purpose |
|---|---|---|
| `X-Request-ID` | `req_01HX4...` | Unique request identifier for tracing |
| `X-API-Version` | `1.0.0` | Served API version |
| `X-RateLimit-Limit` | `1000` | Request limit for the window |
| `X-RateLimit-Remaining` | `847` | Remaining requests in window |
| `X-RateLimit-Reset` | `1705298400` | Unix timestamp when window resets |
| `Idempotency-Replayed` | `true` | Present when response is a cached replay |

---

## 6. Error Handling Specification

### 6.1 Error Response Envelope

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed.",
    "details": [
      {
        "field": "quantity",
        "message": "Quantity must be a positive integer greater than 0.",
        "rejected_value": -5
      }
    ]
  },
  "meta": {
    "request_id": "req_01HX...",
    "api_version": "1.0.0"
  }
}
```

### 6.2 Standard Error Codes

| HTTP Status | Error Code | Meaning |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Request body or query params failed schema validation |
| 400 | `INVALID_REQUEST` | Structurally valid but semantically incorrect request |
| 400 | `INVALID_IDEMPOTENCY_KEY` | Idempotency key reused with different request body |
| 401 | `AUTHENTICATION_REQUIRED` | No or missing Bearer token |
| 401 | `TOKEN_EXPIRED` | Access token has expired |
| 401 | `TOKEN_INVALID` | Token signature/claims invalid |
| 401 | `MFA_REQUIRED` | User has MFA enabled but not verified this session |
| 403 | `PERMISSION_DENIED` | Authenticated but lacks required permission |
| 403 | `SUBSCRIPTION_LIMIT_REACHED` | Feature quota exceeded for current plan |
| 404 | `RESOURCE_NOT_FOUND` | Requested resource does not exist |
| 409 | `CONFLICT` | Optimistic lock violation — retry with latest state |
| 409 | `DUPLICATE_RESOURCE` | Unique constraint violation (e.g. duplicate email) |
| 409 | `INSUFFICIENT_FUNDS` | Portfolio balance insufficient for the operation |
| 422 | `BUSINESS_RULE_VIOLATION` | Valid request violates a domain business rule |
| 429 | `RATE_LIMIT_EXCEEDED` | Too many requests in the sliding window |
| 500 | `INTERNAL_ERROR` | Unexpected server error (no internal details exposed) |
| 503 | `SERVICE_UNAVAILABLE` | Dependency unavailable (DB, Kafka, broker API) |

### 6.3 Error Handling Rules

- **Never expose stack traces** in error responses in production.
- **Never expose SQL error messages** — map to `INTERNAL_ERROR`.
- **Always include `request_id`** so errors can be correlated in logs.
- **Validation errors** must list all failed fields, not just the first.
- **500 errors** must be logged with full stack trace server-side and correlated to `request_id`.

---

## 7. Validation Standards

### 7.1 Validation Layer

All incoming request bodies are validated by Pydantic v2 models before reaching the service layer. Validation is strict — extra fields are rejected unless explicitly allowed.

### 7.2 Validation Rules

| Field Type | Rule |
|---|---|
| UUIDs | Must be valid UUID v4 format |
| Amounts (price, quantity) | Must be positive integers (no floats for money) |
| Dates | ISO 8601 only; reject ambiguous formats |
| Strings | Strip whitespace; enforce min/max length |
| Enums | Reject values not in defined enum set |
| Email | RFC 5322 validated; lowercase normalised |
| Phone numbers | E.164 format (`+919876543210`) |
| ISIN | 12-character alphanumeric |
| Ticker symbols | 1–63 uppercase alphanumeric with hyphens |
| Pagination cursor | Base64 encoded; validated before decode |
| Sort fields | Whitelist allowed; reject arbitrary column names (SQL injection) |

### 7.3 Input Sanitisation

- All text inputs HTML-entity encoded before storage.
- JSONB fields (`config`, `rules`, `condition_params`) must pass a JSON schema validation before persistence.
- File uploads (backtest reports, model weights) are size-limited and MIME-type validated before S3 upload.

---

## 8. Idempotency Strategy

### 8.1 Idempotency-Key Header

Any state-mutating request that can be safely retried MUST support the `Idempotency-Key` header.

```
Idempotency-Key: a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

**Required on**: `POST /orders`, `POST /payments`, `POST /auth/login`, `POST /alerts`, `POST /broker/orders`
**Optional on**: All other POST/PUT/PATCH endpoints
**Forbidden on**: GET, DELETE endpoints

### 8.2 Idempotency Behaviour

```
1. Request arrives with Idempotency-Key header
2. Server checks idempotency_keys table:
   a. No existing row → process normally, store response in idempotency_keys
   b. Existing row (same path + same key) → return cached response immediately
      with header: Idempotency-Replayed: true
   c. Existing row (same key, different path) → 400 INVALID_IDEMPOTENCY_KEY
3. Keys expire after 24 hours (configured in idempotency_keys.expires_at)
```

### 8.3 Key Format Requirements

- UUID v4 format only
- Client-generated (not server-generated)
- Must be unique per operation, not per user session
- Reuse of a key with a different request body is an error (not a replay)

---

## 9. Rate Limiting Strategy

### 9.1 Rate Limit Tiers

| Tier | Applies To | Limit | Window |
|---|---|---|---|
| Global IP | Unauthenticated requests | 60 req | 1 minute |
| Authenticated User | All authenticated requests | 1000 req | 1 minute |
| AI/Prediction endpoints | `/api/v1/ai/*` | 100 req | 1 minute |
| Broker order submission | `/api/v1/broker/orders` | 20 req | 1 minute |
| WebSocket connections | Per user | 5 concurrent connections | — |
| Export endpoints | `/api/v1/*/export` | 10 req | 1 hour |

### 9.2 Redis Sliding Window Counter

```
Key:   ratelimit:{user_id}:{endpoint_group}:{epoch_minute}
Type:  String (INCR)
TTL:   2 minutes (sliding window retention)

Algorithm:
  current_count = INCR ratelimit:{user_id}:{group}:{epoch_minute}
  IF current_count == 1: EXPIRE key 120
  IF current_count > limit: return 429
```

### 9.3 Rate Limit Response

```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1705298460
Retry-After: 47

{
  "success": false,
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded. Retry after 47 seconds."
  }
}
```

---

## 10. OpenAPI 3.1 Base Configuration

```yaml
openapi: "3.1.0"

info:
  title: "Market Intelligence Platform API"
  description: |
    AI-Powered Indian Stock Market Intelligence SaaS Platform.
    Private deployment v1.0 — forward-compatible with multi-tenant SaaS.
  version: "1.0.0"
  contact:
    name: "Platform Support"
    email: "support@yourdomain.com"
  license:
    name: "Proprietary"

servers:
  - url: "https://api.yourdomain.com/api/v1"
    description: "Production"
  - url: "https://staging-api.yourdomain.com/api/v1"
    description: "Staging"
  - url: "http://localhost:8000/api/v1"
    description: "Local Development"

security:
  - BearerAuth: []

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
    ApiKeyAuth:
      type: apiKey
      in: header
      name: X-API-Key

  parameters:
    PageSize:
      name: page_size
      in: query
      schema: { type: integer, default: 50, minimum: 1, maximum: 200 }
    Cursor:
      name: cursor
      in: query
      schema: { type: string }
    IdempotencyKey:
      name: Idempotency-Key
      in: header
      required: false
      schema: { type: string, format: uuid }
    RequestId:
      name: X-Request-ID
      in: header
      schema: { type: string }

  schemas:
    # --- Shared Primitives ---
    UUID:
      type: string
      format: uuid
      example: "550e8400-e29b-41d4-a716-446655440000"

    Timestamp:
      type: string
      format: date-time
      example: "2025-01-15T09:15:00+05:30"

    AmountCents:
      type: integer
      format: int64
      minimum: 0
      description: "Monetary amount in paise (1 INR = 100 paise). Never use floats for money."
      example: 150000

    CursorPagination:
      type: object
      properties:
        cursor: { type: string }
        next_cursor: { type: string, nullable: true }
        has_more: { type: boolean }
        total_count: { type: integer }
        page_size: { type: integer }

    Meta:
      type: object
      properties:
        request_id: { type: string }
        api_version: { type: string }

    SuccessEnvelope:
      type: object
      required: [success, data, meta]
      properties:
        success: { type: boolean, enum: [true] }
        data: {}
        meta: { $ref: '#/components/schemas/Meta' }

    ErrorDetail:
      type: object
      properties:
        field: { type: string }
        message: { type: string }
        rejected_value: {}

    ErrorEnvelope:
      type: object
      required: [success, error, meta]
      properties:
        success: { type: boolean, enum: [false] }
        error:
          type: object
          properties:
            code: { type: string }
            message: { type: string }
            details:
              type: array
              items: { $ref: '#/components/schemas/ErrorDetail' }
        meta: { $ref: '#/components/schemas/Meta' }
```

---

## 11. Identity & Auth APIs

### Base Path: `/api/v1/auth`

---

#### `POST /api/v1/auth/register`
Register a new user (and create their tenant for v1.0).

**Request Body**:
```json
{
  "email": "user@example.com",
  "password": "Str0ng!Password",
  "first_name": "Rajesh",
  "last_name": "Kumar"
}
```

**Response `201`**:
```json
{
  "success": true,
  "data": {
    "user_id": "uuid",
    "tenant_id": "uuid",
    "email": "user@example.com",
    "status": "ACTIVE"
  }
}
```

**Validations**: Password min 12 chars, 1 uppercase, 1 digit, 1 special char. Email normalized to lowercase.

---

#### `POST /api/v1/auth/login`
Authenticate and receive tokens.

**Required header**: `Idempotency-Key`
**Request Body**:
```json
{
  "email": "user@example.com",
  "password": "Str0ng!Password"
}
```

**Response `200`**:
```json
{
  "success": true,
  "data": {
    "access_token": "eyJhbGci...",
    "token_type": "Bearer",
    "expires_in": 900,
    "user": {
      "id": "uuid",
      "email": "user@example.com",
      "roles": ["ADMIN"],
      "mfa_required": false
    }
  }
}
```
**Note**: `refresh_token` is set as an `httpOnly; Secure; SameSite=Strict` cookie — never in the body.

---

#### `POST /api/v1/auth/refresh`
Rotate the refresh token and issue a new access token.

**Required**: Valid `refresh_token` httpOnly cookie
**Response `200`**: Same structure as login response

---

#### `POST /api/v1/auth/logout`
Revoke session and clear tokens.
**Response `200`**: `{ "success": true, "data": null }`

---

#### `POST /api/v1/auth/mfa/enable`
Initiate MFA setup. Returns TOTP QR code URI.
**Permission**: Authenticated (no special permission)

**Response `200`**:
```json
{
  "data": {
    "totp_uri": "otpauth://totp/MarketIntelligence:user@...",
    "backup_codes": ["ABCD-1234", "EFGH-5678"]
  }
}
```

---

#### `POST /api/v1/auth/mfa/verify`
Verify TOTP code to complete MFA and receive a fully-authorized token.

**Request Body**: `{ "totp_code": "123456" }`
**Response `200`**: New access_token with `mfa_verified: true` claim

---

#### `POST /api/v1/auth/password/reset-request`
Initiate password reset flow. Sends email with reset link.
**Request Body**: `{ "email": "user@example.com" }`
**Response**: Always `200` regardless of email existence (prevent enumeration).

---

#### `POST /api/v1/auth/password/reset`
Complete password reset with token from email.
**Request Body**: `{ "reset_token": "...", "new_password": "..." }`

---

## 12. User & Profile APIs

### Base Path: `/api/v1/users`

---

#### `GET /api/v1/users/me`
Get current authenticated user's full profile.
**Permission**: Authenticated
**Response `200`**:
```json
{
  "data": {
    "id": "uuid",
    "email": "user@example.com",
    "status": "ACTIVE",
    "mfa_enabled": false,
    "profile": {
      "first_name": "Rajesh",
      "last_name": "Kumar",
      "phone_number": "+919876543210",
      "avatar_url": "https://cdn.yourdomain.com/avatars/..."
    },
    "roles": ["ADMIN"],
    "tenant_id": "uuid",
    "created_at": "2025-01-15T09:15:00+05:30"
  }
}
```

---

#### `PATCH /api/v1/users/me`
Update current user's profile fields (partial update).
**Permission**: Authenticated
**Request Body** (all fields optional):
```json
{
  "first_name": "Rajesh",
  "last_name": "Kumar",
  "phone_number": "+919876543210"
}
```

---

#### `POST /api/v1/users/me/avatar`
Upload profile avatar. Multipart form data.
**Content-Type**: `multipart/form-data`
**Max size**: 5MB. Allowed types: `image/jpeg`, `image/png`, `image/webp`

---

#### `GET /api/v1/users/me/api-keys`
List all API keys for the current user.
**Permission**: Authenticated

---

#### `POST /api/v1/users/me/api-keys`
Create a new API key. The raw key is returned ONLY once at creation — not stored.
**Request Body**: `{ "name": "My Trading Bot" }`
**Response `201`**:
```json
{
  "data": {
    "id": "uuid",
    "name": "My Trading Bot",
    "key": "mip_live_Xy7Kj9...",
    "key_prefix": "mip_live_Xy7K",
    "created_at": "...",
    "expires_at": null
  }
}
```
> ⚠️ **Warning**: The `key` field is only returned once. Store it securely.

---

#### `DELETE /api/v1/users/me/api-keys/{key_id}`
Revoke an API key immediately.
**Permission**: Authenticated

---

#### `GET /api/v1/users/me/sessions`
List all active sessions for the current user.

---

#### `DELETE /api/v1/users/me/sessions/{session_id}`
Revoke a specific session (remote logout).

---

## 13. Subscription & Billing APIs

### Base Path: `/api/v1/billing`

---

#### `GET /api/v1/billing/plans`
List all available subscription plans.
**Auth**: Not required (public endpoint)
**Response `200`**:
```json
{
  "data": [
    {
      "id": "uuid",
      "name": "FREE",
      "price_cents": 0,
      "currency": "INR",
      "watchlist_limit": 1,
      "portfolio_limit": 1,
      "alerts_daily_limit": 10,
      "features": { "ai_signals": false, "backtesting": false }
    },
    {
      "id": "uuid",
      "name": "PRO",
      "price_cents": 99900,
      "currency": "INR",
      "watchlist_limit": 20,
      "portfolio_limit": 5,
      "alerts_daily_limit": 100,
      "features": { "ai_signals": true, "backtesting": true }
    }
  ]
}
```

---

#### `GET /api/v1/billing/subscription`
Get the current tenant's active subscription.
**Permission**: `billing:read`

---

#### `POST /api/v1/billing/subscription/upgrade`
Initiate a plan upgrade. Creates a Razorpay order.
**Permission**: `billing:manage`
**Required header**: `Idempotency-Key`
**Request Body**: `{ "plan_id": "uuid" }`
**Response `200`**:
```json
{
  "data": {
    "invoice_id": "uuid",
    "amount_cents": 99900,
    "currency": "INR",
    "razorpay_order_id": "order_NbhfxxxxXXXX",
    "razorpay_key_id": "rzp_live_..."
  }
}
```

---

#### `POST /api/v1/billing/payments/webhook`
Receive payment gateway webhook events (Razorpay/Stripe).
**Auth**: HMAC signature verification via `X-Razorpay-Signature` header (not JWT)
**Note**: This endpoint is idempotent by design — duplicate webhook delivery is handled via `external_transaction_id` uniqueness.

---

#### `GET /api/v1/billing/invoices`
List all invoices for the current tenant.
**Permission**: `billing:read`
**Query Params**: `status`, `cursor`, `page_size`

---

#### `GET /api/v1/billing/invoices/{invoice_id}`
Get a specific invoice with its line items.
**Permission**: `billing:read`

---

#### `GET /api/v1/billing/invoices/{invoice_id}/download`
Download invoice PDF. Redirects to a pre-signed S3 URL (expires in 5 minutes).
**Permission**: `billing:read`
**Response `302`**: `Location: https://s3.amazonaws.com/...?X-Amz-Expires=300`

---

## 14. Market Data APIs

### Base Path: `/api/v1/market`

---

#### `GET /api/v1/market/exchanges`
List all supported exchanges.
**Auth**: Authenticated
**Response `200`**:
```json
{
  "data": [
    { "code": "NSE", "name": "National Stock Exchange", "timezone": "Asia/Kolkata", "is_active": true },
    { "code": "BSE", "name": "Bombay Stock Exchange", "timezone": "Asia/Kolkata", "is_active": true }
  ]
}
```

---

#### `GET /api/v1/market/symbols`
Search and list market symbols.
**Query Params**: `q` (search text), `exchange_code`, `type` (EQUITY/FUTURE/OPTION/INDEX), `status`, `cursor`, `page_size`
**Response `200`**: Paginated list of symbols

---

#### `GET /api/v1/market/symbols/{symbol_id}`
Get detailed information for a specific symbol.

---

#### `GET /api/v1/market/symbols/{symbol_id}/quote`
Get current live quote from Redis cache.
**Permission**: `market:read`
**Response `200`**:
```json
{
  "data": {
    "symbol_id": "uuid",
    "ticker": "RELIANCE",
    "exchange": "NSE",
    "ltp_cents": 295000,
    "ltp": 2950.00,
    "change_cents": 2500,
    "change_pct": 0.86,
    "volume": 1234567,
    "open_interest": 0,
    "bid_cents": 294900,
    "ask_cents": 295100,
    "timestamp": "2025-01-15T14:32:01+05:30",
    "is_market_open": true
  }
}
```

---

#### `GET /api/v1/market/symbols/{symbol_id}/candles`
Get OHLCV candle data from TimescaleDB.
**Permission**: `market:read`
**Query Params**:
- `timeframe`: `1m`, `5m`, `1h`, `1d` (required)
- `from`: ISO 8601 timestamp (required)
- `to`: ISO 8601 timestamp (required)
- `limit`: max 2000 candles

**Response `200`**:
```json
{
  "data": {
    "symbol_id": "uuid",
    "ticker": "RELIANCE",
    "timeframe": "5m",
    "candles": [
      {
        "time": "2025-01-15T09:15:00+05:30",
        "open_cents": 292000,
        "high_cents": 295500,
        "low_cents": 291000,
        "close_cents": 295000,
        "volume": 450000
      }
    ]
  }
}
```

---

#### `GET /api/v1/market/holidays`
Get trading holidays for an exchange.
**Query Params**: `exchange_code` (required), `year`

---

#### `GET /api/v1/market/symbols/{symbol_id}/derivatives`
Get derivative contracts for a symbol (futures and options chain).
**Permission**: `market:read`
**Query Params**: `contract_type` (FUTURE/OPTION), `expiry_date`

---

#### `GET /api/v1/market/news`
Get recent news articles with sentiment scores.
**Permission**: `market:read`
**Query Params**: `symbol_id`, `from`, `to`, `cursor`, `page_size`

---

#### `GET /api/v1/market/economic-calendar`
Get upcoming economic events.
**Permission**: `market:read`
**Query Params**: `from`, `to`, `impact_level`, `country`

---

## 15. Portfolio APIs

### Base Path: `/api/v1/portfolios`

---

#### `GET /api/v1/portfolios`
List all portfolios for the current user.
**Permission**: `portfolio:read`

---

#### `POST /api/v1/portfolios`
Create a new portfolio.
**Permission**: `portfolio:create`
**Request Body**:
```json
{
  "name": "My Paper Portfolio",
  "type": "PAPER",
  "initial_balance_cents": 1000000000
}
```
**Response `201`**: Created portfolio object

---

#### `GET /api/v1/portfolios/{portfolio_id}`
Get a specific portfolio with summary.
**Permission**: `portfolio:read`
**Response `200`**:
```json
{
  "data": {
    "id": "uuid",
    "name": "My Paper Portfolio",
    "type": "PAPER",
    "balance_cents": 950000000,
    "version": 1,
    "holdings_count": 5,
    "total_investment_cents": 50000000,
    "current_value_cents": 52300000,
    "unrealised_pnl_cents": 2300000,
    "unrealised_pnl_pct": 4.6,
    "created_at": "2025-01-15T09:15:00+05:30"
  }
}
```

---

#### `PATCH /api/v1/portfolios/{portfolio_id}`
Update portfolio name.
**Permission**: `portfolio:manage`
**Request Body**: `{ "name": "New Name" }`

---

#### `DELETE /api/v1/portfolios/{portfolio_id}`
Soft-delete a portfolio (marks inactive, retains data).
**Permission**: `portfolio:manage`

---

#### `GET /api/v1/portfolios/{portfolio_id}/holdings`
List all holdings in a portfolio.
**Permission**: `portfolio:read`
**Response `200`**:
```json
{
  "data": [
    {
      "id": "uuid",
      "symbol": { "id": "uuid", "ticker": "RELIANCE", "exchange": "NSE" },
      "quantity": 10,
      "average_cost_cents": 290000,
      "current_price_cents": 295000,
      "investment_cents": 2900000,
      "current_value_cents": 2950000,
      "unrealised_pnl_cents": 50000,
      "unrealised_pnl_pct": 1.72,
      "version": 3
    }
  ]
}
```

---

#### `GET /api/v1/portfolios/{portfolio_id}/orders`
List all orders in a portfolio.
**Permission**: `portfolio:read`
**Query Params**: `status`, `direction`, `symbol_id`, `from`, `to`, `cursor`

---

#### `POST /api/v1/portfolios/{portfolio_id}/orders`
Place a paper trading order.
**Permission**: `portfolio:trade`
**Required header**: `Idempotency-Key`
**Request Body**:
```json
{
  "symbol_id": "uuid",
  "direction": "BUY",
  "type": "MARKET",
  "quantity": 10,
  "price_cents": null
}
```
**Response `201`**:
```json
{
  "data": {
    "id": "uuid",
    "status": "PENDING",
    "portfolio_id": "uuid",
    "symbol": { "ticker": "RELIANCE" },
    "direction": "BUY",
    "type": "MARKET",
    "quantity": 10,
    "estimated_value_cents": 2950000,
    "created_at": "..."
  }
}
```

---

#### `DELETE /api/v1/portfolios/{portfolio_id}/orders/{order_id}`
Cancel a pending order.
**Permission**: `portfolio:trade`

---

#### `GET /api/v1/portfolios/{portfolio_id}/ledger`
Get the double-entry ledger for a portfolio.
**Permission**: `portfolio:read`
**Query Params**: `type`, `from`, `to`, `cursor`, `page_size`

---

#### `GET /api/v1/portfolios/{portfolio_id}/pnl`
Get realised and unrealised P&L summary.
**Permission**: `portfolio:read`
**Query Params**: `from`, `to`
**Response `200`**:
```json
{
  "data": {
    "realised_pnl_cents": 15000,
    "unrealised_pnl_cents": 50000,
    "total_brokerage_paid_cents": 1200,
    "total_stt_paid_cents": 890,
    "total_gst_paid_cents": 216,
    "total_charges_cents": 2306,
    "net_pnl_cents": 62694
  }
}
```

---

## 16. Watchlist APIs

### Base Path: `/api/v1/watchlists`

---

#### `GET /api/v1/watchlists`
List all watchlists for the current user.
**Permission**: `watchlist:read`

---

#### `POST /api/v1/watchlists`
Create a new watchlist.
**Permission**: `watchlist:create`
**Request Body**: `{ "name": "Nifty 50 Picks" }`

---

#### `GET /api/v1/watchlists/{watchlist_id}`
Get a watchlist with its symbols and live quotes.
**Permission**: `watchlist:read`
**Response `200`**:
```json
{
  "data": {
    "id": "uuid",
    "name": "Nifty 50 Picks",
    "symbols": [
      {
        "symbol_id": "uuid",
        "ticker": "RELIANCE",
        "exchange": "NSE",
        "ltp_cents": 295000,
        "change_pct": 0.86,
        "added_at": "2025-01-10T..."
      }
    ]
  }
}
```

---

#### `PATCH /api/v1/watchlists/{watchlist_id}`
Rename a watchlist.
**Permission**: `watchlist:manage`

---

#### `DELETE /api/v1/watchlists/{watchlist_id}`
Delete a watchlist.
**Permission**: `watchlist:manage`

---

#### `POST /api/v1/watchlists/{watchlist_id}/symbols`
Add a symbol to a watchlist.
**Permission**: `watchlist:manage`
**Request Body**: `{ "symbol_id": "uuid" }`
**Response `201`**: Updated watchlist

---

#### `DELETE /api/v1/watchlists/{watchlist_id}/symbols/{symbol_id}`
Remove a symbol from a watchlist.
**Permission**: `watchlist:manage`

---

## 17. Scanner APIs

### Base Path: `/api/v1/scanners`

---

#### `GET /api/v1/scanners`
List all scanners for the current user.
**Permission**: `scanner:read`

---

#### `POST /api/v1/scanners`
Create a new stock scanner with filter rules.
**Permission**: `scanner:create`
**Request Body**:
```json
{
  "name": "Breakout Scanner",
  "rules": {
    "filters": [
      { "indicator": "RSI_14", "operator": "lt", "value": 30 },
      { "indicator": "VOLUME_RATIO", "operator": "gt", "value": 2.0 },
      { "indicator": "CLOSE", "operator": "gt", "field": "SMA_200" }
    ],
    "universe": { "exchange": "NSE", "type": "EQUITY", "min_market_cap_cr": 1000 },
    "timeframe": "1d"
  }
}
```
**Response `201`**: Created scanner object

---

#### `GET /api/v1/scanners/{scanner_id}`
Get scanner details with latest run results.
**Permission**: `scanner:read`

---

#### `PATCH /api/v1/scanners/{scanner_id}`
Update scanner rules or status.
**Permission**: `scanner:manage`

---

#### `DELETE /api/v1/scanners/{scanner_id}`
Delete a scanner.
**Permission**: `scanner:manage`

---

#### `POST /api/v1/scanners/{scanner_id}/run`
Trigger an immediate scanner run.
**Permission**: `scanner:execute`
**Response `202`**:
```json
{
  "data": {
    "run_id": "uuid",
    "status": "RUNNING",
    "message": "Scanner run queued. Results available via GET /scanners/{id}/runs/{run_id}"
  }
}
```

---

#### `GET /api/v1/scanners/{scanner_id}/runs`
List all historical runs for a scanner.
**Query Params**: `cursor`, `page_size`

---

#### `GET /api/v1/scanners/{scanner_id}/runs/{run_id}`
Get results of a specific scanner run.
**Permission**: `scanner:read`
**Response `200`**:
```json
{
  "data": {
    "run_id": "uuid",
    "status": "COMPLETED",
    "execution_time": "2025-01-15T09:15:00+05:30",
    "duration_ms": 1243,
    "total_matches": 8,
    "matches": [
      {
        "symbol": { "ticker": "INFY", "exchange": "NSE" },
        "metrics": { "RSI_14": 28.5, "VOLUME_RATIO": 2.3, "SMA_200": 1890.00 }
      }
    ]
  }
}
```

---

## 18. AI Signal APIs

### Base Path: `/api/v1/signals`

---

#### `GET /api/v1/signals`
List all AI-generated trade signals.
**Permission**: `signal:read`
**Query Params**: `symbol_id`, `direction` (BUY/SELL), `from`, `to`, `min_confidence`, `cursor`, `page_size`
**Response `200`**:
```json
{
  "data": [
    {
      "id": "uuid",
      "symbol": { "ticker": "RELIANCE", "exchange": "NSE" },
      "direction": "BUY",
      "version": 1,
      "entry_range": { "low_cents": 290000, "high_cents": 295000 },
      "stop_loss_cents": 280000,
      "target_price_cents": 320000,
      "confidence_calibrated": 72,
      "risk_reward_ratio": 3.0,
      "rationale": "Strong bullish momentum with RSI recovery from oversold...",
      "created_at": "2025-01-15T09:15:00+05:30",
      "performance": {
        "status": null,
        "is_resolved": false
      }
    }
  ]
}
```

---

#### `GET /api/v1/signals/{signal_id}`
Get a specific signal with full details.
**Permission**: `signal:read`

---

#### `GET /api/v1/signals/{signal_id}/explanation`
Get the SHAP explainability data for a signal.
**Permission**: `signal:read`
**Response `200`**:
```json
{
  "data": {
    "signal_id": "uuid",
    "model_version": "v2.1.3",
    "base_value": 0.5,
    "shap_values": [
      { "feature": "RSI_14", "value": 28.5, "shap_contribution": 0.18, "direction": "positive" },
      { "feature": "VOLUME_RATIO_5D", "value": 2.3, "shap_contribution": 0.12, "direction": "positive" },
      { "feature": "MACD_SIGNAL", "value": -0.45, "shap_contribution": -0.05, "direction": "negative" }
    ],
    "confidence_before_calibration": 68,
    "confidence_calibrated": 72
  }
}
```

---

#### `GET /api/v1/signals/{signal_id}/performance`
Get the post-resolution performance outcome for a signal.
**Permission**: `signal:read`
**Response `200`**:
```json
{
  "data": {
    "signal_id": "uuid",
    "status": "TARGET_HIT",
    "mfe_bps": 870,
    "mae_bps": -210,
    "realized_return_bps": 860,
    "analyzed_at": "2025-01-20T15:30:00+05:30"
  }
}
```

---

#### `POST /api/v1/signals/{signal_id}/feedback`
Submit user feedback on a signal's quality.
**Permission**: `signal:read`
**Request Body**:
```json
{
  "rating": 4,
  "user_comment": "Signal was accurate, entered near lower band",
  "followed_trade": true
}
```

---

## 19. AI Prediction & Explainability APIs

### Base Path: `/api/v1/ai`

---

#### `POST /api/v1/ai/predict`
Request an on-demand AI price direction prediction for a symbol.
**Permission**: `ai:predict`
**Rate Limit**: 100 requests/minute
**Required header**: `Idempotency-Key`
**Request Body**:
```json
{
  "symbol_id": "uuid",
  "timeframe": "1d",
  "model_version_id": null
}
```
**Response `200`**:
```json
{
  "data": {
    "prediction_id": "uuid",
    "symbol": { "ticker": "RELIANCE" },
    "direction": "BUY",
    "confidence_calibrated": 68,
    "entry_range": { "low_cents": 290000, "high_cents": 295000 },
    "stop_loss_cents": 280000,
    "target_price_cents": 315000,
    "model_version": "v2.1.3",
    "feature_snapshot_time": "2025-01-15T14:30:00+05:30",
    "explanation_available": true
  }
}
```

---

#### `GET /api/v1/ai/models`
List all AI model versions.
**Permission**: `ai:admin`
**Query Params**: `status`, `model_id`

---

#### `GET /api/v1/ai/models/{model_id}`
Get a specific model's metadata, metrics, and calibration info.
**Permission**: `ai:read`

---

#### `PATCH /api/v1/ai/models/{model_id}/versions/{version_id}/status`
Promote or demote a model version (FSM-governed).
**Permission**: `ai:admin`
**Request Body**: `{ "status": "STAGING" }`
**Note**: Application enforces the CANDIDATE → STAGING → PRODUCTION FSM. Invalid transitions return `422 BUSINESS_RULE_VIOLATION`.

---

#### `GET /api/v1/ai/models/{model_id}/versions/{version_id}/accuracy`
Get prediction accuracy history for a model version.
**Permission**: `ai:read`
**Query Params**: `from`, `to`

---

## 20. AI Chat APIs

### Base Path: `/api/v1/ai/chat`

---

#### `POST /api/v1/ai/chat`
Send a message to the AI market analyst assistant.
**Permission**: `ai:chat`
**Rate Limit**: 60 requests/minute
**Request Body**:
```json
{
  "message": "What is the technical outlook for RELIANCE today?",
  "context": {
    "symbol_id": "uuid",
    "portfolio_id": null
  }
}
```
**Response `200`**:
```json
{
  "data": {
    "conversation_id": "uuid",
    "message_id": "uuid",
    "response": "Based on current technical indicators...",
    "model_version": "v2.1.3",
    "prompt_tokens": 312,
    "completion_tokens": 487,
    "created_at": "2025-01-15T14:32:00+05:30"
  }
}
```

---

#### `GET /api/v1/ai/chat/history`
Get AI chat history for the current user.
**Permission**: `ai:chat`
**Query Params**: `from`, `to`, `cursor`, `page_size`

---

## 21. Alert & Notification APIs

### Base Path: `/api/v1/alerts`

---

#### `GET /api/v1/alerts/rules`
List all alert rules for the current user.
**Permission**: `alert:read`

---

#### `POST /api/v1/alerts/rules`
Create a new price or indicator alert rule.
**Permission**: `alert:create`
**Required header**: `Idempotency-Key`
**Request Body**:
```json
{
  "symbol_id": "uuid",
  "condition_type": "PRICE_CROSSES",
  "condition_params": {
    "direction": "above",
    "price_cents": 300000
  }
}
```

---

#### `PATCH /api/v1/alerts/rules/{rule_id}`
Update an alert rule or toggle active status.
**Permission**: `alert:manage`
**Request Body** (all optional): `{ "is_active": false, "condition_params": {...} }`

---

#### `DELETE /api/v1/alerts/rules/{rule_id}`
Delete an alert rule.
**Permission**: `alert:manage`

---

#### `GET /api/v1/alerts/events`
Get alert event history (fired alerts).
**Permission**: `alert:read`
**Query Params**: `rule_id`, `from`, `to`, `cursor`, `page_size`

---

#### `GET /api/v1/notifications/preferences`
Get notification channel preferences.
**Permission**: Authenticated
**Response `200`**:
```json
{
  "data": {
    "channels": {
      "email": true,
      "push": true,
      "telegram": false,
      "whatsapp": false
    },
    "telegram_chat_id": null,
    "whatsapp_phone_number": null
  }
}
```

---

#### `PUT /api/v1/notifications/preferences`
Update notification preferences.
**Permission**: Authenticated
**Request Body**:
```json
{
  "channels": { "email": true, "telegram": true },
  "telegram_chat_id": "123456789"
}
```

---

#### `GET /api/v1/notifications/logs`
Get notification delivery logs.
**Permission**: Authenticated
**Query Params**: `channel`, `status`, `from`, `to`, `cursor`

---

## 22. Broker Integration APIs

### Base Path: `/api/v1/broker`

---

#### `GET /api/v1/broker/integrations`
List all supported broker integrations.
**Auth**: Authenticated
**Response `200`**:
```json
{
  "data": [
    { "id": "uuid", "name": "ZERODHA", "is_active": true },
    { "id": "uuid", "name": "ANGEL_ONE", "is_active": true },
    { "id": "uuid", "name": "UPSTOX", "is_active": true }
  ]
}
```

---

#### `GET /api/v1/broker/accounts`
List all connected broker accounts for the current user.
**Permission**: `broker:read`

---

#### `POST /api/v1/broker/accounts/connect`
Initiate broker OAuth connection flow.
**Permission**: `broker:connect`
**Request Body**: `{ "integration_id": "uuid" }`
**Response `200`**:
```json
{
  "data": {
    "authorization_url": "https://kite.zerodha.com/connect/login?v=3&api_key=...",
    "state": "csrf_state_token"
  }
}
```

---

#### `POST /api/v1/broker/accounts/callback`
Handle OAuth callback after broker authorization.
**Request Body**: `{ "code": "auth_code", "state": "csrf_state" }`
**Response `201`**: Created broker_account object

---

#### `DELETE /api/v1/broker/accounts/{account_id}`
Disconnect a broker account (revoke tokens).
**Permission**: `broker:manage`

---

#### `GET /api/v1/broker/accounts/{account_id}/balance`
Get live account balance from the broker.
**Permission**: `broker:read`

---

#### `GET /api/v1/broker/accounts/{account_id}/positions`
Get current live positions from the broker.
**Permission**: `broker:read`

---

#### `POST /api/v1/broker/orders`
Place a live order through a connected broker.
**Permission**: `broker:trade`
**Required header**: `Idempotency-Key`
**Request Body**:
```json
{
  "broker_account_id": "uuid",
  "portfolio_id": "uuid",
  "symbol_id": "uuid",
  "direction": "BUY",
  "type": "LIMIT",
  "quantity": 5,
  "price_cents": 292000
}
```
**Response `202`**:
```json
{
  "data": {
    "order_id": "uuid",
    "broker_order_id": "uuid",
    "external_order_id": "240115000123456",
    "status": "SUBMITTED",
    "message": "Order submitted to broker successfully."
  }
}
```

---

#### `GET /api/v1/broker/orders/{order_id}/executions`
Get all fill/execution records for a broker order.
**Permission**: `broker:read`
**Response `200`**:
```json
{
  "data": [
    {
      "id": "uuid",
      "external_execution_id": "...",
      "quantity": 5,
      "execution_price_cents": 291800,
      "brokerage_fee_cents": 40,
      "stt_charges_cents": 8,
      "exchange_txn_fee_cents": 5,
      "gst_cents": 8,
      "sebi_turnover_fee_cents": 1,
      "stamp_duty_cents": 3,
      "executed_at": "2025-01-15T10:32:45+05:30"
    }
  ]
}
```

---

## 23. Backtest APIs

### Base Path: `/api/v1/backtests`

---

#### `GET /api/v1/backtests`
List all backtest runs for the current user.
**Permission**: `backtest:read`

---

#### `POST /api/v1/backtests`
Submit a new backtest job.
**Permission**: `backtest:execute`
**Request Body**:
```json
{
  "strategy_id": "uuid",
  "start_date": "2022-01-01",
  "end_date": "2024-12-31",
  "initial_capital_cents": 1000000000
}
```
**Response `202`**:
```json
{
  "data": {
    "backtest_id": "uuid",
    "status": "QUEUED",
    "estimated_duration_seconds": 45
  }
}
```

---

#### `GET /api/v1/backtests/{backtest_id}`
Get backtest status and results when complete.
**Permission**: `backtest:read`
**Response `200`**:
```json
{
  "data": {
    "id": "uuid",
    "status": "COMPLETED",
    "strategy": { "name": "RSI Reversal", "version": 2 },
    "start_date": "2022-01-01",
    "end_date": "2024-12-31",
    "initial_capital_cents": 1000000000,
    "results": {
      "final_value_cents": 1482000000,
      "total_return_pct": 48.2,
      "cagr_bps": 1820,
      "sharpe_ratio": 142,
      "sortino_ratio": 189,
      "max_drawdown_bps": -1540,
      "total_trades": 87,
      "win_rate_pct": 61.4
    },
    "report_download_url": "https://api.yourdomain.com/api/v1/backtests/{id}/report",
    "completed_at": "2025-01-15T09:18:30+05:30"
  }
}
```

---

#### `GET /api/v1/backtests/{backtest_id}/report`
Download the detailed backtest report PDF.
**Permission**: `backtest:read`
**Response `302`**: Pre-signed S3 URL redirect (expires 5 minutes)

---

#### `GET /api/v1/strategies`
List all strategies.
**Permission**: `strategy:read`

---

#### `POST /api/v1/strategies`
Create a new strategy definition.
**Permission**: `strategy:create`
**Request Body**:
```json
{
  "name": "RSI Reversal",
  "config": {
    "entry_rules": [{ "indicator": "RSI_14", "operator": "lt", "value": 30 }],
    "exit_rules": [{ "indicator": "RSI_14", "operator": "gt", "value": 70 }],
    "stop_loss_pct": 5.0,
    "position_size_pct": 10.0
  }
}
```

---

## 24. Webhook Architecture

### 24.1 Overview

Webhooks allow the platform to push events to user-registered URLs over HTTPS. The platform uses the outbox pattern — webhooks are first written to `outbox_events` and dispatched by a background sweeper.

### 24.2 Webhook Registration

#### `GET /api/v1/webhooks`
List registered webhook endpoints.
**Permission**: `webhook:read`

#### `POST /api/v1/webhooks`
Register a new webhook endpoint.
**Permission**: `webhook:manage`
**Request Body**:
```json
{
  "target_url": "https://yourtradingbot.com/webhook",
  "events": ["signal.created", "alert.triggered", "order.filled", "backtest.completed"],
  "secret_key": "your_secret_for_verification"
}
```
**Response `201`**: Created webhook subscription

#### `DELETE /api/v1/webhooks/{webhook_id}`
Unregister a webhook.
**Permission**: `webhook:manage`

### 24.3 Webhook Payload Structure

All webhook events follow a standard envelope:

```json
{
  "event_id": "evt_01HX4...",
  "event_type": "signal.created",
  "api_version": "1.0.0",
  "created_at": "2025-01-15T09:15:00+05:30",
  "tenant_id": "uuid",
  "data": {
    "signal_id": "uuid",
    "symbol": "RELIANCE",
    "direction": "BUY",
    "confidence_calibrated": 72
  }
}
```

### 24.4 Supported Webhook Events

| Event Type | Trigger |
|---|---|
| `signal.created` | New AI trade signal generated |
| `signal.resolved` | Signal outcome determined (TARGET_HIT/SL_HIT/EXPIRED) |
| `alert.triggered` | An alert rule condition was met |
| `order.filled` | A paper or live order was fully filled |
| `order.cancelled` | An order was cancelled |
| `backtest.completed` | A backtest job finished |
| `scanner.run_completed` | A scanner run finished |
| `payment.settled` | A payment was confirmed |
| `subscription.changed` | Subscription plan upgraded or downgraded |

### 24.5 Webhook Security

```
1. Platform signs each payload with HMAC-SHA256 using the subscription's secret_key
2. Signature sent as header: X-MIP-Signature: sha256=<hex_digest>
3. Receiver MUST verify: hmac.compare_digest(expected_sig, received_sig)
4. Receiver MUST check event_id for replay prevention (store seen event_ids)
5. Platform retries failed deliveries: 3 attempts with backoff (30s, 5min, 30min)
6. After 3 failures, webhook is marked FAILED and user is notified
```

---

## 25. WebSocket Contracts

### 25.1 Connection Handshake

```
Client connects to: wss://api.yourdomain.com/ws/v1/market?token=<access_token>

Server validates token → returns:
{
  "type": "connection_ack",
  "connection_id": "conn_01HX...",
  "server_time": "2025-01-15T09:15:00+05:30"
}
```

### 25.2 Message Frame Format

All WebSocket messages (both directions) use the same JSON frame structure:

```json
{
  "type": "<message_type>",
  "id": "<client_message_id>",
  "payload": {}
}
```

### 25.3 Client → Server Messages

#### Subscribe to Symbol Ticks
```json
{
  "type": "subscribe",
  "id": "msg_001",
  "payload": {
    "channel": "ticks",
    "symbols": ["NSE:RELIANCE", "NSE:INFY", "NSE:TCS"]
  }
}
```

#### Subscribe to Portfolio Updates
```json
{
  "type": "subscribe",
  "id": "msg_002",
  "payload": {
    "channel": "portfolio",
    "portfolio_id": "uuid"
  }
}
```

#### Subscribe to Alert Notifications
```json
{
  "type": "subscribe",
  "id": "msg_003",
  "payload": {
    "channel": "alerts"
  }
}
```

#### Subscribe to AI Signal Stream
```json
{
  "type": "subscribe",
  "id": "msg_004",
  "payload": {
    "channel": "signals",
    "filters": { "min_confidence": 60 }
  }
}
```

#### Unsubscribe
```json
{
  "type": "unsubscribe",
  "id": "msg_005",
  "payload": { "channel": "ticks", "symbols": ["NSE:RELIANCE"] }
}
```

#### Ping (heartbeat)
```json
{ "type": "ping", "id": "msg_006", "payload": {} }
```

### 25.4 Server → Client Events

#### Tick Update
```json
{
  "type": "tick",
  "payload": {
    "symbol": "NSE:RELIANCE",
    "ltp_cents": 295100,
    "change_pct": 0.89,
    "volume": 1240000,
    "bid_cents": 295000,
    "ask_cents": 295200,
    "timestamp": "2025-01-15T10:32:01.234+05:30"
  }
}
```

#### Portfolio Value Update
```json
{
  "type": "portfolio_update",
  "payload": {
    "portfolio_id": "uuid",
    "current_value_cents": 52350000,
    "unrealised_pnl_cents": 2350000,
    "unrealised_pnl_pct": 4.7,
    "timestamp": "2025-01-15T10:32:01+05:30"
  }
}
```

#### Alert Fired
```json
{
  "type": "alert_fired",
  "payload": {
    "alert_event_id": "uuid",
    "rule_id": "uuid",
    "symbol": "NSE:RELIANCE",
    "trigger_message": "RELIANCE crossed ₹3000 (current: ₹3001.50)",
    "timestamp": "2025-01-15T10:32:01+05:30"
  }
}
```

#### New Signal Published
```json
{
  "type": "signal_published",
  "payload": {
    "signal_id": "uuid",
    "symbol": "NSE:INFY",
    "direction": "BUY",
    "confidence_calibrated": 75,
    "entry_range": { "low_cents": 175000, "high_cents": 177000 },
    "timestamp": "2025-01-15T10:32:01+05:30"
  }
}
```

#### Order Status Update
```json
{
  "type": "order_update",
  "payload": {
    "order_id": "uuid",
    "status": "FILLED",
    "filled_at": "2025-01-15T10:32:05+05:30",
    "execution_price_cents": 295050
  }
}
```

#### Pong (heartbeat response)
```json
{ "type": "pong", "payload": { "server_time": "2025-01-15T10:32:01+05:30" } }
```

#### Subscription Acknowledgement
```json
{
  "type": "subscription_ack",
  "id": "msg_001",
  "payload": {
    "channel": "ticks",
    "subscribed_symbols": ["NSE:RELIANCE", "NSE:INFY", "NSE:TCS"],
    "status": "ACTIVE"
  }
}
```

### 25.5 WebSocket Connection Rules

| Rule | Detail |
|---|---|
| Max connections per user | 5 concurrent |
| Heartbeat interval | Client sends `ping` every 30 seconds |
| Server disconnect | After 60 seconds without `ping`, server closes connection |
| Reconnect strategy | Client implements exponential backoff: 1s, 2s, 4s, 8s, 16s |
| Token expiry | Server sends `{ "type": "token_expiry_warning" }` 60s before expiry. Client must reconnect with a new token. |
| Max subscriptions | 50 symbols per connection |

---

## 26. Background Job APIs

### Base Path: `/api/v1/jobs`

---

#### `GET /api/v1/jobs/{job_id}`
Get the status and result of an async background job.
**Permission**: Authenticated (job must belong to the requesting user)

**Response `200`**:
```json
{
  "data": {
    "job_id": "uuid",
    "type": "BACKTEST",
    "status": "COMPLETED",
    "progress_pct": 100,
    "result_ref": "/api/v1/backtests/uuid",
    "error_message": null,
    "created_at": "2025-01-15T09:15:00+05:30",
    "completed_at": "2025-01-15T09:16:12+05:30"
  }
}
```

**Job statuses**: `QUEUED`, `RUNNING`, `COMPLETED`, `FAILED`, `CANCELLED`

---

#### `GET /api/v1/jobs`
List recent background jobs for the current user.
**Permission**: Authenticated
**Query Params**: `type`, `status`, `cursor`, `page_size`

---

#### `DELETE /api/v1/jobs/{job_id}`
Cancel a queued or running job (if cancellable).
**Permission**: Authenticated

---

## 27. Health, Monitoring & Admin APIs

### Health Endpoints (no auth required)

---

#### `GET /health`
Liveness check — returns 200 if the process is alive.
**Response `200`**:
```json
{ "status": "alive", "timestamp": "2025-01-15T09:15:00+05:30" }
```

---

#### `GET /health/ready`
Readiness check — returns 200 only if all dependencies are healthy.
**Response `200`**:
```json
{
  "status": "ready",
  "checks": {
    "database": "healthy",
    "redis": "healthy",
    "kafka": "healthy",
    "timescaledb": "healthy"
  },
  "timestamp": "2025-01-15T09:15:00+05:30"
}
```
**Response `503`** if any dependency is down:
```json
{
  "status": "degraded",
  "checks": {
    "database": "healthy",
    "redis": "unhealthy",
    "kafka": "healthy",
    "timescaledb": "healthy"
  }
}
```

---

#### `GET /health/live`
Basic liveness probe (alias of `/health` — for Kubernetes integration).

---

### Admin APIs (requires `admin:manage` permission)

#### `GET /api/v1/admin/tenants`
List all tenants.

#### `GET /api/v1/admin/tenants/{tenant_id}`
Get a specific tenant's details.

#### `PATCH /api/v1/admin/tenants/{tenant_id}/status`
Activate or suspend a tenant.
**Request Body**: `{ "status": "SUSPENDED", "reason": "Payment overdue" }`

#### `GET /api/v1/admin/users`
List all users across all tenants.
**Query Params**: `tenant_id`, `status`, `cursor`

#### `GET /api/v1/admin/audit-logs`
Query audit logs across all tenants.
**Query Params**: `tenant_id`, `user_id`, `action`, `from`, `to`, `cursor`

#### `GET /api/v1/admin/api-usage`
Query API usage analytics.
**Query Params**: `tenant_id`, `endpoint_path`, `from`, `to`

#### `GET /api/v1/admin/outbox/pending`
View pending outbox events (useful for operational debugging).

#### `POST /api/v1/admin/outbox/{event_id}/retry`
Manually retry a failed outbox event.

---

## 28. Future Roadmap

The following API capabilities are intentionally deferred for v1.0 private deployment. Each is designed to be addable without breaking existing clients.

| # | Capability | Trigger Condition | Design Notes |
|---|---|---|---|
| 1 | **GraphQL Gateway** | When frontend requires flexible field selection | Add `/graphql` alongside REST. REST remains for machine-to-machine. |
| 2 | **Multi-tenant Subdomain Routing** | When public SaaS launches | Gateway resolves `tenant_id` from subdomain before JWT check |
| 3 | **OAuth2 / OpenID Connect** | When third-party app integrations required | Add `/oauth/authorize`, `/oauth/token` endpoints |
| 4 | **gRPC for Internal Services** | When microservice decomposition starts | Use Protobuf schemas derived from existing Avro schemas |
| 5 | **API Monetisation & Metering** | When tiered API access is sold | Integrate with billing via `api_usage_analytics` table (already designed) |
| 6 | **Streaming via SSE** | When WebSocket infra is expensive for simple dashboards | Add `GET /api/v1/stream/signals` (Server-Sent Events) as lightweight alternative |
| 7 | **Admin Multi-tenancy Isolation** | When multiple enterprise customers are onboarded | Full tenant-scoped admin APIs with per-tenant RBAC |
| 8 | **Broker Webhooks Inbound** | When live broker order updates must be pushed | `POST /api/v1/broker/webhook/{integration_name}` endpoint per broker |

---

*Phase 4 — API Specification Design v1.0*
*Status: Draft — Pending User Approval*
*Next Phase: Phase 5 — Backend Service Architecture*
