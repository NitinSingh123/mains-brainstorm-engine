# Handover — read this first

Context for continuing this project in a new session (written 2026-11-07).

## Founder
- Software developer (~4 yrs), full-time job; works on this 1–2 hrs/day on a **personal laptop**, outside work hours.
- Multiple UPSC attempts. Not attempting 2027; may attempt 2028, deciding after Mains 2026 marks (~April 2027). **Nov 2026 – Apr 2027 is the build/validate window.**
- Prefers simple, plain-language discussion of the core loop. Avoid going deep into implementation unless asked.

## Product (current direction)
A brainstorming coach for UPSC Mains. Founder's insight: coaching institutes now hand out 2-minute brainstorming sheets; the skill is points *flashing* instantly on seeing a question, built by repetition (muscle memory). Product = timed practice + feedback on what *this* student missed + bringing misses back until they fire automatically. Not a model-answer generator.

**Core loop (focus only on this for the prototype):**
Student history → pick a PYQ for a weak/untested area → student brainstorms (2 min, typed) → feedback (what you got right, 2–3 nudges without giving away the answer, second attempt, then reveal + which words in the question should have triggered the missed points) → update history.

## Decisions so far
- First topic only: **GS2 Federalism / Centre-State Relations.**
- "Smart" = a curated **knowledge pack** the AI reads each time, not a trained model. One topic fits in the AI's context, so no vector DB/RAG yet.
- Value lies in what general AI lacks: what UPSC actually asks (PYQ patterns), exam-realistic level of points, verified accuracy, recent developments.
- Start with **real PYQs** (exact wording from upsc.gov.in papers); generate new questions only later.
- Every fact starts `verified: false` until checked against a primary source.
- Coaching material is used to build understanding; the pack is written in our own words. Source PDFs stay outside git.
- Student history (simple): per concept a status — not seen → studied → recalled independently → applied to a new question; plus cross-topic habits (e.g. "skips way forward"); plus question history. Recall after a nudge is tracked separately from independent recall.
- Two practice modes later (deep single-question; rapid 5–7 question drill) sharing one evaluation core — see `02-practice-modes-architecture.md`. Not needed for the first prototype.
- Expansion path later: all GS papers → State PSC Mains (incl. Hindi) → other descriptive exams. Not "every exam"; not a general prep app.

## Knowledge pack to build (`knowledge/federalism/`)
1. `syllabus.md`: exact GS2 syllabus lines + in-scope subtopics (~1 hr)
2. `pyqs.yaml` + `pyq-patterns.md`: all federalism PYQs 2013–2025, tagged (year, subtopic, directive, demands) + one-page pattern analysis (~6–8 hrs). **Highest value.**
3. `concept-map.md`: concepts, Articles, cases, committees, examples, arguments (~5–6 hrs)
4. `answer-keys/`: one per PYQ, points ranked must-have/good/bonus with trigger words (~10 min each). Format example: `gs2-fed-001-governor.yaml`
5. `style-guide.md`: what "exam-realistic" means (~1 hr)

AI drafts, founder reviews.

## Plan
| Weeks | Step |
|---|---|
| 1–3 | Build the knowledge pack |
| 4 | Very simple web page (e.g. one-file Streamlit): name login, question, 2-min timer, typed brainstorm, feedback, second attempt, saved history |
| 5 | Founder uses it for a week, then 3 aspirant friends |
| 6–7 | 10–20 aspirants. Watch: do they return unprompted? Do first-attempt brainstorms improve? |

Continue if: ≥5 of 8 testers do the second attempt; ≥3 return for a third session within a week without being chased; most improve on an unseen question; testers say it's clearly better than a ChatGPT prompt; ≥3 pre-pay something (₹99–499). Stop or rethink if most ask for model answers and skip second attempts, or nobody returns unprompted.

Key risks: students may prefer model answers over hints; free ChatGPT as the real competitor; the rate of writing answer keys limits the product more than code does; distribution runs on trust (Telegram/YouTube) — consider a free "daily brainstorm question" Telegram channel early.

## Immediate next step
Collect federalism-related GS2 PYQs (2013–2025) with exact wording from official UPSC papers → AI does first-pass tagging and pattern analysis → founder corrects.

## Docs
- `01-brainstorm-evaluator-v1.md`: detailed rubric/feedback design (more detail than the prototype needs; reference only)
- `02-practice-modes-architecture.md`: engineering design for later stages (reference only)
