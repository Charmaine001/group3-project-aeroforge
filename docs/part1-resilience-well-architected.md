# AeroForge Part 1: Resilience, Disaster Recovery & AWS Well-Architected Review

**Document Identifier:** `docs/part1-resilience-well-architected.md`  
**Course:** MIS 612 — Fundamentals of Cloud Computing (Fall 2026) — Project 2, Part 1  
**Subject:** Enterprise Availability, Tiered RTO/RPO, Sovereignty-Constrained DR, and Six-Pillar Well-Architected Review  

---

## 1. Availability & Failure Domains

AeroForge’s architecture is engineered to survive localized hardware, network, and data-center outages without human intervention while providing a bounded, cost-effective recovery path for regional disasters. The platform isolates fault boundaries across four distinct failure domains:

### 1.1 Domain 1: Plant Shop-Floor & WAN Edge (`WIC`, `OKC`, `QRO`)
* **Fault Scope:** Loss of an industrial edge gateway, plant switch failure, or severance of a telecom carrier circuit.
* **High Availability Pattern:** Each plant deploys an active-standby pair of industrial edge servers running **AWS IoT Greengrass v2** across dual-homed plant VLANs. Primary WAN traffic traverses dedicated **AWS Direct Connect** circuits (1 Gbps at US plants; 500 Mbps Partner Hosted at Querétaro) with automated BGP route failover to an encrypted **AWS Site-to-Site VPN** over an independent internet service provider[cite: 2].
* **Target Availability:** **99.9%** end-to-end ingestion availability[cite: 2].

### 1.2 Domain 2: Single Availability Zone (AZ) Failure (`us-east-1` or `mx-central-1`)
* **Fault Scope:** Loss of power, cooling, or networking within a single physical AWS data center cluster.
* **High Availability Pattern:** All production workloads operate active-active across **3 Availability Zones** (`us-east-1a`, `us-east-1b`, `us-east-1c` in N. Virginia; `mx-central-1a`, `mx-central-1b`, `mx-central-1c` in Querétaro).
  * **Stateless & Managed Services:** Amazon S3, Amazon Kinesis Data Streams, Amazon Data Firehose, AWS Glue, Amazon Athena, Amazon DynamoDB, and AWS Step Functions automatically distribute storage and compute across $\ge 3\text{ AZs}$ within the region[cite: 1, 2].
  * **Stateful & VPC-Bound Services:** **Amazon Managed Service for Apache Flink** runs with multi-AZ operator parallelism; **Amazon Redshift Serverless** provisions compute and managed storage snapshots across multiple AZs; **AWS Transit Gateway** and **Interface VPC Endpoints (PrivateLink)** deploy elastic network interfaces (ENIs) in every active AZ subnet[cite: 2].
* **Impact of Single-AZ Loss:** **Zero data loss ($\text{RPO} = 0$) and zero downtime ($\text{RTO} \approx 0$)**[cite: 1, 2]. Auto Scaling groups and managed service routers shift load to surviving AZs automatically[cite: 1, 2].

### 1.3 Domain 3: Primary Region Failure (`us-east-1` Outage)
* **Fault Scope:** Wide-area control-plane failure or catastrophic regional disruption in US East (N. Virginia)[cite: 1, 2].
* **Disaster Recovery Pattern:** **Pilot Light DR** in **US West (Oregon, `us-west-2`)**[cite: 2]. Core storage and catalog metadata are continuously replicated to `us-west-2`, while compute-intensive serving and orchestration layers (Redshift Serverless workgroups, Glue ETL schedules, Flink applications) remain provisioned as dormant Infrastructure-as-Code (Terraform/CloudFormation) configurations scaled to zero until a failover is declared[cite: 2].

### 1.4 Domain 4: Logical, Data Corruption, or Ransomware Failure
* **Fault Scope:** Buggy Spark ETL deployment, accidental table drop, or compromised IAM principal attempting to encrypt or delete lakehouse objects[cite: 1, 2].
* **Mitigation Pattern:** **S3 Versioning**, **Apache Iceberg Time-Travel Snapshots**, and **S3 Object Lock (Compliance Mode)** on SOX financial audit buckets, combined with **AWS Backup Vault Lock** stored in a dedicated, restricted recovery account[cite: 1, 2].

---

