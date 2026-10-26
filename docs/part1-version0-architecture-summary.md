# AeroForge Part 1: Version 0 Architecture Summary

Status: DRAFT. Provisional because the Data Pack and DATA_DICTIONARY.md have not been provided.

## The five-box journey

Sources -> Ingest -> Lake -> Serve -> Insight

| Box | Primary AWS service | Role |
|---|---|---|
| 1. Sources | AWS IoT Greengrass (plant edge) | Plant systems (MES, CMMS, ERP, quality/CMM, machine gateways) feed the platform. Greengrass buffers and forwards machine data at the plant edge. |
| 2. Ingest | Amazon Kinesis Data Streams (on-demand) | Takes in machine telemetry, with AWS IoT Core as the MQTT front door. Batch sources use DMS (MES change data), Transfer Family (ERP files), and AppFlow (CMMS API). |
| 3. Lake | Amazon S3 (raw, curated, gold) | Stores data in three zones. The gold zone uses Apache Iceberg and is the single authoritative source for finance and Plant-to-P&L numbers. |
| 4. Serve | Amazon Redshift Serverless | Serves SQL for controllers and BI by reading the gold Iceberg tables in place. Athena and SageMaker also read from the lake. |
| 5. Insight | Amazon QuickSight | Dashboards and alerts, plus AI agents (Amazon Bedrock) that answer questions over governed views. |

## Main data flow
Machines and enterprise systems -> edge/ingest services -> S3 raw -> cleaned in curated -> conformed in gold (Iceberg) -> read by Redshift, Athena, and ML -> dashboards, alerts, and AI agents.

## How S3 gold stays authoritative
Finance and Plant-to-P&L numbers are written once, into the S3 gold Iceberg tables. Redshift reads them through an external schema, and Part 2 platforms (Fabric, Snowflake) read the same files through shortcuts or external tables. Anything built on top of them (materialized views, caches) is derived and can be rebuilt, so no separate authoritative copy exists.

## The 20,000-message/second burst
The stream is sized for the 20,000 msg/s peak (about 42 times the 480 msg/s average), not just 10 times average. At about 1 KB per message that is about 20 MB/s, so Kinesis on-demand mode (or about 20 provisioned shards) absorbs shop-floor spikes without throttling. This conflicts with the "10x bursts" requirement, and we chose the larger number.

## Riskiest box: Lake
Every other box depends on the gold tables. Redshift and the Part 2 platforms must all read them correctly, and cross-cloud reads from Azure would add AWS egress costs and latency. A mistake in the gold zone's format or permissions affects finance, analytics, and AI at once.

## Open question
Is TALON (export-controlled) data acceptable in commercial US AWS regions with strict key and access controls, or does it require an isolated government region such as AWS GovCloud?

## AI-use disclosure
Drafted with AI assistance (Claude, Gemini, ChatGPT) from the project documents and the professor's answers. [Team to confirm how outputs were verified against AWS documentation.]