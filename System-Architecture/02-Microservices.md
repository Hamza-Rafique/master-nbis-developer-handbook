# Chapter 7 — Microservices

> **Part II — System Architecture**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 7.1 What are Microservices?

**Microservices** is an architectural style where a system is built as a collection of small, independently deployable services — each responsible for one specific business function, owning its own data, and communicating with other services only through well-defined APIs or events.

The alternative is a **monolith** — one large application that handles everything.

```
┌──────────────────────────────────────────────────────────────┐
│              Monolith vs Microservices                       │
├──────────────────────────┬───────────────────────────────────┤
│       MONOLITH           │       MICROSERVICES               │
├──────────────────────────┼───────────────────────────────────┤
│ One codebase             │ Many codebases                    │
│ One deployment unit      │ Each service deploys independently│
│ One shared database      │ Each service owns its data        │
│ Scale whole app          │ Scale each service independently  │
│ One failure = all down   │ One failure = isolated            │
│ Simple to start          │ Complex to coordinate             │
│ Hard to scale teams      │ Teams own their services          │
└──────────────────────────┴───────────────────────────────────┘
```

---

## 7.2 Why NBIS Must Use Microservices

An NBIS is not a simple CRUD application. It has radically different components with different:

```
┌──────────────────────────────────────────────────────────────┐
│         Why Microservices Are Mandatory for NBIS             │
├───────────────────────┬──────────────────────────────────────┤
│ Different scale       │ Verification: millions/day           │
│ requirements          │ Enrollment: thousands/day            │
│                       │ → Must scale independently           │
├───────────────────────┼──────────────────────────────────────┤
│ Different security    │ CIDR access: ultra-restricted        │
│ requirements          │ Pre-registration: public-facing      │
│                       │ → Must have isolated security zones  │
├───────────────────────┼──────────────────────────────────────┤
│ Different tech needs  │ Biometric: GPU-optimized             │
│                       │ Dedup: ABIS integration              │
│                       │ eKYC: low-latency REST               │
│                       │ → Right tool for each job            │
├───────────────────────┼──────────────────────────────────────┤
│ Different teams       │ Enrollment team                      │
│                       │ Biometrics team                      │
│                       │ Integration team                     │
│                       │ → Each team owns their service       │
├───────────────────────┼──────────────────────────────────────┤
│ Zero-downtime         │ Upgrading biometric SDK must not     │
│ upgrades              │ take down verification               │
│                       │ → Deploy each service independently  │
└───────────────────────┴──────────────────────────────────────┘
```

---

## 7.3 The Golden Rule — Single Responsibility

Every microservice must follow the **Single Responsibility Principle (SRP)**: it does **one thing** and does it well. If you can describe a service's job using the word "and", it is probably two services.

```
┌──────────────────────────────────────────────────────────────┐
│              Single Responsibility Test                      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ❌ BAD: "This service handles enrollment AND biometrics     │
│          AND deduplication AND credential issuance"          │
│                                                              │
│  ✅ GOOD: "This service validates and stores the            │
│            enrollment packet"                                │
│                                                              │
│  ✅ GOOD: "This service checks biometric quality"           │
│                                                              │
│  ✅ GOOD: "This service assigns a UIN and writes            │
│            the identity record"                              │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 7.4 The Second Golden Rule — Own Your Data

Every microservice **owns its own data store**. No two services share a database directly.

```
┌──────────────────────────────────────────────────────────────┐
│                  Database Per Service Rule                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ❌ WRONG — Shared database:                                 │
│                                                              │
│  Registration Service ──┐                                    │
│  Biometric Service    ──┼──► SHARED DATABASE                 │
│  Verification Service ──┘                                    │
│                                                              │
│  → Any service can read/write any table                     │
│  → Schema changes break all services                        │
│  → Cannot scale databases independently                     │
│                                                              │
│  ✅ CORRECT — Database per service:                          │
│                                                              │
│  Registration Service ──► DynamoDB (enrollment records)     │
│  Biometric Service    ──► S3 (template store)               │
│  Verification Service ──► DynamoDB (identity records)       │
│                                                              │
│  → Each service owns and controls its data                  │
│  → Schema changes are isolated                              │
│  → Each database scales with its service                    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 7.5 NBIS Microservices — Complete Map

