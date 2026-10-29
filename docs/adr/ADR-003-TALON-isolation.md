# ADR-003: TALON Export-Control Isolation Architecture

* **Status:** Draft (For Nov 1 Part 1 Submission & Nov 2 Class Review)
* **Date:** October 2026
* **Related Baseline Decisions:** DEC-01, DEC-02, DEC-03, DEC-06, DEC-09, DEC-10

---

## 1. Title and Status
**ADR-003: TALON Export-Control Isolation Architecture** — **Status: Draft**

## 2. Context
AeroForge Precision Components manufactures structural aircraft parts across three programs: SKYLARK (commercial business jet), MERIDIAN (regional jet), and TALON (a military trainer program for Cobalt Defense Systems). Under the CEO's "Plant-to-P&L Digital Thread" initiative, machine telemetry (`s1_edge`), MES production orders (`s2_mes`), CMM quality inspections (`s3_cmm`), CMMS maintenance orders (`s3_cmm`), and ERP financial ledgers (`s4_erp`) across Wichita (WIC), Oklahoma City (OKC), and Querétaro, Mexico (QRO) flow into an AWS lakehouse (`l1_raw`, `l4_curated`, `l5_gold`). Because TALON technical and operational records are export-controlled (ITAR/EAR context), AeroForge must isolate TALON data inside U.S. regions and restrict access to verified U.S. persons while still answering cross-domain operational and financial questions—such as tracing which TALON parts were machined on `WIC-CNC-03` during a spindle vibration alert.

## 3. Requirements and Constraints
* **U.S. Residency & Person Restriction (DEC-09):** TALON data must physically reside only in U.S. AWS Regions (`us-east-1` primary, `us-west-2` DR) and be accessible strictly to verified U.S. persons.
* **Single Authoritative Gold Zone (DEC-01, DEC-02, DEC-03):** The Amazon S3 Gold Zone (`l5_gold`, Apache Iceberg) is the single authoritative System of Record read in place by `Amazon Redshift Serverless` (`sv1_redshift`) and zero-copy by Part 2 platforms (`l6_part2_hook`).
* **AI Entitlement Propagation:** Multi-agent workflows running on **Amazon Bedrock AgentCore** (`in4_agentcore`, which is a distinct AWS offering from declarative Amazon Bedrock Agents) and **Amazon Bedrock Knowledge Bases & Guardrails** (`in3_kb`) must enforce caller entitlements.
* **Budget Ceiling (DEC-06):** Must operate within the `<$18,000/month` AWS Scenario A pilot budget.
* **Provisional Schema (DEC-10):** Because the AeroForge Data Pack and `DATA_DICTIONARY.md` are unprovided, program-filtering attributes (e.g., `prov::program = 'TALON'`) are marked `PROVISIONAL`.

## 4. Options Considered

### Option 1: Dedicated Account/Buckets, KMS CMKs, and Lake Formation ABAC in Commercial U.S. Regions (`us-east-1` / `us-west-2`)
Isolate TALON data within commercial U.S. regions using dedicated S3 buckets/prefixes (`b_talon`) or a dedicated workload account under `AWS Organizations` (`cc2_sec`), encrypted with a dedicated `AWS KMS` Customer Managed Key (CMK), and governed via `AWS Lake Formation` Tag-Based Access Control (`LF-TBAC`) and row/column filters (`cc3_gov`).
* **Pros:** Preserves a unified Plant-to-P&L Star Schema in `l5_gold` and `sv1_redshift`; provides full access to `Amazon S3 Tables`, `Amazon Redshift Serverless`, and `Amazon Bedrock AgentCore`; avoids duplicate infrastructure costs.
* **Cons:** Relies on logical/cryptographic policy enforcement rather than physical government-cloud partitioning; commercial U.S. region acceptability for specific defense contracts requires instructor/compliance confirmation (`UNVERIFIED`).

### Option 2: Physical Enclave in AWS GovCloud (US) (`us-gov-west-1` / `us-gov-east-1`)
Route all TALON shop-floor telemetry, CMM measurements, and ERP records into an isolated `AWS GovCloud (US)` partition.
* **Pros:** Purpose-built sovereign U.S. enclave operated exclusively by U.S. citizens for strict ITAR workloads.
* **Cons:** Fractures the single authoritative S3 Gold Zone across two disconnected AWS partitions (commercial vs. `aws-us-gov`), complicating consolidated Plant-to-P&L queries; increases fixed monthly costs; availability of **Amazon Bedrock AgentCore** and cross-cloud Part 2 OneLake/Snowflake zero-copy shortcuts from GovCloud is constrained or `UNVERIFIED`.

### Option 3: Keep TALON Out of the Shared Cloud Platform (On-Premises Only)
Leave all TALON MES, CMM, and sensor data on local servers at the Wichita and Oklahoma City plants.
* **Pros:** Eliminates cloud export-control scope entirely.
* **Cons:** Breaks the CEO's 24-hour Plant-to-P&L Digital Thread mandate and fails the required Part 1 AI agent question (*"Which TALON parts were machined on WIC-CNC-03 while its vibration was in the Alert band?"*).

