# ccplan — Discovered-Task Backlog

> **The dedicated place for tasks discovered *during* implementation.**
> The stage list in `implementation_checklist.md` is the **stable plan** — do not edit it to add
> work you find along the way. Instead, whenever self-reflection (per-stage rhythm phase F) surfaces
> something real but out of the current stage's scope — a refactor, a missing test, a risk, a
> follow-up, a doc gap, a better idiom found in recon — **append it here**.
>
> Then triage it: do it now if it belongs to the current stage, schedule it into a later stage by
> noting the stage, or defer it to post-`v1.0.0`. Nothing discovered is ever silently dropped.

## How to use
- Add an item with the table row format below. Newest at the bottom of the Open section.
- Give every item an ID (`B-001`, `B-002`, …), the stage it was found in, a priority
  (`P1` blocker / `P2` should-fix-before-ship / `P3` nice-to-have / `later` post-v1.0.0), and a clear
  action.
- When you act on an item, move it to **Resolved** with the commit/stage that closed it.
- Reference backlog IDs from `audit_log.md` entries when a stage raises or closes them.

## Open

| ID | Found in stage | Priority | Description / action | Target |
|----|:--------------:|:--------:|----------------------|--------|
| B-002 | 5 | P2 | Add explicit macOS/Windows dependency-policy coverage once CI can run target-aware `cargo deny` without penalizing inactive Linux-only dependencies. Current Stage 5 deny gate is scoped to `x86_64-unknown-linux-gnu`, matching the Ubuntu CI job. | Stage 8 / CI hardening |
| B-003 | 5 | P2 | Verify the macOS LaunchAgent path and Windows Task Scheduler XML on real interactive OS sessions, including notification delivery from scheduled `fire` and Windows `ccplan-fire.exe` no-console behavior. | Before v1.0.0 ship gate |
| B-004 | 5 | P3 | Decide whether manual `ccplan fire` without scheduler-injected D-Bus env should be supported on Linux. If yes, implement notification sending with an explicit bus address instead of mutating process env, because Rust 2024 makes `std::env::set_var` unsafe. | Post-Stage 6 polish |
| B-005 | 5 | P2 | Ensure Stage 8 packaging/release tooling includes both `ccplan` and the Windows `ccplan-fire` wrapper where needed. | Stage 8 release packaging |
| B-009 | 5 (PR #1 review) | P3 | launchd label conformance test. DESIGN §6.1 updated to the hyphenated `io.ccplan.<date>-<idhash>-<rev>-<event>` (matches the shared `backend_id`); add a per-platform identity conformance test asserting the §6.1 grammar so impl/spec can't drift again. (B-008 added a basic `launchd_label` unit test in `platform::format`; the full §6.1 grammar conformance test is still open.) | Stage 5 correction |
| B-010 | 5 (PR #1 review) | P2 | launchd scheduling is minute-granular (`<Second>` ignored) and recurring (no year) — one-shot relies on `cleanup_after_fire` self-bootout. Verify on real macOS (overlaps B-003); document the precision limit (DESIGN §6.1 done) and confirm self-bootout runs on no-op fires. | Before v1.0.0 |
| B-013 | 4 (PR #1 review) | P3 | `done`/`skip`/`rm` have no `--date` (hardcode today). Add `--date` for parity with `add`/`edit`/`show`; assert via the §8 flag-set conformance test. | Stage 4 correction |
| B-015 | 3 (PR #1 review) | P3 | `is_lock_contention` maps `PermissionDenied` → `Locked`, misreporting a genuinely unreadable lockfile; also untested (it stays under fn-level `coverage(off)` as genuine IO after B-008). Narrow the classification + unit-test the pure part. (B-008 converted the platform IO to fn-level `coverage(off)` but did not change `is_lock_contention` semantics.) | Stage 3 correction |
| B-016 | 5 (PR #1 review) | P3 | `status` can overcount triggers: `triggers.json` isn't updated when a trigger self-fires/auto-cleans on the OS side. Reconcile owned vs live in `status` (the ledger↔OS System-Invariant row). | Before v1.0.0 |

## Resolved

| ID | Description | Closed by |
|----|-------------|-----------|
| B-001 | Decided `directories = "5"` vs `directories = "6"` for storage. Stage 3 uses `directories = "6"`; `directories 5.0.1` still pulled `option-ext` and introduced duplicate older transitive versions, while `6.0.0` keeps the duplicate graph clean. `deny.toml` explicitly allows OSI-approved `MPL-2.0` for `option-ext`. | Stage 3 implementation commits `50ef2c1` + `4afacd2` |
| B-006 | Double-notify. Inv-16: notify trigger omitted when `notify_at >= start`; `config.notify.default_lead` (5m) wired. Trigger-scheduling fix landed in Stage 6 (`b3ab747`); the lead-0 trigger-omission acceptance test added in `9d16ad8`. | `b3ab747` + `9d16ad8` |
| B-007 | Concurrent `add`/`edit` could drop a writer's block. Added `Store::update(date, default_lead, closure)` holding the lock across load→mutate→write (Inv-17); routed `add`/`edit`/`rm`/`done`/`skip` and apply's overdue reconciliation through it; 8-thread no-lost-write test added. | `760d6c8` |
| B-008 | Coverage honesty. Pure helpers moved to a coverage-on `platform::format` module (gated `any(target_os=…, test)` so they compile and are unit-tested on every host, incl. the Linux coverage job); backends keep only IO with fn-level `coverage(off)`. Anti-gaming guard #1 passes. Linux side gate-verified; schtasks/launchd CI-verified on Windows/macOS. | `03c6b20` |
| B-011 | Reads no longer persist or take the write lock (Inv-18): `now`/`next`/`agenda` reconcile in memory via `read_reconciled_plan`; `apply --dry-run` previews without writing; only real `apply`/mutations persist. Byte-identical-read test added. | `cd42d8d` |
| B-012 | Human output for `now`/`next`/`agenda` now renders a scannable, headed, column-aligned table (with a countdown column for agenda); empty results print plain language. Pure table/countdown/label helpers unit-tested; integration test asserts human output. (Used a deterministic string assertion rather than `insta`.) | `cd42d8d` |
| B-014 | Store `atomic_write` test uses `assert_fs::TempDir`; empty-parent branch covered via `ensure_parent` directly. No CWD pollution; anti-gaming guard #2 passes. | `1add9be` |
