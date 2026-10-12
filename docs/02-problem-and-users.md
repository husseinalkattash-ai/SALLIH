# SALLIH — Problem & Users

**Document Version:** 1.0  
**Project Stage:** MVP v0.1  
**Status:** Draft for implementation baseline

---

# 1. Purpose of This Document

This document defines:

- the primary users of SALLIH,
- their working environment,
- the maintenance problems SALLIH is designed to address,
- their jobs-to-be-done,
- key pain points,
- realistic user scenarios,
- and the boundaries of the problem.

This document must be used together with:

- `01-product-vision.md`

SALLIH should not introduce features that do not clearly improve the workflows described here.

---

# 2. Primary User

## Maintenance Technician / Maintenance Engineer

The primary user of MVP v0.1 is a technician or engineer responsible for diagnosing and maintaining industrial machinery.

Typical responsibilities may include:

- responding to machine faults,
- interpreting alarm codes,
- inspecting electrical or mechanical components,
- reading manuals and schematics,
- taking measurements,
- identifying probable root causes,
- replacing or repairing components,
- documenting maintenance work,
- and communicating with other technicians or supervisors.

The user is assumed to be technically qualified to work on industrial equipment.

SALLIH supports the technician.

SALLIH does not replace the technician.

---

# 3. User Environment

The user may work in environments such as:

- food manufacturing,
- packaging,
- textile production,
- plastics,
- pharmaceutical manufacturing,
- printing,
- chemical manufacturing,
- metal processing,
- automated production lines,
- warehouses with industrial equipment,
- or industrial maintenance companies.

The machinery may include:

- legacy equipment,
- semi-modern equipment,
- PLC-controlled machines,
- VFD-controlled motors,
- conveyor systems,
- packaging machinery,
- pumps,
- compressors,
- motors,
- sensors,
- relays,
- contactors,
- and similar industrial components.

---

# 4. Mixed-Age Machine Environment

SALLIH must assume that factories may contain machines from different generations.

Example:

```text
Machine A
1998
Italian packaging machine
Paper/scanned documentation
No live connectivity

Machine B
2011
German production line
PLC-controlled
Digital manuals

Machine C
2025
Modern connected machine
Live alarms and telemetry available
```

The product architecture should therefore not assume that every machine:

- is connected,
- exposes an API,
- has IoT sensors,
- has modern documentation,
- or receives active OEM support.

MVP v0.1 must work without live machine connectivity.

---

# 5. Language Environment

The technician may communicate primarily in Arabic while using English technical terminology.

Typical examples:

> "الدرايف عم يفصل."

> "الموتور عم يسحب أمبير عالي."

> "الكونتاكتور ما عم يلقط."

> "الحساس ما عم يعطي signal."

> "الـ PLC ما عم يعطي output."

> "عم يعطيني overcurrent fault."

Therefore, user language may contain:

- Syrian Arabic,
- Modern Standard Arabic,
- English technical terminology,
- manufacturer terminology,
- part numbers,
- alarm codes,
- abbreviations.

SALLIH should not require the technician to convert this language into formal engineering English before interacting with the system.

---

# 6. Documentation Environment

Machine documentation may be fragmented across:

- manuals,
- scanned PDFs,
- electrical schematics,
- hydraulic diagrams,
- pneumatic diagrams,
- alarm-code references,
- spare-parts catalogs,
- manufacturer datasheets,
- service reports,
- handwritten notes,
- previous maintenance cases.

Documentation may be written in:

- English,
- German,
- Italian,
- Chinese,
- or other languages.

The technician may know that the correct information exists somewhere without knowing:

- which document contains it,
- which section is relevant,
- which page contains the required information,
- or how multiple documents relate to each other.

---

# 7. Core Problem Statement

The core problem addressed by SALLIH is:

> **Industrial maintenance knowledge is fragmented across documents, previous maintenance cases, and individual technician experience, making fault diagnosis slower, less consistent, and overly dependent on specific people.**

SALLIH aims to reduce this fragmentation by building a machine-specific knowledge layer that supports the technician during troubleshooting.

---

# 8. Problem Area 1 — Slow Information Retrieval

When a machine fails, a technician may need to manually search through multiple documents.

Example workflow:

