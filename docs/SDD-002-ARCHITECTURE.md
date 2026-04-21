# SDD-002: VisionLab Architecture Specification

## Document Information
| Field | Value |
|-------|-------|
| **Document ID** | SDD-002-ARCHITECTURE |
| **Version** | 1.0.0 |
| **Status** | Draft |
| **Last Updated** | 2025-04-21 |

---

## 1. Architecture Overview

### 1.1 High-Level Architecture

VisionLab follows a **modular full-stack architecture** with clear separation of concerns between the client interface, API layer, and model inference services.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              VISIONLAB ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────┐      HTTPS/REST       ┌─────────────────────┐     │
│  │                     │◄─────────────────────►│                     │     │
│  │     FRONTEND        │      WebSocket        │      BACKEND        │     │
│  │   (React/Vue.js)    │◄─────────────────────►│      (FastAPI)      │     │
│  │                     │                       │                     │     │
│  │  • Tab Navigation   │                       │  • API Endpoints    │     │
│  │  • Image Upload     │                       │  • Orchestration    │     │
│  │  • Prompt Input     │                       │  • Model Inference  │     │
│  │  • Result Display   │                       │  • File Handling    │     │
│  │                     │                       │                     │     │
│  └─────────────────────┘                       └──────────┬──────────┘     │
│           │                                               │                │
│           │                                               │                │
│           ▼                                               ▼                │
│  ┌─────────────────────┐                       ┌─────────────────────┐     │
│  │   Static Assets     │                       │   Model Services    │     │
│  │   (CDN/Storage)     │                       │                     │     │
│  └─────────────────────┘                       │  • FLUX.2-klein-4B  │     │
│                                                │  • SAM (Seg.)       │     │
│                                                │  • YOLO (Detect)    │     │
│                                                │  • CLIP/StableDiff  │     │
│                                                └─────────────────────┘     │
│                                                          │                  │
│                                                          ▼                  │
│                                                ┌─────────────────────┐     │
│                                                │   External APIs     │     │
│                                                │  • HuggingFace      │     │
│                                                │  • Replicate        │     │
│                                                │  • Cloud GPU        │     │
│                                                └─────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Component Diagram (C4 Model - Level 2)

```mermaid
graph TB
    subgraph Client["Client Layer"]
        Browser[Web Browser]
    end

    subgraph Frontend["Frontend Layer"]
        React[React Application]
        Router[React Router]
        Components[Feature Components]
        Assets[Static Assets]
    end

    subgraph Backend["Backend Layer"]
        API[FastAPI Application]
        RouterF[Feature Routers]
        Services[Business Services]
        ML[ML Pipeline Service]
        Storage[File Storage Service]
    end

    subgraph External["External Services"]
        HF[HuggingFace Hub]
        Replicate[Replicate API]
        CDN[CDN/Static Hosting]
    end

    Browser -->|HTTP/WebSocket| React
    React --> Router
    Router --> Components
    Components --> Assets
    React -->|API Calls| API
    API --> RouterF
    RouterF --> Services
    Services --> ML
    Services --> Storage
    ML --> HF
    ML --> Replicate
    Assets --> CDN
```

---

## 2. Technology Stack

### 2.1 Recommended Stack Overview

| Layer | Technology | Alternative | Selection |
|-------|------------|-------------|-----------|
| **Frontend** | React 18+ | Vue 3, Svelte | React |
| **Frontend Build** | Vite | Create React App, Next.js | Vite |
| **Styling** | Tailwind CSS | Styled Components, Chakra UI | Tailwind |
| **State Management** | React Query | Redux, Zustand | React Query |
| **Backend** | FastAPI | Node.js/Express, Django | FastAPI |
| **ORM** | SQLModel | SQLAlchemy, Prisma | SQLModel |
| **Database** | PostgreSQL | SQLite, MongoDB | PostgreSQL |
| **File Storage** | Local/MinIO | AWS S3, Cloudflare R2 | Local (Dev) → S3 (Prod) |
| **Task Queue** | Celery + Redis | RQ, Dramatiq | Celery + Redis |
| **Container** | Docker | Podman | Docker |
| **Orchestration** | Docker Compose | Kubernetes | Docker Compose |

