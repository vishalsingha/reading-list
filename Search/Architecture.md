# Advanced Job Research System Architecture

## System Overview

A multi-layered, low-latency, personalized job research system designed to deliver highly relevant job recommendations through intelligent search, real-time processing, and adaptive personalization.

---

## Core Design Principles

| Principle | Implementation Strategy |
|-----------|------------------------|
| **Multi-layered** | Separation of concerns across ingestion, processing, search, and presentation layers |
| **Low Latency** | Edge caching, pre-computed recommendations, async processing, optimized indexes |
| **Personalized** | ML-driven user profiling, behavioral tracking, collaborative filtering |

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT LAYER                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │   Web App    │  │  Mobile App  │  │  Browser Ext │  │   API/SDK    │             │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘             │
└─────────┼─────────────────┼─────────────────┼─────────────────┼─────────────────────┘
          │                 │                 │                 │
          └─────────────────┴────────┬────────┴─────────────────┘
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              EDGE LAYER (CDN + Edge Functions)                       │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │  • Response Caching  • Rate Limiting  • Geo-routing  • A/B Testing          │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              API GATEWAY LAYER                                       │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐         │
│  │ Auth Service  │  │ Rate Limiter  │  │ Load Balancer │  │ Request Router│         │
│  └───────────────┘  └───────────────┘  └───────────────┘  └───────────────┘         │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                     │
          ┌──────────────────────────┼──────────────────────────┐
          ▼                          ▼                          ▼
┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
│  SEARCH SERVICE  │      │ RECOMMENDATION   │      │  USER SERVICE    │
│                  │      │     SERVICE      │      │                  │
│ • Query Parser   │      │ • Real-time Rec  │      │ • Profile Mgmt   │
│ • Search Engine  │      │ • Batch Rec      │      │ • Preferences    │
│ • Ranking        │      │ • ML Models      │      │ • History        │
│ • Filters        │      │ • A/B Testing    │      │ • Notifications  │
└────────┬─────────┘      └────────┬─────────┘      └────────┬─────────┘
         │                         │                         │
         └─────────────────────────┼─────────────────────────┘
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         PERSONALIZATION ENGINE                                       │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                      │
│  │ User Embedding  │  │ Job Embedding   │  │ Context Engine  │                      │
│  │    Generator    │  │   Generator     │  │ (Time/Location) │                      │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘                      │
│           │                    │                    │                               │
│           └────────────────────┼────────────────────┘                               │
│                                ▼                                                    │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │              MATCHING & SCORING ENGINE (Real-time + Pre-computed)           │    │
│  │  • Vector Similarity  • Skill Matching  • Salary Alignment  • Culture Fit   │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              DATA LAYER                                              │
│                                                                                      │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐    │
│  │   PostgreSQL    │  │  Elasticsearch  │  │     Redis       │  │   Pinecone   │    │
│  │   (Primary DB)  │  │  (Search Index) │  │    (Cache)      │  │  (Vectors)   │    │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  └──────────────┘    │
│                                                                                      │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                      │
│  │   ClickHouse    │  │    MongoDB      │  │   S3/Object     │                      │
│  │  (Analytics)    │  │  (User Events)  │  │    Storage      │                      │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         DATA INGESTION LAYER                                         │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐    │
│  │   Job Crawlers  │  │  API Ingestors  │  │  Resume Parser  │  │ Event Stream │    │
│  │  (Scrapy/etc)   │  │  (Partner APIs) │  │   (NLP/ML)      │  │   (Kafka)    │    │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  └──────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Detailed Layer Breakdown

