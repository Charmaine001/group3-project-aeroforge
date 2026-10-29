# ADR-002: Regional Topology and Mexico Data Residency

**Status:** Draft | **Date:** 2026-10-29 | **Scope:** Part 1 (AWS) regions; residency rules for Part 2 (Azure)

## Context
The Querétaro (QRO) plant is one of three pilot plants. The project brief says Mexico payroll and HR data stays in-country (consider mx-central-1), and TALON stays in US regions (ADR-003). ADR-001 makes S3 gold Iceberg the single authoritative finance source. This ADR decides what may enter that US lakehouse from Mexico. This ADR is architecture, not legal advice; legal review points are marked **[LEGAL]**.

## Requirements and constraints
- QRO payroll/HR and other personal data stay in Mexico, on AWS and on Azure.
- QRO general-ledger rollups and operational data must still reach the US gold zone (MXN→USD close, Plant-to-P&L).
- Streaming sized for 20,000 msg/s; Scenarios A (3 plants) and B (12). Data Pack and DATA_DICTIONARY not provided, so classifications are provisional.
- Consistent with ADR-001: no third copy; Part 2 reads US gold only.

## Options considered
1. **US regions only.** Pros: one toolset, all services present. Cons: breaks the in-country requirement for payroll/HR.
2. **US primary plus in-country store in mx-central-1.** Pros: meets the requirement; Mexico footprint stays small; US lakehouse unchanged. Cons: second region/account to run; fewer services; no second Mexican region for DR.
3. **Everything in Mexico.** Pros: strongest residency story. Cons: many Part 1 services unconfirmed there; TALON must still be US; conflicts with ADR-001's US gold and Part 2's cross-cloud design; one Mexican region, no DR pair.

## Decision (binding)
Option 2. A dedicated AWS account in mx-central-1 holds Mexican-resident data; the US account holds everything else. An allow-listed export gate is the only path from Mexico to the US.

## Rationale
| Class | Examples (provisional) | Placement |
|---|---|---|
| Stays in Mexico | Payroll, HR records, employee/operator identifiers, timecards, free-text notes naming people, employee-level GL postings | mx-central-1 (AWS); Mexico Central (Azure) |
| May leave **[LEGAL]** | De-identified GL balances, standard-cost variance totals, plant-level aggregates, machine telemetry with identifiers stripped | US raw zone via export gate |

Nothing in the first class reaches gold, Redshift, OneLake or Snowflake in the US, so ADR-001's Part 2 read paths never touch it.

**AWS services.** Confirmed in mx-central-1 by AWS docs: Glue, Athena, Lake Formation, Transfer Family, IAM Identity Center, Redshift (provisioned), MWAA, Bedrock runtime endpoint. AWS says S3, KMS, Lambda, VPC and Kinesis Streams ship in every Region launch (I could not filter its table by Region). **Run in US regions:** IoT Core, Redshift Serverless, SageMaker, QuickSight, AgentCore, Macie, GuardDuty, Security Hub and Backup. I found no mx-central-1 entry for these, or the table was unreachable. Mexico runs only S3 + KMS CMK, Glue/Athena, Lake Formation and the export gate (Lambda/Glue).

**Azure (Part 2).** Mexico Central supports all Fabric workloads, Azure Databricks and Snowflake (separate account; one region each). If Part 2 needs Mexican-resident data, it lives only in Mexico Central. No OneLake shortcut from a US capacity to Mexican storage, because the shortcut cache would hold the bytes in the US for up to 28 days.

## Consequences and trade-offs
QRO telemetry crosses to US IoT Core (Arrow 1), so identifiers must be stripped at the plant edge (Greengrass). Arrow 10 should carry only GL rollups. Payroll analytics run in Mexico with a smaller toolset. Neither cloud has a second Mexican region, so DR for resident data is multi-AZ plus in-country backup **[LEGAL: cross-border backup]**.

## Security and cost implications
- SCP region allow-list: us-east-1, us-west-2, mx-central-1 (opt-in Region; enable explicitly). Bucket policies deny replication out of the Mexico account; CMK stays in mx-central-1.
- AWS says global services (IAM, Organizations, Direct Connect) may process data globally and metering/label data may leave the Region. Keep personal data out of resource names and tags.
- Cost: second-region footprint, VPN/Direct Connect to two regions, small rollup egress. Tag by plant; model for Scenarios A and B.

## Dependencies and unresolved questions
- **[LEGAL]** Basis for the in-country rule (brief versus statute; Mexico's data-protection law was replaced in March 2025). Do rollups, telemetry or pseudonymized data count as personal data?
- Where workforce identity (IAM Identity Center, Entra) and support metadata reside.
- Do telemetry or work orders carry operator IDs? Does QRO make TALON parts?
- Cimarron is out of scope here.

## Facts verified or UNVERIFIED
**Verified:** mx-central-1 GA, 3 AZs, opt-in; service endpoints listed above; Azure Mexico Central 3 AZs, no paired region, all Fabric workloads, Databricks and Snowflake supported; AWS global-service and metadata statement; new Mexican privacy law in force 21 March 2025 (law-firm sources only).
**UNVERIFIED:** the US-run services in mx-central-1; Bedrock model and cross-Region inference location; Fabric tenant and Entra data location; whether Snowflake Cortex or Fabric AI features process outside the Region; Azure Event Hubs/SQL availability.

## AI-use disclosure
Drafted with Claude (Anthropic), 2026-10-29, checking the baseline and service rationale against AWS, Microsoft and Snowflake docs. Team must re-verify UNVERIFIED items. Not legal advice.
