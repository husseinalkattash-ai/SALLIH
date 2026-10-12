# SALLIH — Functional Requirements

**Document Version:** 1.0  
**Project Stage:** MVP v0.1  
**Status:** Implementation Baseline

---

# 1. Purpose

This document defines the functional requirements for SALLIH MVP v0.1.

It converts the product vision, user needs, MVP scope, and user flows into explicit and testable system requirements.

This document must be used together with:

- `01-product-vision.md`
- `02-problem-and-users.md`
- `03-mvp-scope.md`
- `04-user-flows.md`

The purpose of this document is to ensure that implementation decisions can be traced back to a concrete product requirement.

---

# 2. Requirement Format

Each functional requirement uses the format:

```text
FR-<DOMAIN>-<NUMBER>
```

Example:

```text
FR-DIAG-001
```

Each requirement may contain:

- Requirement
- Priority
- Rationale
- Acceptance Criteria
- Notes

Priority levels:

```text
MUST
SHOULD
OPTIONAL
```

Definitions:

## MUST

Required for MVP v0.1 completion.

## SHOULD

Important but may be deferred if it threatens completion of MUST requirements.

## OPTIONAL

May be implemented only after the core MVP is stable.

---

# 3. Functional Requirement Domains

The functional requirements are grouped into the following domains:

```text
AUTH      Authentication
MACH      Machine Passport
QR        Machine Identification
DOC       Document Management
RAG       Retrieval and Evidence
DIAG      Diagnostic Sessions
FAULT     Fault Understanding
HYP       Hypothesis Management
GRAPH     LangGraph Orchestration
TOOL      Agent Tools
HITL      Human-in-the-Loop
OBS       Technician Observations
MEAS      Measurements
ROOT      Root Cause Assessment
RES       Resolution
MEM       Maintenance Memory
VOICE     Voice Recording
ASR       Automatic Speech Recognition
SAFE      Safety
UI        User Interface
ERR       Error Handling
AUDIT     Audit and Diagnostic Timeline
PERSIST   Persistence and Recovery
PROVIDER  AI Provider Abstraction
PERF      Performance
ADMIN     Minimal Administration
EVAL      Evaluation Support
```

---

# 4. Authentication Requirements

## FR-AUTH-001 — Technician Sign In

**Priority:** MUST

The system must allow an authorized technician to sign in using valid credentials.

### Acceptance Criteria

- User can submit credentials.
- Valid credentials create an authenticated session.
- Invalid credentials are rejected.
- User receives a clear error message.
- Authentication state persists across normal page navigation.

---

## FR-AUTH-002 — Protected Application Access

**Priority:** MUST

Unauthenticated users must not access protected machine, diagnostic, or maintenance data.

### Acceptance Criteria

Attempting to access protected routes without authentication must redirect or require authentication.

---

## FR-AUTH-003 — QR Authentication Redirect

**Priority:** MUST

If an unauthenticated technician scans a machine QR code, the system must preserve the requested destination.

### Acceptance Criteria

```text
QR machine URL
↓
Login
↓
Successful authentication
↓
Requested Machine Passport
```

---

## FR-AUTH-004 — MVP Role Model

**Priority:** MUST

MVP v0.1 must support at least one authenticated technician role.

Complex enterprise RBAC is not required.

---

# 5. Machine Passport Requirements

## FR-MACH-001 — View Machines

**Priority:** MUST

The technician must be able to view machines available within the MVP environment.

### Minimum Display Fields

- Machine name
- Machine ID
- Manufacturer
- Model
- Location
- Status

---

## FR-MACH-002 — Open Machine Passport

**Priority:** MUST

The technician must be able to open a dedicated Machine Passport.

---

## FR-MACH-003 — Machine Metadata

**Priority:** MUST

The Machine Passport must support:

```text
machine_id
name
manufacturer
model
serial_number
year
location
description
status
```

---

## FR-MACH-004 — Machine Documents

**Priority:** MUST

The Machine Passport must expose machine-specific technical documents.

---

## FR-MACH-005 — Machine Maintenance History

**Priority:** MUST

The Machine Passport must expose resolved maintenance cases belonging to the machine.

---

## FR-MACH-006 — Start Diagnosis

**Priority:** MUST

The Machine Passport must provide an action for starting a new Diagnostic Session.

---

## FR-MACH-007 — Strict Machine Isolation

**Priority:** MUST

Machine-scoped information must never silently include records from another machine.

### Acceptance Criteria

If the current machine is:

```text
PKG-001
```

then document retrieval, maintenance-memory search, and diagnostic context must default to:

```text
machine_id = PKG-001
```

---

# 6. Machine QR Requirements

## FR-QR-001 — Machine QR Code

**Priority:** MUST

Each MVP machine must have a QR code that resolves to its Machine Passport.

---

## FR-QR-002 — Invalid QR Handling

**Priority:** MUST

If a QR references an unknown machine, the system must display a clear error.

It must not automatically open a different machine.

---

## FR-QR-003 — Stable Machine Link

**Priority:** MUST

The QR target should use a stable machine identifier rather than UI-only state.

Example:

```text
/machines/PKG-001
```

---

# 7. Document Management Requirements

## FR-DOC-001 — Upload PDF Document

**Priority:** MUST

An authorized user must be able to upload PDF documentation to a selected machine.

---

## FR-DOC-002 — Document Ownership

**Priority:** MUST

Every uploaded document must belong to exactly one machine in MVP v0.1.

---

## FR-DOC-003 — Document Type

**Priority:** MUST

The system must support assigning a document type.

Possible values may include:

```text
MANUAL
ELECTRICAL_SCHEMATIC
ALARM_REFERENCE
PARTS_CATALOG
DATASHEET
OTHER
```

---

## FR-DOC-004 — Processing Status

**Priority:** MUST

Each uploaded document must have an explicit processing status.

Minimum statuses:

```text
UPLOADING
PROCESSING
READY
FAILED
```

---

## FR-DOC-005 — Text Extraction

**Priority:** MUST

The system must extract usable text from supported machine PDFs before indexing.

---

## FR-DOC-006 — Chunking

**Priority:** MUST

Extracted document text must be divided into retrievable chunks.

Chunks must not lose the identity of their source document.

---

## FR-DOC-007 — Document Metadata Preservation

**Priority:** MUST

