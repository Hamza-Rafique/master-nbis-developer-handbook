# Chapter 24 — Quality Checks

> **Part III — Enrollment**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 24.1 What are Quality Checks?

**Quality checks** are systematic validations applied to every piece of data captured during enrollment — biometrics, demographics, documents, and signatures — to ensure that the data is good enough to serve its purpose reliably for the lifetime of the identity record.

Quality checking is the NBIS's internal quality assurance mechanism. It answers the question: **"Is this data good enough that we can be confident it will work correctly for authentication and deduplication for the next 10–30 years?"**

```
┌──────────────────────────────────────────────────────────────┐
│              Why Quality Checks Are Critical                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  A fingerprint enrolled with NFIQ2 = 35 (below minimum):   │
│  ├── Authentication failure rate: 15–20%                   │
│  ├── Citizen attends center multiple times to re-enroll    │
│  ├── Government spends money on re-enrollment support      │
│  └── Citizen loses trust in the identity system            │
│                                                              │
│  An iris enrolled with quality = 40 (below minimum):       │
│  ├── IrisCode is unreliable                                │
│  ├── Border crossing failures                               │
│  └── Legitimate citizen treated as unknown person          │
│                                                              │
│  A name with OCR cross-check failure accepted anyway:       │
│  ├── Name on ID card does not match civil registry         │
│  ├── Bank cannot complete KYC (name mismatch)              │
│  └── Citizen must go through legal name correction process │
│                                                              │
│  Quality checks at enrollment are CHEAPER than fixing       │
│  problems after UINs are issued. The cost of a 2-minute    │
│  recapture at enrollment is infinitely cheaper than years  │
│  of authentication failures and re-enrollment campaigns.   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.2 Two Phases of Quality Checking

Quality checks happen at two distinct points in the enrollment pipeline:

```
┌──────────────────────────────────────────────────────────────┐
│              Two-Phase Quality Check Design                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PHASE 1: CLIENT-SIDE (at registration center, real-time)   │
│  ──────────────────────────────────────────────────────     │
│  When: immediately after each biometric capture             │
│  Who: Registration Client software                          │
│  Goal: immediate feedback → officer can recapture now       │
│                                                              │
│  What it checks:                                             │
│  ├── NFIQ2 per finger (quick algorithm)                    │
│  ├── Basic face ICAO checks (pose, eyes open)              │
│  ├── Basic iris quality (area, sharpness)                  │
│  ├── Signature stroke count + area                         │
│  └── Required fields present (demographic completeness)    │
│                                                              │
│  Speed: must be < 2 seconds (real-time feedback)           │
│  Accuracy: good enough for immediate guidance              │
│  Algorithm: lightweight, runs on registration workstation  │
│                                                              │
│  PHASE 2: SERVER-SIDE (back-end pipeline, post-submission)  │
│  ──────────────────────────────────────────────────────     │
│  When: after encrypted packet uploaded to S3               │
│  Who: Registration Processor (ECS + Lambda)                 │
│  Goal: authoritative quality determination                  │
│                                                              │
│  What it checks:                                             │
│  ├── Full NFIQ2 (complete algorithm, more accurate)        │
│  ├── Full face quality (all ICAO dimensions)               │
│  ├── Full iris quality (all ISO 29794-6 dimensions)        │
│  ├── OCR cross-check (document vs entered demographics)    │
│  ├── MRZ validation (check digit verification)             │
│  ├── Document authenticity indicators                      │
│  ├── Signature quality (server-side assessment)            │
│  └── Cross-modality consistency checks                     │
│                                                              │
│  Speed: minutes acceptable (async pipeline)                │
│  Accuracy: authoritative — determines pass/fail            │
│  Algorithm: full production-grade algorithms               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.3 Biometric Quality Standards — Summary

```
┌──────────────────────────────────────────────────────────────┐
│              Biometric Quality Thresholds Reference          │
├────────────────────┬──────────────┬───────────┬─────────────┤
│ Modality           │ Standard     │ Minimum   │ Target      │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Fingerprint        │ NFIQ2        │ 40 / 100  │ 60 / 100   │
│ (per finger)       │ (NIST)       │           │            │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Fingerprint        │ NFIQ2        │ Average   │ Average     │
│ (overall set)      │ average      │ ≥ 50      │ ≥ 65       │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Iris (per eye)     │ ISO 29794-6  │ 50 / 100  │ 70 / 100   │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Face (overall)     │ ISO 19794-5  │ 50 / 100  │ 70 / 100   │
│                    │ + ICAO       │           │            │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Face (pose)        │ ICAO 9303    │ ≤ ±5°     │ ≤ ±3°      │
│                    │              │ yaw/pitch/│            │
│                    │              │ roll      │            │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Signature          │ Internal     │ ≥ 3       │ ≥ 5        │
│ (stroke count)     │              │ strokes   │ strokes    │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Document scan      │ Internal     │ ≥ 300 DPI │ ≥ 600 DPI  │
│                    │              │ legible   │            │
└────────────────────┴──────────────┴───────────┴────────────┘
```

---

## 24.4 Fingerprint Quality Check — Deep Dive

```
┌──────────────────────────────────────────────────────────────┐
│              Fingerprint Quality Check Pipeline              │
└──────────────────────────────────────────────────────────────┘

INPUT: 10 fingerprint images (WSQ compressed, 500 PPI)
         │
         ▼
FOR EACH FINGER:

  CHECK 1: Image completeness
  ├── Is the image size correct? (not truncated)
  ├── Is the image format valid WSQ?
  └── Is the capture device certified? (MDS signature check)

  CHECK 2: NFIQ2 quality score
  ├── Run NFIQ2 algorithm on each finger image
  ├── Score 0–100 returned for each finger
  └── Apply thresholds:
      < 20:  UNUSABLE → mandatory recapture
      20–39: POOR     → strongly recommend recapture
      40–59: FAIR     → warn, accept with supervisor
      60–74: GOOD     → accept
      75+:   EXCELLENT → accept

  CHECK 3: Pattern detection
  └── Can a ridge pattern be detected?
      (some images are blank / all-white = pad not touched)

  CHECK 4: Finger position consistency
  └── Does the image match the declared finger position?
      (a thumb image submitted as index finger → flag)
      (uses aspect ratio + pattern type classification)

OUTPUT PER FINGER:
  {
    "finger":     "RIGHT_INDEX",
    "nfiq2Score": 78,
    "quality":    "GOOD",
    "decision":   "ACCEPTED",
    "attempts":   1
  }

OVERALL FINGERPRINT QUALITY DECISION:
  ├── ALL 10 fingers ≥ 40:   PASS
  ├── ≥ 8 fingers ≥ 40:      PASS (2 exceptions documented)
  ├── < 8 fingers ≥ 40:      FAIL → supervisor review
  └── Average NFIQ2 < 50:    WARN → supervisor recommendation

QUALITY REPORT SUMMARY:
  {
    "fingerprintQuality": {
      "totalFingers":     10,
      "passed":           9,
      "failed":           1,
      "failedFingers":    ["RIGHT_RING"],
      "averageNFIQ2":     74.3,
      "overallDecision":  "PASS_WITH_EXCEPTION",
      "exceptionReason":  "RIGHT_RING below threshold after
                          3 attempts — skin condition noted"
    }
  }
```

---

## 24.5 Face Quality Check — Deep Dive

```
┌──────────────────────────────────────────────────────────────┐
│              Face Quality Check Pipeline                     │
└──────────────────────────────────────────────────────────────┘

INPUT: Face image (JPEG/JPEG2000, minimum 600×800px)
         │
         ▼
CHECK 1: Face detection
  ├── Is exactly one face detected in the image?
  ├── Zero faces → reject (no face captured)
  └── Multiple faces → reject (someone else in frame)

CHECK 2: Face size and position (geometric)
  ├── Face bounding box occupies 70–80% of frame height?
  ├── Face centered within ±10% of frame?
  └── Eye-to-eye distance ≥ 90 pixels?

CHECK 3: Pose angles (ICAO compliance)
  ├── Yaw (left-right rotation): ≤ ±5°?
  ├── Pitch (chin up/down): ≤ ±5°?
  └── Roll (tilt): ≤ ±5°?

CHECK 4: Eye quality
  ├── Both eyes detected?
  ├── Left eye open score ≥ 70?
  ├── Right eye open score ≥ 70?
  └── Eyes looking at camera (gaze direction check)?

CHECK 5: Lighting quality
  ├── Overall brightness within acceptable range?
  ├── Both sides of face equally lit (symmetry)?
  ├── No overexposed regions (washed out)?
  └── No underexposed regions (too dark)?

CHECK 6: Sharpness
  ├── Focus score ≥ 70?
  └── No motion blur detected?

CHECK 7: Occlusion
  ├── No mask covering nose/mouth?
  ├── Hair not covering eyes?
  └── No hand or object in front of face?

CHECK 8: Expression
  ├── Mouth closed (or near-closed)?
  └── Neutral expression (no extreme emotion)?

CHECK 9: Background
  └── Background sufficiently uniform?
      (high texture background = may confuse face detection)

COMBINED QUALITY SCORE:
  Weighted average across all dimensions:
  ├── Pose:       30% weight (ICAO critical)
  ├── Eyes:       25% weight (matching critical)
  ├── Lighting:   20% weight
  ├── Sharpness:  15% weight
  └── Occlusion:  10% weight

  Score ≥ 70: PASS
  Score 50–69: WARN → officer prompted to retake
  Score < 50:  FAIL → mandatory retake

FACE QUALITY REPORT:
  {
    "faceQuality": {
      "faceDetected":      true,
      "faceCount":         1,
      "poseAngles": {
        "yaw":   2.3,
        "pitch": -1.1,
        "roll":  0.8
      },
      "eyesOpen":          true,
      "lightingScore":     82.1,
      "sharpnessScore":    88.4,
      "occlusion":         "NONE",
      "overallScore":      84.7,
      "icaoCompliant":     true,
      "decision":          "ACCEPTED"
    }
  }
```

---

## 24.6 Iris Quality Check — Deep Dive

```
┌──────────────────────────────────────────────────────────────┐
│              Iris Quality Check Pipeline                     │
└──────────────────────────────────────────────────────────────┘

INPUT: Two iris NIR images (left + right, 640×480px minimum)
         │
         ▼
FOR EACH EYE:

  CHECK 1: Image validity
  ├── Is image format correct?
  ├── Is image not corrupted?
  └── Is MDS device signature valid?

  CHECK 2: Eye/iris detection
  ├── Is an eye detected in the image?
  ├── Is the iris boundary locatable?
  └── Is the pupil detectable?

  CHECK 3: Usable iris area
  ├── Segment iris: locate pupil + limbus boundaries
  ├── Exclude eyelid + eyelash occlusion regions
  ├── Calculate: usable area / total iris area
  └── Usable area ≥ 50%? (minimum)
      Usable area ≥ 70%? (target)

  CHECK 4: Iris sharpness
  ├── Measure frequency content of iris texture region
  ├── Blurry image: low frequency → low sharpness score
  └── Sharpness score ≥ 70?

  CHECK 5: Pupil dilation ratio
  ├── pupil_diameter / iris_diameter = ratio
  ├── Acceptable range: 0.2 – 0.7
  ├── < 0.2: unusual (very bright light / medical)
  └── > 0.7: very dilated (very dark / medication)

  CHECK 6: Gaze direction
  ├── Is iris centered in the image?
  ├── Off-center iris = subject not looking at camera
  └── Acceptable: < 10% off-center

  CHECK 7: Motion blur
  └── Blur score < 10? (subject was still during capture)

COMBINED IRIS QUALITY SCORE:
  {
    "irisQuality": {
      "leftEye": {
        "usableArea":     73.4,
        "sharpness":      82.1,
        "pupilRatio":     0.42,
        "gazeOffset":     3.2,
        "motionBlur":     1.8,
        "overallScore":   78.6,
        "decision":       "ACCEPTED"
      },
      "rightEye": {
        "usableArea":     68.9,
        "sharpness":      79.3,
        "pupilRatio":     0.44,
        "gazeOffset":     4.1,
        "motionBlur":     2.1,
        "overallScore":   74.2,
        "decision":       "ACCEPTED"
      }
    }
  }
```

---

## 24.7 Demographic Quality Checks

```
┌──────────────────────────────────────────────────────────────┐
│              Demographic Quality Check Pipeline              │
└──────────────────────────────────────────────────────────────┘

FOR EACH DEMOGRAPHIC FIELD:

  CHECK 1: Required field present
  └── Is this field populated?
      Empty required field → reject packet

  CHECK 2: Format validation
  ├── Name: regex — only allowed characters?
  ├── DOB:  ISO 8601 format? valid calendar date?
  ├── Mobile: E.164 format? valid country code?
  └── Email: RFC 5322 format?

  CHECK 3: Value range validation
  ├── DOB: not in future, not > 150 years ago
  ├── Name length: 1–150 characters
  └── Age-based rules:
      ├── Age < 5: no fingerprints expected
      ├── Age < 18: guardian consent required
      └── Age > 100: supervisor approval required

  CHECK 4: Cross-field consistency
  ├── fullName == firstName + middleName + lastName?
  ├── Current address == Permanent if sameAsPermanent?
  └── DOB consistent with stated age (if age also given)?

  CHECK 5: OCR cross-check (document vs entered)
  ├── Name: normalized comparison
  │   ├── Match score ≥ 80%: ✅ PASS
  │   ├── Match score 60–79%: ⚠️ WARN (officer confirm)
  │   └── Match score < 60%: ❌ FAIL (must correct)
  └── DOB: exact match required
      ├── Match: ✅ PASS
      └── Mismatch: ❌ FAIL (officer must re-check document)

  CHECK 6: Blacklist check
  ├── Is this name + DOB combination flagged?
  │   (fraud watchlist, court order, etc.)
  └── Flagged → escalate to supervisor (do not auto-reject)

DEMOGRAPHIC QUALITY REPORT:
  {
    "demographicQuality": {
      "requiredFieldsComplete": true,
      "formatValidation":       "PASSED",
      "ocrCrossCheck": {
        "nameMatchScore":  91.3,
        "nameDecision":    "PASSED",
        "dobMatch":        true,
        "dobDecision":     "PASSED"
      },
      "crossFieldConsistency":  "PASSED",
      "blacklistCheck":         "CLEAR",
      "overallDecision":        "PASSED"
    }
  }
```

---

## 24.8 Document Quality Checks

```
┌──────────────────────────────────────────────────────────────┐
│              Document Quality Check Pipeline                 │
└──────────────────────────────────────────────────────────────┘

FOR EACH SCANNED DOCUMENT:

  CHECK 1: Scan quality
  ├── Resolution ≥ 300 DPI?
  ├── Image not blurry (sharpness score)?
  ├── Image not overexposed / underexposed?
  └── Both sides scanned (if double-sided document)?

  CHECK 2: Document type recognition
  ├── Does the scan match the declared document type?
  │   (passport template vs ID card template)
  └── Can fields be located in expected positions?

  CHECK 3: MRZ validation (if applicable)
  ├── MRZ lines present at bottom of page?
  ├── MRZ parsed successfully?
  ├── Check digits valid for all fields?
  └── Extracted data (name, DOB) matches entered data?

  CHECK 4: Document validity
  ├── Issue date in the past?
  ├── Expiry date in the future? (not expired)
  ├── Issue date < Expiry date? (logical)
  └── Document number format valid for issuing country?

  CHECK 5: Tamper indicators (automated)
  ├── Can OCR extract text cleanly?
  │   (tampered text: different font, color artifacts)
  ├── Are image statistics consistent?
  │   (copy-paste substitution: different JPEG artifacts)
  └── MRZ inconsistency:
      document number in MRZ ≠ printed number → flag

  CHECK 6: OCR confidence
  ├── Per-field confidence score from OCR engine
  ├── Low confidence (< 60): flag for officer re-scan
  └── Overall OCR confidence ≥ 70?

DOCUMENT QUALITY REPORT:
  {
    "documentQuality": {
      "scanResolution":    600,
      "scanSharpness":     88.2,
      "documentType":      "PASSPORT",
      "documentRecognized":true,
      "mrzValid":          true,
      "mrzCheckDigits":    "ALL_VALID",
      "documentExpired":   false,
      "ocrConfidence":     94.1,
      "tamperIndicators":  "NONE",
      "overallDecision":   "PASSED"
    }
  }
```

---

## 24.9 Cross-Modality Consistency Checks

After individual modality checks pass, cross-modal checks verify that the biometrics from different modalities are internally consistent:

```
┌──────────────────────────────────────────────────────────────┐
│              Cross-Modality Consistency Checks               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  CHECK 1: Face-to-document photo match                       │
│  ├── Compare enrolled face vs face in document photo        │
│  ├── Tool: Amazon Rekognition CompareFaces API              │
│  ├── Similarity < 70%: FLAG → supervisor review            │
│  │   (possible: wrong person using genuine document)        │
│  └── Similarity ≥ 70%: PASS                                │
│                                                              │
│  CHECK 2: Age consistency                                    │
│  ├── Estimated age from face (AI model) vs stated DOB      │
│  ├── Difference > 15 years: FLAG (unusual, not reject)     │
│  └── Example: stated DOB = 1960, face age estimate = 35    │
│      → 30-year discrepancy → flag for review               │
│                                                              │
│  CHECK 3: Gender consistency                                 │
│  ├── AI gender estimate from face vs stated gender          │
│  ├── Mismatch: FLAG (informational — not grounds to reject) │
│  └── Note: gender expression ≠ legal gender                │
│      Do NOT reject based on gender estimate mismatch.       │
│      Only flag for human awareness.                         │
│                                                              │
│  CHECK 4: Biometric set completeness                         │
│  ├── Are all expected biometrics present?                   │
│  ├── Exception flags for missing modalities?               │
│  │   (if missing without flag → reject packet)             │
│  └── Are exception flags authorized (supervisor approval)? │
│                                                              │
│  CHECK 5: Duplicate biometric within packet                  │
│  ├── Was the same fingerprint submitted for multiple       │
│  │   different finger positions?                           │
│  │   (fraud: one good finger submitted 10 times)           │
│  ├── Run quick pairwise match on submitted fingers         │
│  └── Any pair matches > 0.7 threshold → FRAUD FLAG        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.10 Quality Check Decision Matrix

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Check Decision Matrix                   │
└──────────────────────────────────────────────────────────────┘

ENROLLMENT PACKET QUALITY DECISION LOGIC:

  IF all biometrics PASS and all demographics PASS
  AND all documents PASS:
     → QUALITY_PASSED → proceed to deduplication

  ELIF minor issues (some biometrics in WARN range):
     → QUALITY_PASSED_WITH_WARNINGS
     → Logged, supervisor informed, proceed to dedup
     → Citizen may be contacted for re-enrollment later

  ELIF one or two biometrics FAIL below hard minimum:
     → QUALITY_REVIEW → supervisor queue
     Supervisor options:
     ├── APPROVE_WITH_EXCEPTION (document reason)
     └── REJECT_FOR_RECAPTURE → notify officer

  ELIF critical failure:
     (no primary document, face quality < 30, all
     fingerprints below 40 without documented exception)
     → QUALITY_REJECTED → packet rejected
     → Officer notified with specific reason codes
     → Citizen must return for re-enrollment

  ELIF fraud indicators:
     (face-document mismatch, duplicate fingers,
     MRZ manipulation, document tamper indicators)
     → FRAUD_FLAG → escalated to security team
     → Not processed further until cleared

DECISION CODES:
  ┌─────────────────────────────────────────────────────┐
  │ Code                    │ Next step                 │
  ├─────────────────────────┼───────────────────────────┤
  │ QUALITY_PASSED          │ → Deduplication           │
  │ QUALITY_WARN            │ → Deduplication + logged  │
  │ QUALITY_REVIEW          │ → Supervisor queue        │
  │ QUALITY_REJECTED        │ → Officer notified        │
  │ FRAUD_FLAG              │ → Security team           │
  └─────────────────────────┴───────────────────────────┘
```

---

## 24.11 Quality Check Metrics and Monitoring

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Metrics Dashboard                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PER REGISTRATION CENTER (daily report):                     │
│                                                              │
│  Fingerprint metrics:                                        │
│  ├── average_nfiq2_score (target: ≥ 65)                    │
│  ├── below_minimum_rate (target: < 5%)                     │
│  ├── recapture_rate (target: < 15%)                        │
│  └── exception_rate (target: < 2%)                         │
│                                                              │
│  Face metrics:                                               │
│  ├── average_face_quality_score (target: ≥ 70)             │
│  ├── icao_compliance_rate (target: > 95%)                  │
│  ├── recapture_rate (target: < 10%)                        │
│  └── liveness_fail_rate (target: < 1%)                    │
│                                                              │
│  Iris metrics:                                               │
│  ├── average_iris_quality_score (target: ≥ 70)             │
│  ├── usable_area_average (target: ≥ 70%)                   │
│  └── recapture_rate (target: < 10%)                        │
│                                                              │
│  Demographic metrics:                                        │
│  ├── ocr_mismatch_rate (target: < 3%)                      │
│  ├── format_error_rate (target: < 1%)                      │
│  └── document_rejection_rate (target: < 5%)               │
│                                                              │
│  CLOUDWATCH ALARMS:                                          │
│  ├── average_nfiq2 < 50 for 1 hour at any center          │
│  │   → Alert: equipment issue / officer training need      │
│  ├── icao_compliance_rate < 80% for 1 hour                │
│  │   → Alert: camera misconfigured / lighting problem      │
│  ├── liveness_fail_rate > 5% for 30 min                   │
│  │   → Alert: possible spoofing attempt                   │
│  ├── ocr_mismatch_rate > 10% for 1 hour                   │
│  │   → Alert: officer entering data incorrectly           │
│  └── fraud_flag_rate > 1% for 1 day                       │
│      → Alert: investigate center for organized fraud      │
│                                                              │
│  QUALITY HEATMAP (weekly management report):                 │
│  Shows per-center quality scores on a map:                  │
│  ├── GREEN:  All metrics at target                         │
│  ├── YELLOW: 1–2 metrics below target (investigate)        │
│  └── RED:    Multiple metrics below minimum (intervene)    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.12 Quality Feedback Loop

Quality checks are only useful if they drive improvement. A feedback loop connects quality metrics back to operator training and equipment maintenance:

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Improvement Feedback Loop               │
└──────────────────────────────────────────────────────────────┘

STEP 1: Detect quality issue
  CloudWatch alarm:
  CENTER-005 average NFIQ2 < 50 for 3 consecutive days
         │
         ▼
STEP 2: Investigate
  Operations team reviews:
  ├── Which officers have low NFIQ2 captures?
  ├── Is it all fingers or specific fingers?
  ├── Is it one officer or all officers at center?
  └── When did it start? (sudden drop = equipment issue)
         │
         ▼
STEP 3: Root cause
  Possible causes:
  ├── Dirty platen (clean with alcohol → immediate fix)
  ├── Damaged platen (scratched → replace device)
  ├── Officer technique (rolling too fast → training)
  ├── Population factor (many laborers → adjust process)
  └── Algorithm calibration (quality SDK update needed)
         │
         ▼
STEP 4: Remediation
  ├── Equipment: schedule maintenance / replacement
  ├── Training: targeted session for specific officers
  ├── Process: update guidance for specific populations
  └── Software: update quality SDK / thresholds
         │
         ▼
STEP 5: Verify improvement
  Monitor next 7 days:
  ├── NFIQ2 rising back to target? → remediation worked
  └── Still low? → escalate investigation
         │
         ▼
STEP 6: Document and close
  Incident report:
  ├── What was the issue?
  ├── Root cause identified?
  ├── What was the fix?
  └── How many enrollments affected?
      (these may need re-enrollment outreach campaign)
```

---

## 24.13 Quality Check Error Codes

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Check Error Codes                       │
├────────────────┬─────────────────────────────────────────────┤
│ Code           │ Meaning                                    │
├────────────────┼─────────────────────────────────────────────┤
│ QC_FP_001      │ Fingerprint NFIQ2 below hard minimum (< 40)│
│ QC_FP_002      │ Fingerprint image blank / not captured     │
│ QC_FP_003      │ Fewer than 8 fingers above minimum quality │
│ QC_FP_004      │ Duplicate finger detected in set          │
│ QC_FP_005      │ Finger position inconsistent with image   │
│ QC_FC_001      │ No face detected in image                 │
│ QC_FC_002      │ Multiple faces detected                   │
│ QC_FC_003      │ Face pose out of ICAO range               │
│ QC_FC_004      │ Eyes not open / not visible               │
│ QC_FC_005      │ Face quality score below minimum (< 50)   │
│ QC_FC_006      │ Face-document photo mismatch              │
│ QC_IR_001      │ Iris not detected in image                │
│ QC_IR_002      │ Usable iris area below minimum (< 50%)    │
│ QC_IR_003      │ Iris sharpness below threshold            │
│ QC_IR_004      │ Pupil dilation ratio out of range        │
│ QC_DM_001      │ Required demographic field missing        │
│ QC_DM_002      │ Field format validation failed            │
│ QC_DM_003      │ Name OCR mismatch with document          │
│ QC_DM_004      │ DOB mismatch with document               │
│ QC_DC_001      │ Document scan below minimum resolution    │
│ QC_DC_002      │ MRZ check digit failure                  │
│ QC_DC_003      │ Document expired                         │
│ QC_DC_004      │ OCR confidence below threshold           │
│ QC_CM_001      │ Age estimated from face inconsistent      │
│ QC_CM_002      │ Cross-modality fraud indicator detected   │
└────────────────┴─────────────────────────────────────────────┘
```

---

