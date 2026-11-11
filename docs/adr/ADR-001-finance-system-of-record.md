# ADR-001 Finance System of Record

**Status:** Proposed for team review | **Reviewed and revised by:** Wali | **Date:** 2026-11-11

## Context
AeroForge needs one authoritative analytical record for official finance and Plant-to-P&L reporting by 07:00 CT. The diagram puts gold Iceberg tables in S3 and queries them through Redshift Serverless. Part 2 must reuse the record without a third persisted copy of the same gold rows. We assume ERP remains the accounting book of original entry; the team must confirm that distinction. Finance evidence requires seven-year retention; Mexico payroll/HR and TALON stay within their approved boundaries.

## Options
1. **S3 gold Iceberg with Redshift serving:** Fits AWS ingestion, Glue and Lake Formation; separates durable tables from query compute. Cross-cloud reads add transfer cost and compatibility work.
2. **Snowflake authoritative warehouse:** Fits the CFO's preference. Loading gold into managed tables adds a replica; externally managed Iceberg can reuse S3 but leaves authoritative files there.
3. **Fabric authoritative lakehouse:** Fits Power BI and Part 2. Loading gold into OneLake adds a replica; compatible shortcuts need validation and do not make OneLake authoritative.

## Decision
Keep **S3 gold Iceberg, registered in Glue and governed through Lake Formation, as the authoritative analytical finance record**. Redshift Serverless serves approved queries through an external schema; ADR-005 owns engine selection. Writers reconcile gold to ERP and record reporting cutoff, batch and table snapshot IDs before finance approves a published close.

My trade-off is to prioritize consistent, traceable official numbers over each platform's fastest local storage. Prohibit same-grain persisted replicas such as gold CTAS/INSERT copies and BI imports. Allow only approved, rebuildable aggregates and bounded caches with documented ownership, expiry and access controls. Part 2 integrations must prove compatible format, read-only access and acceptable egress before adoption.

## Consequences
Cross-cloud latency and egress remain costs without duplicated tables. Enforce restrictions in each consumer. Do not expose TALON or Mexico payroll/HR through commercial shortcuts or agents; only approved exports enter shared gold. Iceberg snapshots alone do not guarantee seven-year history: preserve required metadata and referenced files under a tested retention policy. Object Lock protects object versions, not table consistency or accounting approval. Use an Iceberg-capable engine for historical audits; do not assume Redshift supports time travel. Confirm general-purpose S3 versus S3 Tables compatibility with the architect before Part 2 integration.

**AWS references:** [Iceberg support](https://docs.aws.amazon.com/redshift/latest/dg/querying-iceberg.html), [external schemas](https://docs.aws.amazon.com/redshift/latest/dg/referencing-iceberg-tables.html), [Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html). Need to review before acceptance.
