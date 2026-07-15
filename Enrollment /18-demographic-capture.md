# Chapter 18 — Demographic Capture

> **Part III — Enrollment**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 18.1 What is Demographic Capture?

**Demographic capture** is the process of collecting, validating, and recording the biographical attributes of a person during enrollment — name, date of birth, gender, nationality, address, and contact information.

Demographics are the **human-readable layer** of identity. Where biometrics say "this is the same body", demographics say "this body belongs to Hamza Rafique, born 15 May 1990, nationality Bahraini."

```
┌──────────────────────────────────────────────────────────────┐
│              Demographics vs Biometrics                      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  DEMOGRAPHICS                    BIOMETRICS                  │
│  ─────────────                   ──────────                  │
│  Who you say you are             Who you physically are      │
│  Can change (name, address)      Cannot change (fingerprint) │
│  Entered by operator             Captured by device          │
│  Verified against documents      Verified against ABIS       │
│  Stored as text / structured     Stored as encrypted binary  │
│  Used for: KYC, address          Used for: authentication,   │
│  verification, correspondence    deduplication               │
│                                                              │
│  Together they form a complete identity record.              │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 18.2 Demographic Fields — Complete Reference

Every NBIS captures a core set of demographic fields. Some are mandatory; some are optional. Some are immutable after capture; others can be updated throughout life.

```
┌──────────────────────────────────────────────────────────────┐
│              NBIS Demographic Field Reference                │
├──────────────────┬───────────┬───────────┬───────────────────┤
│ Field            │ Mandatory │ Mutable   │ Source           │
├──────────────────┼───────────┼───────────┼───────────────────┤
│ Full legal name  │ Yes       │ Yes*      │ Birth certificate │
│ Date of birth    │ Yes       │ No        │ Birth certificate │
│ Place of birth   │ Yes       │ No        │ Birth certificate │
│ Gender           │ Yes       │ Yes*      │ Birth certificate │
│ Nationality      │ Yes       │ Yes*      │ Passport / cert  │
│ Father's name    │ Yes       │ No        │ Birth certificate │
│ Mother's name    │ Yes       │ No        │ Birth certificate │
│ Permanent address│ Yes       │ Yes       │ Self-declared    │
│ Current address  │ No        │ Yes       │ Self-declared    │
│ Mobile number    │ Yes       │ Yes       │ Self-declared    │
│ Email address    │ No        │ Yes       │ Self-declared    │
│ Marital status   │ No        │ Yes       │ Self-declared    │
│ Blood group      │ No        │ No        │ Medical record   │
│ Guardian (minor) │ If minor  │ No        │ Guardian ID      │
└──────────────────┴───────────┴───────────┴───────────────────┘

* Requires legal documentation and supervisor approval to change
```

---

## 18.3 Name Capture — The Most Complex Field

Names are deceptively complex. A national ID system must handle names from multiple languages, scripts, transliteration standards, and cultural naming conventions.

### 18.3.1 Name Structure

```
┌──────────────────────────────────────────────────────────────┐
│              Name Field Structure                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  firstName:    "Hamza"           ← given name               │
│  middleName:   "Ahmed"           ← optional middle / patronym│
│  lastName:     "Rafique"         ← family / tribal name     │
│  fullName:     "Hamza Ahmed Rafique"  ← concatenated        │
│                                                              │
│  For Arabic-speaking countries, add:                         │
│  firstNameAr:  "حمزة"                                       │
│  middleNameAr: "أحمد"                                       │
│  lastNameAr:   "رفيق"                                       │
│  fullNameAr:   "حمزة أحمد رفيق"                             │
│                                                              │
│  Why store both?                                             │
│  ├── Legal documents in Arabic (passport, ID card)         │
│  ├── Government portals in Arabic + English                │
│  ├── Matching against international databases (Latin only) │
│  └── eKYC responses: field requested per language          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 18.3.2 Name Validation Rules

