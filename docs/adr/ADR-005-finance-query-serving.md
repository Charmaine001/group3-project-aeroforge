# ADR-005: Finance & Plant-to-P&L Query Serving Architecture

* **Status:** Draft (For Nov 1 Part 1 Submission & Nov 2 Class Review)[cite: 8, 10]
* **Date:** October 2026[cite: 8]
* **Related Baseline Decisions:** DEC-01, DEC-02, DEC-03, DEC-05, DEC-06, DEC-09, DEC-10[cite: 9]

---

## 1. Title and Status
**ADR-005: Finance & Plant-to-P&L Query Serving Architecture** — **Status: Draft**[cite: 8, 10]

## 2. Context
AeroForge Precision Components currently requires 9 business days to complete month-end financial close, during which plant controllers cannot trace scrap spikes (2.4% of revenue), unplanned downtime ($11.6M), or titanium Purchase Price Variance (PPV) back to specific machines, work orders, and supplier invoices[cite: 8]. Under Binding Decisions 1 and 2 (`DEC-01`, `DEC-02`), AWS is the enterprise finance System of Record, and the Amazon S3 Gold Zone (`l5_gold`) formatted in **Apache Iceberg** is the single authoritative store for the conformed Plant-to-P&L Star Schema[cite: 9, 10]. AeroForge requires an AWS analytical serving layer (`4. SERVE`) that reads these Gold Iceberg tables in place without creating a separate authoritative physical copy (`DEC-03`), while serving `Amazon QuickSight (Quick Suite)` dashboards (`in1_quicksight`, `e23`) and text-to-SQL variance queries from **Amazon Bedrock AgentCore** (`in4_agentcore`, `e27`)[cite: 8, 9, 10].

## 3. Requirements and Constraints
* **Single Authoritative Gold & Zero-Copy Mandate (`DEC-02`, `DEC-03`):** The serving engine must query S3 Gold Apache Iceberg tables (`l5_gold`) in place via the `AWS Glue Data Catalog` (`cc3_gov`)[cite: 9, 10]. Byte-for-byte duplicate warehouse tables are prohibited; only transient derived materialized views and caches are permitted, and Part 2 engines (Microsoft Fabric OneLake shortcuts and Snowflake External Volumes, `l6_part2_hook`) must read the same S3 Iceberg files[cite: 9, 10].
* **Latency & Concurrency SLAs:** Finance Gold tables ready by **07:00 CT daily**; plant operational dashboards refreshed hourly; concurrent support for plant managers, maintenance leads, controllers across 3 pilot plants (Scenario A) scaling to 12+1 plants (Scenario B + Cimarron), and multi-agent text-to-SQL via **Amazon Bedrock AgentCore** (distinct from declarative Amazon Bedrock Agents)[cite: 8, 9, 10].
* **Pilot Budget Ceiling (`DEC-06`):** Total AWS Scenario A pilot run-rate must stay below **<$18,000/month**[cite: 8, 9, 10].
* **Provisional Schemas (`DEC-10`):** Star-schema fact and dimension definitions are marked `PROVISIONAL` (`prov::`) until the AeroForge Data Pack and `DATA_DICTIONARY.md` are provided[cite: 9, 10].

## 4. Options Considered

### Option 1: Amazon Redshift Serverless (In-Place Iceberg Read + Derived Materialized Views)
Deploy `Amazon Redshift Serverless` (`sv1_redshift`) mounting S3 Gold Iceberg tables as an external schema (`e17`) with derived materialized views for high-frequency KPI rollups, paired with `Amazon Athena` (`sv2_athena`, `e18`) for ad-hoc data quality auditing[cite: 9, 10].
* **Cost:** Pay-per-use Redshift Processing Units (RPUs) that auto-scale and auto-pause during idle off-shift hours, fitting easily inside the `<$18,000/month` Scenario A ceiling[cite: 8, 9, 10].
* **Concurrency:** High multi-user BI concurrency, workload management queues, and sub-second semantic query caching for `QuickSight` and `Bedrock AgentCore`[cite: 8, 10].
* **Iceberg Support:** Natively queries S3 Gold Apache Iceberg tables registered in `AWS Glue Data Catalog` (`cc3_gov`) without duplicating base storage[cite: 9, 10].

### Option 2: Amazon Athena-Only (Serverless SQL-on-Lake)
Use `Amazon Athena` (`sv2_athena`) as the sole SQL serving engine for all finance reporting, dashboards, and AI agents[cite: 8, 10].
* **Cost:** Pure pay-per-query pricing per TB scanned with zero idle compute cost[cite: 10].
* **Concurrency:** Lacks dedicated workload management and materialized view acceleration for complex multi-fact star-schema joins during simultaneous 07:00 CT executive brief refreshes and month-end controller close queries[cite: 8, 9, 10].
* **Iceberg Support:** Full native Apache Iceberg read/write support over S3 and Glue Data Catalog[cite: 9, 10].

### Option 3: Provisioned Amazon Redshift RA3 Clusters
Run always-on provisioned `Redshift RA3` nodes with managed storage and Redshift Spectrum over S3 Gold[cite: 8, 9].
* **Cost:** Always-on 24/7 multi-node RA3 compute plus external scan charges significantly increase baseline spend, risking the `<$18,000/month` 3-plant pilot ceiling during quiet night/weekend shifts[cite: 8, 9].
* **Concurrency:** Enterprise-grade concurrency scaling and predictable performance at 12-plant scale[cite: 8, 9].
* **Iceberg Support:** Queries S3 Iceberg tables via Redshift Spectrum external schemas, but encourages loading data into native Redshift Managed Storage, which risks drifting from the single-copy S3 Gold mandate (`DEC-02`, `DEC-03`)[cite: 9].

