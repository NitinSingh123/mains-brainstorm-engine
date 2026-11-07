# Practice Modes — Engineering Design

Status: design draft. Builds on `01-brainstorm-evaluator-v1.md`.

## 1. Principle: one evaluation core, modes are policy

Both modes do the same expensive, hard-to-get-right thing: **turn a brainstorm into labelled atomic points against a question map.** They differ only in *what happens around it*: timer, how much feedback, when feedback arrives, whether there's a second attempt.

So the design is:

```
                ┌──────────── ModeConfig (data, not code) ────────────┐
                │ questions, timer, decode, feedback depth/timing,    │
                │ second attempt, schedule misses                     │
                └─────────────────────────────────────────────────────┘
                                       │
 Question Selector ─► Attempt capture ─► ANALYSE (shared) ─► SCORE (shared) ─► Feedback renderer ─► Profile events (shared)
   (per mode)          (shared)          LLM call #1          code              (per depth)          code
```

**Never fork the analysis prompt per mode.** If rapid mode labels points differently from deep mode, the learning profile ends up mixing two incompatible measurements, and the progress charts mean nothing.

## 2. Modes as configuration

```python
@dataclass(frozen=True)
class ModeConfig:
    name: str
    question_count: tuple[int, int]      # (min, max)
    brainstorm_secs: int
    decode_required: bool                # one-line decode before brainstorming
    feedback_depth: Literal["full", "compact"]
    feedback_timing: Literal["immediate", "end_of_session"]
    second_attempt: bool
    hint_ladder: bool
    schedule_misses: bool                # feed misses into spaced repetition

MODES = {
    "deep": ModeConfig("deep", (1, 1), 120, True,  "full",    "immediate",      True,  True,  True),
    "rapid": ModeConfig("rapid", (5, 7), 90, False, "compact", "end_of_session", False, False, True),
}
```

Store a **snapshot of the config on each session row**. When you change rapid mode from 90s to 75s, old sessions still say what rules they were played under.

Future modes (exam simulation, full answer writing, revision) become new rows here plus at most one new renderer. Don't build them now.

## 3. How the modes differ in behaviour

| | Deep (1 question) | Rapid (5–7 questions) |
|---|---|---|
| Purpose | Learn *why* you missed something; build reasoning | Build speed; make triggers fire automatically |
| Decode | Required (gate scored) | Off by default; the directive is implicit in the brainstorm |
| Feedback | Full format from doc 01 (got right / think further / sharpen / fact check) | Per question: 1 line hit, 1 line miss, 1 trigger cue |
| Timing | Immediately after the attempt | After the whole drill |
| Second attempt | Yes, in session | **No. The "second attempt" is a future question** that tests the same missed dimension (via scheduler) |
| Unique value | Depth on one question | **Cross-question patterns**: "You skipped way-forward in 4 of 7", "you never use judgments under time pressure" |
| LLM calls | analyse ×2 + feedback ×2 | analyse ×N + 1 session summary |

The session-level pattern summary in rapid mode is something a coaching sheet can't do. Treat it as a first-class output, not an afterthought.

## 4. Evidence semantics: same event types, weighted by context

The profile from doc 01 stays the same. The mode and attempt only change *which* event fires:

| Situation | Event |
|---|---|
| Hit in rapid mode (timed, no help) | `recalled_independent`, `timed=true` — the strongest recall signal |
| Hit in deep attempt 1 | `recalled_independent`, `timed=true` |
| Hit in deep attempt 2 after hint at level n | `recalled_prompted(n)` |
| Miss in any first attempt | `missed` → scheduler sets the dimension due soon |
| Hit with `analytical` quality on a never-seen question | additionally `applied` |

## 5. Question selection (the scheduler)

Rapid mode is only as good as the 5–7 questions it picks. v1 is rules, not ML:

1. **Track (student, dimension) pairs**, with a due date. This covers both topic dimensions (`assent_to_bills`) and generic triggers (`contentious→recent_disputes`, `examine→way_forward`).
2. Miss → due tomorrow. Hit → interval × 2.5 (simplified SM-2). Prompted hit → interval × 1.2.
3. Build a drill: ~60% questions whose maps contain **due** dimensions, ~40% new questions. Prefer questions the student **has never seen**; recall on a repeated question is partly memory of the question.
4. Interleave topics within a drill. Never 7 Governor questions in a row.

This needs a question bank where every question has a map. That's the content bottleneck again: the scheduler can only pick from questions you've mapped.

