<img width="1132" alt="Screenshot 2025-04-13 at 4 40 13 PM" src="https://github.com/user-attachments/assets/513dba3a-a36a-4dc8-9f7f-98a0bc414c81" />


# System Design: Online Auction Platform Like eBay

This is a concise, interview-ready summary for designing an eBay-like online auction platform, tailored for a 35-minute interview. It covers all critical points to excel, omitting extraneous details (e.g., exhaustive schemas) while aligning with your 20+ system design cheatsheet for efficient study.

---

## Overview
**Name**: Auction Service (eBay-like)  
**Purpose**: Platform for users to list items for auction, place bids, and view auction details.  
**Scope**: Core features (create auctions, bid on items, view auctions). Excludes search, filtering, history.

---

## Functional Requirements
- **Create Auction**: Users can post an item with a starting price and end date.  
- **Place Bid**: Users can bid on an item; bids accepted if higher than the current highest bid.  
- **View Auction**: Users can view auction details, including the current highest bid.

---

## Non-Functional Requirements
- **Consistency**: Strong consistency for bids to ensure accurate highest bid.  
- **Durability**: No bids dropped, even during failures.  
- **Real-Time**: Display current highest bid in real-time (<1s latency).  
- **Scalability**: Support 10M concurrent auctions (~10K bids/s).

---

## API Design
RESTful APIs with JWT for authentication; SSE for real-time updates.

1. **Create Auction**:  
   - `POST /auctions`  
   - Body: `{ item: { name: string, description: string, image: string }, startDate: int, endDate: int, startingPrice: float }`  
   - Returns: `{ auctionId: string }`  
   - Purpose: Initialize auction and item.  

2. **Place Bid**:  
   - `POST /auctions/{auctionId}/bids`  
   - Body: `{ amount: float }`  
   - Returns: `{ bidId: string, status: "ACCEPTED" | "REJECTED" }`  
   - Purpose: Submit bid, validate against highest.  

3. **View Auction**:  
   - `GET /auctions/{auctionId}`  
   - Returns: `{ id: string, item: { name, description, image }, startDate: int, endDate: int, highestBid: float, bidderId: string }`  
   - Purpose: Fetch auction details.  

4. **Real-Time Bids**:  
   - `SSE /auctions/{auctionId}/bids`  
   - Returns: Stream `{ bidId: string, amount: float, bidderId: string }`  
   - Purpose: Stream bid updates to clients.

---

## Database
**Choices**:  
- **Data**: Postgres for strong consistency, relational data.  
- **Queue**: Kafka for durable bid processing.  
- **Cache**: Redis for real-time bid state.  
- **Storage**: S3 for item images.  
**Key Entities**:  
- **Auction**: auctionId, itemId, startDate, endDate, startingPrice, highestBid, highestBidderId.  
- **Item**: itemId, name, description, imageUrl.  
- **Bid**: bidId, auctionId, userId, amount, timestamp, status.  
**Indexing**:  
- Postgres:  
  - PK `auctionId` (Auction), `itemId` (Item), `bidId` (Bid).  
  - Index `auctionId` on Bid for history queries.  
- Kafka: Topic `bids` partitioned by `auctionId`.  
- Redis: Key `auction:{auctionId}:highestBid` for current bid.  
- S3: `itemId` as key for images.

---

## High-Level Design
**Architecture**: Microservices for independent scaling of auctions and bids.  
**Components**:  
1. **Client**: Web/mobile app for auction creation, bidding, viewing.  
2. **API Gateway**: Routes requests, handles auth, manages SSE.  
3. **Auction Service**: Manages auction/item CRUD.  
4. **Bidding Service**: Processes bids, ensures consistency.  
5. **Database**: Postgres for auctions/items/bids.  
6. **Queue**: Kafka for durable bid ingestion.  
7. **Cache**: Redis for real-time bid state.  
8. **Storage**: S3 for images.  
9. **Cron Job**: Closes expired auctions.

**Data Flow**:  
- **Create Auction**: Client → API Gateway → Auction Service → Postgres/S3 → Client.  
- **Place Bid**: Client → API Gateway → Kafka → Bidding Service → Postgres/Redis → SSE to clients.  
- **View Auction**: Client → API Gateway → Auction Service → Postgres/Redis → Client.

**Diagram** (verbalize):  
- Client → API Gateway → Auction/Bidding Service.  
- Bidding → Kafka → Postgres/Redis; Auction → Postgres/S3; SSE for real-time.

---

## Deep Dives
Key challenges addressing non-functional requirements.

### 1. Strong Consistency for Bids
**Challenge**: Ensure all users see the same highest bid, avoid accepting invalid bids.  
**Solution**:  
- **Pessimistic Locking**:  
  - Bidding Service locks Auction row (`SELECT FOR UPDATE`).  
  - Check `amount > highestBid`, update if valid, release lock.  