Every retrievable chunk must preserve at least:

```text
machine_id
document_id
document_name
document_type
page_number
```

Where available, also preserve:

```text
section
component
heading
```

---

## FR-DOC-008 — Embedding and Indexing

**Priority:** MUST

READY documents must be indexed for semantic retrieval.

---

## FR-DOC-009 — Failed Documents Excluded from Retrieval

**Priority:** MUST

Documents with status:

```text
FAILED
```

must not be eligible for diagnostic retrieval.

---

## FR-DOC-010 — OCR for Scanned PDFs

**Priority:** SHOULD

The system should support OCR when machine documents contain scanned pages without extractable text.

---

## FR-DOC-011 — Document Deletion

**Priority:** SHOULD

An authorized user should be able to remove a document.

Associated retrieval data must no longer be available afterward.

---

# 8. Retrieval and RAG Requirements

## FR-RAG-001 — Machine-Scoped Retrieval

**Priority:** MUST

All diagnostic document retrieval must be constrained by the current `machine_id`.

---

## FR-RAG-002 — Fault-Aware Query Construction

**Priority:** MUST

Retrieval queries must be derived from the structured fault state when available.

Relevant fields may include:

```text
component
symptom
alarm_codes
timing
operating_condition
technician_observations
```

---

## FR-RAG-003 — Alarm Retrieval

**Priority:** MUST

If an alarm code is present, SALLIH must attempt to retrieve exact or highly relevant alarm documentation before generating unsupported explanations.

---

## FR-RAG-004 — Evidence Objects

**Priority:** MUST

Retrieved evidence must exist as structured application data, not only as text embedded in an LLM response.

A document evidence item should contain information such as:

```text
evidence_id
machine_id
document_id
document_name
document_type
page_number
section
content_excerpt
retrieval_score
```

---

## FR-RAG-005 — Evidence Citation

**Priority:** MUST

Technical claims based on machine documentation must be capable of referencing their evidence.

Example:

```text
Drive Manual
Page 74
Section 8.3
```

---

## FR-RAG-006 — No Invented Sources

**Priority:** MUST

The system must not generate fictional document names, page numbers, maintenance cases, or evidence references.

---

## FR-RAG-007 — No-Evidence State

**Priority:** MUST

If retrieval does not find sufficiently relevant machine-specific evidence, the system must expose that state.

Example:

```text
No strong machine-specific documentation was found for this issue.
```

---

## FR-RAG-008 — Evidence Source Distinction

**Priority:** MUST

The system must distinguish among:

```text
MANUFACTURER_DOCUMENT
MAINTENANCE_CASE
TECHNICIAN_OBSERVATION
TECHNICIAN_MEASUREMENT
AI_INFERENCE
```

---

## FR-RAG-009 — Previous Case Retrieval

**Priority:** MUST

The retrieval layer must support searching resolved maintenance cases for the same machine.

---

## FR-RAG-010 — Hybrid Maintenance Retrieval

**Priority:** SHOULD

Maintenance-memory retrieval should support a combination of:

- semantic similarity,
- alarm-code match,
- component match,
- symptom match,
- structured filters.

---

# 9. Diagnostic Session Requirements

## FR-DIAG-001 — Create Diagnostic Session

**Priority:** MUST

A persistent Diagnostic Session must be created before diagnostic orchestration begins.

---

## FR-DIAG-002 — Machine Association

**Priority:** MUST

Each Diagnostic Session must belong to exactly one machine.

---

## FR-DIAG-003 — Technician Association

**Priority:** MUST

Each Diagnostic Session must record the technician who initiated it.

---

## FR-DIAG-004 — Session Identifier

**Priority:** MUST

Each session must have a stable unique identifier.

---

## FR-DIAG-005 — Session Status

**Priority:** MUST

The session must maintain an explicit status.

Recommended statuses:

```text
NEW
WAITING_FOR_FAULT_DESCRIPTION
ANALYZING
WAITING_FOR_TECHNICIAN
READY_FOR_ASSESSMENT
PROBABLE_CAUSE_IDENTIFIED
WAITING_FOR_RESOLUTION
RESOLVED
CANCELLED
FAILED
```

---

## FR-DIAG-006 — Initial Fault Description

**Priority:** MUST

The session must preserve the original technician fault description exactly as submitted.

---

## FR-DIAG-007 — Structured Fault

**Priority:** MUST

The session must maintain a validated structured representation of the fault.

---

## FR-DIAG-008 — Persistent Session State

**Priority:** MUST

The session must preserve:

- observations,
- measurements,
- evidence,
- hypotheses,
- diagnostic steps,
- probable root cause,
- resolution information.

---

## FR-DIAG-009 — Resume Session

**Priority:** MUST

An active session waiting for technician input must be resumable later.

---

## FR-DIAG-010 — Cancel Session

**Priority:** MUST

The technician must be able to cancel an unresolved Diagnostic Session.

Cancelled sessions must not become confirmed Maintenance Memory.

---

## FR-DIAG-011 — Resolved Session Immutability

**Priority:** MUST

A resolved session must not be reused as the active diagnostic session for a new fault.

A new fault requires a new Diagnostic Session.

---

# 10. Fault Understanding Requirements

## FR-FAULT-001 — Natural-Language Fault Input

**Priority:** MUST

SALLIH must accept technician fault descriptions written in natural language.

---

## FR-FAULT-002 — Arabic Support

**Priority:** MUST

Fault descriptions must support Arabic input.

---

## FR-FAULT-003 — Arabic-English Code Switching

**Priority:** MUST

The system must support mixed expressions such as:

```text
الدرايف عم يعطي overcurrent.

الـ PLC ما عم يعطي output.

الكونتاكتور ما عم يلقط.
```

---

## FR-FAULT-004 — Structured Extraction

**Priority:** MUST

Natural-language fault descriptions must be converted into validated structured data.

Possible schema:

```json
{
  "component": "VFD",
  "symptom": "trip",
  "alarm_codes": ["F30005"],
  "timing": "after approximately 10 minutes",
  "operating_condition": null,
  "observations": []
}
```

---

## FR-FAULT-005 — Schema Validation

**Priority:** MUST

LLM-generated structured output must be validated before it mutates diagnostic state.

---

## FR-FAULT-006 — Controlled Retry

**Priority:** MUST

