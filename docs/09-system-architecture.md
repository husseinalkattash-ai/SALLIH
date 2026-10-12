# SALLIH — System Architecture

**Document Version:** 1.0  
**Project Stage:** MVP v0.1  
**Status:** Implementation Baseline

---

# 1. Purpose

This document defines the system architecture for SALLIH MVP v0.1.

It converts the product, AI, data, and user-flow requirements into an implementable technical architecture.

It defines:

- frontend architecture,
- backend architecture,
- module boundaries,
- LangGraph runtime,
- LangChain usage,
- database architecture,
- vector retrieval,
- document ingestion,
- background processing,
- object storage,
- ASR integration,
- provider abstractions,
- persistence,
- API boundaries,
- authentication,
- diagnostic state ownership,
- failure recovery,
- observability,
- deployment topology,
- testing boundaries,
- and future scalability.

This document must be used together with:

- `01-product-vision.md`
- `02-problem-and-users.md`
- `03-mvp-scope.md`
- `04-user-flows.md`
- `05-functional-requirements.md`
- `06-ai-system-design.md`
- `07-data-and-datasets.md`
- `08-asr-and-finetuning.md`

The primary architectural principle is:

> **Build the smallest architecture that reliably supports the full SALLIH diagnostic workflow while preserving clear boundaries for future expansion.**

---

# 2. Architecture Goals

The MVP architecture must support:

```text
Machine Passport
+
Machine Documents
+
Machine-Specific RAG
+
Persistent Diagnostic Sessions
+
LangGraph Guided Troubleshooting
+
Human-in-the-Loop
+
Maintenance Memory
+
Arabic Voice Input
+
Safety Validation
+
Evaluation and Traceability
```

The architecture must also support future expansion without forcing premature implementation of:

- microservices,
- Kafka,
- Kubernetes,
- event-driven distributed systems,
- multi-agent infrastructure,
- large-scale industrial telemetry.

---

# 3. Architectural Style

SALLIH MVP v0.1 will use:

> **Modular Monolith + Dedicated Background Worker**

The core application runs as one backend application with clearly separated internal modules.

Conceptually:

```text
Frontend
   ↓
FastAPI Application
   │
   ├── Authentication
   ├── Machines
   ├── Documents
   ├── Diagnostics
   ├── Maintenance Memory
   ├── AI Orchestration
   ├── Safety
   ├── ASR Integration
   └── Evaluation Support
   │
   ├── PostgreSQL + pgvector
   ├── Redis
   ├── Object Storage
   └── Background Worker
```

The modular monolith should preserve logical boundaries so modules can later become separate services if real operational requirements justify it.

---

# 4. Why Not Microservices for MVP

Microservices would introduce unnecessary complexity such as:

```text
Service discovery
Distributed tracing
Network failure handling
Multiple deployment pipelines
Distributed transactions
Message brokers
API version coordination
```

The MVP does not yet require this complexity.

The preferred architecture is:

```text
Clear module boundaries
+
One primary backend deployment
+
One background worker
+
One database
```

This provides sufficient structure without premature infrastructure overhead.

---

# 5. High-Level System Architecture

```text
┌───────────────────────────────────────────────────────────────┐
│                         Technician                            │
│                                                               │
│                  Desktop / Tablet Browser                     │
└──────────────────────────────┬────────────────────────────────┘
                               │
                               │ HTTPS
                               ▼
┌───────────────────────────────────────────────────────────────┐
│                      Next.js Web App                          │
│                                                               │
│  Machines                                                     │
│  Machine Passport                                             │
│  Troubleshooting                                              │
│  Voice Recording                                              │
│  Evidence Panel                                               │
│  Maintenance History                                          │
└──────────────────────────────┬────────────────────────────────┘
                               │
                               │ REST API
                               │ Optional SSE for progress
                               ▼
┌───────────────────────────────────────────────────────────────┐
│                         FastAPI                               │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ Application Modules                                     │  │
│  │                                                         │  │
│  │ Auth                                                    │  │
│  │ Machines                                                │  │
│  │ Documents                                               │  │
│  │ Diagnostics                                             │  │
│  │ Maintenance Memory                                      │  │
│  │ ASR                                                     │  │
│  │ Evaluation                                              │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ AI Layer                                                │  │
│  │                                                         │  │
│  │ LangGraph Diagnostic Orchestrator                       │  │
│  │ LangChain Tools                                         │  │
│  │ LLM Provider Abstraction                                │  │
│  │ Retrieval                                               │  │
│  │ Safety Layer                                            │  │
│  └─────────────────────────────────────────────────────────┘  │
└─────────────┬──────────────────┬─────────────────┬─────────────┘
              │                  │                 │
              ▼                  ▼                 ▼
      ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
      │ PostgreSQL   │   │ Redis        │   │ Object       │
      │ + pgvector   │   │              │   │ Storage      │
      └──────┬───────┘   └──────┬───────┘   └──────────────┘
             │                  │
             │                  ▼
             │          ┌────────────────┐
             │          │ ARQ Worker     │
             │          │                │
             │          │ PDF processing │
             │          │ OCR            │
             │          │ Embeddings     │
             │          │ Indexing       │
             │          └────────────────┘
             │
             ▼
     ┌───────────────────┐
     │ LangGraph         │
     │ Checkpoints       │
     │                   │
     │ PostgreSQL-backed │
     └───────────────────┘


Additional internal boundary:

┌─────────────────────────┐
│ ASR Provider            │
│                         │
│ Whisper Large v3 Turbo  │
│ faster-whisper          │
│ Local / GPU runtime     │
└─────────────────────────┘

External model provider:

┌─────────────────────────┐
│ LLM Provider            │
│                         │
│ Gemini / OpenAI /       │
│ Anthropic / other       │
└─────────────────────────┘
```

---

# 6. Technology Stack

## Frontend

```text
Next.js
React
TypeScript
```

Recommended approach:

```text
Next.js App Router
```

The frontend remains a web application.

A native mobile application is outside MVP scope.

---

## Backend

```text
Python
FastAPI
Pydantic
SQLAlchemy
Alembic
```

FastAPI is responsible for:

- HTTP APIs,
- authentication,
- validation,
- application services,
- AI workflow entry points,
- diagnostic-session persistence,
- ASR integration.

---

## AI Orchestration

```text
LangGraph
LangChain
```

LangGraph:

```text
Diagnostic orchestration
State
Routing
Interrupt / Resume
Checkpointing
Human-in-the-loop
```

LangChain:

```text
Model integrations
Structured outputs
Tools
Embeddings
Retrievers
Document integrations
```

---

## Database

```text
PostgreSQL
```

with:

```text
pgvector
```

for vector retrieval.

---

## Background Jobs

```text
Redis
+
ARQ Worker
```

used for durable asynchronous work such as:

```text
Document ingestion
OCR
Chunking
Embedding generation
Re-indexing
```

Redis is not the primary application database.

---

## File Storage

MVP development:

```text
Local filesystem
or
S3-compatible object storage
```

Architecture should use a storage abstraction so deployment can move to:

```text
S3
MinIO
compatible object storage
```

without changing domain logic.

---

## ASR

Baseline:

```text
Whisper Large v3 Turbo
```

Preferred inference option:

```text
faster-whisper
+
CTranslate2
```

where practical.

---

# 7. Repository Structure

Recommended monorepo structure:

```text
sallih/
│
├── apps/
│   │
│   ├── web/
│   │   └── Next.js application
│   │
│   └── api/
│       └── FastAPI application
│
├── workers/
│   └── ingestion/
│       └── document background jobs
│
├── docs/
│   ├── 01-product-vision.md
│   ├── 02-problem-and-users.md
│   ├── 03-mvp-scope.md
│   ├── 04-user-flows.md
│   ├── 05-functional-requirements.md
│   ├── 06-ai-system-design.md
│   ├── 07-data-and-datasets.md
│   ├── 08-asr-and-finetuning.md
│   ├── 09-system-architecture.md
│   └── reviews/
│
├── data/
│   ├── manifests/
│   ├── demo/
│   ├── evaluation/
│   └── synthetic/
│
├── scripts/
│
├── docker/
│
├── docker-compose.yml
│
└── README.md
```

The exact tooling structure may evolve, but frontend, backend, workers, documentation, and research data should remain clearly separated.

---

# 8. FastAPI Internal Structure

Recommended backend organization:

