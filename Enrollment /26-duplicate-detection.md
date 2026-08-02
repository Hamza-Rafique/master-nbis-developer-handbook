# Chapter 26 — Duplicate Detection

> **Part III — Enrollment**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 26.1 What is Duplicate Detection?

**Duplicate detection** — also called **biometric deduplication** — is the process of searching the entire national identity database to confirm that a person being enrolled does not already have a UIN under any name, in any registration center, at any point in time.

It is the most computationally intensive operation in the entire NBIS, and the one that makes the system's central promise possible: **one person, one identity**.

```
┌──────────────────────────────────────────────────────────────┐
│              Why Duplicate Detection Is the Core Function    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  WITHOUT deduplication:                                      │
│  ├── Person enrolls as "Ahmed Ali" at Center A             │
│  ├── Same person enrolls as "Ahmad Ali" at Center B        │
│  ├── Receives two UINs: 111111111111 and 222222222222      │
│  ├── Claims government benefits under both UINs            │
│  ├── Votes twice in elections                               │
│  ├── Has two passports under two identities                │
│  └── National registry is fundamentally unreliable         │
│                                                              │
│  WITH biometric deduplication:                               │
│  ├── "Ahmed Ali" enrolls at Center A → UIN assigned        │
│  ├── "Ahmad Ali" tries to enroll at Center B               │
│  ├── ABIS: fingerprints match existing record (0.08 HD)    │
│  ├── DEDUP_FLAGGED → supervisor review                     │
│  ├── Supervisor confirms: same person, different name      │
│  └── Second enrollment REJECTED                            │
│                                                              │
│  Biometrics cannot lie. A person's fingerprints are the    │
│  same regardless of what name they give.                    │
│  That is the entire value proposition of biometric dedup.  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 26.2 Deduplication vs Verification

The same biometric matching technology serves two opposite directions:

```
┌──────────────────────────────────────────────────────────────┐
│              Deduplication vs Verification                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  DEDUPLICATION (1:N — at enrollment):                        │
│  ├── Question: "Is this person already enrolled?"           │
│  ├── Direction: NEW biometric searched against ALL existing  │
│  ├── Goal: find if a match EXISTS anywhere in database      │
│  ├── Result: MATCH FOUND (flag) or NO MATCH (proceed)      │
│  ├── Speed: slow (minutes — searching millions of records)  │
│  └── When: ONCE per person, at enrollment time              │
│                                                              │
│  VERIFICATION (1:1 — at authentication):                     │
│  ├── Question: "Is this person who they claim to be?"       │
│  ├── Direction: LIVE biometric compared to ONE record       │
│  ├── Goal: confirm a SPECIFIC claimed identity             │
│  ├── Result: MATCH (authenticate) or NO MATCH (deny)       │
│  ├── Speed: fast (milliseconds — single comparison)        │
│  └── When: every time citizen uses identity service        │
│                                                              │
│  KEY INSIGHT:                                                │
│  Deduplication uses the same matching algorithm as          │
│  verification, but applied N times instead of 1 time.      │
│  The scale (N = millions) is what makes it different.       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 26.3 ABIS — The Deduplication Engine

**ABIS (Automated Biometric Identification System)** is the external biometric engine responsible for performing 1:N deduplication. MOSIP does not provide an ABIS — it defines a queue-based interface, and countries select a certified vendor to plug in.