## 2. Recovery Point Objectives (RPO) & Recovery Time Objectives (RTO)

The Part 1 specification explicitly defines availability, telemetry RPO/RTO, and warehouse RPO/RTO targets[cite: 2]. For tiers where the project instructions do not specify numerical targets, reasonable enterprise targets are established below and labeled as **`[DESIGN ASSUMPTION — REQUIRES APPROVAL]`**:

| Architecture Tier / Workload | Primary Service(s) | Availability Target | Recovery Point Objective (RPO) | Recovery Time Objective (RTO) | Specification Source / Status | DR Strategy & Mechanism |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **1. Plant Edge & Streaming Ingestion** | IoT Greengrass, IoT Core, Kinesis Data Streams | **99.9%**[cite: 2] | **15 minutes**[cite: 2] | **4 hours**[cite: 2] | **Explicit Project NFR**[cite: 2] | Local Greengrass Stream Manager disk spooling (72-hr buffer) + S3 Cross-Region Replication (CRR) with **S3 Replication Time Control (RTC)** (15-min SLA) to `us-west-2`. |
| **2. Real-Time Anomaly Alerting (<60s Path)** | Managed Service for Apache Flink, Lambda, SNS | **99.9%** (`[DESIGN ASSUMPTION — REQUIRES APPROVAL]`) | **5 minutes** (`[DESIGN ASSUMPTION — REQUIRES APPROVAL]`) | **1 hour** (`[DESIGN ASSUMPTION — REQUIRES APPROVAL]`) | **Design Assumption** | Flink checkpoints saved to S3 every 60s and replicated via S3 RTC; standby Flink application started via Terraform/CLI in `us-west-2` on failover. |
| **3. S3 Lakehouse (Bronze / Silver / Gold Iceberg)** | Amazon S3, S3 Tables (Apache Iceberg), Glue Catalog | **99.99%** (Multi-AZ) / **99.9%** (Cross-Region) | **15 minutes** (Bronze/Silver/Gold) | **2 hours** (`[DESIGN ASSUMPTION — REQUIRES APPROVAL]`) | **Derived from Telemetry NFR**[cite: 2] | S3 CRR with RTC guarantees 99.99% of new objects replicate to `us-west-2` within 15 minutes; Glue Data Catalog metadata replicated via scheduled Lambda/AWS Backup. |
| **4. Enterprise Data Warehouse & BI** | Amazon Redshift Serverless, QuickSight | **99.9%** (`[DESIGN ASSUMPTION — REQUIRES APPROVAL]`) | **24 hours**[cite: 2] | **8 hours**[cite: 2] | **Explicit Project NFR**[cite: 2] | Redshift Serverless mounts replicated S3 Gold Iceberg tables in `us-west-2` and refreshes derived Materialized Views; native Redshift cross-region snapshots replicate every 24h. |
| **5. Batch / CDC Pipelines (MES, CMMS, ERP)** | DMS, AppFlow, Transfer Family, Step Functions, Glue | **99.5%** (`[DESIGN ASSUMPTION — REQUIRES APPROVAL]`) | **24 hours** (`[DESIGN ASSUMPTION — REQUIRES APPROVAL]`) | **8 hours** (`[DESIGN ASSUMPTION — REQUIRES APPROVAL]`) | **Aligned to Finance 07:00 CT SLA**[cite: 2] | Idempotent nightly pipelines re-deployed in `us-west-2` via GitHub Actions; source ERP/MES systems re-send or resume CDC from last committed checkpoint. |
| **6. ML Scoring & Bedrock AI Agents** | SageMaker AI, Bedrock, Bedrock AgentCore, OpenSearch | **99.5%** (`[DESIGN ASSUMPTION — REQUIRES APPROVAL]`) | **24 hours** (`[DESIGN ASSUMPTION — REQUIRES APPROVAL]`) | **8 hours** (`[DESIGN ASSUMPTION — REQUIRES APPROVAL]`) | **Design Assumption** | Model artifacts replicated in S3 Model Registry; Bedrock Agent definitions and SOP Knowledge Base indices rebuilt/synced in `us-west-2` via Terraform. |
| **7. Mexico Sovereign HR & Payroll (`mx-central-1`)** | S3, RDS/Aurora, KMS (`mx-central-1`) | **99.95%** (Multi-AZ in-country) | **0 min (AZ loss) / 1 hr (Snapshot)** (`[DESIGN ASSUMPTION — REQUIRES APPROVAL]`) | **15 min (AZ) / 4 hr (Restore)** (`[DESIGN ASSUMPTION — REQUIRES APPROVAL]`) | **Design Assumption** | **In-Region Multi-AZ + Immutable Local Vault Only** (Cross-border replication prohibited; see Section 4). |