### 2.2 Justification

#### Frontend: React + Vite + Tailwind

**Why React?**
- Industry standard with massive ecosystem
- Excellent TypeScript support
- Component-based architecture fits feature tabs model
- Strong community for AI/ML visualization libraries

**Why Vite?**
- 10-100x faster build times than webpack [Source: Vite documentation]
- Native ESM support for modern development
- Optimized production builds

**Why Tailwind CSS?**
- Rapid UI development for prototyping
- Consistent design system via configuration
- Small production bundle size

#### Backend: FastAPI + SQLModel

**Why FastAPI over Node.js?**

| Criteria | FastAPI | Node.js/Express |
|----------|---------|-----------------|
| ML Ecosystem | Native Python libraries (PyTorch, Transformers) | Requires Python subprocess or APIs |
| Async Support | Native async/await | Event-driven callbacks |
| Type Safety | Pydantic validation built-in | Requires additional libraries |
| Performance | Comparable for I/O bound tasks | Comparable |
| Learning Curve | Steeper for non-Python devs | Lower |
| AI Integration | Direct model loading | API-based only |

**Decision Rationale**:
Since VisionLab involves direct model inference and GPU-intensive operations, Python provides native integration with the ML ecosystem. FastAPI offers:
- Automatic OpenAPI documentation
- Pydantic-based request/response validation
- Native async support for concurrent requests
- Excellent WebSocket support for real-time features

[Source: FastAPI vs Node.js comparison, 2025]

---

## 3. System Components

### 3.1 Frontend Architecture

#### Directory Structure

```
frontend/
├── src/
│   ├── components/           # Reusable UI components
│   │   ├── layout/          # App shell, navigation
│   │   ├── shared/          # Buttons, inputs, cards
│   │   └── features/        # Feature-specific components
│   ├── features/            # Feature modules (co-located)
│   │   ├── montage/         # Image Montage Generation
│   │   │   ├── components/
│   │   │   ├── api/
│   │   │   ├── hooks/
│   │   │   └── types/
│   │   ├── segmentation/    # SAM-based segmentation
│   │   ├── detection/       # YOLO object detection
│   │   ├── style-transfer/  # Neural style transfer
│   │   └── shared/          # Cross-feature utilities
│   ├── hooks/               # Global custom hooks
│   ├── lib/                 # Utilities, API clients
│   ├── types/               # TypeScript definitions
│   └── main.tsx             # Application entry
├── public/                  # Static assets
└── vite.config.ts
```

#### State Management Strategy

