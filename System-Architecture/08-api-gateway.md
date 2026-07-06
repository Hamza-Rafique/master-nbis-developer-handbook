# Chapter 8 — API Gateway

> **Part II — System Architecture**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 8.1 What is an API Gateway?

An **API Gateway** is the single entry point for all external traffic into your system. Every request — from a citizen booking an appointment, a bank calling the verification API, or an enrollment officer uploading a packet — passes through the API Gateway before reaching any microservice.

Think of it as the **front door, security guard, and traffic controller** of your entire system — all in one.

```
┌──────────────────────────────────────────────────────────────┐
│                What API Gateway Does                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Without API Gateway:                                        │
│                                                              │
│  Bank ──────────────────────────► Auth Service :8080        │
│  Hospital ──────────────────────► Auth Service :8080        │
│  Enrollment officer ────────────► Registration Service :8081 │
│  Citizen portal ────────────────► Resident Service :8082    │
│                                                              │
│  → Each service exposed to internet                         │
│  → Each service manages its own auth                        │
│  → No central rate limiting                                 │
│  → No central logging                                       │
│                                                              │
│  With API Gateway:                                           │
│                                                              │
│  Bank ──────────────┐                                        │
│  Hospital ──────────┤                                        │
│  Enrollment officer ┼──► API GATEWAY ──► correct service    │
│  Citizen portal ────┘                                        │
│                                                              │
│  → ONE entry point                                          │
│  → Centralized auth, rate limiting, logging                 │
│  → No service exposed directly                              │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 8.2 AWS API Gateway — The Service

In the NBIS AWS architecture, the API Gateway is implemented using **Amazon API Gateway** — a fully managed service that handles:

```
┌──────────────────────────────────────────────────────────────┐
│              Amazon API Gateway Capabilities                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ✅ TLS termination      HTTPS enforced, cert via ACM       │
│  ✅ Authentication        JWT validation via Cognito         │
│  ✅ Authorization         Role check before routing          │
│  ✅ Request routing       URL + method → correct service     │
│  ✅ Rate limiting         Throttle per client / per route    │
│  ✅ Request validation    Check required fields before fwd   │
│  ✅ Response caching      Cache GET responses (reduce load)  │
│  ✅ Access logging        Every request logged to CloudWatch │
│  ✅ Usage plans           API keys per relying party         │
│  ✅ SDK generation        Auto-generate client SDKs          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 8.3 Request Lifecycle Through the Gateway

Every request travels through a defined sequence of checks before reaching a microservice. Understanding this sequence is essential — it determines where security is enforced and where failures occur.

```
Incoming Request (HTTPS)
         │
         ▼
┌────────────────────┐
│  1. TLS Check      │  HTTP? → 301 redirect to HTTPS
│                    │  HTTPS? → proceed
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│  2. WAF Check      │  Malicious pattern? → 403 Block
│                    │  SQL injection? → 403 Block
│                    │  IP blacklist? → 403 Block
│                    │  Passes? → proceed
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│  3. Rate Limit     │  Over limit? → 429 Too Many Requests
│                    │  Within limit? → proceed
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│  4. Authentication │  No token? → 401 Unauthorized
│                    │  Invalid token? → 401 Unauthorized
│                    │  Valid JWT? → extract claims, proceed
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│  5. Authorization  │  Wrong role? → 403 Forbidden
│                    │  Correct role? → proceed
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│  6. Request        │  Missing required field? → 400 Bad Req
│     Validation     │  Invalid format? → 400 Bad Request
│                    │  Valid? → proceed
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│  7. Routing        │  Match URL + method → target service
│                    │  No match? → 404 Not Found
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│  8. Integration    │  Forward to ECS / Lambda / SQS
│                    │  Await response (sync) or ack (async)
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│  9. Response +     │  Log request + response to CloudWatch
│     Logging        │  Return response to caller
└────────────────────┘
```

---

## 8.4 TLS — Encryption in Transit

**TLS (Transport Layer Security)** ensures that all data traveling between the client and the API Gateway is encrypted. No plaintext traffic is ever allowed.