```
┌──────────────────────────────────────────────────────────────┐
│              What ABIS Does and Does Not Do                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ABIS DOES:                                                  │
│  ├── Receive biometric templates from NBIS via queue       │
│  ├── Search its internal index against ALL enrolled        │
│  │   templates (1:N)                                        │
│  ├── Return ranked candidate list with match scores        │
│  ├── Support multimodal fusion (finger + iris + face)      │
│  └── Process deduplication jobs in parallel (batch)        │
│                                                              │
│  ABIS DOES NOT:                                              │
│  ├── Store citizen demographic data (UIN only)             │
│  ├── Make the final ACCEPT / REJECT decision               │
│  │   (NBIS makes the decision based on ABIS scores)        │
│  ├── Call the NBIS directly (queue-based only)             │
│  └── Have direct database access to the CIDR               │
│                                                              │
│  KNOWN ABIS VENDORS:                                         │
│  ├── Idemia (former Morpho) — Aadhaar ABIS                 │
│  ├── NEC — high accuracy, used in border systems           │
│  ├── Aware — open integration, used in MOSIP deployments   │
│  ├── Neurotechnology — MegaMatcher ABIS                    │
│  └── Daon — cloud-native ABIS                              │
│                                                              │
│  ABIS SELECTION CRITERIA:                                    │
│  ├── NIST MINEX (fingerprint) ranking                      │
│  ├── NIST IREX (iris) ranking                              │
│  ├── NIST FRVT (face) ranking                              │
│  ├── Throughput: enrollments per hour it can process       │
│  ├── Accuracy at target database size (1M / 10M / 100M)   │
│  └── Data sovereignty: templates stay in-country?          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 26.4 ABIS Queue Interface — MOSIP Design

MOSIP uses a **queue-based ABIS interface** — the NBIS and ABIS never call each other directly. All communication flows through SQS queues. This is a deliberate decoupling design:

```
┌──────────────────────────────────────────────────────────────┐
│              MOSIP ABIS Queue Interface                      │
└──────────────────────────────────────────────────────────────┘

NBIS → ABIS (dedup request):

  SQS Queue: nbis-abis-requests
  Message:
  {
    "id":           "ABIS-JOB-20250115-001",
    "requestId":    "PKT-20250115-001",
    "referenceId":  "PKT-20250115-001",
    "requesttime":  "2025-01-15T11:00:00Z",
    "ver":          "1.0",
    "gallery": {
      "referenceURL":"s3://nbis-abis-gallery/"
    },
    "probe": {
      "referenceURL":"s3://nbis-abis-probe/PKT-001/"
    }
  }

  "gallery" = all enrolled templates (ABIS internal index)
  "probe"   = new enrollment templates being checked

ABIS → NBIS (dedup result):

  SQS Queue: nbis-abis-results
  Message:
  {
    "id":           "ABIS-JOB-20250115-001",
    "requestId":    "PKT-20250115-001",
    "returnValue":  "success",
    "candidates": [
      {
        "referenceId": "UIN-987654321098",
        "scaledScore": 94.7,          ← very likely same person
        "analyticsInfo": {
          "fingerMatchScore": 98.2,
          "irisMatchScore":   91.3,
          "faceMatchScore":   94.1,
          "fusionScore":      94.7
        }
      },
      {
        "referenceId": "UIN-345678901234",
        "scaledScore": 31.2,          ← below threshold, ignore
        "analyticsInfo": { ... }
      }
    ]
  }

  "scaledScore" = 0–100 (100 = identical)
  Candidates returned: top N above reporting threshold
  NBIS decision threshold: typically scaledScore ≥ 75
```

---

## 26.5 Deduplication Workflow — Step by Step

```
┌──────────────────────────────────────────────────────────────┐
│              Complete Deduplication Workflow                 │
└──────────────────────────────────────────────────────────────┘

STEP 1: Biometric quality passed (from Chapter 24)
  Enrollment packet quality-checked and validated
         │
         ▼
STEP 2: Template preparation
  Deduplication Service:
  ├── Retrieve decrypted biometric templates from S3
  │   ├── 10 fingerprint templates (ISO 19794-2)
  │   ├── 2 iris templates (ISO 19794-6)
  │   └── 1 face template (ISO 19794-5)
  ├── Bundle into ABIS probe package
  └── Upload probe package to S3 ABIS staging bucket
         │
         ▼
STEP 3: Send dedup job to ABIS
  Deduplication Service → SQS (nbis-abis-requests):
  ├── Job ID: ABIS-JOB-20250115-001
  ├── Reference to probe templates in S3
  └── Reference to gallery (full enrolled database)
         │
         ▼