```
┌─────────────────────────────────────────────────────────────┐
│                    STATE ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌──────────────────┐    ┌──────────────────┐             │
│   │   Global State   │    │   Server State   │             │
│   │   (Zustand)      │    │   (React Query)  │             │
│   ├──────────────────┤    ├──────────────────┤             │
│   │ • UI preferences │    │ • Job status     │             │
│   │ • Theme settings │    │ • Results        │             │
│   │ • Active tab     │    │ • Cached images  │             │
│   └──────────────────┘    └──────────────────┘             │
│                                                              │
│   ┌──────────────────┐    ┌──────────────────┐             │
│   │  Local State     │    │   URL State      │             │
│   │  (useState)      │    │   (React Router) │             │
│   ├──────────────────┤    ├──────────────────┤             │
│   │ • Form inputs    │    │ • Current tab    │             │
│   │ • Modal open     │    │ • Query params   │             │
│   │ • Temp selections│    │ • Shareable state│             │
│   └──────────────────┘    └──────────────────┘             │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Backend Architecture

#### Directory Structure

```
backend/
├── app/
│   ├── __init__.py
│   ├── main.py              # FastAPI application entry
│   ├── config.py            # Configuration management
│   ├── dependencies.py      # FastAPI dependencies
│   ├── api/
│   │   ├── __init__.py
│   │   ├── v1/
│   │   │   ├── __init__.py
│   │   │   ├── router.py    # API v1 aggregator
│   │   │   ├── endpoints/
│   │   │   │   ├── montage.py
│   │   │   │   ├── segmentation.py
│   │   │   │   ├── detection.py
│   │   │   │   └── style_transfer.py
│   │   │   └── websockets/
│   │   │       └── progress.py
│   │   └── deps.py
│   ├── core/
│   │   ├── security.py
│   │   ├── exceptions.py
│   │   └── logging.py
│   ├── models/
│   │   ├── __init__.py
│   │   ├── database.py      # SQLModel base
│   │   ├── job.py           # Job tracking
│   │   └── user.py          # User model (phase 2)
│   ├── services/
│   │   ├── __init__.py
│   │   ├── montage_service.py
│   │   ├── model_loader.py
│   │   ├── inference_queue.py
│   │   └── storage_service.py
│   ├── ml/
│   │   ├── __init__.py
│   │   ├── pipelines/
│   │   │   ├── montage_pipeline.py
│   │   │   ├── segmentation_pipeline.py
│   │   │   └── detection_pipeline.py
│   │   ├── models/
│   │   │   └── flux_loader.py
│   │   └── utils/
│   │       └── image_processing.py
│   └── celery_app.py        # Background task runner
├── alembic/                 # Database migrations
├── tests/
├── scripts/
└── Dockerfile
```

### 3.3 ML Pipeline Architecture

```mermaid
flowchart TB
    subgraph Input["Input Processing"]
        Upload[File Upload]
        Validate[Validation]
        Preprocess[Preprocessing]
    end

    subgraph Queue["Task Management"]
        Submit[Submit Job]
        Redis[(Redis Queue)]
        Worker[Celery Worker]
    end

    subgraph Inference["Model Inference"]
        Load[Load Model]
        Run[Run Inference]
        Postprocess[Postprocess]
    end

    subgraph Output["Output Handling"]
        Save[Save Result]
        Notify[Notify Client]
        Cleanup[Cleanup Temp Files]
    end

    Upload --> Validate --> Preprocess --> Submit
    Submit --> Redis --> Worker --> Load --> Run --> Postprocess --> Save --> Notify --> Cleanup
```

---

## 4. Communication Patterns

### 4.1 API Design: REST vs GraphQL vs WebSocket

| Pattern | Use Case | Implementation |
|---------|----------|----------------|
| **REST** | CRUD operations, synchronous requests | Primary API design |
| **WebSocket** | Real-time progress updates, streaming results | Job status notifications |
| **SSE** | Server-sent events for long-running jobs | Alternative to WebSocket |

#### Decision Rationale

**REST for Primary API:**
- Simple, universally understood
- Excellent tooling (OpenAPI/Swagger)
- Easy caching strategies
- Fits CRUD operations perfectly

**WebSocket for Real-time:**
- Bi-directional communication for progress updates
- Lower overhead than polling for job status
- Enables future features (real-time collaboration, live inference)

**No GraphQL (for MVP):**
- Adds unnecessary complexity for focused feature set
- RESTful endpoints sufficient for current requirements
- Can migrate to GraphQL if features grow significantly

### 4.2 API Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      API GATEWAY                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  REST Endpoints                    WebSocket                 │
│  ─────────────                     ─────────                 │
│                                                              │
│  POST   /api/v1/jobs              /ws/jobs/{job_id}          │
│  GET    /api/v1/jobs/{id}         /ws/features               │
│  GET    /api/v1/results/{id}                                 │
│  DELETE /api/v1/jobs/{id}                                    │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                      Feature Routers                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  /api/v1/montage           Image Montage Generation          │
│  /api/v1/segmentation      SAM-based Segmentation            │
│  /api/v1/detection         YOLO Object Detection             │
│  /api/v1/style-transfer    Neural Style Transfer             │
│  /api/v1/video             Video Processing                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Deployment Architecture

### 5.1 Development Environment

```yaml
# docker-compose.dev.yml
version: '3.8'
services:
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    volumes:
      - ./frontend:/app
      - /app/node_modules
    environment:
      - VITE_API_URL=http://localhost:8000

  backend:
    build: ./backend
    ports:
      - "8000:8000"
    volumes:
      - ./backend:/app
    environment:
      - DATABASE_URL=postgresql://user:pass@postgres:5432/visionlab
      - REDIS_URL=redis://redis:6379

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: visionlab
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass

  redis:
    image: redis:7-alpine

  celery:
    build: ./backend
    command: celery -A app.celery_app worker --loglevel=info
    depends_on:
      - redis
      - postgres