---

## 3. Backup, Restore, Replication & Recovery Patterns

### 3.1 Why Pilot Light Was Chosen Over Warm Standby or Multi-Site Active-Active
* **Budget Alignment:** The pilot AWS run-rate must stay under **$\$18,000/\text{month}$**[cite: 2]. Running a Warm Standby or Active-Active multi-region deployment in `us-west-2` would duplicate 24/7 VPC Interface Endpoints, NAT Gateways, Kinesis pre-warmed shards, OpenSearch Serverless OCUs, and Redshift Serverless base capacity—adding $\$3,500\text{–}\$6,000/\text{month}$ in idle DR overhead.
* **SLA Fit:** Because the non-functional requirements permit a **4-hour RTO for telemetry** and an **8-hour RTO for the warehouse**[cite: 2], AeroForge does not require instantaneous sub-second regional failover. Pilot Light satisfies the 15-minute RPO through continuous **S3 Replication Time Control (RTC)** while incurring near-zero standby compute cost in `us-west-2` (roughly $\$250\text{–}\$400/\text{month}$ for replicated S3 storage, KMS keys, and snapshot storage).

### 3.2 Streaming Backlogs, Disconnected Operation & Replay Mechanics
Manufacturing telemetry cannot simply be "re-requested" from a CNC spindle if a network link drops[cite: 2]. The architecture prevents data loss across three failure scenarios:
1. **WAN Link Outage (Plant to AWS):**
   * **AWS IoT Greengrass v2 Stream Manager** writes incoming OPC UA/MQTT messages to an encrypted local NVMe ring buffer at each plant (`WIC`, `OKC`, `QRO`) sized for **72 hours** of peak local production.
   * When connectivity is restored, Stream Manager drains the backlog in chronological order using token-bucket rate limiting so live real-time alarms are prioritized while historical backlog catches up without saturating the plant WAN.
2. **Downstream Consumer Failure (Flink or Firehose Outage):**
   * **Amazon Kinesis Data Streams** is configured with a **7-day retention period** (extended from the default 24 hours). If a Glue schema deployment or Flink operator crash stalls downstream processing during a **20,000 msg/s** burst, Kinesis buffers the raw stream durably across 3 AZs[cite: 2].
   * Once the consumer is patched, Flink resumes from its last committed **Amazon S3 checkpoint**, and Firehose retries delivery to the S3 Bronze zone.
3. **Corrupted Silver/Gold ETL Logic (Historical Replay):**
   * Because the **S3 Bronze Zone** stores immutable, append-only raw payloads (`JSONL.gz` and `CSV`) partitioned by `ingestion_date`[cite: 1, 2], and because Silver/Gold Glue Spark jobs enforce idempotent `MERGE INTO` / `INSERT OVERWRITE` semantics on `event_id` and `journal_line_id`[cite: 1, 2], engineers can replay any historical date range from Bronze to rebuild Silver and Gold from scratch[cite: 2].

### 3.3 Recovery of the S3 Gold Finance Layer & Redshift Serving Layer
Because **S3 Gold (Apache Iceberg)** is the authoritative Finance System of Record and **Amazon Redshift Serverless** is a stateless/derived serving layer, recovery follows a strict two-step sequence:
1. **Point-in-Time Recovery (PITR) from Logical Corruption:**
   * If a bad nightly ERP load corrupts `fact_financial_gl_variance`, no S3 file restore is needed. The platform administrator executes an Apache Iceberg snapshot rollback in Athena/Spark (`CALL catalog.system.rollback_to_snapshot('gold.fact_financial_gl_variance', <last_valid_snapshot_id>)`), restoring the authoritative table in **under 60 seconds** (`RPO = 0`, `RTO < 5 min`).
