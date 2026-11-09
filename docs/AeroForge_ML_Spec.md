# AeroForge — Part 1 ML Design Specification

**Contributor:** Umar (Data Scientist / ML)  
**Status:** Draft for team review; not trained or validated  
**Scope:** ML half of the AI specification only. Wali owns the Bedrock AgentCore agent half.

## Context and architecture
AeroForge is a three-plant aerospace manufacturer (Wichita, Oklahoma City, Querétaro) planning to scale to twelve plants. The instructor reports 62% OEE (75% target), $11.6M annual unplanned downtime, cost of poor quality at 2.4% of revenue, and a nine-business-day financial close. The target platform collects OPC UA/MQTT machine telemetry, MES work orders, CMMS maintenance history, ERP/GL/budget/AP, and external weather/commodity/FX data. Proposed path: plant edge → Kinesis for telemetry and approved batch/CDC integrations for business systems → S3 bronze/silver/gold with Glue quality controls → governed features → SageMaker AI models → approved scores, dashboards, alerts and agent tools. Streaming critical alerts must reach users within 60 seconds; latency must be verified end-to-end. These are design proposals, not implemented pipelines.

**Data limitation:** The instructor-described Data Pack (24 pilot machines, September 2026 operations, July–September finance) has not been supplied. The exact columns, label availability, model performance, costs, and thresholds below are **assumptions pending verification**. No actual training, test results, or service availability checks are claimed.

## Five use cases

### 1. Predictive maintenance — failure within 48 hours
- **Question and label:** At timestamp *t*, will this machine have an unplanned failure starting in (t, t+48h]? Binary label 1/0 from CMMS failure start times; scheduled maintenance is not a failure. Exclude observations without a full 48-hour outcome window.
- **Features:** Past-only rolling vibration/temperature/load statistics, machine run hours, alarm counts, operating state, recent workload, machine type/age, and time since last *completed* maintenance. Join sensor readings to machine and CMMS IDs with event timestamps.
- **Leakage controls:** Build every feature as of *t*; never include later technician notes, failure codes, repair cost, or work-order completion. Use chronological train/validation/test periods with a 48-hour boundary gap; examine machine-level generalization separately.
- **Model and metric:** XGBoost binary classifier against a simple alarm-rule baseline; prioritize event recall at a manageable alert rate, report precision, PR-AUC, false alerts per machine-week, calibration, and warning lead time. Not raw accuracy (failures are rare).
- **Cost-based threshold:** Alert when estimated avoided failure loss × calibrated failure probability × expected intervention effectiveness exceeds inspection/intervention cost. The case describes one $50K repair, but that is not a universal failure cost. Set final thresholds with maintenance using downtime, scrap, labor, and alert-capacity costs; distinguish P1 from advisory alerts.
- **MLOps:** Time-aware feature pipeline; SageMaker Pipelines training/evaluation; Model Registry approval; scheduled/near-real-time scoring; CloudWatch/model and data-quality monitoring; monitor drift, calibration, false alarms and delayed CMMS labels; retrain on sustained deterioration or process changes; version and roll back models.
- **Business KPI:** Unplanned downtime, MTBF, maintenance spend.

### 2. Streaming anomaly detection
- **Question and output:** Is a sensor pattern unusual for the machine and its current operating state? Output anomaly score, timestamp and reason codes; no ground-truth label is required for unsupervised training, but historical technician-confirmed incidents support evaluation.
- **Features:** Past-only rolling vibration, temperature, spindle/load, deviations from normal ranges, rate of change, sensor cross-correlations, and machine state; normalize by machine/operating mode.
- **Leakage controls:** No centered rolling windows or future readings; chronological holdout of incidents; do not learn a normal baseline from known failure periods.
- **Model and metric:** Compare statistical/engineering rules with Random Cut Forest or Isolation Forest. Measure event-level recall, time-to-detect, false alerts per machine-day and end-to-end alert latency. Critical alerts must be delivered within the instructor's 60-second requirement.
- **Cost-based threshold:** Different score thresholds by machine type/operating state, selected from missed-event severity versus inspection and alarm-fatigue costs. Hard safety limits remain deterministic and do not depend on ML.
- **MLOps:** Kinesis → streaming computation (e.g., Flink) → rule/anomaly scoring → approved alert route (e.g., SNS). Verify service/endpoint throughput and latency in design review. Monitor data gaps, alert volume, detection delay and sensor replacement; recalibrate or retrain when normal behavior shifts.
- **Business KPI:** Detection delay, missed critical events, nuisance alerts.

### 3. Work-order scrap risk
- **Question and label:** At work-order start (or a clearly defined in-process checkpoint), will at least one unit in the order be scrapped? Binary label from final MES/quality outcome; optional second model estimates avoidable scrap dollars.
- **Features:** Part family, material/titanium grade, machine type, historical defect rate for comparable jobs, order quantity, prior machine health, process parameters available at prediction time, and material-batch/supplier information if present.
- **Leakage controls:** No final scrap quantity, final inspection, completed work-order duration, or future quality disposition as features. Define separate feature snapshots for start-of-order vs mid-order predictions. Use chronological evaluation and avoid leaking repeated order/part identifiers.
- **Model and metric:** XGBoost classifier compared with part-family historical rate. Report PR-AUC, precision/recall at inspection capacity, calibration and dollars of scrap risk captured in top flagged orders.
- **Cost-based threshold:** Inspect/intervene if predicted scrap probability × expected avoidable scrap value × intervention effectiveness exceeds inspection cost; higher-value titanium parts can justify a lower risk threshold. Obtain actual costs from operations before setting a numeric threshold.
- **MLOps:** Governed feature creation; SageMaker training/registry; score at agreed production checkpoints; monitor quality labels, false positives and performance by plant/part family; retrain after process/material shifts; retain approvals and rollback.
- **Business KPI:** Scrap dollars, cost of poor quality, rework rate.

