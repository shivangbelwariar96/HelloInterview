<img width="725" alt="Screenshot 2025-04-13 at 4 35 33 PM" src="https://github.com/user-attachments/assets/0e8f272f-1ff0-40c8-bd1d-3904d5c71c0c" />

#System Design: Dating App Like Tinder

This is a concise, interview-ready summary for designing a dating app like Tinder, tailored for a 35-minute interview. It covers all essential points to excel, omitting extraneous details (e.g., exhaustive schemas) while aligning with your 20+ system design cheatsheet for quick study.

---

## Overview
**Name**: Dating App Service (Tinder-like)  
**Purpose**: Mobile app for connecting users via swiping based on location, preferences, and mutual interest.  
**Scope**: Core features (create profile, view matches, swipe, match notifications). Excludes photos, chat, premium features.

---

## Functional Requirements
- **Create Profile**: Users set preferences (age, interests, max distance).  
- **View Matches**: Users see a stack of profiles matching preferences/location.  
- **Swipe**: Users swipe right ("yes") or left ("no") on profiles.  
- **Match Notifications**: Users are notified of mutual "yes" swipes.

---

## Non-Functional Requirements
- **Consistency**: Strong consistency for swipes (immediate match detection).  
- **Scalability**: Support 20M daily active users, ~100 swipes/day (~2B swipes/day).  
- **Performance**: Match stack load <300ms; swipe processing <100ms.  
- **No Repeats**: Avoid showing previously swiped profiles.

---

## API Design
RESTful APIs with JWT for authentication.

1. **Create Profile**:  
   - `POST /profile`  
   - Body: `{ ageMin: int, ageMax: int, distance: int, interestedIn: string }`  
   - Returns: `{ status: string }`  
   - Purpose: Save user preferences.  

2. **View Matches**:  
   - `GET /feed?lat={float}&long={float}`  
   - Returns: `{ profiles: [{ id, name, age, bio }] }`  
   - Purpose: Fetch stack of profiles (no pagination needed).  

3. **Swipe**:  
   - `POST /swipe/{targetId}`  
   - Body: `{ decision: "yes" | "no" }`  
   - Returns: `{ match: bool, matchId: string | null }`  
   - Purpose: Record swipe, check for match.  

---

## Database
**Choices**:  
- **Profiles**: Postgres for structured data, geospatial indexing.  
- **Swipes/Matches**: Cassandra for high write throughput.  
- **Cache**: Redis for feed and swipe history.  
- **Search**: Elasticsearch for profile filtering.  
**Key Entities**:  
- **User**: ID, name, age, location (lat, long), preferences (ageMin, ageMax, distance, interestedIn).  
- **Swipe**: swipingUserId, targetUserId, decision, timestamp.  
- **Match**: ID, userId1, userId2, timestamp.  
**Indexing**:  
- Postgres: Geospatial (PostGIS) on `lat`, `long`; `userId`.  
- Cassandra: Partition key `swipingUserId`, clustering key `targetUserId`.  
- Elasticsearch: Index age, interests, location.  
- Redis: Bloom filter for swipes, feed cache.

---

## High-Level Design
**Architecture**: Microservices for modularity; monolith optional for simplicity.  
**Components**:  
1. **Client**: Mobile app for swiping/profiles.  
2. **API Gateway**: Routes requests, handles auth/rate limiting.  
3. **Profile Service**: Manages user profiles/preferences.  
4. **Feed Service**: Generates match stacks.  
5. **Swipe Service**: Processes swipes, detects matches.  
6. **Database**: Postgres (profiles), Cassandra (swipes/matches).  
7. **Cache**: Redis for feeds/swipe history.  
8. **Search**: Elasticsearch for profile queries.  
9. **Notification Service**: Push via APNS/FCM for matches.  

**Data Flow**:  
- **Profile**: Client → Profile Service → Postgres.  
- **Feed**: Client → Feed Service → Elasticsearch/Redis → Client.  
- **Swipe**: Client → Swipe Service → Cassandra/Redis → Notification Service.  

**Diagram** (verbalize):  
- Client → API Gateway → Profile/Feed/Swipe Service.  
- Profile → Postgres; Feed → Elasticsearch/Redis; Swipe → Cassandra/Redis → APNS/FCM.  

---

## Deep Dives
Key challenges addressing non-functional requirements.

