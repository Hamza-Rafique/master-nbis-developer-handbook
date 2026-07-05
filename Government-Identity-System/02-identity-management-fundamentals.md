# Chapter 2 — Identity Management Fundamentals

> **Part I — Government Identity Systems**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 2.1 What is Identity?

In the physical world, your identity is the answer to: **"Who are you?"**

In a digital system, identity is a structured set of attributes that uniquely represent a person and allow them to be distinguished from every other person in the registry.

```
┌─────────────────────────────────────────────────────────┐
│                  What Makes an Identity                 │
├─────────────────────┬───────────────────────────────────┤
│ Biographical        │ Name, date of birth, gender,      │
│                     │ nationality, address              │
├─────────────────────┼───────────────────────────────────┤
│ Biometric           │ Fingerprints, iris, face geometry │
├─────────────────────┼───────────────────────────────────┤
│ Identifier          │ UIN, passport number, tax ID      │
├─────────────────────┼───────────────────────────────────┤
│ Credential          │ Smart card, mobile ID, token      │
└─────────────────────┴───────────────────────────────────┘
```

The identity itself is not the credential. The **credential** is the artifact (card, token, QR code) that carries or references the identity. You can replace a lost card — you cannot replace the identity behind it.

---

## 2.2 The Unique Identity Number (UIN)

The **UIN** (also called NIN, National ID Number, or UID depending on the country) is the permanent, non-repeating number assigned to a person at the moment their identity is created in the NBIS.

Properties of a well-designed UIN:

```
┌──────────────────────────────────────────────────────────┐
│                    UIN Properties                        │
├──────────────────┬───────────────────────────────────────┤
│ Unique           │ No two people share the same UIN      │
│ Permanent        │ Never changes for the person's life   │
│ Non-sequential   │ Cannot be guessed or enumerated       │
│ Non-meaningful   │ Encodes no personal data (not DOB etc)│
│ Verifiable       │ Has a check digit for typo detection  │
└──────────────────┴───────────────────────────────────────┘
```

In Aadhaar: the UIN is a 12-digit randomly generated number.
In MOSIP: the UIN generation is a configurable module — the format is defined by the adopting country.

---

## 2.3 Identity Lifecycle

Every identity in the system passes through predictable states from creation to retirement.

```
          Birth / Immigration
                 │
                 ▼
          ┌─────────────┐
          │ REGISTERED  │  ← biographic data captured
          └──────┬──────┘
                 │
                 ▼
          ┌─────────────┐
          │  ENROLLED   │  ← biometrics captured + dedup passed
          └──────┬──────┘
                 │
                 ▼
          ┌─────────────┐
          │   ACTIVE    │  ← UIN assigned, credential issued
          └──────┬──────┘
                 │
        ┌────────┴────────┐
        │                 │
        ▼                 ▼
  ┌──────────┐      ┌───────────┐
  │ SUSPENDED│      │  UPDATED  │  ← address, name, biometric refresh
  │ (legal   │      └─────┬─────┘
  │  hold)   │            │
  └──────────┘            ▼
                    ┌───────────┐
                    │  ACTIVE   │  (back to active after update)
                    └─────┬─────┘
                          │
                          ▼
                    ┌───────────┐
                    │  REVOKED  │  ← death, renunciation, fraud
                    └─────┬─────┘
                          │
                          ▼
                    ┌───────────┐
                    │  ARCHIVED │  ← retained for legal / audit purposes
                    └───────────┘
```

---

## 2.4 Identity Proofing

**Identity proofing** is the process of establishing that a person is real, unique, and is who they claim to be — before issuing them an identity credential.

It answers the question: **"Is this a legitimate, real, unique person?"**

### NIST SP 800-63A — Identity Assurance Levels

The global standard for identity proofing is NIST Special Publication 800-63A, which defines three Identity Assurance Levels (IAL):

