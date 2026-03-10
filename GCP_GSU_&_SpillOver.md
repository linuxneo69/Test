# Vertex AI — Unified Alerting (Auto-Discovery)

Purpose: single unified monitoring policy set that auto-discovers Vertex AI models across projects and alerts on Errors, GSU (token) Burndown, Spillover, and Predictive Saturation.

---

## At a glance

* **Monitored scope:** Metrics Scope (monitoring hub) covering multiple projects.
* **Goals:** early detection (predictive), capacity protection (GSU), cost/risk control (spillover), and reliability (error-rate).
* **Key metrics used:**

  * `aiplatform_googleapis_com:publisher_online_serving_model_invocation_count`
  * `aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput`
  * `aiplatform_googleapis_com:publisher_online_serving_dedicated_token_limit`
* **Important labels:** `resource_container` (project), `model_user_id` (model), `response_code`, `request_type` (if present). Confirm exact names in Metrics Explorer before creating policies.

---

## Quick pre-check (do this first)

1. Open **Monitoring → Metrics Explorer → PromQL**. Run:

   ```promql
   topk(20, aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput)
   topk(20, aiplatform_googleapis_com:publisher_online_serving_model_invocation_count)
   topk(20, aiplatform_googleapis_com:publisher_online_serving_dedicated_token_limit)
   ```
2. Inspect legend for exact label keys. Confirm `resource_container` and `model_user_id`. If label names differ, use the exact names in all queries and templates.
3. Decide environment guardrails from the Recommended Thresholds table below.

---

## Global conventions (use consistently)

* **Policy naming:** `VertexAI / <Area> / <AlertType>` (e.g., `VertexAI / Capacity / Predictive GSU Saturation`)
* **Condition naming:** `<AlertType> — <window> — <threshold>` (e.g., `Non-200 Rate — 15m — 10%`)
* **Tag examples:** `vertexai`, `gsu`, `capacity`, `spillover`, `predictive`, `auto-discovery`, `owned-by:ml-platform`, `env:prod`
* **Grouping:** Group incidents by `resource_container`, `model_user_id`. Configure the policy “Group incidents by” to those labels.
* **Notification variables:** Prefer `${metric_or_resource.label.<KEY>}` in templates (falls back between metric & resource labels). Use `${metric.label.value}` or `${metric.value}` for numeric trigger value (test in your org which expands).
* **Guardrail pattern:** Every alert includes a traffic floor such as `increase(...[window]) > X` to reduce noise.

---

# Alerts — single-expression PromQL + metadata (copy/paste)

> Paste each PromQL **as a single expression** into the Cloud Monitoring alert condition. **Do not** use assignment syntax (`name = ...`) or put `[range]` on aggregation results. `predict_linear` must be applied to a raw metric range-vector (e.g., `metric_name[30m]`).

---

### 1) **Non-200 Error Rate (Unified Error)** — *critical reliability alert*

**Policy name:** `VertexAI / Errors / Non200 Rate`
**Condition name:** `Non-200 Rate — 15m — >10%`
**Severity:** `CRITICAL`
**Tags:** `vertexai, errors, unified, auto-discovery`
**Grouping:** `resource_container`, `model_user_id`

**PromQL (paste exactly):**

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

**Subject (notification):**

```
[VertexAI][${policy.display_name}] Error Rate >10% | Project: ${metric_or_resource.label.resource_container} | Model: ${metric_or_resource.label.model_user_id}
```

**Documentation (paste into policy docs):**

```markdown
**Alert:** Non-200 Response Rate >10% over 15 minutes

**Project:** ${metric_or_resource.label.resource_container}  
**Model:** ${metric_or_resource.label.model_user_id}  
**Observed value:** ${metric.label.value}

**Why:** More than 10% of requests returned non-200 responses in the last 15 minutes.

**Guardrail:** Fires only when ≥1000 calls in the last 15 minutes.

**Immediate actions**
1. Open Metrics Explorer → filter by project & model.  
2. Inspect `publisher_online_serving_model_invocation_count` split by `response_code` and `caller`.  
3. If single caller: throttle/block. If global: check recent deployments/configs and model health.  
4. Escalate to model owner & Platform SRE.

**Links:** Metrics Explorer (project prefilled), runbook link: <RUNBOOK_URL>
```

**Testing note:** Lower ratio to 5% and guardrail to 100 in a sandbox to force firing and validate notification variables expand.

---

### 2) **GSU Burndown (Static)** — *capacity urgent / pre-spill*

**Policy name:** `VertexAI / Capacity / GSU Burndown`
**Condition name:** `GSU Burndown — 30m — >90%`
**Severity:** `WARNING` (75%) / `CRITICAL` (90%) — create two conditions for tiering if desired
**Tags:** `vertexai, gsu, capacity, burndown`
**Grouping:** `resource_container`, `model_user_id`

