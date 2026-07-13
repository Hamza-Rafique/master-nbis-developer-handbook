# Chapter 16 — Disaster Recovery

> **Part II — System Architecture**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 16.1 What is Disaster Recovery?

**Disaster Recovery (DR)** is the ability of a system to recover from a catastrophic failure — one that takes down an entire region, corrupts data, or renders the primary infrastructure permanently unavailable.

High Availability (Chapter 15) handles component failures within a single region. Disaster Recovery handles scenarios where the entire region is gone.

```
┌──────────────────────────────────────────────────────────────┐
│              HA vs DR — The Distinction                      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  HIGH AVAILABILITY handles:                                  │
│  ├── Single ECS task crashes                                │
│  ├── One AZ goes down (within same region)                  │
│  ├── RDS primary fails (standby takes over same region)     │
│  └── Application bug causes one service to fail            │
│                                                              │
│  DISASTER RECOVERY handles:                                  │
│  ├── Entire AWS region becomes unavailable                  │
│  ├── Catastrophic data corruption or deletion               │
│  ├── Ransomware attack encrypting all data                  │
│  ├── Government facility destroyed (physical disaster)      │
│  └── Extended outage beyond SLA tolerance                  │
│                                                              │
│  Key difference:                                             │
│  HA → automatic, seamless, seconds                          │
│  DR → planned, potentially manual, minutes to hours         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 16.2 RTO and RPO — The Two DR Metrics

Before designing a DR strategy, define two numbers. Everything else flows from them.

```
┌──────────────────────────────────────────────────────────────┐
│                    RTO and RPO Defined                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  RTO — Recovery Time Objective                               │
│  "How long can the system be down?"                         │
│  The maximum acceptable downtime after a disaster.          │
│                                                              │
│  RPO — Recovery Point Objective                              │
│  "How much data can we afford to lose?"                     │
│  The maximum acceptable data loss measured in time.         │
│                                                              │
└──────────────────────────────────────────────────────────────┘

TIMELINE:

          Disaster          Recovery
          occurs            complete
             │                │
─────────────●────────────────●──────────────► time
             │                │
             │◄──── RTO ─────►│
             │                │
─────────────●────────────────────────────────► time
    Last     │
    backup   │
    taken    │
             │◄── RPO ──►disaster point
             │
             │  This data is LOST if RPO is not met
```

### NBIS RTO and RPO targets

```
┌──────────────────────────────────────────────────────────────┐
│              NBIS DR Targets                                 │
├──────────────────────────┬───────────────┬───────────────────┤
│ System                   │ RTO           │ RPO               │
├──────────────────────────┼───────────────┼───────────────────┤
│ Verification API         │ 4 hours       │ 0 (no data loss)  │
│ (citizens need auth)     │               │                   │
├──────────────────────────┼───────────────┼───────────────────┤
│ Citizen Registry (CIDR)  │ 4 hours       │ 0 (no data loss)  │
│ (identity records)       │               │                   │
├──────────────────────────┼───────────────┼───────────────────┤
│ Enrollment pipeline      │ 24 hours      │ 1 hour            │
│ (new enrollments)        │               │ (lose < 1hr work) │
├──────────────────────────┼───────────────┼───────────────────┤
│ Audit logs               │ 24 hours      │ 0 (no data loss)  │
│                          │               │                   │
├──────────────────────────┼───────────────┼───────────────────┤
│ Admin console            │ 48 hours      │ 4 hours           │
├──────────────────────────┼───────────────┼───────────────────┤
│ Resident portal          │ 24 hours      │ 1 hour            │
└──────────────────────────┴───────────────┴───────────────────┘

Verification API and CIDR: RPO = 0
→ Identity records and biometric templates are the
  national asset. No data loss is acceptable.
