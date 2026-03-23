Hi Team,

Thank you for the response and for looking into this.

I wanted to share a clearer timeline and the current behavior we are seeing in our Cloud Monitoring PromQL alert policies for Vertex AI.

**Summary of the issue**

* We are using PromQL-based alert policies for unified Vertex AI monitoring across a metrics scope.
* Until around **12 March**, our **GSU Burndown** alert was successfully expanding `resource_container` and `model_user_id` in email notifications.
* After that date, the **project value** in the Burndown email started showing as `null` when using `${metric.label.resource_container}`.
* We then changed the project variable to `${resource.project}`, and that resolved the Burndown alert notification.
* However, our other PromQL alerts — including **PT Spillover**, **HTTP error code alerts**, **Non-200 alerts**, and **predictive alerts** — still show the **project name correctly** with `${resource.project}`, but the **model name (`model_user_id`) continues to show as `null`**.

**What we have already verified**

* In **Metrics Explorer**, the PromQL query output does show both:

  * `resource_container`
  * `model_user_id`
* The issue appears only in the **alert notification expansion**, not in the chart/query output itself.
* We have tried the following variable formats in the email subject/body:

  * `${metric.label.resource_container}`
  * `${metric.label.model_user_id}`
  * `${labels.resource_container}`
  * `${labels.model_user_id}`
  * `${metric_or_resource.label.resource_container}`
  * `${metric_or_resource.label.model_user_id}`
  * `${resource.project}` for the project value, which works
* So far, only the **project label** can be reliably shown using `${resource.project}`. The **model label** still does not expand correctly in the notification for the other alert policies.

**Current behavior**

* **Burndown alert**: project and model now display correctly after switching to `${resource.project}` for the project field.
* **PT Spillover / Non-200 / HTTP error / prediction alerts**: project displays correctly with `${resource.project}`, but model still shows `null`.

**What we are asking**
Could you please help us determine:

1. Whether this is a known limitation or bug in PromQL alert notification variable expansion.
2. Whether `model_user_id` should be referenced differently for these Vertex AI metrics.
3. Whether there is any documented workaround for getting `model_user_id` to expand correctly in alert notifications.

**Screenshots / evidence**

* [Screenshot 1: Email notification when Burndown was working correctly]
* [Screenshot 2: Email notification after project label started showing null]
* [Screenshot 3: Current PT Spillover / Non-200 alert notification showing project correct but model as null]
* [Screenshot 4: Metrics Explorer output showing both `resource_container` and `model_user_id` are present in the query result]

We would appreciate any guidance you can provide, especially if there is a supported way to reliably surface `model_user_id` in PromQL alert notifications for Vertex AI metrics.

Thanks again for your help.