```text
apps/api/app/
│
├── main.py
│
├── config.py
│
├── database.py
│
├── security/
│
├── api/
│   ├── auth.py
│   ├── machines.py
│   ├── documents.py
│   ├── diagnostics.py
│   ├── maintenance.py
│   ├── asr.py
│   └── evaluation.py
│
├── domain/
│   ├── machines/
│   ├── documents/
│   ├── diagnostics/
│   ├── maintenance/
│   └── safety/
│
├── ai/
│   ├── graph/
│   ├── providers/
│   ├── tools/
│   ├── retrieval/
│   ├── prompts/
│   ├── schemas/
│   └── safety/
│
├── asr/
│   ├── providers/
│   ├── normalization/
│   └── schemas/
│
├── repositories/
│
├── services/
│
├── models/
│
└── tests/
```

The exact folders may change during implementation.

The important boundary is:

```text
API Layer
≠
Domain Logic
≠
AI Orchestration
≠
Persistence
```

---

# 9. Dependency Direction

Preferred dependency direction:

```text
API
 ↓
Application Services
 ↓
Domain / AI Contracts
 ↓
Repositories / Providers
```

Domain logic should not depend directly on FastAPI request objects.

AI logic should not perform uncontrolled SQL queries directly from prompts or graph nodes.

---

# 10. Frontend Responsibilities

The frontend is responsible for:

```text
Authentication UI
Machine browsing
Machine Passport
Document upload UI
Voice recording
Transcript review
Diagnostic conversation
Measurement forms
Evidence display
Hypothesis display
Session resume UI
Maintenance history
Maintenance case details
```

The frontend must not contain authoritative diagnostic logic.

---

# 11. Frontend State

Frontend state may include:

```text
Current user
Current machine
Current diagnostic session
Pending question
UI loading states
Recording state
Transcript draft
```

The backend remains authoritative for:

```text
Machine records
Session status
Diagnostic evidence
Confirmed measurements
Resolved cases
```

---

# 12. API Design Principle

SALLIH should expose resource-oriented APIs.

The frontend should not directly understand LangGraph internals.

For example, frontend should call:

```text
Submit technician input
```

not:

```text
Execute node update_hypotheses
```

LangGraph is an implementation detail behind the application API.

---

# 13. Conceptual API Surface

Exact endpoint design will be finalized during implementation.

A conceptual API may include:

```text
POST   /auth/login
POST   /auth/logout
GET    /auth/me
```

Machines:

```text
GET    /machines
GET    /machines/{machine_id}
```

Documents:

```text
GET    /machines/{machine_id}/documents
POST   /machines/{machine_id}/documents
GET    /documents/{document_id}
DELETE /documents/{document_id}
```

Diagnostics:

```text
POST   /machines/{machine_id}/diagnostic-sessions
GET    /diagnostic-sessions/{session_id}
POST   /diagnostic-sessions/{session_id}/fault
POST   /diagnostic-sessions/{session_id}/input
POST   /diagnostic-sessions/{session_id}/cancel
POST   /diagnostic-sessions/{session_id}/resolve
GET    /diagnostic-sessions/{session_id}/timeline
```

Maintenance:

```text
GET    /machines/{machine_id}/maintenance-cases
GET    /maintenance-cases/{case_id}
```

ASR:

```text
POST   /asr/transcriptions
```

Evaluation APIs may be internal or development-only.

---

# 14. API Contracts

Every API that changes diagnostic state must use validated request and response models.

Example:

```python
class SubmitMeasurementRequest(BaseModel):
    input_id: UUID
    measurement_type: str
    values: dict[str, float]
    unit: str
```

Malformed client data must never reach LangGraph state directly.

---

# 15. OpenAPI

FastAPI's OpenAPI specification should be treated as the canonical frontend/backend contract during implementation.

The frontend should not rely on undocumented backend fields.

Where practical, API types may be generated or validated against the OpenAPI schema.

---

# 16. Authentication Architecture

MVP authentication remains intentionally simple.

Architecture:

```text
User credentials
      ↓
FastAPI authentication
      ↓
Authenticated session/token
      ↓
Protected application APIs
```

Potential implementation:

```text
JWT access token
+
secure password hashing
```

or an equivalent simple secure authentication mechanism.

Complex enterprise identity integration is out of scope.

---

# 17. User Entity

Conceptual user:

```text
users

id
email
password_hash
display_name
role
created_at
updated_at
```

MVP role:

```text
TECHNICIAN
```

Optional project/admin role:

```text
ADMIN
```

may exist for document/machine setup.

---

# 18. Machine Data Architecture

Core entities:

```text
machines
machine_components
machine_documents
document_chunks
```

Relationships:

```text
Machine
 │
 ├── Components
 │
 ├── Documents
 │     └── Chunks
 │
 ├── Diagnostic Sessions
 │
 └── Maintenance Cases
```

---

# 19. Suggested Machine Tables

Conceptual schema:

```text
machines
-------
id
name
manufacturer
model
serial_number
year
location
description
status
created_at
updated_at
```

Optional:

```text
machine_components
------------------
id
machine_id
name
component_type
manufacturer
model
part_number
description
created_at
updated_at
```

---

# 20. Document Tables

Conceptually:

```text
machine_documents
-----------------
id
machine_id
file_name
storage_key
document_type
language
manufacturer
revision
processing_status
processing_error
created_at
processed_at
```

---

# 21. Document Chunk Table

Conceptually:

```text
document_chunks
---------------
id
machine_id
document_id
chunk_index
content
page_number
section
heading
component
embedding
embedding_model
embedding_version
created_at
```

`embedding` uses:

```text
pgvector
```

---

# 22. Why PostgreSQL + pgvector

Using PostgreSQL and pgvector provides:

```text
Structured application data
+
Vector search
+
Transactional consistency
+
Simple MVP operations
```

without requiring a second dedicated vector database.

This keeps infrastructure small.

A specialized vector database may be considered later only if real scale or retrieval requirements justify it.

---

# 23. Machine Isolation

Machine isolation is a critical architectural invariant.

All machine-specific retrieval must enforce:

```text
machine_id = active_machine_id
```

at the repository/query layer.

Incorrect pattern:

```text
LLM instruction:
"Only search this machine."
```

Correct pattern:

```text
Backend retrieval query:
WHERE machine_id = :machine_id
```

The model must not be responsible for enforcing data isolation.

---

# 24. Diagnostic Data Architecture

Core entities:

```text
diagnostic_sessions
technician_observations
measurements
diagnostic_hypotheses
diagnostic_steps
evidence_items
diagnostic_events
```

---

# 25. Diagnostic Session Table

Conceptually:

```text
diagnostic_sessions
-------------------
id
machine_id
technician_id
langgraph_thread_id

status

original_fault_description

structured_fault_json

probable_root_cause_json

termination_reason

created_at
updated_at
resolved_at
```

Structured JSON may initially be appropriate for some evolving AI state.

Fields critical for querying or integrity should use explicit relational columns.

---

# 26. Technician Observations Table

```text
technician_observations
-----------------------
id
session_id
technician_id
original_text
structured_interpretation_json
created_at
```

Original technician input must remain preserved.

---

# 27. Measurements Table

Conceptually:

```text
measurements
------------
id
session_id
technician_id
measurement_type
values_json
unit
operating_condition
created_at
```

---

# 28. Diagnostic Hypotheses Table

Conceptually:

```text
diagnostic_hypotheses
---------------------
id
session_id
hypothesis_key
description
component
status
confidence
missing_evidence_json
created_at
updated_at
```

Hypothesis-to-evidence relationships may be represented through association tables.

---

# 29. Evidence Architecture

Conceptual table:

```text
evidence_items
--------------
id
session_id
machine_id
source_type

title
content

document_id
document_chunk_id
page_number
section

maintenance_case_id
observation_id
measurement_id

retrieval_score

created_at
```

Evidence should be persisted sufficiently to reproduce the diagnostic trace.

---

# 30. Diagnostic Steps Table

Conceptually:

```text
diagnostic_steps
----------------
id
session_id
step_number

action_type
instruction
reason
safety_level

requested_at
completed_at

result_reference
```

---

# 31. Diagnostic Events Table

An append-oriented timeline should preserve meaningful events.

Conceptually:

```text
diagnostic_events
-----------------
id
session_id

event_type
payload_json

created_at
```

Examples:

```text
SESSION_CREATED
FAULT_SUBMITTED
FAULT_STRUCTURED
EVIDENCE_RETRIEVED
HYPOTHESES_UPDATED
ACTION_REQUESTED
MEASUREMENT_SUBMITTED
PROBABLE_CAUSE_CREATED
RESOLUTION_CONFIRMED
```

