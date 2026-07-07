# Chapter 9 — Authentication

> **Part II — System Architecture**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 9.1 What is Authentication?

**Authentication** is the process of verifying that a person or system is who they claim to be.

It answers one question: **"Are you really who you say you are?"**

Authentication is not about what you can do — that is authorization (Chapter 10). Authentication is purely about confirming identity before anything else happens.

```
┌──────────────────────────────────────────────────────────────┐
│              Authentication in Plain English                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Person walks up and says: "I am Hamza, UIN: 123456789012" │
│                                                              │
│  Authentication asks:                                        │
│  "Prove it."                                                 │
│                                                              │
│  Person proves it by:                                        │
│  ├── Something they KNOW   → PIN, password, security answer │
│  ├── Something they HAVE   → OTP on phone, smart card       │
│  └── Something they ARE    → fingerprint, iris, face        │
│                                                              │
│  System checks proof → MATCH or NO MATCH                    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 9.2 Authentication Factors

Authentication is categorized by the **type of proof** used:

```
┌──────────────────────────────────────────────────────────────┐
│               The Three Authentication Factors               │
├─────────────────────┬────────────────────────────────────────┤
│ Factor              │ Examples in NBIS                      │
├─────────────────────┼────────────────────────────────────────┤
│ Knowledge           │ PIN, password, secret question        │
│ (something you      │                                       │
│  KNOW)              │ Weakest factor — can be stolen,       │
│                     │ guessed, phished                      │
├─────────────────────┼────────────────────────────────────────┤
│ Possession          │ OTP sent to registered mobile,        │
│ (something you      │ smart card, hardware token            │
│  HAVE)              │                                       │
│                     │ Stronger — requires physical device   │
├─────────────────────┼────────────────────────────────────────┤
│ Inherence           │ Fingerprint, iris scan, face scan     │
│ (something you      │                                       │
│  ARE)               │ Strongest — cannot be shared,         │
│                     │ physically bound to the person        │
└─────────────────────┴────────────────────────────────────────┘
```

### Multi-Factor Authentication (MFA)

Using **two or more factors together** dramatically increases security:

```
Single factor:  Password only         → Weak  (IAL1 / AAL1)
Two factors:    Password + OTP        → Strong (IAL2 / AAL2)
Two factors:    Biometric + PIN       → Strong (IAL2 / AAL2)
Two factors:    Biometric + smart card→ Strongest (IAL3 / AAL3)
```

---

## 9.3 Authentication Modes in NBIS

NBIS supports multiple authentication modes because different services have different trust requirements.

```
┌──────────────────────────────────────────────────────────────┐
│              NBIS Authentication Modes                       │
├─────────────────────┬────────────────────────────────────────┤
│ Mode                │ How it works                          │
├─────────────────────┼────────────────────────────────────────┤
│ Biometric Auth      │ UIN + fingerprint / iris / face       │
│ (1:1 match)         │ Strongest. Used at borders, banks     │
├─────────────────────┼────────────────────────────────────────┤
│ OTP Auth            │ UIN + one-time password (SMS/email)   │
│                     │ Medium strength. Used for portals,    │
│                     │ low-risk services                     │
├─────────────────────┼────────────────────────────────────────┤
│ Demographic Auth    │ UIN + name + DOB + address match      │
│                     │ Weakest. Limited use cases only       │
├─────────────────────┼────────────────────────────────────────┤
│ Multi-factor        │ Biometric + OTP                       │
│                     │ OTP + demographic                     │
│                     │ Highest assurance                     │
├─────────────────────┼────────────────────────────────────────┤
│ OIDC / OAuth2       │ "Sign in with National ID"            │
│ (eSignet)           │ For web and mobile applications       │
│                     │ Issues JWT after biometric / OTP auth │
└─────────────────────┴────────────────────────────────────────┘
```

---

## 9.4 Biometric Authentication — Deep Dive

Biometric authentication is the **core and most powerful** authentication mode in an NBIS. It is a **1:1 verification** — not a 1:N search.

```
┌──────────────────────────────────────────────────────────────┐
│             Biometric Authentication Flow                    │
└──────────────────────────────────────────────────────────────┘

