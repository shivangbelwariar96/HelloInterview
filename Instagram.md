<img width="1134" alt="Screenshot 2025-04-13 at 4 43 48 PM" src="https://github.com/user-attachments/assets/b8ecb51c-1577-45e1-a533-8fdd20fe090b" />


# System Design: Photo Sharing App Like Instagram

This is a concise, interview-ready summary for designing a photo-sharing app like Instagram, optimized for a 35-minute interview. It covers all critical points to excel, omitting unnecessary details (e.g., full DB schemas) while aligning with your 20+ system design cheatsheet for efficient study. The design is classified as "hard" due to its scale (500M DAU) and media-heavy requirements.

---

## Overview
**Name**: Photo Sharing Service (Instagram-like)  
**Purpose**: Allow users to create photo/video posts, follow others, and view chronological feeds.  
**Scope**: Core features (create posts, follow users, view feeds). Excludes likes, comments, search, stories, live streaming.

---

## Functional Requirements
- **Create Posts**: Users can create posts with photos (≤8MB), videos (≤4GB), and captions.  
- **Follow Users**: Users can follow other users (unidirectional).  
- **View Feed**: Users see a chronological feed of posts from followed users.

---

## Non-Functional Requirements
- **Availability**: Prioritize availability over consistency (eventual consistency OK, ≤2 min).  
- **Latency**: Feed delivery <500ms; media rendering “instant” (<1s for photos, <2s for videos).  
- **Scalability**: Support 500M DAU, 100M posts/day (~1K posts/s).  
- **Storage**: Handle ~200TB/day of media (2MB/post avg).

---

## Scale Estimations
- **Users**: 500M DAU.  
- **Posts**: 100M posts/day ÷ 86K s/day ≈ 1K posts/s.  
- **Media**: 100M posts × 2MB avg = 200TB/day, ~750PB over 10 years.  
- **Follows**: ~100 follows/user avg, ~50B follow relationships.  
- **Feed Views**: 500M DAU × 5 views/day ÷ 86K s ≈ 30K QPS.  
**Insight**: Media-heavy writes, read-heavy feeds, massive storage needs.

---

## API Design
RESTful APIs with JWT for authentication; pre-signed URLs for media uploads.

1. **Create Post**:  
   - `POST /posts`  
   - Header: `Authorization: Bearer <JWT>`  
   - Body: `{ caption: string }`  
   - Returns: `{ postId: string, uploadUrl: string }`  
   - Purpose: Create post metadata, get pre-signed S3 URL.  

2. **Upload Media**:  
   - `PUT {uploadUrl}` (S3 pre-signed URL)  
   - Body: Media bytes (multipart for videos).  
   - Purpose: Upload media directly to S3.  

3. **Follow User**:  
   - `POST /follows`  
   - Header: `Authorization: Bearer <JWT>`  
   - Body: `{ followedId: string }`  
   - Returns: `{ status: "SUCCESS" }`  
   - Purpose: Create follow relationship.  

4. **Get Feed**:  
   - `GET /feed?cursor={string}&limit={int}`  
   - Returns: `{ posts: [{ id, userId, caption, mediaUrl, createdAt }], nextCursor: string }`  
   - Purpose: Fetch paginated feed posts.

---

## Database
**Choices**:  
- **Data**: DynamoDB for metadata (scalable, eventual consistency).  
- **Cache**: Redis for feeds (low-latency reads).  
- **Storage**: S3 for media; Glacier for cold storage.  
- **Queue**: Kafka for post processing.  
**Key Entities**:  
- **User**: userId, username.  
- **Post**: postId, userId, caption, mediaKey, createdAt, uploadStatus.  
- **Follow**: followerId, followedId.  
- **Media**: S3 key, type (photo/video).  
**Indexing**:  
- DynamoDB:  
  - Post: PK `userId`, SK `createdAt#postId`.  
  - Follow: PK `followerId`, SK `followedId`; GSI PK `followedId`.  
- Redis:  
  - `feed:{userId}` (sorted set, postIds scored by createdAt).  
- S3: `media/{postId}/{type}` for media files.  
- Kafka: Topic `posts`, partitioned by `postId`.

---

## High-Level Design
**Architecture**: Microservices with hybrid feed generation, CDN for media.  
**Components**:  
1. **Client**: Web/mobile app for posting, following, viewing.  
2. **API Gateway**: Routes requests, handles auth, rate limiting.  
3. **Post Service**: Manages post creation, metadata updates.  
4. **Follow Service**: Handles follow relationships.  
5. **Feed Service**: Generates/serves feeds.  
6. **Database**: DynamoDB for metadata.  
7. **Cache**: Redis for precomputed feeds.  
8. **Storage**: S3 for media, Glacier for cold data.  
9. **Queue**: Kafka for post fan-out.  
10. **CDN**: CloudFront for media delivery.

**Data Flow**:  
- **Create Post**: Client → API Gateway → Post Service → Kafka → DynamoDB/S3; fan-out to Redis.  
- **Follow User**: Client → API Gateway → Follow Service → DynamoDB.  
- **View Feed**: Client → API Gateway → Feed Service → Redis/DynamoDB → CDN → Client.

**Diagram** (verbalize):  
- Client → API Gateway → Post/Follow/Feed Service.  
- Post → Kafka → DynamoDB/S3/Redis; Feed → Redis/CDN/DynamoDB.

---

## Deep Dives
Key challenges addressing non-functional requirements.

