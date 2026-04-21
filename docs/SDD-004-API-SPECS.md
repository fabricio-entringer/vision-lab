# SDD-004: VisionLab API Specifications

## Document Information
| Field | Value |
|-------|-------|
| **Document ID** | SDD-004-API-SPECS |
| **Version** | 1.0.0 |
| **Status** | Draft |
| **Last Updated** | 2025-04-21 |
| **Author** | VisionLab Team |

---

## 1. API Overview

### 1.1 Design Philosophy

VisionLab follows a **RESTful API design** with **WebSocket support** for real-time communication. The API is built on FastAPI with automatic OpenAPI documentation.

| Aspect | Choice |
|--------|--------|
| **Protocol** | REST + WebSocket |
| **Format** | JSON (application/json) |
| **Auth** | API Key (Phase 1), JWT/OAuth2 (Phase 2) |
| **Versioning** | URL path (`/api/v1/`) |
| **Documentation** | OpenAPI/Swagger (auto-generated) |

### 1.2 Base URL & Versioning

| Environment | URL |
|-------------|-----|
| **Development** | `http://localhost:8000/api/v1` |
| **Staging** | `https://staging.visionlab.dev/api/v1` |
| **Production** | `https://api.visionlab.dev/api/v1` |

API version is embedded in the path. Breaking changes require a new version (`/api/v2/`).

---

## 2. Authentication

### 2.1 Phase 1: API Key (MVP)

No authentication required during MVP. All endpoints are open.

### 2.2 Phase 2: JWT + OAuth2 (Planned)

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

| Method | Flow | Scopes |
|--------|------|--------|
| **OAuth2 Authorization Code** | Standard 3-legged flow | `read:jobs`, `write:jobs`, `admin` |
| **JWT Bearer Token** | Access token + Refresh token | Scoped access |
| **API Key** | Service-to-service | Full access per key |

---

## 3. Rate Limiting

| Tier | Requests/min | Burst | Concurrent Jobs |
|------|-------------|-------|-----------------|
| **Anonymous (MVP)** | 30 | 5 | 2 |
| **Authenticated** | 100 | 10 | 5 |
| **Admin** | 1000 | 50 | 20 |

Headers returned on every response:
```
X-RateLimit-Limit: 30
X-RateLimit-Remaining: 25
X-RateLimit-Reset: 1650000000
```

When exceeded:
```
HTTP 429 Too Many Requests
Retry-After: 30
Content-Type: application/json

{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Please retry after 30 seconds.",
    "retry_after_seconds": 30
  }
}
```

---

## 4. Error Handling

### 4.1 Error Response Format

All errors follow a consistent structure:

```typescript
interface ErrorResponse {
  error: {
    code: string;          // Machine-readable error code
    message: string;        // Human-readable description
    details?: Record<string, any>;  // Optional context
  };
  request_id: string;      // For support tracing
  timestamp: string;       // ISO 8601
}
```

### 4.2 Standard Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `INVALID_REQUEST` | 400 | Malformed request or validation failure |
| `UNAUTHORIZED` | 401 | Missing or invalid credentials (Phase 2) |
| `FORBIDDEN` | 403 | Insufficient permissions (Phase 2) |
| `NOT_FOUND` | 404 | Resource does not exist |
| `CONFLICT` | 409 | Resource state conflict |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Unexpected server error |
| `SERVICE_UNAVAILABLE` | 503 | System temporarily unavailable |
| `MODEL_ERROR` | 503 | Model inference failure |
| `TIMEOUT` | 504 | Operation timed out |

### 4.3 Error Examples

**Validation Error (400):**
```json
{
  "error": {
    "code": "INVALID_REQUEST",
    "message": "Validation error",
    "details": {
      "field_errors": [
        {
          "field": "prompt",
          "message": "Prompt must be between 10 and 500 characters",
          "value": "hi"
        }
      ]
    }
  },
  "request_id": "req_a1b2c3d4e5f6",
  "timestamp": "2025-04-21T10:30:00Z"
}
```

**Job Not Found (404):**
```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Job 'job_abc123' not found"
  },
  "request_id": "req_a1b2c3d4e5f6",
  "timestamp": "2025-04-21T10:30:00Z"
}
```

