# SALLIH — Documentation & Architecture Review

**Reviewer:** Claude Code
**Review Stage:** Pre-Implementation Architecture Review
**Status:** REVIEW COMPLETE
**Review Date:** 2026-10-12
**Documents Reviewed:** `CLAUDE.md`, `docs/01-product-vision.md` through `docs/09-system-architecture.md` (all read in full)
**Repository State at Review:** documentation only (no code, no dependency manifests)

---

# 1. Executive Assessment

All nine documents describe the same product. The MVP boundary, the "probable vs confirmed" distinction, the confirmed-transcript ASR boundary, machine isolation by query, citation-from-metadata, and the conditional fine-tuning policy are consistent and sound. These are not the problem.

The problem is that the documentation is consistent about **principles**. It is not yet specific about **mechanisms** in the places where a diagnostic copilot actually fails:

1. **State ownership is undecided.** The full diagnostic state is declared in both the LangGraph state (`06 §8`, `09 §40`) and the relational schema (`09 §25–§31`). Session `status` is written by the graph (`06 §41`, `09 §42`), but `06 §9` says the database is authoritative for it. No single writer is defined, and there is no consistency rule between checkpoint and database.
2. **The session state machine is contradictory and incomplete.** Three different status sets appear across documents, and no transition table exists. Several real situations have no defined state: the technician rejects the probable cause, the technician fixes the machine mid-loop, an insufficient-evidence session is later resolved, a provider fails mid-run, or the server crashes during `ANALYZING`.
3. **Resolution has two owners.** It is modelled both as graph nodes (`technician_confirms_resolution`, `save_maintenance_case`) and as a separate `POST /resolve` database transaction (`09 §78`).
4. **The safety boundary has open paths.** `safety_check` runs *before* an LLM node (`prepare_technician_request`) that writes the technician-facing text. Root-cause assessments, escalation messages, clarification questions and explanations are not safety-checked at all. Two incompatible safety vocabularies are in use (LOW/MEDIUM/HIGH and ALLOW/WARN/BLOCK). `03 §29` classifies "bypassing interlocks" as an allowed HIGH action.
5. **The execution and concurrency model for graph runs is unspecified.** Open-source LangGraph does not serialise concurrent runs on one thread. A resume is not bound to the pending action it answers. A crash mid-run leaves no defined recovery.

Below the blockers, the hypothesis lifecycle, confidence model, retrieval design, maintenance-memory provenance and evaluation methodology are described at the level of intent. They will not produce consistent behaviour until a few concrete rules are decided. In most cases a cheap, deterministic MVP option exists; Section 7 recommends one for each.

One factual error in the canonical examples should be fixed before scenarios are authored. On Siemens SINAMICS drives, `F30005` is "Power unit: Overload I²t", not overcurrent (overcurrent is `F30001`). The documents use `F30005` as "overcurrent" throughout (ISSUE-036).

**Overall readiness: NOT READY.** The product definition is stable. The gaps are concentrated in workflow state, safety output, concurrency and evaluation mechanics, and most have a clear recommended resolution. Section 20 lists the minimum work.

Issue count: **5 BLOCKER, 12 HIGH, 18 MEDIUM, 8 LOW.**

---

# 2. BLOCKER Issues

## ISSUE-001 — Diagnostic state has two sources of truth and no single writer

**Severity:** BLOCKER

**Affected Documents:**
- `06-ai-system-design.md` — §8 Diagnostic State Model, §9 State Ownership, §41 Human Interrupt
- `09-system-architecture.md` — §25–§31 diagnostic tables, §37 Database vs LangGraph State, §40 LangGraph State Schema, §41 Graph State Synchronization, §42 HITL Architecture
- `05-functional-requirements.md` — FR-STATE-001, FR-PERSIST-003
- `04-user-flows.md` — §33 Diagnostic Flow State Ownership

### Problem

The same data is declared in two persistent stores:

| Data | LangGraph state (`06 §8`, `09 §40`) | Relational table (`09 §25–31`) |
|---|---|---|
| status | `status` | `diagnostic_sessions.status` |
| structured fault | `structured_fault` | `structured_fault_json` |
| evidence | `evidence: list` | `evidence_items` |
| hypotheses | `hypotheses: list` | `diagnostic_hypotheses` |
| observations / measurements | lists | `technician_observations`, `measurements` |
| diagnostic steps | `diagnostic_steps` | `diagnostic_steps` |
| probable root cause | `probable_root_cause` | `probable_root_cause_json` |

The documents state that the database is authoritative (`06 §9`, `09 §37`, FR-PERSIST-003). They also say "LangGraph may actively manage these values" and "important diagnostic state should still be persisted or reconstructable when required" (`06 §9`). The graph itself sets `Session Status = WAITING_FOR_TECHNICIAN` (`06 §41`, `09 §42`).

The checkpoint is written by the LangGraph checkpointer, on its own connection, after a node completes. Application rows are written by repositories. These are **two non-atomic writes**. The documents define no rule for:

- which store wins after a crash between the two writes,
- which component is allowed to write `status`,
- whether nodes read domain data from graph state or from repositories,
- what happens when a node is re-executed after a crash. LangGraph re-runs a node whose completion was not checkpointed, so any database writes it made happen twice.

FR-STATE-001 explicitly requires this boundary to be defined in `06` and `09`. It is not.

### Why It Matters

This is the core persistence model. Every node, repository, API response and test depends on it. If the choice is made implicitly during coding, the likely result is both stores being updated "where convenient". That produces silent drift: the UI shows the database, the graph reasons from the checkpoint, and the two disagree after any partial failure. It cannot be fixed later without rewriting every node.

### Recommended Decision

Adopt a **thin-checkpoint / authoritative-database** model:

1. **PostgreSQL domain tables are the only source of truth** for all domain records: fault, evidence, observations, measurements, hypothesis snapshots, steps, assessments, status, resolution.
2. **LangGraph state contains only workflow-control data and references:** `session_id`, `machine_id`, `pending_action_id`, `iteration_count`, routing flags, `last_error`, and IDs of the records the current run created. Domain content is loaded from repositories inside nodes.
3. **Session `status` has exactly one writer: the application `DiagnosticSessionService`.** The graph never writes status. After each graph run, the service maps the run outcome (interrupt payload type, normal end, error) to a status transition, validated against the transition table (ISSUE-002).
4. **Every node side effect is idempotent**, keyed by deterministic identifiers such as `(session_id, run_id, node, ordinal)` or `(session_id, step_number)`. A re-executed node then cannot duplicate rows.
5. **Recovery rule:** if the checkpoint and the database disagree, the database wins. The graph is re-entered from the last checkpoint and nodes re-read from repositories.

Alternatives are listed in Section 7 (D1).

### Documentation Impact

- `06 §8–§9`: replace the "full state" model with a "workflow-control state + domain references" model.
- `09 §37–§42`: add the single-writer rule for `status`, the idempotent side-effect rule and the recovery rule.
- `05 FR-STATE-001`: reference the decision.
- Record as an ADR.

### Implementation Impact

Nodes become thin orchestration functions that call domain services. Graph tests can assert database side effects deterministically. Checkpoints stay small (see also ISSUE-016 and ISSUE-041).

---

## ISSUE-002 — Session state machine is contradictory, incomplete and has no transition table

**Severity:** BLOCKER

**Affected Documents:**
- `04-user-flows.md` — §32 Diagnostic Session Statuses, UF-16, UF-17, UF-24, UF-25, §37 Cancellation
- `05-functional-requirements.md` — FR-DIAG-005, FR-DIAG-017, FR-RES-001, FR-ERR-001, FR-ERR-007, FR-GRAPH-009
- `06-ai-system-design.md` — §7 graph, §44 `recommended_resolution_status`, §47–§48
- `09-system-architecture.md` — §133–§134 Diagnostic State Machine

### Problem

**Three different status sets:**

- `04 §32` / FR-DIAG-005: `NEW, WAITING_FOR_FAULT_DESCRIPTION, ANALYZING, WAITING_FOR_TECHNICIAN, READY_FOR_ASSESSMENT, PROBABLE_CAUSE_IDENTIFIED, WAITING_FOR_RESOLUTION, RESOLVED, CANCELLED, FAILED`
- `09 §134`: drops `READY_FOR_ASSESSMENT` and adds `INSUFFICIENT_EVIDENCE / escalation` as an "alternative exit".
- FR-DIAG-017 and `06 §47–§48` require an insufficient-evidence or escalation terminal result, but no document lists a status for it.

**Undefined or contradictory transitions:**

| Situation | What the documents say | Gap |
|---|---|---|
| Technician rejects the probable root cause ("terminal is fine") | nothing | `06 §7` routes `assess_root_cause → WAIT_FOR_RESOLUTION → END`. There is no path back into the loop. `06 §44` defines `NEEDS_MORE_EVIDENCE` but no edge consumes it. |
| Technician finds and fixes the fault while `WAITING_FOR_TECHNICIAN` | FR-RES-001 says "resolve an *active* session" | Is resolve allowed from any active status, or only after a probable cause? The demo itself (`04 §41` steps 12–13) has the technician finding the fault before the AI assessment. |
| Insufficient evidence reached | UF-24 "possible end state"; FR-DIAG-017 "terminal **or** escalation" | Terminal or resumable? Can the session still be resolved and become memory if the technician later fixes it with OEM help? |
| LLM or provider failure mid-run | `04 §32` has `FAILED`; UF-25, FR-ERR-001 and FR-ERR-007 require recoverability | If `FAILED` is terminal, a transient 503 ends the session, contradicting FR-ERR-001. If not terminal, its exits are undefined. |
| Server crash during `ANALYZING` | nothing | The session stays `ANALYZING` forever. No lease, timeout or recovery transition exists. |
| Cancel while a graph run is in progress | `04 §37` | The running graph completes and overwrites the status, so the cancellation is silently lost. |
| `NEW` vs `WAITING_FOR_FAULT_DESCRIPTION` | both listed | No observable difference. The session is created and immediately waits for a fault. |
| `PROBABLE_CAUSE_IDENTIFIED` vs `WAITING_FOR_RESOLUTION` | both listed | No event distinguishes them. |
| Fault description rejected by `understand_fault` (UF-09 → clarification) | UF-09 | Which status holds while waiting for clarification? |

`09 §133` says the backend must validate legal transitions, but no legal-transition table exists to validate against.

### Why It Matters

Status drives the UI, API authorisation of actions (which inputs are accepted), memory eligibility, evaluation outcomes and recovery. Without a table, each endpoint will improvise its own preconditions. "Failure to diagnose is acceptable" (Principle 7) cannot be implemented without a defined insufficient-evidence outcome.

### Recommended Decision

Define one status enum and one explicit transition table in `09` (canonical) and reference it from `04` and `05`. Recommended minimal set:

```text
AWAITING_FAULT            (replaces NEW + WAITING_FOR_FAULT_DESCRIPTION)
ANALYZING                 (graph run in progress; carries a lease, see ISSUE-005)
AWAITING_TECHNICIAN       (interrupt pending; pending_action_id set)
ASSESSMENT_PRESENTED      (probable cause shown; technician may accept, reject, add evidence, or resolve)
INCONCLUSIVE              (insufficient evidence / escalation; NOT terminal: technician may resolve or cancel)
RUN_FAILED_RETRYABLE      (last run failed; last valid state preserved; retry or cancel allowed)
RESOLVED                  (terminal)
CANCELLED                 (terminal)
```

Key transition rules to decide (Section 19, Q1–Q2):

- `RESOLVE` is allowed from `AWAITING_TECHNICIAN`, `ASSESSMENT_PRESENTED` and `INCONCLUSIVE` (recommended). A resolution that came without an AI probable cause still becomes memory, with `probable_root_cause = null`.
- `REJECT_ASSESSMENT` or new evidence on `ASSESSMENT_PRESENTED` → `ANALYZING` (back into the loop, counted against the iteration limit).
- `ANALYZING` with an expired lease → `RUN_FAILED_RETRYABLE`.
- `CANCEL` from any non-terminal status. If a run is in progress, cancellation is recorded and wins when the run ends (compare-and-set, ISSUE-005).
- `termination_reason` (`INSUFFICIENT_EVIDENCE`, `CONTRADICTORY_EVIDENCE`, `DIAGNOSTIC_LIMIT_REACHED`, `SAFETY_RISK`, …) is a separate field, not a status.

### Documentation Impact

`04 §32`, `05 FR-DIAG-005`, `09 §134`: replace with the single table. `06 §7`: add the reject/continue edge and the `INCONCLUSIVE` edge.

### Implementation Impact

One `SessionStateMachine` module, unit-testable without LLMs, used by every mutating endpoint. The frontend renders actions from `allowed_actions` returned by the API instead of inferring them.

---

## ISSUE-003 — Resolution and Maintenance Case creation have two conflicting owners

**Severity:** BLOCKER

**Affected Documents:**
- `03-mvp-scope.md` — §14 Initial Diagnostic Graph (`human_confirmation → save_maintenance_case → END`)
- `06-ai-system-design.md` — §7 graph (`WAIT_FOR_RESOLUTION → technician_confirms_resolution → save_maintenance_case`), §84 node definitions
- `09-system-architecture.md` — §13 `POST /diagnostic-sessions/{id}/resolve`, §38 node list, §76–§79 resolution transaction
- `05-functional-requirements.md` — FR-MEM-001 ("create **or finalize**")

### Problem

The documents describe resolution in two incompatible ways:

- **As graph nodes:** the graph interrupts at `WAIT_FOR_RESOLUTION`, resumes with the technician's resolution, and `save_maintenance_case` persists the case (`03 §14`, `06 §7`, `06 §84`, `09 §38`).
- **As an application transaction:** `POST /resolve` validates, marks `RESOLVED`, creates the case and commits in one database transaction, then triggers indexing (`09 §78`).

If both exist, one of these must happen:

- The API transaction runs and the graph thread stays interrupted forever at `WAIT_FOR_RESOLUTION`, a dangling thread.
- The API resumes the graph, and the case is written inside a node. That node's write is not atomic with the status change, and on re-execution it can create a second case. The unique constraint will then raise inside a graph node, which has no defined handling.

FR-MEM-001's "create or finalize" also implies a draft case that exists before resolution. No other document defines one.

### Why It Matters

Resolution is the only path into Maintenance Memory, which is the product's distinguishing capability. It must be atomic, idempotent and auditable. Two code paths for it guarantee inconsistency.

### Recommended Decision

- **Resolution is an application-domain operation, not a graph node.** `POST /resolve` performs, in one database transaction: compare-and-set the status from an allowed source status (ISSUE-002) to `RESOLVED`; insert `maintenance_cases` (unique on `diagnostic_session_id`); insert `RESOLUTION_CONFIRMED` and `SESSION_RESOLVED` events; write an outbox row for indexing. Commit, then enqueue indexing.
- **The graph's responsibility ends** when it presents an assessment, reaches `INCONCLUSIVE`, or is superseded by resolution or cancellation. After resolution, the thread is marked closed and its checkpoints may be deleted (ISSUE-041).
- Remove `technician_confirms_resolution` and `save_maintenance_case` from the graph node lists. Keep `human_confirmation` only in the sense of "graph stops and waits for a human decision".
- Drop "or finalize" from FR-MEM-001. There is no draft case in MVP.

### Documentation Impact

`03 §14`, `06 §7`, `06 §84`, `09 §38`, `09 §39`: remove the resolution nodes and describe graph termination. `09 §78`: make it the canonical resolution path. `05 FR-MEM-001`: reword.

### Implementation Impact

Resolution becomes a plain, fully testable service function. The graph becomes simpler. The LangGraph rule (CLAUDE.md §18) is unaffected, because the diagnostic loop remains fully represented in the graph.

---

## ISSUE-004 — Safety boundary has unvalidated output paths and contradictory classification semantics

**Severity:** BLOCKER

**Affected Documents:**
- `03-mvp-scope.md` — §29 Safety Layer
- `04-user-flows.md` — UF-15
- `05-functional-requirements.md` — FR-SAFE-001 to FR-SAFE-007, FR-TOOL-006, FR-DET-002
- `06-ai-system-design.md` — §7 graph, §37–§40, §44, §47, §80–§81, §84 `prepare_technician_request`
- `07-data-and-datasets.md` — §88 Safety Dataset
- `09-system-architecture.md` — §39 node table, §72–§75, §89 SSE

### Problem

**(a) Text produced *after* the safety check.** The graph order is `decide_next_action → safety_check → prepare_technician_request`. `prepare_technician_request` is typed "DETERMINISTIC / LLM-assisted" (`09 §39`) and "converts the approved action into concise technician-facing instructions" (`06 §84`). If an LLM writes the final wording, the text the technician reads was never checked. The checked object and the displayed object differ.

**(b) Technician-facing text that is never checked.** Safety applies to "proposed *physical* diagnostic actions" (FR-SAFE-001/002, `06 §37`, `09 §72`). Other technician-facing LLM output has no safety step:

- `assess_root_cause` output: description, remaining uncertainties, and any implied corrective action ("replace the drive", "re-terminate L3"). `06 §81` acknowledges the issue but no node handles it.
- `insufficient_evidence` and escalation text (`06 §47`).
- `ASK_CLARIFICATION` questions, which can embed physical instructions ("power-cycle the drive and tell me if…").
- `technical_explanation` / evidence-synthesis prose (`06 §51`, `§55`).
- Conversation messages (`09 §92`) and any SSE stream (`09 §89`). If SSE streams model tokens, unsafe text reaches the screen before any check.

**(c) "Is this physical?" is decided by the LLM.** Action type comes from `decide_next_action` (an LLM node). An action mislabelled `ASK_CLARIFICATION` or `REQUEST_DOCUMENT_CHECK` bypasses a check that only fires for physical actions.

**(d) "Deterministic rules" have nothing structured to run on.** `09 §73` lists prohibited categories, but the action is a free-text `instruction` in Arabic or English (`06 §31`). Keyword rules over free text produce:

- false negatives on paraphrase ("jumper the door switch", "اعمل جسر على حساس الباب"),
- false positives on negation ("do not bypass the interlock").

The "controlled assessment" half of `safety_check` is unspecified. If it is an LLM call, `06 §39` ("must not depend solely on the same LLM") is satisfied only nominally.

**(e) Two vocabularies with no mapping.** LOW/MEDIUM/HIGH (`03 §29`, `04` UF-15, FR-SAFE-001, `06 §37`) and ALLOW/WARN/BLOCK (`06 §80`, `07 §88`, `09 §72`). `09 §74` shows `HIGH` as either allowed or blocked. The safety evaluation dataset (`07 §88`) is labelled in a vocabulary the system design does not produce.

**(f) Prohibited action classified as allowed.** `03 §29` lists "modifying safety circuits" and "bypassing interlocks" under **HIGH**, whose behaviour is "require qualified technician verification". The same section, FR-SAFE-004 and `06 §39` prohibit bypass entirely.

**(g) HIGH has no defined behaviour.** The user is defined as already qualified (`02 §2`). "Require qualified technician verification" (`03 §29`) therefore has no observable effect beyond a warning string. No acknowledgement, record or UI gate is specified.

**(h) The classification criteria do not survive the demo.** The demo's central action, measuring phase currents with a clamp meter on a *running* motor inside a cabinet (`04 §41`), is energised work. Many site procedures treat that as high-risk, yet `03 §29` calls "standard technician measurements" MEDIUM. The next action, "inspect the L3 terminal", requires a *de-energised*, locked-out state. Actions differ in energy state, and the model has no attribute for it.

### Why It Matters

This is an industrial-safety product. CLAUDE.md §12.17 asks whether any path can expose unsafe instructions before validation. Paths (a), (b) and (c) do. A safety layer that checks one object and displays another provides assurance on paper only.

### Recommended Decision

1. **Single output gate.** Every technician-facing string produced or influenced by an LLM passes through one `SafetyGate.screen(output)` before it is persisted as a message or returned by the API. This covers request text, assessment, escalation, clarification and explanation. The gate is applied in the application service that builds the API response, so no node can skip it.
2. **Validate the final rendered artifact, not a precursor.** Either render request text deterministically from the validated structured action (template plus validated fields; recommended), or run the safety screen *after* `prepare_technician_request`.
3. **Structured action attributes**, so deterministic rules have something to evaluate. Each `DiagnosticAction` carries enumerated attributes validated by schema: `energy_state_required ∈ {NONE, DE_ENERGIZED_LOTO, ENERGIZED}`, `access ∈ {NONE, PANEL_EXTERNAL, CABINET_OPEN, GUARD_REMOVED}`, `touches_safety_function: bool`, `modifies_wiring_or_config: bool`, `machine_state_change ∈ {NONE, RESET, RESTART, JOG}`. Rules map attributes to a safety outcome. The strongest form of this is the check catalog in ISSUE-009.
4. **One vocabulary, two axes:** `risk_level ∈ {LOW, MEDIUM, HIGH}` (hazard description) and `decision ∈ {ALLOW, WARN, BLOCK}` (what the system does). The mapping is fixed: any prohibited category → `BLOCK` regardless of risk level; HIGH → `WARN` with mandatory acknowledgement; uncertain → treat as the higher level (FR-SAFE-006).
5. **Free-text screen as defence in depth only.** A bilingual prohibited-pattern list (Arabic, Syrian colloquial, English) runs on the rendered text. On match → `BLOCK` and escalate. Accept false positives.
6. **Define HIGH behaviour:** the instruction body is withheld until the technician explicitly acknowledges a site-procedure warning. The acknowledgement is recorded as a `SAFETY_ACKNOWLEDGED` event (FR-SAFE-007).
7. **No token streaming of LLM output** to the client. SSE (if added) carries progress events only.
8. **Safety rules are versioned configuration** (`safety_rules_version`, `09 §152`) owned by a named reviewer (Section 19, Q6).

### Documentation Impact

- `03 §29`: move bypass and safety-circuit modification to BLOCK; fix the levels.
- `05 FR-SAFE-001/002`: extend from "physical actions" to "all technician-facing generated content".
- `06 §37–§40`, `§80–§81`: two-axis model, output gate, attribute schema.
- `07 §88`: align the label vocabulary.
- `09 §72–§75`, `§89`: output gate placement; SSE progress-only.

### Implementation Impact

A `safety` domain module with no LLM dependency, testable with a bilingual red-team corpus (Section 16). Action schemas gain attribute fields, so `decide_next_action` prompts must produce them and validation must enforce them.

---

## ISSUE-005 — Graph execution model, resume binding and concurrency control are undefined

**Severity:** BLOCKER

**Affected Documents:**
- `09-system-architecture.md` — §13 API, §14 `SubmitMeasurementRequest`, §43 Human Input Idempotency, §44 Diagnostic Request Flow, §59, §89 SSE, §131–§132 Concurrency
- `05-functional-requirements.md` — FR-PERF-001, FR-PERF-004, FR-PERSIST-004/005
- `04-user-flows.md` — UF-14

### Problem

**(a) Where does a graph run execute?** `09 §44` implies the graph runs inside the HTTP request (`POST input → … → Resume LangGraph → Graph processes → … → Return updated session`). After a measurement, a run is roughly `update_hypotheses` (LLM) → `retrieve_additional_evidence` (embedding + database) → `decide_next_action` (LLM) → `safety_check` → `prepare_technician_request` (LLM?). That is 2–4 sequential LLM calls, commonly 10–60 s. The documents do not say:

