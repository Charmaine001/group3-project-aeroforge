# ADR-004: Multi-Modal Ingestion Patterns (Streaming, CDC, Batch SFTP, and REST APIs)

* **Status:** Draft (For Nov 1 Part 1 Submission & Nov 2 Class Review)[cite: 8, 10]
* **Date:** October 2026[cite: 8]
* **Related Baseline Decisions:** DEC-01, DEC-04, DEC-05, DEC-06, DEC-07, DEC-08, DEC-10[cite: 9]

---

## 1. Title and Status
**ADR-004: Multi-Modal Ingestion Patterns (Streaming, CDC, Batch SFTP, and REST APIs)** — **Status: Draft**[cite: 8, 10]

## 2. Context
As the enterprise System of Record (`DEC-01`), AWS must ingest nine heterogeneous data sources across three pilot plants (Wichita, Oklahoma City, Querétaro) scaling to 12 plants—plus Cimarron Composites (`i6_cimarron`) as a 13th site headroom reservation—into the Amazon S3 Raw Zone (`l1_raw` Bronze)[cite: 8, 9, 10]. These sources differ sharply in velocity, volume, and format: high-frequency OPC UA/MQTT machine telemetry (`s1_edge`), transactional on-premises SQL Server MES production orders (`s2_mes`), CMM quality inspection CSVs (`s3_cmm`), SaaS CMMS maintenance work orders with free-text notes (`s3_cmm`), nightly on-premises ERP general ledger/AP/budget extracts in USD and MXN (`s4_erp`), daily external REST APIs (Open-Meteo weather, titanium commodity indices, treasury FX rates), and ~20,000 SharePoint SOP documents (`s5_ext`)[cite: 8, 10].

## 3. Requirements and Constraints
* **Streaming Sizing Arithmetic & Specification Conflict (`DEC-04`, `DEC-05`):**
  * **Scenario B (12-Plant Enterprise):** $2,400\text{ machines} \times 0.2\text{ msg/s} (1\text{ msg}/5\text{ s}) = 480\text{ msg/s}$ average (~0.47 MB/s at ~1.0 KB/JSON event, yielding ~1.5 TB/month compressed)[cite: 8, 9].
  * **Scenario A (3-Plant Pilot):** $600\text{ machines} \times 0.2\text{ msg/s} = 120\text{ msg/s}$ average (~375 GB/month raw) with a proportional 5,000 msg/s peak burst (while the static development data pack slice contains 24 machines)[cite: 8, 9].
  * **10× vs. 20,000 msg/s Burst Conflict:** The Non-Functional Requirements (NFR) table specifies absorbing "10× telemetry bursts" ($480 \times 10 = 4,800\text{ msg/s}$), which conflicts with the Data Sources table requirement of a **20,000 msg/s peak burst** ($41.67\times$ average, or ~20 MB/s at 1 KB/msg)[cite: 8, 9]. Per `DEC-04`, we size the streaming pipeline for the higher **20,000 msg/s** ceiling so shift-startup or spindle-chatter storms never throttle or drop data[cite: 9, 10].
* **Latency & Availability SLAs:** Critical machine alarms within **60 seconds**; plant operational dashboards refreshed hourly; finance Gold tables (`l5_gold`) ready by **07:00 CT daily**; 99.9% ingestion availability with a 15-minute telemetry RPO and 4-hour RTO[cite: 8, 10].
* **Sovereignty, Budget & Schema Rules (`DEC-06`, `DEC-08`, `DEC-10`):** Scenario A AWS run-rate `< $18,000/month`; Querétaro HR/payroll PII remains in `mx-central-1` (`b_mexico`), exporting only de-identified GL balances and machine telemetry (`e10`); all unprovided schemas are marked `PROVISIONAL` (`prov::`)[cite: 9, 10].

## 4. Options Considered

