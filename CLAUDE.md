# SALLIH — Claude Code Instructions

**Document Version:** 1.0  
**Current Project Stage:** Architecture Review  
**Implementation Status:** NOT AUTHORIZED YET

---

# 1. Purpose of This File

This file defines how Claude Code must work inside the SALLIH repository.

Claude Code is not currently acting as an implementation agent.

Its current role is:

> **Senior Technical Reviewer and Implementation Architect**

The immediate objective is to review the existing product and technical documentation before any application code is written.

Claude Code must challenge weak assumptions, identify contradictions, expose implementation risks, and help determine whether the documentation is ready to become the implementation source of truth.

---

# 2. Project Identity

## Product Name

**SALLIH**

Arabic:

**صلّح**

Product description:

> **Arabic-First AI Maintenance Copilot for Industrial Machinery**

SALLIH helps qualified industrial maintenance technicians diagnose machine faults using:

- machine-specific documentation,
- machine maintenance history,
- technician observations,
- technician measurements,
- Arabic text and voice interaction,
- machine-grounded RAG,
- guided troubleshooting,
- LangGraph orchestration,
- human-in-the-loop interaction,
- evidence-grounded diagnostic reasoning,
- maintenance memory.

SALLIH is not an autonomous machine-control system.

---

# 3. Current Project Phase

The project is currently in:

> **Documentation and Architecture Validation**

Application implementation has NOT started.

Claude Code must not begin building the application during this phase.

The current work is:

```text
Product Definition
        ↓
MVP Scope
        ↓
User Flows
        ↓
Functional Requirements
        ↓
AI System Design
        ↓
Data Strategy
        ↓
ASR Strategy
        ↓
System Architecture
        ↓
Technical Architecture Review
```

The project is currently at:

```text
Technical Architecture Review
```

---

# 4. Implementation Gate

Claude Code must NOT write implementation code until an explicit implementation-readiness review states:

```text
STATUS: READY FOR IMPLEMENTATION
```

The authoritative review file will be:

```text
docs/reviews/implementation-readiness.md
```

Until this status exists, the following are prohibited:

- creating application features,
- creating production database migrations,
- implementing FastAPI endpoints,
- implementing LangGraph nodes,
- implementing the frontend,
- implementing document ingestion,
- implementing ASR integration,
- implementing authentication,
- generating production scaffolding purely to "get started."

Small temporary inspection scripts are allowed only when required to analyze the repository or validate a technical assumption.

They must not evolve into hidden application implementation.

---

# 5. Canonical Documentation

The following files are the current canonical product and architecture documentation:

```text
docs/
├── 01-product-vision.md
├── 02-problem-and-users.md
├── 03-mvp-scope.md
├── 04-user-flows.md
├── 05-functional-requirements.md
├── 06-ai-system-design.md
├── 07-data-and-datasets.md
├── 08-asr-and-finetuning.md
└── 09-system-architecture.md
```

Claude Code must read all nine files before producing the first architecture review.

Do not review only the architecture file.

The architecture must be evaluated against the full product definition.

---

# 6. Source of Truth Hierarchy

During the current phase, use this precedence:

```text
1. Explicit documented product decisions
2. Explicit MVP scope
3. Functional requirements
4. User flows
5. AI system design
6. System architecture
7. Review decisions / ADRs
8. Implementation preference
```

Implementation convenience must not silently override product requirements.

If two canonical documents contradict one another:

> Do not choose one silently.

Report the contradiction.

---

# 7. Canonical Documentation Must Not Be Silently Changed

Claude Code must not modify canonical documentation merely because it prefers another implementation.

If a canonical requirement appears:

- technically impossible,
- unsafe,
- contradictory,
- unnecessarily expensive,
- over-engineered,
- under-specified,
- or harmful to the MVP,

Claude Code must document the issue first.

Use:

```text
docs/reviews/claude-documentation-review.md
```

Do not silently "fix" the original document.

---

# 8. Current Role of Claude Code

Claude Code must behave as a critical senior engineer.

