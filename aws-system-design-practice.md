# 🏗 AWS System Design Practice

------------------------------------------------------------------------

## 🎬 Phase 1: Initial Architecture (Intentionally Imperfect)

    Users
      |
    Route53
      |
    ALB
      |
    EC2 (Auto Scaling Group - 2 instances)
      |
    RDS (Single AZ)
      |
    S3 (public-read)

    Authentication:
    API Gateway → EC2 → RDS

### Configurations:

-   SSH open to 0.0.0.0/0
-   IAM role with full S3 + RDS access
-   Auto scaling based only on CPU \> 70%
-   No WAF
-   No backup strategy mentioned
-   S3 public-read
-   Only basic CloudWatch CPU monitoring

------------------------------------------------------------------------

# 🔍 Phase 2: Availability Improvements

### ❗ RDS Single AZ

-   Risk: AZ failure = full outage
-   Improvement:
    -   RDS Multi-AZ
    -   Or Aurora (better HA, slightly higher cost)

### ❗ Auto Scaling Logic Weakness

Scaling only on CPU is dangerous.

Problems: - DB exhaustion won't increase CPU - Memory pressure ignored -
Latency ignored

Better Metrics: - ALB target response time - Request count per target -
Memory utilization - DB connections

### ❗ Possible Single-AZ ASG

If subnets are single AZ → full outage risk.

------------------------------------------------------------------------

# 🔐 Phase 3: Security Issues Identified

### ❌ SSH Open to 0.0.0.0/0

-   Full infrastructure compromise risk
-   Fix:
    -   Remove SSH
    -   Use SSM Session Manager

### ❌ Over-Permissive IAM

-   Full S3 + RDS access
-   Violates least privilege
-   Fix:
    -   Scoped IAM roles
    -   Fine-grained policies

### ❌ Public S3 Bucket

-   Data scraping risk
-   Fix:
    -   Private bucket
    -   CloudFront + Origin Access Control

### ❌ No WAF

-   ALB exposed to Layer 7 attacks
-   Fix:
    -   AWS WAF

### ❌ No File Scanning

-   Risk: malicious uploads
-   Fix:
    -   S3 trigger → Lambda scan
    -   Optional Macie

------------------------------------------------------------------------

# 🚀 Phase 4: Serverless Discussion

## Can We Remove EC2?

Yes.

Possible Architecture:

    Route53
       |
    CloudFront
       |
    S3 (private)
       |
    API Gateway
       |
    Lambda
       |
    Aurora Serverless / RDS

------------------------------------------------------------------------

# ⚡ Phase 5: Lambda + RDS Risk

### ⚠ Concurrency Shock

Scenario: - Traffic spike - Lambda scales instantly - Hundreds of DB
connections opened - RDS crashes or throttles

Fix Options: - RDS Proxy - Reserved concurrency limit - Aurora
Serverless v2 - Add caching (Redis)

------------------------------------------------------------------------

# 🗄 Phase 6: Moving to DynamoDB

## Why DynamoDB?

-   No connection limits
-   Auto scaling
-   Multi-AZ by default
-   Pay per request
-   Minimal ops

## 🧠 DynamoDB Modeling Strategy

### Single Table Design

Instead of: - Users table - Orders table

Use:

    PK = USER#<userId>
    SK = PROFILE

    PK = USER#<userId>
    SK = ORDER#<orderId>

Benefits: - Fetch profile + orders in one query - No joins required -
Scales cleanly

------------------------------------------------------------------------

# ⚠ DynamoDB Tradeoffs

-   No joins
-   Must design around access patterns
-   Aggregations not natural
-   Eventual consistency (default)

------------------------------------------------------------------------

# 📊 Phase 7: Analytics Requirement Appears

New Requirements: - Orders placed today - Top 10 users by revenue -
Monthly analytics

Problem: DynamoDB ≠ OLAP engine

Proper Solution:

    DynamoDB
       |
    Streams
       |
    Lambda
       |
    S3
       |
    Athena
       |
    Grafana / QuickSight

Key Principle: - DynamoDB → OLTP - Athena/Redshift → OLAP

------------------------------------------------------------------------

# 💸 Phase 8: DynamoDB Cost Explosion Risks

## ❌ Using Scan

-   Reads full table
-   Massive RCU usage

## ❌ Filtering After Read

-   FilterExpression still consumes full read capacity
-   Must use proper keys and GSIs

## ❌ Hot Partitions

Bad Design:

    PK = DATE#2026-02-11

All traffic hits one partition → throttling + cost increase.

Fix: - Distribute partition keys evenly

------------------------------------------------------------------------

# 🛡 Phase 9: Backpressure Strategy

Problem: Lambda scales faster than downstream services.

Solution: Introduce SQS.

    API Gateway
       |
    Lambda (Producer)
       |
    SQS
       |
    Lambda (Consumer, controlled concurrency)
       |
    DynamoDB

Benefits: - Traffic smoothing - Retry logic - DLQ support - Protects
database

------------------------------------------------------------------------

# ⚡ DAX Clarification

DAX helps with: - Read-heavy workloads - Low-latency reads - Reducing
RCUs

DAX does NOT: - Smooth write spikes - Provide backpressure - Protect
from concurrency storms

------------------------------------------------------------------------

# 🏛 Final Architecture Maturity Ladder

1.  EC2 + RDS
2.  Multi-AZ + ASG
3.  Containers (ECS/Fargate)
4.  Serverless + RDS
5.  Fully Serverless + DynamoDB
6.  Add OLAP layer
7.  Add Queue for resilience

------------------------------------------------------------------------

# 🧠 Core Lessons Learned

-   Design for failure
-   Protect slow systems from fast ones
-   Separate OLTP and OLAP
-   Model DynamoDB around access patterns
-   Avoid Scan
-   Use queues for traffic spikes
-   Apply least privilege IAM
-   Security and cost are design decisions