```
┌──────────────────────────────────────────────────────────────┐
│                  TLS in NBIS API Gateway                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  External (client → API Gateway):                           │
│  ├── TLS 1.3 enforced (1.0, 1.1, 1.2 disabled)             │
│  ├── Certificate issued by AWS ACM (auto-renewed)           │
│  ├── HSTS header: force HTTPS on all future requests        │
│  └── HTTP → 301 permanent redirect to HTTPS                 │
│                                                              │
│  Internal (API Gateway → microservice):                     │
│  ├── Traffic stays inside VPC (private network)             │
│  ├── mTLS: service-to-service mutual authentication         │
│  └── ACM Private CA issues internal service certificates    │
│                                                              │
│  AWS implementation:                                         │
│  ├── API Gateway custom domain → ACM certificate            │
│  ├── CloudFront in front for edge TLS termination           │
│  └── VPC Link: API Gateway → private ALB (no internet hop)  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 8.5 Authentication — Who Are You?

Authentication at the API Gateway means: **before a request reaches any microservice, the caller's identity is verified**.

In NBIS, different callers use different authentication mechanisms:

```
┌──────────────────────────────────────────────────────────────┐
│            Authentication by Caller Type                     │
├─────────────────────┬────────────────────────────────────────┤
│ Caller              │ Auth Mechanism                        │
├─────────────────────┼────────────────────────────────────────┤
│ Citizen (portal)    │ Cognito User Pool JWT                 │
│                     │ (username + password + OTP)           │
├─────────────────────┼────────────────────────────────────────┤
│ Relying party       │ API Key + Cognito Client Credentials  │
│ (bank, hospital)    │ (OAuth2 client_credentials flow)      │
├─────────────────────┼────────────────────────────────────────┤
│ Enrollment officer  │ Cognito User Pool JWT                 │
│                     │ (username + password + MFA)           │
├─────────────────────┼────────────────────────────────────────┤
│ Microservice        │ IAM Role (no JWT needed)              │
│ (internal call)     │ AWS SigV4 request signing             │
├─────────────────────┼────────────────────────────────────────┤
│ Admin               │ Cognito User Pool JWT + MFA mandatory │
└─────────────────────┴────────────────────────────────────────┘
```

### JWT token flow

```
Relying party (bank) requests access:

1. Bank sends: POST /oauth2/token
   { client_id, client_secret, grant_type: client_credentials }
                    │
                    ▼
2. Cognito validates credentials
                    │
                    ▼
3. Cognito returns JWT:
   {
     "access_token": "eyJhbGci...",
     "token_type": "Bearer",
     "expires_in": 3600
   }
                    │
                    ▼
4. Bank calls: POST /v1/auth/biometric
   Header: Authorization: Bearer eyJhbGci...
                    │
                    ▼
5. API Gateway validates JWT with Cognito
   ├── Valid signature? ✅
   ├── Not expired? ✅
   └── Correct scope? ✅
                    │
                    ▼
6. Request forwarded to Auth Service
```

---

## 8.6 Authorization — What Are You Allowed to Do?

After authentication tells us **who** the caller is, authorization tells us **what** they are allowed to do.

```
┌──────────────────────────────────────────────────────────────┐
│              Authorization Model in API Gateway              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  JWT contains claims:                                        │
│  {                                                           │
│    "sub": "bank-partner-001",                                │
│    "role": "RELYING_PARTY",                                  │
│    "scope": "auth:biometric auth:otp ekyc:basic",           │
│    "partner_id": "BNK-001",                                  │
│    "exp": 1705312800                                         │
│  }                                                           │
│                                                              │
│  API Gateway Lambda Authorizer checks:                       │
│  ├── Does this role have access to this endpoint?           │
│  ├── Does the scope include this operation?                 │
│  └── Is this partner approved for this service?             │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Route-level authorization matrix

```
┌────────────────────────────────┬──────┬──────┬───────┬──────┐
│ Route                          │ RP   │ ENRL │ ADMIN │ RSDNT│
├────────────────────────────────┼──────┼──────┼───────┼──────┤
│ POST /v1/auth/biometric        │  ✅  │  ❌  │  ✅   │  ❌  │
│ POST /v1/auth/otp              │  ✅  │  ❌  │  ✅   │  ❌  │
│ POST /v1/ekyc                  │  ✅  │  ❌  │  ✅   │  ❌  │
│ POST /v1/enrollments           │  ❌  │  ✅  │  ✅   │  ❌  │
│ GET  /v1/enrollments/{id}      │  ❌  │  ✅  │  ✅   │  ❌  │
│ GET  /v1/resident/history      │  ❌  │  ❌  │  ✅   │  ✅  │
│ PUT  /v1/resident/address      │  ❌  │  ❌  │  ✅   │  ✅  │
│ POST /v1/admin/revoke          │  ❌  │  ❌  │  ✅   │  ❌  │
│ GET  /v1/admin/audit-logs      │  ❌  │  ❌  │  ✅   │  ❌  │
└────────────────────────────────┴──────┴──────┴───────┴──────┘

RP    = Relying Party
ENRL  = Enrollment Officer
ADMIN = System Administrator
RSDNT = Resident (citizen)
```

