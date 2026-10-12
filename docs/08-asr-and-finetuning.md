# SALLIH — ASR & Fine-Tuning Strategy

**Document Version:** 1.0  
**Project Stage:** MVP v0.1  
**Status:** Implementation Baseline

---

# 1. Purpose

This document defines the Automatic Speech Recognition and model-adaptation strategy for SALLIH MVP v0.1.

It specifies:

- why voice is important,
- the ASR role inside SALLIH,
- the baseline ASR architecture,
- Syrian Arabic requirements,
- Arabic-English code switching,
- technical-token handling,
- evaluation methodology,
- speech dataset design,
- noise testing,
- error analysis,
- fine-tuning decision criteria,
- ASR adaptation strategy,
- LLM fine-tuning boundaries,
- inference architecture,
- reproducibility,
- and MVP acceptance criteria.

This document must be used together with:

- `01-product-vision.md`
- `02-problem-and-users.md`
- `03-mvp-scope.md`
- `04-user-flows.md`
- `05-functional-requirements.md`
- `06-ai-system-design.md`
- `07-data-and-datasets.md`

The central principle is:

> **Fine-tuning is not a project goal by itself. It is a response to a measured baseline weakness.**

---

# 2. Voice Role in SALLIH

Voice is a first-class interaction mode because industrial maintenance technicians may:

- stand beside machinery,
- wear gloves,
- hold tools,
- work under time pressure,
- operate in noisy environments,
- prefer speaking over typing.

Voice should reduce interaction friction.

The target interaction is:

```text
Technician speaks naturally
        ↓
SALLIH transcribes
        ↓
Technician reviews transcript
        ↓
Confirmed transcript
        ↓
Diagnostic workflow
```

Voice does not replace text.

Text must remain available as a fallback.

---

# 3. ASR Is Not the Diagnostic Agent

The ASR system has one primary responsibility:

> **Convert technician speech into accurate text.**

The ASR model should not:

- diagnose the machine,
- generate hypotheses,
- decide root cause,
- interpret safety,
- modify diagnostic state directly.

Conceptually:

```text
ASR = Hearing Layer

LLM / Diagnostic Orchestrator = Reasoning Layer
```

The architecture must preserve this separation.

---

# 4. ASR Boundary

The correct pipeline is:

```text
Technician Speech
        ↓
Audio Capture
        ↓
ASR
        ↓
Raw Transcript
        ↓
Technician Review
        ↓
Corrected / Confirmed Transcript
        ↓
Technical Normalization
        ↓
Diagnostic Orchestrator
```

The following pipeline is prohibited:

```text
Technician Speech
        ↓
ASR
        ↓
Diagnostic State Automatically Updated
```

ASR output is not authoritative until confirmed.

---

# 5. Baseline ASR Model

The initial SALLIH ASR baseline is:

> **Whisper Large v3 Turbo**

The baseline should be evaluated before any fine-tuning or domain adaptation is performed.

The implementation may use an optimized inference runtime such as:

```text
faster-whisper
CTranslate2
```

when compatible with the selected model and deployment environment.

---

# 6. Why Whisper Large v3 Turbo Is the Initial Baseline

The initial selection is based on the need for:

- multilingual speech recognition,
- Arabic support,
- practical inference performance,
- compatibility with local or controlled deployment,
- a strong starting point before domain adaptation.

The project should not assume that the baseline is already sufficient for Syrian industrial speech.

It must be measured.

---

# 7. ASR MVP Objective

The ASR MVP must answer the following question:

> **Can SALLIH reliably transcribe Syrian Arabic industrial maintenance speech, including technical terminology and Arabic-English code switching, well enough to support diagnostic interaction?**

The objective is not:

> achieving the lowest possible generic Arabic WER.

The objective is:

> preserving diagnostically important information.

---

# 8. Target Speech Domain

The target speech domain is:

```text
Syrian Arabic
+
Industrial Maintenance Vocabulary
+
Arabic-English Code Switching
+
Alarm Codes
+
Part Numbers
+
Measurements
+
Manufacturer Names
+
Factory Noise
```

Examples:

```text
الدرايف عم يفصل بعد عشر دقايق.

الموتور عم يحمي.

الكونتاكتور ما عم يلقط.

الـ PLC ما عم يعطي output.

الحساس ما عم يعطي signal.

عم يعطيني overcurrent fault.

الانفرتر عم يعطي F30005.
```

---

# 9. Syrian Arabic Priority

The priority dialect for the research prototype is:

> **Syrian Arabic**

General Arabic performance is useful but is not sufficient evidence for the target use case.

The evaluation dataset must therefore contain realistic Syrian speech.

Examples should include colloquial forms such as:

```text
عم يفصل
ما عم يلقط
عم يحمي
ما عم يعطي
شغال شوي وبعدين بيفصل
في صوت من جهة الموتور
```

---

# 10. Arabic-English Code Switching

Industrial maintenance language frequently mixes Arabic and English.

Examples:

```text
الـ PLC ما عم يعطي output.

الـ drive عم يفصل.

في مشكلة بالـ sensor.

عم يعطيني overcurrent.

بدنا نفحص الـ contactor.

الـ encoder ما عم يرجع feedback.
```

The ASR benchmark must explicitly test code-switching.

Code-switching performance must not be inferred from Arabic-only results.

---

# 11. Critical Technical Tokens

Some words are disproportionately important for diagnosis.

Critical token classes include:

```text
Alarm Codes
Part Numbers
PLC Tags
Component Names
Manufacturer Names
Electrical Phase Labels
Units
Measurements
Technical Abbreviations
```

Examples:

```text
F30005
A0501
K17
L1
L2
L3
PLC
VFD
HMI
24V
8.2 A
Siemens
Schneider
ABB
Omron
6SL3210-1KE14-3UF2
```

---

# 12. Critical Token Principle

A transcription may have acceptable general WER while still being unsafe or diagnostically useless.

Example:

Ground truth:

```text
F30005
```

