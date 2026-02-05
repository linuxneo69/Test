
# 📄 Confluence: Hybrid Observability Strategic Integration & Validation Plan

**Document Status:** 🟢 DRAFT FOR ARCHITECTURAL REVIEW

**Project Name:** Hybrid Bridge (On-Premise ↔ GCP GCO)

**Document Owner:** [Your Name/Architecture Team]

**Key Stakeholders:** SRE, Cloud Infrastructure, Security, Operations Center (NOC)

---

## 1. Architectural Context

To achieve a "Single Pane of Glass," we are transitioning our on-premise metric telemetry from a siloed Prometheus instance to **Google Cloud Managed Service for Prometheus (GMP)**. This allows us to leverage **Monarch** (Google’s planetary-scale time-series database) for long-term retention, global querying, and unified alerting within the **Google Cloud Observability (GCO)** suite.

---

## 2. Technical Discovery: Strategic Queries

*The following questions are designed to flush out configuration gaps that often lead to data inconsistency or cost overruns.*

### A. Metric Metadata & Pathing

| Query | Context / Technical Rationale |
| --- | --- |
| **Prefix Identification** | **"Which metric prefix should we target in the GCO Metrics Explorer?"** <br>

<br> *Rationale:* Depending on the collector version, metrics usually appear under `prometheus.googleapis.com/` or `external.googleapis.com/prometheus/`. We need this to standardize our dashboard JSONs. |
| **Resource Mapping** | **"Are metrics mapped to the `prometheus_target` or `generic_node` resource type?"** <br>

<br> *Rationale:* GCO groups data by Resource Type. If the mapping is incorrect, filters like `resource.label.node_id` will return null values in our widgets. |
| **Labeling Strategy** | **"What 'External Labels' (e.g., `env`, `site`, `cluster`) are appended at the source?"** <br>

<br> *Rationale:* These labels act as our primary dimensions for multi-cluster filtering. Without them, we cannot distinguish between Production and Staging traffic. |

### B. Ingestion & Pipeline Integrity

| Query | Context / Technical Rationale |
| --- | --- |
| **Ingestion SLO** | **"What is the target end-to-end latency (lag) from an on-premise event to GCO visualization?"** <br>

<br> *Rationale:* If the pipeline lag exceeds 120s, we must de-prioritize GCO for "flash-incident" response and use it primarily for trend analysis and secondary alerting. |
| **Frequency Jitter** | **"What is the Delta between the 'Scrape Interval' and 'Remote Write' frequency?"** <br>

<br> *Rationale:* A 15s scrape with a 60s write creates "steppy" graphs. We need to align these to ensure smooth visualization in GCO widgets. |
| **Metric Pruning** | **"Are any `relabel_configs` active to drop high-cardinality labels (e.g., `pod_id`)?"** <br>

<br> *Rationale:* To manage GCP costs, we must confirm if critical troubleshooting labels are being stripped before they hit the cloud. |

---

## 3. Validation & Test Scenarios

*The following scenarios must be validated by the SRE team.*

### 🧪 Test Scenario 1: Dashboard Data Parity & Fidelity

* **Objective:** Ensure GCO widgets accurately reflect the on-premise reality.
* **Step:** Select a mission-critical metric (e.g., `request_latency_ms`). Compare the 5-minute moving average in On-Prem Grafana vs. GCO Dashboard.
* **Success Criteria:** Variance between sources must be **< 3%**.
* **Verification:** Ensure that "Template Variables" in GCO correctly filter all charts simultaneously.

### 🧪 Test Scenario 2: End-to-End Alerting & NOC Notification

* **Objective:** Confirm that GCO can act as a reliable source for incident management.
* **Step:** Artificially breach a threshold on a non-critical service (e.g., set `service_memory_usage > 10%`).
* **Success Criteria:** 1. GCO triggers an **Incident** within 90 seconds.
2. The Notification Channel (Slack/PagerDuty/Email) receives a payload containing the `cluster_name` and `severity`.
3. The Ops Team confirms they can navigate from the alert directly to the relevant GCO Dashboard.

### 🧪 Test Scenario 3: Log Explorer Freshness & Parsing

* **Objective:** Ensure logs are searchable and correctly indexed.
* **Step:** Run a search for a specific `correlation_id` generated on-prem within the last 5 minutes.
* **Success Criteria:** 1. Log entry appears with a timestamp within **60 seconds** of actual event time.
2. JSON payloads are correctly parsed (labels like `level`, `module`, and `user_id` are indexed as searchable fields).
3. Severity mapping is accurate (On-prem `FATAL` mapped to GCO `CRITICAL`).

---

## 4. Governance & Lifecycle

* **Retention:** Data is stored in Monarch for **24 months** by default.

---

## 5. Next Steps & Meeting Agenda

1. **Cloud Observability Team:** Provide values for the 6 Strategic Queries in Section 2.
2. **SRE Team:** Execute the 3 Test Scenarios in the `Development` project.
