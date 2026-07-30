# Chapter 25 — Enrollment Validation

> **Part III — Enrollment**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 25.1 What is Enrollment Validation?

**Enrollment validation** is the comprehensive process of verifying that an enrollment packet is complete, structurally correct, legally sufficient, and internally consistent — before the packet proceeds to deduplication and UIN assignment.

While **quality checks** (Chapter 24) ask "Is this data good enough?", enrollment validation asks: **"Is this enrollment legally and technically complete enough to create a national identity?"**

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Checks vs Enrollment Validation         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  QUALITY CHECKS (Chapter 24):                                │
│  ├── Is the fingerprint image sharp enough?                 │
│  ├── Is the face in ICAO-compliant pose?                   │
│  └── Is the document scan readable?                        │
│  → Answers: "Is the DATA GOOD ENOUGH?"                     │
│                                                              │
│  ENROLLMENT VALIDATION (Chapter 25):                         │
│  ├── Are all required fields present?                       │
│  ├── Does this person meet eligibility criteria?           │
│  ├── Is this a valid enrollment process type?              │
│  ├── Are all mandatory biometrics captured (or excepted)?  │
│  ├── Is the operator authorized for this center?           │
│  └── Does the packet pass all business rules?             │
│  → Answers: "Is the ENROLLMENT LEGALLY COMPLETE?"          │
│                                                              │
│  BOTH are required before deduplication proceeds.           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 25.2 Validation Layers

Enrollment validation is structured across five distinct layers, each catching different categories of errors:

```
┌──────────────────────────────────────────────────────────────┐
│              Five Validation Layers                          │
├─────────┬────────────────────────────────────────────────────┤
│ Layer   │ What it checks                                    │
├─────────┼────────────────────────────────────────────────────┤
│ Layer 1 │ STRUCTURAL VALIDATION                             │
│         │ Is the packet format correct?                     │
│         │ Can it be parsed at all?                         │
├─────────┼────────────────────────────────────────────────────┤
│ Layer 2 │ COMPLETENESS VALIDATION                           │
│         │ Are all required fields and modalities present?  │
├─────────┼────────────────────────────────────────────────────┤
│ Layer 3 │ INTEGRITY VALIDATION                              │
│         │ Is the packet authentic and untampered?          │
├─────────┼────────────────────────────────────────────────────┤
│ Layer 4 │ BUSINESS RULE VALIDATION                          │
│         │ Does the enrollment meet all policy rules?       │
├─────────┼────────────────────────────────────────────────────┤
│ Layer 5 │ ELIGIBILITY VALIDATION                            │
│         │ Is this person eligible to be enrolled?          │
└─────────┴────────────────────────────────────────────────────┘
```

---

## 25.3 Layer 1 — Structural Validation

```
┌──────────────────────────────────────────────────────────────┐
│              Layer 1: Structural Validation                  │
└──────────────────────────────────────────────────────────────┘

PURPOSE:
  Verify the packet can be parsed and conforms
  to the expected schema before any deeper checks.

CHECKS:

  CHECK 1.1: Packet decryption
  ├── Can the packet be decrypted using center key?
  ├── Decryption failure → center key mismatch or corruption
  └── FAIL → STRUCT_001 → DLQ (cannot process)

  CHECK 1.2: JSON / XML schema validation
  ├── Does packet body conform to current schema version?
  ├── All required top-level fields present?
  │   ├── packetId (string, unique)
  │   ├── packetVersion (must match processor version)
  │   ├── centerId (valid center identifier)
  │   ├── operatorId (valid operator identifier)
  │   ├── timestamp (ISO 8601)
  │   ├── process (NEW_ENROLLMENT / UPDATE_UIN / LOST_UIN)
  │   ├── demographic (object)
  │   ├── documents (array, ≥ 1)
  │   ├── biometrics (array)
  │   ├── consent (object)
  │   └── operatorSignature (string)
  └── FAIL → STRUCT_002 → DLQ + alert operations

  CHECK 1.3: Packet version compatibility
  ├── packetVersion matches Registration Processor version?
  ├── Older version: can processor handle it?
  │   (backward compatibility window: 2 previous versions)
  └── Too old / too new → STRUCT_003 → reject + alert

  CHECK 1.4: Packet size validation
  ├── Total packet within acceptable size range?
  ├── Too small: biometrics likely missing (< 500 KB)
  ├── Too large: possible injection / corruption (> 50 MB)
  └── Out of range → STRUCT_004 → reject

  CHECK 1.5: S3 reference integrity
  ├── All S3 references in packet point to existing objects?
  │   (biometric images, document scans, consent form)
  ├── SHA-256 hash of each S3 object matches packet metadata?
  │   (detects corruption during upload)
  └── Hash mismatch → STRUCT_005 → DLQ + alert

OUTCOME:
  All pass → proceed to Layer 2
  Any fail → DLQ + operations alert + enrollment status:
             PACKET_CORRUPT / PACKET_SCHEMA_ERROR
```

