# System Design: Local Business Review Site Like Yelp

This is a concise, interview-ready summary for designing a Yelp-like platform, optimized for a 35-minute interview. It covers all essential points to excel, omitting unnecessary details (e.g., full schemas) while aligning with your 20+ system design cheatsheet for quick study.

---

## Overview
**Name**: Business Review Service (Yelp-like)  
**Purpose**: Platform for users to search, view, and review local businesses.  
**Scope**: Core features (search businesses, view businesses/reviews, leave reviews). Excludes admin tasks, maps, recommendations.

---

## Functional Requirements
- **Search Businesses**: Users can search by name, location (lat/long), category.  
- **View Businesses**: Users can view business details and reviews.  
- **Leave Reviews**: Users can submit a 1–5 star rating and optional text review (one review per user per business).

---

## Non-Functional Requirements
- **Latency**: Search operations <500ms.  
- **Availability**: High availability, eventual consistency acceptable.  
- **Scalability**: Support 100M daily active users, 10M businesses (~1B reviews).  
- **Constraint**: One review per user per business.

---

## API Design
RESTful APIs with JWT for authentication.

1. **Search Businesses**:  
   - `GET /businesses?query={string}&lat={float}&long={float}&category={string}&page={int}&limit={int}`  
   - Returns: `{ businesses: [{ id, name, location, category, avgRating }] }`  
   - Purpose: Search with filters, paginated results.  

2. **View Business**:  
   - `GET /businesses/{businessId}`  
   - Returns: `{ id, name, location, category, avgRating, reviews: [{ id, userId, rating, text, timestamp }] }`  
   - Purpose: Fetch business details and reviews.  

3. **Leave Review**:  
   - `POST /businesses/{businessId}/reviews`  
   - Body: `{ rating: int, text: string? }`  
   - Returns: `{ status: string }`  
   - Purpose: Submit a review, enforce one-per-user constraint.

---

## Database
**Choices**:  
- **Data**: Postgres with PostGIS for geospatial queries, business/review storage.  
- **Search**: Elasticsearch for fast text/geo searches.  
- **Cache**: Redis for search results, ratings.  
**Key Entities**:  
- **Business**: businessId, name, location (lat, long), category, reviewCount, ratingSum.  
- **Review**: reviewId, businessId, userId, rating, text, timestamp.  
**Indexing**:  
- Postgres:  
  - PK `businessId` (Business), `reviewId` (Review).  
  - Unique index `(businessId, userId)` for one-review constraint.  
  - Geospatial index (PostGIS) on `location`.  
- Elasticsearch: Index `name`, `category`, `location` (geo_point), `avgRating`.  
- Redis: Key `business:{businessId}:rating` for cached avgRating.

---

## High-Level Design
**Architecture**: Microservices for modularity; monolith viable for simplicity.  
**Components**:  
1. **Client**: Web/mobile app for search/view/review.  
2. **API Gateway**: Routes requests, handles auth.  
3. **Business Service**: Manages business data, search queries.  
4. **Review Service**: Handles review creation, rating updates.  
5. **Database**: Postgres for persistent storage.  
6. **Search Engine**: Elasticsearch for efficient searches.  
7. **Cache**: Redis for ratings, search results.

**Data Flow**:  
- **Search**: Client → API Gateway → Business Service → Elasticsearch/Redis → Client.  
- **View**: Client → API Gateway → Business Service → Postgres/Redis → Client.  
- **Review**: Client → API Gateway → Review Service → Postgres → Client.  

**Diagram** (verbalize):  
- Client → API Gateway → Business/Review Service.  
- Business → Elasticsearch/Redis/Postgres; Review → Postgres.

---

## Deep Dives
Key challenges addressing non-functional requirements.

### 1. Efficient Average Rating Calculation
**Challenge**: Compute avgRating for search without slow queries.  
**Solution**:  
- **Denormalized Counters**:  
  - Store `reviewCount`, `ratingSum` in Business table.  
  - On review: `UPDATE businesses SET ratingSum = ratingSum + :rating, reviewCount = reviewCount + 1 WHERE businessId = :id`.  
  - Compute `avgRating = ratingSum / reviewCount` on read.  
