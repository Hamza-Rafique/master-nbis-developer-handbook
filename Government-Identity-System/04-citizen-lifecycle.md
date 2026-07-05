# Chapter 4 — Citizen Lifecycle

> **Part I — Government Identity Systems**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 4.1 What is the Citizen Lifecycle?

The **Citizen Lifecycle** is the complete journey of a person's identity through a government system — from the moment they are born (or immigrate) to the moment their record is archived after death.

Every event in a person's life that changes their legal status, location, name, or biometrics triggers an update in the identity ecosystem. The NBIS must handle all of these transitions correctly, maintaining data integrity and audit history at every step.

```
┌─────────────────────────────────────────────────────────────────┐
│                    The Citizen Lifecycle                        │
│                                                                 │
│   Birth ──► Registration ──► Enrollment ──► Credential         │
│                                                 │               │
│                                            Active Life          │
│                                         (updates, renewals)     │
│                                                 │               │
│                              ┌──────────────────┤              │
│                              │                  │              │
│                           Death           Emigration /         │
│                              │            Renunciation         │
│                              ▼                  ▼              │
│                           Revoked ──────────► Archived         │
└─────────────────────────────────────────────────────────────────┘
```

Understanding this lifecycle is essential for designing the data model, the workflow engine, and the audit logging strategy of any NBIS.

---

## 4.2 Stage 1 — Birth and Civil Registration

Everything begins with a **birth event** recorded in the Civil Registry.

```
Baby is born
      │
      ▼
Hospital / midwife notifies Civil Registry
      │
      ▼
Civil Registry creates birth record:
  - Full name
  - Date and place of birth
  - Parents' details
  - Nationality (derived from parents)
      │
      ▼
Birth Certificate issued
(legal proof of existence)
```

### Why this matters for NBIS

The birth certificate is the **entry ticket** to the national ID system. A person cannot be enrolled in the NBIS without first existing in the Civil Registry. In countries where birth registration rates are low, NBIS enrollment is blocked for entire populations.

### The registration gap

In many developing countries deploying NBIS for the first time, a large portion of the adult population was never registered at birth. MOSIP and other platforms address this through a **late registration** workflow — adults can submit alternative evidence (school records, community affidavits, hospital records) to establish their legal existence before biometric enrollment.

---

## 4.3 Stage 2 — Identity Enrollment

Once a legal existence is established, the person attends a **registration center** for biometric enrollment.

```
┌──────────────────────────────────────────────────────────────┐
│                   Enrollment Workflow                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Appointment booking (pre-registration portal)           │
│           │                                                  │
│  2. Arrival at registration center                          │
│           │                                                  │
│  3. Document verification (birth cert, supporting docs)     │
│           │                                                  │
│  4. Demographic capture (name, DOB, address, gender)        │
│           │                                                  │
│  5. Biometric capture                                        │
│       ├── 10 fingerprints (rolled + slap)                   │
│       ├── Iris scan (left + right)                          │
│       └── Face photograph                                   │
│           │                                                  │
│  6. Quality check (is the data good enough?)                │
│           │                                                  │
│  7. Operator review + submission                            │
│           │                                                  │
│  8. Packet uploaded to Registration Processor               │
│           │                                                  │
│  9. Deduplication (1:N ABIS search)                         │
│           │                                                  │
│  10. UIN assigned + credential issued                       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### MOSIP reference

In MOSIP, steps 1–7 are handled by the **Registration Client** (a Java desktop application running at the enrollment center). Steps 8–10 are handled by the **Registration Processor** — a back-end pipeline of microservices that runs asynchronously after the enrollment packet is uploaded.

---

## 4.4 Stage 3 — Credential Issuance

After deduplication passes and a UIN is assigned, a **credential** is issued to the person.

```
UIN Assigned
      │
      ├──► Physical smart card printed + mailed / collected
      │
      ├──► Digital credential (W3C VC) issued to mobile wallet
      │
      └──► QR-based credential for offline verification
