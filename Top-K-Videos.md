# System Design: YouTube Top K Videos Feature

This is a concise, interview-ready summary for designing YouTube’s Top K Videos feature, tailored for a 35-minute interview. It addresses all critical points to excel, omitting extraneous details while aligning with your 20+ system design cheatsheet for efficient study. The design is "hard" due to massive scale (70B views/day) and precise counting requirements, avoiding probabilistic solutions like count-min sketch.

---

## Overview
**Name**: Top K Videos Service (YouTube-like)  
**Purpose**: Track video views and query the top K most-viewed videos for fixed time windows.  
**Scope**: Core features (query top K videos for 1 hour, 1 day, 1 month, all-time). Excludes arbitrary time periods.

---

## Functional Requirements
- **Query Top K**: Clients query top K videos (K ≤ 1000) with view counts.  
- **Time Windows**: Support 1 hour, 1 day, 1 month, all-time windows.

---

## Non-Functional Requirements
- **Latency**: Results return in tens of milliseconds (<50ms).  
- **Freshness**: Views tabulated within 1 minute.  
- **Precision**: Exact counts, no approximations.  
- **Scalability**: Handle ~700K views/s, ~3.6B videos.  
- **Cost**: Avoid excessive infrastructure (e.g., <10K hosts).  
- **Reliability**: System remains available despite failures.

---

## Scale Estimations
- **Views**: 70B views/day ÷ 86K s/day ≈ 700K views/s.  
- **Videos**: 1M videos/day × 365 days × 10 years ≈ 3.6B videos.  
- **Storage**: 3.6B videos × (8B videoId + 8B count) ≈ 64GB/window.  
- **Queries**: ~10K QPS (assumed, bursty).  
**Insight**: High write throughput, moderate storage, low-latency reads need precomputation.

---

## API Design
RESTful API with JWT for authentication.

1. **Get Top K Videos**:  
   - `GET /views/top?window={hour|day|month|all}&k={int}`  
   - Returns: `{ videos: [{ videoId: string, views: long }] }`  
   - Purpose: Fetch top K videos with exact view counts.

---

## Database
**Choices**:  
- **Data**: Redis for counters/heaps (low-latency).  
- **Queue**: Kafka for view stream (durable).  
- **Backup**: S3 for snapshots (durability).  
**Key Entities**:  
- **View**: videoId, timestamp.  
- **Counter**: videoId, count (per window).  
**Indexing**:  
- Redis:  
  - `counts:{window}` (hash, videoId → count).  
  - `heap:{window}` (sorted set, videoId, score = count).  
- Kafka: Topic `views`, partitioned by `videoId`.  
- S3: `snapshots/{window}/{date}` for periodic dumps.

---

## High-Level Design
**Architecture**: Sharded counters with in-memory heaps, precomputed for latency.  
**Components**:  
1. **Client**: Apps querying top K videos.  
2. **API Gateway**: Routes requests, handles auth.  
3. **Top-K Service**: Queries heaps, merges results.  
4. **Counter Service**: Increments counts, updates heaps.  
5. **Cache**: Redis for counters/heaps.  
6. **Queue**: Kafka for view ingestion.  
7. **Storage**: S3 for snapshots.

**Data Flow**:  
- **View**: Client → Kafka → Counter Service → Redis (count/heap).  
- **Query**: Client → API Gateway → Top-K Service → Redis → Client.

**Diagram** (verbalize):  
- Client → API Gateway → Top-K Service → Redis.  
- Views → Kafka → Counter Service → Redis → S3.

---

## Deep Dives
Key challenges addressing non-functional requirements.

### 1. Handling Time Windows
**Challenge**: Maintain exact counts for sliding windows (hour, day, month).  
**Solution**:  
- **Two-Pointer Approach**:  
  - Kafka topic `views` stores `(videoId, timestamp)`.  
  - Rising edge: Consumer reads head, increments `counts:{window}` and `heap:{window}`.  
  - Falling edge: Consumer reads offset (e.g., now - 1h), decrements counts/heaps for expired views.  
  - Windows: Separate consumers for hour, day, month; all-time has no falling edge.  
- **Redis**:  
  - `counts:{window}` (hash, O(1) incr/decr).  
  - `heap:{window}` (sorted set, O(log K) update, K ≤ 1000).  
