# Vertex AI Unified Alerting Guide

**Last Updated:** March 17, 2026  
**Status:** Production-Ready  
**Audience:** Platform Engineers, SREs, MLOps Teams

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture & Strategy](#architecture--strategy)
3. [Metrics Used](#metrics-used)
4. [Alert Policies](#alert-policies)
5. [PromQL Design Principles](#promql-design-principles)
6. [Alert Notification Variables](#alert-notification-variables)
7. [Implementation Steps](#implementation-steps)
8. [Best Practices](#best-practices)
9. [Troubleshooting](#troubleshooting)

---

## Overview

This guide provides production-ready monitoring and alerting for Google Cloud Vertex AI deployments. The approach uses **unified auto-discovery alerting** to automatically monitor all projects and models without manual configuration updates when new models or projects are added.

### Why Unified Auto-Discovery?

- **Eliminates Maintenance Burden:** Single alert policy works across all projects and models
- **Auto-Discovers New Models:** When a new model is deployed, it's automatically monitored
- **Reduces Alert Fatigue:** Smart guardrails prevent false positives from low-traffic projects
- **Scalable Architecture:** Supports 10+ projects and dozens of concurrent models
- **Cost Visibility:** Tracks both capacity utilization and cost overruns in real time

### What This Covers

| Category | Alerts Included |
|----------|-----------------|
| **Reliability** | Non-200 error rates, quota throttling, timeouts, server errors |
| **Capacity** | GSU burndown, predictive saturation, spillover detection |
| **Cost** | Spillover ratio monitoring |
| **Coverage** | All models across all projects in your metrics scope |

---

## Architecture & Strategy

### Multi-Project Monitoring with Metrics Scope

Google Cloud Monitoring uses a **Metrics Scope** to aggregate metrics from multiple projects into a single observability hub. This central hub becomes the control plane for all alerting logic.

```
┌─────────────────────────────────────────────────────────┐
│         Central Metrics Scope (Monitoring Hub)           │
│                                                           │
│  PromQL Alert Policies (universal, model-agnostic)       │
│  └─ Applied to all projects simultaneously               │
│                                                           │
│  Receives metrics from:                                  │
│  ├─ Project A (prod-ai, staging-ml, dev-test)          │
│  ├─ Project B (prod-ml, qa-vertex, experiment)         │
│  └─ Project C (model-serving, batch-inference)          │
│                                                           │
│  Alert Results:                                          │
│  ├─ Project: prod-ai | Model: gemini-1.5-pro           │
│  ├─ Project: staging-ml | Model: claude-3-sonnet        │
│  └─ Project: experiment | Model: palm-2                 │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

### How Auto-Discovery Works

The key to auto-discovery is the `sum by (resource_container, model_user_id)` construct in PromQL:

```promql
# Good: Each project/model gets its own time series
sum by (resource_container, model_user_id) (metric)

# Bad: All projects aggregated into one number
sum(metric)
```

When you group by these labels, CloudMonitoring automatically creates separate alert incidents for each project/model combination. When you deploy a new model tomorrow, it appears in the grouped output automatically without changing the policy.

### Setup Requirements

Before creating alerts:

1. **Create a Metrics Scope**
   - In Google Cloud Console → Monitoring → Settings
   - Add all sub-projects that emit Vertex AI metrics

2. **Verify Label Names**
   - Open Metrics Explorer → PromQL
   - Run: `topk(10, aiplatform_googleapis_com:publisher_online_serving_model_invocation_count)`
   - Check the legend for exact label names (usually `resource_container`, `model_user_id`)

3. **Confirm Metrics Availability**
   - Ensure metrics are flowing from all projects
   - Check for any data gaps or permission issues

---

## Metrics Used

### Model Invocation Count

**Metric Name:** `aiplatform_googleapis_com:publisher_online_serving_model_invocation_count`

**What It Measures:** Total number of API calls to any Vertex AI model endpoint

**Key Labels:**
- `resource_container` - GCP project ID
- `model_user_id` - Model name (e.g., `gemini-1.5-pro`, `claude-3-sonnet`)
- `response_code` - HTTP status code (200, 429, 499, 500, 503, etc.)

**Usage:**
- Tracking error rates by status code
- Monitoring request volume (guardrails for alerts)
- Detecting quota throttling and client timeouts

**Example Query:**
```promql
sum by (resource_container, model_user_id, response_code) (
  rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count[5m])
)
```

---

### Consumed Token Throughput

**Metric Name:** `aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput`

**What It Measures:** Rate of token consumption (tokens/second) for model inference

**Key Labels:**
- `resource_container` - GCP project ID
- `model_user_id` - Model name
- `request_type` - Either `dedicated` or `spillover`
  - **dedicated** = Tokens consumed from Provisioned Throughput (PT) allocation
  - **spillover** = Tokens consumed from on-demand pay-as-you-go pool (3x cost)

**Usage:**
- Monitoring GSU capacity utilization
- Detecting spillover (cost overruns)
- Predicting capacity exhaustion

**Example Query:**
```promql
sum by (resource_container, model_user_id, request_type) (
  rate(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput[15m])
)
```

---

### Dedicated Token Limit

**Metric Name:** `aiplatform_googleapis_com:publisher_online_serving_dedicated_token_limit`

**What It Measures:** The maximum token throughput (tokens/second) allocated to a model via Provisioned Throughput (GSU)

**Key Labels:**
- `resource_container` - GCP project ID
- `model_user_id` - Model name

**Usage:**
- Calculating capacity utilization (consumed / limit)
- Predictive saturation detection
- Identifying undersized allocations

**Example Query:**
```promql
sum by (resource_container, model_user_id) (
  max over (30m) (aiplatform_googleapis_com:publisher_online_serving_dedicated_token_limit)
)
```

---

## Alert Policies

Each alert policy below is production-ready and can be copied directly into Google Cloud Monitoring.

### Alert 1: Non-200 Error Rate

**Purpose:** Detects any error response (4xx, 5xx, etc.) exceeding a threshold

**Severity:** CRITICAL  
**Response Time:** 5-10 minutes  

**Policy Configuration:**

| Setting | Value |
|---------|-------|
| Policy Name | `VertexAI / Errors / Non-200 Rate` |
| Condition Name | `Non-200 Rate > 10% (15m, ≥1000 requests)` |
| Severity | CRITICAL |
| Tags | `vertexai`, `errors`, `reliability`, `auto-discovery` |
| Group By | `resource_container`, `model_user_id` |

**PromQL Query:**

```promql
(
  sum by (resource_container, model_user_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count{response_code!="200"}[15m])
  )
  /
  sum by (resource_container, model_user_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count[15m])
  )
) > 0.10
and
(
  sum by (resource_container, model_user_id) (
    increase(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count[15m])
  ) > 1000
)
```

**Why This Query:**
- Ratio calculates error percentage for each project/model
- First `and` clause filters to only projects with >1000 requests (minimum volume)
- Prevents false positives from low-traffic endpoints

**Notification Subject Line:**

```
[${severity}] Non-200 Error Rate >10% | Project: ${metric.label.resource_container} | Model: ${metric.label.model_user_id}
```

**Notification Documentation:**

```markdown
## Alert: Non-200 Response Rate Elevated

**Project:** ${metric.label.resource_container}  
**Model:** ${metric.label.model_user_id}  
**Error Rate:** ${metric.value}  
**Window:** Last 15 minutes

### What This Means
More than 10% of requests returned non-2xx responses. This indicates a reliability issue affecting your model endpoint.

### Common Causes
- **400/403:** IAM permission issues, invalid request format
- **429:** API quota exceeded (see separate quota throttling alert)
- **499:** Client timeout before model responds
- **500/503:** Google Cloud infrastructure issue

### Immediate Actions
1. Open Cloud Monitoring → Metrics Explorer
2. Filter by: `resource_container="${metric.label.resource_container}"` AND `model_user_id="${metric.label.model_user_id}"`
3. Break down by `response_code` to identify the dominant error type
4. Check recent deployments in the affected project
5. Review application logs for error context
6. Contact model owner if issue persists >15min

### Escalation
- **If 5xx errors:** Escalate to Platform SRE immediately
- **If 4xx errors:** Escalate to application team
- **If quota errors (429):** Contact capacity planning team

**Runbook:** [Link to internal runbook]
```

**Threshold & Conditions:**
- Threshold: `> 0.10` (10%)
- Alert Trigger: `Any time series violates`
- Duration: Fires after 1 evaluation period (1 minute)
- Alignment Period: 1 minute

---

### Alert 2: GSU Burndown (Static Capacity)

**Purpose:** Detects when provisioned throughput capacity is running low

**Severity:** WARNING  
**Response Time:** 30 minutes (capacity planning action, not emergency)

**Policy Configuration:**

| Setting | Value |
|---------|-------|
| Policy Name | `VertexAI / Capacity / GSU Burndown` |
| Condition Name | `Capacity > 90% (30m avg)` |
| Severity | WARNING |
| Tags | `vertexai`, `capacity`, `gsu`, `planning` |
| Group By | `resource_container`, `model_user_id` |

**PromQL Query:**

```promql
(
  sum by (resource_container, model_user_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput[30m])
  )
  /
  sum by (resource_container, model_user_id) (
    max over (30m) (aiplatform_googleapis_com:publisher_online_serving_dedicated_token_limit)
  )
) > 0.90
and
(
  sum by (resource_container, model_user_id) (
    increase(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput[30m])
  ) > 50000
)
```

**Why This Query:**
- Calculates utilization = consumed / limit
- `max over (30m)` ensures we use the limit value from the evaluation window
- Guardrail: `> 50000` tokens consumed prevents alerting on idle models
- 30-minute window detects sustained (not transient) high utilization

**Notification Subject Line:**

```
[${severity}] GSU Capacity Alert | Project: ${metric.label.resource_container} | Model: ${metric.label.model_user_id} | Util: ${metric.value}%
```

**Notification Documentation:**

```markdown
## Alert: High GSU Capacity Utilization (90%+)

**Project:** ${metric.label.resource_container}  
**Model:** ${metric.label.model_user_id}  
**Current Utilization:** ${metric.value}%  
**Window:** Last 30 minutes (sustained)

### What This Means
Your Provisioned Throughput (GSU) allocation is nearly saturated. If traffic continues to grow, requests will spill over into on-demand billing (3x cost) and latency may increase.

### Immediate Actions
1. Check recent traffic patterns:
   - Is this expected (new feature launch, seasonal load)?
   - Or unexpected (misconfiguration, runaway client)?

2. Short-term mitigation:
   - Defer non-critical batch inference jobs to off-peak hours
   - Implement request prioritization (tiered SLAs)
   - Throttle low-priority callers

3. Long-term resolution:
   - Provision additional GSU capacity
   - Submit capacity request to platform team

### Cost Impact
- At 90% utilization, spillover begins
- Each spillover token costs 3x more than provisioned
- **Cost forecast for 1 month sustained:** $[estimated overage]

### Escalation Path
1. Notify model owner (responsible for capacity planning)
2. Alert FinOps team if sustained >1 hour (cost implications)
3. Request capacity increase through normal process

**Runbook:** [Link to GSU provisioning guide]
```

**Threshold & Conditions:**
- Threshold: `> 0.90` (90%)
- Alert Trigger: `Any time series violates`
- Duration: Fires after 1 evaluation period
- Alignment Period: 1 minute
- Auto-Close: 60 minutes after condition clears

---

### Alert 3: Spillover Detection (Cost Overage)

**Purpose:** Detects when requests are being served from expensive on-demand pool instead of provisioned capacity

**Severity:** WARNING  
**Response Time:** 1-2 hours (cost/capacity planning action)

**Policy Configuration:**

| Setting | Value |
|---------|-------|
| Policy Name | `VertexAI / Cost / Spillover Ratio` |
| Condition Name | `Spillover > 20% of Total (15m)` |
| Severity | WARNING |
| Tags | `vertexai`, `cost`, `spillover`, `finops` |
| Group By | `resource_container`, `model_user_id` |
| Notification Channel | Email to FinOps team (not on-call pager) |

**PromQL Query:**

```promql
(
  sum by (resource_container, model_user_id) (
    increase(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput{request_type="spillover"}[15m])
  )
  /
  (
    sum by (resource_container, model_user_id) (
      increase(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput[15m])
    ) + 0.001
  )
) > 0.20
and
(
  sum by (resource_container, model_user_id) (
    increase(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput{request_type="spillover"}[15m])
  ) > 5000
)
```

**Why This Query:**
- Numerator: tokens from spillover (on-demand)
- Denominator: total tokens (dedicated + spillover)
- `+ 0.001` prevents division by zero gracefully
- Guardrail: spillover must be >5000 tokens (actual cost impact)
- Ratio > 20% indicates undersized GSU allocation

**Notification Subject Line:**

```
[FinOps] Spillover Alert >20% | Project: ${metric.label.resource_container} | Model: ${metric.label.model_user_id}
```

**Notification Documentation:**

```markdown
## Alert: High Spillover Ratio (Cost Overrun)

**Project:** ${metric.label.resource_container}  
**Model:** ${metric.label.model_user_id}  
**Spillover Ratio:** ${metric.value}%  
**Window:** Last 15 minutes

### What This Means
More than 20% of inference tokens are being served from on-demand pay-as-you-go pool instead of your provisioned capacity (GSU).

### Cost Impact
- **On-demand:** ~$0.075-0.30 per 1M tokens (varies by model)
- **Provisioned (GSU):** ~$0.01 per 1M tokens (pre-paid)
- **Multiplier:** 3-30x cost for spillover traffic
- **This month's overage:** Estimated $[calculated amount]

### Root Causes
1. **PT Undersized:** Allocation too small for actual traffic
2. **Traffic Spike:** Unexpected load (new feature, marketing campaign)
3. **Inefficient Prompts:** Long context window, excessive retries
4. **Resource Contention:** Other models consuming shared capacity

### Remediation
1. **Immediate:** Check if spike is expected
   - Review recent deployments and traffic sources
   
2. **This Week:**
   - Increase PT allocation for this model
   - Implement request throttling if necessary
   
3. **This Month:**
   - Analyze efficiency (tokens per inference)
   - Right-size PT allocation based on trending
   
4. **Budget Planning:**
   - Include spillover costs in capacity budget
   - Consider reserved capacity commitments

**Escalation:** Notify budget owner and platform team

**Runbook:** [Link to capacity planning guide]
```

**Threshold & Conditions:**
- Threshold: `> 0.20` (20%)
- Alert Trigger: `Any time series violates`
- Duration: Fires after 1 evaluation period
- Alignment Period: 1 minute
- Auto-Close: 4 hours after condition clears

---

### Alert 4: Predictive Spillover (Early Warning)

**Purpose:** Predicts spillover 10 minutes in advance to allow proactive capacity provisioning

**Severity:** WARNING  
**Response Time:** 5-10 minutes (for capacity team to act before spillover)

**Policy Configuration:**

| Setting | Value |
|---------|-------|
| Policy Name | `VertexAI / Capacity / Predictive Spillover` |
| Condition Name | `Predicted Spillover > 10% in 10 minutes` |
| Severity | WARNING |
| Tags | `vertexai`, `predictive`, `spillover`, `capacity` |
| Group By | `resource_container`, `model_user_id` |

**PromQL Query:**

```promql
predict_linear(
  (
    sum by (resource_container, model_user_id) (
      rate(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput{request_type="spillover"}[30m])
    )
  ) * 600,
  600
)
>
0.10 * (
  sum by (resource_container, model_user_id) (
    max over (30m) (aiplatform_googleapis_com:publisher_online_serving_dedicated_token_limit)
  )
)
and
(
  sum by (resource_container, model_user_id) (
    increase(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput{request_type="spillover"}[30m])
  ) > 5000
)
```

**Why This Query:**
- `predict_linear(..., 600)`: Projects consumption 10 minutes (600 seconds) into future
- Based on last 30 minutes of trend
- Checks if predicted spillover will exceed 10% of limit
- Guardrail ensures minimum traffic volume

**Notification Subject Line:**

```
[⚠️ Predictive] Spillover in 10 min | Project: ${metric.label.resource_container} | Model: ${metric.label.model_user_id}
```

**Notification Documentation:**

```markdown
## Early Warning: Spillover Predicted in 10 Minutes

**Project:** ${metric.label.resource_container}  
**Model:** ${metric.label.model_user_id}  
**Prediction:** Based on last 30 minutes of traffic

### What This Means
Linear extrapolation of current traffic trends suggests spillover will begin within 10 minutes.

### Actions (Before Spillover Occurs)
1. **Immediate (Next 5 min):**
   - Verify prediction is accurate
   - Check if traffic spike is expected
   - Open Vertex AI console for project

2. **If Legitimate Traffic:**
   - Trigger auto-scaling policy to increase capacity
   - Manual capacity increase (if auto-scaling unavailable)
   
3. **If Unexpected Spike:**
   - Check for runaway client (misconfigured retry loop)
   - Block or throttle problematic traffic source
   - Rollback recent deployments if applicable

### Why Early Warning Matters
- **10-minute lead time** = time to:
  - Provision additional GSU capacity (usually 5-10 min)
  - Implement traffic throttling
  - Notify stakeholders
  
- **Without warning** = spillover = cost overrun + latency increase

**Runbook:** [Link to capacity escalation procedures]
```

**Threshold & Conditions:**
- Threshold: Any predicted spillover
- Alert Trigger: `Any time series violates`
- Duration: Fires after 1 evaluation period
- Alignment Period: 1 minute

---

## PromQL Design Principles

### 1. Label Preservation with `sum by`

**Principle:** Always explicitly group by dimensions you care about

**❌ Wrong:**
```promql
sum(metric)  # All projects/models aggregated into single value
```

**✅ Correct:**
```promql
sum by (resource_container, model_user_id) (metric)
# Each project/model gets its own time series
```

**Why:** When you use `sum by (...)`, those labels are preserved in the output and available for:
- Grouping alert incidents separately per project/model
- Dynamic variables in notifications: `${metric.label.resource_container}`
- Filtering in dashboards

### 2. Guardrails Prevent Alert Fatigue

**Principle:** Combine ratio with absolute floor to prevent alerts on low-traffic endpoints

**❌ Wrong (Too Noisy):**
```promql
rate(errors[5m]) / rate(total[5m]) > 0.05
# Fires even if just 1 error out of 20 requests
```

**✅ Correct (Guardrailed):**
```promql
(rate(errors[5m]) / rate(total[5m]) > 0.05)
and
(increase(total[5m]) > 100)
# Only fires if error rate is high AND traffic volume is significant
```

**Why:** Without guardrails:
- Development/test endpoints alert constantly
- On-call team gets fatigued
- Actual production issues get lost in noise

### 3. Request Type Filtering for Spillover

**Principle:** Use `request_type` label to distinguish provisioned vs on-demand

**Example:**
```promql
rate(consumed_token_throughput{request_type="spillover"}[15m])  # On-demand tokens
rate(consumed_token_throughput{request_type="dedicated"}[15m])   # Provisioned tokens
```

**Why:** This label is set by Google Cloud and accurately reflects billing:
- `spillover` = expensive pay-as-you-go
- `dedicated` = pre-paid provisioned throughput
- Enables cost-aware alerting

### 4. Avoid Hardcoded Filters for Auto-Discovery

**Principle:** Never filter by specific model name or project ID

**❌ Wrong (No Auto-Discovery):**
```promql
sum by (resource_container, model_user_id) (
  metric{model_user_id="gemini-1.5-pro"}  # Hardcoded!
)
# Only monitors this one model
# New models are invisible
```

**✅ Correct (Auto-Discovery):**
```promql
sum by (resource_container, model_user_id) (
  metric
)
# All models automatically included
# No updates needed when new models deploy
```

**Why:** Hardcoded filters require manual updates. The whole point of auto-discovery is to reduce toil.

---

## Alert Notification Variables

When an alert fires, Google Cloud Monitoring substitutes variables with actual values. These are available in notification subject lines and documentation fields.

### Available Variables

| Variable | Description | Example Output |
|----------|-------------|-----------------|
| `${metric.label.resource_container}` | GCP Project ID | `prod-ai` |
| `${metric.label.model_user_id}` | Model name | `gemini-1.5-pro` |
| `${metric.value}` | Metric value that triggered alert | `0.15` (15%) |
| `${metric.label.response_code}` | HTTP response code (if grouped) | `429` |
| `${metric.label.request_type}` | `spillover` or `dedicated` | `spillover` |
| `${severity}` | Alert severity | `CRITICAL` |
| `${policy.display_name}` | Policy name | `VertexAI / Errors / Non-200 Rate` |

### Example Subject Line

**Template:**
```
[${severity}] ${policy.display_name} | Project: ${metric.label.resource_container} | Model: ${metric.label.model_user_id}
```

**Example Output:**
```
[CRITICAL] VertexAI / Errors / Non-200 Rate | Project: prod-ai | Model: gemini-1.5-pro
```

### Troubleshooting `(null)` Variables

If a variable shows as `(null)` in the actual alert:

**Root Cause:** The label was dropped during query execution (aggregation or logical operations)

**Fix:**
1. Ensure `sum by (resource_container, model_user_id)` is in EVERY aggregation
2. Test the PromQL in Metrics Explorer
3. Check the legend to verify labels are present in output

---

## Implementation Steps

### Step 1: Verify Prerequisites

```bash
# In Google Cloud Console → Monitoring → Metrics Explorer

# Check 1: Verify metrics exist
topk(10, aiplatform_googleapis_com:publisher_online_serving_model_invocation_count)
# Expected: Shows multiple time series with labels

# Check 2: Verify label names
# In the legend, confirm you see:
# - resource_container (project ID)
# - model_user_id (model name)
# - response_code (HTTP status)
# - request_type (spillover/dedicated) - if GSU alerts used

# Check 3: Verify metrics scope
# Settings → Metrics Scope tab
# Confirm all target projects are listed
```

### Step 2: Create First Alert (Non-200 Error Rate)

1. **Open Alerting Console**
   - Google Cloud Console → Monitoring → Alerting
   - Click `+ Create Policy`

2. **Select Metric and Condition**
   - Click `Select a Metric`
   - Click `<> PromQL` tab
   - Copy the Non-200 Error Rate PromQL query (see [Alert 1](#alert-1-non-200-error-rate))
   - Click `OK`

3. **Configure Alert Trigger**
   - Set Condition Type: `Threshold`
   - Alert Trigger: `Any time series violates`
   - Threshold Value: `0.10` (for 10%)
   - Note the Alignment Period (should be 1 minute)

4. **Add Notification**
   - Click `Next`
   - Under Notifications, select channel (Slack, Email, PagerDuty)
   - **Important:** Add custom subject line:
     ```
     [${severity}] Non-200 Error >10% | Project: ${metric.label.resource_container} | Model: ${metric.label.model_user_id}
     ```

5. **Add Documentation**
   - Paste the Notification Documentation from [Alert 1](#alert-1-non-200-error-rate)
   - This appears in the alert message and provides runbook context

6. **Name and Save**
   - Alert Name: `VertexAI / Errors / Non-200 Rate`
   - Tags: `vertexai`, `errors`, `reliability`, `auto-discovery`
   - Click `Create Policy`

### Step 3: Test the Alert (Sandbox Environment)

1. **Lower thresholds in staging**
   - Create same alert in staging project
   - Change threshold to 5% (instead of 10%)
   - Change guardrail to 50 requests (instead of 1000)

2. **Generate test traffic**
   - Send requests to a staging model endpoint
   - Intentionally trigger errors (send invalid payloads)
   - Wait 1-2 minutes for evaluation

3. **Verify notification**
   - Check that alert fired
   - Verify variables expanded correctly:
     - Project name shows correctly
     - Model name shows correctly
     - No `(null)` values

4. **Adjust if needed**
   - If variables show `(null)`: Review PromQL for label preservation
   - If alert fires too frequently: Increase guardrail threshold
   - If alert doesn't fire: Lower the error rate threshold

### Step 4: Deploy to Production

1. **Create policies for each alert type**
   - Follow the same steps for GSU Burndown, Spillover, etc.
   - Use production thresholds (not sandbox thresholds)

2. **Configure notification routing**
   - Critical alerts (5xx, Non-200) → PagerDuty + Slack
   - Capacity alerts (GSU, Spillover) → Email to FinOps team
   - See [Routing Strategy](#best-practices) section

3. **Enable and monitor**
   - Create all policies
   - Observe for first 24 hours
   - Adjust thresholds based on your actual traffic patterns

4. **Document in runbooks**
   - Update your on-call runbook with alert links
   - Document escalation paths
   - Add playbooks for common scenarios

---

## Best Practices

### 1. Avoiding False Positives

**Use guardrails for all alerts:**
- Every alert should have minimum request/token floor
- Low-traffic endpoints shouldn't trigger warnings
- Test thresholds in staging first

**Environment-specific thresholds:**

| Environment | Non-200 Guardrail | GSU Guardrail | Spillover Guardrail |
|-------------|-------------------|---------------|-------------------|
| Dev/Test | 50 requests/window | 1,000 tokens | 500 tokens |
| Staging | 500 requests | 10,000 tokens | 5,000 tokens |
| Production | 1,000+ requests | 50,000+ tokens | 25,000+ tokens |

### 2. Maintaining Label Consistency

**Rules for all alerts:**
- Always group by `resource_container` and `model_user_id`
- Never drop labels in aggregations
- Test PromQL in Metrics Explorer before deploying

**Pattern to follow:**
```promql
(
  sum by (resource_container, model_user_id) (
    rate(metric_name[window])
  )
  / 
  sum by (resource_container, model_user_id) (
    rate(metric_total[window])
  )
) > threshold
and
(
  sum by (resource_container, model_user_id) (
    increase(metric_total[window])
  ) > guardrail
)
```

### 3. Notification Routing Strategy

**Don't send everything to on-call:**

| Alert Type | Severity | Channel | Team |
|-----------|----------|---------|------|
| Non-200 >10% | CRITICAL | PagerDuty | On-Call SRE |
| 5xx Errors | CRITICAL | PagerDuty + Slack | Platform SRE |
| GSU Burndown | WARNING | Email | Capacity Planning |
| Spillover >20% | WARNING | Email | FinOps |

**Rationale:**
- Capacity planning doesn't need 3 AM pages
- Cost alerts are business decisions, not incidents
- Only reliability issues need immediate escalation

### 4. Alert Naming Convention

Use hierarchical naming for easy filtering:

```
VertexAI / {Category} / {AlertName}

Examples:
- VertexAI / Errors / Non-200 Rate
- VertexAI / Errors / 5xx Server Errors
- VertexAI / Capacity / GSU Burndown
- VertexAI / Capacity / Predictive Spillover
- VertexAI / Cost / Spillover Ratio
```

### 5. Regular Tuning

**After 1 week:**
- Review alert frequency
- Check for patterns in false positives
- Adjust thresholds up/down as needed

**After 1 month:**
- Analyze all alert incidents
- Calculate signal-to-noise ratio
- Update guardrails for your baseline

**Quarterly review:**
- Re-evaluate thresholds as traffic scales
- Update runbooks based on common incidents
- Add new alerts for emerging issues

---

## Troubleshooting

### Problem: Variables Show `(null)`

**Symptom:**
```
Alert: Project: (null) | Model: (null)
```

**Root Causes:**
1. Labels dropped during query execution
2. Logical operations (`and`, `or`) change label behavior
3. Metric doesn't have the expected labels

**Solutions:**
1. **Check PromQL output:**
   ```promql
   # In Metrics Explorer, run the exact alert PromQL
   # Check legend: do you see resource_container and model_user_id?
   ```

2. **Ensure label preservation:**
   ```promql
   # Make sure every aggregation includes the grouping
   sum by (resource_container, model_user_id) (metric)
   ```

3. **Verify metric labels:**
   ```promql
   # Check what labels this metric actually has
   topk(5, metric_name)
   # Look at the legend
   ```

---

### Problem: Alert Fires Too Frequently

**Symptom:**
```
Alert triggers multiple times per hour
```

**Causes:**
1. Threshold too low
2. Guardrail too low
3. Metric is naturally spiky

**Solutions:**
1. **Increase guardrail:**
   ```promql
   # Was: > 50 requests
   # Now: > 200 requests (less sensitive to noise)
   ```

2. **Increase window:**
   ```promql
   # Was: [5m] window
   # Now: [15m] window (averages out spikes)
   ```

3. **Adjust threshold:**
   - From 5% to 10% error rate
   - From 0.8 to 0.9 utilization

---

### Problem: Alert Doesn't Fire When Expected

**Symptom:**
```
I intentionally triggered errors but no alert
```

**Causes:**
1. Guardrail not met (too high)
2. Threshold not met
3. Policy not enabled

**Solutions:**
1. **Check guardrail:**
   ```promql
   # Verify: how many requests did you actually send?
   sum(increase(metric[5m]))
   # Must be > your guardrail value
   ```

2. **Check threshold:**
   - Verify metric value actually exceeds threshold
   - Lower threshold temporarily to confirm alert fires

3. **Check policy status:**
   - Alerting → Policies
   - Verify policy is "Enabled" (not paused)
   - Check notification channel is configured

---

### Problem: Can't Find the Right Metrics

**Symptom:**
```
Metrics Explorer shows different metric names
```

**Solution:**
1. **List all available metrics:**
   ```promql
   topk(100, count({job=~".*"}) by (__name__))
   ```

2. **Filter by Vertex AI:**
   - Search: `aiplatform`
   - Shows all available Vertex AI metrics

3. **Verify label names:**
   - Run sample query with `topk()`
   - Check legend for exact label names in your environment
   - Some environments may use different names

---

## Summary

This unified alerting approach provides:

✅ **Single source of truth** for monitoring all projects/models  
✅ **Automatic scale** - new models discovered without config changes  
✅ **Reduced noise** - guardrails prevent false positives  
✅ **Cost visibility** - spillover alerts track budget overruns  
✅ **Proactive capacity** - predictive alerts enable scaling before impact  
✅ **Clear escalation** - routing ensures right team handles each alert type  

**Next Steps:**
1. Review your metrics in Metrics Explorer
2. Create the Non-200 Error Rate alert in staging
3. Test and validate variable expansion
4. Deploy all alerts to production
5. Configure notification routing
6. Document in your runbooks
7. Establish review cadence (weekly → monthly → quarterly)

**Support:**
- Share findings with platform team
- Adjust thresholds based on your traffic patterns
- Add custom alerts for business-specific metrics
- Integrate with incident management (PagerDuty, Opsgenie)

---

**Document Version:** 1.0  
**Last Reviewed:** March 17, 2026  
**Next Review:** June 17, 2026