If structured output fails validation, the system may retry in a bounded and controlled manner.

---

## FR-FAULT-007 — Clarification Instead of Guessing

**Priority:** MUST

If reliable structured understanding cannot be produced, SALLIH must ask the technician for clarification.

---

## FR-FAULT-008 — Critical Token Preservation

**Priority:** MUST

The system must preserve critical values such as:

- alarm codes,
- model numbers,
- part numbers,
- manufacturer names,
- measurements.

These values must not be normalized in a way that changes their technical identity.

---

# 11. Hypothesis Management Requirements

## FR-HYP-001 — Explicit Hypotheses

**Priority:** MUST

SALLIH must maintain diagnostic hypotheses as structured state.

---

## FR-HYP-002 — Hypothesis Identifier

**Priority:** MUST

Each hypothesis must have a stable identifier during a Diagnostic Session.

---

## FR-HYP-003 — Hypothesis Description

**Priority:** MUST

Each hypothesis must contain a concise technical description.

---

## FR-HYP-004 — Supporting Evidence

**Priority:** MUST

A hypothesis should be able to reference supporting evidence.

---

## FR-HYP-005 — Contradicting Evidence

**Priority:** MUST

A hypothesis should be able to reference evidence that weakens it.

---

## FR-HYP-006 — Hypothesis Status

**Priority:** MUST

Hypotheses must support explicit status.

Minimum values:

```text
ACTIVE
LOW_PRIORITY
REJECTED
CONFIRMED
```

---

## FR-HYP-007 — Hypothesis Update

**Priority:** MUST

New technician evidence must be capable of changing hypothesis priority or status.

---

## FR-HYP-008 — No Hidden-Only Diagnosis

**Priority:** MUST

The diagnostic workflow must not depend entirely on hidden LLM reasoning.

The application must retain an inspectable structured diagnosis state.

---

## FR-HYP-009 — Confidence Representation

**Priority:** MUST

The MVP must not present arbitrary LLM-generated numerical confidence percentages as calibrated probabilities.

If confidence is exposed, use controlled categories such as:

```text
INSUFFICIENT_EVIDENCE
LOW
MEDIUM
HIGH
```

The exact semantics must be defined in `06-ai-system-design.md`.

---

# 12. LangGraph Orchestration Requirements

## FR-GRAPH-001 — LangGraph Diagnostic Workflow

**Priority:** MUST

The core diagnostic orchestration must use LangGraph.

---

## FR-GRAPH-002 — Real Workflow State

**Priority:** MUST

LangGraph must represent actual diagnostic workflow state.

It must not be used only as a decorative wrapper around a single LLM call.

---

## FR-GRAPH-003 — Initial Nodes

**Priority:** MUST

The initial diagnostic graph should support the functional equivalent of:

```text
understand_fault
load_machine_context
retrieve_evidence
generate_hypotheses
decide_next_action
safety_check
ask_technician
receive_observation
update_hypotheses
assess_root_cause
human_confirmation
save_maintenance_case
```

Exact node names may differ if behavior remains equivalent.

---

## FR-GRAPH-004 — Diagnostic Loop

**Priority:** MUST

The graph must support repeated cycles of:

```text
Hypotheses
↓
Next Diagnostic Action
↓
Technician Input
↓
Hypothesis Update
```

---

## FR-GRAPH-005 — Interrupt

**Priority:** MUST

LangGraph must support interrupting execution when technician input is required.

---

## FR-GRAPH-006 — Resume

**Priority:** MUST

LangGraph must resume the same diagnostic thread after technician input is received.

---

## FR-GRAPH-007 — Checkpointing

**Priority:** MUST

The workflow must use persistent checkpoints appropriate for recovering active sessions.

---

## FR-GRAPH-008 — Thread Association

**Priority:** MUST

Each Diagnostic Session must map unambiguously to its LangGraph thread/checkpoint context.

---

## FR-GRAPH-009 — Bounded Loop Behavior

**Priority:** MUST

The graph must not be allowed to loop indefinitely.

It must support termination conditions such as:

- sufficient evidence,
- unresolved due to insufficient evidence,
- technician cancellation,
- escalation,
- system failure.

---

## FR-GRAPH-010 — No Premature Multi-Agent Architecture

**Priority:** MUST

MVP v0.1 must use:

> one primary Diagnostic Orchestrator + tools

unless a later approved architecture decision explicitly changes this.

---

# 13. Agent Tool Requirements

## FR-TOOL-001 — Machine Context Tool

**Priority:** MUST

The Diagnostic Orchestrator must be able to load machine-specific structured context.

---

## FR-TOOL-002 — Documentation Retrieval Tool

**Priority:** MUST

The agent must be able to retrieve machine-specific documentation.

---

## FR-TOOL-003 — Alarm Lookup Tool

**Priority:** MUST

The agent must be able to search for alarm-code information.

This may share underlying retrieval infrastructure with documentation search.

---

## FR-TOOL-004 — Maintenance Memory Tool

**Priority:** MUST

The agent must be able to search previous resolved cases belonging to the machine.

---

## FR-TOOL-005 — Measurement Collection Contract

**Priority:** MUST

When the diagnostic workflow requires a structured measurement, the system must be able to represent what measurement is requested.

Example:

```json
{
  "measurement_type": "phase_current",
  "required_values": ["L1", "L2", "L3"],
  "unit": "A"
}
```

---

## FR-TOOL-006 — Safety Evaluation Tool or Rule Layer

**Priority:** MUST

A proposed diagnostic action must pass through safety evaluation before being presented to the technician.

---

## FR-TOOL-007 — Tool Output Validation

**Priority:** MUST

Tool outputs used to mutate application state must satisfy explicit schemas or deterministic validation.

---

## FR-TOOL-008 — Tool Failure Isolation

**Priority:** MUST

Failure of one tool call must not corrupt the entire Diagnostic Session.

---

# 14. Guided Troubleshooting Requirements

## FR-DIAG-012 — Next Useful Check

**Priority:** MUST

SALLIH must attempt to determine the next useful diagnostic check rather than returning only a generic list of causes.

---

## FR-DIAG-013 — One Primary Next Action

**Priority:** MUST

The UI should generally emphasize one primary diagnostic action at a time.

---

## FR-DIAG-014 — Explain Why

**Priority:** MUST

The system should briefly explain why a requested measurement or inspection is useful.

