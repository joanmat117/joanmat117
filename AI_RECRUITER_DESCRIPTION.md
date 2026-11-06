# AI Recruiter — Project Summary for CV

> **Tech Stack:** NestJS 11 · TypeScript · LangGraph · LangChain · BullMQ · Drizzle ORM · pgvector · S3 (RustFS/Filebase/AWS/MinIO) · Redis · PostgreSQL
> **Architecture:** Async Queue-Based Pipeline · Hexagonal/Clean Architecture · Dependency Inversion · Strategy Pattern · Value Objects

---

## 🎯 Project Overview

**AI Recruiter** is an **async, queue-based candidate evaluation system** that processes CVs (PDF), matches them against job descriptions using LLMs, and generates hiring recommendations (ADVANCE / REVIEW / REJECT) with personalized feedback, interview questions, and salary suggestions.

### Core Flow
```
Client (Browser) → S3 Presigned POST → PDF Queue (BullMQ) → PDF Worker → AI Recruiter Queue → LangGraph (4 nodes) → PostgreSQL + pgvector
```

**Key differentiator:** Client uploads PDF **directly to S3** (no backend bandwidth), backend only receives `s3Key` + job description. Entire pipeline is async with **native BullMQ job chaining** for atomic queue→queue transitions.

---

## 🏗️ Architecture & Design Patterns

### Layered Architecture (Hexagonal / Clean)

| Layer | Responsibility | Key Components |
|-------|----------------|----------------|
| **Transport** | HTTP Controllers, BullMQ Workers (thin wiring) | `RecruiterController`, `UploadController`, `PdfConsumer`, `AiRecruiterConsumer` |
| **Application** | Use Cases, Policies | `EvaluateCandidateUseCase`, `ScoringPolicy` |
| **Domain/Graph** | Ports, Pure Function Nodes, Tools | `IEvaluationGraph`, 4 LangGraph nodes, `JobMatcherTool` |
| **Infrastructure** | LLM Providers, S3, Drizzle Repositories | `LlmProviderFactory`, `S3FileStorageAdapter`, `DrizzleEvaluationRepository` |

### Mandatory Architectural Decisions (A1–A9)

| # | Decision | Pattern | Implementation |
|---|----------|---------|----------------|
| **A1** | Graph behind Port `IEvaluationGraph` | **Dependency Inversion** | Worker injects port, never imports graph directly |
| **A2** | `EvaluateCandidateUseCase` | **Single Responsibility** | Worker → Use Case → Graph Port |
| **A3** | Structured LLM Calls as Strategy | **Strategy** | `AnalyzeJobDescriptionCall`, `GenerateFeedbackCall` injectable & testable |
| **A4** | Transactional Outbox / Job Chaining | **Transactional Outbox** | `PdfConsumer.process()` returns `nextJob` → BullMQ native chaining (atomic) |
| **A5** | Value Objects for Inputs | **Value Object / Encapsulation** | `S3Key`, `JobDescription`, `JobTitle` validate invariants in constructor |
| **A6** | Injectable `ScoringPolicy` | **Strategy/Policy** | Weights (50/30/20) & thresholds via ConfigService, not hardcoded |
| **A7** | Explicit Node Error Handling | **Robustness** | `handleNodeError` + `createErrorState` per node; conditional Error edge in graph |
| **A8** | Constants for Job/Queue Names | **Hygiene** | `AI_RECRUITER_JOBS.EVALUATE_CANDIDATE` — no magic strings |
| **A9** | Strict Immutability in State | **Immutability** | Nodes return **new complete objects** (no shallow spread on nested) |

---

## 🧠 LangGraph Implementation (4-Node Workflow)

**State:** `cvText`, `jobDescription`, `jobTitle` → flows through nodes with reducers for `meta.errors`/`warnings` (append)

| Node | Type | Responsibility | Dependencies |
|------|------|----------------|--------------|
| **analyzer** | LLM + Zod | Extract structured job requirements (skills, experience, education) | `AnalyzeJobDescriptionCall` (Strategy) |
| **matcher** | Deterministic Tool | Match candidate skills vs. required; calc experience/education match | `JobMatcherTool` |
| **scorer** | Policy/Math | Weighted scoring (50/30/20) + decision via `ScoringPolicy` thresholds | `ScoringPolicy`, `JobMatcherTool` |
| **feedback** | LLM + Zod | Generate interview questions, feedback, next steps, salary range | `GenerateFeedbackCall` (Strategy) |

**Flow:** `START → analyzer → matcher → scorer → feedback → END`

**Error handling:** Each node wraps logic in `handleNodeError(step, fn)` → on failure returns `createErrorState(step, error)` → graph transitions to `Error` step → ends cleanly.

