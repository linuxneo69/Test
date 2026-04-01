
# Google Cloud Operations Onboarding and Production Readiness Test Plan: Application Monitoring

## 1. Introduction

This document defines the production readiness test plan for validating the **unified observability scope** in GCO for application monitoring. The goal is to confirm that the onboarded applications are fully visible and operationally usable through **Metrics Explorer, Logs Explorer, dashboards, and alerting**, and that behavior is consistent across the two unified projects when compared with the traditional project-based monitoring setup.

The validation is intended to ensure that:

* metrics are collected and queryable,
* logs are visible and searchable,
* dashboards accurately reflect the monitored services,
* alerting rules fire correctly and provide meaningful incident context,
* scoping is configured correctly for the onboarded projects.

---

## 2. Scope

This test plan covers validation of the following areas:

* Observability scope and project inclusion
* Metrics Explorer
* Logs Explorer
* Monitoring dashboards
* Alerting policies
* Comparison between unified observability and traditional project monitoring

The test plan applies to:

* **Unified Project 1**
* **Unified Project 2**
* **Traditional Project**

---

## 3. Objectives

The objectives of this test plan are to confirm that:

1. the unified observability scope contains the correct projects,
2. metrics and logs are visible from all intended sources,
3. dashboards reflect accurate and current data,
4. alerts trigger at the correct thresholds,
5. notification content is actionable and includes the right context,
6. monitoring behavior is consistent with the traditional setup where applicable.

---

## 4. Test Approach

Validation should be performed using the following approach:

* Execute each test in **Metrics Explorer**, **Logs Explorer**, dashboards, and alerting.
* Compare unified project behavior against the traditional project.
* Confirm both **visibility** and **accuracy**.
* Capture evidence for every major test scenario.
* Record any differences in label behavior, freshness, aggregation, or alert delivery.

Recommended comparison method:

* run the same query or scenario in the unified scope,
* run the same query or scenario in the traditional project,
* compare the results side by side.

---

## 5. Test Environment

| Environment         | Description                                                 |
| ------------------- | ----------------------------------------------------------- |
| Unified Project 1   | First project onboarded into unified observability scope    |
| Unified Project 2   | Second project onboarded into unified observability scope   |
| Traditional Project | Existing project using traditional project-based monitoring |
| GCO Scope           | Unified monitoring scope used for the test                  |

---

## 6. Test Data and Evidence

For each test case, capture the following where relevant:

* Screenshot of Metrics Explorer query result
* Screenshot of Logs Explorer search result
* Screenshot of dashboard panel
* Screenshot of alert notification
* Timestamp of validation
* Project name tested
* Notes on expected vs actual result

---

## 7. Test Cases

### 7.1 Scoping Validation

| Test ID | Test Area | Test Scenario                                                         | Expected Result                                                                      |
| ------- | --------- | --------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| SC-01   | Scope     | Confirm both unified projects are included in the observability scope | Both unified projects are visible and accessible                                     |
| SC-02   | Scope     | Confirm unrelated projects are excluded                               | No unexpected projects appear in scope                                               |
| SC-03   | Scope     | Validate the scope is returning data for intended resources           | Metrics and logs are queryable from both unified projects                            |
| SC-04   | Scope     | Compare unified scope vs traditional project                          | Unified scope returns cross-project visibility; traditional project remains isolated |

**How to test:**
Open the observability scope and verify project membership. Then use Metrics Explorer and Logs Explorer to confirm the scope is returning data for both unified projects.

---

### 7.2 Metrics Validation

| Test ID | Test Area | Test Scenario                                                   | Expected Result                                                                              |
| ------- | --------- | --------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| MT-01   | Metrics   | Verify base application metrics are visible in Metrics Explorer | Metrics appear for both unified projects                                                     |
| MT-02   | Metrics   | Verify metrics are updated within an acceptable delay           | Data appears within the expected freshness window                                            |
| MT-03   | Metrics   | Validate label availability                                     | Metric labels such as service, instance, environment, or request type are present and usable |
| MT-04   | Metrics   | Validate cross-project aggregation                              | Combined unified query reflects both projects correctly                                      |
| MT-05   | Metrics   | Compare with traditional project                                | Traditional project shows only its own metrics, unified scope shows both                     |

**How to test:**
Run key application metric queries in Metrics Explorer. Confirm values are present, labels are correct, and the combined result matches the expected totals.

---

### 7.3 Logs Validation

| Test ID | Test Area | Test Scenario                                       | Expected Result                                       |
| ------- | --------- | --------------------------------------------------- | ----------------------------------------------------- |
| LG-01   | Logs      | Confirm logs are visible from both unified projects | Logs appear in Logs Explorer                          |
| LG-02   | Logs      | Filter logs by project                              | Only logs from the selected project appear            |
| LG-03   | Logs      | Filter logs by severity                             | Results change according to severity selection        |
| LG-04   | Logs      | Filter logs by service or workload                  | Relevant application logs are returned                |
| LG-05   | Logs      | Correlate logs with a metric spike                  | Logs match the timeframe of the observed metric event |