Example:

```text
Measure L1, L2 and L3 current to check whether the overcurrent is associated with phase imbalance.
```

---

## FR-DIAG-015 — Evidence-Based Action Selection

**Priority:** MUST

Next diagnostic actions should consider:

- current hypotheses,
- retrieved evidence,
- previous observations,
- previous measurements,
- safety level.

---

## FR-DIAG-016 — Avoid Immediate Replacement Recommendation

**Priority:** MUST

The system should not immediately recommend component replacement when diagnostic evidence remains insufficient.

---

## FR-DIAG-017 — Insufficient Evidence Result

**Priority:** MUST

The workflow must support a valid terminal or escalation result in which no reliable probable root cause can be identified.

---

# 15. Human-in-the-Loop Requirements

## FR-HITL-001 — Technician Input Interrupt

**Priority:** MUST

The graph must pause whenever physical technician input is required.

---

## FR-HITL-002 — Pending Question Persistence

**Priority:** MUST

The exact pending diagnostic question must survive session interruption.

---

## FR-HITL-003 — User Return

**Priority:** MUST

When the technician returns to an interrupted session, the pending action must be visible.

---

## FR-HITL-004 — Human Confirmation of Root Cause

**Priority:** MUST

The AI cannot create the final confirmed root cause without technician confirmation.

---

## FR-HITL-005 — Human Authority

**Priority:** MUST

The confirmed technician resolution is authoritative maintenance history.

AI hypotheses must remain distinct from the confirmed resolution.

---

# 16. Observation Requirements

## FR-OBS-001 — Free-Text Observation

**Priority:** MUST

The technician must be able to submit a natural-language observation.

Example:

```text
في صوت احتكاك من جهة الموتور.
```

---

## FR-OBS-002 — Observation Persistence

**Priority:** MUST

Each observation must be stored in the Diagnostic Session.

---

## FR-OBS-003 — Observation Evidence Type

**Priority:** MUST

Technician observations must be classified separately from manufacturer documentation and AI inference.

---

## FR-OBS-004 — Observation Timestamp

**Priority:** MUST

Observations must be timestamped.

---

# 17. Measurement Requirements

## FR-MEAS-001 — Structured Measurement Input

**Priority:** MUST

The system must support structured numeric diagnostic measurements.

---

## FR-MEAS-002 — Measurement Unit

**Priority:** MUST

Measurements must preserve their unit.

Example:

```text
A
V
Ω
°C
bar
rpm
```

---

## FR-MEAS-003 — Measurement Validation

**Priority:** MUST

Invalid numeric input must not enter diagnostic state.

---

## FR-MEAS-004 — Measurement Metadata

**Priority:** MUST

A stored measurement should preserve:

```text
measurement_type
values
unit
timestamp
technician_id
```

---

## FR-MEAS-005 — Measurement Evidence

**Priority:** MUST

Measurements must be available as diagnostic evidence.

---

## FR-MEAS-006 — Requested Measurement Schema

**Priority:** SHOULD

The backend should support a schema-driven measurement request so the frontend can render the correct form dynamically.

---

# 18. Root Cause Assessment Requirements

## FR-ROOT-001 — Probable Root Cause

**Priority:** MUST

When evidence is sufficient, SALLIH may produce a Probable Root Cause.

---

## FR-ROOT-002 — Root Cause Evidence

**Priority:** MUST

A Probable Root Cause must be associated with supporting evidence.

---

## FR-ROOT-003 — Probable vs Confirmed Distinction

**Priority:** MUST

The system must visually and structurally distinguish:

```text
Probable Root Cause
```

from:

```text
Confirmed Root Cause
```

---

## FR-ROOT-004 — Insufficient Evidence

**Priority:** MUST

If the system lacks enough evidence, it must not fabricate a root cause.

---

## FR-ROOT-005 — Confidence Category

**Priority:** MUST

If root-cause confidence is presented, it must use the controlled confidence model defined by the AI system design.

---

# 19. Resolution Requirements

## FR-RES-001 — Resolve Diagnostic Session

**Priority:** MUST

The technician must be able to resolve an active Diagnostic Session.

---

## FR-RES-002 — Confirmed Root Cause Required

**Priority:** MUST

Resolution must require:

```text
confirmed_root_cause
```

---

## FR-RES-003 — Corrective Action Required

**Priority:** MUST

Resolution must require:

```text
corrective_action
```

---

## FR-RES-004 — Optional Resolution Fields

**Priority:** SHOULD

The resolution may support:

```text
parts_replaced
downtime
additional_notes
```

---

## FR-RES-005 — Resolution Timestamp

**Priority:** MUST

The system must record the resolution time.

---

## FR-RES-006 — Resolved Status

**Priority:** MUST

Successful resolution must transition the session to:

```text
RESOLVED
```

---

# 20. Maintenance Memory Requirements

## FR-MEM-001 — Create Maintenance Case

**Priority:** MUST

Resolving a Diagnostic Session must create or finalize a structured Maintenance Case.

---

## FR-MEM-002 — Maintenance Case Machine Association

**Priority:** MUST

Each Maintenance Case must belong to exactly one machine.

---

## FR-MEM-003 — Maintenance Case Content

**Priority:** MUST

A Maintenance Case must preserve at least:

```text
machine_id
initial_fault_description
structured_symptoms
alarm_codes
observations
measurements
diagnostic_steps
probable_root_cause
confirmed_root_cause
corrective_action
resolved_at
```

---

## FR-MEM-004 — Evidence Preservation

**Priority:** MUST

Important evidence used during diagnosis must remain associated with the Maintenance Case.

---

## FR-MEM-005 — Searchable Memory

**Priority:** MUST

Resolved cases must become searchable by future diagnostic sessions.

---

## FR-MEM-006 — Historical Case Warning

**Priority:** MUST

When a similar previous case is retrieved, the system must not imply that the current fault is automatically identical.

---

## FR-MEM-007 — View Maintenance History

**Priority:** MUST

The technician must be able to view resolved Maintenance Cases.

---

## FR-MEM-008 — View Case Details

**Priority:** MUST

The technician must be able to inspect the complete details of a Maintenance Case.

---

## FR-MEM-009 — Maintenance Memory Indexing

**Priority:** MUST

Resolved case information relevant to future retrieval must be indexed or otherwise made searchable.

