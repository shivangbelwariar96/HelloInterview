# System Design: Facebook's News Feed

This is a concise, interview-ready summary for designing a social media news feed like Facebook’s, optimized for a 35-minute interview. It covers all critical points to succeed, omitting unnecessary details (e.g., full DB schemas) while aligning with your 20+ system design cheatsheet for quick study.

---

## Overview
**Name**: News Feed Service (Facebook-like)  
**Purpose**: Social network feed showing recent posts from followed users in chronological order.  
**Scope**: Core features (create posts, follow users, view/paginate feed). Excludes likes, comments, private posts.

---

## Functional Requirements
- **Create Posts**: Users can create posts.  
- **Follow Users**: Users can follow others (uni-directional).  
- **View Feed**: Users see posts from followed users, chronologically.  
- **Paginate Feed**: Users can scroll through older posts.

---

## Non-Functional Requirements
- **Availability**: High availability, prioritizing over consistency (up to 2-min eventual consistency).  
- **Performance**: Post creation and feed view <500ms.  
- **Scalability**: Support 2B users, unlimited follows/followers.  
- **Read-Heavy**: Feed views dominate over posts/follows.

---

## API Design
RESTful APIs with JWT for authentication.

1. **Create Post**:  
   - `POST /posts`  
   - Body: `{ content: object }`  
   - Returns: `{ postId: string }`  
   - Purpose: Create a new post.  

2. **Follow User**:  
   - `POST /users/{userId}/follow`  
   - Body: `{}` (idempotent)  
   - Returns: `{ status: string }`  
   - Purpose: Follow a user.  

3. **View Feed**:  
   - `GET /feed?after={timestamp}&limit={int}`  
   - Returns: `{ posts: [{ id, userId, content, timestamp }] }`  
   - Purpose: Fetch feed posts, paginated by timestamp.

---

## Database
**Choices**:  
- **Posts/Follows**: DynamoDB for scalability, fast key-value access.  
- **Cache**: Redis for feed caching.  
**Key Entities**:  
- **User**: ID, metadata (out of scope for storage).  
- **Post**: ID, user ID, content, timestamp.  
- **Follow**: userIdFollowing:userIdFollowed (key), timestamp.  
**Indexing**:  
- Posts: Partition key `userId`, sort key `timestamp`.  
- Follows: Partition key `userIdFollowing`, GSI on `userIdFollowed`.  
- Redis: Feed cache (`feed:{userId}` → sorted post IDs).

---

## High-Level Design
**Architecture**: Microservices for scale; monolith optional for simplicity.  
**Components**:  
1. **Client**: Web/mobile app for posting/viewing.  
2. **API Gateway**: Routes requests, handles auth/rate limiting.  
3. **Post Service**: Creates/stores posts.  
4. **Follow Service**: Manages follow relationships.  
5. **Feed Service**: Builds/serves feeds.  
6. **Database**: DynamoDB for posts/follows.  
7. **Cache**: Redis for precomputed feeds.  
8. **Feed Workers**: Async fan-out for popular users.

**Data Flow**:  
- **Post**: Client → Post Service → DynamoDB → Feed Workers (async).  
- **Follow**: Client → Follow Service → DynamoDB.  
- **Feed**: Client → Feed Service → Redis/DynamoDB → Client.  

**Diagram** (verbalize):  
- Client → API Gateway → Post/Follow/Feed Service.  
- Post/Follow → DynamoDB; Feed → Redis or DynamoDB.  
- Feed Workers update Redis for popular users.

---

## Deep Dives
Key challenges addressing non-functional requirements.

### 1. Users Following Many Users (Fan-Out on Read)
**Challenge**: Fetching posts from many followed users is slow, high fan-out.  
**Solution**:  
- **Feed Cache**:  
  - Redis stores precomputed feed (`feed:{userId}` → sorted post IDs, TTL 1 min).  
  - On miss: Query Follow table, fetch posts, cache 500 posts.  
