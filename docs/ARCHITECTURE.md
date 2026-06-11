# Architecture Redesign for 10 Million Users

---

# Section 1: Current Architecture (Monolith)

## Existing System

```text
                    10M Users
                        |
                        v
      +----------------------------------+
      |      Node.js Express Server      |
      |      1 CPU · 4GB RAM             |
      |                                  |
      | - Restaurant APIs                |
      | - Order Placement                |
      | - Payment Processing             |
      | - Promo Validation               |
      | - Static Asset Serving           |
      +----------------------------------+
                        |
                        v
      +----------------------------------+
      |       PostgreSQL Database        |
      |      max_connections = 100       |
      +----------------------------------+
```

## Weaknesses

```text
Node.js Server
   |
   +--> Failure 2: Event loop saturation (~12K RPS)

Node.js Server
   |
   +--> Failure 5: NIC saturation (40TB image traffic)

PostgreSQL
   |
   +--> Failure 1: Connection pool exhaustion (~394 RPS)

Payment Processing
   |
   +--> Failure 3: Synchronous payment amplification

Promo Validation
   |
   +--> Failure 4: Promo race condition

Entire Stack
   |
   +--> Single point of failure
```

---

# Section 2: Redesigned Architecture

## High-Level Architecture

```text
                         Users
                           |
                           v
+--------------------------------------------------+
|                CloudFront CDN                    |
|  Cache: Images, JS, CSS, Menus                   |
|  TTL: 5 Minutes (Menus)                          |
|  TTL: 24 Hours (Static Assets)                   |
+--------------------------------------------------+
                           |
                           v
+--------------------------------------------------+
|       Application Load Balancer (ALB)            |
| - SSL Termination                                |
| - Health Checks                                  |
| - Rate Limiting (100 req/IP/sec)                 |
+--------------------------------------------------+
          |           |           |           |
          v           v           v           v
      +------+    +------+    +------+    +------+
      | N1   |    | N2   |    | N3   |    | N4   |
      +------+    +------+    +------+    +------+

      Baseline: 4 instances
      Peak Event: Auto-scale to 20 instances

      Auto-scale Trigger:
      CPU > 70% for 3 minutes

                     |
                     v

+--------------------------------------------------+
|               Redis Cluster                      |
|                                                  |
| Cache: Restaurant Menus                          |
| TTL: 5 Minutes                                   |
| Cache Hit Target: 80%+                           |
|                                                  |
| Promo Lock: Redis SETNX                          |
+--------------------------------------------------+

                     |
                     v

+--------------------------------------------------+
|                 PgBouncer                         |
|                                                  |
| Connection Multiplexing                          |
| 100 Physical Connections                         |
| 5000+ Logical Connections                        |
+--------------------------------------------------+

                     |
                     v

         +-----------------------------+
         | PostgreSQL Primary           |
         | Writes Only                  |
         +-----------------------------+
                |                |
                |                |
                v                v

     +----------------+   +----------------+
     | Read Replica 1 |   | Read Replica 2 |
     | Restaurant     |   | Order History  |
     | Queries        |   | Queries        |
     +----------------+   +----------------+

                     |
                     v

+--------------------------------------------------+
|               Amazon SQS Queue                   |
|                                                  |
| Async Payment Requests                           |
| Dead Letter Queue (DLQ)                          |
+--------------------------------------------------+

                     |
                     v

+--------------------------------------------------+
|             Payment Worker Service               |
|                                                  |
| Reads Messages from SQS                          |
| Calls Razorpay                                   |
| Updates Payment Status                           |
+--------------------------------------------------+

                     |
                     v

                  Razorpay
```

---

## Payment Flow

### Old Flow

```text
User
  |
Order Request
  |
Node.js
  |
PostgreSQL Connection Held
  |
Wait 800ms for Razorpay
  |
Release Connection
```

Problem:

Database connection occupied during payment processing.

---

### New Flow

```text
User
  |
Order Request
  |
Write Order (10ms)
  |
Publish Message to SQS
  |
Release Connection
  |
Return Success

Background Worker
  |
Read SQS
  |
Call Razorpay
  |
Update Payment Status
```

Benefit:

Database connection released almost immediately.

---

# Section 3: Component Justification Table

| Component                  | Failure It Prevents                   | How It Prevents It                                                                             |
| -------------------------- | ------------------------------------- | ---------------------------------------------------------------------------------------------- |
| CloudFront CDN             | Failure 5: NIC Saturation             | Static assets served from edge locations so Node.js never receives image traffic               |
| Application Load Balancer  | Single Point of Failure               | Distributes requests across multiple application servers and performs health checks            |
| Auto Scaling Node.js Fleet | Failure 2: Event Loop Saturation      | Adds additional instances automatically when CPU exceeds 70%                                   |
| Redis Cache                | Failure 1: Connection Pool Exhaustion | Removes 80%+ of menu reads from PostgreSQL                                                     |
| Redis SETNX Promo Lock     | Failure 4: Promo Race Condition       | Provides atomic promo validation and deduction                                                 |
| PgBouncer                  | Failure 1: Connection Pool Exhaustion | Multiplexes thousands of application requests onto a smaller number of physical DB connections |
| PostgreSQL Read Replicas   | Read/Write Contention                 | Separates read traffic from write traffic                                                      |
| Amazon SQS                 | Failure 3: Payment Amplification      | Converts synchronous payment processing into asynchronous processing                           |
| Payment Worker Service     | Failure 3: Payment Amplification      | Handles Razorpay calls outside the user request path                                           |
| Dead Letter Queue (DLQ)    | Payment Reliability Issues            | Captures failed payment events for retry and investigation                                     |

---

# Architecture Summary

The redesigned architecture eliminates every major failure identified in the cascade analysis. CloudFront removes image traffic from the application servers, Redis reduces database load, PgBouncer increases effective database capacity, SQS removes payment latency from the critical request path, and horizontal scaling through ALB and Auto Scaling Groups allows the platform to absorb large traffic spikes.

The new design can scale from normal daily traffic to major event traffic while maintaining low latency and preventing the cascading failures observed in the monolithic architecture.