```

### Credential types by use case

```
┌─────────────────────┬──────────────────────────────────────────┐
│ Credential Type     │ Use Case                                 │
├─────────────────────┼──────────────────────────────────────────┤
│ Smart card          │ Daily identification, service access     │
│ ePassport           │ International travel                     │
│ Mobile ID (Inji VC) │ Digital services, offline verification  │
│ QR code             │ Low-tech environments, quick scan        │
│ OIDC token          │ Web and mobile app authentication        │
└─────────────────────┴──────────────────────────────────────────┘
```

---

## 4.5 Stage 4 — Active Lifecycle Events

Once enrolled and active, a citizen's identity record undergoes many changes throughout their life. Each change is a **lifecycle event** that must be captured, verified, and audited.

```
┌──────────────────────────────────────────────────────────────┐
│               Active Lifecycle Events                        │
├──────────────────────┬───────────────────────────────────────┤
│ Event                │ What changes in the NBIS              │
├──────────────────────┼───────────────────────────────────────┤
│ Address change       │ Demographic record updated            │
│ Name change          │ Legal name updated (court order req.) │
│ Marriage             │ Civil status updated, name change opt │
│ Divorce              │ Civil status updated                  │
│ Biometric refresh    │ Templates re-captured (aging, injury) │
│ Card renewal         │ New credential issued, old revoked    │
│ Lost card            │ Old credential revoked, new issued    │
│ Nationality change   │ Record type updated                   │
│ Residency expiry     │ Resident record suspended             │
│ Residency renewal    │ Resident record reactivated           │
└──────────────────────┴───────────────────────────────────────┘
```

### Resident Services Portal

In MOSIP, citizens manage their own lifecycle events through the **Resident Services Portal** — a self-service web application where a person can:

- View their identity data
- Request demographic updates
- Lock / unlock biometric authentication
- View their authentication history (who queried their identity and when)
- Download their Verifiable Credential

This is the **transparency layer** — a citizen can always see exactly how their identity data has been used.

---

## 4.6 Stage 5 — Suspension

A citizen's identity can be **temporarily suspended** — the record exists but cannot be used for authentication or verification.

```
Reasons for suspension:
      │
      ├──► Legal hold (court order, ongoing investigation)
      ├──► Suspected fraud (pending review)
      ├──► Expired residency permit (resident records)
      └──► Administrative error (pending correction)

While suspended:
      │
      ├──► Authentication attempts → rejected
      ├──► Credential → cannot be used
      └──► Record → visible to authorized admin only

Reactivation:
      │
      └──► Legal clearance / correction → back to Active
```

### Why suspension ≠ revocation

Suspension is **reversible**. Revocation is **permanent**. A suspended record can be reinstated; a revoked record cannot. The data model must distinguish between these two states explicitly.

---

## 4.7 Stage 6 — Biometric Refresh

Biometrics change over time. Fingerprints degrade with age and manual labor. A person injured in an accident may lose fingers. A child enrolled at age 6 will have different biometrics at age 18.

```
Triggers for biometric refresh:
      │
      ├──► Age milestone (child → adult)
      ├──► Physical injury or disability
      ├──► Biometric quality degradation
      ├──► Technology upgrade (new sensor standard)
      └──► Periodic national refresh program

Process:
      │
      ▼
Citizen attends registration center
      │
      ▼
New biometrics captured
      │
      ▼
New deduplication run (1:N search)
      │
      ▼
Old templates replaced in CIDR
      │
      ▼
Audit record created (old templates archived)
```

---

## 4.8 Stage 7 — Death and Revocation

When a citizen dies, their identity record must be **revoked** promptly to prevent posthumous fraud — using a dead person's credentials to claim benefits, cross borders, or open bank accounts.

```
Death event
      │
      ▼
Civil Registry notified (hospital / family)
      │
      ▼
Civil Registry issues death certificate
      │
      ▼
NBIS notified via Civil Registry integration
      │
      ▼
Identity record status → REVOKED
      │
      ├──► All credentials invalidated immediately
      ├──► All active authentication sessions terminated
      ├──► Biometric authentication rejected from this point
      └──► Record locked — no further updates possible
      │
      ▼