---

## 8.7 Request Routing

Routing maps an incoming URL + HTTP method to the correct microservice backend.

```
┌──────────────────────────────────────────────────────────────┐
│                NBIS API Gateway Routing Table                │
├────────────────────────────────┬─────────────────────────────┤
│ Route                          │ Backend                     │
├────────────────────────────────┼─────────────────────────────┤
│ POST /v1/appointments          │ Pre-Registration Service    │
│ GET  /v1/appointments/{id}     │ Pre-Registration Service    │
│ POST /v1/enrollments           │ Registration Service        │
│ GET  /v1/enrollments/{id}      │ Registration Service        │
│ POST /v1/auth/biometric        │ Authentication Service      │
│ POST /v1/auth/otp/send         │ Authentication Service      │
│ POST /v1/auth/otp/verify       │ Authentication Service      │
│ POST /v1/ekyc                  │ eKYC Service                │
│ GET  /v1/resident/profile      │ Resident Services           │
│ PUT  /v1/resident/address      │ Resident Services           │
│ GET  /v1/resident/history      │ Resident Services           │
│ POST /v1/admin/revoke          │ Admin Service               │
│ GET  /v1/admin/audit-logs      │ Admin Service               │
│ POST /v1/credentials/{uin}     │ Credential Service          │
└────────────────────────────────┴─────────────────────────────┘
```

### Routing to different backend types

```
Route type → Backend
─────────────────────────────────────────────────────────
Sync (needs response)    → ECS service via VPC Link + ALB
Async (fire and forget)  → SQS queue (enrollment packets)
Simple function          → Lambda (stateless processing)
```

---

## 8.8 Rate Limiting and Throttling

**Rate limiting** prevents any single caller from overwhelming the system — whether through malicious attack (DDoS) or accidental runaway code.

```
┌──────────────────────────────────────────────────────────────┐
│                Rate Limiting Strategy in NBIS                │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Level 1 — Account level (entire API):                      │
│  10,000 requests/second (hard AWS limit, adjustable)        │
│                                                              │
│  Level 2 — Stage level (e.g. production):                   │
│  5,000 requests/second burst                                │
│  2,000 requests/second steady rate                          │
│                                                              │
│  Level 3 — Usage plan per relying party:                    │
│  Bank A:     500 req/sec    10,000 req/day                  │
│  Hospital B: 100 req/sec    2,000 req/day                   │
│  Startup C:  10 req/sec     500 req/day                     │
│                                                              │
│  Level 4 — Route level:                                     │
│  POST /v1/ekyc → 50 req/sec (expensive, consent required)  │
│  POST /v1/auth/biometric → 500 req/sec (lightweight check) │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Rate limit response

```
When limit exceeded:
HTTP 429 Too Many Requests
{
  "errorCode": "RATE_LIMIT_EXCEEDED",
  "message": "Request limit exceeded. Retry after 1 second.",
  "retryAfter": 1,
  "limit": 500,
  "remaining": 0,
  "resetAt": "2025-01-15T10:30:01Z"
}
```

### Token bucket algorithm

AWS API Gateway uses the **token bucket algorithm** for throttling:

```
Token Bucket (capacity = burst limit):
──────────────────────────────────────
│ Tokens fill at: steady rate/second │
│ Each request consumes: 1 token     │
│ If bucket empty: 429 returned      │
──────────────────────────────────────

Example:
Steady rate:  1,000 req/sec
Burst:        5,000 tokens max