```
┌──────────────────────────────────────────────────────────────┐
│              Name Validation Rules                           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Allowed characters (English):                               │
│  ├── Letters A–Z, a–z                                       │
│  ├── Hyphens (Mary-Jane)                                    │
│  ├── Apostrophes (O'Brien)                                  │
│  ├── Spaces (multi-word names)                              │
│  └── Diacritics: é, ü, ñ (for non-English names)           │
│                                                              │
│  NOT allowed:                                                │
│  ├── Numbers (123, dates)                                   │
│  ├── Special characters (@, #, !, %)                        │
│  ├── Leading or trailing spaces                             │
│  └── Double spaces                                          │
│                                                              │
│  Length rules:                                               │
│  ├── First name: 1–50 characters                           │
│  ├── Middle name: 0–50 characters (optional)               │
│  ├── Last name: 1–50 characters                            │
│  └── Full name: max 150 characters                         │
│                                                              │
│  Consistency check:                                          │
│  fullName must match firstName + middleName + lastName       │
│  (prevent data entry inconsistencies)                       │
│                                                              │
│  Cross-field check:                                          │
│  Name on birth certificate must match entered name          │
│  (OCR extracted from scanned document → compare)           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 18.3.3 Transliteration

```
┌──────────────────────────────────────────────────────────────┐
│              Arabic → Latin Transliteration                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Problem:                                                    │
│  Arabic name: محمد                                           │
│  Can be transliterated as:                                   │
│  ├── Muhammad                                               │
│  ├── Mohammed                                               │
│  ├── Mohamed                                                │
│  └── Mohamad                                                │
│                                                              │
│  All are the same person. All match the same Arabic name.   │
│  But they fail exact string matching.                        │
│                                                              │
│  NBIS solution:                                              │
│  ├── Store the Arabic name as the canonical source          │
│  ├── Store ONE official Latin transliteration               │
│  │   (from passport or chosen at enrollment)               │
│  ├── Use phonetic matching for demographic auth             │
│  │   (not exact match — Soundex or Metaphone variants)     │
│  └── Demographic comparison: normalize before matching      │
│      Normalize: lowercase + remove diacritics + trim        │
│                                                              │
│  Example normalization:                                      │
│  "Mohammed " → "mohammed"                                    │
│  "Mohamad"  → "mohamad"                                     │
│  These still do not match → phonetic needed for auth        │
│  Store exact transliteration from official document.        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 18.4 Date of Birth Capture

```
┌──────────────────────────────────────────────────────────────┐
│              Date of Birth — Capture and Validation          │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Storage format: ISO 8601 — YYYY-MM-DD                      │
│  Example: 1990-05-15                                         │
│                                                              │
│  Validation rules:                                           │
│  ├── Cannot be in the future                               │
│  ├── Cannot be more than 150 years ago (sanity check)      │
│  ├── Must be a valid calendar date                         │
│  │   (Feb 30 does not exist → reject)                      │
│  └── Must match birth certificate                          │
│                                                              │
│  UNKNOWN DATE OF BIRTH:                                      │
│  Common in countries with low historical birth registration. │
│  Adults may not know their exact DOB.                       │
│                                                              │
│  Handling:                                                   │
│  ├── Approximate DOB: year only (1975-01-01 as placeholder) │
│  ├── DOB type field: EXACT / APPROXIMATE / UNKNOWN          │
│  ├── Document: reason for approximate DOB in packet        │
│  └── Flag record: DOB_APPROXIMATE = true                   │
│      Affects age calculation, eligibility checks            │
│                                                              │
│  CALENDAR SYSTEM HANDLING:                                   │
│  Some countries use non-Gregorian calendars:                │
│  ├── Hijri (Islamic): Saudi Arabia, Bahrain                │
│  ├── Persian: Iran                                          │
│  └── Ethiopian: Ethiopia                                    │
│                                                              │
│  Best practice:                                              │
│  ├── Store always in Gregorian (ISO 8601) in database      │
│  ├── Display in local calendar in UI                        │
│  ├── Convert at entry time: Hijri input → Gregorian store  │
│  └── Store original calendar DOB in separate field          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 18.5 Gender Capture

```
┌──────────────────────────────────────────────────────────────┐
│              Gender Field Design                             │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Standard values (ISO/IEC 5218):                            │
│  ├── M — Male                                               │
│  ├── F — Female                                             │
│  ├── O — Other (some jurisdictions)                         │
│  └── U — Unknown (historical records, anonymized)          │
│                                                              │
│  Storage: single character (M / F / O / U)                  │
│  Source: birth certificate (legal gender)                   │
│                                                              │
│  Design consideration:                                       │
│  Store the legal gender as on the legal document.           │
│  NBIS is an identity system — not a social registry.        │
│  Changes require legal documentation (court order).         │
│                                                              │
│  Validation:                                                 │
│  ├── Must be one of the allowed values                      │
│  ├── Must match birth certificate or legal document         │
│  └── Changes require supervisor approval + legal doc        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 18.6 Address Capture

