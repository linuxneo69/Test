
### 🕵️ The Root Cause Analysis: Why are you getting `(null)`?

The issue comes down to a specific quirk in how **PromQL** interacts with **Google Cloud's Alerting Engine**. There are two separate mistakes happening in your screenshots that are combining to cause the `(null)` output.

#### Error 1: PromQL Label Dropping (From your first screenshot)

Look closely at the query in your first screenshot (`image_0ba479.png`):
`sum by (model_id) (rate(...))`

In PromQL, when you use an aggregation operator like `sum by (...)`, **it acts as a destructive filter for labels.** PromQL will mathematically sum the data, but it will immediately **delete** every single label that is not explicitly listed inside those parentheses.

Because you only put `model_id` in the parentheses, PromQL threw away the `resource_container` and `model_user_id` labels before the data ever reached the alerting engine. When the email template asked for the project name, the alerting engine looked at the incoming data, saw it didn't exist, and printed `(null)`.

#### Error 2: GCP Variable Syntax Flattening (From your other screenshots)

In your other screenshots, you experimented with different syntaxes:

* ❌ `${resource_container.model_user_id}` (Invalid syntax)
* ❌ `${resource.labels.project_id}` (This works for normal GCP metrics, but **not** for PromQL)
* ❌ `${metric.labels.model_user_id}` (Close, but plural "labels" can cause rendering bugs in certain GCP alert UI versions)

**The GCP PromQL Quirk:** When you use PromQL in Google Cloud, GCP takes *all* the surviving labels from your query and flattens them into a single category called `metric.label`. It strips away the normal GCP "resource" classifications.

---

### 🛠️ The Bulletproof Fix & Improvement

Here is the exact, improved configuration that solves the `(null)` issue, implements the Auto-Discovery strategy, and adds safeguards against false alarms.

#### 1. The "Elite" PromQL Query

We are changing the `sum by` to preserve the exact variables we want to print in the email. We are also adding an **improvement**: a minimum volume check (`increase > 50`) so you don't get woken up at 3 AM because a project made exactly 1 request and it failed (which is technically a 100% error rate).

```promql
# Part 1: The Error Ratio (Must be > 2%)
(
  sum by (resource_container, model_user_id, response_code) (
    rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count{response_code=~"429|499|500|503"}[15m])
  )
  /
  sum by (resource_container, model_user_id, response_code) (
    rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count[15m])
  )
) > 0.02

# Part 2: The Volume Guardrail (Must have at least 50 requests in 15 mins to care)
and 
(
  sum by (resource_container, model_user_id, response_code) (
    increase(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count[15m])
  ) > 50
)

```

*(Note: I added `response_code` to the denominator's `sum by` as well. In PromQL mathematical operations, the labels on the left side of the `/` and `and` operators must perfectly match the right side for the math to execute properly).*

#### 2. The Exact Variable Syntax (Copy & Paste)

Now that the PromQL is actually exporting the labels, you must use this exact, singular syntax in the GCP UI.

**Notification Subject Line:**

> `[HTTP ${metric.label.response_code}] Error > 2% in ${metric.label.resource_container} | Model: ${metric.label.model_user_id}`

**Documentation Section (Markdown):**

```markdown
# 🚨 Vertex AI Auto-Discovery Alert
An elevated error rate has been detected.

* **Project:** `${metric.label.resource_container}`
* **Model:** `${metric.label.model_user_id}`
* **Error Code:** `${metric.label.response_code}`

**Details:** More than 2% of the last 50+ requests to this model have failed with a `${metric.label.response_code}` status code.

```

---

### 🛡️ Three More Settings to Double-Check in the UI

To ensure this strategic setup works flawlessly, check these specific boxes in the Alert Policy UI:

1. **Missing Data Policy (Crucial):** If a model stops receiving traffic entirely, PromQL returns "No Data." By default, GCP might treat this as an active alert. Find the **"Evaluation missing data"** dropdown and set it to: `Missing data points considered as values that do not violate the policy condition`.
2. **Incident Autoclose:** Set this to **30 minutes**. If the PromQL query stops returning a violation, the alert will heal itself.
3. **Incident Grouping:** Because you are using PromQL, Google Cloud *automatically* groups by the labels in your `sum by` clause. Do not try to override this in the UI.

This setup completely eliminates the need to hardcode `project-xyz` or `claude-sonnet-4-5`. It will dynamically catch every current and future combination.

