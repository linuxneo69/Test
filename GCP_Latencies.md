# Vertex AI Latency Alert Policies

## Overview

These alert policies monitor Vertex AI serving latency for foundation-model traffic in Cloud Monitoring. The metrics used here are still marked **BETA** in Google’s metric catalog, so the policies should first be validated in UAT or a non-production environment before wider rollout. ([Google Cloud Documentation][1])

## 1. First Token Latency

### Purpose

Detect delays in the time taken to return the first token in streaming or token-based generation flows. Google’s metric catalog names this signal **First token latencies** and marks it **BETA**. ([Google Cloud Documentation][1])

### Official metric

* `aiplatform.googleapis.com/PublisherModel`
* `publisher/online_serving/first_token_latencies` ([Google Cloud Documentation][1])

### Suggested alert conditions

* **Warning:** p95 > 2s
* **Critical:** p99 > 3s

### PromQL examples

Use the exact bucket series name exposed by your environment in Metrics Explorer.

```promql
histogram_quantile(
  0.95,
  sum by (le, resource_container, request_type, explicit_caching) (
    rate(aiplatform_googleapis_com:publisher_online_serving_first_token_latencies_bucket[5m])
  )
) > 2000
```

```promql
histogram_quantile(
  0.99,
  sum by (le, resource_container, request_type, explicit_caching) (
    rate(aiplatform_googleapis_com:publisher_online_serving_first_token_latencies_bucket[5m])
  )
) > 3000
```

### Policy name

* `VertexAI / Latency / First Token`

### Condition name

* `First Token Latency — p95`
* `First Token Latency — p99`

### Tags

* `vertexai`
* `latency`
* `first-token`
* `prompt-response`
* `uat`

### Subject line

```text
[VertexAI][Latency] First Token Latency High | Project: ${resource.project} | Request Type: ${metric.label.request_type}
```

### Documentation

```markdown
**Alert:** First token latency is above threshold.

**Project:** ${resource.project}
**Request Type:** ${metric.label.request_type}
**Observed Value:** ${metric.label.value}

**Impact**
- Slow response start time
- Poor streaming experience
- Increased chance of timeout or user drop-off

**Immediate actions**
1. Check recent traffic spikes.
2. Review GSU and spillover conditions.
3. Compare latency across models and environments.
4. Validate whether the issue is isolated to a specific request type.
```

---

## 2. Model Invocation Latency

### Purpose

Detect end-to-end model invocation latency for Vertex AI serving traffic. Google’s metric catalog names this signal **Model invocation latencies** and marks it **BETA**. ([Google Cloud Documentation][1])

### Official metric

* `aiplatform.googleapis.com/PublisherModel`
* `publisher/online_serving/model_invocation_latencies` ([Google Cloud Documentation][1])

### Suggested alert conditions

* **Warning:** p95 > 3s
* **Critical:** p99 > 5s

### PromQL examples

Again, verify the exact bucket series name in Metrics Explorer before finalizing.

```promql
histogram_quantile(
  0.95,
  sum by (le, resource_container, request_type, latency_type, explicit_caching) (
    rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_latencies_bucket[5m])
  )
) > 3000
```

```promql
histogram_quantile(
  0.99,
  sum by (le, resource_container, request_type, latency_type, explicit_caching) (
    rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_latencies_bucket[5m])
  )
) > 5000
```

### Policy name

* `VertexAI / Latency / Model Invocation`

### Condition name

* `Model Invocation Latency — p95`
* `Model Invocation Latency — p99`

### Tags

* `vertexai`
* `latency`
* `invocation`
* `performance`
* `uat`

### Subject line

```text
[VertexAI][Latency] Model Invocation Latency High | Project: ${resource.project} | Request Type: ${metric.label.request_type}
```

### Documentation

```markdown
**Alert:** Model invocation latency is above threshold.

**Project:** ${resource.project}
**Request Type:** ${metric.label.request_type}
**Latency Type:** ${metric.label.latency_type}
**Observed Value:** ${metric.label.value}

**Impact**
- Slow end-to-end responses
- Higher timeout risk
- Potential saturation or capacity pressure

**Immediate actions**
1. Check traffic volume and recent spikes.
2. Review GSU burndown and spillover alerts.
3. Validate whether the slowdown is model-specific or shared across models.
4. Confirm whether explicit caching is affecting the observed latency.
```

---

## Correctness review and improvements

1. The earlier wording “token latency” is better written as **first token latency**, because that is the official metric name Google uses in the catalog. ([Google Cloud Documentation][1])

2. These metrics are **BETA**, so keeping them in UAT/testing first is the safer choice. ([Google Cloud Documentation][1])

3. The official descriptors for these metrics show labels like `request_type`, `explicit_caching`, `accounting_resource`, `latency_type`, and token-size dimensions. I would **not** hardcode `model_user_id` into the document unless Metrics Explorer confirms that label exists in your exported series. ([Google Cloud Documentation][1])

4. For the PromQL queries, the exact `_bucket` series name should be verified in **Metrics Explorer** before rollout, because the Cloud Monitoring catalog lists the metric types, while the PromQL-exported series name is best confirmed in your environment.

[1]: https://docs.cloud.google.com/monitoring/api/metrics_gcp_a_b "Google Cloud metrics: A through B  |  Cloud Monitoring  |  Google Cloud Documentation"