---

# 32. Maintenance Case Architecture

Conceptual table:

```text
maintenance_cases
-----------------
id
machine_id
diagnostic_session_id

fault_summary

structured_symptoms_json
alarm_codes_json

probable_root_cause
confirmed_root_cause
corrective_action

parts_used_json
downtime_minutes

searchable_summary

resolved_by
resolved_at

created_at
```

Only resolved diagnostic sessions create confirmed Maintenance Memory.

---

# 33. Maintenance Case Vector Search

For semantic historical-case retrieval, the system may store an embedding of a maintenance-case search representation.

Example search text:

```text
Machine:
Packaging Line #01

Symptoms:
VFD trip after warm-up

Alarm:
F30005

Confirmed Root Cause:
Loose L3 terminal
```

This may be stored either:

```text
as an embedding column on maintenance_cases
```

or:

```text
in a separate maintenance_case_embeddings table
```

The implementation decision may be finalized during coding.

---

# 34. LangGraph Runtime Architecture

LangGraph runs inside the backend AI/application layer.

Conceptually:

```text
FastAPI Diagnostic Service
        ↓
LangGraph Graph
        ↓
Nodes
        ↓
Tools / Providers
```

LangGraph should not initially run as a separate network microservice.

---

# 35. LangGraph Thread Mapping

Every Diagnostic Session must have exactly one primary graph thread.

Conceptually:

```text
diagnostic_session.id
        ↔
langgraph_thread_id
```

This mapping must be persistent.

---

# 36. LangGraph Checkpointing

LangGraph checkpoints should use durable persistence.

Preferred architecture:

```text
PostgreSQL-backed LangGraph checkpointer
```

rather than in-memory checkpoints.

This supports:

```text
Browser refresh
Backend restart
Human interruption
Long-running diagnostic sessions
```

---

# 37. Database vs LangGraph State

The system must not treat LangGraph checkpoint data as the only business database.

Correct architecture:

```text
Application Database
=
Authoritative business state

LangGraph Checkpoint
=
Workflow execution state
```

Example:

```text
confirmed_root_cause
```

must exist in application/domain persistence.

It must not be recoverable only by inspecting a LangGraph checkpoint.

---

# 38. LangGraph Nodes

The initial graph should implement behavior equivalent to:

```text
understand_fault
load_machine_context
retrieve_evidence
generate_hypotheses
decide_next_action
safety_check
prepare_technician_request
receive_technician_input
validate_technician_input
update_hypotheses
retrieve_additional_evidence
assess_root_cause
insufficient_evidence
technician_confirms_resolution
save_maintenance_case
```

Some nodes may be implemented as deterministic functions.

Not every node requires an LLM call.

---

# 39. Graph Node Classification

Nodes should be classified as:

```text
LLM
DETERMINISTIC
TOOL
HUMAN_INTERRUPT
```

Example:

| Node | Type |
|---|---|
| understand_fault | LLM |
| load_machine_context | DETERMINISTIC |
| retrieve_evidence | TOOL |
| generate_hypotheses | LLM |
| decide_next_action | LLM |
| safety_check | DETERMINISTIC + controlled assessment |
| prepare_technician_request | DETERMINISTIC / LLM-assisted |
| receive_technician_input | HUMAN_INTERRUPT |
| validate_technician_input | DETERMINISTIC |
| update_hypotheses | LLM |
| assess_root_cause | LLM |
| save_maintenance_case | DETERMINISTIC |

This makes the workflow auditable.

---

# 40. LangGraph State Schema

The runtime state must use explicit validated types.

Conceptual:

```python
class DiagnosticGraphState(TypedDict):
    session_id: str
    machine_id: str
    technician_id: str

    status: str

    original_fault_description: str | None
    structured_fault: dict | None

    machine_context: dict | None

    evidence: list[dict]
    hypotheses: list[dict]

    observations: list[dict]
    measurements: list[dict]

    diagnostic_steps: list[dict]

    pending_action: dict | None

    probable_root_cause: dict | None

    iteration_count: int

    termination_reason: str | None
    last_error: dict | None
```

Exact Python types should be stricter than generic `dict` in implementation where practical.

---

# 41. Graph State Synchronization

Application data and graph data must not drift silently.

Example:

```text
Technician submits measurement
        ↓
FastAPI validates
        ↓
Measurement stored
        ↓
Graph resumes with measurement reference
```

The graph should not create an unpersisted measurement independently.

---

# 42. Human-in-the-Loop Architecture

The workflow must support:

```text
Graph runs
↓
Diagnostic action prepared
↓
Checkpoint persisted
↓
Session status = WAITING_FOR_TECHNICIAN
↓
Graph interrupts
```

Later:

```text
Technician submits input
↓
FastAPI validates input
↓
Input persisted
↓
Graph resumes
↓
New diagnostic state produced
```

---

# 43. Human Input Idempotency

Technician input submissions should have stable identifiers.

Example:

```text
input_id
```

If the frontend retries the same network request, the backend should avoid creating duplicate:

```text
measurements
observations
graph resumptions
```

where practical.

---

# 44. Diagnostic Request Flow

Example:

```text
Frontend
   ↓
POST diagnostic input
   ↓
FastAPI validates request
   ↓
Application service validates session state
   ↓
Persist user input
   ↓
Resume LangGraph
   ↓
Graph processes
   ↓
Persist state/events
   ↓
Return updated diagnostic session
```

---

# 45. AI Provider Architecture

LLM providers must be hidden behind a provider abstraction.

Conceptually:

```python
class LLMProvider(Protocol):

    async def structured_output(
        self,
        *,
        task: str,
        schema: type,
        context: dict,
    ):
        ...

    async def generate_text(
        self,
        *,
        task: str,
        context: dict,
    ) -> str:
        ...
```

Provider implementations may include:

```text
GeminiProvider
OpenAIProvider
AnthropicProvider
```

MVP only needs one active provider initially.

---

# 46. Provider Independence

Domain logic must not contain patterns such as:

```text
if provider == Gemini:
    change diagnostic semantics
```

Provider differences should be isolated inside adapters.

Internal schemas remain stable.

---

# 47. Structured Output Architecture

All AI outputs that mutate diagnostic state must pass through:

```text
LLM Provider
↓
Structured Output
↓
Pydantic Validation
↓
Application / Graph State
```

Free-form malformed output must not directly mutate state.

---

# 48. Prompt Architecture

Prompts should be stored in a dedicated module.

Example:

```text
ai/prompts/
│
├── fault_understanding.py
├── hypothesis_generation.py
├── hypothesis_update.py
├── next_action.py
└── root_cause.py
```

Prompt metadata should include:

```text
name
version
```

Evaluation traces should record prompt versions.

---

# 49. LangChain Usage Boundary

LangChain may be used for:

```text
Model interfaces
Structured outputs
Retrievers
Embedding integrations
Tools
Document loaders
```

Application business rules should not become deeply dependent on opaque chains.

The application should remain understandable without inspecting hidden framework behavior.

---

# 50. Tool Architecture

Tools should expose typed application capabilities.

Initial tools:

```text
get_machine_context
search_machine_documents
lookup_alarm_code
search_maintenance_memory
```

Potential future tool:

```text
lookup_parts
```

Tools must not expose unrestricted database access to the LLM.

---

# 51. Retrieval Architecture

Document retrieval flow:

```text
Structured Fault
       ↓
Retrieval Query Builder
       ↓
Exact Identifier Search
       +
Semantic Search
       ↓
Machine Filter
       ↓
Ranking
       ↓
Evidence Objects
```

---

# 52. Hybrid Retrieval

The retrieval layer should support both:

```text
Exact / lexical retrieval
```

and:

```text
semantic vector retrieval
```

because identifiers such as:

```text
F30005
K17
6SL3210-1KE14-3UF2
```

may not perform reliably through semantic similarity alone.

---

# 53. Exact Identifier Path

If a fault contains:

```text
F30005
```

the retrieval layer should first attempt:

```text
exact or normalized alarm-code match
```

before or alongside semantic retrieval.

---

# 54. Semantic Retrieval Path

Semantic retrieval is useful for queries such as:

```text
motor overheats after running under load
```

where exact textual matches may not exist.

---

# 55. Retrieval Results

Retrieval functions should return structured evidence.

Example:

```json
{
  "evidence_id": "EV-001",
  "source_type": "MANUFACTURER_DOCUMENT",
  "document_id": "DOC-005",
  "document_name": "G120 Drive Manual",
  "page_number": 74,
  "section": "Overcurrent Faults",
  "content": "...",
  "score": 0.89
}
```

---

# 56. Retrieval Score Principle

Vector similarity scores are ranking signals.

They are not equivalent to:

```text
truth probability
```

or:

```text
diagnostic confidence
```

The system must not present retrieval scores directly as root-cause confidence.

---

# 57. Document Ingestion Architecture

Document ingestion must occur asynchronously.

Flow:

```text
User uploads PDF
      ↓
FastAPI validates file
      ↓
Document record created
      ↓
File stored
      ↓
Background job queued
      ↓
Document status = PROCESSING
      ↓
Worker processes file
```

Worker pipeline:

```text
Extract Text
↓
OCR if needed
↓
Page Segmentation
↓
Chunking
↓
Metadata Extraction
↓
Embedding Generation
↓
Database Insert
↓
Document status = READY
```

---

# 58. Why Background Processing

Document processing may take seconds or minutes.

The API upload request should not remain blocked while:

```text
OCR
Embeddings
Indexing
```

complete.

Therefore background processing is appropriate.

---

# 59. ARQ Worker

Recommended MVP worker:

```text
ARQ
```

with:

```text
Redis
```

Responsibilities:

```text
Document ingestion
OCR jobs
Embedding generation
Re-indexing
```

AI troubleshooting should not depend on the background worker for every interaction.

Interactive diagnosis remains inside the application request/workflow layer.

---

# 60. Worker Idempotency

Background jobs must be safely retryable where possible.

Example:

```text
process_document(document_id, processing_version)
```

Before processing:

```text
check current document state
```

Repeated delivery of the same job must not create duplicate chunks indefinitely.

---

# 61. Document Processing Version

Document processing artifacts should support a processing version.

Example:

```text
processing_version = "v1"
```

If the chunking or embedding pipeline changes:

```text
processing_version = "v2"
```

may trigger controlled reprocessing.

---

# 62. Object Storage Architecture

Original binary files should not be stored directly as large database blobs unless explicitly justified.

Preferred:

```text
Object Storage
↓
storage_key
↓
Database metadata
```

Examples:

```text
machine-documents/PKG-001/DOC-005/manual.pdf
```

---

# 63. Storage Provider Abstraction

Conceptually:

```python
class FileStorage(Protocol):

    async def put(...):
        ...

    async def get(...):
        ...

    async def delete(...):
        ...
```

Development may use:

```text
LocalFileStorage
```

Deployment may use:

```text
S3FileStorage
MinIOFileStorage
```

---

# 64. ASR Architecture

ASR is a logically separate module.

Conceptually:

```text
Frontend
   ↓
Audio Recording
   ↓
FastAPI
   ↓
ASR Provider
   ↓
Whisper Runtime
   ↓
Transcript
```

---

# 65. ASR Provider Interface

Conceptually:

```python
class ASRProvider(Protocol):

    async def transcribe(
        self,
        audio_file,
    ) -> ASRResult:
        ...
```

---

# 66. ASR Deployment

For MVP development, ASR may run:

```text
inside the backend environment
```

or:

```text
as a separate local process
```

depending on GPU/runtime constraints.

The API contract should remain unchanged.

---

# 67. Why ASR Is Not a Mandatory Microservice

The project does not need distributed ASR infrastructure.

Logical separation is sufficient.

If GPU deployment later requires independent scaling, ASR can become a dedicated service without redesigning the frontend contract.

---

# 68. ASR Model Lifecycle

The Whisper model should ideally load once when the ASR runtime starts.

Avoid:

```text
load multi-gigabyte model
for every transcription request
```

Preferred:

```text
Process starts
↓
Model loaded
↓
Many transcription requests
```

---

# 69. ASR Data Flow

Correct flow:

```text
Audio
↓
ASR Result
↓
Raw Transcript
↓
Frontend Review
↓
Technician Confirmation
↓
Confirmed Transcript
↓
Diagnostic API
```

The diagnostic graph must not start from an unconfirmed ASR transcript.

---

# 70. Technical Normalization Architecture

Technical normalization is a separate layer after transcript confirmation.

Conceptually:

```text
Confirmed Technician Language
        ↓
Technical Normalizer
        ↓
Canonical Concepts
```

Example:

```text
درايف
drive
inverter
```

may map to:

```text
VFD
```

Original wording remains preserved.

---

# 71. Technical Glossary

The project should maintain a machine/domain glossary.

Conceptually:

```text
technical_terms
---------------
canonical_term
category
```

with variants:

```text
technical_term_variants
-----------------------
term_id
variant
language
```

This may initially be implemented in configuration rather than database tables if the dataset is very small.

---

# 72. Safety Architecture

Safety must exist as an independent control layer.

Flow:

```text
Agent proposes physical action
        ↓
Safety Layer
        ↓
ALLOW / WARN / BLOCK
        ↓
Technician-facing output
```

The agent must not bypass this layer.

---

# 73. Safety Rules

The safety layer should contain deterministic prohibited patterns and action classifications.

Examples of blocked categories:

```text
Bypass safety interlock
Disable emergency stop
Disable safety relay
Ignore lockout/tagout
Unauthorized PLC control
Dangerous energized intervention
```

---

# 74. Safety Assessment Output

Conceptually:

```json
{
  "level": "HIGH",
  "allowed": true,
  "requires_qualified_technician": true,
  "warning": "Qualified technician and site electrical safety procedure required.",
  "blocked_reason": null
}
```

Or:

```json
{
  "level": "HIGH",
  "allowed": false,
  "requires_qualified_technician": true,
  "warning": null,
  "blocked_reason": "Bypassing a safety interlock is prohibited."
}
```

---

# 75. Safety Is Separate from Diagnostic Confidence

These are independent:

```text
Diagnostic Confidence
```

and:

```text
Action Safety
```

A HIGH-confidence diagnosis may still involve a HIGH-risk repair action.

---

# 76. Maintenance Memory Architecture

When a session is resolved:

```text
Diagnostic Session
        ↓
Confirmed Root Cause
        ↓
Corrective Action
        ↓
Maintenance Case
        ↓
Search Representation
        ↓
Embedding
        ↓
Future Machine Memory
```

---

# 77. Maintenance Memory Indexing

Indexing of a newly resolved case may happen:

```text
synchronously for small MVP records
```

or:

```text
through a background job
```

if embedding generation is slow.

The maintenance case itself must be saved before indexing.

---

# 78. Resolution Transaction

Resolving a diagnostic session should use a controlled transaction.

Conceptually:

```text
Validate active session
↓
Validate confirmed root cause
↓
Validate corrective action
↓
Mark session RESOLVED
↓
Create Maintenance Case
↓
Commit transaction
↓
Trigger indexing
```

A partial resolution must not produce inconsistent state such as:

```text
session RESOLVED
but no Maintenance Case
```

where avoidable.

---

# 79. Idempotent Resolution

If the client retries the same resolution request, the backend should not create multiple Maintenance Cases.

The operation should use:

```text
diagnostic_session_id
```

as part of the uniqueness guarantee.

---

# 80. Application Database Authority

PostgreSQL is authoritative for:

```text
Users
Machines
Documents
Document processing status
Diagnostic sessions
Observations
Measurements
Evidence references
Diagnostic steps
Maintenance cases
```

---

# 81. Redis Responsibility

Redis is used for:

```text
Background-job queue
Worker coordination
Temporary job infrastructure
```

Redis must not become the only location for:

```text
diagnostic history
confirmed maintenance data
machine records
```

---

# 82. Background Job Failure

If a worker crashes:

```text
Document record remains
↓
processing status remains known
↓
job may retry
↓
user sees PROCESSING / FAILED
```

A worker crash must not silently remove the uploaded document record.

---

# 83. Document Ingestion Failure

Possible states:

```text
UPLOADING
PROCESSING
READY
FAILED
```

Failure reason should be persisted.

Example:

```text
OCR_ERROR
PDF_PARSE_ERROR
EMBEDDING_ERROR
CORRUPTED_FILE
```

---

# 84. API and Worker Transaction Boundary

Correct flow:

```text
API transaction:
create document record

COMMIT

queue job
```

Care must be taken to avoid:

```text
job starts
before document transaction exists
```

