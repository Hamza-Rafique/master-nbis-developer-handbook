# Chapter 5 — Stakeholders

> **Part I — Government Identity Systems**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 5.1 What is a Stakeholder?

A **stakeholder** is any person, organization, or system that has an interest in, responsibility for, or interaction with the NBIS.

Understanding stakeholders is not a soft skill exercise. It is a **systems design requirement**. Every stakeholder has:
- Data they own or consume
- Permissions they need (RBAC roles)
- APIs they call or expose
- SLAs they expect
- Compliance obligations they impose

If you do not map your stakeholders before designing the system, you will build APIs nobody can use, RBAC roles that do not match reality, and audit logs that satisfy no regulator.

```
┌─────────────────────────────────────────────────────────────────┐
│                    NBIS Stakeholder Map                         │
│                                                                 │
│   Government           Technology          Citizens             │
│   ──────────           ──────────          ────────             │
│   Ministry             System              Individual           │
│   Regulator            Integrator          citizen              │
│   Law enforcement      ABIS vendor         Resident             │
│   Civil registry       Device vendor       Diaspora             │
│                        Cloud provider                           │
│                                                                 │
│   Relying Parties      International       Civil Society        │
│   ─────────────        ─────────────       ─────────────        │
│   Banks                ICAO                Privacy NGOs         │
│   Hospitals            World Bank          Legal aid orgs       │
│   Telecoms             UNHCR               Academic             │
│   Border control       MOSIP ecosystem     researchers          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 Stakeholder Group 1 — Government Entities

### 5.2.1 Ministry of Interior / National ID Authority

The **primary owner and operator** of the NBIS. This ministry is responsible for:

```
┌──────────────────────────────────────────────────────────────┐
│           Ministry of Interior / NID Authority               │
├──────────────────────┬───────────────────────────────────────┤
│ Owns                 │ The CIDR (Central Identity Data Repo) │
│ Sets policy          │ Who can enroll, what data is captured │
│ Operates             │ Registration centers                  │
│ Approves             │ Relying party onboarding              │
│ Enforces             │ Data retention and privacy policy     │
│ Reports to           │ Parliament / Cabinet                  │
└──────────────────────┴───────────────────────────────────────┘
```

**RBAC role:** `SYSTEM_ADMIN`, `POLICY_ADMIN`

**Key concerns:** Accuracy of data, prevention of fraud, political accountability, budget, public trust.

---

### 5.2.2 Civil Registry Authority

The civil registry is operated by a **separate ministry** (often Ministry of Justice or Ministry of Interior in a different department). It is the upstream data source for the NBIS.

```
Civil Registry Authority
      │
      │ provides
      ▼
