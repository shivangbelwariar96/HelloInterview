<img width="1134" alt="Screenshot 2025-04-13 at 4 45 40 PM" src="https://github.com/user-attachments/assets/cbb5b181-f9f9-41f3-95ab-b92d540954e6" />


# System Design: Stock Trading Platform Like Robinhood

This is a concise, interview-ready summary for designing a stock trading platform like Robinhood, optimized for a 35-minute interview. It addresses all critical points to excel, aligning with your 20+ system design cheatsheet for efficient study. The design is "hard" due to high consistency, low latency (<200ms), and minimizing exchange connections while handling 100M trades/day.

---

## Overview
**Name**: Stock Trading Platform (Robinhood-like)  
**Purpose**: Enable users to view live stock prices and manage orders (market/limit, create/cancel).  
**Scope**: Core features (live prices, order management for stocks). Excludes ETFs, options, crypto, order book, after-hours trading.

---

## Functional Requirements
- **Live Prices**: Users view real-time stock prices.  
- **Order Management**: Users create/cancel market/limit orders, view their orders.

---

## Non-Functional Requirements
- **Consistency**: Strong consistency for order management (accurate order state).  
- **Latency**: Price updates and order placement <200ms.  
- **Scalability**: 20M DAU, ~100M trades/day (~1K trades/s), ~10K symbols.  
- **Efficiency**: Minimize exchange API connections (costly).  
- **Reliability**: Handle failures without losing order state.

---

## Scale Estimations
- **Users**: 20M DAU, ~5 trades/user/day ≈ 100M trades/day ≈ 1K trades/s (peak 10K/s).  
- **Symbols**: ~10K stocks, ~1K updates/s/symbol (peak 10K/s).  
- **Orders**: 100M orders/day × 50B/order ≈ 5GB/day, ~2TB/year.  
- **Price Updates**: 10K symbols × 1K updates/s ≈ 10M updates/s.  
**Insight**: High read (prices), moderate write (orders), consistency critical.

---

## API Design
RESTful APIs with JWT for authentication.

1. **Get Symbol Price**:  
   - `GET /symbol/{name}`  
   - Header: `Authorization: Bearer <JWT>`  
   - Returns: `{ symbol: string, priceInCents: long, timestamp: long }`  
   - Purpose: Fetch live stock price.  

2. **Create Order**:  
   - `POST /order`  
   - Header: `Authorization: Bearer <JWT>`  
   - Body: `{ symbol: string, position: "buy"|"sell", type: "market"|"limit", priceInCents: long, numShares: int }`  
   - Returns: `{ orderId: string, status: string }`  
   - Purpose: Place market/limit order.  

3. **Cancel Order**:  
   - `DELETE /order/{orderId}`  
   - Header: `Authorization: Bearer <JWT>`  
   - Returns: `{ ok: true }`  
   - Purpose: Cancel pending order.  

4. **List Orders**:  
   - `GET /orders`  
   - Header: `Authorization: Bearer <JWT>`  
   - Query: `?page=int&size=int`  
   - Returns: `{ orders: [{ orderId: string, symbol: string, status: string, ... }], nextPage: int }`  
   - Purpose: View user’s orders (paginated).

---

## Database
**Choices**:  
- **Data**: Postgres for orders (ACID, consistency).  
- **Cache**: Redis for prices (low latency).  
- **Metadata**: DynamoDB for external order mapping (fast lookups).  
- **Queue**: Kafka for trade feed (durability).  
**Key Entities**:  
- **User**: userId, name, funds.  
- **Symbol**: symbol, name.  
- **Order**: orderId, userId, symbol, type, position, priceInCents, numShares, status, externalOrderId.  
- **Trade**: tradeId, symbol, priceInCents, numShares, orderId.  
**Indexing**:  
- Postgres:  
  - PK: `orderId`.  
  - Index: `userId` for listing orders.  
  - Partition: `userId` for scalability.  
- Redis:  
  - `symbol:price:{symbol}` (hash, price/timestamp).  
  - `symbol:pubsub:{symbol}` (pub/sub channel).  
- DynamoDB:  
  - PK: `externalOrderId`, value: `{ orderId, userId }`.  
- Kafka: Topic `trades`, partitioned by `symbol`.

---

## High-Level Design
**Architecture**: Microservices with pub/sub for prices, partitioned DB for orders.  
**Components**:  
1. **Client**: App for price viewing, order management.  
2. **API Gateway**: Routes requests, handles auth/rate limiting.  
3. **Symbol Service**: Manages live prices, subscriptions.  
4. **Order Service**: Creates/cancels orders, syncs with exchange.  
5. **Trade Processor**: Processes exchange trade feed, updates orders.  
6. **Database**: Postgres for orders.  
7. **Cache**: Redis for prices/subscriptions.  
8. **Metadata Store**: DynamoDB for external order lookups.  
9. **Queue**: Kafka for trade feed buffering.  
10. **Exchange**: External API (order placement, trade feed).

**Data Flow**:  
- **Price Update**: Exchange → Kafka → Trade Processor → Redis → Symbol Service → Client (SSE).  
- **Order Create**: Client → API Gateway → Order Service → Postgres → Exchange → DynamoDB → Client.  
- **Order Cancel**: Client → API Gateway → Order Service → Postgres → Exchange → Client.  
- **Order List**: Client → API Gateway → Order Service → Postgres → Client.

**Diagram** (verbalize):  
- Client → API Gateway → Symbol/Order Service.  
- Exchange → Kafka → Trade Processor → Redis/DynamoDB/Postgres → Symbol Service → Client.

---

## Deep Dives
Key challenges addressing non-functional requirements.