2. **Regional Failover to `us-west-2` (8-Hour Warehouse RTO):**
   * **Step 1 (`T+0 to T+30 min`):** S3 CRR with RTC has already replicated all Bronze, Silver, and Gold Parquet data files and Iceberg metadata manifests to `af-prod-gold-us-west-2` within the 15-minute RPO window.
   * **Step 2 (`T+30 to T+90 min`):** Automated Terraform pipeline activates the dormant **Redshift Serverless** workgroup in `us-west-2` and runs the Glue Catalog sync script to register the latest Iceberg `metadata.json` pointers in the `us-west-2` Glue Data Catalog.
   * **Step 3 (`T+90 to T+180 min`):** Redshift mounts the `gold_iceberg_ext` external schema in `us-west-2` and executes `REFRESH MATERIALIZED VIEW` for the 3 derived rollup views (`mv_daily_plant_oee_summary`, `mv_monthly_pnl_variance_bridge`, `mv_titanium_ppv_exposure`).
   * **Step 4 (`T+180 to T+240 min`):** Amazon Route 53 and QuickSight data source endpoints repoint to the `us-west-2` Redshift workgroup and Athena workgroup, restoring full controller and executive access well inside the **8-hour RTO**[cite: 2].

---

## 4. Regional Failure & Data-Residency Implications (Mexico & TALON)

> **CRITICAL GOVERNANCE RULE:** Cross-region disaster recovery replication **must never violate national data sovereignty or defense export-control boundaries**[cite: 2]. Automated S3 Cross-Region Replication (CRR) and AWS Backup cross-region copy rules must be explicitly scoped by data classification before enabling replication.

### 4.1 Implications for Querétaro (QRO) Mexico Payroll & HR Data
* **Why Cross-Border Replication is Prohibited:** Under the project's compliance requirements, Mexico payroll and HR data must remain in-country[cite: 2]. Configuring standard cross-region replication from **AWS Mexico (Central) (`mx-central-1`)** to a US DR region (`us-east-1` or `us-west-2`) would physically export Mexican employee PII and payroll records across international borders, violating corporate residency mandates and Mexican federal data protection law (LFPDPPP)[cite: 2].
* **Architectural Consequence:** Since AWS currently operates only **one** region inside Mexico (`mx-central-1`), geographic cross-region replication *within* Mexico is physically impossible on AWS alone.
* **Approved In-Country Resilience Pattern:**
  1. **3-AZ High Availability:** All QRO HR and payroll databases (`Amazon Aurora PostgreSQL` / `RDS`) and S3 buckets (`af-qro-hr-sovereign-mx-central-1`) run across all **3 isolated Availability Zones** in `mx-central-1` (`mx-central-1a`, `1b`, `1c`).
  2. **Logical Air-Gapped In-Region Vault:** To protect against logical corruption or account compromise without leaving Mexico, **AWS Backup** copies encrypted daily snapshots to a separate, dedicated **In-Country Recovery AWS Account** still pinned to `mx-central-1` using **AWS Backup Vault Lock (Compliance Mode)** and a separate KMS CMK.
  3. **Encrypted On-Premises Cold Escrow (Optional Last Resort):** If physical catastrophic survival outside the `mx-central-1` metropolitan zone is mandated by legal counsel, encrypted backups are exported over private Direct Connect to an immutable storage appliance inside AeroForge's physical Querétaro plant data room.
  4. **SCP Guardrail:** `SCP-05` explicitly denies `s3:PutReplicationConfiguration` and `backup:CopyIntoBackupVault` from `mx-central-1` to any destination region `!= mx-central-1`.

