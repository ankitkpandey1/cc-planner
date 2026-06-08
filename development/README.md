# development/ — dev-only scaffolding

These files drive and record the build of `ccplan`. They are **not part of the shipped product** and
may be removed (or kept out of release artifacts) before/at the open-source release.

Read them in this order before doing any implementation work:

1. **`notes.md`** — durable project memory: locked decisions, pinned toolchain/deps, platform
   gotchas, and a running log. *Read first, every session* (survives context compaction).
2. **`implementation_checklist.md`** — the single authoritative build plan: the goal, the autonomous
   loop, the per-stage rhythm (recon → implement → self-review → reflect), project structure, coding /
   pattern / test conventions, and the staged checklist (Stage 0 → ship).
3. **`backlog.md`** — the dedicated place for tasks *discovered during* implementation. The stage
   list stays stable; discoveries land here.
4. **`audit_log.md`** — one evidence-backed entry per completed stage, proving it was done as specified.

The product spec itself lives at the repo root in **`DESIGN.md`** (source of truth for *what*).
