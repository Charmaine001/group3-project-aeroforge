# AeroForge Part 1 — Medallion Architecture Design

## 1. Purpose

The AeroForge data lakehouse uses a Medallion Architecture on Amazon S3 to organize raw, curated, and business-ready data. The design supports manufacturing operations, predictive maintenance, quality analysis, and finance reporting across the three pilot plants, with scalability to 12 plants.

## 2. Bronze Zone — Raw Data

**Purpose:** Preserve source data in its original form for traceability, replay, and auditing.

**Datasets:**
- Machine telemetry from IoT gateways.
- MES production orders.
- Quality inspection measurements.
- CMMS maintenance work orders and technician notes.
- ERP general ledger, accounts payable, budget, and standard cost.
- Master data for plants, machines, parts, materials, suppliers, cost centers, and GL accounts.
- Weather, commodity prices, and foreign exchange data.
- SOPs and technical documents.

**Storage formats:** Original JSONL.gz, CSV, JSON, PDF, and Markdown formats.

**Processing:** Data is landed in source-specific S3 prefixes with ingestion timestamps, source identifiers, and batch or event metadata. Raw data is retained without overwriting the original records.

## 3. Silver Zone — Curated Data

**Purpose:** Create clean, consistent, validated datasets for analytics and downstream processing.

**Transformations performed using AWS Glue ETL:**
- Validate schemas and data types.
- Remove duplicate records using source identifiers and event timestamps.
- Handle missing and invalid values using documented data-quality rules.
- Standardize timestamps to a common timezone while retaining source timezone information.
- Normalize measurement units.
- Convert financial values into USD using approved exchange-rate records while retaining original currency and amount.
- Standardize plant, machine, work-order, part, and supplier identifiers.
- Validate referential integrity against master data.
- Separate invalid records into a quarantine area with rejection reasons.
- Apply incremental and repeatable ETL processing.

**Storage formats:** Apache Parquet for curated datasets and Apache Iceberg where transactional table management, schema evolution, and table-level consistency are required.

## 4. Gold Zone — Business-Ready Data

**Purpose:** Provide governed, conformed datasets for reporting, analytical queries, and AI-powered business insights.

**Datasets:**
- Production performance and work-order metrics.
- Machine utilization and downtime.
- Quality inspection, scrap, and rework metrics.
- Maintenance cost and equipment reliability.
- Supplier, material, invoice, and purchase price variance metrics.
- General ledger, budget, and financial variance datasets.
- Conformed Plant-to-P&L fact and dimension tables.

**Storage format:** Apache Iceberg tables stored in Amazon S3 and registered in the AWS Glue Data Catalog.

**Consumption:** Amazon Redshift Serverless serves the primary analytical warehouse workload. Amazon Athena supports ad-hoc data exploration and quality investigations. Amazon QuickSight provides dashboards and governed reporting.

## 5. Data Movement and Governance

AWS Glue ETL moves data from Bronze to Silver and from Silver to Gold. Data-quality failures are routed to quarantine instead of being silently discarded. AWS Glue Data Catalog provides table metadata, while AWS Lake Formation and IAM control access to governed datasets.

Restricted TALON data must remain isolated in approved US regions and accessible only to authorized users. Gold datasets and derived analytical outputs must follow the same data-classification and access-control requirements.

## 6. Design Principles

1. Preserve raw source data for traceability.
2. Make transformations repeatable and auditable.
3. Apply data-quality checks before publishing curated data.
4. Keep one authoritative S3 Gold dataset for the conformed analytical model.
5. Use open table formats to support future multi-cloud integration.
6. Partition large datasets according to common query filters, such as event date and plant, after validating workload needs.

**Note:** Final table names, columns, data types, and keys remain provisional until validated against the instructor-provided AeroForge Data Pack and DATA_DICTIONARY.md.
