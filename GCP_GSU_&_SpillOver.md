# Vertex AI — Capacity & Error Alerting (Unified Auto-Discovery)

**Confluence-ready, copy-pasteable documentation**
*Purpose:* single unified monitoring policy set that auto-discovers Vertex AI models across projects and alerts on Errors, GSU (token) Burndown, Spillover, and Predictive Saturation.

---

## 📌 Overview (TL;DR)

* One **auto-discovery** policy set monitors all Vertex AI models across scoped projects.
* Alerts covered here (copy-paste PromQL included):

  1. **Non-200 Error Rate** — if >10% non-200 in last 15m (guardrail: ≥1000 calls)
  2. **GSU Burndown** — if usage >90% of dedicated token limit over 30m (guardrail: ≥10k tokens)
  3. **Spillover** — if spillover >20% of dedicated over 15m (guardrail: ≥5k spillover tokens)
  4. **Predictive GSU Saturation** — predict capacity exhaustion in next 10m (uses `predict_linear`)
  5. **Predictive Spillover** — predict rising spillover (optional)
* Each rule preserves `resource_container` and `model_user_id` in `sum by(...)` so alert templates can reference Project & Model.

---

# 1. Quick Pre-Checks (do **before** you create policies)

1. **Confirm label names** (exact keys) in Metrics Explorer → PromQL:

   ```promql
   topk(20, aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput)
   topk(20, aiplatform_googleapis_com:publisher_online_serving_model_invocation_count)
   topk(20, aiplatform_googleapis_com:publisher_online_serving_dedicated_token_limit)
   ```

   Inspect the series legend — note exact label keys (commonly `resource_container`, `model_user_id`, `response_code`, `request_type`). Use those exact keys everywhere.
2. Confirm the **dedicated token limit** metric exists per model:

   ```promql
   topk(20, aiplatform_googleapis_com:publisher_online_serving_dedicated_token_limit)
   ```
3. Decide guardrail thresholds for your environment (see Recommendations table below).

---

# 2. PromQL — Copy-Pasteable Alert Conditions

> **Important usage notes**
>
> * Paste the entire block into Cloud Monitoring (PromQL mode). Cloud Monitoring does **not** accept `name =` variable assignments — only expressions.
> * `predict_linear` must be given a **range-vector** of the *raw metric selector* (e.g., `metric_name[30m]`). Do **not** put `[30m]` on an aggregation result (that causes `ranges only allowed for vector selectors` error).

---

## 2.1 Non-200 Error Rate (Unified Error)

**When:** Non-200 responses > **10%** in the last **15m** AND at least **1,000** requests in 15m.
**PromQL**

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

**Why:** denominator groups by `resource_container, model_user_id` (no `response_code`), so ratio = non-200 / all requests.

---

## 2.2 GSU Burndown (Static)

**When:** Token throughput / dedicated limit > **90%** over **30m** AND at least **10k** tokens consumed in 30m.
**PromQL**

```promql
(
  sum by (resource_container, model_user_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput[30m])
  )
  /
  sum by (resource_container, model_user_id) (
    aiplatform_googleapis_com:publisher_online_serving_dedicated_token_limit
  )
) > 0.90
and
(
  sum by (resource_container, model_user_id) (
    increase(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput[30m])
  ) > 10000
)
```

**Why:** throughput is tokens/sec (smoothed over 30m). Compare to dedicated capacity (gauge). Guardrail prevents noisy firing on tiny workloads.

---

## 2.3 Spillover (Static)