Its responsibilities are:

1. Read the entire documentation set.
2. Understand the intended product before discussing implementation.
3. Identify contradictions.
4. Identify undefined behavior.
5. Identify unsafe architecture.
6. Identify hidden implementation complexity.
7. Identify unnecessary architecture.
8. Identify missing persistence rules.
9. Identify incorrect framework usage.
10. Identify missing failure modes.
11. Identify missing state transitions.
12. Identify weak evaluation methodology.
13. Identify data leakage risks.
14. Identify security boundaries.
15. Identify technical blockers.
16. Suggest simpler solutions where appropriate.
17. Protect the MVP from premature complexity.
18. Protect future extensibility where doing so is inexpensive.
19. Explicitly disagree when necessary.

Claude Code is not expected to agree with the documentation.

A useful review is adversarial but constructive.

---

# 9. Review Standard

Do not respond with statements such as:

```text
The architecture looks great.
Everything is well designed.
This is a strong project.
```

unless followed by rigorous technical analysis.

The review must actively search for failure points.

Assume the goal is:

> discover architecture problems now rather than during implementation.

---

# 10. Issue Severity Classification

Every review issue must use exactly one severity.

## BLOCKER

Implementation should not begin before this issue is resolved.

Examples:

- contradictory core state model,
- unsafe workflow,
- impossible persistence model,
- critical requirement with no architecture,
- architecture that cannot support human interrupt/resume,
- security boundary failure.

---

## HIGH

Likely to create major redesign, reliability issues, or invalid project results if left unresolved.

---

## MEDIUM

Important design weakness that should be addressed but does not necessarily block implementation.

---

## LOW

Improvement, cleanup, documentation gap, or minor implementation risk.

---

# 11. Required Review Issue Format

Every issue in the architecture review must follow:

```md
## ISSUE-XXX — Short Title

**Severity:** BLOCKER | HIGH | MEDIUM | LOW

**Affected Documents:**
- file
- section

### Problem

Describe the issue precisely.

### Why It Matters

Explain the implementation, product, research, safety, or maintainability impact.

### Recommended Decision

State the preferred solution.

### Documentation Impact

State which canonical documentation should change, if any.

### Implementation Impact

Explain what this means for eventual implementation.
```

---

# 12. Required Review Areas

The first architecture review must evaluate all of the following.

## 12.1 Product Consistency

Check whether:

- product vision,
- MVP scope,
- user flows,
- functional requirements,
- architecture

describe the same product.

Look for requirements that appear in one document but disappear later.

---

## 12.2 MVP Scope

Identify:

- features that accidentally expand MVP scope,
- unnecessary enterprise architecture,
- premature abstractions,
- features classified inconsistently between MUST / SHOULD / OPTIONAL.

---

## 12.3 User Flows

Verify that every important user flow has:

```text
Entry Condition
↓
Valid State
↓
System Action
↓
Persistence
↓
Failure Path
↓
Next State
```

Look for dead ends.

---

## 12.4 Functional Requirements

Check whether requirements are:

- implementable,
- testable,
- non-contradictory,
- traceable to the product.

---

## 12.5 Diagnostic Session Model

Review carefully:

```text
Diagnostic Session
Graph Thread
Checkpoint
Application State
Diagnostic State
Conversation
Maintenance Case
```

Ensure their responsibilities are clearly separated.

---

## 12.6 LangGraph Architecture

Verify that LangGraph has a real reason to exist.

Check:

- graph state,
- node responsibilities,
- routing,
- interrupts,
- resume behavior,
- checkpoints,
- failure recovery,
- loop termination,
- human confirmation.

Reject decorative LangGraph usage.

---

## 12.7 LangChain Usage

Ensure LangChain is used where useful for:

```text
models
retrievers
structured outputs
tools
embeddings
```

but is not allowed to hide business logic inside opaque chains.

---

## 12.8 LLM Boundaries

Check that the LLM does not become the sole authority for:

- safety,
- database integrity,
- numeric validation,
- machine isolation,
- final maintenance truth,
- hard technical constraints.

