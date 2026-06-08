# ccplan — Code Reviews

> Durable record of PR/code reviews and the post-mortems that explain *why* issues slipped
> through, so the plan (`development/implementation_checklist.md`, `DESIGN.md`) can be hardened.

---

## PR #1 — "Initial work" (dev → main, Stages 0–5) — 2026-06-08

Reviewed at `2fc7c70`. CI green on Linux/macOS/Windows. Architecture conforms to `DESIGN.md` and
`CONVENTIONS.md`: pure core (`model`/`time`/`lifecycle`), DI via `Clock`/`Scheduler`/`Notifier`,
parse-don't-validate newtypes, `#![forbid(unsafe_code)]`, argv-only OS calls, centralized
error→exit-code mapping. Test discipline is real (92 tests + 4 property suites; `#[ignore]` only on
the sanctioned integration test; no real-clock/scheduler leakage; no vacuous `is_ok`-only tests).

### Process note (not blocking, per maintainer)
PR merges `dev` at **Stage 5 of 9** into `main`, which contradicts the plan's branch model
("never commit to `main` before the Final Ship Gate"). It would ship a planner where `run:`
automation is a no-op (Stage 6), completions are stubbed (Stage 7), and no release pipeline exists
(Stage 8). Acknowledged as acceptable for now to keep momentum.

### 🔴 MAJOR — correctness bugs

**M1. Default `notify` lead is 0 → every block double-notifies at start.**
`Lead::default()` is 0s (`model.rs:691`); `add` uses `notify.unwrap_or_default()`. In
`desired_triggers` (`commands.rs:412-421`), `notify_at = start - lead`, so at lead 0 the **Notify**
and **Start** triggers land on the *same instant*. At fire, `Notify` sends a notification and
`Activate{notify:true}` (`lifecycle.rs:154`) sends another → two desktop notifications at the same
second, plus a redundant OS trigger, for every default block.
*Fix:* skip the Notify trigger when `notify_at <= start`, or default `Lead` to a non-zero value.
Add a test asserting exactly one trigger/notification when lead is 0.

**M2. Concurrent `add`/`edit` can silently drop a concurrent writer's non-terminal block.**
Command layer reads the plan *outside* the lock (`load_required`, `commands.rs:99/133`), mutates in
memory, then calls `set_plan`, which only locks *inside*. `merge_plan` under `Preserve` retains only
**terminal** blocks from disk (`store.rs:234-258`); a non-terminal block written by a concurrent
invocation between your read and your locked write is absent from your stale `incoming` and is
**lost**. ccplan is built to be agent-driven, so parallel invocations are realistic.
*Fix:* hold the lock across read-modify-write (a `Store::update(date, |plan| …)` closure that loads
*inside* the lock); forbid unlocked load-then-`set` in the command layer. Add a concurrent-edit test.

### 🟠 MAJOR — coverage honesty (violates the DoD coverage rule)

**M3. Pure logic hidden behind module-level `coverage(off)`.**
`platform/{systemd,schtasks,launchd}.rs` each carry a **module-level**
`#![cfg_attr(coverage_nightly, coverage(off))]`, excluding their *pure* helpers too:
`parse_timer_units`, `parse_task_names`, `xml_escape`, `quote_windows_arg`, `systemd_calendar`,
`windows_boundary`, `start_calendar_interval`, `unit_name`/`label`, `is_missing_unit`,
`is_lock_contention`. These parse external command output and build XML — exactly where bugs hide —
and are currently 0% tested, while `--fail-under-lines 100` stays green because excluded lines don't
count as missed. `mod.rs` already does this right (`fire_args`/`trigger_identity`/doctor rendering
are coverage-*on* and tested) — mirror it.
*Fix:* move pure helpers into a coverage-*on* submodule with unit tests; keep only the
`Command`-executing methods `coverage(off)`.

### 🟡 MINOR
- **launchd label off-spec.** DESIGN §6.1 says `io.ccplan.<date>.<idhash>.<rev>.<event>` (dots);
  code emits the hyphenated `backend_id` (`launchd.rs:236`). Fix label or update DESIGN.
- **launchd second-precision is dead + agents recur annually.** `StartCalendarInterval` ignores
  `<Second>` (minute-granular) and has no year field, so the agent re-fires yearly and relies on
  `cleanup_after_fire` self-bootout. Works, but contradicts the "1s accuracy" goal — document the
  platform limitation.
- **Reads mutate + take the lock.** `now`/`next`/`agenda` → `reconciled_plan` → `set_plan` when
  overdue (`commands.rs:400`), so a read can fail with "store locked" against a concurrent writer.
  Reconcile in-memory for pure reads; persist only on `apply`/explicit mutation.
- **Human output for `now`/`next`/`agenda` is unusable** — non-`--json` prints `"N item(s)"`/`"[]"`
  (`write_read_array`, `commands.rs:503-507`), not the blocks.
- **`done`/`skip`/`rm` have no `--date`** (`cli.rs:28-30`; `commands.rs` hardcodes `today`).
- **Store unit test writes into CWD.** `atomic_write_replaces_existing_file` (`store.rs:546-555`)
  uses a relative path instead of `assert_fs::TempDir` — pollutes CWD, fragile under parallel runs.