## 24.14 Complete Quality Check Flow — End to End

```
┌──────────────────────────────────────────────────────────────┐
│              Complete Quality Check Flow                     │
└──────────────────────────────────────────────────────────────┘

ENROLLMENT PACKET RECEIVED (S3 upload)
         │
         ▼
SQS → Registration Processor (ECS Fargate)
         │
    ┌────┴──────────────────────────────────────────────┐
    │                                                   │
    ▼                                                   ▼
MDS Signature Verify                       Packet Structure Check
(device cert validation)                   (all fields present?)
         │                                             │
         └──────────────┬────────────────────────────┘
                        │
                        ▼
              Decrypt packet (KMS)
                        │
          ┌─────────────┼───────────────────┐
          │             │                   │
          ▼             ▼                   ▼
  Fingerprint      Face Quality         Iris Quality
  Quality Check    Check                Check
  (NFIQ2 × 10)    (ICAO + ISO          (ISO 29794-6)
                   19794-5)
          │             │                   │
          └─────────────┼───────────────────┘
                        │
                        ▼
              Demographic Quality Check
              (format + OCR cross-check)
                        │
                        ▼
              Document Quality Check
              (scan + MRZ + OCR confidence)
                        │
                        ▼
              Cross-Modality Checks
              (face-doc match + age + fraud)
                        │
                        ▼
              Quality Decision Engine
                        │
         ┌──────────────┼──────────────────────┐
         │              │                      │
         ▼              ▼                      ▼
    PASSED          REVIEW              REJECTED/FRAUD
         │              │                      │
         ▼              ▼                      ▼
  → Deduplication  → Supervisor         → Officer/Security
    (next stage)     queue                 notification
                       │                      │
                       ▼                      ▼
                  Supervisor            DLQ + alarm
                  approves or           + audit log
                  rejects
                       │
                       ▼
               APPROVED → Dedup
               REJECTED → Officer
```

---

## 24.15 AWS Architecture for Quality Checks

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Check AWS Architecture                  │
└──────────────────────────────────────────────────────────────┘

SQS (nbis-enrollment-received)
         │
         ▼
Quality Check Service (ECS Fargate)
         │
         ├──► S3: retrieve decrypted packet contents
         │
         ├──► NFIQ2 SDK container:
         │    Fingerprint quality per finger
         │
         ├──► Amazon Rekognition:
         │    ├── DetectFaces: face quality + ICAO check
         │    └── CompareFaces: face vs document photo
         │
         ├──► Iris Quality SDK container:
         │    ISO 29794-6 quality scoring
         │
         ├──► Amazon Textract:
         │    OCR + document field extraction
         │    MRZ line extraction
         │
         ├──► Custom MRZ validator (Lambda):
         │    Check digit verification
         │
         ├──► DynamoDB:
         │    Update enrollment status
         │    Store quality scores per modality
         │
         ├──► SQS routing:
         │    ├── PASSED → nbis-abis-requests
         │    ├── REVIEW → nbis-supervisor-review
         │    └── REJECTED → nbis-officer-notification
         │
         └──► CloudWatch:
              Publish quality metrics
              Per-center, per-modality, per-officer

QUALITY SCORES STORED IN DYNAMODB:
  DynamoDB table: nbis-enrollment-quality
  PK: packetId
  Attributes:
  ├── fingerprintScores: {finger: score} map
  ├── faceScore: overall + dimension breakdown
  ├── irisScores: {left: score, right: score}
  ├── demographicDecision: PASSED/FAILED
  ├── documentDecision: PASSED/FAILED
  ├── crossModalityFlags: list of flags
  ├── overallDecision: decision code
  └── processingTime: milliseconds
```

---

## 24.16 MOSIP Reference

```
┌──────────────────────────────────────────────────────────────┐
│              MOSIP Quality Check Reference                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  MOSIP Registration Processor quality stages:               │
│                                                              │
│  Stage 1: Packet Receiver                                   │
│  └── Validates packet structure and signature               │
│                                                              │
│  Stage 2: Packet Validator                                  │
│  └── Schema validation, field completeness                  │
│                                                              │
│  Stage 3: Quality Classifier                                │
│  ├── Biometric quality check (NFIQ2, face, iris)           │
│  ├── Configurable quality thresholds (per country)         │
│  └── Pluggable: country replaces default SDK with own      │
│                                                              │
│  Stage 4: Demo Dedupe (demographic pre-check)               │
│  └── Quick demographic match check before ABIS             │
│      (saves ABIS calls for obvious duplicates)             │
│                                                              │
│  MOSIP quality threshold configuration:                     │
│  In MOSIP admin portal:                                     │
│  ├── fingerprint.quality.threshold = 60                    │
│  ├── iris.quality.threshold = 70                           │
│  ├── face.quality.threshold = 70                           │
│  └── Configurable without code change                      │
│                                                              │
│  MOSIP quality failure handling:                            │
│  ├── Failed packet → BIOMETRIC_QUALITY_CHECK_FAILED        │
│  ├── Notification: officer SMS/email (configurable)        │
│  └── Packet: stays in system for supervisor review         │
│                                                              │
│  MOSIP supervisor portal (Registration Supervisor):         │
│  ├── View all packets in REVIEW queue                      │
│  ├── See quality scores + reason for review               │
│  ├── Approve with documented reason                        │
│  └── Reject (triggers re-enrollment notification)         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.17 Key Terms

| Term | Definition |
|------|-----------|
| **Quality check** | Validation that captured data meets minimum standards for reliable use |
| **Client-side check** | Real-time quality check at the enrollment center (immediate feedback) |
| **Server-side check** | Authoritative quality check in the back-end pipeline (post-submission) |
| **NFIQ2** | NIST Fingerprint Image Quality 2 — fingerprint quality standard (0–100) |
| **ISO 29794-6** | International standard for iris image quality metrics |
| **ISO 19794-5** | International standard for face image data and quality |
| **ICAO compliance** | Face image meets ICAO Doc 9303 requirements for identity documents |
| **OCR cross-check** | Comparing OCR-extracted document data with officer-entered demographics |
| **MRZ validation** | Verifying check digits in the passport machine-readable zone |
| **Cross-modality check** | Comparing consistency between different biometric modalities |
| **Face-document match** | Comparing enrolled face photo against document photo using face recognition |
| **Duplicate finger detection** | Checking if same fingerprint submitted for multiple positions |
| **Quality decision engine** | System that combines all check results into a final enrollment decision |
| **QUALITY_PASSED** | All checks passed — proceed to deduplication |
| **QUALITY_REVIEW** | Some issues — human supervisor must decide |
| **QUALITY_REJECTED** | Fundamental failure — enrollment must be redone |
| **FRAUD_FLAG** | Indicators of deliberate fraud detected — escalate to security |
| **Quality heatmap** | Visual map showing per-center quality performance |
| **Recapture rate** | Percentage of biometric captures requiring a second attempt |
| **Feedback loop** | Process connecting quality metrics to training and equipment maintenance |

---

## 24.18 Key Takeaways

- **Quality checks happen twice** — client-side for immediate feedback (officer can recapture now) and server-side for the authoritative decision (after submission). Never rely on client-side alone.
- **Hard minimums are hard for a reason** — below NFIQ2 40 or iris quality 50, the template is genuinely unreliable. Supervisor overrides below these levels must be documented with a specific, auditable reason.
- **Cross-modality checks catch fraud that single-modality checks miss** — a duplicate finger in the same packet, or a face that does not match the passport photo, are caught only when you compare across modalities.
- **Quality metrics are a center management tool** — a center with consistently low NFIQ2 scores has either a dirty platen, a broken scanner, undertrained officers, or a population with occupational wear. The metric tells you something is wrong; investigation reveals which.
- **OCR cross-check is not a duplicate of document verification** — it specifically checks whether what the officer typed matches what the document actually says. Officers make data entry errors. This check catches them before the UIN is issued.
- **Fraud flags are escalations, not automatic rejections** — face-document mismatch could mean a bad passport photo, not a fraudster. Age inconsistency could mean the citizen looks much younger than their DOB. Human review is required before any fraud-flagged packet is rejected.
- **Quality thresholds are configurable** — MOSIP allows country-specific threshold settings without code changes. Different populations have different biometric characteristics. What works in Sweden may not be appropriate for a country with many manual laborers.
- **The quality feedback loop is as important as the check itself** — quality checks without corrective action are just data. Track, alarm, investigate, remediate, and verify improvement.

---

## 24.19 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 25 | Enrollment Validation — packet validation, completeness, business rules |
| Chapter 26 | Duplicate Detection — ABIS integration, 1:N search, dedup workflow |
| Chapter 27 | Approval Workflow — supervisor review, manual decisions, audit trail |

---

*Chapter 24 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*# Chapter 24 — Quality Checks

> **Part III — Enrollment**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 24.1 What are Quality Checks?

**Quality checks** are systematic validations applied to every piece of data captured during enrollment — biometrics, demographics, documents, and signatures — to ensure that the data is good enough to serve its purpose reliably for the lifetime of the identity record.

Quality checking is the NBIS's internal quality assurance mechanism. It answers the question: **"Is this data good enough that we can be confident it will work correctly for authentication and deduplication for the next 10–30 years?"**

```
┌──────────────────────────────────────────────────────────────┐
│              Why Quality Checks Are Critical                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  A fingerprint enrolled with NFIQ2 = 35 (below minimum):   │
│  ├── Authentication failure rate: 15–20%                   │
│  ├── Citizen attends center multiple times to re-enroll    │
│  ├── Government spends money on re-enrollment support      │
│  └── Citizen loses trust in the identity system            │
│                                                              │
│  An iris enrolled with quality = 40 (below minimum):       │
│  ├── IrisCode is unreliable                                │
│  ├── Border crossing failures                               │
│  └── Legitimate citizen treated as unknown person          │
│                                                              │
│  A name with OCR cross-check failure accepted anyway:       │
│  ├── Name on ID card does not match civil registry         │
│  ├── Bank cannot complete KYC (name mismatch)              │
│  └── Citizen must go through legal name correction process │
│                                                              │
│  Quality checks at enrollment are CHEAPER than fixing       │
│  problems after UINs are issued. The cost of a 2-minute    │
│  recapture at enrollment is infinitely cheaper than years  │
│  of authentication failures and re-enrollment campaigns.   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.2 Two Phases of Quality Checking

Quality checks happen at two distinct points in the enrollment pipeline:

```
┌──────────────────────────────────────────────────────────────┐
│              Two-Phase Quality Check Design                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PHASE 1: CLIENT-SIDE (at registration center, real-time)   │
│  ──────────────────────────────────────────────────────     │
│  When: immediately after each biometric capture             │
│  Who: Registration Client software                          │
│  Goal: immediate feedback → officer can recapture now       │
│                                                              │
│  What it checks:                                             │
│  ├── NFIQ2 per finger (quick algorithm)                    │
│  ├── Basic face ICAO checks (pose, eyes open)              │
│  ├── Basic iris quality (area, sharpness)                  │
│  ├── Signature stroke count + area                         │
│  └── Required fields present (demographic completeness)    │
│                                                              │
│  Speed: must be < 2 seconds (real-time feedback)           │
│  Accuracy: good enough for immediate guidance              │
│  Algorithm: lightweight, runs on registration workstation  │
│                                                              │
│  PHASE 2: SERVER-SIDE (back-end pipeline, post-submission)  │
│  ──────────────────────────────────────────────────────     │
│  When: after encrypted packet uploaded to S3               │
│  Who: Registration Processor (ECS + Lambda)                 │
│  Goal: authoritative quality determination                  │
│                                                              │
│  What it checks:                                             │
│  ├── Full NFIQ2 (complete algorithm, more accurate)        │
│  ├── Full face quality (all ICAO dimensions)               │
│  ├── Full iris quality (all ISO 29794-6 dimensions)        │
│  ├── OCR cross-check (document vs entered demographics)    │
│  ├── MRZ validation (check digit verification)             │
│  ├── Document authenticity indicators                      │
│  ├── Signature quality (server-side assessment)            │
│  └── Cross-modality consistency checks                     │
│                                                              │
│  Speed: minutes acceptable (async pipeline)                │
│  Accuracy: authoritative — determines pass/fail            │
│  Algorithm: full production-grade algorithms               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.3 Biometric Quality Standards — Summary

```
┌──────────────────────────────────────────────────────────────┐
│              Biometric Quality Thresholds Reference          │
├────────────────────┬──────────────┬───────────┬─────────────┤
│ Modality           │ Standard     │ Minimum   │ Target      │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Fingerprint        │ NFIQ2        │ 40 / 100  │ 60 / 100   │
│ (per finger)       │ (NIST)       │           │            │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Fingerprint        │ NFIQ2        │ Average   │ Average     │
│ (overall set)      │ average      │ ≥ 50      │ ≥ 65       │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Iris (per eye)     │ ISO 29794-6  │ 50 / 100  │ 70 / 100   │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Face (overall)     │ ISO 19794-5  │ 50 / 100  │ 70 / 100   │
│                    │ + ICAO       │           │            │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Face (pose)        │ ICAO 9303    │ ≤ ±5°     │ ≤ ±3°      │
│                    │              │ yaw/pitch/│            │
│                    │              │ roll      │            │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Signature          │ Internal     │ ≥ 3       │ ≥ 5        │
│ (stroke count)     │              │ strokes   │ strokes    │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Document scan      │ Internal     │ ≥ 300 DPI │ ≥ 600 DPI  │
│                    │              │ legible   │            │
└────────────────────┴──────────────┴───────────┴────────────┘
```

---

## 24.4 Fingerprint Quality Check — Deep Dive

```
┌──────────────────────────────────────────────────────────────┐
│              Fingerprint Quality Check Pipeline              │
└──────────────────────────────────────────────────────────────┘

INPUT: 10 fingerprint images (WSQ compressed, 500 PPI)
         │
         ▼
FOR EACH FINGER:

  CHECK 1: Image completeness
  ├── Is the image size correct? (not truncated)
  ├── Is the image format valid WSQ?
  └── Is the capture device certified? (MDS signature check)

  CHECK 2: NFIQ2 quality score
  ├── Run NFIQ2 algorithm on each finger image
  ├── Score 0–100 returned for each finger
  └── Apply thresholds:
      < 20:  UNUSABLE → mandatory recapture
      20–39: POOR     → strongly recommend recapture
      40–59: FAIR     → warn, accept with supervisor
      60–74: GOOD     → accept
      75+:   EXCELLENT → accept

  CHECK 3: Pattern detection
  └── Can a ridge pattern be detected?
      (some images are blank / all-white = pad not touched)

  CHECK 4: Finger position consistency
  └── Does the image match the declared finger position?
      (a thumb image submitted as index finger → flag)
      (uses aspect ratio + pattern type classification)

OUTPUT PER FINGER:
  {
    "finger":     "RIGHT_INDEX",
    "nfiq2Score": 78,
    "quality":    "GOOD",
    "decision":   "ACCEPTED",
    "attempts":   1
  }

OVERALL FINGERPRINT QUALITY DECISION:
  ├── ALL 10 fingers ≥ 40:   PASS
  ├── ≥ 8 fingers ≥ 40:      PASS (2 exceptions documented)
  ├── < 8 fingers ≥ 40:      FAIL → supervisor review
  └── Average NFIQ2 < 50:    WARN → supervisor recommendation

QUALITY REPORT SUMMARY:
  {
    "fingerprintQuality": {
      "totalFingers":     10,
      "passed":           9,
      "failed":           1,
      "failedFingers":    ["RIGHT_RING"],
      "averageNFIQ2":     74.3,
      "overallDecision":  "PASS_WITH_EXCEPTION",
      "exceptionReason":  "RIGHT_RING below threshold after
                          3 attempts — skin condition noted"
    }
  }
```

---

## 24.5 Face Quality Check — Deep Dive

```
┌──────────────────────────────────────────────────────────────┐
│              Face Quality Check Pipeline                     │
└──────────────────────────────────────────────────────────────┘

INPUT: Face image (JPEG/JPEG2000, minimum 600×800px)
         │
         ▼
CHECK 1: Face detection
  ├── Is exactly one face detected in the image?
  ├── Zero faces → reject (no face captured)
  └── Multiple faces → reject (someone else in frame)

CHECK 2: Face size and position (geometric)
  ├── Face bounding box occupies 70–80% of frame height?
  ├── Face centered within ±10% of frame?
  └── Eye-to-eye distance ≥ 90 pixels?

CHECK 3: Pose angles (ICAO compliance)
  ├── Yaw (left-right rotation): ≤ ±5°?
  ├── Pitch (chin up/down): ≤ ±5°?
  └── Roll (tilt): ≤ ±5°?

CHECK 4: Eye quality
  ├── Both eyes detected?
  ├── Left eye open score ≥ 70?
  ├── Right eye open score ≥ 70?
  └── Eyes looking at camera (gaze direction check)?

CHECK 5: Lighting quality
  ├── Overall brightness within acceptable range?
  ├── Both sides of face equally lit (symmetry)?
  ├── No overexposed regions (washed out)?
  └── No underexposed regions (too dark)?

CHECK 6: Sharpness
  ├── Focus score ≥ 70?
  └── No motion blur detected?

CHECK 7: Occlusion
  ├── No mask covering nose/mouth?
  ├── Hair not covering eyes?
  └── No hand or object in front of face?

CHECK 8: Expression
  ├── Mouth closed (or near-closed)?
  └── Neutral expression (no extreme emotion)?

CHECK 9: Background
  └── Background sufficiently uniform?
      (high texture background = may confuse face detection)

COMBINED QUALITY SCORE:
  Weighted average across all dimensions:
  ├── Pose:       30% weight (ICAO critical)
  ├── Eyes:       25% weight (matching critical)
  ├── Lighting:   20% weight
  ├── Sharpness:  15% weight
  └── Occlusion:  10% weight

  Score ≥ 70: PASS
  Score 50–69: WARN → officer prompted to retake
  Score < 50:  FAIL → mandatory retake

FACE QUALITY REPORT:
  {
    "faceQuality": {
      "faceDetected":      true,
      "faceCount":         1,
      "poseAngles": {
        "yaw":   2.3,
        "pitch": -1.1,
        "roll":  0.8
      },
      "eyesOpen":          true,
      "lightingScore":     82.1,
      "sharpnessScore":    88.4,
      "occlusion":         "NONE",
      "overallScore":      84.7,
      "icaoCompliant":     true,
      "decision":          "ACCEPTED"
    }
  }
```

---

## 24.6 Iris Quality Check — Deep Dive

```
┌──────────────────────────────────────────────────────────────┐
│              Iris Quality Check Pipeline                     │
└──────────────────────────────────────────────────────────────┘

INPUT: Two iris NIR images (left + right, 640×480px minimum)
         │
         ▼
FOR EACH EYE:

  CHECK 1: Image validity
  ├── Is image format correct?
  ├── Is image not corrupted?
  └── Is MDS device signature valid?

  CHECK 2: Eye/iris detection
  ├── Is an eye detected in the image?
  ├── Is the iris boundary locatable?
  └── Is the pupil detectable?

  CHECK 3: Usable iris area
  ├── Segment iris: locate pupil + limbus boundaries
  ├── Exclude eyelid + eyelash occlusion regions
  ├── Calculate: usable area / total iris area
  └── Usable area ≥ 50%? (minimum)
      Usable area ≥ 70%? (target)

  CHECK 4: Iris sharpness
  ├── Measure frequency content of iris texture region
  ├── Blurry image: low frequency → low sharpness score
  └── Sharpness score ≥ 70?

  CHECK 5: Pupil dilation ratio
  ├── pupil_diameter / iris_diameter = ratio
  ├── Acceptable range: 0.2 – 0.7
  ├── < 0.2: unusual (very bright light / medical)
  └── > 0.7: very dilated (very dark / medication)

  CHECK 6: Gaze direction
  ├── Is iris centered in the image?
  ├── Off-center iris = subject not looking at camera
  └── Acceptable: < 10% off-center

  CHECK 7: Motion blur
  └── Blur score < 10? (subject was still during capture)

COMBINED IRIS QUALITY SCORE:
  {
    "irisQuality": {
      "leftEye": {
        "usableArea":     73.4,
        "sharpness":      82.1,
        "pupilRatio":     0.42,
        "gazeOffset":     3.2,
        "motionBlur":     1.8,
        "overallScore":   78.6,
        "decision":       "ACCEPTED"
      },
      "rightEye": {
        "usableArea":     68.9,
        "sharpness":      79.3,
        "pupilRatio":     0.44,
        "gazeOffset":     4.1,
        "motionBlur":     2.1,
        "overallScore":   74.2,
        "decision":       "ACCEPTED"
      }
    }
  }
```

---

## 24.7 Demographic Quality Checks

```
┌──────────────────────────────────────────────────────────────┐
│              Demographic Quality Check Pipeline              │
└──────────────────────────────────────────────────────────────┘

FOR EACH DEMOGRAPHIC FIELD:

  CHECK 1: Required field present
  └── Is this field populated?
      Empty required field → reject packet

  CHECK 2: Format validation
  ├── Name: regex — only allowed characters?
  ├── DOB:  ISO 8601 format? valid calendar date?
  ├── Mobile: E.164 format? valid country code?
  └── Email: RFC 5322 format?

  CHECK 3: Value range validation
  ├── DOB: not in future, not > 150 years ago
  ├── Name length: 1–150 characters
  └── Age-based rules:
      ├── Age < 5: no fingerprints expected
      ├── Age < 18: guardian consent required
      └── Age > 100: supervisor approval required

  CHECK 4: Cross-field consistency
  ├── fullName == firstName + middleName + lastName?
  ├── Current address == Permanent if sameAsPermanent?
  └── DOB consistent with stated age (if age also given)?

  CHECK 5: OCR cross-check (document vs entered)
  ├── Name: normalized comparison
  │   ├── Match score ≥ 80%: ✅ PASS
  │   ├── Match score 60–79%: ⚠️ WARN (officer confirm)
  │   └── Match score < 60%: ❌ FAIL (must correct)
  └── DOB: exact match required
      ├── Match: ✅ PASS
      └── Mismatch: ❌ FAIL (officer must re-check document)

  CHECK 6: Blacklist check
  ├── Is this name + DOB combination flagged?
  │   (fraud watchlist, court order, etc.)
  └── Flagged → escalate to supervisor (do not auto-reject)

DEMOGRAPHIC QUALITY REPORT:
  {
    "demographicQuality": {
      "requiredFieldsComplete": true,
      "formatValidation":       "PASSED",
      "ocrCrossCheck": {
        "nameMatchScore":  91.3,
        "nameDecision":    "PASSED",
        "dobMatch":        true,
        "dobDecision":     "PASSED"
      },
      "crossFieldConsistency":  "PASSED",
      "blacklistCheck":         "CLEAR",
      "overallDecision":        "PASSED"
    }
  }
```

---

## 24.8 Document Quality Checks

```
┌──────────────────────────────────────────────────────────────┐
│              Document Quality Check Pipeline                 │
└──────────────────────────────────────────────────────────────┘

FOR EACH SCANNED DOCUMENT:

  CHECK 1: Scan quality
  ├── Resolution ≥ 300 DPI?
  ├── Image not blurry (sharpness score)?
  ├── Image not overexposed / underexposed?
  └── Both sides scanned (if double-sided document)?

  CHECK 2: Document type recognition
  ├── Does the scan match the declared document type?
  │   (passport template vs ID card template)
  └── Can fields be located in expected positions?

  CHECK 3: MRZ validation (if applicable)
  ├── MRZ lines present at bottom of page?
  ├── MRZ parsed successfully?
  ├── Check digits valid for all fields?
  └── Extracted data (name, DOB) matches entered data?

  CHECK 4: Document validity
  ├── Issue date in the past?
  ├── Expiry date in the future? (not expired)
  ├── Issue date < Expiry date? (logical)
  └── Document number format valid for issuing country?

  CHECK 5: Tamper indicators (automated)
  ├── Can OCR extract text cleanly?
  │   (tampered text: different font, color artifacts)
  ├── Are image statistics consistent?
  │   (copy-paste substitution: different JPEG artifacts)
  └── MRZ inconsistency:
      document number in MRZ ≠ printed number → flag

  CHECK 6: OCR confidence
  ├── Per-field confidence score from OCR engine
  ├── Low confidence (< 60): flag for officer re-scan
  └── Overall OCR confidence ≥ 70?

DOCUMENT QUALITY REPORT:
  {
    "documentQuality": {
      "scanResolution":    600,
      "scanSharpness":     88.2,
      "documentType":      "PASSPORT",
      "documentRecognized":true,
      "mrzValid":          true,
      "mrzCheckDigits":    "ALL_VALID",
      "documentExpired":   false,
      "ocrConfidence":     94.1,
      "tamperIndicators":  "NONE",
      "overallDecision":   "PASSED"
    }
  }
```

---

## 24.9 Cross-Modality Consistency Checks

After individual modality checks pass, cross-modal checks verify that the biometrics from different modalities are internally consistent:

```
┌──────────────────────────────────────────────────────────────┐
│              Cross-Modality Consistency Checks               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  CHECK 1: Face-to-document photo match                       │
│  ├── Compare enrolled face vs face in document photo        │
│  ├── Tool: Amazon Rekognition CompareFaces API              │
│  ├── Similarity < 70%: FLAG → supervisor review            │
│  │   (possible: wrong person using genuine document)        │
│  └── Similarity ≥ 70%: PASS                                │
│                                                              │
│  CHECK 2: Age consistency                                    │
│  ├── Estimated age from face (AI model) vs stated DOB      │
│  ├── Difference > 15 years: FLAG (unusual, not reject)     │
│  └── Example: stated DOB = 1960, face age estimate = 35    │
│      → 30-year discrepancy → flag for review               │
│                                                              │
│  CHECK 3: Gender consistency                                 │
│  ├── AI gender estimate from face vs stated gender          │
│  ├── Mismatch: FLAG (informational — not grounds to reject) │
│  └── Note: gender expression ≠ legal gender                │
│      Do NOT reject based on gender estimate mismatch.       │
│      Only flag for human awareness.                         │
│                                                              │
│  CHECK 4: Biometric set completeness                         │
│  ├── Are all expected biometrics present?                   │
│  ├── Exception flags for missing modalities?               │
│  │   (if missing without flag → reject packet)             │
│  └── Are exception flags authorized (supervisor approval)? │
│                                                              │
│  CHECK 5: Duplicate biometric within packet                  │
│  ├── Was the same fingerprint submitted for multiple       │
│  │   different finger positions?                           │
│  │   (fraud: one good finger submitted 10 times)           │
│  ├── Run quick pairwise match on submitted fingers         │
│  └── Any pair matches > 0.7 threshold → FRAUD FLAG        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.10 Quality Check Decision Matrix

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Check Decision Matrix                   │
└──────────────────────────────────────────────────────────────┘

ENROLLMENT PACKET QUALITY DECISION LOGIC:

  IF all biometrics PASS and all demographics PASS
  AND all documents PASS:
     → QUALITY_PASSED → proceed to deduplication

  ELIF minor issues (some biometrics in WARN range):
     → QUALITY_PASSED_WITH_WARNINGS
     → Logged, supervisor informed, proceed to dedup
     → Citizen may be contacted for re-enrollment later

  ELIF one or two biometrics FAIL below hard minimum:
     → QUALITY_REVIEW → supervisor queue
     Supervisor options:
     ├── APPROVE_WITH_EXCEPTION (document reason)
     └── REJECT_FOR_RECAPTURE → notify officer

  ELIF critical failure:
     (no primary document, face quality < 30, all
     fingerprints below 40 without documented exception)
     → QUALITY_REJECTED → packet rejected
     → Officer notified with specific reason codes
     → Citizen must return for re-enrollment

  ELIF fraud indicators:
     (face-document mismatch, duplicate fingers,
     MRZ manipulation, document tamper indicators)
     → FRAUD_FLAG → escalated to security team
     → Not processed further until cleared

DECISION CODES:
  ┌─────────────────────────────────────────────────────┐
  │ Code                    │ Next step                 │
  ├─────────────────────────┼───────────────────────────┤
  │ QUALITY_PASSED          │ → Deduplication           │
  │ QUALITY_WARN            │ → Deduplication + logged  │
  │ QUALITY_REVIEW          │ → Supervisor queue        │
  │ QUALITY_REJECTED        │ → Officer notified        │
  │ FRAUD_FLAG              │ → Security team           │
  └─────────────────────────┴───────────────────────────┘
```

---

## 24.11 Quality Check Metrics and Monitoring

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Metrics Dashboard                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PER REGISTRATION CENTER (daily report):                     │
│                                                              │
│  Fingerprint metrics:                                        │
│  ├── average_nfiq2_score (target: ≥ 65)                    │
│  ├── below_minimum_rate (target: < 5%)                     │
│  ├── recapture_rate (target: < 15%)                        │
│  └── exception_rate (target: < 2%)                         │
│                                                              │
│  Face metrics:                                               │
│  ├── average_face_quality_score (target: ≥ 70)             │
│  ├── icao_compliance_rate (target: > 95%)                  │
│  ├── recapture_rate (target: < 10%)                        │
│  └── liveness_fail_rate (target: < 1%)                    │
│                                                              │
│  Iris metrics:                                               │
│  ├── average_iris_quality_score (target: ≥ 70)             │
│  ├── usable_area_average (target: ≥ 70%)                   │
│  └── recapture_rate (target: < 10%)                        │
│                                                              │
│  Demographic metrics:                                        │
│  ├── ocr_mismatch_rate (target: < 3%)                      │
│  ├── format_error_rate (target: < 1%)                      │
│  └── document_rejection_rate (target: < 5%)               │
│                                                              │
│  CLOUDWATCH ALARMS:                                          │
│  ├── average_nfiq2 < 50 for 1 hour at any center          │
│  │   → Alert: equipment issue / officer training need      │
│  ├── icao_compliance_rate < 80% for 1 hour                │
│  │   → Alert: camera misconfigured / lighting problem      │
│  ├── liveness_fail_rate > 5% for 30 min                   │
│  │   → Alert: possible spoofing attempt                   │
│  ├── ocr_mismatch_rate > 10% for 1 hour                   │
│  │   → Alert: officer entering data incorrectly           │
│  └── fraud_flag_rate > 1% for 1 day                       │
│      → Alert: investigate center for organized fraud      │
│                                                              │
│  QUALITY HEATMAP (weekly management report):                 │
│  Shows per-center quality scores on a map:                  │
│  ├── GREEN:  All metrics at target                         │
│  ├── YELLOW: 1–2 metrics below target (investigate)        │
│  └── RED:    Multiple metrics below minimum (intervene)    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.12 Quality Feedback Loop

Quality checks are only useful if they drive improvement. A feedback loop connects quality metrics back to operator training and equipment maintenance:

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Improvement Feedback Loop               │
└──────────────────────────────────────────────────────────────┘

STEP 1: Detect quality issue
  CloudWatch alarm:
  CENTER-005 average NFIQ2 < 50 for 3 consecutive days
         │
         ▼
STEP 2: Investigate
  Operations team reviews:
  ├── Which officers have low NFIQ2 captures?
  ├── Is it all fingers or specific fingers?
  ├── Is it one officer or all officers at center?
  └── When did it start? (sudden drop = equipment issue)
         │
         ▼
STEP 3: Root cause
  Possible causes:
  ├── Dirty platen (clean with alcohol → immediate fix)
  ├── Damaged platen (scratched → replace device)
  ├── Officer technique (rolling too fast → training)
  ├── Population factor (many laborers → adjust process)
  └── Algorithm calibration (quality SDK update needed)
         │
         ▼
STEP 4: Remediation
  ├── Equipment: schedule maintenance / replacement
  ├── Training: targeted session for specific officers
  ├── Process: update guidance for specific populations
  └── Software: update quality SDK / thresholds
         │
         ▼
STEP 5: Verify improvement
  Monitor next 7 days:
  ├── NFIQ2 rising back to target? → remediation worked
  └── Still low? → escalate investigation
         │
         ▼
STEP 6: Document and close
  Incident report:
  ├── What was the issue?
  ├── Root cause identified?
  ├── What was the fix?
  └── How many enrollments affected?
      (these may need re-enrollment outreach campaign)
```

---

## 24.13 Quality Check Error Codes

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Check Error Codes                       │
├────────────────┬─────────────────────────────────────────────┤
│ Code           │ Meaning                                    │
├────────────────┼─────────────────────────────────────────────┤
│ QC_FP_001      │ Fingerprint NFIQ2 below hard minimum (< 40)│
│ QC_FP_002      │ Fingerprint image blank / not captured     │
│ QC_FP_003      │ Fewer than 8 fingers above minimum quality │
│ QC_FP_004      │ Duplicate finger detected in set          │
│ QC_FP_005      │ Finger position inconsistent with image   │
│ QC_FC_001      │ No face detected in image                 │
│ QC_FC_002      │ Multiple faces detected                   │
│ QC_FC_003      │ Face pose out of ICAO range               │
│ QC_FC_004      │ Eyes not open / not visible               │
│ QC_FC_005      │ Face quality score below minimum (< 50)   │
│ QC_FC_006      │ Face-document photo mismatch              │
│ QC_IR_001      │ Iris not detected in image                │
│ QC_IR_002      │ Usable iris area below minimum (< 50%)    │
│ QC_IR_003      │ Iris sharpness below threshold            │
│ QC_IR_004      │ Pupil dilation ratio out of range        │
│ QC_DM_001      │ Required demographic field missing        │
│ QC_DM_002      │ Field format validation failed            │
│ QC_DM_003      │ Name OCR mismatch with document          │
│ QC_DM_004      │ DOB mismatch with document               │
│ QC_DC_001      │ Document scan below minimum resolution    │
│ QC_DC_002      │ MRZ check digit failure                  │
│ QC_DC_003      │ Document expired                         │
│ QC_DC_004      │ OCR confidence below threshold           │
│ QC_CM_001      │ Age estimated from face inconsistent      │
│ QC_CM_002      │ Cross-modality fraud indicator detected   │
└────────────────┴─────────────────────────────────────────────┘
```

---

## 24.14 Complete Quality Check Flow — End to End

```
┌──────────────────────────────────────────────────────────────┐
│              Complete Quality Check Flow                     │
└──────────────────────────────────────────────────────────────┘

ENROLLMENT PACKET RECEIVED (S3 upload)
         │
         ▼
SQS → Registration Processor (ECS Fargate)
         │
    ┌────┴──────────────────────────────────────────────┐
    │                                                   │
    ▼                                                   ▼
MDS Signature Verify                       Packet Structure Check
(device cert validation)                   (all fields present?)
         │                                             │
         └──────────────┬────────────────────────────┘
                        │
                        ▼
              Decrypt packet (KMS)
                        │
          ┌─────────────┼───────────────────┐
          │             │                   │
          ▼             ▼                   ▼
  Fingerprint      Face Quality         Iris Quality
  Quality Check    Check                Check
  (NFIQ2 × 10)    (ICAO + ISO          (ISO 29794-6)
                   19794-5)
          │             │                   │
          └─────────────┼───────────────────┘
                        │
                        ▼
              Demographic Quality Check
              (format + OCR cross-check)
                        │
                        ▼
              Document Quality Check
              (scan + MRZ + OCR confidence)
                        │
                        ▼
              Cross-Modality Checks
              (face-doc match + age + fraud)
                        │
                        ▼
              Quality Decision Engine
                        │
         ┌──────────────┼──────────────────────┐
         │              │                      │
         ▼              ▼                      ▼
    PASSED          REVIEW              REJECTED/FRAUD
         │              │                      │
         ▼              ▼                      ▼
  → Deduplication  → Supervisor         → Officer/Security
    (next stage)     queue                 notification
                       │                      │
                       ▼                      ▼
                  Supervisor            DLQ + alarm
                  approves or           + audit log
                  rejects
                       │
                       ▼
               APPROVED → Dedup
               REJECTED → Officer
```

---

## 24.15 AWS Architecture for Quality Checks

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Check AWS Architecture                  │
└──────────────────────────────────────────────────────────────┘

SQS (nbis-enrollment-received)
         │
         ▼
Quality Check Service (ECS Fargate)
         │
         ├──► S3: retrieve decrypted packet contents
         │
         ├──► NFIQ2 SDK container:
         │    Fingerprint quality per finger
         │
         ├──► Amazon Rekognition:
         │    ├── DetectFaces: face quality + ICAO check
         │    └── CompareFaces: face vs document photo
         │
         ├──► Iris Quality SDK container:
         │    ISO 29794-6 quality scoring
         │
         ├──► Amazon Textract:
         │    OCR + document field extraction
         │    MRZ line extraction
         │
         ├──► Custom MRZ validator (Lambda):
         │    Check digit verification
         │
         ├──► DynamoDB:
         │    Update enrollment status
         │    Store quality scores per modality
         │
         ├──► SQS routing:
         │    ├── PASSED → nbis-abis-requests
         │    ├── REVIEW → nbis-supervisor-review
         │    └── REJECTED → nbis-officer-notification
         │
         └──► CloudWatch:
              Publish quality metrics
              Per-center, per-modality, per-officer

QUALITY SCORES STORED IN DYNAMODB:
  DynamoDB table: nbis-enrollment-quality
  PK: packetId
  Attributes:
  ├── fingerprintScores: {finger: score} map
  ├── faceScore: overall + dimension breakdown
  ├── irisScores: {left: score, right: score}
  ├── demographicDecision: PASSED/FAILED
  ├── documentDecision: PASSED/FAILED
  ├── crossModalityFlags: list of flags
  ├── overallDecision: decision code
  └── processingTime: milliseconds
```

---

## 24.16 MOSIP Reference

```
┌──────────────────────────────────────────────────────────────┐
│              MOSIP Quality Check Reference                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  MOSIP Registration Processor quality stages:               │
│                                                              │
│  Stage 1: Packet Receiver                                   │
│  └── Validates packet structure and signature               │
│                                                              │
│  Stage 2: Packet Validator                                  │
│  └── Schema validation, field completeness                  │
│                                                              │
│  Stage 3: Quality Classifier                                │
│  ├── Biometric quality check (NFIQ2, face, iris)           │
│  ├── Configurable quality thresholds (per country)         │
│  └── Pluggable: country replaces default SDK with own      │
│                                                              │
│  Stage 4: Demo Dedupe (demographic pre-check)               │
│  └── Quick demographic match check before ABIS             │
│      (saves ABIS calls for obvious duplicates)             │
│                                                              │
│  MOSIP quality threshold configuration:                     │
│  In MOSIP admin portal:                                     │
│  ├── fingerprint.quality.threshold = 60                    │
│  ├── iris.quality.threshold = 70                           │
│  ├── face.quality.threshold = 70                           │
│  └── Configurable without code change                      │
│                                                              │
│  MOSIP quality failure handling:                            │
│  ├── Failed packet → BIOMETRIC_QUALITY_CHECK_FAILED        │
│  ├── Notification: officer SMS/email (configurable)        │
│  └── Packet: stays in system for supervisor review         │
│                                                              │
│  MOSIP supervisor portal (Registration Supervisor):         │
│  ├── View all packets in REVIEW queue                      │
│  ├── See quality scores + reason for review               │
│  ├── Approve with documented reason                        │
│  └── Reject (triggers re-enrollment notification)         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.17 Key Terms

| Term | Definition |
|------|-----------|
| **Quality check** | Validation that captured data meets minimum standards for reliable use |
| **Client-side check** | Real-time quality check at the enrollment center (immediate feedback) |
| **Server-side check** | Authoritative quality check in the back-end pipeline (post-submission) |
| **NFIQ2** | NIST Fingerprint Image Quality 2 — fingerprint quality standard (0–100) |
| **ISO 29794-6** | International standard for iris image quality metrics |
| **ISO 19794-5** | International standard for face image data and quality |
| **ICAO compliance** | Face image meets ICAO Doc 9303 requirements for identity documents |
| **OCR cross-check** | Comparing OCR-extracted document data with officer-entered demographics |
| **MRZ validation** | Verifying check digits in the passport machine-readable zone |
| **Cross-modality check** | Comparing consistency between different biometric modalities |
| **Face-document match** | Comparing enrolled face photo against document photo using face recognition |
| **Duplicate finger detection** | Checking if same fingerprint submitted for multiple positions |
| **Quality decision engine** | System that combines all check results into a final enrollment decision |
| **QUALITY_PASSED** | All checks passed — proceed to deduplication |
| **QUALITY_REVIEW** | Some issues — human supervisor must decide |
| **QUALITY_REJECTED** | Fundamental failure — enrollment must be redone |
| **FRAUD_FLAG** | Indicators of deliberate fraud detected — escalate to security |
| **Quality heatmap** | Visual map showing per-center quality performance |
| **Recapture rate** | Percentage of biometric captures requiring a second attempt |
| **Feedback loop** | Process connecting quality metrics to training and equipment maintenance |

---

## 24.18 Key Takeaways

- **Quality checks happen twice** — client-side for immediate feedback (officer can recapture now) and server-side for the authoritative decision (after submission). Never rely on client-side alone.
- **Hard minimums are hard for a reason** — below NFIQ2 40 or iris quality 50, the template is genuinely unreliable. Supervisor overrides below these levels must be documented with a specific, auditable reason.
- **Cross-modality checks catch fraud that single-modality checks miss** — a duplicate finger in the same packet, or a face that does not match the passport photo, are caught only when you compare across modalities.
- **Quality metrics are a center management tool** — a center with consistently low NFIQ2 scores has either a dirty platen, a broken scanner, undertrained officers, or a population with occupational wear. The metric tells you something is wrong; investigation reveals which.
- **OCR cross-check is not a duplicate of document verification** — it specifically checks whether what the officer typed matches what the document actually says. Officers make data entry errors. This check catches them before the UIN is issued.
- **Fraud flags are escalations, not automatic rejections** — face-document mismatch could mean a bad passport photo, not a fraudster. Age inconsistency could mean the citizen looks much younger than their DOB. Human review is required before any fraud-flagged packet is rejected.
- **Quality thresholds are configurable** — MOSIP allows country-specific threshold settings without code changes. Different populations have different biometric characteristics. What works in Sweden may not be appropriate for a country with many manual laborers.
- **The quality feedback loop is as important as the check itself** — quality checks without corrective action are just data. Track, alarm, investigate, remediate, and verify improvement.

---

## 24.19 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 25 | Enrollment Validation — packet validation, completeness, business rules |
| Chapter 26 | Duplicate Detection — ABIS integration, 1:N search, dedup workflow |
| Chapter 27 | Approval Workflow — supervisor review, manual decisions, audit trail |

---

*Chapter 24 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*# Chapter 24 — Quality Checks

> **Part III — Enrollment**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 24.1 What are Quality Checks?

**Quality checks** are systematic validations applied to every piece of data captured during enrollment — biometrics, demographics, documents, and signatures — to ensure that the data is good enough to serve its purpose reliably for the lifetime of the identity record.

Quality checking is the NBIS's internal quality assurance mechanism. It answers the question: **"Is this data good enough that we can be confident it will work correctly for authentication and deduplication for the next 10–30 years?"**

```
┌──────────────────────────────────────────────────────────────┐
│              Why Quality Checks Are Critical                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  A fingerprint enrolled with NFIQ2 = 35 (below minimum):   │
│  ├── Authentication failure rate: 15–20%                   │
│  ├── Citizen attends center multiple times to re-enroll    │
│  ├── Government spends money on re-enrollment support      │
│  └── Citizen loses trust in the identity system            │
│                                                              │
│  An iris enrolled with quality = 40 (below minimum):       │
│  ├── IrisCode is unreliable                                │
│  ├── Border crossing failures                               │
│  └── Legitimate citizen treated as unknown person          │
│                                                              │
│  A name with OCR cross-check failure accepted anyway:       │
│  ├── Name on ID card does not match civil registry         │
│  ├── Bank cannot complete KYC (name mismatch)              │
│  └── Citizen must go through legal name correction process │
│                                                              │
│  Quality checks at enrollment are CHEAPER than fixing       │
│  problems after UINs are issued. The cost of a 2-minute    │
│  recapture at enrollment is infinitely cheaper than years  │
│  of authentication failures and re-enrollment campaigns.   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.2 Two Phases of Quality Checking

Quality checks happen at two distinct points in the enrollment pipeline:

```
┌──────────────────────────────────────────────────────────────┐
│              Two-Phase Quality Check Design                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PHASE 1: CLIENT-SIDE (at registration center, real-time)   │
│  ──────────────────────────────────────────────────────     │
│  When: immediately after each biometric capture             │
│  Who: Registration Client software                          │
│  Goal: immediate feedback → officer can recapture now       │
│                                                              │
│  What it checks:                                             │
│  ├── NFIQ2 per finger (quick algorithm)                    │
│  ├── Basic face ICAO checks (pose, eyes open)              │
│  ├── Basic iris quality (area, sharpness)                  │
│  ├── Signature stroke count + area                         │
│  └── Required fields present (demographic completeness)    │
│                                                              │
│  Speed: must be < 2 seconds (real-time feedback)           │
│  Accuracy: good enough for immediate guidance              │
│  Algorithm: lightweight, runs on registration workstation  │
│                                                              │
│  PHASE 2: SERVER-SIDE (back-end pipeline, post-submission)  │
│  ──────────────────────────────────────────────────────     │
│  When: after encrypted packet uploaded to S3               │
│  Who: Registration Processor (ECS + Lambda)                 │
│  Goal: authoritative quality determination                  │
│                                                              │
│  What it checks:                                             │
│  ├── Full NFIQ2 (complete algorithm, more accurate)        │
│  ├── Full face quality (all ICAO dimensions)               │
│  ├── Full iris quality (all ISO 29794-6 dimensions)        │
│  ├── OCR cross-check (document vs entered demographics)    │
│  ├── MRZ validation (check digit verification)             │
│  ├── Document authenticity indicators                      │
│  ├── Signature quality (server-side assessment)            │
│  └── Cross-modality consistency checks                     │
│                                                              │
│  Speed: minutes acceptable (async pipeline)                │
│  Accuracy: authoritative — determines pass/fail            │
│  Algorithm: full production-grade algorithms               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.3 Biometric Quality Standards — Summary

```
┌──────────────────────────────────────────────────────────────┐
│              Biometric Quality Thresholds Reference          │
├────────────────────┬──────────────┬───────────┬─────────────┤
│ Modality           │ Standard     │ Minimum   │ Target      │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Fingerprint        │ NFIQ2        │ 40 / 100  │ 60 / 100   │
│ (per finger)       │ (NIST)       │           │            │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Fingerprint        │ NFIQ2        │ Average   │ Average     │
│ (overall set)      │ average      │ ≥ 50      │ ≥ 65       │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Iris (per eye)     │ ISO 29794-6  │ 50 / 100  │ 70 / 100   │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Face (overall)     │ ISO 19794-5  │ 50 / 100  │ 70 / 100   │
│                    │ + ICAO       │           │            │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Face (pose)        │ ICAO 9303    │ ≤ ±5°     │ ≤ ±3°      │
│                    │              │ yaw/pitch/│            │
│                    │              │ roll      │            │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Signature          │ Internal     │ ≥ 3       │ ≥ 5        │
│ (stroke count)     │              │ strokes   │ strokes    │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Document scan      │ Internal     │ ≥ 300 DPI │ ≥ 600 DPI  │
│                    │              │ legible   │            │
└────────────────────┴──────────────┴───────────┴────────────┘
```

---

## 24.4 Fingerprint Quality Check — Deep Dive

```
┌──────────────────────────────────────────────────────────────┐
│              Fingerprint Quality Check Pipeline              │
└──────────────────────────────────────────────────────────────┘

INPUT: 10 fingerprint images (WSQ compressed, 500 PPI)
         │
         ▼
FOR EACH FINGER:

  CHECK 1: Image completeness
  ├── Is the image size correct? (not truncated)
  ├── Is the image format valid WSQ?
  └── Is the capture device certified? (MDS signature check)

  CHECK 2: NFIQ2 quality score
  ├── Run NFIQ2 algorithm on each finger image
  ├── Score 0–100 returned for each finger
  └── Apply thresholds:
      < 20:  UNUSABLE → mandatory recapture
      20–39: POOR     → strongly recommend recapture
      40–59: FAIR     → warn, accept with supervisor
      60–74: GOOD     → accept
      75+:   EXCELLENT → accept

  CHECK 3: Pattern detection
  └── Can a ridge pattern be detected?
      (some images are blank / all-white = pad not touched)

  CHECK 4: Finger position consistency
  └── Does the image match the declared finger position?
      (a thumb image submitted as index finger → flag)
      (uses aspect ratio + pattern type classification)

OUTPUT PER FINGER:
  {
    "finger":     "RIGHT_INDEX",
    "nfiq2Score": 78,
    "quality":    "GOOD",
    "decision":   "ACCEPTED",
    "attempts":   1
  }

OVERALL FINGERPRINT QUALITY DECISION:
  ├── ALL 10 fingers ≥ 40:   PASS
  ├── ≥ 8 fingers ≥ 40:      PASS (2 exceptions documented)
  ├── < 8 fingers ≥ 40:      FAIL → supervisor review
  └── Average NFIQ2 < 50:    WARN → supervisor recommendation

QUALITY REPORT SUMMARY:
  {
    "fingerprintQuality": {
      "totalFingers":     10,
      "passed":           9,
      "failed":           1,
      "failedFingers":    ["RIGHT_RING"],
      "averageNFIQ2":     74.3,
      "overallDecision":  "PASS_WITH_EXCEPTION",
      "exceptionReason":  "RIGHT_RING below threshold after
                          3 attempts — skin condition noted"
    }
  }
```

---

## 24.5 Face Quality Check — Deep Dive

```
┌──────────────────────────────────────────────────────────────┐
│              Face Quality Check Pipeline                     │
└──────────────────────────────────────────────────────────────┘

INPUT: Face image (JPEG/JPEG2000, minimum 600×800px)
         │
         ▼
CHECK 1: Face detection
  ├── Is exactly one face detected in the image?
  ├── Zero faces → reject (no face captured)
  └── Multiple faces → reject (someone else in frame)

CHECK 2: Face size and position (geometric)
  ├── Face bounding box occupies 70–80% of frame height?
  ├── Face centered within ±10% of frame?
  └── Eye-to-eye distance ≥ 90 pixels?

CHECK 3: Pose angles (ICAO compliance)
  ├── Yaw (left-right rotation): ≤ ±5°?
  ├── Pitch (chin up/down): ≤ ±5°?
  └── Roll (tilt): ≤ ±5°?

CHECK 4: Eye quality
  ├── Both eyes detected?
  ├── Left eye open score ≥ 70?
  ├── Right eye open score ≥ 70?
  └── Eyes looking at camera (gaze direction check)?

CHECK 5: Lighting quality
  ├── Overall brightness within acceptable range?
  ├── Both sides of face equally lit (symmetry)?
  ├── No overexposed regions (washed out)?
  └── No underexposed regions (too dark)?

CHECK 6: Sharpness
  ├── Focus score ≥ 70?
  └── No motion blur detected?

CHECK 7: Occlusion
  ├── No mask covering nose/mouth?
  ├── Hair not covering eyes?
  └── No hand or object in front of face?

CHECK 8: Expression
  ├── Mouth closed (or near-closed)?
  └── Neutral expression (no extreme emotion)?

CHECK 9: Background
  └── Background sufficiently uniform?
      (high texture background = may confuse face detection)

COMBINED QUALITY SCORE:
  Weighted average across all dimensions:
  ├── Pose:       30% weight (ICAO critical)
  ├── Eyes:       25% weight (matching critical)
  ├── Lighting:   20% weight
  ├── Sharpness:  15% weight
  └── Occlusion:  10% weight

  Score ≥ 70: PASS
  Score 50–69: WARN → officer prompted to retake
  Score < 50:  FAIL → mandatory retake

FACE QUALITY REPORT:
  {
    "faceQuality": {
      "faceDetected":      true,
      "faceCount":         1,
      "poseAngles": {
        "yaw":   2.3,
        "pitch": -1.1,
        "roll":  0.8
      },
      "eyesOpen":          true,
      "lightingScore":     82.1,
      "sharpnessScore":    88.4,
      "occlusion":         "NONE",
      "overallScore":      84.7,
      "icaoCompliant":     true,
      "decision":          "ACCEPTED"
    }
  }
```