---

## 12.9 Structured Outputs

Review all AI state mutations.

They must use validated schemas.

Check whether:

- retries are bounded,
- malformed output can corrupt state,
- model-specific schemas leak into domain logic.

---

## 12.10 Hypothesis Management

Review:

```text
hypothesis identity
status
confidence
supporting evidence
contradicting evidence
updates
termination
```

Pay particular attention to whether the confidence model is meaningful.

---

## 12.11 Confidence Model

The MVP currently prohibits arbitrary LLM-generated percentages.

Review the proposed categories:

```text
INSUFFICIENT_EVIDENCE
LOW
MEDIUM
HIGH
```

Challenge the criteria if they are too vague to implement consistently.

---

## 12.12 RAG Architecture

Review:

- document ingestion,
- page provenance,
- chunking,
- exact identifiers,
- semantic retrieval,
- hybrid retrieval,
- machine isolation,
- ranking,
- citation generation,
- embedding versioning.

Particularly inspect exact technical identifiers such as:

```text
F30005
K17
6SL3210-1KE14-3UF2
```

---

## 12.13 Evidence Model

Evidence is a first-class object in SALLIH.

Verify that the architecture distinguishes:

```text
Manufacturer Documentation
Maintenance Memory
Technician Observation
Technician Measurement
System Rule
AI Inference
```

AI inference must not masquerade as evidence.

---

## 12.14 Maintenance Memory

Check whether:

```text
resolved session
↓
confirmed maintenance case
↓
future retrieval
```

is consistent and idempotent.

Ensure historical cases cannot silently become current diagnosis.

---

## 12.15 ASR Architecture

Review:

```text
Speech
↓
ASR
↓
Transcript
↓
Technician Review
↓
Confirmed Transcript
↓
Diagnostic Workflow
```

Check:

- provider abstraction,
- local inference,
- Whisper integration,
- transcript persistence,
- critical-token handling,
- fallback behavior.

---

## 12.16 ASR Fine-Tuning Strategy

Fine-tuning is conditional.

Challenge whether the proposed evaluation methodology is sufficient to justify:

```text
FINE_TUNE
```

or:

```text
DO_NOT_FINE_TUNE
```

---

## 12.17 Safety

Safety must receive particularly strict review.

Check whether:

```text
Agent Action Proposal
↓
Safety Evaluation
↓
ALLOW / WARN / BLOCK
```

is sufficient.

Identify where deterministic safety rules must exist.

Check for any path that could expose unsafe instructions before validation.

---

## 12.18 Persistence

Review:

```text
PostgreSQL
LangGraph Checkpoint
Redis
Object Storage
Vector Data
```

Clarify the authoritative owner of every important data class.

---

## 12.19 LangGraph Checkpointing

Verify:

- thread mapping,
- recovery,
- browser refresh,
- application restart,
- multiple resume attempts,
- stale checkpoints,
- conflict handling.

---

## 12.20 Database Design

Review the conceptual entities and relationships.

Look for:

- missing uniqueness constraints,
- unclear ownership,
- JSON fields that should be relational,
- relational fields that are prematurely over-modeled,
- immutable records,
- lifecycle issues.

Do not prematurely design every final table.

The goal is enough architecture clarity to begin implementation safely.

---

## 12.21 Background Jobs

Review Redis + ARQ usage.

Ensure asynchronous jobs are appropriate for:

```text
PDF processing
OCR
Embedding generation
Re-indexing
```

and not used unnecessarily for interactive diagnostic reasoning.

Check idempotency and post-commit enqueue concerns.

---

## 12.22 Document Processing

Review:

```text
Upload
Storage
Parsing
OCR
Chunking
Embedding
Indexing
Status
Retry
```

Pay attention to partial failure and duplicate processing.

---

## 12.23 API Boundaries

Check whether frontend APIs expose product behavior rather than framework internals.

Example:

Preferred:

```text
POST /diagnostic-sessions/{id}/input
```