```
┌──────────────────────────────────────────────────────────────┐
│              Address Field Structure                         │
└──────────────────────────────────────────────────────────────┘
{
  "permanent": {
    "line1":    "House 320, Road 1234",
    "line2":    "Block 320",
    "city":     "Manama",
    "state":    "Capital Governorate",
    "country":  "BH",              ← ISO 3166-1 alpha-2
    "postalCode":"12345"
  },
  "current": {
    "sameAsPermanent": true
  }
}

VALIDATION RULES:
  ├── Line1: 1–100 characters (required)
  ├── Line2: 0–100 characters (optional)
  ├── City: 1–50 characters (required)
  ├── Country: ISO 3166-1 alpha-2 code (required)
  ├── Postal code: format per country (regex per country)
  └── Current address: required only if different from permanent

ADDRESS STANDARDIZATION:
  ├── Trim whitespace from all fields
  ├── Title-case: "manama" → "Manama"
  ├── Store raw input AND standardized version
  └── Standardized version used for matching + printing

ADDRESS VERIFICATION (optional enhancement):
  Some NBIS deployments integrate with postal authority:
  ├── Validate address exists in national address registry
  ├── Return standardized format from official database
  └── Flag unrecognized addresses for officer review

INTERNATIONAL ADDRESS (embassy / diaspora):
  ├── Country field indicates foreign address
  ├── State/postal code optional (not all countries use)
  └── Store free-text format for non-standard countries
```

---

## 18.7 Contact Information Capture

```
┌──────────────────────────────────────────────────────────────┐
│              Contact Information Capture                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  MOBILE NUMBER:                                              │
│  ├── Format: E.164 standard (+97366XXXXXX)                  │
│  ├── Validation: regex per country code                     │
│  │   Bahrain: ^\+973[136][0-9]{7}$                         │
│  ├── Mandatory: YES (used for OTP delivery)                 │
│  ├── Uniqueness: NOT enforced (family members share)        │
│  └── Verification: OTP sent to number, must confirm        │
│                                                              │
│  EMAIL ADDRESS:                                              │
│  ├── Format: RFC 5322 standard                              │
│  ├── Validation: regex + MX record check                    │
│  ├── Mandatory: NO (not all citizens have email)            │
│  ├── Uniqueness: NOT enforced                               │
│  └── Verification: link sent to email, must click          │
│                                                              │
│  MOBILE VERIFICATION FLOW:                                   │
│  Officer enters mobile: +97366123456                        │
│         │                                                    │
│         ▼ Registration Client sends                          │
│  OTP → citizen's phone: "Your enrollment code: 847293"     │
│         │                                                    │
│         ▼ Citizen tells officer the code                     │
│  Officer enters OTP in Registration Client                  │
│         │                                                    │
│         ▼ verified                                           │
│  Mobile confirmed: mobile_verified = true                   │
│         │                                                    │
│  Why verify at enrollment?                                   │
│  ├── Ensures OTPs reach the right person                   │
│  ├── Prevents officers entering wrong numbers              │
│  └── Confirmed mobile is the citizen's notification channel│
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 18.8 Demographic Data Validation — Complete Pipeline

Every demographic field goes through a multi-layer validation pipeline:

```
┌──────────────────────────────────────────────────────────────┐
│              Demographic Validation Pipeline                 │
└──────────────────────────────────────────────────────────────┘

LAYER 1: CLIENT-SIDE VALIDATION (Registration Client)
  Real-time, as officer types:
  ├── Field type check (string, date, enum)
  ├── Length validation (min/max characters)
  ├── Format validation (regex: phone, email, date)
  ├── Required field check (cannot submit if empty)
  └── Character set check (no invalid characters)

  Purpose: Immediate feedback to officer
  Failure: Field highlighted red, cannot proceed

LAYER 2: CROSS-FIELD VALIDATION (Registration Client)
  After all fields entered, before submission:
  ├── DOB vs age check:
  │   if DOB < 18 years ago → require guardian
  ├── Father/mother name vs gender:
  │   (informational consistency)
  ├── Nationality vs birth country:
  │   (flag if different — not reject)
  ├── fullName = firstName + middleName + lastName
  └── Current address = permanent if sameAsPermanent

  Purpose: Catch logical inconsistencies before submission
  Failure: Warning shown, officer confirms or corrects