**PromQL:**

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

**Subject:**

```
[VertexAI][${policy.display_name}] GSU Usage >90% | Project: ${metric_or_resource.label.resource_container} | Model: ${metric_or_resource.label.model_user_id}
```

**Documentation:**

```markdown
**Alert:** GSU Burndown — token throughput >90% of dedicated limit over 30 minutes.

**Project:** ${metric_or_resource.label.resource_container}  
**Model:** ${metric_or_resource.label.model_user_id}  
**Observed utilization:** ${metric.label.value}

**Why:** Sustained high consumption relative to dedicated capacity can cause spillover to on-demand and increase latency/cost.

**Immediate actions**
1. Inspect throughput/time-series and recent callers.  
2. If legitimate: request increased dedicated GSU capacity.  
3. If unexpected: identify & throttle problematic callers or move batch jobs off-peak.

**Runbook:** escalate to model owner → Platform SRE for provisioning.
```

**Testing note:** Use a smaller guardrail in staging (e.g., `>1000`) to trigger and validate.

---

### 3) **Spillover (Static)** — *billing & latency risk*

**Policy name:** `VertexAI / Capacity / Spillover`
**Condition name:** `Spillover Ratio — 15m — >20%`
**Severity:** `WARNING`/`CRITICAL` (tune per org)
**Tags:** `vertexai, spillover, capacity, billing`
**Grouping:** `resource_container`, `model_user_id`

**PromQL (preferred if `request_type` exists):**

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

**If `request_type` is not present:** derive spillover = `total_throughput - dedicated_throughput` only if both metrics exist and share compatible labels — validate carefully.

**Subject:**

```
[VertexAI][${policy.display_name}] Spillover >20% | Project: ${metric_or_resource.label.resource_container} | Model: ${metric_or_resource.label.model_user_id}
```

**Documentation:**

```markdown
**Alert:** Spillover — spillover throughput >20% of dedicated throughput (15m).

**Project:** ${metric_or_resource.label.resource_container}  
**Model:** ${metric_or_resource.label.model_user_id}  
**Observed spillover ratio:** ${metric.label.value}

**Why:** Requests are being served from on-demand capacity (spillover) — risk of higher latency and billing.

**Immediate actions**
1. Confirm dedicated saturation and spot callers.  
2. If necessary, increase dedicated GSU, or throttle/redirect non-critical traffic.  
3. Notify billing/cost owner if sustained.

**Runbook:** see GSU Burndown for capacity steps + billing contacts.
```

**Testing note:** Simulate controlled spillover in staging or lower guardrail to validate.

---

### 4) **Predictive GSU Saturation** — *early warning (recommended)*

**Policy name:** `VertexAI / Capacity / Predictive GSU Saturation`
**Condition name:** `Predictive Saturation — 10m projection`
**Severity:** `WARNING` (predicted in 30m) / `CRITICAL` (predicted in 10m) — use tiered policies
**Tags:** `vertexai, predictive, gsu, capacity`
**Grouping:** `resource_container`, `model_user_id`

**IMPORTANT:** `predict_linear` must be applied to a raw metric range-vector selector (e.g., `metric_name[30m]`). Do **not** attach `[30m]` to aggregated expressions.

**PromQL (paste exact):**

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

**Subject:**

```
[VertexAI][${policy.display_name}] Predictive GSU Saturation (10m) | Project: ${metric_or_resource.label.resource_container} | Model: ${metric_or_resource.label.model_user_id}
```

**Documentation:**

```markdown
**Alert:** Predicted throughput will exceed dedicated token limit within 10 minutes (linear projection using last 30m).

**Project:** ${metric_or_resource.label.resource_container}  
**Model:** ${metric_or_resource.label.model_user_id}  
**Predicted throughput (10m):** ${metric.label.value}  
**Dedicated token limit:** ${metric_or_resource.label.publisher_online_serving_dedicated_token_limit}

**Why:** Predictive detection allows preemptive provisioning & throttling before spillover/errors occur.

**Immediate actions**
1. Inspect prediction & recent growth sources.  
2. If legitimate: request/increase GSU capacity.  
3. If unexpected: apply rate limits / block callers.

**Tuning:** Change prediction window (600s) or lookback (30m) per sensitivity needs.
```

**Testing note:** In staging, use smaller lookback/prediction windows to force firing and confirm notifications.

---

### 5) **Predictive Spillover (optional proactive)**

**Policy name:** `VertexAI / Capacity / Predictive Spillover`
**Condition name:** `Predictive Spillover — 10m — >10%`
**Severity:** `WARNING`
**Tags:** `vertexai, predictive, spillover`
**Grouping:** `resource_container`, `model_user_id`

**PromQL (if `request_type="spillover"` exists):**

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