Step 1: Citizen presents their UIN (claim)
        UIN: 123456789012

Step 2: Citizen presents biometric (proof)
        Fingerprint captured at device

Step 3: Auth Service receives request
        POST /v1/auth/biometric
        {
          "uin": "123456789012",
          "biometricData": "base64encodedFingerprint...",
          "modality": "FINGER",
          "partnerId": "BNK-001",
          "transactionId": "TXN-20250115-001"
        }

Step 4: API Gateway validates
        ├── JWT valid? ✅
        ├── API key valid? ✅
        └── Rate limit OK? ✅

Step 5: Auth Service fetches enrolled template
        DynamoDB GetItem (UIN: 123456789012)
        → retrieves biometric template reference
        S3 GetObject (template file)
        → retrieves encrypted biometric template
        KMS Decrypt (template)
        → decrypted template ready

Step 6: 1:1 biometric match
        match(probe_template, enrolled_template)
        → match_score: 98.7

Step 7: Threshold check
        match_score (98.7) ≥ threshold (85.0) → MATCH ✅

Step 8: Response returned
        {
          "status": "MATCH",
          "score": 98.7,
          "authToken": "eyJhbGci...",
          "transactionId": "TXN-20250115-001",
          "timestamp": "2025-01-15T10:30:00Z"
        }

Step 9: Audit log written
        {
          "event": "BIOMETRIC_AUTH_SUCCESS",
          "uin": "123456789012",         ← hashed in log
          "partnerId": "BNK-001",
          "modality": "FINGER",
          "score": 98.7,
          "timestamp": "2025-01-15T10:30:00Z",
          "traceId": "abc-123"
        }
```

### What happens on no-match

```
match_score (42.3) < threshold (85.0) → NO MATCH

Response:
{
  "status": "NO_MATCH",
  "authToken": null,
  "errorCode": "AUTH_001",
  "message": "Biometric verification failed",
  "attemptsRemaining": 2,
  "transactionId": "TXN-20250115-001"
}

After 3 failed attempts:
{
  "status": "LOCKED",
  "errorCode": "AUTH_002",
  "message": "Account temporarily locked. Try again in 30 minutes.",
  "lockExpiresAt": "2025-01-15T11:00:00Z"
}
```

---

## 9.5 OTP Authentication — Deep Dive

**OTP (One-Time Password)** authentication is used for lower-risk services where biometric capture hardware is not available — citizen portals, mobile apps, low-value transactions.

```
┌──────────────────────────────────────────────────────────────┐
│                  OTP Authentication Flow                     │
└──────────────────────────────────────────────────────────────┘

Step 1: Citizen enters UIN on portal
        UIN: 123456789012

Step 2: Portal calls OTP send endpoint
        POST /v1/auth/otp/send
        { "uin": "123456789012", "channel": "SMS" }

Step 3: Auth Service validates UIN exists
        DynamoDB GetItem → record found ✅

Step 4: OTP generated
        ├── Random 6-digit number: 847293
        ├── TTL: 5 minutes
        └── Max attempts: 3

Step 5: OTP stored in cache
        ElastiCache Redis:
        Key:   "otp:123456789012"
        Value: { "otp": "847293", "attempts": 0 }
        TTL:   300 seconds (5 minutes)

Step 6: OTP delivered to citizen
        SMS via AWS SNS → citizen's registered mobile
        "Your NBIS verification code: 847293
         Valid for 5 minutes. Do not share."

Step 7: Citizen enters OTP on portal
        POST /v1/auth/otp/verify
        {
          "uin": "123456789012",
          "otp": "847293",
          "transactionId": "TXN-20250115-002"
        }

Step 8: Auth Service validates OTP
        Redis GET "otp:123456789012"
        ├── OTP matches? ✅
        ├── Not expired? ✅ (TTL not hit)
        └── Attempts < 3? ✅

