
# AeroForge Part 1: Security, Compliance & Governance Design (Deliverable 6)

**Status:** Draft for Nov 1 Submission | **Schemas:** `PROVISIONAL` (AeroForge Data Pack and `DATA_DICTIONARY.md` unprovided)[cite: 24, 25, 27]
**Diagram Alignment:** Directly maps to `part1-network-security.drawio` (Shapes `n1`–`n21`, Connections `c1`–`c19`), `part1-logical-architecture.drawio`, and `aeroforge_part1_erd_1.xml`[cite: 26, 27].

> **Architectural vs. Legal Disclaimer:** This document specifies technical cloud security, access control, encryption, and audit mechanisms designed to *support* AeroForge's internal governance, financial reporting (SOX), export-control (ITAR/EAR), and Mexican data residency requirements[cite: 24, 25]. **No AWS service or technical control guarantees legal or regulatory compliance on its own.** Legal interpretations (such as statutory definitions of personal data in Mexico or U.S. defense export classification boundaries) require formal review by AeroForge's legal and export-compliance counsel[cite: 21, 22]. No real account IDs, credentials, keys, or unprovided corporate policies are invented in this specification[cite: 25].

---

## 1. Shared Responsibility Matrix (Security *of* vs. *in* the Cloud)

Under the AWS Shared Responsibility Model (`CF 4`), security and operational responsibilities are divided between AWS and AeroForge across the platform tiers[cite: 25]:

| Architecture Layer & Services | AWS Responsibility ("Security OF the Cloud") | AeroForge Responsibility ("Security IN the Cloud") |
|---|---|---|
| **Physical & Global Infrastructure** (`us-east-1`, `us-west-2`, `mx-central-1`)[cite: 24, 27] | Physical data center security, hardware lifecycle, availability zone power/networking redundancy, and FIPS 140-3 HSM hardware backing[cite: 24]. | Selecting approved regions (`us-east-1`, `us-west-2`, `mx-central-1`), enabling opt-in regions (`mx-central-1`), and configuring Service Control Policies (SCPs) to block unapproved regions[cite: 21, 24, 27]. |
| **Plant Edge & Hybrid Network** (`s1_edge` IoT Greengrass, `n8_tgw` VPN/Direct Connect + Transit Gateway)[cite: 25, 27] | Managed MQTT broker availability (`AWS IoT Core`), Transit Gateway control plane, and physical Direct Connect colocation routers[cite: 25, 27]. | Patching physical shop-floor edge gateway OS/firmware, rotating X.509 device certificates, stripping operator identifiers at Greengrass (`n1_us_plants`, `n2_qro_plant`), and maintaining redundant VPN/Direct Connect BGP tunnels (`c1`, `c2`, `c3`)[cite: 21, 27]. |
| **Serverless Data & Lakehouse Tier** (`Amazon S3`, `Kinesis On-Demand`, `DMS Serverless`, `Glue`, `Redshift Serverless`, `DynamoDB`)[cite: 25, 27] | Underlying compute patching, hypervisor isolation, multi-AZ storage durability (S3 11 nines), and managed engine execution[cite: 25, 27]. | Enabling `S3 BlockPublicAccess`, configuring S3 Object Lock and lifecycle rules, authoring KMS key policies (`CMK #1`–`#4`), defining Lake Formation `LF-Tags` and row/column filters, and tuning RPU/KPU scaling limits[cite: 25, 27]. |
| **AI & Machine Learning Tier** (`Amazon SageMaker AI`, `Amazon Bedrock`, `Amazon Bedrock AgentCore`)[cite: 25, 27] | Foundation model hosting security, enclave isolation ( AWS does not use customer Bedrock prompts to train base models), and managed runtime infrastructure[cite: 25, 27]. | Configuring **AgentCore Identity** propagation, authoring **Bedrock Guardrails** (`in3_kb`), scoping **AgentCore Gateway** MCP Lambda tools, enforcing Human-in-the-Loop (`in5_hitl`) approval gates, and preventing training data leakage[cite: 25, 27]. |

---

## 2. Multi-Account Structure (`AWS Organizations` & `AWS Control Tower`)

To enforce blast-radius isolation, regulatory segregation of duties, and clean FinOps showback, the AeroForge landing zone (`bg_aws_org` in `part1-network-security.drawio`) is organized under **AWS Organizations** and **AWS Control Tower** (`cc2_sec`) into five logical account boundaries (using symbolic names only, no invented account IDs)[cite: 25, 27]:

1. **Management & Identity Account (`n6_mgmt_acct`):** Hosts `AWS Organizations`, `AWS Control Tower`, `AWS IAM Identity Center` (SSO + MFA), and centralized FinOps (`AWS Budgets`, `Cost Explorer`, `CUR 2.0`)[cite: 25, 27]. Enforces preventive Organization **Service Control Policies (SCPs)** (`c17`):
   * *Region Allow-List SCP:* Denies all non-global service actions outside `us-east-1` (Primary US), `us-west-2` (US DR), and `mx-central-1` (Mexico Residency)[cite: 21, 24, 27].
   * *Storage Hardening SCP:* Denies disabling `s3:PutAccountPublicAccessBlock`, denies deleting S3 Object Lock configurations, and denies unencrypted S3/EBS/Redshift creation[cite: 25, 27].