ASR output:

```text
F3005
```

The difference may appear small in overall WER.

But diagnostically:

> the token may identify a different fault or no valid fault at all.

Therefore WER alone is insufficient.

---

# 13. ASR Evaluation Metrics

The ASR evaluation must include multiple metrics.

Minimum metrics:

```text
Word Error Rate
Technical Term Accuracy
Technical Term Error Rate
Alarm Code Exact Match Accuracy
Part Number Exact Match Accuracy
Measurement Transcription Accuracy
Code-Switch Accuracy
```

Where noise conditions exist:

```text
Clean WER
Noisy WER
```

should be compared separately.

---

# 14. Word Error Rate

WER is defined conceptually as:

```text
WER =
(Substitutions + Deletions + Insertions)
/
Reference Words
```

WER is useful for overall transcription quality.

It must not be the only metric reported.

---

# 15. Technical Term Accuracy

Technical Term Accuracy measures whether important industrial terms are correctly preserved.

Example target terms:

```text
VFD
PLC
contactor
relay
sensor
encoder
overcurrent
overvoltage
Siemens
Schneider
```

Conceptually:

```text
Technical Term Accuracy =
Correct Technical Tokens
/
Total Technical Tokens
```

---

# 16. Alarm Code Accuracy

Alarm codes should use exact-match evaluation.

Example:

Reference:

```text
F30005
```

Prediction:

```text
F3005
```

Result:

```text
INCORRECT
```

Partial similarity is not enough.

---

# 17. Part Number Accuracy

Part numbers should also use exact-match evaluation.

Example:

Reference:

```text
6SL3210-1KE14-3UF2
```

Prediction:

```text
6SL3210-1KE14-3UFZ
```

Result:

```text
INCORRECT
```

One character may refer to a different component.

---

# 18. Measurement Accuracy

The system should evaluate whether spoken technical values remain correct.

Example:

Speech:

```text
الفاز الثالث عم يسحب 13.9 أمبير.
```

Expected structured value:

```text
L3 = 13.9 A
```

Errors such as:

```text
13.9 → 30.9
```

may materially affect diagnosis.

---

# 19. Code-Switch Accuracy

Evaluation should separately inspect English technical terms embedded in Arabic speech.

Example:

Reference:

```text
الـ PLC ما عم يعطي output.
```

Potential failure:

```text
البي ال سي ما عم يعطي أوتبوت
```

This may still be acceptable if downstream normalization correctly maps the transcription.

Therefore two layers should be distinguished:

```text
Literal ASR Accuracy
Technical Interpretation Accuracy
```

---

# 20. Raw Transcript vs Technical Normalization

SALLIH should preserve:

## Raw ASR Transcript

Example:

```text
البي ال سي ما عم يعطي اوتبوت
```

## Confirmed Transcript

Example after technician review:

```text
الـ PLC ما عم يعطي output
```

## Normalized Technical Representation

Example:

```text
PLC output missing
```

These are three different artifacts.

They must not be silently merged.

---

# 21. Technical Normalization Is Not ASR Fine-Tuning

Some ASR errors can be handled more effectively using a downstream normalization layer.

Example:

```text
كونتاكتر
كونتاكتور
contactor
```

may normalize to:

```text
CONTACTOR
```

This does not necessarily require ASR fine-tuning.

The project should first determine whether errors are:

```text
speech recognition errors
```

or:

```text
terminology normalization issues
```

before deciding to fine-tune.

---

# 22. Baseline-First Strategy

The required process is:

```text
Whisper Baseline
        ↓
Evaluation Dataset
        ↓
Measure Performance
        ↓
Analyze Errors
        ↓
Classify Failure Types
        ↓
Choose Cheapest Reliable Fix
        ↓
Re-evaluate
```

Possible fixes include:

```text
Transcript confirmation
Technical glossary
Normalization
Prompt/context biasing where supported
Post-processing
Fine-tuning
```

Fine-tuning is not automatically the first fix.

---

# 23. Baseline Evaluation Conditions

The baseline should be evaluated under multiple conditions.

Minimum categories:

```text
Clean Syrian Arabic
Clean Code-Switched Speech
Technical Vocabulary
Alarm Codes
Measurements
Manufacturer Names
Synthetic Factory Noise
```

If available:

```text
Real Factory Noise
```

should be evaluated separately.

---

# 24. Initial Evaluation Dataset

The initial ASR evaluation dataset should contain representative utterances across all critical categories.

Suggested composition categories:

```text
Natural Fault Descriptions
Technical Components
Alarm Codes
Measurements
Code-Switching
Manufacturer Names
Part Numbers
Short Commands / Observations
```

The exact number of samples should be determined by available speakers and recording quality.

Quality and diversity are more important than an arbitrary sample count.

---

# 25. Initial Evaluation Target

A practical initial target may be:

```text
100–300 utterances
```

for baseline analysis.

This is a development target, not a claim of statistical completeness.

If resources permit, the dataset should expand.

---

# 26. Speaker Diversity

The evaluation dataset should preferably include multiple speakers.

Useful diversity dimensions:

```text
Male / Female where available
Different ages
Different speaking speeds
Different Syrian regional accents where practical
Technical / non-technical speakers
Different microphone conditions
```

The goal is to avoid overfitting evaluation to one voice.

---

# 27. Speaker IDs

Use pseudonymous identifiers:

```text
SPK-001
SPK-002
SPK-003
```

Do not require personal names.

---

# 28. Recording Requirements

Each recording should ideally preserve metadata such as:

```text
sample_id
speaker_id
audio_path
ground_truth_transcript
duration
noise_condition
technical_terms
alarm_codes
part_numbers
dataset_split
```

---

# 29. Audio Format

A consistent storage format should be used.

Recommended development representation:

```text
WAV
Mono
16 kHz
PCM
```

or another ASR-compatible format if required by the selected implementation.

Original recordings may be preserved separately when appropriate.

---

# 30. Ground-Truth Transcription

