<img width="1137" alt="Screenshot 2025-04-13 at 4 43 18 PM" src="https://github.com/user-attachments/assets/146d1665-8d49-4e7e-8ae7-d82dfbe20ec5" />


# System Design: Facebook Post Search

This is a concise, interview-ready summary for designing a Facebook Post Search system, tailored for a 35-minute interview. It covers all critical points to excel, omitting extraneous details (e.g., exhaustive schemas) while aligning with your 20+ system design cheatsheet for efficient study. The design assumes no pre-built search engines (e.g., Elasticsearch) are allowed, focusing on fundamental indexing and scaling.

---

## Overview
**Name**: Post Search Service (Facebook-like)  
**Purpose**: Enable users to create, like, and search posts by keyword, with results sorted by recency or like count.  
**Scope**: Core features (create/like posts, keyword search, sort results). Excludes fuzzy matching, personalization, privacy, media, real-time updates.

---

## Functional Requirements
- **Create/Like Posts**: Users can create posts and like others’ posts.  
- **Search Posts**: Users can search posts by keyword.  
- **Sort Results**: Search results can be sorted by recency or like count.

---

## Non-Functional Requirements
- **Performance**: Median search queries return in <500ms.  
- **Scalability**: Handle high request volume (~10K searches/s, 10K posts/s, 100K likes/s).  
- **Indexing Latency**: New posts searchable in <1 minute.  
- **Discoverability**: All posts (old/unpopular) searchable, slower access acceptable.  
- **Availability**: System remains highly available.

---

## Scale Estimations
- **Users**: 1B.  
- **Posts**: 1 post/user/day → 1B posts/day ÷ 100K s/day ≈ 10K posts/s.  
- **Likes**: 10 likes/user/day → 10B likes/day ÷ 100K s ≈ 100K likes/s.  
- **Searches**: 1 search/user/day → 1B searches/day ÷ 100K s ≈ 10K searches/s.  
- **Storage**: 1B posts/day × 365 days × 10 years × 1KB/post ≈ 3.6PB.  
**Insight**: Write-heavy (likes > posts > searches); massive storage needs optimization.

---

## API Design
RESTful APIs with JWT for authentication.

1. **Create Post**:  
   - `POST /posts`  
   - Header: `Authorization: Bearer <JWT>`  
   - Body: `{ content: string }`  
   - Returns: `{ postId: string }`  
   - Purpose: Create a post.  

2. **Like Post**:  
   - `POST /posts/{postId}/likes`  
   - Header: `Authorization: Bearer <JWT>`  
   - Returns: `{ status: "SUCCESS" }`  
   - Purpose: Record a like.  

3. **Search Posts**:  
   - `GET /posts/search?keyword={string}&sort={created|likes}&limit={int}&offset={int}`  
   - Returns: `{ posts: [{ id, content, createdAt, likeCount }], total: int }`  
   - Purpose: Search posts by keyword, sorted by recency or likes.

---

## Database
**Choices**:  
- **Data**: Postgres for posts/likes (durable, relational).  
- **Index**: Redis for inverted indexes (fast lookups).  
- **Queue**: Kafka for write buffering.  
- **Storage**: S3 for cold indexes.  
**Key Entities**:  
- **Post**: postId, userId, content, createdAt, likeCount.  
- **Like**: postId, userId, timestamp.  
**Indexing**:  
- Postgres:  
  - PK `postId` (Post), `postId,userId` (Like).  
  - Index `createdAt` (Post), `postId` (Like).  
- Redis:  
  - `keyword:{term}:created` (list of postIds, ordered by createdAt).  
  - `keyword:{term}:likes` (sorted set, postIds scored by likeCount).  
- Kafka: Topic `posts` and `likes`, partitioned by `postId`.  
- S3: `keyword/{term}/{range}` for cold indexes.

---

## High-Level Design
**Architecture**: Microservices with inverted index for search.  
**Components**:  
1. **Client**: Web/mobile app for posting, liking, searching.  
2. **API Gateway**: Routes requests, handles auth, rate limiting.  
3. **Ingestion Service**: Processes posts/likes, updates indexes.  
4. **Search Service**: Queries indexes, returns results.  
5. **Database**: Postgres for posts/likes.  
6. **Index**: Redis for hot inverted indexes.  
7. **Queue**: Kafka for write durability.  
8. **Cold Storage**: S3 for old/unpopular indexes.

**Data Flow**:  
- **Create Post**: Client → API Gateway → Ingestion Service → Kafka → Postgres/Redis → Ack.  
- **Like Post**: Client → API Gateway → Ingestion Service → Kafka → Postgres/Redis → Ack.  
- **Search**: Client → API Gateway → Search Service → Redis/Postgres/S3 → Client.

**Diagram** (verbalize):  
- Client → API Gateway → Ingestion/Search Service.  
- Ingestion → Kafka → Postgres/Redis; Search → Redis/S3/Postgres.

---

## Deep Dives
Key challenges addressing non-functional requirements.

### 1. Handling High Request Volume
**Challenge**: ~10K searches/s, no personalization (same query = same results).  
**Solution**:  
- **Caching**:  
  - Redis cache for query results (`query:{keyword}:{sort}:results`, TTL 1m).  
  - Cache hit ratio high due to no personalization.  
