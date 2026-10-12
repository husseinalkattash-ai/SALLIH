# SALLIH — MVP Scope

**Document Version:** 1.0  
**Project Stage:** MVP v0.1  
**Status:** Implementation Baseline

---

# 1. Purpose of This Document

This document defines the exact scope of SALLIH MVP v0.1.

Its purpose is to prevent:

- uncontrolled feature expansion,
- premature enterprise functionality,
- unnecessary multi-agent complexity,
- premature PLC/IoT integration,
- premature predictive maintenance,
- building features that do not contribute to the core experiment.

This document is a scope boundary.

Any feature not explicitly classified as:

- MUST HAVE,
- SHOULD HAVE,
- OPTIONAL,

must be treated as:

> OUT OF SCOPE for MVP v0.1.

---

# 2. MVP Goal

The MVP must prove the following hypothesis:

> **An Arabic-speaking industrial maintenance technician can describe a fault naturally, and SALLIH can use machine-specific documentation, previous maintenance knowledge, and iterative diagnostic reasoning to guide the technician toward a probable root cause with traceable evidence.**

The MVP is not intended to prove that SALLIH can support an entire factory.

It is intended to prove the core diagnostic experience.

---

# 3. MVP Demonstration Environment

MVP v0.1 will initially support:

> **One industrial machine or one realistic industrial machine test environment.**

The selected machine should preferably contain several common industrial components such as:

- electric motor,
- VFD / drive,
- sensors,
- relays,
- contactors,
- basic PLC/control system,
- safety components,
- mechanical load.

Preferred machine category:

- packaging machine,
- conveyor-based system,
- or similar production equipment.

The selected machine should have enough documentation to support meaningful troubleshooting.

---

# 4. Required Machine Data

Before the core diagnostic MVP can be considered complete, the selected test machine should have some or all of the following:

## Required

- machine name,
- manufacturer,
- model,
- serial or internal identifier,
- machine location,
- at least one technical manual,
- alarm/fault documentation where available.

## Strongly Preferred

- electrical schematic,
- component documentation,
- parts list,
- previous maintenance cases,
- technical datasheets.

## Optional

- PLC documentation,
- historical sensor data,
- live telemetry.

Live machine connectivity is NOT required for MVP v0.1.

---

# 5. Core MVP User Flow

The official MVP flow is:

```text
Identify Machine
      ↓
Open Machine Passport
      ↓
Start Diagnostic Session
      ↓
Describe Fault by Voice or Text
      ↓
Interpret Fault Description
      ↓
Retrieve Machine-Specific Evidence
      ↓
Generate Diagnostic Hypotheses
      ↓
Select Next Useful Check
      ↓
Ask Technician
      ↓
Receive Observation / Measurement
      ↓
Update Diagnostic State
      ↓
Repeat if necessary
      ↓
Present Probable Root Cause
      ↓
Show Supporting Evidence
      ↓
Technician Confirms Resolution
      ↓
Save Maintenance Case
      ↓
Case Becomes Machine Memory
```

Every MUST HAVE feature must support this flow.

---

# 6. MUST HAVE Features

The following features are required for MVP v0.1.

---

## 6.1 Machine Passport

The system must provide a dedicated digital identity for the test machine.

Minimum fields:

```text
Machine ID
Name
Manufacturer
Model
Serial Number
Year
Location
Description
Status
```

The Machine Passport must provide access to:

- machine documentation,
- maintenance history,
- diagnostic sessions.

---

## 6.2 Machine Identification

The technician must be able to identify the machine before starting troubleshooting.

MVP-supported methods:

### Required

- selecting the machine from the Machines interface.

### Required for final demo

- QR code linking directly to the Machine Passport.

Example:

```text
Scan QR
   ↓
Open Packaging Line #01
```

No computer vision-based machine identification is required.

---

# 7. Document Knowledge Base

The system must support machine-specific technical documentation.

Supported MVP file type:

- PDF

Additional document formats may be supported if implementation effort is low.

The system must support:

```text
Upload
↓
Parse
↓
Chunk
↓
Index
↓
Retrieve
```

Each retrievable chunk must preserve metadata.

Minimum metadata:

```text
machine_id
document_id
document_name
document_type
page_number
section where available
```

---

# 8. OCR Requirement

OCR should be supported when technically feasible for scanned documents.

However:

> OCR quality is not the primary research objective of MVP v0.1.

If high-quality digital manuals are available for the demo machine, they may be used initially.

Scanned document support should not block the entire MVP.

---

# 9. Machine-Specific Retrieval

Retrieval must always be scoped to the currently selected machine.

