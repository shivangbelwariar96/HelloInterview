<img width="1127" alt="Screenshot 2025-04-13 at 4 39 34 PM" src="https://github.com/user-attachments/assets/af5694b5-024d-48ff-98a6-bfa1d1867f27" />


# System Design: Fitness Tracking App Like Strava

This is a concise, interview-ready summary for designing a Strava-like fitness tracking app, optimized for a 35-minute interview. It covers all critical points to succeed, omitting unnecessary details (e.g., full DB schemas) while aligning with your 20+ system design cheatsheet for efficient study.

---

## Overview
**Name**: Fitness Tracking Service (Strava-like)  
**Purpose**: Mobile app to record, view, and share running/cycling activities with analytics.  
**Scope**: Core features (start/pause/save activities, view activity data, view own/friends’ activities). Excludes friend management, comments, authentication.

---

## Functional Requirements
- **Manage Activities**: Users can start, pause, stop, and save runs/rides.  
- **View Activity Data**: Users can see real-time route, distance, and time during activities.  
- **View Completed Activities**: Users can view details of their own and friends’ completed activities.

---

## Non-Functional Requirements
- **Availability**: High availability, eventual consistency acceptable.  
- **Offline Support**: App must function without network connectivity.  
- **Accuracy**: Provide accurate, up-to-date local stats during activities.  
- **Scalability**: Support 10M concurrent activities (~100M daily users).

---

## API Design
RESTful APIs with JWT for authentication (WebSocket for real-time considered in deep dive).

1. **Create Activity**:  
   - `POST /activities`  
   - Body: `{ type: "RUN" | "RIDE" }`  
   - Returns: `{ activityId: string, type: string, state: string }`  
   - Purpose: Initialize activity.  

2. **Update Activity State**:  
   - `PATCH /activities/{activityId}`  
   - Body: `{ state: "STARTED" | "PAUSED" | "COMPLETE" }`  
   - Returns: `{ activityId: string, state: string }`  
   - Purpose: Update activity status.  

3. **Add Route Data**:  
   - `POST /activities/{activityId}/routes`  
   - Body: `{ location: { lat: float, long: float }, timestamp: int }`  
   - Returns: `{ status: string }`  
   - Purpose: Append GPS coordinate (batched for offline).  

4. **List Activities**:  
   - `GET /activities?mode={USER|FRIENDS}&page={int}&limit={int}`  
   - Returns: `{ activities: [{ id, userId, type, distance, duration, date }] }`  
   - Purpose: Fetch paginated activity summaries.  

5. **View Activity**:  
   - `GET /activities/{activityId}`  
   - Returns: `{ id, userId, type, state, route: [{ lat, long, timestamp }], distance, duration }`  
   - Purpose: Fetch detailed activity data.

---

## Database
**Choices**:  
- **Data**: DynamoDB for activities (scalable, simple queries).  
- **Cache**: Redis for hot activity lists.  
- **Client Storage**: SQLite (iOS Core Data/Android Room) for offline buffering.  
**Key Entities**:  
- **Activity**: activityId, userId, type, state, startTime, duration, distance, route [{lat, long, timestamp}].  
- **Friend**: userId, friendId (bi-directional).  
**Indexing**:  
- DynamoDB:  
  - PK `activityId`; GSI PK `userId`, SK `date` for user activities.  
  - Friend: PK `userId`, SK `friendId`.  
- Redis: Sorted set `activities:user:{userId}` for cached lists.  
- SQLite: Local table for activity buffer (`activityId`, `locations`).

---

## High-Level Design
**Architecture**: Client-heavy monolith to leverage device capabilities, minimize server load.  
**Components**:  
1. **Client**: Mobile app with GPS, local storage, UI (Google Maps API for routes).  
2. **API Gateway**: Routes requests, handles auth.  
3. **Activity Service**: Manages activity lifecycle, persistence.  
4. **Database**: DynamoDB for activities/friends.  
5. **Cache**: Redis for activity lists.  
6. **Client Storage**: SQLite for offline data.

**Data Flow**:  
- **Manage Activity**: Client → API Gateway → Activity Service → DynamoDB; offline data buffered locally.  
- **View Data**: Client computes distance/time locally (Haversine); syncs on completion.  
- **View Activities**: Client → API Gateway → Activity Service → DynamoDB/Redis → Client.

**Diagram** (verbalize):  
- Client ↔ API Gateway → Activity Service → DynamoDB/Redis.  
- Client: GPS → SQLite (offline) → DynamoDB (sync).

---

## Deep Dives
Key challenges addressing non-functional requirements.

### 1. Offline Activity Tracking
**Challenge**: Support activity tracking in remote areas without connectivity.  
**Solution**:  
- **Local Processing**:  
  - Client records GPS every 2s (ride) or 5s (run) using iOS Core Location/Android FusedLocation.  
  - Store in SQLite (in-memory buffer, persist every 10s).  
  - Compute distance (Haversine), time locally.  
