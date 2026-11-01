

**# AeroForge Part 1: Comprehensive Technical Architecture Review**



**\*\*Review Date:\*\* November 1, 2026 | \*\*Status:\*\* Pre-Submission Technical Audit | \*\*Schemas:\*\* `PROVISIONAL`\[cite: 11, 12, 19]**



**---**



**## 1. Proof of File Inspection (Exact First Heading Quoted per Attached File)**



**1. \*\*`Architecture decision baseline\_5.docx`:\*\***

&#x20;  **\* Document Title Line: `"AeroForge Architecture Decision Baseline"`\[cite: 11]**

&#x20;  **\* First Section Heading: `"1. Binding Architectural Decision Baseline"`\[cite: 11]**

**2. \*\*`part1-logical-architecture.drawio` (Diagram XML):\*\***

&#x20;  **\* Root / Diagram Tag: `<mxfile host="app.diagrams.net"> / <diagram name="Page-1" id="Iq7bNxW\_2M4J-Z15czZ7">`\[cite: 18]**

&#x20;  **\* First Visual Heading Banner (`id="title\_banner"`): `"AeroForge Precision Components — Part 1: AWS Enterprise Logical Architecture (Plant-to-P\&L Digital Thread)\&#xa;Scope: Scenario A (3-Plant Pilot: WIC, OKC, QRO | 600 Machines) -> Scenario B (12 Plants | 2,400 Machines, 20,000 msg/s Burst) + Cimarron (13th Site Headroom) | Schemas: PROVISIONAL"`\[cite: 18]**

**3. \*\*`part1-service-rationale\_3.md`:\*\***

&#x20;  **\* First Heading: `"# AeroForge Part 1: AWS Service Rationale \& Traceability"`\[cite: 19]**

**4. \*\*`part1-coverage-map.md`:\*\***

&#x20;  **\* First Heading: `"# AeroForge Part 1: AWS Academy Concept Coverage Map (Deliverable 4)"`\[cite: 12]**

**5. \*\*`ADR-001-finance-system-of-record.md`:\*\***

&#x20;  **\* First Heading: `"# ADR-001: Finance System of Record"`\[cite: 13]**

**6. \*\*`ADR-002-regional-residency.md`:\*\***

&#x20;  **\* First Heading: `"# ADR-002: Regional Topology and Mexico Data Residency"`\[cite: 14]**

**7. \*\*`ADR-003-TALON-isolation.md`:\*\***

&#x20;  **\* First Heading: `"# ADR-003: TALON Export-Control Isolation Architecture"`\[cite: 15]**

**8. \*\*`ADR-004-ingestion-patterns.md`:\*\***

&#x20;  **\* First Heading: `"# ADR-004: Multi-Modal Ingestion Patterns (Streaming, CDC, Batch SFTP, and REST APIs)"`\[cite: 16]**

**9. \*\*`ADR-005-finance-query-serving.md`:\*\***

&#x20;  **\* First Heading: `"# ADR-005: Finance \& Plant-to-P\&L Query Serving Architecture"`\[cite: 17]**



**---**



**## 2. Executive Summary of the 8 Mandatory Technical Checkpoints**



**1. \*\*Service Existence \& Role Fit:\*\* All 33 shapes in `part1-logical-architecture.drawio` and `part1-service-rationale\_3.md` map to real AWS services\[cite: 18, 19]. Two integration patterns (`Amazon AppFlow` for an unnamed SaaS CMMS in `i4\_dms`, and `Amazon S3 Tables` vs. general-purpose S3 Iceberg buckets in `l5\_gold` for OneLake shortcuts) have technical constraints flagged in the table below\[cite: 13, 18, 19].**

**2. \*\*Diagram Arrow Technical Feasibility:\*\* All 29 arrows (`e1`–`e29`) represent valid AWS data/control paths\[cite: 18, 19], with two nuances requiring explicit documentation: (a) several boxes combine two services (`i3\_flink`, `i4\_dms`, `i5\_transfer`), so arrows `e4`/`e5`, `e7`, and `e10` must be read as targeting the specific sub-service inside the box\[cite: 18, 19]; and (b) arrows `e25` (`l1\_raw -> in3\_kb`) and `e27` (`sv1\_redshift -> in4\_agentcore`) represent RAG indexing and MCP tool responses rather than push streaming\[cite: 18, 19].**