2. **Dedicated Security & Log-Archive Account (`n7_sec_acct`):** Acts as the delegated administrator for `Amazon GuardDuty`, `AWS Security Hub`, `Amazon Macie`, and `AWS Config`[cite: 25, 27]. Receives immutable Organization-wide `AWS CloudTrail` and `AWS Config` logs (`c18`) into a locked-down S3 bucket protected by `S3 Object Lock` and `KMS CMK #4` so that even platform administrators in workload accounts cannot alter or delete audit trails (supporting SOX segregation of duties)[cite: 25, 27].
3. **Core Data Platform Account (`bg_us_acct` — `us-east-1` Primary / `us-west-2` DR):** Hosts the primary Data Platform VPC (`bg_vpc`), streaming/batch ingestion (`n12_priv_ingest`), commercial S3 Medallion Lakehouse (`n16_s3_comm`), central `AWS Lake Formation & Glue Data Catalog` (`n17_lf_gov`), `Amazon Redshift Serverless`, `Amazon SageMaker AI`, `Amazon QuickSight`, and **Amazon Bedrock AgentCore** (`n13_priv_serve`)[cite: 24, 25, 27].
4. **Dedicated Mexico In-Country Account (`bg_mex_acct` — `mx-central-1` Only):** Isolates Querétaro (QRO) employee HR, payroll, timecards, and direct operator PII (`n10_mex_store`) within Mexican territory per `ADR-002` and Binding Decision 8 (`DEC-08`)[cite: 21, 24]. An SCP on this Organizational Unit (OU) denies actions in any region other than `mx-central-1` and blocks S3 Cross-Region Replication[cite: 21, 27].
5. **Restricted TALON Account (`bg_talon_acct` — `us-east-1` / `us-west-2` Only):** Isolates export-controlled Cobalt Defense Systems TALON data (`n18_talon_s3`) and its dedicated encryption key (`n19_talon_kms`, `CMK #2`) per `ADR-003` and Binding Decision 9 (`DEC-09`)[cite: 22, 24]. Access requires cross-account Lake Formation grants (`c15`, `c16`) and IAM principal tag `isUSPerson = true`[cite: 22, 24]. *(Note: If simplified during implementation, this boundary can also be deployed as an isolated S3 bucket + dedicated `CMK #2` inside `bg_us_acct`, subject to the same ABAC policies[cite: 22].)*

---

## 3. Network Boundaries & Private Connectivity

All network paths in `part1-network-security.drawio` enforce private plant connectivity and eliminate public internet exposure for data storage and processing (`CF 5`)[cite: 25, 27]:

* **Plant-to-Cloud Hybrid Connectivity (`c1`, `c2`, `c3`, `c6`):**
  * **US Plants (`n1_us_plants` — WIC & OKC):** Connect via redundant IPsec **AWS Site-to-Site VPN** tunnels (or **AWS Direct Connect** for production scale) terminating at the **AWS Transit Gateway (`n8_tgw`)** in `us-east-1`, routing directly into private ingestion subnets (`n12_priv_ingest`)[cite: 25, 27].
  * **Querétaro Plant (`n2_qro_plant` — QRO):** Maintains two logical private routes: (1) `c3` routes QRO HR, payroll, and raw local ERP data directly to the `mx-central-1` VPC (`n10_mex_store`); (2) `c2` routes non-PII shop-floor MQTT telemetry (after local `AWS IoT Greengrass` strips operator IDs) to `n8_tgw` -> `AWS IoT Core` (`us-east-1`), as `IoT Core` is hosted in the US tier per `ADR-002`[cite: 21, 27].
* **Data Platform VPC (`bg_vpc` — `10.10.0.0/16` across 3 Availability Zones):**
  * **Private Ingest & ETL Subnet Tier (`n12_priv_ingest`):** Houses VPC-internal `AWS Transfer Family` SFTP endpoints (no public SFTP IP), `AWS DMS Serverless` replication ENIs, `AWS Glue Spark` ENIs, and private `AWS Lambda` functions[cite: 25, 27].
  * **Private Serve & AI Subnet Tier (`n13_priv_serve`):** Houses the `Amazon Redshift Serverless` workgroup (publicly accessible = `false`), `Amazon SageMaker AI` (configured in `VPC-Only` mode with direct internet access disabled), and `AgentCore Gateway` MCP tool Lambda ENIs[cite: 25, 27].
* **VPC Endpoints (`n14_vpce` — `c11`, `c12`, `c13`, `c14`, `c15`):**
  * **Gateway Endpoints:** Configured for `Amazon S3` and `Amazon DynamoDB` so multi-terabyte lake reads/writes never traverse NAT Gateways or the public internet[cite: 25, 27].
  * **Interface Endpoints (`AWS PrivateLink`):** Configured for `AWS KMS` (FIPS 140-3 endpoints), `AWS Glue`, `AWS Lake Formation`, `Amazon Redshift Data API`, `Amazon Bedrock`, `AWS Secrets Manager`, `AWS Step Functions`, `Amazon CloudWatch`, and `AWS STS`[cite: 24, 25, 27].