Birth certificates ──────────► NBIS enrollment trigger
Death certificates ──────────► NBIS revocation trigger
Marriage records   ──────────► NBIS demographic update
Name change orders ──────────► NBIS name update
```

**Integration pattern:** The civil registry and NBIS must share an **event-driven integration** — not a shared database. Each system publishes events; the other consumes them. This preserves system independence while keeping data synchronized.

**RBAC role:** `CIVIL_REGISTRY_INTEGRATOR` (machine-to-machine service account)

---

### 5.2.3 Ministry of Foreign Affairs

Responsible for the **passport system** and **consular identity services** — issuing national ID documents to citizens living abroad.

```
┌──────────────────────────────────────────────────────────────┐
│           Ministry of Foreign Affairs Interactions           │
├──────────────────────┬───────────────────────────────────────┤
│ Reads from NBIS      │ Citizen identity data for passport    │
│ Writes to NBIS       │ Consular enrollment (diaspora)        │
│ Integrates with      │ ICAO PKD (Public Key Directory)       │
│ Operates             │ Embassies as enrollment stations      │
└──────────────────────┴───────────────────────────────────────┘
```

**RBAC role:** `PASSPORT_OFFICER`, `CONSULAR_OFFICER`

---

### 5.2.4 Law Enforcement / Intelligence

Access to identity data for **criminal investigation and border security** purposes — the most sensitive access in the entire ecosystem.

```
┌──────────────────────────────────────────────────────────────┐
│              Law Enforcement Access Model                    │
├──────────────────────┬───────────────────────────────────────┤
│ Access type          │ Court-ordered query only              │
│ What they can see    │ Identity record for specific UIN      │
│ What they CANNOT     │ Bulk export, biometric database dump  │
│ Audit requirement    │ Every query logged with warrant ref.  │
│ Oversight            │ Judicial authorization required       │
└──────────────────────┴───────────────────────────────────────┘
```

**RBAC role:** `LAW_ENFORCEMENT_QUERY` (most restricted, every action audited)

**Key principle:** Law enforcement access must be **narrow, logged, and authorized** — not open-ended. A poorly designed RBAC model that gives law enforcement broad query access to the CIDR is a fundamental privacy failure.

---

### 5.2.5 Data Protection Authority / Regulator

The independent body that **oversees** how personal data is collected, stored, and used. They do not operate the NBIS — they audit it.

```
┌──────────────────────────────────────────────────────────────┐
│             Data Protection Authority Role                   │
├──────────────────────┬───────────────────────────────────────┤
│ Audits               │ Data handling practices               │
│ Reviews              │ Privacy impact assessments            │
│ Investigates         │ Data breach incidents                 │
│ Enforces             │ Penalties for non-compliance          │
│ Approves             │ Cross-border data transfers           │
└──────────────────────┴───────────────────────────────────────┘
```

**RBAC role:** `AUDITOR` (read-only access to audit logs, no citizen data)

---

## 5.3 Stakeholder Group 2 — Citizens and Residents

### 5.3.1 Citizens

The **primary subject** of the entire system. Every design decision ultimately affects the citizen experience.

```
┌──────────────────────────────────────────────────────────────┐
│                    Citizen Interactions                      │
├──────────────────────┬───────────────────────────────────────┤
│ Enrolls              │ Attends registration center           │
│ Receives             │ UIN + credential (card / mobile VC)  │
│ Uses                 │ Identity for services, border, eKYC  │
│ Updates              │ Address, name, biometrics via portal  │
│ Views                │ Authentication history via portal     │
│ Consents             │ Data sharing with relying parties     │
│ Locks                │ Biometric auth (self-service)         │
└──────────────────────┴───────────────────────────────────────┘
```

**RBAC role:** `RESIDENT` (can only access their own record via authenticated portal)

**Key principle:** Citizens must be able to **see who accessed their data and when**. This transparency is not optional — it is the social contract that makes public trust in NBIS possible.

---

### 5.3.2 Residents (Foreign Nationals)

Same enrollment process as citizens but with **additional permit-driven lifecycle events** (expiry, renewal, deportation). Their record is linked to their visa or residency permit status.

**RBAC role:** `RESIDENT` (same as citizen for self-service, different record type in CIDR)

---

### 5.3.3 Diaspora / Citizens Abroad

Citizens living outside the country who need to access NBIS services — enrollment, renewal, updates — through **embassies and consular offices** acting as remote registration centers.

This creates a special enrollment challenge: the biometric capture hardware must be deployed at embassies worldwide, and the enrollment packets must be securely transmitted back to the CIDR over public networks.

```
Embassy enrollment station (remote)
      │
      ▼ encrypted enrollment packet
    Internet (mTLS)
      │
      ▼
National ID Authority (home country)
      │
      ▼
