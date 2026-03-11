Quick summary: the fix is to **make sure the final evaluated series (the one that actually fires) contains the `resource_container` and `model_user_id` labels**. If those labels are dropped anywhere in your expression (or the alert engine chooses a different series as the canonical metric for the notification), the template sees `(null)`.

---

## 1) Why GSU works but Spillover shows `(null)`

Common causes (in order of likelihood):

* **Label mismatch between operands.** Even though you `sum by(resource_container, model_user_id)` for both operands, the presence of a `request_type` filter (different values `spillover` vs `dedicated`) can still produce subtle label set differences in how the engine matches series during binary ops — which can lead to a final series that does not include the labels you expect for notification expansion.
* **`and` / logical operator effects.** If you combine vectors with `and`/`or` whose label sets differ, the intersection may drop labels.
* **Notification variable scope.** Some alert engines pick the canonical metric for the notification and expose only those labels. Using `${metric.label.*}` can sometimes fail if the metric used by the notification does not include the labels. Using `${metric_or_resource.label.*}` is safer.
* **Some series are present only for spillover or only for dedicated** in a given window (so the ratio exists numerically but the label metadata used for notifications differs).

---

## 2) Quick diagnostics to run in Metrics Explorer (copy-paste)

Run these one at a time and inspect the legend/labels returned:

A — Numerator labels (spillover):

```promql
topk(50,
  sum by (resource_container, model_user_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput{request_type="spillover"}[15m])
  )
)
```

B — Denominator labels (dedicated):

```promql
topk(50,
  sum by (resource_container, model_user_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput{request_type="dedicated"}[15m])
  )
)
```

C — Division result — inspect legend to see which labels survive:

```promql
topk(50,
  (
    sum by (resource_container, model_user_id) (
      rate(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput{request_type="spillover"}[15m])
    )
    /
    sum by (resource_container, model_user_id) (
      rate(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput{request_type="dedicated"}[15m])
    )
  )
)
```

D — Guardrail check (increase spillover):

```promql
topk(50,
  sum by (resource_container, model_user_id) (
    increase(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput{request_type="spillover"}[30m])
  )
)
```

**What to look for**

* Do A and B show the same label keys (`resource_container`, `model_user_id`)?
* Does C's legend show `resource_container` and `model_user_id`?
  If A/B contain the labels but C does not → label drop happens during the division or subsequent matching.

---

## 3) Two fixes (quick → robust). Try Quick Fix first.

### Quick Fix — change notification variables (fast, safe)

Use the `metric_or_resource` variable in the Subject / Documentation so the template falls back to resource labels if `metric.*` labels are missing:

**Subject (paste into policy):**

```
[VertexAI][${policy.display_name}] Spillover >20% | Project: ${metric_or_resource.label.resource_container} | Model: ${metric_or_resource.label.model_user_id}
```

**Doc (body):**

```markdown
**Project:** ${metric_or_resource.label.resource_container}  
**Model:** ${metric_or_resource.label.model_user_id}  
**Spillover ratio:** ${metric.label.value}

Details: ...
```

This often fixes `(null)` immediately because it pulls from the resource label if metric label is absent.

---

### Robust Fix — ensure the final evaluated series *has* the labels

Modify the PromQL to force the final instant vector to include `resource_container` and `model_user_id`. Wrap the ratio with an outer `sum by(...)` (this leaves labels intact) or use explicit matching. Replace your current condition with **this** copy-pasteable expression:

```promql
(
  sum by (resource_container, model_user_id) (
    (
      sum by (resource_container, model_user_id) (
        rate(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput{request_type="spillover"}[15m])
      )
      /
      sum by (resource_container, model_user_id) (
        rate(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput{request_type="dedicated"}[15m])
      )
    )
  )
) > 0.20
and
(
  sum by (resource_container, model_user_id) (
    increase(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput{request_type="spillover"}[30m])
  ) > 5000
)
```

**Why this helps:** the outer `sum by(...)` forces the expression to be aggregated with the exact labels you want; that makes the final series explicitly carry `resource_container` & `model_user_id` at the alert-evaluation stage so `${metric.label.resource_container}` will expand.

> Alternative: if you prefer explicit vector-matching, use `on(resource_container, model_user_id)`:
>
> ```
> sum by(resource_container, model_user_id)(rate(...spill...))
> /
> on(resource_container, model_user_id)
> sum by(resource_container, model_user_id)(rate(...ded...))
> ```
>
> But the wrapped-sum version above is easiest to paste and usually prevents label loss.

---

## 4) Also ensure the guardrail preserves labels

You already used:

```promql
sum by(resource_container, model_user_id)( increase(...{request_type="spillover"}[30m]) ) > 5000
```

Good — verify in Diagnostics step D that this returns the same labels as your numerator. If it doesn’t, the `and` may drop labels. The robust fix (wrapping) keeps them consistent.

---

## 5) Final recommended subject + doc (robust)

Use both approaches together for max reliability:

**Subject**

```
[VertexAI][${policy.display_name}] Spillover >20% | Project: ${metric_or_resource.label.resource_container} | Model: ${metric_or_resource.label.model_user_id}
```

**Documentation body**

```markdown
**Project:** ${metric_or_resource.label.resource_container}  
**Model:** ${metric_or_resource.label.model_user_id}  
**Spillover (ratio):** ${metric.label.value}

Details: Sustained spillover >20% of dedicated throughput over 15m.
Guardrail: spillover tokens > 5k in 30m.

Actions:
1. Verify dedicated capacity and callers in Metrics Explorer.
2. If traffic is expected: increase dedicated GSU.
3. If unexpected: throttle or block callers; notify cost owner.
```

---

## 6) If you still see `(null)` after the robust fix

* Run the Diagnostics A–D again and paste the *exact* output (copy the series legend lines) — I’ll inspect and tell you which label differs.
* Check the **policy’s grouping** config: set “Group incidents by” to `resource_container` and `model_user_id`. If group-by is missing, notifications may not include the labels.
* As a last resort, try using `${metric_or_resource.label.resource_container}` in the subject and doc as your primary variable (it’s the safest across environments).

---

## TL;DR — What to change right now (do these two steps)

1. In the Spillover policy, replace subject & body variables with `${metric_or_resource.label.resource_container}` and `${metric_or_resource.label.model_user_id}`.
2. Replace the PromQL with the **robust wrapped-sum** expression above (copy-paste). Save & test (or lower thresholds in staging) and force a firing event to validate the template expansion.

---