**How to test:**
Search Logs Explorer using project, severity, and workload filters. Validate that logs are visible from both unified projects and can be correlated to metric spikes or alert windows.

---

### 7.4 Dashboard Validation

| Test ID | Test Area | Test Scenario                                       | Expected Result                                            |
| ------- | --------- | --------------------------------------------------- | ---------------------------------------------------------- |
| DB-01   | Dashboard | Open the unified monitoring dashboard               | Dashboard loads without errors                             |
| DB-02   | Dashboard | Confirm panels show data from both unified projects | Panels reflect combined or project-specific data correctly |
| DB-03   | Dashboard | Compare dashboard values with Metrics Explorer      | Values are consistent                                      |
| DB-04   | Dashboard | Drill into a panel                                  | Drill-down opens the expected metric or log view           |
| DB-05   | Dashboard | Validate time range consistency                     | All panels reflect the selected time range correctly       |

**How to test:**
Open the dashboard, verify each panel, and compare values against Metrics Explorer. Confirm no panels are blank, stale, or misconfigured.

---

### 7.5 Alerting Validation

| Test ID | Test Area | Test Scenario                                                 | Expected Result                                          |
| ------- | --------- | ------------------------------------------------------------- | -------------------------------------------------------- |
| AL-01   | Alerting  | Trigger an alert condition intentionally                      | Alert fires correctly                                    |
| AL-02   | Alerting  | Confirm alert does not fire during normal operation           | No false positive alert occurs                           |
| AL-03   | Alerting  | Validate alert routing                                        | Notification reaches the correct channel or team         |
| AL-04   | Alerting  | Validate alert subject and documentation                      | Message contains clear and useful context                |
| AL-05   | Alerting  | Validate alert grouping and deduplication                     | Alerts are grouped correctly and do not flood recipients |
| AL-06   | Alerting  | Compare alert behavior across unified and traditional project | Alert behavior is consistent where expected              |

**How to test:**
Use a controlled threshold breach or test alert scenario. Confirm that the notification is received and includes the correct project, service, severity, and condition context.

---

## 8. Recommended Alert Validation Checklist

When testing alert policies, confirm the following:

* alert condition evaluates correctly,
* notification is delivered,
* subject line contains useful summary details,
* documentation/body contains enough context for triage,
* alert is routed to the right responder,
* alert is grouped correctly if multiple resources are affected,
* alert clears appropriately after the issue is resolved.

---

## 9. Acceptance Criteria

The unified observability setup is considered production-ready only if all of the following are true:

* both unified projects are visible in the observability scope,
* metrics are visible, fresh, and accurate,
* logs are searchable and correlate with metric events,
* dashboards are complete and consistent,
* alerting triggers correctly and routes to the proper channel,
* no critical visibility gaps exist compared with the traditional project,
* the monitoring setup supports incident investigation and operational decision-making.

---

## 10. Evidence Matrix

| Area       | Evidence Required                                                    |
| ---------- | -------------------------------------------------------------------- |
| Scoping    | Screenshot of scope configuration and included projects              |
| Metrics    | Metrics Explorer screenshots with query and labels                   |
| Logs       | Logs Explorer screenshots with filters and timestamps                |
| Dashboards | Dashboard screenshots showing key panels                             |
| Alerting   | Alert notification screenshot and condition details                  |
| Comparison | Notes highlighting differences between unified and traditional setup |

---

## 11. Review Notes and Improvements Applied

A few improvements were made to make the plan easier to execute and review:

* Removed duplicate wording and tightened the language.
* Added a clear comparison structure between unified and traditional monitoring.
* Separated testing into **scoping, metrics, logs, dashboards, and alerting** so it is easier to track.
* Added explicit expected results for each test case.
* Added an evidence matrix so reviewers know what to capture.
* Added acceptance criteria so sign-off is clear and objective.

---

## 12. Summary

This test plan validates that the unified observability setup provides accurate and reliable monitoring across the onboarded applications. It ensures that the transition from traditional project-based monitoring to unified scope-based monitoring does not reduce visibility, observability quality, or operational readiness.

--------


## 2. Pre-requisites

Before executing the test scenarios, ensure the following conditions are met:

### 2.1 Data Availability

- Application metrics must be actively generated and visible in **Metrics Explorer**.
- Application logs must be ingested and queryable in **Logs Explorer**.
- Data should be available for a recent time window (last 15–30 minutes minimum).

### 2.2 Observability Scope Configuration

- Unified observability scope is correctly configured.
- All intended projects (Unified Project 1 and Unified Project 2) are added to the scope.
- No unintended projects are included.

### 2.3 Test Data / Trigger Readiness

- Ability to generate test traffic or simulate conditions (for alert validation).
- Known baseline behavior of the application (normal vs abnormal conditions).

### 2.4 Time Synchronization

- Ensure all systems (application, logs, monitoring) are aligned in time zone and timestamps.
- Helps in accurate correlation between metrics, logs, and alerts.

