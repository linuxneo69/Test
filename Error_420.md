# Sum 429 errors for specific models in a specific project over a 2-minute window
sum by (model_id, resource_container) (
  rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count{
    response_code="429", 
    resource_container="your-project-id"
  }[2m])
) * 120 > 10

# Sum all non-200 responses across models and projects
sum by (model_id, resource_container) (
  rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count{
    response_code!="200"
  }[2m])
) * 120 > 10

---

Advanced: Alerting on Error Rate % (Recommended)
Raw counts can be misleading (10 errors is a lot for a small project, but tiny for a large one). It is usually better to alert when the percentage of errors exceeds a threshold (e.g., > 5%).

# Alert if more than 5% of traffic is failing (non-200)
(
  sum by (model_id, resource_container) (
    rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count{response_code!="200"}[2m])
  )
  /
  sum by (model_id, resource_container) (
    rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count[2m])
  )
) > 0.05  

**Project Specific:
**
(
  sum by (model_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count{
      response_code!="200", 
      resource_container="your-project-id"
    }[2m])
  )
  /
  sum by (model_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count{
      resource_container="your-project-id"
    }[2m])
  )
) > 0.05

---
To alert on any response code starting with 5 in your unified project, use the Regex Matcher (=~) with the pattern 5...
# Alert on any 5xx error across models and projects
sum by (model_id, resource_container) (
  rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count{
    response_code=~"5.."
  }[2m])
) * 120 > 5

---

# Summing 429 errors using the prediction_count metric
sum by (resource_container, model_id) (
  rate(aiplatform_googleapis_com:prediction_online_prediction_count{
    response_code="429"
  }[2m])
) * 120 > 10

When to use which metric?If you are using...Use this PQL MetricCustom Models (AutoML, XGBoost, etc.)aiplatform_googleapis_com:prediction_online_prediction_countFoundation Models (Gemini, Claude, etc.)

---
## The PQL Query for GSU Burndown
# This query monitors your Provisioned Throughput (Dedicated) and calculates if you are consistently hitting 90% of your allocated capacity.

# Alert if GSU utilization is > 90% for a sustained period
avg_over_time(
  aiplatform_googleapis_com:prediction_online_dedicated_token_limit_utilization{
    resource_container="your-project-id"
  }[30m]
) > 0.9

# Pro-Tip: The "Fast Burn" Alert
In SRE best practices, a 30-minute 90% alert is great for capacity planning, but it might be too slow for a sudden traffic spike. Many teams set a secondary alert for a "Fast Burn":

Slow Burn (Capacity): 90% usage for 30 minutes (Your requested alert).
Fast Burn (Emergency): 98% usage for 2 minutes (Immediate page).

---
The Exact PQL Name
If you are using the PromQL (PQL) editor, type this exact string: aiplatform_googleapis_com:prediction_online_dedicated_token_limit_utilization

If that returns an error, use the "Total Consumed" metric instead: Sometimes Google updates the labels. If the utilization metric is missing, you can manually calculate it by dividing your consumed throughput by your limit:

# Manual calculation of GSU utilization
(
  sum(rate(aiplatform_googleapis_com:prediction_online_consumed_token_throughput[5m]))
  /
  sum(aiplatform_googleapis_com:prediction_online_dedicated_token_limit)
) > 0.9

Retest Window: 30min
Evaluation: 1min

NOT Working GSU Burndown:
(
  sum by (model_id) (
    rate(aiplatform.googleapis.com/prediction/online/consumed_token_throughput[5m])
  )
  /
  sum by (model_id) (
    aiplatform.googleapis.com/prediction/online/dedicated_token_limit
  )
) > 0.9


not working one -
(
  sum by (model_id) (
    rate(aiplatform_googleapis_com:prediction_online_consumed_token_throughput{
      resource_container="your-project-id"
    }[5m])
  )
  /
  sum by (model_id) (
    aiplatform_googleapis_com:prediction_online_dedicated_token_limit{
      resource_container="your-project-id"
    }
  )
) > 0.9

Trying this -
(
  sum by (model_id) (
    rate(aiplatform.googleapis.com/prediction/online/consumed_token_throughput{project_id="your-project-id"}[5m])
  )
  /
  sum by (model_id) (
    aiplatform.googleapis.com/prediction/online/dedicated_token_limit{project_id="your-project-id"}
  )
) > 0.9

---

(
  sum by (model_id) (
    rate(aiplatform_googleapis_com:prediction_online_consumed_token_throughput{resource_container="your-project-id"}[5m])
  )
  /
  sum by (model_id) (
    aiplatform_googleapis_com:prediction_online_dedicated_token_limit{resource_container="your-project-id"}
  )
) > 0.9
---

Working one -

(
  sum by (model_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput{resource_container="your-project-id"}[5m])
  )
  /
  sum by (model_id) (
    aiplatform_googleapis_com:publisher_online_serving_dedicated_token_limit{resource_container="your-project-id"}
  )
) > 0.9
