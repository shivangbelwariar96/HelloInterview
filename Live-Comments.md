# System Design: Facebook Live Comments System

This is a concise, interview-ready summary for designing a Facebook Live Comments system, optimized for a 35-minute interview. It addresses all critical points to excel, omitting unnecessary details (e.g., full DB schemas) while aligning with your 20+ system design cheatsheet for efficient study.

---

## Overview
**Name**: Live Comments Service (Facebook Live-like)  
**Purpose**: Enable viewers to post and view comments on live video feeds in near real-time.  
**Scope**: Core features (post comments, view new comments, view past comments). Excludes replies, reactions.

---

## Functional Requirements
- **Post Comments**: Viewers can post comments on a live video feed.  
- **View New Comments**: Viewers see comments posted in real-time while watching.  
- **View Past Comments**: Viewers see comments made before joining the feed.

---

## Non-Functional Requirements
- **Scalability**: Support millions of concurrent videos, thousands of comments/s/video.  
- **Availability**: Prioritize availability over consistency (eventual consistency acceptable).  
- **Latency**: <200ms end-to-end for comment broadcasting under typical conditions.  
- **Durability**: Comments persisted reliably.

---

## API Design
RESTful APIs with JWT for authentication; SSE for real-time updates.

1. **Post Comment**:  
   - `POST /comments/{liveVideoId}`  
   - Header: `Authorization: Bearer <JWT>`  
   - Body: `{ message: string }`  
   - Returns: `{ commentId: string }`  
   - Purpose: Create a comment.  

2. **Fetch Comments**:  
   - `GET /comments/{liveVideoId}?cursor={commentId}&limit={int}&sort=desc`  
   - Returns: `{ comments: [{ id, userId, message, timestamp }], nextCursor: string }`  
   - Purpose: Retrieve paginated historical comments.  

3. **Real-Time Comments**:  
   - `SSE /comments/{liveVideoId}/stream`  
   - Returns: Stream `{ commentId: string, userId: string, message: string, timestamp: int }`  
   - Purpose: Stream new comments to clients.

---

## Database
**Choices**:  
- **Data**: DynamoDB for comments (scalable, low-latency writes).  
- **Cache**: Redis for real-time comment streams.  
- **Queue**: Kafka for durable comment ingestion (optional for spikes).  
**Key Entities**:  
- **Comment**: commentId, liveVideoId, userId, message, timestamp.  
- **Live Video**: liveVideoId (managed externally, referenced).  
**Indexing**:  
- DynamoDB:  
  - PK `liveVideoId`, SK `commentId` (auto-incrementing).  
  - GSI for `timestamp` queries.  
- Redis: Channel `video:{liveVideoId}:comments` for pub/sub.  
- Kafka: Topic `comments` partitioned by `liveVideoId`.

---

## High-Level Design
**Architecture**: Microservices with push-based real-time delivery.  
**Components**:  
1. **Client**: Web/mobile app for posting/viewing comments.  
2. **API Gateway**: Routes requests, manages SSE, handles auth.  
3. **Comment Service**: Manages comment CRUD, persistence.  
4. **Real-Time Service**: Broadcasts comments via SSE.  
5. **Database**: DynamoDB for comments.  
6. **Cache**: Redis for pub/sub.  
7. **Queue**: Kafka for write buffering (optional).  

**Data Flow**:  
- **Post Comment**: Client → API Gateway → Comment Service → DynamoDB/Kafka → Redis pub/sub → Real-Time Service → SSE to clients.  
- **View New Comments**: Client → API Gateway → Real-Time Service (SSE) → Redis pub/sub.  
- **View Past Comments**: Client → API Gateway → Comment Service → DynamoDB → Client.

**Diagram** (verbalize):  
- Client → API Gateway → Comment/Real-Time Service.  
- Comment → DynamoDB/Kafka → Redis → Real-Time → SSE.

---

## Deep Dives
Key challenges addressing non-functional requirements.

### 1. Real-Time Comment Broadcasting
**Challenge**: Broadcast comments to viewers in <200ms.  
**Solution**:  
- **Server-Sent Events (SSE)**:  
  - Clients connect to `/comments/{liveVideoId}/stream`.  
  - Real-Time Service subscribes to Redis `video:{liveVideoId}:comments`.  
  - Comment Service publishes new comments to Redis after DynamoDB write.  
- **Client Handling**:  
  - Buffer comments locally, render smoothly (avoid UI jerks).  
  - Reconnect on drops, fetch missed comments via `GET`.  
- **Why?**: SSE is lightweight, unidirectional; Redis pub/sub scales for fan-out; 200ms achievable.  
- **Alternative**: WebSocket (bidirectional overhead); polling (high QPS, >200ms).  
**Interview Tip**: Propose SSE, justify latency. If probed, discuss WebSocket or client buffering.

