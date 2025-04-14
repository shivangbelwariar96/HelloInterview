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
- **Metrics**: 10M ads × 1KB/min × 1440 min/day ≈ 14TB/day (raw). Pre-aggregated to ~100K active ads/min × 1KB × 1440 min/day ≈ 140GB/day.  
- **Queries**: ~1K queries/s (advertisers checking metrics).  
**Insight**: Write-heavy, real-time aggregation critical, Prometheus needs cardinality reduction.

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
- **Stream**: Kafka for click ingestion (scalable).  
- **Metrics**: Prometheus for time-series metrics (low-latency), Thanos for long-term storage.  
- **Cache**: Redis for deduplication (fast lookups).  
- **Storage**: S3 for raw clicks (reconciliation).  
**Key Entities**:  
- **Click**: adId, impressionId, timestamp, userId.  
- **Metric**: adId, timeBucket (1-min), clickCount (Prometheus labels: `adId`, `timeBucket`).  
**Indexing**:  
- Kafka: Partitioned by `adId`.  
- Prometheus: Labels `adId`, `timeBucket`; sharded by `timeBucket`.  
- Redis: `impression:{impressionId}` (TTL 24h).  
- S3: `clicks/{date}/{adId}`.

---

## High-Level Design
**Architecture**: Real-time stream processing with batch reconciliation.  
**Components**:  
1. **Ad Placement Service**: Serves ads, generates impressionIds.  
2. **Click Processor**: Logs clicks, redirects users.  
3. **Stream (Kafka)**: Buffers clicks for processing.  
4. **Stream Processor (Kafka Streams)**: Aggregates clicks/min/ad, reduces cardinality.  
5. **Metrics DB (Prometheus)**: Stores metrics for queries, Thanos for retention.  
6. **Cache (Redis)**: Tracks impressionIds for deduplication.  
7. **Data Lake (S3)**: Stores raw clicks for reconciliation.  
8. **Batch Job**: Re-aggregates clicks for accuracy.

**Data Flow**:  
- **Click**: Browser → Click Processor → Redis (dedup) → Kafka → S3.  
- **Aggregation**: Kafka → Kafka Streams → Prometheus.  
- **Query**: Advertiser → Prometheus (or Grafana for visualization).  
- **Reconciliation**: S3 → Batch Job → Prometheus (hourly).  

**Diagram** (verbalize):  
- Browser → Click Processor → Kafka → Kafka Streams → Prometheus/S3.  
- Redis for dedup, S3 → Batch → Prometheus for reconciliation.

**Alternatives**:  
- **Kinesis** instead of Kafka for serverless streaming.  
- **Flink** instead of Kafka Streams for heavy-duty processing.

---

## Deep Dives
Key challenges addressing non-functional requirements.

### 1. Scalability (10K Clicks/s)
**Challenge**: Handle peak click throughput, avoid bottlenecks.  
**Solution**:  
- **Click Processor**:  
  - Stateless, horizontally scaled (~20 EC2 instances, 500 req/s/instance).  
  - Load balancer distributes traffic.  
- **Kafka**:  
  - Partition by `adId`, ~10 partitions (1MB/s or 1K rec/s/partition).  
  - Scales to 10K clicks/s × 100B = 1MB/s.  
- **Kafka Streams**:  
  - Parallel tasks per partition (~10 tasks, 1K clicks/s/task).  
  - Aggregates by `adId`, `timeBucket` (1-min), shards high-cardinality ads (e.g., top 100K ads/min).  
- **Prometheus**:  
  - Shard by `timeBucket`, limit labels to ~100K active `adId`s/min.  
  - Thanos scales to 140GB/day (federated storage).  
- **Hot Shards**:  
  - For popular ads, append random suffix to `adId` (e.g., `adId:0-4`).  
  - Kafka Streams re-aggregates by stripping suffix.  
- **Back-of-Envelope**:  
  - 10K clicks/s ÷ 20 instances = 500 req/s/instance.  
  - 10 partitions × 1K rec/s = 10K clicks/s.  
  - 100K ads × 1KB × 1440 min/day = 140GB/day (Prometheus).  
