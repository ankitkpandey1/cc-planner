# /goal — autonomous build driver

> **Usage:** `/goal read development/goal_prompt.md`
>
> This file is the evergreen entry point for autonomously building `ccplan`. It deliberately contains
> **no snapshot of "current state"** (which would go stale) — it tells you how to *derive* the current
> state from the live docs and continue. Anyone can update `DESIGN.md`, the checklist, the backlog, or
> the conventions, and this prompt keeps working.

---

You are implementing `ccplan` autonomously. The repo is the single source of truth — do not improvise
architecture, and do not invent state. Everything you need is in the files below.

## GOAL
Drive the project to a production-ready, shippable **v1.0.0**, exactly as specified in
`development/implementation_checklist.md`. You are done only when its **Final Ship Gate** passes.

## STEP 1 — Load memory (in this order, before doing anything else)
1. `development/notes.md` — durable decisions, gotchas, learnings. Read in full.
2. `development/backlog.md` — open discovered tasks (note every `P1`/blocker).
3. `development/implementation_checklist.md` — the authoritative build plan and per-stage rhythm.
4. `Reviews.md` — past code reviews + post-mortems: the bug classes that already bit us and why.
5. `DESIGN.md` and `CONVENTIONS.md` — the spec and the coding standard.
Then read the `DESIGN.md` sections relevant to whatever you determine is next.

## STEP 2 — Derive the current state yourself (don't assume)
- **Run the previous stage's Acceptance Gate + the global Definition of Done gate** (and the
  Anti-gaming guards) to see the real red/green state right now. A green CI badge is not proof —
  run the gates locally.
- Read the checklist **Progress tracker** and stages to find the first stage whose boxes aren't all
  checked. That, plus any failing gate, tells you where you actually are.

## STEP 3 — Clear blockers before starting new stage work
The regression gate must be green before you build forward, so **resolve open `P1`/blocker backlog
items and any failing DoD/Anti-gaming guard first**, as their own DoD-green commits. If `Reviews.md`
has unresolved MAJOR findings, they are tracked in `backlog.md` — clear them (or, only with explicit
written rationale in the backlog, defer them) before continuing. Never build a new stage on top of a
red gate or a known-broken predecessor.

## STEP 4 — Work the loop
Follow the checklist's **"HOW TO USE THIS CHECKLIST (autonomous loop)"** and the mandatory
**PER-STAGE RHYTHM** precisely, for every stage, until the Final Ship Gate passes:

> A. Recon/research → B. Implement (strict TDD) → C. Self-review & fix → D. Definition of Done gate
> → E. Self-reflect & record learnings (`notes.md`) → F. Capture discovered tasks (`backlog.md`)
> → G. Audit entry (`audit_log.md`) → H. Commit.

Do not skip or collapse phases. Honor every rule already written in the checklist and conventions —
in particular:
- **DoD gate fully green each stage:** `cargo fmt --check`, `clippy --all-targets --all-features
  -D warnings`, tests, `RUSTFLAGS="--cfg coverage_nightly" cargo +nightly llvm-cov --fail-under-lines
  100`, `cargo deny check`, release build.
- **Test-count guard:** tests actually ran — `0 failed`, `0 filtered out`, count non-decreasing; no
  `#[ignore]` outside the sanctioned real-OS integration tests.
- **Anti-gaming guards:** no module-scope `coverage(off)`; tests never write to the real temp dir.
- **System Invariants** (the cross-stage Inv-* in the checklist): each needs a real test.
- **Test the behavior, not the line**; add **conformance tests** binding code to DESIGN §6.1/§8;
  assert human output, not just `--json`.
- **Strong typing, no `unsafe`, comments explain WHY not WHAT, "don't make me think" CLI UX**,
  parse-don't-validate, DI via traits.

## Ground rules
- **Branch:** all work on `dev`. Never commit to `main` before the Final Ship Gate.
- **Commits:** Conventional Commits; one stage = one or a few focused commits. **No commit trailers**
  — no `Co-Authored-By`, no "Generated with…" lines.
- **Backlog discipline:** anything you discover that's out of the current stage's scope goes into
  `backlog.md` (with an ID + priority), never inlined into the stage list.
- **Honesty:** if a gate fails, say so and fix it; never weaken a guard, exclude business logic from
  coverage, or `#[ignore]` a test to go green. If a rule genuinely must bend, that's a backlog item
  with written rationale.

## Begin
Load memory (Step 1), derive state (Step 2), report a short plan (what's the current stage, what
gates are red, what blockers you'll clear first), then execute the loop — committing after each
green stage with its audit entry.