### Layer 1: Data Ingestion Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        DATA INGESTION PIPELINE                               │
│                                                                              │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│   │   Source    │    │   Source    │    │   Source    │    │   Source    │  │
│   │  Crawlers   │    │  API Feeds  │    │   RSS/XML   │    │   Manual    │  │
│   └──────┬──────┘    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘  │
│          │                  │                  │                  │         │
│          └──────────────────┴────────┬─────────┴──────────────────┘         │
│                                      ▼                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    MESSAGE QUEUE (Kafka/RabbitMQ)                   │   │
│   │    Topics: raw_jobs | parsed_jobs | enriched_jobs | indexed_jobs    │   │
│   └─────────────────────────────────┬───────────────────────────────────┘   │
│                                     ▼                                       │
│   ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐           │
│   │   Dedup    │─▶│   Parse    │─▶│   Enrich   │─▶│   Index    │           │
│   │  Service   │  │  Service   │  │  Service   │  │  Service   │           │
│   └────────────┘  └────────────┘  └────────────┘  └────────────┘           │
│                                                                              │
│   Enrichment includes:                                                       │
│   • Company data from Clearbit/LinkedIn                                      │
│   • Salary estimates from market data                                        │
│   • Skill extraction via NLP                                                 │
│   • Location normalization & geocoding                                       │
│   • Industry/sector classification                                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Components:**

| Component | Technology | Purpose |
|-----------|------------|---------|
| Job Crawler | Scrapy + Playwright | Scrape job boards (Indeed, LinkedIn, Glassdoor) |
| API Ingestor | Python workers | Consume partner APIs (Greenhouse, Lever, etc.) |
| Message Queue | Apache Kafka | Decouple ingestion from processing |
| Deduplication | MinHash/SimHash | Prevent duplicate job listings |
| NLP Parser | spaCy + Custom Models | Extract skills, requirements, benefits |
| Enrichment | External APIs + ML | Add company data, salary estimates |

---

### Layer 2: Search & Discovery Engine

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SEARCH & DISCOVERY ENGINE                            │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                         QUERY UNDERSTANDING                           │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐      │   │
│  │  │  Tokenizer │  │  Intent    │  │   Entity   │  │   Query    │      │   │
│  │  │            │─▶│  Classifier│─▶│  Extractor │─▶│  Expander  │      │   │
│  │  └────────────┘  └────────────┘  └────────────┘  └────────────┘      │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                     │                                       │
│                                     ▼                                       │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                      MULTI-STRATEGY RETRIEVAL                         │   │
│  │                                                                        │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐        │   │
│  │  │    Keyword      │  │    Semantic     │  │    Hybrid       │        │   │
│  │  │    Search       │  │    Search       │  │    Search       │        │   │
│  │  │  (BM25/TF-IDF)  │  │  (Vector/ANN)   │  │  (RRF Fusion)   │        │   │
│  │  │                 │  │                 │  │                 │        │   │
│  │  │ Elasticsearch   │  │ Pinecone/Qdrant │  │   Combined      │        │   │
│  │  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘        │   │
│  │           │                    │                    │                 │   │
│  │           └────────────────────┼────────────────────┘                 │   │
│  │                                ▼                                      │   │
│  │  ┌─────────────────────────────────────────────────────────────┐      │   │
│  │  │              RECIPROCAL RANK FUSION (RRF)                   │      │   │
│  │  │         Combines results from multiple retrieval methods     │      │   │
│  │  └─────────────────────────────────────────────────────────────┘      │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                     │                                       │
│                                     ▼                                       │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                         RANKING PIPELINE                              │   │
│  │                                                                        │   │
│  │  Stage 1: Candidate Generation (1000s → 100s)                         │   │
│  │     └─▶ Fast filtering by location, salary, job type                  │   │
│  │                                                                        │   │
│  │  Stage 2: Feature Scoring (100s → 50)                                 │   │
│  │     └─▶ Skill match score, experience alignment, recency              │   │
│  │                                                                        │   │
│  │  Stage 3: Personalized Reranking (50 → 20)                            │   │
│  │     └─▶ ML model with user preferences, history, behavior             │   │
│  │                                                                        │   │
│  │  Stage 4: Diversity & Business Rules (20 → Final)                     │   │
│  │     └─▶ Ensure variety, apply sponsor boosts, remove duplicates       │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Search Features:**

- **Query Understanding**: Intent classification (job search vs. company research vs. salary lookup)
- **Synonym Expansion**: "Software Engineer" → "Developer", "Programmer", "SWE"
- **Skill Matching**: Match user skills to job requirements with partial matching
- **Semantic Search**: Vector similarity for "jobs like this one"
- **Faceted Search**: Filter by location, salary, remote, experience level

---