---

## 5. Endpoint Catalog

### 5.1 Summary Table

| Method | Endpoint | Description | Auth | Status |
|--------|----------|-------------|------|--------|
| `POST` | `/api/v1/montage/jobs` | Create a new montage job | None | ✅ MVP |
| `GET` | `/api/v1/jobs/{job_id}` | Get job status and details | None | ✅ MVP |
| `GET` | `/api/v1/jobs/{job_id}/result` | Get job result image | None | ✅ MVP |
| `DELETE` | `/api/v1/jobs/{job_id}` | Cancel a pending job | None | ✅ MVP |
| `GET` | `/api/v1/features` | List available features | None | ✅ MVP |
| `GET` | `/api/v1/health` | Health check | None | ✅ MVP |
| `WS` | `/ws/jobs/{job_id}` | Real-time job progress | None | ✅ MVP |
| `POST` | `/api/v1/segmentation/jobs` | Create segmentation job | None | 🔲 Phase 2 |
| `POST` | `/api/v1/detection/jobs` | Create detection job | None | 🔲 Phase 2 |
| `POST` | `/api/v1/style-transfer/jobs` | Create style transfer job | None | 🔲 Phase 3 |

---

## 6. Endpoint Specifications

### 6.1 POST `/api/v1/montage/jobs`

Create a new image montage generation job.

**Request:**
- **Content-Type:** `multipart/form-data`

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `prompt` | string | Yes | Text prompt (10-500 chars) |
| `images` | File[] | Yes | Image files (1-10, JPG/PNG/WebP, max 10MB each) |
| `guidance_scale` | number | No | Diffusion guidance (default: 3.5, range: 1.0-20.0) |
| `num_steps` | integer | No | Inference steps (default: 28, range: 20-50) |
| `seed` | integer | No | Random seed for reproducibility |
| `output_format` | string | No | Output format: `png` or `jpg` (default: `png`) |

**Request Example (curl):**
```bash
curl -X POST http://localhost:8000/api/v1/montage/jobs \
  -F "prompt=A dreamy landscape blending ocean waves with mountain peaks at sunset" \
  -F "images=@photo1.jpg" \
  -F "images=@photo2.png" \
  -F "guidance_scale=4.0" \
  -F "num_steps=35"
```

**Success Response (201 Created):**
```json
{
  "job_id": "job_a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "status": "queued",
  "type": "montage",
  "created_at": "2025-04-21T10:30:00Z",
  "estimated_completion": "2025-04-21T10:30:30Z",
  "input_summary": {
    "image_count": 2,
    "prompt_length": 78,
    "parameters": {
      "guidance_scale": 4.0,
      "num_steps": 35,
      "output_format": "png"
    }
  }
}
```

**Error Responses:**

| Status | Code | Scenario |
|--------|------|----------|
| 400 | `INVALID_REQUEST` | Missing prompt or images |
| 400 | `VALIDATION_ERROR` | Invalid image format, size, or prompt length |
| 413 | `PAYLOAD_TOO_LARGE` | Total upload exceeds 100MB |
| 429 | `RATE_LIMIT_EXCEEDED` | Too many requests |
| 503 | `SERVICE_UNAVAILABLE` | Queue is full |

---

### 6.2 GET `/api/v1/jobs/{job_id}`

Get the current status and details of a job.

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `job_id` | string (UUID) | The unique job identifier |

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `include` | string | Optional: `input`, `progress` (comma-separated) |

**Success Response (200 OK):**
```json
{
  "job_id": "job_a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "type": "montage",
  "status": "processing",
  "progress": {
    "step": "inference",
    "percentage": 45,
    "current_step_total": 28,
    "current_step_done": 12
  },
  "created_at": "2025-04-21T10:30:00Z",
  "updated_at": "2025-04-21T10:30:15Z",
  "estimated_completion": "2025-04-21T10:30:30Z"
}
```

**Status Values:**

| Status | Description | Terminal? |
|--------|-------------|-----------|
| `queued` | Job is waiting in queue | No |
| `processing` | Job is actively being processed | No |
| `completed` | Job finished successfully | Yes |
| `failed` | Job failed with error | Yes |
| `cancelled` | Job was cancelled by user | Yes |

