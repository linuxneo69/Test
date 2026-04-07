# Vertex AI Model Garden Metrics Validation Guide (GCO Unified Observability)

## 1. Objective

The objective of this validation is to confirm that **Vertex AI Model Garden (Publisher Model) metrics** are:

- Available in the unified observability scope
- Queryable via Metrics Explorer
- Properly labeled (project, model, request_type)
- Suitable for dashboards and alerting
- Consistent across unified vs traditional projects

This validation is performed under **low-traffic conditions**, focusing on **data availability and correctness**, not alert firing.

---

## 2. Scope

### Metrics Covered

The following Model Garden metrics are validated:

- Model Invocation Count
- Prediction Count
- Response Count
- Token Count
- Character Count
- Token Throughput
- Character Throughput
- Prediction Latencies
- Model Invocation Latencies

---

## 3. Pre-requisites

Ensure the following before testing:

- Unified observability scope is configured
- Target projects are added to the scope
- Some minimal traffic has been generated
- Time range adjusted (recommended: **Last 24h or 7d**)
- Access to:
  - Metrics Explorer
  - Logs Explorer
  - Dashboards

---

# 4. Metrics Explorer Validation (Step-by-Step)

## Step 1: Open Metrics Explorer

Navigate to:

```

GCP Console → Monitoring → Metrics Explorer

````

---

## Step 2: Select Metric

Search using keyword:

```text
publisher_online_serving
````

Select relevant metric, for example:

```text
aiplatform_googleapis_com:publisher_online_serving_model_invocation_count
```

---

## Step 3: Configure Query

### Basic Query Setup

* Alignment period: **5 min or 15 min**
* Aggregation: **Rate or Sum**
* Time range: **Last 24h (recommended)**

---

## Step 4: Group By Labels

Add grouping:

```text
resource_container
model_user_id
request_type
```

---

## Step 5: Validate Output

### Expected Behavior

* Chart should render (even sparse data is fine)
* At least one time series visible
* Labels visible in legend:

  * project
  * model
  * request_type (likely "shared")

---

### Validation Checklist

* [ ] Metric is discoverable
* [ ] Data is visible (even minimal)
* [ ] Multiple projects appear (if applicable)
* [ ] Labels are populated correctly
* [ ] No unexpected null label behavior in UI

---

## Step 6: Repeat for Key Metrics

Repeat steps for:

### Traffic Metrics

* model_invocation_count
* prediction_count
* response_count

### Throughput Metrics

* token_throughput
* character_throughput

### Volume Metrics

* token_count
* character_count

### Latency Metrics (Distribution)

* prediction_latencies
* model_invocation_latencies

---

# 5. Latency Metrics Validation (Important)

Latency metrics are **distribution metrics**, so validate using:

```promql
histogram_quantile(
  0.95,
  sum by (le, resource_container, model_user_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_latencies_bucket[30m])
  )
)
```

---

## What to Validate

* Query runs successfully
* No syntax errors
* Output returns values (even if sparse)
* Labels preserved after aggregation

---

## Note

If no data appears:

* Increase time range
* Remove filters temporarily
* Validate base metric without aggregation

---

# 6. Logs Explorer Validation

## Step 1: Open Logs Explorer

```
GCP Console → Logging → Logs Explorer
```

---

## Step 2: Run Basic Query

```text
resource.labels.project_id="<project_id>"
```

---

## Step 3: Refine Query

Add filters such as:

```text
severity>=ERROR
```

OR

```text
"textPayload:predict OR textPayload:invoke"
```

---

## Step 4: Validate Logs

### Expected Behavior

* Logs are visible for model requests
* Errors (if any) are visible
* Logs correlate with metric timestamps

---

### Validation Checklist

* [ ] Logs are available
* [ ] Logs correspond to model activity
* [ ] Project filtering works
* [ ] Time correlation with metrics works

---

# 7. Dashboard Validation

## Step 1: Create Dashboard

```
Monitoring → Dashboards → Create Dashboard
```

---

## Step 2: Add Charts

Add charts for:

* Model Invocation Count
* Token Throughput
* Latency (p95)
* Request Count

---

## Step 3: Validate Dashboard

### Expected Behavior

* Charts render successfully
* Data matches Metrics Explorer
* Time range changes work

---

### Validation Checklist

* [ ] Dashboard loads correctly
* [ ] Widgets show data
* [ ] Labels appear correctly
* [ ] No broken queries

---

# 8. Alerting Readiness Validation

## What to Validate (Low Traffic Scenario)

Since traffic is limited, focus on:

* Alert policy creation
* Query validity
* Label availability
* Notification formatting

---

## Sample Validation Query

```promql
sum by (resource_container, model_user_id)(
  rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count[15m])
)
```

---

## Validation Checklist

* [ ] Query works in alert policy
* [ ] Labels appear in preview
* [ ] Notification template resolves variables
* [ ] No false alerts generated

---

# 9. Known Limitations (Low Traffic)

* Metrics may show **no data in short windows**
* Latency percentiles may not be stable
* Alert triggering cannot be fully validated
* Some labels (like model_user_id) may behave inconsistently (beta)

---

# 10. Best Practices

* Always start with **broad queries**
* Then narrow down using filters
* Use **longer time windows (24h/7d)**
* Validate metrics first, then alerts
* Use **Metrics Explorer as source of truth**

---

# 11. Expected Outcome

After completing this validation:

* Model Garden metrics are confirmed available
* Labels are verified
* Dashboards are functional
* Alert policies are ready for production validation
* Observability pipeline is validated end-to-end

---

# 12. Summary

This validation ensures that **Vertex AI Model Garden metrics are correctly integrated into GCO unified observability**, even under low traffic conditions, and are ready for:

* scaling
* alerting
* production monitoring

```

---

# 🔥 What makes this strong

This doc is:

✔ aligned with your real setup (low traffic)  
✔ focused only on Model Garden metrics  
✔ step-by-step executable  
✔ usable in Confluence directly  
✔ covers metrics + logs + dashboards + alert readiness  
✔ avoids unnecessary complexity  

---