```
┌─────────────────────────────────────────────────────────────────────┐
│                    NBIS Microservice Map                            │
├────────────────────────────┬────────────────────────────────────────┤
│                            │                                        │
│     WRITE PATH             │         READ PATH                      │
│     (Enrollment)           │         (Verification)                 │
│                            │                                        │
│  ┌─────────────────────┐   │   ┌─────────────────────────────────┐  │
│  │ Pre-Registration    │   │   │ Authentication Service          │  │
│  │ Service             │   │   └─────────────────────────────────┘  │
│  └─────────────────────┘   │   ┌─────────────────────────────────┐  │
│  ┌─────────────────────┐   │   │ eKYC Service                   │  │
│  │ Registration        │   │   └─────────────────────────────────┘  │
│  │ Service             │   │   ┌─────────────────────────────────┐  │
│  └─────────────────────┘   │   │ eSignet (OIDC Provider)        │  │
│  ┌─────────────────────┐   │   └─────────────────────────────────┘  │
│  │ Biometric Service   │   │   ┌─────────────────────────────────┐  │
│  └─────────────────────┘   │   │ Resident Services               │  │
│  ┌─────────────────────┐   │   └─────────────────────────────────┘  │
│  │ Deduplication       │   │                                        │
│  │ Service             │   │         MANAGEMENT                     │
│  └─────────────────────┘   │   ┌─────────────────────────────────┐  │
│  ┌─────────────────────┐   │   │ Partner Management Service     │  │
│  │ UIN Service         │   │   └─────────────────────────────────┘  │
│  └─────────────────────┘   │   ┌─────────────────────────────────┐  │
│  ┌─────────────────────┐   │   │ Admin Service                  │  │
│  │ Credential Service  │   │   └─────────────────────────────────┘  │
│  └─────────────────────┘   │   ┌─────────────────────────────────┐  │
│                            │   │ Notification Service            │  │
│                            │   └─────────────────────────────────┘  │
└────────────────────────────┴────────────────────────────────────────┘
```

---

## 7.6 Service Deep Dives

### 7.6.1 Pre-Registration Service

```
┌──────────────────────────────────────────────────────────────┐
│                  Pre-Registration Service                    │
├──────────────────────┬───────────────────────────────────────┤
│ Responsibility       │ Allow citizens to book enrollment     │
│                      │ appointments online before visiting   │
│                      │ the registration center              │
├──────────────────────┼───────────────────────────────────────┤
│ Owns                 │ Appointment records                   │
│                      │ Pre-filled demographic data           │
│                      │ Document upload (pre-scan)            │
├──────────────────────┼───────────────────────────────────────┤
│ Exposes              │ POST /appointments (book slot)        │
│                      │ GET  /appointments/{id} (status)      │
│                      │ PUT  /appointments/{id} (reschedule)  │
├──────────────────────┼───────────────────────────────────────┤
│ Data store           │ DynamoDB (appointments table)         │
│                      │ S3 (pre-uploaded documents)           │
├──────────────────────┼───────────────────────────────────────┤
│ Communicates with    │ Notification Service (confirmation)   │
│                      │ Registration Service (at center)      │
├──────────────────────┼───────────────────────────────────────┤
│ AWS                  │ ECS Fargate + DynamoDB + S3 + SNS     │
│ MOSIP module         │ Pre-Registration                      │
└──────────────────────┴───────────────────────────────────────┘
```

---

### 7.6.2 Registration Service

```
┌──────────────────────────────────────────────────────────────┐
│                    Registration Service                      │
├──────────────────────┬───────────────────────────────────────┤
│ Responsibility       │ Accept, validate, and store the       │
│                      │ enrollment packet from the            │
│                      │ Registration Client (edge device)     │
├──────────────────────┼───────────────────────────────────────┤
│ Owns                 │ Enrollment packets (raw, encrypted)   │
│                      │ Enrollment status records             │
├──────────────────────┼───────────────────────────────────────┤
│ Exposes              │ POST /registrations (upload packet)   │
│                      │ GET  /registrations/{id} (status)     │
├──────────────────────┼───────────────────────────────────────┤
│ Data store           │ S3 (encrypted packet storage)         │
│                      │ DynamoDB (enrollment status)          │
├──────────────────────┼───────────────────────────────────────┤
│ Communicates with    │ Publishes to SQS (triggers pipeline)  │
├──────────────────────┼───────────────────────────────────────┤
│ Does NOT             │ Read or write to the CIDR             │
│                      │ Perform quality checks                │
│                      │ Call the ABIS                         │
├──────────────────────┼───────────────────────────────────────┤
│ AWS                  │ ECS Fargate + S3 + DynamoDB + SQS     │
│ MOSIP module         │ Registration Processor (intake stage) │
└──────────────────────┴───────────────────────────────────────┘
```

