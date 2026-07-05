# Chapter 1 — What is NBIS?

> **Part I — Government Identity Systems**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 1.1 Definition

A **National Biometric Identity System (NBIS)** is a government-managed digital infrastructure that enrolls, deduplicates, stores, and verifies the identities of every citizen and resident within a country's jurisdiction.

It is the **authoritative source of truth** for who a person is — legally, digitally, and biometrically.

---

## 1.2 Why NBIS Exists

Before digital identity systems, governments relied on paper registries, physical ID cards, and manual verification. The problems were severe:

- **Ghost identities** — people enrolling multiple times under different names to claim multiple benefits
- **Fraud** — forged documents, impersonation at borders, duplicate pensions
- **Exclusion** — citizens with no formal documents could not access banking, healthcare, or voting
- **Inefficiency** — verifying identity required physical presence and manual document checks

NBIS solves all of these by creating one authoritative, biometrically-anchored record per person — unforgeable, deduplicated, and verifiable in milliseconds.

---

## 1.3 Core Functions

Every NBIS, regardless of the country, performs the same five functions:

```
┌─────────────────────────────────────────────────────────┐
│                     NBIS Core Functions                 │
├──────────────┬──────────────────────────────────────────┤
│ Enrollment   │ Capture biographic + biometric data      │
│ Deduplication│ Ensure one record per real person (1:N)  │
│ Issuance     │ Assign UIN + issue credential            │
│ Verification │ Confirm identity for relying parties     │
│ Lifecycle    │ Manage updates, renewals, deaths         │
└──────────────┴──────────────────────────────────────────┘
```

---

## 1.4 What an NBIS is NOT

| Common Misconception | Reality |
|----------------------|---------|
| NBIS = a database | It is a system. The database is one component. |
| NBIS = passport system | A passport is a travel document that relies on NBIS as its identity source. |
| NBIS = surveillance system | NBIS verifies identity on request — it does not track movement. |
| NBIS = one application | It is a distributed system of microservices, APIs, and integrations. |

---

## 1.5 NBIS vs Related Systems

```
Civil Registry          →  Records vital events (birth, death, marriage)
                              ↓
National ID System      →  Converts civil records into digital identities
                              ↓
Population Registry     →  Tracks who is physically in the country
                              ↓
NBIS                    →  Adds biometrics + deduplication + verification API
                              ↓
Relying Party Systems   →  Banks, hospitals, borders use NBIS to verify
```

---

## 1.6 Real-World Examples

| Country | System | Scale |
|---------|--------|-------|
| India | Aadhaar (UIDAI) | 1.4 billion enrollments |
| Estonia | e-ID | 1.3 million citizens, 99% digital services |
| Ghana | Ghana Card (NIA) | 17 million+ enrollments |
| Philippines | PhilSys | MOSIP-based, 70M+ enrolled |
| Morocco | CNIE | MOSIP-based |
| Bahrain | CPR (Central Population Registry) | Full biometric national ID |

---

## 1.7 MOSIP — The Open-Source NBIS Platform

**MOSIP (Modular Open Source Identity Platform)** is the closest thing to a reference implementation of an NBIS. It is:

- Open-source under Mozilla Public License 2.0
- Built and maintained by IIIT-Bangalore (non-profit)
- Used by 20+ countries as the foundation of their national ID systems
- API-first, microservices-based, vendor-neutral

MOSIP is the primary reference platform throughout this handbook. When we study how enrollment works, how deduplication is triggered, or how the verification API is structured — MOSIP's architecture is the reference.

> Official documentation: [docs.mosip.io](https://docs.mosip.io)

---

## 1.8 NBIS Architecture — High Level

```
                        ┌──────────────────┐
                        │  Enrollment      │
                        │  Station         │
                        └────────┬─────────┘
                                 │ HTTPS
                        ┌────────▼─────────┐
                        │  API Gateway     │  ← single entry point
                        └────────┬─────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
    ┌─────────▼──────┐  ┌────────▼───────┐  ┌──────▼─────────┐
    │ Registration   │  │ Biometric      │  │ Verification   │
    │ Service        │  │ Service        │  │ Service        │
    └─────────┬──────┘  └────────┬───────┘  └──────┬─────────┘
              │                  │                  │
              └──────────────────▼──────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │  Central Identity Data  │
                    │  Repository (CIDR)      │
                    └─────────────────────────┘
```

---

## 1.9 Key Terms Introduced in This Chapter

| Term | Definition |
|------|-----------|
| **NBIS** | National Biometric Identity System |
| **UIN** | Unique Identity Number — the permanent ID assigned at enrollment |
| **CIDR** | Central Identity Data Repository — the authoritative record store |
| **ABIS** | Automated Biometric Identification System — performs 1:N deduplication |
| **Relying Party** | Any system (bank, hospital, border) that uses NBIS to verify identity |
| **Deduplication** | Ensuring one person = one record by comparing biometrics at enrollment |
| **1:1 Verification** | Confirming "is this person who they claim to be?" |
| **1:N Identification** | Finding "who is this person?" by searching all records |
| **MOSIP** | Open-source NBIS platform used as the reference throughout this handbook |

---

## 1.10 Key Takeaways

- An NBIS is a **system**, not a single application — it is a collection of microservices, databases, APIs, and integrations working together.
- Its foundational promise is **one person, one record** — enforced biometrically through deduplication at enrollment time.
- The **Civil Registry** provides legal identity; the **NBIS** digitizes and biometrically anchors it.
- **MOSIP** is the open-source reference platform. Understanding MOSIP's architecture means understanding how any NBIS works.
- Every function of an NBIS has a corresponding AWS service — this mapping will be built up across Part II.

---

## 1.11 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 2 | Identity Management Fundamentals — UIN, IAL levels, identity proofing |
| Chapter 3 | Government Identity Ecosystem — civil registry, population registry, passports, eKYC |
| Chapter 6 | Overall NBIS Architecture — full system diagram with all components |

---

*Chapter 1 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*
