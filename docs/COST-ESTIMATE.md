# AWS Cost Estimate

## Objective

Estimate the monthly AWS infrastructure cost required to support the redesigned SwiftEats architecture and compare it against the business impact of a major outage during the World Cup Final promotion.

---

# Scenario 1: Baseline Infrastructure

Assumption:

* ~100,000 Daily Active Users
* Normal business traffic
* 24×7 operation
* 720 hours/month

---

## EC2 Application Servers

Instance Type:

```text id="okm6w0"
t3.medium
2 vCPU
4GB RAM
```

Quantity:

```text id="fy6y4h"
4 instances
```

Pricing:

```text id="8u9zho"
$0.0416/hour
```

Calculation:

```text id="h5azs4"
0.0416 × 4 × 720
= $119.81/month
```

### Monthly Cost

**$119.81**

---

## PostgreSQL Primary Database

Instance Type:

```text id="r8hzkh"
db.r6g.large
2 vCPU
16GB RAM
```

Pricing:

```text id="i5djz5"
$0.182/hour
```

Calculation:

```text id="z1h7f2"
0.182 × 720
= $131.04/month
```

### Monthly Cost

**$131.04**

---

## PostgreSQL Read Replicas

Instance Type:

```text id="8otxj7"
db.r6g.large
```

Quantity:

```text id="8zudgq"
2 replicas
```

Calculation:

```text id="8u6l3s"
0.182 × 2 × 720
= $262.08/month
```

### Monthly Cost

**$262.08**

---

## Redis Cluster (ElastiCache)

Instance Type:

```text id="dwy6w8"
cache.r6g.large
```

Quantity:

```text id="v2e85u"
3 nodes
```

Pricing:

```text id="2jlwm4"
$0.166/hour
```

Calculation:

```text id="2r9zwz"
0.166 × 3 × 720
= $358.56/month
```

### Monthly Cost

**$358.56**

---

## Application Load Balancer

Includes:

* SSL termination
* Health checks
* Traffic routing
* LCU consumption

Calculation:

```text id="9cokq9"
Base Cost = $16.20

Estimated LCU Cost = $40.00

Total = $56.20/month
```

### Monthly Cost

**$56.20**

---

## CloudFront CDN

Assumption:

```text id="k4jlhg"
10 TB transfer/month
```

Pricing:

```text id="chf3ar"
$0.0085/GB
```

Calculation:

```text id="uvr7ux"
10,000 GB × $0.0085

= $85/month
```

### Monthly Cost

**$85.00**

---

## Amazon SQS

Assumption:

```text id="pv40qk"
1 million messages/day
```

Monthly volume:

```text id="2c6w8r"
30 million messages/month
```

Pricing:

```text id="ez72h3"
$0.40 per million requests
```

Calculation:

```text id="gll6du"
30 × $0.40

= $12/month
```

### Monthly Cost

**$12.00**

---

# Baseline Monthly Cost Summary

| Service                   | Monthly Cost        |
| ------------------------- | ------------------- |
| EC2 Application Servers   | $119.81             |
| PostgreSQL Primary        | $131.04             |
| PostgreSQL Replicas       | $262.08             |
| Redis Cluster             | $358.56             |
| Application Load Balancer | $56.20              |
| CloudFront CDN            | $85.00              |
| Amazon SQS                | $12.00              |
| **Total**                 | **$1,024.69/month** |

---

# Scenario 2: World Cup Final Peak Event

Assumption:

* Massive traffic spike
* Auto-scaling enabled
* Extra capacity active for 4 hours

---

## Additional Application Servers

Instance Type:

```text id="twx8ny"
t3.2xlarge
8 vCPU
32GB RAM
```

Quantity:

```text id="o8uq8u"
20 instances
```

Pricing:

```text id="0czswy"
$0.3328/hour
```

Calculation:

```text id="wlg6wc"
0.3328 × 20 × 4

= $26.62
```

### Event Cost

**$26.62**

---

## Temporary Database Upgrade

Instance Type:

```text id="z2h74r"
db.r6g.4xlarge
16 vCPU
128GB RAM
```

Pricing:

```text id="nh2k26"
$1.027/hour
```

Calculation:

```text id="0tjsb3"
1.027 × 4

= $4.11
```

### Event Cost

**$4.11**

---

## CloudFront Traffic Surge

Additional transfer:

```text id="cfr0mx"
50 TB
```

Calculation:

```text id="pkh6v5"
50,000 GB × 0.0085

= $425
```

### Event Cost

**$425.00**

---

# Peak Event Cost Summary

| Service              | Extra Cost  |
| -------------------- | ----------- |
| Additional EC2 Fleet | $26.62      |
| Database Upgrade     | $4.11       |
| CloudFront Surge     | $425.00     |
| **Total**            | **$455.73** |

---

# Business Justification

The redesigned architecture costs approximately:

```text id="ccjv0t"
$1,024/month
```

to operate under normal conditions.

A major World Cup event requires an additional:

```text id="myng7r"
$456
```

for four hours of peak scaling.

The estimated business loss from a platform outage is:

```text id="yikx8v"
₹4.2 crore/minute
```

For a 45-minute outage:

```text id="s4cfzd"
₹4.2 crore × 45

= ₹189 crore
```

Therefore:

* Infrastructure cost ≈ $1,024/month
* Peak event scaling cost ≈ $456/event
* Potential outage loss ≈ ₹189 crore

The infrastructure investment is significantly smaller than the potential financial loss. From both engineering and business perspectives, the redesigned architecture provides an extremely high return on investment and is necessary to support large-scale promotional events safely.

---

# Conclusion

The baseline SwiftEats architecture can be operated for approximately $1,024 per month while supporting normal traffic through caching, load balancing, read replicas, and asynchronous payment processing. During major events, temporary scaling adds only about $456 in additional cost, which is negligible compared to the revenue loss associated with a large-scale outage.