* **Public vs. Private Access Boundary (`n9_public_edge` — `c8`, `c9`, `c10`):**
  * **Zero Public S3 Buckets:** Account-level and bucket-level `BlockPublicAccess` is enabled across all accounts[cite: 25, 27].
  * **Inbound Workforce Portal (`c8`):** Workforce users (`n3_workforce`) access `Amazon QuickSight (Quick Suite)` and the operations portal exclusively over HTTPS (TLS 1.2+) fronted by `Amazon CloudFront` and `AWS WAF` after authenticating via `IAM Identity Center` (`c7`)[cite: 25, 27].
  * **Controlled Outbound Egress (`c9`, `c10`):** Only outbound HTTPS requests from scheduled ingestion/action `AWS Lambda` functions (`n12_priv_ingest`) to external endpoints (`n4_ext_apis`: Open-Meteo Weather API, Market/Treasury FX APIs, SharePoint Online, and SaaS CMMS API) traverse the public-subnet **NAT Gateway + Internet Gateway (IGW)**, restricted by egress Security Groups[cite: 25, 27].
* **Part 2 Cross-Cloud Network Boundary (`n5_part2_azure` — `c19`):** Microsoft Fabric OneLake S3 shortcuts and Snowflake on Azure External Volumes connect over TLS 1.2+ to the commercial S3 Gold bucket (`n16_s3_comm`) only[cite: 20, 24]. Because OneLake S3 shortcuts do not currently support Azure Private Link traversal to S3 (`ADR-001`), access is locked down via bucket policies and IAM conditions (`aws:SourceIp` / VPC / principal restrictions) and completely blocked from `n18_talon_s3` and `n10_mex_store`[cite: 13, 20, 21, 22].

---

## 4. Identity, Authentication & Least-Privilege Access (`IAM` + `Lake Formation` Matrix)

Workforce authentication is centralized in **AWS IAM Identity Center** (`n6_mgmt_acct`, federated with corporate identity / Microsoft Entra ID in Part 2) enforcing **Single Sign-On (SSO)** and phishing-resistant **Multi-Factor Authentication (MFA)** for every human user (`CF 4`, `Security NFR`)[cite: 24, 25, 27]. Long-lived IAM user access keys are prohibited for human users; workloads use short-lived `AWS STS` role sessions.

IAM Identity Center propagates three principal session tags into AWS (`c7`, `c17`):
* `isUSPerson` (`true` | `false`) — Verified U.S. Person status for export-control enforcement[cite: 22, 24, 27].
* `plant_code` (`WIC` | `OKC` | `QRO` | `ALL`) — Home plant assignment for operational/financial scoping[cite: 25, 26, 27].
* `job_persona` (`plant_manager` | `maintenance_lead` | `plant_controller` | `cfo_exec` | `data_engineer` | `data_scientist` | `sox_auditor`)[cite: 25, 27].

### Persona × Data Domain Permission Matrix (`IAM` + `AWS Lake Formation`)

*(All table names reference the provisional schema in `aeroforge_part1_erd_1.xml`[cite: 26] and are tagged `PROVISIONAL` per `DEC-10`[cite: 24].)*