LAYER 3: DOCUMENT CROSS-CHECK (Registration Client + Server)
  After document scan:
  ├── OCR extracts: name, DOB, gender from document
  ├── Compare OCR result vs entered data
  ├── Name match: normalized comparison
  │   ├── MATCH → green checkmark
  │   ├── CLOSE  → yellow warning (officer confirms)
  │   └── MISMATCH → red alert (officer must re-check)
  └── DOB match: exact comparison

  Purpose: Prevent data entry errors vs document
  Failure: Officer alerted, must re-examine document

LAYER 4: SERVER-SIDE VALIDATION (Registration Processor)
  After packet received by back-end:
  ├── Re-run all client-side validations
  │   (cannot trust client — defense in depth)
  ├── Database lookups:
  │   ├── Is nationality code valid? (ISO 3166 check)
  │   ├── Is center ID valid and active?
  │   └── Is operator ID valid and not suspended?
  ├── Blacklist check:
  │   Is this name/DOB combination flagged?
  └── Schema version check:
      Is packet schema compatible with processor version?

  Purpose: Authoritative validation (client can be bypassed)
  Failure: Packet rejected, officer notified with reason

LAYER 5: BUSINESS RULE VALIDATION (Registration Processor)
  Domain-specific rules:
  ├── Age-based rules:
  │   ├── Age < 5: no fingerprints required
  │   ├── Age < 18: guardian consent required
  │   └── Age > 100: supervisor approval required (unusual)
  ├── Document completeness:
  │   Is the combination of documents sufficient for
  │   the stated nationality + country of birth?
  └── Duplicate guardian check:
      If minor: is guardian's UIN already in system?

  Purpose: Enforce business policy
  Failure: Packet held in REVIEW queue for supervisor
```

---

## 18.9 Multi-Language Support

NBIS must handle demographic data in multiple languages and scripts:

```
┌──────────────────────────────────────────────────────────────┐
│              Multi-Language Demographic Design               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  CHARACTER ENCODING:                                         │
│  ├── All text fields: UTF-8 (supports all scripts)          │
│  ├── Database: UTF-8 collation                              │
│  └── API: JSON (natively UTF-8)                             │
│                                                              │
│  FIELD LANGUAGE TAGGING:                                     │
│  {                                                           │
│    "fullName": [                                             │
│      { "value": "Hamza Rafique", "lang": "en" },           │
│      { "value": "حمزة رفيق",    "lang": "ar" }             │
│    ]                                                         │
│  }                                                           │
│                                                              │
│  RENDERING:                                                  │
│  Arabic, Hebrew, Persian → right-to-left (RTL) text        │
│  ├── UI: dir="rtl" for Arabic fields                        │
│  ├── PDF: PDFBox RTL support for ID card printing          │
│  └── Smart card: ICAO LDS supports UTF-8 in chip data      │
│                                                              │
│  SORTING AND COMPARISON:                                     │
│  ├── Use locale-aware collation for sorting                │
│  ├── Arabic: sort by root word (complex, use ICU library)  │
│  └── For demographic matching: normalize + compare         │
│      Normalization: lowercase, NFD decompose, strip marks   │
│                                                              │
│  LANGUAGE SUPPORT MATRIX:                                    │
│  ┌───────────────────────────────────┬────────────────────┐  │
│  │ Country                           │ Languages          │  │
│  ├───────────────────────────────────┼────────────────────┤  │
│  │ Bahrain, UAE, Saudi Arabia        │ Arabic + English   │  │
│  │ Philippines                       │ Filipino + English │  │
│  │ Morocco                           │ Arabic + French    │  │
│  │ Ethiopia                          │ Amharic + English  │  │
│  └───────────────────────────────────┴────────────────────┘  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 18.10 Demographic Data Schema — DynamoDB Design