```

### 5.2 Production Deployment Patterns

#### Pattern A: HuggingFace Spaces (Recommended for MVP)

**When to use:**
- Quick deployment for demonstration
- Limited budget
- HuggingFace ecosystem integration

```
┌─────────────────────────────────────────────────────────────┐
│                  HUGGINGFACE SPACES                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              Docker Space                             │  │
│  │  ┌──────────────┐      ┌──────────────────────────┐  │  │
│  │  │   Gradio/    │      │    FastAPI Backend       │  │  │
│  │  │   React UI   │◄────►│  + Model Inference       │  │  │
│  │  └──────────────┘      └──────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
│                         │                                    │
│                         ▼                                    │
│  ┌───────────────────────────────────────────────────────┐  │
│  │         HuggingFace Hub (Model Cache)                 │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

#### Pattern B: Self-Hosted Docker

**When to use:**
- Full control over infrastructure
- Custom hardware (GPUs)
- Production-scale deployment

```
┌─────────────────────────────────────────────────────────────┐
│                      SELF-HOSTED                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │   Nginx/        │  │   Backend       │  │   Worker     │ │
│  │   Traefik       │◄─┤   Containers    │◄─┤   Nodes      │ │
│  │   (Load Bal.)   │  │   (FastAPI)     │  │   (Celery)   │ │
│  └─────────────────┘  └─────────────────┘  └──────────────┘ │
│           │                       │              │          │
│           ▼                       ▼              ▼          │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │   Static Assets │  │   PostgreSQL    │  │    Redis     │ │
│  │   (CDN/S3)      │  │   (Primary DB)  │  │   (Queue)    │ │
│  └─────────────────┘  └─────────────────┘  └──────────────┘ │
│                                                              │
│  GPU Support: NVIDIA Docker Runtime (nvidia-docker2)        │
└─────────────────────────────────────────────────────────────┘
```

#### Pattern C: Hybrid (Recommended for Scale)

**When to use:**
- Separate compute-heavy inference from API serving
- Cost optimization (scale API independently from workers)
- Multi-cloud deployments

