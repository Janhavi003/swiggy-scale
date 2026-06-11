# Understanding the SwiftEats Monolith

## Current Architecture

The current SwiftEats system consists of a single Node.js Express server connected to a single PostgreSQL database.

### System Components

* Single Node.js Express server

  * 1 process
  * 1 CPU
  * 4GB RAM

* Single PostgreSQL database

  * max_connections = 100

* No Redis cache

* No CDN

* No load balancer

* No auto-scaling

* Synchronous payment processing through Razorpay

  * Payment latency: 200ms–2000ms
  * Database connection remains occupied during payment processing

## System Weaknesses

The architecture contains multiple single points of failure.

1. The application layer consists of only one Node.js process. If the process crashes, the entire platform becomes unavailable.

2. The PostgreSQL database supports only 100 concurrent connections. During heavy traffic, connection exhaustion will occur long before the Node.js server reaches its theoretical request-processing limit.

3. There is no cache layer. Every restaurant listing, menu lookup, and order query directly hits PostgreSQL.

4. Static assets such as restaurant images are served directly from the application server, increasing network and CPU load.

5. Payment processing is synchronous. Database connections remain occupied while waiting for external payment gateways to respond.

## Weakest Component

The weakest component in the system is PostgreSQL.

The database allows only 100 concurrent connections. Since payment requests can hold connections for hundreds of milliseconds or even several seconds, the connection pool becomes exhausted very quickly under load.

Once all database connections are occupied:

* New requests cannot access the database
* Request queues begin growing
* Response times increase rapidly
* Timeouts and errors appear

## Hard Limits

The primary hard limits of the system are:

| Component       | Limit                      |
| --------------- | -------------------------- |
| PostgreSQL      | 100 concurrent connections |
| Node.js         | ~12,000 RPS                |
| Memory          | 4GB heap                   |
| Payment Latency | 200ms–2000ms               |

Among these limits, the PostgreSQL connection limit is reached first.

## Expected First Failure

The first failure expected during the World Cup promotion is PostgreSQL connection pool exhaustion.

When millions of users begin browsing restaurants and placing orders simultaneously, database connections become occupied faster than they can be released. Once the pool reaches 100 active connections, PostgreSQL rejects new requests.

This triggers a cascading failure:

1. PostgreSQL connection pool exhaustion
2. Request queue growth inside Node.js
3. Event loop saturation
4. Increased memory usage
5. Out-of-memory crash
6. Complete service outage

Therefore, PostgreSQL is the first bottleneck and the first component expected to fail during the traffic spike.

# Failure Cascade Analysis

---

# Section 1: Traffic Simulation Math

## Event Scenario

SwiftEats plans to send a 50% discount promotion during the India vs Pakistan World Cup Final.

### Traffic Assumptions

| Metric                                      | Value       |
| ------------------------------------------- | ----------- |
| Push notifications sent                     | 180,000,000 |
| Click-through rate                          | 8%          |
| Users opening app                           | 14,400,000  |
| Concurrent active users used for simulation | 10,000,000  |
| API calls per user                          | 3           |
| Spike duration                              | 60 seconds  |

### API Requests Per User

Most users perform the following actions:

```text
GET /restaurants
GET /restaurant/:id
POST /orders
```

Average API calls:

```text
3 calls per user
```

### Peak RPS Calculation

Formula:

Peak RPS = (Active Users × API Calls Per User) / Duration

Calculation:

10,000,000 × 3 / 60

= 500,000 RPS

### Result

Expected peak traffic:

500,000 requests per second

### Capacity Comparison

| System             | RPS     |
| ------------------ | ------- |
| SwiftEats Monolith | ~12,000 |
| World Cup Demand   | 500,000 |

Demand exceeds capacity by:

500,000 ÷ 12,000

= 41.67×

The system is expected to fail immediately without architectural changes.

---

# Section 2: Component Capacity Numbers

## PostgreSQL

Configuration:

```text
max_connections = 100
```

Hard limit:

100 simultaneous database connections

---

## Node.js Server

Configuration:

```text
1 CPU
4GB RAM
1 Process
```

Estimated capacity:

```text
12,000–15,000 RPS
```

Above this level:

* Event loop latency increases
* Callback queue backs up
* Request latency rises sharply

---

## Payment Gateway

External provider:

```text
Razorpay
```

Latency:

```text
200ms – 2000ms
```

Average latency used:

```text
800ms
```

Database connections remain occupied while waiting for payment completion.

---

## Memory Limit

Available memory:

```text
4GB
```

Expected failure:

```text
~15,000 queued requests
```

Outcome:

```text
Node.js OOM Crash
```

---

## Database Pool Exhaustion Calculation

Assumptions:

```text
Pool Size = 100

70% normal requests
30% payment requests

Normal Query Time = 20ms = 0.02s
Payment Hold Time = 800ms = 0.8s
```

Weighted connection hold time:

(0.7 × 0.02) + (0.3 × 0.8)

= 0.014 + 0.24

= 0.254 seconds

Maximum sustainable throughput:

100 ÷ 0.254

≈ 394 RPS

### Result

PostgreSQL pool exhaustion occurs at approximately:

394 RPS

This is dramatically lower than the incoming demand of:

500,000 RPS

---

# Section 3: Failure Cascade

## Failure 1: PostgreSQL Connection Pool Exhaustion

Severity: CRITICAL

Trigger:

~394 RPS

User Impact:

* Requests timeout
* Order placement fails
* Restaurant pages fail to load
* Database connection errors appear

What Breaks Next:

Node.js begins accumulating pending requests waiting for database access.

---

## Failure 2: Node.js Event Loop Saturation

Severity: CRITICAL

Trigger:

~12,000–15,000 RPS

User Impact:

* Slow page loads
* Increased latency
* HTTP 500 responses
* Request timeouts

What Breaks Next:

Memory consumption rises rapidly, leading to OOM crashes.

---

## Failure 3: Synchronous Payment Call Amplification

Severity: HIGH

Trigger:

Payment latency above 800ms

User Impact:

* Checkout delays
* Failed payments
* Duplicate payment attempts

What Breaks Next:

Database connections remain occupied longer, accelerating pool exhaustion.

---

## Failure 4: Promo Code Race Condition

Severity: HIGH

Trigger:

Millions of simultaneous promo validations

User Impact:

* Promo applied incorrectly
* Budget oversold
* Users receive conflicting discount states

What Breaks Next:

Revenue loss and inconsistent customer experience.

---

## Failure 5: Static Asset NIC Saturation

Severity: CRITICAL

Trigger:

Restaurant images served directly from Node.js

Traffic Calculation:

10,000,000 users × 20 images × 200KB

= 40TB

User Impact:

* Images fail to load
* Application becomes unresponsive
* API requests compete with image traffic

What Breaks Next:

Complete network saturation and service outage.

---

# Section 4: Incident Timeline

| Time  | Event                                               |
| ----- | --------------------------------------------------- |
| T+0s  | Push notification sent                              |
| T+3s  | PostgreSQL connection pool exhausted                |
| T+5s  | Node.js request queue begins growing                |
| T+8s  | New database connections rejected                   |
| T+10s | Payment gateway timeouts begin                      |
| T+12s | Promo race condition oversells discounts            |
| T+15s | Static asset traffic saturates network              |
| T+18s | Node.js process crashes with OOM                    |
| T+20s | Service marked unhealthy                            |
| T+45m | Root cause identified by on-call engineer           |
| T+2h  | Service restored after restart and database cleanup |

---

# Conclusion

The first component to fail is PostgreSQL due to connection pool exhaustion at approximately 394 RPS. Once the database becomes unavailable, Node.js accumulates queued requests, leading to event-loop saturation, memory exhaustion, payment failures, and eventually a full platform outage. The failure follows a predictable cascade driven by hard infrastructure limits rather than random software defects.
