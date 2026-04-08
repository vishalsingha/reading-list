# Abuse Classifier System

A production-grade, multi-tiered abuse classification platform for detecting and acting on abusive user-generated content at scale.

---

## Documentation Suite

| Document | Description |
|----------|-------------|
| [01 — Architecture](docs/01-ARCHITECTURE.md) | Full system architecture, component diagrams, data flows, decision engine, scalability, failure modes, security, and technology choices |
| [02 — API Specification](docs/02-API-SPECIFICATION.md) | REST API contracts, request/response schemas, authentication, error codes, rate limits, and SDK examples |
| [03 — Data & Training](docs/03-DATA-AND-TRAINING.md) | Abuse taxonomy, labeling pipeline, feature engineering, model architecture (Tier-1 and Tier-2), training pipeline, evaluation framework, bias/fairness, and model cards |
| [04 — Deployment & Operations](docs/04-DEPLOYMENT-AND-OPERATIONS.md) | Deployment strategy (shadow/canary/ramp), CI/CD pipelines, monitoring, alerting, incident response runbooks, capacity planning, disaster recovery |
| [05 — Taxonomy & Policy](docs/05-TAXONOMY-AND-POLICY.md) | Governance model, full abuse taxonomy definitions, threshold management, change management, legal/regulatory compliance, ethical guidelines, and glossary |

---

## Architecture at a Glance

```
Client → API Gateway → Feature Build → Rules → Tier-1 ML → Decision Fusion → Action
                                                    ↓ (uncertain)
                                              Tier-2 ML (async)
                                                    ↓ (borderline/severe)
                                              Human Review
```

**Design principles:**

- **Tiered intelligence** — cheap deterministic rules first, fast ML second, deep ML third, humans last
- **Calibrated confidence** — every decision carries a probability, not a boolean
- **Safe iteration** — shadow mode, champion-challenger, automatic rollback on SLO breach
- **Auditability** — every action is traceable for appeals, compliance, and retraining

---

## Key Numbers

| Metric | Target |
|--------|--------|
| Sync latency (p99) | < 200ms |
| Availability | 99.95% |
| Throughput | 50K RPS |
| Tier-2 turnaround | < 30s (p95) |
| Human review SLA | < 4h (urgent) |

---

## Quick Navigation

- **Building the system?** Start with [Architecture](docs/01-ARCHITECTURE.md)
- **Integrating as a client?** Start with [API Specification](docs/02-API-SPECIFICATION.md)
- **Training models?** Start with [Data & Training](docs/03-DATA-AND-TRAINING.md)
- **Running in production?** Start with [Deployment & Operations](docs/04-DEPLOYMENT-AND-OPERATIONS.md)
- **Understanding policy?** Start with [Taxonomy & Policy](docs/05-TAXONOMY-AND-POLICY.md)