```

---

## 16.3 DR Strategies — The Four Tiers

AWS defines four DR strategies at increasing cost and decreasing RTO:

```
┌──────────────────────────────────────────────────────────────┐
│              DR Strategy Spectrum                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  LOWEST COST ◄────────────────────────────► LOWEST RTO      │
│                                                              │
│  1. BACKUP & RESTORE                                         │
│     RTO: Hours to days                                      │
│     RPO: Hours (last backup)                                │
│     Cost: $ (storage only)                                  │
│     Method: Restore from S3 backups into new region        │
│                                                              │
│  2. PILOT LIGHT                                              │
│     RTO: 1–4 hours                                          │
│     RPO: Minutes (continuous replication)                   │
│     Cost: $$ (minimal DR infra running)                     │
│     Method: Core data replicated to DR region.             │
│             Minimal compute in DR (scaled down/off).        │
│             Scale up compute when disaster strikes.         │
│                                                              │
│  3. WARM STANDBY                                             │
│     RTO: 15–60 minutes                                      │
│     RPO: Seconds (near real-time replication)               │
│     Cost: $$$ (reduced-capacity DR always running)          │
│     Method: DR region runs at 25–50% capacity, always.     │
│             Scale up to full capacity on disaster.          │
│                                                              │
│  4. MULTI-SITE ACTIVE/ACTIVE                                 │
│     RTO: Near zero (minutes)                                │
│     RPO: Near zero (synchronous replication)               │
│     Cost: $$$$ (full capacity in two regions)               │
│     Method: Both regions serve live traffic always.        │
│             Failover = just stop routing to failed region.  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### NBIS DR Strategy — Warm Standby

Given the RTO targets (4 hours for verification), NBIS uses **Warm Standby**:

```
┌──────────────────────────────────────────────────────────────┐
│              NBIS Warm Standby DR Strategy                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PRIMARY REGION (e.g. ap-southeast-1 Singapore)             │
│  ├── Full capacity: 6 Auth tasks, all services running      │
│  ├── Serves 100% of traffic                                │
│  └── Continuously replicates data to DR region             │
│                                                              │
│  DR REGION (e.g. ap-south-1 Mumbai)                         │
│  ├── Reduced capacity: 2 Auth tasks (minimum)              │
│  ├── DynamoDB Global Table replica (real-time sync)        │
│  ├── S3 Cross-Region Replication (continuous)              │
│  ├── RDS Read Replica (async replication)                  │
│  ├── ElastiCache not replicated (ephemeral data)           │
│  └── Serves zero traffic normally (standby only)           │
│                                                              │
│  ON DISASTER:                                                │
│  ├── DNS failover: Route 53 redirects to DR region         │
│  ├── ECS auto-scaling: DR tasks scale to full capacity     │
│  ├── RDS Read Replica promoted to Primary                  │
│  ├── ElastiCache: fresh (citizens re-authenticate)         │
│  └── Full capacity restored in 15–60 minutes              │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 16.4 Data Replication Strategy

DR is only as good as the data in the DR region. Each data store has a specific replication strategy:

### 16.4.1 DynamoDB Global Tables

```
┌──────────────────────────────────────────────────────────────┐
│              DynamoDB Global Tables for NBIS                 │
└──────────────────────────────────────────────────────────────┘

SETUP:
  Primary table: ap-southeast-1 (Singapore)
  Replica table: ap-south-1 (Mumbai)

  aws dynamodb create-global-table \
    --global-table-name nbis-identity-records \
    --replication-group \
      RegionName=ap-southeast-1 \
      RegionName=ap-south-1

REPLICATION:
  Every write to Singapore → replicated to Mumbai
  Typical replication lag: < 1 second
  RPO for identity records: < 1 second

DURING NORMAL OPERATION:
  Singapore: reads + writes (all traffic)
  Mumbai: replica (no traffic, just sync)

ON DISASTER:
  Singapore region fails
  Route 53 → DNS points to Mumbai
  Mumbai DynamoDB already has all data (< 1 second old)
  Applications in Mumbai start reading/writing to Mumbai table
  RPO achieved: < 1 second data loss

TABLES REPLICATED (identity-critical):
  ├── nbis-identity-records     ← CIDR (most critical)
  ├── nbis-enrollment-status    ← enrollment tracking
  ├── nbis-auth-transactions    ← auth audit
  └── nbis-partner-configs      ← relying party configs

TABLES NOT REPLICATED (operational, rebuilt on recovery):
  ├── nbis-processed-events     ← idempotency (rebuilt)
  └── nbis-otp-cache            ← ephemeral (not in DynamoDB)
```

---

### 16.4.2 S3 Cross-Region Replication

```
┌──────────────────────────────────────────────────────────────┐
│              S3 Cross-Region Replication                     │
└──────────────────────────────────────────────────────────────┘

