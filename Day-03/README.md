# Day 03 — System Design

## Problem: Design a Scalable OTT Streaming Platform (Netflix / Amazon Prime)

Design a VOD streaming platform supporting 200M DAU with 40M concurrent streams, multi-region delivery, offline downloads, and multi-device sync.

### Key Requirements
- **Scale:** 200M DAU, 40M concurrent streams, 50K title catalog
- **Latency:** p99 < 2s first frame, < 500ms seek
- **Content:** Up to 4K HDR, AV1/H.265 encoding, DRM (Widevine / FairPlay / PlayReady)
- **Delivery:** CDN-first with adaptive bitrate (HLS + DASH), edge-optimized
- **Storage:** Multi-region blob store (S3) + time-series DB (Cassandra / ScyllaDB) + ACID RDBMS (PostgreSQL)
- **Resilience:** Graceful degradation under partition, multi-CDN failover, async batch writes, edge DRM caching

### Interview Levels

Two interview transcripts are available at different experience levels:

| Level | File | Scale Assumption | Key Tech Choices |
|---|---|---|---|
| **Senior Staff / Architect** (15+ YOE) | [01-Full-Interview-Transcript.md](https://github.com/anomalyco/DailyTracker/blob/main/Day-03/Design%20Scalable%20OTT%20Streaming%20Platform/01-Full-Interview-Transcript.md) | 200M DAU, 40M concurrent | Cassandra/ScyllaDB/PostgreSQL/ES, GraphQL BFF, edge DRM caches, buffered batch writes, AV1 multi-CDN cost optimization, three-layer graceful degradation |
| **Software Engineer** (5 YOE) | [05-Years-Experience-Level/01-Full-Interview-Transcript.md](https://github.com/anomalyco/DailyTracker/blob/main/Day-03/Design%20Scalable%20OTT%20Streaming%20Platform/05-Years-Experience-Level/01-Full-Interview-Transcript.md) | 50M DAU, 10M concurrent | PostgreSQL + read replicas, Redis cache, SQS batching, multi-CDN failover, PostgreSQL full-text search |

### Reference Documents

| Document | Description |
|---|---|
| [API & Database Schema (Senior/Architect)](02-API-and-Database-Schema.md) | Full REST + gRPC contracts, 6 sequence diagrams, Cassandra/PostgreSQL/ScyllaDB schemas, caching strategy, observability, rate limiting, failure mode matrix |
| [API & Database Schema (5 YOE)](05-Years-Experience-Level/02-API-and-Database-Schema.md) | REST API contracts, PostgreSQL schemas, 4 sequence diagrams, Redis caching, data retention, scaling strategy |

### Key Decisions (Senior/Architect)
- **Database:** Cassandra for content metadata + watch history (multi-region writes, schema flexibility); ScyllaDB for streaming sessions (3M+ writes/sec); PostgreSQL for accounts/subs (ACID)
- **Caching:** Edge PoP Redis for DRM licenses (92% cache hit), regional Redis for hot metadata, GraphQL BFF deferred resolvers (60% load reduction)
- **Async writes:** Buffered batch progress ingestion (300x write amplification reduction)
- **Cost:** AV1 encoding + tiered multi-CDN = ~35% egress cost reduction
- **Resilience:** Three-layer degradation (CDN cache → Redis → static S3 fallback); never shows an empty screen

### Key Decisions (5 YOE)
- **Database:** PostgreSQL for all data (batch writes handle 50M DAU); read replicas for query scaling
- **Caching:** Redis for sessions, hot metadata, catalog pages
- **Async:** SQS/RabbitMQ decouples progress ingestion from DB writes
- **Search:** PostgreSQL full-text search (good enough for 20K titles)
- **Resilience:** Multi-CDN fallback, read replica lag monitoring, auto-scaling

---

*Solutions and deep-dive materials can be added below this line.*