STEP 4: ABIS processing (external, async)
  ABIS consumes from SQS queue:
  ├── Retrieve probe templates from S3
  ├── Run 1:N search across ALL enrolled templates
  │   ├── Fingerprint: minutiae matching × N records
  │   ├── Iris: Hamming distance × N records
  │   └── Face: embedding cosine similarity × N records
  ├── Fuse scores: weighted combination
  └── Return: ranked candidate list via SQS
  Time: 5 seconds to 30 minutes (depending on N)
         │
         ▼
STEP 5: ABIS result received
  Deduplication Service consumes nbis-abis-results:
  ├── Match any candidates above threshold (75)?
  │
  ├── YES — candidates found:
  │   ├── Sort by scaledScore descending
  │   ├── Top candidate score ≥ 75?
  │   │   → DEDUP_FLAGGED
  │   │   → Publish event with candidate list
  │   └── Proceed to Step 6a
  │
  └── NO — no candidates above threshold:
      → DEDUPLICATION_CLEARED
      → Publish event
      └── Proceed to Step 6b
         │
         ▼
STEP 6a: DEDUP_FLAGGED path
  ├── Enrollment status: DEDUP_FLAGGED
  ├── Candidate record stored (DynamoDB)
  ├── Published to: SQS (nbis-supervisor-review)
  └── Supervisor notified (email + portal)
      → Manual review required (Chapter 27)
         │
STEP 6b: DEDUPLICATION_CLEARED path
  ├── Enrollment status: DEDUP_CLEARED
  └── Published to: SQS (nbis-uin-assignment.fifo)
      → UIN Service assigns new UIN
      → Credential issuance begins
```

---

## 26.6 Score Thresholds and Decision Logic

```
┌──────────────────────────────────────────────────────────────┐
│              Score Threshold Decision Logic                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ABIS returns candidates with scaledScore 0–100:            │
│                                                              │
│  Score ≥ 85: HIGH CONFIDENCE MATCH                          │
│  ├── Almost certainly the same person                       │
│  ├── Likely intentional duplicate enrollment attempt        │
│  └── Action: DEDUP_FLAGGED (high priority review)          │
│                                                              │
│  Score 75–84: MEDIUM CONFIDENCE MATCH                       │
│  ├── Probably the same person                               │
│  ├── May be genuine (re-enrollment, poor quality before)   │
│  └── Action: DEDUP_FLAGGED (standard review)               │
│                                                              │
│  Score 60–74: LOW CONFIDENCE CANDIDATE                      │
│  ├── Possibly the same person, possibly not                 │
│  ├── Below automatic flag threshold                        │
│  └── Action: LOGGED but proceed (human reviewable)         │
│                                                              │
│  Score < 60: NO MATCH                                        │
│  ├── Different person (biometrics do not match)             │
│  └── Action: discard, proceed to CLEARED                   │
│                                                              │
│  DECISION THRESHOLDS (configurable):                         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Threshold Name     │ Value │ Action                  │   │
│  ├────────────────────┼───────┼─────────────────────────┤   │
│  │ Hard flag          │ ≥ 75  │ DEDUP_FLAGGED           │   │
│  │ Soft log           │ 60–74 │ Log + proceed           │   │
│  │ Ignore             │ < 60  │ Discard candidate       │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  These thresholds balance:                                   │
│  ├── FALSE POSITIVES (same person matched twice → good)    │
│  └── FALSE NEGATIVES (different people matched → bad)      │
│                                                              │
│  Setting threshold too LOW → too many false flags          │
│  → supervisor queue overwhelmed → operational breakdown    │
│                                                              │
│  Setting threshold too HIGH → genuine duplicates pass      │
│  → two UINs for one person → integrity failure             │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 26.7 Multimodal Fusion

Modern ABIS systems combine multiple biometric modalities for higher accuracy. The fusion of fingerprint + iris + face is more accurate than any single modality alone:

```
┌──────────────────────────────────────────────────────────────┐
│              Multimodal Fusion in Deduplication              │
└──────────────────────────────────────────────────────────────┘

INDIVIDUAL MODALITY SCORES:
  Fingerprint match score: 98.2 (right index vs enrolled)
  Iris Hamming distance:   0.09 (very similar irises)
  Face cosine similarity:  0.94 (very similar faces)

SCORE NORMALIZATION:
  Each modality uses a different scale.
  Normalize to 0–100 before fusion:
  ├── Fingerprint: already 0–100 → 98.2
  ├── Iris: convert HD to score: (1 - HD) × 100
  │         (1 - 0.09) × 100 = 91.0
  └── Face: convert cosine sim to score:
            0.94 × 100 = 94.0

FUSION (weighted sum — configurable):
  fusionScore = (
    fingerprint × 0.50 +   ← highest weight (most mature)
    iris        × 0.30 +   ← second weight (most accurate)
    face        × 0.20     ← third weight (most variable)
  )

  = (98.2 × 0.50) + (91.0 × 0.30) + (94.0 × 0.20)
  = 49.1 + 27.3 + 18.8
  = 95.2 ← fusion score

FUSION ADVANTAGES:
  ├── Higher accuracy than any single modality
  ├── More robust: even if one modality has poor quality,
  │   other modalities compensate
  ├── Harder to spoof: attacker must defeat ALL modalities
  └── Better for populations where one modality is weak
      (fingerprint-poor population: iris + face compensate)

MISSING MODALITY HANDLING:
  If iris not captured (exception):
  fusionScore = fingerprint × 0.70 + face × 0.30
  (weights redistributed to available modalities)
```

---

## 26.8 Demographic Deduplication (Pre-ABIS)

Before the expensive ABIS 1:N biometric search, MOSIP runs a quick **demographic deduplication** check:

```
┌──────────────────────────────────────────────────────────────┐
│              Demographic Deduplication (Pre-ABIS)            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PURPOSE:                                                    │
│  Quick demographic pre-check to catch obvious duplicates    │
│  before spending ABIS compute on them.                      │
│  Also useful for low-biometric-quality scenarios.           │
│                                                              │
│  HOW IT WORKS:                                               │
│  Query CIDR DynamoDB GSI:                                   │
│  ├── name (normalized) + DOB: exact match?                 │
│  ├── name (normalized) + DOB + gender: exact match?        │
│  └── father's name + mother's name + DOB: exact match?    │
│                                                              │
│  RESULT:                                                     │
│  ├── Match found: DEMO_DEDUP_FLAGGED                       │
│  │   → Added to ABIS job as candidate hint                 │
│  │   → ABIS prioritizes this candidate in its search      │
│  └── No match: proceed to ABIS as normal                  │
│                                                              │
│  IMPORTANT:                                                  │
│  Demographic dedup is NOT a substitute for biometric dedup. │
│  Names change. Data entry errors happen.                    │
│  ├── Mohammed enrolls twice with different spellings       │
│  │   → Demo dedup: different name → no match              │
│  │   → Biometric dedup: same fingerprints → MATCH         │
│  │   → Only biometric catches this                        │
│  └── Two different people named Mohammed Ali, born same   │
│      date (rare but possible)                              │
│      → Demo dedup: match (false positive)                  │
│      → Biometric dedup: different people → no match       │
│      → Biometric dedup corrects the false positive        │
│                                                              │
│  RULE: Biometric dedup is authoritative.                    │
│  Demographic dedup is a performance optimization only.      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 26.9 ABIS Performance and Scalability

```
┌──────────────────────────────────────────────────────────────┐
│              ABIS Performance Considerations                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  SEARCH TIME BY DATABASE SIZE:                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Database Size  │ Approx Search Time (fingerprint)    │   │
│  ├────────────────┼────────────────────────────────────┤   │
│  │ 100,000        │ < 1 second                         │   │
│  │ 1,000,000      │ 5–15 seconds                       │   │
│  │ 10,000,000     │ 30–120 seconds                     │   │
│  │ 100,000,000    │ 2–10 minutes                       │   │
│  │ 1,000,000,000  │ 10–60 minutes (Aadhaar scale)      │   │
│  └──────────────────────────────────────────────────────┘   │
│  Times vary by ABIS vendor, hardware, and modality.         │
│                                                              │
│  THROUGHPUT (enrollments per hour):                          │
│  ├── Small country (< 1M records): 1,000+ / hour           │
│  ├── Medium country (1–10M records): 200–500 / hour        │
│  ├── Large country (10–100M): 50–200 / hour               │
│  └── Aadhaar scale (1B+): ~20 / hour per ABIS instance    │
│      (multiple ABIS instances run in parallel)              │
│                                                              │
│  SCALING ABIS FOR NATIONAL CAMPAIGNS:                        │
│  ├── ABIS can run multiple instances in parallel            │
│  ├── SQS distributes jobs across ABIS instances            │
│  ├── Each instance processes different packets             │
│  └── Linear scale-out (2× ABIS instances = 2× throughput) │
│                                                              │
│  ABIS QUEUE MANAGEMENT:                                      │
│  ├── Visibility timeout: 600 seconds (10 minutes)          │
│  │   (large databases need long processing time)           │
│  ├── DLQ: if ABIS fails 3 times → DLQ + P1 alert          │
│  └── Backpressure: SQS queue depth alarm if > 1000        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 26.10 Duplicate Detection Outcome Scenarios

