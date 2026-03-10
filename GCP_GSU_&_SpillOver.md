# Index
* a production-ready **predictive saturation alert** (PromQL + explanation),
* an improved **spillover detection** query (and guardrails),
* a **predictive spillover** variant,
* alert subject/body templates that work in Cloud Monitoring,
* runbook + testing + tuning notes.

---

# Vertex AI — Predictive GSU Saturation (Confluence-ready)

## Summary

Predict when a model’s dedicated token capacity (GSU) will be exhausted if current consumption growth continues. Fire an alert when the model is predicted to hit 100% of its dedicated token capacity within the next 10 minutes **and** there is meaningful traffic.

**Why**: gives SREs time to increase capacity, throttle non-critical jobs, or adjust traffic routing before spillover / errors / billing surprises occur.

---

## PromQL — Predictive Saturation (copy/paste)

```promql
(
  predict_linear(
    sum by (resource_container, model_user_id) (
      rate(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput[15m])
    )[30m],
    600
  )
)
>
sum by (resource_container, model_user_id) (
  aiplatform_googleapis_com:publisher_online_serving_dedicated_token_limit
)
and
(
  sum by (resource_container, model_user_id) (
    increase(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput[30m])
  ) > 10000
)
```

**Notes**

* `predict_linear(x[30m], 600)` projects the 30-minute trend forward by 600 seconds (10 min).
* The `increase(...[30m]) > 10000` guardrail prevents firing on tiny/experimental workloads. Tune `10000` per environment.
* Use `sum by (resource_container, model_user_id)` so `resource_container` and `model_user_id` survive into notification templates.

---

## Alert thresholds / tuning

* Prediction window: 10 minutes (600s) is a good starting point — change to 5/15 minutes as desired.
* Lookback window for prediction: 30 minutes is safe for smoothing short bursts; shorten to 15m if you need more sensitivity.
* Guardrail threshold (`increase`):

  * Small test: `>1000` tokens/30m
  * Medium production: `>10000` tokens/30m
  * Large / heavy production: `>50000` tokens/30m

---

## Notification templates (subject + body)

**Subject (recommended):**

```
[VertexAI][${policy.display_name}] Predictive GSU Saturation | Project: ${metric_or_resource.label.resource_container} | Model: ${metric_or_resource.label.model_user_id}
```

**Documentation / Body:**

```markdown
**Project:** ${metric_or_resource.label.resource_container}  
**Model:** ${metric_or_resource.label.model_user_id}  
**Predicted Throughput (in 10m):** ${metric.label.value}  
**Dedicated Token Limit:** ${metric_or_resource.label.publisher_online_serving_dedicated_token_limit}

**What:** At current trend, this model is predicted to exceed its dedicated token capacity within 10 minutes.
**Why:** Rapid growth in token consumption risks spillover to on-demand capacity and billing / errors.
**Action:**
1. Check Metrics Explorer for model throughput and recent spike sources.
2. If traffic is legitimate, increase dedicated GSU for the model/project.
3. If traffic is unexpected, identify callers and apply throttling or routing rules.
4. Open an Incident and notify the ownership team.
```

> Use `${metric_or_resource.label.KEY}` for label values (safe across both metric & resource labels). Use `${metric.label.value}` to print the numeric trigger value.

---

## Runbook (short)

1. Open Metrics Explorer and filter by `resource_container` + `model_user_id`.
2. Look at `publisher_online_serving_consumed_token_throughput` (15m, 5m) and `publisher_online_serving_dedicated_token_limit`.
3. If traffic is expected:

   * Increase dedicated token capacity (GSU) via provisioning process.
   * Notify finance if spillover billing could increase.
4. If unexpected:

   * Identify caller(s) (ingress logs, deployment changes).
   * Apply rate limits or circuit breaker on the client side.
5. If unable to mitigate quickly, change deployment routing to fallback model or scale horizontally.

---

# Spillover Detection & Predictive Spillover