Registration Processor → Deduplication → UIN
```

---

## 5.4 Stakeholder Group 3 — Enrollment Operators

### 5.4.1 Enrollment Officers

Front-line staff who operate the biometric capture workstations at registration centers.

```
┌──────────────────────────────────────────────────────────────┐
│                  Enrollment Officer Role                     │
├──────────────────────┬───────────────────────────────────────┤
│ Can do               │ Create new enrollment                 │
│                      │ Capture demographics + biometrics     │
│                      │ View enrollment status                │
│                      │ Update incomplete enrollments         │
│ Cannot do            │ Query the citizen registry            │
│                      │ View other officers' enrollments      │
│                      │ Modify issued credentials             │
│                      │ Access audit logs                     │
└──────────────────────┴───────────────────────────────────────┘
```

**RBAC role:** `ENROLLMENT_OFFICER`

**Critical security principle:** Enrollment officers must **not** be able to query the citizen registry. They process new enrollments — they do not have access to existing records. This prevents insider threats (enrolling a fake person who matches a real resident's data).

---

### 5.4.2 Registration Center Supervisors

Manage the enrollment officers and handle **exception cases** — incomplete enrollments, quality failures, suspected duplicates.

```
┌──────────────────────────────────────────────────────────────┐
│               Registration Center Supervisor Role            │
├──────────────────────┬───────────────────────────────────────┤
│ Can do               │ All enrollment officer actions        │
│                      │ Approve / reject flagged enrollments  │
│                      │ Override quality checks (with reason) │
│                      │ View center-level reports             │
│ Cannot do            │ Access CIDR directly                  │
│                      │ Modify issued credentials             │
└──────────────────────┴───────────────────────────────────────┘
```

**RBAC role:** `CENTER_SUPERVISOR`

---

## 5.5 Stakeholder Group 4 — Relying Parties

Relying parties are the **consumers** of the identity system — the organizations that call the verification API to confirm a person's identity before granting access to services.

```
┌──────────────────────────────────────────────────────────────┐
│                    Relying Party Types                       │
├────────────────────┬─────────────────────────────────────────┤
│ Banks / fintechs   │ eKYC for account opening, AML checks   │
│ Telecoms           │ SIM registration, subscriber KYC       │
│ Hospitals          │ Patient identity, medical record link  │
│ Border control     │ Entry / exit verification              │
│ Social services    │ Benefits eligibility verification      │
│ Electoral body     │ Voter registration, polling station    │
│ Tax authority      │ Taxpayer identity verification         │
│ Universities       │ Student enrollment, certificate issue  │
└────────────────────┴─────────────────────────────────────────┘
```

### Relying party onboarding

A relying party cannot simply start calling the verification API. They must go through a **formal onboarding process**:

```
Relying party applies to NID Authority
      │
      ▼
Legal agreement signed (data use terms)
      │
      ▼
Technical integration review
      │
      ▼
API credentials issued (client ID + secret)
      │
      ▼
Sandbox testing
      │
      ▼
Production access granted
      │
      ▼