```
┌──────────────────────────────────────────────────────────────┐
│              Duplicate Detection Outcome Scenarios           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  SCENARIO 1: Clean — no duplicate found                      │
│  ─────────────────────────────────────                       │
│  ABIS returns: no candidates above threshold                │
│  Decision: DEDUPLICATION_CLEARED                            │
│  Action: → UIN assigned → credential issued                 │
│  Frequency: majority of enrollments (~95%+)                 │
│                                                              │
│  SCENARIO 2: Genuine duplicate — same person enrolled twice  │
│  ─────────────────────────────────────────────────────────  │
│  ABIS returns: scaledScore = 96.3 for UIN-123456789012      │
│  Decision: DEDUP_FLAGGED                                    │
│  Supervisor review: biometrics clearly match               │
│  Reason: person was enrolled before, moved to new center    │
│  Action: REJECT new enrollment, notify citizen of           │
│          existing UIN: 123456789012                         │
│                                                              │
│  SCENARIO 3: Fraud attempt — deliberate dual enrollment      │
│  ─────────────────────────────────────────────────────────  │
│  ABIS returns: scaledScore = 91.7 for UIN-987654321098      │
│  Supervisor review: same biometrics, different names        │
│  Existing: "Mohammed Ali Hassan" (UIN assigned)             │
│  New attempt: "Ahmad Hassan" at different center            │
│  Action: REJECT, log fraud attempt, alert security team     │
│          Flag UIN-987654321098 for investigation           │
│                                                              │
│  SCENARIO 4: Near-duplicate — different people, similar bio  │
│  ─────────────────────────────────────────────────────────  │
│  ABIS returns: scaledScore = 76.8 for UIN-111222333444      │
│  Supervisor review: different faces, different documents    │
│  Analysis: biometric similarity just above threshold        │
│            (rare — 1 in 10,000 pairs may be near-match)    │
│  Action: CLEAR with documented reason (supervisor confirms  │
│          these are different people)                        │
│  Result: new enrollment proceeds to UIN assignment          │
│                                                              │
│  SCENARIO 5: Data recovery — LOST_UIN process               │
│  ─────────────────────────────────────────────────────────  │
│  Citizen forgot their UIN, came to re-enroll                │
│  ABIS: scaledScore = 98.1 for UIN-555666777888             │
│  Supervisor: confirms existing record matches citizen       │
│  Action: NOT a new enrollment — return existing UIN         │
│          Re-issue credential with existing UIN              │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 26.11 Candidate Review Data

When a duplicate is flagged, the supervisor must see enough information to make an informed decision:

```
┌──────────────────────────────────────────────────────────────┐
│              Candidate Review Data Package                   │
└──────────────────────────────────────────────────────────────┘

Presented to supervisor for review:

PROBE (new enrollment):
  ├── Name:   Ahmad Hassan
  ├── DOB:    15/05/1990
  ├── Center: CENTER-005 (Riffa)
  ├── Date:   15 Jan 2025
  ├── Face photo: [displayed]
  └── Biometric quality: all GOOD

