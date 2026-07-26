# Chapter 23 — Signature Capture

> **Part III — Enrollment**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 23.1 What is Signature Capture?

**Signature capture** is the process of recording a person's handwritten signature using a digital signature pad during enrollment. Unlike fingerprint, iris, and face — which capture physical body characteristics — the signature captures a **behavioral biometric**: the unique way a person writes their name or mark.

In an NBIS, the signature serves two distinct purposes:

```
┌──────────────────────────────────────────────────────────────┐
│              Two Roles of Signature in NBIS                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. LEGAL CONSENT RECORD                                     │
│     The signature is the citizen's legal acknowledgment     │
│     that they consent to the enrollment, data collection,   │
│     and use of their biometrics.                            │
│     → Stored as legal evidence of consent                   │
│     → Required by most national data protection laws        │
│     → NOT used for biometric authentication                 │
│                                                              │
│  2. IDENTITY ELEMENT (secondary)                             │
│     The signature is printed on the physical ID card        │
│     as a visual identity element alongside the photo.       │
│     → Used by officers/merchants for visual verification    │
│     → NOT a primary authentication modality in NBIS        │
│     → Behavioral biometric for supplementary use           │
│                                                              │
│  IMPORTANT DISTINCTION:                                      │
│  Unlike fingerprint and iris, signature in most NBIS is     │
│  NOT used for automated 1:1 biometric verification.         │
│  It is primarily a legal and visual identity element.       │
│  Dynamic signature verification (behavioral biometric)      │
│  exists but is rarely deployed in national ID systems.      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 23.2 Static vs Dynamic Signature

There are two types of digital signature capture. Understanding the difference is critical for NBIS design:

```
┌──────────────────────────────────────────────────────────────┐
│              Static vs Dynamic Signature                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  STATIC SIGNATURE (image only):                              │
│  ├── What is captured: a digital image of the signature     │
│  ├── Data stored: JPEG or PNG image                         │
│  ├── Use in NBIS: consent record + ID card printing        │
│  ├── Verification: visual inspection only                   │
│  └── Most NBIS deployments use this                        │
│                                                              │
│  DYNAMIC SIGNATURE (behavioral biometric):                   │
│  ├── What is captured:                                      │
│  │   ├── Signature image (same as static)                  │
│  │   ├── Pen pressure at each point (0–1024 levels)        │
│  │   ├── Pen speed at each point (pixels/second)           │
│  │   ├── Pen angle (azimuth + altitude)                    │
│  │   ├── Stroke sequence (order of pen lifts)              │
│  │   └── Total signing time (milliseconds)                 │
│  ├── Data stored: time-series of (x, y, pressure, time)   │
│  ├── Use in NBIS: behavioral biometric authentication       │
│  │   (rare in national ID — more common in banking)        │
│  └── Verification: algorithmic match on dynamics           │
│                                                              │
│  WHY STATIC IS THE NBIS DEFAULT:                             │
│  ├── Simpler infrastructure                                 │
│  ├── Lower cost signature pads                             │
│  ├── Dynamic verification is not mature enough for         │
│  │   high-stakes national ID use (higher EER than          │
│  │   fingerprint or iris)                                  │
│  └── Legal consent function is served by static image      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 23.3 Signature Capture Equipment

```
┌──────────────────────────────────────────────────────────────┐
│              Signature Capture Devices for NBIS              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ENTRY-LEVEL SIGNATURE PAD (static capture):                 │
│  ├── Resolution: 500 LPI (lines per inch) or higher        │
│  ├── Active area: minimum 75mm × 45mm                      │
│  ├── Connectivity: USB                                      │
│  ├── Pen: passive electromagnetic (no battery)             │
│  ├── Pressure levels: 256 minimum                          │
│  ├── Cost: USD 100–300                                     │
│  └── Examples: Wacom STU-300, Signotec Sigma              │
│                                                              │
│  MID-RANGE (static + dynamic):                               │
│  ├── Resolution: 2540 LPI                                  │
│  ├── Active area: 100mm × 60mm                             │
│  ├── Pressure levels: 1024                                 │
│  ├── Speed: 200+ points/second                             │
│  ├── Angle detection: azimuth + altitude                   │
│  ├── Cost: USD 300–800                                     │
│  └── Examples: Wacom STU-540, Signotec Omega              │
│                                                              │
│  HIGH-END (full biometric dynamic):                          │
│  ├── Resolution: 2540 LPI                                  │
│  ├── Pressure levels: 2048                                 │
│  ├── Speed: 500+ points/second (captures fast signatures)  │
│  ├── Biometric engine: built-in verification SDK           │
│  ├── Display: LCD shows what is being signed               │
│  ├── Cost: USD 800–2000                                    │
│  └── Examples: Wacom DTU series, StepOver naturaSign      │
│                                                              │
│  MOSIP COMPATIBILITY:                                        │
│  ├── Signature pad connected to Registration Client        │
│  ├── SDK: device manufacturer's SDK                        │
│  ├── NOT a MOSIP MDS device (signature is not biometric   │
│  │   in the same sense as fingerprint/iris/face)          │
│  └── Data flows directly into registration packet          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 23.4 Signature Capture Workflow

```
┌──────────────────────────────────────────────────────────────┐
│              Signature Capture Step-by-Step                  │
└──────────────────────────────────────────────────────────────┘