- **Pagination**: Use `after` timestamp to fetch older posts from cache.  
- **Why?**: Cache reduces DynamoDB queries; supports <500ms.  
- **Alternative**: Real-time fetch (slow, high latency); SQL joins (complex).  
**Interview Tip**: Propose Redis cache, mention pagination. If probed, discuss TTL or cache size.

### 2. Users with Many Followers (Fan-Out on Write)
**Challenge**: Posting by a celebrity (e.g., 1M followers) requires updating many feeds.  
**Solution**:  
- **Hybrid Fan-Out**:  
  - **Regular Users**: Fan-out on write—Feed Workers write post ID to followers’ Redis feeds (async via Kafka).  
  - **Celebrities**: Fan-out on read—exclude from feed table, fetch posts dynamically for their followers.  
  - Threshold: ~10K followers (adjustable).  
- **Feed Table**: DynamoDB (`userId` → post IDs, ~200 posts) for regular users.  
- **Why?**: Balances write load; tolerates 2-min consistency.  
- **Alternative**: Full fan-out (write bottleneck); no fan-out (read bottleneck).  
**Interview Tip**: Propose hybrid, explain celebrity threshold. If probed, discuss Kafka or trade-offs.

### 3. Uneven Post Reads (Hot Keys)
**Challenge**: Popular posts overload DynamoDB partitions.  
**Solution**:  
- **Redundant Cache**:  
  - Multiple Redis instances, client hashes to one (`post:{id}:{hash}`).  
  - Cache posts for 1 hour, reducing DynamoDB hits.  
- **DynamoDB Tuning**: Hot partition mitigation via randomized keys.  
- **Why?**: Spreads load across Redis; avoids single-node bottleneck.  
- **Alternative**: Single Redis (hot key); no cache (DB overload).  
**Interview Tip**: Lead with redundant cache, estimate load. If probed, mention DynamoDB tuning.

### 4. Scalability (2B Users)
**Challenge**: Handle massive user base, high read volume.  
**Solution**:  
- **Horizontal Scaling**:  
  - Stateless Post/Follow/Feed Services auto-scale.  
  - DynamoDB auto-scales with `userId` partitioning.  
- **Caching**: Redis cluster for feeds/posts (~90% hit rate).  
- **Async Writes**: Kafka for Feed Workers, batching updates.  
- **Why?**: Scales reads via cache; writes via async processing.  
- **Alternative**: Single DB (bottleneck); no cache (slow).  
**Interview Tip**: Highlight cache/scaling, mention Kafka. If probed, discuss sharding.

---

## Interview Checklist
- **Structure** (35 min):  
  1. Requirements: Functional/non-functional (2 min).  
  2. API: Endpoints (2 min).  
  3. Database: DynamoDB/Redis (2 min).  
  4. High-Level: Microservices, flow (5 min).  
  5. Deep Dives: Fan-out, hot keys, scale (15–20 min).  
- **Whiteboard** (verbalize):  
  - Draw: Client → API Gateway → Services → DynamoDB/Redis/Kafka.  
  - Show: Post (write), feed (cache/read), fan-out (workers).  
- **Avoid Pitfalls**:  
  - Don’t use SQL for scale (join overhead).  
  - Don’t skip cache (DB overload).  
  - Don’t ignore fan-out (performance hit).  
  - Don’t propose full fan-out for celebrities (write bottleneck).  
- **Trade-Offs**:  
  - Fan-out read vs. write: Latency vs. write load.  
  - Redis vs. DynamoDB: Speed vs. durability.  
  - Hybrid vs. full fan-out: Complexity vs. simplicity.  
- **Mid-Level Expectations**:  
  - Define APIs, data model.  
  - Propose DynamoDB for posts/follows, Redis for feeds.  
  - Address fan-out naively (e.g., read-time fetch).  
  - Reason through probes; expect guidance.

---

This summary is lean and tailored for your study. Send the next system design, and I’ll maintain the format!