BUCKETS AND REPLICATION:

  nbis-biometric-templates (Singapore)
    │ Cross-Region Replication (CRR)
    ▼
  nbis-biometric-templates-dr (Mumbai)
  ├── Same KMS encryption (different regional key)
  ├── Same access policies
  ├── Replication lag: seconds to minutes
  └── RPO: minutes (acceptable — templates are stable)

  nbis-enrollment-packets (Singapore)
    │ Cross-Region Replication
    ▼
  nbis-enrollment-packets-dr (Mumbai)
  ├── Replication of unprocessed packets
  └── Allows enrollment pipeline to resume in DR

  nbis-credential-artifacts (Singapore)
    │ Cross-Region Replication
    ▼
  nbis-credential-artifacts-dr (Mumbai)
  ├── Issued VCs, QR codes, card print files
  └── Citizens can re-download credentials after DR

  nbis-audit-logs-archive (Singapore)
    │ Cross-Region Replication
    ▼
  nbis-audit-logs-archive-dr (Mumbai)
  ├── Object Lock enabled on both buckets
  ├── Immutable — cannot be deleted or modified
  └── Compliance requirement: retain even in disaster

S3 CRR CONFIGURATION:
  {
    "ReplicationConfiguration": {
      "Role": "arn:aws:iam::123:role/nbis-s3-replication",
      "Rules": [{
        "Status": "Enabled",
        "Filter": { "Prefix": "" },
        "Destination": {
          "Bucket": "arn:aws:s3:::nbis-biometric-templates-dr",
          "EncryptionConfiguration": {
            "ReplicaKmsKeyID": "arn:aws:kms:ap-south-1:..."
          }
        },
        "DeleteMarkerReplication": { "Status": "Disabled" }
      }]
    }
  }

  Note: DeleteMarkerReplication DISABLED
  → Deletes in primary region do NOT propagate to DR
  → Accidental deletions in primary cannot wipe DR copy
```

---

### 16.4.3 RDS Backup and Cross-Region Snapshot

```
┌──────────────────────────────────────────────────────────────┐
│              RDS DR Strategy                                 │
└──────────────────────────────────────────────────────────────┘

CONTINUOUS BACKUP (automated):
  RDS automated backups:
  ├── Retention period: 7 days
  ├── Point-in-time recovery: any second in last 7 days
  └── Stored in S3 (same region, encrypted)

CROSS-REGION AUTOMATED BACKUP:
  aws rds create-db-cluster-parameter-group (enable PITR)
  aws rds copy-db-snapshot \
    --source-db-snapshot-identifier nbis-rds-snapshot \
    --target-db-snapshot-identifier nbis-rds-snapshot-dr \
    --destination-region ap-south-1 \
    --kms-key-id arn:aws:kms:ap-south-1:...

  Frequency: Daily automated cross-region copy
  RPO for RDS: 24 hours (acceptable for audit/admin tables)
  RPO for identity data: covered by DynamoDB Global Tables

READ REPLICA AS DR TARGET:
  Primary RDS: ap-southeast-1 (Singapore)
  Read Replica: ap-south-1 (Mumbai)
  Replication: async (~seconds lag)

  ON DISASTER:
  Promote Read Replica to Primary in Mumbai:
  aws rds promote-read-replica \
    --db-instance-identifier nbis-rds-dr-replica

  Promotion time: 5–15 minutes
  Data lag: seconds (async replication)

  Tables in RDS (non-identity, acceptable RPO):
  ├── partners (relying party records)
  ├── users (internal staff accounts)
  ├── audit_reports (reporting data)
  └── enrollment_centers (registration center config)
```

---

### 16.4.4 SQS — No DR Replication Needed

```
┌──────────────────────────────────────────────────────────────┐
│              SQS DR Consideration                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  SQS messages are ephemeral pipeline state.                 │
│  Messages in queues at disaster time represent:             │
│  ├── Enrollment packets being processed (minutes old)       │
│  └── ABIS jobs in flight (seconds old)                     │
│                                                              │
│  STRATEGY: Accept loss of in-flight SQS messages.           │
│                                                              │
│  Why acceptable:                                             │
│  ├── Enrollment packets: original files are in S3           │
│  │   (replicated to DR). Re-submit from S3 on recovery.    │
│  ├── Enrollment is idempotent (same packet → same result)  │
│  └── 4-hour RTO allows time for re-submission              │
│                                                              │
│  RECOVERY ACTION:                                            │
│  On DR activation:                                          │
│  1. Identify S3 packets not yet assigned a UIN             │
│     (enrollment status = RECEIVED or IN_PROGRESS)          │
│  2. Re-publish those packet references to DR SQS queues    │
│  3. DR pipeline processes them from S3 (DR copy)           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 16.5 DNS Failover — Route 53