```
┌──────────────────────────────────────────────────────────────┐
│              Identity Record Schema (DynamoDB)               │
└──────────────────────────────────────────────────────────────┘
{
  // Primary key
  "PK":          "UIN#123456789012",
  "SK":          "IDENTITY#v1",

  // Identity metadata
  "uin":         "123456789012",
  "status":      "ACTIVE",
  "recordType":  "CITIZEN",
  "createdAt":   "2025-01-15T12:00:00Z",
  "updatedAt":   "2025-01-15T12:00:00Z",
  "version":     1,

  // Demographics — stored encrypted at field level
  // (KMS envelope encryption per sensitive field)
  "demographics": {
    "names": [
      { "value": "Hamza Ahmed Rafique", "lang": "en",
        "type": "FULL" },
      { "value": "حمزة أحمد رفيق",      "lang": "ar",
        "type": "FULL" }
    ],
    "dob":           "1990-05-15",
    "dobType":       "EXACT",
    "gender":        "M",
    "placeOfBirth":  "Manama, Bahrain",
    "nationality":   "BH",
    "fatherName":    "Ahmed Rafique",
    "motherName":    "Fatima Hassan",
    "address": {
      "permanent": {
        "line1":     "House 320, Road 1234",
        "city":      "Manama",
        "country":   "BH",
        "postalCode":"12345"
      }
    },
    "contact": {
      "mobile":         "+97366XXXXXX",
      "mobileVerified": true,
      "email":          "h@example.com",
      "emailVerified":  true
    }
  },

  // Biometric references (templates in S3)
  "biometricRefs": {
    "fingerprint": "s3://nbis-templates/123456789012/fp.enc",
    "iris":        "s3://nbis-templates/123456789012/iris.enc",
    "face":        "s3://nbis-templates/123456789012/face.enc"
  },

  // Security flags
  "biometricLocked":  false,
  "authAttempts":     0,
  "lastAuthAt":       null,

  // DynamoDB TTL (never expires for active records)
  "ttl": null
}

SECONDARY INDEXES:
  GSI 1: status-index
    PK: status (ACTIVE / SUSPENDED / REVOKED)
    SK: createdAt
    → Query all active citizens (admin reports)

  GSI 2: mobile-index
    PK: mobile (hashed)
    SK: uin
    → Look up UIN by mobile number (OTP flow)

  GSI 3: enrollment-center-index
    PK: centerId
    SK: createdAt
    → Center-level enrollment statistics
```

---

## 18.11 Demographic Update Workflow

After initial enrollment, demographic data can be updated throughout a citizen's life:

```
┌──────────────────────────────────────────────────────────────┐
│              Demographic Update Workflow                     │
└──────────────────────────────────────────────────────────────┘

Citizen logs into Resident Portal
         │
         ▼
Selects: Update Address / Update Mobile / Update Email
         │
         ▼
Authenticates (OTP for contact updates,
               Biometric for name/DOB changes)
         │
         ▼ For LOW-RISK updates (address, mobile, email):
Self-service update:
  ├── Citizen enters new value
  ├── Validates format + verification (OTP to new mobile)
  ├── DynamoDB UpdateItem (address / contact fields)
  ├── Audit log: DEMOGRAPHIC_UPDATED
  └── Confirmation SMS/email sent

         ▼ For HIGH-RISK updates (name, DOB, gender):
Officer-assisted update (requires center visit):
  ├── Citizen attends registration center
  ├── Documents supporting the change (court order, etc.)
  ├── Officer initiates UPDATE_UIN process (MOSIP)
  ├── New enrollment packet created (update type)
  ├── No new biometric capture (unless refresh needed)
  ├── Supervisor approval required
  └── Existing UIN retained — only demographics change

AUDIT TRAIL FOR EVERY UPDATE:
  {
    "eventType":    "DEMOGRAPHIC_UPDATED",
    "uinHash":      "sha256:...",
    "fieldChanged": "address.permanent",
    "changedBy":    "RESIDENT_SELF_SERVICE",
    "changedAt":    "2025-06-10T14:22:00Z",
    "previousValue":"[REDACTED]",    ← old value not stored
    "channel":      "RESIDENT_PORTAL",
    "traceId":      "xyz-123"
  }

  Note: Previous values are NOT stored in audit log
  → Privacy principle: minimal data retention
  → Audit proves THAT a change happened, not WHAT it was
```

---

## 18.12 Data Quality Issues and Handling