```
┌──────┬────────────────┬──────────────────────────────────────────┐
│ IAL  │ Name           │ What it means                            │
├──────┼────────────────┼──────────────────────────────────────────┤
│ IAL1 │ Self-asserted  │ User provides info — no verification.    │
│      │                │ Example: creating an email account.      │
├──────┼────────────────┼──────────────────────────────────────────┤
│ IAL2 │ Remote proofed │ Documents checked remotely (photo, scan).│
│      │                │ Example: online bank account opening.    │
├──────┼────────────────┼──────────────────────────────────────────┤
│ IAL3 │ In-person      │ Physical presence + biometrics +         │
│      │ proofed        │ document verification by trained staff.  │
│      │                │ Example: national ID enrollment.         │
└──────┴────────────────┴──────────────────────────────────────────┘
```

**National ID systems operate at IAL3.** This is why enrollment requires physically attending a registration center — there is no remote shortcut to a government identity.

---

## 2.5 Authentication vs Authorization

These two concepts are the most commonly confused in identity systems. They are fundamentally different operations.

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│   AUTHENTICATION                 AUTHORIZATION               │
│   "Are you who you say?"         "What are you allowed?"     │
│                                                              │
│   Verifies identity              Grants or denies access     │
│                                                              │
│   Happens FIRST                  Happens AFTER auth          │
│                                                              │
│   Example:                       Example:                    │
│   Fingerprint at border gate     Check if visa is valid      │
│                                                              │
│   AWS: Cognito                   AWS: IAM policies           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**The analogy:** Authentication opens the door to the building. Authorization decides which rooms you can enter once inside.

In NBIS:
- A citizen presenting their fingerprint at a service kiosk = **Authentication**
- The system checking whether they are eligible for a subsidy = **Authorization**

---

## 2.6 Verification vs Identification

Both use biometrics. Both produce a match decision. They are architecturally different.

```
VERIFICATION (1:1)                    IDENTIFICATION (1:N)
──────────────────                    ────────────────────
"Is this Hamza?"                      "Who is this person?"

Subject makes a CLAIM first.          No claim is made.

System fetches ONE record             System searches ALL N records
and compares probe against it.        and finds the best match.

Fast — O(1)                           Slow — O(N)
Used at: eGates, ATMs, portals        Used at: enrollment dedup, borders

AWS: Lambda + DynamoDB GetItem        AWS: Lambda + ABIS API call
```

### Why this matters in NBIS

- **Enrollment deduplication** is always **1:N identification** — because at enrollment time, we do not know whether this person already exists in the registry.
- **Every service transaction** (bank, hospital, government portal) is **1:1 verification** — the citizen presents their ID number, and the system verifies their biometric against that one record.

---

## 2.7 Deduplication

**Deduplication** is the mechanism that enforces the core promise of any NBIS: **one person, one identity**.

Without deduplication, a person could enroll five times under five different names and receive five different UINs — defeating the entire purpose of the system.

### How it works

```
New enrollment biometric captured
             │
             ▼
    Sent to ABIS (1:N search)
             │
    Compare probe against
    every template in the DB
             │
        ┌────┴────┐
        │         │
   Match found   No match found
   above         below threshold
   threshold         │
        │             ▼
        ▼        NEW identity created
   FLAG for       UIN assigned
   manual review
```

### Key numbers

| Parameter | Meaning | Typical target |
|-----------|---------|----------------|
| **FAR** | False Accept Rate — impostor accepted as genuine | < 0.01% |
| **FRR** | False Reject Rate — genuine person rejected | < 1% |
| **EER** | Equal Error Rate — where FAR = FRR (system benchmark) | < 0.1% |
| **Threshold** | The score above which two templates are considered a match | Configurable per deployment |

---

## 2.8 Identity Assurance vs Authentication Assurance

A common confusion in government identity projects:

```
┌──────────────────────────────────────────────────────────────┐
│ Identity Assurance Level (IAL)                               │
│ → How confident are we that this is a REAL, UNIQUE person?  │
│ → Set at ENROLLMENT time                                     │
│ → Cannot be changed after issuance without re-enrollment    │
├──────────────────────────────────────────────────────────────┤
│ Authentication Assurance Level (AAL)                         │
│ → How confident are we that the SAME person is present now? │
│ → Set at AUTHENTICATION time (each transaction)             │
│ → Can vary per transaction (low-risk vs high-risk service)  │
└──────────────────────────────────────────────────────────────┘
```

