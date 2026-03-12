(
  sum by (resource_container, model_user_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput{request_type="spillover"}[15m])
  )
  /
  on(resource_container, model_user_id)
  sum by (resource_container, model_user_id) (
    rate(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput[15m])
  )
) > 0.20