---

## 25.4 Layer 2 — Completeness Validation

```
┌──────────────────────────────────────────────────────────────┐
│              Layer 2: Completeness Validation                │
└──────────────────────────────────────────────────────────────┘

PURPOSE:
  Verify all required data elements are present,
  accounting for age-based and exception-based rules.

DEMOGRAPHIC COMPLETENESS:
  Required fields (all enrollment types):
  ├── fullName (at least one language)
  ├── dateOfBirth (or dobType = APPROXIMATE / UNKNOWN)
  ├── gender
  ├── nationality
  ├── fatherName
  └── mobileNumber (verified = true)

  Conditional fields:
  ├── guardianUIN: required if age < 18
  ├── guardianConsent: required if age < 18
  ├── medicalCertificate: required if biometricException present
  └── lateRegistrationEvidence: required if
      dobType = APPROXIMATE and no birth certificate

DOCUMENT COMPLETENESS:
  ├── At least 1 PRIMARY document present?
  │   (PASSPORT / BIRTH_CERTIFICATE / NATIONAL_ID /
  │    RESIDENCY_PERMIT)
  ├── Document scan S3 reference present?
  ├── Document metadata complete?
  │   (type, number, issuingAuthority, issuedDate)
  └── Address proof: required if address differs from
      primary document?

BIOMETRIC COMPLETENESS:
  Age-dependent requirements:
  ┌──────────────────────────────────────────────────────┐
  │ Age Group    │ Fingerprint │ Iris  │ Face │ Signature│
  ├──────────────┼─────────────┼───────┼──────┼──────────┤
  │ < 5 years    │ Not required│ Opt.  │ Req. │ Guardian │
  │ 5–17 years   │ Required    │ Req.  │ Req. │ Guardian │
  │ 18+ years    │ Required    │ Req.  │ Req. │ Required │
  └──────────────┴─────────────┴───────┴──────┴──────────┘

  Exception completeness:
  ├── If biometric MISSING without exception flag → FAIL
  ├── If exception flag present → check:
  │   ├── Reason code present?
  │   ├── Supervisor approval present?
  │   └── Medical certificate S3 ref (if medical reason)?
  └── Exception without approval → FAIL

CONSENT COMPLETENESS:
  ├── Consent record S3 reference present?
  ├── Consent timestamp present?
  ├── consentVersion matches current version?
  │   (outdated consent → flag for re-consent on update)
  ├── signedBy = CITIZEN / GUARDIAN / OFFICER_WITNESS?
  └── If GUARDIAN: guardianUIN present?

COMPLETENESS REPORT:
  {
    "completeness": {
      "demographic":  "COMPLETE",
      "documents":    "COMPLETE",
      "biometrics":   "COMPLETE_WITH_EXCEPTION",
      "exceptions":   [{
        "finger":     "RIGHT_RING",
        "reason":     "SKIN_CONDITION",
        "approved":   true,
        "approvedBy": "SUP-001"
      }],
      "consent":      "COMPLETE",
      "overallResult":"COMPLETE_WITH_EXCEPTION"
    }
  }
```

---

## 25.5 Layer 3 — Integrity Validation