CANDIDATE (existing enrolled record):
  ├── Name:   Mohammed Ali Hassan
  ├── DOB:    15/05/1990
  ├── UIN:    987654321098
  ├── Enrolled: 10 Mar 2022, CENTER-001 (Manama)
  ├── Status: ACTIVE
  └── Face photo: [displayed for comparison]

MATCH SCORES:
  ├── Fingerprint: 93.4 (RIGHT_INDEX most similar)
  ├── Iris:        88.7 (LEFT_EYE most similar)
  ├── Face:        87.2
  └── Fusion:      91.7 ← above threshold (75)

SUPERVISOR SEES:
  ├── Side-by-side face photos
  ├── Match score breakdown per modality
  ├── Fingerprint minutiae overlay (visual match evidence)
  ├── Both enrollment dates and centers
  └── Document numbers from both enrollments

SUPERVISOR OPTIONS:
  [CONFIRM DUPLICATE — reject new enrollment]
  [CLEAR — these are different people]
  [ESCALATE — refer to fraud investigation]
  [REQUEST MORE INFO — hold, contact officer]

  Required: comment field before any action
```

---

## 26.12 Deduplication Against Child Records

Children enrolled before age 5 (no fingerprints or iris) present a special deduplication challenge when they return for full biometric enrollment:

```
┌──────────────────────────────────────────────────────────────┐
│              Child Re-enrollment Deduplication               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  SCENARIO:                                                   │
│  Child enrolled at age 3 (face only, no fingerprints).     │
│  Now age 6 — returns for full biometric enrollment.         │
│                                                              │
│  CHALLENGE:                                                  │
│  ├── ABIS has only a face template from age 3             │
│  ├── Face at age 6 is significantly different              │
│  ├── No fingerprints to match against                      │
│  └── Normal biometric dedup may not find the existing     │
│      record → could create a second UIN                   │
│                                                              │
│  SOLUTION — Parent UIN linkage:                              │
│  ├── At age-3 enrollment: parent UIN linked to child record│
│  ├── At age-6 re-enrollment:                               │
│  │   ├── Officer asks parent UIN or child's UIN           │
│  │   ├── System retrieves existing child record by UIN    │
│  │   ├── Process type: UPDATE_UIN (not NEW_ENROLLMENT)    │
│  │   └── Biometrics added to existing UIN                 │
│  └── If UIN not known: demographic dedup                   │
│      (name + DOB + father's name + mother's name)          │
│      → If found: UPDATE_UIN with supervisor confirmation   │
│      → If not found: new enrollment (rare edge case)       │
│                                                              │
│  ADDITIONAL DEDUP:                                           │
│  After biometrics added, run full ABIS dedup:              │
│  ├── Ensures child was not enrolled twice at different     │
│  │   centers (even with parent UIN linkage strategy)      │
│  └── Provides complete biometric anchor going forward      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 26.13 Deduplication Audit Trail

Every deduplication decision — including the ABIS scores, candidate list, and any supervisor decisions — must be permanently recorded:

```
┌──────────────────────────────────────────────────────────────┐
│              Deduplication Audit Log Entry                   │
└──────────────────────────────────────────────────────────────┘

ABIS JOB COMPLETION LOG:
{
  "eventId":       "EVT-20250115-DEDUP-001",
  "eventType":     "DEDUPLICATION_COMPLETED",
  "timestamp":     "2025-01-15T11:45:00Z",
  "packetId":      "PKT-20250115-001",
  "abisJobId":     "ABIS-JOB-20250115-001",
  "processingTime_sec": 47,
  "databaseSize":  1250000,
  "candidatesFound":    2,
  "candidates": [
    {
      "rank":         1,
      "referenceId":  "UIN-987654321098",
      "fusionScore":  91.7,
      "fingerprintScore": 93.4,
      "irisScore":    88.7,
      "faceScore":    87.2
    },
    {
      "rank":         2,
      "referenceId":  "UIN-111222333444",
      "fusionScore":  31.2,
      "fingerprintScore": 28.1,
      "irisScore":    35.3,
      "faceScore":    30.2
    }
  ],
  "decision":      "DEDUP_FLAGGED",
  "flaggedCandidate":"UIN-987654321098",
  "threshold":     75.0,
  "traceId":       "abc-123-xyz"
}

SUPERVISOR DECISION LOG:
{
  "eventId":       "EVT-20250115-SUP-001",
  "eventType":     "DEDUP_SUPERVISOR_DECISION",
  "timestamp":     "2025-01-15T14:22:00Z",
  "packetId":      "PKT-20250115-001",
  "supervisorId":  "SUP-001",
  "decision":      "CONFIRM_DUPLICATE",
  "flaggedUIN":    "UIN-987654321098",
  "comment":       "Biometric match confirmed. Face photos
                    clearly show same person. Name discrepancy
                    likely transliteration variant.
                    New enrollment rejected.",
  "outcome":       "ENROLLMENT_REJECTED",
  "traceId":       "abc-123-xyz"
}
```