Implementation may use a safe post-commit enqueue pattern.

---

# 85. Embedding Provider Architecture

Embeddings should use an abstraction independent from LLM generation.

Conceptually:

```python
class EmbeddingProvider(Protocol):

    async def embed_documents(...):
        ...

    async def embed_query(...):
        ...
```

This allows embedding-model changes without rewriting RAG business logic.

---

# 86. Embedding Versioning

Every indexed chunk should know:

```text
embedding_model
embedding_version
```

Changing embedding models requires:

```text
re-indexing
```

rather than silently mixing incompatible vectors.

---

# 87. Document Retrieval Repositories

The retrieval layer should not expose raw SQL to graph prompts.

Conceptual service:

```python
MachineKnowledgeRetriever.search(
    machine_id=...,
    query=...,
    alarm_codes=...,
    document_types=...
)
```

This service enforces:

```text
machine isolation
ranking rules
result limits
metadata preservation
```

---

# 88. Evidence Construction Boundary

Retrieval output becomes evidence through deterministic application code.

Correct:

```text
DocumentChunk
↓
EvidenceItem
```

with source metadata copied from the database.

Incorrect:

```text
LLM invents page number
↓
EvidenceItem
```

---

# 89. Streaming / Progress Updates

Basic MVP interaction can use ordinary HTTP.

For longer diagnostic steps, the application may use:

```text
Server-Sent Events (SSE)
```

to show progress such as:

```text
Understanding fault...
Searching machine manual...
Reviewing previous cases...
Updating hypotheses...
```

SSE is:

```text
SHOULD HAVE
```

not a blocker for the first working diagnostic flow.

---

# 90. Why Not WebSockets Initially

The MVP diagnostic workflow is mostly:

```text
request
process
interrupt
human response
resume
```

It does not require constant bidirectional realtime communication.

Therefore WebSockets are unnecessary initially.

They may be introduced later if live telemetry or realtime machine data requires them.

---

# 91. Frontend Diagnostic State Contract

The backend should return a frontend-friendly session representation.

Conceptually:

```json
{
  "session_id": "DS-001",
  "status": "WAITING_FOR_TECHNICIAN",

  "machine": {
    "id": "PKG-001",
    "name": "Packaging Line #01"
  },

  "structured_fault": {},

  "hypotheses": [],

  "pending_action": {},

  "evidence": [],

  "observations": [],

  "measurements": [],

  "probable_root_cause": null
}
```

The frontend should not need to reconstruct workflow state from chat messages.

---

# 92. Conversation Messages

Conversation messages may be stored separately from core diagnostic state.

Conceptually:

```text
diagnostic_messages
-------------------
id
session_id
sender
message_type
content
created_at
```

Possible sender:

```text
TECHNICIAN
SALLIH
SYSTEM
```

The conversation is a user-interface representation.

It is not the only source of diagnostic truth.

---

# 93. Conversation vs Structured State

Example:

Chat message:

```text
SALLIH:
Please measure L1, L2 and L3.
```

Structured state:

```json
{
  "action_type": "REQUEST_MEASUREMENT",
  "measurement_type": "phase_current",
  "fields": ["L1", "L2", "L3"],
  "unit": "A"
}
```

Structured state is used for application behavior.

Message text is used for presentation.

---

# 94. Error Architecture

Errors should be classified into application-level categories.

Examples:

```text
VALIDATION_ERROR
AUTHENTICATION_ERROR
NOT_FOUND
CONFLICT
DOCUMENT_PROCESSING_ERROR
MODEL_ERROR
MODEL_SCHEMA_ERROR
RETRIEVAL_ERROR
ASR_ERROR
SAFETY_BLOCK
CHECKPOINT_ERROR
INVALID_DIAGNOSTIC_STATE
```

---

# 95. Provider Error Mapping

External provider errors must not leak directly into API contracts.

Example:

```text
Gemini 503
```

becomes:

```text
MODEL_TEMPORARILY_UNAVAILABLE
```

The application decides whether to retry.

---

# 96. Retry Policy

Retries are appropriate for:

```text
Temporary provider timeout
Transient network error
Temporary embedding failure
Controlled schema retry
```

Retries are not appropriate for:

```text
Invalid user measurement
Unknown machine
Safety-blocked action
Permanent corrupted PDF
```

---

# 97. Retry Boundaries

Retries must be bounded.

No request may recursively retry indefinitely.

Configuration may include:

```text
max_model_retries
max_tool_retries
max_graph_iterations
```

---

# 98. Diagnostic Graph Limits

The graph should support configured limits such as:

```text
MAX_DIAGNOSTIC_ITERATIONS
MAX_TOOL_FAILURES
MAX_SCHEMA_RETRIES
```

Example initial development value:

```text
MAX_DIAGNOSTIC_ITERATIONS = 12
```

This is a guardrail, not a research result.

---

# 99. Configuration Architecture

Environment-specific configuration should use environment variables or typed settings.

Examples:

```text
DATABASE_URL
REDIS_URL

LLM_PROVIDER
LLM_MODEL

EMBEDDING_PROVIDER
EMBEDDING_MODEL

ASR_MODEL

OBJECT_STORAGE_TYPE

MAX_DIAGNOSTIC_ITERATIONS
```

Secrets must not be committed to Git.

---

# 100. Environment Files

Recommended:

```text
.env.example
```

committed with placeholder values.

Actual:

```text
.env
```

must remain ignored.

---

# 101. Secrets

Secrets include:

```text
LLM API keys
Database passwords
Object-storage credentials
JWT secrets
```

They must remain server-side.

No AI provider secret may be shipped to the frontend.

---

# 102. Local Development Architecture

Recommended development environment:

```text
Docker Compose
```

for infrastructure.

Conceptually:

```text
postgres
redis
api
worker
web
```

ASR may run either:

```text
inside api
```

or:

```text
as a locally launched GPU process
```

depending on Docker/GPU development convenience.

---

# 103. Local Development Topology

```text
Browser
   ↓
localhost:3000
Next.js
   ↓
localhost:8000
FastAPI
   │
   ├── PostgreSQL + pgvector
   ├── Redis
   ├── ARQ Worker
   ├── Local Object Storage
   ├── Whisper ASR
   └── Remote LLM API
```

---

# 104. Deployment Philosophy

MVP deployment should remain simple.

Possible initial deployment:

```text
Single server / workstation
+
PostgreSQL
+
Redis
+
API
+
Worker
+
Web application
```

ASR may run on a machine with GPU access.

The architecture should also allow:

```text
on-premise deployment
```

in future industrial environments.

---

# 105. No Mandatory Cloud Dependency for Machine Data

The architecture should not fundamentally require all machine documents to be sent to a public cloud.

A future deployment should be able to keep:

```text
documents
maintenance history
vector database
```

inside an organization's environment.

The LLM provider strategy may later determine whether some context leaves the environment.

---

# 106. Future Local LLM Compatibility

Provider abstraction should allow future use of:

```text
local / self-hosted LLM
```

without redesigning domain state.

This is a future capability.

The MVP should not sacrifice model quality simply to claim full offline operation.

---

# 107. Observability Architecture

The application should support structured logs.

Common fields:

```text
request_id
user_id
machine_id
session_id
document_id
job_id
graph_node
provider
model
latency
status
error_type
```

---

# 108. Request IDs

Each inbound API request should receive or preserve a request identifier.

This assists debugging across:

```text
FastAPI
AI calls
retrieval
worker jobs
```

---

# 109. Diagnostic Session IDs

AI logs should include:

```text
session_id
```

so the complete diagnostic sequence can be reconstructed.

---

# 110. Model Call Metadata

Development/evaluation logging should be able to record:

```text
provider
model
prompt_name
prompt_version
latency
schema_success
retry_count
```

Do not log sensitive prompt content unnecessarily in production-like environments.

---

# 111. Tool Call Metadata

Useful metadata:

```text
tool_name
session_id
machine_id
query
result_count
latency
success
```

---

# 112. Evaluation Architecture

Evaluation tooling should interact with core application services rather than duplicate the AI system.

Conceptually:

```text
Evaluation Scenario
        ↓
Diagnostic Orchestrator
        ↓
Scripted Technician Inputs
        ↓
Diagnostic Trace
        ↓
Metrics
```

---

# 113. Scripted Technician Interface

For benchmark runs, human interrupts should be replaceable with scripted scenario responses.

Example:

```text
Agent requests:
phase_current

Evaluation Harness returns:
L1=8.2
L2=8.4
L3=13.9
```