- **Redis Cache**: Store `avgRating` in Redis (`business:{businessId}:rating`, TTL 1h).  
- **Async Updates**: If high contention, use Redis `INCR` for counters, sync to Postgres periodically.  
- **Why?**: Denormalization avoids joins; Redis speeds up reads.  
- **Alternative**: Compute on-the-fly (slow); message queue (overkill for ~1 write/s).  
**Interview Tip**: Propose denormalization, mention Redis. If probed, discuss async updates or contention.

### 2. One Review per User per Business
**Challenge**: Enforce constraint to prevent multiple reviews.  
**Solution**:  
- **Unique Index**: Postgres unique constraint on `(businessId, userId)` in Review table.  
- **Application Check**: Review Service checks for existing review before insert.  
- **Transaction**:  
  - Begin transaction: Check review exists, insert new review, update Business counters.  
  - Rollback on violation.  
- **Why?**: DB constraint ensures consistency; app check reduces DB load.  
- **Alternative**: App-only check (race conditions); distributed lock (complex).  
**Interview Tip**: Lead with unique index, mention transaction. If probed, discuss race conditions.

### 3. Efficient Search (<500ms)
**Challenge**: Fast search by name, location, category on 10M businesses.  
**Solution**:  
- **Elasticsearch**:  
  - Index `name` (text), `category` (keyword), `location` (geo_point), `avgRating`.  
  - Query: `bool` filter for category, `match` for name, `geo_distance` for location.  
  - Cache top queries in Redis (`search:{query}:results`, TTL 10m).  
- **Geospatial Indexing**: Elasticsearch handles geo queries internally (quadtrees).  
- **Second-Pass Filtering**: Post-filter by exact Haversine distance if needed.  
- **Why?**: Elasticsearch for fast full-text/geo search; cache for repeated queries.  
- **Alternative**: Postgres/PostGIS (slower for text); geohashing (less precise).  
**Interview Tip**: Propose Elasticsearch, mention caching. If probed, discuss quadtrees vs. geohashing.

### 4. Search by Location Names
**Challenge**: Support searches by city/neighborhood (e.g., “Pizza in NYC”).  
**Solution**:  
- **Location Table**: Postgres table (`name`, `type`, `polygon` via PostGIS).  
  - Example: `("NYC", "city", POLYGON(...))`.  
  - Index `name` for fast lookup.  
- **Precomputed Locations**:  
  - On business creation, compute enclosing locations (e.g., `["NYC", "Manhattan"]`).  
  - Store in Business table and Elasticsearch (`location_names` keyword).  
- **Query Flow**:  
  - Map “NYC” to location_names filter in Elasticsearch.  
  - Combine with text/category filters.  
- **Why?**: Precomputation avoids runtime polygon checks; Elasticsearch for fast filtering.  
- **Alternative**: Runtime polygon queries (slow); geohashing (imprecise shapes).  
**Interview Tip**: Describe precomputation, mention polygons. If probed, discuss data sources (GeoJSON).

---

## Interview Checklist
- **Structure** (35 min):  
  1. Requirements: Functional/non-functional (2 min).  
  2. API: Endpoints (2 min).  
  3. Database: Postgres/Elasticsearch/Redis (2 min).  
  4. High-Level: Microservices, flow (5 min).  
  5. Deep Dives: Ratings, constraint, search, location names (15–20 min).  
- **Whiteboard** (verbalize):  
  - Draw: Client → API Gateway → Business/Review Service → Postgres/Elasticsearch/Redis.  
  - Show: Search (Elasticsearch), view (Postgres), review (constraint).  
- **Avoid Pitfalls**:  
  - Don’t compute ratings on-the-fly (slow).  
  - Don’t skip unique constraint (spam risk).  
  - Don’t use basic SQL for search (inefficient).  
  - Don’t ignore polygons for location names (imprecise).  
- **Trade-Offs**:  
  - Elasticsearch vs. Postgres: Speed vs. simplicity.  
  - Denormalization vs. joins: Performance vs. consistency.  
  - Cache vs. live query: Latency vs. freshness.  
- **Mid-Level Expectations**:  
  - Define APIs, basic data model.  
  - Propose Postgres for storage, Elasticsearch for search.  
  - Address ratings/constraint naively (e.g., DB query).  
  - Reason through probes; expect guidance.

---

This summary is lean and tailored for your preparation. Send the next system design, and I’ll maintain the format!