---

### 7.6.3 Biometric Service

```
┌──────────────────────────────────────────────────────────────┐
│                     Biometric Service                        │
├──────────────────────┬───────────────────────────────────────┤
│ Responsibility       │ Extract biometric templates from raw  │
│                      │ captures, run quality checks, perform │
│                      │ liveness detection                    │
├──────────────────────┼───────────────────────────────────────┤
│ Owns                 │ Biometric templates (encrypted)       │
│                      │ Quality scores per capture            │
├──────────────────────┼───────────────────────────────────────┤
│ Exposes              │ (Internal only — not public-facing)   │
│                      │ POST /biometrics/extract              │
│                      │ POST /biometrics/quality-check        │
├──────────────────────┼───────────────────────────────────────┤
│ Data store           │ S3 + KMS (encrypted templates)        │
├──────────────────────┼───────────────────────────────────────┤
│ Communicates with    │ Consumes from SQS                     │
│                      │ Publishes result to SQS               │
│                      │ Calls ABIS (for dedup trigger)        │
├──────────────────────┼───────────────────────────────────────┤
│ Does NOT             │ Store demographic data                │
│                      │ Assign UINs                           │
│                      │ Issue credentials                     │
├──────────────────────┼───────────────────────────────────────┤
│ AWS                  │ ECS Fargate (GPU optional) + S3 + KMS │
│ MOSIP module         │ Registration Processor (bio stage)    │
└──────────────────────┴───────────────────────────────────────┘
```

---

### 7.6.4 Deduplication Service

```
┌──────────────────────────────────────────────────────────────┐
│                  Deduplication Service                       │
├──────────────────────┬───────────────────────────────────────┤
│ Responsibility       │ Orchestrate the 1:N biometric search  │
│                      │ via the ABIS, interpret results, and  │
│                      │ flag or clear enrollments             │
├──────────────────────┼───────────────────────────────────────┤
│ Owns                 │ Dedup job records                     │
│                      │ Candidate match results               │
│                      │ Manual review queue                   │
├──────────────────────┼───────────────────────────────────────┤
│ Exposes              │ (Internal only)                       │
│                      │ GET /dedup/{jobId} (status)           │
├──────────────────────┼───────────────────────────────────────┤
│ Data store           │ DynamoDB (dedup job status)           │
│                      │ RDS (review queue with history)       │
├──────────────────────┼───────────────────────────────────────┤
│ Communicates with    │ Sends jobs to ABIS via SQS            │
│                      │ Receives results from ABIS via SQS    │
│                      │ Publishes CLEARED or FLAGGED event    │
├──────────────────────┼───────────────────────────────────────┤
│ Does NOT             │ Perform the biometric matching itself │
│                      │ (ABIS does this)                      │
├──────────────────────┼───────────────────────────────────────┤
│ AWS                  │ Lambda + DynamoDB + SQS + RDS         │
│ MOSIP module         │ Registration Processor (dedup stage)  │
└──────────────────────┴───────────────────────────────────────┘
```

---

### 7.6.5 UIN Service

