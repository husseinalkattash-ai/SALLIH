# SALLIH — Product Vision

**Document Version:** 1.0  
**Project Stage:** MVP v0.1  
**Status:** Draft for implementation baseline

---

# 1. Product Name

**SALLIH**

Arabic brand name:

**صلّح**

Working product description:

> **Arabic-First AI Maintenance Copilot for Industrial Machinery**

SALLIH is an AI-assisted industrial maintenance platform designed primarily for Arabic-speaking maintenance technicians and engineers.

It supports both legacy and modern industrial machinery.

The system combines machine-specific documentation, maintenance history, voice interaction, guided troubleshooting, and AI reasoning to help technicians diagnose faults more efficiently while preserving technical knowledge inside the factory.

---

# 2. Product Vision

Industrial maintenance knowledge is often fragmented across:

- machine manuals,
- electrical schematics,
- parts catalogs,
- alarm documentation,
- maintenance notebooks,
- previous technicians,
- service reports,
- and individual experience.

A technician may therefore spend significant time searching documentation, interpreting foreign-language manuals, asking experienced colleagues, or repeating diagnostic work that was already performed in the past.

SALLIH aims to transform this fragmented information into a persistent **Machine Intelligence Layer**.

Each machine should gradually develop its own digital knowledge and maintenance memory.

The long-term vision is:

> **Every industrial machine should have an intelligent technical memory that understands its documentation, history, faults, and maintenance context — and can communicate naturally with the technician.**

---

# 3. Core Product Principle

SALLIH is not a generic industrial chatbot.

It is:

> **Machine-specific, evidence-grounded, human-guided maintenance intelligence.**

The system should reason from the context of the specific machine being serviced.

Relevant context may include:

- manufacturer documentation,
- manuals,
- schematics,
- alarm codes,
- parts information,
- maintenance history,
- technician observations,
- measurements,
- and eventually PLC / CNC / IoT data.

The AI must not present unsupported assumptions as facts.

Important diagnostic recommendations should be traceable to evidence whenever possible.

---

# 4. Primary User

The primary user for MVP v0.1 is:

## Maintenance Technician / Maintenance Engineer

Typical characteristics:

- works directly with industrial machines,
- may communicate primarily in Arabic,
- frequently uses mixed Arabic-English technical terminology,
- may work with machines whose documentation is written in English or another foreign language,
- needs quick access to technical information while physically working near the machine,
- may rely heavily on personal experience or the experience of senior technicians.

Example technician language:

> "الدرايف عم يفصل لما يحمل الموتور."

> "الكونتاكتور ما عم يلقط."

> "الماكينة عم تعطي overcurrent بعد عشر دقايق."

SALLIH should understand this communication style while mapping it to the correct technical concepts.

---

# 5. Core Problem

Industrial maintenance teams may face several recurring information and knowledge problems.

## 5.1 Fragmented Machine Knowledge

Information about a machine may be distributed across multiple documents and people.

Examples:

- PDF manuals,
- scanned documents,
- electrical diagrams,
- spare-parts catalogs,
- maintenance reports,
- handwritten notes,
- experienced technicians.

The technician must manually search and combine this information.

---

## 5.2 Foreign-Language Documentation

Industrial equipment may originate from manufacturers in countries such as:

- Germany,
- Italy,
- China,
- Japan,
- or other international markets.

Documentation is often not available in Arabic.

This creates an additional barrier for Arabic-speaking technicians.

---

## 5.3 Knowledge Loss

A significant portion of maintenance knowledge may exist only in the experience of individual technicians.

When an experienced technician leaves the company, part of that knowledge can disappear.

Previous faults may therefore be diagnosed repeatedly from the beginning.

---

## 5.4 Slow Troubleshooting

When a machine fails, the technician may need to:

1. identify the affected subsystem,
2. locate the correct manual,
3. search the manual,
4. interpret the alarm,
5. inspect schematics,
6. test multiple hypotheses,
7. ask another technician,
8. perform measurements,
9. identify the root cause.

This process may be slow and heavily dependent on individual expertise.

---

## 5.5 Poorly Structured Maintenance History

Previous maintenance cases may not be stored in a form that can easily support future troubleshooting.

Important information such as:

- symptoms,
- measurements,
- root cause,
- corrective action,
- replaced part,
- and downtime

may not be consistently recorded.

---

# 6. Proposed Solution

SALLIH gives every machine a digital profile called a:

## Machine Passport

The Machine Passport contains machine-specific information such as:

- manufacturer,
- model,
- serial number,
- year,
- factory location,
- manuals,
- technical documentation,
- schematics,
- parts information,
- and maintenance history.

Technicians interact with SALLIH through text or voice.

When a fault occurs, SALLIH starts a guided diagnostic session.

Example:

