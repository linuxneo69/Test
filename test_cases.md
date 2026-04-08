# Vertex AI / GCO SRE Alerting Documentation

## Executive summary

This document defines the alerting framework for Vertex AI monitoring in the GCO unified observability setup. The focus is on latency, capacity burn down, HTTP/error rates, PT spillover, and predictive saturation. Google documents the relevant Vertex AI PublisherModel metrics as **BETA**, PT monitoring as **public preview**, and Cloud Monitoring explicitly warns that notification variables can become `null` if aggregation drops the label you want to display. For that reason, this framework is intended for UAT-first validation before production rollout. ([Google Cloud Documentation][1])

---

## 1. Core concepts

### 1.1 First Token Latency

First Token Latency measures how long it takes for a model to return the first token after a request is received. Google’s metric catalog lists `publisher/online_serving/first_token_latencies` as a **BETA** PublisherModel distribution metric in milliseconds. The catalog describes it as the duration from request receipt to first token returned to the client. ([Google Cloud Documentation][1])

### 1.2 Model Invocation Latency

Model Invocation Latency measures the overall time for a model invocation. Google’s metric catalog lists `publisher/online_serving/model_invocation_latencies` as a **BETA** PublisherModel distribution metric in milliseconds. The public catalog also shows this metric family with labels such as `latency_type`, `request_type`, `explicit_caching`, `input_token_size`, and `output_token_size`. ([Google Cloud Documentation][1])

### 1.3 P95 and P99

* **P95** means 95% of observations are at or below the threshold.
* **P99** means 99% of observations are at or below the threshold.

These are tail-latency signals, so they are much more useful than averages for detecting user-impacting slowdown.

### 1.4 `request_type` versus latency labels

These are different dimensions and should not be mixed up:

* `request_type` describes the traffic path:

  * `dedicated`
  * `shared`
  * `spillover`
* latency labels describe which latency slice you are measuring.

### Latency type values

For Vertex AI latency metrics, `latency_type` identifies which part of the request is being measured.

- `model` = time spent in the model execution itself
- `total` = full end-to-end latency seen by the user

For Vertex AI Provisioned Throughput, Google documents `dedicated`, `shared`, and spillover behavior, and notes that spillover can be processed as pay-as-you-go after the PT quota is exceeded. ([Google Cloud Documentation][2])

### 1.5 Dedicated / Shared / Spillover

* **Dedicated**: traffic served through Provisioned Throughput.
* **Shared**: pay-as-you-go traffic.
* **Spillover**: traffic that exceeds Provisioned Throughput quota and is processed as pay-as-you-go.

Google’s PT docs also say PT monitoring is in **public preview**, and that dashboards may show dedicated and spillover lines separately depending on the enforcement window. ([Google Cloud Documentation][2])

---

## 2. Alert design principles

1. Keep the alert scope aligned to the signal you actually control.
2. Preserve the labels you need in the final evaluated series. Google says if aggregation removes a label, the notification variable becomes `null`. ([Google Cloud Documentation][3])
3. Prefer `${resource.project}` for project context when it is stable in your environment. Cloud Monitoring’s incident labels section shows the monitored resource labels and metric labels for the time series that caused the incident. ([Google Cloud Documentation][4])
4. Use the model label only if it survives in the final incident payload and renders reliably in notifications.
5. Keep PT-related alerts separate from general reliability alerts.
6. Treat BETA metrics as UAT-first until the behavior stabilizes. ([Google Cloud Documentation][1])

---

## 3. Policy matrix