---

# 21. Voice Recording Requirements

## FR-VOICE-001 — Voice Fault Description

**Priority:** MUST

The technician must be able to record a voice-based fault description.

---

## FR-VOICE-002 — Recording Control

**Priority:** MUST

The user must be able to:

```text
Start Recording
Stop Recording
Record Again
```

---

## FR-VOICE-003 — Audio Upload

**Priority:** MUST

Recorded audio must be transferred to the backend or ASR service for transcription.

---

## FR-VOICE-004 — Recording Feedback

**Priority:** MUST

The interface must clearly indicate when recording is active.

---

## FR-VOICE-005 — Text Alternative

**Priority:** MUST

Voice must not be the only available fault-input method.

Text input must always remain available.

---

# 22. ASR Requirements

## FR-ASR-001 — Whisper Baseline

**Priority:** MUST

The initial ASR baseline must support Whisper Large v3 Turbo or an explicitly approved equivalent implementation.

---

## FR-ASR-002 — Arabic Speech

**Priority:** MUST

ASR must support Arabic speech.

---

## FR-ASR-003 — Code-Switched Speech

**Priority:** MUST

ASR must be evaluated against Arabic-English industrial code switching.

---

## FR-ASR-004 — Transcript Display

**Priority:** MUST

The ASR transcript must be displayed to the technician before being used for diagnosis.

---

## FR-ASR-005 — Transcript Editing

**Priority:** MUST

The technician must be able to edit the transcript.

---

## FR-ASR-006 — Transcript Confirmation

**Priority:** MUST

Only the confirmed transcript should become authoritative diagnostic input.

---

## FR-ASR-007 — Record Again

**Priority:** MUST

The user must be able to discard the transcript and record again.

---

## FR-ASR-008 — Technical Token Accuracy

**Priority:** MUST

The project must separately evaluate ASR performance on important industrial tokens such as:

```text
PLC
VFD
contactor
overcurrent
F30005
Siemens
Schneider
K17
```

---

## FR-ASR-009 — Fine-Tuning Gate

**Priority:** MUST

ASR fine-tuning must not be assumed necessary before baseline evaluation.

The required process is:

```text
Baseline
↓
Measure
↓
Identify weaknesses
↓
Collect targeted data
↓
Fine-tune or adapt
↓
Re-evaluate
```

---

## FR-ASR-010 — Industrial Noise Evaluation

**Priority:** SHOULD

ASR should be evaluated under realistic or simulated industrial noise.

---

# 23. Safety Requirements

## FR-SAFE-001 — Diagnostic Action Safety Classification

**Priority:** MUST

Each proposed physical diagnostic action must be classified according to safety risk.

Initial categories:

```text
LOW
MEDIUM
HIGH
```

---

## FR-SAFE-002 — Safety Check Before User Presentation

**Priority:** MUST

A physical diagnostic action must pass safety evaluation before being shown to the technician.

---

## FR-SAFE-003 — High-Risk Warning

**Priority:** MUST

HIGH-risk actions must include explicit qualified-technician and site-procedure requirements.

---

## FR-SAFE-004 — Safety Bypass Prohibition

**Priority:** MUST

SALLIH must never instruct the technician to bypass or disable a safety mechanism.

Examples include:

- interlocks,
- emergency stops,
- safety relays,
- protective guards,
- mandatory lockout mechanisms.

---

## FR-SAFE-005 — Autonomous Control Prohibition

**Priority:** MUST

SALLIH must not automatically:

- control machinery,
- reset machinery,
- write PLC values,
- activate actuators,
- bypass safety systems.

---

## FR-SAFE-006 — Uncertain Safety Classification

**Priority:** MUST

If safety level cannot be confidently determined, the system must choose the safer behavior.

---

## FR-SAFE-007 — Safety Auditability

**Priority:** MUST

The safety classification associated with a requested diagnostic action must be recorded in the diagnostic timeline.

---

# 24. User Interface Requirements

## FR-UI-001 — Machines Screen

**Priority:** MUST

The application must provide a Machines screen.

---

## FR-UI-002 — Machine Passport Screen

**Priority:** MUST

The application must provide a Machine Passport screen.

---

## FR-UI-003 — Troubleshooting Screen

**Priority:** MUST

The application must provide a dedicated AI Troubleshooting screen.

---

## FR-UI-004 — Maintenance History Screen

**Priority:** MUST

The application must provide a Maintenance History screen.

---

## FR-UI-005 — Maintenance Case Details Screen

**Priority:** MUST

The application must provide a Maintenance Case Details screen.

---

## FR-UI-006 — Current Machine Visibility

**Priority:** MUST

The current machine must remain clearly visible during troubleshooting.

---

## FR-UI-007 — Current Session Status

**Priority:** MUST

The diagnostic status must be visible to the technician.

---

## FR-UI-008 — Pending Action

**Priority:** MUST

When the graph is waiting for technician input, the pending action must be visually prominent.

---

## FR-UI-009 — Evidence Panel

**Priority:** MUST

The Troubleshooting interface must include a visible evidence area.

---

## FR-UI-010 — Evidence Type Distinction

**Priority:** MUST

The UI must visually distinguish:

- manufacturer documentation,
- technician measurements,
- historical cases,
- AI inference.

---

## FR-UI-011 — Root Cause Distinction

**Priority:** MUST

The UI must clearly distinguish probable root cause from confirmed root cause.

---

## FR-UI-012 — Voice Input

**Priority:** MUST

The Troubleshooting interface must expose voice recording.

---

## FR-UI-013 — Text Input

**Priority:** MUST

Text input must remain available alongside voice.

---

## FR-UI-014 — Structured Measurement Input

**Priority:** MUST

When measurements are requested, the UI should render suitable structured fields.

---

## FR-UI-015 — Arabic-Friendly Interface

**Priority:** MUST

The interface must correctly support Arabic content and RTL rendering where applicable.

---

## FR-UI-016 — English Technical Terms

**Priority:** MUST

The UI must allow English technical terminology to appear naturally within Arabic content.

---

## FR-UI-017 — Responsive Web UI

**Priority:** MUST

MVP v0.1 must provide a responsive web interface suitable for:

- desktop,
- laptop,
- tablet.

Mobile-sized responsive support is desirable when practical.

A native mobile application is not required.

---