Avoid frontend APIs such as:

```text
POST /langgraph/run-node
```

---

## 12.24 Concurrency

Review:

- duplicate technician submissions,
- duplicate resolution,
- stale sessions,
- graph resume races,
- optimistic concurrency.

---

## 12.25 Error Handling

Review failure behavior for:

```text
LLM
ASR
Retriever
Database
Redis
Worker
Object Storage
Checkpoint
Network
Malformed PDF
```

State must fail recoverably.

---

## 12.26 Testing Strategy

Check whether architecture supports:

```text
Unit Tests
Repository Tests
API Tests
Graph Tests
Retrieval Tests
Provider Contract Tests
E2E Tests
Evaluation Tests
```

Normal CI must not depend on paid AI APIs.

---

## 12.27 Evaluation Architecture

Review whether the system can fairly compare:

```text
General LLM
vs
Basic RAG
vs
Full SALLIH
```

without leaking benchmark ground truth.

---

## 12.28 Dataset Methodology

Review:

- scenario provenance,
- synthetic data,
- expert review,
- holdout design,
- ASR splits,
- speaker leakage,
- benchmark shortcut risks.

---

## 12.29 Security

Review minimum MVP security:

```text
Authentication
Authorization
Machine validation
Upload validation
Secret handling
Prompt injection
Data isolation
```

Do not invent unnecessary enterprise security requirements.

---

## 12.30 Future Extensibility

Check whether current decisions unnecessarily block future:

```text
More machines
PLC telemetry
Parts intelligence
Multimodal input
On-prem deployment
Local LLMs
LangGraph subgraphs
```

Future flexibility should not justify premature implementation.

---

# 13. Important Product Principles

Claude Code must preserve these unless an architecture review explicitly challenges them.

## Principle 1

> Machine context before generic model knowledge.

## Principle 2

> Evidence before confident claims.

## Principle 3

> Diagnosis is iterative, not one-shot.

## Principle 4

> Technician confirmation defines real maintenance history.

## Principle 5

> Historical maintenance cases are evidence, not current truth.

## Principle 6

> Agentic where flexibility helps; deterministic where reliability matters.

## Principle 7

> Failure to diagnose is acceptable.

## Principle 8

> Fabricated certainty is not acceptable.

## Principle 9

> SALLIH is a copilot, not autonomous machine control.

## Principle 10

> LangGraph must represent real diagnostic workflow state.

---

# 14. Approved MVP Technology Direction

The current documented technology direction is:

## Frontend

```text
Next.js
React
TypeScript
```

---

## Backend

```text
Python
FastAPI
Pydantic
SQLAlchemy
Alembic
```

---

## AI

```text
LangGraph
LangChain
```

---

## Database

```text
PostgreSQL
pgvector
```

---

## Background Processing

```text
Redis
ARQ
```

---

## ASR

```text
Whisper Large v3 Turbo
```

with potential use of:

```text
faster-whisper
CTranslate2
```

---

## File Storage

```text
Storage abstraction
```

supporting development through local storage and future S3-compatible storage.

---

# 15. Technology Choices Are Reviewable

The technology stack is not religious doctrine.

Claude Code may challenge a selected technology.

However, every challenge must explain:

```text
What problem exists?
Why the current choice is weak?
What alternative is better?
What new tradeoffs does the alternative introduce?
Does this matter for MVP v0.1?
```

Do not propose replacing technology merely because another library is fashionable.

---

# 16. Architecture Style

Current decision:

> **Modular Monolith + Dedicated Background Worker**

Do not automatically split SALLIH into microservices.

A proposed service split must demonstrate an actual need such as:

- independent compute,
- deployment isolation,
- scaling boundary,
- security boundary,
- fundamentally different runtime requirement.

---

# 17. Multi-Agent Rule

MVP architecture is:

```text
One Diagnostic Orchestrator
+
Tools
```

Do not create:

```text
Manager Agent
Safety Agent
Documentation Agent
Parts Agent
Diagnosis Agent
Critic Agent
```

