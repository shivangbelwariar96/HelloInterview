# System Design: Ticket Booking Site Like Ticketmaster

This is a concise, interview-ready summary for designing a ticket booking site like Ticketmaster, tailored for a 35-minute interview. It covers all essential points to excel, omitting extraneous details (e.g., full DB schemas) while aligning with your 20+ system design cheatsheet for quick study.

---

## Overview
**Name**: Ticket Booking Service (Ticketmaster-like)  
**Purpose**: Online platform for viewing, searching, and booking tickets to live events (concerts, sports, theater).  
**Scope**: Core features (view events, search events, book tickets). Excludes viewing booked events, admin event creation, dynamic pricing.

---

## Functional Requirements
- **View Events**: Users can view event details and seat availability.  
- **Search Events**: Users can search events by keyword, date, etc.  
- **Book Tickets**: Users can book tickets without double-booking.

---

## Non-Functional Requirements
- **Consistency**: Strong consistency for bookings (no double-booking).  
- **Availability**: High availability for view/search (eventual consistency OK).  
- **Scalability**: Handle 10M users for a single event (~100K QPS peak).  
- **Performance**: Search latency <500ms; view latency <200ms.  
- **Read-Heavy**: 100:1 read-to-write ratio.

---

## API Design
RESTful APIs for core functions, using JWT for user authentication.

1. **View Event**:  
   - `GET /events/{eventId}`  
   - Returns: `{ event: { id, name, date, description, performer: { id, name }, venue: { id, name, address, seatMap }, tickets: [{ id, seat, status, price }] }`  
   - Purpose: Fetch event details and seat availability.  

2. **Search Events**:  
   - `GET /events/search?keyword={string}&start={date}&end={date}&page={int}&limit={int}`  
   - Returns: `{ events: [{ id, name, date, performer, venue }] }`  
   - Purpose: List matching events with pagination.  

3. **Reserve Tickets**:  
   - `POST /reservations`  
   - Body: `{ eventId: string, ticketIds: string[] }`  
   - Returns: `{ reservationId: string, expiresAt: timestamp }`  
   - Purpose: Lock tickets for checkout.  

4. **Confirm Booking**:  
   - `POST /bookings/{reservationId}`  
   - Body: `{ paymentDetails: object }`  
   - Returns: `{ bookingId: string, status: string }`  
   - Purpose: Finalize purchase, mark tickets as sold.

---

## Database
**Choices**:  
- **Events/Search**: Elasticsearch for fast, flexible search.  
- **Bookings/Tickets**: Postgres for ACID transactions.  
- **Cache**: Redis for locks and event data.  
**Key Entities**:  
- **Event**: ID, name, date, description, performer ID, venue ID.  
- **Performer**: ID, name, description.  
- **Venue**: ID, name, address, seat map.  
- **Ticket**: ID, event ID, seat (section, row, number), status (available/reserved/sold), price.  
- **Booking**: ID, user ID, reservation ID, ticket IDs, status (in-progress/confirmed), total price.  
**Indexing**:  
- Postgres: `eventId`, `ticketId` for bookings; `status` for availability.  
- Elasticsearch: Text fields (name, description) for search.

---

## High-Level Design
**Architecture**: Microservices for modularity; monolith optional for simplicity.  
**Components**:  
1. **Client**: Web/mobile app for browsing/booking.  
2. **API Gateway**: Routes requests, handles auth/rate limiting.  
3. **Event Service**: Manages event views, caches data.  
4. **Search Service**: Handles event searches via Elasticsearch.  
5. **Booking Service**: Processes reservations/bookings, ensures consistency.  
6. **Database**: Postgres for tickets/bookings; Elasticsearch for events.  
7. **Cache**: Redis for locks and event caching.  
8. **Payment Processor**: Stripe for payments.  
9. **Queue Service**: Manages highOctober 20, 3:29 PM high-demand events.

**Data Flow**:  
- **View**: Client → Event Service → Redis/Postgres → Client.  
- **Search**: Client → Search Service → Elasticsearch → Client.  
- **Booking**: Client → Booking Service → Redis (lock) → Postgres (transaction) → Stripe → Client.  

**Diagram** (verbalize):  
- Client → API Gateway → Event/Search/Booking Service.  
- Event Service → Redis/Postgres; Search → Elasticsearch.  
- Booking Service → Redis (lock) → Postgres → Stripe.  
- Queue Service for high-demand events.

---

## Deep Dives
Key challenges addressing non-functional requirements.