**Amazon Route 53** is the mechanism that redirects all traffic from the failed primary region to the DR region:

```
┌──────────────────────────────────────────────────────────────┐
│              Route 53 DNS Failover Configuration             │
└──────────────────────────────────────────────────────────────┘

DNS RECORD SETUP:
  api.nbis.gov → Route 53 Failover routing policy

  PRIMARY record:
  ├── Target: api-gateway.ap-southeast-1.amazonaws.com
  ├── Health check: every 10 seconds
  └── Failover: PRIMARY

  SECONDARY record:
  ├── Target: api-gateway.ap-south-1.amazonaws.com
  ├── No health check (always serve if primary fails)
  └── Failover: SECONDARY

HEALTH CHECK:
  Route 53 health check:
  ├── Endpoint: https://api.nbis.gov/health
  ├── Protocol: HTTPS
  ├── Path: /actuator/health
  ├── Interval: 10 seconds
  ├── Failure threshold: 3 consecutive failures
  └── Action: mark primary as unhealthy → serve secondary

FAILOVER TIMELINE:
  T=0:   Primary region goes down
  T=30s: Route 53 health check fails 3 times
  T=30s: DNS TTL expires (set to 60 seconds)
  T=90s: DNS resolves to DR region for new connections
  T=90s: Citizens and relying parties hit DR region

TTL CONSIDERATIONS:
  DNS TTL must be LOW for fast failover:
  api.nbis.gov TTL: 60 seconds (not 3600!)

  Trade-off: Low TTL → more DNS queries → slightly higher cost
  Worth it: 60s TTL means failover in < 90 seconds
            vs 3600s TTL means failover in up to 1 hour

IMPORTANT:
  Existing connections (not DNS lookups) are NOT affected
  by DNS changes. Long-lived connections to primary region
  will need to reconnect (timeout) to pick up new DNS.
  → Set connection timeouts to 30 seconds for fast recovery.
```

---

## 16.6 DR Activation Runbook

When disaster strikes, every second counts. The DR activation process must be documented, practiced, and executable under pressure:

```
┌──────────────────────────────────────────────────────────────┐
│              NBIS DR Activation Runbook                      │
└──────────────────────────────────────────────────────────────┘

PHASE 1: DETECTION AND DECISION (0–15 minutes)
──────────────────────────────────────────────
Step 1: Incident declared (PagerDuty fires P1 alert)
        On-call engineer checks:
        ├── Is primary region entirely unavailable?
        ├── Is this an application bug or regional failure?
        └── AWS Service Health Dashboard → confirm region issue

Step 2: Incident Commander assigned
        (Senior engineer or Team Lead takes command)

Step 3: Decision: ACTIVATE DR?
        Criteria:
        ├── Primary region unavailable > 30 minutes
        ├── AWS ETA for restoration > RTO target (4 hours)
        └── Data integrity confirmed (no corruption detected)
        → YES → proceed to Phase 2

PHASE 2: DR REGION PREPARATION (15–30 minutes)
───────────────────────────────────────────────
Step 4: Verify DR region data freshness
        Check DynamoDB Global Table replication lag
        Check S3 CRR replication status
        Confirm last successful RDS snapshot timestamp

Step 5: Promote RDS Read Replica to Primary
        aws rds promote-read-replica \
          --db-instance-identifier nbis-rds-dr-replica
        Wait for status: available (~15 minutes)

Step 6: Scale up ECS services in DR region
        aws ecs update-service \
          --cluster nbis-dr \
          --service nbis-auth-service \
          --desired-count 6
        (previously running at minimum 2)
        Wait for tasks: healthy (~5 minutes)

Step 7: Verify DR services healthy
        Synthetic canary in DR region: all green?
        Manual smoke test: auth, eKYC, resident portal

PHASE 3: TRAFFIC CUTOVER (30–45 minutes)
─────────────────────────────────────────
Step 8: Update Route 53 health check
        Mark primary region health check as failed manually
        (if not already auto-failed)
        → DNS failover triggers immediately

Step 9: Monitor traffic shift
        CloudWatch: DR region request count rising
        CloudWatch: Primary region request count → zero
        Wait 5 minutes for DNS propagation

Step 10: Notify stakeholders
         Relying parties (banks, hospitals): email notification
         Government IT team: WhatsApp / SMS
         Citizens: status page update

PHASE 4: STABILIZATION (45–60 minutes)
───────────────────────────────────────
Step 11: Monitor DR region for 30 minutes
         ├── Error rate < 0.1%
         ├── P95 latency < 300ms
         ├── DLQ depth = 0
         └── No SQS backlogs building

Step 12: Re-process missed enrollment packets
         Query S3: packets with status RECEIVED/IN_PROGRESS
         Re-publish to DR SQS queues
         Monitor enrollment pipeline in DR

Step 13: Create incident report (initial)
         ├── Time of failure
         ├── RTO achieved (was it within 4 hours?)
         ├── RPO achieved (any data loss?)
         └── Number of citizens affected

PHASE 5: PRIMARY REGION RECOVERY (hours to days later)
───────────────────────────────────────────────────────
Step 14: Primary region restored by AWS
         Verify data integrity in primary region
         Confirm DynamoDB sync with DR region

Step 15: Plan failback
         Choose low-traffic window
         Route 53: shift 10% traffic back to primary
         Monitor: errors, latency
         Gradually shift: 10% → 50% → 100%

Step 16: Decommission DR as primary
         Scale ECS back to minimum in DR
         Mark DR RDS back as read replica
         Document lessons learned
```