- **`is_lock_contention` maps `PermissionDenied` → `Locked`** (`store.rs:329`) — a genuinely
  unreadable lockfile is misreported (and it's untested, see M3).
- **`status` can overcount triggers** — `triggers.json` isn't updated when a trigger self-fires/
  auto-cleans on the OS side, only on `apply`/`clear`.

### ✅ Done well
Trigger identity `(date,idhash,rev,event)` matches spec on systemd/Windows; argv-only execution with
XML/quote escaping and charset-restricted inputs (no shell, minimal injection surface); `schedule_rev`
correctly excludes status/title/run; centralized, unit-tested exit codes; `FireCleanup` Drop
guarantees launchd bootout even on no-op/early return; ledger gives at-most-once; DST via jiff with a
documented Compatible strategy and gap/fold tests.

---

## Post-mortem — why did a TDD plan still produce these?

The bugs are not sloppiness; they are blind spots **structurally inherent to how the plan was
written**. Root causes, grouped:

### 1. The plan specified things *per-stage / per-event*, but the bugs live in the *interactions* between them.
TDD only drives what a test asserts, and tests only exist for what's specified. The plan decomposed
cleanly into stages and the `§7` event table specified each event *in isolation* — but had no
**cross-cutting system invariants**. Every MAJOR/several MINOR findings are interaction bugs that no
single stage owned:
- **M1** = Notify event × Start event interaction (each correct alone; together they double-notify).
- **M2** = Stage-3 lock × Stage-4 command sequencing (lock works; nobody said command RMW must hold it).
- **status overcount** = Stage-3 trigger ledger × Stage-5 OS self-cleanup (never reconciled).
- **reads-mutate** = reconcile-on-query × read-purity (read/write contract never stated).

> **Plan gap:** there is no "System Invariants" layer above the stages. `Inv-1…Inv-15` are mostly
> single-object properties; none says "≤1 notification per event," "concurrent additive edits
> preserve all blocks," "reads never mutate persisted state," or "the trigger ledger converges with
> OS state." No invariant → no property test → no red bar → bug ships green.

### 2. Several rules were stated as prose but never made into a *gate*. A rule with no gate is a suggestion.
The DoD mechanically checks fmt / clippy / test-count / coverage-number / deny — nothing else.
- **Coverage honesty** ("never exclude business logic") is prose; the coverage *number* is trivially
  gamed by a module-level `coverage(off)`, so M3 sailed through a green 100%.
- **"Tests must use `assert_fs::TempDir`, never touch real paths"** is prose; the CWD-writing test
  (store.rs) violated it and self-review missed it.
- **"Human output scannable"** is prose; only the JSON path got tests, so the human path rotted to a
  placeholder.

> **Plan gap:** policies without an automated check rely on self-review vigilance, which is exactly
> what fails under compaction/low-power agents — the audience this checklist was written for.

### 3. The plan contained a direct contradiction, and the *local* instruction beat the *global* principle.
The coverage-honesty rule says "never exclude business logic," but **Stage 5 literally instructs**
"each backend module is `#[cfg_attr(coverage_nightly, coverage(off))]`." Faced with a specific local
instruction vs. a general principle, the implementer followed the local one — producing M3. The plan
caused the very violation its own rule forbids.

### 4. Spec details were split between DESIGN and the checklist, and "recon-read it" is lossy — only a test pins a detail.
Exact CLI flags (DESIGN §8) and exact backend-identity grammar (DESIGN §6.1) lived in DESIGN; the
checklist only *referenced* them ("the exact flags", recon focus) with no **conformance test**
binding implementation to spec. So the launchd dotted-label and the missing `--date` flags drifted,
and the one identity test (`trigger_identity`) actually *baked in* the hyphenated deviation instead
of catching it. Recon reading carries a detail into the agent's head for one stage; a test carries it
forever.

### 5. The plan assumed cross-platform parity the platforms don't provide, on platforms it couldn't verify.
The dev box is Linux; macOS/Windows backends were written blind, and Stage 5's "verify on the actual
OS" was only satisfiable for Linux. DESIGN asserted uniform "1s accuracy" without per-platform
caveats, so the launchd minute-granularity / annual-recurrence facts were *structurally
undetectable* by the available test loop.

### What to change in the plan (so this class of bug can't recur)
1. **Add a "System Invariants" section** (cross-stage) with a **mandatory property/integration test
   per invariant**, including: ≤1 notification per event; concurrent additive edits preserve all
   blocks; reads are side-effect-free; trigger-ledger ↔ OS-state convergence.
2. **Make `add`/`edit`/`rm`/`done`/`skip` locked read-modify-write transactions** via a
   `Store::update(date, closure)` API; forbid unlocked load-then-`set` in the command layer.
3. **Resolve the coverage contradiction:** replace "each backend module is `coverage(off)`" with
   "only `Command`-executing methods are `coverage(off)`; pure helpers go in a coverage-on submodule
   with tests." **Add a DoD enforcement step** that fails on any *module-scope* `coverage(off)`.
4. **Add CLI + backend conformance tests:** snapshot every command's `--help` and assert flag sets
   against DESIGN §8; assert each platform's identity string against the exact DESIGN §6.1 grammar.
5. **Add a human-output snapshot test** for `now`/`next`/`agenda`.
6. **Add a test-hygiene gate:** grep tests for `env::temp_dir()` / relative-path writes and fail.
7. **Specify the default `notify` lead** explicitly in DESIGN/model; add per-platform scheduler
   caveats to DESIGN §6.1 (launchd minute granularity + self-bootout dependency).
8. **State the read/write contract:** reconcile-on-query is **in-memory only**; persistence happens
   only on `apply` or an explicit mutation command.