BEFORE CAPTURE:
  ├── Officer confirms citizen has reviewed their data
  ├── Consent statement read aloud or displayed on screen:
  │   "I, [NAME], confirm that the information provided
  │    is accurate and I consent to the collection and
  │    use of my biometric data for identity purposes
  │    as defined by [National ID Authority] under
  │    [National Data Protection Law]."
  └── Citizen asked: "Do you agree and wish to proceed?"

CAPTURE PROCESS:

  STEP 1: Signature pad activated
  ├── Blank signing area displayed on pad screen (if LCD)
  ├── OR: on Registration Client screen with pad active
  └── "Please sign in the box below" instruction shown

  STEP 2: Citizen signs
  ├── Citizen picks up stylus
  ├── Signs their usual signature on the pad
  ├── Signature appears on screen in real-time (preview)
  └── Citizen can clear and re-sign if not satisfied

  STEP 3: Quality check (automated)
  ├── Minimum stroke count: ≥ 3 strokes
  │   (prevents accepting a single dot or line)
  ├── Minimum covered area: ≥ 5% of signing region
  │   (prevents nearly empty signature)
  ├── Duration: ≥ 0.5 seconds
  │   (prevents accidental touch capture)
  └── Image not blank: at least some ink detected

  STEP 4: Officer review
  ├── Officer sees signature preview
  ├── "Is this your signature?" — citizen confirms
  ├── Citizen not satisfied → clear and re-sign
  └── Officer not satisfied (too simple/illegible for ID
      card) → encourage re-sign (cannot force)

  STEP 5: Confirm + store
  ├── Citizen: "Yes, that is my signature"
  ├── Officer clicks: CONFIRM
  └── Signature stored in enrollment packet

SPECIAL CASES:
  ├── Citizen cannot sign (illiteracy):
  │   └── Fingerprint impression on consent form
  │       (ink, not digital — physical record)
  │       Flag: SIGNATURE_FINGERPRINT_SUBSTITUTE
  │
  ├── Citizen cannot sign (disability / amputation):
  │   └── Verbal consent witnessed by officer
  │       Officer signs as witness
  │       Flag: SIGNATURE_OFFICER_WITNESS
  │
  └── Minor (child):
      └── Parent / guardian signs on behalf of child
          Guardian's UIN recorded as signatory
          Flag: SIGNATURE_GUARDIAN
```

---

## 23.5 Signature Quality Standards

```
┌──────────────────────────────────────────────────────────────┐
│              Signature Quality Requirements                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  IMAGE QUALITY (for ID card printing):                       │
│  ├── Resolution: minimum 500 DPI                           │
│  ├── Dimensions: minimum 300×100 pixels at 500 DPI         │
│  │   (matches ID card signature strip dimensions)          │
│  ├── Color: black ink on white background                  │
│  ├── Anti-aliased: smooth lines (not pixelated)           │
│  └── No noise: clean capture (no stray marks)             │
│                                                              │
│  CONTENT QUALITY:                                            │
│  ├── Stroke count ≥ 3 (complex enough to be a signature)  │
│  ├── Total signature area ≥ 5% of pad area               │
│  ├── Aspect ratio: not excessively tall or wide           │
│  └── Duration ≥ 0.5 seconds (deliberate, not accidental)  │
│                                                              │
│  FOR DYNAMIC CAPTURE (if deployed):                          │
│  ├── Minimum points: 50 (captures enough behavior data)    │
│  ├── Maximum duration: 30 seconds                          │
│  │   (longer suggests hesitation, copy-attempt)           │
│  └── Pressure range: must use > 2 pressure levels         │
│      (flat pressure = unnatural, possible forgery)        │
│                                                              │
│  WHAT IS NOT ENFORCED:                                       │
│  ├── Legibility: signatures do not need to be readable    │
│  ├── Style: any style accepted (print, cursive, symbol)   │
│  ├── Language: any script accepted                        │
│  └── Complexity: simple marks acceptable (some cultures   │
│      use an "X" or simple mark as their legal signature)  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 23.6 Signature Storage Format