### 4.2 Implications for TALON Export-Controlled Data (ITAR / EAR)
* **Conditions for Permissible Cross-Region Replication:** TALON military trainer technical data must stay in **U.S. regions** and be accessed **only by authorized U.S. persons**[cite: 2]. Therefore, replicating TALON data from `us-east-1` (N. Virginia) to `us-west-2` (Oregon) **is legally permissible ONLY IF the destination DR region enforces identical cryptographic, network, and U.S.-person identity boundaries**[cite: 2].
* **Risks of Unconstrained Replication:** If TALON objects were replicated into a shared DR bucket in `us-west-2` encrypted with a default S3 key (`SSE-S3`) or a general-purpose KMS key, or if `us-west-2` Lake Formation permissions were not pre-synchronized, non-entitled administrators or automated DR scripts could inadvertently gain read access during a failover drill.
* **Approved TALON DR Replication Controls:**
  1. **US-to-US Boundary Enforcement:** `SCP-01` and `SCP-04` restrict all TALON accounts and buckets strictly to `us-east-1` and `us-west-2`[cite: 2]. Replication to `mx-central-1` or any non-US region is blocked at the AWS Organizations root.
  2. **Dedicated DR KMS CMK (`alias/af-talon-export-cmk-dr`):** The destination S3 replication bucket in `us-west-2` re-encrypts incoming TALON objects using a dedicated `us-west-2` KMS Customer Managed Key whose Key Policy enforces the exact same ABAC condition (`aws:PrincipalTag/USPerson = "true"` and `aws:PrincipalTag/ProgramAccess = "TALON"`).
  3. **Replication IAM Role Scoping:** The S3 CRR service role is a system-level cryptographic transport role covered by an explicit `aws:PrincipalArn` exception in the KMS key policy, ensuring no human operator can assume the replication role interactively.
  4. **Pre-Synchronized Lake Formation Grants:** Before any failover is declared, Lake Formation Data Cells Filters (`program_code != 'TALON'` for general users) are continuously mirrored to the `us-west-2` Glue Data Catalog via Infrastructure-as-Code (`Terraform`).

---

## 5. Recovery Testing, Monitoring, Alerting & Operational Ownership

### 5.1 Monitoring, Alerting & Three "2:00 AM Wake-Up" Alarms
In response to Concept 12 from the Class Kickoff (*"Which three alarms would wake you up at 2 a.m.?"*)[cite: 1], AeroForge separates informational tickets from critical pager alerts routed via **Amazon CloudWatch Alarms $\rightarrow$ Amazon SNS $\rightarrow$ PagerDuty / Teams On-Call**:

| Priority | Alarm Name | Metric & Threshold | Business & Operational Impact | Automated & Human Response |
| :--- | :--- | :--- | :--- | :--- |
| **P1 (Wake-Up #1)** | `PlantEdgeToKinesisHeartbeatLost` | `IncomingRecords == 0` for any active plant OR `Kinesis MillisBehindLatest > 60,000 ms` for $\ge 3\text{ min}$ | Real-time $<60\text{s}$ spindle anomaly detection is blind[cite: 2]; imminent risk of unmonitored $50\text{K}$ machine crash[cite: 2]. | Greengrass fails over to backup VPN; on-call Ingestion Engineer paged immediately. |
| **P1 (Wake-Up #2)** | `S3RTCReplicationLagBreach` | `S3 OperationsPendingReplication` or `ReplicationLatency > 900 s (15 min)` | Primary-to-DR RPO SLA ($15\text{ min}$) is breached[cite: 2]; regional outage would cause permanent telemetry/finance data loss. | On-call Cloud Platform Engineer inspects S3 CRR IAM/KMS permissions and regional service health. |
| **P1 (Wake-Up #3)** | `NightlyFinanceGoldSLABreachRisk` | `StepFunctions ExecutionFailed` on ERP-to-Gold DAG at `04:30 CT` (after 3 automated retries) | Finance Gold tables will miss the mandatory **07:00 CT daily controller deadline**[cite: 2]. | Step Functions `Catch` block triggers SNS page to Data Engineering on-call with exact Glue job error log link. |
| **P2 (Security Page)** | `UnauthorizedTalonOrSOXAccessAttempt` | `CloudTrail AccessDenied` on `alias/af-talon-export-cmk` or `ObjectLock` modification attempt | Potential security breach, compromised credential, or ITAR/SOX policy violation[cite: 2]. | EventBridge triggers Lambda to revoke active IAM session tokens and pages SecOps immediately. |

### 5.2 Recovery Testing Schedule & Operational Ownership

| DR Drill / Test Type | Frequency | Scope & Procedure | Primary Owner | Success Criteria |
| :--- | :--- | :--- | :--- | :--- |
| **1. Iceberg Snapshot Rollback Drill** | Monthly | Simulate a corrupted ERP GL load in staging/prod; execute Iceberg snapshot rollback and verify Redshift MV consistency. | Data Engineer (Processing & Warehouse)[cite: 2] | Authoritative Gold table restored in $<5\text{ min}$ with zero row loss. |
| **2. Plant WAN Disconnect & Replay Test** | Quarterly | Block primary Direct Connect interface on a test gateway; verify VPN failover and Greengrass disk buffer drain upon reconnect. | Data Engineer (Ingestion)[cite: 2] | Zero dropped MQTT sequence numbers; `<60s` alerting maintained via VPN. |
| **3. Regional Pilot-Light Failover GameDay** | Semi-Annually | Spin up `us-west-2` Redshift Serverless, Flink, and Step Functions via Terraform against replicated S3 Gold data; run synthetic controller & TALON access tests. | Team Lead / Solution Architect & SecOps[cite: 2] | Telemetry restored $<4\text{h}$, Warehouse restored $<8\text{h}$, and `non-USPerson` blocked from TALON in DR[cite: 2]. |

---

## 6. AWS Well-Architected Framework Six-Pillar Review (One-Page Executive Summary)

Per Part 1 Deliverable 10 and Module CF 9, the architecture is evaluated across all six AWS Well-Architected pillars[cite: 2]:

| Pillar | AeroForge Architectural Implementation | Key Design Strengths |
| :--- | :--- | :--- |
| **1. Operational Excellence** | • All infrastructure, Glue jobs, Lake Formation tags, and DR stacks defined as code (**Terraform** + **GitHub Actions** CI/CD)[cite: 2].<br>• **AWS Step Functions** orchestrates nightly batch/CDC pipelines with automated `Retry` (exponential backoff) and `Catch` quarantine routing[cite: 1, 2].<br>• End-to-end observability via **CloudWatch**, **X-Ray**, and **Glue Data Quality** scorecards published directly on executive dashboards[cite: 2]. | Eliminates manual console drift; isolates bad records into S3 Quarantine without halting the 07:00 CT finance pipeline[cite: 1, 2]. |
| **2. Security** | • **8-Account AWS Control Tower** isolation with preventive **SCPs**[cite: 2].<br>• Federated **IAM Identity Center** SSO + MFA; **Lake Formation** row/column ABAC filters (`TALON`)[cite: 2].<br>• 4 domain-isolated **KMS CMKs** with ABAC key policies; **S3 Object Lock (Compliance Mode)** for SOX audit trails[cite: 2].<br>• **Bedrock AgentCore** session identity passthrough and **Return of Control (RoC)** human approval for CMMS writes[cite: 2]. | Enforces defense-in-depth across network, identity, storage, and AI agent tool execution[cite: 2]. |
| **3. Reliability** | • Loose coupling between shop floor and cloud via **Greengrass Stream Manager** (72h local spool) and **Kinesis Data Streams** (7-day replay buffer)[cite: 2].<br>• Active-active **3-AZ deployment** in `us-east-1` and `mx-central-1`[cite: 2].<br>• **Pilot Light DR** in `us-west-2` backed by **S3 CRR with RTC** (15-min RPO) and idempotent Spark/Iceberg ACID transactions[cite: 1, 2]. | Removes single points of failure[cite: 2]; absorbs $20,000\text{ msg/s}$ bursts (`WarmThroughput = 25 MB/s`) without dropping events[cite: 2]. |
| **4. Performance Efficiency** | • Purpose-built engines: **Apache Flink** for sub-60s stateful windowing; **Glue Spark** for distributed batch ETL; **Athena** for ad-hoc lake exploration; **Redshift Serverless** for sub-second star-schema BI[cite: 2].<br>• **Apache Iceberg** in S3 Gold with hidden partitioning (`plant_code`, `date`), Z-order clustering, and Snappy Parquet compression ($\sim 4\times$ size reduction)[cite: 1, 2]. | Partition pruning and columnar projection reduce bytes scanned per query by $>90\%$ while sustaining low-latency BI[cite: 1, 2]. |
| **5. Cost Optimization** | • Serverless pay-for-use compute (**Redshift Serverless** with 15-min auto-pause, **Glue Flex** execution for non-urgent jobs, **Lambda** for daily APIs)[cite: 1, 2].<br>• **S3 Lifecycle policies** transition raw telemetry to Glacier Instant Retrieval after 90 days and Deep Archive after 365 days[cite: 1, 2].<br>• Mandatory FinOps cost-allocation tags (`Plant`, `Program`, `CostCenter`) and **AWS Budgets** alerts at 50%/80%/100%[cite: 1, 2]. | Keeps the 3-plant pilot comfortably below the **$\$18,000/\text{month}$** ceiling[cite: 2] and prevents idle cluster waste[cite: 1]. |
| **6. Sustainability** | • Serverless auto-pausing (Redshift, Athena, Lambda) eliminates 24/7 idle server power draw[cite: 2].<br>• Columnar Parquet compression + Iceberg partition pruning minimize redundant CPU cycles and disk I/O per analytical scan[cite: 1, 2].<br>• **AWS Graviton3/Graviton4** ARM processors selected for Lambda, Glue, and Flink workers to maximize performance-per-watt. | Reduces both cloud carbon footprint and shop-floor physical scrap/energy waste through predictive maintenance[cite: 2]. |

---

## 7. Prioritized Architecture Risks & Mitigations

Per Deliverable 10 (*"with the top three risks and mitigations"*)[cite: 2], the top three enterprise risks—plus two secondary operational risks—are prioritized below:

### Risk 1 (Priority: CRITICAL — Compliance & Security): Cross-Cloud or AI-Agent Data Leakage of `TALON` Export-Controlled Technical Data
* **Risk Description:** As AeroForge connects `S3 Gold` to Microsoft Fabric OneLake, Snowflake, and Amazon Bedrock AI Agents[cite: 2, 3], a service-principal shortcut or a shared RAG vector index could inadvertently bypass user-level `Lake Formation` filters and expose `TALON` military trainer specifications to non-U.S. persons[cite: 2].
* **Concrete Mitigation:**
  1. Encrypt all `TALON` objects and embeddings with a dedicated **KMS CMK (`alias/af-talon-export-cmk`)** that enforces `aws:PrincipalTag/USPerson == "true"` at the cryptographic layer.
  2. Partition `Bedrock Knowledge Bases` with a mandatory `program_code` metadata filter injected by the **Bedrock AgentCore Runtime** from the caller's verified JWT claims[cite: 2].
  3. Prohibit shared service-principal cross-cloud shortcuts from mounting `TALON` prefixes unless end-to-end US-person identity passthrough is verified[cite: 2, 3].

### Risk 2 (Priority: HIGH — Reliability & Latency): Upstream Throttling or Cold-Shard Dropping During a 41.7× Telemetry Burst ($20,000\text{ msg/s}$)
* **Risk Description:** While average telemetry across 12 plants is only $480\text{ msg/s}$, simultaneous shift startups or plant-wide power sags can trigger a **$20,000\text{ msg/s}$ burst**[cite: 2]. Default AWS IoT Core Rule single-record puts ($10,000\text{ req/s}$ quota) and cold Kinesis On-Demand streams ($4\text{ MB/s}$ default initial capacity) would throttle before auto-scaling reacts, blinding the `<60s` spindle anomaly detector[cite: 2].
* **Concrete Mitigation:**
  1. Configure **Greengrass v2** to micro-batch sensor readings into `500 ms` JSON array envelopes and enable **IoT Core Rule `PutRecords` batch mode**[cite: 2].
  2. Pre-configure **Kinesis Data Streams On-Demand** with **`WarmThroughput = 25 MB/s`** (30 pre-warmed shards supporting $30,000\text{ records/s}$ instantaneously) and **Enhanced Fan-Out (EFO)** dedicated $2\text{ MB/s/shard}$ pipes for Apache Flink.

### Risk 3 (Priority: HIGH — FinOps & Budget): Runaway Query Scanning, Redshift Wake-Loops, and Cross-Cloud Egress Breaching the $\$18\text{K/mo}$ Budget
* **Risk Description:** Unpartitioned ad-hoc Athena queries, high-frequency BI dashboard auto-refreshes that prevent Redshift Serverless from pausing[cite: 1, 2], or repeated full-table scans from Azure/Snowflake across S3 shortcuts[cite: 3] could rapidly exhaust the **$\$18,000/\text{month}$ pilot budget**[cite: 2].
* **Concrete Mitigation:**
  1. Enforce **Athena Workgroup per-query scan limits** ($50\text{ GB}$ hard cutoff) and require partition predicates (`plant_code`, `date`) via Glue/Athena views[cite: 1, 2].
  2. Route high-concurrency BI dashboards to pre-computed **Redshift Materialized Views** refreshed hourly rather than continuous live streaming queries, allowing Redshift Serverless to auto-pause during off-hours[cite: 1, 2].
  3. Configure **AWS Budgets** with automated SNS/Slack alerts at 50%, 80%, and 100% of monthly spend by `Plant` tag[cite: 1, 2].

### Risk 4 (Priority: MEDIUM — Data Integrity): Uncaught Sensor Unit Drift or Late-Arriving Events Corrupting ML Features & P&L Variances
* **Risk Description:** Firmware updates at plant gateways (such as the Querétaro gateway reporting `°F` instead of `°C`) or delayed batches can trigger false-positive maintenance work orders or skewed hourly OEE calculations[cite: 1, 2, 3].
* **Concrete Mitigation:** Enforce **Glue Data Quality** unit-range checks in the Silver ETL pipeline (auto-converting `°F` to `°C` when `temp_unit == 'F'` or `spindle_temp > 115`) and apply a 2-hour event-time watermark in Apache Flink and Glue Spark `MERGE` jobs[cite: 1, 2].

### Risk 5 (Priority: MEDIUM — Sovereignty & DR): Single-Region Dependency for Mexican HR/Payroll Data in `mx-central-1`
* **Risk Description:** Because Mexican HR and payroll data cannot replicate outside Mexico[cite: 2] and AWS has only one Mexican region (`mx-central-1`), a catastrophic regional loss of all 3 AZs in Querétaro cannot fail over to another AWS region without violating residency rules.
* **Concrete Mitigation:** Rely on 3-AZ synchronous/multi-AZ deployment plus an air-gapped **In-Country Recovery AWS Account** in `mx-central-1` with **Backup Vault Lock**, supplemented by encrypted daily cold backups written over Direct Connect to the physical Querétaro plant vault.

---

## 8. Unresolved Resilience & Compliance Questions for the Professor

1. **Approval of Assumed Tier RPO/RTO Targets:**
   * *Question:* "The Part 1 instructions specify RPO/RTO for Telemetry (`15m / 4h`) and the Warehouse (`24h / 8h`)[cite: 2]. Does the grading rubric approve our assumed targets for Real-Time Alerting (`5m RPO / 1h RTO`), ML/Agents (`24h RPO / 8h RTO`), and In-Country Mexico HR (`0m AZ / 1h Snapshot RPO`, `4h RTO`)?"
2. **Mexico Catastrophic Regional DR Fallback:**
   * *Question:* "Given that Mexican HR and payroll data must remain in-country[cite: 2] and AWS operates only a single region in Mexico (`mx-central-1`), is 3-AZ high availability plus an isolated in-region AWS Backup Vault sufficient for DR, or should we propose a cross-cloud DR vault in **Azure Mexico Central** (or on-premises at the QRO plant) for catastrophic regional survival?"
3. **Inclusion of Standby Pilot-Light DR Costs in the $\$18\text{K/month}$ Pilot Target:**
   * *Question:* "Should the $<\$18,000/\text{month}$ Scenario A pilot cost estimate[cite: 2] include the continuous S3 Cross-Region Replication (CRR/RTC) and KMS/Backup storage costs in the `us-west-2` DR region ($\approx \$300/\text{month}$), or is DR priced only in the Scenario B (12-plant) enterprise run-rate?"

---

## 9. AI-Use Disclosure

* **Tool Used:** Generative AI collaborator (Gemini)[cite: 2].
* **How Used:** Used to structure the tiered RPO/RTO resilience table, synthesize the sovereignty-constrained replication rules for `mx-central-1` and `TALON`, and format the six-pillar AWS Well-Architected review[cite: 2].
* **Verification:** All explicit targets (`99.9%` ingestion availability, `15 min / 4 h` telemetry RPO/RTO, `24 h / 8 h` warehouse RPO/RTO, `2-year` / `7-year` retention, and `<$18K/month` pilot cost) were verified directly against `AeroForge_Final_Project_Part1_AWS_Design (1).docx`[cite: 2], and AWS service capabilities (`S3 Replication Time Control`, `AWS Backup Vault Lock`, `mx-central-1` 3-AZ topology) were verified against official AWS documentation.