# SwiftEats Incident Response Runbook

## Purpose

This runbook provides a step-by-step guide for detecting, diagnosing, responding to, and documenting production incidents in the SwiftEats platform.

Target audience:

* Junior engineers
* On-call engineers
* Site Reliability Engineers (SREs)

Goal:

Identify the root cause of an outage within 30 seconds and restore service as quickly as possible.

---

# STEP 1 – DETECT

## Required CloudWatch Alarms

| Metric                 | Threshold                | Alert Type | Reason                        |
| ---------------------- | ------------------------ | ---------- | ----------------------------- |
| ALB 5XX Error Rate     | > 5% for 2 minutes       | Critical   | Users receiving server errors |
| PostgreSQL Connections | > 80% of max_connections | Warning    | Risk of pool exhaustion       |
| PostgreSQL Connections | > 95% of max_connections | Critical   | Imminent outage               |
| EC2 CPU Utilization    | > 80% for 3 minutes      | Warning    | Compute saturation            |
| Redis Memory Usage     | > 75%                    | Warning    | Risk of cache eviction        |
| Redis Cache Miss Rate  | > 50%                    | Critical   | Cache failure                 |
| SQS Queue Depth        | > 10,000 messages        | Warning    | Payment backlog               |
| SQS Queue Depth        | > 50,000 messages        | Critical   | Payment processing failure    |
| P99 API Latency        | > 2 seconds              | Critical   | User experience degradation   |
| Promo Budget Remaining | < 10%                    | Warning    | Risk of overselling discounts |

---

# STEP 2 – TRIAGE

## Goal

Identify the root cause within 30 seconds.

Follow this checklist in order.

---

### Check 1 – Database Connections

Dashboard:

```text id="crz6nj"
CloudWatch
→ RDS
→ DatabaseConnections
```

Question:

```text id="rvf0qs"
Connections > 95%?
```

YES →

Go to Step 3A

NO →

Continue

---

### Check 2 – EC2 CPU

Dashboard:

```text id="yrcj4e"
CloudWatch
→ EC2
→ CPUUtilization
```

Question:

```text id="zefccu"
All application servers > 80% CPU?
```

YES →

Go to Step 3B

NO →

Continue

---

### Check 3 – Redis Cache

Dashboard:

```text id="vovw2g"
ElastiCache
→ CacheMisses
```

Question:

```text id="wcbj5y"
Cache miss rate > 50%?
```

YES →

Go to Step 3C

NO →

Continue

---

### Check 4 – Payment Queue

Dashboard:

```text id="2a6e7r"
CloudWatch
→ SQS
→ ApproximateNumberOfMessagesVisible
```

Question:

```text id="o1uxzb"
Queue depth > 50,000?
```

YES →

Go to Step 3D

NO →

Escalate to Platform On-Call Team.

---

# STEP 3 – RESPOND

---

## 3A – Database Pool Exhaustion

### Symptoms

* DatabaseConnections > 95%
* Increased API latency
* Database timeout errors

### Immediate Actions

Check active connections:

```bash id="uzslpz"
SELECT count(*) FROM pg_stat_activity;
```

Check slow queries:

```sql id="8lt1to"
SELECT *
FROM pg_stat_activity
WHERE state != 'idle';
```

Increase PgBouncer pool if necessary.

Scale RDS vertically if saturation persists.

### Success Criteria

| Metric         | Target  |
| -------------- | ------- |
| DB Connections | < 70%   |
| P99 Latency    | < 500ms |
| API Error Rate | < 1%    |

Expected recovery:

```text id="vqwy2i"
5–10 minutes
```

Responsible Team:

```text id="v1dhgi"
#db-oncall
```

---

## 3B – Compute Saturation

### Symptoms

* CPU > 80%
* High response times
* Increased ALB errors

### Immediate Actions

Increase Auto Scaling Group size:

```bash id="j6k0jr"
aws autoscaling update-auto-scaling-group \
--auto-scaling-group-name swifteats-prod \
--desired-capacity 20
```

Verify new instances enter service.

### Success Criteria

| Metric     | Target  |
| ---------- | ------- |
| CPU        | < 70%   |
| ALB Errors | < 1%    |
| Latency    | < 500ms |