| Persona / Service Role | Allowed AWS Services (IAM Role Scope) | Bronze (`l1_raw`) & Quarantine (`l3_quarantine`) | Silver (`l4_curated`) & Gold Commercial (`l5_gold` / `n16_s3_comm`) | TALON Export-Controlled (`n18_talon_s3`, `program_code = 'TALON'`) | Mexico HR/Payroll (`n10_mex_store` in `mx-central-1`) | SOX Segregation of Duties (SoD) Rule |
|---|---|---|---|---|---|---|
| **Plant Manager** (`WIC`, `OKC`, `QRO`)[cite: 25, 27] | `QuickSight` (Reader), `Bedrock AgentCore` (Invoke Copilot)[cite: 25, 27] | **No Access** | **Read-Only** (via Redshift/QuickSight RLS) scoped to assigned `plant_code` (OEE, downtime, daily Plant-to-P&L brief)[cite: 25, 26, 27]. | **Read-Only ONLY if** `isUSPerson = true` (QRO manager has `program_code = 'TALON'` rows filtered out)[cite: 22, 24, 26]. | **No Access** (HR PII excluded from operational views)[cite: 21, 24]. | Cannot modify Gold tables, ETL code, or financial journal entries[cite: 25, 27]. |
| **Maintenance Lead / Cell Lead**[cite: 25, 27] | `QuickSight`, `SNS` alerts, `Bedrock AgentCore` (Maintenance Advisor + HITL Approval `in5_hitl`)[cite: 25, 27] | **No Direct S3 Access** (reads indexed SOPs in `in3_kb` via RAG)[cite: 25, 27]. | **Read-Only** on `fact_machine_telemetry_hourly`, `fact_maintenance_work_order`, `dim_machine`, and `DynamoDB` risk scores[cite: 26, 27]. | **Read-Only ONLY if** `isUSPerson = true` (required to query `WIC-CNC-03` TALON parts)[cite: 22, 25, 26]. | **No Access** (operator IDs stripped at Greengrass edge)[cite: 21, 27]. | Can approve drafted P1 CMMS work orders (`in5_hitl`), logged in DynamoDB/CloudTrail[cite: 25, 27]. |
| **Plant Controller** (`WIC`, `OKC`, `QRO`) & **CFO**[cite: 25, 27] | `QuickSight` (Author/Reader), `Redshift Serverless` (SQL), `Bedrock AgentCore` (Variance & Supply Agents)[cite: 25, 27] | **No Access** to raw files; Read-Only on aggregated DQ scores[cite: 25, 27]. | **Read-Only** on `fact_financial_gl_variance`, `fact_production_work_order`, `dim_cost_center`, `dim_gl_account`, `dim_supplier` (scoped by `plant_code`, or `ALL` for CFO)[cite: 26, 27]. | **Read-Only ONLY if** `isUSPerson = true` (Non-US controllers see only rolled-up non-technical commercial totals or have TALON rows masked)[cite: 22, 24, 26]. | **No Access** from US Lakehouse (QRO HR/Payroll queried only in `mx-central-1` by authorized Mexican HR/Finance)[cite: 21, 24]. | Strictly **Read-Only** on Gold Iceberg (`l5_gold`); cannot alter ETL pipelines or overwrite ERP actuals[cite: 13, 25, 27]. |
| **Data Engineer**[cite: 25, 27] | `Glue`, `Step Functions`, `DMS`, `Transfer Family`, `Athena`, `CloudWatch` (Non-Prod Write; Prod via CI/CD)[cite: 25, 27] | **Read-Only** in Prod for troubleshooting; **Read/Write** on `l3_quarantine` for DQ remediation[cite: 25, 27]. | **No Interactive Write in Prod** (only the automated `Glue ETL` execution role deployed via `GitHub Actions / CodePipeline` can write to `l4_curated` and `l5_gold`)[cite: 25, 27]. | **Metadata / Pipeline Admin ONLY if** `isUSPerson = true` and granted `CMK #2` access[cite: 22, 24]. | Scoped to de-identification pipeline code via CI/CD; no interactive raw HR PII read[cite: 21, 27]. | Cannot approve their own PRs to Production (`cc1_orch`); cannot modify CloudTrail logs (`n7_sec_acct`)[cite: 25, 27]. |
| **Data Scientist (ML)**[cite: 25, 27] | `SageMaker AI` (`sv3_sagemaker`), `Athena` (`sv2_athena`)[cite: 25, 27] | **No Access** (trains strictly on validated Silver data)[cite: 27]. | **Read-Only** on `l4_curated` Silver features; automated `SageMaker Pipeline` role writes batch scores/SHAP to dedicated `prov::fact_ml_*` Gold tables (`e20`) and `DynamoDB` (`e21`)[cite: 26, 27]. | **Read/Train ONLY if** `isUSPerson = true`[cite: 22, 24]. | **No Access** (models never train on employee PII)[cite: 21, 24]. | Cannot overwrite ERP financial actuals (`fact_financial_gl_variance`)[cite: 13, 26]. |
| **SOX / Internal Auditor**[cite: 25, 27] | `Athena` (Time-Travel SQL), `CloudTrail`, `Config`, `Security Hub` (`n7_sec_acct`)[cite: 13, 25, 27] | **Read-Only** on `l1_raw` ERP Object Lock buckets and `l3_quarantine` logs[cite: 25, 27]. | **Read-Only** (including Apache Iceberg `FOR SYSTEM_TIME AS OF` snapshot queries in `Athena`)[cite: 13, 24, 27]. | **No Access** unless verified `isUSPerson = true` and explicitly scoped for defense audit[cite: 22, 24]. | **No Access** to Mexican HR PII from US account[cite: 21, 24]. | Strictly Read-Only across all data and log accounts; zero write/admin privileges[cite: 25, 27]. |
| **Part 2 Cross-Cloud Reader Role** (`n5_part2_azure`)[cite: 20, 24, 27] | Dedicated IAM Role/Access Key for `Microsoft Fabric OneLake` & `Snowflake External Volume` (`c19`)[cite: 13, 24, 27] | **Explicit Deny** | **Read-Only** (`s3:GetObject`, `s3:ListBucket`, `glue:GetTable`) strictly on commercial Gold Iceberg prefixes (`n16_s3_comm`)[cite: 13, 24, 27]. | **Explicit Deny** (Blocked at IAM policy, Bucket Policy, and `CMK #2` Key Policy)[cite: 13, 22, 24]. | **Explicit Deny** (No OneLake shortcut permitted to `mx-central-1`)[cite: 21, 24]. | Read-only consumer (`ADR-001`); cannot execute DML writes against AWS Gold Iceberg[cite: 13, 24]. |

---

## 5. Encryption in Transit and at Rest & Key Management (`AWS KMS`)

### 5.1 Encryption in Transit (`TLS 1.2+` Everywhere)
* All shop-floor MQTT streams from `AWS IoT Greengrass` (`s1_edge`) to `AWS IoT Core` (`i1_iot`) use mutually authenticated **TLS 1.2+ (X.509 device certificates)** over IPsec Site-to-Site VPN / Direct Connect tunnels (`c1`, `c2`)[cite: 25, 27].
* Nightly ERP and CMM batch transfers use **SSH/SFTP (FIPS-approved ciphers)** into VPC-hosted `AWS Transfer Family` endpoints (`i5_transfer`, `c6`)[cite: 24, 25, 27].
* Every S3 bucket policy (`n10_mex_store`, `n16_s3_comm`, `n18_talon_s3`, and `n7_sec_acct`) enforces a mandatory condition denying unencrypted HTTP requests (`"Bool": {"aws:SecureTransport": "false"}` -> `Deny`) and requiring minimum TLS version `1.2` (`s3:TlsVersion`)[cite: 25, 27].