→ Can handle sudden burst of 5,000 requests
→ Then settles to 1,000 req/sec sustained
→ Burst capacity refills at 1,000/sec
```

---

## 8.9 WAF — Web Application Firewall

The **WAF (Web Application Firewall)** sits in front of the API Gateway and blocks malicious requests before they even reach the gateway logic.

```
Internet ──► CloudFront ──► WAF ──► API Gateway ──► Services
```

### WAF rule groups for NBIS

```
┌──────────────────────────────────────────────────────────────┐
│                    WAF Rule Groups                           │
├─────────────────────────┬────────────────────────────────────┤
│ Rule Group              │ What it blocks                    │
├─────────────────────────┼────────────────────────────────────┤
│ AWS Core Rule Set       │ OWASP Top 10 (SQL injection,      │
│ (AWSManagedRulesCommon) │ XSS, path traversal, etc.)        │
├─────────────────────────┼────────────────────────────────────┤
│ Known Bad Inputs        │ Log4Shell, Spring4Shell,          │
│ (AWSManagedRulesKBI)    │ known exploit patterns            │
├─────────────────────────┼────────────────────────────────────┤
│ Anonymous IP List       │ Tor exit nodes, VPN providers,   │
│                         │ anonymous proxies                  │
├─────────────────────────┼────────────────────────────────────┤
│ IP Reputation List      │ Known malicious IPs (AWS threat   │
│                         │ intelligence)                      │
├─────────────────────────┼────────────────────────────────────┤
│ Rate-based rules        │ IP sending > 1,000 req/5min       │
│ (custom)                │ → auto-block for 10 minutes       │
├─────────────────────────┼────────────────────────────────────┤
│ Geo-restriction         │ Block all countries except        │
│ (custom for NBIS)       │ deploying country + embassies     │
├─────────────────────────┼────────────────────────────────────┤
│ Biometric endpoint      │ Extra scrutiny on /auth/biometric │
│ protection (custom)     │ payload size limits, field rules  │
└─────────────────────────┴────────────────────────────────────┘
```

### WAF action types

```
ALLOW  → Request passes through
BLOCK  → HTTP 403 Forbidden returned immediately
COUNT  → Request counted but allowed (for testing rules)
CAPTCHA → Human verification challenge (for suspicious patterns)
```

---

## 8.10 API Keys and Usage Plans

**API keys** identify relying parties (banks, hospitals, telecoms) calling the NBIS. They are combined with **usage plans** to enforce per-partner rate limits and quotas.

```
┌──────────────────────────────────────────────────────────────┐
│               API Key + Usage Plan Flow                      │
└──────────────────────────────────────────────────────────────┘