### 2. Scalability for Millions of Viewers
**Challenge**: Handle millions of videos, thousands of comments/s/video (~1B writes/s total).  
**Solution**:  
- **Redis Pub/Sub**:  
  - Channel per `liveVideoId` (~millions of channels).  
  - Cluster mode, sharded by `liveVideoId`.  
- **Real-Time Service**:  
  - Horizontal scaling (~10K servers, 100K connections/server).  
  - Co-locate viewers of same video on same server (hash `liveVideoId` to server).  
- **DynamoDB**:  
  - Partition by `liveVideoId` (~10K partitions).  
  - ~1B writes/s (1KB/comment) across shards.  
- **Hot Videos**:  
  - Split popular videos into sub-channels (e.g., `video:{liveVideoId}:{0-9}`).  
  - Cache recent comments in Redis (TTL 10m).  
- **Back-of-Envelope**:  
  - 1M videos × 1K comments/s = 1B writes/s.  
  - 10M viewers/video × 1KB/comment = 10TB/s bandwidth (mitigated by sub-channels).  
- **Why?**: Co-location reduces pub/sub overhead; DynamoDB scales writes; sub-channels handle hotspots.  
- **Alternative**: Kafka pub/sub (higher latency); polling (QPS bottleneck).  
**Interview Tip**: Suggest co-location, estimate QPS. If probed, discuss hot videos or CDN.

### 3. Availability Over Consistency
**Challenge**: Prioritize availability, allow eventual consistency.  
**Solution**:  
- **DynamoDB**:  
  - Eventual consistency for reads (`GET /comments`).  
  - Write to leader, async replicate (~100ms lag tolerable).  
- **Redis**:  
  - Fire-and-forget pub/sub (missed messages OK).  
  - Fallback to DynamoDB on Redis failure.  
- **Client**:  
  - Fetch missed comments on reconnect (`GET` with cursor).  
  - Tolerate duplicates (idempotent rendering).  
- **Why?**: Eventual consistency ensures reads always return; duplicates rare, non-critical.  
- **Alternative**: Strong consistency (lower availability); sync replication (higher latency).  
**Interview Tip**: Emphasize availability, mention client recovery. If probed, discuss PACELC (PA/EL).

### 4. Handling High-Volume Streams
**Challenge**: Popular videos with millions of viewers overwhelm bandwidth.  
**Solution**:  
- **Sampling**:  
  - Show subset of comments to viewers (e.g., 20 comments/s).  
  - Prioritize friends’ comments or sample randomly.  
- **CDN Fallback**:  
  - For >1M viewers, switch to polling CDN-cached snapshots (1s intervals).  
  - Redis stores 100-comment batches (TTL 1m).  
- **Client Tricks**:  
  - Spread comment rendering (e.g., 100 comments over 1s for smooth UI).  
  - Limit in-memory comments (~200).  
- **Why?**: Sampling reduces load; CDN scales reads; client fakes “live” feel.  
- **Alternative**: Full broadcast (bandwidth explosion); rate limiting (poor UX).  
**Interview Tip**: Propose sampling/CDN, mention UI tricks. If probed, discuss trade-offs for viral streams.

---

## Interview Checklist
- **Structure** (35 min):  
  1. Requirements: Functional/non-functional (2 min).  
  2. API: Endpoints, SSE (2 min).  
  3. Database: DynamoDB/Redis (2 min).  
  4. High-Level: Microservices, flow (5 min).  
  5. Deep Dives: Real-time, scale, availability, hot streams (15–20 min).  
- **Whiteboard** (verbalize):  
  - Draw: Client → API Gateway → Comment/Real-Time Service → DynamoDB/Redis.  
  - Show: Post (DynamoDB → Redis), broadcast (SSE), history (DynamoDB).  
- **Avoid Pitfalls**:  
  - Don’t rely on polling (high QPS, >200ms).  
  - Don’t use WebSocket (unnecessary complexity).  
  - Don’t enforce strong consistency (hurts availability).  
  - Don’t ignore hot videos (bandwidth bottleneck).  
- **Trade-Offs**:  
  - SSE vs. WebSocket: Simplicity vs. flexibility.  
  - Eventual vs. strong consistency: Availability vs. accuracy.  
  - Sampling vs. full broadcast: Scalability vs. completeness.  
- **Mid-Level Expectations**:  
  - Define APIs, basic microservices.  
  - Propose DynamoDB, SSE for real-time.  
  - Address scalability naively (e.g., add servers).  
  - Reason through probes; expect guidance.

---

This summary is streamlined for your study. Send the next system design, and I’ll keep the format consistent!