### 5.2 Encryption at Rest & 4-Tier `AWS KMS` Customer Managed Key (CMK) Hierarchy (`n20_kms_summary`)
All persistent storage (`Amazon S3`, `Kinesis Data Streams`, `Redshift Serverless`, `DynamoDB`, `EBS` edge volumes, `Glue Data Catalog`, `SNS` topics, `CloudWatch Logs`, and `Secrets Manager`) is encrypted at rest using **AWS Key Management Service (KMS) Customer Managed Keys (CMKs)** with automatic annual key rotation enabled (`CF 4`, `Security NFR`)[cite: 25, 27]. To enforce cryptographic blast-radius isolation across sensitivity levels (`n20_kms_summary` in `part1-network-security.drawio`), four distinct CMKs are provisioned:

1. **`CMK #1` — Commercial Analytics & SOX Finance Key (`us-east-1`, replicated as Multi-Region Key to `us-west-2` DR):** Encrypts the commercial Medallion Lakehouse (`n16_s3_comm`), `Kinesis Data Streams` (`i2_kds`), `Redshift Serverless` (`sv1_redshift`), `DynamoDB` (`sv4_dynamodb`), and `SageMaker AI` volumes[cite: 24, 27]. Key policy grants `kms:Encrypt` only to automated pipeline IAM roles (`Firehose`, `DMS`, `Glue`, `SageMaker`) and `kms:Decrypt` to authorized analytical roles and the Part 2 commercial cross-cloud reader role (`c19`)[cite: 13, 24, 27].
2. **`CMK #2` — TALON Export-Controlled Defense Key (`n19_talon_kms` — `us-east-1` / `us-west-2` Only):** FIPS 140-3 HSM-backed CMK dedicated to `n18_talon_s3` and TALON Glue catalog metadata[cite: 22, 24]. Its KMS Key Policy contains an explicit **Deny** statement blocking `kms:Decrypt` for any principal where `aws:PrincipalTag/isUSPerson != 'true'`, providing a cryptographic backstop even if an IAM policy is misconfigured[cite: 22, 24, 27].
3. **`CMK #3` — Mexico Sovereign PII Key (`n10_mex_store` — `mx-central-1` Only):** Single-region CMK created inside the Dedicated Mexico Account (`bg_mex_acct`) in `mx-central-1`[cite: 21, 24, 27]. Because single-region KMS keys can never leave the AWS region in which they were created, data encrypted under `CMK #3` cannot be decrypted outside Mexican territory[cite: 21].
4. **`CMK #4` — Security & Audit Archive Key (`n7_sec_acct` — Security Account):** Encrypts the Organization `AWS CloudTrail`, `AWS Config`, `GuardDuty`, and `Macie` S3 archive buckets[cite: 25, 27]. Key administration is restricted strictly to the Security Account officer role (workload account admins have zero decrypt/delete rights)[cite: 25, 27].

---

## 6. Secrets Management (`AWS Secrets Manager`)

Hardcoded credentials, database passwords, API tokens, and SFTP private keys in source code, environment variables, or Glue scripts are strictly prohibited (`Security NFR`: *"Secrets in a vault"*)[cite: 25, 27]:
* **Vault Storage (`n15_secrets`):** All external credentials—including on-premises SQL Server MES CDC credentials (`i4_dms`), SaaS CMMS REST API tokens (`i4_dms`), Market/Treasury FX API keys (`i5_transfer`), SharePoint Online OAuth client secrets (`s5_ext`), and `AWS Transfer Family` SFTP user keys (`i5_transfer`)—are stored in **AWS Secrets Manager** encrypted under `KMS CMK #1`[cite: 25, 27].
* **Automated Rotation & Private Retrieval:** Secrets Manager is configured with automated Lambda-backed rotation schedules (e.g., 30-to-90-day rotation) and accessed privately by `DMS`, `Lambda`, and `Glue` over the `Secrets Manager AWS PrivateLink Interface Endpoint` (`n14_vpce`, `c11`, `c12`)[cite: 25, 27].

---

## 7. Audit Logging, Threat Detection & SOX Controls (`n7_sec_acct`)

To support Sarbanes-Oxley (SOX) financial governance (segregation of duties, audit trail, change control) and operational security observability (`CF 4`, `Compliance NFR`)[cite: 25, 27]:
1. **Immutable API & Data Access Audit Trail (`AWS CloudTrail`):** An Organization-wide `AWS CloudTrail` trail logs all management events and S3/Lambda/DynamoDB **data events** on Gold (`l5_gold`), TALON (`n18_talon_s3`), and Mexico (`n10_mex_store`) buckets, delivering logs with **CloudTrail Log File Validation** (SHA-256 digest chaining) to `n7_sec_acct` (`c18`)[cite: 22, 25, 27].
2. **Configuration & Change Control (`AWS Config` & CI/CD):** `AWS Config` continuously records resource configuration changes and evaluates conformance packs (e.g., `s3-bucket-public-read-prohibited`, `s3-bucket-replication-enabled`, `encrypted-volumes`)[cite: 25, 27]. All production ETL (`l2_glue`), schema DDL (`cc3_gov`), and IAM/Lake Formation policy changes must flow through peer-reviewed pull requests in `GitHub Actions / AWS CodePipeline` (`cc1_orch`) deploying `CloudFormation / AWS CDK / Terraform` templates—providing a complete SOX change-control trail[cite: 25, 27].
3. **Continuous Threat & PII Detection (`GuardDuty`, `Security Hub`, `Macie`):**
   * **Amazon Macie** runs automated sensitive-data discovery scans across US S3 Bronze/Silver buckets (`n16_s3_comm`) to detect and alert if Mexican employee PII or unmasked SSNs/identifiers accidentally cross the `c5` export gate[cite: 21, 25, 27].
   * **Amazon GuardDuty** monitors VPC Flow Logs (`n15_secrets`), DNS logs, CloudTrail events, and S3 data events for anomalous exfiltration or credential misuse, aggregating findings into **AWS Security Hub** (`n7_sec_acct`)[cite: 25, 27].

