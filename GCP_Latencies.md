````markdown
# Vertex AI Latency Alert Policies

## Overview

This document defines the latency alert policies for Vertex AI serving. The alerts are based on Google Cloud Monitoring metrics that are currently marked **BETA**, so they should be validated in UAT before production rollout. The goal is to detect sustained degradation in first-token response time and full model invocation latency. :contentReference[oaicite:1]{index=1}

## Validation Notes

- These metrics are in **BETA** in the official Google Cloud metric catalog. :contentReference[oaicite:2]{index=2}
- Use the exact label keys returned by **Metrics Explorer** in your environment.
- For notifications, prefer `${resource.project}` for project context.
- Include `model_user_id` only if your query output and alert notifications consistently expose it in your environment.

---

# 1. First Token Latency Alert

## Purpose

Detect slow time-to-first-token behavior for Vertex AI model responses.

## Official Metric

- Metric family: `aiplatform.googleapis.com/PublisherModel`
- Metric type: `publisher/online_serving/first_token_latencies`
- Unit: milliseconds
- Metric kind: `DELTA`
- Value type: `DISTRIBUTION`
- Status: `BETA` :contentReference[oaicite:3]{index=3}

## Documented Labels

The official catalog lists labels such as:
- `input_token_size`
- `output_token_size`
- `max_token_size`
- `request_type`
- `explicit_caching` :contentReference[oaicite:4]{index=4}

## Policy Design

Create one policy with two conditions:

- **Warning:** p95 > 30,000 ms over 30 minutes
- **Critical:** p99 > 180,000 ms over 30 minutes

## PromQL

> Adjust the `sum by(...)` labels to match the exact labels you see in Metrics Explorer.

### p95
```promql
histogram_quantile(
  0.95,
  sum by (le, resource_container, request_type, explicit_caching) (
    rate(aiplatform_googleapis_com:publisher_online_serving_first_token_latencies_bucket[30m])
  )
) > 30000
````

### p99

```promql
histogram_quantile(
  0.99,
  sum by (le, resource_container, request_type, explicit_caching) (
    rate(aiplatform_googleapis_com:publisher_online_serving_first_token_latencies_bucket[30m])
  )
) > 180000
```

## Suggested Policy Name

`VertexAI / Latency / First Token`

## Suggested Condition Names

* `First Token Latency - p95 - 30m`
* `First Token Latency - p99 - 30m`

## Suggested Tags

* `vertexai`
* `latency`
* `first-token`
* `uat`
* `performance`

## Subject Line

```text
[VertexAI][Latency] First Token Delay | Project: ${resource.project} | Request Type: ${metric.label.request_type}
```

If `model_user_id` is consistently available in your environment, you can append it as:

```text
| Model: ${metric.label.model_user_id}
```

## Documentation Body

```markdown
**Alert:** First token latency exceeded threshold.

**Project:** ${resource.project}
**Request Type:** ${metric.label.request_type}
**Observed Value:** ${metric.label.value}

**Impact**
- Slow response start time
- Reduced user experience
- Potential timeout risk

**Immediate Actions**
1. Check recent traffic and request spikes.
2. Review GSU utilization and spillover.
3. Compare the behavior across projects and models.
4. Validate whether the issue is isolated to a specific request type.
```

---

# 2. Model Invocation Latency Alert

## Purpose

Detect end-to-end model invocation latency degradation for Vertex AI model responses.

## Official Metric

* Metric family: `aiplatform.googleapis.com/PublisherModel`
* Metric type: `publisher/online_serving/model_invocation_latencies`
* Unit: milliseconds
* Metric kind: `DELTA`
* Value type: `DISTRIBUTION`
* Status: `BETA` ([Google Cloud Documentation][1])

## Documented Labels

The official catalog lists labels such as:

* `input_token_size`
* `output_token_size`
* `max_token_size`
* `latency_type`
* `request_type`
* `explicit_caching` ([Google Cloud Documentation][1])

## Policy Design

Create one policy with two conditions:

* **Warning:** p95 > 15,000 ms over 30 minutes
* **Critical:** p99 > 42,000 ms over 30 minutes

## PromQL

> Adjust the labels in `sum by(...)` to match your Metrics Explorer output.

### p95

```promql
histogram_quantile(
  0.95,
  sum by (le, resource_container, request_type, latency_type, explicit_caching) (
    rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_latencies_bucket[30m])
  )
) > 15000
```

### p99

```promql
histogram_quantile(
  0.99,
  sum by (le, resource_container, request_type, latency_type, explicit_caching) (
    rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_latencies_bucket[30m])
  )
) > 42000
```

## Suggested Policy Name

`VertexAI / Latency / Model Invocation`

## Suggested Condition Names

* `Model Invocation Latency - p95 - 30m`
* `Model Invocation Latency - p99 - 30m`

## Suggested Tags

* `vertexai`
* `latency`
* `invocation`
* `uat`
* `performance`

## Subject Line

```text
[VertexAI][Latency] Invocation Delay | Project: ${resource.project} | Request Type: ${metric.label.request_type}
```

If `model_user_id` is stable in your environment, you may add:

```text
| Model: ${metric.label.model_user_id}
```

## Documentation Body

```markdown
**Alert:** Model invocation latency exceeded threshold.

**Project:** ${resource.project}
**Request Type:** ${metric.label.request_type}
**Latency Type:** ${metric.label.latency_type}
**Observed Value:** ${metric.label.value}

**Impact**
- Slow end-to-end response time
- Higher timeout risk
- Potential backend saturation

**Immediate Actions**
1. Check traffic volume and recent spikes.
2. Compare latency with error-rate and spillover alerts.
3. Review whether the issue is model-specific or broad.
4. Validate any recent deployment or configuration changes.
```

---

# Implementation Notes

1. Validate both metrics in **Metrics Explorer** first.
2. Confirm the exact `_bucket` series name in your environment before creating the alert.
3. Keep these policies in **UAT/testing** until the BETA metrics stabilize. ([Google Cloud Documentation][1])
4. Use `${resource.project}` in notifications for the most reliable project context.
5. Only include model labels if they consistently resolve in your environment.

---

# Review Summary

This version is more correct than the earlier draft because:

* it uses the official metric names from Google’s catalog,
* it clearly marks the metrics as BETA,
* it avoids assuming `model_user_id` is an official catalog label,
* and it keeps the query structure flexible so you can align it to the labels actually returned by Metrics Explorer. ([Google Cloud Documentation][1])

```

[1]: https://docs.cloud.google.com/monitoring/api/metrics_gcp_a_b "Google Cloud metrics: A through B  |  Cloud Monitoring  |  Google Cloud Documentation"
