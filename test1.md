### Part 1: Jira Ticket Documentation

**Jira Ticket ID:** JIRA-1234

**Title:** Strategic Transition: Unified Multi-Project & Auto-Discovery Alerting for Vertex AI

**Description:**
The current alerting infrastructure for Vertex AI (Gemini 2.5 Pro/Flash) is managed via independent, project-specific policies. While functional, this "old way" creates significant maintenance overhead and lacks scalability as new models (Claude, Gemini updates) or new projects are introduced.

**Objective:**
Implement a centralized, "Zero-Hardcode" alerting strategy using Google Cloud Metrics Scoping and PromQL-based auto-discovery.

**Key Technical Requirements:**

* **Unified Metrics Scope:** Configure a central scoping project to aggregate metrics from Projects 1 through 10.
* **Auto-Discovery Logic:** Replace hardcoded `model_user_id` and `resource_container` filters with dynamic `sum by` grouping in PromQL.
* **Multi-Code Consolidation:** Merge individual error alerts (429, 499, 5xx) into a single "Universal Error Policy" using regex code matching.
* **Dynamic Notification Injection:** Utilize variable expansion (`${metric.label.resource_container}`) in notification subject lines to ensure instant incident identification.

**Business Value:**

* **Zero Maintenance:** New models are discovered and monitored automatically upon deployment.
* **Reduced Alert Fatigue:** Consolidated policies reduce "noise" while providing more granular data in the subject line.
* **Scalability:** Allows the AI initiative to grow from 10 to 100 projects without requiring additional alerting configurations.

**Acceptance Criteria:**

1. Verified Metrics Scope across all production projects.
2. Deployment of Universal Error, Burndown, and Spillover policies.
3. Confirmation of dynamic variables appearing correctly in Slack/Email test alerts.

---

### Part 2: Teams Status Update

"Hi everyone! Quick update on the Vertex AI monitoring front. I’ve successfully deployed the initial alerting suite for **Projects 1, 2, 3, 4, and 5**. This covers our immediate production needs with dedicated alerts for response codes (429, 499, 5xx), GSU burndown, and spillover ratios for each specific model. While this 'v1' setup ensures we are fully covered today, I’ve also opened **Jira1234** to transition us to a more strategic, 'auto-discovery' model. This new approach will consolidate our alerts into a single unified policy that automatically detects and monitors new projects and models as they launch. It’s a much leaner setup that removes hardcoded values and uses dynamic variables to tell us exactly which project and model are failing directly in the alert subject line—keeping us ahead of the curve as our model portfolio expands. Let me know if you have any questions!"

---

### 💡 Pro-Tip for the Transition

As you move to the "New Way" in **Jira1234**, keep the "Old Way" running for 48 hours in parallel. Once you confirm the dynamic subject lines are firing correctly for Projects 1-5, you can delete the old manual alerts with 100% confidence.
