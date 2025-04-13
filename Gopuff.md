# System Design: Local Delivery Service Like Gopuff

This is a streamlined, interview-focused summary for designing a local delivery service like Gopuff, covering all critical points for a 35-minute interview. It’s concise, omitting unnecessary details (e.g., exhaustive schemas) but including everything needed to excel. The format aligns with your 20+ system design cheatsheet for quick study.

---

## Overview
**Name**: Local Delivery Service (Gopuff-like)  
**Purpose**: Rapid delivery of convenience goods from micro-fulfillment centers (DCs), focusing on item availability and ordering.  
**Scope**: Core features (query availability, place orders). Excludes payments, driver routing, search/catalog, cancellations/returns.

---

## Functional Requirements
- **Query Availability**: Customers check item availability by location, deliverable in 1 hour (aggregates nearby DCs).  
- **Place Orders**: Customers order multiple items simultaneously without double-booking inventory.

---

## Non-Functional Requirements
- **Performance**: Availability queries <100ms (e.g., for search).  
- **Consistency**: Strong consistency for orders (no double-booking).  
- **Scalability**: Support 10,000 DCs, 100,000 items, ~10M orders/day (~115 orders/second).  
- **Availability**: Prioritize uptime for availability queries, even if slightly stale.

---

## API Design
RESTful APIs for core functions, using location data and secure user sessions.

1. **Query Availability**:  
   - `GET /availability?lat={float}&long={float}&keyword={optional}&page={int}&limit={int}`  
   - Returns: `{ items: [{ itemId: string, name: string, description: string, quantity: int }] }`  
   - Purpose: List items available within 1 hour.  

2. **Place Order**:  
   - `POST /orders`  
   - Body: `{ items: [{ itemId: string, quantity: int }], lat: float, long: float }`  
   - Returns: `{ orderId: string, status: string }`  
   - Purpose: Create order, reserve inventory atomically.  
   - Note: User ID from JWT/session for security.

---

## Database
**Choice**: SQL (e.g., Postgres) for strong consistency and joins.  
**Key Entities**:  
- **Item**: Catalog (ID, name, description).  
- **Inventory**: Physical stock (item ID, DC ID, quantity).  
- **DistributionCenter (DC)**: Location (ID, lat, long).  
- **Order**: User order (ID, user ID, status, timestamp).  
- **OrderItem**: Ordered items (order ID, item ID, quantity).  
**Indexing**:  
- Inventory: `dcId`, `itemId` for fast availability.  
- Orders: `userId`, `orderId` for history/status.  
- DCs: `lat`, `long` for proximity.

---

## High-Level Design
**Architecture**: Monolith for simplicity; microservices optional for scale.  
**Components**:  
1. **Client**: Mobile/web app for browsing/ordering.  
2. **Availability Service**: Aggregates inventory for queries.  
3. **Order Service**: Processes orders, ensures atomic updates.  
4. **Nearby Service**: Finds DCs within 1-hour delivery.  
5. **Database**: Postgres (leader for writes, replicas for reads).  
6. **Cache**: Redis for availability results.  

**Data Flow**:  
- **Availability**: Client → Availability Service → Nearby Service → Cache/Database → Client.  
- **Order**: Client → Order Service → Nearby Service → Database (transaction) → Client.  

**Diagram** (verbalize):  
- Client → Availability/Order Service.  
- Availability: Nearby Service → Cache/DB → Response.  
- Order: Locks inventory → Postgres leader write.  
- Redis caches availability; replicas scale reads.

---

## Deep Dives
Key challenges with clear solutions to address non-functional requirements.

### 1. Accurate Nearby DCs (1-Hour Delivery)
**Challenge**: Distance-based DC selection ignores drive time, traffic, barriers.  
**Solution**:  
- **Travel Time Service**:  
  - Precompute drive times from DCs to location grids (0.01-degree squares) using Google Maps API.  
  - Cache in Redis: `dc:{lat}:{long}` → DC IDs within 1 hour.  
  - Refresh daily or on traffic changes (e.g., rush hour).  