Example: A citizen enrolled at IAL3 (in-person, biometric). When accessing a low-risk portal, AAL1 (OTP) is sufficient. When accessing a high-value benefit or crossing a border, AAL3 (biometric) is required.

---

## 2.9 MOSIP Reference

| Concept | MOSIP Implementation |
|---------|---------------------|
| UIN generation | `id-repository` module — configurable UIN format and length |
| Identity proofing (IAL3) | Registration Client + Registration Processor pipeline |
| Authentication (1:1) | `id-authentication` module — OTP, biometric, demographic modes |
| Deduplication (1:N) | Registration Processor → ABIS middleware queue |
| Lifecycle management | Resident Services portal — self-service updates |
| Credential issuance | Credential Service — smart card, QR, W3C VC |

---

## 2.10 AWS Mapping

```
┌──────────────────────┬───────────────────────────────────────┐
│ Concept              │ AWS Service                           │
├──────────────────────┼───────────────────────────────────────┤
│ Authentication       │ Amazon Cognito (user pools)           │
│ Authorization (RBAC) │ AWS IAM (policies and roles)          │
│ UIN storage          │ Amazon DynamoDB (citizen registry)    │
│ 1:1 verification     │ Lambda → DynamoDB GetItem             │
│ 1:N deduplication    │ Lambda → ABIS API (external call)     │
│ Identity lifecycle   │ DynamoDB item attributes + TTL        │
│ Audit log            │ CloudWatch Logs (immutable)           │
└──────────────────────┴───────────────────────────────────────┘
```

---

## 2.11 Key Terms

| Term | Definition |
|------|-----------|
| **Identity** | The complete set of attributes that uniquely represent a person |
| **Credential** | The artifact (card, token, QR) that carries or references an identity |
| **UIN** | Unique Identity Number — permanent, non-meaningful, non-sequential |
| **Identity Lifecycle** | The states an identity passes through: registered → enrolled → active → revoked → archived |
| **Identity Proofing** | Establishing that a person is real and unique before issuing a credential |
| **IAL1 / IAL2 / IAL3** | Identity Assurance Levels — IAL3 is the standard for national ID |
| **Authentication** | Verifying that a person is who they claim to be (fingerprint, OTP) |
| **Authorization** | Determining what an authenticated person is allowed to do |
| **Verification (1:1)** | Comparing a biometric probe against one claimed record |
| **Identification (1:N)** | Searching all records to find who an unknown person is |
| **Deduplication** | Ensuring one person = one record by 1:N search at enrollment |
| **FAR** | False Accept Rate — impostors incorrectly matched |
| **FRR** | False Reject Rate — genuine users incorrectly rejected |
| **EER** | Equal Error Rate — benchmark where FAR = FRR |
| **AAL** | Authentication Assurance Level — confidence in the current authentication event |

---

## 2.12 Key Takeaways

- **Identity ≠ credential.** The identity is the record in the registry. The credential (card, QR, token) is just the artifact that references it.
- **UINs must be non-meaningful** — encoding DOB or location into the number creates a privacy risk and creates update problems when data changes.
- **National ID enrollment is always IAL3** — in-person, biometric, document-verified. There is no shortcut.
- **Authentication and Authorization are sequential, not synonymous** — you cannot authorize without first authenticating.
- **Verification (1:1) is fast and cheap. Identification (1:N) is slow and expensive.** Design your system to use verification everywhere except where identification is genuinely required (enrollment dedup, border unknowns).
- **Deduplication is the integrity guarantee** of the entire system. If it fails, the foundational promise breaks.

---

## 2.13 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 3 | Government Identity Ecosystem — civil registry, population registry, passports, eKYC, smart cards |
| Chapter 4 | Citizen Lifecycle — birth registration through death, the full journey |
| Chapter 6 | Overall NBIS Architecture — full system diagram, all components |

---

*Chapter 2 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*
