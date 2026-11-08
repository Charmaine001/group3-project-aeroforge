# ADR-005: Finance & Plant-to-P&L Query Serving Architecture

**Status:** Draft – Proposed for Part 1  
**Date:** November 2026  
**Related Decisions:** DEC-01, DEC-02, DEC-03, DEC-05, DEC-06, DEC-08, DEC-09, DEC-10

## 1. Context

AeroForge needs a reliable analytical serving layer for finance and plant operations. The current month-end close takes nine business days, and plant controllers need to trace scrap, downtime, and titanium Purchase Price Variance (PPV) to machines, work orders, suppliers, and invoices.

Under DEC-01 and DEC-02, AWS is the enterprise finance System of Record, and Amazon S3 Gold (`l5_gold`) contains the authoritative Plant-to-P&L star schema in Apache Iceberg format. DEC-03 requires the serving layer to query these Gold tables without creating another authoritative copy.

The serving architecture must support Amazon QuickSight dashboards, operational reporting, and text-to-SQL queries initiated through Amazon Bedrock AgentCore. It must also meet the daily 07:00 CT finance reporting target, hourly operational dashboard refreshes, and the Scenario A pilot budget ceiling of $18,000 per month.

## 2. Options Considered

**Option 1 – Amazon Redshift Serverless with Amazon Athena (Selected)**

Use Redshift Serverless as the primary analytical serving warehouse. It queries S3 Gold Iceberg tables through the AWS Glue Data Catalog and can provide governed SQL views and materialized views for frequently used reporting queries. Use Athena for ad-hoc lake exploration and data-quality checks.

- **Advantages:** Automatically scales compute capacity with workload demand; supports SQL analytics over external Iceberg tables; separates the authoritative S3 data from the serving layer.
- **Trade-offs:** RPU consumption and Redshift Managed Storage must be monitored. Materialized views require refresh and security planning.

**Option 2 – Amazon Athena Only**

Use Athena for finance reporting, dashboards, data-quality queries, and AI-agent queries.

- **Advantages:** Serverless SQL directly over S3; pay-per-query pricing for standard queries; no separate warehouse to operate.
- **Trade-offs:** Query performance and concurrency must be tested against complex, frequently repeated star-schema queries. Athena workgroups and capacity reservations can manage workload isolation and concurrency, but reserved capacity adds cost and requires capacity planning.

**Option 3 – Provisioned Amazon Redshift RA3**

Use a provisioned Redshift cluster with managed storage and Redshift Spectrum for external S3 data.

- **Advantages:** Dedicated, configurable warehouse capacity and predictable performance for steady workloads.
- **Trade-offs:** Requires cluster sizing and ongoing capacity management. Provisioned capacity may be less cost-effective for a variable pilot workload, although actual cost depends on utilization and sizing.

## 3. Decision

Select **Option 1: Amazon Redshift Serverless as the primary finance and Plant-to-P&L serving warehouse, complemented by Amazon Athena**.

Redshift Serverless (`sv1_redshift`) will query the authoritative S3 Gold Iceberg tables (`l5_gold`) through the AWS Glue Data Catalog. Athena (`sv2_athena`) will support ad-hoc exploration, data-quality checks, and quarantine-zone analysis (`l3_quarantine`).

The S3 Gold Iceberg tables remain the authoritative source. Any Redshift materialized views will be derived, refreshable reporting structures—not independent systems of record—and must comply with DEC-03.

## 4. Rationale and Trade-Offs

Redshift Serverless provides a SQL warehouse for recurring finance and operational queries while scaling compute capacity as workload demand changes. It supports querying external Iceberg tables and creating materialized views on supported external data-lake tables.

Materialized views can improve repeated reporting queries, but they are not automatically rewritten or automatically created for data-lake tables. Refresh behavior and limitations must be considered when designing them.

Redshift Serverless compute is billed by RPU usage, with a 60-second minimum charge, and Redshift Managed Storage is billed separately. S3 data-lake queries through Redshift Serverless do not incur a separate Spectrum per-TB scan charge. Cost controls and workload testing are still required to demonstrate that the pilot remains below $18,000 per month.

Athena remains useful for flexible exploration and data-quality workloads. However, relying on Athena alone would require additional performance and concurrency testing for the recurring finance dashboards and complex reporting queries.

## 5. Consequences and Implementation Considerations

- **Data integrity:** Canonical facts and dimensions remain in S3 Gold as Apache Iceberg tables. Redshift Serverless and Athena query the same governed data.
- **Performance:** Use materialized views only for high-value, frequently repeated reporting queries after validating refresh behavior and query performance.
- **Security:** Configure AWS Lake Formation permissions and Redshift access controls for sensitive data, including TALON-related restrictions (DEC-09) and plant-level access requirements (DEC-08). Do not assume that permissions automatically propagate to every dashboard, materialized view, or AI-agent query.
- **Identity:** Configure and test IAM Identity Center trusted identity propagation where supported. QuickSight-to-Redshift identity propagation has configuration and dataset limitations. The proposed AgentCore-to-Redshift access path must be validated separately.
- **Cost:** Configure Redshift Serverless base and maximum RPU capacity, usage limits, monitoring, and alerts. Include storage, data transfer, and other applicable service costs in the pilot estimate.
- **Dependencies:** AWS Glue ETL must complete currency normalization (MXN to USD), data-quality checks, and Iceberg maintenance before the 07:00 CT reporting target. Final fact and dimension definitions depend on the approved Data Pack and `DATA_DICTIONARY.md` (DEC-10).

## 6. Open Items to Validate

1. Confirm materialized-view creation and refresh behavior for the project's Iceberg tables, including the impact of Lake Formation row- and column-level filters. Do not use materialized views for restricted data until the security model is validated.
2. Test Redshift Serverless query performance and concurrency using representative finance and plant workloads.
3. Validate the AgentCore Gateway integration path to Redshift, including authentication, authorization, and audit logging. Do not assume caller identity is automatically propagated.
4. Complete the Scenario A AWS Pricing Calculator estimate. Confirm the expected daily active hours, base RPU capacity, maximum RPU capacity, storage, and other service costs with the project team.
5. Confirm that the selected S3 Iceberg catalog and table implementation is compatible with the planned Part 2 Azure access pattern.

## 7. References

- Amazon Redshift – Referencing Iceberg tables: https://docs.aws.amazon.com/redshift/latest/dg/referencing-iceberg-tables.html
- Amazon Redshift – Materialized views on external data lake tables: https://docs.aws.amazon.com/redshift/latest/dg/materialized-view-external-table.html
- Amazon Redshift – Serverless billing: https://docs.aws.amazon.com/redshift/latest/mgmt/serverless-billing.html
- Amazon Redshift – Lake Formation integration: https://docs.aws.amazon.com/redshift/latest/dg/spectrum-lake-formation.html
- Amazon Athena – Capacity reservations: https://docs.aws.amazon.com/athena/latest/ug/capacity-management.html
- Amazon QuickSight – Redshift trusted identity propagation: https://docs.aws.amazon.com/quick/latest/userguide/enabling-access-redshift.html