Expected recovery:

```text id="wjlwmn"
3–5 minutes
```

Responsible Team:

```text id="itmk7q"
#platform-oncall
```

---

## 3C – Redis Cache Miss Spike

### Symptoms

* Cache misses > 50%
* Increased DB load
* Rising response times

### Immediate Actions

Check Redis memory:

```bash id="f6t0o3"
INFO MEMORY
```

Verify cache TTL configuration.

Warm cache using menu preload jobs.

Restart failed cache nodes if necessary.

### Success Criteria

| Metric          | Target  |
| --------------- | ------- |
| Cache Miss Rate | < 10%   |
| Redis Memory    | < 75%   |
| DB Connections  | Reduced |

Expected recovery:

```text id="uq2dr7"
5 minutes
```

Responsible Team:

```text id="wuh5qa"
#platform-oncall
```

---

## 3D – Payment Queue Backup

### Symptoms

* SQS queue > 50,000
* Delayed payment confirmation
* Growing DLQ entries

### Immediate Actions

Check worker health.

Scale payment workers:

```bash id="rt93mv"
aws ecs update-service \
--cluster swifteats-prod \
--service payment-worker \
--desired-count 20
```

Inspect Dead Letter Queue.

Replay failed messages if required.

### Success Criteria

| Metric        | Target   |
| ------------- | -------- |
| Queue Depth   | < 10,000 |
| Worker Errors | < 1%     |
| DLQ Growth    | Stopped  |

Expected recovery:

```text id="okg0xg"
10–15 minutes
```

Responsible Team:

```text id="1rw2s5"
#payments-oncall
```

---

# STEP 4 – ROLLBACK

## When To Roll Back

Rollback only if ALL conditions are true:

* 5XX rate > 20%
* No improvement after mitigation
* Recent deployment occurred
* Root cause linked to application code

---

## When NOT To Roll Back

Do NOT roll back if:

* Issue is infrastructure-related
* Issue is caused by external providers
* Database is healthy and unchanged

---

## Rollback Command

```bash id="rjlwmm"
aws ecs update-service \
  --cluster swifteats-prod \
  --service api \
  --task-definition swifteats-api:PREVIOUS_STABLE_VERSION
```

---

## Critical Warning

```text id="4zt6e5"
Never rollback database schemas.

Rollback application code only.
```

Database schema rollbacks may cause data corruption and data loss.

---

# STEP 5 – POSTMORTEM TEMPLATE

Complete within 24 hours.

---

## Incident Summary

Describe:

* What happened
* When it started
* When it ended

---

## Timeline

Instructions:

List every major event with timestamps.

Example:

```text id="du2uxj"
20:00 Push notification sent

20:03 PostgreSQL pool exhausted

20:05 API latency exceeded 2 seconds

20:18 Node.js process crashed

20:45 Root cause identified

22:00 Service restored
```

---

## Root Cause

Instructions:

Identify the deepest technical cause.

Bad example:

```text id="br5mri"
Server crashed
```

Good example:

```text id="1h7k8v"
PgBouncer pool_size remained configured for old traffic profile, causing connection exhaustion at 394 RPS.
```

---

## Impact

Document:

* Duration
* Users affected
* Orders affected
* Revenue impact
* SLA violations

---

## What Worked Well

Document:

* Alerts that fired correctly
* Automation that reduced recovery time
* Successful mitigation actions

---

## What Failed

Document:

* Missing alerts
* Incorrect dashboards
* Manual interventions
* Communication gaps

---

## Action Items

Every action item must include an owner and due date.

| Action Item                             | Owner         | Due Date   |
| --------------------------------------- | ------------- | ---------- |
| Example: Increase Redis memory          | Platform Team | 2026-07-01 |
| Example: Improve payment worker scaling | Payments Team | 2026-07-05 |

---

# Runbook Validation

A first-year engineer should be able to:

1. Detect the incident.
2. Identify the root cause within 30 seconds.
3. Execute mitigation steps.
4. Determine whether rollback is required.
5. Complete the postmortem.

If any step requires guessing or tribal knowledge, update the runbook until it is unambiguous.