---

## 16.7 Backup Strategy

Backups are the last line of defense. Even with Global Tables and CRR, explicit backups protect against logical corruption (accidental deletes, ransomware):

```
┌──────────────────────────────────────────────────────────────┐
│              NBIS Backup Strategy                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  DYNAMODB:                                                   │
│  ├── Point-in-Time Recovery (PITR): enabled                 │
│  │   Restore to any second in last 35 days                  │
│  ├── On-demand backups: weekly full backup                  │
│  │   Retained: 1 year                                       │
│  └── Export to S3: monthly export to S3 Glacier            │
│      Retained: 7 years (legal compliance)                   │
│                                                              │
│  S3 (biometric templates):                                   │
│  ├── S3 Versioning: enabled on all buckets                  │
│  │   Deleted objects → kept as versioned delete marker      │
│  │   Restore any version within retention period            │
│  ├── S3 Object Lock: COMPLIANCE mode                        │
│  │   Objects cannot be deleted for retention period        │
│  ├── Cross-Region Replication: continuous to DR            │
│  └── S3 Lifecycle: move to Glacier after 90 days           │
│                                                              │
│  RDS:                                                        │
│  ├── Automated backups: daily, 7-day retention              │
│  ├── Manual snapshots: weekly, 1-year retention             │
│  └── Cross-region copy: daily to DR region                 │
│                                                              │
│  AUDIT LOGS (CloudWatch + S3):                               │
│  ├── CloudWatch Object Lock: COMPLIANCE mode                │
│  │   Cannot be deleted by anyone (including root)          │
│  ├── S3 archive: monthly export                             │
│  ├── Retention: 7 years (regulatory minimum)               │
│  └── Cross-region replication: continuous                  │
│                                                              │
│  ENCRYPTION KEYS (KMS / CloudHSM):                          │
│  ├── CloudHSM: hardware-backed (physically redundant)       │
│  ├── KMS key material: backed up by AWS (managed)          │
│  └── Customer-managed keys: exported and stored in         │
│      government-controlled secure hardware (HSM copy)       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 16.8 Data Integrity Verification

Backups are useless if they are corrupt. Regular integrity checks are mandatory:

```
┌──────────────────────────────────────────────────────────────┐
│              Data Integrity Verification Schedule            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  WEEKLY (automated):                                         │
│  ├── DynamoDB: restore weekly backup to test environment    │
│  │   Run checksum comparison: test restore vs production   │
│  ├── S3: verify random sample of biometric template files  │
│  │   Decrypt and check file integrity (MD5/SHA-256)        │
│  └── RDS: restore latest snapshot to test DB               │
│      Run: SELECT COUNT(*) from key tables                   │
│      Compare with production count                          │
│                                                              │
│  MONTHLY (manual + automated):                               │
│  ├── Full DR test (see Section 16.9)                        │
│  ├── CloudHSM: key availability verification                │
│  └── Audit log completeness check (no gaps in sequence)    │
│                                                              │
│  AFTER ANY RESTORE:                                          │
│  ├── Run automated data quality checks                      │
│  ├── Verify record counts match expected                    │
│  ├── Spot-check specific records against known-good data    │
│  └── Confirm biometric templates are decryptable           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 16.9 DR Testing — Practice Makes Perfect