Step 9: OTP consumed (deleted from cache)
        Redis DEL "otp:123456789012"
        → Cannot be used again

Step 10: Response returned
         {
           "status": "MATCH",
           "authToken": "eyJhbGci...",
           "transactionId": "TXN-20250115-002"
         }
```

### OTP security rules

```
┌──────────────────────────────────────────────────────────────┐
│                    OTP Security Rules                        │
├──────────────────────────────────────────────────────────────┤
│ Rule                          │ Why                         │
├──────────────────────────────┼─────────────────────────────┤
│ 6-digit minimum               │ 10^6 = 1M combinations      │
│ Expires in 5 minutes          │ Limits phishing window      │
│ Max 3 attempts                │ Prevents brute force        │
│ Single use (delete after use) │ Prevents replay attacks     │
│ Sent to registered number     │ Possession factor           │
│ Rate limit OTP sends          │ Prevents SMS flood attacks  │
│ No OTP in response body       │ Never echo OTP back         │
│ Audit every OTP event         │ Fraud detection             │
└──────────────────────────────┴─────────────────────────────┘
```

---

## 9.6 Demographic Authentication

**Demographic authentication** matches the person's claimed attributes (name, date of birth, gender, address) against what is stored in the CIDR.

This is the **weakest** authentication mode — demographic data can be obtained by third parties. It is used only in limited scenarios:

```
┌──────────────────────────────────────────────────────────────┐
│             Demographic Authentication                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Use cases (limited):                                        │
│  ├── Supplementary factor alongside biometric               │
│  ├── Cross-check during enrollment quality review           │
│  └── Low-risk administrative verification                   │
│                                                              │
│  NEVER use alone for:                                        │
│  ├── Financial transactions                                 │
│  ├── Border crossing                                        │
│  └── High-value benefit claims                              │
│                                                              │
│  Request:                                                    │
│  POST /v1/auth/demographic                                   │
│  {                                                           │
│    "uin": "123456789012",                                    │
│    "name": "Hamza Rafique",                                  │
│    "dateOfBirth": "1990-05-15",                             │
│    "gender": "M"                                            │
│  }                                                           │
│                                                              │
│  Matching:                                                   │
│  ├── Exact match (name must match exactly)                  │
│  ├── Normalized match (case, diacritics ignored)            │
│  └── Partial match (configurable per field)                 │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 9.7 OIDC Authentication — eSignet

**eSignet** is MOSIP's OpenID Connect (OIDC) identity provider. It enables web and mobile applications to implement **"Sign in with National ID"** — the same pattern as "Sign in with Google" but backed by the government's NBIS.

```
┌──────────────────────────────────────────────────────────────┐
│                  eSignet / OIDC Flow                         │
└──────────────────────────────────────────────────────────────┘

1. User clicks "Sign in with National ID" on bank website

2. Browser redirects to eSignet:
   GET https://esignet.nbis.gov/authorize
   ?client_id=bank-001
   &redirect_uri=https://bank.com/callback
   &response_type=code
   &scope=openid profile email
   &state=random-state-value

3. eSignet shows authentication screen:
   ├── User enters UIN
   └── User authenticates (OTP or biometric)

4. After successful auth, eSignet redirects back:
   GET https://bank.com/callback
   ?code=AUTH_CODE_xyz123
   &state=random-state-value

5. Bank's back-end exchanges code for tokens:
   POST https://esignet.nbis.gov/token
   {
     client_id: "bank-001",
     client_secret: "...",
     code: "AUTH_CODE_xyz123",
     grant_type: "authorization_code"
   }

6. eSignet returns:
   {
     "access_token": "eyJhbGci...",
     "id_token": "eyJhbGci...",    ← contains user claims
     "token_type": "Bearer",
     "expires_in": 3600
   }

7. Bank decodes id_token (JWT):
   {
     "sub": "hashed-uin-abc123",  ← never the real UIN
     "name": "Hamza Rafique",     ← only consented claims
     "email": "h@example.com",
     "iss": "https://esignet.nbis.gov",
     "aud": "bank-001",
     "exp": 1705316400
   }

8. Bank creates session for Hamza
   → Real UIN never exposed to bank
```