**When:** Spillover throughput > **20%** of dedicated throughput (over 15m) AND spillover tokens > **5k** in 30m.
**PromQL**

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
and
(
  sum by (resource_container, model_user_id) (
    increase(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput{request_type="spillover"}[30m])
  ) > 5000
)
```

**Notes:** if `request_type` label is not always present in your metrics, compute spillover as `total - dedicated` *only if both metrics exist and label sets match*.

---

## 2.4 Predictive GSU Saturation (Predictive)

**When:** Current trend predicts throughput will exceed dedicated limit within **10 minutes** (600s), and consumption > **10k** tokens in 30m.
**PromQL (WORKING pattern)**

```promql
predict_linear(
  aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput[30m],
  600
)
>
aiplatform_googleapis_com:publisher_online_serving_dedicated_token_limit
and
(
  sum by (resource_container, model_user_id) (
    increase(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput[30m])
  ) > 10000
)
```

**Why this form works**

* `predict_linear(metric[30m],600)` accepts the raw metric range-vector, projects the per-series value 600s into the future and returns an instant vector with the same labels as the metric.
* Compare the projected value to the dedicated limit metric (which is usually a per-model gauge). If dedicated_limit requires aggregation first, do that carefully — **do not** apply `[30m]` to an aggregation result.

**Tuning**

* prediction window: 600s (10m) recommended; can use 300s / 900s depending on needs.
* lookback for prediction: 30m is stable; reduce to 15m for higher sensitivity.

---

## 2.5 Predictive Spillover (Optional)

**When:** Predicted spillover will exceed **10%** of dedicated capacity in 10 minutes.
**PromQL**

```promql
predict_linear(
  aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput{request_type="spillover"}[30m],
  600
)
>
0.10 * aiplatform_googleapis_com:publisher_online_serving_dedicated_token_limit
and
(
  sum by (resource_container, model_user_id) (
    increase(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput{request_type="spillover"}[30m])
  ) > 5000
)
```

---

# 3. Notification Templates (subject + body)

> Use `${metric_or_resource.label.<KEY>}` if you need a fallback that covers either metric or resource labels. In many environments `${metric.label.<KEY>}` works when the label survived aggregation. Validate in test alerts.

### GSU Burndown — Subject

```
[VertexAI][${policy.display_name}] GSU >90% | Project: ${metric_or_resource.label.resource_container} | Model: ${metric_or_resource.label.model_user_id}
```

### Predictive Saturation — Subject

```
[VertexAI][${policy.display_name}] Predictive GSU Saturation (10m) | Project: ${metric_or_resource.label.resource_container} | Model: ${metric_or_resource.label.model_user_id}
```

### Non-200 Error — Subject

```
[VertexAI][${policy.display_name}] Error Rate >10% | Project: ${metric_or_resource.label.resource_container} | Model: ${metric_or_resource.label.model_user_id}
```

### Body (common pattern)

```markdown
**Project:** ${metric_or_resource.label.resource_container}  
**Model:** ${metric_or_resource.label.model_user_id}  
**Metric:** ${metric.label.__name__}  
**Observed value:** ${metric.label.value}

**What:** Brief explanation of the alerted condition.