## 5. Decision
Per Binding Decision 3 (`DEC-03`), we select **Option 1: Amazon Redshift Serverless (`sv1_redshift`)** reading the S3 Gold Apache Iceberg tables (`l5_gold`) in place as the primary finance and Plant-to-P&L serving warehouse, complemented by **Amazon Athena (`sv2_athena`)** for ad-hoc lake exploration and Data Quality (`l3_quarantine`) scorecards[cite: 9, 10].

## 6. Rationale
`Amazon Redshift Serverless` (`sv1_redshift`) balances performance, governance, and FinOps discipline[cite: 9, 10]. Unlike `Athena`-only, Redshift Serverless provides governed semantic views, derived materialized views (explicitly permitted under the `DEC-03` Data Classification Taxonomy because they are transient and recomputable from S3 Gold), and low-latency join performance for the Variance Analyst and Supply & Cost agents running on **Amazon Bedrock AgentCore** (`in4_agentcore`, `e27`)[cite: 8, 9, 10]. Unlike provisioned `Redshift RA3`, Redshift Serverless scales RPUs dynamically for the 07:00 CT daily brief and month-end close bursts while scaling down during idle windows to protect the `<$18,000/month` pilot target[cite: 8, 9, 10].

## 7. Consequences and Trade-Offs
* **Authoritative Storage Integrity (`DEC-02`, `DEC-03`):** All canonical Star Schema facts and dimensions remain in `l5_gold` as Apache Iceberg tables written by `AWS Glue ETL` (`l2_glue`, `e16`), ensuring AWS `Redshift Serverless` (`e17`) and Part 2 Azure/Fabric/Snowflake engines (`l6_part2_hook`, `e22`) read identical numbers[cite: 9, 10].
* **Cold-Start vs. Auto-Pause Trade-Off:** Aggressive RPU auto-pause saves cost in Scenario A but can introduce a brief resume delay on the first query after an idle window; pre-warming scheduled runs before 07:00 CT mitigates this[cite: 8, 10].

## 8. Security and Cost Implications
* **Security (`cc2_sec`, `cc3_gov`, `b_talon`):** Access to Gold Iceberg tables is governed centrally by `AWS Lake Formation` (`cc3_gov`) tag-based and row/column filters[cite: 9, 10]. Queries from `QuickSight` (`in1_quicksight`) and **Amazon Bedrock AgentCore** (`in4_agentcore`, using **AgentCore Identity** and **AgentCore Gateway** MCP tools) propagate caller identity so non-U.S. persons cannot read export-controlled `prov::program = 'TALON'` records (`DEC-09`) and Querétaro controllers see only authorized plant views (`DEC-08`)[cite: 8, 9, 10].
* **Cost (`cc5_cost`):** Redshift Serverless RPU caps (e.g., max RPU limits per workgroup) and Athena per-query scan limits prevent runaway query bills, tracked via plant/domain showback tags (`DEC-06`)[cite: 8, 9, 10].

## 9. Dependencies and Unresolved Questions
* **Dependencies:** Requires `AWS Glue ETL` (`l2_glue`) to complete nightly ERP (`s4_erp`) currency normalization (`MXN -> USD`) and Iceberg table compaction before 07:00 CT, plus final star-schema keys once `DATA_DICTIONARY.md` is provided (`DEC-10`)[cite: 8, 9, 10].
* **Unresolved Question for the Professor:** For the Scenario A (`<$18,000/month`) Pricing Calculator model in Deliverable 8, how many daily active hours and base RPUs should we assume for `Redshift Serverless`[cite: 8, 9, 10]?

## 10. Facts to Verify Against Official AWS Documentation
1. **Redshift Serverless External Iceberg Materialized Views (`UNVERIFIED`):** Verify current AWS limitations on creating incremental auto-refreshing materialized views in `Amazon Redshift Serverless` (`sv1_redshift`) directly over external `Apache Iceberg` tables registered in `AWS Glue Data Catalog` / `Amazon S3 Tables` (`l5_gold`) (`UNVERIFIED`)[cite: 9, 10].
2. **Redshift Serverless RPU Billing vs. Spectrum Scan Charges on Iceberg (`UNVERIFIED`):** Verify whether querying S3 Iceberg tables from `Redshift Serverless` consumes only workgroup RPU compute or incurs separate per-TB Spectrum scan charges compared to `Redshift RA3` (`UNVERIFIED`)[cite: 9, 10].
3. **Amazon Bedrock AgentCore Gateway Redshift Integration (`UNVERIFIED`):** Confirm whether **Amazon Bedrock AgentCore Gateway** (`in4_agentcore`) invokes `Redshift Serverless` via the Redshift Data API through an AWS Lambda MCP tool with propagated `IAM Identity Center` credentials (`UNVERIFIED`)[cite: 8, 10].

## 11. AI-Use Disclosure
Drafted with generative AI assistance (Google Gemini) using the MIS 612 Part 1 specification, `docs/architecture-decision-baseline.md`, and `part1-service-rationale.md`[cite: 8, 9, 10]. The team verified consistency with Binding Decisions `DEC-01` through `DEC-10`, diagram flows (`e16`, `e17`, `e18`, `e22`, `e23`, `e24`, `e27`), and marked specific engine/billing capabilities as `UNVERIFIED` for confirmation against official AWS documentation[cite: 9, 10].