Ground-truth transcripts should represent what was actually spoken.

Example spoken phrase:

```text
الدرايف عم يفصل بعد عشر دقايق
```

Ground truth should not automatically become formal Arabic such as:

```text
يتوقف محرك التردد المتغير بعد عشر دقائق
```

because that would change the speech content.

---

# 31. Ground-Truth Consistency

The project should define transcription conventions.

Examples:

```text
How English words are written
How numbers are represented
How alarm codes are written
How fillers are treated
How repeated words are treated
```

Without consistent annotation, WER results become unreliable.

---

# 32. Suggested Transcription Conventions

Initial conventions:

## English Technical Terms

Use their canonical technical form where clearly spoken.

Example:

```text
PLC
VFD
output
sensor
overcurrent
```

---

## Alarm Codes

Use exact canonical representation.

Example:

```text
F30005
```

---

## Numbers

Preserve semantic numeric value consistently.

Example:

Spoken:

```text
تلاتطعش فاصلة تسعة
```

Transcript representation:

```text
13.9
```

if the project chooses numeric normalization for evaluation.

The convention must remain consistent across the dataset.

---

# 33. Dual Transcript Strategy

Where useful, maintain two transcript fields:

```text
verbatim_transcript
normalized_transcript
```

Example:

```yaml
verbatim_transcript:
  "البي ال سي ما عم يعطي اوتبوت"

normalized_transcript:
  "الـ PLC ما عم يعطي output"
```

WER should specify which transcript representation is being used.

---

# 34. Noise Conditions

The dataset should distinguish noise environments.

Possible categories:

```text
CLEAN
LOW_NOISE
MODERATE_FACTORY_NOISE
HIGH_FACTORY_NOISE
SYNTHETIC_NOISE
REAL_FACTORY_NOISE
```

Not all categories must exist in MVP v0.1.

---

# 35. Synthetic Noise

Synthetic factory noise may be mixed into clean recordings for controlled experiments.

Possible sources:

```text
motor hum
fans
compressors
conveyors
general workshop ambience
```

Noise level should vary.

Example:

```text
SNR 20 dB
SNR 10 dB
SNR 5 dB
```

if the research methodology chooses signal-to-noise-based evaluation.

---

# 36. Real Noise vs Synthetic Noise

Results must distinguish:

```text
Synthetic Factory Noise
```

from:

```text
Real Factory Recording
```

Synthetic augmentation must not be described as proof of real factory robustness.

---

# 37. Baseline Experiment

The first official ASR experiment should be:

```text
Model:
Whisper Large v3 Turbo

Dataset:
Initial Syrian Industrial ASR Test Set

Conditions:
Clean
Code-Switched
Technical Tokens
Synthetic Noise
```

Record:

```text
WER
Technical Term Accuracy
Alarm Code Accuracy
Part Number Accuracy
Measurement Accuracy
Code-Switch Accuracy
Latency
```

---

# 38. Error Analysis

After the baseline, every important error should be categorized.

Possible categories:

```text
DIALECT_ERROR
TECHNICAL_TERM_ERROR
CODE_SWITCH_ERROR
ALARM_CODE_ERROR
PART_NUMBER_ERROR
NUMBER_ERROR
MANUFACTURER_ERROR
NOISE_ERROR
SEGMENTATION_ERROR
HALLUCINATED_WORD
DELETION_ERROR
```

---

# 39. Error Analysis Example

Example:

Reference:

```text
الدرايف عم يعطي F30005.
```

Prediction:

```text
الدرايف عم يعطي F3005.
```

Classify:

```text
ALARM_CODE_ERROR
```

Severity:

```text
CRITICAL
```

Another example:

Reference:

```text
الموتور عم يحمي.
```

Prediction:

```text
الموتور عم يسخن.
```

This may be:

```text
DIALECT_ERROR
```

and potentially diagnostically significant.

---

# 40. Error Severity

ASR errors should also be categorized by impact.

Suggested categories:

```text
LOW
MEDIUM
HIGH
CRITICAL
```

## LOW

Grammar or wording error that preserves meaning.

## MEDIUM

Meaning degraded but likely recoverable.

## HIGH

Technical meaning changes.

## CRITICAL

Error may cause incorrect diagnostic direction.

Examples:

```text
F30005 → F3005
13.9 A → 30.9 A
L3 → L2
```

---

# 41. ASR Fine-Tuning Decision Gate

Fine-tuning should occur only if the baseline shows meaningful domain weakness.

The decision should consider:

```text
General WER
Technical-token performance
Critical-error rate
Noise performance
Downstream diagnostic impact
```

---

# 42. Fine-Tuning Trigger

Fine-tuning is justified when:

```text
Baseline errors repeatedly affect
critical industrial terminology
or Syrian dialect understanding
or downstream diagnostic state
```

and simpler mitigations are insufficient.

---

# 43. Fine-Tuning Non-Trigger

Fine-tuning should not be performed merely because:

```text
WER is not perfect
```

or:

```text
fine-tuning looks impressive academically
```

If transcript confirmation and normalization solve the meaningful errors reliably, fine-tuning may provide little practical value.

---

# 44. Fine-Tuning Decision Matrix

Conceptual decision table:

| Baseline Result | Recommended Action |
|---|---|
| Good general WER, good technical tokens | Do not fine-tune |
| Good WER, weak terminology variants | Try glossary / normalization first |
| Good clean speech, weak noise | Add noise robustness strategy |
| Weak Syrian dialect transcription | Consider dialect-focused adaptation |
| Weak alarm/part-number accuracy | Add targeted token strategy and training data |
| Persistent critical domain errors | Fine-tuning justified |
| Very small dataset only | Delay fine-tuning |

---

# 45. ASR Adaptation Priority

If adaptation is needed, priority should be:

```text
1. Industrial technical vocabulary
2. Syrian dialect patterns
3. Arabic-English code switching
4. Alarm and part-number robustness
5. Factory noise robustness
```

The exact priority should be driven by measured errors.