```text
Technician
    ↓
Voice / Text Fault Description
    ↓
SALLIH understands the symptom
    ↓
Loads the machine context
    ↓
Searches documentation and history
    ↓
Builds diagnostic hypotheses
    ↓
Requests the next useful inspection or measurement
    ↓
Technician provides evidence
    ↓
Hypotheses are updated
    ↓
Probable root cause is identified
    ↓
Technician confirms the real resolution
    ↓
Case becomes part of Machine Memory
```

---

# 7. Core MVP Value Proposition

For MVP v0.1, SALLIH should prove one main value proposition:

> **A technician can describe a machine problem naturally in Arabic and receive evidence-grounded, step-by-step diagnostic assistance based on that machine's own documentation and maintenance history.**

The objective is not to autonomously repair equipment.

The objective is to help a qualified technician reach the correct diagnosis faster and preserve the resulting knowledge.

---

# 8. Core MVP Capabilities

MVP v0.1 will focus on six capabilities.

## 8.1 Machine Passport

A structured digital identity for one industrial machine.

Contains:

- machine metadata,
- uploaded documentation,
- technical files,
- components,
- and maintenance cases.

---

## 8.2 Arabic Text and Voice Interaction

The technician can describe faults using natural Arabic.

The MVP should support:

- typed Arabic,
- Syrian Arabic speech,
- mixed Arabic-English industrial terminology.

Speech will be converted to text through an ASR layer.

---

## 8.3 Machine-Grounded AI Copilot

The AI should retrieve information from the selected machine's documentation rather than relying only on generic model knowledge.

Important answers should include evidence such as:

- document name,
- page,
- section,
- or previous maintenance case.

---

## 8.4 Guided Troubleshooting

SALLIH should conduct an iterative troubleshooting session instead of producing a single final answer.

The system should:

1. understand the initial fault,
2. generate possible hypotheses,
3. identify missing information,
4. request useful checks or measurements,
5. update hypotheses,
6. narrow the diagnostic path,
7. present a probable root cause.

---

## 8.5 Maintenance Memory

Resolved diagnostic sessions become structured maintenance cases.

Each case may contain:

- original complaint,
- symptoms,
- alarms,
- measurements,
- diagnostic steps,
- confirmed root cause,
- corrective action,
- parts replaced,
- downtime,
- technician confirmation.

Future sessions may retrieve similar previous cases.

---

## 8.6 Parts Compatibility Lite

The MVP may contain a small controlled dataset of replacement components.

Compatibility must not be decided solely by the LLM.

A deterministic compatibility layer should validate predefined technical constraints such as:

- voltage,
- current,
- contact configuration,
- dimensions where available,
- mounting requirements,
- or other required specifications.

This capability is secondary to troubleshooting and may be reduced if MVP scope becomes too large.

---

# 9. Agentic AI Role

SALLIH will use an agentic architecture because troubleshooting is an iterative, stateful process.

The main MVP agent is:

## Diagnostic Orchestrator Agent

The agent is responsible for deciding:

- what information is currently known,
- what information is missing,
- which tool should be called,
- what evidence should be retrieved,
- what question should be asked next,
- whether enough evidence exists to narrow the diagnosis,
- and when human confirmation is required.

The agent may use tools such as:

- Machine Documentation Retrieval,
- Alarm Lookup,
- Maintenance Memory Search,
- Measurement Collection,
- Parts Lookup,
- Safety Rules.

The agent does **not** control the machine.

The agent does **not** autonomously execute repairs.

---

# 10. AI Architecture Principle

SALLIH follows the principle:

> **Agentic where flexibility helps. Deterministic where reliability matters.**

LLMs may be used for:

- understanding technician language,
- extracting structured fault information,
- generating hypotheses,
- selecting the next diagnostic question,
- summarizing evidence,
- explaining technical findings.

Deterministic logic should be preferred for:

- safety rules,
- numerical comparisons,
- compatibility constraints,
- required fields,
- validation,
- permissions,
- and critical system invariants.

---

# 11. Voice Vision

Voice is a first-class interaction mode for SALLIH.

Maintenance technicians may:

- wear gloves,
- work with tools,
- stand next to machinery,
- operate in noisy environments,
- and prefer speaking over typing.

The voice pipeline is:

```text
Technician Speech
        ↓
ASR
        ↓
Arabic Transcript
        ↓
Technical Language Understanding
        ↓
Diagnostic Agent
```

The initial ASR baseline is:

**Whisper Large v3 Turbo**

The project will evaluate adaptation for:

- Syrian Arabic,
- industrial terminology,
- Arabic-English code switching,
- alarm codes,
- manufacturer names,
- part numbers,
- and factory noise.

Fine-tuning should be performed only when supported by measurable baseline weaknesses.

---

# 12. Technology Direction

