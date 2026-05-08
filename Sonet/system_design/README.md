# System Design Interview Prep — Prince Patel
> Complete self-study guide for FAANG-level system design interviews

---

## Two-Track Learning System

| Folder | Purpose | When to Use |
|--------|---------|-------------|
| `interview_prep/` | Concise, pattern-focused, interview-ready | Daily practice, quick review before interview |
| `deep_dive/` | Textbook depth, internals, research-level | Build real understanding, senior/staff prep |

---

## Interview Prep Track (`interview_prep/`)

### Foundation Topics
| File | Topic | Time to Read |
|------|-------|-------------|
| [00_how_to_answer.md](interview_prep/00_how_to_answer.md) | Framework for answering any SD question | 10 min |
| [01_core_concepts.md](interview_prep/01_core_concepts.md) | Scalability, Availability, CAP, ACID, BASE | 20 min |
| [02_building_blocks.md](interview_prep/02_building_blocks.md) | DNS, CDN, Load Balancer, Reverse Proxy, API GW | 20 min |
| [03_databases.md](interview_prep/03_databases.md) | SQL vs NoSQL, Sharding, Replication, Indexing | 25 min |
| [04_caching.md](interview_prep/04_caching.md) | Cache strategies, Redis, CDN, Eviction, Stampede | 20 min |
| [05_messaging.md](interview_prep/05_messaging.md) | Kafka, RabbitMQ, Pub/Sub, delivery guarantees | 20 min |
| [06_api_design.md](interview_prep/06_api_design.md) | REST, GraphQL, gRPC, WebSocket, Rate Limiting | 20 min |
| [07_distributed_systems.md](interview_prep/07_distributed_systems.md) | Consistent hashing, consensus, transactions | 25 min |
| [08_numbers_cheatsheet.md](interview_prep/08_numbers_cheatsheet.md) | Latency numbers, capacity math formulas | 10 min |

### Case Studies (Real System Designs)

#### Classic Systems (Must Know)
| File | System | Companies That Ask |
|------|--------|-------------------|
| [designs/01_url_shortener.md](interview_prep/designs/01_url_shortener.md) | TinyURL / Bit.ly | Google, Amazon, Meta |
| [designs/02_twitter_feed.md](interview_prep/designs/02_twitter_feed.md) | Twitter / Instagram Feed | Meta, Twitter, LinkedIn |
| [designs/03_whatsapp.md](interview_prep/designs/03_whatsapp.md) | WhatsApp / Messenger | Meta, Amazon |
| [designs/04_youtube.md](interview_prep/designs/04_youtube.md) | YouTube / Netflix | Google, Netflix, Amazon |
| [designs/05_uber.md](interview_prep/designs/05_uber.md) | Uber / Lyft | Uber, Google, Amazon |
| [designs/06_google_drive.md](interview_prep/designs/06_google_drive.md) | Google Drive / Dropbox | Google, Microsoft, Dropbox |

#### Advanced Systems
| File | System | Companies That Ask |
|------|--------|-------------------|
| [designs/07_rate_limiter.md](interview_prep/designs/07_rate_limiter.md) | Rate Limiter | Google, Amazon, Meta |
| [designs/08_notification_system.md](interview_prep/designs/08_notification_system.md) | Push Notifications | Meta, Amazon, Apple |
| [designs/09_search_autocomplete.md](interview_prep/designs/09_search_autocomplete.md) | Search Typeahead | Google, Amazon, LinkedIn |
| [designs/10_web_crawler.md](interview_prep/designs/10_web_crawler.md) | Web Crawler / Indexer | Google, Amazon |

#### Expert Systems (Senior/Staff Level)
| File | System | Companies That Ask |
|------|--------|-------------------|
| [designs/11_distributed_cache.md](interview_prep/designs/11_distributed_cache.md) | Redis / Memcached | Amazon, Google |
| [designs/12_payment_system.md](interview_prep/designs/12_payment_system.md) | Stripe / PayPal | Amazon, Stripe, Google Pay |
| [designs/13_stock_exchange.md](interview_prep/designs/13_stock_exchange.md) | Trading Platform | Amazon, Goldman Sachs |
| [designs/14_hotel_booking.md](interview_prep/designs/14_hotel_booking.md) | Airbnb / Booking.com | Airbnb, Amazon, Google |

---

## Deep Dive Track (`deep_dive/`)

| File | Topic |
|------|-------|
| [01_scalability_and_reliability.md](deep_dive/01_scalability_and_reliability.md) | Scalability patterns, fault tolerance, SRE concepts |
| [02_database_internals.md](deep_dive/02_database_internals.md) | B-trees, LSM-trees, WAL, MVCC, storage engines |
| [03_caching_deep.md](deep_dive/03_caching_deep.md) | Redis internals, consistent hashing, CDN deep dive |
| [04_messaging_deep.md](deep_dive/04_messaging_deep.md) | Kafka internals, log compaction, exactly-once delivery |
| [05_distributed_consensus.md](deep_dive/05_distributed_consensus.md) | Paxos, Raft, Zookeeper, leader election |
| [06_microservices_and_patterns.md](deep_dive/06_microservices_and_patterns.md) | Service mesh, Circuit Breaker, Saga, CQRS, Event Sourcing |
| [07_data_partitioning.md](deep_dive/07_data_partitioning.md) | Sharding strategies, consistent hashing, rebalancing |
| [08_real_world_architectures.md](deep_dive/08_real_world_architectures.md) | How Google, Amazon, Meta actually built their systems |

---

## 30-Day Study Schedule

| Week | Focus | Daily Task |
|------|-------|-----------|
| Week 1 (Days 1-7) | Foundations | Read one `interview_prep/` topic file per day |
| Week 2 (Days 8-14) | Classic Designs | Do one classic case study per day |
| Week 3 (Days 15-21) | Advanced + Expert | Do one advanced/expert case study per day |
| Week 4 (Days 22-30) | Deep Dive | Read deep_dive files, revisit weak areas |

### Daily Practice Routine
1. Read the day's topic (30 min)
2. Whiteboard the design without looking at notes (20 min)
3. Compare with notes, identify gaps (10 min)
4. Explain the design out loud as if in an interview (10 min)

---

## Key Interview Tips

1. **Ask before designing** — Clarify requirements, scale, constraints first (5 min)
2. **Estimate before architecting** — DAU, QPS, storage, bandwidth (5 min)
3. **Start high-level** — Draw the full system in one diagram, then deep dive
4. **State trade-offs** — Interviewers want to hear "I chose X over Y because..."
5. **Drive the conversation** — Don't wait for prompts; lead through the design
6. **Know your numbers** — Latency, throughput, capacity estimates (see cheatsheet)