**Action:** 1) Open Metrics Explorer for project/model. 2) Check throughput, errors, spillover charts. 3) Follow runbook.
```

---

# 4. Runbook — First Responder (playbook)

**A. Non-200 Error**

1. Open Metrics Explorer: filter `resource_container`, `model_user_id`.
2. Check `publisher_online_serving_model_invocation_count` by `response_code`: verify if one caller or many.
3. If localized to a caller: block/throttle client. If global: check model health, rollout, recent deploys.
4. Escalate to model owner; roll back if new deployment suspected.

**B. GSU Burndown / Predictive**

1. Open throughput vs dedicated limit graphs (1m/5m/15m/30m).
2. If predicted to hit capacity:

   * If traffic legitimate: increase dedicated GSU (provisioning flow).
   * If spike unexpected: identify clients; apply rate limits; move batch jobs to off-peak.
3. Notify billing if spillover could raise costs.

**C. Spillover**

1. Confirm dedicated is saturated.
2. Find which callers cause spillover. Inspect ingress logs.
3. If unavoidable, increase GSU or re-route to alternative model.

**Escalation:** Slack/Pager → Model Owner → Platform SRE → Cost Owner.

---

# 5. Dashboard (recommended panels)

Create a Vertex AI Capacity dashboard with tiles per `resource_container/model_user_id` (top N models) showing:

* Current throughput (1m/5m/15m), predicted throughput overlay.
* Dedicated token limit (gauge).
* Spillover rate and ratio (spillover / dedicated).
* Error rate (non-200 %) and raw counts.
* Heatmap: % models >75%, >90% usage and predicted >100% in 10m.

---

# 6. Threshold Recommendations (starter presets)

| Environment          | Non-200 guardrail (calls/15m) | GSU guardrail (tokens/30m) | Spillover guardrail (tokens/30m) |
| -------------------- | ----------------------------: | -------------------------: | -------------------------------: |
| Test / Dev           |                            50 |                      1,000 |                              500 |
| Staging / Small Prod |                         1,000 |                     10,000 |                            5,000 |
| Large Prod           |                         5,000 |                     50,000 |                           25,000 |

Tune thresholds over 2–4 weeks from observed traffic.

---

# 7. Grouping & Noise Reduction

* **Group incidents** by `resource_container` and `model_user_id` in the alert policy to avoid alert storms (one incident per model/project).
* Keep the guardrail `increase(...)` checks to avoid firing on tiny, ephemeral spikes.
* Consider **tiered alerts**: WARNING at 75% or predicted in 30m, CRITICAL at 90% or predicted in 10m.

---

# 8. Troubleshooting & Common Errors (keep this section near the bottom)

### Error: `parse error: unexpected '='`

Cause: using `name =` assignment style in the Cloud Monitoring PromQL editor.
Fix: remove assignments — paste only expressions (e.g., the `predict_linear(...) > ...` expression, not `throughput_15m = ...`).

---

### Error: `parse error: ranges only allowed for vector selectors`

Cause: placing a `[30m]` range on an aggregated expression (e.g., `sum(...) [30m]`) — PromQL allows range selectors **only** on raw metric selectors (or other range-vector expressions), not on instant vectors produced by aggregation.
Fix: apply `predict_linear` to the raw metric range-vector:

```promql
predict_linear(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput[30m], 600)
```

Do **not** write `predict_linear(sum(...)[30m], 600)`.

---

### Variable expansion in notifications shows `(null)` for Project/Model

Cause: the final evaluated series for the alert condition did **not** contain the label (the label got dropped during aggregation or by vector matching / logical operators). Common culprits:

* Using `and`/`or` between vectors that have different label sets — the intersection can drop labels.
* Aggregating with `sum` without the required label in `sum by(...)`.

Fixes:

1. Ensure the **final** expression that evaluates to true has `sum by(resource_container, model_user_id)` so those labels are present.
2. If you need to combine expressions, make sure all operands preserve the same label set (e.g., use `sum by(resource_container, model_user_id, response_code)` consistently, or align with `ignoring(...) / on(...)` vector matching where supported).
3. Test the exact PromQL in Metrics Explorer and inspect series legend to confirm labels appear.

---

# 9. Testing Checklist (before you flip to prod)

* [ ] Run `topk(...)` queries to confirm label names.
* [ ] Create each alert in a sandbox or with lowered thresholds to force a firing incident.
* [ ] Inspect the alert notification payload — confirm **Project** and **Model** expand (not `(null)`).
* [ ] Verify runbook steps with an engineer (simulate the incident and follow playbook).
* [ ] Tune guardrails for 2 weeks and adjust thresholds.

---

# 10. References (official)

* Prometheus `predict_linear` and functions: [https://prometheus.io/docs/prometheus/latest/querying/functions/](https://prometheus.io/docs/prometheus/latest/querying/functions/)
* Cloud Monitoring PromQL migration & examples: [https://cloud.google.com/monitoring/promql/promql-migrate](https://cloud.google.com/monitoring/promql/promql-migrate)
* Alert notification variables: [https://cloud.google.com/monitoring/alerts/doc-variables](https://cloud.google.com/monitoring/alerts/doc-variables)

---

## Appendix — Example: full Predictive alert as one expression (copy-paste)

```promql
predict_linear(
  aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput[30m],
  600
)
>
aiplatform_googleapis_com:publisher_online_serving_dedicated_token_limit
and
(
  sum by (resource_container, model_user_id) (
    increase(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput[30m])
  ) > 10000
)
```

---
