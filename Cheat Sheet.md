# **System Design Cheat Sheet**

## **► Read-Heavy Systems**  
Focus: *Scalability, low latency, efficient reads*

### 1. Design a URL Shortener (e.g., Bitly)  
- Key generation strategies, collision avoidance  
- Persistent DB storage  
- High-traffic: Use caching & DB sharding

### 2. Design an Image Hosting Service  
- Use object storage (S3, GCS) + CDN for delivery  
- Implement image deduplication & resizing strategies

### 3. Design a Social Media Platform (e.g., Twitter, Facebook)  
- Data models: posts, timelines, relationships (followers/friends)  
- Denormalized storage and horizontal sharding

### 4. Design a NewsFeed System *(Hard)*  
- Push vs Pull models  
- Fanout on Write vs Fanout on Read  
- Caching, pagination, and ranking algorithms

## **► Write-Heavy Systems**  
Focus: *Durability, ingestion throughput, consistency*

### 5. Design a Rate Limiter  
- Use token bucket or leaky bucket algorithms  
- Redis-backed counters with TTL logic

### 6. Design a Log Collection and Analysis System  
- Kafka for log ingestion  
- ELK stack (Elasticsearch, Logstash, Kibana) for processing  
- Partitioning, buffering, real-time querying

### 7. Design a Voting System  
- Ensure idempotency and fraud prevention  
- Result aggregation strategies  
- Real-time vs eventual consistency for vote counts

### 8. Design a Trending Topics System  
- Use count-min sketch or approximate counters  
- Sliding window aggregation & ranking logic

## **► Strong Consistency Systems**  
Focus: *Transactional integrity, consistency under failure*

### 9. Design an Online Ticket Booking System  
- Prevent race conditions with locking or optimistic concurrency  
- Handle seat reservation and payment atomicity

### 10. Design an E-Commerce Website (e.g., Amazon)  
- Components: product catalog, cart service, order flow  
- Ensure DB consistency and checkout idempotency

### 11. Design an Online Messaging App (e.g., WhatsApp, Slack)  
- Message queues, retries, delivery receipts  
- Offline storage, real-time notifications, scaling chat infra

### 12. Design a Task Management Tool  
- CRUD APIs with user auth  
- Task assignment, background jobs, status tracking  
- Audit trails for history and compliance

## **► Scheduler Services**  
Focus: *Timing accuracy, retry logic, scalability*

### 13. Design a Web Crawler  
- Crawling strategies: BFS vs DFS  
- Politeness rules, rate limits  
- Distributed queues & duplicate URL filtering

### 14. Design a Task Scheduler  
- Cron-based triggers and job queues  
- Retry and backoff logic  
- Priority queues and deduplication

### 15. Design a Real-Time Notification System  
- Push vs Polling, use of webhooks  
- Device token management  
- Scalable delivery infrastructure

## **► Trie / Proximity Systems**  
Focus: *Low latency retrieval, intelligent suggestions*

### 16. Design a Search Autocomplete System  
- Trie or Ternary Search Tree with frequency ranking  
- Debouncing, caching, typo-tolerance

### 17. Design a Ride-Sharing App (e.g., Uber, Lyft)  
- Matchmaking engine with real-time location tracking  
- ETA calculation, surge pricing algorithms  
- Scalable DB schema for users and rides