### 4. Energy-use forecasting (chosen from energy OR titanium-cost forecasting)
- **Question and label:** Predict each plant's daily electricity consumption (kWh) for the next seven days; calculate projected spend separately from approved tariffs.
- **Features:** Historical kWh and lagged/rolling values, day-of-week, holiday calendar, planned production shifts and run hours, machine mix, and *weather forecasts available at prediction time*.
- **Leakage controls:** No realized future weather, future actual production, or future meter readings. Use rolling-origin backtests with prediction-time snapshots and compare against seasonal-naive forecasts.
- **Model and metric:** Seasonal-naive baseline versus lag-feature XGBoost regression; optimize WAPE/MAE and report forecast bias and estimated dollar error. Guard WAPE calculations against near-zero actual totals.
- **Cost-based threshold:** Notify controller when forecast cost exceeds approved budget by an amount greater than expected noise and response cost. No fixed dollar threshold is assumed without utility rates, intervention options and controller materiality guidance.
- **MLOps:** Daily batch feature generation/forecast; model registry and versioned forecasts; monitor WAPE, bias, missing readings and plant changes; retrain when error remains high across backtests or after major equipment/shift changes.
- **Business KPI:** Utility budget forecast accuracy and energy cost variance.

### 5. Financial variance driver analysis
- **Question and target:** For a plant/account/month, explain actual minus budget dollars and link meaningful drivers to work orders, downtime, scrap, supplier invoices and commodity/FX movements. This is **explainable analytics**, not necessarily a classification model.
- **Features/data:** Reconciled GL actuals and approved budget, AP invoice quantities/prices, material usage, MES output/scrap, CMMS repair/downtime, commodity index, and FX where applicable; use consistent currency, reporting period, and entity mapping.
- **Leakage controls:** For retrospective close commentary, use an explicitly versioned as-of close snapshot and distinguish later adjustments. For any forward prediction, only use data available at the prediction cutoff; never describe later postings as known earlier.
- **Method and metric:** First perform accounting-consistent decomposition (volume, price, mix, downtime, scrap) and reconcile contributions to total variance. Optional regression/SHAP ranks associations; SHAP does not establish causation. Report percentage/dollars reconciled, unexplained residual, controller-validated driver agreement, and time to draft commentary—not accuracy.
- **Cost-based threshold:** Prioritize investigation by materiality and expected value of explanation; choose dollar/percentage cutoffs with controllers, taking account of effort and compliance. No unapproved fixed thresholds.
- **MLOps/governance:** Governed Redshift/gold SQL calculations, versioned mappings and analytical models, traceable source-to-commentary evidence, reconciliation checks, monthly review, retraining or revalidation on account mapping/costing changes; controller approves final commentary.
- **Business KPI:** Financial close days, variance explained percentage, auditability.

## Shared MLOps and controls
1. **Quality and labeling:** Glue data-quality rules on timestamps, machine IDs, units, duplicates, missing readings and finance joins; quarantine bad records. Document target definitions and label delays.
2. **Reproducibility:** Version code, feature definitions, training snapshots, hyperparameters and metrics; use SageMaker Pipelines and Model Registry for trained models. Feature Store is optional where reusable online/offline features justify its cost.
3. **Deployment:** Choose streaming for anomaly alerts, periodic scoring for failure/scrap risk, daily batch for energy, and governed close-cycle analytics for variances. Keep safe deterministic rules as fallbacks.
4. **Monitoring:** CloudWatch and model/data monitoring for pipeline health, latency, drift, false-alert rates, calibration, delayed outcome quality and cost. Require human review for retraining/promotion and keep rollback versions.
5. **Security:** IAM least privilege, Lake Formation governance, KMS encryption, TLS, audit logs. TALON technical data stays in approved US regions with US-person-only access. Mexico payroll/HR residency boundaries follow approved ADR-2; do not copy restricted raw records into US training sets. Use approved aggregates or in-region processing only after review.
6. **AI agents interface:** Publish approved risk scores, forecasts and evidence to controlled tools; agents cannot bypass the caller's entitlements. CMMS work-order creation and financial commentary require human approval. Wali separately documents AgentCore agents, identity, memory, guardrails and cost.
7. **Design vs sandbox:** This Part 1 describes an enterprise target; Part 3 Lab 8 is a smaller SageMaker demonstration, not a production implementation of all five use cases.

## Review items / assumptions before final submission
- Data Pack not supplied: validate source columns, machine-to-CMMS joins, failure/scrap labels, history length and energy meter availability when received.
- Confirm gold table names and feature ownership with Neha; confirm Kinesis/Flink/alert path with Bradlynn; confirm score interfaces with Wali and the architecture diagram with team lead.
- Validate IAM/regional deployment and service availability with Luke and ADR-2 owner/reviewer.
- Obtain inspection, missed-failure, scrap, utility and materiality costs from business stakeholders before numeric thresholds.
- No model training, tested performance, implemented endpoint, or service-cost estimate is claimed in this draft.

**Instructor source:** MIS 612, *AeroForge Final Project — Part 1 AWS Enterprise Architecture & Design*, sections “Data Science & AI Insights Features,” “Machine Learning Use Cases,” and “Technical (Non-Functional) Requirements.”
