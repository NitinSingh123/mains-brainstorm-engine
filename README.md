# Mains Brainstorm Engine

A brainstorming coach for UPSC Mains aspirants. It trains points to come to
mind the moment you read a question, through repeated timed practice and
feedback based on what *you* missed.

**Status:** prototype planning. First topic: GS2 – Federalism / Centre-State Relations.

## Core loop

Student history → pick a PYQ → student brainstorms (2 min) → feedback (hits,
2–3 nudges, second attempt, then reveal) → update history.

## Layout

| Path | What |
|---|---|
| `docs/` | Design notes |
| `knowledge/federalism/` | The UPSC-specific knowledge pack (in our own words) |
| `knowledge/federalism/answer-keys/` | One reviewed answer key per PYQ |
| `prompts/` | Engine instructions (later) |
| `app/` | Simple web prototype (later) |

## Rules

- Coaching PDFs and books stay **outside** this repo (`sources/` is ignored).
- Student data never goes in git.
- Every fact in the knowledge pack starts `verified: false` until checked against a primary source.