```
┌──────────────────────────────────────────────────────────────┐
│              Layer 3: Integrity Validation                   │
└──────────────────────────────────────────────────────────────┘

PURPOSE:
  Verify the packet was created by an authorized operator
  at a legitimate center and has not been tampered with
  after submission.

CHECK 3.1: Operator digital signature verification
  ├── Extract operator's public key from:
  │   Operator registry (DynamoDB: nbis-operators table)
  ├── Verify: operatorSignature over packet body hash
  │   using operator's public key
  ├── Signature invalid → INTEG_001 → reject + alert
  │   (possible: tampered packet, wrong operator key)
  └── Signature valid → operator is confirmed as author

CHECK 3.2: Operator status check
  ├── Query DynamoDB: is operatorId ACTIVE?
  ├── Operator SUSPENDED: → INTEG_002 → reject
  ├── Operator REVOKED: → INTEG_003 → reject + fraud alert
  └── Operator ACTIVE → proceed

CHECK 3.3: Center authorization
  ├── Is centerId a valid, active registration center?
  ├── Is operatorId authorized to work at centerId?
  │   (operator assigned to different center → reject)
  ├── Center SUSPENDED: → INTEG_004 → reject
  └── Center ACTIVE + operator authorized → proceed

CHECK 3.4: Timestamp freshness
  ├── Packet timestamp within acceptable window?
  │   ├── Too old (> 7 days): INTEG_005
  │   │   (offline center uploaded very late — flag)
  │   └── In future: INTEG_006 (clock manipulation?)
  ├── Offline enrollment: allowed up to 7 days old
  │   (Registration Client has 7-day offline limit)
  └── Within window → proceed

CHECK 3.5: MDS device signature verification
  ├── Each biometric capture has device JWS signature
  ├── Verify JWS against device certificate from:
  │   Device registry (DynamoDB: nbis-devices table)
  ├── Device NOT in registry → INTEG_007 → reject
  │   (uncertified device used — serious violation)
  ├── Device REVOKED → INTEG_008 → reject + alert
  └── Device ACTIVE + signature valid → proceed

CHECK 3.6: Replay detection
  ├── Has this packetId been received before?
  │   Query: DynamoDB nbis-processed-packets table
  ├── FOUND: duplicate submission → INTEG_009
  │   (idempotency: return previous result, not error)
  └── NOT FOUND: first submission → proceed
      Record packetId in processed-packets (TTL: 30 days)

INTEGRITY REPORT:
  {
    "integrity": {
      "operatorSignature": "VALID",
      "operatorStatus":    "ACTIVE",
      "centerAuthorized":  true,
      "timestampValid":    true,
      "timestampAge_hours":0.5,
      "deviceSignatures":  "ALL_VALID",
      "replayCheck":       "NOT_DUPLICATE",
      "overallResult":     "PASSED"
    }
  }
```

---

## 25.6 Layer 4 — Business Rule Validation

```
┌──────────────────────────────────────────────────────────────┐
│              Layer 4: Business Rule Validation               │
└──────────────────────────────────────────────────────────────┘

PURPOSE:
  Apply domain-specific rules that govern what makes
  a legally valid national identity enrollment.

RULE 4.1: Process type validation
  ├── process = "NEW_ENROLLMENT"
  │   └── No existing UIN for this person in CIDR
  │       (dedup will confirm — but pre-check demographics)
  ├── process = "UPDATE_UIN"
  │   ├── Existing UIN must be present in packet
  │   └── UIN must exist and be ACTIVE in CIDR
  ├── process = "LOST_UIN"
  │   └── No UIN provided (person forgot their UIN)
  │       → Dedup will find existing record by biometrics
  └── process = "CORRECTION"
      ├── Officer-initiated data correction
      └── Must reference original packet ID

RULE 4.2: Age-based rules
  ├── Age calculated from DOB to enrollment date
  ├── Age < 0: impossible → reject (DOB in future)
  ├── Age 0–4: no fingerprints / no iris (age exception)
  ├── Age < 18: guardian consent + guardian UIN required
  ├── Age 18–100: standard enrollment
  └── Age > 100: supervisor approval required
      (genuine cases exist but unusual — warrant review)

RULE 4.3: Nationality and record type consistency
  ├── Nationality = BH (Bahrain) + valid CPR → CITIZEN
  ├── Nationality ≠ BH + valid residency → RESIDENT
  ├── Nationality ≠ BH + no residency → reject
  │   (cannot enroll without legal right to be in country)
  └── Refugee status documentation → REFUGEE record type

RULE 4.4: Guardian validation (for minors)
  ├── guardianUIN present?
  ├── Query CIDR: is guardianUIN ACTIVE?
  ├── Guardian record type = CITIZEN or RESIDENT?
  └── Guardian age ≥ 18?
      (guardian cannot be a minor themselves)

RULE 4.5: Document-nationality consistency
  ├── Stated nationality matches issuing country
  │   of primary document?
  ├── Example: Bahraini passport issued by BH for
  │   a person claiming BH nationality → CONSISTENT
  ├── Example: Indian passport for person claiming
  │   BH nationality → FLAG for review (dual national?)
  └── Inconsistency → WARN (not reject) → supervisor

RULE 4.6: Duplicate mobile check (soft rule)
  ├── Is this mobile number already used by > 5 UINs?
  ├── > 5 UINs on same mobile → FLAG (not reject)
  ├── Legitimate: large family sharing one phone
  └── Suspicious pattern: many unrelated people → alert

RULE 4.7: Enrollment frequency check
  ├── Has this center submitted too many packets today?
  ├── Center daily limit: configurable (e.g. 200 / day)
  ├── Over limit → WARN operations (center working overtime)
  └── Far over limit → FLAG (possible bulk fraud)

RULE 4.8: Consent version check
  ├── Is consent signed with current consent version?
  ├── Outdated consent: accept but flag for
  │   re-consent at next interaction
  └── Very outdated consent (> 2 versions old):
      reject → require fresh consent

BUSINESS RULE REPORT:
  {
    "businessRules": {
      "processType":          "NEW_ENROLLMENT",
      "processTypeValid":     true,
      "ageBasedRules":        "PASSED",
      "nationalityConsistency":"CONSISTENT",
      "guardianValid":        "N/A",
      "documentConsistency":  "CONSISTENT",
      "mobileCheck":          "CLEAR",
      "enrollmentFrequency":  "WITHIN_LIMIT",
      "consentVersion":       "CURRENT",
      "overallResult":        "PASSED"
    }
  }
```

