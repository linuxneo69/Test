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

---
To alert on any response code starting with 5 in your unified project, use the Regex Matcher (=~) with the pattern 5...
# Alert on any 5xx error across models and projects
sum by (model_id, resource_container) (
  rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count{
    response_code=~"5.."
  }[2m])
) * 120 > 5

---