### Layer 3: Personalization Engine

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PERSONALIZATION ENGINE                               │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                      USER PROFILE CONSTRUCTION                        │   │
│  │                                                                        │   │
│  │  EXPLICIT DATA                    IMPLICIT DATA                        │   │
│  │  ├─ Resume/CV                     ├─ Search queries                    │   │
│  │  ├─ Skills listed                 ├─ Jobs viewed (dwell time)          │   │
│  │  ├─ Preferences set               ├─ Jobs applied to                   │   │
│  │  ├─ Saved searches                ├─ Jobs saved/bookmarked             │   │
│  │  ├─ Location preferences          ├─ Jobs dismissed                    │   │
│  │  └─ Salary expectations           └─ Session patterns                  │   │
│  │                                                                        │   │
│  │                         ▼                                              │   │
│  │  ┌─────────────────────────────────────────────────────────────┐      │   │
│  │  │                   USER EMBEDDING MODEL                       │      │   │
│  │  │   Dense vector representation of user preferences (384-dim)  │      │   │
│  │  │   Updated: Real-time (lightweight) + Batch (full retrain)    │      │   │
│  │  └─────────────────────────────────────────────────────────────┘      │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                      RECOMMENDATION STRATEGIES                        │   │
│  │                                                                        │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐          │   │
│  │  │   Content      │  │ Collaborative  │  │   Contextual   │          │   │
│  │  │   Based        │  │  Filtering     │  │   Bandits      │          │   │
│  │  │                │  │                │  │                │          │   │
│  │  │ Job features   │  │ Similar users  │  │ Time of day    │          │   │
│  │  │ match user     │  │ liked similar  │  │ Device type    │          │   │
│  │  │ preferences    │  │ jobs           │  │ Location       │          │   │
│  │  └───────┬────────┘  └───────┬────────┘  └───────┬────────┘          │   │
│  │          │                   │                   │                    │   │
│  │          └───────────────────┼───────────────────┘                    │   │
│  │                              ▼                                        │   │
│  │  ┌─────────────────────────────────────────────────────────────┐      │   │
│  │  │              ENSEMBLE MODEL (Weighted Combination)           │      │   │
│  │  │   Weights learned via online learning (Thompson Sampling)    │      │   │
│  │  └─────────────────────────────────────────────────────────────┘      │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                      MATCHING DIMENSIONS                              │   │
│  │                                                                        │   │
│  │  Dimension          Weight    Calculation Method                       │   │
│  │  ─────────────────────────────────────────────────────────────────     │   │
│  │  Skill Match        0.30      Jaccard + semantic similarity            │   │
│  │  Experience Fit     0.20      Years match + level alignment            │   │
│  │  Salary Alignment   0.15      Range overlap percentage                 │   │
│  │  Location Match     0.15      Distance + remote preference             │   │
│  │  Company Culture    0.10      Glassdoor + stated preferences           │   │
│  │  Growth Potential   0.10      Career path alignment                    │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### Layer 4: Low Latency Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      LOW LATENCY OPTIMIZATION STRATEGY                       │
│                                                                              │
│  TARGET: p50 < 50ms | p95 < 150ms | p99 < 300ms                             │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                         CACHING HIERARCHY                             │   │
│  │                                                                        │   │
│  │  Level 1: Browser/Client Cache (< 1ms)                                │   │
│  │     └─▶ Static assets, user preferences, recent searches              │   │
│  │                                                                        │   │
│  │  Level 2: Edge Cache - CDN (1-10ms)                                   │   │
│  │     └─▶ Popular searches, trending jobs, location-based results       │   │
│  │                                                                        │   │
│  │  Level 3: Application Cache - Redis Cluster (5-15ms)                  │   │
│  │     └─▶ User sessions, pre-computed recommendations, hot jobs         │   │
│  │                                                                        │   │
│  │  Level 4: Database Cache - Query Results (20-50ms)                    │   │
│  │     └─▶ Materialized views, frequently accessed data                  │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                      PRE-COMPUTATION STRATEGIES                       │   │
│  │                                                                        │   │
│  │  ┌─────────────────────────────────────────────────────────────┐      │   │
│  │  │  OFFLINE COMPUTATION (Every 1-6 hours)                       │      │   │
│  │  │  • Full user embedding regeneration                          │      │   │
│  │  │  • Batch recommendations for all users                       │      │   │
│  │  │  • Popular job rankings by segment                           │      │   │
│  │  │  • Skill taxonomy updates                                    │      │   │
│  │  └─────────────────────────────────────────────────────────────┘      │   │
│  │                                                                        │   │
│  │  ┌─────────────────────────────────────────────────────────────┐      │   │
│  │  │  NEAR-REAL-TIME (Every 1-5 minutes)                          │      │   │
│  │  │  • Trending job updates                                      │      │   │
│  │  │  • Hot search query caching                                  │      │   │
│  │  │  • New job notifications                                     │      │   │
│  │  └─────────────────────────────────────────────────────────────┘      │   │
│  │                                                                        │   │
│  │  ┌─────────────────────────────────────────────────────────────┐      │   │
│  │  │  REAL-TIME (Request-time)                                    │      │   │
│  │  │  • Lightweight reranking                                     │      │   │
│  │  │  • Context-based adjustments                                 │      │   │
│  │  │  • Diversity injection                                       │      │   │
│  │  └─────────────────────────────────────────────────────────────┘      │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                      INFRASTRUCTURE OPTIMIZATIONS                     │   │
│  │                                                                        │   │
│  │  • Connection pooling (PgBouncer, Redis connection pools)             │   │
│  │  • gRPC for inter-service communication (vs REST)                     │   │
│  │  • Async I/O everywhere (Python asyncio, Node.js)                     │   │
│  │  • Database read replicas for search workloads                        │   │
│  │  • Sharded Elasticsearch clusters by region                           │   │
│  │  • Kubernetes HPA for auto-scaling                                    │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Data Flow Diagrams