Current planned technical direction:

### Frontend

Web application optimized for:

- desktop,
- tablet,
- and technician-friendly responsive use.

---

### Backend

**FastAPI**

Responsible for:

- application APIs,
- authentication,
- machine data,
- documents,
- diagnostic sessions,
- maintenance cases,
- AI orchestration integration.

---

### Agent Framework

**LangGraph**

Used for:

- diagnostic workflow orchestration,
- state management,
- iterative troubleshooting,
- human-in-the-loop transitions,
- persistence,
- future subgraphs.

---

### AI Building Blocks

**LangChain**

Used where appropriate for:

- model abstraction,
- structured outputs,
- tools,
- retrievers,
- document processing,
- RAG integrations.

---

### Data

**PostgreSQL**

For structured application and maintenance data.

Vector search may be implemented through PostgreSQL/pgvector or another explicitly justified vector layer.

---

### Speech

**Whisper Large v3 Turbo** as the initial ASR baseline.

---

# 13. Human-in-the-Loop Principle

SALLIH is a maintenance copilot, not an autonomous maintenance system.

A qualified technician remains responsible for:

- physical inspection,
- measurements,
- repair decisions,
- electrical work,
- component replacement,
- final confirmation of the root cause.

The system should explicitly expose uncertainty.

Examples:

- Low confidence
- More evidence required
- Expert review required
- Unable to determine safely

SALLIH should prefer admitting uncertainty over inventing a solution.

---

# 14. Safety Principle

The system must never encourage unsafe actions merely to continue a diagnostic workflow.

High-risk operations may include:

- high-voltage work,
- safety-circuit bypass,
- rewiring,
- machine safety interlock modification,
- pressurized systems,
- dangerous rotating equipment,
- or other hazardous procedures.

Such actions must require appropriate warnings and qualified human verification.

The system must never instruct the user to bypass a safety mechanism.

---

# 15. MVP v0.1 Scope

MVP v0.1 will demonstrate SALLIH using:

> **One industrial machine or realistic machine test environment.**

The MVP must provide one polished end-to-end workflow:

```text
Identify Machine
        ↓
Open Machine Passport
        ↓
Describe Fault via Voice or Text
        ↓
Understand Fault
        ↓
Retrieve Machine Evidence
        ↓
Begin Guided Troubleshooting
        ↓
Collect Measurements / Answers
        ↓
Update Diagnostic Hypotheses
        ↓
Present Probable Root Cause
        ↓
Show Supporting Evidence
        ↓
Technician Confirms Resolution
        ↓
Save Maintenance Case
        ↓
Future Sessions Can Reuse It
```

---

# 16. Explicit MVP Non-Goals

The following are intentionally outside MVP v0.1:

- full CMMS replacement,
- work-order management,
- preventive maintenance scheduling,
- predictive maintenance,
- complete PLC integration,
- broad IoT analytics,
- autonomous machine control,
- procurement workflows,
- supplier marketplace,
- inventory management,
- native mobile applications,
- support for many machine families,
- fully autonomous repair,
- enterprise multi-factory management.

These may become future roadmap items only after real factory validation.

---

# 17. MVP Success Definition

MVP v0.1 is considered successful when a technician can:

1. access the test machine,
2. describe a fault in Arabic,
3. receive a correctly interpreted transcript,
4. start a diagnostic session,
5. receive machine-specific evidence,
6. provide requested checks or measurements,
7. see hypotheses narrowed over multiple steps,
8. receive a plausible root-cause assessment,
9. inspect supporting evidence,
10. confirm the actual resolution,
11. save the case,
12. later retrieve that case as maintenance memory.

The MVP should demonstrate that SALLIH is more than a chatbot.

---

# 18. Validation Strategy After MVP

Once MVP v0.1 is stable, SALLIH will be presented to real industrial companies.

The preferred engagement model is a:

## Design Partner

The partner provides access to a limited pilot scope such as:

- one or more machines,
- manuals,
- schematics,
- historical fault cases,
- maintenance technicians,
- and structured feedback.

SALLIH is then adapted and evaluated using real factory data.

MVP v0.2 should be driven by validated factory requirements rather than assumptions made before customer contact.

---

# 19. Long-Term Vision

SALLIH may eventually evolve into an industrial intelligence platform supporting:

- legacy machinery,
- modern PLC-connected machinery,
- live machine telemetry,
- preventive maintenance,
- predictive maintenance,
- spare-part intelligence,
- industrial knowledge management,
- multilingual technical support,
- multi-factory operations,
- and localized industrial AI across the MENA region.

However, these ambitions must not expand the scope of MVP v0.1.

---

# 20. Product Statement

> **SALLIH gives every machine a technical memory and gives every technician an intelligent, Arabic-first maintenance copilot grounded in the machine's own evidence.**