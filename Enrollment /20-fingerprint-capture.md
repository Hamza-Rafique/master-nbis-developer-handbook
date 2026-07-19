# Chapter 20 — Fingerprint Capture

> **Part III — Enrollment**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 20.1 What is Fingerprint Capture?

**Fingerprint capture** is the process of recording the unique ridge patterns on a person's fingertips using a certified biometric sensor during enrollment. The captured images are then processed into templates and stored in the CIDR for future 1:1 verification and 1:N deduplication.

Fingerprints are the **most widely used biometric modality** in national identity systems worldwide — they are mature technology, low cost, fast, and extremely accurate when captured correctly.

```
┌──────────────────────────────────────────────────────────────┐
│              Why Fingerprints Dominate National ID           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ✅ Unique:      No two people share the same fingerprints   │
│                 (including identical twins)                  │
│  ✅ Permanent:   Ridge patterns stable from 6 months in      │
│                 utero to death                              │
│  ✅ Mature:      50+ years of AFIS technology                │
│  ✅ Cost:        Sensors as cheap as $50 for low-end         │
│  ✅ Speed:       Capture in < 5 seconds                      │
│  ✅ Acceptance:  Globally understood and accepted            │
│  ✅ 10 fingers:  Multiple backups — losing 1 finger          │
│                 does not disable authentication              │
│                                                              │
│  ⚠️  Challenges:                                             │
│  ├── Manual laborers: worn ridges, low quality             │
│  ├── Elderly: faded ridges, dry skin                        │
│  ├── Certain medical conditions: eczema, burns             │
│  ├── Children < 5: ridges too small / unstable             │
│  └── Amputees: fewer than 10 fingers available             │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 20.2 Fingerprint Anatomy — What the System Captures

Understanding what makes a fingerprint unique is essential for understanding capture quality requirements:

```
┌──────────────────────────────────────────────────────────────┐
│              Fingerprint Anatomy                             │
└──────────────────────────────────────────────────────────────┘

RIDGE PATTERNS (three primary types):
  ┌──────────────────────────────────────────────────────┐
  │  LOOP (60–65% of all fingerprints)                   │
  │  Ridges enter from one side, curve, exit same side   │
  │  ├── Ulnar loop (curves toward little finger)        │
  │  └── Radial loop (curves toward thumb)               │
  │                                                      │
  │  WHORL (30–35% of all fingerprints)                  │
  │  Ridges form circular or spiral patterns             │
  │  ├── Plain whorl                                     │
  │  ├── Central pocket loop                             │
  │  ├── Double loop                                     │
  │  └── Accidental                                      │
  │                                                      │
  │  ARCH (5% of all fingerprints)                       │
  │  Ridges enter from one side, rise, exit other side   │
  │  ├── Plain arch                                      │
  │  └── Tented arch                                     │
  └──────────────────────────────────────────────────────┘

MINUTIAE POINTS (what the template captures):
  ├── Ridge ending: a ridge that stops
  ├── Bifurcation: a ridge that splits into two
  ├── Short ridge: a ridge shorter than surrounding
  ├── Enclosure: a ridge that splits and rejoins
  └── Dot: isolated ridge shorter than its width

  A fingerprint has 40–100 minutiae points.
  No two fingerprints share the same set.
  Template = compact map of minutiae positions + angles.
  Original image is NOT stored — only the minutiae map.
