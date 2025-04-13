<img width="1122" alt="Screenshot 2025-04-13 at 4 49 52 PM" src="https://github.com/user-attachments/assets/482768c2-2abb-4f92-a065-8091f063aba1" />



# System Design: Ad Click Aggregator

This is a concise, interview-ready summary for designing an ad click aggregator, optimized for a 35-minute interview. It covers all critical points to excel, aligning with your 20+ system design cheatsheet for efficient study. The design is "hard" due to scalability (10K clicks/s), low-latency queries (<1s), fault tolerance, real-time processing, and idempotency.

---

## Overview
**Name**: Ad Click Aggregator  
**Purpose**: Track ad clicks and provide aggregated metrics for advertisers.  
**Scope**: Core features (track clicks, redirect users, query metrics at 1-min granularity). Excludes ad targeting, serving, fraud detection, conversion tracking.

---

## Functional Requirements
- **Track Clicks**: Users click ads, redirect to advertiser’s site.  
- **Query Metrics**: Advertisers query click metrics (1-min intervals).

---

## Non-Functional Requirements
- **Scalability**: Handle 10K clicks/s peak, 100M clicks/day.  
- **Low Latency**: Sub-second analytics queries.  
- **Fault Tolerance**: No click data loss.  
- **Real-Time**: Metrics available ASAP after clicks.  
- **Idempotency**: Prevent duplicate click counting.

---

## Scale Estimations
- **Clicks**: 10K clicks/s peak, 100M/day ≈ 1.2K clicks/s avg.  
- **Data**: 10K clicks/s × 100B/click ≈ 1MB/s, 100M × 100B ≈ 10GB/day.  
- **Metrics**: 10M ads × 1KB/min × 1440 min/day ≈ 14TB/day (pre-aggregated).  
- **Queries**: ~1K queries/s (advertisers checking metrics).  
**Insight**: Write-heavy, real-time aggregation critical, storage grows with ads.

---

## System Interface
- **Input**: Ad click data (adId, timestamp, user info).  
- **Output**: Aggregated metrics (clicks/ad/min).

---

## Data Flow
1. User clicks ad, browser sends click to system.  
2. System logs click, redirects user to advertiser’s site.  
3. Click data is streamed for aggregation.  
4. Aggregated metrics stored for querying.  
5. Advertisers query metrics by adId, time range.

---

## Database
**Choices**:  
- **Stream**: Kinesis for click ingestion (scalable).  
- **Metrics**: Druid for OLAP queries (low-latency, time-series).  
- **Cache**: Redis for deduplication (fast lookups).  
- **Storage**: S3 for raw clicks (reconciliation).  
**Key Entities**:  
- **Click**: adId, impressionId, timestamp, userId.  
- **Metric**: adId, timeBucket (1-min), clickCount.  
**Indexing**:  
- Kinesis: Sharded by `adId`.  
- Druid: Partition `timeBucket`, index `adId`.  
- Redis: `impression:{impressionId}` (TTL 24h).  
- S3: `clicks/{date}/{adId}`.

---

## High-Level Design
**Architecture**: Real-time stream processing with batch reconciliation.  
**Components**:  
1. **Ad Placement Service**: Serves ads, generates impressionIds.  
2. **Click Processor**: Logs clicks, redirects users.  
3. **Stream (Kinesis)**: Buffers clicks for processing.  
4. **Stream Processor (Flink)**: Aggregates clicks/min/ad.  
5. **OLAP DB (Druid)**: Stores metrics for queries.  
6. **Cache (Redis)**: Tracks impressionIds for deduplication.  
7. **Data Lake (S3)**: Stores raw clicks for reconciliation.  
8. **Batch Job**: Re-aggregates clicks for accuracy.

**Data Flow**:  
- **Click**: Browser → Click Processor → Redis (dedup) → Kinesis → S3.  
- **Aggregation**: Kinesis → Flink → Druid.  
- **Query**: Advertiser → Druid.  
- **Reconciliation**: S3 → Batch Job → Druid (hourly).  

**Diagram** (verbalize):  
- Browser → Click Processor → Kinesis → Flink → Druid/S3.  
- Redis for dedup, S3 → Batch → Druid for reconciliation.

---

## Deep Dives
Key challenges addressing non-functional requirements.

### 1. Scalability (10K Clicks/s)
**Challenge**: Handle peak click throughput, avoid bottlenecks.  
**Solution**:  
- **Click Processor**:  
  - Stateless, horizontally scaled (~20 EC2 instances, 500 req/s/instance).  
  - Load balancer distributes traffic.  
- **Kinesis**:  
  - Shard by `adId`, ~10 shards (1MB/s or 1K rec/s/shard).  
  - Scales to 10K clicks/s × 100B = 1MB/s.  
- **Flink**:  
  - Parallel jobs per shard (~10 tasks, 1K clicks/s/task).  
  - Aggregates by `adId`, `timeBucket` (1-min).  
- **Druid**:  
  - Partition by `timeBucket`, index `adId`.  
  - Scales to 10M ads × 1440 min/day = 14B records/day.  
- **Hot Shards**:  
  - For popular ads, append random suffix to `adId` (e.g., `adId:0-4`).  
  - Flink re-aggregates by stripping suffix.  