| #  | Policy                                 | Severity | Window       | Threshold                   | Operational meaning              |
| -- | -------------------------------------- | -------- | ------------ | --------------------------- | -------------------------------- |
| 1  | First Token Latency P95                | Warning  | 30m          | > 30,000 ms                 | Early response-start degradation |
| 2  | First Token Latency P99                | Critical | 30m          | > 180,000 ms                | Severe response-start delay      |
| 3  | Model Invocation Latency P95           | Warning  | 30m          | > 15,000 ms                 | End-to-end slowdown              |
| 4  | Model Invocation Latency P99           | Critical | 30m          | > 42,000 ms                 | Severe tail latency              |
| 5  | GSU Burn Down at 75%                   | Warning  | 30m          | > 75%                       | Capacity tightening              |
| 6  | GSU Burn Down at 90%                   | Critical | 30m          | > 90%                       | Near saturation                  |
| 7  | HTTP Error Rate above 2%               | Critical | 35m          | > 2% on 400/429/499/500/503 | User-visible failures            |
| 8  | PT Spillover Ratio above 10%           | Warning  | 15m          | > 10%                       | Early spillover warning          |
| 9  | PT Spillover Ratio above 25%           | Critical | 15m          | > 25%                       | Heavy spillover / cost risk      |
| 10 | Predicted GSU saturation in 10 minutes | Critical | 10m forecast | Saturation predicted        | Proactive capacity alert         |
| 11 | Non-200 rate                           | Warning  | 15m          | > 10%                       | Supplemental reliability signal  |

---

## 4. Detailed alert policies

### 4.1 First Token Latency P95

**Condition name**
`First Token Latency - P95 - 30m`

**Purpose**
Detect early degradation in the time to first token.

**Why it matters**
If first-token P95 rises, users are waiting longer before any visible output appears. This is often the earliest user-experience warning before the tail becomes critical. Google documents this metric as a BETA PublisherModel distribution metric. ([Google Cloud Documentation][1])

**Threshold**
P95 > 30,000 ms over 30 minutes.

**Operational meaning**
Response start time is slowing down and the user experience is starting to degrade.

**Recommended tags**
`vertexai`, `latency`, `first-token`, `warning`

**Suggested subject line**
`[VertexAI][Latency][P95] First Token Delay | Project: ${resource.project} | Model: ${resource.model_user_id}`

**Suggested documentation**
**Alert:** First token latency P95 exceeded 30,000 ms over 30 minutes.

**Project:** `${resource.project}`
**Model:** `${resource.model_user_id}`
**Observed Value:** `${metric.label.value}`

**Impact**

* Response start time is degrading
* User experience may be impacted
* This may precede a more severe latency event

**Actions**

1. Check traffic spikes and recent request volume.
2. Review burn down and spillover alerts.
3. Validate whether the slowdown is isolated to one model.

---

### 4.2 First Token Latency P99

**Condition name**
`First Token Latency - P99 - 30m`

**Purpose**
Detect severe delays in the time to first token.

**Why it matters**
A high first-token P99 means the slowest requests are experiencing a very poor start-to-response experience. That is often a strong signal of queuing, backend pressure, or model warm-up issues. Google documents this metric as a BETA PublisherModel distribution metric. ([Google Cloud Documentation][1])

**Threshold**
P99 > 180,000 ms over 30 minutes.

**Operational meaning**
Users are waiting far too long before the first token appears.

**Recommended tags**
`vertexai`, `latency`, `first-token`, `p99`, `critical`

**Suggested subject line**
`[VertexAI][Latency][P99] First Token Slow | Project: ${resource.project} | Model: ${resource.model_user_id}`

**Suggested documentation**
**Alert:** First token latency P99 exceeded 180,000 ms over 30 minutes.

**Project:** `${resource.project}`
**Model:** `${resource.model_user_id}`
**Observed Value:** `${metric.label.value}`

**Impact**

* Slow response start time
* Higher timeout risk
* Poor user experience

**Actions**

1. Check traffic spikes and recent request volume.
2. Review burn down and spillover alerts.
3. Validate whether the slowdown is isolated to one model.

---

### 4.3 Model Invocation Latency P95

**Condition name**
`Model Invocation Latency - P95 - 30m`