This allows automated benchmarking.

---

# 114. System Modes

Evaluation should support:

```text
GENERAL_LLM
BASIC_RAG
SALLIH_FULL
```

The same scenario definitions should be reusable across modes.

---

# 115. Evaluation Data Separation

Ground-truth fields such as:

```text
expected_root_cause
expected_next_actions
```

must be available only to the evaluator.

They must not be passed into the diagnostic model during the run.

---

# 116. Testing Architecture

Tests should exist at several levels.

```text
Unit Tests
Integration Tests
AI Contract Tests
Graph Tests
Retrieval Tests
API Tests
End-to-End Tests
Evaluation Tests
```

---

# 117. Unit Tests

Examples:

```text
Measurement validation
Safety rules
Session transition rules
Technical normalization
Document metadata handling
```

These should not require external LLM calls.

---

# 118. Repository Tests

Repositories should test:

```text
machine filtering
document ownership
maintenance-case retrieval
transaction behavior
uniqueness constraints
```

---

# 119. Retrieval Tests

Retrieval tests must include:

```text
machine-isolation tests
exact alarm retrieval
semantic retrieval
metadata preservation
```

---

# 120. LangGraph Tests

Graph tests should verify:

```text
correct routing
interrupt
resume
checkpoint recovery
loop limits
insufficient-evidence path
safety-block path
root-cause path
```

External LLM behavior may be replaced by deterministic fakes in unit/integration graph tests.

---

# 121. AI Schema Tests

Every structured model output should have tests for:

```text
valid output
missing fields
invalid enums
malformed JSON
unexpected extra fields
retry behavior
```

---

# 122. API Integration Tests

Examples:

```text
Start diagnostic session
Submit fault
Submit measurement
Resume session
Resolve session
Retrieve maintenance case
```

---

# 123. End-to-End Demo Test

A test should cover the official MVP flow:

```text
Login
↓
Open machine
↓
Start diagnosis
↓
Submit fault
↓
Retrieve evidence
↓
Request measurement
↓
Interrupt
↓
Submit measurement
↓
Resume
↓
Probable cause
↓
Resolve
↓
Maintenance memory
```

---

# 124. ASR Tests

ASR integration tests should include:

```text
valid audio
unsupported audio
empty recording
ASR failure
transcript return
text fallback
```

Research accuracy tests are separate from functional ASR integration tests.

---

# 125. Migration Architecture

Database schema changes must use:

```text
Alembic
```

All schema changes should have explicit migrations.

Do not depend on runtime automatic table creation for the final project.

---

# 126. Database Constraints

Important invariants should use database constraints where appropriate.

Examples:

```text
document.machine_id NOT NULL

diagnostic_session.machine_id NOT NULL

maintenance_case.diagnostic_session_id UNIQUE

valid foreign keys
```

Database constraints complement application validation.

---

# 127. Soft vs Hard Delete

MVP may use hard deletion for setup data where safe.

However, resolved maintenance cases and audit-relevant diagnostic history should not be casually deleted by normal technician workflows.

Detailed retention policy may evolve later.

---

# 128. Time Handling

All backend timestamps should be stored consistently.

Preferred:

```text
UTC in database
```

Frontend converts for display.

---

# 129. Identifier Strategy

Internal records should use stable globally unique identifiers.

Recommended:

```text
UUID
```

Human-friendly labels may exist separately.

Example:

```text
internal id:
UUID

display code:
PKG-001
```

---

# 130. Machine QR Identifier

QR should reference stable machine identity.

Preferred:

```text
machine public ID / UUID
```

or stable slug/code.

It should not embed sensitive internal information unnecessarily.

---

# 131. Concurrency

A technician may leave a diagnostic session open and later resume it.

The system should protect against conflicting submissions where practical.

Example:

```text
session version
```

or controlled session-state validation can prevent:

```text
two incompatible resumes
```

from corrupting graph state.

---

# 132. Optimistic Concurrency

Where needed, session update APIs may use:

```text
version
```

or equivalent state checks.

Example:

```text
Client submits input for version 7
Database is already version 8
↓
409 Conflict
```

The exact implementation may be determined during coding.

---

# 133. Session State Transition Validation

The backend must validate legal transitions.

Examples:

Allowed:

```text
WAITING_FOR_TECHNICIAN
→
ANALYZING
```

Not allowed:

```text
RESOLVED
→
ANALYZING
```

unless explicit administrative recovery behavior exists.

---

# 134. Diagnostic State Machine

Conceptual transitions:

```text
NEW
↓
WAITING_FOR_FAULT_DESCRIPTION
↓
ANALYZING
↓
WAITING_FOR_TECHNICIAN
↕
ANALYZING
↓
PROBABLE_CAUSE_IDENTIFIED
↓
WAITING_FOR_RESOLUTION
↓
RESOLVED
```

Alternative exits:

```text
CANCELLED
FAILED
INSUFFICIENT_EVIDENCE / escalation
```

The final status model should avoid contradictory application and LangGraph states.

---

# 135. Domain Events vs Infrastructure Events

Diagnostic events are domain records.

Examples:

```text
MEASUREMENT_SUBMITTED
ROOT_CAUSE_ASSESSED
```

ARQ job events are infrastructure events.

Examples:

```text
DOCUMENT_JOB_STARTED
```

These should not be conflated.

---

# 136. Document Schematics Limitation

MVP document ingestion primarily targets textual and OCR-readable information.

Complex electrical schematic reasoning from images is technically difficult and must not be overclaimed.

For MVP v0.1:

```text
schematics may be stored
referenced
and surfaced to technicians
```

but fully automatic diagram connectivity reasoning is not required.

---

# 137. Multimodal Future Compatibility

The architecture should allow future document/page-image retrieval.

Potential future evidence:

```text
diagram page image
thermal image
machine photo
component photo
```

This should not complicate MVP implementation.

---

# 138. Parts Compatibility Architecture

`Parts Compatibility Lite` remains SHOULD HAVE.

If implemented:

```text
parts
part_specs
replacement_candidates
```

may be introduced.

Compatibility should use:

```text
deterministic constraints
```

before LLM explanation.

This module should remain isolated from the core diagnostic workflow until needed.

---

# 139. No Automatic PLC Integration

MVP does not connect directly to:

```text
PLC
SCADA
Modbus
OPC-UA
CNC
```

Technician observations and measurements are entered manually.

This protects scope and safety.

---

# 140. Future PLC Integration Boundary

Future architecture may introduce:

```text
Machine Telemetry Adapter
```

that produces structured observations.

The diagnostic graph should then consume telemetry similarly to technician measurements.

The core reasoning model should not require redesign.

---

# 141. No Autonomous Machine Control

The backend must not contain MVP APIs such as:

```text
/start-machine
/reset-vfd
/write-plc
/disable-interlock
```

SALLIH is a diagnostic copilot.

---

# 142. Performance Strategy

MVP performance focus:

```text
interactive diagnosis
reliable persistence
practical retrieval
acceptable ASR latency
```

not:

```text
thousands of concurrent factories
```

---

# 143. Database Indexing

Likely relational indexes include:

```text
machine_documents(machine_id)
document_chunks(machine_id)
diagnostic_sessions(machine_id)
diagnostic_sessions(technician_id)
maintenance_cases(machine_id)
diagnostic_events(session_id)
```

Vector indexes may be added based on pgvector configuration and dataset size.

---

# 144. Retrieval Dataset Size

The MVP dataset will be small enough that architecture should prioritize:

```text
correctness
provenance
machine filtering
```

over premature search optimization.

---

# 145. Caching

Complex caching is not required initially.

Potential future caching:

```text
machine metadata
frequently used alarm lookups
embedding results
```

should only be introduced after measuring actual bottlenecks.

---

# 146. Security Boundaries

Minimum architecture requirements:

```text
Authentication
Server-side authorization
Machine ownership/context validation
Input validation
File validation
Secret isolation
SQL-safe repository layer
Provider key protection
```

---

# 147. Uploaded File Validation

Document uploads should validate at least:

```text
file type
maximum size
PDF readability
```

The server must not trust the filename extension alone.

---

# 148. Prompt Injection Protection

Retrieved machine documents are untrusted data.

Architecture:

```text
System Instructions
      ↓
Prompt Template
      ↓
Explicit Retrieved Context Boundary
```

Document contents must not be treated as executable instructions.

---

# 149. Generated Content Boundary