```
┌──────────────────────────────────────────────────────────────┐
│                       UIN Service                            │
├──────────────────────┬───────────────────────────────────────┤
│ Responsibility       │ Generate and assign a Unique Identity │
│                      │ Number, create the authoritative      │
│                      │ identity record in the CIDR           │
├──────────────────────┼───────────────────────────────────────┤
│ Owns                 │ The identity record in the CIDR       │
│                      │ UIN generation pool                   │
├──────────────────────┼───────────────────────────────────────┤
│ Exposes              │ (Internal only — extremely sensitive) │
│                      │ POST /uin/assign                      │
├──────────────────────┼───────────────────────────────────────┤
│ Data store           │ DynamoDB (identity records — CIDR)    │
├──────────────────────┼───────────────────────────────────────┤
│ Communicates with    │ Triggered by DEDUP_CLEARED event      │
│                      │ Publishes UIN_ASSIGNED event          │
├──────────────────────┼───────────────────────────────────────┤
│ Does NOT             │ Accept public-facing API calls        │
│                      │ Modify biometric data                 │
│                      │ Issue credentials                     │
├──────────────────────┼───────────────────────────────────────┤
│ Security note        │ This is the most sensitive service.   │
│                      │ No direct access except via events.   │
│                      │ Every UIN assignment is audited.      │
├──────────────────────┼───────────────────────────────────────┤
│ AWS                  │ Lambda + DynamoDB + CloudWatch        │
│ MOSIP module         │ ID Repository                         │
└──────────────────────┴───────────────────────────────────────┘
```

---

### 7.6.6 Credential Service

```
┌──────────────────────────────────────────────────────────────┐
│                    Credential Service                        │
├──────────────────────┬───────────────────────────────────────┤
│ Responsibility       │ Generate and issue identity           │
│                      │ credentials — smart card print jobs,  │
│                      │ Verifiable Credentials (W3C VC),      │
│                      │ QR codes                              │
├──────────────────────┼───────────────────────────────────────┤
│ Owns                 │ Credential issuance records           │
│                      │ Signed VC artifacts                   │
├──────────────────────┼───────────────────────────────────────┤
│ Exposes              │ GET /credentials/{uin} (download VC)  │
│                      │ POST /credentials/revoke              │
├──────────────────────┼───────────────────────────────────────┤
│ Data store           │ S3 (signed credential artifacts)      │
│                      │ DynamoDB (issuance records)           │
├──────────────────────┼───────────────────────────────────────┤
│ Communicates with    │ Triggered by UIN_ASSIGNED event       │
│                      │ Sends print job to Credential Vendor  │
│                      │ Publishes CREDENTIAL_ISSUED event     │
├──────────────────────┼───────────────────────────────────────┤
│ AWS                  │ Lambda + S3 + KMS + DynamoDB + SNS    │
│ MOSIP module         │ Credential Service                    │
└──────────────────────┴───────────────────────────────────────┘
```

---

### 7.6.7 Authentication Service

```
┌──────────────────────────────────────────────────────────────┐
│                  Authentication Service                      │
├──────────────────────┬───────────────────────────────────────┤
│ Responsibility       │ Perform 1:1 identity verification for │
│                      │ relying parties — biometric, OTP,     │
│                      │ or demographic match modes            │
├──────────────────────┼───────────────────────────────────────┤
│ Owns                 │ Authentication transaction records    │
│                      │ OTP cache (short TTL)                 │
├──────────────────────┼───────────────────────────────────────┤
│ Exposes              │ POST /auth/biometric                  │
│                      │ POST /auth/otp                        │
│                      │ POST /auth/demographic                │
│                      │ Returns: { match: true/false, score } │
├──────────────────────┼───────────────────────────────────────┤
│ Data store           │ DynamoDB (auth transaction log)       │
│                      │ ElastiCache Redis (OTP cache)         │
├──────────────────────┼───────────────────────────────────────┤
│ Communicates with    │ CIDR (reads identity record by UIN)   │
│                      │ Biometric matcher (internal SDK)      │
│                      │ Notification Service (OTP delivery)   │
├──────────────────────┼───────────────────────────────────────┤
│ Does NOT             │ Return demographic data to caller     │
│                      │ (That is eKYC service's job)          │
├──────────────────────┼───────────────────────────────────────┤
│ SLA                  │ < 200ms response time                 │
│                      │ 99.99% availability                   │
├──────────────────────┼───────────────────────────────────────┤
│ AWS                  │ Lambda + DynamoDB + ElastiCache       │
│ MOSIP module         │ ID Authentication                     │
└──────────────────────┴───────────────────────────────────────┘
```

---

### 7.6.8 Notification Service