## FR-UI-018 — Avoid Information Overload

**Priority:** MUST

The troubleshooting screen should prioritize:

```text
Next action
Evidence
Diagnostic context
```

over long generated explanations.

---

# 25. Error Handling Requirements

## FR-ERR-001 — LLM Failure

**Priority:** MUST

If an LLM call fails, the session must remain recoverable.

---

## FR-ERR-002 — Retrieval Failure

**Priority:** MUST

A retrieval failure must not delete or corrupt the Diagnostic Session.

---

## FR-ERR-003 — ASR Failure

**Priority:** MUST

If ASR fails, the user must be able to:

- retry,
- record again,
- use text input.

---

## FR-ERR-004 — Tool Failure

**Priority:** MUST

Tool exceptions must be caught and converted into controlled application errors.

---

## FR-ERR-005 — User-Friendly Error

**Priority:** MUST

The user must not receive raw stack traces or internal provider exceptions.

---

## FR-ERR-006 — Retry

**Priority:** MUST

Recoverable failures should provide an explicit retry path.

---

## FR-ERR-007 — Preserve State on Failure

**Priority:** MUST

If an AI or tool operation fails after session creation, the last valid diagnostic state must remain available.

---

## FR-ERR-008 — Document Processing Failure

**Priority:** MUST

Failed document processing must expose:

- failure status,
- useful failure reason,
- retry or replacement path.

---

# 26. Persistence and Recovery Requirements

## FR-PERSIST-001 — Durable Business Data

**Priority:** MUST

Core business data must persist independently of transient LLM state.

---

## FR-PERSIST-002 — LangGraph Checkpoints

**Priority:** MUST

Active diagnostic workflows must use durable LangGraph checkpoints appropriate for MVP deployment.

---

## FR-PERSIST-003 — Application Database Is Authoritative

**Priority:** MUST

LangGraph checkpoints must not replace the main application database as the authoritative source of business records.

---

## FR-PERSIST-004 — Recovery After Browser Refresh

**Priority:** MUST

Refreshing the browser must not destroy an active Diagnostic Session.

---

## FR-PERSIST-005 — Recovery After Temporary Backend/Provider Error

**Priority:** MUST

Once the service is restored, an interrupted valid session should be resumable where practical.

---

# 27. Audit and Diagnostic Timeline Requirements

## FR-AUDIT-001 — Diagnostic Event Timeline

**Priority:** MUST

The system must maintain an auditable timeline of meaningful diagnostic events.

---

## FR-AUDIT-002 — Event Types

The timeline should support events such as:

```text
SESSION_CREATED
FAULT_SUBMITTED
FAULT_STRUCTURED
EVIDENCE_RETRIEVED
HYPOTHESES_UPDATED
ACTION_REQUESTED
OBSERVATION_SUBMITTED
MEASUREMENT_SUBMITTED
SAFETY_CHECKED
PROBABLE_CAUSE_CREATED
RESOLUTION_CONFIRMED
SESSION_RESOLVED
```

---

## FR-AUDIT-003 — Event Timestamp

**Priority:** MUST

Every diagnostic event must have a timestamp.

---

## FR-AUDIT-004 — Event Session Association

**Priority:** MUST

Every diagnostic timeline event must belong to one Diagnostic Session.

---

## FR-AUDIT-005 — Research Trace

**Priority:** MUST

The stored trace must provide enough information to later analyze:

- diagnostic path,
- number of steps,
- evidence used,
- technician interactions,
- final result.

---

# 28. AI Provider Abstraction Requirements

## FR-PROVIDER-001 — Provider Abstraction

**Priority:** MUST

The architecture must avoid hard-coding business logic directly to a single LLM provider.

---

## FR-PROVIDER-002 — Provider Interface

**Priority:** MUST

Core AI operations should use provider abstractions that allow changing the underlying model with limited application impact.

---

## FR-PROVIDER-003 — Structured Output Contract

**Priority:** MUST

Changing provider must not change the internal validated schema expected by the application.

---

## FR-PROVIDER-004 — Provider-Specific Errors

**Priority:** MUST

Provider-specific exceptions must be mapped into application-level errors.

---

## FR-PROVIDER-005 — Model Configuration

**Priority:** SHOULD

Model selection should be configurable rather than scattered through application code.

---

# 29. Parts Compatibility Lite Requirements

This domain is SHOULD HAVE and secondary to the core diagnostic MVP.

## FR-PART-001 — View Known Part

**Priority:** SHOULD

The system may support basic machine part records.

---

## FR-PART-002 — Technical Constraints

**Priority:** SHOULD

Replacement compatibility must be based on explicit technical constraints where available.

Examples:

```text
voltage
current
contact configuration
mounting
dimensions
sensor type
signal type
```

---

## FR-PART-003 — Deterministic Compatibility

**Priority:** SHOULD

LLM reasoning alone must not determine that two parts are compatible.

---

## FR-PART-004 — Compatibility Result

**Priority:** SHOULD

Possible result categories may include:

```text
COMPATIBLE
INCOMPATIBLE
REQUIRES_REVIEW
INSUFFICIENT_DATA
```

---

# 30. Minimal Administration Requirements

## FR-ADMIN-001 — Add Demo Machine

**Priority:** MUST

The MVP must provide a practical way to seed or create the test machine.

This may initially be through:

- admin endpoint,
- seed script,
- minimal admin interface.

A full administration portal is not required.

---

## FR-ADMIN-002 — Upload Documents to Machine

**Priority:** MUST

An authorized project/admin workflow must exist for associating documents with the demo machine.

---

## FR-ADMIN-003 — Create Test Maintenance Cases

**Priority:** SHOULD

The development/evaluation environment should support creating controlled historical maintenance cases for benchmark scenarios.

---

# 31. Performance Requirements

These are functional performance expectations for the MVP, not enterprise-scale SLAs.

## FR-PERF-001 — Interactive Diagnostic Experience

**Priority:** MUST

Normal text-based diagnostic interactions should return progress or a result without leaving the user with an unexplained frozen interface.

---

## FR-PERF-002 — Long Operation Feedback

**Priority:** MUST

For operations such as:

- ASR,
- document processing,
- retrieval,
- LLM reasoning,

the UI must display an appropriate processing state.

---

## FR-PERF-003 — Document Processing Asynchronous Behavior

**Priority:** SHOULD