- **Search Service**:  
  - Horizontal scaling (~100 servers, auto-scale on QPS).  
  - Load balance queries by keyword hash.  
- **Rate Limiting**: API Gateway throttles abusive clients.  
- **Why?**: Caching reduces DB/index load; scaling handles bursts.  
- **Alternative**: No cache (Redis overload); CDN (complex for dynamic data).  
**Interview Tip**: Lead with caching, estimate cache hit rate. If probed, discuss eviction (LRU) or sharding.

### 2. Multi-Keyword/Phrase Queries
**Challenge**: Support queries like “Taylor Swift” beyond single keywords.  
**Solution**:  
- **Bigrams**:  
  - Index consecutive word pairs (e.g., “Taylor Swift” → `taylor_swift`).  
  - Query: Intersect `keyword:taylor:created` and `keyword:swift:created`.  
- **Preprocessing**:  
  - Tokenize posts, remove stop words (e.g., “the”, “is”).  
  - Store bigrams in Redis (`keyword:taylor_swift:created`).  
- **Query Logic**:  
  - Split query into keywords/bigrams.  
  - Fetch postIds, intersect for multi-keyword, sort by createdAt/likeCount.  
- **Why?**: Bigrams balance precision and index size; intersection is fast in Redis.  
- **Alternative**: Trie (complex updates); full-text scan (slow).  
**Interview Tip**: Propose bigrams, explain intersection. If probed, discuss stop words or phrase limits.

### 3. Handling High Write Volume
**Challenge**: 10K posts/s, 100K likes/s overwhelm indexes.  
**Solution**:  
- **Kafka Queue**:  
  - Buffer posts (`posts` topic) and likes (`likes` topic), partitioned by `postId`.  
  - Ingestion Service consumes asynchronously, batches writes.  
- **Sharding**:  
  - Shard Redis by keyword hash (~10K shards).  
  - Distribute writes across Ingestion Service instances.  
- **Like Optimization**:  
  - Batch like updates (e.g., update `keyword:{term}:likes` every 10 likes or 1s).  
  - Use Redis sorted set (`score = log(likeCount)` to normalize).  
- **Idempotency**:  
  - Dedupe likes via Postgres `postId,userId` constraint.  
- **Back-of-Envelope**:  
  - 100 words/post × 10K posts/s = 1M keyword writes/s.  
  - 100K likes/s ÷ 10 batch = 10K updates/s.  
- **Why?**: Kafka ensures durability; sharding scales writes; batching reduces Redis load.  
- **Alternative**: Direct writes (drops data); no batching (Redis bottleneck).  
**Interview Tip**: Suggest Kafka and batching, note like volume. If probed, discuss sharding or deduplication.

### 4. Optimizing Storage
**Challenge**: 3.6PB for 3.6T posts; Redis can’t hold all indexes.  
**Solution**:  
- **Index Capping**:  
  - Limit Redis lists/sorted sets to 10K postIds per keyword.  
  - Evict old/unpopular posts to S3 (`keyword/{term}/{createdAt_range}`).  
- **Cold Storage**:  
  - Batch job moves rarely searched keywords to S3 (TTL 30 days).  
  - Query S3 on Redis miss (~1s penalty, acceptable for cold data).  
- **Compression**:  
  - Delta-encode postIds in Redis lists (reduce memory).  
- **Hot/Cold Segregation**:  
  - Keep recent/popular keywords in Redis (e.g., top 1M terms).  
- **Back-of-Envelope**:  
  - 1M keywords × 10K posts × 8B/postId ≈ 80TB (Redis, sharded).  
  - Cold data (~3PB) in S3.  
- **Why?**: Capping fits Redis; S3 is cheap for cold data; compression saves memory.  
- **Alternative**: Single DB (costly); no cold storage (Redis overload).  
**Interview Tip**: Propose capping and S3, estimate storage. If probed, discuss batch job or compression.

---

## Interview Checklist
- **Structure** (35 min):  
  1. Requirements: Functional/non-functional (2 min).  
  2. Estimations: QPS, storage (2 min).  
  3. API: Endpoints (2 min).  
  4. Database: Postgres/Redis (2 min).  
  5. High-Level: Microservices, flow (5 min).  
  6. Deep Dives: Requests, multi-keyword, writes, storage (15–20 min).  
- **Whiteboard** (verbalize):  
  - Draw: Client → API Gateway → Ingestion/Search → Kafka/Postgres/Redis/S3.  
  - Show: Post/like (Kafka → Redis), search (Redis → S3/Postgres).  
- **Avoid Pitfalls**:  
  - Don’t use full-text scan (slow).  
  - Don’t ignore like volume (index overload).  
  - Don’t store all indexes in Redis (OOM).  
  - Don’t skip caching (search latency).  
- **Trade-Offs**:  
  - Redis vs. Cassandra: Speed vs. durability.  
  - Bigrams vs. single keywords: Precision vs. simplicity.  
  - Kafka vs. direct writes: Durability vs. latency.  
- **Mid-Level Expectations**:  
  - Define APIs, basic ingestion/query flow.  
  - Propose inverted index (Redis), Postgres for data.  
  - Address writes naively (e.g., single service).  
  - Reason through probes; expect guidance.

---

This summary is lean and optimized for your preparation. Send the next system design, and I’ll maintain the format!