### Why eSignet matters

```
Without eSignet:
  Bank integrates directly with Auth Service
  → Each bank builds custom integration
  → Inconsistent security
  → Direct access to NBIS internals

With eSignet:
  Bank uses standard OAuth2 / OIDC
  → Standard protocol any web developer knows
  → NBIS internals never exposed
  → Consistent security across all integrations
  → Government controls consent
```

---

## 9.8 Authentication Assurance Levels (AAL)

**AAL** defines the strength of an authentication event. Different services require different AALs.

```
┌──────────────────────────────────────────────────────────────┐
│          Authentication Assurance Levels (NIST 800-63B)      │
├──────────┬──────────────────────────────────────────────────┤
│ AAL      │ Requirements                                     │
├──────────┼──────────────────────────────────────────────────┤
│ AAL1     │ Single factor                                    │
│          │ Password or OTP (not both required)              │
│          │ Use for: low-risk portal login                   │
├──────────┼──────────────────────────────────────────────────┤
│ AAL2     │ Two factors (MFA required)                       │
│          │ OTP + biometric                                  │
│          │ OTP + password                                   │
│          │ Use for: eKYC, benefit claims, banking           │
├──────────┼──────────────────────────────────────────────────┤
│ AAL3     │ Hardware-based + biometric                       │
│          │ Smart card + fingerprint                         │
│          │ Use for: border crossing, high-value            │
│          │ transactions, admin access                       │
└──────────┴──────────────────────────────────────────────────┘
```

### AAL per NBIS use case

```
┌──────────────────────────────┬──────────────────────────────┐
│ Use Case                     │ Required AAL                 │
├──────────────────────────────┼──────────────────────────────┤
│ View resident portal         │ AAL1 (OTP)                  │
│ Update address               │ AAL2 (OTP + biometric)      │
│ eKYC for bank account        │ AAL2 (biometric + OTP)      │
│ Border crossing              │ AAL3 (smart card + biometric)│
│ Lock biometrics              │ AAL2 (biometric + OTP)      │
│ Admin system access          │ AAL3 (MFA mandatory)        │
│ Benefit claim                │ AAL2 (biometric)            │
│ Low-risk govt portal         │ AAL1 (OTP)                  │
└──────────────────────────────┴──────────────────────────────┘
```

---

## 9.9 Biometric Locking — Citizen Control

MOSIP provides citizens with the ability to **lock their biometric authentication** — preventing anyone from using their biometrics to authenticate, even with valid capture hardware.

```
┌──────────────────────────────────────────────────────────────┐
│                  Biometric Lock / Unlock                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Lock biometrics:                                            │
│  POST /v1/resident/biometrics/lock                          │
│  Auth: OTP (citizen must prove identity to lock)            │
│                                                              │
│  Effect:                                                     │
│  ├── Record in CIDR: biometric_locked = true                │
│  └── Any biometric auth attempt → rejected with             │
│      AUTH_003: "Biometrics locked by citizen"               │
│                                                              │
│  Unlock biometrics:                                          │
│  POST /v1/resident/biometrics/unlock                        │
│  Auth: OTP (citizen must prove identity to unlock)          │
│                                                              │
│  Use case:                                                   │
│  Citizen suspects their biometric was compromised           │
│  → Lock immediately via portal                              │
│  → Visit registration center to investigate                 │
│  → Unlock after resolution                                  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 9.10 Authentication Audit Trail

Every authentication event — success, failure, lock — must be written to the immutable audit log.

```
┌──────────────────────────────────────────────────────────────┐
│              Authentication Audit Log Entry                  │
└──────────────────────────────────────────────────────────────┘
{
  "eventId":       "EVT-20250115-00123456",
  "eventType":     "BIOMETRIC_AUTH_SUCCESS",
  "timestamp":     "2025-01-15T10:30:00.123Z",
  "traceId":       "abc-123-xyz",
  "uinHash":       "sha256:a1b2c3...",   ← hashed, never plain
  "partnerId":     "BNK-001",
  "partnerName":   "National Bank",
  "modality":      "FINGER",
  "authMode":      "BIOMETRIC",
  "aal":           "AAL2",
  "matchScore":    98.7,
  "outcome":       "SUCCESS",
  "sourceIp":      "203.0.113.42",
  "deviceId":      "DEVICE-REG-001",
  "responseTime":  143
}