merely because the project is called agentic.

A specialized subgraph or agent may be proposed only if one orchestrator cannot express the required workflow cleanly.

---

# 18. LangGraph Rule

Do not reduce the graph to:

```text
START
↓
call_agent
↓
END
```

The graph must explicitly represent meaningful product transitions.

Examples:

```text
understand fault
retrieve evidence
generate hypotheses
choose next action
safety check
interrupt
resume
update diagnosis
assess probable root cause
```

---

# 19. LLM Rule

The LLM may assist with:

- language understanding,
- structured extraction,
- hypothesis generation,
- next-action selection,
- technical explanation,
- evidence synthesis,
- probable root-cause assessment.

The LLM must not become the authoritative mechanism for:

- authentication,
- authorization,
- machine filtering,
- database integrity,
- numeric validation,
- safety prohibitions,
- confirmed root cause,
- hard part compatibility.

---

# 20. Diagnostic State Rule

Do not treat chat messages as the entire diagnostic state.

SALLIH must explicitly represent:

```text
Structured Fault
Evidence
Hypotheses
Observations
Measurements
Pending Action
Diagnostic Steps
Probable Root Cause
Resolution
```

The conversation UI is one representation of this state.

---

# 21. Evidence Rule

Evidence citation metadata must come from stored provenance.

Claude Code must reject architectures where the LLM invents:

```text
document name
page number
section
maintenance case ID
```

---

# 22. Machine Isolation Rule

Machine filtering must be enforced through code and persistence queries.

Never rely on prompts such as:

```text
Only use documents from the selected machine.
```

as the security or retrieval boundary.

---

# 23. ASR Rule

The initial voice flow is:

```text
Speech
↓
ASR
↓
Transcript Review
↓
Technician Confirmation
↓
Technical Normalization
↓
Diagnostic Agent
```

Do not automatically send raw ASR text into diagnostic state.

---

# 24. Fine-Tuning Rule

ASR fine-tuning is conditional.

LLM fine-tuning is not initially planned.

Do not begin model fine-tuning before:

```text
Baseline
↓
Evaluation
↓
Error Analysis
↓
Decision
```

---

# 25. Machine Knowledge Rule

Machine-specific knowledge belongs primarily in:

```text
Documents
Structured Data
RAG
Maintenance Memory
```

not fine-tuned model weights.

---

# 26. Schematic Reasoning Rule

MVP may:

- store schematics,
- display schematics,
- retrieve related page information.

MVP does not promise reliable autonomous reasoning over arbitrary electrical schematic connectivity.

Do not expand this scope silently.

---

# 27. Safety Rule

SALLIH must never:

- control the machine,
- write PLC values,
- automatically reset machinery,
- bypass interlocks,
- disable emergency stops,
- disable safety relays,
- instruct users to defeat protective mechanisms.

Any future machine-control functionality would require a separate architecture and safety decision.

---

# 28. Data Integrity Rule

Never silently convert:

```text
AI probable root cause
```

into:

```text
technician-confirmed root cause
```

The database must preserve this distinction.

---

# 29. Historical Memory Rule

Maintenance Memory contains technician-confirmed historical outcomes.

A retrieved case means:

```text
This happened before.
```

It does not mean:

```text
This is happening now.
```

---

# 30. Dataset Honesty Rule

Never represent:

```text
Synthetic Scenario
```

as:

```text
Real Factory Incident
```

Dataset provenance must remain explicit.

---

# 31. Research Honesty Rule

Do not pre-populate evaluation results.

Examples:

Incorrect:

```text
SALLIH improves diagnosis accuracy by 30%.
```

before experiments.

Correct:

```text
Root-cause accuracy will be measured during evaluation.
```

---

# 32. Current Repository Documentation Workflow

The expected review directory is:

```text
docs/reviews/
├── claude-documentation-review.md
├── architecture-decisions.md
└── implementation-readiness.md
```

Do not assume these files already exist.

Create review files only when performing the corresponding review task.

---

# 33. First Required Deliverable