**Doc & Runbook:** same workflow as Spillover, but act proactively.

---

## Recommended dashboard panels (single glance)

* **Model Inventory tile:** top N models by throughput (show `resource_container` + `model_user_id`)
* **Throughput vs Dedicated Limit:** 1m/5m/15m + dedicated limit gauge + predictive overlay (predicted value line)
* **Spillover ratio & absolute spillover tokens**
* **Error rate sweep:** non-200 % by model
* **Heatmap:** % models >75% / >90% usage and predicted >100% in 10m

---

## Recommended thresholds (starter presets)

|                Env | Non-200 guardrail (calls/15m) | GSU guardrail (tokens/30m) | Spillover guardrail (tokens/30m) | Predict lookback / proj |
| -----------------: | ----------------------------: | -------------------------: | -------------------------------: | ----------------------: |
|           Dev/Test |                            50 |                      1,000 |                              500 |                15m / 5m |
| Staging/Small Prod |                         1,000 |                     10,000 |                            5,000 |               30m / 10m |
|         Large Prod |                         5,000 |                     50,000 |                           25,000 |               30m / 10m |

Tune thresholds over 2–4 weeks.

---

## Runbook — playbook (short actionable steps)

**Non-200 Error**

1. Open Metrics Explorer → filter by project & model.
2. Inspect `model_invocation_count` by `response_code` and caller.
3. If single caller: rate-limit or block. If system-wide: check recent deploys/config.
4. Reproduce & confirm resolution. Escalate to model owner.

**GSU Burndown / Predictive**

1. Confirm throughput & prediction.
2. If traffic legitimate: request GSU increase (platform provisioning).
3. If unexpected: identify callers, throttle or re-route. Notify Cost Owner if spillover imminent.

**Spillover**

1. Confirm dedicated limit saturated and spillover magnitude.
2. Evaluate immediate mitigation: throttle, deprioritize batch jobs, allocate additional capacity. Notify Billing team if sustained.

---

## Creation & config checklist (use when creating policy)

* [ ] Verify metric names & labels in Metrics Explorer.
* [ ] Paste PromQL expression as a single expression.
* [ ] Set evaluation period / alignment to match query window (15m / 30m).
* [ ] Configure grouping by `resource_container`, `model_user_id`.
* [ ] Add tags and select appropriate notification channels (PagerDuty, Slack).
* [ ] Configure incident auto-close & deduplication settings.
* [ ] Test in sandbox (lower thresholds) and verify variables expand in notifications.

---

## Troubleshooting (bottom of page — keep for reference)

**1. `parse error: unexpected '='`**
Cause: using assignment (`name = ...`) in Cloud Monitoring.
Fix: remove assignments; paste only the expression.

**2. `ranges only allowed for vector selectors`**
Cause: putting `[30m]` onto aggregated expression (e.g., `sum(...)[30m]`).
Fix: apply `predict_linear` to a raw metric range-vector: `predict_linear(metric_name[30m], 600)`.

**3. Notifications show `(null)` for project/model**
Cause: the final evaluated series lost labels (dropped by aggregation or logical vector matching such as `and/or`).
Fix:

* Ensure final expression preserves labels using `sum by(resource_container, model_user_id)`;
* When combining vectors, ensure operands preserve same labels or use explicit `on(...)`/`ignoring(...)` matching (test behavior in Cloud Monitoring — some vector matching semantics differ).
* Test the exact PromQL in Metrics Explorer and check legend for labels.

**4. “No data in timeframe” for graphs**
Cause: metric did not emit in selected window or model had no traffic.
Fix: expand time window or verify metric presence with `topk()`; ensure Monitoring Hub has metrics from target projects.

---

## References (official)

* Prometheus functions — `predict_linear`: [https://prometheus.io/docs/prometheus/latest/querying/functions/](https://prometheus.io/docs/prometheus/latest/querying/functions/)
* Cloud Monitoring — PromQL migration & examples: [https://cloud.google.com/monitoring/promql/promql-migrate](https://cloud.google.com/monitoring/promql/promql-migrate)
* Cloud Monitoring — Alert notification variables: [https://cloud.google.com/monitoring/alerts/doc-variables](https://cloud.google.com/monitoring/alerts/doc-variables)

---

## Appendices

### Example short email (what responders see)

**Subject:** `[VertexAI][Predictive GSU Saturation (10m)] Predictive GSU Saturation (10m) | Project: ai-prod | Model: gemini-1.5-pro`
**Body:**

```
Project: ai-prod
Model: gemini-1.5-pro
Predicted throughput (10m): 22,050 tokens/sec
Dedicated token limit: 20,000 tokens/sec

What: Prediction indicates model will exceed dedicated token limit in 10 minutes.
Action: Consider increasing dedicated capacity or throttling non-critical callers. See runbook: <RUNBOOK_URL>
```

---
