# Provenance Guard

A multi-signal AI content attribution API for writing platforms — classifies text as human-written or AI-generated, communicates uncertainty honestly, and gives creators a path to appeal.

## Architecture Overview

A submission enters through `POST /submit`, which receives the raw `text` and a `creator_id`. The text is first passed to **Signal 1**, the LLM signal (Groq `llama-3.3-70b-versatile`), which returns a float `0.0–1.0`; it is then passed to **Signal 2**, the stylometric signal (pure Python), which returns its own float `0.0–1.0`. The two scores are combined via a weighted average into a single confidence score, which maps to a transparency label, after which an entry is written to the SQLite audit log and a JSON response is returned to the caller. The **appeal flow** runs through `POST /appeal`, which receives a `content_id` and `creator_reasoning`, updates the corresponding audit log entry's status to `under_review`, and thereby makes the entry — with the creator's reasoning attached — visible to human reviewers via `GET /log`.

```
SUBMISSION FLOW
Creator
| raw text + creator_id
v
+------------------+ raw text +---------------------+
| POST /submit |------------>| Signal 1: LLM |
| (route) | | (Groq llama-3.3-70b) |
| |<------------| |
| | llm_score +---------------------+
| | (0.0-1.0)
| | raw text +---------------------+
| |------------>| Signal 2: Stylometric|
| | | (pure Python) |
| |<------------| |
| | styl_score +---------------------+
| | (0.0-1.0)
| |
| | both signal scores
| v
| +------------------------+
| | Confidence Scoring |
| | 0.65llm + 0.35styl |
| +------------------------+
| | combined confidence (0.0-1.0)
| v
| +------------------------+
| | Transparency Label |
| | (score -> label text) |
| +------------------------+
| | label text + confidence + signal scores
| v
| +------------------------+
| | Audit Log (append) |
| | status="classified" |
| +------------------------+
| | content_id
| v
+-------> JSON response
{content_id, attribution, confidence, label, signal_scores}

APPEAL FLOW
Creator
| content_id + creator_reasoning
v
+-----------------+ content_id +--------------------------+
| POST /appeal |--------------->| Audit Log (update) |
| (route) | | status: classified |
| | | -> under_review |
| | | + appeal_reasoning |
| | | + appeal_timestamp |
| |<--------------| |
| | content_id +--------------------------+
| v
+-------> JSON response
{status, message, content_id}
```

## Detection Signals

### Signal 1 — LLM-based (Groq, llama-3.3-70b-versatile)

- **What it measures:** holistic semantic and stylistic coherence — flow, predictability, and voice that statistical methods cannot capture.
- **Why AI text differs:** LLMs produce smooth, predictable prose; human writing has idiosyncratic voice, unexpected phrasing, and irregular flow.
- **Output:** float `0.0–1.0` (probability text is AI-generated).
- **What it misses:** non-deterministic across runs; heavily edited AI output may score low; very formal human writing may score high; results are prompt-sensitive.

### Signal 2 — Stylometric heuristics (pure Python, no external libraries)

Three sub-metrics with equal weight:

- **Sub-metric A — Sentence length variance:** AI text has uniform sentence lengths. Standard deviation of sentence word-counts is computed. Low std dev pushes toward AI. Bounds: `std_dev >= 15` maps to score `0.0` (human-like), `std_dev <= 2` maps to score `1.0` (AI-like), linear interpolation in between.
- **Sub-metric B — Type-token ratio (TTR):** vocabulary diversity. `TTR = unique tokens / total tokens`. AI reuses words more, so low TTR pushes toward AI. Bounds: `TTR >= 0.85` maps to `0.0` (human), `TTR <= 0.50` maps to `1.0` (AI), linear interpolation.
- **Sub-metric C — Informal punctuation density:** count of characters in the set `[! ? ... — - ( ) " ']` divided by total word count. AI avoids informal punctuation. Bounds: `density >= 0.15` maps to `0.0` (human), `density <= 0.02` maps to `1.0` (AI), linear interpolation.

Final stylometric score = `(variance_score + ttr_score + punct_score) / 3.0`

**What it misses:** formal human writing (academic papers, legal documents) has low sentence variance and low TTR, scoring as AI-like. Cannot capture intent or creativity.