```
┌──────────────────────────────────────────────────────────────┐
│              Signature Storage Design                        │
└──────────────────────────────────────────────────────────────┘

STATIC SIGNATURE (image):

  Captured: PNG or TIFF (lossless during capture)
  Stored as: JPEG (compressed for storage)
  Resolution: 500 DPI
  Color mode: Grayscale (1-bit or 8-bit)
  Size: typically 5–20 KB per signature

  Storage paths:
  ├── Consent record:
  │   s3://nbis-consent-records/{uinHash}/signature.enc
  │   (legal archive — restricted access, Object Lock)
  │
  └── ID card print:
      s3://nbis-card-print-data/{uinHash}/signature-print.enc
      (card vendor access — separate bucket)

DYNAMIC SIGNATURE (time-series data, if captured):

  Format: JSON array of points
  {
    "signatureData": {
      "points": [
        { "x": 142, "y": 87,  "p": 312, "t": 0    },
        { "x": 145, "y": 89,  "p": 445, "t": 12   },
        { "x": 151, "y": 94,  "p": 521, "t": 24   },
        ...
      ],
      "strokes": [
        { "start": 0, "end": 47  },
        { "start": 48, "end": 112 },
        { "start": 113, "end": 189 }
      ],
      "duration_ms": 2340,
      "total_points": 190,
      "pad_resolution": "2540_LPI",
      "capture_device": "WACOM_STU540"
    }
  }

  Fields per point:
  ├── x: horizontal position (pixels from left)
  ├── y: vertical position (pixels from top)
  ├── p: pressure (0–1023)
  └── t: timestamp (milliseconds from first point)

  Stored as: encrypted JSON in S3
  Size: ~10–50 KB per dynamic signature (190 points typical)

IN THE ENROLLMENT PACKET:
  {
    "signature": {
      "type":      "STATIC",
      "format":    "JPEG",
      "imageRef":  "s3://nbis-packets/PKT-001/sig.enc",
      "capturedAt":"2025-01-15T10:45:00Z",
      "deviceId":  "SIGPAD-CENTER001-001",
      "strokeCount": 4,
      "quality":   87.3,
      "signedBy":  "CITIZEN"
    }
  }
```

---

## 23.7 Consent Documentation

The signature's primary function in NBIS is legal consent. The consent framework must be carefully designed:

