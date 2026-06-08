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
| B-006 | 4 (PR #1 review M1) | **P1** | **Default `notify` lead is 0 → double-notify.** `Lead::default()`=0 + `notify_at = start - 0 = start` schedules a `notify` trigger coincident with `start`, and both `fire(notify)` and `fire(start)` notify. Implement Inv-16: omit the `notify` trigger when `notify_at >= start`; wire `config.notify.default_lead` (default `5m`). Add a test asserting exactly one notification/trigger at lead 0. | Correction pass before ship |
| B-007 | 4 (PR #1 review M2) | **P1** | **Concurrent `add`/`edit` can drop a concurrent writer's non-terminal block.** Command layer loads *outside* the lock, then `set_plan` locks + merges (Preserve keeps only terminal blocks). Implement Inv-17: a `Store::update(date, closure)` that loads→mutates→writes under one held lock; route `add`/`edit`/`rm`/`done`/`skip` through it; forbid unlocked load-then-`set`. Add a concurrent-additive-edit test. | Correction pass before ship |
| B-008 | 5 (PR #1 review M3) | **P1** | **Coverage honesty: pure logic hidden behind module-scope `coverage(off)`.** `platform/{systemd,schtasks,launchd}.rs` exclude their pure helpers (`parse_timer_units`, `parse_task_names`, `xml_escape`, `quote_windows_arg`, `systemd_calendar`, `windows_boundary`, `start_calendar_interval`, `unit_name`/`label`, `is_missing_unit`, `is_lock_contention`) — 0% tested under a green 100%. Move them to coverage-on submodules with unit tests; keep `coverage(off)` on IO methods only. The new Anti-gaming guard #1 must pass. | Correction pass before ship |
| B-009 | 5 (PR #1 review) | P3 | launchd label conformance test. DESIGN §6.1 updated to the hyphenated `io.ccplan.<date>-<idhash>-<rev>-<event>` (matches the shared `backend_id`); add a per-platform identity conformance test asserting the §6.1 grammar so impl/spec can't drift again. | Stage 5 correction |
| B-010 | 5 (PR #1 review) | P2 | launchd scheduling is minute-granular (`<Second>` ignored) and recurring (no year) — one-shot relies on `cleanup_after_fire` self-bootout. Verify on real macOS (overlaps B-003); document the precision limit (DESIGN §6.1 done) and confirm self-bootout runs on no-op fires. | Before v1.0.0 |
| B-011 | 4 (PR #1 review) | P3 | **Reads mutate + take the write lock.** `now`/`next`/`agenda` → `reconciled_plan` → `set_plan` when overdue, so a read can fail "store locked" against a writer. Implement Inv-18: reconcile-on-query in memory only; persist solely on `apply`/`fire`/mutation. Add a "read leaves file byte-identical while writer holds lock" test. | Correction pass before ship |
| B-012 | 4 (PR #1 review) | P2 | **Human output for `now`/`next`/`agenda` is unusable** — non-`--json` prints `"N item(s)"`/`"[]"` instead of the blocks. Render a scannable human table; add an `insta` human-output snapshot (not just `--json`). | Correction pass before ship |
| B-013 | 4 (PR #1 review) | P3 | `done`/`skip`/`rm` have no `--date` (hardcode today). Add `--date` for parity with `add`/`edit`/`show`; assert via the §8 flag-set conformance test. | Stage 4 correction |
| B-014 | 3 (PR #1 review) | P2 | Store unit test `atomic_write_replaces_existing_file` writes a relative path into CWD instead of `assert_fs::TempDir` — pollutes CWD, fragile under parallel/read-only runs. Use a temp dir. Anti-gaming guard #2 must pass. | Correction pass before ship |
| B-015 | 3 (PR #1 review) | P3 | `is_lock_contention` maps `PermissionDenied` → `Locked`, misreporting a genuinely unreadable lockfile; also untested (currently under module-scope `coverage(off)`). Narrow the classification + unit-test it (folds into B-008's extraction). | Stage 3 correction |
| B-016 | 5 (PR #1 review) | P3 | `status` can overcount triggers: `triggers.json` isn't updated when a trigger self-fires/auto-cleans on the OS side. Reconcile owned vs live in `status` (the ledger↔OS System-Invariant row). | Before v1.0.0 |

## Resolved

| ID | Description | Closed by |
|----|-------------|-----------|
| B-001 | Decided `directories = "5"` vs `directories = "6"` for storage. Stage 3 uses `directories = "6"`; `directories 5.0.1` still pulled `option-ext` and introduced duplicate older transitive versions, while `6.0.0` keeps the duplicate graph clean. `deny.toml` explicitly allows OSI-approved `MPL-2.0` for `option-ext`. | Stage 3 implementation commits `50ef2c1` + `4afacd2` |
