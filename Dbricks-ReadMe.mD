# Databricks on AWS – SRE & Observability Deep Dive Guide

## Purpose: Learning, Onboarding, and Internal Knowledge Sharing
## Audience: Experienced SRE / DevOps / Observability Engineers

---

# 📌 Table of Contents

1. Introduction
2. Databricks Architecture (AWS)
3. Core Components Explained
4. SRE Responsibilities in Databricks
5. Observability in Databricks (Native)
6. Logs & Telemetry
7. AWS Integration Layer
8. External Observability Integration
9. Monitoring Strategy (Recommended)
10. Failure Scenarios & Debugging
11. Cost Observability
12. Security & Audit Observability
13. Best Practices
14. Quick Start Checklist
15. Advanced Topics

---

# 1. Introduction

Databricks is a **unified data analytics and AI platform** built on Apache Spark that enables:
- Distributed data processing
- Machine learning workflows
- SQL analytics
- Data lakehouse architecture

For SREs, Databricks introduces **new observability challenges**:
- Ephemeral compute (clusters)
- Distributed execution (Spark)
- Data pipeline reliability
- Cost-performance tradeoffs

---

# 2. Databricks Architecture (AWS)

## 2.1 Control Plane vs Data Plane

### Control Plane (Databricks Managed)
- Workspace UI
- REST APIs
- Job scheduler
- Cluster manager

### Data Plane (Customer AWS Account)
- EC2 instances (clusters)
- S3 (data + logs)
- VPC networking
- IAM roles

### Key Insight:
> You **only control and monitor Data Plane infra directly**.

---

## 2.2 High-Level Flow

User → Workspace UI → Job → Cluster → Spark Execution → S3 Storage

---

# 3. Core Components Explained

## 3.1 Workspace
- Logical environment
- Contains notebooks, jobs, dashboards

## 3.2 Clusters (Critical for SRE)
- Driver node + Worker nodes
- Backed by EC2 instances
- Supports autoscaling

### Cluster Types:
- All-purpose (interactive)
- Job clusters (ephemeral, recommended)

---

## 3.3 Jobs
- Production pipelines
- Scheduled or event-triggered

---

## 3.4 Delta Lake
- ACID-compliant storage layer
- Time travel capability
- Schema enforcement

---

## 3.5 Unity Catalog
- Centralized governance
- Data lineage
- Access control

---

# 4. SRE Responsibilities in Databricks

## 4.1 Infrastructure Reliability
- Cluster health monitoring
- Node lifecycle tracking
- Autoscaling validation

## 4.2 Job Reliability
- Failures and retries
- SLA adherence
- Execution delays

## 4.3 Performance Monitoring
- Spark job latency
- Resource utilization
- Data skew

## 4.4 Cost Optimization
- Idle clusters
- Over-provisioned workloads
- DBU consumption

## 4.5 Data Pipeline Observability
- Data freshness
- Missing partitions
- Schema drift

---

# 5. Observability in Databricks (Native)

## 5.1 Cluster Metrics
Location: Compute → Cluster → Metrics

Metrics include:
- CPU usage
- Memory usage
- JVM metrics
- Executor metrics

Limitations:
- No long-term retention
- No alerting

---

## 5.2 Spark UI

Provides:
- DAG visualization
- Stage-level metrics
- Task-level execution
- Shuffle analysis

Use Cases:
- Performance debugging
- Bottleneck identification
- Skew detection

---

## 5.3 Job Monitoring

Location: Workflows → Jobs

Provides:
- Run history
- Execution duration
- Failure logs
- Retry tracking

---

## 5.4 SQL & Query Monitoring

- Query history
- Execution plans
- Warehouse metrics

---

# 6. Logs & Telemetry

## 6.1 Log Types

### Cluster Logs
- Driver logs
- Executor logs
- stdout / stderr

### Job Logs
- Notebook output
- Execution logs

### Audit Logs
- User activity
- API access
- Permissions changes

---

## 6.2 Log Delivery (CRITICAL)

Recommended:
- Configure cluster log delivery to S3

Example Path:
```

s3://<bucket>/databricks-logs/<cluster-id>/

```

---

# 7. AWS Integration Layer

## 7.1 Key AWS Services

- EC2 → Compute
- S3 → Storage & logs
- CloudWatch → Metrics & logs
- IAM → Access control
- VPC → Networking

---

## 7.2 What You Monitor in AWS

| Resource | Metrics |
|----------|--------|
| EC2 | CPU, Memory, Network |
| S3 | Storage growth |
| VPC | Network flow |
| IAM | Access anomalies |

---

# 8. External Observability Integration

## 8.1 Logging

Recommended:
- Splunk
- ELK Stack

Flow:
S3 → Log Forwarder → Observability Tool

---

## 8.2 Metrics

Options:
- Prometheus + Grafana
- Datadog
- New Relic

---

## 8.3 Data Observability

Tools:
- Great Expectations
- Monte Carlo

---

# 9. Monitoring Strategy (Recommended)

## 9.1 Layered Observability Model

| Layer | Tool |
|------|------|
| Infra | CloudWatch |
| Compute | Cluster Metrics |
| Execution | Spark UI |
| Pipeline | Jobs UI |
| Logs | S3 + Splunk |

---

## 9.2 Key Signals

### Golden Signals:
- Latency (job duration)
- Errors (failures)
- Saturation (CPU/memory)
- Traffic (job frequency)

---

# 10. Failure Scenarios & Debugging

## 10.1 Cluster Startup Failure
Check:
- IAM roles
- Instance limits
- VPC configuration

---

## 10.2 Job Failure

Steps:
1. Check job run logs
2. Open Spark UI
3. Identify failing stage
4. Review executor logs

---

## 10.3 Slow Jobs

Check:
- Data skew
- Shuffle size
- Executor imbalance

---

## 10.4 Out of Memory

Symptoms:
- Executor crashes
- GC overhead

Fix:
- Increase memory
- Optimize partitioning

---

# 11. Cost Observability

## 11.1 Key Cost Drivers
- Cluster uptime
- Node type
- DBU usage

---

## 11.2 Optimization Strategies

- Use job clusters
- Enable autoscaling
- Auto-terminate idle clusters

---

# 12. Security & Audit Observability

## 12.1 Audit Logs

Track:
- Login events
- API usage
- Permission changes

---

## 12.2 Best Practices

- Centralize logs
- Monitor anomalies
- Integrate with SIEM

---

# 13. Best Practices

- Always use job clusters for production
- Enable log delivery to S3
- Avoid long-running clusters
- Monitor autoscaling behavior
- Use tagging for cost tracking

---

# 14. Quick Start Checklist

## Day 1
- Access workspace
- Explore clusters
- Run sample notebook

## Day 2
- Analyze Spark UI
- Trigger job failure

## Day 3
- Configure log delivery
- Integrate with CloudWatch

---

# 15. Advanced Topics

- Delta Lake optimization (Z-ordering)
- Structured streaming observability
- ML model monitoring
- Unity Catalog lineage tracking

---

# 🚀 Key Takeaways

- Databricks is NOT a full observability platform
- Observability is **distributed across layers**
- Logs are your most reliable source of truth
- AWS integration is essential
- External tools are required for production-grade monitoring

---

# 📚 Suggested Next Steps

1. Hands-on cluster exploration
2. Create and fail a job intentionally
3. Analyze Spark UI deeply
4. Integrate logs with observability stack
5. Build internal dashboards

---