1. Partner onboarding:
   Partner Management Service creates:
   ├── API Key: x-api-key: aBcD1234EfGh5678
   ├── Usage Plan: 500 req/sec, 100,000 req/day
   └── Associated with: /v1/auth/* and /v1/ekyc/*

2. Partner makes API call:
   POST /v1/auth/biometric
   Headers:
     Authorization: Bearer eyJhbGci...  (JWT)
     x-api-key: aBcD1234EfGh5678        (API key)

3. API Gateway checks:
   ├── JWT valid? ✅
   ├── API key valid? ✅
   ├── API key matches JWT partner_id? ✅
   ├── Under rate limit? ✅
   └── Under daily quota? ✅
   → Forward to Auth Service

4. If quota exceeded:
   HTTP 429
   { "errorCode": "QUOTA_EXCEEDED",
     "message": "Daily quota of 100,000 requests reached",
     "quotaResetAt": "2025-01-16T00:00:00Z" }
```

---

## 8.11 Request Validation

API Gateway can validate requests **before** they reach your microservice — rejecting malformed requests at the gateway level saves compute costs and protects services.

```
┌──────────────────────────────────────────────────────────────┐
│              Request Validation Rules                        │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Validate request body:                                      │
│  POST /v1/auth/biometric MUST have:                         │
│  {                                                           │
│    "uin": "string, required, length: 12",                   │
│    "biometricData": "string, required, base64",             │
│    "modality": "enum: [FINGER, IRIS, FACE]",                │
│    "partnerId": "string, required"                          │
│  }                                                           │
│                                                              │
│  Missing uin? → 400 Bad Request (before service is called)  │
│  Invalid modality? → 400 Bad Request                        │
│                                                              │
│  Validate path parameters:                                   │
│  GET /v1/enrollments/{id}                                    │
│  id must match: [A-Z0-9]{10}                                │
│  Non-matching → 400 Bad Request                             │
│                                                              │
│  Validate headers:                                           │
│  Content-Type: application/json REQUIRED on POST            │
│  Missing → 415 Unsupported Media Type                       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 8.12 Access Logging and Audit

Every request through the API Gateway is logged. This is both an operational requirement (debugging) and a legal requirement (audit trail).

```
┌──────────────────────────────────────────────────────────────┐
│                 Access Log Format (NBIS)                     │
└──────────────────────────────────────────────────────────────┘

{
  "requestId":     "abc-123-xyz-456",
  "traceId":       "1-65a12345-abcdef123456",
  "timestamp":     "2025-01-15T10:30:00.123Z",
  "httpMethod":    "POST",
  "path":          "/v1/auth/biometric",
  "sourceIp":      "203.0.113.42",
  "userAgent":     "BankApp/2.1.0",
  "partnerId":     "BNK-001",
  "role":          "RELYING_PARTY",
  "statusCode":    200,
  "responseTime":  143,
  "requestSize":   1240,
  "responseSize":  256,
  "errorMessage":  null
}

Stored in: CloudWatch Logs
Retained:  7 years (legal compliance)
Encrypted: Yes (KMS)
Immutable: Yes (Object Lock)
```

### Correlation ID (Trace ID)

Every request gets a **unique trace ID** at the API Gateway. This ID flows through every microservice that handles the request.

```
Request arrives → API Gateway assigns traceId: abc-123
      │
      ├──► Auth Service logs: traceId: abc-123
      │
      ├──► Biometric matcher logs: traceId: abc-123
      │
      └──► Audit log entry: traceId: abc-123

When something goes wrong → search CloudWatch for traceId: abc-123
→ See the complete journey across all services
```

---

## 8.13 Caching

The API Gateway can cache responses to reduce load on backend services.

```
┌──────────────────────────────────────────────────────────────┐
│                  Caching Strategy in NBIS                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ✅ CACHE these endpoints:                                   │
│  ├── GET /v1/partner/{id}/config (rarely changes)           │
│  ├── GET /v1/error-codes (static reference data)            │
│  └── GET /v1/system/health (30-second cache)                │
│                                                              │
│  ❌ NEVER cache these endpoints:                             │
│  ├── POST /v1/auth/biometric (every call is unique)         │
│  ├── POST /v1/ekyc (consent required per call)              │
│  ├── GET /v1/resident/history (personal, real-time)         │
│  └── Any endpoint that returns personal identity data       │
│                                                              │
│  Cache key:                                                  │
│  ├── URL path                                               │
│  ├── Query string parameters                                │
│  └── Authorization header (per-user caching)               │
│                                                              │
│  AWS: API Gateway stage-level cache (0.5GB to 237GB)        │
│  For larger caches: ElastiCache in front of services        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 8.14 CloudFront — Edge Layer

**CloudFront** sits in front of the API Gateway, providing global edge presence, DDoS protection, and TLS offloading at AWS edge locations worldwide.

```
Citizen in Manila                    API Gateway in
(Philippines) ──► CloudFront Edge ──► Singapore (AWS region)
                  (Singapore PoP)

Citizen in Casablanca                API Gateway in
(Morocco) ──────► CloudFront Edge ──► Paris (AWS region)
                  (Paris PoP)

Benefits:
├── TLS terminated at nearest edge (lower latency)
├── DDoS absorbed at edge (Shield Advanced)
├── Geo-blocking enforced at edge
└── Static responses cached globally
```

---

## 8.15 Complete NBIS API Gateway Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│   INTERNET                                                   │
│                                                              │
│   Citizens, Banks, Hospitals, Enrollment Stations           │
│                                                              │
└──────────────────────┬───────────────────────────────────────┘
                       │ HTTPS
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  CLOUDFRONT (Edge Layer)                                     │
│  ├── DDoS protection (Shield Advanced)                      │
│  ├── TLS 1.3 termination at edge                            │
│  ├── Geo-restriction (country whitelist)                    │
│  └── WAF rule evaluation                                    │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  AWS WAF                                                     │
│  ├── OWASP Core Rule Set                                    │
│  ├── Known Bad Inputs                                       │
│  ├── IP Reputation + Anonymous IP                           │
│  └── Custom NBIS rules (rate-based, geo, payload)           │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  AWS API GATEWAY                                             │
│  ├── Rate limiting (token bucket per usage plan)            │
│  ├── JWT authentication (Cognito authorizer)                │
│  ├── Lambda authorizer (role + scope check)                 │
│  ├── Request validation (schema check)                      │
│  ├── Request routing (URL → backend)                        │
│  └── Access logging (CloudWatch)                            │
└───────────┬──────────────────────────────────────┬───────────┘
            │ VPC Link                             │ Direct
            ▼                                     ▼
┌───────────────────────┐             ┌────────────────────────┐
│  Internal ALB         │             │  Lambda Functions      │
│  (private subnet)     │             │  (async processing)    │
└───────────┬───────────┘             └────────────────────────┘
            │
            ▼
┌───────────────────────────────────────────────────────────────┐
│  MICROSERVICES (ECS Fargate — private subnet, no public IP)   │
│                                                               │
│  Auth Service  │  eKYC Service  │  Registration  │  Resident  │
└───────────────────────────────────────────────────────────────┘
```

---

## 8.16 API Gateway vs Application Load Balancer

A common confusion: when do you use API Gateway vs ALB?

```
┌──────────────────────────────────────────────────────────────┐
│           API Gateway vs ALB — When to Use Which            │
├──────────────────────┬───────────────────────────────────────┤
│ API Gateway          │ ALB                                  │
├──────────────────────┼───────────────────────────────────────┤
│ External-facing APIs │ Internal service-to-service          │
│ (internet traffic)   │ routing (inside VPC)                 │
│                      │                                      │
│ Need: auth, rate     │ Need: simple HTTP routing,           │
│ limiting, API keys,  │ health checks, path-based            │
│ WAF, usage plans     │ routing, WebSocket                   │
│                      │                                      │
│ Per-request pricing  │ Per-hour + LCU pricing               │
│                      │                                      │
│ NBIS: external entry │ NBIS: internal routing from          │
│ point for all APIs   │ API Gateway to ECS services          │
└──────────────────────┴───────────────────────────────────────┘
```

**In NBIS:** API Gateway handles the external world. Internal ALB routes traffic from the API Gateway VPC Link to the correct ECS service.

---

## 8.17 Key Terms

| Term | Definition |
|------|-----------|
| **API Gateway** | Single entry point for all external traffic — auth, routing, rate limiting |
| **TLS** | Transport Layer Security — encrypts data in transit |
| **JWT** | JSON Web Token — signed token containing caller identity and claims |
| **Cognito** | AWS identity service — issues and validates JWTs |
| **Lambda Authorizer** | Custom auth logic in Lambda — checks role and scope per route |
| **Rate limiting** | Controlling how many requests a caller can make per second/day |
| **Token bucket** | Algorithm for rate limiting — tokens fill at steady rate, burst allowed |
| **WAF** | Web Application Firewall — blocks malicious requests before gateway |
| **Usage plan** | Per-partner API quota — rate limit + daily request cap |
| **API key** | Identifier for a relying party — combined with JWT for full auth |
| **Request validation** | Schema check at gateway level — rejects malformed requests early |
| **Trace ID** | Unique request identifier that flows through all services for debugging |
| **CloudFront** | AWS CDN — edge layer in front of API Gateway for DDoS + TLS at edge |
| **VPC Link** | Secure connection from API Gateway to private ALB inside VPC |
| **mTLS** | Mutual TLS — both parties authenticate — used for service-to-service calls |
| **HSTS** | HTTP Strict Transport Security — forces HTTPS on all future requests |
| **Object Lock** | S3 / CloudWatch feature — makes logs immutable (WORM) |

---

## 8.18 Key Takeaways

- **API Gateway is the security boundary** — every single piece of security enforcement (TLS, auth, rate limiting, WAF) happens here before any microservice sees the request.
- **Authentication ≠ Authorization** — the gateway does both, in sequence. First confirm who the caller is (JWT), then confirm what they are allowed to do (role + scope check).
- **Rate limiting has multiple levels** — account, stage, usage plan per partner, and route level. NBIS uses all four for fine-grained control.
- **WAF is mandatory for a government system** — OWASP rules, IP reputation, geo-restriction, and rate-based blocking are the minimum baseline.
- **API keys + JWT together** — API key identifies the partner organization; JWT identifies the specific user or application. Both required for relying party access.
- **Trace IDs are essential for production debugging** — when an authentication fails at 2am, the trace ID is the thread that connects logs across API Gateway, Auth Service, Biometric matcher, and the audit log.
- **Never cache identity responses** — only cache truly static data. Authentication results, eKYC responses, and auth history must never be cached.
- **VPC Link keeps microservices private** — API Gateway reaches into the VPC through a VPC Link. Microservices have no public IPs and no direct internet exposure.

---

## 8.19 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 9 | Authentication — deep dive into OTP, biometric, and multi-factor auth flows |
| Chapter 10 | Authorization (RBAC) — role design, policy enforcement, least privilege |
| Chapter 11 | Event-Driven Architecture — async patterns, SQS, idempotency |

---

*Chapter 8 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*