---

## 25.7 Layer 5 — Eligibility Validation

```
┌──────────────────────────────────────────────────────────────┐
│              Layer 5: Eligibility Validation                 │
└──────────────────────────────────────────────────────────────┘

PURPOSE:
  Verify this person has the legal right to receive
  a national identity number in this jurisdiction.

CHECK 5.1: Legal presence verification
  ├── CITIZENS: birth certificate or naturalization cert?
  ├── RESIDENTS: valid residency permit / work visa?
  ├── Both: issued by authorized government authority?
  └── Cannot verify → hold for supervisor

CHECK 5.2: Existing record check (pre-dedup)
  ├── Demographic pre-check against CIDR:
  │   Query GSI: name + DOB combination
  ├── EXACT demographic match found?
  │   → FLAG: likely already enrolled (ABIS will confirm)
  │   → Continue to dedup (ABIS gives definitive answer)
  └── No demographic match → likely new person → proceed

CHECK 5.3: Watchlist check
  ├── Is name + DOB on any watchlist?
  │   ├── National security watchlist
  │   ├── International sanctions list
  │   └── Court-ordered enrollment prohibition
  ├── HIT: WATCHLIST_FLAG → hold for authority review
  │   (do NOT auto-reject — false positives are common)
  │   (human review required before any rejection)
  └── CLEAR: proceed

CHECK 5.4: Previous rejection check
  ├── Was this person previously rejected?
  │   (same biometrics rejected in previous attempt)
  ├── Previous rejection for FRAUD → ESCALATE
  ├── Previous rejection for QUALITY → allow retry
  └── No previous rejection → proceed

CHECK 5.5: Revocation check (for UPDATE / LOST_UIN)
  ├── If process = UPDATE_UIN or LOST_UIN:
  │   Is the existing UIN ACTIVE (not revoked)?
  ├── REVOKED: → ELIG_005 → reject
  │   (cannot update a revoked identity)
  └── ACTIVE: → proceed

ELIGIBILITY REPORT:
  {
    "eligibility": {
      "legalPresence":       "VERIFIED",
      "existingRecord":      "NOT_FOUND",
      "watchlistCheck":      "CLEAR",
      "previousRejection":   "NONE",
      "revocationCheck":     "N/A",
      "overallResult":       "ELIGIBLE"
    }
  }
```

---

## 25.8 Validation Pipeline — Complete Flow

```
┌──────────────────────────────────────────────────────────────┐
│              Complete Validation Pipeline Flow               │
└──────────────────────────────────────────────────────────────┘

PACKET RECEIVED (from S3, triggered by SQS)
         │
         ▼
┌─────────────────────┐
│ LAYER 1:            │
│ Structural          │
│ Validation          │
└──────────┬──────────┘
           │ PASS
           ▼
┌─────────────────────┐
│ LAYER 2:            │
│ Completeness        │
│ Validation          │
└──────────┬──────────┘
           │ PASS (or PASS_WITH_EXCEPTION)
           ▼
┌─────────────────────┐
│ LAYER 3:            │
│ Integrity           │
│ Validation          │
└──────────┬──────────┘
           │ PASS
           ▼
┌─────────────────────┐
│ LAYER 4:            │
│ Business Rule       │
│ Validation          │
└──────────┬──────────┘
           │ PASS (or PASS_WITH_FLAGS)
           ▼
┌─────────────────────┐
│ LAYER 5:            │
│ Eligibility         │
│ Validation          │
└──────────┬──────────┘
           │
    ┌──────┼──────────────────────────────────┐
    │      │                                  │
    ▼      ▼                                  ▼
  FAIL   REVIEW                            PASS
    │      │                                  │
    ▼      ▼                                  ▼
 Reject  Supervisor                    Quality Check
 + notify queue                        (Chapter 24)
 officer   │                                  │
           ▼                                  ▼
      Supervisor                        Deduplication
      approves?                         (Chapter 26)
      ├── YES → Quality Check
      └── NO  → Reject + notify

VALIDATION STATES:
  VALIDATION_PASSED        → proceed to quality check
  VALIDATION_REVIEW        → supervisor queue
  VALIDATION_REJECTED      → officer notification
  VALIDATION_FRAUD_FLAG    → security escalation
  VALIDATION_DUPLICATE     → idempotent return (replay)
```