Stored in:   CloudWatch Logs (Object Lock — immutable)
Retained:    7 years minimum
Indexed in:  RDS for complex queries (fraud investigation)
Encrypted:   KMS
```

### UIN in audit logs

**The UIN is never stored in plain text in audit logs.** It is always hashed (SHA-256). This prevents the audit log itself from becoming a surveillance tool — you can prove an event happened for a given UIN, but you cannot reconstruct a list of all UIns in the log.

---

## 9.11 Authentication Service — Internal Architecture

```
┌──────────────────────────────────────────────────────────────┐
│           Authentication Service Internal Design             │
└──────────────────────────────────────────────────────────────┘

                    Incoming Request
                          │
                          ▼
              ┌─────────────────────┐
              │   Auth Controller   │
              │   (API layer)       │
              │   Validates input   │
              │   Extracts UIN      │
              └──────────┬──────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │   Auth Service      │
              │   (Business logic)  │
              │                     │
              │ 1. Check UIN exists │
              │ 2. Check bio locked?│
              │ 3. Check attempt    │
              │    count            │
              │ 4. Fetch template   │
              │ 5. Run 1:1 match    │
              │ 6. Apply threshold  │
              │ 7. Build response   │
              └──────────┬──────────┘
                         │
           ┌─────────────┼──────────────┐
           │             │              │
           ▼             ▼              ▼
    ┌────────────┐ ┌──────────┐ ┌────────────┐
    │ DynamoDB   │ │  S3+KMS  │ │ Biometric  │
    │            │ │          │ │ Matcher    │
    │ Identity   │ │ Template │ │ (SDK)      │
    │ record     │ │ store    │ │            │
    │ Attempt    │ │          │ │ Returns    │
    │ counter    │ │          │ │ score      │
    └────────────┘ └──────────┘ └────────────┘
           │
           ▼
    ┌────────────┐
    │ CloudWatch │
    │            │
    │ Audit log  │
    │ write      │
    └────────────┘