Record moved to ARCHIVE after legal retention period
```

### The integration challenge

In many countries, the Civil Registry and NBIS are run by different ministries and do not share a real-time integration. This creates a window where a deceased person's credentials remain valid. Well-designed systems build an **automated event bridge** — when a death certificate is issued in the Civil Registry, an event is published that triggers the NBIS revocation workflow automatically.

```
Civil Registry (death cert issued)
      │
      ▼ publishes event
    SQS queue
      │
      ▼ triggers
    Lambda function
      │
      ▼ calls
    NBIS Revocation API
      │
      ▼
    Record revoked within seconds
```

---

## 4.9 Stage 8 — Archival

After revocation, records are not deleted — they are **archived**. Legal, forensic, and historical reasons require that identity records be retained for a defined period after a person's death or departure.

```
┌──────────────────────────────────────────────────────────┐
│                    Archival Policy                       │
├──────────────────────┬───────────────────────────────────┤
│ Retention period     │ Typically 10–30 years post-death  │
│ Access              │ Court order / law enforcement only │
│ Biometric data       │ May be purged earlier (privacy)   │
│ Demographic data     │ Retained for genealogy / audit    │
│ Audit logs           │ Retained permanently              │
└──────────────────────┴───────────────────────────────────┘
```

### AWS implementation

```
Active record     → DynamoDB (hot storage, fast access)
      │
      ▼ on revocation
Archived record   → S3 Glacier (cold storage, low cost)
      │
Audit logs        → CloudWatch Logs with Object Lock
                    (immutable, cannot be deleted)
```

---

## 4.10 Resident Lifecycle vs Citizen Lifecycle

Residents follow the same lifecycle stages but with additional **permit-driven transitions** that citizens do not have.

```
┌──────────────────────────────────────────────────────────────┐
│           Citizen vs Resident Lifecycle Differences          │
├───────────────────────┬──────────────────────────────────────┤
│ Stage                 │ Citizen          │ Resident          │
├───────────────────────┼──────────────────┼───────────────────┤
│ Entry trigger         │ Birth / naturali-│ Visa / permit     │
│                       │ zation           │ issuance          │
│ Identity duration     │ Permanent        │ Tied to permit    │
│ Expiry                │ None (identity)  │ With permit       │
│ Suspension trigger    │ Legal / fraud    │ Permit expiry     │
│ Renewal               │ Credential only  │ Permit + credent. │
│ Revocation trigger    │ Death / renounc. │ Deportation /     │
│                       │                  │ permit cancel     │
│ Naturalization path   │ Already citizen  │ Can become        │
│                       │                  │ citizen           │
└───────────────────────┴──────────────────┴───────────────────┘
```

---

## 4.11 Complete Lifecycle State Machine

```
                    ┌────────────────────┐
                    │    UNREGISTERED    │
                    │  (born / arrived)  │
                    └─────────┬──────────┘
                              │ civil registration
                              ▼
                    ┌────────────────────┐
                    │    REGISTERED      │
                    │  (civil record     │
                    │   exists)          │
                    └─────────┬──────────┘
                              │ biometric enrollment
                              ▼
                    ┌────────────────────┐
                    │    ENROLLED        │
                    │  (biometrics       │
                    │   captured)        │
                    └─────────┬──────────┘
                              │ dedup passed + UIN assigned
                              ▼
                    ┌────────────────────┐         ┌──────────────┐
                    │      ACTIVE        │◄────────│   UPDATED    │
                    │  (credential       │         │  (name, addr,│
                    │   issued)          │─────────►  biometric)  │
                    └──────┬─────┬───────┘         └──────────────┘
                           │     │
              legal hold   │     │ death /
              / fraud      │     │ renunciation
                           ▼     ▼
                    ┌──────────┐ ┌──────────────┐
                    │SUSPENDED │ │   REVOKED    │
                    │(reversib.)│ │ (permanent)  │
                    └──────┬───┘ └──────┬───────┘
                           │            │
                    cleared │            │ retention
                           │            │ period ends
                           ▼            ▼
                         ACTIVE    ┌──────────────┐
                                   │   ARCHIVED   │
                                   │ (read-only)  │
                                   └──────────────┘