```
┌──────────────────────────────────────────────────────────────┐
│              Consent Documentation Design                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  WHAT CONSENT COVERS:                                        │
│  ├── Collection of biographic data                         │
│  ├── Collection of biometric data                          │
│  │   (fingerprint, iris, face, signature)                  │
│  ├── Storage in national identity repository               │
│  ├── Use for identity verification by relying parties      │
│  │   (with citizen consent per transaction for eKYC)      │
│  ├── Use for deduplication against existing records        │
│  └── Retention period and data lifecycle                   │
│                                                              │
│  CONSENT FORM ELEMENTS:                                      │
│  ├── Citizen's full name (pre-filled from demographics)    │
│  ├── Enrollment date and center                            │
│  ├── Statement of rights:                                  │
│  │   ├── Right to view own data (resident portal)         │
│  │   ├── Right to update data                             │
│  │   ├── Right to lock biometrics                         │
│  │   └── Right to file complaint with regulator           │
│  ├── Signature (digital pad)                               │
│  ├── Officer's name and ID                                 │
│  └── Witness signature (if applicable)                    │
│                                                              │
│  CONSENT IN MULTIPLE LANGUAGES:                              │
│  ├── Form displayed in citizen's preferred language        │
│  ├── Officer reads consent aloud in citizen's language     │
│  ├── Both Arabic and English stored (GCC region)          │
│  └── Translation certificate if third language used        │
│                                                              │
│  CONSENT RECORD STORAGE:                                     │
│  ├── Stored: S3 + KMS (encrypted)                          │
│  ├── Object Lock: COMPLIANCE mode                          │
│  │   (cannot be deleted for retention period)             │
│  ├── Retention: same as identity record lifetime          │
│  └── Access: legal team + data protection authority only   │
│                                                              │
│  WITHDRAWAL OF CONSENT:                                      │
│  ├── Citizen may withdraw consent at any time              │
│  │   (some countries: triggers identity revocation)        │
│  ├── Withdrawal process: Resident Portal or center visit  │
│  ├── Withdrawal stored: new consent record                 │
│  │   (type: WITHDRAWAL, signed, timestamped)              │
│  └── Effect: defined by national law                      │
│      (in most NBIS: identity suspended, not deleted)       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 23.8 Signature on the Physical ID Card

```
┌──────────────────────────────────────────────────────────────┐
│              Signature on Physical ID Card                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  STANDARD ID CARD LAYOUT:                                    │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  [COUNTRY LOGO]    NATIONAL IDENTITY CARD            │   │
│  │                                                      │   │
│  │  [PHOTO]   Name:  Hamza Ahmed Rafique               │   │
│  │            UIN:   XXXX XXXX XXXX                    │   │
│  │            DOB:   15/05/1990                        │   │
│  │            Gender:Male                               │   │
│  │                                                      │   │
│  │  Signature: [SIGNATURE IMAGE]                        │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  SIGNATURE STRIP SPECIFICATIONS:                             │
│  ├── Dimensions: typically 50mm × 15mm on card             │
│  ├── Printed: digital offset or laser printing             │
│  ├── Color: black on white or light background             │
│  ├── Resolution: 600 DPI for print quality                 │
│  └── Protected: UV varnish overlay (tamper evident)        │
│                                                              │
│  SIGNATURE ABSENCE (if citizen cannot sign):                 │
│  ├── Strip shows: "Signature on file"                      │
│  ├── OR: thumbprint image printed instead                  │
│  └── OR: strip left blank (country policy)                │
│                                                              │
│  DIGITAL SIGNATURE ON CHIP:                                  │
│  The chip on a smart ID card stores a CRYPTOGRAPHIC         │
│  digital signature (PKI) — not the handwritten signature.  │
│  These are completely different:                            │
│  ├── Handwritten signature → printed on card surface       │
│  └── Cryptographic signature → stored on chip,            │
│      proves card is genuine (issued by government)         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 23.9 Dynamic Signature Verification (Advanced)

Although most NBIS deployments use signatures only for consent and visual ID, some advanced deployments use dynamic signature as a biometric authentication modality:

```
┌──────────────────────────────────────────────────────────────┐
│              Dynamic Signature Verification                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  HOW IT WORKS:                                               │
│  ├── Enrollment: 3–5 signature samples captured             │
│  │   (multiple samples to build stable behavioral model)   │
│  ├── Feature extraction:                                    │
│  │   ├── Duration (signing time)                           │
│  │   ├── Pressure profile (average, peaks, distribution)   │
│  │   ├── Velocity profile (fast/slow sections)             │
│  │   ├── Stroke count and sequence                         │
│  │   ├── Pen-up/pen-down ratio                            │
│  │   └── Spatial distribution (where pen goes)            │
│  └── Template: statistical model of above features        │
│                                                              │
│  AUTHENTICATION:                                             │
│  ├── Citizen signs again (authentication attempt)          │
│  ├── Features extracted from new signature                 │
│  ├── Compared to enrolled model (DTW or HMM algorithm)    │
│  └── Score above threshold → MATCH                        │
│                                                              │
│  ACCURACY (typical):                                         │
│  ├── EER: 2–5% (much higher than fingerprint/iris)        │
│  ├── NOT suitable as standalone authentication in NBIS     │
│  └── Useful as: additional factor in MFA                  │
│      (something you do — behavioral factor)               │
│                                                              │
│  WHY NOT PRIMARY IN NBIS:                                    │
│  ├── Signatures change over time (age, injury)            │
│  ├── Vary with stress, health, environment                │
│  ├── High EER compared to physiological biometrics        │
│  └── Skilled forgers can replicate signatures             │
│                                                              │
│  POSSIBLE FUTURE USE:                                        │
│  ├── Step-up authentication for high-value transactions    │
│  ├── Additional factor alongside OTP                       │
│  └── Fraud detection (signature change = red flag)        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 23.10 Cultural and Legal Considerations

```
┌──────────────────────────────────────────────────────────────┐
│              Cultural and Legal Considerations               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  SIGNATURE VARIATIONS BY CULTURE:                            │
│  ├── Western: cursive name signature                        │
│  ├── Arabic: stylized calligraphic mark (often complex)    │
│  ├── East Asian: name stamp (hanko/chop) + signature       │
│  ├── Low-literacy populations: "X" mark                    │
│  └── All are legally valid — NBIS must accept all          │
│                                                              │
│  ARABIC SIGNATURE CONSIDERATIONS:                            │
│  ├── Arabic signatures often more complex than Latin       │
│  ├── Written right-to-left (RTL)                           │
│  ├── Signature pad software: must support RTL preview      │
│  └── Signature image stored as-is (no RTL processing)     │
│                                                              │
│  LEGAL STANDING OF DIGITAL SIGNATURE:                        │
│  Most countries have eSignature laws establishing that      │
│  a digitally captured handwritten signature has the same    │
│  legal force as a wet ink signature when:                   │
│  ├── Captured at a witnessed, official enrollment center   │
│  ├── Linked to the specific consent document               │
│  ├── Timestamped with officer witness                      │
│  └── Stored in tamper-evident archive                      │
│                                                              │
│  BAHRAIN EXAMPLE:                                            │
│  ├── Electronic Transactions Law (2002) recognizes         │
│  │   electronic signatures for government transactions     │
│  ├── IGA enrollment signature: legally binding            │
│  └── Stored by NID Authority: admissible in court         │
│                                                              │
│  ILLITERACY ACCOMMODATION:                                   │
│  ├── Bahrain adult literacy: ~97% (high, few exceptions)   │
│  ├── Other NBIS deployments (Ethiopia, rural India):       │
│  │   ├── Fingerprint on physical consent form (ink pad)   │
│  │   ├── Thumbprint scanned and stored digitally          │
│  │   └── Flag: SIGNATURE_THUMBPRINT_SUBSTITUTE            │
│  └── System must support this workflow gracefully          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 23.11 Signature in the NBIS Pipeline

```
┌──────────────────────────────────────────────────────────────┐
│              Signature in the Enrollment Pipeline            │
└──────────────────────────────────────────────────────────────┘

WHERE SIGNATURE FITS IN THE WORKFLOW:

  Demographics captured
         │
         ▼
  Documents scanned and verified
         │
         ▼
  Biometrics captured (fingerprint, iris, face)
         │
         ▼
  ──────────────────────────────
  Consent statement displayed
  Citizen reads / officer reads aloud
  SIGNATURE CAPTURED ← here
  Citizen confirms consent
  ──────────────────────────────
         │
         ▼
  Enrollment packet assembled
  (demographics + documents + biometrics + signature)
         │
         ▼
  Operator signs packet (digital signature)
         │
         ▼
  Packet submitted → back-end pipeline

NOTE ON SEQUENCING:
  Signature is captured LAST (after all biometrics)
  because:
  ├── Citizen has seen all their data on screen
  ├── Citizen can confirm everything is correct
  └── Signature is meaningful consent: "I saw this and agree"
      Not: "I agree to whatever you capture next"
```

---

## 23.12 AWS Architecture for Signature Capture