LLM-generated explanation is presentation content.

Structured application state is updated only from validated structured outputs.

This separation reduces accidental state corruption.

---

# 150. Auditability

For an important AI conclusion, the project should be able to reconstruct:

```text
Which machine?
Which session?
Which technician inputs?
Which document chunks?
Which maintenance cases?
Which model?
Which prompt version?
Which hypotheses?
Which diagnostic actions?
Which final confirmation?
```

This is important for both research and debugging.

---

# 151. Architecture for Reproducibility

Evaluation results should record:

```text
application commit
dataset version
model provider
model name
prompt versions
embedding model
retrieval configuration
ASR model
```

where practical.

---

# 152. Versioned AI Configuration

A conceptual configuration record:

```text
AIConfig
--------
llm_provider
llm_model
embedding_provider
embedding_model
prompt_versions
retrieval_config_version
safety_rules_version
```

The first implementation may store this through code/configuration rather than a database table.

---

# 153. Development vs Evaluation Modes

The application should support configuration modes.

Example:

```text
APP_MODE=development
APP_MODE=evaluation
```

Evaluation mode may enable:

```text
additional trace metadata
deterministic test providers
scripted technician input
```

Production-like user behavior must not depend on benchmark ground truth.

---

# 154. Fake Providers for Testing

Testing should include fake implementations:

```text
FakeLLMProvider
FakeEmbeddingProvider
FakeASRProvider
```

This enables deterministic test suites without paying for external APIs.

---

# 155. External Provider Calls in Tests

Normal CI tests should not require real paid provider calls.

Live provider tests should be:

```text
explicit
separate
optional
```

or part of controlled evaluation runs.

---

# 156. Cost Observability

Development/evaluation may record model usage where supported.

Example:

```text
input tokens
output tokens
estimated cost
```

This helps compare architecture choices.

Cost optimization is secondary to reliability in MVP.

---

# 157. Architecture Decision — Modular Monolith

**Decision:**

Use a modular monolith for the main backend.

**Reason:**

The MVP requires strong internal consistency but does not require independent service scaling.

**Future Trigger for Change:**

Real production requirements demonstrate a need for separately deployable components.

---

# 158. Architecture Decision — PostgreSQL + pgvector

**Decision:**

Use PostgreSQL as the primary database and pgvector for vector retrieval.

**Reason:**

Reduces infrastructure complexity while supporting both relational and semantic retrieval.

---

# 159. Architecture Decision — PostgreSQL LangGraph Checkpointing

**Decision:**

Use durable PostgreSQL-backed LangGraph checkpointing where supported by the chosen LangGraph version.

**Reason:**

Diagnostic sessions can pause for human actions and survive application restarts.

---

# 160. Architecture Decision — Redis + ARQ

**Decision:**

Use Redis + ARQ for background document-processing jobs.

**Reason:**

Document ingestion is asynchronous and should be isolated from interactive request latency.

Redis is not used as the primary source of business truth.

---

# 161. Architecture Decision — One Diagnostic Orchestrator

**Decision:**

MVP v0.1 uses:

```text
One Diagnostic Orchestrator
+
Tools
```

**Reason:**

Multi-agent architecture is not currently justified.

---

# 162. Architecture Decision — Provider Abstraction

**Decision:**

LLM, embedding, ASR, and storage implementations use explicit interfaces/adapters.

**Reason:**

Allows experimentation and future provider changes without contaminating domain logic.

---

# 163. Architecture Decision — Confirmed ASR Boundary

**Decision:**

ASR output does not enter diagnostic reasoning until technician confirmation.

**Reason:**

Technical-token transcription errors can materially affect diagnosis.

---

# 164. Architecture Decision — Human-Confirmed Maintenance Truth

**Decision:**

AI root-cause assessment and technician-confirmed root cause remain separate fields.

**Reason:**

AI inference must never silently become historical fact.

---

# 165. Architecture Decision — Document Evidence Provenance

**Decision:**

Evidence citations are constructed from stored document metadata.

**Reason:**

The LLM must not invent sources.

---

# 166. Architecture Decision — No Full Schematic Reasoning in MVP

**Decision:**

MVP stores and surfaces electrical schematics but does not promise reliable autonomous schematic connectivity reasoning.

**Reason:**

This would substantially increase multimodal reasoning complexity and project risk.

---

# 167. Architecture Decision — Text-First Diagnostic Engine

**Decision:**

Core diagnosis must work fully using text before voice is required.

**Reason:**

ASR should not block validation of the main agentic diagnostic architecture.

---

# 168. Architecture Decision — REST First

**Decision:**

Use standard HTTP APIs as the primary frontend/backend communication model.

SSE may be added for progress.

**Reason:**

The MVP does not require persistent realtime bidirectional communication.

---

# 169. Architecture Decision — No Live Machine Control

**Decision:**

No direct industrial control interfaces exist in MVP v0.1.

**Reason:**

Safety and scope.

---

# 170. Initial Implementation Layers

Recommended implementation order:

```text
Layer 1
Infrastructure foundation

PostgreSQL
Redis
FastAPI
Next.js
Migrations
Auth
```

Then:

```text
Layer 2
Machine domain

Machines
Machine Passport
Documents metadata
```

Then:

```text
Layer 3
Document intelligence

Upload
Storage
Worker
Parsing
Chunking
Embeddings
Retrieval
```

Then:

```text
Layer 4
Diagnostic core

Sessions
State
Evidence
Measurements
Hypotheses
```

Then:

```text
Layer 5
LangGraph

Fault understanding
Retrieval
Hypothesis generation
Interrupt/resume
Next actions
```

Then:

```text
Layer 6
Safety + Resolution

Safety rules
Root-cause assessment
Resolution
Maintenance Memory
```

Then:

```text
Layer 7
Voice

Whisper
Transcript confirmation
Technical normalization
```

Then:

```text
Layer 8
Evaluation

Scenarios
Baselines
Metrics
ASR benchmark
```

---

# 171. Proposed Implementation Phases

A possible roadmap:

```text
Phase 0 — Repository / Infrastructure Foundation

Phase 1 — Machine Passport

Phase 2 — Document Knowledge Base

Phase 3 — Diagnostic Session Domain

Phase 4 — Machine-Grounded RAG

Phase 5 — LangGraph Diagnostic Loop

Phase 6 — Human-in-the-Loop + Measurements

Phase 7 — Safety + Root Cause + Resolution

Phase 8 — Maintenance Memory

Phase 9 — Voice / ASR

Phase 10 — Evaluation & Benchmark

Phase 11 — Demo Hardening
```

The final roadmap will be defined in:

`12-implementation-roadmap.md`.

---

# 172. Architecture Anti-Patterns

The implementation must avoid the following.

## Anti-Pattern 1

```text
One giant FastAPI route containing all AI logic.
```

---

## Anti-Pattern 2

```text
One giant prompt controlling the entire diagnostic workflow.
```

---

## Anti-Pattern 3

```text
LLM directly writing database records.
```

---

## Anti-Pattern 4

```text
LLM deciding machine_id filtering.
```

---

## Anti-Pattern 5

```text
Chat history treated as the only diagnostic state.
```

---

## Anti-Pattern 6

```text
Vector retrieval used for exact identifiers only.
```

---

## Anti-Pattern 7

```text
LangGraph used only to call one LLM node.
```

---

## Anti-Pattern 8

```text
Multiple agents created before one reliable diagnostic workflow exists.
```

---

## Anti-Pattern 9

```text
Redis used as permanent maintenance history.
```

---

## Anti-Pattern 10

```text
Document page references generated by the LLM.
```

---

## Anti-Pattern 11

```text
Whisper transcript accepted automatically for critical alarm codes.
```

---

## Anti-Pattern 12

```text
AI probable root cause automatically saved as confirmed root cause.
```

---

## Anti-Pattern 13

```text
High-risk maintenance action bypasses deterministic safety validation.
```

---

# 173. Architecture Acceptance Criteria

The architecture is implementation-ready when all of the following are clearly supported.

## Frontend

- [ ] Next.js web application has clear API boundary.
- [ ] Voice recording can be integrated.
- [ ] Diagnostic structured state can be rendered.
- [ ] Evidence can be rendered separately from conversation.

## Backend

- [ ] FastAPI owns application API.
- [ ] Domain modules are separated.
- [ ] Pydantic validates external and AI inputs.
- [ ] SQLAlchemy/Alembic manage persistence.

## Database