```text
Fault occurs
   ↓
Find the correct manual
   ↓
Search the alarm code
   ↓
Find the electrical schematic
   ↓
Locate the component
   ↓
Check previous maintenance notes
   ↓
Ask another technician
```

This may consume valuable troubleshooting time before physical diagnosis even begins.

## Desired improvement

SALLIH should help the technician move toward:

```text
Fault occurs
   ↓
Describe fault
   ↓
SALLIH retrieves relevant machine evidence
   ↓
Technician begins targeted diagnosis
```

---

# 9. Problem Area 2 — Foreign-Language Technical Information

A technician may understand the machine operationally but struggle to quickly interpret:

- foreign-language manuals,
- complex manufacturer terminology,
- long technical documentation.

This becomes more difficult under production pressure.

## Desired improvement

The technician should be able to ask questions naturally in Arabic while SALLIH retrieves and explains information from foreign-language documentation.

Example:

```text
Technician:
"شو يعني F30005؟"

Machine documentation:
English VFD manual

SALLIH:
Retrieves the correct alarm definition
and explains it in Arabic with citation.
```

---

# 10. Problem Area 3 — Repeated Troubleshooting Work

A machine may experience the same or similar fault multiple times.

However, previous cases may be:

- undocumented,
- written in notebooks,
- stored in isolated reports,
- remembered only by experienced technicians.

As a result, another technician may repeat the entire troubleshooting process.

## Desired improvement

Previous resolved cases should become searchable Machine Memory.

Example:

```text
Current symptom:
Motor trips after ten minutes.

Machine Memory:
Similar case six months ago.

Previous confirmed cause:
Loose terminal on phase L3.
```

The system must present this as historical evidence, not as automatic proof that the current root cause is identical.

---

# 11. Problem Area 4 — Knowledge Dependency on Individuals

Factories may depend heavily on a small number of experienced technicians.

A senior technician may know:

- recurring machine problems,
- undocumented adjustments,
- component history,
- common failure patterns,
- practical diagnostic sequences.

If that technician leaves, retires, changes roles, or is unavailable, the factory may lose valuable knowledge.

## Desired improvement

SALLIH should help transform maintenance experience into structured organizational knowledge.

The product should gradually convert:

```text
Knowledge in technician's memory
```

into:

```text
Machine-specific organizational memory
```

---

# 12. Problem Area 5 — Poorly Structured Fault Reports

Initial fault descriptions are often informal.

Example:

> "الماكينة ما عم تشتغل منيح."

This is insufficient for reliable diagnosis.

The technician may need to clarify:

- when the fault occurs,
- which component is affected,
- whether an alarm appears,
- what changed before the fault,
- what measurements are available.

## Desired improvement

SALLIH should transform an informal complaint into a structured diagnostic session.

Example:

```text
Initial report:
"الدرايف عم يفصل."

SALLIH asks:
- Which alarm appears?
- Does it happen immediately or under load?
- What is the motor current?
- Was any component recently replaced?
```

This guided process is a major part of the product value.

---

# 13. Problem Area 6 — Diagnostic Premature Conclusions

Technicians may sometimes jump directly to a familiar explanation.

Example:

```text
Drive trips
→ assume drive is defective
→ replace drive
```

But the actual problem may be:

- wiring,
- motor overload,
- mechanical load,
- phase imbalance,
- a sensor,
- supply voltage.

## Desired improvement

SALLIH should encourage evidence-driven troubleshooting.

It should prefer:

```text
Hypothesis
↓
Evidence request
↓
Measurement
↓
Hypothesis update
```

over:

```text
Symptom
↓
Immediate replacement recommendation
```

---

# 14. Jobs to Be Done

The primary technician jobs-to-be-done are:

## JTBD-01 — Understand a Fault

> When a machine reports a fault, I want to quickly understand what the fault means in the context of this specific machine.

---

## JTBD-02 — Find Relevant Documentation

> When I need technical information, I want the relevant manual page, schematic, or maintenance case without manually searching through multiple files.

---

## JTBD-03 — Know What to Check Next

> When several possible causes exist, I want guidance on the next useful inspection or measurement that will narrow the diagnosis.

---

## JTBD-04 — Reuse Previous Experience

> When a similar fault happened before, I want to know what was previously observed and how it was resolved.

---

## JTBD-05 — Work Naturally in Arabic