```

---

## 4.12 MOSIP Reference

| Lifecycle Stage | MOSIP Module |
|----------------|-------------|
| Pre-enrollment booking | Pre-registration portal |
| Biometric enrollment | Registration Client |
| Deduplication + UIN assignment | Registration Processor + ABIS |
| Credential issuance | Credential Service |
| Demographic updates | Resident Services portal |
| Biometric lock / unlock | Resident Services portal |
| Authentication history | Resident Services portal |
| Death / revocation | Admin Services + Civil Registry integration |
| Archival | Configurable retention policy |

---

## 4.13 AWS Mapping

```
┌───────────────────────────────────┬──────────────────────────────┐
│ Lifecycle Stage                   │ AWS Services                 │
├───────────────────────────────────┼──────────────────────────────┤
│ Birth / civil reg. integration    │ API Gateway + Lambda         │
│ Enrollment packet upload          │ S3 + SQS                     │
│ Deduplication pipeline            │ SQS → Lambda → ABIS API      │
│ UIN assignment + record creation  │ Lambda → DynamoDB PutItem    │
│ Credential issuance               │ Lambda → S3 (signed VC/QR)  │
│ Demographic update                │ API Gateway → Lambda →       │
│                                   │ DynamoDB UpdateItem          │
│ Suspension                        │ DynamoDB attribute update    │
│ Revocation (death event)          │ SQS → Lambda → DynamoDB     │
│ Archival                          │ DynamoDB → S3 Glacier        │
│ Audit at every stage              │ CloudWatch Logs (immutable)  │
└───────────────────────────────────┴──────────────────────────────┘
```

---

## 4.14 Key Terms

| Term | Definition |
|------|-----------|
| **Citizen Lifecycle** | The complete journey of an identity from birth to archival |
| **Civil Registration** | Recording a birth event in the civil registry — the entry point |
| **Late Registration** | Enrollment of adults who were never registered at birth |
| **Registration Center** | Physical location where biometric enrollment takes place |
| **Biometric Refresh** | Re-capture of biometrics due to aging, injury, or technology upgrade |
| **Suspension** | Temporary deactivation of an identity — reversible |
| **Revocation** | Permanent deactivation of an identity — not reversible |
| **Archival** | Long-term retention of a revoked record for legal and audit purposes |
| **Resident Services** | MOSIP's self-service portal for citizens to manage their own identity |
| **Death Notification** | Civil Registry event that triggers NBIS revocation automatically |
| **State Machine** | A model of an identity record's possible states and transitions |
| **Naturalization** | The process by which a resident becomes a citizen |
| **Posthumous fraud** | Using a deceased person's credentials — prevented by prompt revocation |
| **Object Lock** | AWS S3 feature making audit logs immutable (cannot be modified/deleted) |
| **S3 Glacier** | AWS cold storage service — used for archived identity records |

---

## 4.15 Key Takeaways

- The citizen lifecycle is the **backbone of the NBIS data model** — every table, every API, and every workflow maps to a lifecycle stage.
- **Civil registration is the prerequisite** — without a birth record, there is no enrollment. Solving identity exclusion starts with civil registration.
- **Suspension and revocation are different** — one is reversible, one is not. The data model must distinguish them explicitly with different status codes.
- **Death revocation must be automated** — a manual process creates a window for posthumous fraud. Civil Registry → SQS → Lambda → NBIS revocation is the correct pattern.
- **Biometric data changes over time** — the system must support biometric refresh without changing the UIN.
- **Archived records are never deleted** — audit trails, legal compliance, and forensic investigation require long-term retention. Use S3 Glacier for cold storage.
- **Residents have additional lifecycle triggers** — permit expiry, renewal, and potential naturalization create states that citizen records do not have.

---

## 4.16 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 5 | Stakeholders — who builds, funds, operates, and uses the NBIS |
| Chapter 6 | Overall NBIS Architecture — full technical system diagram |
| Chapter 17 | Enrollment Workflow — deep dive into the enrollment pipeline |

---

*Chapter 4 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*