**Checkpointer:** `MemorySaver` (dev) → `PostgresSaver` (prod) for persistence across restarts.

---

## ⚙️ Key Technical Implementations

### 1. Multi-Provider LLM Abstraction
```typescript
// LlmProviderFactory creates: OpenAI, OpenRouter, Gemini
// Injected as 'LLM_PROVIDER' token → used by Strategies
LLM_PROVIDER=openai|openrouter|gemini
```

### 2. S3 Direct Upload (Presigned POST)
- Client calls `POST /recruiter/upload/presigned` → gets `{ url, fields, key }`
- Uploads PDF **directly to S3** (RustFS local / Filebase / AWS / MinIO)
- Backend never handles multipart/file buffers → scales horizontally

### 3. BullMQ Job Chaining (Transactional Outbox)
```typescript
// PdfConsumer.process() returns:
{
  text: extractedText,
  nextJob: { name: 'evaluate-candidate', data: { cvText, jobDescription, ... } }
}
// BullMQ enqueues next job ATOMICALLY in same Redis transaction
// No onCompleted side effects → no lost jobs on crash
```

### 4. Drizzle ORM + pgvector (Ready for Semantic Search)
```sql
CREATE TABLE evaluations (
  job_id VARCHAR(255) UNIQUE NOT NULL,
  cv_embedding VECTOR(1536),  -- pgvector for future semantic search
  scores JSONB, recommendation JSONB, match_details JSONB, output JSONB
);
CREATE INDEX idx_evaluations_cv_embedding ON evaluations USING ivfflat (cv_embedding vector_cosine_ops);
```

### 5. Value Objects (Boundary Validation)
```typescript
S3Key.from(raw)       // validates: cvs/<uuid>.pdf
JobDescription.from() // validates: min 50 chars
JobTitle.from()       // defaults: 'Untitled Position'
EvaluationRequest.from(dto) // composes all three
```

### 6. Configurable Scoring Policy
```env
SCORING_WEIGHT_SKILLS=0.5
SCORING_WEIGHT_EXPERIENCE=0.3
SCORING_WEIGHT_EDUCATION=0.2
SCORING_THRESHOLD_ADVANCE=85
SCORING_THRESHOLD_REVIEW=50
```

---

## 🐳 Infrastructure as Code

**docker-compose.yml** spins up:
- **PostgreSQL + pgvector** (`pgvector/pgvector:pg16`)
- **Redis 7** (BullMQ backend)
- **RustFS** (S3-compatible, API + Console on :9000/:9001)
- **App** (NestJS, hot-reload in dev)

**Local dev option:** Infra in Docker, app on host with `.env.local` pointing to localhost ports.

---

## 🧪 Testing & Quality

- **Unit tests** for tools (`JobMatcherTool`, `PdfExtractorTool`)
- **Integration tests** for graph nodes (mock LLM)
- **E2E test script** (`test-flow.sh`): presigned POST → S3 upload → enqueue → poll status → assert result
- **Strict TypeScript** + ESLint + Prettier
- **Conventional Commits** for git history

---

## 📦 Project Structure (Domain-Driven)

```
src/
├── modules/
│   ├── llm/                          # Multi-provider LLM abstraction
│   └── recruiter/
│       ├── controllers/              # Transport layer (thin)
│       │   ├── recruiter.controller.ts   # POST /evaluate, GET /status
│       │   └── upload.controller.ts      # POST /upload/presigned
│       ├── use-cases/                # Application logic
│       │   └── evaluate-candidate.use-case.ts
│       ├── policies/                 # Business rules (Strategy)
│       │   └── scoring.policy.ts
│       ├── interfaces/               # Ports (Dependency Inversion)
│       │   ├── evaluation-graph.interface.ts
│       │   ├── evaluation.repository.interface.ts
│       │   ├── file-storage.interface.ts
│       │   └── extracted-pdf.repository.interface.ts
│       ├── repositories/             # Adapters (Infrastructure)
│       │   ├── drizzle-evaluation.repository.ts
│       │   └── in-memory-extracted-pdf.repository.ts
│       ├── queues/                   # Transport (BullMQ Workers)
│       │   ├── pdf/pdf.consumer.ts          # S3 download → extract → job chaining
│       │   └── ai-recruiter/ai-recruiter.consumer.ts  # UseCase → Graph → persist
│       ├── nodes/                    # Pure function graph nodes
│       │   ├── analyzer.node.ts
│       │   ├── matcher.node.ts
│       │   ├── scorer.node.ts
│       │   └── feedback.node.ts
│       ├── graph/                    # Graph factory + error handling
│       │   ├── recruiter.graph.ts
│       │   └── node-error-handler.ts
│       ├── state/                    # LangGraph State (Annotation.Root)
│       │   └── recruiter.state.ts
│       ├── tools/                    # Deterministic domain tools
│       │   ├── pdf-extractor.tool.ts
│       │   └── job-matcher.tool.ts
│       ├── llm/                      # LLM Strategies (A3)
│       │   └── structured-llm-calls.ts
│       ├── value-objects/            # Input validation (A5)
│       │   └── evaluation-request.vo.ts
│       ├── constants/                # No magic strings (A8)
│       │   └── job-names.ts
│       └── validators/               # Class-validator integration
└── db/
    └── schema/
        ├── evaluations.ts
        └── extracted-pdfs.ts
```