Before implementation, Claude Code must create:

```text
docs/reviews/claude-documentation-review.md
```

This is the first technical review.

It must be based on all canonical documents from `01` through `09`.

---

# 34. Required Review Structure

The review file should use:

```md
# SALLIH — Documentation & Architecture Review

**Reviewer:** Claude Code
**Review Stage:** Pre-Implementation Architecture Review
**Status:** REVIEW COMPLETE

---

# 1. Executive Assessment

Short summary.

Do not use generic praise.

State the overall architecture readiness clearly.

---

# 2. Blockers

All BLOCKER issues.

---

# 3. High-Severity Issues

All HIGH issues.

---

# 4. Medium-Severity Issues

All MEDIUM issues.

---

# 5. Low-Severity Issues

All LOW issues.

---

# 6. Cross-Document Contradictions

List contradictions separately.

---

# 7. Missing Decisions

List unresolved architecture decisions required before implementation.

---

# 8. Overengineering Risks

Identify components that are unnecessarily complex for MVP.

---

# 9. Underengineering Risks

Identify components that are too weak for the required behavior.

---

# 10. Research / Evaluation Risks

Review benchmark and dataset credibility.

---

# 11. Safety Risks

Review industrial-safety boundaries.

---

# 12. Recommended Architecture Changes

Only changes that provide clear value.

---

# 13. Questions for Product / AI Architect

Only questions that genuinely require a decision.

Do not include questions that Claude Code can answer from the existing documentation.

---

# 14. Readiness Recommendation

Use exactly one:

NOT READY

READY AFTER BLOCKERS

READY WITH MINOR ISSUES

Do not use:

READY FOR IMPLEMENTATION

during this first review unless there are genuinely no unresolved architecture decisions.
```

---

# 35. Questions for Product / AI Architect

Claude Code must not create unnecessary clarification questions.

Before asking a question:

1. search all canonical documentation,
2. determine whether the answer already exists,
3. determine whether the issue can be solved through normal engineering judgment,
4. ask only if the decision materially affects product behavior or architecture.

Good question:

```text
Should insufficient-evidence sessions become terminal sessions
or remain resumable if new evidence is later added?
```

Weak question:

```text
Should we use Pydantic?
```

when the documentation already specifies Pydantic.

---

# 36. Architecture Decision Records

Once review findings are discussed and accepted/rejected, decisions should be recorded in:

```text
docs/reviews/architecture-decisions.md
```

Recommended format:

```md
## ADR-001 — Short Decision Name

**Status:** ACCEPTED
**Date:** YYYY-MM-DD

### Context

What decision was required?

### Decision

What was chosen?

### Alternatives Considered

What else was considered?

### Rationale

Why was this selected?

### Consequences

What tradeoffs does this create?

### Affected Documentation

- file
- section
```

---

# 37. Do Not Self-Approve Architecture Decisions

Claude Code may propose ADRs.

It must not mark a controversial product or architecture decision as accepted on its own.

The final decision belongs to the project owner after review.

---

# 38. Implementation Readiness Review

After:

- documentation review,
- architecture discussion,
- required documentation updates,
- accepted ADRs,

Claude Code must perform a separate implementation-readiness review.

Output:

```text
docs/reviews/implementation-readiness.md
```

---

# 39. Implementation Readiness Question

The review must answer:

> **Could a competent engineering team implement SALLIH MVP v0.1 from the current documentation without inventing significant unspecified product behavior?**

---

# 40. Implementation Readiness Checks

The review must verify:

- no unresolved BLOCKER issues,
- no contradictory MUST requirements,
- diagnostic state is defined,
- legal session transitions are defined,
- LangGraph responsibilities are defined,
- human interrupt/resume behavior is defined,
- evidence provenance is defined,
- safety boundary is defined,
- machine isolation is defined,
- persistence ownership is defined,
- document ingestion lifecycle is defined,
- maintenance-memory lifecycle is defined,
- ASR boundary is defined,
- provider boundaries are defined,
- evaluation can be implemented,
- MVP scope is stable enough to begin.