## 5. Decision
We select **Option 1**—isolating TALON data within Commercial U.S. Regions (`us-east-1` primary, `us-west-2` DR) using dedicated S3 prefixes/buckets (`b_talon`), a dedicated `AWS KMS` CMK, `IAM Identity Center` attribute-based access control (`isUSPerson = true`), and `AWS Lake Formation` row/column security (`cc3_gov`). Whether TALON requires **AWS GovCloud (US)** versus commercial U.S. regions is documented as an open architectural question for the professor rather than a legal conclusion.

## 6. Rationale
Option 1 is the only design that satisfies all project requirements simultaneously: it keeps TALON data inside U.S. regions (`b_talon`), prevents Mexican (`mx-central-1`) or non-U.S.-person identities from decrypting or querying TALON records, maintains a single authoritative Apache Iceberg Gold Zone (`l5_gold`), and enables **Amazon Bedrock AgentCore** (`in4_agentcore`) to answer `WIC-CNC-03` maintenance and quality questions for authorized U.S. engineers within the `<$18,000/month` pilot target.

## 7. Consequences and Trade-Offs
* **Unified Analytics with Dynamic Masking:** Authorized U.S. persons see full TALON operational and financial detail across `sv1_redshift`, `sv2_athena`, and `in1_quicksight`, while non-U.S. persons (e.g., QRO controllers) have `prov::program = 'TALON'` rows or sensitive technical columns filtered out by Lake Formation.
* **Cross-Cloud Restriction:** Part 2 zero-copy hooks (`l6_part2_hook` via Microsoft OneLake S3 shortcuts and Snowflake External Volumes) are blocked at the S3 bucket/KMS policy layer from reading TALON prefixes unless the target Azure/Snowflake tenant independently enforces U.S.-person controls.

## 8. Security and Cost Implications
* **Security (`cc2_sec`, `cc3_gov`):** `AWS Control Tower` Service Control Policies (SCPs) block TALON replication outside `us-east-1`/`us-west-2`; a dedicated FIPS-backed `AWS KMS` CMK requires IAM principal tag `isUSPerson = true` for `kms:Decrypt`; **AgentCore Identity** propagates user identity to MCP Lambda tools (`in4_agentcore`) while **Bedrock Guardrails** (`in3_kb`) block unauthorized TALON outputs; `AWS CloudTrail` and `Amazon Macie` log and audit all access.
* **Cost (`cc5_cost`):** Lake Formation tag/row filtering incurs no extra license cost, and a dedicated KMS CMK adds minimal monthly overhead (~$1/month per key plus per-10K API request fees, `UNVERIFIED` exact usage volume), preserving the `<$18,000/month` Scenario A budget.

## 9. Dependencies and Unresolved Questions
* **Dependencies:** Requires `IAM Identity Center` (`cc2_sec`) populated with verified `isUSPerson` identity attributes, Lake Formation data cell filters (`cc3_gov`), and final schema mapping of `prov::program` once `DATA_DICTIONARY.md` is provided.
* **Open Question for the Professor:** For grading purposes, is commercial U.S. region isolation (`us-east-1` / `us-west-2` with KMS CMKs and Lake Formation ABAC) sufficient for TALON, or should we assume an `AWS GovCloud (US)` enclave? (Framed as a course-scoping question, not a legal determination.)

## 10. Facts to Verify Against Official AWS Documentation
1. **Lake Formation Data Cell Filters on Apache Iceberg (`Verified` capability, `UNVERIFIED` exact DDL syntax):** Verify exact configuration for enforcing row-level filters (`prov::program = 'TALON'`) across `Amazon Redshift Serverless` (`sv1_redshift`) external Iceberg schemas and `Amazon Athena` (`sv2_athena`).
2. **Amazon Bedrock AgentCore Identity Propagation (`UNVERIFIED`):** **Amazon Bedrock AgentCore** (`in4_agentcore`) is distinct from Amazon Bedrock Agents; verify the exact session-tag / IAM role-chaining mechanism used by **AgentCore Identity** and **AgentCore Gateway** when querying Redshift and Bedrock Knowledge Bases (`in3_kb`).
3. **AWS GovCloud (US) Feature Parity (`UNVERIFIED`):** Regional availability of **Amazon Bedrock AgentCore** and **Amazon S3 Tables** in `us-gov-west-1` / `us-gov-east-1` is `UNVERIFIED`.

## 11. AI-Use Disclosure
Drafted with generative AI assistance (Google Gemini) based on the MIS 612 Part 1 project brief, `docs/architecture-decision-baseline.md`, and `part1-service-rationale.md`. The team verified alignment with the 33-shape logical diagram, separated technical architecture from legal conclusions, and marked unconfirmed service behaviors as `UNVERIFIED` for validation against official AWS documentation.