### 1. Preventing Double-Booking
**Challenge**: Ensure no two users book the same ticket.  
**Solution**:  
- **Distributed Lock**:  
  - Use Redis to lock ticket IDs (`SETNX`, TTL 10 min).  
  - Reserve: Lock tickets, create reservation.  
  - Confirm: Validate lock, update Postgres (ticket status → sold, booking → confirmed).  
  - Release: TTL expires or explicit release on abandon.  
- **Postgres Transaction**:  
  - `SELECT FOR UPDATE` on ticket IDs.  
  - Update status, insert booking atomically.  
- **Why?**: Redis ensures fast locking; Postgres guarantees consistency.  
- **Alternative**: Status field + cron (slower, complex cleanup).  
**Interview Tip**: Propose Redis lock, mention Postgres fallback. If probed, discuss TTL or Redlock.

### 2. Scaling View API (10M Users)
**Challenge**: Handle ~100K QPS for popular events.  
**Solution**:  
- **Caching**:  
  - Redis cache for event details (`event:{id}`), TTL 1 min.  
  - Cache hit ~90%, reducing Postgres load.  
- **Load Balancing**: API Gateway distributes to Event Service instances.  
- **Horizontal Scaling**: Auto-scale Event Service based on CPU.  
- **CDN**: Static assets (images) via CloudFront.  
- **Why?**: Cache reduces DB hits; scaling handles spikes.  
- **Alternative**: No cache (DB overload); single server (bottleneck).  
**Interview Tip**: Lead with Redis cache, estimate QPS. If probed, mention CDN or sharding.

### 3. High-Demand Events (UX)
**Challenge**: Prevent stale seat maps, ensure smooth UX during spikes.  
**Solution**:  
- **Virtual Waiting Room**:  
  - Queue users (e.g., AWS SQS) during peak load (>10K QPS).  
  - Assign time slots, limit concurrent bookings.  
- **Real-Time Updates**:  
  - WebSocket for seat map updates (ticket reserved/sold).  
  - Push only changed seats to reduce bandwidth.  
- **Why?**: Queue manages load; WebSocket keeps UI fresh.  
- **Alternative**: No queue (system crash); polling (high latency).  
**Interview Tip**: Propose queue + WebSocket. If probed, discuss queue fairness or bandwidth.

### 4. Low-Latency Search (<500ms)
**Challenge**: Fast, flexible event search with keywords, dates.  
**Solution**:  
- **Elasticsearch**:  
  - Index event name, description, performer, venue.  
  - Use inverted indices for full-text search.  
  - Query example: `name:Taylor OR description:Taylor`.  
- **Caching**: Redis for frequent queries (`search:{query}`), TTL 10s.  
- **Sharding**: Distribute Elasticsearch across nodes by event ID.  
- **Why?**: Elasticsearch handles complex queries; cache reduces load.  
- **Alternative**: Postgres LIKE (slow, full scan); no cache (high latency).  
**Interview Tip**: Lead with Elasticsearch, mention cache. If probed, explain sharding or query optimization.

---

## Interview Checklist
- **Structure** (35 min):  
  1. Requirements: Functional/non-functional (2 min).  
  2. API: Endpoints (2 min).  
  3. Database: Postgres/Elasticsearch/Redis (2 min).  
  4. High-Level: Microservices, flow (5 min).  
  5. Deep Dives: Booking, scale, UX, search (15–20 min).  
- **Whiteboard** (verbalize):  
  - Draw: Client → API Gateway → Services → Postgres/Elasticsearch/Redis.  
  - Show: View (cache), search (index), booking (lock → transaction).  
- **Avoid Pitfalls**:  
  - Don’t use eventual consistency for bookings (double-booking).  
  - Don’t skip cache (DB overload).  
  - Don’t propose slow search (e.g., SQL LIKE).  
  - Don’t ignore high-demand UX (stale UI).  
- **Trade-Offs**:  
  - Redis vs. DB lock: Speed vs. durability.  
  - Queue vs. no queue: Stability vs. complexity.  
  - Elasticsearch vs. SQL: Speed vs. simplicity.  
- **Mid-Level Expectations**:  
  - Define APIs, data model.  
  - Propose Postgres for bookings, Elasticsearch for search, Redis for locks.  
  - Solve double-booking with status field or lock.  
  - Reason through probes; expect guidance.

---

This summary is lean and comprehensive, tailored for your study. Send the next system design, and I’ll keep the format consistent!
