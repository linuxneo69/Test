#  Vertex AI Dynamic SRE Playbook: Universal Model Alerting

This document outlines the setup for a "Zero-Hardcode" alerting suite. One policy per error type will automatically monitor all projects and all generative AI models (Gemini, Claude, etc.).

##  Phase 1: The Variable Syntax (Crucial)

When using PromQL in Google Cloud Alerting, the fields you include in your `sum by (...)` clause are exported as **Metric Labels**.

To dynamically display the failing Project and Model in your alert emails/Slack messages, you must use this exact syntax in the **"Notification subject line"** field in the GCP Console:

* **For the Project:** `${metric.label.resource_container}`
* **For the Model:** `${metric.label.model_user_id}`

*(Note: If GCP natively maps the project back to the resource level in your specific environment, you can also use `${resource.project}`, but `${metric.label.resource_container}` is the safest bet when driven by PromQL).*

---

##  Phase 2: Alert Creation UI Steps

For every alert listed in Phase 3, follow this exact sequence in the Google Cloud Console:

1. Go to **Monitoring > Alerting > + Create Policy**.
2. Click **Select a Metric**, then click the **<> PromQL** button.
3. Paste the provided **PromQL Query**.
4. Set the **Condition Type** to `Threshold`.
5. Set the **Alert trigger** to `Any time series violates`.
6. Enter the **Threshold value** (e.g., `0.02` for 2%).
7. Click **Next** to go to **Notifications and name**.
8. Select your Notification Channels (Slack, Email, PagerDuty).
9. **CRITICAL STEP:** Check the box/expand the field for **Notification subject line** and paste the provided Subject Template.
10. Paste the provided **Documentation Template** into the Documentation text box.
11. Enter the static **Alert Policy Name** (this is just for the GCP Console UI) and save.

---

##  Phase 3: The Universal Alerting Suite

Below are the 6 dynamic alerts. Notice that `model_user_id="xyz"` has been completely removed from the filters.

### 1. Quota Throttling (429 Rate Limit)

**Purpose:** Detects when any model in any project hits its Vertex AI API quota limits.

* **Alert Policy Name:** `Universal | 429 Quota Throttling > 2%`
* **Notification Subject Line:** `[429 Alert] >2% Throttling for Project: ${metric.label.resource_container} | Model: ${metric.label.model_user_id}`
* **Condition:** `> 0.02` (over 15m)
* **PromQL:**

```promql
(
  sum by (resource_container, model_user_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count{response_code="429"}[15m])
  )
  /
  sum by (resource_container, model_user_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count[15m])
  )
) > 0.02

```

* **Documentation Section:**

```markdown
#  429 Rate Limit Breach Detected
**Failing Project:** `${metric.label.resource_container}`
**Affected Model:** `${metric.label.model_user_id}`

**Details:** More than 2% of requests to `${metric.label.model_user_id}` in project `${metric.label.resource_container}` are failing with HTTP 429 (Too Many Requests) over the last 15 minutes. 

**Playbook:**
1. Check if an application has gone rogue and is spamming requests.
2. Review the Quotas page in GCP for this specific project.
3. If traffic is legitimate, request a Quota Increase for Vertex AI.

```

---

### 2. Client Cancellations / Ghost Requests (499 Errors)

**Purpose:** Detects latency issues where the client/browser times out before the model generates the response.

* **Alert Policy Name:** `Universal | 499 Client Cancel Rate > 5%`
* **Notification Subject Line:** `[499 Alert] High Timeout Rate for Project: ${metric.label.resource_container} | Model: ${metric.label.model_user_id}`
* **Condition:** `> 0.05` (over 15m)
* **PromQL:**

```promql
(
  sum by (resource_container, model_user_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count{response_code="499"}[15m])
  )
  /
  sum by (resource_container, model_user_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count[15m])
  )
) > 0.05

```

* **Documentation Section:**

```markdown
# High Client Timeout Rate (499)
**Failing Project:** `${metric.label.resource_container}`
**Affected Model:** `${metric.label.model_user_id}`

**Details:** Clients interacting with `${metric.label.model_user_id}` are dropping connections (499) before the response finishes.

**Playbook:**
1. Check the Time-To-First-Token (TTFT) metrics for this model.
2. Verify if users are submitting excessively large context payloads.
3. Check the frontend/gateway application to ensure API timeouts are set high enough for Generative AI (recommend 60s+).

```

---

### 3. Server-Side Failures (500 & 503 Errors)

**Purpose:** Detects Google-side outages or internal container crashes across any model.

* **Alert Policy Name:** `Universal | CRITICAL 5xx Server Errors > 1%`
* **Notification Subject Line:** `[CRITICAL 5xx] Vertex API Down for Project: ${metric.label.resource_container} | Model: ${metric.label.model_user_id}`
* **Condition:** `> 0.01` (over 15m)
* **PromQL:**