- **Example**:  
  - At T=0:05, view A → incr A in all windows.  
  - At T=1:05, hour consumer at T=0:05 decrements A (hour window expires).  
  - Day/all-time undisturbed until T=24:05.  
- **Back-of-Envelope**:  
  - 700K views/s × 3 windows = 2.1M incr/s (Redis handles).  
  - Falling edge: ~700K decr/s for hour window.  
- **Why?**: Kafka preserves stream; two pointers maintain exact counts; Redis is fast.  
- **Alternative**: Micro-buckets (stale data); TSDB (complex for precise top K).  
**Interview Tip**: Propose two pointers, walk through example. If probed, discuss consumer lag or heap updates.

### 2. Scaling Write Throughput
**Challenge**: 700K views/s overwhelms single host.  
**Solution**:  
- **Sharding**:  
  - Partition Kafka `views` by `videoId` (~10K partitions).  
  - Counter Service sharded by `videoId` modulo (~100 servers).  
  - Each shard owns full counts/heap for its video subset.  
- **Elastic Scaling**:  
  - Zookeeper tracks shard-to-server mapping.  
  - Rebalance shards on server failure/addition.  
- **Replication**:  
  - Redis replicas (~3 per shard) for fault tolerance.  
  - Periodic S3 snapshots for recovery.  
- **Back-of-Envelope**:  
  - 700K views/s ÷ 100 servers = 7K views/s/server.  
  - 64GB ÷ 100 shards = 640MB/shard (in-memory).  
- **Why?**: Sharding distributes load; Zookeeper enables elasticity; replicas ensure reliability.  
- **Alternative**: Fixed partitioning (hot shards); single host (SPOF).  
**Interview Tip**: Lead with sharding, mention Zookeeper. If probed, discuss hot videos or snapshot recovery.

### 3. Low-Latency Queries
**Challenge**: Return top K in <50ms for bursty QPS.  
**Solution**:  
- **Precomputation**:  
  - Maintain `heap:{window}` in Redis per shard.  
  - Top-K Service merges top K from each shard’s heap (O(N log K), N = shards).  
- **Caching**:  
  - Cache query results in Redis (`topk:{window}:{k}`, TTL 1m).  
  - High hit ratio for popular K values (e.g., K=10,100).  
- **Optimization**:  
  - Limit shards queried (e.g., top 10 shards hold most views).  
  - Parallelize shard queries (~1ms/shard).  
- **Back-of-Envelope**:  
  - 100 shards × 1ms + 10ms merge = ~110ms (cache reduces to <10ms).  
  - 10K QPS × 1KB/response = 10MB/s bandwidth.  
- **Why?**: Precomputed heaps minimize computation; cache eliminates shard queries; parallelization scales.  
- **Alternative**: Compute on read (slow); no cache (shard overload).  
**Interview Tip**: Emphasize cache and precomputation, estimate latency. If probed, discuss cache eviction (LRU) or shard skew.

---

## Interview Checklist
- **Structure** (35 min):  
  1. Requirements: Functional/non-functional (2 min).  
  2. Estimations: TPS, storage (2 min).  
  3. API: Endpoint (1 min).  
  4. Database: Redis/Kafka (2 min).  
  5. High-Level: Sharded counters (5 min).  
  6. Deep Dives: Windows, writes, queries (18–23 min).  
- **Whiteboard** (verbalize):  
  - Draw: Client → API Gateway → Top-K → Redis; Views → Kafka → Counter → Redis/S3.  
  - Show: View (Kafka → Redis count/heap), query (Redis → merge).  
- **Avoid Pitfalls**:  
  - Don’t approximate counts (violates precision).  
  - Don’t use single host (SPOF, unscalable).  
  - Don’t scan all videos (slow).  
  - Don’t ignore falling edge (wrong counts).  
- **Trade-Offs**:  
  - Redis vs. TSDB: Speed vs. native windows.  
  - Sharding vs. single host: Scalability vs. simplicity.  
  - Cache vs. compute: Latency vs. freshness.  
- **Mid-Level Expectations**:  
  - Define API, basic counter/heap on single host.  
  - Address time windows naively (e.g., buckets).  
  - Suggest sharding with hints.  
  - Reason through probes; expect guidance.

---

This summary is lean for your preparation. Send the next system design, and I’ll maintain the format!