---

## 8. Catalog, Data Classification, Lineage & Access Governance (`AWS Glue` & `AWS Lake Formation`)

Centralized metadata and fine-grained access control are enforced by **AWS Glue Data Catalog** and **AWS Lake Formation** (`n17_lf_gov`, `cc3_gov`)[cite: 25, 27]:

### 8.1 Four-Tier Data Classification & LF-Tag Taxonomy
Every database, table, and column registered in the Glue Data Catalog is assigned four mandatory **Lake Formation Tags (`LF-Tags`)** (`DE 5`)[cite: 25, 27]:
* `sensitivity`: `public_external` (Weather/Commodity indices) | `internal_ops` (Telemetry, MES, CMMS, CMM) | `confidential_sox` (ERP GL, AP, Budget, Standard Cost, Gold P&L Star Schema) | `restricted_talon` (Cobalt Defense TALON technical & operational records) | `restricted_mexico_pii` (QRO HR/payroll in `mx-central-1` only)[cite: 21, 22, 24, 25, 27].
* `plant_code`: `WIC` | `OKC` | `QRO` | `CIM` (Cimarron 13th-site headroom reservation `i6_cimarron`) | `ENTERPRISE`[cite: 24, 26, 27].
* `domain`: `telemetry` | `manufacturing` | `quality` | `maintenance` | `finance` | `ml_predictions`[cite: 25, 26, 27].
* `schema_status`: `PROVISIONAL` (`prov::` namespace per `DEC-10` until `DATA_DICTIONARY.md` is provided)[cite: 24, 26, 27].

### 8.2 End-to-End Source-to-Report Data Lineage (`Observability NFR`)
To trace every dollar of P&L variance back to a machine, work order, and supplier invoice within 24 hours[cite: 25]:
* Every Bronze-to-Silver and Silver-to-Gold `AWS Glue ETL (Spark)` job (`l2_glue`) stamps deterministic audit columns onto every Iceberg row: `prov::source_system`, `prov::source_file_uri`, `prov::ingestion_ts`, `prov::glue_job_run_id`, `prov::dq_ruleset_version`, and (for currency conversion) `prov::original_currency` and `prov::fx_rate_to_usd` (`aeroforge_part1_erd_1.xml`)[cite: 24, 26, 27].
* **Apache Iceberg Snapshot Lineage (`l5_gold`):** Every write to `fact_financial_gl_variance`, `fact_production_work_order`, `fact_maintenance_work_order`, and `fact_machine_telemetry_hourly` generates an immutable Iceberg snapshot ID, allowing SOX auditors to query the exact historical state of any P&L report via `Athena` (`FOR SYSTEM_TIME AS OF`)[cite: 13, 24, 26, 27].

---

## 9. Mexico Payroll & HR Data Residency (`ADR-002` & `DEC-08`)