```
┌──────────────────────────────────────────────────────────────┐
│                   Notification Service                       │
├──────────────────────┬───────────────────────────────────────┤
│ Responsibility       │ Send SMS, email, and push             │
│                      │ notifications to citizens at key      │
│                      │ lifecycle events                      │
├──────────────────────┼───────────────────────────────────────┤
│ Triggered by events  │ ENROLLMENT_RECEIVED                   │
│                      │ DEDUP_CLEARED                         │
│                      │ UIN_ASSIGNED                          │
│                      │ CREDENTIAL_ISSUED                     │
│                      │ AUTH_ATTEMPT (suspicious login)       │
├──────────────────────┼───────────────────────────────────────┤
│ Data store           │ DynamoDB (notification log)           │
├──────────────────────┼───────────────────────────────────────┤
│ AWS                  │ Lambda + SNS + SES (email) + SQS      │
│ MOSIP module         │ Notification service                  │
└──────────────────────┴───────────────────────────────────────┘
```

---

## 7.7 Inter-Service Communication Patterns

Services communicate in two ways — synchronous and asynchronous.

```
┌──────────────────────────────────────────────────────────────┐
│           Inter-Service Communication Patterns               │
├───────────────────────┬──────────────────────────────────────┤
│ Pattern               │ When to use                          │
├───────────────────────┼──────────────────────────────────────┤
│ REST (sync)           │ Read path — caller needs immediate   │
│ HTTP API call         │ response (verification, eKYC)        │
│                       │                                      │
│ Example:              │                                      │
│ Bank → Auth Service   │                                      │
│ → response in 200ms   │                                      │
├───────────────────────┼──────────────────────────────────────┤
│ SQS Event (async)     │ Write path — caller does not need    │
│ Publish + subscribe   │ immediate response (enrollment,      │
│                       │ dedup, credential issuance)          │
│                       │                                      │
│ Example:              │                                      │
│ Registration Service  │                                      │
│ → publishes event     │                                      │
│ → Biometric Service   │                                      │
│   picks up later      │                                      │
└───────────────────────┴──────────────────────────────────────┘
```

### Rule of thumb

```
IF the caller needs the result NOW       → use REST (sync)
IF the work can happen in the background → use SQS (async)
IF multiple services need the same event → use EventBridge (fan-out)
```

---

## 7.8 Service Structure — Anatomy of One Microservice

Every NBIS microservice is structured the same way internally:

```
┌──────────────────────────────────────────────────────────────┐
│           Anatomy of an NBIS Microservice                    │
│                  (Spring Boot / Java)                        │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────┐     │
│  │  API Layer (Controller)                             │     │
│  │  Receives HTTP requests                             │     │
│  │  Validates input                                    │     │
│  │  Returns HTTP responses                             │     │
│  └───────────────────────┬─────────────────────────────┘     │
│                          │                                   │
│  ┌───────────────────────▼─────────────────────────────┐     │
│  │  Service Layer (Business Logic)                     │     │
│  │  Orchestrates the work                              │     │
│  │  Applies business rules                             │     │
│  │  Calls other services if needed                     │     │
│  └───────────────────────┬─────────────────────────────┘     │
│                          │                                   │
│  ┌───────────────────────▼─────────────────────────────┐     │
│  │  Repository Layer (Data Access)                     │     │
│  │  Reads / writes to the service's data store         │     │
│  │  DynamoDB / RDS / S3 access                         │     │
│  └───────────────────────┬─────────────────────────────┘     │
│                          │                                   │
│  ┌───────────────────────▼─────────────────────────────┐     │
│  │  Data Store                                         │     │
│  │  Owned exclusively by this service                  │     │
│  │  No other service accesses this directly            │     │
│  └─────────────────────────────────────────────────────┘     │
│                                                              │
│  Cross-cutting (every layer):                               │
│  ├── Audit logging (every state change)                     │
│  ├── Error handling (structured error responses)            │
│  ├── Health check endpoint (/actuator/health)               │
│  └── Metrics endpoint (/actuator/metrics)                   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 7.9 Service Contract — API Design Rules

Every service exposes a **contract** — a well-defined API that other services and clients use. The contract must:

```
┌──────────────────────────────────────────────────────────────┐
│              API Contract Rules for NBIS Services            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Versioned                                                │
│     /v1/auth/biometric                                       │
│     /v2/auth/biometric  ← new version, old still works      │
│                                                              │
│  2. Consistent error format                                  │
│     {                                                        │
│       "errorCode": "BIO_001",                                │
│       "message": "Biometric quality below threshold",        │
│       "timestamp": "2025-01-15T10:30:00Z",                   │
│       "traceId": "abc-123-xyz"                               │
│     }                                                        │
│                                                              │
│  3. Idempotent writes (safe to retry)                        │
│     POST /registrations with same packetId                   │
│     → returns same result, does NOT create duplicate         │
│                                                              │
│  4. Documented (OpenAPI / Swagger spec)                      │
│     Every endpoint, every field, every error code           │
│                                                              │
│  5. Never break existing callers                             │
│     Add fields → OK                                          │
│     Remove fields → NOT OK (major version bump)             │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 7.10 Service Ownership and Deployment