```promql
(
  sum by (resource_container, model_user_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count{response_code=~"50[03]"}[15m])
  )
  /
  sum by (resource_container, model_user_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count[15m])
  )
) > 0.01

```

* **Documentation Section:**

```markdown
# CRITICAL: 5xx Vertex API Errors
**Failing Project:** `${metric.label.resource_container}`
**Affected Model:** `${metric.label.model_user_id}`

**Details:** The Google Cloud backend is returning 500/503 errors. This is usually an infrastructure issue on Google's end or a severe malformed prompt crashing the model endpoint.

**Playbook:**
1. Check the Google Cloud Status Dashboard immediately.
2. Escalate to GCP Support if sustained.

```

---

### 4. General Error Rate (Any Non-200)

**Purpose:** A catch-all safety net for 400 (Bad Request) or 403 (Permission Denied).

* **Alert Policy Name:** `Universal | Total Non-200 Error Rate > 5%`
* **Notification Subject Line:** `[API Degradation] Non-200 Errors > 5% for Project: ${metric.label.resource_container} | Model: ${metric.label.model_user_id}`
* **Condition:** `> 0.05` (over 15m)
* **PromQL:**

```promql
(
  sum by (resource_container, model_user_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count{response_code!="200"}[15m])
  )
  /
  sum by (resource_container, model_user_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count[15m])
  )
) > 0.05

```

* **Documentation Section:**

```markdown
#  General API Degradation
**Failing Project:** `${metric.label.resource_container}`
**Affected Model:** `${metric.label.model_user_id}`

**Playbook:**
1. Open Log Explorer for `${metric.label.resource_container}`.
2. Filter by `severity>=ERROR` to identify the dominant error code (e.g., sudden IAM permission drops causing 403s).

```

---

### 5. GSU Capacity Burndown (90% over 30 min)

**Purpose:** Monitors Provisioned Throughput (PT) health. *Note: If a project/model combination doesn't have a PT limit configured, this query automatically ignores it because the denominator won't exist.*

* **Alert Policy Name:** `Universal | GSU Capacity Burndown > 90%`
* **Notification Subject Line:** `[Capacity Warning] >90% GSU Limit in Project: ${metric.label.resource_container} | Model: ${metric.label.model_user_id}`
* **Condition:** `> 0.9` (over 30m)
* **PromQL:**

```promql
(
  sum by (resource_container, model_user_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput[30m])
  )
  /
  sum by (resource_container, model_user_id) (
    aiplatform_googleapis_com:publisher_online_serving_dedicated_token_limit
  )
) > 0.9

```

* **Documentation Section:**

```markdown
#  High Capacity Utilization (GSU)
**Project:** `${metric.label.resource_container}`
**Model:** `${metric.label.model_user_id}`

**Details:** Sustained token consumption has exceeded 90% of the dedicated limit for the last 30 minutes.

**Playbook:**
1. No immediate outage, but we are near the ceiling.
2. Prepare for potential spillover billing or API throttling.

```

---

### 6. Spillover Token Ratio (>20%)

**Purpose:** Fires when more than 20% of your total token throughput is being handled by the expensive Pay-As-You-Go shared pool.

* **Alert Policy Name:** `Universal | PT vs Spillover Token Ratio > 20%`
* **Notification Subject Line:** `[Cost Alert] High Spillover in Project: ${metric.label.resource_container} | Model: ${metric.label.model_user_id}`
* **Condition:** `> 0.20` (over 15m)
* **PromQL:**

```promql
(
  sum by (resource_container, model_user_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput{request_type="spillover"}[15m])
  )
  /
  sum by (resource_container, model_user_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput{request_type="dedicated"}[15m])
  )
) > 0.20

```

* **Documentation Section:**

```markdown
#  Cost Alert: High Spillover Traffic
**Project:** `${metric.label.resource_container}`
**Model:** `${metric.label.model_user_id}`

**Details:** More than 20% of the token volume for this model is spilling over into standard billing. Our Provisioned Throughput (PT) is undersized.

**Playbook:**
1. Notify the FinOps/Billing team.
2. Evaluate if the GSU capacity needs an increase for `${metric.label.model_user_id}`.

```

---

##  Tips for this Implementation

1. **Handling "No Data" States:** When configuring these alerts in GCP, look for the **"Evaluation missing data"** setting. Set it to `Missing data points considered as values that don't break the policy requirement`. If you don't do this, the alerts will auto-resolve incorrectly during moments of zero traffic.
2. **Auto-Discovery:** Because we removed the model filters, if your team deploys `claude-3.5-sonnet` tomorrow in any of the 10 projects, these alerts will begin tracking it instantly with zero configuration changes required from you.

**Would you like me to help draft a Terraform snippet for these alerts so you can deploy them as Code instead of clicking through the UI?**