```

---

## 20.3 Fingerprint Sensor Types

Different sensor technologies have different strengths and weaknesses. NBIS deployments select sensors based on environment and use case:

```
┌──────────────────────────────────────────────────────────────┐
│              Fingerprint Sensor Technology Comparison        │
├──────────────────────────┬───────────────────────────────────┤
│ Technology               │ How it works                     │
├──────────────────────────┼───────────────────────────────────┤
│ OPTICAL                  │ LED light + CCD/CMOS camera.     │
│ (most common in NBIS)    │ Light reflects off finger,       │
│                          │ camera captures ridge image.     │
│                          │                                  │
│                          │ Pros: high resolution, durable,  │
│                          │ works on dry/moist fingers.      │
│                          │ Cons: can be spoofed with        │
│                          │ high-quality fake finger.        │
│                          │ NBIS sensors: LiveScan optical   │
├──────────────────────────┼───────────────────────────────────┤
│ CAPACITIVE               │ Electrical charge grid.          │
│ (smartphones, tokens)    │ Ridges conduct electricity,      │
│                          │ valleys do not.                  │
│                          │                                  │
│                          │ Pros: compact, spoof-resistant.  │
│                          │ Cons: moisture sensitive,        │
│                          │ lower resolution than optical.   │
│                          │ NBIS use: limited (mobile only)  │
├──────────────────────────┼───────────────────────────────────┤
│ THERMAL                  │ Temperature difference between   │
│ (specialized)            │ ridges (warm) and valleys (air). │
│                          │                                  │
│                          │ Pros: works through gloves,      │
│                          │ spoof-resistant.                 │
│                          │ Cons: expensive, slow.           │
│                          │ NBIS use: high-security borders  │
├──────────────────────────┼───────────────────────────────────┤
│ ULTRASOUND               │ Sound waves penetrate skin.      │
│ (high-end, modern)       │ Captures subsurface ridge detail.│
│                          │                                  │
│                          │ Pros: best spoof resistance,     │
│                          │ works with wet/dirty fingers.    │
│                          │ Cons: most expensive.            │
│                          │ NBIS use: premium deployments   │
└──────────────────────────┴───────────────────────────────────┘
```

---

## 20.4 Capture Types — Rolled vs Slap vs Single

NBIS enrollment captures fingerprints in multiple ways, each serving a different purpose:

```
┌──────────────────────────────────────────────────────────────┐
│              Fingerprint Capture Types                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. ROLLED PRINTS                                            │
│  Finger rolled from nail to nail across the platen.         │
│  Captures the maximum ridge area including sides.           │
│                                                              │
│  Purpose:                                                    │
│  ├── Stored in CIDR as the primary enrolled template       │
│  ├── Used for ABIS deduplication (most data = best match)  │
│  └── Used for 1:1 verification (best accuracy)             │
│                                                              │
│  Process:                                                    │
│  ├── Officer instructs citizen to place finger on platen   │
│  ├── Citizen rolls finger slowly from left to right         │
│  ├── Sensor captures image at peak quality                 │
│  └── Takes 3–5 seconds per finger                          │
│                                                              │
│  ────────────────────────────────────────────────────────  │
│                                                              │
│  2. SLAP PRINTS (four-finger simultaneous)                   │
│  Four fingers placed flat simultaneously on large platen.   │
│  Also called "flat" or "plain" impressions.                 │
│                                                              │
│  Standard NBIS capture sequence:                             │
│  ├── Left slap: left index, middle, ring, little (4+)      │
│  ├── Right slap: right index, middle, ring, little (4+)    │
│  └── Two-thumb slap: both thumbs together                  │
│  Total: 10 fingers captured in 3 placements                │
│                                                              │
│  Purpose:                                                    │
│  ├── Quality check (compare with rolled prints)            │
│  ├── Fast eGate authentication (flat finger on reader)     │
│  └── Backup if rolled quality insufficient                 │
│                                                              │
│  ────────────────────────────────────────────────────────  │
│                                                              │
│  3. SINGLE FINGER (index)                                    │
│  Right index finger only — most common for 1:1 verification.│
│                                                              │
│  Purpose:                                                    │
│  ├── Daily authentication (bank eKYC, portal login)        │
│  ├── eGate crossing (one finger = fast)                    │
│  └── Mobile verification (compact sensor)                  │
│                                                              │
│  STANDARD NBIS ENROLLMENT SEQUENCE:                          │
│  All 10 rolled prints → Left slap → Right slap → 2-thumb   │
│  = 13 captures total                                        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 20.5 NFIQ2 — Fingerprint Quality Standard

**NFIQ2 (NIST Fingerprint Image Quality 2)** is the international standard for measuring fingerprint image quality. It produces a score from 0 to 100, where higher is better.