---

# 46. Fine-Tuning Dataset Design

Fine-tuning data should deliberately contain the failure patterns found during baseline evaluation.

Example:

Baseline repeatedly fails on:

```text
كونتاكتور
overcurrent
F30005
Schneider
```

Then training data should intentionally include varied examples containing those tokens.

This is:

> **error-driven data collection**

rather than random data accumulation.

---

# 47. Fine-Tuning Data Categories

Possible training categories:

```text
Natural Syrian Maintenance Speech
Technical Component Vocabulary
Arabic-English Code Switching
Alarm Codes
Part Numbers
Measurements
Manufacturer Names
Fault Descriptions
Industrial Noise Variants
```

---

# 48. Training Data Diversity

Training data should avoid excessive repetition from one speaker.

Useful variation includes:

```text
multiple speakers
multiple speaking speeds
different microphones
different noise conditions
different sentence contexts
```

---

# 49. Train / Validation / Test Separation

Fine-tuning requires strict dataset separation:

```text
TRAIN
VALIDATION
TEST
```

The test set must not participate in:

- training,
- threshold tuning,
- prompt tuning,
- manual optimization.

---

# 50. Speaker-Level Split

Where possible:

```text
Train Speakers
≠
Test Speakers
```

This better measures generalization.

Example:

```text
SPK-001 → Train
SPK-002 → Train
SPK-003 → Validation
SPK-004 → Test
SPK-005 → Test
```

Actual split depends on speaker availability.

---

# 51. Avoid Utterance Leakage

Very similar recordings should not be split across train and test in a way that makes the benchmark trivial.

Example:

Train:

```text
الدرايف عم يعطي F30005.
```

Test:

```text
الدرايف عم يعطي F30005.
```

recorded by the same speaker under the same conditions.

This provides weak evidence of generalization.

---

# 52. Fine-Tuning Experiment Naming

Each adaptation experiment should have a stable identifier.

Example:

```text
ASR-BASELINE-001
ASR-FT-001
ASR-FT-002
```

---

# 53. Fine-Tuning Run Metadata

Record at least:

```text
experiment_id
base_model
dataset_version
train_split
validation_split
training_configuration
checkpoint
timestamp
hardware
evaluation_dataset_version
```

---

# 54. Model Versioning

Fine-tuned models must be versioned.

Example:

```text
sallih-whisper-syrian-industrial-v0.1
sallih-whisper-syrian-industrial-v0.2
```

Do not overwrite model versions without preserving evaluation lineage.

---

# 55. Fine-Tuning Success Criteria

A fine-tuned model should not be considered better solely because overall WER decreases.

It should improve one or more target metrics such as:

```text
Technical Term Accuracy
Alarm Code Accuracy
Part Number Accuracy
Code-Switch Accuracy
Noisy Speech Accuracy
Critical Error Rate
```

without causing unacceptable regression elsewhere.

---

# 56. Required Comparison

If fine-tuning is performed, the project must compare:

```text
Base Whisper Large v3 Turbo
vs
SALLIH Adapted ASR
```

using the same unseen test set.

---

# 57. Fine-Tuning Evaluation Table

Example reporting format:

| Metric | Base | Adapted | Difference |
|---|---:|---:|---:|
| WER | TBD | TBD | TBD |
| Technical Term Accuracy | TBD | TBD | TBD |
| Alarm Code Accuracy | TBD | TBD | TBD |
| Part Number Accuracy | TBD | TBD | TBD |
| Code-Switch Accuracy | TBD | TBD | TBD |
| Critical Error Rate | TBD | TBD | TBD |

All numbers must come from real experiment results.

Do not pre-fill fictional improvements.

---

# 58. Statistical Honesty

If the evaluation set is small, the final report must not overstate performance.

Example:

Preferred:

```text
On our 180-utterance held-out industrial test set...
```

Not:

```text
The model reliably understands all Syrian industrial speech.
```

---

# 59. Downstream Evaluation

ASR quality must also be evaluated by its impact on the diagnostic system.

Important question:

> Did the transcription preserve enough technical meaning for SALLIH to build the correct diagnostic state?

---

# 60. Downstream Structured Extraction Accuracy

Possible experiment:

```text
Ground-Truth Transcript
        ↓
Fault Understanding
```

versus:

```text
ASR Transcript
        ↓
Fault Understanding
```

Compare extraction of:

```text
component
symptom
alarm code
measurement
timing
```

This measures practical ASR impact.

---

# 61. End-to-End Voice Diagnostic Evaluation

An advanced MVP evaluation may compare:

```text
Text Input
vs
Voice Input
```

for the same diagnostic scenarios.

Measure:

```text
Structured fault accuracy
Root-cause accuracy
Diagnostic path
Additional clarification requests
ASR-caused failures
```

This reveals whether ASR errors materially degrade SALLIH.

---

# 62. Transcript Confirmation as Safety Mechanism

Transcript confirmation is an intentional reliability layer.

Even after fine-tuning:

> technician confirmation remains required for critical initial voice input in MVP v0.1.

Fine-tuning must not remove this safeguard.

---

# 63. Critical Token Highlighting

Where practical, the UI should detect and emphasize possible critical tokens.

Examples:

```text
F30005
13.9 A
K17
VFD
Siemens
```

This encourages technician review.

---

# 64. Low-Confidence ASR Handling

If the ASR implementation exposes confidence or uncertainty signals, the UI may use them cautiously.

Example:

```text
Possible alarm code:
F30005

Please verify.
```

However ASR confidence values must not be treated as perfectly calibrated.

---

# 65. Unknown Technical Token Handling

If ASR produces an uncertain token:

```text
F thirty thousand five
```

instead of guessing, the interface may ask:

```text
Please confirm the alarm code.
```

This is preferable to silently normalizing an uncertain identifier.

---

# 66. Vocabulary Support

A project-specific technical vocabulary should be maintained.

Possible entries:

```yaml
- canonical: VFD
  variants:
    - drive
    - درايف
    - inverter
    - انفرتر

- canonical: CONTACTOR
  variants:
    - contactor
    - كونتاكتور
    - كونتاكتر

- canonical: PLC
  variants:
    - PLC
    - بي ال سي
```

This vocabulary supports downstream normalization.

It may also support ASR adaptation strategies where technically possible.

---

# 67. Demo-Machine Vocabulary

The initial technical vocabulary should prioritize the selected demo machine.

Example:

If the machine uses:

```text
Siemens G120
S7 PLC
photoelectric sensors
contactors
```

then those terms should receive priority.

The MVP should not attempt to build a universal industrial dictionary.

---

# 68. Manufacturer Vocabulary

Potential manufacturer terms include:

```text
Siemens
Schneider Electric
ABB
Omron
Mitsubishi
Delta
Allen-Bradley
SEW
Festo
Bosch Rexroth
```

Only terms relevant to the chosen machine need prioritized evaluation.

---

# 69. Alarm Vocabulary

Alarm codes should preferably be extracted from the selected machine's documentation.

This produces a machine-specific alarm vocabulary.

Example:

```text
F30005
F30001
A0501
```

The exact codes depend on real documentation.

---

# 70. Number Handling

Arabic speech may express numbers in several ways.

Examples:

```text
تلاتة عشر فاصلة تسعة

13.9

تلاتعش وتسعة أعشار
```

The ASR/normalization pipeline should preserve intended numeric meaning where possible.

Critical measurements should still be confirmed through structured input when needed.

---

# 71. Measurement Voice Input

MVP v0.1 does not need to depend on voice for all numerical measurements.

Preferred pattern:

```text
Agent asks for measurement
↓
Frontend displays structured form
↓
Technician enters numeric values
```

Voice measurement entry may be added later.

This reduces risk from number transcription errors.

---

# 72. Local Inference Strategy

The ASR architecture should allow local inference.

Conceptually:

```text
Browser / Client
      ↓
Audio Upload
      ↓
FastAPI ASR Endpoint
      ↓
Local ASR Runtime
      ↓
Transcript
```

This avoids requiring audio to pass through the diagnostic LLM provider.

---

# 73. ASR Service Separation

ASR should be implemented as a dedicated service/module boundary.

Conceptual interface:

```python
class ASRProvider:
    transcribe(audio) -> ASRResult
```

This allows future replacement or comparison.

---

# 74. ASR Result Schema

Conceptual result:

```python
ASRResult = {
    "transcript": str,
    "language": str | None,

    "segments": list | None,

    "duration_seconds": float,

    "model_name": str,
    "model_version": str | None,

    "processing_time_ms": int | None
}
```

Optional:

```text
token confidence
segment confidence
```

if reliably provided by the selected engine.

---

# 75. ASR Provider Abstraction

The system should be capable of evaluating alternative ASR implementations later.

Potential future baselines may include:

```text
Whisper variants
Arabic-specific ASR models
Multilingual speech models
Commercial ASR APIs
```

MVP should begin with one strong baseline.

---

# 76. Do Not Over-Engineer ASR Providers

Provider abstraction does not mean implementing five ASR systems immediately.

MVP requirement:

```text
One working baseline
+
Clean abstraction
```

Additional providers should be introduced only for evaluation or clear need.

---

# 77. Hardware-Aware Inference

The project should measure practical inference behavior such as:

```text
Model load time
Audio processing time
VRAM usage
RAM usage
Real-time factor
```

This matters because the intended prototype may run locally.

---

# 78. Real-Time Factor

A useful ASR runtime metric is:

```text
RTF =
processing time
/
audio duration
```

Example:

```text
10-second audio
processed in 5 seconds

RTF = 0.5
```

Lower values indicate faster-than-real-time transcription.

---

# 79. Latency Target

The MVP should aim for a user experience where a short fault description is transcribed within a practical interactive delay.

No strict production SLA should be claimed before measurement.

Latency must be benchmarked on the actual deployment hardware.

---

# 80. Streaming ASR

Streaming transcription is not required for MVP v0.1.

The initial interaction may use:

```text
Record
↓
Stop
↓
Transcribe
```

This simplifies:

- implementation,
- transcript review,
- evaluation.

Streaming may be future scope.

---

# 81. Audio Retention

The project must define whether recorded audio is retained.

Recommended MVP default:

```text
Audio may be stored temporarily for transcription.
```

Long-term retention should occur only when:

- needed for research,
- user consent exists,
- privacy requirements are satisfied.

---

# 82. Research Audio Storage

Speech samples collected for the research dataset must be stored separately from ordinary diagnostic audio.

Conceptually:

```text
Operational Voice Input
≠
Research Speech Dataset
```

Using technician production audio for training should never happen silently.

---

# 83. Consent for Training Use

If human voice recordings are used for ASR training or fine-tuning, contributors should know:

- that recordings are collected,
- that they may be used for model improvement,
- whether they may be published,
- how they will be stored.

Applicable university ethics procedures must be respected.

---

# 84. LLM Fine-Tuning Policy

SALLIH should not initially fine-tune the diagnostic LLM.

The starting approach is:

```text
Strong Base LLM
+
Structured Prompts
+
Machine RAG
+
LangGraph
+
Tools
+
Structured Outputs
```

LLM fine-tuning should be considered only after evaluating this system.

---

# 85. Why LLM Fine-Tuning Is Not Initial Priority

The main MVP challenges are expected to involve:

```text
machine grounding
retrieval
diagnostic workflow
human evidence
safety
state management
```

These problems are not automatically solved by LLM fine-tuning.

Fine-tuning an LLM before measuring these components may hide architecture problems.

---

# 86. LLM Fine-Tuning Trigger

LLM fine-tuning may become justified if evaluation shows a persistent, measurable weakness such as:

```text
failure to understand Syrian industrial expressions
```

despite:

```text
good ASR
+
technical normalization
+
few-shot prompting
+
structured schemas
```