---

## 24.6 Iris Quality Check — Deep Dive

```
┌──────────────────────────────────────────────────────────────┐
│              Iris Quality Check Pipeline                     │
└──────────────────────────────────────────────────────────────┘

INPUT: Two iris NIR images (left + right, 640×480px minimum)
         │
         ▼
FOR EACH EYE:

  CHECK 1: Image validity
  ├── Is image format correct?
  ├── Is image not corrupted?
  └── Is MDS device signature valid?

  CHECK 2: Eye/iris detection
  ├── Is an eye detected in the image?
  ├── Is the iris boundary locatable?
  └── Is the pupil detectable?

  CHECK 3: Usable iris area
  ├── Segment iris: locate pupil + limbus boundaries
  ├── Exclude eyelid + eyelash occlusion regions
  ├── Calculate: usable area / total iris area
  └── Usable area ≥ 50%? (minimum)
      Usable area ≥ 70%? (target)

  CHECK 4: Iris sharpness
  ├── Measure frequency content of iris texture region
  ├── Blurry image: low frequency → low sharpness score
  └── Sharpness score ≥ 70?

  CHECK 5: Pupil dilation ratio
  ├── pupil_diameter / iris_diameter = ratio
  ├── Acceptable range: 0.2 – 0.7
  ├── < 0.2: unusual (very bright light / medical)
  └── > 0.7: very dilated (very dark / medication)

  CHECK 6: Gaze direction
  ├── Is iris centered in the image?
  ├── Off-center iris = subject not looking at camera
  └── Acceptable: < 10% off-center

  CHECK 7: Motion blur
  └── Blur score < 10? (subject was still during capture)

COMBINED IRIS QUALITY SCORE:
  {
    "irisQuality": {
      "leftEye": {
        "usableArea":     73.4,
        "sharpness":      82.1,
        "pupilRatio":     0.42,
        "gazeOffset":     3.2,
        "motionBlur":     1.8,
        "overallScore":   78.6,
        "decision":       "ACCEPTED"
      },
      "rightEye": {
        "usableArea":     68.9,
        "sharpness":      79.3,
        "pupilRatio":     0.44,
        "gazeOffset":     4.1,
        "motionBlur":     2.1,
        "overallScore":   74.2,
        "decision":       "ACCEPTED"
      }
    }
  }
```

---

## 24.7 Demographic Quality Checks

```
┌──────────────────────────────────────────────────────────────┐
│              Demographic Quality Check Pipeline              │
└──────────────────────────────────────────────────────────────┘

FOR EACH DEMOGRAPHIC FIELD:

  CHECK 1: Required field present
  └── Is this field populated?
      Empty required field → reject packet

  CHECK 2: Format validation
  ├── Name: regex — only allowed characters?
  ├── DOB:  ISO 8601 format? valid calendar date?
  ├── Mobile: E.164 format? valid country code?
  └── Email: RFC 5322 format?

  CHECK 3: Value range validation
  ├── DOB: not in future, not > 150 years ago
  ├── Name length: 1–150 characters
  └── Age-based rules:
      ├── Age < 5: no fingerprints expected
      ├── Age < 18: guardian consent required
      └── Age > 100: supervisor approval required

  CHECK 4: Cross-field consistency
  ├── fullName == firstName + middleName + lastName?
  ├── Current address == Permanent if sameAsPermanent?
  └── DOB consistent with stated age (if age also given)?

  CHECK 5: OCR cross-check (document vs entered)
  ├── Name: normalized comparison
  │   ├── Match score ≥ 80%: ✅ PASS
  │   ├── Match score 60–79%: ⚠️ WARN (officer confirm)
  │   └── Match score < 60%: ❌ FAIL (must correct)
  └── DOB: exact match required
      ├── Match: ✅ PASS
      └── Mismatch: ❌ FAIL (officer must re-check document)

  CHECK 6: Blacklist check
  ├── Is this name + DOB combination flagged?
  │   (fraud watchlist, court order, etc.)
  └── Flagged → escalate to supervisor (do not auto-reject)

DEMOGRAPHIC QUALITY REPORT:
  {
    "demographicQuality": {
      "requiredFieldsComplete": true,
      "formatValidation":       "PASSED",
      "ocrCrossCheck": {
        "nameMatchScore":  91.3,
        "nameDecision":    "PASSED",
        "dobMatch":        true,
        "dobDecision":     "PASSED"
      },
      "crossFieldConsistency":  "PASSED",
      "blacklistCheck":         "CLEAR",
      "overallDecision":        "PASSED"
    }
  }
```

---

## 24.8 Document Quality Checks

```
┌──────────────────────────────────────────────────────────────┐
│              Document Quality Check Pipeline                 │
└──────────────────────────────────────────────────────────────┘

FOR EACH SCANNED DOCUMENT:

  CHECK 1: Scan quality
  ├── Resolution ≥ 300 DPI?
  ├── Image not blurry (sharpness score)?
  ├── Image not overexposed / underexposed?
  └── Both sides scanned (if double-sided document)?

  CHECK 2: Document type recognition
  ├── Does the scan match the declared document type?
  │   (passport template vs ID card template)
  └── Can fields be located in expected positions?

  CHECK 3: MRZ validation (if applicable)
  ├── MRZ lines present at bottom of page?
  ├── MRZ parsed successfully?
  ├── Check digits valid for all fields?
  └── Extracted data (name, DOB) matches entered data?

  CHECK 4: Document validity
  ├── Issue date in the past?
  ├── Expiry date in the future? (not expired)
  ├── Issue date < Expiry date? (logical)
  └── Document number format valid for issuing country?

  CHECK 5: Tamper indicators (automated)
  ├── Can OCR extract text cleanly?
  │   (tampered text: different font, color artifacts)
  ├── Are image statistics consistent?
  │   (copy-paste substitution: different JPEG artifacts)
  └── MRZ inconsistency:
      document number in MRZ ≠ printed number → flag

  CHECK 6: OCR confidence
  ├── Per-field confidence score from OCR engine
  ├── Low confidence (< 60): flag for officer re-scan
  └── Overall OCR confidence ≥ 70?

DOCUMENT QUALITY REPORT:
  {
    "documentQuality": {
      "scanResolution":    600,
      "scanSharpness":     88.2,
      "documentType":      "PASSPORT",
      "documentRecognized":true,
      "mrzValid":          true,
      "mrzCheckDigits":    "ALL_VALID",
      "documentExpired":   false,
      "ocrConfidence":     94.1,
      "tamperIndicators":  "NONE",
      "overallDecision":   "PASSED"
    }
  }
```

---

## 24.9 Cross-Modality Consistency Checks

After individual modality checks pass, cross-modal checks verify that the biometrics from different modalities are internally consistent:

```
┌──────────────────────────────────────────────────────────────┐
│              Cross-Modality Consistency Checks               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  CHECK 1: Face-to-document photo match                       │
│  ├── Compare enrolled face vs face in document photo        │
│  ├── Tool: Amazon Rekognition CompareFaces API              │
│  ├── Similarity < 70%: FLAG → supervisor review            │
│  │   (possible: wrong person using genuine document)        │
│  └── Similarity ≥ 70%: PASS                                │
│                                                              │
│  CHECK 2: Age consistency                                    │
│  ├── Estimated age from face (AI model) vs stated DOB      │
│  ├── Difference > 15 years: FLAG (unusual, not reject)     │
│  └── Example: stated DOB = 1960, face age estimate = 35    │
│      → 30-year discrepancy → flag for review               │
│                                                              │
│  CHECK 3: Gender consistency                                 │
│  ├── AI gender estimate from face vs stated gender          │
│  ├── Mismatch: FLAG (informational — not grounds to reject) │
│  └── Note: gender expression ≠ legal gender                │
│      Do NOT reject based on gender estimate mismatch.       │
│      Only flag for human awareness.                         │
│                                                              │
│  CHECK 4: Biometric set completeness                         │
│  ├── Are all expected biometrics present?                   │
│  ├── Exception flags for missing modalities?               │
│  │   (if missing without flag → reject packet)             │
│  └── Are exception flags authorized (supervisor approval)? │
│                                                              │
│  CHECK 5: Duplicate biometric within packet                  │
│  ├── Was the same fingerprint submitted for multiple       │
│  │   different finger positions?                           │
│  │   (fraud: one good finger submitted 10 times)           │
│  ├── Run quick pairwise match on submitted fingers         │
│  └── Any pair matches > 0.7 threshold → FRAUD FLAG        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.10 Quality Check Decision Matrix

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Check Decision Matrix                   │
└──────────────────────────────────────────────────────────────┘

ENROLLMENT PACKET QUALITY DECISION LOGIC:

  IF all biometrics PASS and all demographics PASS
  AND all documents PASS:
     → QUALITY_PASSED → proceed to deduplication

  ELIF minor issues (some biometrics in WARN range):
     → QUALITY_PASSED_WITH_WARNINGS
     → Logged, supervisor informed, proceed to dedup
     → Citizen may be contacted for re-enrollment later

  ELIF one or two biometrics FAIL below hard minimum:
     → QUALITY_REVIEW → supervisor queue
     Supervisor options:
     ├── APPROVE_WITH_EXCEPTION (document reason)
     └── REJECT_FOR_RECAPTURE → notify officer

  ELIF critical failure:
     (no primary document, face quality < 30, all
     fingerprints below 40 without documented exception)
     → QUALITY_REJECTED → packet rejected
     → Officer notified with specific reason codes
     → Citizen must return for re-enrollment

  ELIF fraud indicators:
     (face-document mismatch, duplicate fingers,
     MRZ manipulation, document tamper indicators)
     → FRAUD_FLAG → escalated to security team
     → Not processed further until cleared

DECISION CODES:
  ┌─────────────────────────────────────────────────────┐
  │ Code                    │ Next step                 │
  ├─────────────────────────┼───────────────────────────┤
  │ QUALITY_PASSED          │ → Deduplication           │
  │ QUALITY_WARN            │ → Deduplication + logged  │
  │ QUALITY_REVIEW          │ → Supervisor queue        │
  │ QUALITY_REJECTED        │ → Officer notified        │
  │ FRAUD_FLAG              │ → Security team           │
  └─────────────────────────┴───────────────────────────┘
```

---

## 24.11 Quality Check Metrics and Monitoring

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Metrics Dashboard                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PER REGISTRATION CENTER (daily report):                     │
│                                                              │
│  Fingerprint metrics:                                        │
│  ├── average_nfiq2_score (target: ≥ 65)                    │
│  ├── below_minimum_rate (target: < 5%)                     │
│  ├── recapture_rate (target: < 15%)                        │
│  └── exception_rate (target: < 2%)                         │
│                                                              │
│  Face metrics:                                               │
│  ├── average_face_quality_score (target: ≥ 70)             │
│  ├── icao_compliance_rate (target: > 95%)                  │
│  ├── recapture_rate (target: < 10%)                        │
│  └── liveness_fail_rate (target: < 1%)                    │
│                                                              │
│  Iris metrics:                                               │
│  ├── average_iris_quality_score (target: ≥ 70)             │
│  ├── usable_area_average (target: ≥ 70%)                   │
│  └── recapture_rate (target: < 10%)                        │
│                                                              │
│  Demographic metrics:                                        │
│  ├── ocr_mismatch_rate (target: < 3%)                      │
│  ├── format_error_rate (target: < 1%)                      │
│  └── document_rejection_rate (target: < 5%)               │
│                                                              │
│  CLOUDWATCH ALARMS:                                          │
│  ├── average_nfiq2 < 50 for 1 hour at any center          │
│  │   → Alert: equipment issue / officer training need      │
│  ├── icao_compliance_rate < 80% for 1 hour                │
│  │   → Alert: camera misconfigured / lighting problem      │
│  ├── liveness_fail_rate > 5% for 30 min                   │
│  │   → Alert: possible spoofing attempt                   │
│  ├── ocr_mismatch_rate > 10% for 1 hour                   │
│  │   → Alert: officer entering data incorrectly           │
│  └── fraud_flag_rate > 1% for 1 day                       │
│      → Alert: investigate center for organized fraud      │
│                                                              │
│  QUALITY HEATMAP (weekly management report):                 │
│  Shows per-center quality scores on a map:                  │
│  ├── GREEN:  All metrics at target                         │
│  ├── YELLOW: 1–2 metrics below target (investigate)        │
│  └── RED:    Multiple metrics below minimum (intervene)    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.12 Quality Feedback Loop

Quality checks are only useful if they drive improvement. A feedback loop connects quality metrics back to operator training and equipment maintenance:

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Improvement Feedback Loop               │
└──────────────────────────────────────────────────────────────┘

STEP 1: Detect quality issue
  CloudWatch alarm:
  CENTER-005 average NFIQ2 < 50 for 3 consecutive days
         │
         ▼
STEP 2: Investigate
  Operations team reviews:
  ├── Which officers have low NFIQ2 captures?
  ├── Is it all fingers or specific fingers?
  ├── Is it one officer or all officers at center?
  └── When did it start? (sudden drop = equipment issue)
         │
         ▼
STEP 3: Root cause
  Possible causes:
  ├── Dirty platen (clean with alcohol → immediate fix)
  ├── Damaged platen (scratched → replace device)
  ├── Officer technique (rolling too fast → training)
  ├── Population factor (many laborers → adjust process)
  └── Algorithm calibration (quality SDK update needed)
         │
         ▼
STEP 4: Remediation
  ├── Equipment: schedule maintenance / replacement
  ├── Training: targeted session for specific officers
  ├── Process: update guidance for specific populations
  └── Software: update quality SDK / thresholds
         │
         ▼
STEP 5: Verify improvement
  Monitor next 7 days:
  ├── NFIQ2 rising back to target? → remediation worked
  └── Still low? → escalate investigation
         │
         ▼
STEP 6: Document and close
  Incident report:
  ├── What was the issue?
  ├── Root cause identified?
  ├── What was the fix?
  └── How many enrollments affected?
      (these may need re-enrollment outreach campaign)
```

---

## 24.13 Quality Check Error Codes

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Check Error Codes                       │
├────────────────┬─────────────────────────────────────────────┤
│ Code           │ Meaning                                    │
├────────────────┼─────────────────────────────────────────────┤
│ QC_FP_001      │ Fingerprint NFIQ2 below hard minimum (< 40)│
│ QC_FP_002      │ Fingerprint image blank / not captured     │
│ QC_FP_003      │ Fewer than 8 fingers above minimum quality │
│ QC_FP_004      │ Duplicate finger detected in set          │
│ QC_FP_005      │ Finger position inconsistent with image   │
│ QC_FC_001      │ No face detected in image                 │
│ QC_FC_002      │ Multiple faces detected                   │
│ QC_FC_003      │ Face pose out of ICAO range               │
│ QC_FC_004      │ Eyes not open / not visible               │
│ QC_FC_005      │ Face quality score below minimum (< 50)   │
│ QC_FC_006      │ Face-document photo mismatch              │
│ QC_IR_001      │ Iris not detected in image                │
│ QC_IR_002      │ Usable iris area below minimum (< 50%)    │
│ QC_IR_003      │ Iris sharpness below threshold            │
│ QC_IR_004      │ Pupil dilation ratio out of range        │
│ QC_DM_001      │ Required demographic field missing        │
│ QC_DM_002      │ Field format validation failed            │
│ QC_DM_003      │ Name OCR mismatch with document          │
│ QC_DM_004      │ DOB mismatch with document               │
│ QC_DC_001      │ Document scan below minimum resolution    │
│ QC_DC_002      │ MRZ check digit failure                  │
│ QC_DC_003      │ Document expired                         │
│ QC_DC_004      │ OCR confidence below threshold           │
│ QC_CM_001      │ Age estimated from face inconsistent      │
│ QC_CM_002      │ Cross-modality fraud indicator detected   │
└────────────────┴─────────────────────────────────────────────┘
```

---

## 24.14 Complete Quality Check Flow — End to End

```
┌──────────────────────────────────────────────────────────────┐
│              Complete Quality Check Flow                     │
└──────────────────────────────────────────────────────────────┘

ENROLLMENT PACKET RECEIVED (S3 upload)
         │
         ▼
SQS → Registration Processor (ECS Fargate)
         │
    ┌────┴──────────────────────────────────────────────┐
    │                                                   │
    ▼                                                   ▼
MDS Signature Verify                       Packet Structure Check
(device cert validation)                   (all fields present?)
         │                                             │
         └──────────────┬────────────────────────────┘
                        │
                        ▼
              Decrypt packet (KMS)
                        │
          ┌─────────────┼───────────────────┐
          │             │                   │
          ▼             ▼                   ▼
  Fingerprint      Face Quality         Iris Quality
  Quality Check    Check                Check
  (NFIQ2 × 10)    (ICAO + ISO          (ISO 29794-6)
                   19794-5)
          │             │                   │
          └─────────────┼───────────────────┘
                        │
                        ▼
              Demographic Quality Check
              (format + OCR cross-check)
                        │
                        ▼
              Document Quality Check
              (scan + MRZ + OCR confidence)
                        │
                        ▼
              Cross-Modality Checks
              (face-doc match + age + fraud)
                        │
                        ▼
              Quality Decision Engine
                        │
         ┌──────────────┼──────────────────────┐
         │              │                      │
         ▼              ▼                      ▼
    PASSED          REVIEW              REJECTED/FRAUD
         │              │                      │
         ▼              ▼                      ▼
  → Deduplication  → Supervisor         → Officer/Security
    (next stage)     queue                 notification
                       │                      │
                       ▼                      ▼
                  Supervisor            DLQ + alarm
                  approves or           + audit log
                  rejects
                       │
                       ▼
               APPROVED → Dedup
               REJECTED → Officer
```

---

## 24.15 AWS Architecture for Quality Checks

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Check AWS Architecture                  │
└──────────────────────────────────────────────────────────────┘

SQS (nbis-enrollment-received)
         │
         ▼
Quality Check Service (ECS Fargate)
         │
         ├──► S3: retrieve decrypted packet contents
         │
         ├──► NFIQ2 SDK container:
         │    Fingerprint quality per finger
         │
         ├──► Amazon Rekognition:
         │    ├── DetectFaces: face quality + ICAO check
         │    └── CompareFaces: face vs document photo
         │
         ├──► Iris Quality SDK container:
         │    ISO 29794-6 quality scoring
         │
         ├──► Amazon Textract:
         │    OCR + document field extraction
         │    MRZ line extraction
         │
         ├──► Custom MRZ validator (Lambda):
         │    Check digit verification
         │
         ├──► DynamoDB:
         │    Update enrollment status
         │    Store quality scores per modality
         │
         ├──► SQS routing:
         │    ├── PASSED → nbis-abis-requests
         │    ├── REVIEW → nbis-supervisor-review
         │    └── REJECTED → nbis-officer-notification
         │
         └──► CloudWatch:
              Publish quality metrics
              Per-center, per-modality, per-officer

QUALITY SCORES STORED IN DYNAMODB:
  DynamoDB table: nbis-enrollment-quality
  PK: packetId
  Attributes:
  ├── fingerprintScores: {finger: score} map
  ├── faceScore: overall + dimension breakdown
  ├── irisScores: {left: score, right: score}
  ├── demographicDecision: PASSED/FAILED
  ├── documentDecision: PASSED/FAILED
  ├── crossModalityFlags: list of flags
  ├── overallDecision: decision code
  └── processingTime: milliseconds
```

---

## 24.16 MOSIP Reference

```
┌──────────────────────────────────────────────────────────────┐
│              MOSIP Quality Check Reference                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  MOSIP Registration Processor quality stages:               │
│                                                              │
│  Stage 1: Packet Receiver                                   │
│  └── Validates packet structure and signature               │
│                                                              │
│  Stage 2: Packet Validator                                  │
│  └── Schema validation, field completeness                  │
│                                                              │
│  Stage 3: Quality Classifier                                │
│  ├── Biometric quality check (NFIQ2, face, iris)           │
│  ├── Configurable quality thresholds (per country)         │
│  └── Pluggable: country replaces default SDK with own      │
│                                                              │
│  Stage 4: Demo Dedupe (demographic pre-check)               │
│  └── Quick demographic match check before ABIS             │
│      (saves ABIS calls for obvious duplicates)             │
│                                                              │
│  MOSIP quality threshold configuration:                     │
│  In MOSIP admin portal:                                     │
│  ├── fingerprint.quality.threshold = 60                    │
│  ├── iris.quality.threshold = 70                           │
│  ├── face.quality.threshold = 70                           │
│  └── Configurable without code change                      │
│                                                              │
│  MOSIP quality failure handling:                            │
│  ├── Failed packet → BIOMETRIC_QUALITY_CHECK_FAILED        │
│  ├── Notification: officer SMS/email (configurable)        │
│  └── Packet: stays in system for supervisor review         │
│                                                              │
│  MOSIP supervisor portal (Registration Supervisor):         │
│  ├── View all packets in REVIEW queue                      │
│  ├── See quality scores + reason for review               │
│  ├── Approve with documented reason                        │
│  └── Reject (triggers re-enrollment notification)         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.17 Key Terms

| Term | Definition |
|------|-----------|
| **Quality check** | Validation that captured data meets minimum standards for reliable use |
| **Client-side check** | Real-time quality check at the enrollment center (immediate feedback) |
| **Server-side check** | Authoritative quality check in the back-end pipeline (post-submission) |
| **NFIQ2** | NIST Fingerprint Image Quality 2 — fingerprint quality standard (0–100) |
| **ISO 29794-6** | International standard for iris image quality metrics |
| **ISO 19794-5** | International standard for face image data and quality |
| **ICAO compliance** | Face image meets ICAO Doc 9303 requirements for identity documents |
| **OCR cross-check** | Comparing OCR-extracted document data with officer-entered demographics |
| **MRZ validation** | Verifying check digits in the passport machine-readable zone |
| **Cross-modality check** | Comparing consistency between different biometric modalities |
| **Face-document match** | Comparing enrolled face photo against document photo using face recognition |
| **Duplicate finger detection** | Checking if same fingerprint submitted for multiple positions |
| **Quality decision engine** | System that combines all check results into a final enrollment decision |
| **QUALITY_PASSED** | All checks passed — proceed to deduplication |
| **QUALITY_REVIEW** | Some issues — human supervisor must decide |
| **QUALITY_REJECTED** | Fundamental failure — enrollment must be redone |
| **FRAUD_FLAG** | Indicators of deliberate fraud detected — escalate to security |
| **Quality heatmap** | Visual map showing per-center quality performance |
| **Recapture rate** | Percentage of biometric captures requiring a second attempt |
| **Feedback loop** | Process connecting quality metrics to training and equipment maintenance |

---

## 24.18 Key Takeaways

- **Quality checks happen twice** — client-side for immediate feedback (officer can recapture now) and server-side for the authoritative decision (after submission). Never rely on client-side alone.
- **Hard minimums are hard for a reason** — below NFIQ2 40 or iris quality 50, the template is genuinely unreliable. Supervisor overrides below these levels must be documented with a specific, auditable reason.
- **Cross-modality checks catch fraud that single-modality checks miss** — a duplicate finger in the same packet, or a face that does not match the passport photo, are caught only when you compare across modalities.
- **Quality metrics are a center management tool** — a center with consistently low NFIQ2 scores has either a dirty platen, a broken scanner, undertrained officers, or a population with occupational wear. The metric tells you something is wrong; investigation reveals which.
- **OCR cross-check is not a duplicate of document verification** — it specifically checks whether what the officer typed matches what the document actually says. Officers make data entry errors. This check catches them before the UIN is issued.
- **Fraud flags are escalations, not automatic rejections** — face-document mismatch could mean a bad passport photo, not a fraudster. Age inconsistency could mean the citizen looks much younger than their DOB. Human review is required before any fraud-flagged packet is rejected.
- **Quality thresholds are configurable** — MOSIP allows country-specific threshold settings without code changes. Different populations have different biometric characteristics. What works in Sweden may not be appropriate for a country with many manual laborers.
- **The quality feedback loop is as important as the check itself** — quality checks without corrective action are just data. Track, alarm, investigate, remediate, and verify improvement.

---

## 24.19 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 25 | Enrollment Validation — packet validation, completeness, business rules |
| Chapter 26 | Duplicate Detection — ABIS integration, 1:N search, dedup workflow |
| Chapter 27 | Approval Workflow — supervisor review, manual decisions, audit trail |

---

*Chapter 24 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*# Chapter 24 — Quality Checks