---

## 🎯 Skills & Competencies Demonstrated

### Architecture & Design
- ✅ **Hexagonal / Clean Architecture** with clear layer boundaries
- ✅ **Dependency Inversion** via ports (`IEvaluationGraph`, `IFileStorage`, `IEvaluationRepository`)
- ✅ **Strategy Pattern** for LLM calls (`AnalyzeJobDescriptionCall`, `GenerateFeedbackCall`) and scoring (`ScoringPolicy`)
- ✅ **Value Objects** for domain-driven input validation
- ✅ **Immutability** discipline in state updates (LangGraph nodes return new complete objects)
- ✅ **Transactional Outbox / Job Chaining** for reliable async queue→queue transitions

### Backend Engineering
- ✅ **NestJS 11** modular architecture (feature modules, dynamic providers, factory injection)
- ✅ **BullMQ** for distributed job processing with retry/backoff/DLQ
- ✅ **LangGraph** (TypeScript) for structured LLM workflows with checkpointers
- ✅ **Drizzle ORM** + **pgvector** for type-safe SQL + vector embeddings
- ✅ **S3-compatible storage** abstraction (RustFS local, Filebase, AWS, MinIO, Cloudflare R2)
- ✅ **Multi-provider LLM** abstraction (OpenAI, OpenRouter, Gemini) with structured output (Zod)

### DevOps & Infrastructure
- ✅ **Docker Compose** for full local stack (PostgreSQL, Redis, RustFS, App)
- ✅ **Environment-based configuration** (ConfigService, `.env`, `.env.local`)
- ✅ **Health checks** for all services
- ✅ **Volume permission handling** for RustFS (UID 10001)

### Code Quality & Practices
- ✅ **Conventional Commits** + semantic versioning readiness
- ✅ **TypeScript strict mode** + ESLint + Prettier
- ✅ **Domain-driven folder structure** (not technical-layered)
- ✅ **Error handling** at every layer (node-level, graph-level, worker-level)
- ✅ **Observability** via structured logging (NestJS Logger)

---

## 📊 Metrics & Scale Considerations

| Aspect | Implementation |
|--------|----------------|
| **Concurrency** | BullMQ workers scale horizontally; Redis-backed queue |
| **Latency** | Client uploads to S3 directly (no backend bandwidth) |
| **Reliability** | Job chaining = atomic queue transitions; retries with exponential backoff |
| **Observability** | Structured logs per worker; job status polling endpoint |
| **Extensibility** | New LLM providers = 1 factory case; new graph nodes = pure functions; new storage = implement `IFileStorage` |
| **Future-ready** | pgvector index for semantic CV search; checkpointer swap for production persistence |

---

## 🏆 Highlights for CV

> **Built a production-grade async candidate evaluation pipeline** using **NestJS + LangGraph + BullMQ** with **Clean Architecture** principles:
>
> - **Designed** hexagonal architecture with Dependency Inversion (ports for Graph, Storage, Repository)
> - **Implemented** 4-node LangGraph workflow (Analyzer → Matcher → Scorer → Feedback) with Zod-structured LLM outputs
> - **Engineered** transactional BullMQ job chaining for atomic PDF→AI queue transitions (no lost jobs)
> - **Abstracted** multi-provider LLM support (OpenAI, OpenRouter, Gemini) via Strategy pattern
> - **Delivered** S3 direct upload via presigned POST (zero backend file handling)
> - **Integrated** Drizzle ORM + pgvector for type-safe persistence with vector search readiness
> - **Containerized** full stack (PostgreSQL, Redis, RustFS S3, App) with Docker Compose

---

## 🔗 Quick Links

- **Repo:** (add your GitHub URL)
- **Architecture Diagram:** See `README.md` (Mermaid flowchart)
- **Implementation Plan:** `IMPLEMENTATION_PLAN.md` (phases, decisions, dependencies)
- **Run locally:** `docker-compose up -d --build` → `./test-flow.sh cv.pdf`

---

*Generated from codebase analysis — commits `7595824` through `b1c3179`*