or:

```text
repeated failure to follow domain-specific output behavior
```

that cannot be adequately addressed through architecture or prompting.

---

# 87. LLM Fine-Tuning Non-Trigger

Do not fine-tune the LLM merely to improve:

```text
machine-specific factual knowledge
```

Machine-specific knowledge belongs primarily in:

```text
RAG
Machine Memory
Structured Data
```

not model weights.

---

# 88. ASR vs LLM Fine-Tuning Priority

If the project requires model adaptation, the preferred order is:

```text
1. Evaluate ASR
2. Adapt ASR if required
3. Evaluate diagnostic LLM with correct transcripts
4. Use prompting / glossary / RAG improvements
5. Fine-tune LLM only if a persistent language/reasoning gap remains
```

Therefore:

> **ASR adaptation has higher initial priority than LLM fine-tuning.**

---

# 89. Why Machine Knowledge Must Not Be Fine-Tuned Into LLM

Machine-specific data changes.

Examples:

```text
new maintenance case
new machine revision
new manual
new component
```

If this knowledge is stored in model weights:

- updates become expensive,
- provenance is lost,
- citations become difficult,
- stale knowledge becomes dangerous.

Therefore machine knowledge belongs in external retrieval systems.

---

# 90. Fine-Tuning Contribution to Graduation Research

Fine-tuning should only be presented as a meaningful research contribution if the project can demonstrate:

```text
Clear baseline weakness
+
Domain-specific dataset
+
Controlled fine-tuning experiment
+
Held-out evaluation
+
Measured improvement
+
Ablation/comparison
```

Simply fine-tuning a model is not sufficient research contribution.

---

# 91. ASR Experiment Pipeline

The complete experiment pipeline should be:

```text
Prepare Evaluation Dataset
        ↓
Run Base Whisper
        ↓
Calculate Metrics
        ↓
Perform Error Analysis
        ↓
Identify Critical Failure Classes
        ↓
Try Non-Fine-Tuning Mitigations
        ↓
Re-evaluate
        ↓
If Still Weak:
Collect Targeted Training Data
        ↓
Fine-Tune / Adapt
        ↓
Evaluate on Same Held-Out Test
        ↓
Compare
```

---

# 92. Non-Fine-Tuning Mitigations

Before ASR fine-tuning, the project should consider:

```text
Technical normalization
Critical-token confirmation
Glossary
Better audio segmentation
Noise preprocessing
Prompt/context features if supported
Exact alarm-code correction workflow
Structured measurement entry
```

Any mitigation must be tested, not assumed effective.

---

# 93. Ablation Experiments

If time permits, useful ASR ablations include:

```text
Base ASR

Base ASR
+ Technical Normalization

Base ASR
+ Transcript Confirmation

Adapted ASR

Adapted ASR
+ Technical Normalization
```

This reveals which component produces the actual improvement.

---

# 94. Critical Error Rate

The project should define a metric for diagnostically dangerous transcription errors.

Conceptually:

```text
Critical Error Rate =
Utterances containing at least one critical technical error
/
Total evaluated utterances
```

Critical errors may include:

```text
wrong alarm code
wrong part number
wrong numeric measurement
wrong phase label
wrong critical component
```

---

# 95. Diagnostic Preservation Metric

A useful project-specific metric may be:

> **Diagnostic Information Preservation Rate**

This measures whether the ASR transcript preserves the key information required for fault structuring.

Possible required fields:

```text
component
symptom
alarm code
timing
measurement
```

This metric must be clearly defined before use.

---

# 96. Example Diagnostic Preservation

Reference:

```text
الدرايف عم يفصل بعد عشر دقايق وبيعطيني F30005.
```

ASR:

```text
الدرايف عم يفصل بعد عشر دقايق وبيعطيني F30005.
```

Preserved:

```text
component = yes
symptom = yes
timing = yes
alarm code = yes
```

Another ASR result:

```text
الدرايف عم يفصل بعد عشر دقايق.
```

Preserved:

```text
component = yes
symptom = yes
timing = yes
alarm code = no
```

This may be diagnostically more meaningful than raw WER alone.

---

# 97. ASR Evaluation Script Requirements

Evaluation tooling should be capable of:

```text
Load dataset manifest
Run ASR model
Store predictions
Calculate WER
Calculate token metrics
Calculate critical error metrics
Produce CSV / JSON results
```

Results should be reproducible.

---

# 98. Prediction Storage

Each test prediction should preserve:

```text
sample_id
reference_transcript
predicted_transcript
model
model_version
dataset_version
latency
technical_token_results
error_categories
```

---

# 99. Fine-Tuning Dataset Versioning

Example versions:

```text
sallih-syrian-industrial-speech-v0.1
sallih-syrian-industrial-speech-v0.2
```

Each version should include a manifest describing:

```text
number of samples
speakers
duration
noise distribution
technical token distribution
train/validation/test split
```

---

# 100. Dataset Growth Strategy

Dataset growth should be error-driven.

Example:

```text
Baseline shows:
poor contactor vocabulary
↓
Collect more contactor utterances

Baseline shows:
good clean speech,
poor compressor noise performance
↓
Collect / augment noisy speech

Baseline shows:
poor part-number accuracy
↓
Collect targeted alphanumeric examples
```

---

# 101. Avoid Artificial Benchmark Inflation

Do not create hundreds of nearly identical phrases merely to increase sample count.

Example:

```text
الدرايف عم يفصل
الدرايف عم يفصل هلق
الدرايف عم يفصل حاليا
```

These may provide limited additional diversity if all recorded by the same speaker under identical conditions.

---

# 102. Evaluation Fairness

When comparing Base and Fine-Tuned ASR:

- use the same held-out test set,
- use the same normalization rules,
- use the same metric implementation,
- document decoding settings,
- document audio preprocessing.

---

# 103. Reproducibility

Each official ASR result should record:

```text
experiment_id
model_name
model_version
dataset_version
test_split
runtime
hardware
decoding_configuration
normalization_configuration
timestamp
```