---

## 25.9 Validation Error Codes — Complete Reference

```
┌──────────────────────────────────────────────────────────────┐
│              Enrollment Validation Error Codes               │
├──────────────────┬───────────────────────────────────────────┤
│ Code             │ Meaning                                  │
├──────────────────┼───────────────────────────────────────────┤
│                  │ STRUCTURAL                               │
│ STRUCT_001       │ Packet decryption failed                 │
│ STRUCT_002       │ Schema validation failed — missing field │
│ STRUCT_003       │ Packet version incompatible             │
│ STRUCT_004       │ Packet size out of acceptable range     │
│ STRUCT_005       │ S3 object hash mismatch — corruption    │
├──────────────────┼───────────────────────────────────────────┤
│                  │ COMPLETENESS                             │
│ COMP_001         │ Required demographic field missing      │
│ COMP_002         │ No primary identity document            │
│ COMP_003         │ Required biometric missing (no exception)│
│ COMP_004         │ Exception flag without supervisor OK   │
│ COMP_005         │ Guardian consent missing for minor      │
│ COMP_006         │ Consent record S3 reference missing    │
│ COMP_007         │ Medical certificate missing for med exc.│
├──────────────────┼───────────────────────────────────────────┤
│                  │ INTEGRITY                                │
│ INTEG_001        │ Operator signature invalid              │
│ INTEG_002        │ Operator account suspended              │
│ INTEG_003        │ Operator account revoked                │
│ INTEG_004        │ Registration center inactive            │
│ INTEG_005        │ Packet timestamp too old (> 7 days)    │
│ INTEG_006        │ Packet timestamp in future             │
│ INTEG_007        │ Biometric device not in registry       │
│ INTEG_008        │ Biometric device revoked               │
│ INTEG_009        │ Duplicate packet (replay detected)     │
├──────────────────┼───────────────────────────────────────────┤
│                  │ BUSINESS RULES                           │
│ BIZ_001          │ Invalid process type                   │
│ BIZ_002          │ UPDATE_UIN: UIN not found or inactive  │
│ BIZ_003          │ Age rule violation (e.g. minor, >100) │
│ BIZ_004          │ Nationality / record type mismatch     │
│ BIZ_005          │ Guardian UIN invalid or inactive       │
│ BIZ_006          │ Center daily enrollment limit exceeded │
│ BIZ_007          │ Consent version too outdated          │
├──────────────────┼───────────────────────────────────────────┤
│                  │ ELIGIBILITY                              │
│ ELIG_001         │ No legal presence document            │
│ ELIG_002         │ Watchlist flag — hold for review      │
│ ELIG_003         │ Previous fraud rejection on record    │
│ ELIG_004         │ Demographic pre-match found in CIDR   │
│ ELIG_005         │ UIN is revoked — cannot update        │
└──────────────────┴───────────────────────────────────────────┘
```

---

## 25.10 Validation for Different Process Types

Different enrollment process types have different validation rules:

```
┌──────────────────────────────────────────────────────────────┐
│              Validation Rules by Process Type                │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  NEW_ENROLLMENT (first time):                                │
│  ├── All layers: full validation                           │
│  ├── No existing UIN expected                              │
│  ├── All biometrics required (age-appropriate)             │
│  └── Full document proof required                          │
│                                                              │
│  UPDATE_UIN (demographic update):                            │
│  ├── Existing UIN: must be present and ACTIVE in CIDR      │
│  ├── Biometrics: not recaptured (reuse enrolled templates) │
│  │   UNLESS biometric refresh is part of the update       │
│  ├── Changed fields: must have supporting document         │
│  │   (name change: court order; address: utility bill)    │
│  ├── Layers 1–3: full validation                          │
│  └── Layer 4–5: modified (some rules N/A for updates)    │
│                                                              │
│  LOST_UIN (re-issue after losing UIN number):               │
│  ├── No UIN provided in packet (citizen forgot it)         │
│  ├── Biometrics: fully recaptured (to find the UIN)       │
│  ├── Documents: required (prove identity without UIN)      │
│  ├── Process: dedup WILL find existing record by biometrics│
│  │   → dedup result: existing UIN returned                │
│  └── If dedup finds no match: treat as new enrollment     │
│                                                              │
│  CORRECTION (officer error fix):                             │
│  ├── References original packetId                          │
│  ├── Operator must be SUPERVISOR or higher                 │
│  ├── Changed fields: documented with reason               │
│  ├── Audit trail: original value + corrected value        │
│  └── No biometric recapture required                      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 25.11 Validation Audit Trail

Every validation decision — pass, fail, or review — must be recorded in the immutable audit log:

```
┌──────────────────────────────────────────────────────────────┐
│              Validation Audit Log Entry                      │
└──────────────────────────────────────────────────────────────┘
{
  "eventId":         "EVT-20250115-VAL-001",
  "eventType":       "ENROLLMENT_VALIDATION_COMPLETED",
  "timestamp":       "2025-01-15T11:00:00Z",
  "packetId":        "PKT-20250115-001",
  "centerId":        "CENTER-001",
  "operatorId":      "OFR-007",
  "processType":     "NEW_ENROLLMENT",

  "validationResult": {
    "structural":    "PASSED",
    "completeness":  "PASSED_WITH_EXCEPTION",
    "integrity":     "PASSED",
    "businessRules": "PASSED",
    "eligibility":   "PASSED",
    "overallDecision":"VALIDATION_PASSED"
  },

  "exceptions": [{
    "finger":  "RIGHT_RING",
    "reason":  "SKIN_CONDITION",
    "approvedBy":"SUP-001"
  }],

  "flags": [],

  "processingTime_ms": 847,
  "nextStage":         "QUALITY_CHECK",
  "traceId":           "abc-123-xyz"
}
```

---

## 25.12 Supervisor Review Interface

Packets in VALIDATION_REVIEW state are presented to supervisors through a dedicated portal:

```
┌──────────────────────────────────────────────────────────────┐
│              Supervisor Review — Validation Queue            │
└──────────────────────────────────────────────────────────────┘

SUPERVISOR SEES:
  ┌────────────────────────────────────────────────────────┐
  │ Packet: PKT-20250115-042                               │
  │ Center: CENTER-001  Operator: OFR-007                  │
  │ Submitted: 15 Jan 2025, 10:45 AM                       │
  │                                                        │
  │ Citizen: Hamza Ahmed Rafique  DOB: 15/05/1990          │
  │                                                        │
  │ REVIEW REASON:                                         │
  │ ⚠️  ELIG_004: Demographic pre-match found in CIDR      │
  │    Name + DOB combination matches existing record      │
  │    Possible: already enrolled, or name/DOB collision   │
  │                                                        │
  │ SUPERVISOR ACTIONS:                                    │
  │ [APPROVE — Continue to dedup (ABIS will confirm)]     │
  │ [REJECT  — Send back to officer for investigation]    │
  │ [ESCALATE — Refer to fraud investigation team]        │
  │                                                        │
  │ Required: comment before any action                    │
  │ Comment: ________________________________              │
  └────────────────────────────────────────────────────────┘

SUPERVISOR DECISION:
  ├── APPROVE → packet proceeds to quality check + dedup
  ├── REJECT  → officer notified with reason
  │             citizen informed: bring more documentation
  └── ESCALATE → fraud team notified
                 packet frozen pending investigation

ALL DECISIONS AUDITED:
  {
    "eventType":      "SUPERVISOR_VALIDATION_DECISION",
    "packetId":       "PKT-20250115-042",
    "supervisorId":   "SUP-001",
    "decision":       "APPROVED",
    "reason":         "ELIG_004 — reviewed demographics,
                       no existing record found on manual
                       CIDR lookup. Name collision only.
                       Proceed to ABIS dedup.",
    "timestamp":      "2025-01-15T11:15:00Z"
  }