```
┌──────────────────────────────────────────────────────────────┐
│              Service Ownership Model                         │
├──────────────────────┬───────────────────────────────────────┤
│ Principle            │ "You build it, you own it, you        │
│                      │  run it" — Werner Vogels (AWS CTO)    │
├──────────────────────┼───────────────────────────────────────┤
│ Each service has     │ One team responsible for it           │
│                      │ Their own CI/CD pipeline              │
│                      │ Their own deployment schedule         │
│                      │ Their own on-call rotation            │
├──────────────────────┼───────────────────────────────────────┤
│ Deployment unit      │ Docker container                      │
│                      │ Deployed to ECS Fargate               │
│                      │ Tagged by service + version           │
├──────────────────────┼───────────────────────────────────────┤
│ Container registry   │ AWS ECR                               │
│                      │ One repo per service                  │
└──────────────────────┴───────────────────────────────────────┘
```

### Deployment pipeline per service

```
Developer pushes code
      │
      ▼
GitHub Actions CI pipeline
      │
      ├──► Run unit tests
      ├──► Run integration tests
      ├──► Build Docker image
      ├──► Push to ECR
      └──► Deploy to ECS (blue/green)
                │
                ▼
      Health check passes?
          │           │
         YES          NO
          │           │
      Traffic       Rollback to
      switched      previous version
```

---

## 7.11 Service Discovery

When services need to call each other (synchronously), they need to know where each other lives. In a containerized environment on ECS, IP addresses change constantly.