- **Back-of-Envelope**:  
  - 10K clicks/s ÷ 20 instances = 500 req/s/instance.  
  - 10 shards × 1K rec/s = 10K clicks/s.  
  - 14B records × 1KB = 14TB/day (Druid scales).  
- **Why?**: Kinesis/Flink shard data; Druid optimizes queries; randomization mitigates hot shards.  
- **Alternative**: Single shard (hotspots); no scaling (dropped clicks).  
**Interview Tip**: Sketch sharding, explain hot shard fix. If probed, discuss shard count or Flink parallelism.

### 2. Fault Tolerance (No Data Loss)
**Challenge**: Ensure clicks are never lost despite failures.  
**Solution**:  
- **Kinesis**:  
  - Replicates data across AZs, 7-day retention.  
  - If Flink fails, re-reads from last offset.  
- **S3**:  
  - Raw clicks dumped for redundancy (10GB/day).  
  - Durable (99.999999999% durability).  
- **Flink**:  
  - Lightweight aggregation (1-min windows).  
  - No checkpointing (small loss tolerable, reprocessed from Kinesis).  
- **Reconciliation**:  
  - Hourly batch job reads S3, re-aggregates clicks.  
  - Compares with Druid, updates discrepancies.  
- **Back-of-Envelope**:  
  - 100M clicks/day × 100B = 10GB/day (S3).  
  - 10M ads × 1-min × 1KB = 14TB/day (Druid).  
- **Why?**: Kinesis/S3 ensure durability; batch fixes errors; no checkpointing simplifies Flink.  
- **Alternative**: Checkpointing (overhead); no batch (inaccurate).  
**Interview Tip**: Highlight Kinesis retention, batch job. If probed, discuss trade-off vs. checkpointing.

### 3. Idempotency (Prevent Duplicate Clicks)
**Challenge**: Avoid counting repeated clicks from same impression.  
**Solution**:  
- **Impression ID**:  
  - Ad Placement Service generates unique `impressionId` per ad view.  
  - Browser sends `impressionId` with click.  
- **Redis Deduplication**:  
  - Click Processor checks `impression:{impressionId}` (TTL 24h).  
  - If exists, skip; else set and process.  
- **Signed Data**:  
  - Impression data `{adId, impressionId}` signed by Ad Placement Service.  
  - Click Processor verifies signature to prevent tampering.  
- **Back-of-Envelope**:  
  - 10K clicks/s × 24h × 100B = 86GB/day (Redis, manageable).  
  - 10K checks/s (Redis scales).  
- **Why?**: Redis for fast dedup; signing prevents abuse; TTL limits storage.  
- **Alternative**: DB dedup (slower); no signing (vulnerable).  
**Interview Tip**: Explain Redis flow, signature. If probed, discuss TTL or false positives.

### 4. Low-Latency Queries (<1s)
**Challenge**: Sub-second metrics queries for advertisers.  
**Solution**:  
- **Druid**:  
  - Real-time ingestion from Flink (1-min aggregates).  
  - Partition by `timeBucket`, index `adId` for fast lookups.  
  - Handles 1K queries/s, ~10M ads.  
- **Pre-Aggregation**:  
  - Store daily/weekly aggregates in Druid (cron job).  
  - Query higher granularity first, drill down if needed.  
- **Back-of-Envelope**:  
  - 10M ads × 1-min × 1KB = 14TB/day.  
  - 1K queries/s × 1KB = 1MB/s (Druid scales).  
- **Why?**: Druid optimizes time-series; pre-aggregation speeds up large windows.  
- **Alternative**: Relational DB (slow joins); no pre-aggregation (query lag).  
**Interview Tip**: Highlight Druid, pre-aggregation. If probed, discuss query patterns or caching.

---

## Interview Checklist
- **Structure** (35 min):  
  1. Requirements: Functional/non-functional (2 min).  
  2. Estimations: Clicks, storage (2 min).  
  3. Interface: Input/output (1 min).  
  4. Data Flow: Click to metrics (2 min).  
  5. High-Level: Stream, DB, batch (5 min).  
  6. Deep Dives: Scalability, fault tolerance, idempotency, latency (20–23 min).  
- **Whiteboard** (verbalize):  
  - Draw: Browser → Click Processor → Kinesis → Flink → Druid/S3.  
  - Show: Redis dedup, S3 batch reconciliation.  
- **Avoid Pitfalls**:  
  - Don’t skip deduplication (overcounting).  
  - Don’t use relational DB (slow queries).  
  - Don’t ignore hot shards (latency spikes).  
  - Don’t drop raw clicks (no recovery).  
- **Trade-Offs**:  
  - Kinesis vs. Kafka: Simplicity vs. control.  
  - Druid vs. TimescaleDB: Query speed vs. cardinality.  
  - Real-time vs. batch: Latency vs. accuracy.  
- **Mid-Level Expectations**:  
  - Define flow, batch processing, basic dedup.  
  - Propose Cassandra, pivot to OLAP with hints.  
  - Miss hot shards/reconciliation, scale naively.  
  - Reason through probes; expect guidance.

---

This summary is streamlined for your study. Send the next system design, and I’ll maintain the format!