**3. \*\*20,000 msg/s Streaming Arithmetic:\*\***

&#x20;  **\* \*\*Average Load (Scenario B, 12 Plants):\*\* $2,400\\text{ machines} \\times 0.2\\text{ msg/s} = 480\\text{ msg/s}$ ($\\approx 480\\text{ KB/s} = 0.469\\text{ MB/s}$ at $1\\text{ KB/msg}$; $\\approx 1.24\\times 10^9\\text{ events/month} \\approx 1.24\\text{ TB/month}$ raw uncompressed; $\\approx 1.5\\text{ TB/month}$ with wrapper metadata)\[cite: 11, 16].**

&#x20;  **\* \*\*Average Load (Scenario A, 3-Plant Pilot):\*\* $600\\text{ machines} \\times 0.2\\text{ msg/s} = 120\\text{ msg/s}$ ($\\approx 0.117\\text{ MB/s}$) with a $5,000\\text{ msg/s}$ ($5\\text{ MB/s}$) burst ceiling\[cite: 11, 16].**

&#x20;  **\* \*\*Peak Burst Arithmetic:\*\* The project's Data Sources table requires a \*\*$20,000\\text{ msg/s}$\*\* burst ($41.67\\times$ the $480\\text{ msg/s}$ average), conflicting with the NFR table's "10× telemetry bursts" ($4,800\\text{ msg/s}$)\[cite: 11, 16]. At $1\\text{ KB/msg}$, $20,000\\text{ msg/s} = 20\\text{ MB/s}$\[cite: 11, 16]. Because 1 Kinesis shard supports \*\*1,000 records/s\*\* or \*\*1 MB/s\*\* write capacity, provisioned mode requires exactly $\\frac{20,000\\text{ records/s}}{1,000\\text{ records/s/shard}} = \\mathbf{20\\text{ shards}}$\[cite: 11, 16]. In \*\*Kinesis On-Demand Mode\*\*, the stream ceiling is 200 MB/s and 200,000 records/s, but AWS On-Demand streams only accommodate instantaneous bursts up to \*\*$2\\times$ the previous peak within a 15-minute window\*\* (see Finding #3 below).**

**4. \*\*S3 Gold Single Authoritative Copy ("No Third Copy"):\*\* `l5\_gold` (Apache Iceberg on S3) is strictly preserved as the single authoritative store\[cite: 11, 13, 17, 19]. `Redshift Serverless` (`sv1\_redshift`) mounts the Glue Catalog as an external schema (`e17`) and is restricted to aggregate-only derived materialized views (`ADR-001` explicitly forbids `CTAS`/`INSERT` local table copies and row-level pass-through MVs)\[cite: 13, 17, 19].**

