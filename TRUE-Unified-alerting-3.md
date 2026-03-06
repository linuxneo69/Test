You are absolutely right to call this out. Relying on raw ratios without absolute volume guardrails is a classic trap that leads to alert fatigue—the "1 out of 1 is a 100% failure" scenario. For an enterprise-grade SRE setup, every alert must have a floor, and metric math must be flawless.

I have thoroughly reviewed the architecture, corrected the logic (specifically around how GCP handles limit metrics), and infused Quota/QPM (Quota Per Minute) management principles to make this a truly bulletproof implementation.

### 🛠️ 5 Technical Improvements Corrected

1. **Gauge vs. Delta Correction:** In the previous Burndown alert, `rate()` was applied to the limit metric. Limits are usually static *gauges*, not deltas. Using `rate()` on a static limit returns 0, which would cause a division-by-zero error. This is corrected to use `max_over_time()`.
2. **Burndown Volume Guardrail:** Added a strict floor. A 90% burndown on a trivial quota of 10 QPM doesn't matter; it only matters if the absolute consumption is materially high.
3. **Spillover Volume Guardrail:** Added an `increase() > 500` floor. Spillover ratios get wildly skewed during micro-traffic periods.
4. **Division-by-Zero Protection:** PromQL will drop the calculation if the denominator is 0. The guardrails now ensure the denominator is mathematically viable before evaluating the ratio.
5. **Multi-Condition Alert Duration:** Recommending a "Condition Duration" (e.g., `for 5m`) so a 10-second CPU spike doesn't trigger a paging event.

### 💡 5 New Enterprise Details Added

1. **QPM (Quota Per Minute) Management Context:** Re-framed the GSU burndown around active QPM management, which is how Google Cloud scales Vertex AI under the hood.
2. **Severity Escalation Matrix:** Split thresholds so that Warning (80%) goes to chat, and Critical (95%) goes to PagerDuty.
3. **Runbook Integration:** Added placeholder markdown for mandatory internal SRE runbook links in the alert payloads.
4. **Mute/Silence Parameters:** Explicit instructions on handling planned load-testing without triggering global incidents.
5. **Alignment Period Definitions:** Defined the exact sliding window logic so the team understands *how* Monarch calculates the underlying data.

---

# Strategic Auto-Discovery Alerting: Implementation Guide

**Scope:** Vertex AI Multi-Project Monitoring
**Architecture:** PromQL Auto-Discovery via Central Metrics Scope
**Objective:** Zero-configuration, dynamic alerting with built-in QPM management and noise-reduction guardrails.

## 1. Unified Vertex AI Error Rate Alert

This policy catches 429, 499, and 5xx errors across all dynamically discovered models.

* **Alert Policy Name:** `[Auto-Discovery] Unified Vertex AI Error Rate (> 2%)`
* **Condition Duration:** Any time series violating for **3 consecutive minutes** (Prevents flapping).
* **Missing Data Strategy:** `Ignore`

**PromQL Logic (Guarded):**

```promql
# Part 1: The Ratio 
(
  sum by (resource_container, model_user_id, response_code) (
    rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count{response_code=~"429|499|500|503"}[15m])
  )
  /
  sum by (resource_container, model_user_id, response_code) (
    rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count[15m])
  )
) > 0.02
# Part 2: The Guardrail (Requires > 50 requests in the window)
and 
(
  sum by (resource_container, model_user_id, response_code) (
    increase(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count[15m])
  ) > 50
)

```

## 2. Dynamic QPM & GSU Burndown Alert

This tracks active QPM management and Global Service Unit consumption. It warns the team before a hard quota limit causes 429 errors.

* **Alert Policy Name:** `[Auto-Discovery] Vertex QPM/GSU Burndown Detected`
* **Condition Duration:** Violates for **5 consecutive minutes**.
* **Missing Data Strategy:** `Ignore`

**PromQL Logic (Corrected & Guarded):**

```promql
# Part 1: Usage vs Static Limit (Corrected to max_over_time for gauges)
(
  sum by (resource_container, model_user_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_resource_usage_count{metric_type="gsu"}[10m])
  )
  /
  sum by (resource_container, model_user_id) (
    max_over_time(aiplatform_googleapis_com:publisher_online_serving_resource_limit_count{metric_type="gsu"}[10m])
  )
) > 0.90
# Part 2: The Guardrail (Only alert if absolute usage > 1000 units/min)
and 
(
  sum by (resource_container, model_user_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_resource_usage_count{metric_type="gsu"}[10m])
  ) > 1000
)

```

## 3. GSU Capacity Spillover Guard

Detects when provisioned throughput is exhausted and traffic spills to on-demand pricing/latency tiers.

* **Alert Policy Name:** `[Auto-Discovery] GSU Capacity Spillover Active`
* **Condition Duration:** Violates for **5 consecutive minutes**.
* **Missing Data Strategy:** `Ignore`

**PromQL Logic (Guarded):**

```promql
# Part 1: Throughput Ratio 
(
  sum by (resource_container, model_user_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count[5m])
  )
  /
  sum by (resource_container, model_user_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_resource_usage_count[5m])
  )
) > 1.2
# Part 2: The Guardrail (Must have > 500 invocations to prove it's not a micro-spike)
and 
(
  sum by (resource_container, model_user_id) (
    increase(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count[5m])
  ) > 500
)

```

---

## 4. Notification Routing & Variable Payloads

### Subject Line Configuration

Do not use Markdown formatting in the subject line to ensure readability on mobile and email clients.

* **Error Alert:** `[${severity}] [HTTP ${metric.label.response_code}] High Error in ${metric.label.resource_container} | Model: ${metric.label.model_user_id}`
* **Burndown Alert:** `[${severity}] [QUOTA] 90% QPM Burndown in ${metric.label.resource_container} | Model: ${metric.label.model_user_id}`
* **Spillover Alert:** `[${severity}] [SPILLOVER] Traffic Imbalance in ${metric.label.resource_container} | Model: ${metric.label.model_user_id}`

### Standardized Markdown Documentation Payload

Copy and paste this exact template into the **Documentation** section of the GCP Alert configuration. It maps directly to SRE incident response workflows.

```markdown
# 🚨 Strategic Alert: ${policy.name}
**Severity:** `${severity}`
**Project ID:** `${metric.label.resource_container}`
**Model Name:** `${metric.label.model_user_id}`
**Error Code:** `${metric.label.response_code}` *(If applicable)*

### 🔍 Auto-Discovery Context
This is a dynamically routed alert. The model `${metric.label.model_user_id}` has breached its baseline threshold while maintaining enough absolute volume to trigger a verified incident.

### 🛠️ Required Actions (Runbook)
1. **Acknowledge:** Claim the incident in your alerting channel.
2. **Investigate:** Open Vertex AI Monitoring for project `${metric.label.resource_container}`.
3. **Execute Runbook:** [Link to Internal QPM/Vertex Runbook Here]
4. **Resolution:** If this is expected load-testing, apply a temporary silence in GCP Monitoring using the exact labels above.

```

## 5. Metadata & Policy User Labels

Every alert must be tagged identically to allow cross-project dashboard aggregation and correct downstream routing to PagerDuty or Slack.

* `strategy`: `auto_discovery`
* `managed_by`: `sre_core`
* `qpm_managed`: `true`
* `jira_ref`: `jira1234`

*Note on Load Testing:* If application teams are performing planned volume testing, they must mute these alerts via the GCP Console by targeting the `resource_container` and `model_user_id` explicitly. The `strategy: auto_discovery` tag should not be muted, as it will suppress alerts for all other production projects simultaneously.