**Why these two signals are genuinely independent:** Signal 1 is semantic (meaning and voice), Signal 2 is structural (measurable statistics). Neither can substitute for the other.

## Confidence Scoring

The two signals are combined with a weighted average:

```
confidence = (0.65 x llm_score) + (0.35 x stylometric_score)
```

The LLM signal gets the higher weight because it captures semantic coherence that the heuristics fundamentally miss, while the stylometric signal provides independent structural verification that does not depend on a remote API. Weighting the LLM more heavily reflects its broader sensitivity, but keeping a meaningful 0.35 on stylometrics ensures a single noisy or unavailable API call cannot fully dictate the verdict.

**Thresholds:**

| Confidence range            | Attribution    |
|-----------------------------|----------------|
| `>= 0.75`                   | `likely_ai`    |
| `>= 0.45` and `< 0.75`      | `uncertain`    |
| `< 0.45`                    | `likely_human` |

**The asymmetry is intentional.** The uncertain band is deliberately wide — 30 percentage points — because false positives (mislabeling human work as AI) are more harmful on a writing platform than false negatives. When in doubt, the system says "uncertain" rather than accusing a human writer of something they did not do.

**Example 1 — High-confidence AI:**

> "Artificial intelligence represents a transformative paradigm shift in modern society. It is important to note that while the benefits of AI are numerous, it is equally essential to consider the ethical implications. Furthermore, stakeholders across various sectors must collaborate to ensure responsible deployment."

Results: `llm_score: 0.92`, `stylometric_score: 0.81`, combined confidence: `0.88`, attribution: `likely_ai`

**Example 2 — High-confidence human:**

> "ok so i finally tried that new ramen place downtown and honestly? underwhelming. the broth was fine but they put WAY too much sodium in it and i was thirsty for like three hours after."

Results: `llm_score: 0.08`, `stylometric_score: 0.11`, combined confidence: `0.09`, attribution: `likely_human`

## Transparency Label Variants

**HIGH-CONFIDENCE AI** (`confidence >= 0.75`):

> ⚠️ AI-Generated Content — Our system is highly confident (88%) this content was AI-generated. If you are the creator and believe this is incorrect, you can submit an appeal.

**HIGH-CONFIDENCE HUMAN** (`confidence < 0.45`):

> ✓ Likely Human-Written — Our analysis suggests this content was written by a person (9% confidence). Attribution signals indicate authentic human authorship.

**UNCERTAIN** (`0.45 <= confidence < 0.75`):

> ? Attribution Unclear — Our system cannot confidently determine whether this content is human- or AI-written (confidence: 58%). The creator may appeal if they believe the classification is inaccurate.

## Rate Limiting

Limits applied to `POST /submit`: **10 requests per minute, 100 requests per day, per IP address.** `POST /appeal` is **not** rate-limited.

**Reasoning:** A writer submitting their own work might submit 5–10 pieces in a session but rarely more than a few per minute. The per-minute limit stops scripted flooding while not inconveniencing any legitimate user. The per-day cap of 100 prevents sustained automated probing of the classifier while leaving room for heavy legitimate use. Appeals are rare, manual actions with no realistic abuse vector, so they are not limited.

**Rate limit evidence** — running 12 rapid requests returns `200` for the first 10 and `429` for requests 11 and 12:

```
200
200
200
200
200
200
200
200
200
200
429
429
```

## API Reference

### POST /submit

- **Request body:** `{"text": "string", "creator_id": "string"}`
- **Response:** `{"content_id": "uuid", "attribution": "likely_ai|uncertain|likely_human", "confidence": float, "label": "string", "signal_scores": {"llm": float, "stylometric": float}}`
- **Rate limited:** 10/min, 100/day
- **Errors:** `400` if `text` or `creator_id` is missing; `500` if the Groq call fails.

```bash
curl -s -X POST http://localhost:5000/submit \
  -H "Content-Type: application/json" \
  -d '{"text": "Your text here", "creator_id": "user-1"}' | python -m json.tool
```

### POST /appeal

- **Request body:** `{"content_id": "uuid", "creator_reasoning": "string"}`
- **Response:** `{"status": "under_review", "message": "string", "content_id": "uuid"}`
- **Errors:** `400` if a field is missing; `404` if the `content_id` is not found.

