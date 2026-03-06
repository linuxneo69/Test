
---

## 🛠️ The "Final Fix" for the Variable Issue

### 1. The PromQL Requirement (The Data Source)

In your previous configurations, the grouping was either missing or inconsistent. For the variables to work in the subject line, they **must** be present in the `sum by` clause of your query.

**Use this exact PromQL structure for your Error Alert:**

```promql
(
  sum by (resource_container, model_user_id, response_code) (
    rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count{response_code=~"429|499|500|503"}[15m])
  )
  /
  sum by (resource_container, model_user_id, response_code) (
    rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count[15m])
  )
) > 0.02
and 
(
  sum by (resource_container, model_user_id, response_code) (
    increase(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count[15m])
  ) > 50
)

```

### 2. The Notification Syntax (The Display)

Your screenshots show various attempts like `${resource.labels.project_id}` and `${metric.labels.model_user_id}`. In the current Google Cloud Monitoring "Monarch" engine for PromQL, those paths are often invalid.

**Use these exact strings (No backticks, singular `label`):**

* **Project ID:** `${metric.label.resource_container}`
* **Model ID:** `${metric.label.model_user_id}`
* **Response Code:** `${metric.label.response_code}`
* **Severity:** `${severity}`

---

## 📄 Updated Enterprise Documentation (Jira1234)

This version incorporates the **5 Guardrail Improvements** and **5 Detail Enhancements** to ensure an enterprise-grade, "null-free" deployment.

### 1. Unified Error Alert

* **Policy Name:** `[Auto-Discovery] Unified Vertex AI Error Rate (> 2%)`
* **Condition Name:** `High Error Rate per Project/Model`
* **Subject Line:** `[${severity}] [HTTP ${metric.label.response_code}] High Error in ${metric.label.resource_container} | Model: ${metric.label.model_user_id}`
* **Policy User Labels:** `strategy:auto_discovery`, `managed_by:sre_core`, `env:prod`

### 2. GSU Quota Burndown (with Guardrails)

* **Policy Name:** `[Auto-Discovery] GSU Quota Burndown (> 90%)`
* **Condition Name:** `GSU Usage Approaching Limit`
* **Subject Line:** `[${severity}] [QUOTA] 90% GSU Burndown in ${metric.label.resource_container} | Model: ${metric.label.model_user_id}`
* **PromQL Guardrail:** Added `max_over_time` to handle static limits and a volume floor of `> 1000` units to prevent alerts on negligible usage.

### 3. GSU Capacity Spillover (with Guardrails)

* **Policy Name:** `[Auto-Discovery] GSU Capacity Spillover Active`
* **Condition Name:** `Throughput Imbalance Detected`
* **Subject Line:** `[${severity}] [SPILLOVER] Imbalance in ${metric.label.resource_container} | Model: ${metric.label.model_user_id}`
* **PromQL Guardrail:** Added a floor of `increase(...) > 500` to ensure the ratio is based on a statistically significant sample of traffic.

---

## 🚀 5 Key Improvements & 5 New Details

### Improvements (Correcting common SRE errors):

1. **Fixed Denominator Zero-Division:** All queries now include volume guardrails (`> 50` or `> 500`) which prevents the "null" math errors that occur when a project has zero traffic.
2. **Corrected Gauge Aggregation:** Switched limit monitoring from `rate()` to `max_over_time()`, ensuring static quota limits are read correctly.
3. **Variable Path Standardization:** Standardized all variables to the `${metric.label.KEY}` format, which is the most stable for PromQL-based alerts in GCP.
4. **Duration Filtering:** Set a "Condition Duration" of **5 minutes** to filter out momentary transient spikes that don't require human intervention.
5. **Clean Subject Lines:** Removed all backticks (```) from subject lines to prevent rendering issues in mobile notifications.

### New Enterprise Details:

1. **QPM Management Integration:** Specifically tailored for "Quota Per Minute" monitoring, essential for Vertex AI scaling.
2. **Cross-Project Scoping:** Instructions to use a central "Monitoring Hub" project to discover all 10 project IDs dynamically.
3. **Severity Escalation:** Documentation now supports dynamic `${severity}` mapping for automated routing to PagerDuty or Slack.
4. **Missing Data Strategy:** Explicitly set to `Ignore` to prevent "No Data" false alarms during maintenance or low-traffic periods.
5. **Standardized Runbooks:** Documentation field now includes a mandatory template for SRE troubleshooting steps.

### Final Markdown Documentation Payload

Paste this into the **Documentation** field to replace the current failing text:

```markdown
# 🚨 Vertex AI Strategic Alert
**Project:** `${metric.label.resource_container}`
**Model:** `${metric.label.model_user_id}`
**Status:** `${metric.label.response_code}`

### 🔍 Incident Detail
This alert was triggered by the **Jira1234** Auto-Discovery framework. 
The current violation has sustained for over 5 minutes with significant traffic volume.

### 🛠️ Response Steps
1. Open **Metrics Explorer** for project `${metric.label.resource_container}`.
2. Verify if the error is specific to `${metric.label.model_user_id}`.
3. Refer to the **Internal Vertex Runbook** for scaling/quota instructions.

```

**Would you like me to generate a specific PromQL query for a 4th alert to monitor "Latency P99" across these same projects?**