- **Fallback**: Haversine formula (~10-mile threshold) if API fails.  
- **Why?**: Drive time ensures 1-hour delivery; caching avoids API latency.  
- **Alternative**: Real-time API (slow, costly); Euclidean distance (inaccurate).  
**Interview Tip**: Propose precomputed Redis cache, mention Maps API. If probed, explain refresh or fallback.

### 2. Fast Availability Lookups (<100ms)
**Challenge**: ~20,000 queries/second (10M orders/day × 10 pages × 5% conversion) overloads DB.  
**Solution**:  
- **Redis Cache**:  
  - Store availability: `avail:{lat}:{long}:{itemId}` → quantity.  
  - Refresh every 5–10s from DB.  
  - Handles ~90% queries, reducing DB to <2,000 queries/second.  
- **Read Replicas**: Scale Postgres reads regionally.  
- **Precomputed DCs**: Cache DCs per location grid for speed.  
- **Why?**: Cache ensures <100ms; replicas handle misses.  
- **Alternative**: DB-only (slow); Elasticsearch (overkill).  
**Interview Tip**: Lead with Redis, estimate QPS (~20K). If probed, discuss refresh or sharding.

### 3. Strong Consistency for Orders
**Challenge**: Prevent double-booking inventory.  
**Solution**:  
- **Atomic Transactions**:  
  - Order Service uses Postgres leader:  
    1. Lock inventory (`SELECT FOR UPDATE` on `itemId`, `dcId`).  
    2. Verify quantity > 0.  
    3. Decrement inventory, insert order/orderItems.  
    4. Commit or rollback if unavailable.  
- **Regional Sharding**: Shard by zip code prefix to reduce contention.  
- **Error Handling**: Return specific errors (e.g., “Item X unavailable”).  
- **Why?**: ACID ensures no double-booking; sharding scales writes.  
- **Alternative**: Distributed locks (complex); eventual consistency (double-books).  
**Interview Tip**: Emphasize transactions. If probed, mention sharding or errors. Avoid distributed systems unless asked.

### 4. Scalability (10M Orders/Day)
**Challenge**: Handle ~115 orders/second, 20,000 availability queries/second.  
**Solution**:  
- **Horizontal Scaling**:  
  - Availability Service: Auto-scale stateless instances.  
  - Order Service: Scale with sharded Postgres.  
  - Nearby Service: Stateless, scales easily.  
- **Caching**: Redis cluster for availability (~90% hit rate).  
- **Database**: Replicas for reads; sharded leader for writes (~115 writes/second).  
- **Why?**: Cache/replicas handle reads; sharding supports writes.  
- **Alternative**: Single DB (bottleneck); microservices (overkill).  
**Interview Tip**: Highlight cache/replicas, estimate load. If probed, mention sharding.

---

## Interview Checklist
- **Structure** (35 min):  
  1. Requirements: Functional/non-functional (2 min).  
  2. API: Endpoints (2 min).  
  3. Database: SQL, entities (2 min).  
  4. High-Level: Monolith, flow (5 min).  
  5. Deep Dives: DCs, availability, consistency, scale (15–20 min).  
- **Whiteboard** (verbalize):  
  - Draw: Client → Availability/Order Service → Nearby Service → Postgres, Redis.  
  - Show: Availability (cache → DB), order (lock → write).  
- **Avoid Pitfalls**:  
  - Don’t use eventual consistency for orders (double-booking).  
  - Don’t skip cache (DB overload).  
  - Don’t pass `userId` in body (use JWT).  
  - Don’t overcomplicate with microservices.  
- **Trade-Offs**:  
  - Cache vs. DB: Speed vs. accuracy.  
  - Transactions vs. locks: Simplicity vs. flexibility.  
  - Precomputed vs. real-time DCs: Speed vs. precision.  
- **Mid-Level Expectations**:  
  - Define APIs, data model.  
  - Propose SQL transactions for orders, cache for availability.  
  - Explain Nearby Service.  
  - Reason through probes; expect interviewer guidance.

---

This summary is lean, covering all key points for a Gopuff-like design, optimized for your study. Send the next system design for consistent formatting!