- [ ] PostgreSQL is authoritative.
- [ ] pgvector supports retrieval.
- [ ] Machine-scoped records preserve machine ownership.
- [ ] Diagnostic history is persistent.

## LangGraph

- [ ] Every Diagnostic Session maps to a graph thread.
- [ ] Durable checkpoints are available.
- [ ] Interrupt/resume is supported.
- [ ] Routing uses structured state.
- [ ] Graph loops are bounded.

## RAG

- [ ] Exact and semantic retrieval paths exist.
- [ ] Retrieval is machine-scoped.
- [ ] Evidence provenance is preserved.
- [ ] Citations cannot be invented by model output.

## Documents

- [ ] Upload is separate from processing.
- [ ] Background worker processes documents.
- [ ] Failed documents cannot enter retrieval.
- [ ] Embedding version is traceable.

## ASR

- [ ] ASR uses provider boundary.
- [ ] Transcript review occurs before diagnosis.
- [ ] Text fallback exists.
- [ ] ASR may run locally.

## Safety

- [ ] Safety logic is separate from unrestricted LLM reasoning.
- [ ] Blocked actions cannot reach technician as instructions.
- [ ] No machine control exists.

## Maintenance Memory

- [ ] Resolved session produces one Maintenance Case.
- [ ] Maintenance Case can become searchable memory.
- [ ] Historical cases remain distinct from current truth.

## Background Processing

- [ ] Redis/ARQ responsibilities are bounded.
- [ ] Background-job retries are safe.
- [ ] Redis is not business source of truth.

## Evaluation

- [ ] Evaluation harness can run diagnostic scenarios.
- [ ] Scripted technician input is possible.
- [ ] Ground truth remains hidden from diagnostic model.

---

# 174. Official MVP Deployment Shape

The recommended MVP deployment is:

```text
                 ┌──────────────────┐
                 │     Browser      │
                 └────────┬─────────┘
                          │
                          ▼
                 ┌──────────────────┐
                 │     Next.js      │
                 └────────┬─────────┘
                          │
                          ▼
                 ┌──────────────────┐
                 │     FastAPI      │
                 │                  │
                 │ LangGraph        │
                 │ LangChain        │
                 │ Safety           │
                 │ Retrieval        │
                 └───┬────┬────┬────┘
                     │    │    │
              ┌──────┘    │    └───────────┐
              ▼           ▼                ▼
       ┌────────────┐ ┌─────────┐   ┌─────────────┐
       │ PostgreSQL │ │  Redis  │   │ File/Object │
       │ + pgvector │ └────┬────┘   │   Storage   │
       └────────────┘      │        └─────────────┘
                           ▼
                    ┌────────────┐
                    │ ARQ Worker │
                    └────────────┘

                 FastAPI / ASR boundary
                          │
                          ▼
                 ┌──────────────────┐
                 │ Whisper Large v3 │
                 │ Turbo            │
                 └──────────────────┘

                 FastAPI / LLM boundary
                          │
                          ▼
                 ┌──────────────────┐
                 │ LLM Provider     │
                 └──────────────────┘
```

---

# 175. Evolution Path

If SALLIH succeeds in a real factory pilot, the architecture can evolve incrementally.

Possible future evolution:

```text
MVP Modular Monolith
        ↓
Real Pilot
        ↓
Identify Actual Bottlenecks
        ↓
Extract Only Necessary Services
```

Possible future separations:

```text
ASR Service
Document Intelligence Service
Telemetry Service
Evaluation Service
```

Only when justified.

---

# 176. Future Industrial Connectivity

A future architecture may add:

```text
Machine Connector Layer

OPC-UA
Modbus
PLC APIs
SCADA integration
```

Data would enter SALLIH as structured machine observations.

This must remain read-only initially.

Autonomous control requires an entirely separate safety and product decision.

---

# 177. Future Multi-Agent Architecture

If future complexity requires specialized reasoning:

```text
Diagnostic Orchestrator
        │
        ├── Parts Intelligence Subgraph
        ├── PLC Diagnostic Subgraph
        └── Visual Inspection Subgraph
```

LangGraph subgraphs are preferred over an uncontrolled collection of independent agents.

---

# 178. Future Multi-Factory Architecture

Potential future entities:

```text
organization
factory
production_line
machine
```

MVP does not require them unless needed for clean initial data ownership.

The architecture should not prematurely implement enterprise tenancy.

---

# 179. Future On-Prem Architecture

Industrial customers may require:

```text
local database
local documents
local vector store
local ASR
possibly local LLM
```

The current abstractions should keep this option possible.

It is not an MVP requirement.

---

# 180. Architecture Principle — Reliability Over Framework Complexity

Framework usage is justified only where it supports SALLIH's product behavior.

Use:

```text
LangGraph
```

because diagnosis is stateful.

Use:

```text
LangChain
```

where it simplifies model/retrieval/tool integrations.

Do not use either framework merely to increase perceived technical complexity.

---

# 181. Architecture Principle — State Over Chat

The central backend abstraction is:

```text
Diagnostic Session
```

not:

```text
Chat Conversation
```

Conversation is one representation of the session.

The session contains:

```text
Fault
Evidence
Hypotheses
Measurements
Actions
Safety
Assessment
Resolution
```

---

# 182. Architecture Principle — Evidence Over Generated Text

The system architecture must make:

```text
evidence
```

a first-class object.

Evidence is not just text appended to an LLM answer.

It has:

```text
identity
provenance
source
machine
page
session relationship
```

---

# 183. Architecture Principle — Human Reality Over AI Inference

The system architecture must distinguish:

```text
AI probable root cause
```

from:

```text
technician-confirmed root cause
```

at the database level.

This distinction must not rely only on wording in the UI.

---

# 184. Architecture Principle — Machine Isolation by Code

Machine-scoped security and retrieval boundaries must be enforced through:

```text
repository queries
database relationships
service validation
```

not prompts.

---

# 185. Architecture Principle — Fail Recoverably

External AI services will fail.

The architecture assumes:

```text
LLM timeout can happen
ASR can fail
embedding call can fail
worker can crash
network can disconnect
```

Therefore SALLIH must persist valid state before depending on the next external operation.

---

# 186. Architecture Principle — Expand from Real Need

The MVP architecture intentionally leaves room for:

```text
telemetry
parts intelligence
multimodal inspection
more machines
more agents
on-prem deployment
```

but does not implement them before customer and technical validation.

---

# 187. First Architecture Review Gate

After this document is completed, the project has enough information for an external technical review.

The reviewer should read:

```text
01-product-vision.md
02-problem-and-users.md
03-mvp-scope.md
04-user-flows.md
05-functional-requirements.md
06-ai-system-design.md
07-data-and-datasets.md
08-asr-and-finetuning.md
09-system-architecture.md
```

The reviewer must evaluate:

```text
Requirement contradictions
Missing state transitions
Database ownership
LangGraph responsibilities
RAG design
Checkpoint strategy
Human-in-the-loop design
Safety boundaries
ASR boundary
Background-job design
Scalability assumptions
Overengineering
Underengineering
Implementation blockers
```

No production implementation should begin before critical architecture issues from this review are resolved.

---

# 188. Final System Architecture Statement

SALLIH MVP v0.1 is architected as:

> **A modular web application with a Next.js technician interface, FastAPI application backend, PostgreSQL/pgvector machine knowledge store, LangGraph stateful diagnostic orchestrator, LangChain AI integrations, Redis/ARQ background document processing, machine-specific RAG, locally compatible Whisper ASR, deterministic safety controls, and persistent human-confirmed Maintenance Memory.**

The architecture deliberately avoids:

```text
unnecessary microservices
premature multi-agent complexity
autonomous machine control
unbounded AI behavior
opaque diagnostic state
untraceable evidence
```

The complete technical flow is:

```text
Technician
↓
Next.js
↓
FastAPI
↓
Persistent Diagnostic Session
↓
LangGraph Diagnostic Orchestrator
↓
Machine-Scoped Retrieval
↓
Evidence
↓
Hypotheses
↓
Next Diagnostic Action
↓
Safety Layer
↓
Human Interrupt
↓
Technician Measurement / Observation
↓
Graph Resume
↓
Updated Diagnosis
↓
Probable Root Cause
↓
Technician Confirmation
↓
Maintenance Case
↓
Machine Memory
```

The architecture exists to preserve one fundamental principle:

> **SALLIH does not replace industrial maintenance expertise. It structures, grounds, preserves, and amplifies it.**