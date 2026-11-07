# Brainstorm Evaluator v1 — GS2 Federalism / Centre-State Relations

Status: design draft, not yet validated with any aspirant.
Companion file: `../knowledge/federalism/answer-keys/gs2-fed-001-governor.yaml` (worked reference map).

---

## 0. Where I'm pushing back on the brief

Four changes before the rubric itself. Each one comes from a problem I see in the brief.

**0.1 Don't let the AI decode the question before the student tries.**
The workflow in the brief has the AI decode the question (Step 2) *before* the student brainstorms (Step 3). Decoding is the first skill that fails in the exam hall. If the AI does it for them, we can't train it and we can't measure "question interpretation".
→ The student writes a one-line decode first (directive + what's being asked, about 15 seconds). Then they brainstorm. The AI's decode is shown as *feedback*. A "guided mode" that shows the decode first can exist for beginners, but those attempts get flagged and aren't counted as independent evidence.

**0.2 Ten dimensions on a 100-point scale is too fine for a 90-second brainstorm.**
A 90-second brainstorm is 8–15 keywords and arrows. Asking an LLM to give "depth: 14/20" on that produces noise, and the noise changes between runs. Humans can't grade it consistently at that resolution either.
→ One pass/fail-style **gate** (interpretation), six dimensions scored **0–3 with written anchors**, and scores **computed by code** from per-point labels. The LLM labels points; it doesn't pick numbers.

**0.3 The evaluator's intelligence is in the Question Map, not the prompt.**
If you ask an LLM "what's missing?" with no reference, it will always find something missing. It drifts toward sophisticated, AI-sounding points, which is exactly what the brief says to avoid. Every question needs a human-reviewed **Question Map**: demands, dimensions ranked core/strong/bonus, the *trigger words* in the question that should activate each dimension, accepted evidence, and anti-patterns.
For the MVP you hand-author 3 maps. Later the LLM drafts maps and you review them. Over time this library of maps across PYQs becomes the defensible asset, more than any prompt.

**0.4 Validate the hint ladder *without* personal notes first.**
Notes-based personalisation is the differentiator, so it needs to be tested on its own. Run v0 (map + hint ladder, no notes) and v1 (plus notes retrieval) on the same students. If v1 doesn't beat v0 on second-attempt improvement, the personal knowledge base needs rethinking before you spend months building it.

---

## 1. Unit of evaluation: the atomic point

The evaluator first splits the brainstorm into **atomic points**: one idea each, keeping the student's own words. Every later step works on these labels.

| Field | Values | Meaning |
|---|---|---|
| `text` | student's words, verbatim | Never paraphrased upward |
| `maps_to` | dimension id \| `unmapped_relevant` \| `null` | Which map dimension it serves |
| `relevance` | `relevant` / `tangential` / `irrelevant` | See §3 |
| `quality` | `label` / `specific` / `analytical` | See §3 |
| `evidence[]` | anchors cited (Article, case, committee, example) + `accurate` / `inaccurate` / `unverifiable` | Only what the student wrote |
| `duplicate_of` | point id \| null | Repetition |
| `ocr_uncertain` | bool | From transcription; uncertain points are never penalised |

**Hard rule for the labelling prompt: do not complete the student's thought.** LLMs tend to be generous. They read "356 – Bommai" and credit the floor-test argument the student never wrote. A point earns `analytical` only if the link is visible on the page: an arrow, "→", "because", "leads to", or a juxtaposition the student clearly intended. Abbreviations count ("SC", "Punchhi- fixed tenure"). Imagined reasoning doesn't.

---

## 2. The rubric

### Gate — Question Interpretation (0 / 1 / 2)

Judged from the student's one-line decode plus the brainstorm's overall orientation.

| Level | Anchor |
|---|---|
| 2 Clear | Identifies all directives and the central theme. Brainstorm is oriented to them. |
| 1 Partial | Misses a directive (e.g. only "powers", no "issues") or drifts (writes about federalism in general). |
| 0 Misread | Answering a different question (e.g. Delhi LG, or President's powers). |

- **0** → stop. Feedback is *only* about interpretation. The student re-decodes and re-brainstorms. Dimension hints on a misread question teach the wrong lesson.
- **1** → interpretation is the first feedback item. Scoring continues.

### Six scored dimensions (0–3 each, computed from point labels)

| # | Dimension | What it measures | Computed from | Weight |
|---|---|---|---|---|
| A | **Demand coverage** | Every explicit + implicit demand has ≥1 relevant point | demands with a `relevant` point mapped to them | 3 |
| B | **Dimensional coverage** | Right *kinds* of thinking present, weighted by importance | core and strong dimensions covered (saturating, §4) | 3 |
| C | **Specificity** | Points are concrete rather than topic labels | share of relevant points at `specific` or `analytical` | 2 |
| D | **Evidence** | Accurate, relevant anchors tied to points | count of accurate anchors on relevant points, capped; inaccurate ones deduct | 2 |
| E | **Focus** | Little irrelevance and repetition | precision = relevant ÷ total non-duplicate points | 1 |
| F | **Prioritisation & structure** | Exam-usable skeleton: grouping, order, intro/conclusion seed, top points marked | structural labels + starred points (§4.4) | 1 |

Fact accuracy is reported **separately** as flags, not as a dimension. A wrong judgment name is a different problem from weak thinking, and the student should see it as a correction, not a lower score.

**Anchors (first-draft thresholds; calibrate on real data):**

| Score | A Demand | B Dimensional | C Specificity | D Evidence (250w) | E Focus | F Structure |
|---|---|---|---|---|---|---|
| 3 | all incl. implicit | ≥85% weighted | ≥70% | ≥4 accurate, 0 wrong | ≥90% | grouped + ordered + top points marked |
| 2 | all explicit, misses an implicit | ≥60% | ≥45% | 2–3 | ≥75% | grouped or ordered |
| 1 | misses one explicit | ≥35% | ≥20% | 1 | ≥55% | some sign of intent |
| 0 | misses most | <35% | <20% | 0, or any fabricated | <55% | flat list |

Weighted composite = Σ(score × weight) / 36. **Use it internally to track progress. Don't show it to the student on attempt 1.** A number before they've had a chance to improve pushes them toward chasing the score instead of thinking. Show a band and the profile after attempt 2.

How the brief's ten criteria map onto this, so nothing is dropped:

| Brief | Here |
|---|---|
| Interpretation | Gate |
| Coverage of demands | A |
| Relevance | relevance label → E, and filters A–D |
| Variety of dimensions | B |
| Depth | C (`analytical` share) |
| Examples / evidence | D |
| Provisions, judgments, committees | D + accuracy flags |
| Logical connections | `analytical` label → C |
| Repetition / irrelevance | E |
| Prioritisation / structure | F |

---

## 3. Relevant vs superficial — the three tests

Apply in order to every atomic point.

**Test 1 — Demand test: which demand does this serve?**
None → `irrelevant`. Serves the topic but not this question → `tangential`.

**Test 2 — Trigger test: would this point appear *because of this question*, or in *any* question on the topic?**
"Governor can pardon (Art 161)" is true in every Governor question, so it's topic-true. "Governor delays assent to Bills" appears *because* the question says "contentious", so it's question-relevant. Topic-true points can still be relevant (D1 legitimately needs "appointed by President"), but a brainstorm made *mostly* of topic-true points is the classic sign of a student reciting a chapter instead of answering.

**Test 3 — So-what test: does the point carry a claim, a direction or a link?**

| Quality | Example (student's words) | Why |
|---|---|---|
| `label` | "discretionary powers" | A chapter heading. Reveals nothing about thinking. |
| `specific` | "discretion Art 163 – hung assembly" | Concrete anchor + situation |
| `analytical` | "hung assy → Gov picks CM → partisan calls (Goa 17, Kar 18) → Bommai floor test" | Anchor + consequence + evidence + remedy, all visible |

"Superficial" = relevant but stuck at `label`. The fix is a **sharpening prompt** (§5), not a penalty speech.

**Credit original thinking.** A relevant point that isn't in the map is labelled `unmapped_relevant`. It gets credit in C/D/E (not B) and is queued for your review. If you accept it, it goes into the map. Without this, the system punishes exactly the independent thinking we want to grow.

---

## 4. Missing dimensions without rewarding information dumping

**4.1 Saturation.** A dimension is either covered or not. One good point fully covers it. A second point *in the same dimension* adds nothing to B. Ten points on Art 356 ≠ breadth.

**4.2 Tiered weighting.** B = (2 × core covered + 1 × strong covered) / (2 × core total + 1 × strong total). Bonus dimensions are praised in feedback but **never enter B**, so they can't offset a missing core dimension.

**4.3 Precision is scored.** E punishes the "write everything I know" strategy directly. Dumping raises point count, lowers precision and raises repetition.

**4.4 Exam feasibility.** Each map has a `feasible_point_range` (8–12 for 250 words, ~5–7 for 150). Above the upper bound, the feedback asks the student to **star their top N**, and F is judged on whether the starred set is the right one. This turns dumping into a prioritisation exercise instead of a scolding.
*Hypothesis to test:* asking students to star their top points is a cheap, strong signal of prioritisation skill.

**4.5 Gaps are only judged against the map.** "Missing" means a core or strong dimension in the reviewed map, never "something else the LLM thought of". This keeps feedback finite, fair and exam-realistic.

**4.6 Missing ≠ unknown.** A gap in the brainstorm is evidence about *recall under time pressure*, not about knowledge. The profile records it that way (§6).

---

## 5. Feedback format

### Attempt 1 feedback (no score, no model answer, no full map)

```
1. WHAT YOU GOT RIGHT            2–3 items, quoting their own words
2. DECODE CHECK                  only if gate < 2
3. THINK FURTHER                 max 3 gaps, core first, each at the lowest hint level that might work
4. SHARPEN                       max 2 'label' points -> one question each
5. CHECK THIS FACT               only if an inaccurate anchor was found
6. YOUR TURN                     "Add to your brainstorm. 60 seconds. Don't rewrite — add."
```

The cap of 3 gaps + 2 sharpening prompts is deliberate. Five items is what a student can act on in 60 seconds. Twelve items turns the feedback into a model answer.

### Hint ladder (per missing dimension)

| Level | What it does | Gives away |
|---|---|---|
| **L1 Trigger** | Points at the words in the question that should have activated the dimension | Nothing but the cue |
| **L2 Direction** | Names the *area* to think about, as a question | The area, not the point |
| **L3 Anchor** | Points to *their own notes* (heading/page), or to a concrete situation if no notes | Where to look |
| **L4 Reveal** | States the dimension, why it's relevant, and how the trigger should have led there | Everything; only after attempt 2 |

Default: L1 for core gaps on attempt 1. Escalate only when the same gap survives the next attempt. The hint level that finally worked is recorded; it's one of the most useful signals in the profile.

### Attempt 2 → delta report (now show the band)

For each new point, classify:
- **Hinted & incorporated**: maps to a dimension that got a hint.
- **Independent**: maps to a dimension that got *no* hint. **This is the strongest signal in the whole system.** Highlight it to the student.
- **Parroted**: restates the hint's words without adding substance (the hint said "think about the legislative process" and the student wrote "legislative process"). Isn't credited as recall.

Then L4 reveal for whatever's still missing, each with the trigger explained. The lesson to leave with is "when I see *contentious*, I look for recent disputes", not "the answer had five more points".

---

## 6. Using personal notes without confusing *studied* with *demonstrated*

### Five rules

1. **Notes never add to the score.** The score reflects only what is on the page.
2. **Notes decide hint *type*, not hint *existence*.** Every core gap gets a hint. Notes decide whether L3 says "your notes on X, p.6 cover this" or offers a concrete situation instead.
3. **Absence in notes is not a gap in knowledge.** If notes have nothing, the hint stays conceptual, and after the session the student gets a one-tap question: *"Did you know this? Yes / Vaguely / No."* Cheap, honest, and it stops false gap claims.
4. **Point at notes; don't quote them.** L3 names the heading or page, not the content. Quoting the line turns a recall exercise into reading.
5. **Notes are not verified truth.** Coaching notes contain errors. If a note conflicts with a verified fact in the map, the hint uses the verified fact and flags the note.

### Evidence events → profile

Store an append-only event log per concept/dimension; derive state from it. As a developer you'll find this easier to debug and re-score than a mutable "mastery" number.

| Event | Fired when |
|---|---|
| `studied` | concept found in uploaded notes, or self-reported |
| `missed` | core/strong dimension absent in an independent attempt |
| `recalled_prompted(level)` | appears on attempt 2 after a hint at L1/L2/L3 |
| `recalled_independent` | appears unprompted, at ≥ `specific` quality |
| `applied` | `recalled_independent` + `analytical` quality + in a question **not seen before** and not a near-duplicate of a practised one |
| `self_report(yes/vaguely/no)` | one-tap answer after the session |

Derived states: `no_evidence → studied → recalled → applied`, each with **confidence = number of independent observations, decayed by age**. One lucky recall is not mastery.

The diagnostic gold for the product is **`studied` + repeated `missed` + `recalled_prompted(L1)`**: the student knows it and the cue didn't fire. That's the "Oh, I could have thought of that" moment from the brief, and this pattern is how you detect it.

---

## 7. Pipeline (2 LLM calls + code)

```
[offline] Question Map  (human-reviewed YAML)
     │
1. Transcribe   vision model → text; student confirms/edits     (MVP: typed input first)
2. ANALYSE      LLM call #1, temperature 0, strict JSON:
                decode-check + atomic points + labels against the map
3. SCORE        plain code: gate, A–F, flags, ranked gaps
4. RETRIEVE     notes for each gap   (MVP: one topic's notes fit in context — no vector DB yet)
5. FEEDBACK     LLM call #2: labels + gaps + hint levels + notes refs → feedback JSON → rendered text
6. Attempt 2    repeat 2–3, plus diff (hinted / independent / parroted)
7. PROFILE      plain code: append evidence events
```

Why analysis and feedback are separate calls:
- Step 2's output can be **compared directly to human labels**. That's your evaluation harness.
- Step 3 is deterministic, so the same labels always give the same score.
- Feedback tone can be iterated without touching scoring.

---

## 8. Worked example (`gs2-fed-001`)

**Student decode:** "Discuss Gov const position + examine issues"
**Student brainstorm (90s, simulated, typical repeat aspirant):**

```
Intro: Gov – const head, Art 153
- appointed by Pres, pleasure (156)
- discretion Art 163
- agent of centre
- misuse 356 – Bommai
- political appointees
- delay in bills – TN, Kerala
- powers: pardon, ordinance, exec, legis
- Sarkaria comm.
- Way fwd: reforms
```

**Labels (abridged):**

| Point | maps_to | rel | quality | note |
|---|---|---|---|---|
| const head, Art 153 | dual_role | relevant | specific | |
| agent of centre | dual_role | relevant | label | not linked to "const head", so the tension isn't drawn |
| appointed… pleasure (156) | appointment_tenure | relevant | specific | |
| discretion Art 163 | discretionary_powers | relevant | label | no situation named |
| misuse 356 – Bommai | art356 | relevant | specific | link to floor test not on page; don't infer it |
| political appointees | politicisation | relevant | label | |
| delay in bills – TN, Kerala | assent_to_bills | relevant | specific | examples count as evidence |
| powers: pardon, ordinance… | anti-pattern | tangential | label | chapter recital |
| Sarkaria comm. | commission_reforms | relevant | label | name only, no recommendation |
| Way fwd: reforms | conventions_timelines? | relevant | label | too vague to map confidently, so treated as D3 coverage only |

**Computed:** Gate 2 · A 3 (D4 is implicit via "agent of centre", weak) · B: core 4/5 (missing **govt_formation**), strong 3/5 → 0.73 → **2** · C 4/9 specific ≈ 44% → **1** · D: Art 153, 156, 163, 356, Bommai, TN/Kerala → **3** · E 9/10 → **3** · F: intro + way-forward seed, ungrouped → **1**.

**Rendered feedback (attempt 1):**

> **What you got right**
> - You opened with the Governor as "const head" and also wrote "agent of centre". That pair is the heart of this question.
> - "delay in bills – TN, Kerala" is exactly the kind of recent, named example "increasingly contentious" is asking for.
>
> **Think further**
> - The question says the role has become *contentious*. You've covered contention in law-making. Where else does a Governor make a call that can decide *which party governs* a state? *(L1)*
>
> **Sharpen**
> - You wrote "Sarkaria comm." In the exam the name alone earns little. Which *one* Sarkaria recommendation answers an issue already on your list?
> - You wrote both "const head" and "agent of centre". Can you join them with an arrow that states the tension?
>
> **Your turn.** 60 seconds. Add, don't rewrite. Also: "powers: pardon, ordinance…" — is any of these actually *contentious* today? If not, would it earn its words in 250?

(One gap, because there was only one core gap; strong-tier gaps aren't used to fill slots on attempt 1. Two sharpens, chosen because they turn existing `label` points into the most marks. "discretion Art 163" is held back for the next round.)

**L3 with notes**, if the gap survives: *"Your notes `Polity_Governor.pdf`, p.6, have a section 'Situational discretion'. What situations does it list?"*
**L3 without notes:** *"Think about Maharashtra after the 2019 results."*

---

## 9. How to know the evaluator works

You can do this before any UI exists, in a notebook.

1. **Build 3 maps** from *real* PYQs, copied verbatim from UPSC question papers (not coaching paraphrases). Verify every `evidence` item against the Bare Act, the judgment text or the commission report, and flip `verified: true`. My factual entries in the sample map are a starting point and must be checked, especially the 2025 judgments.
2. **Collect 10–15 brainstorms**: your own timed attempts, plus 2–3 aspirant friends. Typed is fine.
3. **Two humans label independently**: you and one other experienced aspirant, using §1–§3. Measure agreement on `relevance`, `quality` and `maps_to`.
4. **Run the analysis prompt** and compare it to the human labels. Target: LLM–human agreement close to human–human agreement. Where they disagree, fix the map first and the prompt second.
5. **Track the false-gap rate**: dimensions the LLM calls missing that the student actually wrote (abbreviation, OCR, odd phrasing). This is the error that destroys trust fastest.
6. **Learning test (the real one):** same student, question A with the hint ladder → a few days later, *unseen* question B on an adjacent theme. Did independent core coverage go up? Compare against a control where the student was shown a plain list of points. If the ladder doesn't beat the list on transfer, the core thesis needs revisiting.

---

## 10. Confirmed vs hypothesis

| Confirmed (design decision) | Hypothesis (must validate) |
|---|---|
| Student decodes before the AI does | 15s decode is enough and doesn't annoy users |
| LLM labels points; code computes scores | 0–3 anchors are reliable enough between humans |
| Human-reviewed question maps are the reference | LLM-drafted maps need only light review later |
| Notes never add to score | Notes-based L3 hints beat generic L3 hints |
| No score shown on attempt 1 | Withholding the score improves attempt-2 effort |
| Max 3 gaps + 2 sharpens per round | That cap is the right size |
| `applied` requires an unseen question | Star-your-top-N is a valid prioritisation signal |
| | Hint ladder beats a model-points list on transfer |

---

## 11. Next 3–4 weeks at 1–2 hrs/day

| Week | Output |
|---|---|
| 1 | Pick 3 real PYQs on federalism/Centre-State. Write and verify 3 question maps. |
| 2 | Collect 10–15 brainstorms; hand-label with one other aspirant. |
| 3 | Analysis prompt + JSON schema + Python scoring script; compare to labels. |
| 4 | Feedback prompt; Wizard-of-Oz with 3 aspirants (feedback sent over WhatsApp, attempt 2 photographed back). Measure attempt-1 → attempt-2 → unseen-question change. |

No frontend, no database, no handwriting pipeline until week 4 shows the loop teaches something.
