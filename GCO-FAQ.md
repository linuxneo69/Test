This documentation is designed for high-level architectural alignment while providing the ground-level detail needed for a successful Go-Live.

---

# 📄 Confluence: Hybrid Observability Strategic Integration & Validation

**Document Status:** 🟢 DRAFT FOR ARCHITECTURAL REVIEW

**Project Name:** Hybrid Bridge (On-Premise ↔ GCP GCO)

**Document Owner:** [Your Name/Architecture Team]

**Key Stakeholders:** SRE, Cloud Infrastructure, Operations Center (NOC)

---

## 1. Architectural Overview

This initiative establishes a secure data bridge between on-premise telemetry and **Google Cloud Observability (GCO)**. We are leveraging **Prometheus Remote Write** to ship metrics to **Monarch** (Google’s global time-series database) to achieve centralized long-term retention and unified hybrid-cloud alerting.

---

## 2. Technical Discovery: Strategic Queries

*These queries are designed to be addressed in the stakeholder meeting to ensure the data pipeline is optimized for cost and performance.*

| Category | Strategic Query | Architect's Intent / Rationale |
| --- | --- | --- |
| **Pathing** | **"Which metric prefix should we target in the GCO Metrics Explorer?"** | To standardize dashboards. We must confirm if metrics appear under `prometheus.googleapis.com/` or `external.googleapis.com/`. |
| **Topology** | **"Are metrics mapped to the `prometheus_target` or `generic_node` resource type?"** | Resource mapping dictates how GCO groups data. Incorrect mapping causes "null" results when filtering by `node_id`. |
| **Labels** | **"What 'External Labels' (e.g., `env`, `site`) are appended at the source?"** | Essential for multi-cluster filtering. Without these, we cannot distinguish between "On-Prem" and "GCP-Native" traffic. |
| **SLO** | **"What is the target end-to-end ingestion lag for on-premise events?"** | Defines the threshold for "Real-time" response. If lag > 120s, GCO is for trend analysis; local is for flash-incident response. |
| **Jitter** | **"What is the Delta between 'Scrape Interval' and 'Remote Write' frequency?"** | A 15s scrape with a 60s write creates "steppy" graphs. Alignment is required for smooth visual fidelity in GCO widgets. |
| **Pruning** | **"Are any `relabel_configs` active to drop high-cardinality labels at the source?"** | To manage costs, we must know if critical troubleshooting labels (e.g., `pod_id`) are being stripped before ingestion. |

---

## 3. Test & Validation Scenarios (UAT)

*Each scenario must be validated and signed off. "Failure Criteria" are included to help identify common integration pitfalls.*

### 🧪 Scenario 1: Dashboard Data Parity

**Objective:** Confirm GCO accurately reflects the on-premise state.

* **Steps:** Select a high-traffic metric (e.g., `request_latency_ms`). Compare the 5-minute moving average in On-Prem Grafana vs. a GCO Widget.
* **✅ Success Criteria:** * Variance between sources is **< 3%**.
* Template variables (e.g., Cluster dropdown) correctly filter all charts.


* **❌ Failure Criteria (Red Flags):**
* **Data Flatlining:** GCO shows a straight line while on-prem shows variability (indicates batching/queue issues).
* **"Unknown" Metric Type:** PromQL functions like `rate()` fail because metric type metadata was dropped.
* **Label Mismatch:** Metrics appear but lack the `env` or `site` labels required for filtering.



### 🧪 Scenario 2: Alerting & NOC Notification

**Objective:** Validate the reliability of the "Cloud-Triggered" alert loop.

* **Steps:** Set a synthetic "Critical" threshold in GCO (e.g., `memory_utilization > 5%` for a test pod).
* **✅ Success Criteria:** * GCO Incident is created within **90 seconds**.
* NOC confirms receipt of notification via Slack/PagerDuty with full context (Server Name, Metric Value).


* **❌ Failure Criteria (Red Flags):**
* **Delayed Firing:** Alert takes > 5 minutes to trigger (indicates Remote Write queue backlog).
* **Missing Metadata:** Alert arrives but doesn't tell the NOC *which* datacenter or cluster is affected.
* **"Flapping" Alerts:** GCO triggers and clears the alert rapidly due to data ingestion gaps.


### 🧪 Scenario 3: Log Explorer Freshness

**Objective:** Ensure logs are searchable and correctly categorized.

* **Steps:** Generate a unique log string (e.g., `TEST_LOG_ID_001`) on-prem and search in Log Explorer.
* **✅ Success Criteria:** * Log entry appears in GCO within **60 seconds**.
* Severity mapping is correct (e.g., On-prem `FATAL` appears as `CRITICAL` in GCO).


* **❌ Failure Criteria (Red Flags):**
* **Timestamp Skew:** Log appears but is "hidden" in the past or future due to time-sync issues between On-prem and GCP.
* **Unstructured Payloads:** JSON logs from on-premise appear as a single "text" string, making individual fields unsearchable.
* **Missing Resource ID:** Logs are present but cannot be correlated to a specific model or Containers.


---

## 4. Governance & Lifecycle

* **Data Retention:** On-premise metrics ingested into the Monarch backend will follow the standard **24-month retention** policy unless otherwise specified for compliance.
* **Archiving:** Historical data beyond 24 months is currently **not** scoped for export to Cloud Storage buckets.

---

## 5. Next Steps

1. **Ops Team:** Review Section 2 and provide the specific Prefix and Labeling values.
2. **SRE Team:** Execute the 3 Test Scenarios in the `Staging` project and document the results below.