**Error Responses:**

| Status | Code | Scenario |
|--------|------|----------|
| 404 | `NOT_FOUND` | Job ID does not exist |

---

### 6.3 GET `/api/v1/jobs/{job_id}/result`

Get the result of a completed job.

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `job_id` | string (UUID) | The unique job identifier |

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `format` | string | Optional: `png` or `jpg` (default: original format) |
| `download` | boolean | Optional: if `true`, forces file download |

**Success Response (200 OK):**

When `download=false` (default):
```json
{
  "job_id": "job_a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "status": "completed",
  "result": {
    "image_url": "https://storage.visionlab.dev/results/job_a1b2c3d4/output.png?sig=abc123",
    "expires_at": "2025-04-22T10:30:00Z",
    "dimensions": {
      "width": 1024,
      "height": 1024
    },
    "file_size": 2458624,
    "format": "png",
    "seed_used": 42
  },
  "completed_at": "2025-04-21T10:30:28Z",
  "processing_time_ms": 28000
}
```

When `download=true`:
```
HTTP/1.1 200 OK
Content-Type: image/png
Content-Disposition: attachment; filename="montage_a1b2c3d4.png"
Content-Length: 2458624

[binary image data]
```

**Error Responses:**

| Status | Code | Scenario |
|--------|------|----------|
| 404 | `NOT_FOUND` | Job ID does not exist |
| 410 | `GONE` | Result has expired (past retention period) |
| 409 | `CONFLICT` | Job not yet completed (status: queued/processing) |

---

### 6.4 DELETE `/api/v1/jobs/{job_id}`

Cancel a pending or processing job.

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `job_id` | string (UUID) | The unique job identifier |

**Success Response (200 OK):**
```json
{
  "job_id": "job_a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "status": "cancelled",
  "cancelled_at": "2025-04-21T10:30:05Z",
  "message": "Job cancelled successfully"
}
```

**Error Responses:**

| Status | Code | Scenario |
|--------|------|----------|
| 404 | `NOT_FOUND` | Job ID does not exist |
| 409 | `CONFLICT` | Job already in terminal state (completed/failed/cancelled) |

---

### 6.5 GET `/api/v1/features`

List all available features and their status.

**Success Response (200 OK):**
```json
{
  "features": [
    {
      "id": "montage",
      "name": "Image Montage Generation",
      "description": "Generate composed images from multiple inputs and a text prompt",
      "model": "FLUX.2-klein-4B",
      "status": "available",
      "endpoint": "/api/v1/montage/jobs",
      "max_images": 10,
      "max_prompt_length": 500,
      "min_prompt_length": 10,
      "supported_formats": ["jpg", "png", "webp"],
      "max_file_size_mb": 10,
      "parameters": {
        "guidance_scale": {
          "type": "number",
          "default": 3.5,
          "min": 1.0,
          "max": 20.0
        },
        "num_steps": {
          "type": "integer",
          "default": 28,
          "min": 20,
          "max": 50
        },
        "output_format": {
          "type": "string",
          "enum": ["png", "jpg"],
          "default": "png"
        }
      }
    },
    {
      "id": "segmentation",
      "name": "Image Segmentation",
      "description": "Segment objects in an image using SAM",
      "model": "SAM-vit-huge",
      "status": "unavailable",
      "note": "Coming in Phase 2"
    },
    {
      "id": "detection",
      "name": "Object Detection",
      "description": "Detect and localize objects using YOLO",
      "model": "YOLOv8",
      "status": "unavailable",
      "note": "Coming in Phase 2"
    }
  ],
  "total": 3,
  "available": 1
}
```

---

### 6.6 GET `/api/v1/health`

Health check endpoint for monitoring.

**Success Response (200 OK):**
```json
{
  "status": "healthy",
  "timestamp": "2025-04-21T10:30:00Z",
  "version": "1.0.0",
  "components": {
    "database": {
      "status": "healthy",
      "latency_ms": 2
    },
    "redis": {
      "status": "healthy",
      "latency_ms": 1
    },
    "storage": {
      "status": "healthy",
      "available_space_gb": 48
    },
    "model": {
      "status": "healthy",
      "name": "FLUX.2-klein-4B",
      "loaded": true,
      "gpu_memory_used_gb": 4.2
    }
  }
}
```