> **Part III — Enrollment**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 24.1 What are Quality Checks?

**Quality checks** are systematic validations applied to every piece of data captured during enrollment — biometrics, demographics, documents, and signatures — to ensure that the data is good enough to serve its purpose reliably for the lifetime of the identity record.

Quality checking is the NBIS's internal quality assurance mechanism. It answers the question: **"Is this data good enough that we can be confident it will work correctly for authentication and deduplication for the next 10–30 years?"**

```
┌──────────────────────────────────────────────────────────────┐
│              Why Quality Checks Are Critical                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  A fingerprint enrolled with NFIQ2 = 35 (below minimum):   │
│  ├── Authentication failure rate: 15–20%                   │
│  ├── Citizen attends center multiple times to re-enroll    │
│  ├── Government spends money on re-enrollment support      │
│  └── Citizen loses trust in the identity system            │
│                                                              │
│  An iris enrolled with quality = 40 (below minimum):       │
│  ├── IrisCode is unreliable                                │
│  ├── Border crossing failures                               │
│  └── Legitimate citizen treated as unknown person          │
│                                                              │
│  A name with OCR cross-check failure accepted anyway:       │
│  ├── Name on ID card does not match civil registry         │
│  ├── Bank cannot complete KYC (name mismatch)              │
│  └── Citizen must go through legal name correction process │
│                                                              │
│  Quality checks at enrollment are CHEAPER than fixing       │
│  problems after UINs are issued. The cost of a 2-minute    │
│  recapture at enrollment is infinitely cheaper than years  │
│  of authentication failures and re-enrollment campaigns.   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.2 Two Phases of Quality Checking

Quality checks happen at two distinct points in the enrollment pipeline:

```
┌──────────────────────────────────────────────────────────────┐
│              Two-Phase Quality Check Design                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PHASE 1: CLIENT-SIDE (at registration center, real-time)   │
│  ──────────────────────────────────────────────────────     │
│  When: immediately after each biometric capture             │
│  Who: Registration Client software                          │
│  Goal: immediate feedback → officer can recapture now       │
│                                                              │
│  What it checks:                                             │
│  ├── NFIQ2 per finger (quick algorithm)                    │
│  ├── Basic face ICAO checks (pose, eyes open)              │
│  ├── Basic iris quality (area, sharpness)                  │
│  ├── Signature stroke count + area                         │
│  └── Required fields present (demographic completeness)    │
│                                                              │
│  Speed: must be < 2 seconds (real-time feedback)           │
│  Accuracy: good enough for immediate guidance              │
│  Algorithm: lightweight, runs on registration workstation  │
│                                                              │
│  PHASE 2: SERVER-SIDE (back-end pipeline, post-submission)  │
│  ──────────────────────────────────────────────────────     │
│  When: after encrypted packet uploaded to S3               │
│  Who: Registration Processor (ECS + Lambda)                 │
│  Goal: authoritative quality determination                  │
│                                                              │
│  What it checks:                                             │
│  ├── Full NFIQ2 (complete algorithm, more accurate)        │
│  ├── Full face quality (all ICAO dimensions)               │
│  ├── Full iris quality (all ISO 29794-6 dimensions)        │
│  ├── OCR cross-check (document vs entered demographics)    │
│  ├── MRZ validation (check digit verification)             │
│  ├── Document authenticity indicators                      │
│  ├── Signature quality (server-side assessment)            │
│  └── Cross-modality consistency checks                     │
│                                                              │
│  Speed: minutes acceptable (async pipeline)                │
│  Accuracy: authoritative — determines pass/fail            │
│  Algorithm: full production-grade algorithms               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.3 Biometric Quality Standards — Summary

```
┌──────────────────────────────────────────────────────────────┐
│              Biometric Quality Thresholds Reference          │
├────────────────────┬──────────────┬───────────┬─────────────┤
│ Modality           │ Standard     │ Minimum   │ Target      │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Fingerprint        │ NFIQ2        │ 40 / 100  │ 60 / 100   │
│ (per finger)       │ (NIST)       │           │            │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Fingerprint        │ NFIQ2        │ Average   │ Average     │
│ (overall set)      │ average      │ ≥ 50      │ ≥ 65       │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Iris (per eye)     │ ISO 29794-6  │ 50 / 100  │ 70 / 100   │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Face (overall)     │ ISO 19794-5  │ 50 / 100  │ 70 / 100   │
│                    │ + ICAO       │           │            │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Face (pose)        │ ICAO 9303    │ ≤ ±5°     │ ≤ ±3°      │
│                    │              │ yaw/pitch/│            │
│                    │              │ roll      │            │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Signature          │ Internal     │ ≥ 3       │ ≥ 5        │
│ (stroke count)     │              │ strokes   │ strokes    │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Document scan      │ Internal     │ ≥ 300 DPI │ ≥ 600 DPI  │
│                    │              │ legible   │            │
└────────────────────┴──────────────┴───────────┴────────────┘
```

---

## 24.4 Fingerprint Quality Check — Deep Dive

```
┌──────────────────────────────────────────────────────────────┐
│              Fingerprint Quality Check Pipeline              │
└──────────────────────────────────────────────────────────────┘

INPUT: 10 fingerprint images (WSQ compressed, 500 PPI)
         │
         ▼
FOR EACH FINGER:

  CHECK 1: Image completeness
  ├── Is the image size correct? (not truncated)
  ├── Is the image format valid WSQ?
  └── Is the capture device certified? (MDS signature check)

  CHECK 2: NFIQ2 quality score
  ├── Run NFIQ2 algorithm on each finger image
  ├── Score 0–100 returned for each finger
  └── Apply thresholds:
      < 20:  UNUSABLE → mandatory recapture
      20–39: POOR     → strongly recommend recapture
      40–59: FAIR     → warn, accept with supervisor
      60–74: GOOD     → accept
      75+:   EXCELLENT → accept

  CHECK 3: Pattern detection
  └── Can a ridge pattern be detected?
      (some images are blank / all-white = pad not touched)

  CHECK 4: Finger position consistency
  └── Does the image match the declared finger position?
      (a thumb image submitted as index finger → flag)
      (uses aspect ratio + pattern type classification)

OUTPUT PER FINGER:
  {
    "finger":     "RIGHT_INDEX",
    "nfiq2Score": 78,
    "quality":    "GOOD",
    "decision":   "ACCEPTED",
    "attempts":   1
  }

OVERALL FINGERPRINT QUALITY DECISION:
  ├── ALL 10 fingers ≥ 40:   PASS
  ├── ≥ 8 fingers ≥ 40:      PASS (2 exceptions documented)
  ├── < 8 fingers ≥ 40:      FAIL → supervisor review
  └── Average NFIQ2 < 50:    WARN → supervisor recommendation

QUALITY REPORT SUMMARY:
  {
    "fingerprintQuality": {
      "totalFingers":     10,
      "passed":           9,
      "failed":           1,
      "failedFingers":    ["RIGHT_RING"],
      "averageNFIQ2":     74.3,
      "overallDecision":  "PASS_WITH_EXCEPTION",
      "exceptionReason":  "RIGHT_RING below threshold after
                          3 attempts — skin condition noted"
    }
  }
```

---

## 24.5 Face Quality Check — Deep Dive

```
┌──────────────────────────────────────────────────────────────┐
│              Face Quality Check Pipeline                     │
└──────────────────────────────────────────────────────────────┘

INPUT: Face image (JPEG/JPEG2000, minimum 600×800px)
         │
         ▼
CHECK 1: Face detection
  ├── Is exactly one face detected in the image?
  ├── Zero faces → reject (no face captured)
  └── Multiple faces → reject (someone else in frame)

CHECK 2: Face size and position (geometric)
  ├── Face bounding box occupies 70–80% of frame height?
  ├── Face centered within ±10% of frame?
  └── Eye-to-eye distance ≥ 90 pixels?

CHECK 3: Pose angles (ICAO compliance)
  ├── Yaw (left-right rotation): ≤ ±5°?
  ├── Pitch (chin up/down): ≤ ±5°?
  └── Roll (tilt): ≤ ±5°?

CHECK 4: Eye quality
  ├── Both eyes detected?
  ├── Left eye open score ≥ 70?
  ├── Right eye open score ≥ 70?
  └── Eyes looking at camera (gaze direction check)?

CHECK 5: Lighting quality
  ├── Overall brightness within acceptable range?
  ├── Both sides of face equally lit (symmetry)?
  ├── No overexposed regions (washed out)?
  └── No underexposed regions (too dark)?

CHECK 6: Sharpness
  ├── Focus score ≥ 70?
  └── No motion blur detected?

CHECK 7: Occlusion
  ├── No mask covering nose/mouth?
  ├── Hair not covering eyes?
  └── No hand or object in front of face?

CHECK 8: Expression
  ├── Mouth closed (or near-closed)?
  └── Neutral expression (no extreme emotion)?

CHECK 9: Background
  └── Background sufficiently uniform?
      (high texture background = may confuse face detection)

COMBINED QUALITY SCORE:
  Weighted average across all dimensions:
  ├── Pose:       30% weight (ICAO critical)
  ├── Eyes:       25% weight (matching critical)
  ├── Lighting:   20% weight
  ├── Sharpness:  15% weight
  └── Occlusion:  10% weight

  Score ≥ 70: PASS
  Score 50–69: WARN → officer prompted to retake
  Score < 50:  FAIL → mandatory retake

FACE QUALITY REPORT:
  {
    "faceQuality": {
      "faceDetected":      true,
      "faceCount":         1,
      "poseAngles": {
        "yaw":   2.3,
        "pitch": -1.1,
        "roll":  0.8
      },
      "eyesOpen":          true,
      "lightingScore":     82.1,
      "sharpnessScore":    88.4,
      "occlusion":         "NONE",
      "overallScore":      84.7,
      "icaoCompliant":     true,
      "decision":          "ACCEPTED"
    }
  }
```

---

## 24.6 Iris Quality Check — Deep Dive

```
┌──────────────────────────────────────────────────────────────┐
│              Iris Quality Check Pipeline                     │
└──────────────────────────────────────────────────────────────┘

INPUT: Two iris NIR images (left + right, 640×480px minimum)
         │
         ▼
FOR EACH EYE:

  CHECK 1: Image validity
  ├── Is image format correct?
  ├── Is image not corrupted?
  └── Is MDS device signature valid?

  CHECK 2: Eye/iris detection
  ├── Is an eye detected in the image?
  ├── Is the iris boundary locatable?
  └── Is the pupil detectable?

  CHECK 3: Usable iris area
  ├── Segment iris: locate pupil + limbus boundaries
  ├── Exclude eyelid + eyelash occlusion regions
  ├── Calculate: usable area / total iris area
  └── Usable area ≥ 50%? (minimum)
      Usable area ≥ 70%? (target)

  CHECK 4: Iris sharpness
  ├── Measure frequency content of iris texture region
  ├── Blurry image: low frequency → low sharpness score
  └── Sharpness score ≥ 70?

  CHECK 5: Pupil dilation ratio
  ├── pupil_diameter / iris_diameter = ratio
  ├── Acceptable range: 0.2 – 0.7
  ├── < 0.2: unusual (very bright light / medical)
  └── > 0.7: very dilated (very dark / medication)

  CHECK 6: Gaze direction
  ├── Is iris centered in the image?
  ├── Off-center iris = subject not looking at camera
  └── Acceptable: < 10% off-center

  CHECK 7: Motion blur
  └── Blur score < 10? (subject was still during capture)

COMBINED IRIS QUALITY SCORE:
  {
    "irisQuality": {
      "leftEye": {
        "usableArea":     73.4,
        "sharpness":      82.1,
        "pupilRatio":     0.42,
        "gazeOffset":     3.2,
        "motionBlur":     1.8,
        "overallScore":   78.6,
        "decision":       "ACCEPTED"
      },
      "rightEye": {
        "usableArea":     68.9,
        "sharpness":      79.3,
        "pupilRatio":     0.44,
        "gazeOffset":     4.1,
        "motionBlur":     2.1,
        "overallScore":   74.2,
        "decision":       "ACCEPTED"
      }
    }
  }
```

---

## 24.7 Demographic Quality Checks

```
┌──────────────────────────────────────────────────────────────┐
│              Demographic Quality Check Pipeline              │
└──────────────────────────────────────────────────────────────┘

FOR EACH DEMOGRAPHIC FIELD:

  CHECK 1: Required field present
  └── Is this field populated?
      Empty required field → reject packet

  CHECK 2: Format validation
  ├── Name: regex — only allowed characters?
  ├── DOB:  ISO 8601 format? valid calendar date?
  ├── Mobile: E.164 format? valid country code?
  └── Email: RFC 5322 format?

  CHECK 3: Value range validation
  ├── DOB: not in future, not > 150 years ago
  ├── Name length: 1–150 characters
  └── Age-based rules:
      ├── Age < 5: no fingerprints expected
      ├── Age < 18: guardian consent required
      └── Age > 100: supervisor approval required

  CHECK 4: Cross-field consistency
  ├── fullName == firstName + middleName + lastName?
  ├── Current address == Permanent if sameAsPermanent?
  └── DOB consistent with stated age (if age also given)?

  CHECK 5: OCR cross-check (document vs entered)
  ├── Name: normalized comparison
  │   ├── Match score ≥ 80%: ✅ PASS
  │   ├── Match score 60–79%: ⚠️ WARN (officer confirm)
  │   └── Match score < 60%: ❌ FAIL (must correct)
  └── DOB: exact match required
      ├── Match: ✅ PASS
      └── Mismatch: ❌ FAIL (officer must re-check document)

  CHECK 6: Blacklist check
  ├── Is this name + DOB combination flagged?
  │   (fraud watchlist, court order, etc.)
  └── Flagged → escalate to supervisor (do not auto-reject)

DEMOGRAPHIC QUALITY REPORT:
  {
    "demographicQuality": {
      "requiredFieldsComplete": true,
      "formatValidation":       "PASSED",
      "ocrCrossCheck": {
        "nameMatchScore":  91.3,
        "nameDecision":    "PASSED",
        "dobMatch":        true,
        "dobDecision":     "PASSED"
      },
      "crossFieldConsistency":  "PASSED",
      "blacklistCheck":         "CLEAR",
      "overallDecision":        "PASSED"
    }
  }
```

---

## 24.8 Document Quality Checks

```
┌──────────────────────────────────────────────────────────────┐
│              Document Quality Check Pipeline                 │
└──────────────────────────────────────────────────────────────┘

FOR EACH SCANNED DOCUMENT:

  CHECK 1: Scan quality
  ├── Resolution ≥ 300 DPI?
  ├── Image not blurry (sharpness score)?
  ├── Image not overexposed / underexposed?
  └── Both sides scanned (if double-sided document)?

  CHECK 2: Document type recognition
  ├── Does the scan match the declared document type?
  │   (passport template vs ID card template)
  └── Can fields be located in expected positions?

  CHECK 3: MRZ validation (if applicable)
  ├── MRZ lines present at bottom of page?
  ├── MRZ parsed successfully?
  ├── Check digits valid for all fields?
  └── Extracted data (name, DOB) matches entered data?

  CHECK 4: Document validity
  ├── Issue date in the past?
  ├── Expiry date in the future? (not expired)
  ├── Issue date < Expiry date? (logical)
  └── Document number format valid for issuing country?

  CHECK 5: Tamper indicators (automated)
  ├── Can OCR extract text cleanly?
  │   (tampered text: different font, color artifacts)
  ├── Are image statistics consistent?
  │   (copy-paste substitution: different JPEG artifacts)
  └── MRZ inconsistency:
      document number in MRZ ≠ printed number → flag

  CHECK 6: OCR confidence
  ├── Per-field confidence score from OCR engine
  ├── Low confidence (< 60): flag for officer re-scan
  └── Overall OCR confidence ≥ 70?

DOCUMENT QUALITY REPORT:
  {
    "documentQuality": {
      "scanResolution":    600,
      "scanSharpness":     88.2,
      "documentType":      "PASSPORT",
      "documentRecognized":true,
      "mrzValid":          true,
      "mrzCheckDigits":    "ALL_VALID",
      "documentExpired":   false,
      "ocrConfidence":     94.1,
      "tamperIndicators":  "NONE",
      "overallDecision":   "PASSED"
    }
  }
```

---

## 24.9 Cross-Modality Consistency Checks

After individual modality checks pass, cross-modal checks verify that the biometrics from different modalities are internally consistent:

```
┌──────────────────────────────────────────────────────────────┐
│              Cross-Modality Consistency Checks               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  CHECK 1: Face-to-document photo match                       │
│  ├── Compare enrolled face vs face in document photo        │
│  ├── Tool: Amazon Rekognition CompareFaces API              │
│  ├── Similarity < 70%: FLAG → supervisor review            │
│  │   (possible: wrong person using genuine document)        │
│  └── Similarity ≥ 70%: PASS                                │
│                                                              │
│  CHECK 2: Age consistency                                    │
│  ├── Estimated age from face (AI model) vs stated DOB      │
│  ├── Difference > 15 years: FLAG (unusual, not reject)     │
│  └── Example: stated DOB = 1960, face age estimate = 35    │
│      → 30-year discrepancy → flag for review               │
│                                                              │
│  CHECK 3: Gender consistency                                 │
│  ├── AI gender estimate from face vs stated gender          │
│  ├── Mismatch: FLAG (informational — not grounds to reject) │
│  └── Note: gender expression ≠ legal gender                │
│      Do NOT reject based on gender estimate mismatch.       │
│      Only flag for human awareness.                         │
│                                                              │
│  CHECK 4: Biometric set completeness                         │
│  ├── Are all expected biometrics present?                   │
│  ├── Exception flags for missing modalities?               │
│  │   (if missing without flag → reject packet)             │
│  └── Are exception flags authorized (supervisor approval)? │
│                                                              │
│  CHECK 5: Duplicate biometric within packet                  │
│  ├── Was the same fingerprint submitted for multiple       │
│  │   different finger positions?                           │
│  │   (fraud: one good finger submitted 10 times)           │
│  ├── Run quick pairwise match on submitted fingers         │
│  └── Any pair matches > 0.7 threshold → FRAUD FLAG        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.10 Quality Check Decision Matrix

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Check Decision Matrix                   │
└──────────────────────────────────────────────────────────────┘

ENROLLMENT PACKET QUALITY DECISION LOGIC:

  IF all biometrics PASS and all demographics PASS
  AND all documents PASS:
     → QUALITY_PASSED → proceed to deduplication

  ELIF minor issues (some biometrics in WARN range):
     → QUALITY_PASSED_WITH_WARNINGS
     → Logged, supervisor informed, proceed to dedup
     → Citizen may be contacted for re-enrollment later

  ELIF one or two biometrics FAIL below hard minimum:
     → QUALITY_REVIEW → supervisor queue
     Supervisor options:
     ├── APPROVE_WITH_EXCEPTION (document reason)
     └── REJECT_FOR_RECAPTURE → notify officer

  ELIF critical failure:
     (no primary document, face quality < 30, all
     fingerprints below 40 without documented exception)
     → QUALITY_REJECTED → packet rejected
     → Officer notified with specific reason codes
     → Citizen must return for re-enrollment

  ELIF fraud indicators:
     (face-document mismatch, duplicate fingers,
     MRZ manipulation, document tamper indicators)
     → FRAUD_FLAG → escalated to security team
     → Not processed further until cleared

DECISION CODES:
  ┌─────────────────────────────────────────────────────┐
  │ Code                    │ Next step                 │
  ├─────────────────────────┼───────────────────────────┤
  │ QUALITY_PASSED          │ → Deduplication           │
  │ QUALITY_WARN            │ → Deduplication + logged  │
  │ QUALITY_REVIEW          │ → Supervisor queue        │
  │ QUALITY_REJECTED        │ → Officer notified        │
  │ FRAUD_FLAG              │ → Security team           │
  └─────────────────────────┴───────────────────────────┘
```

---

## 24.11 Quality Check Metrics and Monitoring

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Metrics Dashboard                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PER REGISTRATION CENTER (daily report):                     │
│                                                              │
│  Fingerprint metrics:                                        │
│  ├── average_nfiq2_score (target: ≥ 65)                    │
│  ├── below_minimum_rate (target: < 5%)                     │
│  ├── recapture_rate (target: < 15%)                        │
│  └── exception_rate (target: < 2%)                         │
│                                                              │
│  Face metrics:                                               │
│  ├── average_face_quality_score (target: ≥ 70)             │
│  ├── icao_compliance_rate (target: > 95%)                  │
│  ├── recapture_rate (target: < 10%)                        │
│  └── liveness_fail_rate (target: < 1%)                    │
│                                                              │
│  Iris metrics:                                               │
│  ├── average_iris_quality_score (target: ≥ 70)             │
│  ├── usable_area_average (target: ≥ 70%)                   │
│  └── recapture_rate (target: < 10%)                        │
│                                                              │
│  Demographic metrics:                                        │
│  ├── ocr_mismatch_rate (target: < 3%)                      │
│  ├── format_error_rate (target: < 1%)                      │
│  └── document_rejection_rate (target: < 5%)               │
│                                                              │
│  CLOUDWATCH ALARMS:                                          │
│  ├── average_nfiq2 < 50 for 1 hour at any center          │
│  │   → Alert: equipment issue / officer training need      │
│  ├── icao_compliance_rate < 80% for 1 hour                │
│  │   → Alert: camera misconfigured / lighting problem      │
│  ├── liveness_fail_rate > 5% for 30 min                   │
│  │   → Alert: possible spoofing attempt                   │
│  ├── ocr_mismatch_rate > 10% for 1 hour                   │
│  │   → Alert: officer entering data incorrectly           │
│  └── fraud_flag_rate > 1% for 1 day                       │
│      → Alert: investigate center for organized fraud      │
│                                                              │
│  QUALITY HEATMAP (weekly management report):                 │
│  Shows per-center quality scores on a map:                  │
│  ├── GREEN:  All metrics at target                         │
│  ├── YELLOW: 1–2 metrics below target (investigate)        │
│  └── RED:    Multiple metrics below minimum (intervene)    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.12 Quality Feedback Loop

Quality checks are only useful if they drive improvement. A feedback loop connects quality metrics back to operator training and equipment maintenance:

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Improvement Feedback Loop               │
└──────────────────────────────────────────────────────────────┘

STEP 1: Detect quality issue
  CloudWatch alarm:
  CENTER-005 average NFIQ2 < 50 for 3 consecutive days
         │
         ▼
STEP 2: Investigate
  Operations team reviews:
  ├── Which officers have low NFIQ2 captures?
  ├── Is it all fingers or specific fingers?
  ├── Is it one officer or all officers at center?
  └── When did it start? (sudden drop = equipment issue)
         │
         ▼
STEP 3: Root cause
  Possible causes:
  ├── Dirty platen (clean with alcohol → immediate fix)
  ├── Damaged platen (scratched → replace device)
  ├── Officer technique (rolling too fast → training)
  ├── Population factor (many laborers → adjust process)
  └── Algorithm calibration (quality SDK update needed)
         │
         ▼
STEP 4: Remediation
  ├── Equipment: schedule maintenance / replacement
  ├── Training: targeted session for specific officers
  ├── Process: update guidance for specific populations
  └── Software: update quality SDK / thresholds
         │
         ▼
STEP 5: Verify improvement
  Monitor next 7 days:
  ├── NFIQ2 rising back to target? → remediation worked
  └── Still low? → escalate investigation
         │
         ▼
STEP 6: Document and close
  Incident report:
  ├── What was the issue?
  ├── Root cause identified?
  ├── What was the fix?
  └── How many enrollments affected?
      (these may need re-enrollment outreach campaign)
```

---

## 24.13 Quality Check Error Codes

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Check Error Codes                       │
├────────────────┬─────────────────────────────────────────────┤
│ Code           │ Meaning                                    │
├────────────────┼─────────────────────────────────────────────┤
│ QC_FP_001      │ Fingerprint NFIQ2 below hard minimum (< 40)│
│ QC_FP_002      │ Fingerprint image blank / not captured     │
│ QC_FP_003      │ Fewer than 8 fingers above minimum quality │
│ QC_FP_004      │ Duplicate finger detected in set          │
│ QC_FP_005      │ Finger position inconsistent with image   │
│ QC_FC_001      │ No face detected in image                 │
│ QC_FC_002      │ Multiple faces detected                   │
│ QC_FC_003      │ Face pose out of ICAO range               │
│ QC_FC_004      │ Eyes not open / not visible               │
│ QC_FC_005      │ Face quality score below minimum (< 50)   │
│ QC_FC_006      │ Face-document photo mismatch              │
│ QC_IR_001      │ Iris not detected in image                │
│ QC_IR_002      │ Usable iris area below minimum (< 50%)    │
│ QC_IR_003      │ Iris sharpness below threshold            │
│ QC_IR_004      │ Pupil dilation ratio out of range        │
│ QC_DM_001      │ Required demographic field missing        │
│ QC_DM_002      │ Field format validation failed            │
│ QC_DM_003      │ Name OCR mismatch with document          │
│ QC_DM_004      │ DOB mismatch with document               │
│ QC_DC_001      │ Document scan below minimum resolution    │
│ QC_DC_002      │ MRZ check digit failure                  │
│ QC_DC_003      │ Document expired                         │
│ QC_DC_004      │ OCR confidence below threshold           │
│ QC_CM_001      │ Age estimated from face inconsistent      │
│ QC_CM_002      │ Cross-modality fraud indicator detected   │
└────────────────┴─────────────────────────────────────────────┘
```

---

## 24.14 Complete Quality Check Flow — End to End