```

---

## 25.13 Notification Design

When validation fails, the right people must be notified with actionable information:

```
┌──────────────────────────────────────────────────────────────┐
│              Validation Failure Notifications                │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  RECIPIENT: Enrollment Officer                               │
│  TRIGGER: VALIDATION_REJECTED                               │
│  CHANNEL: Registration Client alert + SMS                   │
│  MESSAGE:                                                    │
│  "Enrollment PKT-20250115-001 rejected.                     │
│   Reason: COMP_003 — Right iris biometric missing          │
│   without documented exception.                            │
│   Action: Citizen must return to center.                   │
│   Please re-enroll with complete biometric capture."       │
│                                                              │
│  RECIPIENT: Registration Center Supervisor                   │
│  TRIGGER: VALIDATION_REVIEW                                 │
│  CHANNEL: Supervisor portal notification + email            │
│  MESSAGE:                                                    │
│  "Packet PKT-20250115-042 requires your review.            │
│   Reason: ELIG_004 — demographic pre-match detected.       │
│   Action required: approve, reject, or escalate."          │
│                                                              │
│  RECIPIENT: Security / Fraud Team                           │
│  TRIGGER: VALIDATION_FRAUD_FLAG                             │
│  CHANNEL: SNS → email + PagerDuty (P1)                     │
│  MESSAGE:                                                    │
│  "FRAUD FLAG on PKT-20250115-099.                          │
│   Reason: INTEG_003 — revoked operator used.              │
│   Center: CENTER-001. Immediate investigation required."   │
│                                                              │
│  RECIPIENT: Operations Team                                  │
│  TRIGGER: STRUCT_001 / STRUCT_002 (packet corrupt)          │
│  CHANNEL: CloudWatch alarm → SNS → PagerDuty               │
│  MESSAGE:                                                    │
│  "Packet PKT-20250115-007 cannot be processed.             │
│   Reason: STRUCT_001 — decryption failed.                  │
│   Packet in DLQ. Engineering investigation needed."        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 25.14 AWS Architecture for Enrollment Validation

```
┌──────────────────────────────────────────────────────────────┐
│              Enrollment Validation — AWS Architecture        │
└──────────────────────────────────────────────────────────────┘

SQS (nbis-enrollment-received)
         │
         ▼
Enrollment Validator (ECS Fargate)
  │
  ├── LAYER 1: Structural
  │   ├── S3: retrieve packet
  │   ├── KMS: decrypt packet
  │   └── JSON Schema validation (in-memory)
  │
  ├── LAYER 2: Completeness
  │   ├── Field completeness check (in-memory logic)
  │   └── S3: verify biometric + document S3 refs exist
  │       + SHA-256 hash verification
  │
  ├── LAYER 3: Integrity
  │   ├── DynamoDB: operator lookup (nbis-operators)
  │   ├── DynamoDB: center lookup (nbis-centers)
  │   ├── DynamoDB: device lookup (nbis-devices)
  │   ├── RSA signature verification (in-memory)
  │   └── DynamoDB: replay check (nbis-processed-packets)
  │
  ├── LAYER 4: Business Rules
  │   ├── Age calculation (in-memory)
  │   ├── DynamoDB: guardian UIN lookup (if minor)
  │   ├── DynamoDB: center daily count (nbis-center-stats)
  │   └── RDS: consent version check (nbis-consent-versions)
  │
  ├── LAYER 5: Eligibility
  │   ├── DynamoDB GSI: demographic pre-match check
  │   ├── External: watchlist API call (if configured)
  │   └── DynamoDB: previous rejection check
  │
  ├── Decision routing:
  │   ├── PASSED    → SQS: nbis-biometric-processing
  │   ├── REVIEW    → SQS: nbis-supervisor-review
  │   ├── REJECTED  → SNS: officer notification
  │   └── FRAUD     → SNS: security team (P1)
  │
  ├── DynamoDB:
  │   └── Update enrollment status + store validation report
  │
  └── CloudWatch:
      └── Audit log + validation metrics

AWS SERVICES:
  ├── ECS Fargate:   validation processor
  ├── DynamoDB:      operators, centers, devices, CIDR lookups
  ├── S3 + KMS:      packet retrieval + decryption
  ├── RDS:           consent version + business rule config
  ├── SQS:           routing to next pipeline stage
  ├── SNS:           notifications to officers + security
  └── CloudWatch:    metrics + audit logging
```

---

## 25.15 MOSIP Reference