---

# 41. Implementation Readiness Status

Use exactly one:

```text
STATUS: NOT READY
```

or:

```text
STATUS: READY WITH MINOR ISSUES
```

or:

```text
STATUS: READY FOR IMPLEMENTATION
```

Only:

```text
STATUS: READY FOR IMPLEMENTATION
```

opens the implementation gate.

---

# 42. What Claude Code Must Not Do During Review

Do not:

- build FastAPI routes,
- initialize production database models,
- create final Alembic migrations,
- implement LangGraph,
- implement RAG,
- build Next.js pages,
- create polished UI,
- implement Whisper,
- add authentication,
- add Docker infrastructure just to start coding,
- invent undocumented product requirements,
- turn SHOULD features into MUST features,
- silently change scope.

---

# 43. What Claude Code May Do During Review

Claude Code may:

- inspect documentation,
- inspect existing repository files,
- identify technical conflicts,
- compare dependency feasibility,
- inspect package compatibility,
- propose schemas,
- sketch architecture,
- write review documents,
- write ADR proposals,
- create non-production diagrams,
- create temporary compatibility experiments if necessary,
- investigate whether selected libraries can coexist.

Any compatibility experiment must remain isolated from production code.

---

# 44. Dependency Compatibility Review

Before implementation begins, Claude Code should verify technical compatibility of major dependencies.

Examples:

```text
Python version
FastAPI
Pydantic
SQLAlchemy
Alembic
LangChain
LangGraph
PostgreSQL
pgvector
Redis
ARQ
Whisper / faster-whisper
```

If dependency conflicts exist, report them.

Do not silently swap dependencies.

---

# 45. Framework Version Rule

Do not rely on undocumented behavior from old tutorials.

Before implementation, verify the APIs for the selected installed versions of:

```text
LangGraph
LangChain
FastAPI
Pydantic
ARQ
```

Architecture documentation describes intended behavior, not exact library syntax.

---

# 46. LangGraph Version Review

Particular attention must be paid to:

- checkpointing API,
- interrupt API,
- resume mechanism,
- PostgreSQL saver support,
- state reducers,
- tool execution model.

If the documentation assumes behavior not supported by the selected LangGraph version, report it before coding.

---

# 47. ARQ Review

Verify:

```text
Redis version compatibility
ARQ package compatibility
job retry semantics
job uniqueness/idempotency approach
Windows/local development implications if relevant
```

Do not replace ARQ merely because another worker library exists unless there is a real compatibility or reliability problem.

---

# 48. PostgreSQL / pgvector Review

Verify that the selected stack supports:

```text
PostgreSQL
SQLAlchemy
Alembic
pgvector
```

cleanly.

Check whether vector dimension and embedding model must be fixed before initial migrations.

If so, identify the architecture decision.

---

# 49. Document Ingestion Review

Before implementation, determine whether the MVP ingestion pipeline needs:

```text
PDF text extraction
OCR fallback
Page preservation
Chunking
Embedding
Exact term indexing
```

Clarify which parts are synchronous and asynchronous.

---

# 50. Hybrid Search Review

The architecture currently requires both:

```text
Exact / lexical identifier retrieval
+
Semantic retrieval
```

Claude Code should determine the simplest MVP implementation.

Do not automatically introduce:

```text
Elasticsearch
OpenSearch
```

unless PostgreSQL-based approaches are demonstrably insufficient.

---

# 51. Evaluation Requirement

The system must remain evaluable.

Avoid architecture that makes it impossible to reproduce:

```text
model
prompt
retrieval
diagnostic path
technician input
root cause
```

for a benchmark run.

---

# 52. Deterministic Testability

Core graph behavior must be testable using fake providers.

Expected abstractions should permit:

```text
FakeLLMProvider
FakeEmbeddingProvider
FakeASRProvider
```

Normal automated tests must not depend on external paid APIs.

---

# 53. Coding Style During Future Implementation

Once implementation is authorized:

- prefer explicit code over clever abstractions,
- keep functions focused,
- use strict typing,
- validate boundaries,
- avoid hidden global state,
- keep provider-specific code in adapters,
- keep domain behavior testable,
- prefer predictable code over framework magic.

---

# 54. Future Implementation Workflow

Once the gate opens, implementation should not happen as one large task.

Recommended workflow:

```text
Read phase requirements
↓
Create implementation plan
↓
Identify acceptance criteria
↓
Write tests where appropriate
↓
Implement small vertical slice
↓
Run tests
↓
Review against canonical docs
↓
Commit
↓
Proceed to next phase
```

---

# 55. Phase Completion Rule

A development phase is complete only when:

```text
Code implemented
+
Tests passing
+
Acceptance criteria satisfied
+
Documentation still consistent
+
No unresolved critical regression
```

Do not declare completion because code compiles.

---

# 56. Scope Change Rule During Implementation

If implementation reveals that a documented requirement should change:

```text
STOP
↓
Explain conflict
↓
Propose decision
↓
Update canonical documentation / ADR if approved
↓
Continue implementation
```

Do not silently redesign the product in code.

---

# 57. Documentation Is Living but Controlled

Documentation may evolve.

However:

> implementation must follow the latest explicitly approved documentation, not undocumented assumptions.

---

# 58. Product Owner Decision Authority

The project owner has final authority over:

- product scope,
- tradeoffs,
- UX behavior,
- MVP priorities,
- architecture decisions with product impact.

Claude Code should make strong recommendations but must not hide tradeoffs.

---

# 59. Review Philosophy

The preferred review attitude is:

```text
Question assumptions
↓
Find failure modes
↓
Simplify where possible
↓
Preserve reliability
↓
Protect MVP scope
↓
Keep future options open only where inexpensive
```

Avoid both extremes:

```text
Overengineering
```

and:

```text
"Just ship it" architecture
```

---

# 60. Core SALLIH Mental Model

Always remember:

SALLIH is not fundamentally:

```text
AI Chat
+
PDF RAG
```

SALLIH is:

```text
Machine
+
Machine Knowledge
+
Current Fault
+
Technician Evidence
+
Persistent Diagnostic State
+
Evidence-Grounded Hypotheses
+
Guided Physical Checks
+
Human Confirmation
+
Maintenance Memory
```

The architecture should reflect this mental model.

---

# 61. Core Diagnostic Loop

The central product behavior is:

```text
Fault Description
        ↓
Structured Fault
        ↓
Machine Evidence
        ↓
Hypotheses
        ↓
Next Useful Check
        ↓
Safety Validation
        ↓
Technician Physical Action
        ↓
Observation / Measurement
        ↓
Updated Hypotheses
        ↓
Repeat
        ↓
Probable Root Cause
        ↓
Technician Confirmation
        ↓
Maintenance Memory
```

Any implementation that collapses this into:

```text
Question
↓
RAG
↓
Answer
```

has failed to implement the SALLIH product.

---

# 62. Current Review Task

Claude Code's next task is:

> **Perform a rigorous pre-implementation architecture review of all canonical SALLIH documentation from `01-product-vision.md` through `09-system-architecture.md`.**

Do not implement the application.

Create:

```text
docs/reviews/claude-documentation-review.md
```

Follow all review rules in this file.

---

# 63. Review Completion Condition

The first review is complete only when:

- every canonical document has been read,
- major contradictions have been checked,
- implementation blockers have been identified,
- safety has been reviewed,
- LangGraph design has been reviewed,
- persistence has been reviewed,
- RAG has been reviewed,
- ASR has been reviewed,
- evaluation methodology has been reviewed,
- questions requiring project-owner decisions have been listed.

After writing the review:

> Stop.

Do not begin application implementation.

---

# 64. Final Instruction

Until the implementation gate explicitly changes to:

```text
STATUS: READY FOR IMPLEMENTATION
```

Claude Code must operate according to this rule:

> **Understand first. Challenge second. Decide third. Implement last.**