### 1. Consistent, Low-Latency Swiping
**Challenge**: Ensure mutual swipes trigger immediate matches without misses.  
**Solution**:  
- **Cassandra with Locks**:  
  - Write swipe to Cassandra (`swipingUserId:targetUserId`).  
  - Use lightweight transactions (Paxos) to check inverse swipe atomically.  
  - If match, write to Match table, trigger notifications.  
- **Redis Cache**: Cache recent swipes (`swipe:{userId}`) for faster checks.  
- **Why?**: Cassandra ensures consistency; Redis reduces latency (<100ms).  
- **Alternative**: Periodic reconciliation (inconsistent); Postgres (slower writes).  
**Interview Tip**: Propose Cassandra transactions, mention Redis. If probed, discuss trade-offs or Paxos overhead.

### 2. Low-Latency Feed Generation (<300ms)
**Challenge**: Generate match stacks quickly with filters and location.  
**Solution**:  
- **Elasticsearch**:  
  - Index profiles (age, interests, geohash).  
  - Query: `geo_distance` filter + preference match, return ~50 profiles.  
- **Feed Cache**:  
  - Redis stores precomputed stacks (`feed:{userId}`, TTL 1h).  
  - Background job refreshes for active users.  
- **Geohashing**: Precompute nearby users per grid (~1km).  
- **Why?**: Elasticsearch for fast filtering; cache for <300ms loads.  
- **Alternative**: Postgres/PostGIS (slower); no cache (query bottleneck).  
**Interview Tip**: Lead with Elasticsearch/cache, mention geohashing. If probed, discuss TTL or ML ranking.

### 3. Avoid Re-Showing Swiped Profiles
**Challenge**: Prevent showing profiles already swiped on (~2B swipes/day).  
**Solution**:  
- **Bloom Filter**:  
  - Redis Bloom filter (`swiped:{userId}`) tracks swiped target IDs.  
  - Check before adding profile to feed; false positives OK (rare misses).  
  - Size: ~1GB for 20M users, 100 swipes/day, 1% error rate.  
- **Cassandra Backup**: Query swipe history on cache miss.  
- **Why?**: Bloom filter is space-efficient, fast; avoids DB hits.  
- **Alternative**: Full swipe history (storage heavy); no filter (bad UX).  
**Interview Tip**: Propose Bloom filter, estimate size. If probed, discuss false positives or cleanup.

### 4. Scalability (20M DAU)
**Challenge**: Handle ~23K swipes/second (2B/day ÷ 86,400s).  
**Solution**:  
- **Horizontal Scaling**:  
  - Stateless Profile/Feed/Swipe Services auto-scale.  
  - Cassandra scales writes via partitioning (`swipingUserId`).  
- **Caching**: Redis cluster for feeds/swipes (~90% hit rate).  
- **Async Notifications**: Kafka for APNS/FCM push.  
- **Why?**: Cassandra for write scale; cache for reads; Kafka for async.  
- **Alternative**: Single DB (bottleneck); no cache (slow).  
**Interview Tip**: Highlight Cassandra/Redis, estimate QPS. If probed, mention sharding or Kafka.

---

## Interview Checklist
- **Structure** (35 min):  
  1. Requirements: Functional/non-functional (2 min).  
  2. API: Endpoints (2 min).  
  3. Database: Postgres/Cassandra/Redis/Elasticsearch (2 min).  
  4. High-Level: Microservices, flow (5 min).  
  5. Deep Dives: Swipes, feed, repeats, scale (15–20 min).  
- **Whiteboard** (verbalize):  
  - Draw: Client → API Gateway → Services → Postgres/Cassandra/Redis/Elasticsearch.  
  - Show: Profile (write), feed (cache), swipe (transaction → push).  
- **Avoid Pitfalls**:  
  - Don’t use eventual consistency for swipes (missed matches).  
  - Don’t skip Bloom filter (repeat profiles).  
  - Don’t use slow queries (e.g., SQL for feed).  
  - Don’t ignore location (bad UX).  
- **Trade-Offs**:  
  - Cassandra vs. Postgres: Write scale vs. simplicity.  
  - Bloom filter vs. DB: Space vs. accuracy.  
  - Cache vs. live query: Speed vs. freshness.  
- **Mid-Level Expectations**:  
  - Define APIs, data model.  
  - Propose Postgres for profiles, Cassandra for swipes, Elasticsearch for feed.  
  - Address repeats naively (e.g., DB check).  
  - Reason through probes; expect guidance.

---

This summary is lean and comprehensive, tailored for your study. Send the next system design, and I’ll keep the format consistent!