```
┌──────────────────────────────────────────────────────────────┐
│              Signature Capture — AWS Architecture            │
└──────────────────────────────────────────────────────────────┘

Registration Center (signature pad → Registration Client)
  │
  │ Signature image + dynamic data (if captured)
  │ Stored in enrollment packet
  ▼
API Gateway → Registration Service (ECS)
  │
  ├──► S3 (nbis-enrollment-packets)
  │    Store encrypted enrollment packet
  │    (signature included in packet body)
  │
  └──► After processing (Registration Processor):

Registration Processor (ECS Fargate):
  │
  ├──► Extract signature from decrypted packet
  │
  ├──► Quality validation:
  │    ├── Stroke count ≥ 3?
  │    ├── Minimum area covered?
  │    └── Duration ≥ 0.5 seconds?
  │    Fail → SIGNATURE_QUALITY_FAILED → supervisor
  │
  ├──► Store consent record:
  │    S3 (nbis-consent-records)
  │    ├── Key: /{uinHash}/consent-{timestamp}.enc
  │    ├── SSE-KMS (per-citizen key)
  │    ├── Object Lock: COMPLIANCE (cannot delete)
  │    └── Metadata: { operator, center, timestamp,
  │                    consentVersion, language }
  │
  ├──► Store card print version:
  │    S3 (nbis-card-print-data)
  │    ├── Key: /{uinHash}/signature-print.enc
  │    └── Card vendor IAM role has read access
  │
  ├──► Update DynamoDB:
  │    └── enrollment status → SIGNATURE_CAPTURED
  │
  └──► Audit log (CloudWatch):
       {
         "eventType": "SIGNATURE_CAPTURED",
         "uinHash":   "sha256:...",
         "centerId":  "CENTER-001",
         "operatorId":"OFR-007",
         "type":      "STATIC",
         "quality":   87.3,
         "signedBy":  "CITIZEN",
         "timestamp": "2025-01-15T10:45:00Z"
       }

S3 BUCKET SEPARATION:
  ┌────────────────────────────────────────────────────┐
  │ nbis-consent-records   │ Legal archive             │
  │                        │ Object Lock COMPLIANCE    │
  │                        │ Access: legal team only   │
  ├────────────────────────┼───────────────────────────┤
  │ nbis-card-print-data   │ Card vendor access        │
  │                        │ Deleted after card printed│
  │                        │ Access: credential svc    │
  │                        │ + card vendor only        │
  └────────────────────────┴───────────────────────────┘
```

---

## 23.13 Error Codes for Signature Capture

```
┌──────────────────────────────────────────────────────────────┐
│              Signature Capture Error Codes                   │
├────────────────┬─────────────────────────────────────────────┤
│ Code           │ Meaning                                    │
├────────────────┼─────────────────────────────────────────────┤
│ SIG_001        │ Signature area blank — no strokes detected  │
│ SIG_002        │ Stroke count below minimum (< 3 strokes)   │
│ SIG_003        │ Signature duration too short (< 0.5s)      │
│ SIG_004        │ Signature area too small (< 5% of pad)     │
│ SIG_005        │ Signature pad not connected                 │
│ SIG_006        │ Signature pad driver error                 │
│ SIG_007        │ Consent not confirmed by citizen           │
│ SIG_008        │ Guardian signature required but missing    │
│ SIG_009        │ Signature device timeout                   │
│ SIG_010        │ Consent version mismatch                   │
└────────────────┴─────────────────────────────────────────────┘
```

---

## 23.14 Comparison with Other Biometrics

```
┌──────────────────────────────────────────────────────────────┐
│        Signature vs Other NBIS Biometric Modalities          │
├──────────────────────────┬───────────────────────────────────┤
│                          │                                  │
│  MODALITY COMPARISON:    │                                  │
│                          │                                  │
│  Fingerprint:            │  Iris:                          │
│  Physical biometric      │  Physical biometric             │
│  Permanent               │  Permanent                      │
│  EER: ~0.01%             │  EER: ~0.0001%                  │
│  Primary in NBIS         │  Primary in NBIS                │
│                          │                                  │
│  Face:                   │  Signature:                     │
│  Physical biometric      │  Behavioral biometric           │
│  Changes with age        │  Changes with age/health        │
│  EER: ~0.1%              │  EER: ~2–5%                     │
│  Used in NBIS            │  NOT used for auth in NBIS      │
│                          │  (consent + visual ID only)     │
│                          │                                  │
│  KEY TAKEAWAY:           │                                  │
│  Signature is the ONLY   │                                  │
│  biometric captured at   │                                  │
│  enrollment that is NOT  │                                  │
│  used for automated      │                                  │
│  biometric authentication│                                  │
│  in standard NBIS.       │                                  │
│                          │                                  │
└──────────────────────────┴───────────────────────────────────┘
```

---

## 23.15 MOSIP Reference

