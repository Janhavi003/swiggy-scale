# Swiggy Scale Simulation & Incident Architecture

A large-scale Site Reliability Engineering (SRE) and system design analysis of a Swiggy-like food delivery platform facing a massive World Cup Final traffic surge.

---

## Scenario

SwiftEats is a food delivery platform running on a monolithic architecture:

* Single Node.js Express server
* Single PostgreSQL database
* No Redis cache
* No CDN
* No load balancer
* Synchronous payment processing

During the India vs Pakistan World Cup Final, the platform launches a 50% discount promotion to 180 million users.

Expected outcome:

* ~14 million users open the application
* ~10 million users become active simultaneously
* Peak traffic reaches approximately 500,000 requests per second

This project analyzes how the existing system fails, redesigns the architecture to survive the event, estimates infrastructure costs, and provides an incident response runbook.

---

## Repository Contents

| Document                | Description                                                                                        |
| ----------------------- | -------------------------------------------------------------------------------------------------- |
| docs/FAILURE-CASCADE.md | Traffic simulation, capacity calculations, failure analysis, and incident timeline                 |
| docs/ARCHITECTURE.md    | Redesigned multi-tier architecture with scaling and reliability improvements                       |
| docs/COST-ESTIMATE.md   | AWS infrastructure cost calculations for baseline and peak-event scenarios                         |
| docs/RUNBOOK.md         | Production incident response guide with alerts, triage, recovery, rollback, and postmortem process |

---

## Key Findings

### 1. Peak Demand Reaches 500,000 RPS

```text
10,000,000 users × 3 API calls ÷ 60 seconds
= 500,000 RPS
```

The monolithic system can handle only approximately 12,000 RPS.

---

### 2. PostgreSQL Fails First

Database pool exhaustion occurs at approximately:

```text
394 RPS
```

because only 100 database connections are available and payment requests hold connections for up to 800ms on average.

---

### 3. Demand Exceeds Capacity by More Than 40×

```text
500,000 ÷ 12,000
≈ 41.67×
```

The platform cannot survive the event without architectural changes.

---

### 4. Static Assets Create Massive Network Load

```text
10M users × 20 images × 200KB
≈ 40TB
```

Without a CDN, image traffic alone can saturate the application server network interface.

---

### 5. A 45-Minute Outage Could Cost ₹189 Crore

```text
₹4.2 crore/minute × 45 minutes
= ₹189 crore
```

Infrastructure investment is significantly cheaper than outage-related business losses.

---

## Architecture Overview

The redesigned architecture replaces the monolithic system with a horizontally scalable platform built on CloudFront, Application Load Balancers, Redis, PgBouncer, PostgreSQL read replicas, Amazon SQS, and dedicated payment workers.

Static assets are served through a CDN, database load is reduced through caching and read replicas, payment processing becomes asynchronous, and application servers scale automatically during traffic spikes.

This design eliminates the major failure modes identified in the cascade analysis and enables the platform to handle large promotional events safely.

---

## Technologies Covered

### Application Layer

* Node.js
* Express.js

### Data Layer

* PostgreSQL
* PgBouncer
* Redis (ElastiCache)

### Cloud Infrastructure

* Amazon EC2
* Amazon RDS
* Amazon CloudFront
* Application Load Balancer (ALB)
* Amazon SQS
* Auto Scaling Groups

### Reliability Engineering

* Capacity Planning
* Failure Cascade Analysis
* Incident Response
* Monitoring & Alerting
* Postmortems
* Cost Optimization

---

## Learning Outcomes

This project demonstrates:

* Traffic and capacity modeling
* Failure cascade analysis
* Distributed system architecture design
* Database scaling strategies
* Caching and CDN design
* Incident management practices
* AWS infrastructure cost estimation
* SRE operational thinking

---

## Conclusion

Large-scale outages are rarely random. They occur when known system limits are exceeded. By identifying bottlenecks, quantifying capacity limits, and designing systems around those constraints, engineering teams can prevent cascading failures and maintain reliability during high-traffic events.

The redesigned SwiftEats architecture transforms a fragile monolith into a resilient, scalable platform capable of supporting millions of users during peak demand.
