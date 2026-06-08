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

*(All items explicitly deferred to post-v1.0.0 at Stage 9 — see rationale per item in Resolved.)*

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
| B-002 | CI now runs a five-target cargo-deny matrix using `cargo metadata --all-features --filter-platform <target>` and `cargo deny --all-features check --metadata-path …`. This covers Linux, macOS, and Windows release targets without penalizing inactive Linux-only notify-rust transitive target deps. Local filtered checks passed for all five targets. | Stage 8 |
| B-005 | Release packaging declares both Cargo binaries (`ccplan`, `ccplan-fire`); `dist plan --allow-dirty` shows both binaries in every platform archive and the Windows MSI. `wix/main.wxs` is generated and tested for both `.exe` entries. | Stage 8 |
| B-003 | **Deferred post-v1.0.0 (Stage 9).** Verify macOS LaunchAgent path and Windows Task Scheduler XML on real interactive sessions, including notification delivery and `ccplan-fire.exe` no-console behavior. The dev machine is Linux; macOS/Windows backends are CI-verified (build + unit tests in platform::format) but manual dogfood is not possible without access to those OSes. Documented limitation; tracked for post-v1.0.0 when a macOS/Windows runner or machine is available. | Stage 9 (deferred) |
| B-004 | **Deferred post-v1.0.0 (Stage 9).** Manual `ccplan fire` without scheduler-injected D-Bus env: Rust 2024 makes `std::env::set_var` unsafe; current supported path always injects env from `apply`. If standalone fire without apply-set env becomes a product requirement, file as post-v1.0.0 feature. | Stage 9 (deferred) |
| B-009 | **Deferred post-v1.0.0 (Stage 9).** launchd label full §6.1 grammar conformance test. The existing `launchd_label_and_calendar_interval` test in `platform::format` asserts the `io.ccplan.` prefix and the format function output. A more explicit §6.1-grammar assertion (pinning the exact spec text) would be additive; deferred as a P3 quality improvement. | Stage 9 (deferred) |
| B-010 | **Deferred post-v1.0.0 (Stage 9).** launchd scheduling is minute-granular (`<Second>` ignored) and recurring (no year — relies on `cleanup_after_fire` self-bootout for one-shot). This limitation is documented in DESIGN §6.1 and `development/notes.md` §3. Verification on real macOS overlaps B-003; deferred together. | Stage 9 (deferred) |
| B-013 | **Deferred post-v1.0.0 (Stage 9).** `done`/`skip`/`rm` have no `--date` (hardcode today). This is a UX gap discovered in PR #1; the commands work correctly for today; `--date` parity is a post-v1.0.0 enhancement. | Stage 9 (deferred) |
| B-015 | **Deferred post-v1.0.0 (Stage 9).** `is_lock_contention` maps `PermissionDenied` → `Locked`, misreporting a genuinely unreadable lockfile. The function is under fn-level `coverage(off)` as genuine OS-IO; narrowing the classification is a defensive improvement, not a correctness bug for normal use. Deferred as P3. | Stage 9 (deferred) |
| B-016 | **Deferred post-v1.0.0 (Stage 9).** `status` can overcount triggers when a trigger self-fires/auto-cleans on the OS side without `apply` or `clear` reconciling `triggers.json`. The UI inaccuracy is cosmetic; `apply` always converges correctly. Deferred as P3 with a note that accurate `status` reconciliation is a post-v1.0.0 quality improvement. | Stage 9 (deferred) |