- **Redis Cache**:  
  - Store `auction:{auctionId}:highestBid` for fast checks.  
  - Update cache and DB in transaction (write-through).  
- **Kafka Ordering**:  
  - Partition by `auctionId` ensures sequential bid processing.  
- **Why?**: Locks prevent race conditions; Kafka ensures order; Redis reduces DB load.  
- **Alternative**: Optimistic locking (retries on collisions); eventual consistency (invalid bids).  
**Interview Tip**: Propose pessimistic locking, mention Redis. If probed, discuss optimistic locking or eventual consistency trade-offs.

### 2. Fault Tolerance and Durability
**Challenge**: No bids dropped, even during crashes or spikes.  
**Solution**:  
- **Kafka Queue**:  
  - Write bids to `bids` topic immediately (durable, replicated).  
  - Bidding Service consumes asynchronously, retries on failure.  
- **At-Least-Once Delivery**:  
  - Kafka acknowledges write; Bidding Service idempotently processes (check `bidId`).  
- **Buffering**: Handles spikes (e.g., 10K bids/s → 100K bids/s).  
- **Why?**: Kafka ensures durability; async processing prevents overload.  
- **Alternative**: Direct DB writes (risk loss); SQS (less throughput).  
**Interview Tip**: Lead with Kafka, emphasize durability. If probed, discuss idempotency or spike handling.

### 3. Real-Time Bid Updates
**Challenge**: Display highest bid to all clients in <1s.  
**Solution**:  
- **Server-Sent Events (SSE)**:  
  - Clients connect to `/auctions/{auctionId}/bids` via API Gateway.  
  - Bidding Service publishes new bids to Redis pub/sub (`channel:auction:{auctionId}`).  
  - API Gateway subscribes, streams updates to clients.  
- **Redis Cache**: Stores `highestBid` for low-latency reads.  
- **Fallback**: Poll `/auctions/{auctionId}` every 5s if SSE fails.  
- **Why?**: SSE scales for low fan-out; Redis pub/sub is fast; polling as backup.  
- **Alternative**: WebSocket (heavier); long polling (inefficient).  
**Interview Tip**: Suggest SSE with pub/sub, mention fallback. If probed, compare WebSocket or discuss connection limits.

### 4. Scalability for 10M Concurrent Auctions
**Challenge**: Handle 10M auctions, ~10K bids/s (~1B bids/day).  
**Solution**:  
- **Kafka**:  
  - Partition `bids` topic by `auctionId` (~10K partitions).  
  - Handles 10K–100K bids/s with replication.  
- **Bidding Service**:  
  - Horizontal scaling (~100 servers, auto-scale on CPU).  
  - Each consumes subset of partitions.  
- **Postgres**:  
  - Shard by `auctionId` (~100 shards, Citus or manual).  
  - ~25TB/year (1KB/auction, 0.5KB/bid × 100 bids × 520M auctions).  
  - Hot replication for availability.  
- **Redis**: Cluster mode for `highestBid` keys (~10M keys).  
- **Back-of-Envelope**:  
  - 1B bids ÷ 86,400s = ~10K bids/s.  
  - 520M auctions × 51KB = 25TB/year.  
- **Why?**: Sharding/partitions distribute load; auto-scaling handles spikes.  
- **Alternative**: Monolith DB (write bottleneck); no cache (slow reads).  
**Interview Tip**: Estimate QPS/storage, propose sharding. If probed, discuss partitioning or caching.

---

## Interview Checklist
- **Structure** (35 min):  
  1. Requirements: Functional/non-functional (2 min).  
  2. API: Endpoints, SSE (2 min).  
  3. Database: Postgres/Kafka/Redis (2 min).  
  4. High-Level: Microservices, flow (5 min).  
  5. Deep Dives: Consistency, durability, real-time, scale (15–20 min).  
- **Whiteboard** (verbalize):  
  - Draw: Client → API Gateway → Auction/Bidding Service → Kafka/Postgres/Redis.  
  - Show: Auction creation (Postgres), bid (Kafka → DB), real-time (SSE).  
- **Avoid Pitfalls**:  
  - Don’t use eventual consistency for bids (invalid bids).  
  - Don’t skip queue (drops bids).  
  - Don’t poll for real-time (inefficient).  
  - Don’t store images in DB (costly).  
- **Trade-Offs**:  
  - Pessimistic vs. optimistic locking: Safety vs. throughput.  
  - SSE vs. WebSocket: Simplicity vs. flexibility.  
  - Kafka vs. direct writes: Durability vs. latency.  
- **Mid-Level Expectations**:  
  - Define APIs, basic microservices.  
  - Propose Postgres for bids, Redis for real-time.  
  - Address consistency naively (e.g., DB locks).  
  - Reason through probes; expect guidance.

---

This summary is lean and tailored for your preparation. Send the next system design, and I’ll maintain the format!