Ongoing audit (quarterly usage review)
```

**RBAC role:** `RELYING_PARTY` (can call verification / eKYC API only — no direct registry access)

**Key constraint:** Relying parties **never** receive biometric data. They receive a yes/no match result, or a signed set of consented demographic attributes. Raw biometrics never leave the NBIS.

---

## 5.6 Stakeholder Group 5 — Technology Partners

### 5.6.1 System Integrator (SI)

The organization that **builds and implements** the NBIS for the government. In MOSIP deployments, the SI configures MOSIP modules, integrates with the ABIS vendor, connects the civil registry, deploys the infrastructure, and trains government staff.

```
┌──────────────────────────────────────────────────────────────┐
│                  System Integrator Role                      │
├──────────────────────┬───────────────────────────────────────┤
│ Builds               │ Custom modules, integrations          │
│ Configures           │ MOSIP platform, ABIS, workflows       │
│ Deploys              │ Infrastructure (AWS / on-prem)        │
│ Trains               │ Government staff                      │
│ Hands over           │ Full system ownership to government   │
└──────────────────────┴───────────────────────────────────────┘
```

**RBAC role:** `SYSTEM_ADMIN` (during implementation, removed at handover)

---

### 5.6.2 ABIS Vendor

The company providing the **Automated Biometric Identification System** — the engine that performs 1:N deduplication and biometric matching.

```
┌──────────────────────────────────────────────────────────────┐
│                      ABIS Vendor Role                        │
├──────────────────────┬───────────────────────────────────────┤
│ Provides             │ Matching engine + dedup algorithms    │
│ Receives             │ Biometric probe + existing templates  │
│ Returns              │ Match score + candidate list          │
│ Does NOT store       │ Citizen records (templates only)      │
│ Certified by         │ NIST MINEX / IREX / FRVT             │
└──────────────────────┴───────────────────────────────────────┘
```

**Integration pattern:** MOSIP does not include an ABIS — it defines an **ABIS middleware interface** (a queue-based contract over SQS/Kafka). The ABIS vendor implements their engine behind this interface. The government owns the templates; the ABIS vendor never has direct database access.

**Known ABIS vendors:** Idemia, NEC, Aware, Neurotechnology, Daon

---

### 5.6.3 Biometric Device Vendor

Manufactures and certifies the **capture hardware** — fingerprint scanners, iris cameras, face capture devices.

```
┌──────────────────────────────────────────────────────────────┐
│              Biometric Device Vendor Role                    │
├──────────────────────┬───────────────────────────────────────┤
│ Provides             │ Certified capture devices             │
│ Certified by         │ MOSIP Device Specification (MDS)      │
│ SDK                  │ Device driver + quality SDK           │
│ Security             │ Device must sign captured data        │
└──────────────────────┴───────────────────────────────────────┘
```

**Key point:** In MOSIP, every biometric capture device must be **certified** against the MOSIP Device Specification (MDS). Uncertified devices are rejected. The device cryptographically signs each capture — proving the data came from a certified device, not a software fake.

---

### 5.6.4 Cloud / Infrastructure Provider

The organization providing the compute, storage, and networking infrastructure on which the NBIS runs.

```
┌──────────────────────────────────────────────────────────────┐
│              Cloud Provider Considerations                   │
├──────────────────────┬───────────────────────────────────────┤
│ AWS services used    │ ECS, API Gateway, Lambda, DynamoDB,   │
│                      │ RDS, S3, KMS, CloudHSM, CloudWatch    │
│ Data sovereignty     │ CIDR must stay within national        │
│                      │ borders (government requirement)      │
│ Compliance           │ ISO 27001, SOC 2, FedRAMP             │
│ Key management       │ Government holds master keys          │
└──────────────────────┴───────────────────────────────────────┘
```

**Critical requirement:** The government must **hold the encryption keys** — not the cloud provider. Even if AWS hosts the infrastructure, the keys in CloudHSM are controlled by the government. This is **data sovereignty** in practice.

---

### 5.6.5 Credential Vendor

Manufactures and personalizes the **physical credentials** — smart ID cards, ePassport booklets.

```
Credential Vendor responsibilities:
      │
      ├──► Smart card chip programming
      ├──► Card body printing (photo, name, MRZ)
      ├──► PKI certificate loading onto chip
      ├──► Quality inspection
      └──► Secure delivery to citizen