Long document ingestion work should be capable of running asynchronously rather than blocking the upload request until all indexing completes.

---

## FR-PERF-004 — No Duplicate Submissions

**Priority:** MUST

The system should prevent or safely handle accidental duplicate diagnostic submissions caused by repeated user clicks or network retries.

---

# 32. Evaluation Support Requirements

## FR-EVAL-001 — Controlled Scenario Support

**Priority:** MUST

The MVP must support running controlled diagnostic scenarios for evaluation.

---

## FR-EVAL-002 — Diagnostic Trace Exportability

**Priority:** MUST

Evaluation code or researchers must be able to access diagnostic traces programmatically.

---

## FR-EVAL-003 — Baseline Comparison

**Priority:** MUST

The implementation must permit evaluation of at least:

```text
General LLM
Basic RAG
SALLIH
```

without requiring three completely independent applications.

---

## FR-EVAL-004 — Root Cause Outcome

**Priority:** MUST

Evaluation data must make it possible to compare:

```text
expected_root_cause
system_probable_root_cause
```

---

## FR-EVAL-005 — Diagnostic Step Evaluation

**Priority:** MUST

Evaluation data must make it possible to inspect the sequence of diagnostic checks requested by the system.

---

## FR-EVAL-006 — Citation Evaluation

**Priority:** MUST

Evaluation data must allow verifying whether cited evidence actually supports generated technical claims.

---

## FR-EVAL-007 — ASR Evaluation

**Priority:** MUST

The project must support comparing ASR transcripts against ground-truth transcripts.

---

# 33. Security and Data Boundary Requirements

Detailed security design is outside this document, but the MVP requires basic functional boundaries.

## FR-SEC-001 — Authentication Required

**Priority:** MUST

Private machine and maintenance information must require authenticated access.

---

## FR-SEC-002 — Machine Context Validation

**Priority:** MUST

The backend must validate machine association rather than trusting only frontend-provided UI state.

---

## FR-SEC-003 — Uploaded File Validation

**Priority:** MUST

Uploaded documents must undergo basic validation before processing.

---

## FR-SEC-004 — No Direct Provider Secrets in Frontend

**Priority:** MUST

LLM, ASR, embedding, and other private provider credentials must not be exposed in frontend code.

---

# 34. Application State vs AI State

The implementation must distinguish application state from AI reasoning state.

## Application State

Examples:

```text
machine_id
session_id
user_id
document_id
session_status
confirmed_root_cause
corrective_action
```

These values belong to the application/domain layer.

---

## AI Diagnostic State

Examples:

```text
structured_fault
hypotheses
retrieved_evidence
next_action
probable_root_cause
```

These values may be managed through LangGraph but must remain structured and persistable where required.

---

## FR-STATE-001 — Explicit State Contract

**Priority:** MUST

The boundary between application state and LangGraph state must be explicitly defined in `06-ai-system-design.md` and `09-system-architecture.md`.

---

# 35. Deterministic vs Agentic Requirements

## FR-DET-001 — Deterministic Validation

**Priority:** MUST

The following must use deterministic validation rather than free-form LLM judgment where practical:

- required fields,
- numeric values,
- enums,
- machine ownership,
- document status,
- session status,
- technical value formats,
- API contracts.

---

## FR-DET-002 — Safety Cannot Depend Solely on Free LLM Output

**Priority:** MUST

Safety enforcement must include deterministic rules or validation layers.

---

## FR-DET-003 — Parts Compatibility Cannot Depend Solely on LLM Output

**Priority:** MUST

If Parts Compatibility Lite is implemented, hard technical constraints must be evaluated deterministically.

---

## FR-DET-004 — Agentic Decisions

**Priority:** MUST

The LLM/agent may support flexible decisions such as:

- interpreting natural-language faults,
- generating hypotheses,
- deciding what evidence is needed,
- choosing the next useful diagnostic question,
- explaining evidence.

---

# 36. Unsupported Functionality

The following must not be added to MVP v0.1 unless the canonical documentation is formally revised:

```text
Full CMMS
Work orders
Preventive maintenance scheduling
Predictive maintenance
Live industrial telemetry platform
PLC write/control
SCADA replacement
Machine autonomous control
Procurement workflow
Supplier marketplace
Warehouse inventory
Full enterprise RBAC
SSO
Native mobile app
Multi-factory administration
Large multi-agent system
Generic universal machine diagnosis
```

---

# 37. Core End-to-End Functional Contract

The system must support the following complete interaction:

```text
Technician signs in
↓
Technician selects or scans machine
↓
Machine Passport opens
↓
Technician starts diagnosis
↓
Persistent Diagnostic Session created
↓
Technician speaks or types fault
↓
If voice:
ASR → transcript → technician confirmation
↓
Fault converted to validated structured state
↓
Machine-specific evidence retrieved
↓
Diagnostic hypotheses generated
↓
Next useful diagnostic action selected
↓
Safety layer evaluates action
↓
Technician receives request
↓
LangGraph interrupts
↓
Technician performs physical check
↓
Technician submits observation or measurement
↓
Backend validates input
↓
LangGraph resumes
↓
Hypotheses update
↓
Loop repeats as needed
↓
Sufficient evidence reached
↓
Probable Root Cause presented
↓
Supporting evidence displayed
↓
Technician performs appropriate repair
↓
Technician confirms actual root cause
↓
Technician records corrective action
↓
Session becomes RESOLVED
↓
Maintenance Case created
↓
Case becomes searchable Machine Memory
```

---

# 38. MVP Functional Acceptance Checklist

MVP v0.1 must not be considered functionally complete until all MUST items below work end-to-end.

## Authentication

- [ ] Technician can sign in.
- [ ] Protected routes require authentication.
- [ ] QR login redirect preserves target machine.

## Machines

- [ ] Machine list loads.
- [ ] Machine Passport opens.
- [ ] Machine metadata is visible.
- [ ] QR opens the correct machine.

## Documents

- [ ] PDF can be uploaded.
- [ ] Document can be processed.
- [ ] READY/FAILED state is visible.
- [ ] Document chunks preserve page metadata.
- [ ] Failed document is excluded from retrieval.

## RAG

- [ ] Retrieval is machine-scoped.
- [ ] Relevant manual content can be retrieved.
- [ ] Alarm lookup works.
- [ ] Sources can be displayed.
- [ ] Missing evidence is handled without fabrication.

