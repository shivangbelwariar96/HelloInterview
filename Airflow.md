<img width="1130" alt="Screenshot 2025-04-13 at 4 48 35 PM" src="https://github.com/user-attachments/assets/659ba9cd-262a-4154-9585-8167a9eec129" />


# System Design: Distributed Job Scheduler Like Airflow

This is a concise, interview-ready summary for designing a distributed job scheduler like Airflow, optimized for a 35-minute interview. It covers all critical points to excel, aligning with your 20+ system design cheatsheet for efficient study. The design is "hard" due to high availability, precise execution (<2s), scalability (10K jobs/s), and at-least-once execution guarantees.

---

## Overview
**Name**: Distributed Job Scheduler (Airflow-like)  
**Purpose**: Schedule and execute jobs (instances of tasks) on immediate, future, or recurring schedules.  
**Scope**: Core features (schedule jobs, monitor status). Excludes cancellation, rescheduling, security.

---

## Functional Requirements
- **Schedule Jobs**: Execute jobs immediately, at a future date, or recurring (e.g., CRON).  
- **Monitor Status**: Users track job execution status.

---

## Non-Functional Requirements
- **Availability**: Highly available, prioritizes availability over consistency.  
- **Precision**: Execute jobs within 2s of scheduled time.  
- **Scalability**: Handle 10K jobs/s.  
- **Reliability**: At-least-once execution.  
- **Durability**: Jobs persist across failures.

---

## Scale Estimations
- **Jobs**: 10K jobs/s ≈ 864M jobs/day.  
- **Storage**: 10K jobs/s × 1KB/job ≈ 10MB/s, ~864GB/day.  
- **Queries**: 10K status queries/s (subset of users monitoring).  
**Insight**: High write throughput, time-based queries, fault tolerance critical.

---

## API Design
RESTful APIs for job scheduling and monitoring.

1. **Schedule Job**:  
   - `POST /jobs`  
   - Header: `Authorization: Bearer <JWT>`  
   - Body: `{ taskId: string, schedule: { type: "CRON" | "DATE", expression: string }, parameters: object }`  
   - Returns: `{ jobId: string }`  
   - Purpose: Create job with schedule and parameters.  

2. **Monitor Status**:  
   - `GET /jobs?userId={userId}&status={status}&startTime={startTime}&endTime={endTime}`  
   - Header: `Authorization: Bearer <JWT>`  
   - Returns: `[{ jobId: string, executionTime: long, status: string, attempt: int }]`  
   - Purpose: Query job execution status.

---

## Database
**Choices**:  
- **Jobs/Executions**: DynamoDB for scalability, time-based partitioning.  
- **Queue**: SQS for precise scheduling.  
- **Logs**: S3 for archived executions (post-retention).  
**Key Entities**:  
- **Job**: jobId, userId, taskId, schedule { type, expression }, parameters.  
- **Execution**: timeBucket, executionTime-jobId, jobId, userId, status, attempt.  
**Indexing**:  
- DynamoDB:  
  - Jobs: Partition `jobId`.  
  - Executions: Partition `timeBucket` (hourly), sort `executionTime-jobId`.  
  - GSI: Partition `userId`, sort `executionTime` (status queries).  
- SQS: Delayed messages by `executionTime`.  
- S3: `executions/{year}/{month}/{jobId}`.

---

## High-Level Design
**Architecture**: Microservices with two-layered scheduling (DB + queue).  
**Components**:  
1. **Client**: App/CLI for scheduling/monitoring.  
2. **API Gateway**: Routes requests, handles auth.  
3. **Job Service**: Persists jobs, manages schedules.  
4. **Watcher**: Queries upcoming executions, enqueues to SQS.  
5. **Worker**: Executes jobs from SQS, updates status.  
6. **Database**: DynamoDB for jobs/executions.  
7. **Queue**: SQS for precise scheduling.  
8. **Storage**: S3 for logs.

**Data Flow**:  
- **Schedule**: Client → API Gateway → Job Service → DynamoDB (job + execution).  
- **Immediate (<5min)**: Job Service → SQS (bypass Watcher).  
- **Execute**: Watcher → DynamoDB → SQS → Worker → DynamoDB (status).  
- **Monitor**: Client → API Gateway → Job Service → DynamoDB (GSI).  

**Diagram** (verbalize):  
- Client → API Gateway → Job Service → DynamoDB/SQS.  
- Watcher → DynamoDB → SQS → Worker → DynamoDB.  
- Monitor: Client ← Job Service (GSI).

---

## Deep Dives
Key challenges addressing non-functional requirements.

### 1. Precise Execution (<2s)
**Challenge**: Execute jobs within 2s of scheduled time.  
**Solution**:  
- **Two-Layered Scheduling**:  
  - **Layer 1 (DynamoDB)**: Watcher queries `Executions` every 5min for jobs in next 5min.  
  - Partition: `timeBucket` (hourly, `(executionTime // 3600) * 3600`).  
  - Sort: `executionTime-jobId` avoids collisions.  
  - **Layer 2 (SQS)**: Watcher enqueues jobs with delay (`executionTime - now`).  
  - Workers poll SQS, execute at precise time.  