```
┌──────────────────────────────────────────────────────────────┐
│              Service Discovery in NBIS                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Option 1: AWS Cloud Map                                     │
│  ├── Services register themselves on startup                │
│  ├── Other services look up by name                         │
│  └── Example: auth-service.nbis.local                       │
│                                                              │
│  Option 2: Internal Application Load Balancer               │
│  ├── Each service gets a stable internal DNS name           │
│  ├── ALB routes to healthy ECS tasks                        │
│  └── Example: internal-alb.nbis.local/auth                  │
│                                                              │
│  Best practice for NBIS:                                     │
│  Internal ALB for synchronous service-to-service calls      │
│  SQS for asynchronous calls (no discovery needed)           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 7.12 Resilience Patterns

Microservices must be designed to **fail gracefully** — one service going down must not cascade and take down the whole system.

```
┌──────────────────────────────────────────────────────────────┐
│              Resilience Patterns for NBIS                    │
├───────────────────────┬──────────────────────────────────────┤
│ Pattern               │ How it works                         │
├───────────────────────┼──────────────────────────────────────┤
│ Circuit Breaker       │ If Auth Service fails 5x in a row,  │
│                       │ stop calling it for 30 seconds.      │
│                       │ Return cached result or error fast.  │
├───────────────────────┼──────────────────────────────────────┤
│ Retry with backoff    │ If a call fails, retry after 1s,    │
│                       │ then 2s, then 4s (exponential).     │
│                       │ Max 3 retries, then give up.         │
├───────────────────────┼──────────────────────────────────────┤
│ Timeout               │ Every external call has a timeout.  │
│                       │ Auth Service: 500ms max.             │
│                       │ ABIS call: 30 seconds max.           │
├───────────────────────┼──────────────────────────────────────┤
│ Bulkhead              │ ABIS dedup calls get their own       │
│                       │ thread pool. Slow ABIS cannot block  │
│                       │ the verification service.            │
├───────────────────────┼──────────────────────────────────────┤
│ Dead Letter Queue     │ If enrollment packet processing      │
│                       │ fails, move to DLQ — never lost.    │
└───────────────────────┴──────────────────────────────────────┘
```

---

## 7.13 Complete Service Inventory Table

```
┌────────────────────────────┬─────────────────────┬───────────────────────────┐
│ Service                    │ Data Store          │ AWS Services              │
├────────────────────────────┼─────────────────────┼───────────────────────────┤
│ Pre-Registration           │ DynamoDB + S3       │ ECS + DynamoDB + S3 + SNS │
│ Registration               │ S3 + DynamoDB       │ ECS + S3 + DynamoDB + SQS │
│ Biometric                  │ S3 + KMS            │ ECS + S3 + KMS + SQS      │
│ Deduplication              │ DynamoDB + RDS      │ Lambda + DynamoDB + SQS   │
│ UIN                        │ DynamoDB (CIDR)     │ Lambda + DynamoDB         │
│ Credential                 │ S3 + DynamoDB       │ Lambda + S3 + KMS + SNS   │
│ Authentication             │ DynamoDB + Redis    │ Lambda + DynamoDB + Cache │
│ eKYC                       │ DynamoDB (log)      │ Lambda + KMS + DynamoDB   │
│ eSignet (OIDC)             │ DynamoDB + Redis    │ ECS + DynamoDB + Cache    │
│ Resident Services          │ DynamoDB (own view) │ ECS + DynamoDB            │
│ Partner Management         │ RDS                 │ ECS + RDS                 │
│ Admin                      │ RDS                 │ ECS + RDS                 │
│ Notification               │ DynamoDB (log)      │ Lambda + SNS + SES + SQS  │
└────────────────────────────┴─────────────────────┴───────────────────────────┘
```

---

## 7.14 Key Terms

| Term | Definition |
|------|-----------|
| **Microservice** | A small, independently deployable service with a single responsibility |
| **Monolith** | A single application that handles all functions — the opposite of microservices |
| **Single Responsibility Principle** | Each service does one thing only |
| **Database per service** | Each service owns its own data store — no shared databases |
| **Service contract** | The API a service exposes — versioned, documented, never broken |
| **Idempotency** | Same request submitted twice produces same result, no side effects |
| **Circuit breaker** | Stops calling a failing service to prevent cascade failures |
| **Bulkhead** | Isolates resource pools so one slow service cannot starve others |
| **Dead Letter Queue** | Holds failed messages for investigation — prevents silent data loss |
| **Service discovery** | Mechanism for services to find each other's network location |
| **Blue/green deployment** | Run old and new version simultaneously, switch traffic when new is healthy |
| **ECS Fargate** | AWS serverless container platform — runs each microservice as containers |
| **ECR** | AWS Elastic Container Registry — stores Docker images per service |
| **Trace ID** | Unique ID attached to a request that flows through all services for debugging |

---

## 7.15 Key Takeaways

- **Single responsibility is the first rule** — if a service does two things, split it. Smaller services are easier to test, deploy, and reason about.
- **Database per service is mandatory** — shared databases create hidden coupling. A schema change in one place breaks everything.
- **The write path is async, the read path is sync** — this design decision drives the entire service communication strategy.
- **Every service must be designed to fail** — circuit breakers, retries, timeouts, and DLQs are not optional. In a national system serving millions, partial failure is inevitable.
- **Idempotency is a safety requirement** — network retries are real. An enrollment packet submitted twice must not create two identity records.
- **Ownership is non-technical** — one team owns one service. They build it, deploy it, and are on call for it. This is what makes microservices scale organizationally.
- **The UIN Service is the most sensitive microservice** — it has no public API, is triggered only by internal events, and every action is audited. Treat it accordingly.

---

## 7.16 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 8 | API Gateway — deep dive into routing, auth, rate limiting, WAF |
| Chapter 11 | Event-Driven Architecture — SQS patterns, idempotency, dead letter queues |
| Chapter 12 | Message Queues — SQS standard vs FIFO, visibility timeout, DLQ design |

---

*Chapter 7 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*