```
┌─────────────────────────────────────────────────────────────┐
│                      CLOUD ARCHITECTURE                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  FRONTEND TIER                    BACKEND TIER              │
│  ────────────                     ───────────               │
│                                                              │
│  ┌─────────────┐                 ┌──────────────────┐       │
│  │   Vercel/   │◄───────────────►│   API Gateway    │       │
│  │   Cloudflare│   HTTPS/WSS     │   (AWS/GCP)      │       │
│  └─────────────┘                 └────────┬─────────┘       │
│                                           │                  │
│                              ┌────────────┼────────────┐    │
│                              ▼            ▼            ▼    │
│                         ┌────────┐   ┌────────┐   ┌────────┐│
│                         │Stateless│   │Stateless│   │Inference│
│                         │ API 1  │   │ API 2  │   │ Workers │
│                         └────────┘   └────────┘   └────────┘│
│                              │            │            │    │
│                              └────────────┴────────────┘    │
│                                           │                  │
│                              ┌────────────┼────────────┐    │
│                              ▼            ▼            ▼    │
│                         ┌────────┐   ┌────────┐   ┌────────┐│
│                         │PostgreSQL│  │  Redis  │   │  S3    ││
│                         └────────┘   └────────┘   └────────┘│
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 5.3 Deployment Environments

| Environment | Infrastructure | Purpose | Scale |
|-------------|----------------|---------|-------|
| **Local** | Docker Compose | Development, testing | Single machine |
| **Staging** | HuggingFace Spaces / VPS | Pre-production testing | Shared resources |
| **Production** | Cloud (AWS/GCP/Azure) | Live application | Auto-scaling |

---

## 6. Infrastructure Components

### 6.1 Service Dependencies

```
┌─────────────────────────────────────────────────────────────────┐
│                    SERVICE DEPENDENCY GRAPH                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Frontend                                                        │
│     └─► Backend API                                              │
│            ├─► PostgreSQL (persistent storage)                   │
│            ├─► Redis (caching & queuing)                         │
│            ├─► Celery Workers (background tasks)                 │
│            └─► Model Services                                    │
│                   ├─► HuggingFace Hub (model loading)            │
│                   ├─► Local GPU (inference)                      │
│                   └─► External APIs (optional fallback)          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 Resource Requirements

#### Minimum Requirements (MVP)

| Component | CPU | RAM | Storage | Notes |
|-----------|-----|-----|---------|-------|
| Frontend | 0.5 | 512MB | 1GB | Static build |
| Backend | 1 | 2GB | 5GB | Without model |
| PostgreSQL | 0.5 | 1GB | 10GB | Including backups |
| Redis | 0.25 | 512MB | - | Ephemeral |
| Worker | 2 | 4GB | 10GB | FLUX.2-klein (CPU) |

#### Recommended (Production)

| Component | CPU | RAM | Storage | GPU | Notes |
|-----------|-----|-----|---------|-----|-------|
| Frontend | 1 | 1GB | 5GB | - | CDN-backed |
| Backend | 2 | 4GB | 20GB | - | Load balanced |
| PostgreSQL | 2 | 4GB | 100GB | - | SSD required |
| Redis | 1 | 2GB | - | - | Cluster mode |
| Worker | 4 | 16GB | 50GB | RTX 4090 / A100 | For fast inference |

---

## 7. Security Architecture

### 7.1 Security Layers

```
┌─────────────────────────────────────────────────────────────┐
│                    SECURITY LAYERS                          │
├─────────────────────────────────────────────────────────────┤
│  Layer 1: Network                                           │
│  ───────────────                                            │
│  • HTTPS only (TLS 1.3)                                     │
│  • Rate limiting (100 req/min per IP)                       │
│  • DDoS protection (Cloudflare/AWS Shield)                  │
│                                                             │
│  Layer 2: Application                                       │
│  ───────────────────                                        │
│  • Input validation (Pydantic schemas)                      │
│  • File upload limits (10MB max, image types only)          │
│  • SQL injection prevention (SQLModel ORM)                  │
│  • XSS protection (React sanitization)                      │
│                                                             │
│  Layer 3: Authentication (Phase 2)                          │
│  ───────────────────────────────                            │
│  • OAuth 2.0 / OIDC                                         │
│  • JWT tokens with refresh                                  │
│  • Session management                                       │
│                                                             │
│  Layer 4: Infrastructure                                    │
│  ─────────────────────                                      │
│  • Secrets management (environment variables)               │
│  • Container security (non-root users)                      │
│  • Network segmentation                                     │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 Data Protection

| Data Type | Storage | Encryption | Retention |
|-----------|---------|------------|-----------|
| User Images | S3/MinIO | AES-256 at rest | 24 hours (configurable) |
| Generated Images | S3/MinIO | AES-256 at rest | 7 days |
| Job Metadata | PostgreSQL | Encrypted backups | 30 days |
| API Keys | Environment | - | N/A |

---

## 8. Monitoring and Observability

### 8.1 Observability Stack

```
┌─────────────────────────────────────────────────────────────┐
│                 OBSERVABILITY STACK                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Metrics                Logs                 Traces         │
│  ───────                ────                 ──────         │
│                                                              │
│  ┌─────────────┐       ┌─────────────┐      ┌─────────────┐│
│  │  Prometheus │       │   Loki      │      │   Jaeger    ││
│  │  + Grafana  │       │             │      │  / Zipkin   ││
│  └─────────────┘       └─────────────┘      └─────────────┘│
│                                                              │
│  Key Metrics:                                                │
│  • HTTP request latency (p50, p95, p99)                      │
│  • Model inference time                                     │
│  • Queue depth and processing rate                          │
│  • Error rates by endpoint                                  │
│  • GPU utilization (if applicable)                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 8.2 Health Checks