**Purpose**
Detect broad end-to-end request slowdown.

**Why it matters**
This indicates the full invocation path is slowing down, not just the first token. Google documents `publisher/online_serving/model_invocation_latencies` as a BETA PublisherModel metric and shows the latency label dimensions for the family. ([Google Cloud Documentation][1])

**Threshold**
P95 > 15,000 ms over 30 minutes.

**Operational meaning**
The request path is getting slower overall and may be approaching saturation.

**Recommended tags**
`vertexai`, `latency`, `invocation`, `warning`

**Suggested subject line**
`[VertexAI][Latency][P95] Invocation Delay | Project: ${resource.project} | Model: ${resource.model_user_id}`

**Suggested documentation**
**Alert:** Model invocation latency P95 exceeded 15,000 ms over 30 minutes.

**Project:** `${resource.project}`
**Model:** `${resource.model_user_id}`
**Observed Value:** `${metric.label.value}`

**Impact**

* Slower end-to-end responses
* Increased timeout risk
* Possible backend saturation

**Actions**

1. Check request volume and recent spikes.
2. Compare against error-rate and capacity alerts.
3. Validate whether the slowdown is model-specific.

---

### 4.4 Model Invocation Latency P99

**Condition name**
`Model Invocation Latency - P99 - 30m`

**Purpose**
Detect severe tail latency.

**Why it matters**
A high P99 means a small but important slice of traffic is taking much longer than expected. That often causes retries, timeouts, or user-visible degradation. Google documents this metric family as BETA. ([Google Cloud Documentation][1])

**Threshold**
P99 > 42,000 ms over 30 minutes.

**Operational meaning**
The slowest requests are extremely delayed.

**Recommended tags**
`vertexai`, `latency`, `invocation`, `p99`, `critical`

**Suggested subject line**
`[VertexAI][Latency][P99] Invocation Slow | Project: ${resource.project} | Model: ${resource.model_user_id}`

**Suggested documentation**
**Alert:** Model invocation latency P99 exceeded 42,000 ms over 30 minutes.

**Project:** `${resource.project}`
**Model:** `${resource.model_user_id}`
**Observed Value:** `${metric.label.value}`

**Impact**

* Severe tail latency
* Higher timeout risk
* User-visible slowdowns

**Actions**

1. Check traffic spikes and recent request patterns.
2. Compare against capacity and error alerts.
3. Validate whether the issue is localized or broad.

---

### 4.5 GSU Burn Down at 75%

**Condition name**
`GSU Burn Down - 75% - 30m`

**Purpose**
Provide an early capacity warning.

**Why it matters**
This is the first clear sign that provisioned capacity is getting tight and spillover risk is increasing.

**Threshold**
Utilization > 75% over 30 minutes.

**Operational meaning**
Capacity is tightening, but there is still time to react before saturation becomes critical.

**Recommended tags**
`vertexai`, `capacity`, `gsu`, `warning`

**Suggested subject line**
`[VertexAI][Capacity][Warning] GSU Burn Down >75% | Project: ${resource.project} | Model: ${resource.model_user_id}`

**Suggested documentation**
**Alert:** Dedicated capacity utilization exceeded 75% over 30 minutes.

**Project:** `${resource.project}`
**Model:** `${resource.model_user_id}`
**Observed Value:** `${metric.label.value}`

**Impact**

* Capacity is tightening
* Spillover risk is increasing
* Latency may rise soon

**Actions**

1. Review current throughput trends.
2. Check for traffic spikes or deployment changes.
3. Compare with latency and error alerts.
4. Plan capacity increase if usage remains sustained.

---

### 4.6 GSU Burn Down at 90%

**Condition name**
`GSU Burn Down - 90% - 30m`

**Purpose**
Critical near-saturation alert.

**Why it matters**
At this point, action is usually required to avoid performance degradation or fallback traffic.

**Threshold**
Utilization > 90% over 30 minutes.