### User Search Flow

```
User Query: "remote python developer jobs $120k+"
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  1. QUERY PARSING (5ms)                                       │
│     • Tokenize: ["remote", "python", "developer", "$120k+"]   │
│     • Extract intent: JOB_SEARCH                              │
│     • Extract entities:                                       │
│       - work_type: remote                                     │
│       - skills: [python]                                      │
│       - role: developer                                       │
│       - salary_min: 120000                                    │
└──────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  2. CACHE CHECK (2ms)                                         │
│     • Check Redis for similar recent queries                  │
│     • Check if user has cached recommendations                │
│     • Cache HIT? → Return cached + lightweight personalize    │
│     • Cache MISS? → Continue to retrieval                     │
└──────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  3. PARALLEL RETRIEVAL (30ms)                                 │
│                                                               │
│     ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│     │  Keyword    │  │  Semantic   │  │  User-based │        │
│     │  Search     │  │  Search     │  │  Recommend  │        │
│     │  (ES)       │  │  (Pinecone) │  │  (Precomp)  │        │
│     └──────┬──────┘  └──────┬──────┘  └──────┬──────┘        │
│            │                │                │                │
│            └────────────────┼────────────────┘                │
│                             ▼                                 │
│     Merge via Reciprocal Rank Fusion → 500 candidates         │
└──────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  4. FILTERING & SCORING (15ms)                                │
│     • Apply hard filters (salary >= 120k, remote = true)      │
│     • Score remaining candidates:                             │
│       - skill_match_score                                     │
│       - experience_fit_score                                  │
│       - salary_alignment_score                                │
│       - recency_score                                         │
│     • 500 → 100 candidates                                    │
└──────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  5. PERSONALIZED RERANKING (20ms)                             │
│     • Load user embedding from cache                          │
│     • Compute user-job similarity scores                      │
│     • Apply learned preferences model                         │
│     • Boost/penalize based on:                                │
│       - Past application patterns                             │
│       - Company preference signals                            │
│       - Similar user behaviors                                │
│     • 100 → 30 candidates                                     │
└──────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  6. DIVERSITY & BUSINESS RULES (5ms)                          │
│     • Ensure company diversity (max 3 per company)            │
│     • Ensure location diversity                               │
│     • Apply sponsored job placements                          │
│     • Remove near-duplicate listings                          │
│     • 30 → 20 final results                                   │
└──────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  7. RESPONSE ASSEMBLY (3ms)                                   │
│     • Hydrate job details from cache/DB                       │
│     • Generate snippets with query highlighting               │
│     • Add match explanations ("95% skill match")              │
│     • Include facet counts for filtering                      │
│     • Total latency: ~80ms (p50)                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Technology Stack

### Core Services

| Layer | Technology | Rationale |
|-------|------------|-----------|
| **API Gateway** | Kong / AWS API Gateway | Rate limiting, auth, routing |
| **Backend Services** | Python (FastAPI) + Go | FastAPI for ML services, Go for high-throughput |
| **Search Engine** | Elasticsearch 8.x | Full-text search, aggregations, geo queries |
| **Vector Database** | Pinecone / Qdrant | Semantic search, ANN queries |
| **Primary Database** | PostgreSQL 15 | ACID compliance, complex queries |
| **Cache** | Redis Cluster | Session storage, hot data, pub/sub |
| **Message Queue** | Apache Kafka | Event streaming, data pipeline |
| **ML Platform** | MLflow + Ray Serve | Model versioning, serving |

### Infrastructure

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Container Orchestration** | Kubernetes (EKS/GKE) | Service deployment, scaling |
| **Service Mesh** | Istio | Traffic management, observability |
| **CDN** | Cloudflare / CloudFront | Edge caching, DDoS protection |
| **Monitoring** | Prometheus + Grafana | Metrics, alerting |
| **Logging** | ELK Stack | Centralized logging |
| **Tracing** | Jaeger / Datadog APM | Distributed tracing |

### ML/AI Components

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Embeddings** | sentence-transformers | Text to vector conversion |
| **NLP** | spaCy + Custom NER | Skill extraction, parsing |
| **Ranking Model** | XGBoost / LightGBM | Learn-to-rank |
| **Recommendations** | TensorFlow Recommenders | Collaborative filtering |
| **Feature Store** | Feast | ML feature management |

---

## Database Schema (Simplified)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CORE ENTITIES                                   │
│                                                                              │
│  ┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐      │
│  │      USERS      │      │      JOBS       │      │    COMPANIES    │      │
│  ├─────────────────┤      ├─────────────────┤      ├─────────────────┤      │
│  │ id              │      │ id              │      │ id              │      │
│  │ email           │      │ title           │      │ name            │      │
│  │ name            │      │ description     │      │ industry        │      │
│  │ resume_text     │      │ company_id (FK) │◀─────│ size            │      │
│  │ skills[]        │      │ location        │      │ website         │      │
│  │ experience_years│      │ salary_min      │      │ logo_url        │      │
│  │ preferences     │      │ salary_max      │      │ glassdoor_rating│      │
│  │ embedding       │      │ remote_type     │      │ description     │      │
│  │ created_at      │      │ skills_required │      └─────────────────┘      │
│  └────────┬────────┘      │ experience_level│                               │
│           │               │ job_type        │                               │
│           │               │ embedding       │                               │
│           │               │ posted_at       │                               │
│           │               │ expires_at      │                               │
│           │               └────────┬────────┘                               │
│           │                        │                                        │
│           │    ┌───────────────────┴───────────────────┐                    │
│           │    │                                       │                    │
│           ▼    ▼                                       ▼                    │
│  ┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐      │
│  │  APPLICATIONS   │      │   USER_EVENTS   │      │  SAVED_JOBS     │      │
│  ├─────────────────┤      ├─────────────────┤      ├─────────────────┤      │
│  │ id              │      │ id              │      │ user_id (FK)    │      │
│  │ user_id (FK)    │      │ user_id (FK)    │      │ job_id (FK)     │      │
│  │ job_id (FK)     │      │ event_type      │      │ saved_at        │      │
│  │ status          │      │ job_id (FK)     │      │ notes           │      │
│  │ applied_at      │      │ search_query    │      └─────────────────┘      │
│  │ resume_version  │      │ dwell_time_ms   │                               │
│  └─────────────────┘      │ timestamp       │                               │
│                           │ context         │                               │
│                           └─────────────────┘                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Implementation Phases

### Phase 1: Foundation (Weeks 1-4)
- [ ] Set up infrastructure (Kubernetes, databases, caching)
- [ ] Implement basic job ingestion pipeline
- [ ] Build core search with Elasticsearch
- [ ] Create user authentication and profile management
- [ ] Develop basic web UI

### Phase 2: Search Enhancement (Weeks 5-8)
- [ ] Add semantic search with vector embeddings
- [ ] Implement query understanding (intent, entities)
- [ ] Build multi-stage ranking pipeline
- [ ] Add faceted search and filters
- [ ] Optimize for low latency (caching layers)

### Phase 3: Personalization (Weeks 9-12)
- [ ] Build user embedding generation system
- [ ] Implement content-based recommendations
- [ ] Add collaborative filtering
- [ ] Create real-time behavior tracking
- [ ] Develop personalized ranking model

### Phase 4: Scale & Optimize (Weeks 13-16)
- [ ] Implement A/B testing framework
- [ ] Add monitoring and alerting
- [ ] Optimize ML model serving
- [ ] Build admin dashboard
- [ ] Performance tuning and load testing

---

## Key Performance Indicators

| Metric | Target | Measurement |
|--------|--------|-------------|
| Search Latency (p50) | < 50ms | APM tracing |
| Search Latency (p95) | < 150ms | APM tracing |
| Recommendation Relevance | > 0.7 NDCG@10 | Offline evaluation |
| Click-through Rate | > 15% | Analytics |
| Application Rate | > 5% | Analytics |
| User Retention (7-day) | > 40% | Cohort analysis |
| Index Freshness | < 15 min | Pipeline monitoring |

---

## Security Considerations

- **Authentication**: OAuth 2.0 / OIDC with JWT tokens
- **Authorization**: RBAC for admin functions
- **Data Protection**: PII encryption at rest and in transit
- **Rate Limiting**: Per-user and per-IP limits
- **Input Validation**: Sanitize all search queries
- **Audit Logging**: Track all data access

---

## Scalability Design

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         HORIZONTAL SCALING STRATEGY                          │
│                                                                              │
│  Component              Scaling Trigger          Scale Factor               │
│  ─────────────────────────────────────────────────────────────────────────  │
│  API Servers            CPU > 70%                +2 pods                    │
│  Search Workers         Queue depth > 1000      +1 pod                     │
│  ML Inference           GPU util > 80%          +1 GPU node                │
│  Redis                  Memory > 80%            Add shard                  │
│  Elasticsearch          Disk > 70%              Add data node              │
│  Kafka                  Lag > 10k msgs          Add partition              │
│                                                                              │
│  Expected capacity at launch:                                                │
│  • 10M jobs indexed                                                          │
│  • 1M daily active users                                                     │
│  • 100 searches/second sustained                                             │
│  • 1000 searches/second peak                                                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Directory Structure

```
SearchSystem/
├── services/
│   ├── api-gateway/           # Kong/Express gateway
│   ├── search-service/        # Elasticsearch + ranking
│   ├── recommendation-service/# ML-based recommendations  
│   ├── user-service/          # User management
│   ├── ingestion-service/     # Job crawling & processing
│   └── notification-service/  # Email/push notifications
├── ml/
│   ├── embeddings/            # Text embedding models
│   ├── ranking/               # Learn-to-rank models
│   ├── recommendations/       # Collaborative filtering
│   └── nlp/                   # Skill extraction, NER
├── infrastructure/
│   ├── kubernetes/            # K8s manifests
│   ├── terraform/             # Infrastructure as code
│   └── monitoring/            # Prometheus, Grafana configs
├── shared/
│   ├── proto/                 # gRPC definitions
│   ├── schemas/               # JSON schemas
│   └── utils/                 # Shared utilities
├── web/                       # Frontend application
├── mobile/                    # Mobile apps
└── docs/                      # Documentation
```

---

## Next Steps

1. **Review and refine requirements** with stakeholders
2. **Set up development environment** and CI/CD pipelines
3. **Begin Phase 1** with infrastructure setup
4. **Create detailed API specifications** for each service
5. **Design data migration strategy** if existing data exists

---

*Architecture Version: 1.0 | Last Updated: April 2026*
