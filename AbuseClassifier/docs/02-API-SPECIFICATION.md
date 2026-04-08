# Abuse Classifier System — API Specification

> **Version**: 1.0.0  
> **Status**: Draft  
> **Last Updated**: 2026-04-08

---

## Table of Contents

1. [Overview](#1-overview)
2. [Base URL and Versioning](#2-base-url-and-versioning)
3. [Authentication](#3-authentication)
4. [Core Endpoints](#4-core-endpoints)
5. [Webhook Callbacks](#5-webhook-callbacks)
6. [Error Handling](#6-error-handling)
7. [Rate Limits](#7-rate-limits)
8. [SDK Examples](#8-sdk-examples)

---

## 1. Overview

The Abuse Classifier API provides synchronous and asynchronous content classification. All endpoints accept JSON and return JSON. Media is uploaded via multipart or pre-signed URL.

```
┌─────────────────────────────────────────────────────────┐
│                     API Surface                          │
│                                                         │
│  Sync:   POST /v1/classify          (text / metadata)   │
│  Sync:   POST /v1/classify/media    (image / video)     │
│  Async:  POST /v1/classify/async    (any, webhook)      │
│  Query:  GET  /v1/decisions/{id}    (lookup by trace)    │
│  Appeal: POST /v1/appeals           (user appeal)        │
│  Admin:  GET  /v1/admin/health      (health check)       │
│  Admin:  GET  /v1/admin/metrics     (prometheus)         │
│  Config: GET  /v1/config/thresholds (current thresholds) │
│  Config: PUT  /v1/config/thresholds (update thresholds)  │
└─────────────────────────────────────────────────────────┘
```

---

## 2. Base URL and Versioning

| Environment | Base URL |
|-------------|----------|
| Production | `https://abuse-api.example.com/v1` |
| Staging | `https://abuse-api-staging.example.com/v1` |
| Development | `http://localhost:8080/v1` |

Versioning is path-based (`/v1`, `/v2`). Deprecation is communicated via `Sunset` and `Deprecation` headers 90 days before removal.

---

## 3. Authentication

All requests require a bearer token in the `Authorization` header:

```
Authorization: Bearer <tenant_api_key>
```

Service-to-service calls use **mTLS** with client certificates. Both methods populate the `X-Tenant-ID` header downstream.

---

## 4. Core Endpoints

### 4.1 Synchronous Text Classification

```
POST /v1/classify
```

**Request**:

```json
{
  "content": {
    "text": "I will find you and hurt you",
    "language_hint": "en",
    "content_type": "comment"
  },
  "metadata": {
    "user_id": "usr_abc123",
    "account_age_days": 3,
    "session_id": "sess_xyz",
    "ip_country": "US",
    "platform": "mobile_ios"
  },
  "options": {
    "classes": ["hate", "violence", "spam", "self_harm", "sexual"],
    "return_explanations": true,
    "experiment_id": "exp_shadow_v2"
  }
}
```

**Response** (`200 OK`):

```json
{
  "trace_id": "trc_7f3a2b1c",
  "request_id": "req_9e8d7c6b",
  "timestamp": "2026-04-08T14:30:00.123Z",
  "action": "remove",
  "confidence": 0.87,
  "scores": [
    {
      "class": "violence",
      "score": 0.87,
      "threshold_applied": 0.75,
      "action_taken": "remove"
    },
    {
      "class": "hate",
      "score": 0.42,
      "threshold_applied": 0.80,
      "action_taken": "none"
    },
    {
      "class": "spam",
      "score": 0.03,
      "threshold_applied": 0.85,
      "action_taken": "none"
    },
    {
      "class": "self_harm",
      "score": 0.08,
      "threshold_applied": 0.70,
      "action_taken": "none"
    },
    {
      "class": "sexual",
      "score": 0.01,
      "threshold_applied": 0.70,
      "action_taken": "none"
    }
  ],
  "explanations": [
    {
      "class": "violence",
      "highlights": [
        {"text": "find you and hurt you", "offset": 7, "length": 21, "salience": 0.92}
      ],
      "model_rationale": "Direct threat of physical harm toward an individual."
    }
  ],
  "decision_path": ["rule_engine:pass", "tier1_ml:violence:0.87", "fusion:remove"],
  "processing_ms": 78,
  "model_version": "v2.4.1-20260401"
}
```

### 4.2 Synchronous Media Classification

```
POST /v1/classify/media
Content-Type: multipart/form-data
```

**Form fields**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `file` | binary | Yes | Image (JPEG, PNG, WebP) or video (MP4, WebM) up to 50 MB |
| `metadata` | JSON string | Yes | Same schema as text metadata |
| `options` | JSON string | No | Same schema as text options |
| `accompanying_text` | string | No | Caption or surrounding text context |

**Response**: Same schema as text classification, with additional media-specific fields:

```json
{
  "trace_id": "trc_4a5b6c7d",
  "action": "queue_for_review",
  "scores": [ "..." ],
  "media_signals": {
    "perceptual_hash": "ph_a1b2c3d4e5f6",
    "known_hash_match": false,
    "nsfw_score": 0.62,
    "ocr_text_detected": true,
    "ocr_text": "Buy cheap pills now!",
    "keyframes_analyzed": 12
  }
}
```

### 4.3 Asynchronous Classification

```
POST /v1/classify/async
```

For large media or when the caller does not need an immediate decision.

**Request**: Same as sync, plus:

```json
{
  "content": { "..." },
  "callback": {
    "url": "https://your-service.example.com/webhooks/abuse",
    "headers": {
      "X-Webhook-Secret": "whsec_abc123"
    }
  },
  "priority": "high"
}
```

**Response** (`202 Accepted`):

```json
{
  "trace_id": "trc_8e9f0a1b",
  "status": "queued",
  "estimated_completion_ms": 15000,
  "poll_url": "/v1/decisions/trc_8e9f0a1b"
}
```

### 4.4 Decision Lookup

```
GET /v1/decisions/{trace_id}
```

**Response** (`200 OK`):

```json
{
  "trace_id": "trc_8e9f0a1b",
  "status": "completed",
  "action": "remove",
  "scores": [ "..." ],
  "decision_path": [ "..." ],
  "created_at": "2026-04-08T14:30:00.000Z",
  "completed_at": "2026-04-08T14:30:12.456Z",
  "human_review": {
    "required": true,
    "status": "pending",
    "assigned_to": null,
    "sla_deadline": "2026-04-08T18:30:00.000Z"
  }
}
```

### 4.5 Appeals

```
POST /v1/appeals
```

**Request**:

```json
{
  "trace_id": "trc_7f3a2b1c",
  "user_id": "usr_abc123",
  "reason": "This was a quote from a movie, not a real threat.",
  "supporting_context": "The comment was in a movie discussion thread."
}
```

**Response** (`201 Created`):

```json
{
  "appeal_id": "apl_1a2b3c4d",
  "trace_id": "trc_7f3a2b1c",
  "status": "submitted",
  "estimated_review_hours": 24,
  "created_at": "2026-04-08T15:00:00.000Z"
}
```

### 4.6 Threshold Configuration

```
GET /v1/config/thresholds
```

```json
{
  "version": "cfg_20260408_001",
  "classes": {
    "hate": {
      "soft_limit": 0.30,
      "queue": 0.55,
      "block": 0.80
    },
    "violence": {
      "soft_limit": 0.25,
      "queue": 0.50,
      "block": 0.75
    },
    "spam": {
      "soft_limit": 0.40,
      "queue": 0.60,
      "block": 0.85
    },
    "csam": {
      "queue": 0.01,
      "block": 0.50
    }
  },
  "updated_at": "2026-04-07T10:00:00.000Z",
  "updated_by": "policy_team"
}
```

```
PUT /v1/config/thresholds
```

Requires `admin:thresholds:write` permission. Changes are versioned, audited, and take effect after propagation (~30s).

---

## 5. Webhook Callbacks

When an async classification completes, the system sends:

```
POST {callback.url}
X-Webhook-Secret: whsec_abc123
X-Signature: sha256=<hmac_of_body>
Content-Type: application/json
```

```json
{
  "event": "classification.completed",
  "trace_id": "trc_8e9f0a1b",
  "action": "remove",
  "scores": [ "..." ],
  "completed_at": "2026-04-08T14:30:12.456Z"
}
```

**Retry policy**: 3 attempts with exponential backoff (1s, 10s, 60s). After exhaustion, event is available via poll endpoint for 7 days.

---

## 6. Error Handling

### 6.1 Error Response Schema

```json
{
  "error": {
    "code": "CONTENT_TOO_LARGE",
    "message": "Media file exceeds the 50 MB limit.",
    "trace_id": "trc_err_123",
    "details": {
      "max_bytes": 52428800,
      "received_bytes": 78643200
    }
  }
}
```

### 6.2 Error Codes

| HTTP Status | Code | Description |
|-------------|------|-------------|
| 400 | `INVALID_REQUEST` | Malformed JSON, missing required fields |
| 400 | `UNSUPPORTED_MEDIA_TYPE` | File type not supported |
| 400 | `CONTENT_TOO_LARGE` | Payload exceeds size limit |
| 401 | `UNAUTHORIZED` | Missing or invalid auth token |
| 403 | `FORBIDDEN` | Insufficient permissions |
| 404 | `NOT_FOUND` | Trace ID or resource not found |
| 409 | `DUPLICATE_APPEAL` | Appeal already exists for this trace |
| 429 | `RATE_LIMITED` | Rate limit exceeded; retry after `Retry-After` header |
| 500 | `INTERNAL_ERROR` | Unexpected server error |
| 503 | `SERVICE_DEGRADED` | System in degraded mode; partial results returned |

---

## 7. Rate Limits

| Tier | Sync RPS | Async RPS | Burst | Daily Quota |
|------|----------|-----------|-------|-------------|
| Free | 10 | 5 | 20 | 10,000 |
| Standard | 100 | 50 | 200 | 500,000 |
| Enterprise | 5,000 | 2,000 | 10,000 | Unlimited |

Rate limit headers on every response:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 87
X-RateLimit-Reset: 1712588400
Retry-After: 2  (only on 429)
```

---

## 8. SDK Examples

### 8.1 Python

```python
import abuse_classifier as ac

client = ac.Client(api_key="sk_live_...", base_url="https://abuse-api.example.com")

result = client.classify(
    text="I will find you and hurt you",
    content_type="comment",
    user_id="usr_abc123",
    classes=["hate", "violence", "spam"],
    return_explanations=True,
)

print(f"Action: {result.action}")
print(f"Violence score: {result.scores['violence']}")
if result.action == "remove":
    print(f"Reason: {result.explanations[0].model_rationale}")
```

### 8.2 Node.js

```javascript
const { AbuseClassifier } = require("@trust-safety/abuse-classifier");

const client = new AbuseClassifier({
  apiKey: "sk_live_...",
  baseUrl: "https://abuse-api.example.com",
});

const result = await client.classify({
  text: "I will find you and hurt you",
  contentType: "comment",
  userId: "usr_abc123",
});

if (result.action === "remove") {
  await removeContent(contentId);
  await notifyUser(result.explanations[0].modelRationale);
}
```

### 8.3 cURL

```bash
curl -X POST https://abuse-api.example.com/v1/classify \
  -H "Authorization: Bearer sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{
    "content": {
      "text": "I will find you and hurt you",
      "content_type": "comment"
    },
    "metadata": {
      "user_id": "usr_abc123"
    },
    "options": {
      "classes": ["hate", "violence", "spam"]
    }
  }'
```

---

*Next: [03-DATA-AND-TRAINING.md](./03-DATA-AND-TRAINING.md) — Data pipelines, labeling, model training, evaluation*
