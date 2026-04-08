# Abuse Classifier System — Architecture Document

> **Version**: 1.0.0  
> **Status**: Draft  
> **Last Updated**: 2026-04-08  
> **Owner**: Platform Trust & Safety Engineering

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [System Overview](#2-system-overview)
3. [High-Level Architecture](#3-high-level-architecture)
4. [Request Lifecycle](#4-request-lifecycle)
5. [Component Deep Dives](#5-component-deep-dives)
6. [Data Flow Diagrams](#6-data-flow-diagrams)
7. [Decision Engine](#7-decision-engine)
8. [Scalability and Performance](#8-scalability-and-performance)
9. [Failure Modes and Resilience](#9-failure-modes-and-resilience)
10. [Security and Compliance](#10-security-and-compliance)
11. [Technology Choices](#11-technology-choices)

---

## 1. Executive Summary

This document describes the architecture for a **production-grade abuse classification system** capable of detecting and acting on abusive user-generated content (text, images, video, metadata) at scale. The system is designed around four principles:

- **Tiered intelligence** — cheap rules first, ML second, humans last.
- **Calibrated confidence** — every decision carries a probability, not a boolean.
- **Safe iteration** — shadow mode, champion–challenger, automatic rollback.
- **Auditability** — every action is traceable for appeals, compliance, and retraining.

---

## 2. System Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                      ABUSE CLASSIFIER PLATFORM                   │
│                                                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌───────────┐   │
│  │  Ingestion │→ │  Scoring   │→ │  Decision  │→ │  Action   │   │
│  │  & Routing │  │  Pipeline  │  │  Fusion    │  │  Engine   │   │
│  └────────────┘  └────────────┘  └────────────┘  └───────────┘   │
│         ↑               ↑               ↑              ↓         │
│  ┌──────┴───────────────┴───────────────┴──────────────┴──────┐  │
│  │              Observability · MLOps · Governance            │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3. High-Level Architecture

### 3.1 Full System Diagram

```mermaid
flowchart TB
    subgraph Clients["Client Layer"]
        MobileApp["Mobile App"]
        WebApp["Web App"]
        InternalAPI["Internal Services"]
        BulkIngest["Bulk / Batch Ingestion"]
    end

    subgraph Edge["API Gateway & Policy Edge"]
        LB["Load Balancer<br/>(ALB / Envoy)"]
        Auth["AuthN / AuthZ"]
        RL["Rate Limiter"]
        WAF["WAF + Bot Detection"]
        Router["Traffic Router<br/>(experiments, shadow)"]
    end

    subgraph Preprocessing["Feature Construction"]
        TN["Text Normalizer<br/>(unicode, casing, deobfuscation)"]
        LID["Language Identifier"]
        MDP["Media Pipeline<br/>(keyframes, transcoding, hash)"]
        MF["Metadata & Velocity<br/>Feature Builder"]
        FS["Feature Store<br/>(online serving)"]
    end

    subgraph Rules["Deterministic Layer"]
        BL["Blocklist / Allowlist<br/>Lookup"]
        HDB["Hash DB<br/>(PhotoDNA, pHash, CSAM)"]
        RE["Rule Engine<br/>(regex, threshold, policy)"]
    end

    subgraph Tier1["Tier-1: Fast ML"]
        TC["Text Classifier<br/>(distilled transformer)"]
        IC["Image Classifier<br/>(lightweight CNN / ViT)"]
        SC["Spam / Scam Classifier"]
        CAL1["Calibration Layer<br/>(temperature / isotonic)"]
    end

    subgraph Tier2["Tier-2: Deep ML (Async)"]
        Q["Priority Queue<br/>(Kafka / SQS)"]
        LLM["LLM / Large Ensemble"]
        MM["Multimodal Fusion"]
        NN["Embedding Nearest<br/>Neighbor Index"]
        CAL2["Calibration Layer"]
    end

    subgraph Decision["Decision Fusion & Routing"]
        DF["Score Aggregation<br/>+ Policy Thresholds"]
        CR["Cost-Sensitive<br/>Routing Matrix"]
    end

    subgraph Actions["Action & Enforcement"]
        ALLOW["✅ Allow"]
        SOFT["⚠️ Soft Limit<br/>(rate limit, friction)"]
        REMOVE["🚫 Remove / Hide"]
        ESCALATE["🔴 Escalate<br/>(law enforcement)"]
        APPEAL["📝 Appeal Path"]
    end

    subgraph Human["Human Review"]
        MQ["Moderation Queue<br/>(priority, language, skill)"]
        RUI["Reviewer UI<br/>(macros, translation,<br/>side-by-side scores)"]
        QA["QA & Calibration<br/>Audit"]
    end

    subgraph Data["Data Platform"]
        Lake["Data Lakehouse<br/>(raw + curated)"]
        LabelDB["Label Store<br/>(adjudication)"]
        AuditLog["Immutable Audit Log"]
        ObjStore["Encrypted Object Store<br/>(media, TTL)"]
    end

    subgraph MLOps["ML Operations"]
        Train["Training Pipeline<br/>(versioned, reproducible)"]
        Eval["Evaluation Harness<br/>(golden sets, bias)"]
        Registry["Model Registry<br/>(signed artifacts)"]
        Deploy["Staged Deploy<br/>(shadow → canary → full)"]
        Monitor["Quality & Drift<br/>Monitoring"]
    end

    %% Main flow
    MobileApp & WebApp & InternalAPI & BulkIngest --> LB
    LB --> Auth --> RL --> WAF --> Router

    Router --> TN & MDP & MF
    TN --> LID
    FS --> TN & MDP & MF

    TN & LID & MDP & MF --> BL & HDB & RE
    BL & HDB & RE --> TC & IC & SC
    TC & IC & SC --> CAL1 --> DF

    DF -->|uncertain / policy| Q
    Q --> LLM & MM & NN
    LLM & MM & NN --> CAL2 --> DF

    DF --> CR
    CR --> ALLOW & SOFT & REMOVE & ESCALATE
    CR -->|borderline / severe| MQ
    MQ --> RUI --> QA

    REMOVE & ESCALATE --> APPEAL

    %% Data flows
    Router --> Lake
    DF --> AuditLog
    RUI --> LabelDB
    MDP --> ObjStore
    Lake & LabelDB --> Train
    Train --> Eval --> Registry --> Deploy
    Deploy --> TC & IC & SC & LLM & MM
    Monitor --> Registry
```

### 3.2 Simplified Flow (4-Stage Pipeline)

```mermaid
flowchart LR
    A["📥 Ingest"] --> B["⚙️ Feature Build"]
    B --> C["🧠 Score"]
    C --> D["⚖️ Decide"]
    D --> E["🔨 Act"]

    style A fill:#e3f2fd,stroke:#1565c0
    style B fill:#fff3e0,stroke:#ef6c00
    style C fill:#f3e5f5,stroke:#7b1fa2
    style D fill:#e8f5e9,stroke:#2e7d32
    style E fill:#fce4ec,stroke:#c62828
```

---

## 4. Request Lifecycle

### 4.1 Synchronous Path (< p99 200ms target)

```mermaid
sequenceDiagram
    participant C as Client
    participant GW as API Gateway
    participant FE as Feature Engine
    participant RL as Rule Layer
    participant T1 as Tier-1 ML
    participant DF as Decision Fusion
    participant ACT as Action Engine

    C->>GW: POST /v1/classify {content}
    GW->>GW: Auth, rate-limit, WAF
    GW->>FE: Extract features
    FE->>FE: Normalize text, lang-ID, media hash
    FE->>RL: Deterministic check
    
    alt Rule match (blocklist / hash / regex)
        RL-->>DF: Hard signal (block / allow)
    else No rule match
        RL->>T1: Forward features
        T1->>T1: Multi-label inference + calibration
        T1-->>DF: Probability vector
    end
    
    DF->>DF: Aggregate scores, apply thresholds
    DF->>ACT: Routing decision
    ACT-->>C: {action, scores[], trace_id}
```

### 4.2 Asynchronous Path (Tier-2 / Human)

```mermaid
sequenceDiagram
    participant DF as Decision Fusion
    participant Q as Priority Queue
    participant T2 as Tier-2 ML
    participant HQ as Human Queue
    participant REV as Reviewer
    participant ACT as Action Engine

    DF->>Q: Enqueue (uncertain / policy-flagged)
    Q->>T2: Dequeue by priority
    T2->>T2: Deep model / ensemble / LLM
    T2-->>DF: Updated scores

    alt Still uncertain or severe
        DF->>HQ: Route to human queue
        HQ->>REV: Present case + model rationale
        REV->>REV: Review, label, macro action
        REV-->>ACT: Final decision + label
    else Resolved by Tier-2
        DF->>ACT: Updated action
    end

    ACT->>ACT: Enforce + write audit log
```

---

## 5. Component Deep Dives

### 5.1 API Gateway & Policy Edge

```
┌─────────────────────────────────────────────────────────────┐
│                        API GATEWAY                           │
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────┐  │
│  │   Auth    │→│   Rate    │→│   WAF     │→│  Experiment │  │
│  │  (JWT/    │  │  Limiter  │  │  + Bot   │  │   Router   │  │
│  │  mTLS)    │  │  (sliding │  │  Detect  │  │  (shadow,  │  │
│  │          │  │  window)  │  │          │  │  canary)   │  │
│  └──────────┘  └──────────┘  └──────────┘  └────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

| Responsibility | Detail |
|---|---|
| **Authentication** | mTLS for service-to-service; JWT/OAuth2 for external clients |
| **Rate limiting** | Per-tenant sliding window; burst allowance; adaptive under load |
| **WAF** | Block known attack patterns targeting the classifier itself |
| **Experiment router** | Shadow scoring (log but don't act), canary (small % live), A/B |

### 5.2 Feature Construction

```mermaid
flowchart LR
    subgraph Text
        RAW_T["Raw Text"] --> NORM["Unicode Normalize<br/>+ Deobfuscate"]
        NORM --> LID["Lang-ID"]
        NORM --> TOK["Tokenize<br/>(model-aligned)"]
        LID --> TOK
    end

    subgraph Media
        RAW_M["Raw Media"] --> SCAN["Virus Scan<br/>+ MIME Validate"]
        SCAN --> KF["Keyframe Extract"]
        SCAN --> HASH["Perceptual Hash<br/>(pHash, PhotoDNA)"]
        SCAN --> TRANS["Transcode<br/>(safe preview)"]
    end

    subgraph Metadata
        REQ["Request Context"] --> VEL["Velocity Features<br/>(posts/min, reports)"]
        REQ --> REP["Reputation Score"]
        REQ --> DEV["Device / Session<br/>Fingerprint"]
    end

    TOK --> FS["Feature Vector"]
    KF & HASH --> FS
    VEL & REP & DEV --> FS
```

### 5.3 Scoring Pipeline (Tiered)

```
                    ┌─────────────────────┐
                    │   Incoming Features  │
                    └─────────┬───────────┘
                              │
                    ┌─────────▼───────────┐
              ┌─────│  Deterministic Rules │─────┐
              │     │  (blocklist, hash,   │     │
              │     │   regex, hard policy)│     │
              │     └─────────┬───────────┘     │
         HARD BLOCK      NO MATCH          HARD ALLOW
              │              │                   │
              │     ┌────────▼────────┐          │
              │     │  Tier-1: Fast ML │          │
              │     │  (< 20ms, GPU)   │          │
              │     │  multi-label +   │          │
              │     │  calibration     │          │
              │     └────────┬────────┘          │
              │              │                   │
              │     ┌────────▼────────┐          │
              │     │  Decision Fusion │◄─────────┘
              │     │  (score agg +    │
              │     │   thresholds)    │
              │     └──┬─────┬────┬───┘
              │        │     │    │
         ┌────▼──┐  ┌──▼──┐ │ ┌──▼──────────┐
         │ Block │  │Allow│ │ │ Queue: Tier-2│
         └───────┘  └─────┘ │ │ (async, deep │
                             │ │  models)     │
                             │ └──────────────┘
                        ┌────▼─────────┐
                        │ Human Review │
                        └──────────────┘
```

### 5.4 Decision Fusion Logic

```mermaid
flowchart TD
    S["Score Vector<br/>{hate: 0.82, spam: 0.12,<br/>violence: 0.45, csam: 0.03}"]
    
    S --> P1{"CSAM > 0.01?"}
    P1 -->|Yes| BLOCK_CSAM["🔴 BLOCK + Mandatory<br/>Human + LE Report"]
    P1 -->|No| P2{"Any class > BLOCK<br/>threshold?"}
    
    P2 -->|Yes| BLOCK["🚫 BLOCK<br/>(auto-remove)"]
    P2 -->|No| P3{"Any class in<br/>UNCERTAIN zone?"}
    
    P3 -->|Yes| QUEUE["📋 QUEUE for<br/>Tier-2 / Human"]
    P3 -->|No| P4{"Any class > SOFT<br/>threshold?"}
    
    P4 -->|Yes| SOFT["⚠️ SOFT LIMIT<br/>(friction, rate limit)"]
    P4 -->|No| ALLOW["✅ ALLOW"]
```

**Threshold Configuration (per class, per policy)**:

| Class | Allow | Soft Limit | Queue (uncertain) | Block |
|-------|-------|------------|-------------------|-------|
| Hate/Harassment | < 0.30 | 0.30–0.55 | 0.55–0.80 | > 0.80 |
| Spam/Scam | < 0.40 | 0.40–0.60 | 0.60–0.85 | > 0.85 |
| Violence | < 0.25 | 0.25–0.50 | 0.50–0.75 | > 0.75 |
| CSAM-adjacent | < 0.01 | — | 0.01–0.50 | > 0.50 |
| Self-harm | < 0.20 | 0.20–0.45 | 0.45–0.70 | > 0.70 |

> Thresholds are tuned by precision/recall trade-offs per class and updated via the governance process.

---

## 6. Data Flow Diagrams

### 6.1 Training Data Lifecycle

```mermaid
flowchart LR
    subgraph Collection
        PROD["Production<br/>Traffic Logs"]
        AL["Active Learning<br/>Sampler"]
        RED["Red Team<br/>Adversarial"]
        EXT["External<br/>Datasets"]
    end

    subgraph Labeling
        GUIDE["Rater Guidelines<br/>+ Examples"]
        LABEL["Human Labeling<br/>(multi-rater)"]
        ADJ["Adjudication<br/>(disagreements)"]
        GOLD["Golden Set<br/>Maintenance"]
    end

    subgraph Training
        FEAT["Feature<br/>Engineering"]
        TRAIN["Model<br/>Training"]
        EVAL["Evaluation<br/>Harness"]
        CARD["Model Card<br/>Generation"]
    end

    subgraph Deploy
        REG["Model<br/>Registry"]
        SHADOW["Shadow<br/>Mode"]
        CANARY["Canary<br/>Rollout"]
        FULL["Full<br/>Deployment"]
    end

    PROD & AL & RED & EXT --> LABEL
    GUIDE --> LABEL
    LABEL --> ADJ --> GOLD
    GOLD --> FEAT --> TRAIN --> EVAL --> CARD --> REG
    REG --> SHADOW --> CANARY --> FULL
    FULL -.->|feedback| PROD
```

### 6.2 Audit and Compliance Data Flow

```mermaid
flowchart TB
    subgraph Inputs
        AUTO["Automated<br/>Decisions"]
        HUMAN["Human<br/>Decisions"]
        APPEAL["Appeal<br/>Outcomes"]
    end

    subgraph AuditPipeline["Audit Pipeline"]
        LOG["Immutable<br/>Event Log"]
        ENRICH["Enrichment<br/>(user, content,<br/>model version)"]
        STORE["Compliance<br/>Store"]
    end

    subgraph Outputs
        DASH["Compliance<br/>Dashboard"]
        EXPORT["Legal / Regulator<br/>Export"]
        RETRAIN["Retraining<br/>Signal"]
    end

    AUTO & HUMAN & APPEAL --> LOG
    LOG --> ENRICH --> STORE
    STORE --> DASH & EXPORT & RETRAIN
```

---

## 7. Decision Engine

### 7.1 Cost-Sensitive Routing Matrix

Each abuse class has an asymmetric cost profile:

```
                        PREDICTED
                   Positive    Negative
              ┌────────────┬────────────┐
   ACTUAL     │            │            │
   Positive   │  True Pos  │ False Neg  │
              │  (correct  │ (MISSED    │
              │   block)   │  ABUSE)    │
              │  Cost: 0   │ Cost: HIGH │
              ├────────────┼────────────┤
   Negative   │ False Pos  │ True Neg   │
              │ (wrongly   │ (correct   │
              │  blocked)  │  allow)    │
              │ Cost: MED  │ Cost: 0    │
              └────────────┴────────────┘
```

Cost weights vary by class:

| Class | False Negative Cost | False Positive Cost | Ratio |
|-------|-------------------|-------------------|-------|
| CSAM | Extreme | Low | 100:1 |
| Violence (threat) | Very High | Medium | 20:1 |
| Hate speech | High | High | 3:1 |
| Spam | Medium | Low | 2:1 |

### 7.2 Appeals Flow

```mermaid
stateDiagram-v2
    [*] --> ContentSubmitted
    ContentSubmitted --> AutoScored
    
    AutoScored --> Allowed: Below threshold
    AutoScored --> Removed: Above threshold
    AutoScored --> Queued: Uncertain zone
    
    Queued --> HumanReviewed
    HumanReviewed --> Allowed
    HumanReviewed --> Removed
    
    Removed --> AppealFiled: User appeals
    AppealFiled --> SecondReview: Different reviewer
    SecondReview --> Reinstated: Overturned
    SecondReview --> UpheldRemoval: Confirmed
    
    Reinstated --> Allowed
    Reinstated --> RetrainSignal
    UpheldRemoval --> [*]
    Allowed --> [*]
    
    RetrainSignal --> [*]
```

---

## 8. Scalability and Performance

### 8.1 Latency Budget (Synchronous Path)

```
┌─────────────────────────────────────────────────────────────┐
│                 200ms Total Budget (p99)                      │
│                                                             │
│  ┌──────┐  ┌──────────┐  ┌───────┐  ┌────────┐  ┌──────┐  │
│  │ Auth │  │ Features  │  │ Rules │  │ Tier-1 │  │ Fuse │  │
│  │ 5ms  │  │  25ms     │  │ 5ms   │  │  50ms  │  │ 5ms  │  │
│  └──────┘  └──────────┘  └───────┘  └────────┘  └──────┘  │
│                                                             │
│  Network overhead + serialization: ~10ms                     │
│  Buffer for spikes: ~100ms                                   │
└─────────────────────────────────────────────────────────────┘
```

### 8.2 Scaling Strategy

```mermaid
flowchart LR
    subgraph Horizontal["Horizontal Scaling"]
        API["API pods<br/>(CPU, autoscale<br/>on RPS)"]
        ML["ML pods<br/>(GPU, autoscale<br/>on queue depth)"]
        WORKER["Queue workers<br/>(autoscale on<br/>lag)"]
    end

    subgraph Caching["Caching Layers"]
        FEAT_C["Feature cache<br/>(Redis, TTL 5m)"]
        SCORE_C["Score cache<br/>(content hash,<br/>TTL 1h)"]
        RULE_C["Rule cache<br/>(in-process,<br/>TTL 30s)"]
    end

    subgraph Partitioning["Data Partitioning"]
        SHARD["Queue sharding<br/>by content type"]
        GEO["Geo-partitioned<br/>storage"]
        TENANT["Tenant isolation<br/>(noisy neighbor)"]
    end
```

### 8.3 Target SLOs

| Metric | Target | Measurement |
|--------|--------|-------------|
| Sync latency (p50) | < 80ms | Gateway to response |
| Sync latency (p99) | < 200ms | Gateway to response |
| Availability | 99.95% | Monthly uptime |
| Throughput | 50K RPS | Sustained, per region |
| Tier-2 turnaround | < 30s (p95) | Enqueue to score |
| Human review SLA | < 4h (urgent), < 24h (standard) | Queue to decision |

---

## 9. Failure Modes and Resilience

### 9.1 Degradation Cascade

```mermaid
flowchart TD
    HEALTHY["🟢 Healthy<br/>All tiers active"]
    
    HEALTHY -->|Tier-2 down| DEG1["🟡 Degraded-1<br/>Rules + Tier-1 only<br/>Queue accumulates"]
    
    DEG1 -->|Tier-1 down| DEG2["🟠 Degraded-2<br/>Rules + hash only<br/>Unknown traffic queued"]
    
    DEG2 -->|Rules engine down| DEG3["🔴 Degraded-3<br/>Fail-open or fail-closed<br/>(policy-configured per class)"]
    
    DEG3 -->|Total outage| CIRCUIT["⚫ Circuit Breaker<br/>Return cached decisions<br/>+ alert on-call"]
```

### 9.2 Failure Matrix

| Component | Failure Mode | Impact | Mitigation |
|-----------|-------------|--------|------------|
| Tier-1 GPU pool | OOM / crash | No ML scores | Fallback to rules-only; queue for async |
| Feature store | Latency spike | Slow features | In-process cache; degrade to text-only features |
| Kafka / Queue | Partition loss | Tier-2 delayed | Multi-AZ; DLQ; alert on lag |
| Rule engine | Stale rules | Missed patterns | Versioned rules in Git; health check on freshness |
| Human queue | Reviewer unavailable | SLA breach | Auto-escalate; expand pool; temporary threshold tighten |
| Model registry | Corrupt artifact | Bad predictions | Signed checksums; canary catches before full deploy |

---

## 10. Security and Compliance

### 10.1 Data Classification

```
┌────────────────────────────────────────────────────────┐
│                  DATA SENSITIVITY TIERS                  │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │  🔴 CRITICAL: CSAM hashes, LE reports            │  │
│  │     → Encrypted at rest + transit, minimal        │  │
│  │       access, regulatory retention only           │  │
│  ├──────────────────────────────────────────────────┤  │
│  │  🟠 HIGH: Raw abusive content, user PII          │  │
│  │     → Encrypted, access-logged, TTL-bound,       │  │
│  │       pseudonymized for training                  │  │
│  ├──────────────────────────────────────────────────┤  │
│  │  🟡 MEDIUM: Model scores, feature vectors        │  │
│  │     → Encrypted in transit, standard access       │  │
│  │       controls                                    │  │
│  ├──────────────────────────────────────────────────┤  │
│  │  🟢 LOW: Aggregated metrics, model cards          │  │
│  │     → Standard protection                         │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
```

### 10.2 Access Control

```mermaid
flowchart LR
    subgraph Roles
        ENG["Engineer"]
        DS["Data Scientist"]
        MOD["Moderator"]
        LEGAL["Legal / Compliance"]
        ONCALL["On-Call SRE"]
    end

    subgraph Resources
        CODE["Code & Config"]
        MODEL["Models & Weights"]
        CONTENT["Raw Content"]
        LABELS["Labels & Decisions"]
        AUDIT["Audit Logs"]
        METRICS["Dashboards"]
    end

    ENG -->|read/write| CODE
    ENG -->|read| METRICS
    DS -->|read/write| MODEL
    DS -->|read (pseudonymized)| LABELS
    MOD -->|read (case-scoped)| CONTENT
    MOD -->|write| LABELS
    LEGAL -->|read| AUDIT
    ONCALL -->|read| METRICS
    ONCALL -->|emergency write| CODE
```

---

## 11. Technology Choices

### 11.1 Recommended Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **API Gateway** | Envoy / Kong / AWS API Gateway | Native gRPC + HTTP, rate limiting, observability |
| **Inference: Tier-1** | NVIDIA Triton / TorchServe on GPU | Dynamic batching, multi-model, low latency |
| **Inference: Tier-2** | Ray Serve / vLLM (for LLMs) | Elastic GPU scaling, pipeline parallelism |
| **Queue** | Apache Kafka / AWS SQS | Durable, partitioned, replay-capable |
| **Feature Store** | Feast / Redis (online) + Delta Lake (offline) | Online/offline parity, low-latency serving |
| **Rule Engine** | Open Policy Agent (OPA) / custom | Declarative, auditable, hot-reloadable |
| **Object Storage** | S3 / GCS with lifecycle policies | Encrypted, TTL, cross-region replication |
| **Audit Log** | Immutable append-only (Kafka → Iceberg) | Tamper-evident, queryable, long retention |
| **Monitoring** | Prometheus + Grafana / Datadog | Metrics, alerts, SLO tracking |
| **ML Training** | PyTorch + W&B / MLflow | Experiment tracking, reproducibility |
| **Model Registry** | MLflow / Vertex AI Model Registry | Versioning, signing, promotion gates |
| **Orchestration** | Kubernetes (EKS / GKE) | GPU node pools, autoscaling, multi-tenant |
| **CI/CD** | GitHub Actions / Argo Workflows | Model + infra pipelines |

### 11.2 Infrastructure Topology

```
┌─────────────────────────────────────────────────────────────────┐
│                         REGION: us-east-1                        │
│                                                                 │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐       │
│  │    AZ-1       │  │    AZ-2       │  │    AZ-3       │       │
│  │               │  │               │  │               │       │
│  │  API pods (3) │  │  API pods (3) │  │  API pods (3) │       │
│  │  GPU pods (2) │  │  GPU pods (2) │  │  GPU pods (1) │       │
│  │  Workers (4)  │  │  Workers (4)  │  │  Workers (4)  │       │
│  │  Redis node   │  │  Redis node   │  │  Redis node   │       │
│  └───────────────┘  └───────────────┘  └───────────────┘       │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Shared: Kafka cluster (3 brokers), RDS (multi-AZ),     │   │
│  │          S3, Model Registry, Monitoring                  │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      REGION: eu-west-1                           │
│                  (same topology, geo-compliance)                  │
└─────────────────────────────────────────────────────────────────┘
```

---

*Next: [02-API-SPECIFICATION.md](./02-API-SPECIFICATION.md) — API contracts, request/response schemas, error codes*