- **Immediate Jobs**:  
  - Jobs with `executionTime < 5min` go directly to SQS with delay.  
  - Persisted to DynamoDB in parallel.  
- **Back-of-Envelope**:  
  - 10K jobs/s × 300s (5min) = 3M jobs/window.  
  - 3M × 200B = 600MB (SQS handles).  
  - Query: 2 partitions/hour × 1MB = 2MB/5min (DynamoDB scales).  
- **Why?**: DynamoDB reduces query load; SQS ensures precision; delay handles immediate jobs.  
- **Alternative**: Redis ZSET (custom queue, complex); frequent DB polls (high load).  
**Interview Tip**: Sketch DB → SQS flow, explain delay. If probed, discuss overlap or deduplication.

### 2. Scalability (10K Jobs/s)
**Challenge**: Handle 10K jobs/s creation and execution.  
**Solution**:  
- **Job Service**:  
  - Stateless, horizontally scaled (~50 servers for 10K req/s).  
  - Load balancer distributes writes.  
- **DynamoDB**:  
  - Jobs: Partition `jobId` (uniform).  
  - Executions: Partition `timeBucket` (distributes writes).  
  - GSI: `userId` for status queries (~1K queries/s).  
  - Throughput: 10K writes/s × 1KB = 10MB/s (DynamoDB scales).  
- **SQS**:  
  - Auto-scales to 10K msg/s, ~600MB/5min.  
  - Multiple queues for priority if needed.  
- **Workers**:  
  - ECS containers (~100 nodes, 100 jobs/s/node).  
  - Auto-scale on queue depth, spot instances for cost.  
- **Archival**:  
  - Executions → S3 after 1 year (864GB/day × 365 = 315TB).  
- **Back-of-Envelope**:  
  - 10K jobs/s ÷ 50 servers = 200 req/s/server.  
  - 3M jobs × 1KB = 3GB/5min (DynamoDB/SQS).  
- **Why?**: DynamoDB/SQS scale writes; ECS optimizes compute; S3 reduces costs.  
- **Alternative**: Kafka (complex partitioning); Lambda (cold starts).  
**Interview Tip**: Highlight DynamoDB partitioning, SQS. If probed, discuss worker scaling or hot buckets.

### 3. At-Least-Once Execution
**Challenge**: Guarantee jobs execute despite failures, ensure idempotency.  
**Solution**:  
- **Visible Failures**:  
  - Workers wrap tasks in try/catch, log errors.  
  - Update `Executions` to `RETRYING`, increment `attempt`.  
  - Requeue to SQS with exponential backoff (e.g., 2^attempt * 5s).  
  - Max 3 retries, then mark `FAILED`.  
- **Invisible Failures**:  
  - SQS visibility timeout (30s): Job reappears if worker crashes.  
  - Workers heartbeat to extend timeout for long jobs.  
  - `Executions` tracks `attempt` to prevent infinite retries.  
- **Idempotency**:  
  - Tasks designed idempotent (e.g., unique `jobId` in task logic).  
  - Deduplicate on `executionTime-jobId` in `Executions`.  
- **Back-of-Envelope**:  
  - 10K jobs/s × 1% fail × 3 retries = 300 retries/s.  
  - 30s timeout × 10K jobs/s = 300K active jobs (SQS scales).  
- **Why?**: SQS timeout ensures retries; DynamoDB tracks state; idempotency prevents duplicates.  
- **Alternative**: Exactly-once (complex); no retries (missed jobs).  
**Interview Tip**: Explain SQS timeout, idempotency. If probed, discuss backoff or deduplication.

---

## Interview Checklist
- **Structure** (35 min):  
  1. Requirements: Functional/non-functional (2 min).  
  2. Estimations: Jobs, storage (2 min).  
  3. API: REST endpoints (1 min).  
  4. Database: DynamoDB/SQS (2 min).  
  5. High-Level: Services, flow (5 min).  
  6. Deep Dives: Precision, scalability, retries (20–23 min).  
- **Whiteboard** (verbalize):  
  - Draw: Client → API Gateway → Job Service → DynamoDB/SQS → Watcher → Worker.  
  - Show: Schedule (DB + SQS), execute (SQS → Worker), monitor (GSI).  
- **Avoid Pitfalls**:  
  - Don’t poll DB frequently (high load).  
  - Don’t skip retries (missed jobs).  
  - Don’t use single table (hot partitions).  
  - Don’t ignore idempotency (duplicates).  
- **Trade-Offs**:  
  - SQS vs. Redis: Simplicity vs. control.  
  - DynamoDB vs. Cassandra: Managed vs. open-source.  
  - ECS vs. Lambda: Cost vs. elasticity.  
- **Mid-Level Expectations**:  
  - Define APIs, basic job/execution tables, naive scheduler.  
  - Propose DB polling, pivot to queue with hints.  
  - Miss retries/idempotency, address scaling naively.  
  - Reason through probes; expect guidance.

---

This summary is streamlined for your study. Send the next system design, and I’ll maintain the format!
