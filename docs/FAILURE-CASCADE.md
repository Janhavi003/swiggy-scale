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