## Purpose

Detect when requests are spilling from dedicated capacity to on-demand processing (spillover). This often precedes increased billing, latency, and errors.

Your original query looked like:

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

This is a valid approach if `request_type` exists and metrics are emitted with those labels. Below are improvements.

---

## Improved Spillover Ratio (recommended)

```promql
# spillover tokens/sec (15m)
spillover_rate_15m =
  sum by (resource_container, model_user_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput{request_type="spillover"}[15m])
  )

# dedicated tokens/sec (15m)
dedicated_rate_15m =
  sum by (resource_container, model_user_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput{request_type="dedicated"}[15m])
  )

# ratio
(
  spillover_rate_15m
  /
  (dedicated_rate_15m + 1)    # +1 to avoid divide-by-zero noise; optional
) > 0.20
and
(
  sum by (resource_container, model_user_id) (
    increase(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput{request_type="spillover"}[30m])
  ) > 1000/
)
```

**Notes**

* Group by `resource_container, model_user_id` so labels are preserved.
* Add an absolute guardrail on `increase(...spillover...)` to ignore tiny spillover volumes.
* You can do `>0.20` (i.e., >20% of dedicated) as a starting point — tune by traffic pattern.

---

## Predictive Spillover (detect rising spillover trend)

Detect if spillover is increasing fast enough that it will exceed some fraction of dedicated capacity in the near future.

```promql
# project spillover rate forward by 10m
predicted_spillover_in_10m =
  predict_linear(
    sum by (resource_container, model_user_id) (
      rate(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput{request_type="spillover"}[5m])
    )[30m],
    600
  )

# compare predicted spillover to dedicated capacity
predicted_spillover_in_10m > 
  ( 0.10 * sum by (resource_container, model_user_id) ( aiplatform_googleapis_com:publisher_online_serving_dedicated_token_limit ) )
and
(
  sum by (resource_container, model_user_id) (
    increase(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput{request_type="spillover"}[30m])
  ) > 5000
)
```

This warns when predicted spillover will be >10% of dedicated capacity in 10 minutes. Adjust the `0.10` and guardrail as needed.

---

## Practical suggestions / improvements

* If `request_type` label is **not consistently present**, derive spillover by comparing `total_throughput - dedicated_throughput` (if both metrics exist).
* Prefer `sum by (resource_container, model_user_id)` for both numerator & denominator so series match.
* Use guardrails (`increase(...) > X`) to avoid noisy alerts.
* Consider also alerting on absolute spillover throughput (e.g., spillover tokens/sec > N) for immediate billing risks.

---

# Dashboard & Visuals (what to show)

For each model/project tile:

* Current throughput (1m / 5m / 15m)
* Dedicated token limit (gauge)
* Predicted throughput (10m projection) vs dedicated limit (overlay)
* Spillover ratio (spillover_rate / dedicated_rate)
* Recent alerts and trend lines (last 2 hours)

Make a “heatmap” showing % of models in >90% bucket and predicted saturation within 10m.

---

# Testing & Validation steps

1. In Metrics Explorer, paste the throughput and dedicated queries, `sum by (resource_container, model_user_id) (...)` and confirm labels appear in the legend.
2. Temporarily lower predictive threshold or prediction window to force a firing incident for a single model (sanbox environment recommended).
3. Confirm notification variables expand (project/model appear) in the resulting test alert email.
4. Verify runbook steps and escalation flow work (Email, any other channel).

---

# Short FAQ (quick answers)

* **Should I use `predict_linear`?** Yes — it gives actionable lead time. Use conservative windows to limit false positives.
* **How do I avoid `(null)` in notifications?** Ensure the final evaluated series is `sum by(resource_container, model_user_id)` (labels preserved), and use `${metric_or_resource.label.<KEY>}` or `${metric.label.<KEY>}` in templates (pick the one that resolved correctly in your environment —  `${metric.label.resource_container}` works for burndown).