---

## 7. WebSocket Protocol

### 7.1 Connection

```
ws://localhost:8000/ws/jobs/{job_id}
wss://api.visionlab.dev/ws/jobs/{job_id}  (production)
```

### 7.2 Connection Lifecycle

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server

    C->>S: WS Connect /ws/jobs/{job_id}
    S->>C: { "type": "connected", "job_id": "..." }

    loop Job Progress
        S->>C: { "type": "progress", "status": "processing", "percentage": 25 }
        S->>C: { "type": "progress", "status": "processing", "percentage": 50 }
        S->>C: { "type": "progress", "status": "processing", "percentage": 75 }
    end

    S->>C: { "type": "completed", "status": "completed", "result": {...} }
    C->>S: Close
```

### 7.3 Message Types

#### Server → Client Messages

**Connection Established:**
```json
{
  "type": "connected",
  "job_id": "job_a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "timestamp": "2025-04-21T10:30:00Z"
}
```

**Progress Update:**
```json
{
  "type": "progress",
  "job_id": "job_a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "status": "processing",
  "step": "inference",
  "percentage": 45,
  "eta_seconds": 15,
  "timestamp": "2025-04-21T10:30:15Z"
}
```

**Job Completed:**
```json
{
  "type": "completed",
  "job_id": "job_a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "status": "completed",
  "result": {
    "image_url": "https://storage.visionlab.dev/results/job_a1b2c3d4/output.png?sig=abc123",
    "dimensions": { "width": 1024, "height": 1024 },
    "file_size": 2458624,
    "format": "png"
  },
  "processing_time_ms": 28000,
  "timestamp": "2025-04-21T10:30:28Z"
}
```

**Job Failed:**
```json
{
  "type": "failed",
  "job_id": "job_a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "status": "failed",
  "error": {
    "code": "MODEL_ERROR",
    "message": "Inference failed: CUDA out of memory"
  },
  "timestamp": "2025-04-21T10:30:20Z"
}
```

**Ping (Keep-alive, every 30s):**
```json
{
  "type": "ping",
  "timestamp": "2025-04-21T10:30:30Z"
}
```

**Pong (Keep-alive response):**
```json
{
  "type": "pong",
  "timestamp": "2025-04-21T10:30:30Z"
}
```

### 7.4 WebSocket Protocol Rules

| Rule | Value |
|------|-------|
| **Connection timeout** | 60 seconds (no job activity) |
| **Ping interval** | 30 seconds |
| **Pong timeout** | 10 seconds (disconnect if no pong) |
| **Max message size** | 1MB |
| **Reconnection** | Exponential backoff (1s, 2s, 4s, max 30s) |
| **Maximum concurrent connections per job** | 5 |

---

## 8. Pagination

For endpoints that return lists (future phases):

```
GET /api/v1/jobs?page=1&page_size=20&sort=-created_at
```

**Response:**
```json
{
  "items": [...],
  "total": 150,
  "page": 1,
  "page_size": 20,
  "total_pages": 8,
  "has_next": true,
  "has_prev": false
}
```

---

## 9. CORS Configuration

| Setting | Value |
|---------|-------|
| **Allowed Origins** | `http://localhost:3000`, `https://visionlab.dev` (configured) |
| **Allowed Methods** | `GET`, `POST`, `DELETE`, `OPTIONS` |
| **Allowed Headers** | `Content-Type`, `Authorization`, `X-Request-ID` |
| **Max Age** | 600 seconds |

---

## 10. References

| Reference | Description |
|-----------|-------------|
| [SDD-002](./SDD-002-ARCHITECTURE.md) | Architecture specification |
| [FastAPI Documentation](https://fastapi.tiangolo.com/) | FastAPI reference |
| [OpenAPI Specification](https://swagger.io/specification/) | OpenAPI 3.0 reference |

---

## 11. Change Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | 2025-04-21 | VisionLab Team | Initial API specification |