```
┌──────────────────────────────────────────────────────────────┐
│              MOSIP Enrollment Validation Reference           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  MOSIP Registration Processor validation stages:            │
│                                                              │
│  Stage: Packet Receiver                                     │
│  └── Validates packet can be received + decrypted           │
│      (our Layer 1: Structural)                              │
│                                                              │
│  Stage: Packet Validator                                    │
│  ├── Schema validation against MOSIP ID Schema             │
│  ├── Operator + center authorization checks                │
│  ├── Machine (Registration Client device) authorization    │
│  └── (our Layer 2: Completeness + Layer 3: Integrity)     │
│                                                              │
│  Stage: Operator Validator                                  │
│  ├── Validates operator digital signature on packet        │
│  └── Checks operator is authorized (our Layer 3)          │
│                                                              │
│  Stage: Demo Dedupe                                         │
│  ├── Lightweight demographic pre-check before ABIS         │
│  └── (our Layer 5 demographic pre-match check)            │
│                                                              │
│  MOSIP supervisor portal:                                   │
│  ├── Registration Supervisor application                   │
│  ├── Displays packets awaiting approval                    │
│  ├── Shows validation failure reasons                      │
│  └── Approve / reject / request more info                  │
│                                                              │
│  MOSIP configurable rules:                                  │
│  ├── Age thresholds configurable per country              │
│  ├── Required documents configurable via UI               │
│  ├── Center daily limits configurable                     │
│  └── Consent versions managed via admin portal            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 25.16 Key Terms

| Term | Definition |
|------|-----------|
| **Enrollment validation** | Verifying a packet is complete, authentic, and legally sufficient before UIN assignment |
| **Structural validation** | Checking packet format, schema, size, and file integrity |
| **Completeness validation** | Ensuring all required fields, documents, and biometrics are present |
| **Integrity validation** | Verifying packet authenticity — operator signature, device certification, replay detection |
| **Business rule validation** | Applying domain-specific rules — age thresholds, process types, guardian requirements |
| **Eligibility validation** | Confirming the person has legal right to a national identity in this jurisdiction |
| **Operator signature** | Cryptographic signature proving the packet was created by the registered operator |
| **MDS device signature** | JWS signature from certified biometric device proving capture is genuine |
| **Replay detection** | Checking if this packetId has been submitted before (idempotency) |
| **Watchlist check** | Comparing name + DOB against national security / sanctions lists |
| **Demographic pre-match** | Quick name + DOB lookup in CIDR before full ABIS dedup |
| **Process type** | NEW_ENROLLMENT / UPDATE_UIN / LOST_UIN / CORRECTION |
| **VALIDATION_REVIEW** | Outcome requiring human supervisor decision before proceeding |
| **VALIDATION_REJECTED** | Hard failure — enrollment must be redone by officer |
| **VALIDATION_FRAUD_FLAG** | Security escalation — possible deliberate fraud detected |
| **Guardian UIN** | Parent or guardian's identity number required for minor enrollment |
| **Consent version** | Version number of consent text — ensures citizen signed current version |
| **SHA-256 hash** | Cryptographic hash used to verify S3 biometric files are untampered |

---

## 25.17 Key Takeaways

- **Validation and quality checks are complementary, not the same** — quality checks verify data is good enough; validation verifies the enrollment is legally and technically complete. Both must pass before deduplication.
- **Structural validation must run first** — if the packet cannot be decrypted or parsed, no other validation can run. Structural failures go to the DLQ; they are engineering problems, not business problems.
- **Integrity validation catches insider threats** — verifying the operator signature, device certification, and center authorization ensures that even someone with physical access to the Registration Client cannot create fraudulent packets without leaving a traceable audit trail.
- **Replay detection is idempotency at the validation layer** — a packet submitted twice returns the same result without creating a duplicate enrollment. This is critical for offline enrollment scenarios where sync may retry submissions.
- **Watchlist checks are flags, not automatic rejections** — false positive rates on name/DOB watchlist matching are significant. Human review is always required before a watchlist flag becomes a rejection.
- **Business rules are configurable by design** — age thresholds, document requirements, and center daily limits must be configurable without code changes. What is appropriate for Bahrain's small urban population differs from a country with 100 million rural citizens.
- **The supervisor queue is a safety valve, not a failure state** — VALIDATION_REVIEW means the system detected something unusual but cannot make the decision autonomously. Human judgment fills the gap.
- **Every validation decision is audited** — pass, fail, review, supervisor approval, and supervisor rejection. The audit trail must capture what was decided, by whom, at what time, and with what stated reason.

---

## 25.18 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 26 | Duplicate Detection — ABIS 1:N search, dedup workflow, candidate scoring |
| Chapter 27 | Approval Workflow — supervisor review portal, decision audit, escalation |
| Chapter 33 | Matching Algorithms — how ABIS scores are computed |

---

*Chapter 25 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*