---

## 26.14 AWS Architecture for Duplicate Detection

```
┌──────────────────────────────────────────────────────────────┐
│              Duplicate Detection — AWS Architecture          │
└──────────────────────────────────────────────────────────────┘

Quality Check PASSED
         │
         ▼
Deduplication Service (ECS Fargate)
         │
         ├──► S3: retrieve biometric templates
         │    (fingerprint, iris, face from nbis-templates)
         │
         ├──► DynamoDB GSI: demographic pre-check
         │    (name + DOB quick lookup)
         │
         ├──► S3 (nbis-abis-probe): upload probe package
         │    (templates formatted for ABIS)
         │
         └──► SQS (nbis-abis-requests): send job
              {jobId, probeRef, galleryRef}
                       │
                       ▼
              ABIS System (external vendor)
              ├── Polls SQS queue
              ├── Retrieves probe from S3
              ├── Runs 1:N search (5s – 30min)
              └── Posts result to SQS
                       │
                       ▼
              SQS (nbis-abis-results)
                       │
                       ▼
              Deduplication Result Processor (Lambda)
              │
              ├── Parse ABIS result
              ├── Apply threshold logic
              ├── Update DynamoDB (enrollment status)
              ├── Store candidate list (DynamoDB)
              │
              ├── DEDUP_CLEARED:
              │   └── SQS (nbis-uin-assignment.fifo)
              │
              └── DEDUP_FLAGGED:
                  ├── SQS (nbis-supervisor-review)
                  ├── SNS: supervisor notification
                  └── CloudWatch: flag rate metric

CLOUDWATCH METRICS:
  ├── dedup_processing_time_seconds (per ABIS job)
  ├── dedup_flag_rate (% flagged — target < 2%)
  ├── dedup_clear_rate (% cleared — target > 98%)
  ├── abis_queue_depth (alarm: > 1000 → ABIS too slow)
  └── abis_dlq_depth (alarm: > 0 → P1 alert)
```

---

## 26.15 MOSIP Reference