```
┌──────────────────────────────────────────────────────────────┐
│              Common Demographic Data Quality Issues          │
├──────────────────────────────┬───────────────────────────────┤
│ Issue                        │ Handling                     │
├──────────────────────────────┼───────────────────────────────┤
│ Inconsistent name spelling   │ Store as on legal document.  │
│ (Mohammed vs Muhammad)       │ Phonetic matching for auth.  │
│                              │ Officer cannot "fix" names.  │
├──────────────────────────────┼───────────────────────────────┤
│ Single name (no family name) │ Store in firstName only.     │
│ (common in some cultures)    │ lastName = null (allowed).   │
│                              │ Flag: SINGLE_NAME = true.    │
├──────────────────────────────┼───────────────────────────────┤
│ Unknown exact DOB            │ Store year only (1975-01-01).│
│                              │ Set dobType = APPROXIMATE.   │
├──────────────────────────────┼───────────────────────────────┤
│ Special characters in name   │ Allow: hyphens, apostrophes, │
│ (O'Brien, Jean-Paul)         │ diacritics (é, ü, ñ).        │
│                              │ Strip on normalization only.  │
├──────────────────────────────┼───────────────────────────────┤
│ Address does not exist in    │ Accept and flag for review.  │
│ address registry             │ Do not block enrollment.     │
├──────────────────────────────┼───────────────────────────────┤
│ Duplicate mobile number      │ Allow (family phones shared).│
│ (family shares one phone)    │ Warn officer, not reject.    │
├──────────────────────────────┼───────────────────────────────┤
│ Name differs from passport   │ Flag for supervisor review.  │
│                              │ Accept with documented reason.│
└──────────────────────────────┴───────────────────────────────┘
```

---

## 18.13 Privacy and Data Minimization

```
┌──────────────────────────────────────────────────────────────┐
│              Privacy Principles for Demographic Data         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  DATA MINIMIZATION:                                          │
│  Collect only what is needed for identity purposes.         │
│  ├── Blood group: optional, many NBIS skip it              │
│  ├── Religion: NOT collected (not needed for identity)      │
│  ├── Political affiliation: NEVER collected                 │
│  └── Ethnicity: NOT collected (discrimination risk)        │
│                                                              │
│  PURPOSE LIMITATION:                                         │
│  Data collected for identity must only be used for          │
│  identity purposes.                                         │
│  ├── Address cannot be sold to commercial companies        │
│  ├── Mobile cannot be used for government marketing        │
│  └── eKYC: release only fields the citizen consents to     │
│                                                              │
│  FIELD-LEVEL ENCRYPTION:                                     │
│  Sensitive fields encrypted individually:                   │
│  ├── Mobile number: AES-256 encrypted at rest               │
│  ├── Email address: AES-256 encrypted at rest               │
│  ├── Address: AES-256 encrypted at rest                    │
│  └── DOB: AES-256 encrypted at rest                        │
│                                                              │
│  Why field-level?                                            │
│  ├── Breach of one field key exposes only that field       │
│  ├── Different access permissions per field               │
│  └── Address team can read address, not mobile            │
│                                                              │
│  DATA RETENTION:                                             │
│  ├── Active citizen: indefinite retention                   │
│  ├── After death/revocation: per national data law         │
│  │   (typically 10–30 years for identity records)          │
│  └── Consent form: same retention as identity record       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 18.14 eKYC — Demographic Data Sharing

When a relying party (bank) calls the eKYC API, they receive a subset of demographic data:

```
┌──────────────────────────────────────────────────────────────┐
│              eKYC Demographic Response                       │
└──────────────────────────────────────────────────────────────┘

Bank calls: POST /v1/ekyc
Request:
{
  "uin": "123456789012",
  "biometricData": "base64...",
  "requestedFields": ["fullName", "dob", "gender", "address"],
  "consentToken": "CITIZEN_CONSENT_TOKEN_xyz"
}

NBIS validates:
  ├── Biometric match ✅
  ├── Consent token valid ✅
  └── Partner authorized for requested fields ✅

NBIS returns signed eKYC packet:
{
  "kycData": {
    "fullName":  "Hamza Ahmed Rafique",
    "dob":       "1990-05-15",
    "gender":    "M",
    "address":   {
      "line1": "House 320, Road 1234",
      "city":  "Manama",
      "country":"BH"
    }
  },
  "responseTime": "2025-01-15T10:30:00Z",
  "transactionId":"TXN-001",
  "signature": "base64-signed-with-nbis-key..."
}

WHAT IS NEVER SHARED via eKYC:
  ❌ Mobile number (privacy — unless citizen consents)
  ❌ Email address (privacy — unless citizen consents)
  ❌ Father's / mother's name (unnecessary for KYC)
  ❌ Biometric templates (NEVER — under any circumstance)
  ❌ UIN history or previous names
  ❌ Authentication history