### 1. Scaling Live Price Updates
**Challenge**: 10M price updates/s to millions of users, <200ms latency.  
**Solution**:  
- **Pub/Sub**:  
  - Redis pub/sub: `symbol:pubsub:{symbol}` per symbol.  
  - Symbol Service subscribes to symbols users care about.  
  - Trade Processor publishes updates to Redis.  
- **Fan-Out**:  
  - Symbol Service maps `symbol → Set<userId>`.  
  - Pushes updates via SSE to clients (HTTP/2).  
- **Scaling**:  
  - Shard Symbol Service by `userId` (~100 servers).  
  - Redis cluster (~10 shards, 1M ops/s/shard).  
  - Load balance SSE with sticky sessions (consistent hashing).  
- **Optimization**:  
  - Batch updates (~10ms intervals).  
  - Drop updates for inactive clients (heartbeat).  
- **Back-of-Envelope**:  
  - 10M updates/s ÷ 10 shards = 1M updates/s/shard.  
  - 20M users × 10 symbols × 1KB/update = 200MB/s bandwidth.  
- **Why?**: Redis pub/sub scales subscriptions; SSE is lightweight; sharding distributes load.  
- **Alternative**: Polling (high latency); WebSocket (overhead).  
**Interview Tip**: Propose Redis pub/sub, sketch SSE flow. If probed, discuss bandwidth or client disconnects.

### 2. Tracking Order Updates
**Challenge**: Sync order state with exchange trade feed efficiently.  
**Solution**:  
- **Metadata Store**:  
  - DynamoDB: `externalOrderId → { orderId, userId }` (O(1) lookup).  
  - Populated on order creation after exchange response.  
- **Trade Processing**:  
  - Kafka topic `trades` buffers exchange feed (partitioned by `symbol`).  
  - Trade Processor consumes, lookups `externalOrderId` in DynamoDB.  
  - Updates Postgres order via `userId`, `orderId` (sharded).  
- **State Management**:  
  - Order states: `pending`, `submitted`, `filled`, `cancelled`, `failed`.  
  - Partial fills: Update `numShares` remaining.  
- **Back-of-Envelope**:  
  - 10K trades/s × 1KB = 10MB/s (Kafka handles).  
  - 10K lookups/s (DynamoDB scales).  
  - 10K writes/s ÷ 100 shards = 100 writes/s/shard (Postgres handles).  
- **Why?**: DynamoDB enables fast mapping; Kafka ensures durability; Postgres maintains consistency.  
- **Alternative**: Postgres index on `externalOrderId` (slow across shards); no metadata (full scan).  
**Interview Tip**: Lead with DynamoDB, explain trade flow. If probed, discuss partial fills or Kafka lag.

### 3. Ensuring Order Consistency
**Challenge**: Maintain consistent order state despite failures, <200ms latency.  
**Solution**:  
- **Create Workflow**:  
  1. Write `pending` order to Postgres (userId shard).  
  2. Submit to exchange (synchronous, get `externalOrderId`).  
  3. Write `externalOrderId` to DynamoDB, update Postgres to `submitted`.  
  4. Respond to client.  
- **Cancel Workflow**:  
  1. Update Postgres to `pending_cancel`.  
  2. Cancel on exchange (synchronous).  
  3. Update Postgres to `cancelled`.  
  4. Respond to client.  
- **Failure Handling**:  
  - **Store Failure**: Fail fast, client retries.  
  - **Exchange Failure**: Mark order `failed` in Postgres.  
  - **Post-Exchange Failure**: Cleanup job scans `pending`/`pending_cancel` orders, queries exchange via `clientOrderId`, updates state.  
- **Cleanup Job**:  
  - Runs every 1m, checks orders stuck >5m.  
  - Uses `clientOrderId` (passed to exchange) for status.  
- **Back-of-Envelope**:  
  - 10K orders/s × 3 writes = 30K writes/s (Postgres/DynamoDB scale).  
  - Cleanup: 1K stuck orders/day × 1s/query = 1K s/day (negligible).  
- **Why?**: Postgres ensures ACID; DynamoDB decouples lookups; cleanup resolves inconsistencies.  
- **Alternative**: No cleanup (stale orders); queue-based (higher latency).  
**Interview Tip**: Walk through create flow, highlight cleanup. If probed, discuss `clientOrderId` or transaction isolation.

---

## Interview Checklist
- **Structure** (35 min):  
  1. Requirements: Functional/non-functional (2 min).  
  2. Estimations: TPS, storage (2 min).  
  3. API: Endpoints (2 min).  
  4. Database: Postgres/Redis/DynamoDB (2 min).  
  5. High-Level: Microservices, flow (5 min).  
  6. Deep Dives: Prices, orders, consistency (18–22 min).  
- **Whiteboard** (verbalize):  
  - Draw: Client → API Gateway → Symbol/Order → Postgres/Redis/DynamoDB; Exchange → Kafka → Trade Processor.  
  - Show: Price (Kafka → Redis → SSE), order (Postgres → Exchange → DynamoDB).  
- **Avoid Pitfalls**:  
  - Don’t poll for prices (high latency).  
  - Don’t skip metadata store (slow order updates).  
  - Don’t ignore failures (inconsistent orders).  
  - Don’t overuse exchange connections (costly).  
- **Trade-Offs**:  
  - Redis vs. Kafka: Speed vs. durability for prices.  
  - Postgres vs. DynamoDB: ACID vs. scalability.  
  - SSE vs. WebSocket: Simplicity vs. bidirectionality.  
- **Mid-Level Expectations**:  
  - Define APIs, basic Symbol/Order Service, Postgres for orders.  
  - Propose naive price updates (e.g., polling), pivot to pub/sub with hints.  
  - Address consistency naively (e.g., DB writes), miss cleanup.  
  - Reason through probes; expect guidance.

---

This summary is streamlined for your preparation. Send the next system design, and I’ll maintain the format!