A DR plan that has never been tested is not a DR plan — it is a hypothesis:

```
┌──────────────────────────────────────────────────────────────┐
│              NBIS DR Testing Schedule                        │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  TEST 1: Tabletop Exercise (monthly)                         │
│  ├── Team walks through DR runbook verbally                 │
│  ├── Identify gaps, ambiguities, outdated steps            │
│  ├── Update runbook with findings                           │
│  └── Duration: 2 hours, no system impact                   │
│                                                              │
│  TEST 2: Component Recovery Test (quarterly)                 │
│  ├── Test individual recovery actions in staging            │
│  │   ├── RDS read replica promotion                        │
│  │   ├── ECS scale-up in DR region                        │
│  │   └── Route 53 DNS failover                            │
│  ├── Measure actual RTO for each step                      │
│  └── Duration: 4 hours, staging only                       │
│                                                              │
│  TEST 3: Full DR Test (annually)                             │
│  ├── Simulate complete regional failure                     │
│  ├── Activate DR region in production (low-traffic window) │
│  ├── Shift 100% traffic to DR region                       │
│  ├── Run for 2 hours with live traffic                     │
│  ├── Measure: RTO achieved, RPO achieved, errors           │
│  ├── Failback to primary region                            │
│  └── Full incident report published                        │
│                                                              │
│  PASS CRITERIA for full DR test:                             │
│  ├── RTO achieved: < 4 hours (verification API)            │
│  ├── RPO achieved: 0 data loss (identity records)          │
│  ├── Error rate during switchover: < 1%                    │
│  └── All relying parties reconnected within RTO            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 16.10 Ransomware Protection

A special DR scenario deserves its own section — ransomware attacks that encrypt or delete data:

```
┌──────────────────────────────────────────────────────────────┐
│              Ransomware Protection Strategy                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PREVENTION:                                                 │
│  ├── No IAM user with full S3 delete/overwrite rights       │
│  ├── S3 Object Lock: COMPLIANCE mode — not even root        │
│  │   can delete within retention period                    │
│  ├── MFA Delete: required for S3 bucket deletion           │
│  ├── SCPs (Service Control Policies): org-level deny       │
│  │   Deny: s3:DeleteBucket on critical buckets             │
│  │   Deny: dynamodb:DeleteTable on critical tables         │
│  └── GuardDuty: detects unusual bulk delete patterns       │
│                                                              │
│  DETECTION:                                                  │
│  ├── CloudTrail: all API calls logged                       │
│  ├── GuardDuty: UnauthorizedAccess:S3/MaliciousIPCaller    │
│  ├── CloudWatch alarm: S3 delete rate > baseline           │
│  └── DynamoDB: ConsumedWriteCapacityUnits spike alarm      │
│                                                              │
│  RECOVERY:                                                   │
│  ├── DynamoDB PITR: restore to point before attack         │
│  │   aws dynamodb restore-table-to-point-in-time           │
│  ├── S3 versioning: restore previous versions              │
│  ├── Object Lock: encrypted objects readable from lock     │
│  │   copy (attacker cannot overwrite locked objects)       │
│  └── DR region: unaffected by primary region attack        │
│      (separate AWS account recommended for DR)             │
│                                                              │
│  SEPARATE AWS ACCOUNT FOR DR:                                │
│  Primary region: AWS Account A                              │
│  DR region:      AWS Account B                              │
│                                                              │
│  Why separate accounts?                                      │
│  If attacker compromises Account A credentials:            │
│  → They cannot access Account B resources                  │
│  → DR data in Account B is safe                            │
│  → Recover using Account B as new primary                  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 16.11 Business Continuity During DR

When DR is active, some capabilities may be limited. Citizens and relying parties must know what works and what does not:

```
┌──────────────────────────────────────────────────────────────┐
│              Business Continuity During DR                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  FULLY AVAILABLE in DR region:                               │
│  ✅ Biometric verification (1:1 match)                       │
│  ✅ OTP authentication                                       │
│  ✅ eKYC for relying parties                                 │
│  ✅ Resident portal (view profile, auth history)            │
│  ✅ Smart card verification (offline — card itself)         │
│  ✅ Border crossing (ePassport — ICAO, separate system)     │
│                                                              │
│  DEGRADED in DR region:                                      │
│  ⚠️  New enrollments: accepted, queued (slower processing)  │
│  ⚠️  Biometric refresh: accepted, queued                    │
│  ⚠️  Address / name updates: accepted, queued              │
│                                                              │
│  NOT AVAILABLE in DR region:                                 │
│  ❌ Admin console (not critical during DR)                  │
│  ❌ Reporting dashboards (data may be 24h stale)           │
│  ❌ New relying party onboarding (postponed until recovery) │
│                                                              │
│  CITIZEN COMMUNICATION:                                      │
│  ├── Status page: status.nbis.gov (hosted on Cloudflare,   │
│  │   independent of AWS — always available)                 │
│  ├── SMS alert: sent to all enrolled citizens              │
│  └── Government press release for extended outages         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 16.12 Cost of DR

```
┌──────────────────────────────────────────────────────────────┐
│              NBIS DR Cost Estimation                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ONGOING DR COSTS (warm standby):                           │
│                                                              │
│  ECS tasks (minimum, DR):     $200/month                    │
│  (2 Auth tasks + 1 each of other critical services)        │
│                                                              │
│  DynamoDB Global Tables:      $150/month                    │
│  (replicated write capacity)                               │
│                                                              │
│  S3 Cross-Region Replication: $100/month                    │
│  (data transfer + storage in DR region)                    │
│                                                              │
│  RDS Read Replica:            $300/month                    │
│  (db.r6g.large in DR region)                               │
│                                                              │
│  Route 53 Health Checks:      $3/month                     │
│                                                              │
│  TOTAL DR ONGOING COST:       ~$753/month                   │
│                                                              │
│  COST OF NOT HAVING DR:                                      │
│  1 day of NBIS outage:                                      │
│  ├── Citizens cannot access banking: millions in blocked   │
│  │   transactions                                           │
│  ├── Border delays: international incident                 │
│  ├── Government penalty clauses: contractual liability     │
│  └── Reputational damage: priceless (in the bad way)      │
│                                                              │
│  $753/month is not a cost. It is an insurance premium.     │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 16.13 Complete NBIS DR Architecture

```
┌──────────────────────────────────────────────────────────────┐
│              Complete NBIS DR Architecture                   │
└──────────────────────────────────────────────────────────────┘

AWS Account A (Primary)          AWS Account B (DR)
ap-southeast-1 (Singapore)       ap-south-1 (Mumbai)
──────────────────────────       ──────────────────────────────

Citizens + Relying Parties
         │
    Route 53 DNS
    api.nbis.gov
         │
    ┌────┴────────────────────────────────┐
    │ HEALTH CHECK PASSES (normal)        │
    │ PRIMARY                             │
    ▼                                     ▼ HEALTH CHECK FAILS
API Gateway (primary)              API Gateway (DR)
ECS Services (full)                ECS Services (minimum)
    │                                     │
    ▼                                     ▼
┌──────────────────────┐     ┌────────────────────────────┐
│ DynamoDB (primary)   │────►│ DynamoDB Global Table (DR) │
│ (read + write)       │     │ (< 1 second lag)           │
└──────────────────────┘     └────────────────────────────┘
         │                                │
┌──────────────────────┐     ┌────────────────────────────┐
│ S3 (primary)         │────►│ S3 DR (CRR)                │
│ Biometric templates  │     │ seconds-minutes lag        │
└──────────────────────┘     └────────────────────────────┘
         │                                │
┌──────────────────────┐     ┌────────────────────────────┐
│ RDS Primary          │────►│ RDS Read Replica (DR)      │
│ (read + write)       │     │ async replication          │
└──────────────────────┘     └────────────────────────────┘
         │
┌──────────────────────┐
│ SQS Queues           │     (No replication — re-submit
│ (enrollment pipeline)│      from S3 on DR activation)
└──────────────────────┘
         │
┌──────────────────────┐     ┌────────────────────────────┐
│ CloudWatch Logs      │────►│ CloudWatch Logs (DR)       │
│ (audit logs)         │     │ (S3 CRR backup)            │
└──────────────────────┘     └────────────────────────────┘

On Disaster:
  Route 53 detects primary unhealthy
  DNS → DR region
  DR ECS auto-scales to full
  RDS read replica promoted
  ElastiCache starts fresh
  Enrollment re-processed from S3
  Full service in DR: 15–60 minutes
```

---

## 16.14 MOSIP Reference