```

---

## 9.12 Authentication Error Codes

Consistent, well-designed error codes are essential for relying parties to handle failures correctly.

```
┌──────────────────────────────────────────────────────────────┐
│              Authentication Error Codes                      │
├────────────┬─────────────────────────────────────────────────┤
│ Code       │ Meaning                                        │
├────────────┼─────────────────────────────────────────────────┤
│ AUTH_001   │ Biometric no-match — score below threshold     │
│ AUTH_002   │ Account locked — too many failed attempts      │
│ AUTH_003   │ Biometrics locked by citizen                   │
│ AUTH_004   │ UIN not found in registry                      │
│ AUTH_005   │ UIN record suspended                           │
│ AUTH_006   │ UIN record revoked                             │
│ AUTH_007   │ OTP expired                                    │
│ AUTH_008   │ OTP invalid — does not match                   │
│ AUTH_009   │ OTP max attempts exceeded                      │
│ AUTH_010   │ Demographic match failed                       │
│ AUTH_011   │ Biometric quality below minimum threshold      │
│ AUTH_012   │ Invalid biometric modality for this partner    │
│ AUTH_013   │ Partner not authorized for this auth mode      │
│ AUTH_014   │ Transaction ID already used (replay detected)  │
└────────────┴─────────────────────────────────────────────────┘
```

---

## 9.13 Anti-Fraud Measures

Authentication is the primary attack surface of the NBIS. Multiple layers of fraud prevention are required.

```
┌──────────────────────────────────────────────────────────────┐
│              Authentication Anti-Fraud Measures              │
├───────────────────────┬──────────────────────────────────────┤
│ Threat                │ Countermeasure                      │
├───────────────────────┼──────────────────────────────────────┤
│ Replay attack         │ Transaction ID: one-use, TTL 5 min  │
│ (resubmit old request)│ Reject duplicate transaction IDs    │
├───────────────────────┼──────────────────────────────────────┤
│ Brute force OTP       │ Max 3 attempts, 30-min lockout      │
├───────────────────────┼──────────────────────────────────────┤
│ Spoofed biometric     │ Liveness detection required         │
│ (photo / silicone)    │ Device certification (MDS)          │
├───────────────────────┼──────────────────────────────────────┤
│ SIM swap attack       │ Rate limit OTP sends per UIN        │
│ (intercept OTP SMS)   │ Alert on unusual OTP pattern        │
├───────────────────────┼──────────────────────────────────────┤
│ Bulk identity probe   │ Rate limiting per partner API key   │
│ (is this UIN valid?)  │ Generic error — no info leak        │
├───────────────────────┼──────────────────────────────────────┤
│ Insider abuse         │ Audit all auth events with partner  │
│ (partner probing)     │ Regular usage pattern review        │
├───────────────────────┼──────────────────────────────────────┤
│ Compromised partner   │ Revoke API key immediately          │
│ API key               │ Invalidate all active sessions      │
├───────────────────────┼──────────────────────────────────────┤
│ Auth from unusual     │ GuardDuty anomaly detection         │
│ geography / time      │ Alert + require step-up auth        │
└───────────────────────┴──────────────────────────────────────┘
```

---

## 9.14 AWS Architecture for Authentication

```
┌──────────────────────────────────────────────────────────────┐
│           Authentication Service — AWS Components            │
└──────────────────────────────────────────────────────────────┘

Relying Party (Bank)
      │
      │ HTTPS + JWT + API Key
      ▼
API Gateway
      │ validates JWT (Cognito)
      │ checks rate limit (usage plan)
      ▼
Lambda Authorizer
      │ checks role + scope
      ▼
Auth Service (ECS Fargate — private subnet)
      │
      ├──► DynamoDB
      │    GET identity record by UIN
      │    UPDATE attempt counter
      │    CHECK biometric_locked flag
      │
      ├──► S3 + KMS
      │    GET encrypted biometric template
      │    DECRYPT template (KMS)
      │
      ├──► Biometric Matcher (SDK in same container)
      │    COMPARE probe vs enrolled template
      │    RETURN match score
      │
      ├──► ElastiCache Redis
      │    GET / SET / DEL OTP cache
      │
      ├──► SNS
      │    SEND OTP via SMS gateway
      │
      └──► CloudWatch Logs
           WRITE audit event (immutable)

Response:
      └──► { match: true/false, authToken, score }
      → back through ECS → API Gateway → Bank