### Option 1: Source-Aligned Hybrid Ingestion (Serverless Streaming + CDC + Managed SFTP + Event-Driven APIs)
Match each source to a purpose-built AWS ingestion service:
1. **Machine Telemetry (`s1_edge`):** Local buffering and OPC UA-to-MQTT conversion on `AWS IoT Greengrass` -> `AWS IoT Core` (`i1_iot`) -> `Amazon Kinesis Data Streams` On-Demand (`i2_kds`) -> dual consumption by `Amazon Managed Service for Apache Flink` (`<60s` anomaly rules) and `Amazon Data Firehose` (micro-batched `JSONL.gz` to `l1_raw`) (`i3_flink`)[cite: 8, 10].
2. **MES (`s2_mes`) & CMMS (`s3_cmm`):** Ongoing SQL Server transaction log CDC (~60K orders/month) via `AWS DMS (Serverless CDC)` and hourly SaaS CMMS polling (~4K orders/month) via `Amazon AppFlow` (or Lambda fallback) (`i4_dms`)[cite: 8, 9, 10].
3. **ERP (`s4_erp`), CMM CSVs (`s3_cmm`) & External APIs/Docs (`s5_ext`):** Nightly ERP extracts (~2M GL lines/month at 23:00 CT) and CMM CSVs (~150K/month) via `AWS Transfer Family (SFTP)`, plus daily Weather, FX, and Commodity REST polling via `AWS Lambda + EventBridge` (`i5_transfer`)[cite: 8, 9, 10].
* **Pros:** Aligns latency and cost with each source's Five Vs; satisfies both the `<60s` telemetry alarm SLA and the 07:00 CT finance close SLA; scales serverlessly within the `<$18K/month` pilot ceiling[cite: 8, 9, 10].
* **Cons:** Multi-service footprint requires centralized observability (`cc4_mon`) and orchestration (`cc1_orch`)[cite: 10].

### Option 2: Unified Kafka Streaming Backbone (Amazon MSK for All Sources)
Deploy `Amazon MSK` clusters and Kafka Connect / Debezium connectors to ingest telemetry, MES CDC, ERP financial extracts, and REST APIs as Kafka topics[cite: 8, 9].
* **Pros:** Single streaming abstraction across all nine sources[cite: 8].
* **Cons:** High operational overhead managing Kafka partitions and connectors; converting nightly 2M-line ERP CSV batches and 150K CMM CSV files into Kafka messages inflates compute costs without improving the 07:00 CT batch SLA[cite: 8, 9, 10].

### Option 3: Micro-Batch File Polling Only (No Real-Time Streaming Tier)
Spool all shop-floor sensor telemetry into 5-to-15-minute CSV/JSONL files on plant gateways and upload via SFTP/S3 alongside ERP and MES batches[cite: 8].
* **Pros:** Lowest ingestion cost and simplest architecture[cite: 8].
* **Cons:** Violates the mandatory **60-second** critical machine alarm NFR, leaving AeroForge blind to fast-developing CNC spindle bearing failures[cite: 8, 10].

## 5. Decision
We select **Option 1 (Source-Aligned Hybrid Ingestion)**, combining serverless streaming (`i1_iot`, `i2_kds`, `i3_flink`) for machine telemetry with serverless database CDC (`i4_dms`), managed SFTP (`i5_transfer`), and scheduled Lambda API polling (`i5_transfer`) for operational and financial sources[cite: 8, 9, 10].

## 6. Rationale
A single Kinesis shard provides **1,000 records/s** or **1 MB/s** of write capacity[cite: 9]. Absorbing AeroForge's **20,000 msg/s** (~20 MB/s) peak burst requires either **20 provisioned shards** or **Kinesis Data Streams On-Demand Mode**, which automatically scales up to 200 MB/s and 200,000 records/s[cite: 9, 10]. Pairing `Kinesis On-Demand` (`i2_kds`) with `Managed Service for Apache Flink` (`i3_flink`) delivers sub-60-second anomaly alerts (`e5` -> `in2_alerts`), while `Data Firehose` (`e4`) writes compressed `JSONL.gz` files to `l1_raw`[cite: 10]. For transactional and financial systems, `AWS DMS Serverless` (`i4_dms`) captures incremental MES changes without taxing on-premises SQL Server production tables, and `AWS Transfer Family` (`i5_transfer`) lands nightly ERP files at 23:00 CT so `AWS Glue ETL` (`l2_glue`) refreshes the authoritative Iceberg Gold tables (`l5_gold`) before 07:00 CT[cite: 9, 10].