The system must NOT accidentally retrieve technical information from another machine.

Example:

```text
machine_id = MACHINE_001
```

must constrain retrieval to:

```text
MACHINE_001 documents
+
MACHINE_001 maintenance memory
```

Cross-machine retrieval is outside MVP scope unless explicitly requested by the user in a future release.

---

# 10. Evidence-Grounded Responses

Important technical recommendations must include supporting evidence whenever available.

Possible evidence types:

```text
Manufacturer Manual
Electrical Schematic
Alarm Documentation
Maintenance Case
Technician Measurement
```

Example:

```text
Possible cause:
Motor overload

Evidence:
Drive Manual
Page 74
Section 8.3 — Overcurrent Protection
```

The backend must preserve evidence independently from generated explanation text.

---

# 11. Diagnostic Session

The system must support a persistent Diagnostic Session.

Each session must belong to:

- one machine,
- one user/technician,
- one fault investigation.

A session must preserve at least:

```text
session_id
machine_id
initial_fault_description
structured_fault
current_status
hypotheses
observations
measurements
diagnostic_steps
retrieved_evidence
probable_root_cause
final_root_cause
resolution
created_at
resolved_at
```

---

# 12. Diagnostic Orchestrator Agent

The primary agent in MVP v0.1 is:

> **SALLIH Diagnostic Orchestrator**

The agent is responsible for managing the diagnostic process.

It must be able to:

1. understand the fault description,
2. identify missing information,
3. retrieve relevant evidence,
4. create or update hypotheses,
5. determine the next useful diagnostic action,
6. request technician input,
7. interpret technician responses,
8. determine whether further diagnosis is required,
9. present a probable root cause,
10. stop when human confirmation is required.

---

# 13. LangGraph Requirement

The diagnostic workflow must be implemented using:

> **LangGraph**

LangGraph will manage:

- diagnostic state,
- graph transitions,
- persistence/checkpoints,
- human-in-the-loop interruptions,
- tool execution,
- diagnostic loops.

The implementation should not use LangGraph merely as a wrapper.

The graph must represent real workflow state.

---

# 14. Initial Diagnostic Graph

The initial graph should conceptually resemble:

```text
START
  ↓
understand_fault
  ↓
load_machine_context
  ↓
retrieve_evidence
  ↓
generate_hypotheses
  ↓
decide_next_action
  ↓
safety_check
  ↓
┌──────────────────────────┐
│ Need technician evidence?│
└──────────────────────────┘
       ↓ YES
ask_technician
       ↓
INTERRUPT / WAIT
       ↓
receive_observation
       ↓
update_hypotheses
       ↓
decide_next_action
       ↑
       └──────── loop

       ↓ NO / sufficient evidence
assess_root_cause
       ↓
human_confirmation
       ↓
save_maintenance_case
       ↓
END
```

The exact graph may evolve during implementation, but the diagnostic behavior must remain equivalent.

---

# 15. LangChain Usage

LangChain should be used where it provides clear value.

Expected uses include:

- LLM abstraction,
- embeddings,
- document loaders,
- text splitting,
- retrievers,
- structured outputs,
- tools.

Business-critical state must not exist only inside opaque LangChain objects.

Application data should remain explicitly represented in backend models.

---

# 16. Structured Fault Understanding

The first diagnostic stage must convert technician language into structured information.

Example input:

> "الدرايف عم يفصل بعد عشر دقايق وبيعطيني F30005."

Possible structured representation:

```json
{
  "component": "VFD",
  "symptom": "trip",
  "alarm_codes": ["F30005"],
  "timing": "after approximately 10 minutes",
  "operating_condition": null,
  "technician_observations": []
}
```

Structured output must use validated schemas.

Free-form LLM output must not become diagnostic state without validation.

---

# 17. Hypothesis Management

The agent must maintain explicit diagnostic hypotheses.

A hypothesis should include information such as:

```text
id
description
confidence
supporting_evidence
contradicting_evidence
status
```

Possible statuses:

```text
ACTIVE
LOW_PRIORITY
REJECTED
CONFIRMED
```

The system should not depend entirely on hidden LLM reasoning.

The visible diagnostic state must be structured and inspectable.

---

# 18. Guided Troubleshooting

This is a core MVP capability.

SALLIH should generally ask:

> **the next most useful diagnostic question or check**

rather than generating a large generic checklist.

Example:

```text
Current hypotheses:

1. Mechanical overload
2. Phase imbalance
3. Motor winding problem

Next useful check:

Measure current on L1, L2, and L3.
```

The system should briefly explain why the check is useful.