---

# 104. ASR Logging

Operational ASR logs may record:

```text
request_id
session_id
audio_duration
model
processing_latency
language
error status
```

Avoid unnecessary logging of raw audio content.

---

# 105. Failure Handling

Possible ASR failures:

```text
No speech detected
Audio too short
Audio unreadable
Model unavailable
Out of memory
Transcription timeout
Unsupported audio
```

The user-facing response should remain recoverable.

Example:

```text
SALLIH could not transcribe this recording.

Try again or describe the fault using text.
```

---

# 106. No-Speech Handling

If no useful speech is detected:

```text
Do not submit an empty fault description.
```

Instead:

```text
No speech was detected.
Please record again or type the issue.
```

---

# 107. ASR Timeout

A transcription timeout must not:

- create a diagnostic session failure,
- insert partial corrupted transcript,
- destroy prior state.

The user should be able to retry.

---

# 108. Resource Failure

If local inference fails due to memory or device constraints:

```text
ASR service error
↓
Preserve application state
↓
Offer retry / text fallback
```

---

# 109. ASR Service Startup

If the model requires significant loading time, the application architecture should avoid loading it separately for every request.

Preferred pattern:

```text
ASR service starts
↓
Model loaded
↓
Multiple requests processed
```

subject to deployment constraints.

---

# 110. Model Resource Monitoring

During development, measure:

```text
VRAM
RAM
Model load time
Average transcription latency
Peak resource usage
```

This informs deployment decisions.

---

# 111. Development Hardware Strategy

Local ASR development should prefer a model/runtime combination that is practical on the available development environment.

The project should not require an unnecessarily expensive cloud deployment merely to demonstrate voice interaction.

Cloud GPU use remains an option for fine-tuning experiments if local hardware is insufficient.

---

# 112. Fine-Tuning Compute Strategy

Fine-tuning infrastructure should be selected only after:

```text
dataset exists
+
baseline proves fine-tuning is justified
```

Possible options:

```text
Local GPU for small experiments
University GPU
Cloud GPU
Notebook environment
```

The project should avoid committing to infrastructure before knowing the training workload.

---

# 113. Fine-Tuning Reproducibility

Training configuration should preserve:

```text
learning rate
batch size
gradient accumulation
number of epochs
optimizer
scheduler
precision
seed
checkpoint selection
```

when applicable to the chosen fine-tuning framework.

---

# 114. Overfitting Detection

Potential signs of overfitting:

```text
Training loss improves
but validation worsens

Known speakers perform well
but unseen speakers fail

Technical terms seen in training work
but novel combinations fail
```

The project should monitor validation performance.

---

# 115. Fine-Tuning Stopping Rule

Do not automatically train for a fixed large number of epochs.

Training should use validation behavior to determine when additional training no longer improves generalization.

The exact method depends on implementation.

---

# 116. Fine-Tuned Model Deployment

If adapted ASR clearly outperforms baseline, the deployed ASR interface should not change.

Conceptually:

```python
ASRProvider.transcribe(...)
```

remains stable.

Only the configured model changes.

---

# 117. Model Rollback

The system should make it possible to return to the baseline model if a fine-tuned model introduces regression.

Model versions must therefore remain identifiable.

---

# 118. Research Experiment Set

The ASR chapter of the graduation project may include:

## Experiment A

```text
Whisper Large v3 Turbo
on Syrian Industrial Speech
```

## Experiment B

```text
Base ASR + Technical Normalization
```

## Experiment C

If justified:

```text
Fine-Tuned / Adapted Whisper
```

## Experiment D

```text
Clean vs Industrial Noise
```

## Experiment E

```text
Text Diagnostic Pipeline
vs
Voice Diagnostic Pipeline
```

Not all experiments are mandatory if time becomes limited.

---

# 119. Research Questions

Potential research questions include:

### RQ-ASR-01

How accurately does a strong multilingual ASR baseline transcribe Syrian Arabic industrial maintenance speech?

### RQ-ASR-02

Which error categories cause the largest downstream diagnostic impact?

### RQ-ASR-03

Does domain adaptation improve technical-token recognition compared with the baseline?

### RQ-ASR-04

How much does industrial noise degrade transcription quality?

### RQ-ASR-05

Does transcript confirmation sufficiently mitigate remaining critical ASR errors in the SALLIH workflow?

---

# 120. Fine-Tuning Decision Record

Before fine-tuning begins, create a written decision record containing:

```text
Baseline metrics
Major error categories
Critical error rate
Non-fine-tuning mitigations attempted
Remaining weaknesses
Expected benefit of fine-tuning
Available dataset size
Available compute
```

Decision:

```text
FINE_TUNE
```

or:

```text
DO_NOT_FINE_TUNE
```

This prevents fine-tuning from becoming an unexamined project assumption.

---

# 121. ASR MVP Acceptance Criteria

The ASR layer is considered functionally ready when:

- [ ] Technician can record Arabic speech.
- [ ] Recording is successfully sent to ASR.
- [ ] Whisper baseline produces a transcript.
- [ ] Transcript is shown before diagnostic use.
- [ ] Transcript can be edited.
- [ ] Transcript can be rejected and recorded again.
- [ ] Confirmed transcript enters fault understanding.
- [ ] Arabic-English code switching is supported operationally.
- [ ] Technical-token errors can be measured.
- [ ] Alarm codes can be evaluated by exact match.
- [ ] Text input remains available as fallback.
- [ ] ASR failure does not destroy diagnostic state.
- [ ] ASR baseline evaluation exists.

Fine-tuning is not required for this functional acceptance gate.

---

# 122. ASR Research Acceptance Criteria

The ASR research component is considered valid when:

- [ ] Test data has ground-truth transcripts.
- [ ] Dataset provenance is documented.
- [ ] Test set is held out from training.
- [ ] Multiple speakers are included where practical.
- [ ] Technical terms are annotated.
- [ ] Alarm codes are evaluated separately.
- [ ] Overall WER is reported.
- [ ] Domain-specific metrics are reported.
- [ ] Critical error categories are analyzed.
- [ ] Noise conditions are clearly labeled.
- [ ] Claims match actual dataset size and quality.

If fine-tuning is performed:

- [ ] Base and adapted models use the same held-out test set.
- [ ] Training configuration is documented.
- [ ] Improvement is measured rather than assumed.
- [ ] Regressions are reported.
- [ ] Final model version is reproducible.

---

# 123. Official Voice Demo Scenario

The final demo should include:

## Step 1

Technician presses microphone.

## Step 2

Technician says:

```text
الدرايف عم يفصل بعد عشر دقايق
وعم يعطيني overcurrent.
```

## Step 3

SALLIH transcribes the statement.

## Step 4

The transcript appears for review.

Example:

```text
الدرايف عم يفصل بعد عشر دقايق
وعم يعطيني overcurrent.
```

## Step 5

Technician confirms.

## Step 6

The confirmed text enters:

```text
Structured Fault Understanding
```

## Step 7

SALLIH continues with machine-grounded Guided Troubleshooting.

The demo must make clear that:

> voice is an input modality to the diagnostic agent, not a replacement for the agent.

---

# 124. Official Fine-Tuning Position

The official MVP position is:

```text
ASR Fine-Tuning:
Conditional

LLM Fine-Tuning:
Not Planned Initially
```

ASR fine-tuning becomes active work only after baseline evaluation demonstrates a measurable need.

LLM fine-tuning becomes active work only if architecture, RAG, normalization, prompting, and structured outputs fail to solve a persistent domain-language problem.

---

# 125. Decision Hierarchy

When an ASR problem is discovered, use this order:

```text
1. Is the transcript actually incorrect?

2. Is the error technically important?

3. Can technician confirmation solve it?

4. Can normalization solve it?

5. Can improved data preprocessing solve it?

6. Can a targeted vocabulary strategy solve it?

7. Does the problem remain frequent and harmful?

8. If yes:
   consider fine-tuning.
```

---

# 126. What ASR Fine-Tuning Must Not Become

The project must avoid this pattern:

```text
"We want to use fine-tuning because it sounds advanced."
```

The correct pattern is:

```text
"We measured a specific domain failure,
collected targeted data,
adapted the model,
and demonstrated measurable improvement."
```

---

# 127. What LLM Fine-Tuning Must Not Become

Do not fine-tune the diagnostic LLM to memorize:

```text
manuals
alarm tables
maintenance history
machine configuration
```

Those belong in:

```text
RAG
Structured Data
Machine Memory
```

The LLM should reason over machine knowledge, not permanently absorb it into opaque weights.

---

# 128. ASR Implementation Sequence

Recommended implementation order:

```text
1. Create ASR provider interface

2. Integrate Whisper Large v3 Turbo baseline

3. Implement audio recording endpoint

4. Implement transcript review UX

5. Build small Syrian industrial ASR test set

6. Run baseline

7. Add technical-token metrics

8. Perform error analysis

9. Implement normalization

10. Re-evaluate

11. Add noise evaluation

12. Decide whether fine-tuning is justified

13. If justified:
    collect targeted training data

14. Fine-tune / adapt

15. Compare against baseline

16. Integrate winning model
```

---

# 129. MVP Priority Order

If project time becomes limited, prioritize:

```text
1. Working voice capture
2. Reliable transcript confirmation
3. Whisper baseline
4. Technical-token evaluation
5. Diagnostic integration
6. Error analysis
7. Noise benchmark
8. ASR fine-tuning
9. LLM fine-tuning
```

Do not sacrifice the diagnostic workflow merely to complete a fine-tuning experiment.

---

# 130. Key Risks

## Risk 1 — ASR Works Well Without Fine-Tuning

This is not a project failure.

It means fine-tuning may be unnecessary.

Research value may instead come from:

```text
evaluation
domain metrics
normalization
end-to-end agentic diagnosis
```

---

## Risk 2 — Dataset Too Small for Fine-Tuning

Mitigation:

```text
Do not fine-tune prematurely.
```

Use the dataset for evaluation first.

---

## Risk 3 — Fine-Tuning Improves Training Vocabulary but Hurts Generalization

Mitigation:

- speaker-separated testing,
- holdout data,
- diverse sentence contexts,
- report regressions.

---

## Risk 4 — Noise Dominates Performance

Mitigation:

- noise augmentation,
- microphone placement,
- push-to-talk interaction,
- denoising if justified,
- transcript confirmation.

---

## Risk 5 — Alarm Codes Remain Unreliable

Mitigation:

Use explicit confirmation or structured alarm-code entry.

Do not force ASR to be the only input channel.

---

## Risk 6 — Voice Becomes a Development Bottleneck

Mitigation:

The diagnostic system must work fully through text first.

Voice is layered on top of a functioning diagnostic core.

---

# 131. Final ASR Principle

SALLIH does not need perfect speech recognition.

It needs:

> **speech recognition reliable enough to preserve the technical information required for safe, evidence-grounded diagnosis.**

The success criterion is therefore not:

```text
lowest possible WER
```

alone.

It is:

```text
Natural Syrian Speech
        ↓
Reliable Technical Transcript
        ↓
Technician Verification
        ↓
Correct Structured Fault
        ↓
Reliable Diagnostic Workflow
```

---

# 132. Final Fine-Tuning Principle

Fine-tuning is justified only when the project can answer all four questions:

```text
1. What specific baseline problem exists?

2. How frequently does it occur?

3. Why do simpler approaches fail?

4. Does fine-tuning measurably improve the held-out result?
```

If these questions cannot be answered with evidence:

> **Do not fine-tune yet.**

The objective of SALLIH is not to demonstrate that models can be fine-tuned.

The objective is to build:

> **a reliable Arabic-first industrial maintenance copilot whose voice interface works well enough to support real diagnostic reasoning.**