```

---

## 5.7 Stakeholder Group 6 — International Organizations

```
┌────────────────────────────────────────────────────────────────┐
│               International Stakeholders                       │
├───────────────────┬────────────────────────────────────────────┤
│ ICAO              │ Sets standards for travel documents        │
│                   │ (Doc 9303, CSCA, PKD)                     │
├───────────────────┼────────────────────────────────────────────┤
│ World Bank / ID4D │ Funds and advises national ID programs     │
│                   │ in developing countries                    │
├───────────────────┼────────────────────────────────────────────┤
│ UNHCR             │ Refugee identity — coordinates with NBIS  │
│                   │ for refugee enrollment and documentation   │
├───────────────────┼────────────────────────────────────────────┤
│ MOSIP ecosystem   │ Open-source platform, technical support,  │
│                   │ partner network (IIIT-B)                  │
├───────────────────┼────────────────────────────────────────────┤
│ NIST              │ Sets biometric standards (MINEX, IREX,    │
│                   │ FRVT) and IAL/AAL levels (SP 800-63)      │
└───────────────────┴────────────────────────────────────────────┘
```

---

## 5.8 Stakeholder Group 7 — Civil Society

Often overlooked in technical design, but critical for **public trust and adoption**.

```
┌──────────────────────────────────────────────────────────────┐
│               Civil Society Stakeholders                     │
├────────────────────┬─────────────────────────────────────────┤
│ Privacy NGOs       │ Monitor data use, advocate for          │
│                    │ privacy-by-design                       │
├────────────────────┼─────────────────────────────────────────┤
│ Legal aid orgs     │ Support excluded populations in         │
│                    │ accessing identity documents            │
├────────────────────┼─────────────────────────────────────────┤
│ Academic           │ Research on system accuracy, bias in    │
│ researchers        │ biometric algorithms (especially for    │
│                    │ dark skin tones, elderly)               │
├────────────────────┼─────────────────────────────────────────┤
│ Journalists        │ Investigative reporting on data         │
│                    │ breaches, government overreach          │
└────────────────────┴─────────────────────────────────────────┘
```

**Why engineers must care:** Biometric algorithms have documented accuracy disparities across demographic groups. An engineer who understands this will insist on demographic testing of the ABIS before deployment. An engineer who ignores it builds a system that discriminates against the people it was designed to serve.

---

## 5.9 Complete RBAC Role Matrix

```
┌─────────────────────────────┬───────┬───────┬───────┬──────┬────────┬────────┐
│ Action                      │ ENRL  │ SUPVR │ ADMIN │ RSDNT│ RP     │ AUDIT  │
│                             │ OFFCR │       │       │      │        │        │
├─────────────────────────────┼───────┼───────┼───────┼──────┼────────┼────────┤
│ Create enrollment           │  ✅   │  ✅   │  ✅   │  ❌  │  ❌    │  ❌    │
│ Capture biometrics          │  ✅   │  ✅   │  ❌   │  ❌  │  ❌    │  ❌    │
│ Approve flagged enrollment  │  ❌   │  ✅   │  ✅   │  ❌  │  ❌    │  ❌    │
│ Query citizen record        │  ❌   │  ❌   │  ✅   │  ❌  │  ❌    │  ❌    │
│ Update own record           │  ❌   │  ❌   │  ✅   │  ✅  │  ❌    │  ❌    │
│ Call verification API       │  ❌   │  ❌   │  ❌   │  ❌  │  ✅    │  ❌    │
│ Call eKYC API               │  ❌   │  ❌   │  ❌   │  ❌  │  ✅    │  ❌    │
│ Revoke identity             │  ❌   │  ❌   │  ✅   │  ❌  │  ❌    │  ❌    │
│ Read audit logs             │  ❌   │  ❌   │  ✅   │  ❌  │  ❌    │  ✅    │
│ View own auth history       │  ❌   │  ❌   │  ❌   │  ✅  │  ❌    │  ❌    │
│ Manage users / roles        │  ❌   │  ❌   │  ✅   │  ❌  │  ❌    │  ❌    │
│ Export bulk data            │  ❌   │  ❌   │  ❌   │  ❌  │  ❌    │  ❌    │
└─────────────────────────────┴───────┴───────┴───────┴──────┴────────┴────────┘