## 6. Data model (Postgres, v1)

```sql
question_maps   (id, question_id, version, yaml, reviewed_by, reviewed_at)   -- immutable versions
questions       (id, source, paper, topic, marks, text)
sessions        (id, student_id, mode, mode_config jsonb, started_at, ended_at)
session_items   (id, session_id, question_id, position)
attempts        (id, session_item_id, attempt_no, input_type, raw_text, image_key,
                 started_at, submitted_at, decode_text)
analyses        (id, attempt_id, map_version, prompt_version, model,
                 raw_output jsonb, points jsonb, scores jsonb, status, latency_ms, tokens_in, tokens_out)
feedback        (id, session_id, attempt_id NULL, prompt_version, model, content jsonb)
evidence_events (id, student_id, dimension_id, event_type, hint_level, timed, attempt_id, created_at)
schedule        (student_id, dimension_id, due_at, interval_days)
```

The main points:
- `analyses` records **map_version, prompt_version and model**. When you fix a map, you can re-run old attempts and re-derive the profile, because `evidence_events` is append-only.
- Store `raw_output` from the LLM always. When a student says "it said I missed X but I wrote it", this is how you debug.
- Store tokens and latency per call from day one. That's your unit-economics dashboard.

## 7. Execution, latency, failure handling

- **Rapid mode: analyse each question as soon as it's submitted**, in the background while the student does the next one. By the end of the drill, 6 of 7 analyses are done. Only the last analysis and the summary run while the student waits.
- **Deep mode** is synchronous. Target under 15s from submit to feedback, including transcription.
- Job queue: a Postgres `jobs` table polled by a worker (or FastAPI background tasks to start). No Redis/Celery until there's load.
- Validate every LLM output against a JSON schema. On failure, retry once with the validation error appended. On second failure, mark `status=failed` and show "couldn't evaluate this one", never a fabricated result.
- **Partial batch failure:** if 1 of 7 fails, show 6 and retry the 7th in the background. The session summary states it's based on 6.
- Idempotency key per attempt submission, because phone uploads on bad networks get double-submitted.
- Put the question map first in the prompt so the provider's prompt caching applies. Every attempt on the same question reuses the same prefix.

## 8. Input capture details

- **v1: typed input in both modes.** It removes transcription errors while you're still validating the evaluator.
- v2 photos: **one photo per question** in rapid mode. One page with 7 brainstorms means splitting the page into regions, which is a separate failure mode not worth taking on early.
- Timer runs on the client; the server records `started_at`/`submitted_at`. Photo uploads after paper practice are trust-based. Record the elapsed time and flag it, but don't block.

## 9. Cost model: measure, don't guess

Per session, rapid ≈ N analyses + 1 summary; deep ≈ 2 analyses + 2 feedback calls. A daily rapid user is therefore ~200+ LLM calls a month. Whether ₹199 works depends on cost per analysis call, which you'll only know once the golden-set harness tells you **the cheapest model that labels as accurately as humans**. Run every candidate model on the golden set; pick on agreement first, cost second.

Freemium follows from this naturally (still a hypothesis): free = 1 rapid drill a day; premium = unlimited rapid + deep mode + notes personalisation.

## 10. The golden-set harness is the most important piece of engineering

LLM apps break silently: a prompt tweak that improves one question degrades three others. Treat this like a test suite:

```
golden/
  gs2-fed-001/attempt-01.txt  + attempt-01.labels.json   (human labels)
  ...
make eval  → runs ANALYSE on every golden attempt
           → reports agreement per label field, false-gap rate, cost, latency
           → fails if agreement drops vs. last baseline
```

Run it on every change to a prompt, a map or the model.

## 11. Build order

The single question is the primitive. Rapid is N singles with compact feedback delivered at the end. Build in this order:

1. **Analysis core + golden harness** (CLI only, no UI). *Gate: agreement with human labels close to agreement between two humans.*
2. **Deep mode**, typed: one question, full feedback, second attempt. You'll debug most here because you can see the whole loop.
3. **Rapid mode**: background analysis, compact renderer, session summary. Mostly configuration plus one new renderer and one summary prompt.
4. **Scheduler**, once there are ~40+ mapped questions. Before that, there's nothing to schedule.
5. Photo input.
6. Notes personalisation (the v0 vs v1 experiment from doc 01).

What's deliberately left out: microservices, an agent framework, vector DB, mobile app, Redis, Kubernetes. A single FastAPI monolith with Postgres handles this up to thousands of users.