To satisfy the project requirement that Querétaro (QRO) payroll and HR data remain in Mexico (`Compliance NFR`, `DEC-08`, `ADR-002`)[cite: 21, 24, 25]:
1. **In-Country Physical Storage (`n10_mex_store` in `mx-central-1`):** All QRO payroll, HR records, employee/operator identifiers, timecards, and employee-level GL postings are ingested over `c3` into the Dedicated Mexico Account (`bg_mex_acct`) in **AWS Mexico (Central) (`mx-central-1`)**, encrypted with `KMS CMK #3`[cite: 21, 24, 27].
2. **Allow-Listed Export Gate (`n11_mex_gate` -> `c5`):** Only an allow-listed de-identification pipeline (`AWS Lambda / Glue` in `mx-central-1`) can read `n10_mex_store` and export **strictly de-identified, aggregated MXN general ledger journal balances and standard-cost variance rollups** (`c5`) to the US ingestion tier (`n12_priv_ingest`) for MXN->USD conversion and Plant-to-P&L consolidation[cite: 21, 24, 27].
3. **Edge De-Identification of Shop-Floor Telemetry (`c2`):** Because `AWS IoT Core` runs in `us-east-1`, QRO `AWS IoT Greengrass` edge gateways (`n2_qro_plant`) strip all operator badges/identifiers locally before transmitting machine sensor JSON over `c2`[cite: 21].
4. **No Cross-Border Caching or DR Leakage:** Per `ADR-002`, no Part 2 Microsoft Fabric OneLake shortcut is permitted from a US Fabric capacity to `mx-central-1` (because OneLake shortcut caching would hold Mexican bytes in the US for up to 28 days)[cite: 21]. Similarly, `mx-central-1` disaster recovery relies on **3-AZ Multi-AZ replication + in-country `AWS Backup` vaults** inside `mx-central-1` (`n21_dr_summary`) rather than replicating Mexican HR PII to a US region[cite: 21]. *(Whether pseudonymized/aggregated data or cross-border backup qualifies as personal data under Mexico's March 2025 data protection framework is flagged as a legal review item, not a technical guarantee[cite: 21].)*

---

## 10. TALON Export-Control Isolation (`ADR-003`, `ADR-001` & `DEC-09`)

To protect export-controlled TALON military trainer data (`Compliance NFR`, `DEC-09`, `ADR-003`) while reconciling the cross-cloud read constraints identified in `ADR-001`[cite: 13, 22, 24, 25]:
1. **US-Only Regional Pinning (`bg_talon_acct` / `b_talon`):** TALON data is restricted by SCP (`c17`) and S3 bucket policy to **US Regions only (`us-east-1` primary and `us-west-2` DR)**[cite: 22, 24, 27]. *(As documented in `ADR-003`, whether AeroForge's defense contract requires **AWS GovCloud (US)** vs. commercial US regions with FIPS 140-3 KMS CMKs is maintained as an open scoping question for the professor, not a legal conclusion[cite: 22, 24].)*
2. **Physical Bucket/Table Separation vs. Lake Formation Row Filtering (Reconciling `ADR-001` & `ADR-003`):**
   * **Internal AWS Readers (`Redshift Serverless`, `Athena`, `Bedrock AgentCore`):** Controlled via `AWS Lake Formation` Tag-Based Access Control (`LF-TBAC`, `sensitivity = restricted_talon`), row-level data cell filters (`program_code != 'TALON'` for non-US persons on shared tables like `fact_production_work_order` and `dim_part` in `aeroforge_part1_erd_1.xml`[cite: 26]), and `KMS CMK #2` (`aws:PrincipalTag/isUSPerson = 'true'`)[cite: 22, 26, 27]. Furthermore, per `ADR-001`, **no Redshift Materialized Views (`mv_*`) are created over restricted TALON rows** because MVs cache pre-computed results and do not dynamically re-evaluate Lake Formation row revocations per user[cite: 13].
   * **Part 2 Cross-Cloud Readers (`n5_part2_azure` — Fabric OneLake & Snowflake):** Because OneLake S3 shortcuts and Snowflake External Volumes read underlying S3 Parquet files directly via IAM role/key credentials—bypassing Lake Formation row filters (`ADR-001`)—all TALON technical data (CMM coordinate inspections, CAD/SOPs, and TALON-specific S3 prefixes `n18_talon_s3`) are stored in **physically separate S3 prefixes/buckets encrypted with `CMK #2` that are never exposed to Part 2 cross-cloud shortcuts (`c19`)**[cite: 13, 22, 24].

---

## 11. Data Retention, Tiering & Deletion Policies (`CF 7` & `Retention NFR`)

Retention and deletion controls are automated via **Amazon S3 Lifecycle Policies**, **S3 Versioning**, and **S3 Object Lock** (`l1_raw`, `n16_s3_comm`, `n7_sec_acct`)[cite: 25, 27]:
* **Raw Machine Telemetry (`2-Year Retention`):** Stored in `l1_raw` (`raw/iot_telemetry/`) with S3 Lifecycle rules automatically transitioning objects: **S3 Standard** (Days 0–30 for active ETL/reprocessing) -> **S3 Glacier Instant Retrieval** (Days 31–180) -> **S3 Glacier Deep Archive** (Days 181–730), followed by automated expiration at **Day 731 (2 years)**[cite: 25, 27].
* **Finance & SOX Audit Evidence (`7-Year Retention`):** ERP general ledger, AP, budget, and standard cost extracts in `l1_raw` (`raw/erp/*`), Gold financial snapshots, and `n7_sec_acct` CloudTrail/Config archives are protected with **S3 Versioning** and **S3 Object Lock (Governance/Compliance WORM mode)** with a retention period of **2,555 days (7 years)**, preventing premature deletion or tampering[cite: 13, 25, 27].
* **Curated & Gold Rebuildability (`l4_curated`, `l5_gold`):** Because `l1_raw` Bronze is immutable and append-only, any corrupted Silver or Gold Iceberg table can be deterministically rebuilt from Raw at any time (`Retention NFR`), supplemented by automated Iceberg snapshot expiration and compaction jobs to control metadata bloat[cite: 13, 25, 27].

---

## 12. Security of AI Agents & Tool Access (`Amazon Bedrock AgentCore` & `Guardrails`)

*(Architectural Note: **Amazon Bedrock AgentCore** (`in4_agentcore`) and **Amazon Bedrock Agents** are distinct AWS offerings; AeroForge uses **Amazon Bedrock AgentCore**—specifically `AgentCore Runtime`, `AgentCore Gateway`, `AgentCore Memory`, `AgentCore Identity`, and `AgentCore Observability`—together with `Amazon Bedrock Knowledge Bases` and `Amazon Bedrock Guardrails` (`in3_kb`)[cite: 25, 27].)*

To satisfy all `AI Governance NFR` mandates[cite: 25]:
1. **End-to-End User Identity Propagation (`AgentCore Identity`):** The multi-agent system (Supervisor *"AeroForge Copilot"*, *Maintenance Advisor*, *Variance Analyst*, and *Supply & Cost Agent*) never queries data using a god-mode superuser role[cite: 25, 27]. **AgentCore Identity** propagates the calling user's `IAM Identity Center` principal identity and session tags (`isUSPerson`, `plant_code`, `job_persona`) across **AgentCore Gateway** MCP tool invocations (`c12`, `c14`, `c15`)[cite: 22, 25, 27]:
   * When the *Maintenance Advisor* queries `Amazon Bedrock Knowledge Bases` (`in3_kb`) for SOPs or queries `Redshift Serverless` (`sv1_redshift`) for *"Which TALON parts were machined on WIC-CNC-03 while its vibration was in the Alert band?"*, the vector store metadata filter (`is_export_controlled`) and Lake Formation/KMS policy check the caller's `isUSPerson = true` entitlement before returning any TALON data[cite: 22, 25, 26, 27].
2. **Least-Privilege MCP Tool Scoping (`AgentCore Gateway`):** Agents cannot execute arbitrary SQL (`DROP`, `INSERT`, `UPDATE`) or unrestricted shell code[cite: 13, 25, 27]. **AgentCore Gateway** exposes strictly parameterized AWS Lambda MCP tools:
   * *Read Tools:* Query only read-only governed semantic views in `Redshift Serverless` (e.g., `v_variance_analyst_agent` in `aeroforge_part1_erd_1.xml`[cite: 26]) and read-only `DynamoDB` health state (`sv4_dynamodb`)[cite: 25, 26, 27].
   * *Write Tools:* The CMMS work-order creation tool (`Open a P1 work order for WIC-CNC-03 and notify the cell lead`) cannot call the SaaS CMMS API directly from the LLM; it only writes a draft payload to `Human-in-the-Loop Action Approval` (`in5_hitl`, Arrow `e29`)[cite: 25, 27].
3. **Input/Output Filtering (`Amazon Bedrock Guardrails` — `in3_kb`):** Attached to all four agents on `AgentCore Runtime` to enforce: (a) **Denied Topics** (blocking off-topic non-manufacturing/finance queries and unauthorized legal/HR advice), (b) **Sensitive Information Filters (PII Redaction)** (masking names, emails, badge IDs, or phone numbers in CMMS free-text notes), (c) **Prompt-Attack & Content Filters** (blocking jailbreaks/prompt injection inside free-text technician notes), and (d) **Contextual Grounding Checks** (blocking hallucinated variance numbers or ungrounded SOP citations)[cite: 25, 27].
4. **Mandatory Human-in-the-Loop (HITL) Gate & Audit Logging (`in5_hitl`):** Every state-changing agent action pauses in an `AWS Step Functions` callback / `AgentCore` confirmation state until an authorized human supervisor or cell lead reviews and approves the drafted CMMS work order[cite: 25, 27]. Both the agent's draft and the human approver's user ID (`hitl_approver_user_id` in `fact_maintenance_work_order`[cite: 26]), timestamp, and payload are immutably logged in `Amazon DynamoDB` (`sv4_dynamodb`), **AgentCore Observability**, and `AWS CloudTrail` (`n7_sec_acct`)[cite: 25, 26, 27].

---

## 13. Items Requiring Verification Against Official AWS Documentation (`UNVERIFIED`)

1. **Amazon Bedrock AgentCore Identity & PrivateLink Mechanics (`UNVERIFIED`):** Confirm the exact IAM session-tag propagation syntax used by **Amazon Bedrock AgentCore Identity** and **AgentCore Gateway** when invoking `Amazon Redshift Data API` and `AWS Lake Formation`-governed tables over `AWS PrivateLink`[cite: 17, 22, 27].
2. **Commercial US Regions vs. `AWS GovCloud (US)` for TALON (`UNVERIFIED` — Pending Professor Clarification):** As noted in `ADR-003` and `n19_talon_kms`, confirm with the course instructor whether the grading rubric expects commercial US regions (`us-east-1`/`us-west-2`) with FIPS KMS CMKs and ABAC (`Option 1`), or an `AWS GovCloud (US)` enclave (`Option 2`)[cite: 22, 24].
3. **Security Service Parity in `mx-central-1` (`UNVERIFIED`):** Per `ADR-002`, while `S3`, `KMS`, `VPC`, `Glue`, `Athena`, and `Lake Formation` are verified in `mx-central-1`, regional availability of `Amazon Macie`, `Amazon GuardDuty`, `AWS Security Hub`, and `AWS Backup` in `mx-central-1` is `UNVERIFIED`[cite: 21].
4. **Lake Formation Cross-Account Resource Links on Iceberg (`UNVERIFIED`):** Verify whether querying cross-account `Apache Iceberg` Resource Links (`n18_talon_s3` in `bg_talon_acct` queried from `sv1_redshift` in `bg_us_acct`) has any current Redshift Serverless external-schema limitations compared to single-account bucket/prefix isolation[cite: 22].

## 14. AI-Use Disclosure
Drafted with generative AI assistance (Google Gemini) using the MIS 612 Part 1 project brief[cite: 25], `docs/architecture-decision-baseline.md`[cite: 24], `ADR-001` through `ADR-005`[cite: 13, 16, 17, 21, 22], `part1-service-rationale.md`[cite: 27], `aeroforge_part1_erd_1.xml`[cite: 26], and `part1-network-security.drawio`. The team verified all node/connection mappings (`n1`–`n21`, `c1`–`c19`), separated technical controls from legal compliance determinations, and flagged unconfirmed service mechanics as `UNVERIFIED`.