### 1. Low-Latency Feed Delivery (<500ms)
**Challenge**: Fan-out on read (query followed users’ posts) scales poorly for 500M DAU.  
**Solution**:  
- **Hybrid Fan-Out**:  
  - **Fan-Out on Write**: For users with <1K followers, precompute feeds in Redis (`feed:{userId}`) when posts are created.  
  - **Fan-Out on Read**: For celebrities (>1K followers), query posts at read time, merge with cached feeds.  
  - Kafka topic `posts` fans out posts to followers’ Redis sets.  
- **Redis Cache**:  
  - Store ~100 posts/user in `feed:{userId}` (sorted set, score = createdAt).  
  - TTL 7 days; fallback to DynamoDB on miss.  
- **Indexing**:  
  - DynamoDB GSI on `followedId` for celebrity post queries.  
- **Back-of-Envelope**:  
  - 500M users × 100 posts × 100B/postId = 5TB Redis (sharded).  
  - 30K QPS × 100 posts × 1ms = 300K ops/s (Redis handles).  
- **Why?**: Precomputing reduces read QPS; hybrid handles celebrities; Redis ensures <500ms.  
- **Alternative**: Full fan-out on read (slow); full fan-out on write (write amplification).  
**Interview Tip**: Lead with hybrid, justify threshold. If probed, discuss Redis durability (AOF, Sentinel).

### 2. Instant Media Rendering
**Challenge**: Deliver 8MB photos, 4GB videos with minimal latency.  
**Solution**:  
- **Upload**:  
  - Client → Post Service → `POST /posts` → `{ postId, uploadUrl }`.  
  - Client → S3 multipart upload via pre-signed URLs (chunks ~10MB).  
  - S3 → Lambda → Update DynamoDB `uploadStatus = COMPLETE`.  
- **Delivery**:  
  - CDN (CloudFront) caches media at edge locations.  
  - Dynamic optimization: Compress photos (WebP), transcode videos (HLS/DASH).  
  - Signed URLs for access control (TTL 1h).  
- **Back-of-Envelope**:  
  - 200TB/day × 30 days = 6PB hot storage (S3).  
  - 500M DAU × 5 views × 2MB = 5PB/day CDN bandwidth.  
- **Why?**: Multipart ensures reliable uploads; CDN cuts latency; optimization saves bandwidth.  
- **Alternative**: Direct server upload (bottleneck); no CDN (high latency).  
**Interview Tip**: Propose multipart and CDN, explain S3-Lambda. If probed, discuss video streaming or compression.

### 3. Scalability for 500M DAU
**Challenge**: Handle 1K posts/s, 30K feed QPS, 200TB/day media.  
**Solution**:  
- **Services**:  
  - Horizontal scaling for Post/Follow/Feed Services (~1K servers total).  
  - Load balancers distribute traffic (hash `userId`).  
- **Database**:  
  - DynamoDB auto-scales partitions (~10K for posts/follows).  
  - Eventual consistency (≤2 min) for fan-out.  
- **Cache**:  
  - Redis Cluster, sharded by `userId` (~100 nodes).  
  - Evict old feeds to DynamoDB/S3.  
- **Storage**:  
  - S3 for hot media; Glacier for posts >1 year old.  
  - ~750PB over 10 years, tiered storage saves costs.  
- **Queue**:  
  - Kafka partitions `posts` by `postId` (~1K partitions).  
  - Batch fan-out to Redis (100 posts/batch).  
- **Back-of-Envelope**:  
  - 1K posts/s × 100 followers × 100B = 10M writes/s (Redis/Kafka).  
  - 30K QPS × 100 posts × 1KB = 3GB/s feed bandwidth.  
- **Why?**: Sharding scales writes; CDN/storage tiering manages media; Kafka buffers bursts.  
- **Alternative**: Single DB (chokes); no tiering (costly).  
**Interview Tip**: Summarize scaling for each component, estimate QPS. If probed, discuss cost optimization (Glacier).

---

## Interview Checklist
- **Structure** (35 min):  
  1. Requirements: Functional/non-functional (2 min).  
  2. Estimations: QPS, storage (2 min).  
  3. API: Endpoints, upload flow (2 min).  
  4. Database: DynamoDB/Redis/S3 (2 min).  
  5. High-Level: Microservices, flow (5 min).  
  6. Deep Dives: Feed, media, scale (15–20 min).  
- **Whiteboard** (verbalize):  
  - Draw: Client → API Gateway → Post/Follow/Feed → Kafka/DynamoDB/Redis/S3/CDN.  
  - Show: Post (Kafka → S3/Redis), feed (Redis → CDN), follow (DynamoDB).  
- **Avoid Pitfalls**:  
  - Don’t use fan-out on read only (slow).  
  - Don’t upload media via servers (bottleneck).  
  - Don’t ignore celebrity accounts (write overload).  
  - Don’t store all media in hot storage (costly).  
- **Trade-Offs**:  
  - Fan-out write vs. read: Write cost vs. read latency.  
  - DynamoDB vs. Postgres: Scalability vs. relations.  
  - Redis vs. DB feeds: Speed vs. durability.  
- **Mid-Level Expectations**:  
  - Define APIs, basic microservices, S3 for media.  
  - Propose fan-out on read, pivot to write with hints.  
  - Address media naively (e.g., direct S3).  
  - Reason through probes; expect guidance.

---

This summary is streamlined for your study. Send the next system design, and I’ll keep the format consistent!