```
┌──────────────────────────────────────────────────────────────┐
│              MOSIP Disaster Recovery Reference               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  MOSIP does not prescribe a specific DR architecture —      │
│  it is a platform, and DR is the deploying nation's         │
│  responsibility.                                             │
│                                                              │
│  However, MOSIP's architecture supports DR:                 │
│                                                              │
│  ├── ID Repository: externally backed (any database)       │
│  │   DynamoDB Global Tables replicate it automatically     │
│  │                                                          │
│  ├── Biometric templates: stored separately from records   │
│  │   S3 CRR replicates them to DR region                   │
│  │                                                          │
│  ├── MOSIP modules are stateless where possible            │
│  │   Container images are in ECR (replicated to DR region) │
│  │                                                          │
│  └── Kubernetes deployments can be re-applied to DR        │
│      cluster from Helm charts (GitOps — config in Git)     │
│                                                              │
│  Key MOSIP DR consideration:                                │
│  ABIS vendor must also have DR capability.                 │
│  ABIS deduplication templates must be replicated to DR.    │
│  If ABIS has no DR → deduplication stops in disaster.      │
│  → Verify ABIS vendor SLA and DR plan before signing.      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 16.15 Key Terms

| Term | Definition |
|------|-----------|
| **Disaster Recovery (DR)** | Ability to recover from catastrophic failure affecting entire region |
| **RTO** | Recovery Time Objective — maximum acceptable downtime after disaster |
| **RPO** | Recovery Point Objective — maximum acceptable data loss in time |
| **Warm Standby** | DR strategy with minimal infrastructure running in DR region, scaled on disaster |
| **Pilot Light** | DR strategy with only core data replicated, no compute running |
| **Active/Active** | DR strategy with full capacity in both regions simultaneously |
| **DynamoDB Global Tables** | Multi-region active replication with < 1 second lag |
| **S3 Cross-Region Replication** | Automatic copying of S3 objects to a bucket in another region |
| **RDS Read Replica** | Asynchronous replica of RDS for DR and reporting purposes |
| **PITR** | Point-in-Time Recovery — restore database to any second within retention period |
| **Route 53 Failover** | DNS-based traffic redirection from primary to DR region |
| **DNS TTL** | How long DNS clients cache an answer — low TTL enables faster failover |
| **Object Lock** | S3 feature preventing object deletion within retention period (WORM) |
| **MFA Delete** | S3 protection requiring MFA to delete versioned objects |
| **SCP** | Service Control Policy — AWS Organizations policy preventing dangerous API calls |
| **Failback** | Returning production traffic to the restored primary region after DR |
| **Chaos engineering** | Deliberately injecting failures to test resilience |
| **GitOps** | Storing infrastructure configuration in Git for reproducible deployments |
| **Tabletop exercise** | Verbal walkthrough of DR procedure without touching systems |

---

## 16.16 Key Takeaways

- **DR is not the same as HA** — HA handles AZ failures within a region. DR handles complete regional failures. Both are required for a national identity system.
- **Define RTO and RPO first** — everything in DR design follows from these numbers. NBIS targets 4-hour RTO and zero RPO for identity records.
- **DynamoDB Global Tables solve the hardest problem** — identity records replicated across regions with < 1 second lag means RPO = 0 for the CIDR without complex custom replication code.
- **S3 delete marker replication must be DISABLED** — accidental deletes in the primary region must not propagate to the DR copy. Your DR backup must be immune to primary region mistakes.
- **Test DR every year with live traffic** — a DR plan that has never moved real traffic to the DR region is untested. The annual full DR test is non-negotiable for a government system.
- **Separate AWS accounts for primary and DR** — if ransomware compromises Account A credentials, DR data in Account B remains safe. Shared accounts make DR data vulnerable to the same attack.
- **Route 53 TTL must be low (60 seconds)** — a 1-hour TTL means 1 hour of DNS-based delay after disaster. Pay the extra Route 53 query cost for 60-second TTL.
- **The ABIS vendor needs a DR plan too** — deduplication during enrollment depends on the external ABIS. If the ABIS vendor has no DR, new enrollment deduplication stops during disaster. Verify their SLA before signing.

---

## 16.17 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 17 | Enrollment Workflow — deep dive into the registration center pipeline |
| Chapter 73 | Monitoring — CloudWatch dashboards, X-Ray tracing, synthetic canaries |
| Chapter 96 | Data Retention — legal requirements, lifecycle policies, archival |

---

*Chapter 16 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*