- **Why?**: Kafka partitions data; Prometheus optimizes metrics; pre-aggregation reduces cardinality.  
- **Alternative**: Single partition (hotspots); no scaling (dropped clicks). Kinesis + Flink could simplify scaling but require AWS integration.  
**Interview Tip**: Sketch partitioning, explain hot shard fix and cardinality reduction. If probed, discuss partition count or Streams parallelism.

### 2. Fault Tolerance (No Data Loss)
**Challenge**: Ensure clicks are never lost despite failures.  
**Solution**:  
- **Kafka**:  
  - Replicates data across brokers, 7-day retention.  
  - If Kafka Streams fails, re-reads from last offset.  
- **S3**:  
  - Raw clicks dumped for redundancy (10GB/day).  
  - Durable (99.999999999% durability).  
- **Kafka Streams**:  
  - Lightweight aggregation (1-min windows).  
  - State stores backed by changelog topics; no heavy checkpointing (small loss tolerable, reprocessed from Kafka).  
- **Prometheus**:  
  - Short-term storage (e.g., 1h); Thanos ensures long-term durability.  
- **Reconciliation**:  
  - Hourly batch job reads S3, re-aggregates clicks.  
  - Updates Prometheus via remote write or re-ingestion.  
- **Back-of-Envelope**:  
  - 100M clicks/day × 100B = 10GB/day (S3).  
  - 100K ads × 1KB × 1440 min/day = 140GB/day (Prometheus/Thanos).  
- **Why?**: Kafka/S3 ensure durability; batch fixes errors; Thanos backs Prometheus.  
- **Alternative**: Heavy checkpointing (overhead); no batch (inaccurate). Kinesis + Flink could offer managed durability but add complexity.  
**Interview Tip**: Highlight Kafka retention, batch job. If probed, discuss trade-off vs. checkpointing or Thanos setup.

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
- **Prometheus**:  
  - Real-time ingestion from Kafka Streams (1-min aggregates).  
  - Labels `adId`, `timeBucket`; sharded by `timeBucket`.  
  - Handles 1K queries/s for ~100K active ads/min.  
- **Pre-Aggregation**:  
  - Kafka Streams pre-aggregates to limit cardinality (e.g., top 100K ads/min).  
  - Store daily/weekly rollups in Thanos (cron job).  
- **Grafana** (optional):  
  - Visualize metrics for advertisers, query Prometheus API directly for programmatic access.  
- **Back-of-Envelope**:  
  - 100K ads × 1KB × 1440 min/day = 140GB/day.  
  - 1K queries/s × 1KB = 1MB/s (Prometheus scales).  
- **Why?**: Prometheus optimizes time-series; pre-aggregation reduces load; Thanos extends storage.  
- **Alternative**: Relational DB (slow joins); no pre-aggregation (query lag).  
**Interview Tip**: Highlight Prometheus, pre-aggregation. If probed, discuss cardinality limits or Grafana’s role.

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
  - Draw: Browser → Click Processor → Kafka → Kafka Streams → Prometheus/S3.  
  - Show: Redis dedup, S3 batch reconciliation, Thanos for Prometheus.  
- **Avoid Pitfalls**:  
  - Don’t skip deduplication (overcounting).  
  - Don’t use relational DB (slow queries).  
  - Don’t ignore hot shards (latency spikes).  
  - Don’t drop raw clicks (no recovery).  
- **Trade-Offs**:  
  - Kafka vs. Kinesis: Control vs. simplicity.  
  - Prometheus vs. Druid: Lightweight vs. high-cardinality.  
  - Real-time vs. batch: Latency vs. accuracy.  
- **Mid-Level Expectations**:  
  - Define flow, batch processing, basic dedup.  
  - Propose Cassandra, pivot to time-series with hints.  
  - Miss hot shards/reconciliation, scale naively.  
  - Reason through probes; expect guidance.

---

This summary is streamlined for your study, with Kafka and Kafka Streams as primary, Kinesis/Flink as alternatives, and Prometheus (with Thanos) replacing Druid. Send the next system design, and I’ll maintain the format!
