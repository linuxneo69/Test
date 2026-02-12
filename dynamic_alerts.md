
Below is the **80% PT Burndown** query reimagined in PromQL, along with a detailed breakdown of how it works in the context of Vertex AI metrics.

---

## 📊 PromQL: 80% Provisioned Throughput (PT) Burndown Alert

In PromQL, we don't use "pipes" (`|`). Instead, we use nested functions and operators. To calculate a "burndown" percentage, we divide the **Actual Consumed Throughput** by the **Dedicated Capacity Limit**.

### The Query

```promql
# Calculate the ratio of weighted token consumption to total GSU capacity
(
  sum by (model_id) (
    rate(aiplatform_googleapis_com:prediction_consumed_token_throughput[2m])
  )
  /
  sum by (model_id) (
    aiplatform_googleapis_com:prediction_dedicated_token_limit
  )
) > 0.8

```

---

## 🔍 Detailed Component Breakdown

### 1. The Metric Transformation

You’ll notice the metric names look slightly different: `aiplatform.googleapis.com/metric` becomes `aiplatform_googleapis_com:metric`.

* **Rule:** When Google Cloud metrics are pulled into PromQL, the first `/` becomes a `:` and all subsequent dots or slashes become `_`. This makes them compliant with Prometheus naming standards.

### 2. The Numerator: Weighted Consumed Throughput

`rate(aiplatform_googleapis_com:prediction_consumed_token_throughput[2m])`

* **Weighted Accounting:** This metric is crucial for PT. Vertex AI doesn't count every token as "1." Output tokens (generation) typically burn quota faster than input tokens (prompts). This metric reflects that "weighted" cost.
* **The 2-Minute Window:** I have used `[2m]` specifically because Vertex AI enforces PT quotas over a **rolling 120-second window**. Using a 2-minute window in your rate calculation aligns your alert perfectly with how Google actually throttles your traffic.

### 3. The Denominator: Dedicated Token Limit

`aiplatform_googleapis_com:prediction_dedicated_token_limit`

* **Dynamic Scaling:** This is a "Gauge" metric that represents the maximum throughput available based on the number of **Scale Units (GSUs)** you have purchased.
* **Why it's "Dynamic":** If you increase your PT order from 10 GSUs to 20 GSUs in the Google Cloud Console, this value updates automatically. Your alert threshold of 0.8 (80%) stays the same, but the absolute number it triggers at will double instantly without you touching the code.

### 4. The Division & Aggregation

`sum by (model_id) (...)`

* Since a project might have multiple models (e.g., Gemini 1.5 Pro and Gemini 1.5 Flash), the `sum by (model_id)` ensures we are calculating the burndown percentage for each model independently. Without this, the alert might accidentally sum all your model usage together, causing a false alarm.

---

## 🧪 Operational Scenario: The 80% Warning

**The Goal:** You want to know when your reserved "private lane" is getting crowded so you can either divert traffic to "Pay-As-You-Go" or purchase more GSUs.

* **Trigger Condition:** The ratio hits `0.81`.
* **System Action:** The alert fires. Your SRE team sees that **Model A** is at 81% capacity.
* **Business Value:** You have a 19% "safety buffer" to react before users start receiving **429 (Too Many Requests)** errors or before the system forces traffic into the more expensive "Spillover" (on-demand) pricing tier.

---

### ⚠️ A Note on PromQL "Labels"

When setting this up in the Alert Policy editor, ensure your **Alert Strategy** is set to "Open" for at least 2 or 3 minutes. Because Vertex AI metrics can have a slight ingestion delay (up to 60-90 seconds), a 1-minute alert might occasionally "flap" due to delayed data points.