```
┌──────────────────────────────────────────────────────────────┐
│              Complete Quality Check Flow                     │
└──────────────────────────────────────────────────────────────┘

ENROLLMENT PACKET RECEIVED (S3 upload)
         │
         ▼
SQS → Registration Processor (ECS Fargate)
         │
    ┌────┴──────────────────────────────────────────────┐
    │                                                   │
    ▼                                                   ▼
MDS Signature Verify                       Packet Structure Check
(device cert validation)                   (all fields present?)
         │                                             │
         └──────────────┬────────────────────────────┘
                        │
                        ▼
              Decrypt packet (KMS)
                        │
          ┌─────────────┼───────────────────┐
          │             │                   │
          ▼             ▼                   ▼
  Fingerprint      Face Quality         Iris Quality
  Quality Check    Check                Check
  (NFIQ2 × 10)    (ICAO + ISO          (ISO 29794-6)
                   19794-5)
          │             │                   │
          └─────────────┼───────────────────┘
                        │
                        ▼
              Demographic Quality Check
              (format + OCR cross-check)
                        │
                        ▼
              Document Quality Check
              (scan + MRZ + OCR confidence)
                        │
                        ▼
              Cross-Modality Checks
              (face-doc match + age + fraud)
                        │
                        ▼
              Quality Decision Engine
                        │
         ┌──────────────┼──────────────────────┐
         │              │                      │
         ▼              ▼                      ▼
    PASSED          REVIEW              REJECTED/FRAUD
         │              │                      │
         ▼              ▼                      ▼
  → Deduplication  → Supervisor         → Officer/Security
    (next stage)     queue                 notification
                       │                      │
                       ▼                      ▼
                  Supervisor            DLQ + alarm
                  approves or           + audit log
                  rejects
                       │
                       ▼
               APPROVED → Dedup
               REJECTED → Officer
```

---

## 24.15 AWS Architecture for Quality Checks

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Check AWS Architecture                  │
└──────────────────────────────────────────────────────────────┘

SQS (nbis-enrollment-received)
         │
         ▼
Quality Check Service (ECS Fargate)
         │
         ├──► S3: retrieve decrypted packet contents
         │
         ├──► NFIQ2 SDK container:
         │    Fingerprint quality per finger
         │
         ├──► Amazon Rekognition:
         │    ├── DetectFaces: face quality + ICAO check
         │    └── CompareFaces: face vs document photo
         │
         ├──► Iris Quality SDK container:
         │    ISO 29794-6 quality scoring
         │
         ├──► Amazon Textract:
         │    OCR + document field extraction
         │    MRZ line extraction
         │
         ├──► Custom MRZ validator (Lambda):
         │    Check digit verification
         │
         ├──► DynamoDB:
         │    Update enrollment status
         │    Store quality scores per modality
         │
         ├──► SQS routing:
         │    ├── PASSED → nbis-abis-requests
         │    ├── REVIEW → nbis-supervisor-review
         │    └── REJECTED → nbis-officer-notification
         │
         └──► CloudWatch:
              Publish quality metrics
              Per-center, per-modality, per-officer

QUALITY SCORES STORED IN DYNAMODB:
  DynamoDB table: nbis-enrollment-quality
  PK: packetId
  Attributes:
  ├── fingerprintScores: {finger: score} map
  ├── faceScore: overall + dimension breakdown
  ├── irisScores: {left: score, right: score}
  ├── demographicDecision: PASSED/FAILED
  ├── documentDecision: PASSED/FAILED
  ├── crossModalityFlags: list of flags
  ├── overallDecision: decision code
  └── processingTime: milliseconds
```

---

## 24.16 MOSIP Reference

```
┌──────────────────────────────────────────────────────────────┐
│              MOSIP Quality Check Reference                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  MOSIP Registration Processor quality stages:               │
│                                                              │
│  Stage 1: Packet Receiver                                   │
│  └── Validates packet structure and signature               │
│                                                              │
│  Stage 2: Packet Validator                                  │
│  └── Schema validation, field completeness                  │
│                                                              │
│  Stage 3: Quality Classifier                                │
│  ├── Biometric quality check (NFIQ2, face, iris)           │
│  ├── Configurable quality thresholds (per country)         │
│  └── Pluggable: country replaces default SDK with own      │
│                                                              │
│  Stage 4: Demo Dedupe (demographic pre-check)               │
│  └── Quick demographic match check before ABIS             │
│      (saves ABIS calls for obvious duplicates)             │
│                                                              │
│  MOSIP quality threshold configuration:                     │
│  In MOSIP admin portal:                                     │
│  ├── fingerprint.quality.threshold = 60                    │
│  ├── iris.quality.threshold = 70                           │
│  ├── face.quality.threshold = 70                           │
│  └── Configurable without code change                      │
│                                                              │
│  MOSIP quality failure handling:                            │
│  ├── Failed packet → BIOMETRIC_QUALITY_CHECK_FAILED        │
│  ├── Notification: officer SMS/email (configurable)        │
│  └── Packet: stays in system for supervisor review         │
│                                                              │
│  MOSIP supervisor portal (Registration Supervisor):         │
│  ├── View all packets in REVIEW queue                      │
│  ├── See quality scores + reason for review               │
│  ├── Approve with documented reason                        │
│  └── Reject (triggers re-enrollment notification)         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.17 Key Terms

| Term | Definition |
|------|-----------|
| **Quality check** | Validation that captured data meets minimum standards for reliable use |
| **Client-side check** | Real-time quality check at the enrollment center (immediate feedback) |
| **Server-side check** | Authoritative quality check in the back-end pipeline (post-submission) |
| **NFIQ2** | NIST Fingerprint Image Quality 2 — fingerprint quality standard (0–100) |
| **ISO 29794-6** | International standard for iris image quality metrics |
| **ISO 19794-5** | International standard for face image data and quality |
| **ICAO compliance** | Face image meets ICAO Doc 9303 requirements for identity documents |
| **OCR cross-check** | Comparing OCR-extracted document data with officer-entered demographics |
| **MRZ validation** | Verifying check digits in the passport machine-readable zone |
| **Cross-modality check** | Comparing consistency between different biometric modalities |
| **Face-document match** | Comparing enrolled face photo against document photo using face recognition |
| **Duplicate finger detection** | Checking if same fingerprint submitted for multiple positions |
| **Quality decision engine** | System that combines all check results into a final enrollment decision |
| **QUALITY_PASSED** | All checks passed — proceed to deduplication |
| **QUALITY_REVIEW** | Some issues — human supervisor must decide |
| **QUALITY_REJECTED** | Fundamental failure — enrollment must be redone |
| **FRAUD_FLAG** | Indicators of deliberate fraud detected — escalate to security |
| **Quality heatmap** | Visual map showing per-center quality performance |
| **Recapture rate** | Percentage of biometric captures requiring a second attempt |
| **Feedback loop** | Process connecting quality metrics to training and equipment maintenance |

---

## 24.18 Key Takeaways

- **Quality checks happen twice** — client-side for immediate feedback (officer can recapture now) and server-side for the authoritative decision (after submission). Never rely on client-side alone.
- **Hard minimums are hard for a reason** — below NFIQ2 40 or iris quality 50, the template is genuinely unreliable. Supervisor overrides below these levels must be documented with a specific, auditable reason.
- **Cross-modality checks catch fraud that single-modality checks miss** — a duplicate finger in the same packet, or a face that does not match the passport photo, are caught only when you compare across modalities.
- **Quality metrics are a center management tool** — a center with consistently low NFIQ2 scores has either a dirty platen, a broken scanner, undertrained officers, or a population with occupational wear. The metric tells you something is wrong; investigation reveals which.
- **OCR cross-check is not a duplicate of document verification** — it specifically checks whether what the officer typed matches what the document actually says. Officers make data entry errors. This check catches them before the UIN is issued.
- **Fraud flags are escalations, not automatic rejections** — face-document mismatch could mean a bad passport photo, not a fraudster. Age inconsistency could mean the citizen looks much younger than their DOB. Human review is required before any fraud-flagged packet is rejected.
- **Quality thresholds are configurable** — MOSIP allows country-specific threshold settings without code changes. Different populations have different biometric characteristics. What works in Sweden may not be appropriate for a country with many manual laborers.
- **The quality feedback loop is as important as the check itself** — quality checks without corrective action are just data. Track, alarm, investigate, remediate, and verify improvement.

---

## 24.19 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 25 | Enrollment Validation — packet validation, completeness, business rules |
| Chapter 26 | Duplicate Detection — ABIS integration, 1:N search, dedup workflow |
| Chapter 27 | Approval Workflow — supervisor review, manual decisions, audit trail |

---

*Chapter 24 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*# Chapter 24 — Quality Checks

> **Part III — Enrollment**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 24.1 What are Quality Checks?

**Quality checks** are systematic validations applied to every piece of data captured during enrollment — biometrics, demographics, documents, and signatures — to ensure that the data is good enough to serve its purpose reliably for the lifetime of the identity record.

Quality checking is the NBIS's internal quality assurance mechanism. It answers the question: **"Is this data good enough that we can be confident it will work correctly for authentication and deduplication for the next 10–30 years?"**

```
┌──────────────────────────────────────────────────────────────┐
│              Why Quality Checks Are Critical                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  A fingerprint enrolled with NFIQ2 = 35 (below minimum):   │
│  ├── Authentication failure rate: 15–20%                   │
│  ├── Citizen attends center multiple times to re-enroll    │
│  ├── Government spends money on re-enrollment support      │
│  └── Citizen loses trust in the identity system            │
│                                                              │
│  An iris enrolled with quality = 40 (below minimum):       │
│  ├── IrisCode is unreliable                                │
│  ├── Border crossing failures                               │
│  └── Legitimate citizen treated as unknown person          │
│                                                              │
│  A name with OCR cross-check failure accepted anyway:       │
│  ├── Name on ID card does not match civil registry         │
│  ├── Bank cannot complete KYC (name mismatch)              │
│  └── Citizen must go through legal name correction process │
│                                                              │
│  Quality checks at enrollment are CHEAPER than fixing       │
│  problems after UINs are issued. The cost of a 2-minute    │
│  recapture at enrollment is infinitely cheaper than years  │
│  of authentication failures and re-enrollment campaigns.   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.2 Two Phases of Quality Checking

Quality checks happen at two distinct points in the enrollment pipeline:

```
┌──────────────────────────────────────────────────────────────┐
│              Two-Phase Quality Check Design                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PHASE 1: CLIENT-SIDE (at registration center, real-time)   │
│  ──────────────────────────────────────────────────────     │
│  When: immediately after each biometric capture             │
│  Who: Registration Client software                          │
│  Goal: immediate feedback → officer can recapture now       │
│                                                              │
│  What it checks:                                             │
│  ├── NFIQ2 per finger (quick algorithm)                    │
│  ├── Basic face ICAO checks (pose, eyes open)              │
│  ├── Basic iris quality (area, sharpness)                  │
│  ├── Signature stroke count + area                         │
│  └── Required fields present (demographic completeness)    │
│                                                              │
│  Speed: must be < 2 seconds (real-time feedback)           │
│  Accuracy: good enough for immediate guidance              │
│  Algorithm: lightweight, runs on registration workstation  │
│                                                              │
│  PHASE 2: SERVER-SIDE (back-end pipeline, post-submission)  │
│  ──────────────────────────────────────────────────────     │
│  When: after encrypted packet uploaded to S3               │
│  Who: Registration Processor (ECS + Lambda)                 │
│  Goal: authoritative quality determination                  │
│                                                              │
│  What it checks:                                             │
│  ├── Full NFIQ2 (complete algorithm, more accurate)        │
│  ├── Full face quality (all ICAO dimensions)               │
│  ├── Full iris quality (all ISO 29794-6 dimensions)        │
│  ├── OCR cross-check (document vs entered demographics)    │
│  ├── MRZ validation (check digit verification)             │
│  ├── Document authenticity indicators                      │
│  ├── Signature quality (server-side assessment)            │
│  └── Cross-modality consistency checks                     │
│                                                              │
│  Speed: minutes acceptable (async pipeline)                │
│  Accuracy: authoritative — determines pass/fail            │
│  Algorithm: full production-grade algorithms               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.3 Biometric Quality Standards — Summary

```
┌──────────────────────────────────────────────────────────────┐
│              Biometric Quality Thresholds Reference          │
├────────────────────┬──────────────┬───────────┬─────────────┤
│ Modality           │ Standard     │ Minimum   │ Target      │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Fingerprint        │ NFIQ2        │ 40 / 100  │ 60 / 100   │
│ (per finger)       │ (NIST)       │           │            │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Fingerprint        │ NFIQ2        │ Average   │ Average     │
│ (overall set)      │ average      │ ≥ 50      │ ≥ 65       │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Iris (per eye)     │ ISO 29794-6  │ 50 / 100  │ 70 / 100   │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Face (overall)     │ ISO 19794-5  │ 50 / 100  │ 70 / 100   │
│                    │ + ICAO       │           │            │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Face (pose)        │ ICAO 9303    │ ≤ ±5°     │ ≤ ±3°      │
│                    │              │ yaw/pitch/│            │
│                    │              │ roll      │            │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Signature          │ Internal     │ ≥ 3       │ ≥ 5        │
│ (stroke count)     │              │ strokes   │ strokes    │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Document scan      │ Internal     │ ≥ 300 DPI │ ≥ 600 DPI  │
│                    │              │ legible   │            │
└────────────────────┴──────────────┴───────────┴────────────┘
```

---

## 24.4 Fingerprint Quality Check — Deep Dive

```
┌──────────────────────────────────────────────────────────────┐
│              Fingerprint Quality Check Pipeline              │
└──────────────────────────────────────────────────────────────┘

INPUT: 10 fingerprint images (WSQ compressed, 500 PPI)
         │
         ▼
FOR EACH FINGER:

  CHECK 1: Image completeness
  ├── Is the image size correct? (not truncated)
  ├── Is the image format valid WSQ?
  └── Is the capture device certified? (MDS signature check)

  CHECK 2: NFIQ2 quality score
  ├── Run NFIQ2 algorithm on each finger image
  ├── Score 0–100 returned for each finger
  └── Apply thresholds:
      < 20:  UNUSABLE → mandatory recapture
      20–39: POOR     → strongly recommend recapture
      40–59: FAIR     → warn, accept with supervisor
      60–74: GOOD     → accept
      75+:   EXCELLENT → accept

  CHECK 3: Pattern detection
  └── Can a ridge pattern be detected?
      (some images are blank / all-white = pad not touched)

  CHECK 4: Finger position consistency
  └── Does the image match the declared finger position?
      (a thumb image submitted as index finger → flag)
      (uses aspect ratio + pattern type classification)

OUTPUT PER FINGER:
  {
    "finger":     "RIGHT_INDEX",
    "nfiq2Score": 78,
    "quality":    "GOOD",
    "decision":   "ACCEPTED",
    "attempts":   1
  }

OVERALL FINGERPRINT QUALITY DECISION:
  ├── ALL 10 fingers ≥ 40:   PASS
  ├── ≥ 8 fingers ≥ 40:      PASS (2 exceptions documented)
  ├── < 8 fingers ≥ 40:      FAIL → supervisor review
  └── Average NFIQ2 < 50:    WARN → supervisor recommendation

QUALITY REPORT SUMMARY:
  {
    "fingerprintQuality": {
      "totalFingers":     10,
      "passed":           9,
      "failed":           1,
      "failedFingers":    ["RIGHT_RING"],
      "averageNFIQ2":     74.3,
      "overallDecision":  "PASS_WITH_EXCEPTION",
      "exceptionReason":  "RIGHT_RING below threshold after
                          3 attempts — skin condition noted"
    }
  }
```

---

## 24.5 Face Quality Check — Deep Dive

```
┌──────────────────────────────────────────────────────────────┐
│              Face Quality Check Pipeline                     │
└──────────────────────────────────────────────────────────────┘

INPUT: Face image (JPEG/JPEG2000, minimum 600×800px)
         │
         ▼
CHECK 1: Face detection
  ├── Is exactly one face detected in the image?
  ├── Zero faces → reject (no face captured)
  └── Multiple faces → reject (someone else in frame)

CHECK 2: Face size and position (geometric)
  ├── Face bounding box occupies 70–80% of frame height?
  ├── Face centered within ±10% of frame?
  └── Eye-to-eye distance ≥ 90 pixels?

CHECK 3: Pose angles (ICAO compliance)
  ├── Yaw (left-right rotation): ≤ ±5°?
  ├── Pitch (chin up/down): ≤ ±5°?
  └── Roll (tilt): ≤ ±5°?

CHECK 4: Eye quality
  ├── Both eyes detected?
  ├── Left eye open score ≥ 70?
  ├── Right eye open score ≥ 70?
  └── Eyes looking at camera (gaze direction check)?

CHECK 5: Lighting quality
  ├── Overall brightness within acceptable range?
  ├── Both sides of face equally lit (symmetry)?
  ├── No overexposed regions (washed out)?
  └── No underexposed regions (too dark)?

CHECK 6: Sharpness
  ├── Focus score ≥ 70?
  └── No motion blur detected?

CHECK 7: Occlusion
  ├── No mask covering nose/mouth?
  ├── Hair not covering eyes?
  └── No hand or object in front of face?

CHECK 8: Expression
  ├── Mouth closed (or near-closed)?
  └── Neutral expression (no extreme emotion)?

CHECK 9: Background
  └── Background sufficiently uniform?
      (high texture background = may confuse face detection)

COMBINED QUALITY SCORE:
  Weighted average across all dimensions:
  ├── Pose:       30% weight (ICAO critical)
  ├── Eyes:       25% weight (matching critical)
  ├── Lighting:   20% weight
  ├── Sharpness:  15% weight
  └── Occlusion:  10% weight

  Score ≥ 70: PASS
  Score 50–69: WARN → officer prompted to retake
  Score < 50:  FAIL → mandatory retake

FACE QUALITY REPORT:
  {
    "faceQuality": {
      "faceDetected":      true,
      "faceCount":         1,
      "poseAngles": {
        "yaw":   2.3,
        "pitch": -1.1,
        "roll":  0.8
      },
      "eyesOpen":          true,
      "lightingScore":     82.1,
      "sharpnessScore":    88.4,
      "occlusion":         "NONE",
      "overallScore":      84.7,
      "icaoCompliant":     true,
      "decision":          "ACCEPTED"
    }
  }
```

---

## 24.6 Iris Quality Check — Deep Dive

```
┌──────────────────────────────────────────────────────────────┐
│              Iris Quality Check Pipeline                     │
└──────────────────────────────────────────────────────────────┘

INPUT: Two iris NIR images (left + right, 640×480px minimum)
         │
         ▼
FOR EACH EYE:

  CHECK 1: Image validity
  ├── Is image format correct?
  ├── Is image not corrupted?
  └── Is MDS device signature valid?

  CHECK 2: Eye/iris detection
  ├── Is an eye detected in the image?
  ├── Is the iris boundary locatable?
  └── Is the pupil detectable?

  CHECK 3: Usable iris area
  ├── Segment iris: locate pupil + limbus boundaries
  ├── Exclude eyelid + eyelash occlusion regions
  ├── Calculate: usable area / total iris area
  └── Usable area ≥ 50%? (minimum)
      Usable area ≥ 70%? (target)

  CHECK 4: Iris sharpness
  ├── Measure frequency content of iris texture region
  ├── Blurry image: low frequency → low sharpness score
  └── Sharpness score ≥ 70?

  CHECK 5: Pupil dilation ratio
  ├── pupil_diameter / iris_diameter = ratio
  ├── Acceptable range: 0.2 – 0.7
  ├── < 0.2: unusual (very bright light / medical)
  └── > 0.7: very dilated (very dark / medication)

  CHECK 6: Gaze direction
  ├── Is iris centered in the image?
  ├── Off-center iris = subject not looking at camera
  └── Acceptable: < 10% off-center

  CHECK 7: Motion blur
  └── Blur score < 10? (subject was still during capture)

COMBINED IRIS QUALITY SCORE:
  {
    "irisQuality": {
      "leftEye": {
        "usableArea":     73.4,
        "sharpness":      82.1,
        "pupilRatio":     0.42,
        "gazeOffset":     3.2,
        "motionBlur":     1.8,
        "overallScore":   78.6,
        "decision":       "ACCEPTED"
      },
      "rightEye": {
        "usableArea":     68.9,
        "sharpness":      79.3,
        "pupilRatio":     0.44,
        "gazeOffset":     4.1,
        "motionBlur":     2.1,
        "overallScore":   74.2,
        "decision":       "ACCEPTED"
      }
    }
  }
```

---

## 24.7 Demographic Quality Checks

```
┌──────────────────────────────────────────────────────────────┐
│              Demographic Quality Check Pipeline              │
└──────────────────────────────────────────────────────────────┘

FOR EACH DEMOGRAPHIC FIELD:

  CHECK 1: Required field present
  └── Is this field populated?
      Empty required field → reject packet

  CHECK 2: Format validation
  ├── Name: regex — only allowed characters?
  ├── DOB:  ISO 8601 format? valid calendar date?
  ├── Mobile: E.164 format? valid country code?
  └── Email: RFC 5322 format?

  CHECK 3: Value range validation
  ├── DOB: not in future, not > 150 years ago
  ├── Name length: 1–150 characters
  └── Age-based rules:
      ├── Age < 5: no fingerprints expected
      ├── Age < 18: guardian consent required
      └── Age > 100: supervisor approval required

  CHECK 4: Cross-field consistency
  ├── fullName == firstName + middleName + lastName?
  ├── Current address == Permanent if sameAsPermanent?
  └── DOB consistent with stated age (if age also given)?

  CHECK 5: OCR cross-check (document vs entered)
  ├── Name: normalized comparison
  │   ├── Match score ≥ 80%: ✅ PASS
  │   ├── Match score 60–79%: ⚠️ WARN (officer confirm)
  │   └── Match score < 60%: ❌ FAIL (must correct)
  └── DOB: exact match required
      ├── Match: ✅ PASS
      └── Mismatch: ❌ FAIL (officer must re-check document)

  CHECK 6: Blacklist check
  ├── Is this name + DOB combination flagged?
  │   (fraud watchlist, court order, etc.)
  └── Flagged → escalate to supervisor (do not auto-reject)

DEMOGRAPHIC QUALITY REPORT:
  {
    "demographicQuality": {
      "requiredFieldsComplete": true,
      "formatValidation":       "PASSED",
      "ocrCrossCheck": {
        "nameMatchScore":  91.3,
        "nameDecision":    "PASSED",
        "dobMatch":        true,
        "dobDecision":     "PASSED"
      },
      "crossFieldConsistency":  "PASSED",
      "blacklistCheck":         "CLEAR",
      "overallDecision":        "PASSED"
    }
  }
```

---

## 24.8 Document Quality Checks

```
┌──────────────────────────────────────────────────────────────┐
│              Document Quality Check Pipeline                 │
└──────────────────────────────────────────────────────────────┘

FOR EACH SCANNED DOCUMENT:

  CHECK 1: Scan quality
  ├── Resolution ≥ 300 DPI?
  ├── Image not blurry (sharpness score)?
  ├── Image not overexposed / underexposed?
  └── Both sides scanned (if double-sided document)?

  CHECK 2: Document type recognition
  ├── Does the scan match the declared document type?
  │   (passport template vs ID card template)
  └── Can fields be located in expected positions?

  CHECK 3: MRZ validation (if applicable)
  ├── MRZ lines present at bottom of page?
  ├── MRZ parsed successfully?
  ├── Check digits valid for all fields?
  └── Extracted data (name, DOB) matches entered data?

  CHECK 4: Document validity
  ├── Issue date in the past?
  ├── Expiry date in the future? (not expired)
  ├── Issue date < Expiry date? (logical)
  └── Document number format valid for issuing country?

  CHECK 5: Tamper indicators (automated)
  ├── Can OCR extract text cleanly?
  │   (tampered text: different font, color artifacts)
  ├── Are image statistics consistent?
  │   (copy-paste substitution: different JPEG artifacts)
  └── MRZ inconsistency:
      document number in MRZ ≠ printed number → flag

  CHECK 6: OCR confidence
  ├── Per-field confidence score from OCR engine
  ├── Low confidence (< 60): flag for officer re-scan
  └── Overall OCR confidence ≥ 70?

DOCUMENT QUALITY REPORT:
  {
    "documentQuality": {
      "scanResolution":    600,
      "scanSharpness":     88.2,
      "documentType":      "PASSPORT",
      "documentRecognized":true,
      "mrzValid":          true,
      "mrzCheckDigits":    "ALL_VALID",
      "documentExpired":   false,
      "ocrConfidence":     94.1,
      "tamperIndicators":  "NONE",
      "overallDecision":   "PASSED"
    }
  }
```

---

## 24.9 Cross-Modality Consistency Checks

After individual modality checks pass, cross-modal checks verify that the biometrics from different modalities are internally consistent:

```
┌──────────────────────────────────────────────────────────────┐
│              Cross-Modality Consistency Checks               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  CHECK 1: Face-to-document photo match                       │
│  ├── Compare enrolled face vs face in document photo        │
│  ├── Tool: Amazon Rekognition CompareFaces API              │
│  ├── Similarity < 70%: FLAG → supervisor review            │
│  │   (possible: wrong person using genuine document)        │
│  └── Similarity ≥ 70%: PASS                                │
│                                                              │
│  CHECK 2: Age consistency                                    │
│  ├── Estimated age from face (AI model) vs stated DOB      │
│  ├── Difference > 15 years: FLAG (unusual, not reject)     │
│  └── Example: stated DOB = 1960, face age estimate = 35    │
│      → 30-year discrepancy → flag for review               │
│                                                              │
│  CHECK 3: Gender consistency                                 │
│  ├── AI gender estimate from face vs stated gender          │
│  ├── Mismatch: FLAG (informational — not grounds to reject) │
│  └── Note: gender expression ≠ legal gender                │
│      Do NOT reject based on gender estimate mismatch.       │
│      Only flag for human awareness.                         │
│                                                              │
│  CHECK 4: Biometric set completeness                         │
│  ├── Are all expected biometrics present?                   │
│  ├── Exception flags for missing modalities?               │
│  │   (if missing without flag → reject packet)             │
│  └── Are exception flags authorized (supervisor approval)? │
│                                                              │
│  CHECK 5: Duplicate biometric within packet                  │
│  ├── Was the same fingerprint submitted for multiple       │
│  │   different finger positions?                           │
│  │   (fraud: one good finger submitted 10 times)           │
│  ├── Run quick pairwise match on submitted fingers         │
│  └── Any pair matches > 0.7 threshold → FRAUD FLAG        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.10 Quality Check Decision Matrix

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Check Decision Matrix                   │
└──────────────────────────────────────────────────────────────┘

ENROLLMENT PACKET QUALITY DECISION LOGIC:

  IF all biometrics PASS and all demographics PASS
  AND all documents PASS:
     → QUALITY_PASSED → proceed to deduplication

  ELIF minor issues (some biometrics in WARN range):
     → QUALITY_PASSED_WITH_WARNINGS
     → Logged, supervisor informed, proceed to dedup
     → Citizen may be contacted for re-enrollment later

  ELIF one or two biometrics FAIL below hard minimum:
     → QUALITY_REVIEW → supervisor queue
     Supervisor options:
     ├── APPROVE_WITH_EXCEPTION (document reason)
     └── REJECT_FOR_RECAPTURE → notify officer

  ELIF critical failure:
     (no primary document, face quality < 30, all
     fingerprints below 40 without documented exception)
     → QUALITY_REJECTED → packet rejected
     → Officer notified with specific reason codes
     → Citizen must return for re-enrollment

  ELIF fraud indicators:
     (face-document mismatch, duplicate fingers,
     MRZ manipulation, document tamper indicators)
     → FRAUD_FLAG → escalated to security team
     → Not processed further until cleared

DECISION CODES:
  ┌─────────────────────────────────────────────────────┐
  │ Code                    │ Next step                 │
  ├─────────────────────────┼───────────────────────────┤
  │ QUALITY_PASSED          │ → Deduplication           │
  │ QUALITY_WARN            │ → Deduplication + logged  │
  │ QUALITY_REVIEW          │ → Supervisor queue        │
  │ QUALITY_REJECTED        │ → Officer notified        │
  │ FRAUD_FLAG              │ → Security team           │
  └─────────────────────────┴───────────────────────────┘
```

---

## 24.11 Quality Check Metrics and Monitoring

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Metrics Dashboard                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PER REGISTRATION CENTER (daily report):                     │
│                                                              │
│  Fingerprint metrics:                                        │
│  ├── average_nfiq2_score (target: ≥ 65)                    │
│  ├── below_minimum_rate (target: < 5%)                     │
│  ├── recapture_rate (target: < 15%)                        │
│  └── exception_rate (target: < 2%)                         │
│                                                              │
│  Face metrics:                                               │
│  ├── average_face_quality_score (target: ≥ 70)             │
│  ├── icao_compliance_rate (target: > 95%)                  │
│  ├── recapture_rate (target: < 10%)                        │
│  └── liveness_fail_rate (target: < 1%)                    │
│                                                              │
│  Iris metrics:                                               │
│  ├── average_iris_quality_score (target: ≥ 70)             │
│  ├── usable_area_average (target: ≥ 70%)                   │
│  └── recapture_rate (target: < 10%)                        │
│                                                              │
│  Demographic metrics:                                        │
│  ├── ocr_mismatch_rate (target: < 3%)                      │
│  ├── format_error_rate (target: < 1%)                      │
│  └── document_rejection_rate (target: < 5%)               │
│                                                              │
│  CLOUDWATCH ALARMS:                                          │
│  ├── average_nfiq2 < 50 for 1 hour at any center          │
│  │   → Alert: equipment issue / officer training need      │
│  ├── icao_compliance_rate < 80% for 1 hour                │
│  │   → Alert: camera misconfigured / lighting problem      │
│  ├── liveness_fail_rate > 5% for 30 min                   │
│  │   → Alert: possible spoofing attempt                   │
│  ├── ocr_mismatch_rate > 10% for 1 hour                   │
│  │   → Alert: officer entering data incorrectly           │
│  └── fraud_flag_rate > 1% for 1 day                       │
│      → Alert: investigate center for organized fraud      │
│                                                              │
│  QUALITY HEATMAP (weekly management report):                 │
│  Shows per-center quality scores on a map:                  │
│  ├── GREEN:  All metrics at target                         │
│  ├── YELLOW: 1–2 metrics below target (investigate)        │
│  └── RED:    Multiple metrics below minimum (intervene)    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.12 Quality Feedback Loop

Quality checks are only useful if they drive improvement. A feedback loop connects quality metrics back to operator training and equipment maintenance:

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Improvement Feedback Loop               │
└──────────────────────────────────────────────────────────────┘

STEP 1: Detect quality issue
  CloudWatch alarm:
  CENTER-005 average NFIQ2 < 50 for 3 consecutive days
         │
         ▼
STEP 2: Investigate
  Operations team reviews:
  ├── Which officers have low NFIQ2 captures?
  ├── Is it all fingers or specific fingers?
  ├── Is it one officer or all officers at center?
  └── When did it start? (sudden drop = equipment issue)
         │
         ▼
STEP 3: Root cause
  Possible causes:
  ├── Dirty platen (clean with alcohol → immediate fix)
  ├── Damaged platen (scratched → replace device)
  ├── Officer technique (rolling too fast → training)
  ├── Population factor (many laborers → adjust process)
  └── Algorithm calibration (quality SDK update needed)
         │
         ▼
STEP 4: Remediation
  ├── Equipment: schedule maintenance / replacement
  ├── Training: targeted session for specific officers
  ├── Process: update guidance for specific populations
  └── Software: update quality SDK / thresholds
         │
         ▼
STEP 5: Verify improvement
  Monitor next 7 days:
  ├── NFIQ2 rising back to target? → remediation worked
  └── Still low? → escalate investigation
         │
         ▼
STEP 6: Document and close
  Incident report:
  ├── What was the issue?
  ├── Root cause identified?
  ├── What was the fix?
  └── How many enrollments affected?
      (these may need re-enrollment outreach campaign)
```

---

## 24.13 Quality Check Error Codes

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Check Error Codes                       │
├────────────────┬─────────────────────────────────────────────┤
│ Code           │ Meaning                                    │
├────────────────┼─────────────────────────────────────────────┤
│ QC_FP_001      │ Fingerprint NFIQ2 below hard minimum (< 40)│
│ QC_FP_002      │ Fingerprint image blank / not captured     │
│ QC_FP_003      │ Fewer than 8 fingers above minimum quality │
│ QC_FP_004      │ Duplicate finger detected in set          │
│ QC_FP_005      │ Finger position inconsistent with image   │
│ QC_FC_001      │ No face detected in image                 │
│ QC_FC_002      │ Multiple faces detected                   │
│ QC_FC_003      │ Face pose out of ICAO range               │
│ QC_FC_004      │ Eyes not open / not visible               │
│ QC_FC_005      │ Face quality score below minimum (< 50)   │
│ QC_FC_006      │ Face-document photo mismatch              │
│ QC_IR_001      │ Iris not detected in image                │
│ QC_IR_002      │ Usable iris area below minimum (< 50%)    │
│ QC_IR_003      │ Iris sharpness below threshold            │
│ QC_IR_004      │ Pupil dilation ratio out of range        │
│ QC_DM_001      │ Required demographic field missing        │
│ QC_DM_002      │ Field format validation failed            │
│ QC_DM_003      │ Name OCR mismatch with document          │
│ QC_DM_004      │ DOB mismatch with document               │
│ QC_DC_001      │ Document scan below minimum resolution    │
│ QC_DC_002      │ MRZ check digit failure                  │
│ QC_DC_003      │ Document expired                         │
│ QC_DC_004      │ OCR confidence below threshold           │
│ QC_CM_001      │ Age estimated from face inconsistent      │
│ QC_CM_002      │ Cross-modality fraud indicator detected   │
└────────────────┴─────────────────────────────────────────────┘
```

---

## 24.14 Complete Quality Check Flow — End to End

```
┌──────────────────────────────────────────────────────────────┐
│              Complete Quality Check Flow                     │
└──────────────────────────────────────────────────────────────┘

ENROLLMENT PACKET RECEIVED (S3 upload)
         │
         ▼
SQS → Registration Processor (ECS Fargate)
         │
    ┌────┴──────────────────────────────────────────────┐
    │                                                   │
    ▼                                                   ▼
MDS Signature Verify                       Packet Structure Check
(device cert validation)                   (all fields present?)
         │                                             │
         └──────────────┬────────────────────────────┘
                        │
                        ▼
              Decrypt packet (KMS)
                        │
          ┌─────────────┼───────────────────┐
          │             │                   │
          ▼             ▼                   ▼
  Fingerprint      Face Quality         Iris Quality
  Quality Check    Check                Check
  (NFIQ2 × 10)    (ICAO + ISO          (ISO 29794-6)
                   19794-5)
          │             │                   │
          └─────────────┼───────────────────┘
                        │
                        ▼
              Demographic Quality Check
              (format + OCR cross-check)
                        │
                        ▼
              Document Quality Check
              (scan + MRZ + OCR confidence)
                        │
                        ▼
              Cross-Modality Checks
              (face-doc match + age + fraud)
                        │
                        ▼
              Quality Decision Engine
                        │
         ┌──────────────┼──────────────────────┐
         │              │                      │
         ▼              ▼                      ▼
    PASSED          REVIEW              REJECTED/FRAUD
         │              │                      │
         ▼              ▼                      ▼
  → Deduplication  → Supervisor         → Officer/Security
    (next stage)     queue                 notification
                       │                      │
                       ▼                      ▼
                  Supervisor            DLQ + alarm
                  approves or           + audit log
                  rejects
                       │
                       ▼
               APPROVED → Dedup
               REJECTED → Officer
```

---

## 24.15 AWS Architecture for Quality Checks

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Check AWS Architecture                  │
└──────────────────────────────────────────────────────────────┘

SQS (nbis-enrollment-received)
         │
         ▼
Quality Check Service (ECS Fargate)
         │
         ├──► S3: retrieve decrypted packet contents
         │
         ├──► NFIQ2 SDK container:
         │    Fingerprint quality per finger
         │
         ├──► Amazon Rekognition:
         │    ├── DetectFaces: face quality + ICAO check
         │    └── CompareFaces: face vs document photo
         │
         ├──► Iris Quality SDK container:
         │    ISO 29794-6 quality scoring
         │
         ├──► Amazon Textract:
         │    OCR + document field extraction
         │    MRZ line extraction
         │
         ├──► Custom MRZ validator (Lambda):
         │    Check digit verification
         │
         ├──► DynamoDB:
         │    Update enrollment status
         │    Store quality scores per modality
         │
         ├──► SQS routing:
         │    ├── PASSED → nbis-abis-requests
         │    ├── REVIEW → nbis-supervisor-review
         │    └── REJECTED → nbis-officer-notification
         │
         └──► CloudWatch:
              Publish quality metrics
              Per-center, per-modality, per-officer

QUALITY SCORES STORED IN DYNAMODB:
  DynamoDB table: nbis-enrollment-quality
  PK: packetId
  Attributes:
  ├── fingerprintScores: {finger: score} map
  ├── faceScore: overall + dimension breakdown
  ├── irisScores: {left: score, right: score}
  ├── demographicDecision: PASSED/FAILED
  ├── documentDecision: PASSED/FAILED
  ├── crossModalityFlags: list of flags
  ├── overallDecision: decision code
  └── processingTime: milliseconds
```

---

## 24.16 MOSIP Reference

```
┌──────────────────────────────────────────────────────────────┐
│              MOSIP Quality Check Reference                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  MOSIP Registration Processor quality stages:               │
│                                                              │
│  Stage 1: Packet Receiver                                   │
│  └── Validates packet structure and signature               │
│                                                              │
│  Stage 2: Packet Validator                                  │
│  └── Schema validation, field completeness                  │
│                                                              │
│  Stage 3: Quality Classifier                                │
│  ├── Biometric quality check (NFIQ2, face, iris)           │
│  ├── Configurable quality thresholds (per country)         │
│  └── Pluggable: country replaces default SDK with own      │
│                                                              │
│  Stage 4: Demo Dedupe (demographic pre-check)               │
│  └── Quick demographic match check before ABIS             │
│      (saves ABIS calls for obvious duplicates)             │
│                                                              │
│  MOSIP quality threshold configuration:                     │
│  In MOSIP admin portal:                                     │
│  ├── fingerprint.quality.threshold = 60                    │
│  ├── iris.quality.threshold = 70                           │
│  ├── face.quality.threshold = 70                           │
│  └── Configurable without code change                      │
│                                                              │
│  MOSIP quality failure handling:                            │
│  ├── Failed packet → BIOMETRIC_QUALITY_CHECK_FAILED        │
│  ├── Notification: officer SMS/email (configurable)        │
│  └── Packet: stays in system for supervisor review         │
│                                                              │
│  MOSIP supervisor portal (Registration Supervisor):         │
│  ├── View all packets in REVIEW queue                      │
│  ├── See quality scores + reason for review               │
│  ├── Approve with documented reason                        │
│  └── Reject (triggers re-enrollment notification)         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.17 Key Terms

| Term | Definition |
|------|-----------|
| **Quality check** | Validation that captured data meets minimum standards for reliable use |
| **Client-side check** | Real-time quality check at the enrollment center (immediate feedback) |
| **Server-side check** | Authoritative quality check in the back-end pipeline (post-submission) |
| **NFIQ2** | NIST Fingerprint Image Quality 2 — fingerprint quality standard (0–100) |
| **ISO 29794-6** | International standard for iris image quality metrics |
| **ISO 19794-5** | International standard for face image data and quality |
| **ICAO compliance** | Face image meets ICAO Doc 9303 requirements for identity documents |
| **OCR cross-check** | Comparing OCR-extracted document data with officer-entered demographics |
| **MRZ validation** | Verifying check digits in the passport machine-readable zone |
| **Cross-modality check** | Comparing consistency between different biometric modalities |
| **Face-document match** | Comparing enrolled face photo against document photo using face recognition |
| **Duplicate finger detection** | Checking if same fingerprint submitted for multiple positions |
| **Quality decision engine** | System that combines all check results into a final enrollment decision |
| **QUALITY_PASSED** | All checks passed — proceed to deduplication |
| **QUALITY_REVIEW** | Some issues — human supervisor must decide |
| **QUALITY_REJECTED** | Fundamental failure — enrollment must be redone |
| **FRAUD_FLAG** | Indicators of deliberate fraud detected — escalate to security |
| **Quality heatmap** | Visual map showing per-center quality performance |
| **Recapture rate** | Percentage of biometric captures requiring a second attempt |
| **Feedback loop** | Process connecting quality metrics to training and equipment maintenance |

---

## 24.18 Key Takeaways

- **Quality checks happen twice** — client-side for immediate feedback (officer can recapture now) and server-side for the authoritative decision (after submission). Never rely on client-side alone.
- **Hard minimums are hard for a reason** — below NFIQ2 40 or iris quality 50, the template is genuinely unreliable. Supervisor overrides below these levels must be documented with a specific, auditable reason.
- **Cross-modality checks catch fraud that single-modality checks miss** — a duplicate finger in the same packet, or a face that does not match the passport photo, are caught only when you compare across modalities.
- **Quality metrics are a center management tool** — a center with consistently low NFIQ2 scores has either a dirty platen, a broken scanner, undertrained officers, or a population with occupational wear. The metric tells you something is wrong; investigation reveals which.
- **OCR cross-check is not a duplicate of document verification** — it specifically checks whether what the officer typed matches what the document actually says. Officers make data entry errors. This check catches them before the UIN is issued.
- **Fraud flags are escalations, not automatic rejections** — face-document mismatch could mean a bad passport photo, not a fraudster. Age inconsistency could mean the citizen looks much younger than their DOB. Human review is required before any fraud-flagged packet is rejected.
- **Quality thresholds are configurable** — MOSIP allows country-specific threshold settings without code changes. Different populations have different biometric characteristics. What works in Sweden may not be appropriate for a country with many manual laborers.
- **The quality feedback loop is as important as the check itself** — quality checks without corrective action are just data. Track, alarm, investigate, remediate, and verify improvement.

---

## 24.19 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 25 | Enrollment Validation — packet validation, completeness, business rules |
| Chapter 26 | Duplicate Detection — ABIS integration, 1:N search, dedup workflow |
| Chapter 27 | Approval Workflow — supervisor review, manual decisions, audit trail |

---

*Chapter 24 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*# Chapter 24 — Quality Checks

> **Part III — Enrollment**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 24.1 What are Quality Checks?

**Quality checks** are systematic validations applied to every piece of data captured during enrollment — biometrics, demographics, documents, and signatures — to ensure that the data is good enough to serve its purpose reliably for the lifetime of the identity record.

Quality checking is the NBIS's internal quality assurance mechanism. It answers the question: **"Is this data good enough that we can be confident it will work correctly for authentication and deduplication for the next 10–30 years?"**

```
┌──────────────────────────────────────────────────────────────┐
│              Why Quality Checks Are Critical                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  A fingerprint enrolled with NFIQ2 = 35 (below minimum):   │
│  ├── Authentication failure rate: 15–20%                   │
│  ├── Citizen attends center multiple times to re-enroll    │
│  ├── Government spends money on re-enrollment support      │
│  └── Citizen loses trust in the identity system            │
│                                                              │
│  An iris enrolled with quality = 40 (below minimum):       │
│  ├── IrisCode is unreliable                                │
│  ├── Border crossing failures                               │
│  └── Legitimate citizen treated as unknown person          │
│                                                              │
│  A name with OCR cross-check failure accepted anyway:       │
│  ├── Name on ID card does not match civil registry         │
│  ├── Bank cannot complete KYC (name mismatch)              │
│  └── Citizen must go through legal name correction process │
│                                                              │
│  Quality checks at enrollment are CHEAPER than fixing       │
│  problems after UINs are issued. The cost of a 2-minute    │
│  recapture at enrollment is infinitely cheaper than years  │
│  of authentication failures and re-enrollment campaigns.   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.2 Two Phases of Quality Checking

Quality checks happen at two distinct points in the enrollment pipeline:

```
┌──────────────────────────────────────────────────────────────┐
│              Two-Phase Quality Check Design                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PHASE 1: CLIENT-SIDE (at registration center, real-time)   │
│  ──────────────────────────────────────────────────────     │
│  When: immediately after each biometric capture             │
│  Who: Registration Client software                          │
│  Goal: immediate feedback → officer can recapture now       │
│                                                              │
│  What it checks:                                             │
│  ├── NFIQ2 per finger (quick algorithm)                    │
│  ├── Basic face ICAO checks (pose, eyes open)              │
│  ├── Basic iris quality (area, sharpness)                  │
│  ├── Signature stroke count + area                         │
│  └── Required fields present (demographic completeness)    │
│                                                              │
│  Speed: must be < 2 seconds (real-time feedback)           │
│  Accuracy: good enough for immediate guidance              │
│  Algorithm: lightweight, runs on registration workstation  │
│                                                              │
│  PHASE 2: SERVER-SIDE (back-end pipeline, post-submission)  │
│  ──────────────────────────────────────────────────────     │
│  When: after encrypted packet uploaded to S3               │
│  Who: Registration Processor (ECS + Lambda)                 │
│  Goal: authoritative quality determination                  │
│                                                              │
│  What it checks:                                             │
│  ├── Full NFIQ2 (complete algorithm, more accurate)        │
│  ├── Full face quality (all ICAO dimensions)               │
│  ├── Full iris quality (all ISO 29794-6 dimensions)        │
│  ├── OCR cross-check (document vs entered demographics)    │
│  ├── MRZ validation (check digit verification)             │
│  ├── Document authenticity indicators                      │
│  ├── Signature quality (server-side assessment)            │
│  └── Cross-modality consistency checks                     │
│                                                              │
│  Speed: minutes acceptable (async pipeline)                │
│  Accuracy: authoritative — determines pass/fail            │
│  Algorithm: full production-grade algorithms               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.3 Biometric Quality Standards — Summary

```
┌──────────────────────────────────────────────────────────────┐
│              Biometric Quality Thresholds Reference          │
├────────────────────┬──────────────┬───────────┬─────────────┤
│ Modality           │ Standard     │ Minimum   │ Target      │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Fingerprint        │ NFIQ2        │ 40 / 100  │ 60 / 100   │
│ (per finger)       │ (NIST)       │           │            │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Fingerprint        │ NFIQ2        │ Average   │ Average     │
│ (overall set)      │ average      │ ≥ 50      │ ≥ 65       │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Iris (per eye)     │ ISO 29794-6  │ 50 / 100  │ 70 / 100   │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Face (overall)     │ ISO 19794-5  │ 50 / 100  │ 70 / 100   │
│                    │ + ICAO       │           │            │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Face (pose)        │ ICAO 9303    │ ≤ ±5°     │ ≤ ±3°      │
│                    │              │ yaw/pitch/│            │
│                    │              │ roll      │            │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Signature          │ Internal     │ ≥ 3       │ ≥ 5        │
│ (stroke count)     │              │ strokes   │ strokes    │
├────────────────────┼──────────────┼───────────┼─────────────┤
│ Document scan      │ Internal     │ ≥ 300 DPI │ ≥ 600 DPI  │
│                    │              │ legible   │            │
└────────────────────┴──────────────┴───────────┴────────────┘
```

---

## 24.4 Fingerprint Quality Check — Deep Dive

```
┌──────────────────────────────────────────────────────────────┐
│              Fingerprint Quality Check Pipeline              │
└──────────────────────────────────────────────────────────────┘

INPUT: 10 fingerprint images (WSQ compressed, 500 PPI)
         │
         ▼
FOR EACH FINGER:

  CHECK 1: Image completeness
  ├── Is the image size correct? (not truncated)
  ├── Is the image format valid WSQ?
  └── Is the capture device certified? (MDS signature check)

  CHECK 2: NFIQ2 quality score
  ├── Run NFIQ2 algorithm on each finger image
  ├── Score 0–100 returned for each finger
  └── Apply thresholds:
      < 20:  UNUSABLE → mandatory recapture
      20–39: POOR     → strongly recommend recapture
      40–59: FAIR     → warn, accept with supervisor
      60–74: GOOD     → accept
      75+:   EXCELLENT → accept

  CHECK 3: Pattern detection
  └── Can a ridge pattern be detected?
      (some images are blank / all-white = pad not touched)

  CHECK 4: Finger position consistency
  └── Does the image match the declared finger position?
      (a thumb image submitted as index finger → flag)
      (uses aspect ratio + pattern type classification)

OUTPUT PER FINGER:
  {
    "finger":     "RIGHT_INDEX",
    "nfiq2Score": 78,
    "quality":    "GOOD",
    "decision":   "ACCEPTED",
    "attempts":   1
  }

OVERALL FINGERPRINT QUALITY DECISION:
  ├── ALL 10 fingers ≥ 40:   PASS
  ├── ≥ 8 fingers ≥ 40:      PASS (2 exceptions documented)
  ├── < 8 fingers ≥ 40:      FAIL → supervisor review
  └── Average NFIQ2 < 50:    WARN → supervisor recommendation

QUALITY REPORT SUMMARY:
  {
    "fingerprintQuality": {
      "totalFingers":     10,
      "passed":           9,
      "failed":           1,
      "failedFingers":    ["RIGHT_RING"],
      "averageNFIQ2":     74.3,
      "overallDecision":  "PASS_WITH_EXCEPTION",
      "exceptionReason":  "RIGHT_RING below threshold after
                          3 attempts — skin condition noted"
    }
  }
```

---

## 24.5 Face Quality Check — Deep Dive

```
┌──────────────────────────────────────────────────────────────┐
│              Face Quality Check Pipeline                     │
└──────────────────────────────────────────────────────────────┘

INPUT: Face image (JPEG/JPEG2000, minimum 600×800px)
         │
         ▼
CHECK 1: Face detection
  ├── Is exactly one face detected in the image?
  ├── Zero faces → reject (no face captured)
  └── Multiple faces → reject (someone else in frame)

CHECK 2: Face size and position (geometric)
  ├── Face bounding box occupies 70–80% of frame height?
  ├── Face centered within ±10% of frame?
  └── Eye-to-eye distance ≥ 90 pixels?

CHECK 3: Pose angles (ICAO compliance)
  ├── Yaw (left-right rotation): ≤ ±5°?
  ├── Pitch (chin up/down): ≤ ±5°?
  └── Roll (tilt): ≤ ±5°?

CHECK 4: Eye quality
  ├── Both eyes detected?
  ├── Left eye open score ≥ 70?
  ├── Right eye open score ≥ 70?
  └── Eyes looking at camera (gaze direction check)?

CHECK 5: Lighting quality
  ├── Overall brightness within acceptable range?
  ├── Both sides of face equally lit (symmetry)?
  ├── No overexposed regions (washed out)?
  └── No underexposed regions (too dark)?

CHECK 6: Sharpness
  ├── Focus score ≥ 70?
  └── No motion blur detected?

CHECK 7: Occlusion
  ├── No mask covering nose/mouth?
  ├── Hair not covering eyes?
  └── No hand or object in front of face?

CHECK 8: Expression
  ├── Mouth closed (or near-closed)?
  └── Neutral expression (no extreme emotion)?

CHECK 9: Background
  └── Background sufficiently uniform?
      (high texture background = may confuse face detection)

COMBINED QUALITY SCORE:
  Weighted average across all dimensions:
  ├── Pose:       30% weight (ICAO critical)
  ├── Eyes:       25% weight (matching critical)
  ├── Lighting:   20% weight
  ├── Sharpness:  15% weight
  └── Occlusion:  10% weight

  Score ≥ 70: PASS
  Score 50–69: WARN → officer prompted to retake
  Score < 50:  FAIL → mandatory retake

FACE QUALITY REPORT:
  {
    "faceQuality": {
      "faceDetected":      true,
      "faceCount":         1,
      "poseAngles": {
        "yaw":   2.3,
        "pitch": -1.1,
        "roll":  0.8
      },
      "eyesOpen":          true,
      "lightingScore":     82.1,
      "sharpnessScore":    88.4,
      "occlusion":         "NONE",
      "overallScore":      84.7,
      "icaoCompliant":     true,
      "decision":          "ACCEPTED"
    }
  }
```