```

---

## 9.15 Authentication vs Authorization — Final Clarity

Since these are confused constantly, here is the definitive side-by-side for NBIS:

```
┌──────────────────────────────────────────────────────────────┐
│          Authentication vs Authorization in NBIS             │
├────────────────────────┬─────────────────────────────────────┤
│ AUTHENTICATION         │ AUTHORIZATION                       │
├────────────────────────┼─────────────────────────────────────┤
│ "Who are you?"         │ "What can you do?"                  │
│                        │                                     │
│ Checked FIRST          │ Checked AFTER authentication        │
│                        │                                     │
│ Uses: UIN + biometric  │ Uses: Role + scope in JWT           │
│ Uses: UIN + OTP        │                                     │
│                        │                                     │
│ Result: MATCH /        │ Result: ALLOW / DENY                │
│         NO_MATCH        │                                     │
│                        │                                     │
│ Failure: 401           │ Failure: 403                        │
│ Unauthorized           │ Forbidden                           │
│                        │                                     │
│ AWS: Cognito           │ AWS: IAM + Lambda Authorizer        │
│                        │                                     │
│ Chapter: 9             │ Chapter: 10 (RBAC)                  │
└────────────────────────┴─────────────────────────────────────┘
```

---

## 9.16 MOSIP Reference

```
┌──────────────────────────────────────────────────────────────┐
│              MOSIP Authentication Reference                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Module:    ID Authentication (IDA)                         │
│                                                              │
│  Auth modes supported:                                       │
│  ├── BIO — biometric (finger, iris, face)                   │
│  ├── OTP — one-time password (SMS)                          │
│  ├── DEMO — demographic attributes                          │
│  └── Combinations: BIO+OTP, OTP+DEMO                       │
│                                                              │
│  Key design decisions:                                       │
│  ├── All auth requests are signed by the partner            │
│  │   (MOSIP partner SDK signs the request payload)          │
│  ├── All biometric data is encrypted with partner key       │
│  │   before transmission                                    │
│  ├── Response is signed by NBIS private key                 │
│  └── Transaction ID enforced for replay prevention         │
│                                                              │
│  eSignet:                                                    │
│  ├── OIDC Core 1.0 compliant                                │
│  ├── OAuth2 authorization code + PKCE                       │
│  └── Backed by IDA for the actual auth                      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 9.17 Key Terms

| Term | Definition |
|------|-----------|
| **Authentication** | Proving you are who you claim to be |
| **Authorization** | What you are allowed to do after authentication |
| **Knowledge factor** | Something you know — password, PIN |
| **Possession factor** | Something you have — OTP, smart card |
| **Inherence factor** | Something you are — fingerprint, iris, face |
| **MFA** | Multi-Factor Authentication — two or more factors combined |
| **OTP** | One-Time Password — single-use code delivered via SMS or email |
| **Biometric auth** | 1:1 match of live capture against enrolled template |
| **Demographic auth** | Matching biographical attributes (name, DOB) against registry |
| **AAL** | Authentication Assurance Level — strength of the auth event |
| **eSignet** | MOSIP's OIDC identity provider — enables "Sign in with National ID" |
| **OIDC** | OpenID Connect — protocol for federated identity authentication |
| **JWT** | JSON Web Token — signed token carrying identity claims |
| **Replay attack** | Resubmitting a captured valid request to gain unauthorized access |
| **Biometric lock** | Citizen-controlled flag preventing biometric auth on their account |
| **Transaction ID** | Unique ID per auth request — prevents replay attacks |
| **Token bucket** | Rate limiting algorithm — burst allowed, steady rate enforced |
| **Step-up auth** | Requiring stronger auth for higher-risk operations within a session |

---

## 9.18 Key Takeaways

- **Authentication is always about proof** — something you know, have, or are. Biometric (something you ARE) is the strongest factor in an NBIS context.
- **Biometric auth is 1:1 verification** — the system fetches one enrolled template and compares the live capture against it. It is not a search.
- **OTP has strict rules** — 6 digits minimum, 5-minute expiry, single use, max 3 attempts, rate limited. Violating any of these creates a fraud vector.
- **eSignet is the right bridge to the internet** — it exposes a standard OIDC interface so relying parties never touch NBIS internals directly.
- **AAL determines which auth mode is required** — not all services need biometric. Match the AAL to the risk level of the operation.
- **Audit every authentication event** — success, failure, lock, unlock. The UIN is always hashed in logs. The audit trail is immutable.
- **Transaction IDs prevent replay attacks** — every auth request must have a unique, short-lived transaction ID. Duplicate IDs are rejected immediately.
- **Biometric locking gives citizens control** — a citizen who suspects compromise can instantly lock their biometric auth via the self-service portal.

---

## 9.19 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 10 | Authorization (RBAC) — role design, policy enforcement, least privilege |
| Chapter 11 | Event-Driven Architecture — async patterns, SQS, idempotency |
| Chapter 51 | JWT — structure, claims, signing, validation deep dive |

---

*Chapter 9 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*