**Operational meaning**
Provisioned capacity is near exhaustion.

**Recommended tags**
`vertexai`, `capacity`, `gsu`, `critical`

**Suggested subject line**
`[VertexAI][Capacity][Critical] GSU Burn Down >90% | Project: ${resource.project} | Model: ${resource.model_user_id}`

**Suggested documentation**
**Alert:** Dedicated capacity utilization exceeded 90% over 30 minutes.

**Project:** `${resource.project}`
**Model:** `${resource.model_user_id}`
**Observed Value:** `${metric.label.value}`

**Impact**

* Dedicated capacity is near exhaustion
* Spillover or degraded performance may occur
* Immediate capacity review is needed

**Actions**

1. Review current throughput trends.
2. Check for traffic spikes or deployment changes.
3. Decide whether to increase capacity or reduce load.
4. Compare with spillover and latency alerts.

---

### 4.7 HTTP Error Rate above 2%

**Condition name**
`HTTP Error Rate - 2% - 35m`

**Purpose**
Detect meaningful request failures for response codes 400, 429, 499, 500, and 503.

**Why it matters**
This captures client errors, throttling, backend faults, and saturation-related failures.

**Threshold**
Error rate > 2% over 35 minutes.

**Operational meaning**
User-visible failures are increasing and should be triaged immediately.

**Recommended tags**
`vertexai`, `errors`, `http`, `critical`

**Suggested subject line**
`[VertexAI][Errors] HTTP Error Rate >2% | Project: ${resource.project} | Model: ${resource.model_user_id}`

**Suggested documentation**
**Alert:** HTTP error rate exceeded 2% over 35 minutes.

**Project:** `${resource.project}`
**Model:** `${resource.model_user_id}`
**Observed Value:** `${metric.label.value}`

**Response codes included**

* 400
* 429
* 499
* 500
* 503

**Impact**

* User-visible request failures
* Throttling or backend instability
* Possible model, routing, or capacity issue

**Actions**

1. Check whether errors are concentrated in one code.
2. Review recent deployment or routing changes.
3. Correlate with logs and capacity alerts.
4. Identify whether the issue is client-side, backend, or quota-related.

---

### 4.8 PT Spillover Ratio above 10%

**Condition name**
`PT Spillover Ratio - 10% - 15m`

**Purpose**
Detect early spillover from Provisioned Throughput into pay-as-you-go traffic. Google explains that when PT quota is exceeded, traffic is processed as pay-as-you-go and shown as spillover in monitoring views. ([Google Cloud Documentation][2])

**Threshold**
Spillover ratio > 10% over 15 minutes.

**Operational meaning**
PT capacity is being used aggressively and fallback traffic is starting to appear.

**Recommended tags**
`vertexai`, `pt`, `spillover`, `warning`

**Suggested subject line**
`[VertexAI][PT][Warning] Spillover >10% | Project: ${resource.project} | Model: ${resource.model_user_id}`

**Suggested documentation**
**Alert:** PT spillover ratio exceeded 10% over 15 minutes.

**Project:** `${resource.project}`
**Model:** `${resource.model_user_id}`
**Observed Value:** `${metric.label.value}`

**Impact**

* Traffic is starting to fall back to pay-as-you-go
* Cost and latency risk is increasing
* Capacity review may be needed

**Actions**

1. Check whether PT is nearing exhaustion.
2. Review current demand and recent changes.
3. Compare against burn down and latency alerts.
4. Decide whether capacity needs to be adjusted.

---

### 4.9 PT Spillover Ratio above 25%

**Condition name**
`PT Spillover Ratio - 25% - 15m`

**Purpose**
Detect heavy spillover and high PT pressure.

**Why it matters**
A large share of traffic is no longer being handled by provisioned capacity.

**Threshold**
Spillover ratio > 25% over 15 minutes.

