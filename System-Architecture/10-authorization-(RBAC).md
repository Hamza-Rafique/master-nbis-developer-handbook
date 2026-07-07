# Chapter 10 — Authorization (RBAC)

> **Part II — System Architecture**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 10.1 What is Authorization?

**Authorization** is the process of determining what an authenticated identity is **permitted to do**.

Authentication (Chapter 9) answers: *"Who are you?"*
Authorization answers: *"What are you allowed to do?"*

Authorization always happens **after** authentication. You cannot authorize an unknown caller.

```
┌──────────────────────────────────────────────────────────────┐
│              Authorization in Plain English                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Person authenticated as: Hamza (Enrollment Officer)        │
│                                                              │
│  Hamza tries to: POST /v1/admin/revoke                       │
│                                                              │
│  Authorization checks:                                       │
│  ├── Does Hamza's role (ENROLLMENT_OFFICER) have            │
│  │   permission to call /v1/admin/revoke?                   │
│  └── NO → 403 Forbidden                                     │
│                                                              │
│  Hamza tries to: POST /v1/enrollments                        │
│                                                              │
│  Authorization checks:                                       │
│  ├── Does Hamza's role (ENROLLMENT_OFFICER) have            │
│  │   permission to call /v1/enrollments?                    │
│  └── YES → 200 OK                                           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 10.2 What is RBAC?

**Role-Based Access Control (RBAC)** is the authorization model where:

- **Permissions** are attached to **roles**
- **Roles** are assigned to **users or systems**
- Users get permissions **through their roles** — not directly

```
┌──────────────────────────────────────────────────────────────┐
│                    RBAC Structure                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   USER ──────────► ROLE ──────────► PERMISSIONS             │
│                                                              │
│   Hamza ─────────► ENROLLMENT_OFFICER ──► can create        │
│                                           enrollment         │
│                                           can capture        │
│                                           biometrics         │
│                                                              │
│   Sara ──────────► SYSTEM_ADMIN ────────► can revoke        │
│                                           identity           │
│                                           can read audit     │
│                                           logs               │
│                                                              │
│   BankApp ───────► RELYING_PARTY ───────► can call          │
│                                           /v1/auth/*         │
│                                           can call          │
│                                           /v1/ekyc           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Why RBAC over direct permission assignment

```
┌──────────────────────────────────────────────────────────────┐
│           Direct Assignment vs RBAC                          │
├──────────────────────────┬───────────────────────────────────┤
│ Direct Assignment        │ RBAC                             │
├──────────────────────────┼───────────────────────────────────┤
│ 500 enrollment officers  │ 1 role definition                │
│ × 15 permissions each    │ (ENROLLMENT_OFFICER)             │
│ = 7,500 entries to manage│ 500 users assigned to role       │
│                          │ = 500 entries to manage          │
├──────────────────────────┼───────────────────────────────────┤
│ Add a new permission:    │ Add permission to role:          │
│ Update 500 user records  │ Update 1 role definition         │
├──────────────────────────┼───────────────────────────────────┤
│ Hard to audit            │ Easy to audit                    │
│ ("what can Hamza do?")   │ ("what roles does Hamza have?") │
└──────────────────────────┴───────────────────────────────────┘
```

---

## 10.3 The Principle of Least Privilege

The **Principle of Least Privilege (PoLP)** states:

> Every user, system, and service must have the **minimum permissions** needed to perform their function — nothing more.

This is the single most important security principle in NBIS design. A breach is only as damaging as the permissions of the compromised account.

```
┌──────────────────────────────────────────────────────────────┐
│           Least Privilege in Practice                        │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ❌ WRONG — Over-privileged:                                  │
│                                                              │
│  Enrollment officer role:                                    │
│  ├── CREATE enrollment ✅ (needed)                           │
│  ├── READ any citizen record ❌ (not needed)                 │
│  ├── REVOKE identity ❌ (not needed)                         │
│  └── EXPORT bulk data ❌ (never needed)                      │
│                                                              │
│  → If officer account is compromised:                       │
│    attacker can read all citizen records                    │
│                                                              │
│  ✅ CORRECT — Least privilege:                               │
│                                                              │
│  Enrollment officer role:                                    │
│  ├── CREATE enrollment ✅ (needed)                           │
│  ├── READ own enrollment status ✅ (needed)                  │
│  └── UPDATE incomplete enrollment ✅ (needed)               │
│                                                              │
│  → If officer account is compromised:                       │
│    attacker can only create enrollments                     │
│    → contained blast radius                                 │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 10.4 NBIS Role Definitions

### Role 1 — ENROLLMENT_OFFICER

```
┌──────────────────────────────────────────────────────────────┐
│  Role: ENROLLMENT_OFFICER                                    │
│  Assigned to: Front-line enrollment center staff            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PERMITTED:                                                  │
│  ✅ POST /v1/enrollments          Create new enrollment      │
│  ✅ GET  /v1/enrollments/{id}     View own submitted packet  │
│  ✅ PUT  /v1/enrollments/{id}     Update incomplete record   │
│  ✅ POST /v1/biometrics/capture   Trigger biometric capture  │
│  ✅ GET  /v1/enrollments/status   Check processing status    │
│                                                              │
│  NOT PERMITTED:                                              │
│  ❌ GET  /v1/citizens/{uin}       Cannot query CIDR          │
│  ❌ POST /v1/admin/revoke         Cannot revoke identity     │
│  ❌ GET  /v1/admin/audit-logs     Cannot see audit logs      │
│  ❌ GET  /v1/enrollments          Cannot list all enrollments│
│  ❌ POST /v1/ekyc                 Cannot call eKYC           │
│                                                              │
│  Scope: own-center only (cannot see other centers' records) │
│  MFA required: Yes (password + OTP at login)               │
│  Session timeout: 4 hours                                   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

### Role 2 — CENTER_SUPERVISOR

```
┌──────────────────────────────────────────────────────────────┐
│  Role: CENTER_SUPERVISOR                                     │
│  Assigned to: Registration center managers                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PERMITTED (all of ENROLLMENT_OFFICER plus):                 │
│  ✅ GET  /v1/enrollments          View all center enrollments│
│  ✅ POST /v1/enrollments/{id}/approve  Approve flagged case  │
│  ✅ POST /v1/enrollments/{id}/reject   Reject flagged case   │
│  ✅ POST /v1/biometrics/override  Override quality failure   │
│  ✅ GET  /v1/center/reports       View center statistics     │
│                                                              │
│  NOT PERMITTED:                                              │
│  ❌ GET  /v1/citizens/{uin}       Cannot query CIDR          │
│  ❌ POST /v1/admin/revoke         Cannot revoke identity     │
│  ❌ GET  /v1/admin/audit-logs     Cannot see audit logs      │
│                                                              │
│  Scope: own-center only                                      │
│  MFA required: Yes                                           │
│  Session timeout: 4 hours                                    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

### Role 3 — SYSTEM_ADMIN

```
┌──────────────────────────────────────────────────────────────┐
│  Role: SYSTEM_ADMIN                                          │
│  Assigned to: NID Authority technical staff                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PERMITTED:                                                  │
│  ✅ GET  /v1/citizens/{uin}       Query specific citizen     │
│  ✅ POST /v1/admin/revoke         Revoke identity           │
│  ✅ POST /v1/admin/suspend        Suspend identity          │
│  ✅ POST /v1/admin/reinstate      Reinstate identity        │
│  ✅ GET  /v1/admin/audit-logs     Read audit logs           │
│  ✅ POST /v1/admin/users          Create user accounts      │
│  ✅ PUT  /v1/admin/users/{id}     Manage roles              │
│  ✅ GET  /v1/admin/reports        System-wide reports       │
│  ✅ POST /v1/partners             Onboard relying parties   │
│                                                              │
│  NOT PERMITTED:                                              │
│  ❌ GET  /v1/citizens             Bulk citizen list export   │
│  ❌ DELETE /v1/admin/audit-logs   Cannot delete audit logs  │
│  ❌ GET  /v1/biometrics/templates Cannot access raw templates│
│                                                              │
│  MFA required: Yes (biometric + OTP — AAL3)                 │
│  Session timeout: 1 hour                                     │
│  Every action: individually audited with reason code        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

### Role 4 — AUDITOR

```
┌──────────────────────────────────────────────────────────────┐
│  Role: AUDITOR                                               │
│  Assigned to: Data Protection Authority / compliance staff  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PERMITTED:                                                  │
│  ✅ GET  /v1/admin/audit-logs     Read all audit entries     │
│  ✅ GET  /v1/admin/reports        Generate compliance reports│
│  ✅ GET  /v1/auth/history         View authentication history│
│  ✅ GET  /v1/partner/usage        View partner usage stats   │
│                                                              │
│  NOT PERMITTED:                                              │
│  ❌ GET  /v1/citizens/{uin}       Cannot view citizen data   │
│  ❌ POST (any endpoint)           Read-only — no writes      │
│  ❌ GET  /v1/biometrics/templates Cannot access templates    │
│  ❌ PUT / DELETE (any endpoint)   No modifications          │
│                                                              │
│  MFA required: Yes                                           │
│  Session timeout: 2 hours                                    │
│  Purpose: Read-only visibility into system behaviour        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

### Role 5 — RELYING_PARTY

```
┌──────────────────────────────────────────────────────────────┐
│  Role: RELYING_PARTY                                         │
│  Assigned to: Banks, hospitals, telecoms, govt portals      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PERMITTED:                                                  │
│  ✅ POST /v1/auth/biometric       1:1 biometric verify       │
│  ✅ POST /v1/auth/otp/send        Send OTP to citizen        │
│  ✅ POST /v1/auth/otp/verify      Verify citizen OTP         │
│  ✅ POST /v1/auth/demographic     Demographic match          │
│  ✅ POST /v1/ekyc                 eKYC (with citizen consent)│
│                                                              │
│  NOT PERMITTED:                                              │
│  ❌ GET  /v1/citizens/{uin}       Cannot query CIDR directly │
│  ❌ GET  /v1/auth/history         Cannot see auth history    │
│  ❌ POST /v1/enrollments          Cannot create enrollments  │
│  ❌ POST /v1/admin/*              No admin access            │
│  ❌ GET  /v1/biometrics/*         No biometric data access   │
│                                                              │
│  Scoped by:                                                  │
│  ├── Allowed auth modes (defined at onboarding)             │
│  ├── Rate limit (per usage plan)                            │
│  └── Daily quota (per usage plan)                           │
│                                                              │
│  Auth: API Key + OAuth2 client credentials JWT              │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

### Role 6 — RESIDENT (Citizen Self-Service)

```
┌──────────────────────────────────────────────────────────────┐
│  Role: RESIDENT                                              │
│  Assigned to: Enrolled citizens using the self-service portal│
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PERMITTED (own record only):                                │
│  ✅ GET  /v1/resident/profile     View own profile           │
│  ✅ PUT  /v1/resident/address     Update own address         │
│  ✅ PUT  /v1/resident/email       Update contact details     │
│  ✅ GET  /v1/resident/history     View own auth history      │
│  ✅ POST /v1/resident/bio/lock    Lock own biometrics        │
│  ✅ POST /v1/resident/bio/unlock  Unlock own biometrics      │
│  ✅ GET  /v1/credentials/mine     Download own credentials   │
│                                                              │
│  NOT PERMITTED:                                              │
│  ❌ GET  /v1/citizens/{uin}       Cannot see other citizens  │
│  ❌ PUT  /v1/resident/name        Name change needs officer  │
│  ❌ POST /v1/auth/*               Cannot call auth APIs      │
│  ❌ POST /v1/admin/*              No admin access            │
│                                                              │
│  Scope: strictly own UIN — server enforces this             │
│  MFA: OTP required for all write operations                 │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

### Role 7 — SERVICE_ACCOUNT

```
┌──────────────────────────────────────────────────────────────┐
│  Role: SERVICE_ACCOUNT                                       │
│  Assigned to: Internal microservices calling each other     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Design principle:                                           │
│  Each microservice has its OWN service account              │
│  with permissions for ONLY what it needs                    │
│                                                              │
│  Example — Auth Service account:                            │
│  ✅ DynamoDB:GetItem (identity records table)               │
│  ✅ S3:GetObject (biometric templates bucket)               │
│  ✅ KMS:Decrypt (biometric template key only)               │
│  ✅ ElastiCache:GetItem / SetItem (OTP cache)               │
│  ✅ CloudWatch:PutLogEvents (audit log write)               │
│  ❌ DynamoDB:Scan (no bulk access)                          │
│  ❌ S3:ListBucket (no bucket enumeration)                   │
│  ❌ KMS:Encrypt (no encryption rights, only decrypt)        │
│                                                              │
│  Auth: AWS IAM Role (no username/password)                  │
│  Rotation: Automatic (IAM role, no static credentials)      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 10.5 Permission Taxonomy

Every permission in the NBIS follows a structured naming convention:

```
┌──────────────────────────────────────────────────────────────┐
│              Permission Naming Convention                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Format: {resource}:{action}                                 │
│                                                              │
│  enrollment:create         Create a new enrollment packet   │
│  enrollment:read           Read enrollment status           │
│  enrollment:update         Update incomplete enrollment     │
│  enrollment:approve        Approve flagged enrollment       │
│  enrollment:reject         Reject flagged enrollment        │
│                                                              │
│  citizen:read              Read a citizen record by UIN     │
│  citizen:update            Update citizen demographics      │
│  citizen:revoke            Revoke a citizen's identity      │
│  citizen:suspend           Suspend a citizen's identity     │
│                                                              │
│  auth:biometric            Call biometric verify endpoint   │
│  auth:otp                  Call OTP send/verify endpoints   │
│  auth:demographic          Call demographic match endpoint  │
│                                                              │
│  ekyc:basic                eKYC with name + DOB only        │
│  ekyc:full                 eKYC with full attribute set     │
│                                                              │
│  audit:read                Read audit log entries           │
│  audit:export              Export audit log (restricted)    │
│                                                              │
│  admin:users               Manage user accounts             │
│  admin:roles               Manage role assignments          │
│  admin:partners            Manage relying party onboarding  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 10.6 Complete Role-Permission Matrix

```
┌──────────────────────────┬──────┬──────┬───────┬──────┬──────┬──────┐
│ Permission               │ ENRL │ SUPVR│ ADMIN │ AUDT │ RP   │ RSDT │
├──────────────────────────┼──────┼──────┼───────┼──────┼──────┼──────┤
│ enrollment:create        │  ✅  │  ✅  │  ✅   │  ❌  │  ❌  │  ❌  │
│ enrollment:read          │  ✅  │  ✅  │  ✅   │  ❌  │  ❌  │  ❌  │
│ enrollment:update        │  ✅  │  ✅  │  ✅   │  ❌  │  ❌  │  ❌  │
│ enrollment:approve       │  ❌  │  ✅  │  ✅   │  ❌  │  ❌  │  ❌  │
│ enrollment:reject        │  ❌  │  ✅  │  ✅   │  ❌  │  ❌  │  ❌  │
├──────────────────────────┼──────┼──────┼───────┼──────┼──────┼──────┤
│ citizen:read             │  ❌  │  ❌  │  ✅   │  ❌  │  ❌  │  ❌  │
│ citizen:update           │  ❌  │  ❌  │  ✅   │  ❌  │  ❌  │  ❌  │
│ citizen:revoke           │  ❌  │  ❌  │  ✅   │  ❌  │  ❌  │  ❌  │
│ citizen:suspend          │  ❌  │  ❌  │  ✅   │  ❌  │  ❌  │  ❌  │
├──────────────────────────┼──────┼──────┼───────┼──────┼──────┼──────┤
│ auth:biometric           │  ❌  │  ❌  │  ✅   │  ❌  │  ✅  │  ❌  │
│ auth:otp                 │  ❌  │  ❌  │  ✅   │  ❌  │  ✅  │  ❌  │
│ auth:demographic         │  ❌  │  ❌  │  ✅   │  ❌  │  ✅  │  ❌  │
├──────────────────────────┼──────┼──────┼───────┼──────┼──────┼──────┤
│ ekyc:basic               │  ❌  │  ❌  │  ✅   │  ❌  │  ✅  │  ❌  │
│ ekyc:full                │  ❌  │  ❌  │  ✅   │  ❌  │  ✅* │  ❌  │
├──────────────────────────┼──────┼──────┼───────┼──────┼──────┼──────┤
│ audit:read               │  ❌  │  ❌  │  ✅   │  ✅  │  ❌  │  ❌  │
│ audit:export             │  ❌  │  ❌  │  ❌   │  ✅* │  ❌  │  ❌  │
├──────────────────────────┼──────┼──────┼───────┼──────┼──────┼──────┤
│ admin:users              │  ❌  │  ❌  │  ✅   │  ❌  │  ❌  │  ❌  │
│ admin:roles              │  ❌  │  ❌  │  ✅   │  ❌  │  ❌  │  ❌  │
│ admin:partners           │  ❌  │  ❌  │  ✅   │  ❌  │  ❌  │  ❌  │
├──────────────────────────┼──────┼──────┼───────┼──────┼──────┼──────┤
│ resident:read-own        │  ❌  │  ❌  │  ✅   │  ❌  │  ❌  │  ✅  │
│ resident:update-own      │  ❌  │  ❌  │  ✅   │  ❌  │  ❌  │  ✅  │
│ resident:bio-lock        │  ❌  │  ❌  │  ✅   │  ❌  │  ❌  │  ✅  │
└──────────────────────────┴──────┴──────┴───────┴──────┴──────┴──────┘

* = requires additional approval or court order
ENRL  = Enrollment Officer     SUPVR = Center Supervisor
ADMIN = System Administrator   AUDT  = Auditor
RP    = Relying Party          RSDT  = Resident
```

---

## 10.7 RBAC Implementation — AWS

### AWS Cognito — Human Users

```
┌──────────────────────────────────────────────────────────────┐
│              Cognito RBAC Implementation                     │
└──────────────────────────────────────────────────────────────┘

1. Create Cognito User Pool (one per environment):
   ├── Production user pool: nbis-prod-users
   └── Staging user pool:    nbis-staging-users

2. Create Cognito Groups (one per role):
   ├── enrollment-officers
   ├── center-supervisors
   ├── system-admins
   ├── auditors
   └── residents

3. Assign users to groups:
   Hamza → enrollment-officers
   Sara  → system-admins

4. JWT contains group membership:
   {
     "sub": "hamza-uuid-123",
     "cognito:groups": ["enrollment-officers"],
     "custom:centerId": "CENTER-001",
     "exp": 1705316400
   }

5. Lambda Authorizer reads JWT claims:
   ├── Extract role from cognito:groups
   ├── Check permission matrix for this role + endpoint
   └── ALLOW or DENY
```

### AWS IAM — Service Accounts

```
┌──────────────────────────────────────────────────────────────┐
│              IAM Role Per Microservice                       │
└──────────────────────────────────────────────────────────────┘

Auth Service IAM Role (arn:aws:iam::123:role/nbis-auth-service):
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["dynamodb:GetItem"],
      "Resource": "arn:aws:dynamodb:*:*:table/nbis-identity-records"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::nbis-biometric-templates/*"
    },
    {
      "Effect": "Allow",
      "Action": ["kms:Decrypt"],
      "Resource": "arn:aws:kms:*:*:key/biometric-template-key-id"
    },
    {
      "Effect": "Allow",
      "Action": ["logs:PutLogEvents"],
      "Resource": "arn:aws:logs:*:*:log-group:/nbis/audit"
    }
  ]
}

Registration Service IAM Role — different policy:
├── DynamoDB:PutItem (enrollment records)
├── S3:PutObject (enrollment packets)
├── SQS:SendMessage (enrollment queue)
└── KMS:Encrypt (packet encryption)
→ Cannot access identity records table at all
```

---

## 10.8 Lambda Authorizer — Enforcement Point

The **Lambda Authorizer** is a Lambda function invoked by API Gateway before every request reaches a microservice. It is the central enforcement point for RBAC.

```
┌──────────────────────────────────────────────────────────────┐
│              Lambda Authorizer Logic                         │
└──────────────────────────────────────────────────────────────┘

function authorize(event):

  1. Extract JWT from Authorization header
     token = event.headers.Authorization.replace("Bearer ", "")

  2. Validate JWT signature (Cognito public key)
     if invalid → return DENY (401)

  3. Check JWT expiry
     if expired → return DENY (401)

  4. Extract claims
     role     = token.claims["cognito:groups"][0]
     centerId = token.claims["custom:centerId"]
     scope    = token.claims["scope"]

  5. Look up permission matrix
     method   = event.httpMethod        (POST)
     path     = event.path              (/v1/enrollments)
     required = permissionMatrix[path][method]

  6. Check role has required permission
     if role.permissions includes required → ALLOW
     else → return DENY (403)

  7. For RESIDENT role: enforce own-record-only
     if role == RESIDENT:
       uin_in_path = extract_uin(event.path)
       uin_in_token = token.claims["custom:uin"]
       if uin_in_path != uin_in_token → DENY (403)

  8. Return IAM policy
     {
       "principalId": token.sub,
       "policyDocument": {
         "Statement": [{
           "Effect": "Allow",  // or "Deny"
           "Action": "execute-api:Invoke",
           "Resource": event.methodArn
         }]
       },
       "context": {
         "role": role,
         "centerId": centerId,
         "userId": token.sub
       }
     }
```

---

## 10.9 Attribute-Based Access Control (ABAC) Extension

Pure RBAC sometimes is not fine-grained enough. NBIS extends RBAC with **ABAC (Attribute-Based Access Control)** for specific cases:

```
┌──────────────────────────────────────────────────────────────┐
│                    ABAC Extensions                           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Center-scoped access:                                       │
│  ENROLLMENT_OFFICER can only see enrollments                │
│  from their OWN center (centerId attribute)                 │
│                                                              │
│  Resource: enrollment:{centerId}:create                     │
│  Condition: token.centerId == enrollment.centerId           │
│                                                              │
│  ─────────────────────────────────────────────────────────  │
│                                                              │
│  Partner-scoped auth modes:                                 │
│  RELYING_PARTY Bank A → auth:biometric + auth:otp only     │
│  RELYING_PARTY NGO B  → auth:demographic only              │
│                                                              │
│  Condition: token.allowedModes includes requestedMode       │
│                                                              │
│  ─────────────────────────────────────────────────────────  │
│                                                              │
│  Time-based access:                                         │
│  ENROLLMENT_OFFICER → only 07:00–20:00 local time          │
│  Admin actions → require additional approval 22:00–06:00   │
│                                                              │
│  Condition: currentTime within allowedHours[role]          │
│                                                              │
│  ─────────────────────────────────────────────────────────  │
│                                                              │
│  Own-record enforcement (RESIDENT):                         │
│  RESIDENT can only access /v1/resident/{uin}                │
│  where {uin} matches the UIN in their JWT                   │
│                                                              │
│  Condition: pathParam.uin == token.uin                      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 10.10 Separation of Duties

**Separation of Duties (SoD)** ensures that no single person can complete a sensitive operation alone. Critical NBIS operations require multiple people.

```
┌──────────────────────────────────────────────────────────────┐
│            Separation of Duties in NBIS                     │
├───────────────────────┬──────────────────────────────────────┤
│ Operation             │ Required Parties                    │
├───────────────────────┼──────────────────────────────────────┤
│ Revoke a citizen      │ SYSTEM_ADMIN initiates              │
│ identity              │ + SENIOR_ADMIN approves             │
│                       │ (4-eyes principle)                  │
├───────────────────────┼──────────────────────────────────────┤
│ Export bulk audit     │ AUDITOR requests                    │
│ logs                  │ + LEGAL_OFFICER approves            │
│                       │ + Court order attached              │
├───────────────────────┼──────────────────────────────────────┤
│ Override biometric    │ CENTER_SUPERVISOR initiates         │
│ quality failure       │ + documented reason required        │
├───────────────────────┼──────────────────────────────────────┤
│ Onboard relying       │ PARTNER_ADMIN creates               │
│ party                 │ + SYSTEM_ADMIN activates            │
├───────────────────────┼──────────────────────────────────────┤
│ Deploy to production  │ DEV creates pull request            │
│                       │ + SENIOR_DEV approves               │
│                       │ + SECURITY_OFFICER approves         │
└───────────────────────┴──────────────────────────────────────┘
```

---

## 10.11 Role Lifecycle Management

Roles must be managed throughout the user's employment lifecycle:

```
┌──────────────────────────────────────────────────────────────┐
│               Role Lifecycle Events                          │
├───────────────────────┬──────────────────────────────────────┤
│ Event                 │ Action Required                     │
├───────────────────────┼──────────────────────────────────────┤
│ New employee joins    │ Create account                      │
│                       │ Assign minimum role for job         │
│                       │ Complete training first             │
├───────────────────────┼──────────────────────────────────────┤
│ Employee transfers    │ Remove old role immediately         │
│ department            │ Assign new role for new job         │
│                       │ Audit access gap period             │
├───────────────────────┼──────────────────────────────────────┤
│ Employee resigns      │ Disable account SAME DAY            │
│                       │ Revoke all sessions immediately     │
│                       │ Retain account 90 days for audit   │
│                       │ then delete                         │
├───────────────────────┼──────────────────────────────────────┤
│ Employee on leave     │ Disable account (not delete)        │
│                       │ Re-enable on return                 │
├───────────────────────┼──────────────────────────────────────┤
│ Suspected compromise  │ Disable IMMEDIATELY                 │
│                       │ Force password reset on return      │
│                       │ Review audit log for abuse          │
├───────────────────────┼──────────────────────────────────────┤
│ Quarterly review      │ Review all role assignments         │
│                       │ Remove excess permissions           │
│                       │ Document justification for each role│
└───────────────────────┴──────────────────────────────────────┘
```

---

## 10.12 Authorization Failure Handling

When authorization fails, the response must be consistent and non-informative — never reveal WHY access was denied beyond the HTTP status code.

```
┌──────────────────────────────────────────────────────────────┐
│             Authorization Failure Responses                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  401 Unauthorized — authentication failure:                  │
│  {                                                           │
│    "errorCode": "AUTH_REQUIRED",                             │
│    "message": "Authentication required",                     │
│    "traceId": "abc-123"                                      │
│  }                                                           │
│  → No detail about WHY auth failed                          │
│                                                              │
│  403 Forbidden — authorization failure:                      │
│  {                                                           │
│    "errorCode": "ACCESS_DENIED",                             │
│    "message": "You do not have permission for this action",  │
│    "traceId": "abc-123"                                      │
│  }                                                           │
│  → No detail about which permission is missing              │
│  → No detail about what roles would grant access            │
│                                                              │
│  WHY no detail?                                              │
│  → Telling an attacker "you need SYSTEM_ADMIN role"         │
│    helps them understand the permission model                │
│  → Generic messages reveal nothing useful to attackers      │
│    but are sufficient for legitimate developers             │
│    (who have documentation)                                 │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 10.13 Audit Logging for Authorization

Every authorization decision — allow or deny — must be logged:

```
┌──────────────────────────────────────────────────────────────┐
│           Authorization Audit Log Entry                      │
└──────────────────────────────────────────────────────────────┘
{
  "eventId":     "EVT-20250115-00456789",
  "eventType":   "AUTHORIZATION_DENIED",
  "timestamp":   "2025-01-15T14:22:00.456Z",
  "traceId":     "xyz-789-abc",
  "userId":      "hamza-uuid-123",
  "userRole":    "ENROLLMENT_OFFICER",
  "httpMethod":  "POST",
  "path":        "/v1/admin/revoke",
  "requiredPerm":"citizen:revoke",
  "outcome":     "DENIED",
  "sourceIp":    "10.0.1.45",
  "reason":      "INSUFFICIENT_ROLE"
}
```

**Why log denied requests?**
Multiple denied requests from the same user trying to access admin endpoints is a signal of either:
- Misconfigured role (legitimate — fix the role)
- Insider threat attempting privilege escalation (security incident)

GuardDuty anomaly detection watches for this pattern.

---

## 10.14 Common RBAC Mistakes in NBIS Projects

```
┌──────────────────────────────────────────────────────────────┐
│              Common RBAC Mistakes to Avoid                   │
├───────────────────────┬──────────────────────────────────────┤
│ Mistake               │ Consequence + Fix                   │
├───────────────────────┼──────────────────────────────────────┤
│ Too few roles         │ Everyone gets ADMIN because          │
│                       │ granular roles "take too long"       │
│                       │ Fix: Design roles before coding     │
├───────────────────────┼──────────────────────────────────────┤
│ Role proliferation    │ 50 roles that overlap and           │
│                       │ contradict each other               │
│                       │ Fix: Start with 6-8 core roles      │
├───────────────────────┼──────────────────────────────────────┤
│ Shared accounts       │ 5 officers sharing one login        │
│                       │ → Cannot audit individual actions   │
│                       │ Fix: One account per human, always  │
├───────────────────────┼──────────────────────────────────────┤
│ Enforcement in        │ Check role in front-end only        │
│ wrong layer           │ → Bypass by calling API directly    │
│                       │ Fix: Enforce at API Gateway +       │
│                       │ service layer (defence in depth)    │
├───────────────────────┼──────────────────────────────────────┤
│ Forgotten service     │ Old SI contractor account still     │
│ accounts              │ has SYSTEM_ADMIN 2 years later      │
│                       │ Fix: Quarterly access review        │
├───────────────────────┼──────────────────────────────────────┤
│ No role expiry        │ Employee leaves, account stays      │
│                       │ active indefinitely                 │
│                       │ Fix: HR system integration for      │
│                       │ auto-disable on offboarding         │
└───────────────────────┴──────────────────────────────────────┘
```

---

## 10.15 Defence in Depth — Three Enforcement Layers

RBAC must be enforced at **three layers** — not just one. An attacker who bypasses one layer must still face the next.

```
┌──────────────────────────────────────────────────────────────┐
│              Three Layers of Authorization                   │
└──────────────────────────────────────────────────────────────┘

Layer 1 — API Gateway (Lambda Authorizer):
  Checks JWT role + endpoint permission matrix
  → Blocks 99% of unauthorized access

  If bypassed (misconfigured route):
                    ↓

Layer 2 — Service Layer (application code):
  Auth Service re-checks: "does this caller have
  auth:biometric permission for this partnerId?"
  → Blocks access even if gateway missed it

  If bypassed (bug in service code):
                    ↓

Layer 3 — Data Layer (IAM policies):
  DynamoDB resource policy: Auth Service IAM role
  can only GetItem on identity-records table
  → Cannot do Scan, cannot access other tables

No single bug can bypass all three layers.
```

---

## 10.16 MOSIP Reference

```
┌──────────────────────────────────────────────────────────────┐
│              MOSIP Authorization Reference                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  MOSIP uses Spring Security for RBAC                        │
│  ├── @PreAuthorize annotations on service methods           │
│  ├── Role hierarchy configurable per deployment             │
│  └── Keycloak as identity and authorization server         │
│                                                              │
│  MOSIP roles (mapped to NBIS roles above):                  │
│  ├── REGISTRATION_OFFICER → ENROLLMENT_OFFICER              │
│  ├── REGISTRATION_SUPERVISOR → CENTER_SUPERVISOR            │
│  ├── ZONAL_ADMIN → SYSTEM_ADMIN (region-scoped)             │
│  ├── GLOBAL_ADMIN → SYSTEM_ADMIN (global)                   │
│  └── RESIDENT → RESIDENT                                    │
│                                                              │
│  Partner management:                                         │
│  ├── Partners (relying parties) register via portal         │
│  ├── Each partner gets a Partner ID + certificate           │
│  ├── Partner certificate used to sign auth requests         │
│  └── NBIS validates partner signature on every call         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 10.17 Key Terms

| Term | Definition |
|------|-----------|
| **Authorization** | Determining what an authenticated identity is permitted to do |
| **RBAC** | Role-Based Access Control — permissions attached to roles, not individuals |
| **Role** | A named collection of permissions representing a job function |
| **Permission** | A specific allowed action on a specific resource |
| **Least Privilege** | Grant only the minimum access needed to perform a function |
| **Lambda Authorizer** | AWS Lambda function invoked by API Gateway to enforce RBAC |
| **IAM Role** | AWS identity for services — grants permissions to AWS resources |
| **ABAC** | Attribute-Based Access Control — permissions based on attributes |
| **Separation of Duties** | No single person can complete a sensitive operation alone |
| **4-Eyes Principle** | Two people must approve a sensitive action |
| **Defence in Depth** | RBAC enforced at multiple layers — gateway, service, data |
| **Service Account** | Machine identity for a microservice — IAM role in AWS |
| **Scope** | OAuth2 concept — specific permissions granted to a token |
| **403 Forbidden** | HTTP status for authorization failure (authenticated but not permitted) |
| **401 Unauthorized** | HTTP status for authentication failure (not authenticated) |
| **Role Proliferation** | Too many overlapping roles — a design anti-pattern |
| **Access Review** | Periodic audit of who has what roles — quarterly minimum |

---

## 10.18 Key Takeaways

- **Authorization always follows authentication** — you cannot authorize an unknown identity. 401 means not authenticated; 403 means authenticated but not authorized.
- **RBAC attaches permissions to roles, not users** — one role definition governs thousands of users. Add a permission once; all role members get it.
- **Least privilege is non-negotiable** — every role, every service account, every IAM policy must start from zero and add only what is needed.
- **Service accounts are roles too** — each microservice has its own IAM role with only the DynamoDB tables, S3 buckets, and KMS keys it actually needs.
- **Enforce at three layers** — API Gateway (Lambda Authorizer), service code, and IAM data-layer policies. One layer is a single point of failure.
- **Separation of Duties prevents insider abuse** — no single person should be able to revoke an identity, export audit logs, or deploy to production alone.
- **Never reveal RBAC details in error messages** — 403 responses say "access denied", not "you need SYSTEM_ADMIN role". Attackers use permission model knowledge to escalate.
- **Quarterly access reviews are mandatory** — employees leave, transfer, and get promoted. Old permissions accumulate silently. Review and remove regularly.

---

## 10.19 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 11 | Event-Driven Architecture — SQS patterns, async flows, idempotency |
| Chapter 12 | Message Queues — SQS standard vs FIFO, DLQ, visibility timeout |
| Chapter 62 | Secrets Management — where to store API keys, DB passwords, certs |

---

*Chapter 10 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*