> When I am standing near the machine, I want to describe the problem naturally in Arabic instead of translating my thoughts into formal technical English.

---

## JTBD-06 — Preserve the Resolution

> After solving a problem, I want the important diagnostic information stored so another technician can reuse it later.

---

# 15. User Persona — MVP Primary Persona

## Name

**Ahmad — Maintenance Technician**

This is a fictional persona used for product design.

## Role

Electrical / electromechanical maintenance technician.

## Experience

4–8 years.

## Working Environment

Medium-sized manufacturing facility.

## Languages

- Arabic: primary working language
- English: understands common industrial terms
- foreign manuals: may require translation or additional effort

## Typical Tools

- multimeter,
- clamp meter,
- laptop,
- smartphone/tablet,
- PLC software,
- printed/manual PDFs,
- WhatsApp,
- maintenance notebooks.

## Frustrations

Ahmad may experience:

- difficulty finding the correct manual section,
- long English documentation,
- incomplete machine history,
- dependence on senior technicians,
- repeated troubleshooting of previously solved faults,
- difficulty remembering exact past repairs,
- production pressure during downtime.

## Goal

Restore the machine safely and efficiently.

Ahmad does not want:

> "an AI conversation."

He wants:

> **useful help that reduces troubleshooting effort.**

---

# 16. Secondary Future Users

These are not primary MVP users.

They should influence architecture only when doing so does not increase MVP complexity.

## Maintenance Manager

Potential future needs:

- maintenance analytics,
- recurring-fault visibility,
- technician knowledge sharing,
- downtime reporting,
- fleet overview.

---

## Reliability Engineer

Potential future needs:

- failure-pattern analysis,
- preventive recommendations,
- reliability metrics,
- root-cause trends.

---

## Spare Parts / Procurement Team

Potential future needs:

- component identification,
- replacement compatibility,
- parts availability,
- procurement workflows.

---

## Factory Manager

Potential future needs:

- downtime visibility,
- maintenance performance,
- cost trends,
- asset health.

None of these roles require dedicated MVP v0.1 workflows.

---

# 17. Core User Scenario

The primary MVP scenario is:

## Machine Fault During Production

Packaging Line #01 stops unexpectedly.

The technician approaches the machine.

The drive shows:

```text
F30005
```

The technician scans the machine QR code.

SALLIH opens:

```text
Packaging Line #01
```

The technician presses the microphone button and says:

> "الدرايف عم يفصل بعد تقريباً عشر دقايق وبيعطيني F30005."

The ASR converts the speech to text.

SALLIH extracts:

```text
Machine:
Packaging Line #01

Component:
Drive / motor system

Alarm:
F30005

Temporal behavior:
after approximately 10 minutes
```

SALLIH retrieves the relevant drive manual section.

It identifies several possible causes.

Instead of providing an immediate conclusion, it asks:

> "ما قيم التيار على الفازات الثلاثة أثناء التشغيل؟"

The technician measures:

```text
L1 = 8.2 A
L2 = 8.4 A
L3 = 13.9 A
```

SALLIH updates the diagnostic state.

The system reduces the likelihood of some hypotheses and prioritizes phase-related problems.

It asks the technician to inspect the relevant connection.

The technician discovers:

> loose L3 motor terminal.

The repair is completed.

The technician confirms:

```text
Root cause:
Loose L3 terminal

Action:
Connection tightened

Downtime:
37 minutes
```

The diagnostic case becomes part of the machine's Maintenance Memory.

---

# 18. Returning Fault Scenario

Several months later, another technician reports:

> "الدرايف عم يعطي overcurrent بعد شوي من التشغيل."

SALLIH retrieves:

- relevant documentation,
- the previous L3 case,
- current diagnostic evidence.

The system states:

> A similar previous case existed, but the current root cause must still be verified.

This is critical.

Maintenance Memory is:

> **evidence**

not:

> **automatic diagnosis.**

---

# 19. Voice Interaction Scenario

The technician may be:

- standing beside the machine,
- holding tools,
- wearing gloves,
- unable to comfortably type.

Therefore:

```text
Press microphone
↓
Speak naturally
↓
Review transcript
↓
Continue troubleshooting
```

The transcript should remain visible.

For critical technical information such as:

- alarm codes,
- measurements,
- part numbers,

the user should be able to correct the transcript before the information becomes part of the diagnostic state.