| Endpoint | Purpose | Frequency |
|----------|---------|-----------|
| `/health` | Liveness probe | 10s |
| `/health/ready` | Readiness probe | 10s |
| `/health/db` | Database connectivity | 30s |
| `/health/redis` | Cache connectivity | 30s |

---

## 9. Scalability Considerations

### 9.1 Horizontal Scaling Strategy

| Component | Scaling Approach | Bottleneck |
|-----------|------------------|------------|
| Frontend | Static CDN | None (stateless) |
| Backend API | Container replicas | Database connections |
| Workers | Celery worker pool | GPU/memory availability |
| Database | Read replicas, sharding | Write throughput |
| Redis | Cluster mode | Memory |

### 9.2 Performance Optimization

```
┌─────────────────────────────────────────────────────────────┐
│              PERFORMANCE OPTIMIZATIONS                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Model Loading                                            │
│     • Pre-load models on worker startup                     │
│     • Keep models in VRAM between requests                  │
│     • Model quantization (FP16, INT8) for faster inference  │
│                                                              │
│  2. Caching Strategy                                         │
│     • Cache model results for identical inputs              │
│     • Redis for job status and intermediate results         │
│     • CDN for static assets and generated images            │
│                                                              │
│  3. File Upload                                              │
│     • Direct-to-S3 upload with presigned URLs               │
│     • Chunk upload for large files                          │
│     • Image compression before processing                   │
│                                                              │
│  4. Async Processing                                         │
│     • Background jobs for all inference                     │
│     • WebSocket/SSE for real-time progress                  │
│     • Callbacks for completion notification                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 10. References

| Reference | Description |
|-----------|-------------|
| [HuggingFace Spaces Docker](https://huggingface.co/docs/hub/spaces-run-with-docker) | Deployment patterns for ML apps |
| [FastAPI Best Practices](https://fastapi.tiangolo.com/) | Official FastAPI documentation |
| [Celery with FastAPI](https://docs.celeryq.dev/) | Background task processing |
| [Docker Best Practices for ML](https://www.docker.com/blog/build-machine-learning-apps-with-hugging-faces-docker-spaces/) | ML containerization patterns |

---

## 11. Appendix

### A. Technology Comparison Matrix

| Aspect | Current Choice | Alt 1 | Alt 2 | Notes |
|--------|----------------|-------|-------|-------|
| Frontend Framework | React | Vue 3 | Svelte | React has largest ecosystem |
| Build Tool | Vite | Next.js | Parcel | Vite fastest for dev |
| Backend | FastAPI | Express | Django | FastAPI for AI integration |
| Database | PostgreSQL | MySQL | MongoDB | SQLModel + PostgreSQL |
| Queue | Celery/Redis | RQ | Kafka | Celery standard for Python |
| Storage | MinIO/S3 | Local | GCS | MinIO for dev, S3 for prod |