---

## 24.6 Iris Quality Check — Deep Dive

```
┌──────────────────────────────────────────────────────────────┐
│              Iris Quality Check Pipeline                     │
└──────────────────────────────────────────────────────────────┘

INPUT: Two iris NIR images (left + right, 640×480px minimum)
         │
         ▼
FOR EACH EYE:

  CHECK 1: Image validity
  ├── Is image format correct?
  ├── Is image not corrupted?
  └── Is MDS device signature valid?

  CHECK 2: Eye/iris detection
  ├── Is an eye detected in the image?
  ├── Is the iris boundary locatable?
  └── Is the pupil detectable?

  CHECK 3: Usable iris area
  ├── Segment iris: locate pupil + limbus boundaries
  ├── Exclude eyelid + eyelash occlusion regions
  ├── Calculate: usable area / total iris area
  └── Usable area ≥ 50%? (minimum)
      Usable area ≥ 70%? (target)

  CHECK 4: Iris sharpness
  ├── Measure frequency content of iris texture region
  ├── Blurry image: low frequency → low sharpness score
  └── Sharpness score ≥ 70?

  CHECK 5: Pupil dilation ratio
  ├── pupil_diameter / iris_diameter = ratio
  ├── Acceptable range: 0.2 – 0.7
  ├── < 0.2: unusual (very bright light / medical)
  └── > 0.7: very dilated (very dark / medication)

  CHECK 6: Gaze direction
  ├── Is iris centered in the image?
  ├── Off-center iris = subject not looking at camera
  └── Acceptable: < 10% off-center

  CHECK 7: Motion blur
  └── Blur score < 10? (subject was still during capture)

COMBINED IRIS QUALITY SCORE:
  {
    "irisQuality": {
      "leftEye": {
        "usableArea":     73.4,
        "sharpness":      82.1,
        "pupilRatio":     0.42,
        "gazeOffset":     3.2,
        "motionBlur":     1.8,
        "overallScore":   78.6,
        "decision":       "ACCEPTED"
      },
      "rightEye": {
        "usableArea":     68.9,
        "sharpness":      79.3,
        "pupilRatio":     0.44,
        "gazeOffset":     4.1,
        "motionBlur":     2.1,
        "overallScore":   74.2,
        "decision":       "ACCEPTED"
      }
    }
  }
```

---

## 24.7 Demographic Quality Checks

```
┌──────────────────────────────────────────────────────────────┐
│              Demographic Quality Check Pipeline              │
└──────────────────────────────────────────────────────────────┘

FOR EACH DEMOGRAPHIC FIELD:

  CHECK 1: Required field present
  └── Is this field populated?
      Empty required field → reject packet

  CHECK 2: Format validation
  ├── Name: regex — only allowed characters?
  ├── DOB:  ISO 8601 format? valid calendar date?
  ├── Mobile: E.164 format? valid country code?
  └── Email: RFC 5322 format?

  CHECK 3: Value range validation
  ├── DOB: not in future, not > 150 years ago
  ├── Name length: 1–150 characters
  └── Age-based rules:
      ├── Age < 5: no fingerprints expected
      ├── Age < 18: guardian consent required
      └── Age > 100: supervisor approval required

  CHECK 4: Cross-field consistency
  ├── fullName == firstName + middleName + lastName?
  ├── Current address == Permanent if sameAsPermanent?
  └── DOB consistent with stated age (if age also given)?

  CHECK 5: OCR cross-check (document vs entered)
  ├── Name: normalized comparison
  │   ├── Match score ≥ 80%: ✅ PASS
  │   ├── Match score 60–79%: ⚠️ WARN (officer confirm)
  │   └── Match score < 60%: ❌ FAIL (must correct)
  └── DOB: exact match required
      ├── Match: ✅ PASS
      └── Mismatch: ❌ FAIL (officer must re-check document)

  CHECK 6: Blacklist check
  ├── Is this name + DOB combination flagged?
  │   (fraud watchlist, court order, etc.)
  └── Flagged → escalate to supervisor (do not auto-reject)

DEMOGRAPHIC QUALITY REPORT:
  {
    "demographicQuality": {
      "requiredFieldsComplete": true,
      "formatValidation":       "PASSED",
      "ocrCrossCheck": {
        "nameMatchScore":  91.3,
        "nameDecision":    "PASSED",
        "dobMatch":        true,
        "dobDecision":     "PASSED"
      },
      "crossFieldConsistency":  "PASSED",
      "blacklistCheck":         "CLEAR",
      "overallDecision":        "PASSED"
    }
  }
```

---

## 24.8 Document Quality Checks

```
┌──────────────────────────────────────────────────────────────┐
│              Document Quality Check Pipeline                 │
└──────────────────────────────────────────────────────────────┘

FOR EACH SCANNED DOCUMENT:

  CHECK 1: Scan quality
  ├── Resolution ≥ 300 DPI?
  ├── Image not blurry (sharpness score)?
  ├── Image not overexposed / underexposed?
  └── Both sides scanned (if double-sided document)?

  CHECK 2: Document type recognition
  ├── Does the scan match the declared document type?
  │   (passport template vs ID card template)
  └── Can fields be located in expected positions?

  CHECK 3: MRZ validation (if applicable)
  ├── MRZ lines present at bottom of page?
  ├── MRZ parsed successfully?
  ├── Check digits valid for all fields?
  └── Extracted data (name, DOB) matches entered data?

  CHECK 4: Document validity
  ├── Issue date in the past?
  ├── Expiry date in the future? (not expired)
  ├── Issue date < Expiry date? (logical)
  └── Document number format valid for issuing country?

  CHECK 5: Tamper indicators (automated)
  ├── Can OCR extract text cleanly?
  │   (tampered text: different font, color artifacts)
  ├── Are image statistics consistent?
  │   (copy-paste substitution: different JPEG artifacts)
  └── MRZ inconsistency:
      document number in MRZ ≠ printed number → flag

  CHECK 6: OCR confidence
  ├── Per-field confidence score from OCR engine
  ├── Low confidence (< 60): flag for officer re-scan
  └── Overall OCR confidence ≥ 70?

DOCUMENT QUALITY REPORT:
  {
    "documentQuality": {
      "scanResolution":    600,
      "scanSharpness":     88.2,
      "documentType":      "PASSPORT",
      "documentRecognized":true,
      "mrzValid":          true,
      "mrzCheckDigits":    "ALL_VALID",
      "documentExpired":   false,
      "ocrConfidence":     94.1,
      "tamperIndicators":  "NONE",
      "overallDecision":   "PASSED"
    }
  }
```

---

## 24.9 Cross-Modality Consistency Checks

After individual modality checks pass, cross-modal checks verify that the biometrics from different modalities are internally consistent:

```
┌──────────────────────────────────────────────────────────────┐
│              Cross-Modality Consistency Checks               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  CHECK 1: Face-to-document photo match                       │
│  ├── Compare enrolled face vs face in document photo        │
│  ├── Tool: Amazon Rekognition CompareFaces API              │
│  ├── Similarity < 70%: FLAG → supervisor review            │
│  │   (possible: wrong person using genuine document)        │
│  └── Similarity ≥ 70%: PASS                                │
│                                                              │
│  CHECK 2: Age consistency                                    │
│  ├── Estimated age from face (AI model) vs stated DOB      │
│  ├── Difference > 15 years: FLAG (unusual, not reject)     │
│  └── Example: stated DOB = 1960, face age estimate = 35    │
│      → 30-year discrepancy → flag for review               │
│                                                              │
│  CHECK 3: Gender consistency                                 │
│  ├── AI gender estimate from face vs stated gender          │
│  ├── Mismatch: FLAG (informational — not grounds to reject) │
│  └── Note: gender expression ≠ legal gender                │
│      Do NOT reject based on gender estimate mismatch.       │
│      Only flag for human awareness.                         │
│                                                              │
│  CHECK 4: Biometric set completeness                         │
│  ├── Are all expected biometrics present?                   │
│  ├── Exception flags for missing modalities?               │
│  │   (if missing without flag → reject packet)             │
│  └── Are exception flags authorized (supervisor approval)? │
│                                                              │
│  CHECK 5: Duplicate biometric within packet                  │
│  ├── Was the same fingerprint submitted for multiple       │
│  │   different finger positions?                           │
│  │   (fraud: one good finger submitted 10 times)           │
│  ├── Run quick pairwise match on submitted fingers         │
│  └── Any pair matches > 0.7 threshold → FRAUD FLAG        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.10 Quality Check Decision Matrix

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Check Decision Matrix                   │
└──────────────────────────────────────────────────────────────┘

ENROLLMENT PACKET QUALITY DECISION LOGIC:

  IF all biometrics PASS and all demographics PASS
  AND all documents PASS:
     → QUALITY_PASSED → proceed to deduplication

  ELIF minor issues (some biometrics in WARN range):
     → QUALITY_PASSED_WITH_WARNINGS
     → Logged, supervisor informed, proceed to dedup
     → Citizen may be contacted for re-enrollment later

  ELIF one or two biometrics FAIL below hard minimum:
     → QUALITY_REVIEW → supervisor queue
     Supervisor options:
     ├── APPROVE_WITH_EXCEPTION (document reason)
     └── REJECT_FOR_RECAPTURE → notify officer

  ELIF critical failure:
     (no primary document, face quality < 30, all
     fingerprints below 40 without documented exception)
     → QUALITY_REJECTED → packet rejected
     → Officer notified with specific reason codes
     → Citizen must return for re-enrollment

  ELIF fraud indicators:
     (face-document mismatch, duplicate fingers,
     MRZ manipulation, document tamper indicators)
     → FRAUD_FLAG → escalated to security team
     → Not processed further until cleared

DECISION CODES:
  ┌─────────────────────────────────────────────────────┐
  │ Code                    │ Next step                 │
  ├─────────────────────────┼───────────────────────────┤
  │ QUALITY_PASSED          │ → Deduplication           │
  │ QUALITY_WARN            │ → Deduplication + logged  │
  │ QUALITY_REVIEW          │ → Supervisor queue        │
  │ QUALITY_REJECTED        │ → Officer notified        │
  │ FRAUD_FLAG              │ → Security team           │
  └─────────────────────────┴───────────────────────────┘
```

---

## 24.11 Quality Check Metrics and Monitoring

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Metrics Dashboard                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PER REGISTRATION CENTER (daily report):                     │
│                                                              │
│  Fingerprint metrics:                                        │
│  ├── average_nfiq2_score (target: ≥ 65)                    │
│  ├── below_minimum_rate (target: < 5%)                     │
│  ├── recapture_rate (target: < 15%)                        │
│  └── exception_rate (target: < 2%)                         │
│                                                              │
│  Face metrics:                                               │
│  ├── average_face_quality_score (target: ≥ 70)             │
│  ├── icao_compliance_rate (target: > 95%)                  │
│  ├── recapture_rate (target: < 10%)                        │
│  └── liveness_fail_rate (target: < 1%)                    │
│                                                              │
│  Iris metrics:                                               │
│  ├── average_iris_quality_score (target: ≥ 70)             │
│  ├── usable_area_average (target: ≥ 70%)                   │
│  └── recapture_rate (target: < 10%)                        │
│                                                              │
│  Demographic metrics:                                        │
│  ├── ocr_mismatch_rate (target: < 3%)                      │
│  ├── format_error_rate (target: < 1%)                      │
│  └── document_rejection_rate (target: < 5%)               │
│                                                              │
│  CLOUDWATCH ALARMS:                                          │
│  ├── average_nfiq2 < 50 for 1 hour at any center          │
│  │   → Alert: equipment issue / officer training need      │
│  ├── icao_compliance_rate < 80% for 1 hour                │
│  │   → Alert: camera misconfigured / lighting problem      │
│  ├── liveness_fail_rate > 5% for 30 min                   │
│  │   → Alert: possible spoofing attempt                   │
│  ├── ocr_mismatch_rate > 10% for 1 hour                   │
│  │   → Alert: officer entering data incorrectly           │
│  └── fraud_flag_rate > 1% for 1 day                       │
│      → Alert: investigate center for organized fraud      │
│                                                              │
│  QUALITY HEATMAP (weekly management report):                 │
│  Shows per-center quality scores on a map:                  │
│  ├── GREEN:  All metrics at target                         │
│  ├── YELLOW: 1–2 metrics below target (investigate)        │
│  └── RED:    Multiple metrics below minimum (intervene)    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.12 Quality Feedback Loop

Quality checks are only useful if they drive improvement. A feedback loop connects quality metrics back to operator training and equipment maintenance:

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Improvement Feedback Loop               │
└──────────────────────────────────────────────────────────────┘

STEP 1: Detect quality issue
  CloudWatch alarm:
  CENTER-005 average NFIQ2 < 50 for 3 consecutive days
         │
         ▼
STEP 2: Investigate
  Operations team reviews:
  ├── Which officers have low NFIQ2 captures?
  ├── Is it all fingers or specific fingers?
  ├── Is it one officer or all officers at center?
  └── When did it start? (sudden drop = equipment issue)
         │
         ▼
STEP 3: Root cause
  Possible causes:
  ├── Dirty platen (clean with alcohol → immediate fix)
  ├── Damaged platen (scratched → replace device)
  ├── Officer technique (rolling too fast → training)
  ├── Population factor (many laborers → adjust process)
  └── Algorithm calibration (quality SDK update needed)
         │
         ▼
STEP 4: Remediation
  ├── Equipment: schedule maintenance / replacement
  ├── Training: targeted session for specific officers
  ├── Process: update guidance for specific populations
  └── Software: update quality SDK / thresholds
         │
         ▼
STEP 5: Verify improvement
  Monitor next 7 days:
  ├── NFIQ2 rising back to target? → remediation worked
  └── Still low? → escalate investigation
         │
         ▼
STEP 6: Document and close
  Incident report:
  ├── What was the issue?
  ├── Root cause identified?
  ├── What was the fix?
  └── How many enrollments affected?
      (these may need re-enrollment outreach campaign)
```

---

## 24.13 Quality Check Error Codes

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Check Error Codes                       │
├────────────────┬─────────────────────────────────────────────┤
│ Code           │ Meaning                                    │
├────────────────┼─────────────────────────────────────────────┤
│ QC_FP_001      │ Fingerprint NFIQ2 below hard minimum (< 40)│
│ QC_FP_002      │ Fingerprint image blank / not captured     │
│ QC_FP_003      │ Fewer than 8 fingers above minimum quality │
│ QC_FP_004      │ Duplicate finger detected in set          │
│ QC_FP_005      │ Finger position inconsistent with image   │
│ QC_FC_001      │ No face detected in image                 │
│ QC_FC_002      │ Multiple faces detected                   │
│ QC_FC_003      │ Face pose out of ICAO range               │
│ QC_FC_004      │ Eyes not open / not visible               │
│ QC_FC_005      │ Face quality score below minimum (< 50)   │
│ QC_FC_006      │ Face-document photo mismatch              │
│ QC_IR_001      │ Iris not detected in image                │
│ QC_IR_002      │ Usable iris area below minimum (< 50%)    │
│ QC_IR_003      │ Iris sharpness below threshold            │
│ QC_IR_004      │ Pupil dilation ratio out of range        │
│ QC_DM_001      │ Required demographic field missing        │
│ QC_DM_002      │ Field format validation failed            │
│ QC_DM_003      │ Name OCR mismatch with document          │
│ QC_DM_004      │ DOB mismatch with document               │
│ QC_DC_001      │ Document scan below minimum resolution    │
│ QC_DC_002      │ MRZ check digit failure                  │
│ QC_DC_003      │ Document expired                         │
│ QC_DC_004      │ OCR confidence below threshold           │
│ QC_CM_001      │ Age estimated from face inconsistent      │
│ QC_CM_002      │ Cross-modality fraud indicator detected   │
└────────────────┴─────────────────────────────────────────────┘
```

---

## 24.14 Complete Quality Check Flow — End to End

```
┌──────────────────────────────────────────────────────────────┐
│              Complete Quality Check Flow                     │
└──────────────────────────────────────────────────────────────┘

ENROLLMENT PACKET RECEIVED (S3 upload)
         │
         ▼
SQS → Registration Processor (ECS Fargate)
         │
    ┌────┴──────────────────────────────────────────────┐
    │                                                   │
    ▼                                                   ▼
MDS Signature Verify                       Packet Structure Check
(device cert validation)                   (all fields present?)
         │                                             │
         └──────────────┬────────────────────────────┘
                        │
                        ▼
              Decrypt packet (KMS)
                        │
          ┌─────────────┼───────────────────┐
          │             │                   │
          ▼             ▼                   ▼
  Fingerprint      Face Quality         Iris Quality
  Quality Check    Check                Check
  (NFIQ2 × 10)    (ICAO + ISO          (ISO 29794-6)
                   19794-5)
          │             │                   │
          └─────────────┼───────────────────┘
                        │
                        ▼
              Demographic Quality Check
              (format + OCR cross-check)
                        │
                        ▼
              Document Quality Check
              (scan + MRZ + OCR confidence)
                        │
                        ▼
              Cross-Modality Checks
              (face-doc match + age + fraud)
                        │
                        ▼
              Quality Decision Engine
                        │
         ┌──────────────┼──────────────────────┐
         │              │                      │
         ▼              ▼                      ▼
    PASSED          REVIEW              REJECTED/FRAUD
         │              │                      │
         ▼              ▼                      ▼
  → Deduplication  → Supervisor         → Officer/Security
    (next stage)     queue                 notification
                       │                      │
                       ▼                      ▼
                  Supervisor            DLQ + alarm
                  approves or           + audit log
                  rejects
                       │
                       ▼
               APPROVED → Dedup
               REJECTED → Officer
```

---

## 24.15 AWS Architecture for Quality Checks

```
┌──────────────────────────────────────────────────────────────┐
│              Quality Check AWS Architecture                  │
└──────────────────────────────────────────────────────────────┘

SQS (nbis-enrollment-received)
         │
         ▼
Quality Check Service (ECS Fargate)
         │
         ├──► S3: retrieve decrypted packet contents
         │
         ├──► NFIQ2 SDK container:
         │    Fingerprint quality per finger
         │
         ├──► Amazon Rekognition:
         │    ├── DetectFaces: face quality + ICAO check
         │    └── CompareFaces: face vs document photo
         │
         ├──► Iris Quality SDK container:
         │    ISO 29794-6 quality scoring
         │
         ├──► Amazon Textract:
         │    OCR + document field extraction
         │    MRZ line extraction
         │
         ├──► Custom MRZ validator (Lambda):
         │    Check digit verification
         │
         ├──► DynamoDB:
         │    Update enrollment status
         │    Store quality scores per modality
         │
         ├──► SQS routing:
         │    ├── PASSED → nbis-abis-requests
         │    ├── REVIEW → nbis-supervisor-review
         │    └── REJECTED → nbis-officer-notification
         │
         └──► CloudWatch:
              Publish quality metrics
              Per-center, per-modality, per-officer

QUALITY SCORES STORED IN DYNAMODB:
  DynamoDB table: nbis-enrollment-quality
  PK: packetId
  Attributes:
  ├── fingerprintScores: {finger: score} map
  ├── faceScore: overall + dimension breakdown
  ├── irisScores: {left: score, right: score}
  ├── demographicDecision: PASSED/FAILED
  ├── documentDecision: PASSED/FAILED
  ├── crossModalityFlags: list of flags
  ├── overallDecision: decision code
  └── processingTime: milliseconds
```

---

## 24.16 MOSIP Reference

```
┌──────────────────────────────────────────────────────────────┐
│              MOSIP Quality Check Reference                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  MOSIP Registration Processor quality stages:               │
│                                                              │
│  Stage 1: Packet Receiver                                   │
│  └── Validates packet structure and signature               │
│                                                              │
│  Stage 2: Packet Validator                                  │
│  └── Schema validation, field completeness                  │
│                                                              │
│  Stage 3: Quality Classifier                                │
│  ├── Biometric quality check (NFIQ2, face, iris)           │
│  ├── Configurable quality thresholds (per country)         │
│  └── Pluggable: country replaces default SDK with own      │
│                                                              │
│  Stage 4: Demo Dedupe (demographic pre-check)               │
│  └── Quick demographic match check before ABIS             │
│      (saves ABIS calls for obvious duplicates)             │
│                                                              │
│  MOSIP quality threshold configuration:                     │
│  In MOSIP admin portal:                                     │
│  ├── fingerprint.quality.threshold = 60                    │
│  ├── iris.quality.threshold = 70                           │
│  ├── face.quality.threshold = 70                           │
│  └── Configurable without code change                      │
│                                                              │
│  MOSIP quality failure handling:                            │
│  ├── Failed packet → BIOMETRIC_QUALITY_CHECK_FAILED        │
│  ├── Notification: officer SMS/email (configurable)        │
│  └── Packet: stays in system for supervisor review         │
│                                                              │
│  MOSIP supervisor portal (Registration Supervisor):         │
│  ├── View all packets in REVIEW queue                      │
│  ├── See quality scores + reason for review               │
│  ├── Approve with documented reason                        │
│  └── Reject (triggers re-enrollment notification)         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 24.17 Key Terms

| Term | Definition |
|------|-----------|
| **Quality check** | Validation that captured data meets minimum standards for reliable use |
| **Client-side check** | Real-time quality check at the enrollment center (immediate feedback) |
| **Server-side check** | Authoritative quality check in the back-end pipeline (post-submission) |
| **NFIQ2** | NIST Fingerprint Image Quality 2 — fingerprint quality standard (0–100) |
| **ISO 29794-6** | International standard for iris image quality metrics |
| **ISO 19794-5** | International standard for face image data and quality |
| **ICAO compliance** | Face image meets ICAO Doc 9303 requirements for identity documents |
| **OCR cross-check** | Comparing OCR-extracted document data with officer-entered demographics |
| **MRZ validation** | Verifying check digits in the passport machine-readable zone |
| **Cross-modality check** | Comparing consistency between different biometric modalities |
| **Face-document match** | Comparing enrolled face photo against document photo using face recognition |
| **Duplicate finger detection** | Checking if same fingerprint submitted for multiple positions |
| **Quality decision engine** | System that combines all check results into a final enrollment decision |
| **QUALITY_PASSED** | All checks passed — proceed to deduplication |
| **QUALITY_REVIEW** | Some issues — human supervisor must decide |
| **QUALITY_REJECTED** | Fundamental failure — enrollment must be redone |
| **FRAUD_FLAG** | Indicators of deliberate fraud detected — escalate to security |
| **Quality heatmap** | Visual map showing per-center quality performance |
| **Recapture rate** | Percentage of biometric captures requiring a second attempt |
| **Feedback loop** | Process connecting quality metrics to training and equipment maintenance |

---

## 24.18 Key Takeaways

- **Quality checks happen twice** — client-side for immediate feedback (officer can recapture now) and server-side for the authoritative decision (after submission). Never rely on client-side alone.
- **Hard minimums are hard for a reason** — below NFIQ2 40 or iris quality 50, the template is genuinely unreliable. Supervisor overrides below these levels must be documented with a specific, auditable reason.
- **Cross-modality checks catch fraud that single-modality checks miss** — a duplicate finger in the same packet, or a face that does not match the passport photo, are caught only when you compare across modalities.
- **Quality metrics are a center management tool** — a center with consistently low NFIQ2 scores has either a dirty platen, a broken scanner, undertrained officers, or a population with occupational wear. The metric tells you something is wrong; investigation reveals which.
- **OCR cross-check is not a duplicate of document verification** — it specifically checks whether what the officer typed matches what the document actually says. Officers make data entry errors. This check catches them before the UIN is issued.
- **Fraud flags are escalations, not automatic rejections** — face-document mismatch could mean a bad passport photo, not a fraudster. Age inconsistency could mean the citizen looks much younger than their DOB. Human review is required before any fraud-flagged packet is rejected.
- **Quality thresholds are configurable** — MOSIP allows country-specific threshold settings without code changes. Different populations have different biometric characteristics. What works in Sweden may not be appropriate for a country with many manual laborers.
- **The quality feedback loop is as important as the check itself** — quality checks without corrective action are just data. Track, alarm, investigate, remediate, and verify improvement.

---

## 24.19 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 25 | Enrollment Validation — packet validation, completeness, business rules |
| Chapter 26 | Duplicate Detection — ABIS integration, 1:N search, dedup workflow |
| Chapter 27 | Approval Workflow — supervisor review, manual decisions, audit trail |

---

*Chapter 24 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*