## Diagnosis

- [ ] New Diagnostic Session can be created.
- [ ] Original complaint is preserved.
- [ ] Fault becomes validated structured state.
- [ ] Hypotheses exist as structured data.
- [ ] Evidence is associated with hypotheses where appropriate.
- [ ] Agent chooses a next useful action.
- [ ] Safety check executes before action display.

## LangGraph

- [ ] Diagnostic state exists in the graph.
- [ ] Graph can interrupt.
- [ ] Checkpoint is persisted.
- [ ] Graph can resume.
- [ ] Diagnostic loop works.
- [ ] Graph has bounded termination behavior.

## Technician Input

- [ ] Text observation can be submitted.
- [ ] Structured numeric measurement can be submitted.
- [ ] Invalid measurements are rejected.
- [ ] Technician input updates diagnostic state.

## Voice

- [ ] Voice can be recorded.
- [ ] Audio can be transcribed.
- [ ] Transcript is visible.
- [ ] Transcript can be edited.
- [ ] Transcript requires confirmation.
- [ ] Text fallback exists.

## Root Cause

- [ ] Probable root cause can be produced.
- [ ] Probable root cause has evidence.
- [ ] Insufficient-evidence state is supported.
- [ ] Probable and confirmed root causes are distinct.

## Resolution

- [ ] Technician can resolve session.
- [ ] Confirmed root cause is captured.
- [ ] Corrective action is captured.
- [ ] Session becomes RESOLVED.

## Maintenance Memory

- [ ] Resolved session creates Maintenance Case.
- [ ] Maintenance Case appears in history.
- [ ] Case details can be opened.
- [ ] Previous case can be retrieved in a future diagnosis.
- [ ] Previous case is treated as evidence, not automatic diagnosis.

## Safety

- [ ] Physical actions receive safety classification.
- [ ] High-risk actions receive safety protection.
- [ ] Safety bypass guidance is prohibited.
- [ ] System cannot autonomously control machine.

## Recovery

- [ ] Browser refresh does not destroy active session.
- [ ] Provider error does not destroy valid state.
- [ ] Retry path exists for recoverable failures.

---

# 39. Traceability to User Flows

Functional requirements should map back to user flows.

Example:

| User Flow | Core Requirements |
|---|---|
| UF-01 Sign In | FR-AUTH-* |
| UF-02 Browse Machines | FR-MACH-001 |
| UF-03 Machine Passport | FR-MACH-002 to FR-MACH-006 |
| UF-04 QR | FR-QR-* |
| UF-05 Start Diagnostic Session | FR-DIAG-001 to FR-DIAG-005 |
| UF-06 Text Fault | FR-FAULT-* |
| UF-07 Voice Fault | FR-VOICE-* |
| UF-08 Transcript Review | FR-ASR-004 to FR-ASR-006 |
| UF-09 Structured Fault | FR-FAULT-004 to FR-FAULT-008 |
| UF-10 Retrieve Evidence | FR-RAG-* |
| UF-11 Guided Troubleshooting | FR-GRAPH-*, FR-DIAG-012 to FR-DIAG-017 |
| UF-12 Observation | FR-OBS-* |
| UF-13 Measurement | FR-MEAS-* |
| UF-14 Resume Session | FR-GRAPH-005 to FR-GRAPH-008, FR-PERSIST-* |
| UF-15 Safety | FR-SAFE-* |
| UF-16 Probable Root Cause | FR-ROOT-* |
| UF-17 Confirm Resolution | FR-RES-* |
| UF-18 Save Memory | FR-MEM-* |
| UF-19 Retrieve Memory | FR-MEM-005 to FR-MEM-006 |
| UF-20 History | FR-MEM-007 |
| UF-21 Case Details | FR-MEM-008 |
| UF-22 Document Upload | FR-DOC-* |
| UF-23 Document Failure | FR-DOC-009, FR-ERR-008 |
| UF-24 Insufficient Evidence | FR-DIAG-017, FR-ROOT-004 |
| UF-25 AI Failure Recovery | FR-ERR-*, FR-PERSIST-* |

---

# 40. Implementation Priority

If development constraints require tradeoffs, implementation must protect the following order:

```text
1. Persistent Diagnostic Session
2. Machine-scoped RAG
3. Structured fault understanding
4. LangGraph diagnostic loop
5. Human-in-the-loop interrupt/resume
6. Evidence and citations
7. Hypothesis management
8. Safety layer
9. Confirmed resolution
10. Maintenance Memory
11. Voice / ASR
12. ASR evaluation
13. Parts Compatibility Lite
14. Optional analytics / polish
```

Voice remains a MUST HAVE for the final MVP, but the diagnostic engine should be proven through text before voice complexity is allowed to block progress.

---

# 41. Functional Design Principles

All implementation decisions should preserve the following principles.

## Principle 1

> Machine context before model knowledge.

## Principle 2

> Evidence before confident claims.

## Principle 3

> Diagnosis is iterative, not one-shot.

## Principle 4

> Technician confirmation defines real maintenance history.

## Principle 5

> Previous maintenance cases are evidence, not automatic truth.

## Principle 6

> Agentic where flexibility helps; deterministic where reliability matters.

## Principle 7

> Failure to diagnose is safer than fabricated certainty.

## Principle 8

> LangGraph represents real workflow state.

## Principle 9

> Business data remains explicit and persistent.

## Principle 10

> The AI assists the technician; it does not control the machine.

---

# 42. Final Functional Requirement Statement

SALLIH MVP v0.1 must function as:

> **A persistent, machine-specific, Arabic-first diagnostic copilot that understands a technician's fault description, retrieves evidence from that machine, iteratively guides physical troubleshooting through LangGraph and human-in-the-loop interaction, safely proposes a probable root cause, and converts the technician-confirmed resolution into reusable Maintenance Memory.**

The MVP should not be considered complete if it only provides:

```text
User Question
↓
RAG Search
↓
LLM Answer
```

The defining functional behavior is:

```text
Fault
↓
Structured Understanding
↓
Evidence
↓
Hypotheses
↓
Next Diagnostic Check
↓
Human Evidence
↓
Updated Hypotheses
↓
Probable Root Cause
↓
Human Confirmation
↓
Persistent Machine Memory
```