**Operational meaning**
The provisioned path is no longer covering a substantial part of traffic.

**Recommended tags**
`vertexai`, `pt`, `spillover`, `critical`

**Suggested subject line**
`[VertexAI][PT][Critical] Spillover >25% | Project: ${resource.project} | Model: ${resource.model_user_id}`

**Suggested documentation**
**Alert:** PT spillover ratio exceeded 25% over 15 minutes.

**Project:** `${resource.project}`
**Model:** `${resource.model_user_id}`
**Observed Value:** `${metric.label.value}`

**Impact**

* Significant traffic is leaving provisioned capacity
* Pay-as-you-go usage is increasing
* Immediate PT review is required

**Actions**

1. Review capacity and current demand.
2. Check for traffic bursts or demand growth.
3. Compare with latency and error alerts.
4. Plan immediate mitigation if this is sustained.

---

### 4.10 Predicted GSU saturation in 10 minutes

**Condition name**
`Predicted GSU Saturation - 10m`

**Purpose**
Forecast capacity exhaustion before the actual limit is reached.

**Why it matters**
This is the proactive version of burn down monitoring. It gives you time to act before spillover or major latency spikes begin.

**Threshold**
Forecasted utilization crosses the limit in the next 10 minutes.

**Operational meaning**
Capacity exhaustion is imminent.

**Recommended tags**
`vertexai`, `capacity`, `predictive`, `critical`

**Suggested subject line**
`[VertexAI][Capacity][Predictive] GSU Saturation in 10m | Project: ${resource.project} | Model: ${resource.model_user_id}`

**Suggested documentation**
**Alert:** Capacity is predicted to saturate within 10 minutes.

**Project:** `${resource.project}`
**Model:** `${resource.model_user_id}`
**Observed Value:** `${metric.label.value}`

**Impact**

* Imminent capacity exhaustion
* Useful for pre-emptive mitigation
* Can help reduce spillover and latency spikes

**Actions**

1. Review recent throughput growth.
2. Compare against burn down and spillover alerts.
3. Decide whether capacity should be increased now.
4. Validate whether the growth is expected or anomalous.

---

### 4.11 Non-200 rate

**Condition name**
`Non-200 Rate - 15m`

**Purpose**
Provide a supplemental reliability signal for broader error behavior.

**Why it matters**
It captures abnormal behavior that may not map cleanly to the narrower HTTP code list.

**Threshold**
Non-200 rate > 10% over 15 minutes.

**Operational meaning**
A general reliability issue is present.

**Recommended tags**
`vertexai`, `errors`, `non-200`, `warning`

**Suggested subject line**
`[VertexAI][Errors][Warning] Non-200 Rate High | Project: ${resource.project} | Model: ${resource.model_user_id}`

**Suggested documentation**
**Alert:** Non-200 response rate exceeded threshold over 15 minutes.

**Project:** `${resource.project}`
**Model:** `${resource.model_user_id}`
**Observed Value:** `${metric.label.value}`

**Impact**

* Wider reliability issue
* Useful as a complementary signal to HTTP code-based alerts

---

## 5. Label strategy

Cloud Monitoring says metric, resource, and metadata variables can become `null` if aggregation removes the label. It also notes that the incident Labels section shows the monitored resource labels and metric labels for the time series that caused the incident. ([Google Cloud Documentation][3])

Recommended practice:

* Use `${resource.project}` for project context.
* Use the model label only if it survives in the final incident payload.
* Confirm variable expansion with a real incident or test notification.
* Keep aggregation and grouping aligned with the labels you want in the notification.

---

## 6. Implementation and validation

### 6.1 Implementation steps

1. Confirm the metric is visible in Metrics Explorer.
2. Confirm the exact labels in the query output.
3. Confirm whether the metric family is BETA or preview.
4. Create the alert policy.
5. Add the notification channel.
6. Add the subject line and documentation template.
7. Save the policy.
8. Verify the incident preview and Labels section.