```
┌──────────────────────────────────────────────────────────────┐
│              MOSIP Duplicate Detection Reference             │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  MOSIP ABIS integration stages in Registration Processor:   │
│                                                              │
│  Stage: Demo Dedupe                                         │
│  └── Lightweight demographic dedup before ABIS             │
│                                                              │
│  Stage: ABIS Handler                                        │
│  ├── Sends INSERT request: add probe to ABIS gallery       │
│  │   (registers new biometrics in ABIS index)              │
│  ├── Sends IDENTIFY request: 1:N search                    │
│  └── Receives: candidate list + scores                     │
│                                                              │
│  MOSIP ABIS middleware interface spec:                      │
│  ├── REQUEST types: INSERT, IDENTIFY, DELETE               │
│  ├── RESPONSE: referenceId + analytics info                │
│  ├── Transport: Kafka / SQS (configurable)                 │
│  └── Defined in: MOSIP ABIS API specification (GitHub)     │
│                                                              │
│  MOSIP supports multiple ABIS instances:                    │
│  ├── PRIMARY ABIS: main dedup (fingerprint + iris)        │
│  ├── SECONDARY ABIS: face dedup (optional, parallel)       │
│  └── Scores merged for final decision                      │
│                                                              │
│  MOSIP dedup threshold configuration:                       │
│  └── abis.threshold.score = 75 (configurable per country) │
│                                                              │
│  Countries using MOSIP ABIS integration:                    │
│  ├── Philippines: Aware ABIS + MOSIP                       │
│  ├── Morocco: NEC ABIS + MOSIP                             │
│  └── Ethiopia: Idemia ABIS + MOSIP (planned)              │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 26.16 Key Terms

| Term | Definition |
|------|-----------|
| **Duplicate detection** | 1:N biometric search to confirm a new enrollee does not already exist in the registry |
| **ABIS** | Automated Biometric Identification System — external engine performing 1:N dedup |
| **1:N search** | Comparing one probe against all N enrolled records in the database |
| **Probe** | The new biometric templates being checked for duplicates |
| **Gallery** | The complete set of enrolled biometric templates in the ABIS index |
| **Fusion score** | Combined match score across multiple biometric modalities (0–100) |
| **Scaled score** | ABIS output normalized to 0–100 for consistent threshold application |
| **DEDUP_FLAGGED** | Outcome when ABIS finds a candidate above the duplicate threshold |
| **DEDUPLICATION_CLEARED** | Outcome when no candidate above threshold is found — proceed to UIN |
| **Candidate** | An enrolled record that the ABIS identifies as a potential match |
| **Flag threshold** | Score above which a candidate triggers a supervisor review |
| **Multimodal fusion** | Combining fingerprint + iris + face scores for higher accuracy |
| **Demo dedupe** | Quick demographic pre-check before expensive ABIS biometric search |
| **False positive** | ABIS flags two different people as duplicates (supervisor must clear) |
| **False negative** | ABIS misses a genuine duplicate (allows same person to get two UINs) |
| **ABIS queue interface** | MOSIP's queue-based contract between NBIS and ABIS vendor |
| **INSERT request** | ABIS command to add a new biometric to the searchable gallery |
| **IDENTIFY request** | ABIS command to search the gallery for matches to a probe |
| **Hamming distance** | Iris match metric — lower = more similar (opposite of match score) |
| **Visibility timeout** | SQS setting for ABIS queue — set longer than max ABIS search time |

---

## 26.17 Key Takeaways

- **Biometric deduplication is the only reliable way to prevent duplicate identities** — names change, are misspelled, and are transliterated differently. Only biometrics are the same regardless of the name given.
- **ABIS is always external to NBIS** — MOSIP defines the queue-based interface; the country selects and contracts their own ABIS vendor. The NBIS never performs the 1:N matching itself.
- **The queue-based interface is a deliberate security design** — ABIS vendors never have direct database access. They receive templates, search their index, and return scores. They never see citizen demographics.
- **Multimodal fusion dramatically improves dedup accuracy** — a fusion score from fingerprint + iris + face has EER approaching 0.00001%. Relying on fingerprint alone risks misses for citizens with worn ridges.
- **False positives are expected and must be handled gracefully** — approximately 1 in 10,000 pairs of different people will have biometric similarity above the flag threshold. The supervisor review queue exists for exactly this reason.
- **The flag threshold is a critical operational parameter** — too low and supervisors are overwhelmed with false positives. Too high and genuine duplicates slip through. Calibrate based on your ABIS vendor's FMR/FNMR curves at your specific database size.
- **Demographic pre-dedup is a performance optimization, not a substitute** — it saves ABIS compute on obvious cases but cannot replace biometric dedup. The same person with different name spellings will pass demographic dedup but be caught by biometrics.
- **Child re-enrollment requires a special workflow** — children enrolled without fingerprints or iris (age < 5) cannot be biometrically matched when they return at age 5+. Parent UIN linkage solves this. Design this from day one.

---

## 26.18 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 27 | Approval Workflow — supervisor review portal, manual decisions, escalation |
| Chapter 33 | Matching Algorithms — minutiae matching, Hamming distance, cosine similarity |
| Chapter 34 | False Match Rate — FAR, threshold calibration, ROC curves |

---

*Chapter 26 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*
