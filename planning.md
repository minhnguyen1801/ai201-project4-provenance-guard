# Provenance Guard — Planning Document

AI content attribution system for a writing platform. This is a pre-implementation planning document.

---

## Architecture

The **submission flow** accepts raw text via `POST /submit`, runs it through two independent detection signals (an LLM-based holistic assessment and a pure-Python stylometric analysis), combines their outputs into a single confidence score via a weighted average, maps that score to a human-readable transparency label, writes an immutable audit log entry, and returns a JSON response with the content ID, attribution, confidence, label, and per-signal scores. The **appeal flow** accepts a `content_id` and the creator's reasoning via `POST /appeal`, updates the existing audit log entry's status from `classified` to `under_review` (appending the reasoning and a timestamp), and returns a JSON acknowledgement. No automated re-classification ever occurs on appeal — a human reviewer inspects flagged entries via `GET /log`.

```
SUBMISSION FLOW
===============

  Creator
    │  raw text + creator_id
    ▼
┌──────────────┐  raw text   ┌─────────────────────┐
│ POST /submit │────────────▶│ Signal 1: LLM        │
│   (route)    │             │ (Groq llama-3.3-70b) │
│              │◀────────────│                      │
│              │ signal score└─────────────────────┘
│              │   (0.0–1.0)
│              │  raw text   ┌─────────────────────┐
│              │────────────▶│ Signal 2: Stylometric│
│              │             │ (pure Python)        │
│              │◀────────────│                      │
│              │ signal score└─────────────────────┘
│              │   (0.0–1.0)
│              │
│              │  both signal scores
│              ▼
│        ┌──────────────────────┐
│        │ Confidence Scoring   │
│        │ weighted avg         │
│        │ (0.65 LLM/0.35 styl) │
│        └──────────────────────┘
│              │ combined confidence (0.0–1.0)
│              ▼
│        ┌──────────────────────┐
│        │ Transparency Label   │
│        │ (score → label text) │
│        └──────────────────────┘
│              │ label text + confidence + signal scores
│              ▼
│        ┌──────────────────────┐
│        │ Audit Log (append)   │
│        │ status="classified"  │
│        └──────────────────────┘
│              │ content_id
│              ▼
└──────▶ JSON response
         {content_id, attribution, confidence, label, signal_scores}


APPEAL FLOW
===========

  Creator
    │  content_id + creator_reasoning
    ▼
┌──────────────┐  content_id   ┌──────────────────────────┐
│ POST /appeal │──────────────▶│ Audit Log (update)       │
│   (route)    │               │ status: classified       │
│              │               │   → under_review         │
│              │               │ + appeal_reasoning       │
│              │               │ + appeal_timestamp       │
│              │◀──────────────│                          │
│              │   content_id  └──────────────────────────┘
│              ▼
└──────▶ JSON response
         {status, message}
```

---

## Detection Signals

### Signal 1 — LLM-based (Groq, `llama-3.3-70b-versatile`)

- **What it measures:** Holistic semantic and stylistic coherence. The LLM is asked to assess whether the text reads as AI-generated, drawing on patterns it cannot reduce to simple counts (flow, predictability, "voice").
- **Output format:** Float `0.0`–`1.0` — the estimated probability that the text is AI-generated.
- **Blind spot:** LLMs can disagree with themselves across runs; heavily edited AI output may score low; very formal human writing may score high. The signal is non-deterministic and prompt-sensitive.

### Signal 2 — Stylometric heuristics (pure Python, no external libraries)

Computes three sub-metrics, each combined into a single AI-probability:

- **(a) Sentence length variance** — AI text tends to be more uniform; low variance pushes the score toward AI.
- **(b) Type-token ratio (TTR)** — vocabulary diversity; AI text reuses words more, so low TTR pushes toward AI.
- **(c) Punctuation density** — AI text uses fewer informal punctuation patterns; low density of informal punctuation pushes toward AI.

- **Output format:** Float `0.0`–`1.0` — probability the text is AI-generated, derived by combining the three sub-metrics.
- **Blind spot:** Formal human writing (academic papers, legal documents) has low variance and low TTR, scoring as AI-like. The signal cannot capture intent or creativity.

### Combination

**Weighted average** — LLM signal weight **0.65**, stylometric weight **0.35**:

```
confidence = (0.65 × llm_score) + (0.35 × stylometric_score)
```

The LLM signal gets higher weight because it captures semantic coherence that heuristics miss, but the stylometric signal provides independent structural verification that does not depend on a remote model's mood.

---

## Uncertainty Representation

| Confidence range          | Classification               |
|---------------------------|------------------------------|
| `>= 0.75`                 | "Likely AI-generated" (high-confidence AI) |
| `>= 0.45` and `< 0.75`    | "Uncertain — could be either" |
| `< 0.45`                  | "Likely human-written" (high-confidence human) |

A score of **0.51** lands in the uncertain zone and displays a cautious label. A score of **0.92** displays a definitive AI label. The asymmetry is intentional: the uncertain band is wide (`0.45`–`0.75`) because **false positives — mislabeling human work as AI — are more harmful than false negatives** on a writing platform. We would rather say "we're not sure" than wrongly brand a person's authentic writing as machine-generated.

---

## Transparency Label Variants

The exact reader-facing text for each variant (`X%` is the confidence rendered as a percentage):

**HIGH-CONFIDENCE AI** (`confidence >= 0.75`):

> ⚠️ AI-Generated Content — Our system is highly confident (X%) this content was AI-generated. If you are the creator and believe this is incorrect, you can submit an appeal.

**HIGH-CONFIDENCE HUMAN** (`confidence < 0.45`):

> ✓ Likely Human-Written — Our analysis suggests this content was written by a person (X% confidence). Attribution signals indicate authentic human authorship.

**UNCERTAIN** (`0.45 <= confidence < 0.75`):

> ? Attribution Unclear — Our system cannot confidently determine whether this content is human- or AI-written (confidence: X%). The creator may appeal if they believe the classification is inaccurate.

---

## Appeals Workflow

- **Who can appeal:** Any `creator_id` associated with the `content_id`.
- **What they provide:** `content_id` + `creator_reasoning` (free text explaining why they believe the classification is wrong).
- **System actions:**
  - Update the audit log entry's status from `classified` to `under_review`.
  - Append `appeal_reasoning` and `appeal_timestamp` fields to the existing log entry.
- **What a human reviewer sees:** `GET /log` returns entries with status `under_review`, including the original classification, confidence score, both signal scores, and the creator's reasoning.
- **No automated re-classification.** An appeal only flags the entry for human review; the system never re-scores content automatically.

---

## Anticipated Edge Cases

1. **Heavily repetitive poetry** (anaphora, refrains): sentence length variance will be very low and TTR artificially low, so the stylometric signal will over-flag clearly human work as AI. The wide uncertain band and the appeal workflow mitigate this.
2. **Non-native English speakers writing formal text:** produces low sentence variance and formal vocabulary similar to AI patterns. Both signals may score high. This is called out in the label as a known limitation, and the appeal path is available.

---

## API Surface

| Method & path  | Request body                          | Response |
|----------------|---------------------------------------|----------|
| `POST /submit` | `{text, creator_id}`                  | `{content_id, attribution, confidence, label, signal_scores}` |
| `POST /appeal` | `{content_id, creator_reasoning}`     | `{status, message}` |
| `GET /log`     | —                                     | `{entries: [...]}` (most recent 50 audit log entries) |

---

## AI Tool Plan

### M3 — Submit endpoint + Signal 1
- **Provide:** Detection Signals section + Architecture diagram.
- **Ask for:** Flask app skeleton with a `POST /submit` stub + Groq LLM signal function returning a float `0`–`1`.
- **Verify:** Call the function directly on 3 test inputs before wiring it into the route.

### M4 — Signal 2 + confidence scoring
- **Provide:** Detection Signals + Uncertainty Representation sections + diagram.
- **Ask for:** Stylometric function computing sentence variance + TTR + punctuation density, plus the weighted combination logic.
- **Verify:** Run 4 test inputs (clear AI, clear human, two borderline) and confirm scores vary meaningfully.

### M5 — Production layer
- **Provide:** Transparency Label + Appeals Workflow sections + diagram.
- **Ask for:** Label generation function mapping score to text, plus the `POST /appeal` endpoint.
- **Verify:** Confirm all 3 label variants are reachable and that an appeal updates the log status correctly.