### 6.2 Validation checklist

* [ ] Metric appears in Metrics Explorer.
* [ ] Query returns a visible time series.
* [ ] Project label expands correctly.
* [ ] Model label expands correctly, if required in the notification.
* [ ] Alert does not fire during normal traffic.
* [ ] Alert fires when the threshold is intentionally breached.
* [ ] Dashboard charts match Metrics Explorer output.

### 6.3 Missing-data behavior

If a label becomes `null`, Cloud Monitoring says the aggregation likely removed it. For low-traffic environments, a short window may also simply mean there is no activity, so missing data should be interpreted carefully rather than treated as a failure by default. ([Google Cloud Documentation][3])

---

## 7. Dashboard and troubleshooting notes

Google’s Vertex AI model observability dashboard is useful for confirming first-token latency, API error rates, and token throughput, but it should be treated as a validation layer rather than the only source of truth. Google also notes that the dashboard only shows a subset of collected Cloud Monitoring metrics. ([Google Cloud Documentation][5])

Recommended panels:

* first token latency p95/p99
* model invocation latency p95/p99
* GSU burn down
* PT spillover ratio
* HTTP error rate
* non-200 rate
* predicted saturation line

---

## 8. Runbook

### If a label renders as `null`

Check whether aggregation removed the label. Google’s troubleshooting guidance says to verify the aggregation settings and preserve the label you want to display. ([Google Cloud Documentation][3])

### If PT alerting does not match the dashboard

Remember that PT monitoring is in public preview and spillover can appear differently depending on the enforcement window. Validate the behavior across the same time window you use for the alert. ([Google Cloud Documentation][2])

### If latency values look inconsistent

Treat the latency families as BETA and validate the exact label values in Metrics Explorer before relying on them in production notifications. Google’s catalog documents the latency metrics and their label dimensions. ([Google Cloud Documentation][1])

---

## 9. Final recommendations

* Keep the documentation and alerting model aligned with the labels you actually see in the incident payload.
* Use `${resource.project}` consistently for project context.
* Validate model labels before making them mandatory in notification templates.
* Keep PT alerting, latency alerting, and error alerting separate so each policy stays actionable.
* Treat these metrics as UAT-first until the behavior stabilizes in your environment. ([Google Cloud Documentation][1])

---

## 10. References

* Google Cloud metrics: A through B — Vertex AI PublisherModel metric catalog. ([Google Cloud Documentation][1])
* Use Provisioned Throughput. ([Google Cloud Documentation][2])
* Monitor models. ([Google Cloud Documentation][5])
* Annotate notifications with user-defined documentation. ([Google Cloud Documentation][6])
* Troubleshoot alerting policies. ([Google Cloud Documentation][3])
* Incidents for metric-based alerting policies. ([Google Cloud Documentation][4])

[1]: https://docs.cloud.google.com/monitoring/api/metrics_gcp_a_b "Google Cloud metrics: A through B  |  Cloud Monitoring  |  Google Cloud Documentation"
[2]: https://docs.cloud.google.com/vertex-ai/generative-ai/docs/provisioned-throughput/use-provisioned-throughput?utm_source=chatgpt.com "Use Provisioned Throughput | Generative AI on Vertex AI"
[3]: https://docs.cloud.google.com/monitoring/alerts/troubleshooting-alerts?utm_source=chatgpt.com "Troubleshoot alerting policies | Cloud Monitoring"
[4]: https://docs.cloud.google.com/monitoring/alerts/incidents-events?utm_source=chatgpt.com "Incidents for metric-based alerting policies | Cloud Monitoring"
[5]: https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/model-observability?utm_source=chatgpt.com "Monitor models | Generative AI on Vertex AI"
[6]: https://docs.cloud.google.com/monitoring/alerts/doc-variables?utm_source=chatgpt.com "Annotate notifications with user-defined documentation"