```bash
curl -s -X POST http://localhost:5000/appeal \
  -H "Content-Type: application/json" \
  -d '{"content_id": "PASTE-CONTENT-ID-HERE", "creator_reasoning": "I wrote this myself from personal experience."}' | python -m json.tool
```

### GET /log

- **Response:** `{"entries": [...]}` — most recent 50 audit log entries.

```bash
curl -s http://localhost:5000/log | python -m json.tool
```

## Audit Log

Each classification appends a row; an appeal updates the existing row in place. Example entries:

```json
{
  "entries": [
    {
      "content_id": "3f2a9c1e-7b4d-4e62-9a81-0c5d6e2f1a7b",
      "creator_id": "user-1",
      "timestamp": "2026-06-29T14:02:17.483921+00:00",
      "attribution": "likely_ai",
      "confidence": 0.88,
      "llm_score": 0.92,
      "stylometric_score": 0.81,
      "status": "classified",
      "appeal_reasoning": null,
      "appeal_timestamp": null
    },
    {
      "content_id": "a17c4d90-6e2b-4f3a-8c11-9d7e0b3a5f24",
      "creator_id": "user-2",
      "timestamp": "2026-06-29T14:05:43.117602+00:00",
      "attribution": "likely_human",
      "confidence": 0.09,
      "llm_score": 0.08,
      "stylometric_score": 0.11,
      "status": "classified",
      "appeal_reasoning": null,
      "appeal_timestamp": null
    },
    {
      "content_id": "c84b1f2d-3a90-4d57-b6e8-2f1c7a90d4e3",
      "creator_id": "user-3",
      "timestamp": "2026-06-29T14:09:11.902845+00:00",
      "attribution": "likely_ai",
      "confidence": 0.83,
      "llm_score": 0.87,
      "stylometric_score": 0.76,
      "status": "under_review",
      "appeal_reasoning": "I wrote this essay myself over two weeks; it is formal because it is an academic paper.",
      "appeal_timestamp": "2026-06-29T15:22:05.660413+00:00"
    }
  ]
}
```

## Known Limitations

- **Repetitive poetry (anaphora, refrains):** sentence length variance will be very low and TTR artificially compressed, causing the stylometric signal to over-flag genuinely human poems as AI-generated. Structural repetition is indistinguishable from AI uniformity in the signal's math.
- **Non-native English speakers writing formal text:** produces low sentence variance and formal vocabulary that mirrors AI patterns. Both signals may score high, causing a false positive. The wide uncertain band and appeal path mitigate but do not eliminate this.
- **AI detection is an unsolved problem:** a sufficiently edited AI draft with informal voice added will score low. The system acknowledges uncertainty rather than claiming precision it cannot achieve.

## Spec Reflection

**One way the spec helped:** Writing the three label variants verbatim in `planning.md` before any code was written gave `make_label()` an exact contract to implement — no ambiguity about what the uncertain label should say while coding.

**One way implementation diverged:** The planning doc described the stylometric punctuation signal as capturing informal punctuation density but did not specify which characters counted as informal. During implementation the specific character set and the decision to count literal `"..."` as an ellipsis were made on the fly. The planning doc should have listed the exact characters.

## AI Usage

**Instance 1:** Directed AI to generate the Flask app skeleton and Groq signal function for M3. The generated structure was correct but the initial output was missing `response_format={"type": "json_object"}` and `temperature=0.0` on the Groq API call. I added both to make the LLM signal deterministic and reliably parseable as JSON.

**Instance 2:** Directed AI to generate the stylometric signal function for M4. The `_interpolate()` helper used correct linear interpolation math, but I manually verified the direction of each bound before accepting — confirming that `human_bound=15` and `ai_bound=2` for std_dev correctly produces `score=0` at high variance and `score=1` at low variance.

## Setup

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Create a `.env` file in the repo root (do not commit it — add to `.gitignore`):

```
GROQ_API_KEY=your_key_here
```

Run the server:

```bash
python app.py
```

The server starts on `http://localhost:5000`. The SQLite database `audit_log.db` is created automatically on first run.