- what happens on browser refresh or tab close during the run (FR-PERSIST-004),
- whether a client timeout and retry starts a second run,
- whether an SSE-driven run is cancelled when the client disconnects (a streaming response's generator is cancelled on disconnect, which would abort the graph mid-node).

**(b) No per-session mutual exclusion.** LangGraph's open-source library does not handle concurrent runs on the same thread. "Double-texting" strategies (reject, enqueue, interrupt, rollback) are a feature of the hosted LangSmith deployment, not the OSS framework. Two tabs, a double click, or a network retry can resume the same thread twice from the same checkpoint. That forks the checkpoint lineage and writes duplicate domain rows. `09 §131–§132` mentions a version check "where practical" and leaves it to "during coding".

**(c) Input is not bound to the question it answers.** `SubmitMeasurementRequest` (`09 §14`) has `input_id` but no `pending_action_id`. A stale tab can submit an answer to question N after the session has moved on to question N+1, and the backend cannot detect it.

**(d) `input_id` idempotency is "where practical"** (`09 §43`). It has no storage rule (unique constraint), no defined response for a replay (return the original result?), and does not cover `POST /fault` or `POST /diagnostic-sessions`.

**(e) No crash recovery.** If the API process dies mid-run, the status remains `ANALYZING` (ISSUE-002) and nothing re-drives the run.

### Why It Matters

Human-in-the-loop resume is a MUST (FR-GRAPH-005/006, FR-DIAG-009) and the demo's centrepiece. Without serialisation and binding, duplicate or out-of-order resumes corrupt the very state the product is built around. These failures are intermittent and hard to debug after the fact.

### Recommended Decision

1. **Bind every input to the pending action.** Every technician input request carries `pending_action_id` and a client-generated `input_id`. The backend rejects with `409 STALE_ACTION` if `pending_action_id` is not the session's current one.
2. **Compare-and-set run lease on the session row.** Start a run with a single atomic update:

   ```sql
   UPDATE diagnostic_sessions
      SET status = 'ANALYZING', run_id = :new, run_lease_expires_at = now() + :ttl, version = version + 1
    WHERE id = :sid
      AND status = 'AWAITING_TECHNICIAN'
      AND pending_action_id = :pid
   ```

   Zero rows → 409. The input row is inserted with a unique `(session_id, input_id)` in the same transaction. A replay returns the existing outcome.
3. **Execution model** (decision D4): recommended for MVP is **202 Accepted + in-process background task + polling.** `POST /input` validates, persists the input, acquires the lease, starts the graph run in an `asyncio` task in the API process, and returns `202 {run_id}`. The frontend polls `GET /diagnostic-sessions/{id}` (or subscribes to a progress-only SSE stream). Refresh is harmless. Disconnects cannot cancel runs. The graph is not moved into ARQ, as `09 §59` intends.
4. **Lease expiry recovery.** A periodic sweeper (or a check on `GET`) moves sessions whose lease has expired to `RUN_FAILED_RETRYABLE`. `POST /retry` re-acquires the lease and re-invokes the graph from its last checkpoint.
5. **Cancellation uses the same compare-and-set rule.** The run's final status write is conditional on `run_id` still matching. A cancelled session's run cannot overwrite `CANCELLED`.
6. **Configure LangGraph durability explicitly** (`durability="sync"`) rather than relying on a default that varies across versions and sources. Verify against the installed version.

### Documentation Impact

`09 §13–§14`, `§43–§44`, `§89`, `§131–§132`: replace "where practical" with the mechanism above. `04` UF-14: add the stale-action and in-progress behaviours.

### Implementation Impact

Adds `run_id`, `run_lease_expires_at`, `version` and `pending_action_id` columns, a `diagnostic_inputs` table with a unique `(session_id, input_id)`, and a small sweeper. This is a modest amount of code, and it makes concurrency testable (Section 16).

---

# 3. HIGH-Severity Issues

## ISSUE-006 — Hypothesis identity, update semantics and history are undefined

**Severity:** HIGH

**Affected Documents:**
- `06-ai-system-design.md` — §22–§30, §83
- `07-data-and-datasets.md` — §20
- `09-system-architecture.md` — §28
- `05-functional-requirements.md` — FR-HYP-002, FR-HYP-007, FR-AUDIT-005

### Problem

`generate_hypotheses` and `update_hypotheses` return a full `HypothesisSet` from the LLM (`06 §27`, `§83`). The LLM assigns IDs (`"H1"`, `"H2"`). Nothing specifies:

- how an updated set is reconciled with the previous one. An LLM can renumber, merge, rename or silently drop a hypothesis, so "stable identifier" (FR-HYP-002) is not enforced;
- whether a hypothesis can disappear without being `REJECTED`. It must not, or the trace loses why it was abandoned;
- whether `supporting_evidence_ids` and `contradicting_evidence_ids` are validated to exist in this session's evidence. Without validation, the LLM can cite fabricated evidence IDs, which reintroduces fabricated citations by another route (CLAUDE.md §21);
- how history is kept. `09 §28` has one mutable row per hypothesis (`updated_at`), so the evolution "MEDIUM → HIGH after measurement" (`06 §29`) required for research traces (FR-AUDIT-005, `04 §36`) is overwritten.

### Why It Matters

Hypothesis evolution is the core evidence that SALLIH "narrows the diagnosis over multiple steps" (`01 §17` item 7). It is also what the evaluation measures. Without update semantics, behaviour is non-deterministic and untestable.

### Recommended Decision

- **The LLM returns operations, not a replacement set:** `ADD {description, component, ...}`, `UPDATE {hypothesis_id, status?, proposed_confidence?, add_support[], add_contradiction[], rationale}`, `REJECT {hypothesis_id, contradicting_evidence_ids[], rationale}`.
- **The application assigns IDs** (UUID plus a per-session display key `H1…`). The LLM may only reference existing IDs. Unknown IDs or evidence IDs → validation failure → bounded retry → controlled error.
- **Invariant:** a hypothesis leaves the active set only through `REJECT` with at least one contradicting evidence ID, or by demotion to `LOW_PRIORITY`.
- **Persist an immutable hypothesis-set snapshot per iteration** (`hypothesis_snapshots(session_id, iteration, snapshot_jsonb, created_at)`), plus the current-set view. This is simpler than mutable rows with association tables and preserves full history.
- Enforce the 3–5 active-hypothesis target (`06 §28`) deterministically: cap `ADD` operations, and auto-demote beyond the cap by confidence.

### Documentation Impact

`06 §22–§30`: define the operation protocol and invariants. `09 §28`: snapshot table instead of (or in addition to) mutable rows. `07 §20`: note snapshots.

### Implementation Impact

A deterministic `apply_hypothesis_ops` node after each LLM hypothesis node. It is the natural place for the confidence ceiling (ISSUE-007). It is fully unit-testable with fake LLM output.

---

## ISSUE-007 — Confidence categories cannot be applied consistently as written

**Severity:** HIGH

**Affected Documents:**
- `06-ai-system-design.md` — §24–§26, §45, §86–§87
- `05-functional-requirements.md` — FR-HYP-009, FR-ROOT-005
- `03-mvp-scope.md` — §21

### Problem

- The semantics in `06 §25` are qualitative ("meaningful supporting evidence", "major alternatives have been weakened"). Two runs, or two implementers, will disagree. `06 §26` says "application logic should validate", but "the exact deterministic confidence rules may evolve", so no rule exists to implement.
- **Three overlapping vocabularies:**
  - hypothesis confidence `{INSUFFICIENT_EVIDENCE, LOW, MEDIUM, HIGH}` (`06 §24`),
  - evidence sufficiency `{INSUFFICIENT, PARTIAL, SUFFICIENT_FOR_PROBABLE_ASSESSMENT}` (`06 §86`),
  - termination reason `INSUFFICIENT_EVIDENCE` (`06 §48`).

  "INSUFFICIENT_EVIDENCE" as the confidence of a single *hypothesis* is semantically odd: a plausible but unsupported hypothesis is LOW.
- **Circularity:** the deterministic check would count evidence links that the LLM itself asserted. Rule-checking structure does not verify that the cited evidence actually supports the claim. That property must be measured offline (citation accuracy, FR-EVAL-006), not assumed.
- **"Independent pieces of evidence"** (`06 §25` HIGH) is undefined. Two chunks from the same manual page are not independent.

### Why It Matters

Confidence is displayed to technicians (`03 §21`) and recorded as an evaluation output (`06 §95` `final_confidence`). An inconsistent label erodes trust and makes calibration analysis meaningless.

### Recommended Decision

Treat confidence as **LLM proposal capped by a deterministic ceiling**: `final = min(proposed, ceiling(evidence_links))`. Recommended ceiling rules (MVP v1, versioned as `confidence_rules_version`):

| Ceiling | Required (all links validated to exist, same session, same machine) |
|---|---|
| LOW | default; any hypothesis |
| MEDIUM | ≥1 supporting **current-session** evidence item (`TECHNICIAN_MEASUREMENT`, `TECHNICIAN_OBSERVATION`, or a technician-confirmed alarm) **and** no unresolved contradicting current-session evidence |
| HIGH | MEDIUM conditions **and** ≥1 supporting current-session *discriminating* item (a measurement or inspection requested specifically to test this hypothesis, linked via `related_hypothesis_ids`) **and** ≥1 supporting item from a different source class (document or system rule) **and** every other active hypothesis is LOW or has contradicting current-session evidence |

Additional rules:

- Support **only** from `MAINTENANCE_CASE` evidence caps at LOW. This is the anti-anchoring rule (Principle 5).
- Remove `INSUFFICIENT_EVIDENCE` from the hypothesis-confidence enum. Keep it only as an assessment outcome and termination reason.
- Rename sufficiency categories so they cannot be confused with confidence: `assessment_readiness ∈ {NOT_READY, READY}`.

### Documentation Impact

`06 §24–§26`, `§86–§87`: replace the prose with the rule table and version. `05 FR-HYP-009`: drop `INSUFFICIENT_EVIDENCE` from the enum or redefine it.

### Implementation Impact

A pure function `confidence_ceiling(hypothesis, evidence, session)`, table-tested. Benchmark results can then report confidence against correctness meaningfully.

---

## ISSUE-008 — Numeric interpretation of measurements is left to the LLM; `SYSTEM_RULE` evidence has no source

**Severity:** HIGH

**Affected Documents:**
- `01-product-vision.md` — §10 ("deterministic logic should be preferred for numerical comparisons")
- `06-ai-system-design.md` — §15 (`SYSTEM_RULE`), §29, §43 ("a deterministic diagnostic rule produces a strong conclusion")
- `07-data-and-datasets.md` — §22
- `02-problem-and-users.md` — §17

### Problem

In the demo, "L3 is significantly higher than the other phases" is the pivotal inference. No component computes it. `update_hypotheses` (LLM) is expected to read `8.2 / 8.4 / 13.9` and conclude phase imbalance. This contradicts `01 §10` (numerical comparisons deterministic).

`SYSTEM_RULE` is listed as an evidence type (`06 §15`, `07 §22`), and `06 §43` relies on "a deterministic diagnostic rule", but no document defines a rule engine, a rule format, or any rule.

### Why It Matters

LLMs are unreliable at threshold arithmetic and unit handling, and results vary between runs. Because the demo hinges on this inference, its reproducibility depends on it.

### Recommended Decision

Add a small deterministic **measurement interpretation** step after input validation. For each measurement type, registered functions compute derived facts and emit `SYSTEM_RULE` evidence items, for example:

- `phase_current`: max deviation from mean (%), flag if above a configured threshold; compare to motor rated current from machine components if available.
- `voltage`: out-of-range relative to nameplate.

Each emitted fact stores `rule_id`, `rule_version`, inputs and output. This is the only MVP source of `SYSTEM_RULE` evidence. Keep the rule set small (2–5 rules for the demo machine).

### Documentation Impact

`06 §15` and a new subsection under §42: define `SYSTEM_RULE` provenance and the interpretation step. `07 §22`: same.

### Implementation Impact

One deterministic node (`interpret_measurements`) between input validation and `update_hypotheses`. It improves reproducibility and gives HIGH confidence an independent, non-LLM support class.

---

## ISSUE-009 — No controlled vocabulary of diagnostic checks; safety, forms, evaluation and the scripted technician all depend on free text

**Severity:** HIGH

**Affected Documents:**
- `06-ai-system-design.md` — §31–§36 (free-text `instruction`, LLM-generated `required_input_schema`), §93
- `07-data-and-datasets.md` — §42–§43 (uses action keys such as `MEASURE_PHASE_CURRENT`, `REPLACE_VFD` that no other document defines)
- `09-system-architecture.md` — §113 (scripted technician maps "Agent requests: phase_current")
- `05-functional-requirements.md` — FR-TOOL-005, FR-MEAS-006, FR-EVAL-005

### Problem

Four subsystems need to know *which* check was requested:

1. **Safety** needs structured attributes (ISSUE-004).
2. **Measurement forms** need a trusted input schema. Rendering an LLM-authored form schema directly (`06 §36`) lets the model invent fields, units and types.
3. **Evaluation** compares requested checks against `expected_useful_checks` and `acceptable_next_actions` (`07 §42–§43`). `07 §43` already assumes enumerated action keys.
4. **The scripted technician** (`09 §113`) must map a request to a scripted answer.

With free-text actions, each of these needs fuzzy matching or an LLM judge. That makes safety non-deterministic and the benchmark oracle leaky (ISSUE-014).

### Why It Matters

This one design choice decides whether safety, forms, evaluation and the oracle are deterministic or each need its own matcher.

### Recommended Decision

Introduce a **Diagnostic Check Catalog** per machine, as versioned configuration rather than enterprise tables. It is curated for the demo machine (likely 20–40 entries). Each entry has:

- `check_key` (for example `MEASURE_PHASE_CURRENT_MOTOR_M1`),
- localized instruction templates (AR/EN),
- `input_schema` (typed fields, units from an enum),
- safety attributes (ISSUE-004),
- related components.

`decide_next_action` selects a `check_key` plus parameters and writes the `reason`. For coverage, allow an `OTHER` action with free text. `OTHER` is always `WARN` with the generic site-procedure warning, uses a free-text observation response only, and goes through the full text screen.

Tradeoff: this reduces agent open-endedness and requires curation effort. It buys deterministic safety, trustworthy forms, exact evaluation matching and a non-leaky oracle. The decision belongs to the product owner (Section 19, Q5).

### Documentation Impact

`06 §31–§36`: action references the catalog. `07 §42–§43`: scenario checks reference `check_key`s. `09 §113`: the oracle maps `check_key` to scripted results.

### Implementation Impact

One YAML/JSON catalog file per machine, one loader, and prompt context listing the available checks. Evaluation matching becomes exact.

---

## ISSUE-010 — Technician response paths are missing: "cannot perform", partial answers, and free-text replies to structured requests

**Severity:** HIGH

**Affected Documents:**
- `04-user-flows.md` — UF-11, UF-12, UF-13, UF-24
- `06-ai-system-design.md` — §35 Repeated Action Prevention, §42
- `05-functional-requirements.md` — FR-MEAS-003

### Problem

Every flow assumes the technician performs the requested check and returns a valid result. Real outcomes have no representation:

- "I can't measure: the cabinet is locked / no clamp meter / production won't allow a stop."
- "L1 and L2 only; L3 is not reachable."
- The technician answers a measurement request with free text ("the readings are jumping between 9 and 14 A").
- The technician volunteers an unrequested finding ("I noticed the terminal was discoloured").

With only "valid measurement or rejected input", the session dead-ends. Or the agent re-asks the same check, which `06 §35` forbids, but nothing records that the check was attempted.

### Why It Matters

This is the most common real-world deviation from the happy path. It also directly affects next-step selection and evaluation (unnecessary-check counts).

### Recommended Decision

Every pending action accepts a typed response union:

- `RESULT` (schema-valid values or observation),
- `PARTIAL_RESULT` (subset plus reason),
- `UNABLE_TO_PERFORM {reason_code, note}`,
- `FREE_TEXT_INSTEAD` (stored as an observation linked to the action).

Plus a separate `ADD_OBSERVATION` input allowed in `AWAITING_TECHNICIAN` without answering the pending action. The step record stores the outcome. `decide_next_action` receives "attempted/unable" checks and must not re-request them unless conditions change.

### Documentation Impact

`04` UF-12/13: add branches. `06 §35`, `§42`: response types. `09 §14`: request models.

### Implementation Impact

A discriminated-union request model and one extra step status (`UNABLE`). The UI needs a "Can't do this check" button.

---

## ISSUE-011 — Cross-lingual retrieval and the embedding model are undecided, and pgvector requires a fixed dimension before the first migration

**Severity:** HIGH

**Affected Documents:**
- `02-problem-and-users.md` — §6 (manuals in English, German, Italian, Chinese)
- `06-ai-system-design.md` — §20–§21, §78
- `07-data-and-datasets.md` — §82–§83, §91
- `09-system-architecture.md` — §21, §51–§54, §85–§86, §99

### Problem

Fault descriptions arrive in Syrian Arabic with code-switching. Manuals are in English (and possibly German or Italian). No document states how a query in one language retrieves passages in another. The options are: build the retrieval query from normalized English concepts (LLM- or glossary-translated), use a multilingual cross-lingual embedding model, or both. `StructuredFault` fields such as `symptoms: ["trip"]` suggest English canonical values, but nothing requires it.

Separately, `document_chunks.embedding` is a pgvector column whose dimension is fixed in the migration. pgvector's HNSW/IVFFlat indexes on `vector` are limited to 2,000 dimensions (`halfvec` raises this). The embedding model must therefore be chosen before the first schema migration.

The choice also determines whether full manual text leaves the machine. A remote embedding API receives every chunk at ingestion time, which matters for future partner data (`07 §115`, `09 §105`).

### Why It Matters

Retrieval quality is the foundation of "machine context before generic knowledge". A monolingual embedding model with Arabic queries will retrieve poorly, and the failure will look like an LLM problem.

### Recommended Decision

- Retrieval queries are built from **both** the normalized English canonical concepts (from `StructuredFault`) and the original confirmed text.
- Use a **multilingual embedding model with documented Arabic↔English cross-lingual retrieval**. Choose between a local model (privacy, no per-call cost, adds a model runtime to the worker) and a remote API. Benchmark 2 candidates on the retrieval evaluation set (`07 §84`) *before* fixing the dimension.
- Store `embedding_model` and `embedding_version` per row. Retrieval filters on the active version (`07 §83`).

### Documentation Impact

`06 §20`: add the cross-lingual strategy. `09 §21`, `§85`: record the chosen model and dimension (ADR).

### Implementation Impact

It gates the Phase 2 migration. A small retrieval spike with 20–30 labelled queries is enough to decide.

---

## ISSUE-012 — Exact-identifier retrieval is required but has no defined mechanism

**Severity:** HIGH

**Affected Documents:**
- `06-ai-system-design.md` — §18, §20
- `09-system-architecture.md` — §51–§53
- `07-data-and-datasets.md` — §14 (OCR critical identifiers)

### Problem

The documents correctly require an "exact or lexical-aware retrieval path" for `F30005`, `K17`, `L1–L3` and `6SL3210-1KE14-3UF2`. They do not define one. Naive options fail:

- PostgreSQL full-text search with the `english` configuration stems and tokenizes. Hyphenated part numbers are split into fragments, and short tags like `K17` and `L3` are noisy.
- `ILIKE '%K17%'` matches `K170`.
- Semantic search does not distinguish `F30005` from `F30001`.

Normalization variants also exist: `F30005` vs `F 30005` vs `30005`, part numbers with or without hyphens, and OCR confusions (`0/O`, `1/I`).

### Why It Matters

An alarm code is often the strongest single clue. Returning the wrong alarm's documentation is worse than returning nothing.

### Recommended Decision

The simplest robust PostgreSQL-only design:

1. **Identifier extraction at ingestion:** machine-configurable regex patterns (for example alarm codes `^[FA]\d{5}$` for the selected drive family, device tags `^[KQS]\d{1,3}$`, part-number patterns) populate `chunk_identifiers(chunk_id, machine_id, identifier_type, raw, normalized)` with a B-tree index on `(machine_id, normalized)`. Normalization: uppercase, strip spaces and hyphens.
2. **Query time:** identifiers extracted from the structured fault are looked up exactly first. Exact hits are ranked first and labelled `match_type=EXACT`.
3. **Lexical fallback:** a `tsvector` with the `simple` configuration over the chunk content.
4. **Fusion:** reciprocal-rank fusion of the exact, lexical and vector result lists. Elasticsearch/OpenSearch is not needed (CLAUDE.md §50).
5. **"No exact match" is reported explicitly** to the graph (`06 §18`).

### Documentation Impact

`09 §51–§53`: specify the identifier table and fusion. `06 §20`: reference it.

### Implementation Impact

One table, one ingestion step, one query. It is fully testable with fixtures, including the isolation fixture (`07 §86`).

---

## ISSUE-013 — Maintenance Memory can be contaminated by AI text and by unlabelled synthetic or imported cases; no correction path exists

**Severity:** HIGH

**Affected Documents:**
- `04-user-flows.md` — UF-18 ("Diagnostic summary generated"), §38
- `07-data-and-datasets.md` — §24–§27, §98 Phase 2 ("Synthetic maintenance cases"), §112–§113
- `09-system-architecture.md` — §32–§33, §76, §126 (`diagnostic_session_id UNIQUE`)
- `05-functional-requirements.md` — FR-ADMIN-003, FR-MEM-003
- `03-mvp-scope.md` — §4 ("previous maintenance cases" as machine data)

### Problem

**(a) AI text masquerading as memory.** UF-18 generates a "diagnostic summary". `09 §32` stores `fault_summary` and `searchable_summary`, and `09 §33` embeds a search representation. If any of these is LLM-written, future sessions retrieve AI prose as `MAINTENANCE_CASE` evidence, a human-confirmed source class. This violates "AI inference must not masquerade as evidence" (`07 §22`, CLAUDE.md §12.13). It also violates the data-integrity rule.

**(b) Cases without a session.** FR-ADMIN-003 and `07 §98` require seeded historical and synthetic cases. `03 §4` lists previous maintenance cases as machine data, and a design partner will provide notebook history. These have no diagnostic session. `07 §24` types `diagnostic_session_id: str` (required) and `09 §126` makes it `UNIQUE`. The schema cannot represent imported history.

**(c) Provenance missing on cases.** No `source` field distinguishes a technician-resolved session, imported history, a synthetic demo fixture or an evaluation-run case. A synthetic case displayed in the demo's Maintenance History is indistinguishable from a real one, which violates the dataset honesty rule (CLAUDE.md §30).

**(d) No correction path.** `04 §38` mentions "explicitly allowed administrative corrections" but defines none. A wrong confirmed root cause permanently biases future retrieval.

**(e) Content mismatch.** `03 §23`, FR-MEM-003 and UF-18 list observations, measurements and diagnostic steps on the case. `07 §24` and `09 §32` do not include them. It is unspecified whether they are copied (snapshot) or referenced through the session.

### Why It Matters

Memory is the long-term asset. Once contaminated by AI text or mislabelled fixtures, it cannot be audited back to trustworthy content.

### Recommended Decision

- `maintenance_cases.source ∈ {RESOLVED_SESSION, IMPORTED_HISTORY, SYNTHETIC_FIXTURE, EVALUATION}`. `diagnostic_session_id` is nullable, with a partial unique index where it is not null. The UI badges non-`RESOLVED_SESSION` cases. Production retrieval excludes `SYNTHETIC_FIXTURE` and `EVALUATION` unless explicitly configured.
- **The search representation is built deterministically** from confirmed fields (confirmed root cause, corrective action, alarm codes, structured symptoms, measurement summaries). Any LLM-written summary is stored in a separately labelled `ai_summary` field. It is never embedded and never shown as confirmed content, unless the technician edits and approves it in the resolution form.
- Snapshot observations, measurements and steps into the case at resolution (immutable), so the case survives later document or session changes.
- `status ∈ {ACTIVE, RETRACTED}` with a reason and actor. Retracted cases are excluded from retrieval but kept for audit.

### Documentation Impact

`07 §24–§27`, `09 §32–§33`, `§126`, `04` UF-18, `§38`, FR-MEM-003.

### Implementation Impact

A few columns and a deterministic text builder. The demo still shows a convincing memory-retrieval moment, now with honest labels.

---

## ISSUE-014 — The baseline comparison is not like-for-like, and the oracle and scoring are undefined

**Severity:** HIGH

**Affected Documents:**
- `03-mvp-scope.md` — §38–§40
- `06-ai-system-design.md` — §93–§95
- `07-data-and-datasets.md` — §29–§30, §42–§43, §48, §85 (references the non-existent `10-evaluation-benchmark.md`)
- `09-system-architecture.md` — §112–§115
- `05-functional-requirements.md` — FR-EVAL-003/004/005

### Problem

**(a) Unequal information.** Mode A (General LLM) and Mode B (Basic RAG) are one-shot: fault → answer (`06 §94`). Mode C receives additional technician evidence (measurements, inspection results) through scripted interaction. Any accuracy gain in C is confounded: it may come from the structured workflow or simply from receiving more facts. The comparison cannot support the claim "the agentic workflow adds measurable value" (`03 §39`).

**(b) The oracle is undefined for unscripted requests.** The scripted technician (`09 §113`) answers requests that match the script. It is not specified what happens when SALLIH requests a check the scenario author did not anticipate. Answering "not available" penalises legitimate alternatives (`07 §42` explicitly allows acceptable alternatives). Answering with an LLM that sees ground truth risks leaking the root cause through the answer.

**(c) Root-cause scoring is undefined.** `expected_root_cause` is free text ("Loose L3 motor terminal"), and system outputs are free text ("Poor L3 electrical connection…"). Top-1 and Top-3 require a judge: human expert, LLM-as-judge, or a canonical label set. None is defined. For Mode A/B, "Top-3" requires a prompt for ranked candidates, which is also undefined.

**(d) Nondeterminism.** LLM outputs vary even at low temperature. A single run per scenario on 20–30 scenarios cannot separate systems.

**(e) The methodology document is missing.** `07 §85` defers to `10-evaluation-benchmark.md`, which does not exist. Evaluation is a MUST (`03 §38`, FR-EVAL-*).

### Why It Matters

This is the main research claim of a graduation project. A reviewer will immediately ask whether the workflow or the extra information produced the gain.

### Recommended Decision

Define two evaluation conditions:

- **Condition 1 — Equal information (fixed facts):** all modes receive the same initial description *plus* all scenario facts up front. This isolates reasoning and grounding quality.
- **Condition 2 — Interactive (information acquisition):** Mode C versus a **"RAG chat agent" baseline** that may ask questions answered by the *same* oracle. This isolates the value of structured state and guided checks. Report the number of checks requested and unnecessary checks.

Oracle:

- The scenario defines a fact table keyed by `check_key` (ISSUE-009).
- Requests outside the table return `NOT_AVAILABLE`, and this is counted.
- The oracle never sees `expected_root_cause`.
- No LLM is in the oracle loop.

Scoring:

- Each scenario has a canonical root-cause label plus accepted synonyms, authored before any runs.
- The judge is a human expert for the holdout set. An LLM judge is allowed for the development set only, with its model recorded and human agreement measured on a sample.

Runs: N≥3 repetitions per scenario per mode. Report the mean, the spread, and per-scenario tables. Make no significance claims beyond what N supports.

Write `10-evaluation-benchmark.md` before Phase 10, or move this content into `07`.

### Documentation Impact

`03 §39`, `06 §93–§95`, `07 §29–§53`, `09 §112–§115`. Create the evaluation methodology document.

### Implementation Impact

The harness drives the graph with a deterministic oracle and stores the run matrix. The `RAG chat agent` baseline is a small additional mode (FR-EVAL-003 already requires shared infrastructure).

---

## ISSUE-015 — Evaluation runs are not isolated from product state and Maintenance Memory

**Severity:** HIGH

**Affected Documents:**
- `09-system-architecture.md` — §112, §153
- `07-data-and-datasets.md` — §87, §97, §105
- `05-functional-requirements.md` — FR-ADMIN-003, FR-EVAL-001

### Problem

`SALLIH_FULL` evaluation uses the real application services (`09 §112`), and therefore the real database. If evaluation scenarios run to resolution (or seed their historical cases into the shared memory), later scenarios can retrieve cases created by earlier ones. This is cross-scenario leakage of ground truth through Maintenance Memory. Memory-behaviour cases (`07 §87` A/B/C) need *controlled* memory contents per scenario, and a shared mutable database cannot provide that. Run order would change results.

### Why It Matters

This directly invalidates benchmark results, and the effect is invisible unless someone looks for it.

### Recommended Decision

- Each evaluation run uses an **isolated database** (a dedicated schema or database created per run from migrations plus fixtures), with the scenario's declared memory fixture loaded and nothing else.
- Evaluation runs never trigger memory indexing of their own outcomes into shared stores.
- The run manifest records the fixture hashes.

### Documentation Impact

`09 §112`, `§153`; `07 §87`.

### Implementation Impact

The harness creates and tears down databases. With Docker Postgres this is straightforward.

---

## ISSUE-016 — Checkpoint and graph versioning, checkpointer schema ownership and driver integration are unaddressed

**Severity:** HIGH

**Affected Documents:**
- `09-system-architecture.md` — §35–§36, §125 Alembic, §159
- `06-ai-system-design.md` — §8
- `05-functional-requirements.md` — FR-GRAPH-007, FR-PERSIST-005

### Problem

- **Stale checkpoints.** Diagnostic sessions can stay interrupted for hours or days. Changing the graph topology (node names, edges) or the state schema (field types, Pydantic models serialized into checkpoints) can make existing checkpoints fail to resume or deserialize. No graph version is recorded per session, and no migration or invalidation policy exists.
- **Schema ownership.** `langgraph-checkpoint-postgres` creates its own tables via `setup()`. `09 §125` requires that all schema changes go through Alembic. The two must be reconciled. Alembic autogenerate will otherwise try to drop the checkpointer tables.
- **Driver duality.** The Postgres checkpointer uses psycopg 3. If SQLAlchemy uses asyncpg, the project runs two Postgres drivers with two connection pools, and transactions cannot span both.
- **Windows.** psycopg's async mode does not work with Windows' default `ProactorEventLoop` (see Section 17). The user's development machine is Windows 11.

### Why It Matters

Development will iterate on the graph many times while demo sessions sit in the database. Without a policy, the first refactor breaks every interrupted session.

### Recommended Decision

- `diagnostic_sessions.graph_version`. On resume with a mismatched version, the service rebuilds the workflow state from the authoritative database (enabled by ISSUE-001) into a new thread and records `GRAPH_MIGRATED`. Or, for MVP, it moves the session to `RUN_FAILED_RETRYABLE` with "restart from current evidence".
- Keep checkpoint state primitive (IDs and enums, ISSUE-001). That minimises deserialization fragility.
- Use **psycopg 3 for SQLAlchemy as well** (`postgresql+psycopg`), so there is one driver and one pool family.
- Exclude the checkpointer tables from Alembic autogenerate (`include_object`). Call the saver's `setup()` from a dedicated, idempotent migration step, and pin the checkpointer package version.
- Run the backend in Docker/WSL2 during development (Section 17).

### Documentation Impact

`09 §35–§36`, `§125`, `§159`; ADR for the driver and checkpointer ownership.

### Implementation Impact

Small, but it must be decided before the first migration.

---

## ISSUE-017 — Document lifecycle conflicts with evidence provenance; ingestion failure states are incomplete

**Severity:** HIGH

**Affected Documents:**
- `05-functional-requirements.md` — FR-DOC-004, FR-DOC-011, FR-MEM-004, FR-ERR-008
- `04-user-flows.md` — UF-22, UF-23
- `09-system-architecture.md` — §13 `DELETE /documents/{id}`, §29 `evidence_items.document_chunk_id`, §57, §60–§61, §82–§84
- `07-data-and-datasets.md` — §10

### Problem

**(a) Deletion vs provenance.** FR-DOC-011 says that after deletion "associated retrieval data must no longer be available". `evidence_items.document_chunk_id` (`09 §29`) references chunks used in past sessions and resolved cases (FR-MEM-004). A hard delete either violates the foreign key or cascades into deleting historical evidence, which destroys the audit trail. Reprocessing (`09 §61` "v2") has the same problem: new chunks replace old ones that are referenced.

**(b) Ingestion state gaps:**

- `UPLOADING` is listed, but with a synchronous multipart upload the server never observes "uploading". The `09 §57` order (record created → file stored → job queued → PROCESSING) leaves an orphaned record if the storage write fails.
- Enqueue after commit can fail (Redis down). The document is then stuck in `PROCESSING` forever, with no reconciler.
- A partial failure mid-pipeline (for example 40 of 300 chunks embedded) has no defined cleanup: are partial chunks deleted, or is the run resumed?
- Retry versus "Replace File" semantics are undefined. Replace creates a new document or a new version?
- Duplicate uploads of the same PDF (same hash) are not handled.

### Why It Matters

Documents are the evidence base. Losing provenance for past sessions breaks auditability. Stuck documents break the demo.

### Recommended Decision

- **Documents are never hard-deleted once any chunk is referenced by evidence.** `processing_status` gains `RETIRED` (excluded from retrieval; files and chunks kept). Evidence items also snapshot `document_name`, `page_number`, `section` and `content_excerpt` at creation (`09 §29` already copies content).
- **Chunk sets are versioned** by `processing_run_id`. Only the active run's chunks are retrievable. A new run is built in full, then atomically switched to active. Partial runs are discarded.
- **States:** `QUEUED → PROCESSING → READY | FAILED`, plus `RETIRED`. Storage write first (to a staging key), then insert the record, commit, then enqueue. A reconciler (ARQ cron or startup sweep) re-enqueues `QUEUED` or `PROCESSING` documents whose `updated_at` is older than a threshold.
- **Duplicate upload:** store `sha256`; for the same machine and hash, return the existing document.

### Documentation Impact

`04` UF-22/23, `05 FR-DOC-004/011`, `09 §57–§61`, `§82–§84`.

### Implementation Impact

Moderate. These are mostly state and constraint decisions that must exist before the first ingestion migration.

---

# 4. MEDIUM-Severity Issues

## ISSUE-018 — Structured-output schemas are not portable across providers

**Severity:** MEDIUM

**Affected Documents:**
- `06-ai-system-design.md` — §31 (`required_input_schema: dict | None`), §53, §64
- `09-system-architecture.md` — §40, §45 (`context: dict`), §47
- `05-functional-requirements.md` — FR-PROVIDER-003

### Problem

FR-PROVIDER-003 requires the internal schema to stay unchanged across providers. Native structured-output modes differ in which JSON Schema features they accept. For example, strict modes typically require closed objects with no free-form `dict`, and they handle optional and union types differently. Open dictionaries such as `required_input_schema: dict` cannot be expressed in strict modes at all. Domain models used directly as LLM output schemas will be shaped by whichever provider is tried first.

### Why It Matters

This is a hidden coupling. Switching providers later breaks validation in unexpected places.

### Recommended Decision

Separate the **LLM wire schemas** (flat, closed, provider-friendly, no open dicts; for example a check is referenced by `check_key` instead of an LLM-authored form schema) from the **domain models**, with explicit mapping functions. Forbid extra fields (`extra="forbid"`) on wire models. Retry budgets are explicit configuration.

### Documentation Impact

`06 §53`, `09 §47`.

### Implementation Impact

One `ai/schemas/wire` package plus mappers. Fakes return wire objects.

---

## ISSUE-019 — Provider abstraction is layered twice; LangChain's vector store should not own the retrieval schema

**Severity:** MEDIUM

**Affected Documents:**
- `06-ai-system-design.md` — §64–§65
- `09-system-architecture.md` — §45, §49, §87, §154
- `03-mvp-scope.md` — §15

### Problem

`09 §45` defines a custom `LLMProvider` with planned `GeminiProvider`, `OpenAIProvider` and `AnthropicProvider` classes. LangChain chat models already abstract providers and structured output. Building both means maintaining two abstraction layers.

Meanwhile, the fakes required by `09 §154` are awkward at LangChain's model level, because structured output on fake chat models generally depends on tool-binding support.

Separately, if LangChain's PGVector store is used for retrieval, it brings its own table layout and metadata-filtering model. That conflicts with the Alembic-managed `document_chunks` table, the SQL-level machine filter (`09 §23`) and the hybrid design (ISSUE-012).

### Why It Matters

Each layer is defensible in isolation. Together they produce extra code and leave the fakes in the wrong place.

### Recommended Decision

- Define **task-level ports** used by graph nodes, for example `FaultExtractor.extract(text) -> StructuredFaultWire` and `HypothesisReasoner.update(context) -> HypothesisOpsWire`, or one generic `StructuredLLM.invoke(task, wire_schema, context)`.
- Implement them with **one** LangChain-backed adapter. Provider switching is configuration of the LangChain model, not new adapter classes.
- `FakeLLMProvider` implements the ports directly with scripted responses keyed by task.
- Retrieval is a SQLAlchemy plus `pgvector` repository (`MachineKnowledgeRetriever`, `09 §87`). Optionally wrap it as a LangChain `BaseRetriever`, but do not use a LangChain vector store for storage.
- Keep LangChain for chat-model integration, embeddings, text splitting and, optionally, PDF loading.

### Documentation Impact

`09 §45`, `§49`; `06 §64`.

### Implementation Impact

Less code and cleaner fakes. This satisfies CLAUDE.md §12.7 (LangChain without opaque business logic).

---

## ISSUE-020 — ASR transcripts are not persisted or linked to sessions; audio retention is undecided

**Severity:** MEDIUM

**Affected Documents:**
- `08-asr-and-finetuning.md` — §20 (three artifacts must be preserved), §81–§82, §119 RQ-ASR-05
- `09-system-architecture.md` — §13 `POST /asr/transcriptions`, §69
- `05-functional-requirements.md` — FR-ASR-006

### Problem

`POST /asr/transcriptions` is not session-scoped and returns text. The "confirmed transcript" is whatever the client later sends to `POST /fault`. The server therefore:

- does not persist the raw transcript, so the raw → confirmed → normalized lineage that `08 §20` requires does not exist;
- cannot tell whether a given fault text came from voice, or how much the technician corrected it. RQ-ASR-05 ("does confirmation mitigate critical ASR errors?") and FR-EVAL-007 need exactly this data;
- has no rule for how long audio is kept (`08 §81` says "the project must define").

### Why It Matters

This is a cheap structural decision now and an impossible retrofit for data collected in pilot sessions later.

### Recommended Decision

- `asr_transcriptions(id, session_id?, user_id, model, model_version, raw_transcript, segments_json, duration, latency, created_at)`.
- `POST /fault` and observation inputs accept `{text, source: TEXT|VOICE, transcription_id?}`. The server stores the confirmed text, and a diff can be derived.
- Audio is deleted after transcription by default. Retention only with explicit research consent (Section 19, Q9).

### Documentation Impact

`08 §20`, `§81`; `09 §13`, `§69`.

### Implementation Impact

One table and one field. It enables a confirmation-edit-rate metric at no extra cost.

---

## ISSUE-021 — Voice scope during the loop and ownership of technical normalization are inconsistent

**Severity:** MEDIUM

**Affected Documents:**
- `02-problem-and-users.md` — §22.1 ("Voice should be available for fault descriptions and conversational troubleshooting")
- `05-functional-requirements.md` — FR-VOICE-001 (fault description), FR-UI-012
- `04-user-flows.md` — UF-07 (fault only)
- `08-asr-and-finetuning.md` — §4, §62 ("critical *initial* voice input"), §66, §71
- `03-mvp-scope.md` — §33.3 (normalization is SHOULD)
- `06-ai-system-design.md` — §12, §53 (`TechnicalNormalization` as an LLM output), §78
- `09-system-architecture.md` — §70–§71 (glossary-based normalizer after confirmation)

### Problem

- Whether voice is available for **observations** during the loop is unclear. If it is, `08 §62` limits mandatory confirmation to "initial" voice input, which would let an unconfirmed transcript enter state mid-session.
- **Normalization ownership:** `06` has the LLM normalize inside `understand_fault` and as a `TechnicalNormalization` structured output. `08 §66` and `09 §70–§71` describe a separate glossary layer. `03 §33.3` makes normalization SHOULD, while the `08 §4` pipeline makes it a mandatory stage.

### Why It Matters

These are implementable either way, but the code differs depending on the choice. The confirmation-scope gap is a correctness risk.

### Recommended Decision

- Voice is allowed for any free-text input. **Every** voice input goes through transcript review, not only the initial one. Measurements stay as structured forms (`08 §71`).
- Normalization = **deterministic glossary pass** (configuration, `09 §71`) that annotates canonical concepts, followed by the LLM `understand_fault`, which may use the annotations. The original text is always preserved. There is no separate LLM `TechnicalNormalization` output in MVP.
- Mark normalization as MUST for the glossary pass (it is cheap and required for retrieval, ISSUE-011). Advanced normalization remains SHOULD.

### Documentation Impact

`02 §22.1`, `05 FR-VOICE-001`, `08 §62`, `06 §53`, `§78`, `03 §33.3`.

### Implementation Impact

Small. It removes one LLM call.

---

## ISSUE-022 — ASR evaluation methodology lacks text normalization, pre-registered thresholds and deployed-runtime evaluation

**Severity:** MEDIUM

**Affected Documents:**
- `08-asr-and-finetuning.md` — §13–§19, §25, §31–§33, §41–§44, §55–§58, §102–§103, §120
- `07-data-and-datasets.md` — §61–§68

### Problem

- **Arabic WER normalization is unspecified.** WER depends heavily on orthographic normalization (diacritics, alef/hamza variants, taa marbuta, alef maqsura, tatweel, Arabic vs Latin digits, punctuation). Without a fixed normalizer, WER is not comparable across runs or with the literature.
- **Loanword convention.** `08 §32` says English terms are written in canonical English form "where clearly spoken". A technician saying "كونتاكتور" is speaking an Arabic loanword, and whether the reference is `كونتاكتور` or `contactor` changes both Technical Term Accuracy and WER. This must be fixed before annotation.
- **The fine-tuning gate has no quantitative thresholds** (`08 §41–§44`: "meaningful domain weakness", "persistent critical domain errors"). Without thresholds written *before* the baseline is measured, the decision can be rationalised after the fact. CLAUDE.md §12.16 asks for exactly this challenge.
- **Statistical power.** With 100–300 utterances, the alarm-code and part-number subsets may contain only tens of tokens. A 10-point change may be noise.
- **Runtime mismatch.** Fine-tuning will likely happen in Hugging Face Transformers, while deployment uses CTranslate2 (converted and possibly quantized). The evaluation must use the *deployed* runtime and settings, or the reported improvement may not exist in the product.
- **Regression.** Fine-tuning on a small domain set can degrade general Arabic. There is no general-Arabic regression set.

### Why It Matters

These are the difference between a defensible ASR chapter and one a reviewer can dismiss.

### Recommended Decision

- Fix a normalization function (versioned) and report WER and CER for both verbatim and normalized references.
- Write `asr-transcription-conventions.md` before recording.
- Pre-register the decision thresholds in the fine-tuning decision record template *before* the baseline run. For example: `FINE_TUNE` if, after confirmation-free evaluation, Critical Error Rate > X% **and** glossary/hotword/prompt mitigations reduce it by less than Y points.
- Report bootstrap confidence intervals.
- Evaluate base and adapted models through the same faster-whisper runtime with identical decoding settings (`beam_size`, `language`, VAD, `initial_prompt`/`hotwords`).
- Include a small general-Arabic regression set.
- Add `language` forcing (`ar` vs auto-detect) as an explicit baseline variable, because it affects code-switching output.

### Documentation Impact

`08 §13`, `§31–§33`, `§41–§44`, `§102–§103`, `§120`.

### Implementation Impact

Evaluation scripts only. No product impact.

---

## ISSUE-023 — ASR runtime placement inside the API process is unresolved

**Severity:** MEDIUM

**Affected Documents:**
- `09-system-architecture.md` — §64–§68, §102–§104
- `08-asr-and-finetuning.md` — §72–§73, §109

### Problem

"Inside the backend environment" or "a separate local process" (`09 §66`) is left open. Inside the FastAPI process:

- inference is CPU/GPU-bound and blocks the event loop unless it is offloaded,
- each uvicorn worker process loads its own model copy (memory × workers),
- GPU access from a Dockerised API on Windows requires the WSL2 NVIDIA container toolkit,
- an ASR out-of-memory error can take down the diagnostic API.

### Why It Matters

This affects development setup, demo reliability and the failure isolation that `08 §108` requires.

### Recommended Decision

Run ASR as a **separate local process** (a small HTTP service or worker) behind `ASRProvider`, loading the model once (`09 §68`). In development, this can run natively on the Windows host using the detected RTX 4060 (8 GB, sufficient for large-v3-turbo with fp16 or int8_float16) while the rest runs in Docker. The API calls it with a timeout and maps failures to `ASR_ERROR`. This is logical separation, not a microservice platform.

### Documentation Impact

`09 §66`, `§102`.

### Implementation Impact

One small process and one HTTP adapter. `FakeASRProvider` is trivial.

---

## ISSUE-024 — Browser and tablet constraints affect voice, QR and transcript editing

**Severity:** MEDIUM

**Affected Documents:**
- `03-mvp-scope.md` — §6.2 (QR required for the final demo), §24
- `04-user-flows.md` — UF-04, UF-07, UF-08
- `05-functional-requirements.md` — FR-UI-015/016/017
- `09-system-architecture.md` — §102–§104

### Problem

- **Microphone access requires a secure context.** Browsers only expose `getUserMedia` on HTTPS or `localhost`. A tablet scanning a QR code that points to `http://<laptop-LAN-IP>:3000` gets no microphone. The final demo combines QR, tablet and voice, so HTTPS on the LAN (local CA or reverse proxy) is required. No document mentions it.
- **Audio formats differ by browser.** Chromium MediaRecorder typically produces WebM/Opus; Safari/iPadOS produces MP4/AAC. The backend must decode both. faster-whisper decodes through PyAV, which should handle both, but this must be tested.
- **Bidirectional text.** Alarm codes, part numbers and measurements embedded in Arabic (RTL) text reorder visually. `6SL3210-1KE14-3UF2` and `L1 = 8.2 A` can display scrambled inside an RTL `textarea`. Transcript correction is the critical-token safeguard (UF-08), and it is exactly where this bites.

### Why It Matters

Each of these can break the live demo on the day, and none can be found by reading code.

### Recommended Decision

- Add HTTPS-on-LAN to the local/demo topology.
- Accept `webm` and `mp4` audio. Add a decode test.
- Render critical tokens inside `<bdi>`/`dir="ltr"` isolates. Offer the extracted critical tokens as separate LTR fields beside the free transcript (alarm codes, numbers) for correction.

### Documentation Impact

`09 §102–§104`; `04` UF-08.

### Implementation Impact

Small, but it must be planned into Phase 0 infrastructure.

---

## ISSUE-025 — Authorization model is under-specified (session ownership, document authority, ID access)

**Severity:** MEDIUM

**Affected Documents:**
- `05-functional-requirements.md` — FR-AUTH-004, FR-DOC-001, FR-ADMIN-002, FR-SEC-002
- `09-system-architecture.md` — §17, §146
- `04-user-flows.md` — UF-02 ("machines available to user")

### Problem

- "Machines available to user" (UF-02) implies per-user machine access, but no user–machine relationship exists. Is every authenticated user allowed to see every machine (single-site MVP)?
- **Session ownership:** can technician B view, continue or resolve technician A's session? Shift handover makes "yes" plausible. Without a rule, either IDOR exposure or blocked handover will be decided by accident.
- **Document authority:** FR-DOC-001 "authorized user", FR-ADMIN-002 "project/admin workflow", `09 §17` "optional ADMIN". Who uploads and who deletes (retires) documents is unspecified.

### Why It Matters

CLAUDE.md §12.29 asks that users cannot reach arbitrary data by changing IDs. The rule must exist before it can be enforced.

### Recommended Decision

Single-tenant MVP:

- All authenticated users can read all machines, sessions and cases.
- Any `TECHNICIAN` may submit input to or resolve a non-terminal session. Record `actor_id` on every input and event (supports handover). Confirm with the owner (Section 19, Q3).
- Only `ADMIN` uploads, retires or reprocesses documents and imports cases.
- Every endpoint validates that `session.machine_id` matches the `machine_id` in the path or derived context (FR-SEC-002).

### Documentation Impact

`05 FR-AUTH-004`, `09 §17`, `§146`.

### Implementation Impact

Simple role checks and one authorization dependency.

---

## ISSUE-026 — Three overlapping timelines; the source of LLM context is unspecified

**Severity:** MEDIUM

**Affected Documents:**
- `09-system-architecture.md` — §30 `diagnostic_steps`, §31 `diagnostic_events`, §92 `diagnostic_messages`
- `04-user-flows.md` — §35–§36
- `06-ai-system-design.md` — §55, §71

### Problem

Steps, events and messages all record "what happened", with no responsibility split. If messages are written independently, they can disagree with steps and events.

It is also not specified whether LLM nodes receive the **chat history** or only structured state. Using chat history reintroduces "chat as state" (CLAUDE.md §20). Not using it means technician remarks outside structured inputs are lost unless captured as observations.

### Why It Matters

This causes duplication and drift, and it creates ambiguity about what the model actually sees. The latter matters for reproducibility.

### Recommended Decision

- `diagnostic_events` = append-only audit and research trace (source of truth for "what happened").
- `diagnostic_steps` = requested-action records with outcomes (domain).
- **Messages are a projection** rendered from events and steps for the UI, not an independent store. Or, if stored, they are write-only presentation and never read by AI nodes.
- **LLM context is built only from structured state** (fault, evidence, observations, measurements, steps, hypotheses). Free technician remarks become observations (ISSUE-010).

### Documentation Impact

`09 §30–§31`, `§92–§93`; `06 §55`.

### Implementation Impact

Removes a table or makes it derived. Prompt builders become deterministic functions of state, which aids reproducibility.

---

## ISSUE-027 — Routing authority between LLM proposals and deterministic gates is unspecified

**Severity:** MEDIUM

**Affected Documents:**
- `06-ai-system-design.md` — §7, §32, §43, §49, §85–§87
- `09-system-architecture.md` — §98

### Problem

- `decide_next_action` (LLM) proposes `ASSESS_ROOT_CAUSE` or `ESCALATE` as action types. `06 §85` routes on "`evidence_sufficient_for_assessment`". It is not specified who computes it, or what happens when the LLM proposes assessment but the deterministic gate disagrees. Re-asking the LLM can loop.
- `ASSESS_ROOT_CAUSE` and `ESCALATE` flow through `safety_check` (`06 §7`), which is meaningless for non-physical actions and blurs the safety contract.
- "Iteration" is undefined: graph super-steps, LLM calls, or technician round-trips? `MAX_DIAGNOSTIC_ITERATIONS = 12` means different things under each definition.
- There is no path for the technician to request "your best assessment now".
- `06 §7` routes `retrieve_additional_evidence → decide_next_action`, so a new measurement skips re-generating hypotheses when retrieval adds new documents. This may be intentional but is not stated.

### Why It Matters

These rules determine loop termination, which is a MUST (FR-GRAPH-009).

### Recommended Decision

- **Iteration** = one technician round-trip (one pending action answered).
- A deterministic `assessment_gate` evaluates readiness (ISSUE-007 ceiling: top hypothesis HIGH, or MEDIUM with no competitor above LOW). If the LLM proposes assessment and the gate says NOT_READY, the gate overrides and the LLM is re-asked **once** with the gate's reason. A second disagreement routes to the best-available check, or to `INCONCLUSIVE` if none remain.
- At `max_iterations`, route to `assess_root_cause` with readiness recorded (it may legitimately produce `INCONCLUSIVE`), not directly to insufficient evidence.
- Non-physical actions do not pass through `safety_check`, but all their *text* passes through the output gate (ISSUE-004).
- Add a technician action `REQUEST_ASSESSMENT` that forces `assess_root_cause`.

### Documentation Impact

`06 §7`, `§43`, `§49`, `§85–§87`.

### Implementation Impact

Pure routing functions, fully graph-testable with fakes.

---

## ISSUE-028 — Dependency and platform decisions are missing: versions, ARQ status, Windows async, Redis licensing

**Severity:** MEDIUM

**Affected Documents:**
- `09-system-architecture.md` — §6, §59, §102, §160
- CLAUDE.md §44–§48

### Problem

No dependency manifest exists, and no versions are selected. Findings that affect decisions:

- **ARQ is in maintenance-only mode** (its PyPI page states this, referencing python-arq/arq issue #510). It is still usable, but expect no new features and slower fixes.
- **psycopg async does not work on Windows' default `ProactorEventLoop`.** The Postgres checkpointer is psycopg-based. Running the API natively on Windows requires a selector event-loop policy set before the loop starts. Uvicorn's own Windows loop selection can interfere, and so can ARQ's worker signal handling on Windows. This should be verified, not assumed.
- **faster-whisper/CTranslate2** GPU builds require CUDA 12 + cuDNN 9.
- **Redis licensing** changed for recent versions. This is acceptable for a graduation project but relevant to future on-prem commercial deployment. Valkey is a drop-in alternative if needed.
- The default Python on the development machine is 3.13.12; 3.12 is also installed.

### Why It Matters

CLAUDE.md §44–§47 requires verifying compatibility before implementation.

### Recommended Decision

See Section 17. In short: Python 3.12, all backend services in Docker (WSL2) for development, psycopg 3 as the single Postgres driver, keep ARQ but pin it and add the reconciler (ISSUE-017), and ASR native on the host GPU. Do not replace ARQ on maintenance status alone. Record the alternative (a Postgres-backed job table, which would remove Redis and make enqueue transactional) for revisit if ARQ issues appear.

### Documentation Impact

`09 §6`, `§102`; new ADRs.

### Implementation Impact

Phase 0 setup.

---

## ISSUE-029 — Evidence model inconsistencies (AI_INFERENCE, evidence lifecycle, fault report)

**Severity:** MEDIUM

**Affected Documents:**
- `05-functional-requirements.md` — FR-RAG-008 (includes `AI_INFERENCE`), FR-UI-010
- `06-ai-system-design.md` — §14–§16, §59
- `07-data-and-datasets.md` — §21–§22
- `09-system-architecture.md` — §29

### Problem

- FR-RAG-008 lists `AI_INFERENCE` as an evidence source type. `06 §15` and `07 §22` exclude it and add `SYSTEM_RULE`. FR-UI-010 requires the UI to show AI inference distinctly. Where AI inference lives is therefore contradictory.
- **Evidence item lifecycle:** is an `evidence_item` created for every retrieved chunk (top-k per retrieval, so many rows) or only for items the model actually used? This affects the citation-accuracy metric and storage.
- **The initial complaint is not evidence.** `06 §45` counts an "observed alarm" as current-session evidence, but the alarm arrives inside the fault description, which has no evidence type. Its status as a technician-confirmed fact is therefore undefined.
- `06 §59` claim categories (`SUPPORTED_BY_DOCUMENT`, …) assume claim-level attribution inside free-text explanations, which has no structure.

### Why It Matters

Evidence is a first-class object (CLAUDE.md §12.13). Its type system must be closed and consistent.

### Recommended Decision

- Evidence types: `MANUFACTURER_DOCUMENT`, `MAINTENANCE_CASE`, `TECHNICIAN_REPORT` (the confirmed fault description and its extracted alarm codes), `TECHNICIAN_OBSERVATION`, `TECHNICIAN_MEASUREMENT`, `SYSTEM_RULE`.
- AI inference is represented by hypotheses, assessments and explanations, which are separate entities. The UI styles them distinctly. Remove `AI_INFERENCE` from FR-RAG-008.
- Persist every retrieved candidate as `retrieval_results` (trace). Promote to `evidence_items` only those included in model context, and mark `cited=true` for those referenced by hypotheses or assessments.
- Implement `06 §59` as structured `claims[]` with `evidence_ids[]` on assessments and explanations rather than free-text tagging. Or defer it to post-MVP and say so.

### Documentation Impact

FR-RAG-008, `06 §15`, `§59`, `07 §22`, `09 §29`.

### Implementation Impact

Clarifies table semantics before migration.

---

## ISSUE-030 — Benchmark construction bias, holdout protection and small-sample claims

**Severity:** MEDIUM

**Affected Documents:**
- `07-data-and-datasets.md` — §31–§53, §93–§97, §102 Risk 3
- `03-mvp-scope.md` — §38

### Problem

- The same person likely authors scenarios, prompts and the system. Scenarios will unconsciously match the system's reasoning style.
- `DOCUMENT_DERIVED` scenarios are built from the same manual the RAG indexes. They can be "solved" by retrieving the manual's cause list, which inflates RAG modes without testing diagnosis.
- `SYNTHETIC` scenarios generated by an LLM from the same model family as the system under test share its biases.
- Holdout protection is procedural ("should not be repeatedly used") with no mechanism.
- With 20–30 scenarios (10 holdout), per-mode differences of a few scenarios are not meaningful.

### Why It Matters

This affects the credibility of the project's headline result.

### Recommended Decision

- Freeze the holdout early: author it, have it expert-reviewed, and commit a hash of the holdout file before prompt development starts. Keep the content outside the working tree used for tuning.
- Ensure at least part of the holdout is authored or reviewed by someone other than the system author.
- Cap the share of `DOCUMENT_DERIVED` and `SYNTHETIC` scenarios in scored results, and report results by `source_type`.
- When synthetic generation is used, use a different model family from the system under test.
- Report per-scenario outcomes. Frame results as a case study, not a statistical claim.

### Documentation Impact

`07 §50–§53`, `§93–§96`.

### Implementation Impact

None on product code.

---

## ISSUE-031 — Measurement validation: precision, units, plausibility, mixed units

**Severity:** MEDIUM

**Affected Documents:**
- `07-data-and-datasets.md` — §18–§19
- `09-system-architecture.md` — §14, §27
- `05-functional-requirements.md` — FR-MEAS-002/003

### Problem

- `values: dict[str, float]` (`09 §14`) conflicts with "preserve the technician-entered value" (`07 §19`): `8.20` and `8.2` become identical, and float representation artifacts appear.
- `unit` is free text, so `A`, `amps` and `أمبير` would all be accepted.
- There is no plausibility check (`139` typed instead of `13.9`).
- One `unit` per measurement cannot represent mixed measurements.

### Why It Matters

Measurements are the primary discriminating evidence, so their integrity is critical.

### Recommended Decision

- Accept values as decimal strings. Store the original string plus a `Decimal`.
- Units come from an enum per catalog `input_schema` (ISSUE-009).
- Add soft plausibility bounds per check from catalog or nameplate data. Out-of-bounds values require confirmation rather than being rejected.
- Mixed units = separate fields with per-field units in the schema.

### Documentation Impact

`09 §14`, `§27`; `07 §18`.

### Implementation Impact

Small.

---

## ISSUE-032 — File-handling security and evidence viewing are under-specified

**Severity:** MEDIUM

**Affected Documents:**
- `09-system-architecture.md` — §13, §62, §147
- `05-functional-requirements.md` — FR-SEC-003
- `04-user-flows.md` — §4.5 (evidence must be inspectable)

### Problem

- `09 §147` validates type, size and readability, but omits page-count limits, encrypted-PDF handling, parser timeouts and memory limits. PDF parsers have a history of crash and DoS bugs.
- Storage keys must never derive from user filenames (path traversal).
- **No endpoint serves the document or page image** for the "inspect evidence" interaction (`04 §4.5`: the technician opens Drive Manual page 74). Without an authenticated endpoint, the likely shortcut is serving storage statically, which bypasses authentication.

### Why It Matters

This is a security boundary and a core UX promise.

### Recommended Decision

- Parse only in the worker, with a timeout and resource limits.
- Enforce maximum size and pages; reject encrypted PDFs with a clear reason; use server-generated UUID keys.
- Add `GET /documents/{id}/file` and `GET /documents/{id}/pages/{n}` (authenticated, streaming).
- Optionally pre-render page images at ingestion.

### Documentation Impact

`09 §13`, `§147`.

### Implementation Impact

Two endpoints and worker limits.

---

## ISSUE-033 — Session creation idempotency, abandoned sessions and concurrent sessions per machine

**Severity:** MEDIUM

**Affected Documents:**
- `04-user-flows.md` — UF-05
- `05-functional-requirements.md` — FR-DIAG-001, FR-PERF-004
- `09-system-architecture.md` — §13

### Problem

- A double-clicked "Start Diagnosis" creates two sessions.
- Sessions abandoned in `AWAITING_TECHNICIAN` accumulate indefinitely, with no expiry or "stale" marking.
- Two technicians can start separate sessions for the same fault on the same machine. Is this allowed? Should the second be warned?

### Why It Matters

This creates history noise, confuses the Machine Passport, and produces duplicate memory if both are resolved.

### Recommended Decision

- Use an idempotency key on `POST /diagnostic-sessions`.
- The Machine Passport shows active sessions with "continue" as the primary action. Starting a new one is allowed but shows a warning.
- No auto-expiry in MVP. Display the last-activity age, and allow cancel.

### Documentation Impact

`04` UF-05, `09 §13`.

### Implementation Impact

Small.

---

## ISSUE-034 — Untrusted-content and data-egress boundaries are incomplete

**Severity:** MEDIUM

**Affected Documents:**
- `06-ai-system-design.md` — §57
- `09-system-architecture.md` — §105, §148
- `07-data-and-datasets.md` — §72–§73, §115

### Problem

- Prompt-injection guidance covers manuals only. Technician-entered text (observations, maintenance-case fields retrieved later as memory) is equally untrusted in prompts.
- The graph-constrained tool design (`06 §68`) already limits damage, since the LLM has no side-effect tools. Neither document records this as the primary mitigation.
- With a remote LLM (and possibly remote embeddings), every manual chunk and case enters a third-party API. For PUBLIC demo manuals this is fine. For PARTNER_CONFIDENTIAL data (`07 §73`) it requires consent. No per-dataset egress rule exists.

### Why It Matters

This becomes a future pilot blocker if left implicit, and it is a cheap rule to state now.

### Recommended Decision

- All non-system content goes into delimited data blocks, including technician and case text.
- The LLM has no side-effecting tools (state this as an invariant).
- The safety output gate (ISSUE-004) is the backstop.
- Add `data_classification` on machines and documents. Remote providers are allowed only for `PUBLIC` and `PRIVATE_PROJECT` unless consent is recorded.

### Documentation Impact

`06 §57`, `09 §105`, `§148`; `07 §73`.

### Implementation Impact

Prompt-builder discipline plus one field.

---

## ISSUE-035 — No behaviour defined when the technician reports an unsafe condition or practice

**Severity:** MEDIUM

**Affected Documents:**
- `06-ai-system-design.md` — §37–§39
- `04-user-flows.md` — UF-12, UF-15

### Problem

Safety is defined only for system-proposed actions. If the technician reports "I bridged the safety relay to keep testing" or "the E-stop is jumpered", the system's response is undefined. A naive loop will continue diagnosing within the unsafe configuration, or suggest checks that assume it.

### Why It Matters

An industrial copilot must not normalise a defeated safety function (Principle 9; FR-SAFE-004 in spirit).

### Recommended Decision

Run the same bilingual prohibited-pattern screen on technician input. On a match, record a `SAFETY_CONCERN` event, show a fixed (non-LLM) advisory to restore the safety function and follow site procedure, and constrain next actions so that none assume the defeated state. The session is not blocked, because the technician may be reporting something they found.

### Documentation Impact

`06 §39` and UF-12.

### Implementation Impact

Reuses the ISSUE-004 screen.

---

# 5. LOW-Severity Issues

## ISSUE-036 — Canonical examples misstate the meaning of `F30005`

**Severity:** LOW

**Affected Documents:**
- `04-user-flows.md` — UF-10 ("F30005 — Overcurrent"), §4.5
- `06-ai-system-design.md` — §17 (query "F30005 overcurrent motor trip")
- `02-problem-and-users.md` — §17
- `07-data-and-datasets.md` — §33 (scenario mixes `overcurrent` with the F30005 narrative)

### Problem

On Siemens SINAMICS drives, `F30005` is "Power unit: Overload I²t", a thermal or duty-cycle overload of the power unit. `F30001` is "Power unit: Overcurrent". This is consistent across third-party references and Siemens forum material; verify against the selected drive's own list manual. The documents use F30005 as "overcurrent" throughout and build the demo narrative and sample scenario on that pairing.

### Why It Matters

This is the exact error SALLIH exists to prevent: generic association instead of machine documentation. If copied into fixtures or scenarios, it would make the ground truth wrong. A correctly grounded system would then be scored as failing.

### Recommended Decision

Treat all alarm-code semantics in the documents as placeholders until verified against the chosen machine's manual. Correct the examples: use `F30001` for overcurrent, or keep `F30005` and describe it as I²t overload, which still fits "trips after ten minutes" well.

### Documentation Impact

The sections listed above.

### Implementation Impact

Scenario authoring must source alarm semantics from the indexed manual.

---

## ISSUE-037 — Referenced documents do not exist

**Severity:** LOW

**Affected Documents:** `07 §85` (`10-evaluation-benchmark.md`), `09 §171` (`12-implementation-roadmap.md`); `03 §36` (LLM provider "documented separately").

### Problem

These forward references point to nothing. The evaluation document is substantive (ISSUE-014).

### Why It Matters

Implementers will look for decisions that do not exist.

### Recommended Decision

Create the documents, or retarget the references.

### Documentation Impact

Listed sections.

### Implementation Impact

None directly.

---

## ISSUE-038 — Priority orders conflict; safety is scheduled late

**Severity:** LOW

**Affected Documents:** `03 §44`, `05 §40`, `06 §100`, `08 §129`, `09 §170–§171`

### Problem

Four priority orders differ. Safety ranks 6th (`03`), 8th (`05`), 13th of 16 (`06`) and sits in Layer 6 (`09`), after the LangGraph loop exists. Retrofitting the output gate (ISSUE-004) into an existing graph is harder than building with it.

### Why It Matters

Under schedule pressure, the "may be deferred" ordering will be followed literally.

### Recommended Decision

One canonical priority list. Introduce the safety module and output gate together with the first technician-facing output (default-deny stub at first). Only rule *coverage* grows later.

### Documentation Impact

Listed sections.

### Implementation Impact

Safety appears in Phase 5 rather than Phase 7.

---

## ISSUE-039 — Field-name and content inconsistencies across documents

**Severity:** LOW

**Affected Documents:** `03 §11` (`final_root_cause`), `04 §10` / `05` (`confirmed_root_cause`); `03 §22` (`parts_replaced`) vs `03 §23` / `07 §24` (`parts_used`); `03 §11` (`initial_fault_description`) vs `06 §8` (`original_fault_description`); `03 §16` / `05 FR-FAULT-004` (`component`, `symptom`, `observations`) vs `06 §11` (`components[]`, `symptoms[]`, `reported_observations`); `04 §10` (`evidence`) vs `03 §11` (`retrieved_evidence`).

### Problem

These are naming drift across documents.

### Why It Matters

API and schema names will be chosen arbitrarily.

### Recommended Decision

`06 §11` and `07` are the canonical schema sources. Update the examples in `03`, `04` and `05` to match.

### Documentation Impact

Listed sections.

### Implementation Impact

None if fixed before coding.

---

## ISSUE-040 — Hypothesis status `CONFIRMED` conflicts with the data-integrity rule

**Severity:** LOW

**Affected Documents:** `03 §17`, `05 FR-HYP-006`, `06 §23`

### Problem

`06 §23` itself recommends `STRONGLY_SUPPORTED` "where ambiguity may occur" but leaves the decision open. CLAUDE.md §28 forbids blurring AI and confirmed outcomes.

### Why It Matters

`CONFIRMED` will appear in the UI and in data exports next to `confirmed_root_cause`.

### Recommended Decision

Use `STRONGLY_SUPPORTED`. Remove `CONFIRMED` from the hypothesis enum.

### Documentation Impact

Listed sections.

### Implementation Impact

None if decided now.

---

## ISSUE-041 — Checkpoint retention and cleanup are undefined

**Severity:** LOW

**Affected Documents:** `09 §36`, `§127`

### Problem

Checkpoints accumulate per super-step for every session indefinitely.

### Why It Matters

This means database growth and slower inspection. It is minor at MVP scale.

### Recommended Decision

Delete a thread's checkpoints when its session becomes terminal (the database is authoritative, ISSUE-001). Retain them in evaluation mode for trace analysis if needed.

### Documentation Impact

`09 §36`.

### Implementation Impact

One call in the resolve and cancel services.

---

## ISSUE-042 — "The agent decides which tool to call" overstates the actual design

**Severity:** LOW

**Affected Documents:** `01 §9`, `06 §4` vs `06 §68`, `09 §51`

### Problem

Retrieval is executed by graph nodes with a deterministic query builder (`06 §68`, `09 §51`). The LLM chooses *checks* and hypotheses, not tools. The vision text describes a tool-calling agent.

### Why It Matters

This creates expectation mismatch for reviewers, and the temptation to add ReAct-style tool calling "to match the docs".

### Recommended Decision

Describe the design accurately as "a stateful LangGraph workflow with LLM decision nodes and graph-constrained tools". This matches CLAUDE.md §17–§18 and is the right MVP design.

### Documentation Impact

`01 §9`, `06 §4`.

### Implementation Impact

None.

---

## ISSUE-043 — LLM call capture for reproducibility is unspecified

**Severity:** LOW

**Affected Documents:** `06 §56`, `§95`; `09 §110`, `§151–§152`; CLAUDE.md §51

### Problem

Traces record model, prompt version and outputs, but not the rendered prompt and raw response per call. Reconstructing a benchmark run (CLAUDE.md §51) needs them.

### Why It Matters

Without them, failures found during evaluation cannot be debugged.

### Recommended Decision

Add an `llm_calls` table (task, prompt name and version, rendered-input hash and optional full text in evaluation mode, raw output, parsed output, validation result, retries, tokens, latency), linked to `run_id` and `session_id`.

### Documentation Impact

`09 §110`.

### Implementation Impact

One table and one decorator in the adapter.

---

# 6. Cross-Document Contradictions

| # | Contradiction | Locations |
|---|---|---|
| C-01 | `AI_INFERENCE` is an evidence source type vs AI inference is explicitly *not* evidence; `SYSTEM_RULE` present in one set only | FR-RAG-008 vs `06 §15`, `07 §22` |
| C-02 | Safety vocabulary LOW/MEDIUM/HIGH vs ALLOW/WARN/BLOCK, no mapping; HIGH shown both allowed and blocked | `03 §29`, UF-15, FR-SAFE-001, `06 §37` vs `06 §80`, `07 §88`, `09 §72–§74` |
| C-03 | "Bypassing interlocks" and "modifying safety circuits" are classified HIGH (allowed with verification) vs absolutely prohibited | `03 §29` HIGH list vs `03 §29` final line, FR-SAFE-004, `06 §39`, `09 §73` |
| C-04 | Session status sets differ (`READY_FOR_ASSESSMENT` vs `INSUFFICIENT_EVIDENCE/escalation`) | `04 §32`, FR-DIAG-005 vs `09 §134` |
| C-05 | `FAILED` status vs the requirement that provider failures remain recoverable | `04 §32` vs UF-25, FR-ERR-001, FR-ERR-007 |
| C-06 | Resolution as graph nodes vs as an API transaction | `03 §14`, `06 §7`, `§84`, `09 §38` vs `09 §13`, `§78` |
| C-07 | Database authoritative for session status vs graph sets status | `06 §9`, `09 §37` vs `06 §41`, `09 §42`; status also in graph state (`06 §8`, `09 §40`) |
| C-08 | Maintenance case requires a unique diagnostic session vs seeded or historical cases without sessions | `07 §24`, `09 §126` vs FR-ADMIN-003, `07 §98`, `03 §4` |
| C-09 | Case contains observations, measurements and diagnostic steps vs case tables omit them | `03 §23`, FR-MEM-003, UF-18 vs `07 §24`, `09 §32` |
| C-10 | Normalization is SHOULD vs a mandatory pipeline stage; LLM-performed vs glossary-performed | `03 §33.3` vs `08 §4`, `09 §70`; `06 §12`, `§53` vs `08 §66`, `09 §71` |
| C-11 | Voice for conversational troubleshooting vs voice for fault description only; confirmation for "initial" input only | `02 §22.1`, FR-UI-012 vs FR-VOICE-001, UF-07, `08 §62` |
| C-12 | Agent decides which tool to call vs graph-constrained, deterministic retrieval | `01 §9`, `06 §4` vs `06 §68`, `09 §51` |
| C-13 | Preserve the technician-entered value vs `dict[str, float]` | `07 §19` vs `09 §14` |
| C-14 | Deletion removes retrieval data vs evidence must remain associated with cases | FR-DOC-011 vs FR-MEM-004, `09 §29` |
| C-15 | Assessment can recommend `NEEDS_MORE_EVIDENCE` vs the graph has no edge back into the loop | `06 §44` vs `06 §7` |
| C-16 | Four different implementation priority orders | `03 §44`, `05 §40`, `06 §100`, `09 §170` |
| C-17 | `INSUFFICIENT_EVIDENCE` used as hypothesis confidence, as sufficiency state and as termination reason | `06 §24`, `§86`, `§48`; FR-HYP-009 |
| C-18 | Hypothesis `CONFIRMED` vs the recommendation to avoid it | `03 §17`, FR-HYP-006 vs `06 §23` |
| C-19 | Async ingestion SHOULD vs "must occur asynchronously" | FR-PERF-003 vs `09 §57` |
| C-20 | `UPLOADING` status vs a flow that never sets it | FR-DOC-004, UF-22 vs `09 §57` |
| C-21 | Field names (`final_` vs `confirmed_root_cause`, `parts_replaced` vs `parts_used`, `initial_` vs `original_fault_description`, singular vs plural fault fields) | see ISSUE-039 |
| C-22 | Documents reference non-existent `10-evaluation-benchmark.md` and `12-implementation-roadmap.md` | `07 §85`, `09 §171` |
| C-23 | `F30005` described as overcurrent vs Siemens definition (I²t overload) | UF-10, `06 §17`, `07 §33` vs external documentation (ISSUE-036) |

---

# 7. Missing Architecture Decisions

Each decision below should become a proposed ADR in `docs/reviews/architecture-decisions.md`. I have not marked any as accepted (CLAUDE.md §37).

### D1 — Diagnostic state ownership (ISSUE-001)

- **Why needed:** every node, table and test depends on it.
- **Recommended:** a thin checkpoint (workflow control and references); PostgreSQL authoritative for all domain data; idempotent node side effects; the application service is the single writer of `status`.
- **Alternatives:**
  - (B) A thick checkpoint as working state, projected to the database at each interrupt. This gives simpler nodes but two copies of every field, and projection bugs become silent drift.
  - (C) No `interrupt()`: each API call runs the graph from a deterministic entry router over database state. This is the most robust request/response model, but it weakens the documented interrupt/resume MUSTs (FR-GRAPH-005–007) and reduces LangGraph to routing.
- **Tradeoff:** the recommended option needs repository reads in nodes (slightly slower; negligible at MVP scale) in exchange for one truth.

### D2 — Session status enum and transition table (ISSUE-002)

- **Why needed:** API preconditions, UI actions and memory eligibility.
- **Recommended:** the 8-state set in ISSUE-002, with `termination_reason` as a separate field.
- **Alternatives:** keep the 10 documented states. This is more granular but contains redundant and undefined states.
- **Tradeoff:** fewer states means more information carried in `termination_reason` and `pending_action`.

### D3 — Resolution path (ISSUE-003)

- **Why needed:** atomic, idempotent Maintenance Memory.
- **Recommended:** a domain transaction outside the graph, with an outbox for indexing.
- **Alternatives:** a graph node for resolution. This is "more LangGraph", but it is non-atomic and leaves dangling threads.
- **Tradeoff:** the graph no longer contains the final step. This is acceptable, because confirmation is a human or domain act, not workflow reasoning.

### D4 — Graph run execution model (ISSUE-005)

- **Why needed:** refresh, timeout and disconnect behaviour.
- **Recommended:** `202 Accepted` + in-process background task + lease + polling (progress-only SSE optional).
- **Alternatives:**
  - (B) Synchronous request with long timeouts. This is simplest, but refresh, retry and disconnect are fragile.
  - (C) An ARQ job per run. This survives API restarts, but adds latency and complexity to interactive reasoning, contrary to `09 §59`.
- **Tradeoff:** (A) loses an in-flight run on API crash. That is acceptable, because lease expiry leads to a retryable status.

### D5 — Concurrency control (ISSUE-005)

- **Recommended:** a compare-and-set lease on the session row; `pending_action_id` binding; unique `(session_id, input_id)`.
- **Alternatives:** a Postgres advisory lock held for the whole run. This is simpler to reason about, but it ties up a connection during LLM calls and can deadlock with pools.
- **Tradeoff:** the compare-and-set approach needs a lease sweeper.

### D6 — Safety architecture (ISSUE-004, ISSUE-009)

- **Recommended:** two-axis risk/decision model; structured action attributes; Diagnostic Check Catalog with `OTHER` fallback; single output gate on all technician-facing generated text; bilingual pattern screen as defence in depth; HIGH requires recorded acknowledgement.
- **Alternatives:**
  - (B) Free-text actions with an LLM safety classifier plus keyword rules. This is more flexible, but not deterministically enforceable.
  - (C) Attributes without a catalog. This is a middle ground: the LLM fills attributes and rules check them, but attribute honesty depends on the LLM.
- **Tradeoff:** the catalog requires curation and limits open-ended suggestions.

### D7 — Embedding model, dimension, local vs remote (ISSUE-011)

- **Recommended:** a multilingual model with Arabic↔English cross-lingual capability, chosen by a 20–30-query retrieval spike; dimension ≤ 2000 for HNSW on `vector`; versioned.
- **Alternatives:** an English-only model plus LLM query translation. This is cheaper, but translation errors compound.
- **Tradeoff:** a local model adds a runtime to the worker but keeps documents on-premises.

### D8 — Lexical and exact retrieval (ISSUE-012)

- **Recommended:** a `chunk_identifiers` exact table + `simple` tsvector + vector, fused by RRF.
- **Alternatives:** `pg_trgm` fuzzy matching (useful for OCR noise, but risks matching wrong codes); BM25 extensions (extra dependency).
- **Tradeoff:** regex patterns are machine-specific configuration.

### D9 — Hypothesis update protocol (ISSUE-006)

- **Recommended:** operation-based updates; application-assigned IDs; immutable per-iteration snapshots.
- **Alternatives:** full-set regeneration plus a diff algorithm. This needs fuzzy matching of descriptions, which is unreliable.
- **Tradeoff:** more complex prompts.

### D10 — Confidence rules (ISSUE-007)

- **Recommended:** LLM proposal capped by a deterministic ceiling table; `confidence_rules_version`.
- **Alternatives:** a fully deterministic score; or LLM-only (rejected by `06 §26`).
- **Tradeoff:** the ceiling counts LLM-asserted links, so semantic validity must be measured offline.

### D11 — Checkpointer integration (ISSUE-016)

- **Recommended:** psycopg 3 as the single driver; checkpointer tables excluded from Alembic autogenerate and created through an idempotent migration step calling `setup()`; `graph_version` per session; rebuild-from-database on mismatch.
- **Alternatives:** asyncpg for the application plus psycopg for the checkpointer (two drivers).
- **Tradeoff:** psycopg 3 async requires a selector event loop on native Windows, which Docker avoids.

### D12 — Evaluation harness (ISSUE-014, ISSUE-015)

- **Recommended:** two conditions (equal information; interactive with a shared deterministic oracle); a "RAG chat agent" interactive baseline; per-run isolated database; canonical labels with a human judge on the holdout; N≥3 repeats.
- **Alternatives:** the one-shot comparison only. This is simpler but confounded.
- **Tradeoff:** more runs and cost. The cost stays small at 20–30 scenarios.

### D13 — Maintenance case provenance and search text (ISSUE-013)

- **Recommended:** `source` enum, nullable session reference with a partial unique index, deterministic search text, labelled `ai_summary`, a `RETRACTED` status, and snapshots of steps and measurements at resolution.
- **Alternatives:** a separate table for imported history. This creates two retrieval paths.

### D14 — Runtime versions and development platform (ISSUE-028)

- **Recommended:** Python 3.12; backend, worker, Postgres (pgvector image) and Redis in Docker Compose on WSL2; ASR native on the Windows host GPU or in a CUDA container; pin all major packages; verify the LangGraph interrupt, durability and checkpointer APIs against the pinned version before Phase 5.
- **Alternatives:** native Windows Python for everything. This requires selector-loop workarounds and is fragile with ARQ, psycopg and uvicorn.

### D15 — ASR placement (ISSUE-023)

- **Recommended:** a separate local process behind `ASRProvider`.
- **Alternatives:** inside the API process (single worker only, threadpool offload).

### D16 — Document deletion and versioning semantics (ISSUE-017)

- **Recommended:** `RETIRED` instead of delete once referenced; chunk sets versioned by processing run with an atomic switch; hash-based deduplication; reconciler.

### D17 — Technician response types (ISSUE-010)

- **Recommended:** a typed response union, including `UNABLE_TO_PERFORM`, plus `ADD_OBSERVATION` at any time.

---

# 8. Overengineering Risks

1. **Multiple hand-written LLM provider adapters** (`09 §45`: Gemini, OpenAI, Anthropic) on top of LangChain. One LangChain-backed adapter behind task ports is enough (ISSUE-019).
2. **Full diagnostic state duplicated** in checkpoint and database (ISSUE-001). This is the most expensive overbuild because it doubles every write path.
3. **Three timeline stores** (messages, events, steps) (ISSUE-026). Make messages a projection.
4. **Relational hypothesis rows with association tables** (`09 §28`). For AI-derived, schema-evolving data queried only per session, immutable JSONB snapshots per iteration are simpler and keep history. Keep evidence, measurements and observations relational.
5. **Glossary as database tables** (`09 §71`). Use configuration files for MVP (`09 §71` already allows this). Make it the default.
6. **Storage abstraction with three backends.** Implement `LocalFileStorage` only. Keep the interface; S3/MinIO is future work.
7. **SSE** is a SHOULD. Polling `GET /diagnostic-sessions/{id}` is sufficient for the MVP with the D4 model.
8. **`06 §59` claim-level support categories** in free text. Either use structured claims on assessments or defer.
9. **Multiple model roles** (`06 §65`). Already marked not for MVP; keep it that way.
10. **Redis + ARQ for a handful of PDFs** is more infrastructure than the workload needs. It is the documented decision and acceptable. The alternative is recorded in ISSUE-028, not recommended as a change.

---

# 9. Underengineering Risks

1. **Safety enforcement** relies on free-text keyword rules and an unspecified "controlled assessment", with unchecked output paths (ISSUE-004).
2. **Concurrency and resume binding** are left to "where practical" (ISSUE-005).
3. **The hypothesis lifecycle** has no update protocol or history (ISSUE-006).
4. **Confidence** has no implementable rule (ISSUE-007).
5. **Numeric interpretation** is delegated to the LLM despite `01 §10` (ISSUE-008).
6. **Exact-identifier retrieval** is required but undesigned (ISSUE-012).
7. **Cross-lingual retrieval** is unaddressed (ISSUE-011).
8. **The ingestion state machine** lacks reconciliation, partial-failure cleanup and versioned chunk sets (ISSUE-017).
9. **The evaluation oracle and judge** are undefined (ISSUE-014).
10. **Maintenance memory** has no provenance, retraction or AI-text separation (ISSUE-013).
11. **Technician non-happy-path responses** (ISSUE-010).
12. **Authorization rules** (ISSUE-025).

---

# 10. LangGraph Review

**Is LangGraph justified?** Yes, but for narrower reasons than the documents claim. The real value is (1) durable interrupt/resume across a physical-world pause of minutes to hours, (2) explicit conditional routing with bounded loops, and (3) an inspectable, testable workflow topology. Tool autonomy is *not* a justification, because tools are graph-constrained (ISSUE-042). That is fine and should be stated plainly. The graph is not decorative: the diagnostic loop, gates and interrupt points are real product transitions.

**Graph state.** It is too thick (ISSUE-001). It should hold workflow control: `pending_action_id`, `iteration`, `run_id`, routing flags, `last_error`, and references. Avoid untyped `dict` fields (`09 §40`). If list channels remain, define reducers explicitly: a default overwrite loses concurrent appends, and an `operator.add` append cannot express updates. With the thin state, most list channels disappear.

**Nodes.** The node list is reasonable. Changes:

- Remove `technician_confirms_resolution` and `save_maintenance_case` (ISSUE-003).
- Add `apply_hypothesis_ops` (deterministic), `interpret_measurements` (deterministic, `SYSTEM_RULE`), `assessment_gate` (deterministic) and an output safety gate.
- Make `prepare_technician_request` template-rendered from the catalog, with an optional LLM "reason" field that is safety-screened.

**Interrupts.** `interrupt()` re-executes its whole node on resume. The node that calls `interrupt()` must therefore contain *no* side effects or LLM calls before the interrupt. Use a dedicated `await_technician` node whose only job is `interrupt(payload)` and returning the resume value, and put validation in the next node. Never put `interrupt()` in a loop inside a node; loop via edges. Resume only with `Command(resume=...)` on the same `thread_id`.

**Resume and concurrency.** OSS LangGraph does not serialise concurrent runs on a thread, so the application must (ISSUE-005). Resume values carry `{input_id, pending_action_id}` *references*; the content is already persisted and validated by the API (consistent with `09 §41`).

**Checkpointing.** Use the Postgres saver with explicit `durability="sync"`; verify against the pinned version. Thread ID = session UUID, with `graph_version` per session (ISSUE-016). Delete checkpoints when the session becomes terminal (ISSUE-041).

**Bounded loops.** Count technician round-trips. Add schema-retry and tool-failure caps per node. Loop exits go through `assess_root_cause` (which may produce `INCONCLUSIVE`) rather than skipping assessment (ISSUE-027).

**Application-state synchronisation.** The service maps the run outcome to status in one transaction with the domain projections. The graph never writes status. On mismatch, the database wins.

**Recommended topology (sketch, for discussion):**

```text
START
 → route_entry                 (deterministic: fault pending? resume? retry?)
 → understand_fault            (LLM; on failure → await_technician[ASK_CLARIFICATION])
 → load_machine_context        (deterministic)
 → retrieve_evidence           (tool: exact + lexical + vector, machine-scoped)
 → propose_hypotheses          (LLM → ops)
 → apply_hypothesis_ops        (deterministic: IDs, evidence validation, confidence ceiling)
 → decide_next_action          (LLM → catalog check | ASSESS | ESCALATE)
 → assessment_gate             (deterministic)
     ├─ check → safety_check (deterministic, catalog attributes)
     │           ├─ BLOCK → decide_next_action (once) | inconclusive
     │           └─ ALLOW/WARN → render_request (template) → await_technician (interrupt only)
     │                 → ingest_input (deterministic, loads persisted input)
     │                 → interpret_measurements (deterministic, SYSTEM_RULE)
     │                 → [retrieve_additional_evidence?] → propose_hypotheses (update mode) …
     ├─ ASSESS → assess_root_cause (LLM) → apply ceiling → output gate
     │           → await_technician[ASSESSMENT_FEEDBACK] (interrupt)
     │                 ├─ reject / new evidence → ingest_input …
     │                 └─ accept → END (status ASSESSMENT_PRESENTED; resolution via API)
     └─ ESCALATE / limit → inconclusive → END (status INCONCLUSIVE)
```

---

# 11. RAG & Evidence Review

- **Ingestion:** the pipeline shape is right. It needs a state machine with `QUEUED`, versioned chunk sets, reconciliation, deduplication and `RETIRED` (ISSUE-017). Page provenance should come from the PDF page index during extraction (one chunk never spans pages; split at page boundaries), so `page_number` is never inferred. OCR stays SHOULD; when used, store OCR text per page with engine and version (`07 §13`) and mark chunks `ocr=true`, because identifier extraction from OCR text is lower-trust.
- **Chunking:** heading-aware chunking within pages. Store `section` from detected headings when available, otherwise null. Never let the LLM assign sections.
- **Retrieval:** exact identifiers, then `simple` tsvector, then vector, with RRF fusion, a hard `machine_id` predicate, an `embedding_version` predicate and a `processing_status = READY` predicate in every query (ISSUE-012). Cross-lingual strategy per ISSUE-011.
- **Machine isolation:** enforced at the repository layer (`09 §23`, correct). Remaining leak paths to close:
  - (1) maintenance-case retrieval must use the same predicate;
  - (2) evidence IDs cited by the LLM must be validated as belonging to the session (ISSUE-006), otherwise a hallucinated ID from another session could resolve;
  - (3) any cache (`09 §145`) must be keyed by `machine_id`;
  - (4) evaluation databases are isolated (ISSUE-015);
  - (5) API endpoints must check that the session's machine matches the path (ISSUE-025).

  Put `machine_id` directly on `document_chunks` (already in `09 §21`) and on `evidence_items`. Add a database `CHECK` or trigger ensuring `evidence_items.machine_id = sessions.machine_id`.
- **Provenance and citations:** citation metadata is copied from database rows into evidence (`09 §88`, correct). The UI renders citations from evidence IDs, never from model text. Model text may only reference evidence IDs. A post-processing step rejects or strips any document name or page pattern in generated text that does not correspond to a cited evidence ID.
- **Retrieval scores:** never shown as confidence (`09 §56`, correct). They are also not comparable across fused lists, so store the rank and the fusion score separately.

---

# 12. Data & Persistence Review

| Data class | Authoritative owner | Notes |
|---|---|---|
| Users, machines, components | PostgreSQL | |
| Documents metadata, status | PostgreSQL | `RETIRED` instead of delete once referenced |
| Original files, audio | Object storage (local FS in MVP) | Server-generated keys; audio deleted after transcription by default |
| Chunks, identifiers, embeddings | PostgreSQL + pgvector | Versioned by processing run and embedding version |
| Sessions, status, leases | PostgreSQL | Single writer: session service |
| Inputs, observations, measurements | PostgreSQL | Immutable; original strings preserved |
| Evidence items, retrieval results | PostgreSQL | Snapshot citation fields |
| Hypothesis snapshots, assessments | PostgreSQL (JSONB snapshots) | AI-derived, labelled |
| Steps, events | PostgreSQL | Events append-only |
| Maintenance cases | PostgreSQL | `source`, `status`, deterministic search text |
| Workflow position | LangGraph checkpoint (Postgres) | Disposable after terminal state |
| Job queue | Redis (ARQ) | Never business truth; reconciler covers loss |
| Evaluation runs, LLM calls | PostgreSQL (isolated database per run) | |

- **JSON vs relational:** keep fault, hypothesis snapshots, assessments and measurement `values` as JSONB. Keep foreign-keyed, queryable entities relational: evidence, measurements, inputs, steps, events, cases, identifiers. `alarm_codes` on cases should be a `text[]` with a GIN index (exact filter, `FR-RAG-010`), not opaque JSON.
- **Missing constraints:**
  - unique `(session_id, input_id)`;
  - partial unique `maintenance_cases(diagnostic_session_id) WHERE NOT NULL`;
  - unique `(machine_id, sha256)` on documents;
  - unique `(session_id, step_number)`;
  - unique `(session_id, iteration)` on snapshots;
  - `evidence_items.machine_id` consistency;
  - `NOT NULL machine_id` everywhere relevant (`09 §126`).
- **Immutable records:** inputs, measurements, observations, events, snapshots, resolved cases (except retraction).
- **pgvector:** sufficient for MVP. The dimension must be fixed before the first chunk migration (D7). An HNSW index is optional at this size; an exact scan is fine for a few thousand chunks.
- **Redis/ARQ:** the role is appropriate and bounded. Enqueue happens post-commit with a reconciler. Use ARQ's `_job_id` (for example `ingest:{document_id}:{processing_run_id}`) to deduplicate enqueues. Jobs must be idempotent because ARQ may run a job more than once.
- **Maintenance memory:** the transaction design in `09 §78–§79` is correct once moved fully out of the graph (ISSUE-003). Index via an outbox. "Case not yet searchable" is a visible, retryable state.

---

# 13. ASR & Fine-Tuning Review

- **Whisper baseline:** large-v3-turbo through faster-whisper is a sound baseline and fits the detected RTX 4060 Laptop GPU (8 GB). Enable VAD filtering to reduce hallucinated text on silence or noise. Treat `language` (forced `ar` vs auto) and `initial_prompt`/`hotwords` (domain vocabulary from the check catalog and glossary) as explicit, recorded baseline variables. These are the cheapest mitigations in the `08 §92` list.
- **Transcript confirmation:** the boundary is correct and well defended across documents. Gaps: confirmation must apply to *every* voice input (ISSUE-021); raw transcripts must be persisted and linked (ISSUE-020); bidi-safe correction UI (ISSUE-024).
- **Technical tokens:** separate exact-match metrics are correct. Operationally, extract critical tokens from the transcript with the same identifier patterns as retrieval (ISSUE-012) and present them as discrete LTR confirmation fields. One extractor then serves both ASR review and retrieval.
- **Dataset methodology:** sound principles (speaker split, error-driven collection, consent). Missing: Arabic text normalization for WER, loanword transcription conventions, and confidence intervals (ISSUE-022).
- **Fine-tuning gate:** conceptually correct, but without pre-registered thresholds it can be rationalised after the fact. Evaluate on the deployed CTranslate2 runtime. Add a general-Arabic regression set (ISSUE-022).
- **LLM fine-tuning policy:** correct and well argued (`08 §84–§89`). No changes.

---

# 14. Safety Review

**Unsafe or ambiguous behaviours found:**

1. LLM-generated request text produced after the safety check (ISSUE-004a).
2. Root-cause, escalation, clarification and explanation text never checked (ISSUE-004b).
3. The physical/non-physical determination is made by the LLM (ISSUE-004c).
4. Deterministic rules over free text are trivially paraphrased around (ISSUE-004d).
5. "Bypassing interlocks" listed as an allowed HIGH action (`03 §29`; ISSUE-004f).
6. HIGH behaviour is a no-op for already-qualified users (ISSUE-004g).
7. No energy-state attribute; the demo's energised measurement and de-energised inspection are indistinguishable to the classifier (ISSUE-004h).
8. Possible streaming of unscreened tokens via SSE (`09 §89`).
9. No response to technician-reported defeated safety functions (ISSUE-035).
10. Safety ranked 6th–13th in priority lists and scheduled after the loop exists (ISSUE-038).

**What is already correct:** no machine-control APIs (`09 §141`); read-only future telemetry (`09 §176`); diagnostic confidence is independent of action safety (`06 §81`, `09 §75`); safety labels not generated by the system under test (`07 §89`); the prohibited-category list (`09 §73`).

**HIGH diagnostic confidence never relaxes action risk:** `06 §81` states this. It is enforceable only if the safety decision function takes no confidence input. Make that an explicit invariant: `safety_check(action) → decision`, with the signature excluding hypotheses and confidence.

**Minimum before implementation:** the decisions in ISSUE-004 (output gate placement, vocabulary, attribute schema or catalog, HIGH acknowledgement behaviour, rule ownership).

---

# 15. Evaluation & Research Validity Review

The evaluation intentions are good: provenance labels, holdout, negative and contradictory cases, memory A/B/C cases, failure cases, and no pre-filled results. As specified, however, the headline comparison would not survive examination:

- **Confounded comparison:** interactive SALLIH vs one-shot baselines (ISSUE-014a). Fix: two conditions and an interactive RAG-agent baseline.
- **Oracle undefined:** leakage risk if an LLM plays the technician (ISSUE-014b). Fix: a deterministic fact table keyed by `check_key`.
- **Scoring undefined:** canonical labels plus a human judge for the holdout (ISSUE-014c).
- **Run isolation:** memory contamination across scenarios (ISSUE-015).
- **Variance:** N≥3 runs, report spread (ISSUE-014d).
- **Construction bias:** author independence, frozen hashed holdout, source-type breakdown (ISSUE-030).
- **Ground-truth correctness:** the F30005 example shows that documented "facts" in scenario drafts must be sourced from the indexed manual (ISSUE-036).
- **ASR:** see Section 13.

With these fixes, a 20–30-scenario single-machine study can credibly support a *case-study* claim: "on this machine and benchmark, structured guided diagnosis vs RAG chat, under equal information and under interactive acquisition". It cannot support a general claim, and the documents already say so (`07 §102`, `08 §58`).

Traceability: `09 §151` and `07 §110` list the right fields. Add the commit SHA, `safety_rules_version`, `confidence_rules_version`, check catalog version, oracle and fixture hashes, and `llm_calls` (ISSUE-043).

---

# 16. Testing Review

The architecture can be made deterministically testable. The provider ports and fakes in `09 §154` are the right idea, but place the fakes at the **task-port** level (ISSUE-019).

| Layer | Approach | Notes |
|---|---|---|
| Unit | Pure functions: state machine, confidence ceiling, hypothesis-op application, measurement rules, safety rules, identifier extraction, normalization | No I/O |
| Safety corpus | Bilingual (MSA, Syrian, English, code-switched) red-team set of paraphrased bypass instructions and negations; expected decision per item | Run in CI; grows over time |
| Repository | Real PostgreSQL + pgvector in Docker (not SQLite: vector types, partial indexes, CAS semantics) | Includes isolation fixture `07 §86` |
| Retrieval | Fixtures with near-identical alarm codes (`F30001` vs `F30005`), hyphenated part numbers, a second machine | Exact-match assertions |
| Graph | `FakeLLM` scripted by task; in-memory saver for routing tests; Postgres saver for resume, restart and `graph_version` tests | Assert the interrupt payload, the database side effects and the absence of duplicate rows on node re-execution |
| Concurrency | Two concurrent `POST /input` calls with the same and different `input_id`; stale `pending_action_id`; cancel during run; lease expiry → retry | These are the tests most likely to catch real bugs |
| API | Contract tests against OpenAPI; authorization (other machine's session, technician uploading a document) | |
| Output gate | Property test: every API response field containing generated text has passed the gate (for example a typed `ScreenedText` wrapper that only the gate can construct) | Makes ISSUE-004 structurally enforced |
| E2E | Playwright: text flow with fake providers; one voice flow with `FakeASR` | Run on the CI profile |
| Evaluation | Separate command, real providers, never in CI | `09 §155` correct |

Normal CI requires Docker (Postgres and Redis) but no paid APIs.

---

# 17. Dependency / Implementation Feasibility

The repository contains no dependency manifests, so no versions are selected. Environment observed on the development machine: Windows 11, Python 3.13.12 (default; 3.12 and 3.11 also installed), Docker 28.5 with Compose v2.40, WSL2 Ubuntu, Node 24, NVIDIA RTX 4060 Laptop GPU with 8 GB.

| Component | Finding | Recommendation |
|---|---|---|
| Python | 3.13 is the machine default; some of the stack (ARQ, CTranslate2 wheels, scientific dependencies) is most reliably supported on 3.12 | Standardise on **3.12** for backend, worker and ASR; pin it in Docker images |
| FastAPI / Pydantic | Pydantic v2 is required by current LangChain/LangGraph | Pin FastAPI and Pydantic v2 together |
| SQLAlchemy / Alembic | SQLAlchemy 2.x async works with psycopg 3 (`postgresql+psycopg`) | Use psycopg 3 for both the application and the checkpointer (one driver). Exclude checkpointer tables from autogenerate |
| PostgreSQL / pgvector | Use the `pgvector/pgvector:pg16` or `pg17` image. HNSW/IVFFlat on `vector` support ≤2000 dims (`halfvec` more) | Fix the embedding dimension before the first chunk migration (D7) |
| LangGraph | `interrupt()` re-runs the whole node on resume; resume uses `Command(resume=…)` with the same `thread_id`; `durability` modes `sync`/`async`/`exit` exist; OSS has no double-texting protection | Pin the version; verify the interrupt, durability and Postgres saver APIs on the pinned version before Phase 5 (CLAUDE.md §45–§46) |
| LangGraph Postgres saver | psycopg 3 async; incompatible with Windows `ProactorEventLoop` | Run the API in Docker/WSL2; if run natively, set a selector loop policy before the loop starts |
| LangChain | Use for chat models, embeddings, splitters. Do not use its vector store for the chunk schema | ISSUE-019 |
| Redis / ARQ | ARQ is in **maintenance-only mode** per its PyPI page; jobs may execute more than once; native-Windows worker behaviour unverified | Keep ARQ, pin it, run the worker in Docker, make jobs idempotent, use `_job_id`, add a reconciler cron. Record a Postgres job table as the fallback |
| faster-whisper / CTranslate2 | GPU requires CUDA 12 + cuDNN 9 builds; CTranslate2 conversions of large-v3-turbo are available | Run ASR as a separate process on the host GPU (or a CUDA container via the NVIDIA container toolkit on WSL2). Pin the CTranslate2 and faster-whisper versions. Measure RTF on the actual laptop |
| Next.js | Node 24 available | Plan HTTPS-on-LAN for tablet microphone and QR (ISSUE-024) |

No hard incompatibility was found that forces replacing a chosen technology. The risks are platform (Windows async, GPU in containers) and maintenance (ARQ), and all are manageable with the recommendations above.

Sources consulted for this section:

- [arq on PyPI (maintenance-only notice)](https://pypi.org/project/arq)
- [LangGraph interrupts documentation](https://docs.langchain.com/oss/python/langgraph/interrupts)
- [LangGraph Durability reference](https://reference.langchain.com/python/langgraph-sdk/schema/Durability)
- [LangSmith double-texting (not available in OSS LangGraph)](https://docs.langchain.com/langsmith/double-texting)
- [psycopg 3 async on Windows](https://www.psycopg.org/psycopg3/docs/_sources/advanced/async.rst.txt)
- [faster-whisper package metadata (CUDA 12 / cuDNN 9)](https://download.hermesedi.duckdns.org/.cache/uv/archive-v0/GJ4vLZYJENBOlg1L/faster_whisper-1.2.1.dist-info/METADATA)
- [Siemens G120 fault codes overview](https://flexa-systems.com/blogs/news/siemens-g120-fault-codes-complete-guide) and [click2electro F30005](https://click2electro.com/forum/siemens-drives/f30005-fault-in-siemens-sinamics-v90-drive/) (F30005 = I²t overload; to be verified against the selected drive's manual)

---

# 18. Recommended Documentation Changes

Not applied in this task. Each requires project-owner approval (CLAUDE.md §7, §37).

| Document | Section | Change | Reason |
|---|---|---|---|
| `03-mvp-scope.md` | §14 | Remove `human_confirmation → save_maintenance_case` from the graph; graph ends at assessment or inconclusive | ISSUE-003 |
| `03-mvp-scope.md` | §29 | Move bypass and safety-circuit modification to BLOCK; adopt the two-axis risk/decision model | ISSUE-004 |
| `03-mvp-scope.md` | §11, §17, §22–§23, §33.3, §44 | Align field names, hypothesis enum, normalization priority, priority order | ISSUE-021, 039, 040, 038 |
| `04-user-flows.md` | §32 | Replace with the canonical status set and reference the transition table | ISSUE-002 |
| `04-user-flows.md` | UF-12, UF-13, UF-14, UF-16, UF-17 | Add unable/partial/free-text responses, stale-action handling, assessment reject, resolve-from-any-active | ISSUE-002, 005, 010 |
| `04-user-flows.md` | UF-18, §38 | Deterministic case search text; labelled AI summary; retraction | ISSUE-013 |
| `04-user-flows.md` | UF-22, UF-23 | `QUEUED`/`RETIRED`, dedup, reconciler, replace semantics | ISSUE-017 |
| `04-user-flows.md`, `02`, `06`, `07` | UF-10, §17, §33 | Correct F30005 semantics or mark as placeholder | ISSUE-036 |
| `05-functional-requirements.md` | FR-DIAG-005, FR-RES-001 | Canonical statuses; resolve preconditions | ISSUE-002 |
| `05-functional-requirements.md` | FR-SAFE-001/002 | Extend to all technician-facing generated content | ISSUE-004 |
| `05-functional-requirements.md` | FR-RAG-008 | Remove `AI_INFERENCE`; add `TECHNICIAN_REPORT`, `SYSTEM_RULE` | ISSUE-029 |
| `05-functional-requirements.md` | FR-HYP-006, FR-HYP-009 | `STRONGLY_SUPPORTED`; confidence enum without `INSUFFICIENT_EVIDENCE` | ISSUE-007, 040 |
| `05-functional-requirements.md` | FR-MEM-001, FR-DOC-011 | Remove "or finalize"; deletion → retire | ISSUE-003, 017 |
| `05-functional-requirements.md` | FR-AUTH-004, FR-DOC-001 | Explicit role and session-access rules | ISSUE-025 |
| `06-ai-system-design.md` | §7–§9 | Thin workflow state; revised topology; no resolution nodes; reject edge | ISSUE-001, 002, 003 |
| `06-ai-system-design.md` | §22–§30 | Hypothesis operation protocol, invariants, snapshots | ISSUE-006 |
| `06-ai-system-design.md` | §24–§26, §86–§87 | Confidence ceiling table; rename sufficiency | ISSUE-007 |
| `06-ai-system-design.md` | §31–§36 | Check catalog reference; typed response union; wire schemas | ISSUE-009, 010, 018 |
| `06-ai-system-design.md` | §37–§40, §80–§81 | Output gate, attributes, HIGH acknowledgement, confidence-free safety signature | ISSUE-004 |
| `06-ai-system-design.md` | §15, §42–§43 | `SYSTEM_RULE` from deterministic measurement interpretation | ISSUE-008 |
| `06-ai-system-design.md` | §85, §49 | Iteration definition; assessment gate; `REQUEST_ASSESSMENT` | ISSUE-027 |
| `06-ai-system-design.md` | §93–§95 | Two evaluation conditions, oracle, judge, repeats | ISSUE-014 |
| `07-data-and-datasets.md` | §24–§27 | Case `source`, nullable session, snapshots, retraction | ISSUE-013 |
| `07-data-and-datasets.md` | §42–§43, §50–§53, §88 | `check_key` references; frozen hashed holdout; safety label vocabulary | ISSUE-009, 030, 004 |
| `07-data-and-datasets.md` | §85 | Create or retarget `10-evaluation-benchmark.md` | ISSUE-037 |
| `08-asr-and-finetuning.md` | §20, §62, §81 | Persisted transcription records; confirmation for all voice input; audio retention default | ISSUE-020, 021 |
| `08-asr-and-finetuning.md` | §13, §31–§33, §41–§44, §120 | WER normalization, loanword conventions, pre-registered thresholds, deployed-runtime eval, regression set | ISSUE-022 |
| `09-system-architecture.md` | §13–§14 | `202` run model; `pending_action_id`; response union; document file/page endpoints; transcription linkage; retry endpoint | ISSUE-005, 010, 020, 032 |
| `09-system-architecture.md` | §25–§33 | Lease/version columns; inputs table; hypothesis snapshots; evidence snapshot fields; case provenance | ISSUE-005, 006, 013, 017 |
| `09-system-architecture.md` | §35–§42, §125, §159 | Single status writer; durability; `graph_version`; checkpointer/Alembic ownership; psycopg 3 | ISSUE-001, 016 |
| `09-system-architecture.md` | §51–§55, §85–§87 | Identifier table + RRF; embedding model ADR | ISSUE-011, 012 |
| `09-system-architecture.md` | §57–§61, §82–§84 | Ingestion state machine, versioned chunk sets, reconciler, dedup | ISSUE-017 |
| `09-system-architecture.md` | §64–§68, §102–§104 | ASR as separate local process; HTTPS-on-LAN; Docker/WSL2 development | ISSUE-023, 024, 028 |
| `09-system-architecture.md` | §72–§75, §89 | Output gate; progress-only SSE | ISSUE-004 |
| `09-system-architecture.md` | §92–§93 | Messages as projection; LLM context from structured state only | ISSUE-026 |
| `09-system-architecture.md` | §112–§115, §153 | Isolated evaluation databases; deterministic oracle | ISSUE-014, 015 |
| `01-product-vision.md`, `06` | §9, §4 | Describe the agent as graph-constrained | ISSUE-042 |

---

# 19. Questions for Product / AI Architect

Each question was checked against all nine documents. None is answered there, and each changes product behaviour or architecture.

**Q1 — Inconclusive sessions.** When SALLIH reaches insufficient evidence or escalation, should the session remain open (technician can add evidence, retry, or later resolve it after fixing the machine with OEM help) or become terminal? If it is later resolved, should it become Maintenance Memory with `probable_root_cause = null`? *(Recommendation: open, resolvable, becomes memory.)*

**Q2 — Resolution timing.** May a technician resolve a session at any active point, including before SALLIH produces a probable cause (for example, they found the loose terminal themselves)? When the technician rejects a presented probable cause, should the loop continue automatically? *(Recommendation: resolve from any active status; rejection returns to the loop.)*

**Q3 — Session ownership and handover.** Can any technician continue or resolve a session another technician started (shift handover)? *(Recommendation: yes, with every input attributed to its actor.)*

**Q4 — Voice during the loop.** Is voice input in MVP limited to the initial fault description, or also available for observations during troubleshooting (always with transcript review)? *(Recommendation: available for any free-text input; measurements stay form-based.)*

**Q5 — Diagnostic Check Catalog.** Do you accept constraining `decide_next_action` to a curated per-machine catalog of checks (with an `OTHER` free-text fallback treated conservatively)? It makes safety, forms, evaluation and the oracle deterministic, at the cost of curation effort and less open-ended suggestions.

**Q6 — Safety ownership and HIGH semantics.** Who owns and reviews the safety rule set (a named qualified electrical or maintenance expert)? For HIGH-risk checks, is "explicit acknowledgement recorded before the instruction is shown" the intended behaviour? Is an energised clamp-meter measurement inside the cabinet (the demo's central check) MEDIUM or HIGH for your target sites?

**Q7 — Historical and synthetic cases in the demo.** Will the demo machine's Maintenance History include seeded synthetic or imported historical cases? If so, are visible "synthetic / imported" labels acceptable in the demo UI? *(Required by the dataset honesty rule.)*

**Q8 — Primary research claim.** Which comparison is the primary claim: equal-information reasoning quality, interactive diagnosis efficiency, or both? Who will judge root-cause equivalence on the holdout set (a named expert), and is that expert available before the holdout is frozen?

**Q9 — Provider and data egress.** Which LLM provider and embedding approach (local vs remote) will MVP use (`03 §36` defers this)? Is sending full manual text to a remote provider acceptable for the demo machine's documents? Should operational audio be deleted immediately after transcription by default?

**Q10 — Document management authority.** Should only an ADMIN role upload, retire and reprocess documents (with technicians read-only)? Is "retire instead of delete once referenced by any session" acceptable?

---

# 20. Final Readiness Recommendation

**NOT READY**

The product definition is stable and coherent. The project is not ready because a competent team would currently have to **invent** significant behaviour in exactly the areas that determine correctness and safety:

- who owns diagnostic state,
- what the legal session transitions are,
- who performs resolution,
- what text is safety-checked,
- how concurrent resumes are prevented,
- how hypotheses and confidence evolve,
- how identifiers and Arabic queries are retrieved,
- how the benchmark is scored without leakage.

**Minimum work before the implementation-readiness review:**

1. Resolve the five BLOCKERs by accepting or amending D1–D6 (ISSUE-001 to ISSUE-005). Record them as ADRs in `docs/reviews/architecture-decisions.md`.
2. Answer Q1–Q6 (they gate D2, D3, D5 and D6).
3. Decide the HIGH items that gate early schema or migrations: D7 (embedding model and dimension, via a small retrieval spike), D8 (identifier retrieval), D9–D10 (hypothesis protocol and confidence rules), D11 (checkpointer and driver), D13 (case provenance), D16 (document lifecycle), D17 (response types).
4. Write the evaluation methodology (`10-evaluation-benchmark.md` or an expanded `07`) covering D12 and Q8.
5. Apply the approved changes to the canonical documents (Section 18), including the F30005 correction and the field-name alignment.
6. Pin dependency versions (Python 3.12, LangGraph, LangChain, FastAPI/Pydantic, SQLAlchemy/psycopg 3, ARQ, faster-whisper/CTranslate2). Verify the LangGraph interrupt, durability and Postgres saver APIs on the pinned versions (a small isolated compatibility experiment is permitted under CLAUDE.md §43).

MEDIUM and LOW issues can be resolved during the relevant implementation phases, provided they are tracked. With items 1–6 done, the project would plausibly qualify as **READY WITH MINOR ISSUES** at the implementation-readiness review.

---

*End of review. No canonical documentation was modified and no application code was written. Awaiting project-owner review of the findings.*
