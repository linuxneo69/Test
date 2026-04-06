````markdown
# Vertex AI Latency Alert Policies (UAT / Beta Metrics)

## Overview

This document defines the alert policies for monitoring Vertex AI serving latency in Google Cloud Monitoring.

These alerts are intended to detect sustained degradation in:
- **First token latency**
- **Model invocation latency**

The metrics used for these policies are currently marked **BETA** in Google Cloud documentation, so these alerts should be validated in UAT or a non-production environment before rollout to production.

---

## Validation Notes

- Use the exact series and label names returned by **Metrics Explorer** in your environment.
- Prefer `${resource.project}` for project context.
- Use `${resource.label.model_user_id}` for model context if it resolves correctly in alert notifications.
- The public metric catalog documents labels such as `request_type`, `explicit_caching`, and `latency_type`. Validate `model_user_id` in your environment before relying on it in production notifications.

---

# 1. First Token Latency Alert

## Purpose

Detect delays in the time taken to return the **first token** of a Vertex AI response.

## Official Metric

- Metric family: `aiplatform.googleapis.com/PublisherModel`
- Metric type: `publisher/online_serving/first_token_latencies`
- Metric kind: `DELTA`
- Value type: `DISTRIBUTION`
- Unit: `ms`

## Important Labels

Use the labels visible in your environment, typically:
- `request_type`
- `explicit_caching`
- `resource_container`
- `model_user_id` (if exposed in your alert payload)

## Alert Thresholds

| Severity | Condition | Window |
|---|---|---|
| Warning | p95 > 30,000 ms | 30 minutes |
| Critical | p99 > 180,000 ms | 30 minutes |

## PromQL

### p95
```promql
histogram_quantile(
  0.95,
  sum by (le, resource_container, model_user_id, request_type, explicit_caching) (
    rate(aiplatform_googleapis_com:publisher_online_serving_first_token_latencies_bucket[30m])
  )
) > 30000
````

### p99

```promql
histogram_quantile(
  0.99,
  sum by (le, resource_container, model_user_id, request_type, explicit_caching) (
    rate(aiplatform_googleapis_com:publisher_online_serving_first_token_latencies_bucket[30m])
  )
) > 180000
```

## Policy Name

`VertexAI / Latency / First Token`

## Condition Names

* `First Token Latency - p95 - 30m`
* `First Token Latency - p99 - 30m`

## Suggested Tags

* `vertexai`
* `latency`
* `first-token`
* `performance`
* `uat`

## Subject Line

```text
[VertexAI][Latency] First Token Delay | Project: ${resource.project} | Model: ${resource.model_user_id}
```

## Documentation Body

```markdown
**Alert:** First token latency exceeded threshold.

**Project:** ${resource.project}  
**Model:** ${resource.model_user_id}  
**Request Type:** ${metric.label.request_type}  
**Observed Value:** ${metric.label.value}

**Impact**
- Slow response start time
- Reduced streaming experience
- Higher timeout risk

**Immediate Actions**
1. Check recent traffic and request spikes.
2. Review GSU utilization and spillover.
3. Compare the behavior across projects and models.
4. Confirm whether the issue is isolated to a specific request type.
```

---

# 2. Model Invocation Latency Alert

## Purpose

Detect delays in the **end-to-end model invocation** response time.

## Official Metric

* Metric family: `aiplatform.googleapis.com/PublisherModel`
* Metric type: `publisher/online_serving/model_invocation_latencies`
* Metric kind: `DELTA`
* Value type: `DISTRIBUTION`
* Unit: `ms`

## Important Labels

Use the labels visible in your environment, typically:

* `request_type`
* `latency_type`
* `explicit_caching`
* `resource_container`
* `model_user_id` (if exposed in your alert payload)

## Alert Thresholds

| Severity | Condition       | Window     |
| -------- | --------------- | ---------- |
| Warning  | p95 > 15,000 ms | 30 minutes |
| Critical | p99 > 42,000 ms | 30 minutes |

## PromQL

### p95

```promql
histogram_quantile(
  0.95,
  sum by (le, resource_container, model_user_id, request_type, latency_type, explicit_caching) (
    rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_latencies_bucket[30m])
  )
) > 15000
```

### p99

```promql
histogram_quantile(
  0.99,
  sum by (le, resource_container, model_user_id, request_type, latency_type, explicit_caching) (
    rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_latencies_bucket[30m])
  )
) > 42000
```

## Policy Name

`VertexAI / Latency / Model Invocation`

## Condition Names

* `Model Invocation Latency - p95 - 30m`
* `Model Invocation Latency - p99 - 30m`

## Suggested Tags

* `vertexai`
* `latency`
* `invocation`
* `performance`
* `uat`

## Subject Line

```text
[VertexAI][Latency] Invocation Delay | Project: ${resource.project} | Model: ${resource.label.model_user_id}
```

## Documentation Body

```markdown
**Alert:** Model invocation latency exceeded threshold.

**Project:** ${resource.project}  
**Model:** ${resource.label.model_user_id}  
**Request Type:** ${metric.label.request_type}  
**Latency Type:** ${metric.label.latency_type}  
**Observed Value:** ${metric.label.value}

**Impact**
- Slow end-to-end response time
- Increased timeout risk
- Potential backend saturation

**Immediate Actions**
1. Check traffic volume and recent spikes.
2. Compare latency with error-rate and spillover alerts.
3. Review whether the slowdown is model-specific or broad.
4. Validate recent deployment or configuration changes.
```

---

# Implementation Notes

1. Validate both metric series in **Metrics Explorer** before creating alert policies.
2. Confirm the exact `_bucket` series names in your environment.
3. Keep these policies in **UAT/testing** until the BETA metrics behave consistently.
4. Use `${resource.project}` in notification templates for stable project context.
5. Only rely on `${resource.label.model_user_id}` if your alert notifications consistently expand it.

---

# Testing Checklist

* [ ] Metrics appear in Metrics Explorer.
* [ ] Bucket series names are correct.
* [ ] p95 and p99 queries return expected values.
* [ ] Alert notification expands project correctly.
* [ ] Alert notification expands model correctly.
* [ ] Alert fires only when the threshold is sustained over 30 minutes.
* [ ] Alert clears after the latency normalizes.

---

# Notes for Production Readiness

These latency policies should be treated as **observability validation alerts** until the underlying BETA metrics stabilize. The recommended path is:

1. Test in UAT
2. Confirm label behavior in alert notifications
3. Verify thresholds with real traffic
4. Promote only after the behavior is stable across projects and models

```

Review note: the public Google Cloud metric catalog confirms the two latency metrics are BETA and documents the labels shown above, but it does not publicly list `model_user_id`, so the model field should be validated in your alert payload before you rely on it as a hard production dependency. :contentReference[oaicite:1]{index=1}


[1]: https://docs.cloud.google.com/monitoring/api/metrics_gcp_a_b "Google Cloud metrics: A through B  |  Cloud Monitoring  |  Google Cloud Documentation"
