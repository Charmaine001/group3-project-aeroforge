# ADR-001: Finance System of Record

**Status:** Draft | **Date:** 2026-10-27 | **Scope:** Part 1 (AWS), with read-side rules for Part 2

## Context
Official P&L and Plant-to-P&L numbers need one authoritative source, or close stays slow and variances untraceable. Part 2 adds Fabric, Databricks and Snowflake (the CFO's finance-warehouse choice); the CIO's rule is "no third copy". The professor said to pick one platform for official numbers and keep that rule. *Assumption (confirm):* the on-prem ERP stays the book of original entry; "system of record" here means the authoritative analytical record.

## Requirements and constraints
- One authoritative store; SOX audit trail and as-of queries; gold ready by 07:00 CT; 7-year finance retention.
- Part 2 reads the same data in place. Mexico payroll/HR stays in Mexico; TALON stays in US regions.
- Streaming sized for 20,000 msg/s; Scenario A (3 plants) and B (12 plants). Data Pack and DATA_DICTIONARY not provided, so schemas are provisional.

## Options considered
1. **AWS: S3 gold Iceberg + Redshift Serverless.** Pros: ingest, Glue, Lake Formation and ML already on AWS; Iceberg snapshots allow as-of reads (Athena/Spark); open readers. Cons: Part 2 reads cross clouds (egress, latency).
2. **Snowflake.** Pros: CFO preference; masking, row access, sharing. Cons: Snowflake-managed storage would copy AWS-landed data; external Iceberg storage is still AWS; Part 1 already requires an AWS warehouse, so two compete.
3. **Fabric/OneLake.** Pros: Direct Lake, Power BI, Fabric IQ. Cons: needs data moved into OneLake (a copy) or the AWS plant platform reversed; Fabric IQ ontology still preview.

## Decision (binding)
S3 gold Iceberg tables (Glue Data Catalog, Lake Formation) are the single authoritative source. Redshift Serverless is the AWS serving engine over them. Part 2 reads in place.

## Rationale
**What counts as a copy:**

| Class | Test | Examples | Status |
|---|---|---|---|
| Authoritative | Official numbers originate here | Gold Iceberg (Parquet + metadata) | Exactly one |
| Copy | Persisted, queryable replica of gold rows at the same grain | Redshift local table via INSERT/CTAS; Delta in OneLake/ADLS; Snowflake-managed Iceberg; Power BI Import | Prohibited |
| Derived | Persisted coarser-grain result, rebuildable, labelled non-authoritative | Redshift aggregate MV | Allowed, controlled |
| Metadata/cache | Pointers, stats, in-memory or time-limited caches; never a table source | Virtual `_delta_log`, Direct Lake memory, shortcut cache (1-28 days) | Allowed |

An MV that merely selects gold rows is a copy. **Redshift tables loaded from gold are a third copy.** The pattern that keeps the rule: an external schema on the Glue catalog (Redshift queries Iceberg in place) plus aggregate-only MVs.

**Part 2 in place:** Fabric uses a read-only S3 shortcut to the Iceberg folder; OneLake generates virtual Delta metadata, Parquet stays in S3. Snowflake uses a catalog integration to an externally managed Iceberg table; files stay in S3. Both avoid a copy only if nobody runs Copy jobs, Import mode, CTAS or conversion on gold.

**Egress/latency:** zero-copy storage is not zero-cost compute. Uncached cross-cloud scans are billed by AWS as egress and add latency. Mitigate with compact gold marts, partition pruning, shortcut caching, Direct Lake, and Azure capacity near the AWS region.

## Consequences and trade-offs
Part 2 cost and speed depend on AWS egress. Gold commits must be 2+ minutes apart for Fabric conversion (nightly/hourly loads fit). Fabric cannot time-travel older snapshots; as-of audits run in Athena/Spark.

## Security and cost implications
- The Fabric S3 shortcut uses one long-lived IAM-user key for all its users; CloudTrail shows the key, not the person. Scope it read-only and by prefix; enforce per-user access in Fabric.
- Snowflake can write to externally managed Iceberg, so read-only must come from IAM and Lake Formation.
- External-engine setup requires full-table access, so row filters likely cannot protect TALON. Keep TALON and Mexico HR data in separate, unexposed tables; exposure is the compliance owner's call.
- Redshift MVs ignore later Lake Formation revocations; no MVs over restricted data.
- Cost: cross-cloud GB scanned × AWS egress rate, tagged per consumer, for Scenarios A and B.

## Dependencies and unresolved questions
- Is the ERP the book of original entry?
- Where do Cimarron's ledger and Snowflake consolidation outputs live?
- S3 Tables or self-managed Iceberg (ADR-003)? Fabric shortcuts need a general-purpose bucket.
- Snapshot retention versus 7-year as-of queries.

## Facts verified or UNVERIFIED
**Verified (official docs):** OneLake virtualizes S3 Iceberg as Delta (Parquet only, latest metadata only, no Private Link); S3 shortcuts are read-only, IAM-user key, optional cache; Snowflake supports cross-cloud external-catalog Iceberg, egress billed by storage provider; Redshift queries Iceberg in place, supports MVs, no Iceberg time travel; Direct Lake caches columns in memory.
**UNVERIFIED:** shortcuts to S3 Tables buckets; Lake Formation row filters for Fabric/Snowflake reads; the ~$0.09/GB egress rate (third-party sources only); Snowflake auditor sharing of external Iceberg; Databricks-on-Azure reading AWS gold; Redshift writing Iceberg.

## AI-use disclosure
Drafted with Claude (Anthropic), 2026-10-27, to cross-check the Gemini-written baseline against the project documents and vendor docs. Team must re-verify UNVERIFIED items. Not legal advice.
