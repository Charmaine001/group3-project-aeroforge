# AeroForge: Plant-to-P&L Digital Thread

MIS 612 | Fundamentals of Cloud Computing | Fall 2026 | Project 2: Enterprise Multi-Cloud Solution

**Team:** [Group 3]

A three-part project for the fictional company AeroForge Precision Components:

| Part | What | Where in this repo |
|---|---|---|
| 1. AWS Enterprise Design | Lakehouse, Redshift, SageMaker, Bedrock AgentCore agents | `docs/`, `diagrams/`, `docs/adr/` |
| 2. Azure Enterprise Design | Fabric IQ ontology, Databricks, Snowflake, Foundry agents | `docs/`, `diagrams/`, `docs/adr/` |
| 3. AWS Sandbox Build | Working pipeline in the AWS Academy Learner Lab | `labs/` |

## Team and roles

| Role | Name | GitHub |
|---|---|---|
| Team Lead / Solution Architect | [Charmaine Aggson] | [@Charmaine001] |
| Data Engineer, Ingestion | [Name] | [@username] |
| Data Engineer, Processing & Warehouse | [Name] | [@username] |
| Data Scientist / ML | [Name] | [@username] |
| Security, FinOps & AI Agents | [Luke Youngdahl] | [@lyoungdahl] |
| AI Agents & Ontology | [Name] | [@username] |

Roles may rotate between Phase 1 (design) and Phase 2 (build).

## Repo layout

```
docs/            team charter, design write-ups, logs, AI-use statements
docs/adr/        architecture decision records (one page max each)
diagrams/        draw.io sources (.drawio) and exports (.png)
labs/lab00..12/  Part 3 build: code, notes, evidence per lab
```

## Key dates

| Date | Milestone |
|---|---|
| Mon Oct 19 | Team charter + service research assignments |
| Sun Nov 1 | Part 1 AWS design draft |
| Sun Nov 8 | Part 2 Azure design draft |
| Wed Nov 18 | Project 2 checkpoint (live demo: lake + streaming + Glue job 1) |
| Sun Nov 29 | Designs v2, findings log, report and repo frozen |
| Wed Dec 2 | Final submission and team demo |

## Ground rules

- **Never commit credentials**: no AWS access keys, secret keys, session tokens, or account IDs. See `.gitignore`.
- Commit from your own GitHub account with small, frequent commits and clear messages. Individual contribution is graded from commit history.
- Log every problem you solve in `docs/troubleshooting-log.md`.
- Record anything AI-assisted in `docs/ai-use-statement.md` (tool, what it was used for, how you verified it).