Example:

```text
Detected:
F3005

Technician correction:
F30005
```

This prevents ASR mistakes from silently altering diagnosis.

---

# 20. Evidence Expectations

The technician should be able to distinguish between:

## Documentation Evidence

Example:

```text
Drive Manual
Page 74
Section: Motor Overload
```

## Maintenance Memory

Example:

```text
Case M-004
Resolved 2026-04-17
```

## Technician Observation

Example:

```text
L3 current = 13.9 A
```

## AI Inference

Example:

```text
Phase imbalance is currently a high-priority hypothesis.
```

The UI and backend should preserve these distinctions.

AI inference must not be presented visually as equivalent to manufacturer documentation.

---

# 21. User Trust Requirements

The technician should trust SALLIH because:

- it identifies which machine is being analyzed,
- it shows the source of important technical claims,
- it remembers previous confirmed cases,
- it exposes uncertainty,
- it does not pretend to know missing information,
- it asks for evidence,
- it does not bypass the technician,
- it avoids unsafe instructions.

Trust should come from:

> **traceability and behavior**

not from:

> confident wording.

---

# 22. User Experience Principles

## 22.1 Reduce Typing

Voice should be available for fault descriptions and conversational troubleshooting.

Structured numeric values may still use dedicated inputs where more reliable.

---

## 22.2 Keep Technical Context Visible

The interface should always clearly show:

- current machine,
- current fault,
- current hypotheses,
- collected measurements,
- relevant evidence.

---

## 22.3 One Question at a Time

During Guided Troubleshooting, SALLIH should generally avoid presenting a long checklist of ten possible tests.

Prefer:

> the next most useful diagnostic action.

---

## 22.4 Explain Why

When requesting an inspection or measurement, explain briefly why it matters.

Example:

> "نحتاج قياس تيار الفازات لمعرفة إن كان سبب الـ overcurrent مرتبطًا بعدم توازن الحمل."

---

## 22.5 Do Not Overload the User

The technician is troubleshooting a machine, not reading an AI report.

Responses should prioritize:

1. action,
2. evidence,
3. explanation.

---

# 23. Pain Severity Assumptions

The following are project hypotheses and must not yet be presented as validated facts for all factories:

- technicians spend excessive time searching documentation,
- knowledge loss is economically significant,
- Syrian Arabic voice materially improves maintenance workflows,
- maintenance teams will trust AI-guided troubleshooting,
- factories will pay for the product,
- machine documentation quality is sufficient for reliable RAG.

These assumptions must be validated with real factory users after MVP v0.1.

---

# 24. What SALLIH Does Not Solve

SALLIH does not solve:

- lack of spare parts,
- lack of qualified technicians,
- power shortages,
- production planning,
- procurement delays,
- mechanical damage that requires physical inspection,
- missing manufacturer information that does not exist anywhere,
- every possible failure on every industrial machine.

SALLIH improves:

> **access to knowledge, diagnostic structure, and preservation of maintenance experience.**

---

# 25. User Validation Goals After MVP

When presenting the MVP to factories, the project must investigate:

1. How technicians currently diagnose faults.
2. How often manuals are used.
3. Whether documentation search is slow.
4. Where maintenance history is stored.
5. Whether previous fault knowledge is lost.
6. Whether Arabic voice interaction is useful.
7. Whether technicians already use general AI tools.
8. Which SALLIH capability creates the highest value.
9. Which capability appears unnecessary.
10. Whether the company would provide one machine for a real pilot.
11. Whether the company would share anonymized maintenance cases.
12. What would make the company willing to pay for the product.

---

# 26. Validation Success Signal

A strong validation signal would be a factory saying:

> "Use one of our machines and test this with our maintenance team."

An even stronger signal would be:

> "Here are the machine documents and previous faults."

The strongest commercial signal would be:

> "If the pilot reduces troubleshooting time or preserves our maintenance knowledge, we would pay for it."

---

# 27. Product/User Alignment Statement

SALLIH should be designed around one fundamental relationship:

```text
Machine-specific knowledge
          +
Technician observations
          +
Previous maintenance experience
          ↓
Evidence-grounded guided troubleshooting
```

The primary user is not looking for an AI assistant.

The primary user is trying to:

> **get the machine running again safely and efficiently.**

Every MVP feature must support that goal.