---

# 19. Technician Input Types

During a diagnostic session, the technician must be able to provide:

## Text Observation

Example:

> "في صوت احتكاك من جهة الموتور."

## Numeric Measurement

Example:

```text
L1 = 8.2 A
L2 = 8.4 A
L3 = 13.9 A
```

## Alarm Code

Example:

```text
F30005
```

## Confirmation

Example:

```text
Loose L3 terminal found.
```

Image input is optional and not required for MVP v0.1.

---

# 20. Human-in-the-Loop

The diagnostic graph must support interruption and continuation.

Example:

```text
Agent:
Measure L1, L2 and L3 current.

Graph:
PAUSE

Technician:
8.2 / 8.4 / 13.9 A

Graph:
RESUME
```

The user must not lose the diagnostic state between these steps.

---

# 21. Probable Root Cause

The system may present:

> **Probable Root Cause**

It must not present an AI-generated conclusion as a confirmed repair outcome.

Example:

```text
Probable root cause:
Phase imbalance related to L3.

Confidence:
High

Evidence:
- current measurements
- manual reference
- diagnostic history
```

The confirmed root cause must come from technician resolution.

---

# 22. Resolution Confirmation

A technician must explicitly resolve the diagnostic session.

The resolution should record:

```text
confirmed_root_cause
corrective_action
parts_replaced
notes
downtime
```

Not every field must be mandatory.

At minimum:

```text
confirmed_root_cause
corrective_action
```

must be captured.

---

# 23. Maintenance Memory

Resolved sessions must become reusable Maintenance Cases.

Future diagnostic sessions should be able to retrieve relevant past cases.

A Maintenance Case must preserve:

```text
machine_id
symptoms
alarm_codes
measurements
confirmed_root_cause
corrective_action
parts_used
diagnostic_summary
resolved_at
```

The AI must clearly distinguish:

```text
Previous similar case
```

from:

```text
Current confirmed diagnosis
```

---

# 24. Voice Input

Voice interaction is a MUST HAVE feature for the final MVP demonstration.

The technician must be able to:

```text
Press microphone
↓
Speak
↓
Receive transcript
↓
Review transcript
↓
Submit transcript
```

The first implementation may use the baseline ASR model without fine-tuning.

---

# 25. ASR Baseline

Initial baseline:

> **Whisper Large v3 Turbo**

Expected implementation may use an optimized inference runtime such as:

> faster-whisper / CTranslate2

if appropriate.

Initial languages:

- Arabic,
- Arabic-English code switching.

---

# 26. Transcript Confirmation

Critical technical tokens must be reviewable before entering diagnostic state.

This is particularly important for:

- alarm codes,
- measurements,
- part numbers,
- manufacturer names,
- model numbers.

Example:

```text
ASR:
F3005

User correction:
F30005
```

The corrected value becomes authoritative.

---

# 27. Syrian Industrial Speech Evaluation

The MVP should include a small evaluation dataset containing realistic maintenance expressions.

Examples:

```text
الدرايف عم يفصل.

الموتور عم يحمي.

الكونتاكتور ما عم يلقط.

الـ PLC ما عم يعطي output.

عم يعطيني overcurrent fault.
```

Evaluation should include technical English terms embedded in Arabic speech.

---

# 28. ASR Fine-Tuning

ASR fine-tuning is:

> **SHOULD HAVE, not a blocker for the first functional MVP.**

Fine-tuning should occur only after measuring baseline performance.

Required sequence:

```text
Baseline
↓
Evaluation
↓
Identify errors
↓
Collect targeted dataset
↓
Fine-tune / adapt
↓
Compare again
```

The project should not fine-tune merely to claim that fine-tuning was used.

---

# 29. Safety Layer

The MVP must include a safety classification mechanism.

Suggested action levels:

```text
LOW
MEDIUM
HIGH
```

Examples:

### LOW

- read an alarm code,
- inspect visible component state,
- retrieve documentation.

### MEDIUM

- perform standard technician measurements,
- inspect wiring with appropriate procedures.

### HIGH

- high-voltage electrical work,
- modifying safety circuits,
- bypassing interlocks,
- rewiring critical systems.

For HIGH-risk operations:

> SALLIH must require qualified technician verification.

SALLIH must never recommend bypassing a safety mechanism.

---

# 30. User Interface — MUST HAVE Screens

MVP v0.1 requires the following screens.

## 30.1 Machines

Purpose:

- view available machines,
- open Machine Passport.

For MVP:

one machine is sufficient.

---

## 30.2 Machine Details / Machine Passport

Contains:

- machine metadata,
- documents,
- recent maintenance cases,
- Start Diagnosis action,
- QR identity.

---

## 30.3 AI Troubleshooting

This is the main application screen.

Must display:

- selected machine,
- conversation,
- microphone input,
- structured diagnostic status,
- current question/action,
- evidence,
- hypotheses where appropriate.

---

## 30.4 Maintenance History

Shows resolved cases for the machine.

---

## 30.5 Maintenance Case Details

Shows:

- symptom,
- diagnostic steps,
- measurements,
- root cause,
- corrective action,
- evidence.

---

# 31. Evidence Panel

The Troubleshooting UI must include an Evidence section.

Possible representation:

```text
Evidence

Drive Manual
Page 74

Maintenance Case M-004
Resolved 17 Apr 2026

Technician Measurement
L3 = 13.9 A
```

Evidence must not be hidden inside generated answer text only.

---

# 32. Authentication

For MVP v0.1, authentication should remain simple.

Minimum requirement:

- authenticated technician account.

Complex enterprise identity features are not required.

Out of scope:

- SSO,
- Active Directory,
- complex RBAC,
- multi-tenant enterprise administration.

---

# 33. SHOULD HAVE Features

These features are valuable but may be reduced if implementation risk threatens the core diagnostic workflow.

## 33.1 Parts Compatibility Lite

Small controlled replacement-parts dataset.

Possible component classes:

- contactor,
- relay,
- sensor,
- fuse,
- motor,
- VFD.

Compatibility must be based on deterministic constraints where possible.

This feature is secondary to Guided Troubleshooting.

---

## 33.2 OCR for Scanned Manuals

Support scanned PDFs when needed.

---

## 33.3 Arabic Technical Normalization

Normalize common technical language.

Example:

```text
درايف
drive
VFD
inverter
```

into a consistent technical concept where context supports it.

---

## 33.4 Similar Maintenance Case Retrieval

Retrieve previous machine cases using:

- structured filtering,
- semantic similarity,
- component,
- alarm,
- symptom.

---

# 34. OPTIONAL Features

These features may be implemented only if the MUST HAVE scope is fully stable.

Examples:

- diagnostic-session export,
- PDF maintenance report,
- machine health summary,
- simple dashboard statistics,
- image attachment to maintenance cases,
- simple parts search,
- basic offline caching.

Optional features must not delay core MVP completion.

---

# 35. Explicitly Out of Scope

The following features MUST NOT be implemented in MVP v0.1 unless this document is formally revised.

## Enterprise / CMMS

- work orders,
- technician scheduling,
- preventive maintenance calendar,
- asset depreciation,
- purchasing workflows,
- supplier management,
- warehouse inventory,
- invoicing.

## Predictive AI

- predictive maintenance,
- remaining useful life,
- automatic failure forecasting.

## Machine Connectivity

- full PLC integration,
- continuous SCADA integration,
- industrial protocol implementation,
- Modbus monitoring,
- OPC-UA production integration.

## Autonomous Actions

- machine control,
- PLC writes,
- automatic reset,
- actuator control,
- remote repair execution.

## Broad Multi-Machine Support

- hundreds of machine types,
- generic universal fault library,
- fleet optimization.

## Multi-Agent Complexity

MVP v0.1 must not start as a system of multiple autonomous agents such as:

```text
Diagnostic Agent
Parts Agent
Safety Agent
Documentation Agent
Manager Agent
```

Instead:

> one primary Diagnostic Orchestrator + tools.

Future subgraphs may be introduced after the MVP demonstrates a clear need.

---

# 36. AI Provider Independence

The architecture should not tightly couple SALLIH to one LLM provider.

The application should support an abstraction layer allowing models to be changed.

Possible providers may include:

- Gemini,
- OpenAI,
- Anthropic,
- open-source models.

Provider selection for the MVP will be documented separately.

---

# 37. No LLM-Generated Technical Facts Without Evidence

The application must distinguish between:

```text
Retrieved technical fact
```

and:

```text
LLM-generated interpretation
```

For machine-specific claims, the preferred order is:

```text
Manufacturer documentation
↓
Machine maintenance history
↓
Technician evidence
↓
Model inference
```

Generic LLM knowledge must not silently override machine documentation.

---

# 38. MVP Benchmark Requirement

Evaluation is part of the MVP, not an optional academic appendix.

The system must be tested using approximately:

> **20–30 controlled fault scenarios**

where feasible.

Each scenario should define:

```text
symptoms
alarm
available evidence
measurements
expected diagnostic path
confirmed root cause
relevant document evidence
```

---

# 39. Baselines