## 7. Consequences and Trade-Offs
* **Edge & Burst Resilience:** Local `AWS IoT Greengrass` disk spooling (`s1_edge`) preserves the 15-minute telemetry RPO during WAN interruptions, and `Kinesis On-Demand` (`i2_kds`) decouples 20,000 msg/s spikes from downstream `Glue` and `Redshift Serverless` workloads[cite: 9, 10].
* **Veracity & Quarantine Handling:** Because raw payloads contain deliberate defects (duplicate telemetry, Fahrenheit gateway readings, missing GL cost centers, duplicate AP invoices), ingestion lands raw records immutably in `l1_raw` while `AWS Glue Data Quality` (`l2_glue`) routes rejects to `l3_quarantine`[cite: 8, 10].

## 8. Security and Cost Implications
* **Security (`cc2_sec`):** Plant edge gateways, DMS, and SFTP traffic traverse redundant **Site-to-Site VPN / Direct Connect + AWS Transit Gateway** (`i1_iot`) encrypted in transit (TLS 1.2+) and at rest (`AWS KMS` CMKs), with API credentials stored in `AWS Secrets Manager`[cite: 8, 10].
* **Cost (`cc5_cost`):** Serverless scaling across `IoT Core`, `Kinesis On-Demand`, `Firehose`, `DMS Serverless`, and `Lambda` eliminates idle over-provisioning for the 20,000 msg/s peak, keeping Scenario A within the `<$18,000/month` AWS pilot ceiling[cite: 9, 10].

## 9. Dependencies and Unresolved Questions
* **Dependencies:** Requires MS-CDC enabled on on-premises SQL Server MES (`s2_mes`) and schema reconciliation of all `prov::` attributes once `DATA_DICTIONARY.md` is provided (`DEC-10`)[cite: 9, 10].
* **Unresolved Questions for the Professor:**
  1. For Deliverable 8 pricing, should we cost 20 provisioned Kinesis shards (20,000 msg/s) or Kinesis On-Demand scaling from the 480 msg/s average[cite: 9]?
  2. Should Scenario A cost 24 sample machines or 600 pilot machines[cite: 9]?
  3. Should Cimarron (`i6_cimarron`) route shop-floor telemetry into `AWS IoT Core` or directly to Azure Event Hubs in Part 2[cite: 9]?

## 10. Facts to Verify Against Official AWS Documentation
1. **Amazon AppFlow SaaS CMMS Connector (`UNVERIFIED`):** Because the Part 1 brief does not name the specific SaaS CMMS vendor, native `Amazon AppFlow` connector availability (`i4_dms`) is `UNVERIFIED`; if unsupported, an AppFlow Custom Connector SDK or `EventBridge + AWS Lambda` REST polling pattern is required[cite: 8, 10].
2. **Kinesis On-Demand Instantaneous $41.67\times$ Burst Behavior (`UNVERIFIED`):** While `Kinesis Data Streams On-Demand` scales up to 200 MB/s and 200,000 records/s[cite: 9, 10], instantaneous throttling limits when jumping from 480 msg/s to 20,000 msg/s in seconds if prior peak traffic was low are `UNVERIFIED` (verify whether pre-warmed shards or provisioned mode with Auto Scaling is required).
3. **SharePoint SOP Ingestion Mechanism (`UNVERIFIED`):** Verify whether the ~20,000 SharePoint SOP documents (`s5_ext`) ingest directly via the managed SharePoint data source connector in `Amazon Bedrock Knowledge Bases` (`in3_kb`) or require staging into `l1_raw` (`UNVERIFIED`)[cite: 8, 10].

## 11. AI-Use Disclosure
Drafted with generative AI assistance (Google Gemini) from the MIS 612 Part 1 project brief, `docs/architecture-decision-baseline.md`, and `part1-service-rationale.md`[cite: 8, 9, 10]. The team verified all ingestion throughput calculations (480 msg/s average vs. 20,000 msg/s peak), diagram arrow mappings (`e1`–`e12`), and marked unconfirmed vendor connector behaviors as `UNVERIFIED`[cite: 9, 10].