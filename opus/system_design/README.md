# System Design Interview Prep — Complete Guide

## How to Use This Guide

This is structured like a **textbook** — read the fundamentals first, then study individual system designs.

### Reading Order

#### Part 1: Foundations (Read First)
| # | Chapter | File | Time |
|---|---------|------|------|
| 1 | How to Answer System Design Questions | `fundamentals/01_framework.md` | 30 min |
| 2 | Back-of-the-Envelope Estimation | `fundamentals/02_estimation.md` | 45 min |
| 3 | Networking & Protocols | `fundamentals/03_networking.md` | 45 min |
| 4 | Databases Deep Dive | `fundamentals/04_databases.md` | 60 min |
| 5 | Caching Strategies | `fundamentals/05_caching.md` | 45 min |
| 6 | Message Queues & Async | `fundamentals/06_messaging.md` | 45 min |
| 7 | Load Balancing & Scaling | `fundamentals/07_scaling.md` | 45 min |
| 8 | Storage & CDN | `fundamentals/08_storage.md` | 30 min |
| 9 | Consistency & Consensus | `fundamentals/09_consistency.md` | 60 min |
| 10 | API Design | `fundamentals/10_api_design.md` | 30 min |

#### Part 2: Design Patterns
| # | Pattern | File |
|---|---------|------|
| 1 | Common Architectural Patterns | `patterns/01_common_patterns.md` |
| 2 | Data-Intensive Patterns | `patterns/02_data_patterns.md` |
| 3 | Real-time & Streaming | `patterns/03_realtime_patterns.md` |

#### Part 3: System Designs (Practice These)
| # | System | File | Difficulty | Frequency |
|---|--------|------|------------|-----------|
| 1 | URL Shortener (TinyURL) | `designs/01_url_shortener.md` | ⭐⭐ | Very High |
| 2 | Rate Limiter | `designs/02_rate_limiter.md` | ⭐⭐ | Very High |
| 3 | Twitter/X Feed | `designs/03_twitter_feed.md` | ⭐⭐⭐ | Very High |
| 4 | Instagram/Photo Sharing | `designs/04_instagram.md` | ⭐⭐⭐ | High |
| 5 | WhatsApp/Chat System | `designs/05_chat_system.md` | ⭐⭐⭐ | Very High |
| 6 | YouTube/Video Streaming | `designs/06_youtube.md` | ⭐⭐⭐⭐ | High |
| 7 | Google Drive/Dropbox | `designs/07_file_storage.md` | ⭐⭐⭐ | High |
| 8 | Uber/Ride Sharing | `designs/08_uber.md` | ⭐⭐⭐⭐ | Very High |
| 9 | Search Autocomplete | `designs/09_autocomplete.md` | ⭐⭐⭐ | High |
| 10 | Web Crawler | `designs/10_web_crawler.md` | ⭐⭐⭐ | Medium |
| 11 | Notification System | `designs/11_notifications.md` | ⭐⭐⭐ | High |
| 12 | Payment System (Stripe) | `designs/12_payment.md` | ⭐⭐⭐⭐ | High |
| 13 | Distributed Cache (Redis) | `designs/13_distributed_cache.md` | ⭐⭐⭐ | High |
| 14 | News Feed Ranking | `designs/14_newsfeed_ranking.md` | ⭐⭐⭐⭐ | High |
| 15 | Ticket Booking (BookMyShow) | `designs/15_ticket_booking.md` | ⭐⭐⭐ | High |

---

## 45-Minute Interview Framework (Quick Reference)

```
[0-5 min]   Requirements & Scope
[5-10 min]  High-Level Design (boxes and arrows)
[10-30 min] Deep Dive (the meat — data model, APIs, scaling)
[30-40 min] Address Bottlenecks & Trade-offs
[40-45 min] Wrap-up & Extensions
```

---

## Study Plan

**Week 1-2:** Read all fundamentals (1 chapter per day)
**Week 3-4:** Study designs 1-8 (1 per day, practice explaining aloud)
**Week 5-6:** Study designs 9-15 + patterns, do mock interviews
**Week 7+:** Practice with timer, focus on weak areas

> **Pro Tip:** After studying each design, close the file and try to whiteboard it from memory. The gaps you find are what you need to review.