ENRL OFFCR = Enrollment Officer
SUPVR      = Center Supervisor
ADMIN      = System Administrator
RSDNT      = Resident (citizen self-service)
RP         = Relying Party
AUDIT      = Auditor
```

**Note:** No role has bulk export access. Not even the system administrator. Bulk data export requires a separate, judicially authorized process with full audit trail.

---

## 5.10 Stakeholder Communication Map

As an NBIS engineer, you will interact differently with each stakeholder group.

```
┌──────────────────────┬─────────────────────────────────────────┐
│ Stakeholder          │ What they care about (talk their lang.) │
├──────────────────────┼─────────────────────────────────────────┤
│ Ministry official    │ Cost, timeline, political risk, numbers │
│ Regulator            │ Compliance, audit trails, legal basis   │
│ Enrollment officer   │ Ease of use, speed, what to do on error │
│ Bank (relying party) │ API uptime, response time, SLA, eKYC    │
│ ABIS vendor          │ Throughput, accuracy rates, integration │
│ Privacy NGO          │ Data minimization, consent, breach risk │
│ Citizen              │ "Does this work? Is my data safe?"      │
└──────────────────────┴─────────────────────────────────────────┘
```

---

## 5.11 AWS Mapping

```
┌──────────────────────────────────┬──────────────────────────────┐
│ Stakeholder Need                 │ AWS Implementation           │
├──────────────────────────────────┼──────────────────────────────┤
│ RBAC for all roles               │ AWS IAM + Cognito user pools │
│ Relying party API access         │ API Gateway + usage plans    │
│ Audit log (all stakeholders)     │ CloudWatch Logs + Object Lock│
│ Civil registry event integration │ EventBridge + Lambda         │
│ Law enforcement query (audited)  │ API Gateway + CloudTrail     │
│ Data sovereignty (key control)   │ CloudHSM (govt holds keys)   │
│ ABIS vendor integration          │ SQS (queue-based interface)  │
│ Device certification check       │ Lambda (signature verify)    │
└──────────────────────────────────┴──────────────────────────────┘
```

---

## 5.12 Key Terms

| Term | Definition |
|------|-----------|
| **Stakeholder** | Any person or organization with an interest in or interaction with the NBIS |
| **Relying Party** | Organization that calls the NBIS verification API to confirm identity |
| **System Integrator** | Company that builds and implements the NBIS for the government |
| **ABIS Vendor** | Company providing the biometric matching and deduplication engine |
| **RBAC** | Role-Based Access Control — permissions attached to roles, not individuals |
| **Data Sovereignty** | Government retains control of its citizens' data and the encryption keys |
| **MOSIP Device Spec (MDS)** | MOSIP's certification standard for biometric capture devices |
| **Relying Party Onboarding** | Formal process to grant a private organization access to the verification API |
| **Enrollment Officer** | Front-line staff operating biometric capture workstations |
| **Principle of Least Privilege** | Every role gets only the minimum access needed for their function |
| **Bulk Export** | Extraction of large volumes of citizen data — should never be permitted via role |
| **ICAO PKD** | Public Key Directory — international repository of country signing certificates |
| **ID4D** | Identification for Development — World Bank program funding NBIS in developing nations |

---

## 5.13 Key Takeaways

- **Stakeholder mapping is a design input**, not an afterthought. Every stakeholder maps to RBAC roles, API endpoints, audit requirements, and SLAs.
- **No single stakeholder should have unrestricted access** — not the Ministry, not the System Admin, not law enforcement. The principle of least privilege applies universally.
- **Relying parties never receive raw biometrics** — they get a yes/no match result or a signed set of consented attributes. This is non-negotiable.
- **ABIS vendors never own the data** — they process templates; the government owns them. The queue-based ABIS interface in MOSIP enforces this separation.
- **Enrollment officers cannot query the registry** — a key insider-threat control. They process new enrollments only.
- **Civil society and privacy organizations are legitimate stakeholders** — their concerns about algorithmic bias and data misuse are engineering problems, not political ones.
- **Data sovereignty means the government holds the keys** — CloudHSM, not AWS-managed KMS, for the master encryption keys.

---

## 5.14 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 6 | Overall NBIS Architecture — full technical system diagram, all components |
| Chapter 7 | Microservices — how each service is structured and why |
| Chapter 10 | Authorization (RBAC) — deep dive into role design and enforcement |

---

*Chapter 5 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*