- **Sync**:  
  - On reconnect, batch upload route data to `/activities/{activityId}/routes`.  
  - Background sync for partial uploads if online mid-activity.  
- **Data Safety**: Max 10s data loss on crash (persistence interval).  
- **Why?**: Client offloads server; SQLite ensures durability.  
- **Alternative**: Server-side tracking (network-dependent); no persistence (data loss).  
**Interview Tip**: Emphasize client-side logic, batch sync. If probed, discuss battery trade-offs.

### 2. Scalability for 10M Concurrent Activities
**Challenge**: Handle ~100M activities/day (~1K QPS writes, 547TB/year).  
**Solution**:  
- **Client Offloading**: Local tracking reduces server writes to completion (~1K QPS vs. 2M QPS).  
- **DynamoDB**:  
  - Shard by `activityId`; GSI for `userId` queries.  
  - Auto-scale for ~1K writes/s, ~15KB/activity.  
- **Data Tiering**:  
  - Hot: Recent activities (DynamoDB).  
  - Warm: 3–12 months (DynamoDB cheaper tables).  
  - Cold: >1 year (S3 Glacier).  
- **Activity Service**: Horizontal scaling (~10 servers, low CPU).  
- **Back-of-Envelope**:  
  - 100M activities × 15KB = 1.5TB/day, 547TB/year.  
  - 100M ÷ 86,400s ≈ 1K QPS.  
- **Why?**: Client reduces load; DynamoDB scales writes; tiering saves costs.  
- **Alternative**: Microservices (overkill); SQL (complex sharding).  
**Interview Tip**: Highlight client offload, estimate storage. If probed, discuss tiering or caching.

### 3. Real-Time Sharing with Friends
**Challenge**: Allow friends to follow activities live.  
**Solution**:  
- **Polling**:  
  - Client sends GPS updates every 5s to server (if online).  
  - Friends’ clients poll `/activities/{activityId}` every 5s (offset for latency).  
- **Buffering**: Lag display by 10s for smooth animation (client-side).  
- **DynamoDB**: Store live updates in `activityId` with TTL (1h post-completion).  
- **Why?**: Polling leverages predictability (5s updates); buffering improves UX.  
- **Alternative**: WebSocket (complex, high connections); pub/sub (overkill for low fan-out).  
**Interview Tip**: Propose polling, justify simplicity. If probed, discuss WebSocket trade-offs.

### 4. Leaderboard of Top Athletes
**Challenge**: Show top athletes by activity type, distance, region.  
**Solution**:  
- **Redis Sorted Sets**:  
  - Key: `leaderboard:{type}:{region}:{distance}` (e.g., `leaderboard:run:USA:5K`).  
  - Score: Activity duration; member: `userId`.  
  - Update on activity completion; TTL 24h.  
- **Aggregation**:  
  - Activity Service writes to Redis on `COMPLETE`.  
  - Query `ZRANGE` for top N (e.g., top 100).  
- **Filters**: Precompute sets for common regions (USA, NYC) and distances (5K, 10K).  
- **Why?**: Redis for low-latency ranking; TTL manages size.  
- **Alternative**: DB query (slow); Kafka (complex for top-K).  
**Interview Tip**: Suggest Redis sorted sets, mention filters. If probed, discuss granularity or stale data.

---

## Interview Checklist
- **Structure** (35 min):  
  1. Requirements: Functional/non-functional (2 min).  
  2. API: Endpoints (2 min).  
  3. Database: DynamoDB/Redis/SQLite (2 min).  
  4. High-Level: Client-heavy monolith (5 min).  
  5. Deep Dives: Offline, scale, real-time, leaderboard (15–20 min).  
- **Whiteboard** (verbalize):  
  - Draw: Client → API Gateway → Activity Service → DynamoDB/Redis.  
  - Show: Offline (SQLite → sync), view (DynamoDB), leaderboard (Redis).  
- **Avoid Pitfalls**:  
  - Don’t rely on network for tracking (offline fails).  
  - Don’t use microservices (unneeded complexity).  
  - Don’t skip client storage (data loss).  
  - Don’t propose WebSocket first (over-engineered).  
- **Trade-Offs**:  
  - Client vs. server: Offline vs. load.  
  - Polling vs. WebSocket: Simplicity vs. latency.  
  - DynamoDB vs. SQL: Scale vs. relations.  
- **Mid-Level Expectations**:  
  - Define APIs, client/server model.  
  - Propose DynamoDB for storage, client for offline.  
  - Address offline naively (e.g., local buffer).  
  - Reason through probes; expect guidance.

---

This summary is streamlined for your study. Send the next system design, and I’ll keep the format consistent!