SALLIH should eventually be compared against at least:

## Baseline A — General LLM

Fault description sent to a general LLM without machine context.

## Baseline B — Basic RAG

LLM with retrieved machine documentation but no structured diagnostic workflow.

## System C — SALLIH

Machine-grounded RAG + structured state + guided troubleshooting + memory.

This comparison helps determine whether the agentic workflow adds measurable value.

---

# 40. Diagnostic Metrics

Potential MVP metrics include:

## Root Cause Accuracy

```text
Top-1 Accuracy
Top-3 Accuracy
```

## Next-Step Accuracy

Was the requested diagnostic step technically useful?

## Citation Accuracy

Does the cited evidence actually support the claim?

## Unsupported Claim Rate

How often does the system generate claims unsupported by available evidence?

## Diagnostic Efficiency

Possible measures:

```text
number of diagnostic steps
time to useful diagnosis
number of unnecessary checks
```

## Technician Agreement

Expert evaluation of:

- usefulness,
- technical correctness,
- safety.

---

# 41. Voice Metrics

ASR evaluation should include:

## Word Error Rate

WER

## Technical Term Accuracy

Examples:

```text
VFD
PLC
contactor
overcurrent
F30005
Siemens
Schneider
```

## Alarm Code Accuracy

Alarm codes should be evaluated separately because a single character error may materially change diagnosis.

## Factory-Noise Performance

Where practical, evaluation should include simulated or real industrial background noise.

---

# 42. MVP Demo Scenario

The final demonstration should include at least one polished end-to-end scenario.

Recommended scenario:

```text
Packaging Line #01
```

Technician scans QR.

Technician says:

> "الدرايف عم يفصل بعد عشر دقايق وعم يعطيني overcurrent."

SALLIH:

1. transcribes the speech,
2. identifies the machine context,
3. retrieves the relevant manual,
4. generates hypotheses,
5. asks for phase-current measurements.

Technician enters:

```text
L1 = 8.2 A
L2 = 8.4 A
L3 = 13.9 A
```

SALLIH updates hypotheses.

The system recommends inspecting the relevant phase connection.

The technician confirms:

```text
Loose L3 terminal
```

The case is resolved and stored.

A second diagnostic session later demonstrates that SALLIH can retrieve the previous case as Maintenance Memory.

---

# 43. Definition of Done

MVP v0.1 is DONE only if the following end-to-end flow works without developer intervention:

- [ ] User can sign in.
- [ ] User can open the Machines screen.
- [ ] User can open the demo machine.
- [ ] Machine Passport displays correct metadata.
- [ ] Machine technical documents are indexed.
- [ ] Document retrieval is machine-scoped.
- [ ] User can start a diagnostic session.
- [ ] User can describe a fault through text.
- [ ] User can describe a fault through voice.
- [ ] ASR transcript is visible and editable.
- [ ] Fault description is converted to structured state.
- [ ] Agent retrieves relevant machine evidence.
- [ ] Agent creates diagnostic hypotheses.
- [ ] Agent chooses the next diagnostic action.
- [ ] LangGraph can pause for technician input.
- [ ] Technician can submit an observation or measurement.
- [ ] LangGraph resumes the same session.
- [ ] Hypotheses update based on new evidence.
- [ ] Safety checks are applied.
- [ ] System can produce a probable root cause.
- [ ] Important claims display evidence.
- [ ] Technician can confirm the actual root cause.
- [ ] Technician can record the corrective action.
- [ ] Resolved session becomes a Maintenance Case.
- [ ] Previous Maintenance Case can be retrieved in a later session.
- [ ] The complete demo can be performed without modifying code or database records manually.

If these conditions are satisfied:

> **SALLIH MVP v0.1 has achieved its core product objective.**

---

# 44. Scope Priority Order

If development time becomes limited, features must be protected in this order:

```text
1. Machine-grounded troubleshooting
2. Diagnostic state / LangGraph workflow
3. Evidence and citations
4. Maintenance Memory
5. Human-in-the-loop
6. Safety
7. Voice
8. ASR evaluation
9. Parts Compatibility Lite
10. Optional UI enhancements
```

Under no circumstances should visual polish or secondary features be prioritized over the correctness of the diagnostic workflow.

---

# 45. Final MVP Scope Statement

SALLIH MVP v0.1 is:

> **A machine-specific, Arabic-first, voice-enabled AI maintenance copilot that guides a qualified technician through an evidence-grounded diagnostic process and preserves the confirmed solution as reusable machine knowledge.**

Anything that does not materially contribute to proving this statement should remain outside MVP v0.1.