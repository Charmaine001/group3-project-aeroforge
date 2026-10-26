# AeroForge Part 1: Step-by-Step AWS Pricing Calculator Input Guide

**Document Identifier:** `docs/part1-pricing-calculator-inputs.md`  
**Course:** MIS 612 — Fundamentals of Cloud Computing (Fall 2026) — Project 2, Part 1[cite: 2]  
**Purpose:** Exact service configurations and numerical inputs to enter at **`https://calculator.aws`** to generate the official shareable calculator links for **Scenario A (3-Plant Pilot)** and **Scenario B (12-Plant Enterprise)**[cite: 2].

---

## How to Generate Your Official Shareable Links in 10 Minutes

1. Open **[https://calculator.aws](https://calculator.aws)** in your browser and click **Create estimate**.
2. Create **Group 1: `Scenario A - AeroForge 3-Plant Pilot (600 Machines)`** and add the services below using the **Scenario A Inputs** column.
3. Once Scenario A is complete, click **Share** in the top-right corner of the AWS Pricing Calculator, copy the public **`https://calculator.aws/#/estimate?id=...`** URL, and paste it into your final report.
4. Duplicate the group (or create a second estimate) named **`Scenario B - AeroForge 12-Plant Enterprise (2400 Machines)`**, update the quantities using the **Scenario B Inputs** column, and click **Share** to generate your second URL[cite: 2].

---

## Master Calculator Input Worksheet

### Group 1A / 1B: Primary Production Region — `US East (N. Virginia) [us-east-1]`

| # | Calculator Service Search Name | Configuration Parameter / Field Name | Scenario A Input (3-Plant Pilot) | Scenario B Input (12-Plant Enterprise) | Expected Calculator Monthly Output (Approx.) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **1** | **AWS IoT Core** | • Region<br>• Device connectivity (minutes)<br>• Average message size<br>• Messaging (Basic Ingest / MQTT)<br>• Rules Engine (Rules triggered / Actions executed) | • `US East (N. Virginia)`<br>• `600 devices` $\times 43,800\text{ min}$<br>• `5 KB` (micro-batched envelope)<br>• `62,200,000` messages/month<br>• `62,200,000` rules & actions/month | • `US East (N. Virginia)`<br>• `2,400 devices` $\times 43,800\text{ min}$<br>• `5 KB` (micro-batched envelope)<br>• `248,800,000` messages/month<br>• `248,800,000` rules & actions/month | **Scen A:** `~$81.11`<br>**Scen B:** `~$324.44` |
| **2** | **Amazon Kinesis Data Streams** | • Capacity mode<br>• Number of streams<br>• Data ingested<br>• Average record size<br>• Enhanced fan-out (EFO) consumers<br>• Extended data retention (Days 2–7) | • `On-Demand`<br>• `1` stream<br>• `375 GB/month` (`~0.14 MB/s` avg)<br>• `1 KB`<br>• `2` EFO consumers (`750 GB/mo` retrieved)<br>• `Enabled (168 hours / 7 days)` | • `On-Demand`<br>• `1` stream<br>• `1,500 GB/month` (`~0.58 MB/s` avg)<br>• `1 KB`<br>• `2` EFO consumers (`3,000 GB/mo` retrieved)<br>• `Enabled (168 hours / 7 days)` | **Scen A:** `~$104.20`<br>**Scen B:** `~$329.20` |
| **3** | **Amazon Data Firehose** | • Direct PUT / Kinesis stream to S3<br>• Data ingested per month | • `375 GB per month` | • `1,500 GB per month` | **Scen A:** `~$10.88`<br>**Scen B:** `~$43.50` |
| **4** | **Amazon Managed Service for Apache Flink** | • Number of Apache Flink applications<br>• Kinesis Processing Units (KPUs) per app<br>• Running application storage | • `1` application<br>• `2 KPUs` (plus 1 app KPU = `2–3 KPUs` billed 730 hrs/mo)<br>• `100 GB/month` | • `1` application<br>• `8 KPUs` (plus 1 app KPU = `9 KPUs` billed 730 hrs/mo)<br>• `400 GB/month` | **Scen A:** `~$222.31`<br>**Scen B:** `~$856.86` |
| **5** | **AWS Database Migration Service (DMS)** | • Instance class<br>• Deployment type<br>• Hours per month<br>• General Purpose SSD (gp3) storage | • `dms.t3.medium`<br>• `Single-AZ` (`1` instance)<br>• `730 hours/month`<br>• `50 GB` | • `dms.c6i.large`<br>• `Multi-AZ` (`2` instances)<br>• `730 hours/month`<br>• `200 GB` | **Scen A:** `~$58.31`<br>**Scen B:** `~$472.68` |
| **6** | **AWS Transfer Family** | • Protocol enabled<br>• Number of endpoints<br>• Hours per month<br>• Data uploaded (SFTP IN) | • `SFTP`<br>• `1 endpoint`<br>• `730 hours/month`<br>• `25 GB/month` | • `SFTP`<br>• `2 endpoints`<br>• `730 hours/month` each<br>• `100 GB/month` | **Scen A:** `~$220.00`<br>**Scen B:** `~$442.00` |
| **7** | **Amazon AppFlow** | • Number of flow runs per month<br>• Data processed per month | • `2,500 runs/month`<br>• `10 GB/month` | • `9,500 runs/month`<br>• `40 GB/month` | **Scen A:** `~$2.70`<br>**Scen B:** `~$10.30` |
| **8** | **AWS Lambda** | • Architecture<br>• Number of requests<br>• Duration per request & Memory<br>• *(Optional)* Add small EC2 `t4g.medium` edge/API helper instance | • `ARM (Graviton2)`<br>• `500,000 requests/month` at `512 MB`, `500 ms`<br>• Add helper/API edge services buffer (`~$200`) | • `ARM (Graviton2)`<br>• `2,500,000 requests/month` at `512 MB`, `500 ms`<br>• Add helper/API edge services buffer (`~$220`) | **Scen A:** `~$201.09`<br>**Scen B:** `~$223.42` |
| **9** | **Amazon Simple Storage Service (S3)** | • **S3 Standard:** Storage + PUT/GET requests<br>• **S3 Glacier Instant Retrieval:** Storage<br>• **S3 Glacier Deep Archive:** Storage<br>• **S3 Tables (Apache Iceberg):** Storage + Compaction GB<br>• **Outbound Data Transfer:** Internet/Cross-Cloud | • S3 Standard: `2,200 GB`; `5M PUT`, `10M GET`<br>• Glacier IR: `3,500 GB`<br>• Deep Archive: `4,500 GB`<br>• S3 Tables: `1,200 GB` stored; `3,000 GB` compacted<br>• Data Transfer Out: `600 GB/month` | • S3 Standard: `8,800 GB`; `20M PUT`, `40M GET`<br>• Glacier IR: `14,000 GB`<br>• Deep Archive: `18,000 GB`<br>• S3 Tables: `4,800 GB` stored; `12,000 GB` compacted<br>• Data Transfer Out: `2,000 GB/month` | **Scen A:** `~$201.40` *(incl. `$45` egress)*<br>**Scen B:** `~$793.65` *(incl. `$171` egress)* |
| **10** | **AWS Glue** | • **ETL Spark Jobs (Standard):** DPU-hours/mo<br>• **ETL Spark Jobs (Flex):** DPU-hours/mo<br>• **Glue Data Quality & Crawlers:** DPU-hours/mo<br>• **Data Catalog:** Objects stored & requests | • Standard ETL: `528 DPU-hours/mo`<br>• Flex ETL: `300 DPU-hours/mo`<br>• Data Quality/Crawlers: `365 DPU-hours/mo`<br>• Catalog: `2,500,000 objects`, `20M requests` | • Standard ETL: `2,040 DPU-hours/mo`<br>• Flex ETL: `1,000 DPU-hours/mo`<br>• Data Quality/Crawlers: `876 DPU-hours/mo`<br>• Catalog: `6,500,000 objects`, `70M requests` | **Scen A:** `~$514.80`<br>**Scen B:** `~$1,698.20` |
| **11** | **Amazon Redshift** | • Mode: `Redshift Serverless`<br>• Base capacity (RPUs)<br>• Active hours per day<br>• Redshift Managed Storage (RMS) | • `Redshift Serverless`<br>• `32 RPUs`<br>• `6 hours/day` (`180 hours/month`)<br>• `500 GB` | • `Redshift Serverless`<br>• `64 RPUs`<br>• `7 hours/day` (`210 hours/month`)<br>• `2,000 GB` | **Scen A:** `~$2,172.00`<br>**Scen B:** `~$5,088.00` |
| **12** | **Amazon Athena** | • Data scanned per month | • `4.6 TB per month` | • `18.4 TB per month` | **Scen A:** `~$23.00`<br>**Scen B:** `~$92.00` |
| **13** | **Amazon QuickSight** | • Edition: `Enterprise`<br>• Authors<br>• Readers / Sessions | • `3 Authors` (`$24/mo` each)<br>• `10 Readers` (`~93 sessions/mo` at `$0.30`) | • `5 Authors` (`$24/mo` each)<br>• `30 Readers` (`~266 sessions/mo` at `$0.30`) | **Scen A:** `~$100.00`<br>**Scen B:** `~$200.00` |
| **14** | **Amazon SageMaker** | • Studio Notebooks (`ml.t3.medium`)<br>• Training & Batch Transform (`ml.m5.xlarge` / `ml.c5.2xlarge`)<br>• Feature Store & Model Monitor | • `2` notebooks (`160 hrs/mo` each)<br>• `~1,200 instance-hours/mo` (`ml.m5.xlarge`) across daily batch scoring, weekly training, and drift monitoring | • `4` notebooks (`160 hrs/mo` each)<br>• `~3,250 instance-hours/mo` (`ml.m5.xlarge`) across 12-plant scoring, SHAP, and forecasting | **Scen A:** `~$300.00`<br>**Scen B:** `~$780.00` |
| **15** | **Amazon OpenSearch Service (Serverless)** | • Mode: `OpenSearch Serverless (Vector Search)`<br>• Indexing OCUs + Search OCUs<br>• Storage | • `2 OCUs` total (stepped 0.5 OCU mode) $\times 730\text{ hrs}$<br>• `400 GB` index storage | • `4 OCUs` total $\times 730\text{ hrs}$<br>• `800 GB` index storage | **Scen A:** `~$360.00`<br>**Scen B:** `~$720.00` |
| **16** | **Amazon Bedrock** | • Model 1: `Claude 3.5 Haiku` (Routing/Simple Q&A)<br>• Model 2: `Claude 3.5 Sonnet` (SQL/RAG/Agents)<br>• Bedrock Guardrails & Knowledge Base sync | • `15,000 agent requests/mo` (`~150M` input tokens / `30M` output tokens mixed across Haiku & Sonnet + Guardrails + KB sync) | • `60,000 agent requests/mo` (`~350M` input tokens / `70M` output tokens mixed across Haiku & Sonnet + Guardrails + KB sync) | **Scen A:** `~$932.00`<br>**Scen B:** `~$2,115.00` |
| **17** | **AWS Direct Connect + AWS Site-to-Site VPN** | • Direct Connect Dedicated/Hosted 1 Gbps ports<br>• Site-to-Site VPN connections | • `3` Direct Connect 1G ports (`730 hrs/mo` each)<br>• `3` Site-to-Site VPN connections (`730 hrs/mo` each) | • `12` Direct Connect 1G ports (`730 hrs/mo` each)<br>• `12` Site-to-Site VPN connections (`730 hrs/mo` each) | **Scen A:** `~$766.50`<br>**Scen B:** `~$3,066.00` |
| **18** | **AWS Transit Gateway + AWS PrivateLink + NAT Gateway** | • Transit Gateway attachments & GB processed<br>• Interface VPC Endpoints (AZ-hours)<br>• NAT Gateways & Hub Inspection VPC | • `6` TGW attachments (`600 GB` processed)<br>• `8` Interface Endpoints $\times 2\text{ AZs}$<br>• `2` NAT Gateways + Network Firewall/Inspection (`~$720`) | • `10` TGW attachments (`2,500 GB` processed)<br>• `8` Interface Endpoints $\times 3\text{ AZs}$<br>• `3` NAT Gateways + Network Firewall/Inspection (`~$1,359`) | **Scen A:** `~$1,133.50`<br>**Scen B:** `~$2,048.00` |
| **19** | **AWS Step Functions** | • Workflow type: `Standard`<br>• State transitions per month | • `1,500,000 state transitions/month` | • `5,000,000 state transitions/month` | **Scen A:** `~$37.50`<br>**Scen B:** `~$125.00` |
| **20** | **AWS KMS + Secrets Manager + CloudWatch + Security Suite** | • Customer Managed Keys (CMKs) & API requests<br>• Secrets stored<br>• CloudWatch Logs ingested (`GB`) & Alarms<br>• CloudTrail, GuardDuty, Security Hub, Config, Macie | • `8` KMS keys + `15M` API calls (with S3 Bucket Keys)<br>• `25` secrets<br>• `450 GB` CloudWatch logs + `300` metrics/alarms<br>• Security suite baseline (`~$485`) | • `12` KMS keys + `46M` API calls<br>• `60` secrets<br>• `1,100 GB` CloudWatch logs + `650` metrics/alarms<br>• Security suite scaled (`~$890`) | **Scen A:** `~$891.30`<br>**Scen B:** `~$1,848.60` |

---

### Group 2: Disaster Recovery Region — `US West (Oregon) [us-west-2]`

| # | Calculator Service Search Name | Configuration Parameter / Field Name | Scenario A Input (3-Plant Pilot) | Scenario B Input (12-Plant Enterprise) | Expected Calculator Monthly Output (Approx.) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **21** | **Amazon S3 (CRR + Replication Time Control) & AWS Backup** | • Region: `US West (Oregon)`<br>• Inbound Cross-Region Replication volume with S3 RTC (`$0.02/GB` transfer + `$0.015/GB` RTC)<br>• Replicated S3 Standard/Glacier storage in `us-west-2`<br>• Redshift & AWS Backup cross-region snapshot storage | • `500 GB/month` replicated via S3 RTC<br>• `3,500 GB` replicated Gold/Silver/Bronze DR storage<br>• `1,500 GB` AWS Backup / Redshift snapshot vault | • `2,000 GB/month` replicated via S3 RTC<br>• `10,000 GB` replicated Gold/Silver/Bronze DR storage<br>• `3,000 GB` AWS Backup / Redshift snapshot vault | **Scen A:** `~$470.00`<br>**Scen B:** `~$550.00` |

---

### Group 3: Sovereign Mexico Spoke — `Mexico (Central) [mx-central-1]`

| # | Calculator Service Search Name | Configuration Parameter / Field Name | Scenario A Input (3-Plant Pilot) | Scenario B Input (12-Plant Enterprise) | Expected Calculator Monthly Output (Approx.) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **22** | **Amazon S3 (`mx-central-1`) + Local Backup Vault** | • Region: `Mexico (Central)`<br>• Sovereign QRO HR/payroll storage + in-country Backup Vault Lock storage | • `500 GB` S3 Standard + AWS Backup Vault in `mx-central-1` | • `1,500 GB` S3 Standard + AWS Backup Vault in `mx-central-1` | **Scen A:** `~$40.00`<br>**Scen B:** `~$120.00` |

---

## Reconciliation Check Before Saving Your Calculator Link

* **Scenario A Calculator Target Total:** **$\approx \$8,842.60 / \text{month}$** ($\sim \$106,111.20 / \text{year}$) — confirms compliance with the **$<\$18,000 / \text{month}$** pilot requirement[cite: 2].
* **Scenario B Calculator Target Total:** **$\approx \$21,946.85 / \text{month}$** ($\sim \$263,362.20 / \text{year}$) — confirms sub-linear scaling across 12 plants[cite: 2].