```

---

## 18.15 MOSIP Reference

```
┌──────────────────────────────────────────────────────────────┐
│              MOSIP Demographic Capture Reference             │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  MOSIP ID Schema:                                           │
│  ├── Configurable JSON schema (country-specific)            │
│  ├── Each field: type, required, validators, labels         │
│  ├── Multi-language fields: stored as value array           │
│  └── Schema versioned: changes tracked                     │
│                                                              │
│  MOSIP ID Schema example field definition:                  │
│  {                                                           │
│    "id": "fullName",                                         │
│    "type": "simpleType",                                     │
│    "required": true,                                         │
│    "inputRequired": true,                                    │
│    "validators": [{                                          │
│      "type": "regex",                                        │
│      "validator": "^[a-zA-Z \\-\\']{1,50}$",               │
│      "langCode": "en"                                        │
│    }],                                                       │
│    "transliteration": true,                                  │
│    "bioAttributes": []                                       │
│  }                                                           │
│                                                              │
│  MOSIP Registration Client (demographic tab):               │
│  ├── Dynamic form: rendered from ID Schema JSON             │
│  ├── Fields appear/hide based on country config             │
│  ├── Real-time validation as officer types                  │
│  └── Pre-fill from pre-registration data                   │
│                                                              │
│  MOSIP demographic matching (ID Authentication):            │
│  ├── Exact match (default)                                  │
│  ├── Partial match (configurable threshold)                 │
│  └── Phonetic match (configurable per language)            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 18.16 Key Terms

| Term | Definition |
|------|-----------|
| **Demographic capture** | Collecting biographical attributes during enrollment |
| **Legal name** | Name as it appears on official documents — the authoritative version |
| **Transliteration** | Converting a name from one script to another (Arabic → Latin) |
| **E.164** | International phone number format standard (+country_code + number) |
| **ISO 3166-1** | Standard for country codes (BH = Bahrain, PH = Philippines) |
| **ISO 8601** | Standard for date format (YYYY-MM-DD) |
| **DOB type** | Whether date of birth is EXACT, APPROXIMATE, or UNKNOWN |
| **UTF-8** | Character encoding supporting all world scripts |
| **RTL** | Right-to-Left — text direction for Arabic, Hebrew, Persian |
| **Field-level encryption** | Encrypting individual database fields rather than entire records |
| **Data minimization** | Collecting only the minimum data necessary for the purpose |
| **Purpose limitation** | Using collected data only for the purpose it was collected |
| **eKYC** | Electronic Know Your Customer — sharing verified attributes with relying parties |
| **Consent token** | Proof that the citizen authorized a specific data sharing transaction |
| **Demographic auth** | Verifying identity by matching submitted attributes against stored record |
| **Phonetic matching** | Comparing names by sound rather than exact spelling |
| **OCR** | Optical Character Recognition — extracting text from scanned documents |
| **NFD** | Unicode Normalization Form D — decomposes characters for consistent comparison |
| **GSI** | Global Secondary Index — DynamoDB index for querying by non-primary key |

---

## 18.17 Key Takeaways

- **Demographics are the human layer of identity** — they change (address, name) while biometrics stay constant. Design the data model to handle both mutable and immutable fields correctly.
- **Name capture is the most complex field** — multiple scripts, transliteration variants, cultural naming conventions, and phonetic matching requirements make names far harder than they appear.
- **Always store DOB in ISO 8601 (YYYY-MM-DD)** — regardless of the local calendar (Hijri, Persian, Ethiopian), convert to Gregorian at entry time and store the standard format. Display in local calendar in the UI.
- **Mobile number verification at enrollment time is mandatory** — an unverified mobile means OTPs go to the wrong person. Verify with a challenge-response at the enrollment workstation.
- **Multi-layer validation is non-negotiable** — client-side for UX, server-side for security. Never trust client validation alone. The server re-runs everything.
- **Field-level encryption protects against partial breaches** — if the address encryption key is compromised, fingerprint templates are unaffected. Different fields, different keys, different risk exposure.
- **Never collect religion, ethnicity, or political data** — it is not needed for identity, it creates discrimination risk, and it violates data minimization principles.
- **eKYC shares only consented fields** — the bank gets name and DOB because the citizen consented. The bank never gets the mobile number, biometrics, or authentication history unless explicitly consented and authorized.

---

## 18.18 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 19 | Document Verification — authenticity checks, OCR, document types |
| Chapter 20 | Fingerprint Capture — device types, NFIQ2, rolled vs slap, exceptions |
| Chapter 21 | Face Capture — ICAO compliance, pose, lighting, liveness |

---

*Chapter 18 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*