**5. \*\*Amazon Bedrock AgentCore vs. Amazon Bedrock Agents:\*\* `part1-logical-architecture.drawio`, `part1-service-rationale\_3.md`, `part1-coverage-map.md`, `ADR-003`, and `ADR-005` correctly distinguish \*\*Amazon Bedrock AgentCore\*\* (Runtime, Gateway for MCP tools, Memory, Identity, Observability) from declarative \*\*Amazon Bedrock Agents\*\*\[cite: 12, 15, 17, 18, 19]. However, `Architecture decision baseline\_5.docx` (Section 6) still conflates them under the vendor product label `Amazon Web Services (Bedrock Agents)`\[cite: 11] (see Finding #1).**

**6. \*\*Mexico Region (`mx-central-1`) Assumptions:\*\* `ADR-002` and `b\_mexico` properly isolate QRO HR/payroll PII inside `mx-central-1` (launched Jan 2025, 3 AZs, opt-in region)\[cite: 11, 14, 18, 19]. However, `Architecture decision baseline\_5.docx` cites the repealed Mexican privacy law acronym (`LFPDPPP`), whereas `ADR-002` notes Mexico's data protection law was replaced in March 2025\[cite: 11, 14], and `b\_mexico` vs. `ADR-002` have a minor discrepancy on how QRO machine telemetry exits Mexico (see Finding #6)\[cite: 14, 18, 19].**

**7. \*\*TALON Isolation Claims:\*\* `ADR-001` (Security section) and `ADR-003` contain a direct technical contradiction on whether `AWS Lake Formation` row filters vs. separate unexposed physical tables protect TALON data and whether `Redshift` Materialized Views can be built over restricted data\[cite: 13, 15] (see Finding #4).**

**8. \*\*Cross-File Contradictions:\*\* Seven cross-file contradictions (`ADR-003`/`ADR-004` numbering mismatches across documents, `Amazon S3 Tables` vs. general-purpose S3 bucket compatibility for Fabric OneLake shortcuts, and Cimarron ingestion routing in the baseline) are cataloged below\[cite: 11, 12, 13, 15, 16, 19].**



**---**



**## 3. Technical Review Findings Table**



**| Severity | Finding | Affected file | Evidence or official source | Recommended correction |**

**|---|---|---|---|---|**

**| \*\*HIGH\*\* | \*\*Check 5 — Baseline Conflates Amazon Bedrock AgentCore with Amazon Bedrock Agents:\*\* Section 6 of the baseline lists `"Amazon Bedrock AgentCore Architecture"` under the product name `"Amazon Web Services (Bedrock Agents)"` and states that AgentCore merely `"aligns functionally with the Bedrock Agent Runtime and Gateway architecture"`\[cite: 11]. Conversely, the diagram (`in4\_agentcore`), service rationale, coverage map (Row 23), `ADR-003`, and `ADR-005` explicitly treat \*\*Amazon Bedrock AgentCore\*\* (modular Runtime, Gateway/MCP, Memory, Identity, Observability) and \*\*Amazon Bedrock Agents\*\* (declarative Action Groups + Knowledge Bases) as distinct AWS offerings\[cite: 12, 15, 17, 18, 19]. | `Architecture decision baseline\_5.docx` (Section 6 table)\[cite: 11] | AWS Documentation distinguishes \*\*Amazon Bedrock Agents\*\* (managed declarative agent orchestration) from \*\*Amazon Bedrock AgentCore\*\* (enterprise agentic primitives: Runtime, Gateway for MCP tools, Memory, Identity, Observability). | Update Section 6 of `Architecture decision baseline\_5.docx` to list `Amazon Web Services (Amazon Bedrock AgentCore)` as a separate product from `Amazon Bedrock Agents`, matching `part1-service-rationale\_3.md` (`in4\_agentcore`)\[cite: 11, 19]. |**

**| \*\*HIGH\*\* | \*\*Check 4 \& Check 8 — S3 Tables Bucket vs. General-Purpose S3 Bucket Contradiction for Part 2 OneLake Shortcuts:\*\* `part1-logical-architecture.drawio` (`l5\_gold`), `part1-service-rationale\_3.md`, and `Architecture decision baseline\_5.docx` (Section 1 DEC-02 \& Section 6) specify `Amazon S3 Tables` (`Apache Iceberg / S3 Tables`) for the Gold Zone\[cite: 11, 18, 19]. However, `ADR-001` (Dependencies section) explicitly warns: \*"S3 Tables or self-managed Iceberg (ADR-003)? Fabric shortcuts need a general-purpose bucket"\* and flags `"shortcuts to S3 Tables buckets"` as `UNVERIFIED`\[cite: 13]. | `part1-logical-architecture.drawio` (`l5\_gold`), `part1-service-rationale\_3.md` (#17), `Architecture decision baseline\_5.docx` (Section 6), `ADR-001-finance-system-of-record.md`\[cite: 11, 13, 18, 19] | Microsoft Fabric OneLake S3 shortcuts require a standard S3 HTTPS bucket endpoint (`https://<bucket>.s3.<region>.amazonaws.com`) and IAM Access Key/Role credentials; AWS `s3tables` uses a specialized table-bucket ARN namespace (`arn:aws:s3tables:...`) whose direct compatibility with OneLake S3 shortcuts is `UNVERIFIED`\[cite: 13, 19]. | Standardize the Gold Zone (`l5\_gold`) across all files as \*\*Apache Iceberg tables stored in a general-purpose Amazon S3 bucket registered in the AWS Glue Data Catalog\*\*, while noting `Amazon S3 Tables` as an AWS-native alternative whose OneLake shortcut interop is `UNVERIFIED`\[cite: 13, 19]. |**

**| \*\*HIGH\*\* | \*\*Check 3 — Kinesis Data Streams On-Demand Instantaneous $41.67\\times$ Burst Limit (480 msg/s to 20,000 msg/s):\*\* The baseline (`DEC-04`), diagram (`i2\_kds`), service rationale, and `ADR-004` size the stream for `480 msg/s` average ($0.469\\text{ MB/s}$) and a `20,000 msg/s` ($20\\text{ MB/s}$) peak burst ($41.67\\times$ average) using \*\*Kinesis On-Demand Mode\*\*\[cite: 11, 16, 18, 19]. While On-Demand mode scales up to $200\\text{ MB/s}$ and $200,000\\text{ records/s}$, a brand-new On-Demand stream starts at $4\\text{ MB/s}$ ($4,000\\text{ records/s}$) default capacity and only permits instantaneous bursts up to \*\*$2\\times$ the prior peak throughput observed in the last 30 days\*\* (scaling further over \~15 minutes). An instantaneous jump from $0.47\\text{ MB/s}$ to $20\\text{ MB/s}$ on an un-warmed On-Demand stream will trigger `ProvisionedThroughputExceededException` until shards split\[cite: 16]. | `Architecture decision baseline\_5.docx` (Section 3.1), `part1-service-rationale\_3.md` (`i2\_kds`), `part1-coverage-map.md` (Row 10), `ADR-004-ingestion-patterns.md`\[cite: 11, 12, 16, 19] | Official Amazon Kinesis Data Streams Quotas: On-Demand streams provide $4\\text{ MB/s}$ and $4,000\\text{ writes/s}$ initially, scale automatically in 15-minute intervals, and instantaneously absorb up to $2\\times$ the previous peak write throughput. Provisioned mode with \*\*20 shards\*\* ($20 \\times 1,000\\text{ msg/s} = 20,000\\text{ msg/s}$, $20\\text{ MB/s}$) guarantees immediate zero-throttle capacity from second zero\[cite: 11, 16]. | Keep `ADR-004`'s verification note (`UNVERIFIED` instantaneous $41.67\\times$ burst) and specify in the configuration that if \*\*Kinesis On-Demand\*\* is used, the stream must be \*\*pre-warmed to a 10 MB/s ($10,000\\text{ msg/s}$) peak baseline\*\* (so $2\\times$ instantaneous headroom $= 20,000\\text{ msg/s}$), or else provisioned with \*\*20 shards\*\* during shift-startup windows\[cite: 11, 16, 19]. |**

**| \*\*HIGH\*\* | \*\*Check 7 \& Check 8 — Contradiction Between `ADR-001` and `ADR-003` on TALON Table Segregation vs. Lake Formation Row Filtering \& Redshift MVs:\*\* `ADR-003` (Sections 5 \& 7) and `part1-service-rationale\_3.md` (`b\_talon`) state that TALON data resides in a unified Plant-to-P\&L Star Schema where non-US persons have `program = 'TALON'` rows dynamically filtered out by \*\*AWS Lake Formation row/column filters\*\*\[cite: 15, 19]. Conversely, `ADR-001` (Security section) states: \*"External-engine setup requires full-table access, so row filters likely cannot protect TALON. Keep TALON and Mexico HR data in separate, unexposed tables"\* and \*"Redshift MVs ignore later Lake Formation revocations; no MVs over restricted data"\*\[cite: 13]. | `ADR-001-finance-system-of-record.md` vs. `ADR-003-TALON-isolation.md` and `part1-service-rationale\_3.md` (`b\_talon`, `sv1\_redshift`)\[cite: 13, 15, 19] | AWS Lake Formation Data Cell Filters support row-level security for Athena and Redshift Spectrum/Serverless external tables inside AWS, \*\*but\*\* cross-cloud IAM roles used by Microsoft Fabric OneLake S3 shortcuts and Snowflake External Volumes read underlying S3 Parquet files directly via S3 APIs—bypassing Lake Formation row filters entirely\[cite: 13, 15]. | Reconcile `ADR-001` and `ADR-003` by specifying \*\*physical table/prefix separation (`b\_talon` separate S3 prefix + separate KMS CMK) for any table exposed to Part 2 cross-cloud shortcuts\*\*, while using Lake Formation row filters (`program != 'TALON'`) and non-cached dynamic views (not MVs) for internal AWS Redshift/Athena users\[cite: 13, 15, 19]. |**

**| \*\*MEDIUM\*\* | \*\*Check 8 — Cross-File ADR Numbering \& Topic Mismatch (`ADR-002`, `ADR-003`, `ADR-004`):\*\* `part1-coverage-map.md` and `part1-service-rationale\_3.md` cite the wrong ADR numbers compared to the actual attached ADR files: (1) Coverage Map Row 7 and Service Rationale Row 17 state that S3 Tables vs. Glue Iceberg is evaluated in `"ADR-003"` (actual `ADR-003` is \*\*TALON Isolation\*\*)\[cite: 12, 13, 15, 19]; (2) Coverage Map Row 16 states Kinesis vs. MSK is in `"ADR-002"` (actual `ADR-002` is \*\*Regional Topology and Mexico Data Residency\*\*, while Kinesis vs. MSK is in `ADR-004`)\[cite: 12, 14, 16]; (3) Coverage Map Row 18 states Glue vs. EMR Serverless is in `"ADR-004"` (actual `ADR-004` is \*\*Ingestion Patterns\*\*)\[cite: 12, 16]. | `part1-coverage-map.md` (Rows 7, 16, 18), `part1-service-rationale\_3.md` (Row 17), `ADR-001-finance-system-of-record.md` (Dependencies)\[cite: 12, 13, 19] | Direct inspection of attached files: `ADR-001` = Finance System of Record\[cite: 13]; `ADR-002` = Regional Topology \& Mexico Residency\[cite: 14]; `ADR-003` = TALON Export-Control Isolation\[cite: 15]; `ADR-004` = Multi-Modal Ingestion Patterns\[cite: 16]; `ADR-005` = Finance \& Plant-to-P\&L Query Serving\[cite: 17]. | Update cross-references in `part1-coverage-map.md`, `part1-service-rationale\_3.md`, and `ADR-001` so that `ADR-001` = SoR/Iceberg, `ADR-002` = Mexico Residency, `ADR-003` = TALON Isolation, `ADR-004` = Ingestion/Kinesis, and `ADR-005` = Redshift Serverless Serving\[cite: 12, 13, 14, 15, 16, 17, 19]. |**

**| \*\*MEDIUM\*\* | \*\*Check 2 \& Check 6 — Diagram Arrow `e10` vs. `ADR-002` on QRO Machine Telemetry Path:\*\* In `part1-logical-architecture.drawio`, Arrow `e10` (`b\_mexico -> i5\_transfer`) is labeled `"10. De-identified GL \& Telemetry Only"` routing into `AWS Transfer Family (SFTP) \& AWS Lambda + EventBridge` (`i5\_transfer`)\[cite: 18, 19]. However, `ADR-002` (Consequences section) states: \*"QRO telemetry crosses to US IoT Core (Arrow 1), so identifiers must be stripped at the plant edge (Greengrass). Arrow 10 should carry only GL rollups."\*\[cite: 14] | `part1-logical-architecture.drawio` (`e10`), `part1-service-rationale\_3.md` (`b\_mexico`, `i5\_transfer`), `ADR-002-regional-residency.md`\[cite: 14, 18, 19] | `ADR-002` notes that `AWS IoT Core` is run in the US region (not `mx-central-1`), meaning QRO shop-floor Greengrass gateways (`s1\_edge`) stream MQTT telemetry via Arrow `e1` to US `AWS IoT Core` (`i1\_iot`) after stripping operator IDs at the edge, while Arrow `e10` carries de-identified QRO ERP GL rollups via SFTP/Lambda (`i5\_transfer`)\[cite: 14, 18, 19]. | Align `part1-logical-architecture.drawio` (`e10`) and `part1-service-rationale\_3.md` with `ADR-002` by clarifying that QRO real-time MQTT telemetry flows through `s1\_edge -> i1\_iot` (`e1`) with operator PII stripped at Greengrass, while `e10` (`b\_mexico -> i5\_transfer`) carries de-identified QRO financial GL rollups and batch operational summaries\[cite: 14, 18, 19]. |**

**| \*\*MEDIUM\*\* | \*\*Check 1 \& Check 2 — Composite Diagram Boxes (`i4\_dms`, `i5\_transfer`) Require Sub-Service Routing Clarity on Arrows `e7` and `e25`:\*\* (1) Arrow `e7` (`s3\_cmm -> i4\_dms`, `"7. CMMS API / CMM CSV"`) routes both SaaS CMMS API data and CMM CSV inspection files into `i4\_dms` (`AWS DMS \& Amazon AppFlow`), whereas `i5\_transfer` (`AWS Transfer Family SFTP \& Lambda`) also claims `"SFTP: ERP GL/AP/Budget \& CMM CSV"`\[cite: 18, 19].Neither DMS nor AppFlow ingests on-prem CMM CSV exports—Transfer Family SFTP (`i5\_transfer`) or S3 file drop does\[cite: 16, 19]. (2) Arrow `e25` (`l1\_raw -> in3\_kb`, `"25. SOP Documents Index"`) connects `S3 Raw` directly to `Bedrock Knowledge Bases` without depicting the required vector store (e.g., Amazon OpenSearch Serverless or Aurora PostgreSQL `pgvector`)\[cite: 18, 19]. | `part1-logical-architecture.drawio` (`s3\_cmm`, `i4\_dms`, `i5\_transfer`, `e7`, `e25`), `part1-service-rationale\_3.md` (#3, #10, #11, #26)\[cite: 18, 19] | AWS DMS replicates databases; Amazon AppFlow polls supported SaaS APIs (native support for unnamed CMMS is rightly marked `UNVERIFIED`)\[cite: 16, 19]. Amazon Bedrock Knowledge Bases requires a backing vector store (such as OpenSearch Serverless or S3 Vectors / Aurora `pgvector`) to store embeddings generated from S3 objects. | In `part1-service-rationale\_3.md`, clarify that CMM CSV exports land via SFTP/S3 drop (`i5\_transfer` / `e7` logical grouping) while `i4\_dms` handles MES SQL CDC + CMMS API, and note the backing vector index inside `in3\_kb` (`Amazon Bedrock Knowledge Bases`)\[cite: 18, 19]. |**

**| \*\*MEDIUM\*\* | \*\*Check 2 — Directionality of Tool-Call Arrow `e27` (`sv1\_redshift -> in4\_agentcore`) and `e28` (`sv4\_dynamodb -> in4\_agentcore`):\*\* In `part1-logical-architecture.drawio`, arrows `e27` (`"27. Text-to-SQL Tool (Gateway)"`) and `e28` (`"28. Health State \& Memory"`) point uni-directionally from `Redshift Serverless` (`sv1\_redshift`) and `DynamoDB` (`sv4\_dynamodb`) into `Amazon Bedrock AgentCore` (`in4\_agentcore`)\[cite: 18]. Technically, `AgentCore Runtime` initiates the request through `AgentCore Gateway` (AWS Lambda MCP tool) \*to\* Redshift (via Redshift Data API) and DynamoDB, and the data flows back in the response\[cite: 17, 19]. | `part1-logical-architecture.drawio` (`e27`, `e28`), `part1-service-rationale\_3.md` (#20, #23, #27)\[cite: 18, 19] | In data-flow diagrams, arrows pointing from `Serve` to `Insight` correctly show \*what data flows\* (query results and health state flowing into the agents), whereas control-flow invocation goes from `in4\_agentcore` -> `sv1\_redshift` / `sv4\_dynamodb`\[cite: 18, 19]. | No XML structural change needed for a data-flow diagram, but state in the walkthrough text (and `ADR-005` Verification #3) that arrows `e27` and `e28` represent \*\*data flow (returned query results/state)\*\* triggered by request-response MCP tool calls from `AgentCore Gateway` (`startArrow=classic` bi-directional optional)\[cite: 17, 18, 19]. |**

**| \*\*LOW\*\* | \*\*Check 6 — Mexican Privacy Law Citation Discrepancy Between Baseline and `ADR-002`:\*\* `Architecture decision baseline\_5.docx` (Section 3.3) and `part1-service-rationale\_3.md` (`b\_mexico`) cite the \*"Mexican Federal Law on Protection of Personal Data Held by Private Parties (LFPDPPP)"\* without qualification\[cite: 11, 19], whereas `ADR-002` (Dependencies \& Facts Verified sections) notes that \*"Mexico's data-protection law was replaced in March 2025"\* and marks legal interpretations with `\*\*\[LEGAL]\*\*`\[cite: 14]. | `Architecture decision baseline\_5.docx` (Section 3.3), `part1-service-rationale\_3.md` (`b\_mexico`), `part1-coverage-map.md` (Row 3), `ADR-002-regional-residency.md`\[cite: 11, 12, 14, 19] | Mexico enacted a reformed Federal Law on Protection of Personal Data Held by Private Parties in March 2025; moreover, project instructions prohibit asserting legal conclusions\[cite: 14]. | Attribute the in-country QRO payroll/HR residency rule strictly to the \*\*project specification mandate (Binding Decision 8)\*\* rather than asserting statutory legal conclusions in the baseline and rationale\[cite: 11, 14, 19]. |**

**| \*\*LOW\*\* | \*\*Check 8 — Baseline Internal Contradiction on Cimarron Composites Shop-Floor Ingestion (`DEC-01` vs. `DEC-07`):\*\* In `Architecture decision baseline\_5.docx`, Decision 1 (`DEC-01`) states that \*"All raw ingestion from physical plants... terminates in AWS"\*\[cite: 11], whereas Decision 7 (`DEC-07`) states that \*"Cimarron's shop floor is modeled as a native Azure IoT Operations edge connecting directly to Azure Event Hubs and Microsoft Fabric Real-Time Intelligence"\*\[cite: 11] (and Section 5 Question 4 asks the professor which of the two is expected)\[cite: 11]. | `Architecture decision baseline\_5.docx` (Section 1 `DEC-01` vs. `DEC-07`, Section 5 Q4)\[cite: 11] | `part1-logical-architecture.drawio` (`i6\_cimarron`) and `ADR-004` properly handle Cimarron in Part 1 as pre-sized 12+1 architectural headroom while deferring the active Azure edge vs. AWS IoT Core routing question to the professor\[cite: 16, 18, 19]. | Add a qualifier to `DEC-01` in `Architecture decision baseline\_5.docx` stating `"for all 12 core AeroForge plants (with Cimarron 13th-site edge routing subject to DEC-07 / Part 2 multi-cloud coexistence rules)"`\[cite: 11]. |**

**| \*\*LOW\*\* | \*\*Check 4 — SageMaker Batch Predictions Writing Directly to Authoritative Gold (`e20`):\*\* In `part1-logical-architecture.drawio`, Arrow `e20` (`sv3\_sagemaker -> l5\_gold`, `"20. Batch Predictions \& SHAP"`) writes ML inference outputs (work-order scrap risk scores, titanium/energy forecasts, SHAP variance attributions) directly into the `Amazon S3 — Gold Zone`\[cite: 18, 19]. Under `ADR-001`, `l5\_gold` is the SOX-governed financial System of Record\[cite: 13]. | `part1-logical-architecture.drawio` (`e20`), `part1-service-rationale\_3.md` (`sv3\_sagemaker`), `ADR-001-finance-system-of-record.md`\[cite: 13, 18, 19] | Writing stochastic ML predictions into the Gold Zone is valid only if they are written to \*\*separate ML prediction/attribution Iceberg tables\*\* (e.g., `prov::fact\_ml\_predictions`, `prov::fact\_variance\_shap`) rather than mutating audited ERP GL/AP actual columns\[cite: 13, 19]. | Explicitly note in Deliverable 3 (Data Model) that Arrow `e20` writes to dedicated Gold ML/SHAP Iceberg tables (`prov::fact\_ml\_\*`) and never overwrites immutable ERP financial actuals in `l5\_gold`\[cite: 13, 18, 19]. |**