```
┌──────────────────────────────────────────────────────────────┐
│              MOSIP Signature Capture Reference               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  MOSIP Registration Client — signature:                     │
│  ├── Signature captured via external signature pad          │
│  ├── Integrated through device SDK (not MDS)               │
│  ├── Signature image stored in enrollment packet           │
│  └── Displayed on screen for citizen + officer review      │
│                                                              │
│  MOSIP biometric attribute:                                 │
│  └── "signature" — stored as biometric in CBEFF format     │
│      (Image type: SIGNATURE, Format: JPEG)                 │
│                                                              │
│  MOSIP consent framework:                                   │
│  ├── Consent screen: configurable per country              │
│  ├── Consent text: stored in MOSIP config (multilingual)  │
│  ├── Consent version: tracked (updates trigger re-consent) │
│  └── Consent record: stored alongside enrollment packet    │
│                                                              │
│  MOSIP countries using signature capture:                   │
│  ├── Philippines: signature on ID card                     │
│  ├── Morocco: signature as consent + ID card element       │
│  └── Ethiopia: thumbprint substitute for low-literacy      │
│                                                              │
│  MOSIP signature in Registration Processor:                 │
│  ├── Signature image passed through quality check          │
│  ├── NOT sent to ABIS (not used for deduplication)        │
│  ├── Stored in ID Repository as:                          │
│  │   document type: PROOF_OF_EXCEPTION (if thumbprint)    │
│  │   OR as part of packet in secure S3 archive            │
│  └── Printed on credential by Credential Service          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 23.16 Key Terms

| Term | Definition |
|------|-----------|
| **Signature capture** | Recording a person's handwritten signature on a digital pad during enrollment |
| **Static signature** | Captures only the signature image — most common in NBIS |
| **Dynamic signature** | Captures image plus pressure, speed, timing — behavioral biometric |
| **Behavioral biometric** | Biometric based on how a person does something (sign, type, walk) |
| **Physiological biometric** | Biometric based on physical body characteristics (fingerprint, iris, face) |
| **Consent record** | Legally binding document proving citizen agreed to identity enrollment |
| **DTW** | Dynamic Time Warping — algorithm for comparing time-series signatures |
| **HMM** | Hidden Markov Model — statistical model used in dynamic signature verification |
| **LPI** | Lines per inch — resolution measurement for signature pads |
| **Stroke** | A continuous pen movement from pen-down to pen-up |
| **Pen-up event** | Moment the stylus leaves the signature pad surface |
| **Pressure levels** | Number of distinct pressure values the pad can detect (256–2048) |
| **Object Lock** | S3 feature preventing consent records from deletion (WORM) |
| **RTL** | Right-to-Left — text/signature direction for Arabic script |
| **Hanko** | Japanese name stamp used in place of or alongside handwritten signature |
| **Wet ink signature** | Traditional physical paper signature |
| **eSignature law** | Legislation establishing legal equivalence of digital and wet ink signatures |
| **Consent withdrawal** | Citizen's right to revoke consent to data collection and use |
| **CBEFF** | Common Biometric Exchange Formats Framework — wraps biometric data |
| **EER** | Equal Error Rate — dynamic signature EER is ~2–5% (much higher than fingerprint/iris) |

---

## 23.17 Key Takeaways

- **Signature in NBIS is primarily a legal consent record, not a biometric authentication tool** — its EER of 2–5% makes it unsuitable as a standalone authentication modality. Its value is legal, not algorithmic.
- **Signature is captured last in the enrollment sequence** — only after the citizen has seen all their captured data on screen. This makes the consent meaningful: the citizen is signing what they can actually see and verify.
- **Static signature is sufficient for most NBIS deployments** — the additional infrastructure cost of dynamic capture is only justified if you intend to use signature as a behavioral biometric authentication factor.
- **The consent record must be stored with Object Lock in COMPLIANCE mode** — it is a legal document. No one — not the system admin, not the Minister — should be able to delete it within the retention period.
- **Never enforce legibility** — a signature does not need to be readable. It needs to be the citizen's mark. Forcing citizens to sign differently than their normal signature undermines the legal value.
- **Accommodate alternatives for citizens who cannot sign** — thumbprint on ink pad, witnessed verbal consent, guardian signature. Every NBIS must have these fallback workflows ready, not as afterthoughts.
- **Consent withdrawal is a legal right** — build the withdrawal workflow into the Resident Portal from day one. Citizens who discover NBIS after enrollment may invoke this right. The system must handle it gracefully.
- **Arabic signatures require RTL-aware UI** — if the signature pad software does not support RTL preview, Arabic signatures may appear mirrored or distorted on screen. Test this before deployment in Arabic-speaking countries.

---

## 23.18 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 24 | Quality Checks — overall multi-modal quality pipeline and rejection handling |
| Chapter 25 | Enrollment Validation — packet validation, data completeness checks |
| Chapter 26 | Duplicate Detection — ABIS integration, dedup workflow, manual review |

---

*Chapter 23 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*