```
┌──────────────────────────────────────────────────────────────┐
│              NFIQ2 Score Interpretation                      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Score 0–20:   UNUSABLE                                      │
│  ├── Image too blurry, smeared, or featureless             │
│  ├── Template extraction will fail                         │
│  └── Must recapture — no exceptions                        │
│                                                              │
│  Score 21–39:  POOR                                          │
│  ├── Some ridge detail visible but insufficient            │
│  ├── High risk of false non-match in authentication       │
│  └── Recapture strongly recommended                        │
│                                                              │
│  Score 40–59:  FAIR                                          │
│  ├── Acceptable for deduplication at lower accuracy        │
│  ├── May cause issues in 1:1 verification                  │
│  └── NBIS minimum threshold: 40 (operational minimum)     │
│                                                              │
│  Score 60–74:  GOOD                                          │
│  ├── Reliable for both deduplication and verification      │
│  ├── Recommended minimum for enrollment                    │
│  └── NBIS target threshold: 60                            │
│                                                              │
│  Score 75–100: EXCELLENT                                     │
│  ├── High-quality ridge detail, clear minutiae             │
│  ├── Best accuracy for 1:1 and 1:N matching               │
│  └── Target for high-security use cases                   │
│                                                              │
│  NBIS THRESHOLD POLICY:                                      │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Hard minimum:  NFIQ2 ≥ 40 (reject below this)        │   │
│  │ Soft target:   NFIQ2 ≥ 60 (warn officer if below)    │   │
│  │ Exception path:NFIQ2 < 40 after 3 attempts →         │   │
│  │                supervisor decides (document reason)  │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### NFIQ2 Factors

```
┌──────────────────────────────────────────────────────────────┐
│              What Affects NFIQ2 Score                        │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Increases quality:                                          │
│  ├── Clean, dry platen (wipe before each capture)          │
│  ├── Proper pressure (not too light, not too hard)         │
│  ├── Slow, deliberate roll (not jerky)                     │
│  ├── Room temperature fingers (not too cold)               │
│  └── Clean, moisturized fingers                            │
│                                                              │
│  Decreases quality:                                          │
│  ├── Dirty or oily platen (residue from previous)         │
│  ├── Too much pressure (ridges spread, blur)               │
│  ├── Too little pressure (ridges don't contact platen)     │
│  ├── Fast roll (motion blur)                               │
│  ├── Dry skin (no ridge-valley contrast)                   │
│  ├── Wet/sweaty fingers (ridges bleed together)            │
│  ├── Cuts, scars crossing ridges                           │
│  └── Cold fingers (insufficient blood flow to ridges)      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 20.6 The 10-Finger Capture Sequence

```
┌──────────────────────────────────────────────────────────────┐
│              Standard 10-Finger Capture Sequence             │
└──────────────────────────────────────────────────────────────┘

FINGER NUMBERING (ISO standard):
  Right hand: 1=Thumb  2=Index  3=Middle  4=Ring  5=Little
  Left hand:  6=Thumb  7=Index  8=Middle  9=Ring  10=Little

CAPTURE SEQUENCE (NBIS standard):

  Phase 1: Rolled prints (individual fingers)
  ─────────────────────────────────────────
  Step 1:  Right thumb (finger 1)  → Rolled
  Step 2:  Right index (finger 2)  → Rolled
  Step 3:  Right middle (finger 3) → Rolled
  Step 4:  Right ring (finger 4)   → Rolled
  Step 5:  Right little (finger 5) → Rolled
  Step 6:  Left thumb (finger 6)   → Rolled
  Step 7:  Left index (finger 7)   → Rolled
  Step 8:  Left middle (finger 8)  → Rolled
  Step 9:  Left ring (finger 9)    → Rolled
  Step 10: Left little (finger 10) → Rolled

  Phase 2: Slap prints (simultaneous)
  ─────────────────────────────────────────
  Step 11: Right slap (fingers 2,3,4,5)
  Step 12: Left slap (fingers 7,8,9,10)
  Step 13: Two-thumb slap (fingers 1,6)

  Quality check after each:
  ├── NFIQ2 displayed immediately after capture
  ├── Green (≥60): proceed
  ├── Yellow (40–59): officer prompted to retry
  └── Red (<40): must retry (up to 3 attempts)

TOTAL CAPTURE TIME: 5–10 minutes for all 13 captures
```

---

## 20.7 Fingerprint Capture Workflow — Step by Step

```
┌──────────────────────────────────────────────────────────────┐
│              Fingerprint Capture Workflow                    │
└──────────────────────────────────────────────────────────────┘

BEFORE CAPTURE:
  ├── Officer cleans platen with alcohol wipe
  ├── Citizen washes hands (if visibly dirty)
  ├── If hands cold: citizen rubs hands together to warm
  └── Officer explains process to citizen

DURING CAPTURE:
  For each finger:
  │
  ├── Officer positions citizen's finger on platen
  ├── Software guides: "Roll slowly left to right"
  ├── Sensor captures image (500 DPI or higher)
  ├── Software displays: captured image + NFIQ2 score
  │
  ├── NFIQ2 ≥ 60 (GREEN):
  │   └── Software accepts automatically → next finger
  │
  ├── NFIQ2 40–59 (YELLOW):
  │   ├── Officer sees warning
  │   ├── Officer chooses: accept or retry
  │   └── Typically retry once for better quality
  │
  └── NFIQ2 < 40 (RED):
      ├── Software prompts retry
      ├── Officer tries improved technique:
      │   ├── Clean platen again
      │   ├── Adjust pressure guidance
      │   └── Warm hands (if cold)
      ├── After 3 attempts: EXCEPTION handling
      └── Exception: document reason, supervisor approves

AFTER CAPTURE:
  ├── Software shows 10-finger quality summary grid
  ├── Officer reviews: are all fingers captured?
  ├── Any below threshold? → retry or document exception
  └── Officer confirms: submit biometrics

QUALITY SUMMARY GRID (what officer sees):
  ┌────────────────────────────────────────────────────┐
  │  R Thumb  R Index  R Middle  R Ring  R Little       │
  │  NFIQ:87  NFIQ:92  NFIQ:88  NFIQ:45* NFIQ:79       │
  │  ✅ Good  ✅ Excel ✅ Good  ⚠️ Fair   ✅ Good        │
  │                                                    │
  │  L Thumb  L Index  L Middle  L Ring  L Little       │
  │  NFIQ:83  NFIQ:91  NFIQ:87  NFIQ:82  NFIQ:76       │
  │  ✅ Good  ✅ Excel ✅ Good  ✅ Good   ✅ Good        │
  └────────────────────────────────────────────────────┘
  * R Ring below soft target — officer prompted to retry
```

---

## 20.8 Fingerprint Image Format Standards

```
┌──────────────────────────────────────────────────────────────┐
│              Fingerprint Image Standards                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  RAW CAPTURE IMAGE:                                          │
│  ├── Resolution: 500 PPI minimum (1000 PPI for premium)    │
│  ├── Bit depth: 8-bit grayscale                            │
│  ├── Format: RAW (uncompressed from sensor)                │
│  └── Size: ~500KB per finger at 500 PPI                   │
│                                                              │
│  STORED IMAGE FORMAT:                                        │
│  ├── Standard: ISO/IEC 19794-4 (Finger Image Data)        │
│  ├── Compression: WSQ (Wavelet Scalar Quantization)        │
│  │   FBI-certified compression for fingerprints            │
│  │   Ratio: ~15:1 (500KB raw → ~33KB WSQ)                 │
│  └── Alternative: JPEG 2000 (for interoperability)        │
│                                                              │
│  STORED TEMPLATE FORMAT:                                     │
│  ├── Standard: ISO/IEC 19794-2 (Minutiae Data)            │
│  ├── Size: ~300–800 bytes per finger                       │
│  ├── Content: minutiae positions + angles + types          │
│  └── NOT reversible — cannot reconstruct image from        │
│      minutiae template                                     │
│                                                              │
│  WHAT GETS STORED IN NBIS:                                   │
│  ├── Template (minutiae): S3 + KMS — ALWAYS                │
│  ├── Compressed image (WSQ): S3 + KMS — OPTIONAL          │
│  │   (some systems keep image for quality re-assessment)   │
│  └── Raw capture image: NEVER stored (too large + risk)   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 20.9 Exception Handling — Missing or Poor Fingerprints

Real-world NBIS deployments face frequent biometric exceptions. Every scenario must be handled and documented:

```
┌──────────────────────────────────────────────────────────────┐
│              Fingerprint Exception Scenarios                 │
├────────────────────────────────┬─────────────────────────────┤
│ Scenario                       │ Handling                   │
├────────────────────────────────┼─────────────────────────────┤
│ Single finger amputated        │ Capture remaining 9.       │
│                                │ Document: RIGHT_INDEX_     │
│                                │ MISSING_AMPUTATION         │
│                                │ Flag: finger_5 = MISSING   │
├────────────────────────────────┼─────────────────────────────┤
│ Multiple fingers amputated     │ Capture available fingers. │
│                                │ If < 6 fingers: use iris   │
│                                │ as primary dedup modality. │
│                                │ Document all missing.      │
├────────────────────────────────┼─────────────────────────────┤
│ Worn ridges (manual laborer)   │ Try all 10 fingers.        │
│ NFIQ2 consistently < 40        │ Best 6 above 40 accepted.  │
│                                │ Supervisor override for    │
│                                │ remaining fingers.         │
│                                │ Flag: POOR_RIDGE_QUALITY   │
├────────────────────────────────┼─────────────────────────────┤
│ Burns / scarring across ridges │ Capture unaffected fingers.│
│                                │ Document scar location.    │
│                                │ Medical certificate if     │
│                                │ available.                 │
├────────────────────────────────┼─────────────────────────────┤
│ Eczema / skin condition        │ Capture on good days if    │
│                                │ possible (return visit).   │
│                                │ Accept best available.     │
│                                │ Medical documentation.     │
├────────────────────────────────┼─────────────────────────────┤
│ Newborn / child < 5 years      │ No fingerprints captured.  │
│                                │ Face + parent link only.   │
│                                │ Re-enroll biometrics at 5. │
│                                │ Flag: AGE_EXCEPTION        │
├────────────────────────────────┼─────────────────────────────┤
│ ALL fingerprints unusable      │ Supervisor authorizes.     │
│ (rare — very severe condition) │ Iris becomes primary.      │
│                                │ Face as secondary.         │
│                                │ Medical certificate req.   │
│                                │ Flag: NO_FINGERPRINTS      │
└────────────────────────────────┴─────────────────────────────┘

EXCEPTION DATA STRUCTURE IN PACKET:
{
  "biometricExceptions": [{
    "finger": "RIGHT_INDEX",
    "reason": "AMPUTATION",
    "documentedBy": "OFR-007",
    "supervisorApproval": "SUP-001",
    "medicalCertificate": "s3://nbis-packets/.../medical.enc"
  }]
}
```

---

## 20.10 Liveness Detection for Fingerprints

**Fingerprint liveness detection** (also called Presentation Attack Detection — PAD) ensures the captured fingerprint comes from a live person, not a spoof:

```
┌──────────────────────────────────────────────────────────────┐
│              Fingerprint Liveness Detection                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  COMMON SPOOF ATTACKS:                                       │
│  ├── Gelatin finger (made from fingerprint mold)           │
│  ├── Silicone fake finger                                  │
│  ├── Latent lift (fingerprint lifted from surface,         │
│  │   printed on thin film, placed on real finger)         │
│  └── 2D print (high-res photo of fingerprint)             │
│                                                              │
│  LIVENESS DETECTION METHODS:                                 │
│                                                              │
│  HARDWARE-BASED (built into sensor):                         │
│  ├── Pulse detection: detects blood flow in finger          │
│  ├── Temperature: live finger is 32–37°C (spoof is colder) │
│  ├── Perspiration: live skin shows micro-sweat patterns    │
│  ├── Electrical conductivity: live skin conducts           │
│  │   differently than gelatin or silicone                  │
│  └── Multispectral: captures subsurface ridge detail       │
│      (fake fingers have no subsurface structure)           │
│                                                              │
│  SOFTWARE-BASED (after image capture):                       │
│  ├── Texture analysis: live skin has specific texture      │
│  │   patterns fake materials cannot replicate perfectly    │
│  ├── Ridge frequency analysis: fake fingers show           │
│  │   artifacts in ridge patterns at pixel level            │
│  └── Deep learning classifier: trained on spoof vs live   │
│      (ISO/IEC 30107-3 evaluation standard)                 │
│                                                              │
│  NBIS REQUIREMENT:                                           │
│  All MOSIP-certified fingerprint sensors must pass          │
│  PAD Level 1 or PAD Level 2 testing under ISO 30107-3.     │
│  ├── PAD Level 1: basic spoof detection (2D prints)        │
│  └── PAD Level 2: advanced spoof detection (3D molds)      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 20.11 MOSIP Device Specification (MDS)

In MOSIP, every fingerprint sensor must be **certified** against the MOSIP Device Specification. This is a key security control:

```
┌──────────────────────────────────────────────────────────────┐
│              MOSIP Device Specification (MDS)                │
└──────────────────────────────────────────────────────────────┘

WHY MDS?
  Without certification:
  ├── Any device (including software fake) can send data
  ├── Attacker writes software that generates fake fingerprints
  └── NBIS has no way to know capture is genuine

  With MDS:
  ├── Device has a chip with a unique device key
  ├── Every capture: device signs the image with its key
  ├── Server verifies: did this capture come from a
  │   MOSIP-certified device?
  └── Software fake: cannot sign with a valid device key

MDS REQUIREMENTS FOR FINGERPRINT SENSORS:
  ├── Hardware security: device stores keys in secure element
  ├── Certified by: MOSIP partner / government-approved lab
  ├── PAD level: minimum PAD Level 1
  ├── Resolution: minimum 500 PPI
  ├── Quality: NFIQ2 support built-in
  └── Interface: MDS-compliant API (MOSIP standard)

MDS CAPTURE PACKET:
  {
    "deviceCode":    "FP-SENSOR-MODEL-X",
    "deviceServiceVersion": "1.2.3",
    "env":           "Production",
    "domainUri":     "https://nbis.gov.bh",
    "captureTime":   "2025-01-15T10:30:00Z",
    "transactionId": "TXN-001",
    "specVersion":   ["0.9.5"],
    "biometrics": [{
      "specVersion":  "0.9.5",
      "data":         "base64-encoded-JWS",
      "sessionKey":   "base64-encrypted-session-key",
      "thumbprint":   "sha256-of-device-cert",
      "error":        null
    }]
  }

  "data" field is a JWS (JSON Web Signature):
  Header.Payload.Signature
  ├── Header: device key ID, algorithm
  ├── Payload: fingerprint image + metadata
  └── Signature: device's private key signature
  → Server verifies signature using device's public cert
```

---

## 20.12 Template Extraction Process

```
┌──────────────────────────────────────────────────────────────┐
│              Fingerprint Template Extraction                 │
└──────────────────────────────────────────────────────────────┘

INPUT: Fingerprint image (500 PPI, 8-bit grayscale, WSQ)
         │
         ▼
STEP 1: Image Enhancement
  ├── Orientation field estimation
  │   (which direction ridges run at each point)
  ├── Frequency estimation
  │   (how closely packed ridges are)
  ├── Gabor filtering (enhance ridges, suppress noise)
  └── Binarization (ridges → black, valleys → white)

STEP 2: Segmentation
  └── Separate foreground (ridge area) from background

STEP 3: Thinning
  └── Ridge lines reduced to single-pixel width
      (makes minutiae detection more accurate)

STEP 4: Minutiae Detection
  ├── Scan thinned image for:
  │   ├── Ridge endings (ridge line stops)
  │   └── Bifurcations (ridge line splits)
  └── Record for each minutia:
      ├── X coordinate
      ├── Y coordinate
      ├── Angle (direction of ridge at that point)
      └── Type (ending or bifurcation)

STEP 5: Minutiae Filtering
  ├── Remove spurious minutiae (noise artifacts)
  ├── Remove boundary minutiae (unreliable at edge)
  └── Keep 40–100 highest-quality minutiae

STEP 6: Template Encoding
  ├── Format: ISO/IEC 19794-2 (standard format)
  ├── Size: 300–800 bytes per finger
  └── Output: compact binary template

OUTPUT: ISO 19794-2 template
         │
         ▼
AES-256 ENCRYPTION (KMS per-citizen key)
         │
         ▼
S3 STORAGE (nbis-biometric-templates)
  Path: /{uinHash}/fingerprints/{fingerPosition}.enc
```

---

## 20.13 Fingerprint Matching — How 1:1 Verification Works

```
┌──────────────────────────────────────────────────────────────┐
│              Fingerprint 1:1 Matching Process                │
└──────────────────────────────────────────────────────────────┘

Authentication Request:
  Citizen claims UIN: 123456789012
  Citizen provides: live fingerprint (right index)
         │
         ▼
Auth Service:
  1. Fetch enrolled template from S3
     Key: /{uinHash}/fingerprints/RIGHT_INDEX.enc
  2. Decrypt template (KMS)
  3. Extract probe template from live capture
  4. Run matching algorithm

MATCHING ALGORITHM (minutiae-based):
  PROBE template:    40 minutiae (from live capture)
  ENROLLED template: 65 minutiae (from enrollment)
         │
  Algorithm:
  ├── Find correspondences between probe and enrolled
  ├── Align templates (rotation + translation)
  ├── Count matching minutiae pairs
  │   (within tolerance: 10 pixels, 20 degrees)
  └── Compute match score:
      score = (matching_pairs / total_minutiae) × 100
      adjusted for quality and number of minutiae

SCORE INTERPRETATION:
  score ≥ 85:  MATCH (proceed with authentication)
  score < 85:  NO MATCH (authentication fails)

  Note: threshold is configurable per deployment
  High security (border): threshold = 90
  Normal service (banking): threshold = 80

MATCH DECISION:
  ├── MATCH → auth token issued → service proceeds
  └── NO MATCH → increment attempt counter
                  → 3 failures → lock account 30 min
```

---

## 20.14 Fingerprint Quality vs Authentication Accuracy

```
┌──────────────────────────────────────────────────────────────┐
│         Enrollment Quality Impact on Authentication          │
├───────────────────────────────────────────────────────────────┤
│                                                              │
│  NFIQ2 at enrollment → Authentication accuracy impact:       │
│                                                              │
│  Enrolled NFIQ2 90 (excellent):                              │
│  ├── FRR at authentication: ~0.1%                           │
│  └── Citizen rarely fails their own fingerprint             │
│                                                              │
│  Enrolled NFIQ2 60 (good):                                   │
│  ├── FRR at authentication: ~1%                             │
│  └── 1 in 100 auth attempts fails (acceptable)             │
│                                                              │
│  Enrolled NFIQ2 40 (fair/minimum):                           │
│  ├── FRR at authentication: ~5–10%                          │
│  └── 1 in 10–20 auth attempts fails (problematic)          │
│      Citizen needs multiple attempts, loses confidence      │
│                                                              │
│  This is why QUALITY AT ENROLLMENT MATTERS SO MUCH:         │
│  A 2-minute effort to get NFIQ2 from 40 to 75              │
│  reduces lifetime authentication failures by 80–90%.        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 20.15 AWS Architecture for Fingerprint Capture Processing

```
┌──────────────────────────────────────────────────────────────┐
│         Fingerprint Capture Processing — AWS                 │
└──────────────────────────────────────────────────────────────┘

Registration Center (MDS-certified device)
  │
  │ MDS-signed biometric payload (JWS)
  │ Encrypted enrollment packet
  ▼
API Gateway → Registration Service (ECS)
  │
  ├──► S3: store encrypted packet
  ├──► SQS: publish ENROLLMENT_RECEIVED
  └──► DynamoDB: status = RECEIVED

SQS → Biometric Service (ECS Fargate)
  │
  ├──► STEP 1: Verify MDS device signature
  │    └── Validate JWS with device cert from
  │        Device Registry (DynamoDB)
  │
  ├──► STEP 2: Decrypt biometric payload (KMS)
  │
  ├──► STEP 3: NFIQ2 quality check
  │    └── NFIQ2 SDK (runs in ECS container)
  │        ├── Score < 40: reject → DLQ + notify
  │        └── Score ≥ 40: proceed
  │
  ├──► STEP 4: Template extraction
  │    └── ISO 19794-2 extractor (SDK in container)
  │
  ├──► STEP 5: Encrypt template (KMS, per-citizen key)
  │
  ├──► STEP 6: Store template
  │    └── S3: s3://nbis-templates/{uinHash}/fp/{pos}.enc
  │
  ├──► STEP 7: Update enrollment status
  │    └── DynamoDB: status = BIOMETRIC_COMPLETE
  │
  └──► STEP 8: Publish to SQS
       └── nbis-abis-requests (deduplication next)

AWS SDK / SERVICES USED:
  ├── ECS Fargate: biometric processing container
  ├── S3 + KMS:    encrypted template storage
  ├── DynamoDB:    device registry + status tracking
  ├── SQS:         async pipeline
  └── CloudWatch:  quality metrics + alarms

CLOUDWATCH METRICS (fingerprint-specific):
  ├── average_nfiq2_score (per center, per day)
  ├── recapture_rate (% of fingers re-captured)
  ├── exception_rate (% requiring supervisor)
  └── below_threshold_rate (% NFIQ2 < 40)
```

---

## 20.16 MOSIP Reference

```
┌──────────────────────────────────────────────────────────────┐
│              MOSIP Fingerprint Capture Reference             │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  MOSIP Registration Client biometric capture:               │
│  ├── Uses MOSIP Device Specification (MDS) interface        │
│  ├── Calls device SDK via MDS-compliant API                │
│  ├── Displays NFIQ2 quality score per finger               │
│  ├── Supports up to 3 recapture attempts per finger        │
│  └── Exception documentation built into UI                │
│                                                              │
│  MOSIP biometric attributes for fingerprint:               │
│  ├── leftIndex, leftMiddle, leftRing, leftLittle           │
│  ├── leftThumb                                              │
│  ├── rightIndex, rightMiddle, rightRing, rightLittle       │
│  ├── rightThumb                                             │
│  └── Configured in MOSIP ID Object Definition (JSON)       │
│                                                              │
│  MOSIP biometric exception handling:                        │
│  ├── Operator marks finger as exception in UI              │
│  ├── Reason code selected from dropdown                    │
│  ├── Exception type stored in packet                       │
│  └── Supervisor approval flow if required                  │
│                                                              │
│  ABIS integration for deduplication:                        │
│  ├── MOSIP sends: ISO 19794-2 templates via ABIS queue     │
│  ├── ABIS performs 1:N matching across all enrolled        │
│  ├── ABIS returns: candidate list + match scores           │
│  └── Score above threshold → DEDUP_FLAGGED event          │
│                                                              │
│  Known MOSIP-certified fingerprint sensor vendors:          │
│  ├── Mantra (MFS100, MFS110)                               │
│  ├── Morpho (MSO Series)                                   │
│  ├── Secugen (Hamster Pro series)                          │
│  └── Idemia (MA VP series)                                 │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 20.17 Key Terms

| Term | Definition |
|------|-----------|
| **Fingerprint capture** | Recording fingerprint ridge patterns using a biometric sensor |
| **Minutiae** | Specific ridge features — endings and bifurcations — that make fingerprints unique |
| **NFIQ2** | NIST Fingerprint Image Quality 2 — standard quality score (0–100) |
| **Rolled print** | Finger rolled nail-to-nail across platen — captures maximum ridge area |
| **Slap print** | Multiple fingers placed flat simultaneously — faster, less area per finger |
| **WSQ** | Wavelet Scalar Quantization — FBI standard compression for fingerprint images |
| **ISO 19794-2** | International standard format for fingerprint minutiae templates |
| **ISO 19794-4** | International standard format for fingerprint image data |
| **PAD** | Presentation Attack Detection — liveness detection for biometrics |
| **ISO 30107-3** | Standard for evaluating biometric presentation attack detection |
| **MDS** | MOSIP Device Specification — certification standard for biometric sensors |
| **JWS** | JSON Web Signature — MDS-signed biometric capture payload |
| **Template** | Compact mathematical representation of fingerprint minutiae (not the image) |
| **FAR** | False Accept Rate — impostor incorrectly authenticated |
| **FRR** | False Reject Rate — genuine user incorrectly rejected |
| **Bifurcation** | A ridge that splits into two — one of the two main minutiae types |
| **Ridge ending** | A ridge that stops — one of the two main minutiae types |
| **500 PPI** | 500 pixels per inch — minimum capture resolution for NBIS enrollment |
| **Gabor filter** | Image processing technique that enhances ridge patterns in fingerprints |
| **Match score** | Numerical similarity score between probe and enrolled template (0–100) |
| **AFIS** | Automated Fingerprint Identification System — 1:N search engine |

---

## 20.18 Key Takeaways

- **Fingerprint quality at enrollment determines lifetime authentication accuracy** — a 2-minute effort to get NFIQ2 from 40 to 75 reduces authentication failures by 80–90% for that citizen's entire lifetime.
- **Rolled prints are the gold standard** — they capture the maximum ridge area and give the ABIS the most data for accurate deduplication. Never skip rolled prints for slap-only capture.
- **NFIQ2 < 40 is a hard reject** — below this threshold, the template is unreliable and will cause frequent false non-matches. Three attempts; if still below 40, document the exception.
- **Ten fingers means ten backups** — the system should use all available fingers. If the right index is injured on the day of authentication, the left index is available. Design authentication to try alternate fingers on failure.
- **MDS certification is a security control, not a procurement preference** — an uncertified device can be replaced by software generating fake biometrics. MDS device signature prevents this.
- **Templates are not images** — the ISO 19794-2 minutiae template cannot be used to reconstruct the original fingerprint image. This is a critical privacy property: even if templates are stolen, fingers cannot be cloned from them.
- **Liveness detection is mandatory** — without PAD, a gelatin mold of a fingerprint passes the sensor. Every NBIS sensor must meet ISO 30107-3 PAD Level 1 minimum.
- **Manual laborers and elderly citizens will have low NFIQ2** — this is not an edge case. In many countries it affects 10–20% of the adult population. Officers must be trained in techniques for these cases, and the exception workflow must be smooth.

---

## 20.19 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 21 | Face Capture — ICAO compliance, pose, lighting, liveness |
| Chapter 22 | Iris Capture — NIR camera, IrisCode, quality metrics |
| Chapter 23 | Signature Capture — digital pad, format, use cases |

---

*Chapter 20 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*
