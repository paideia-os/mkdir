# mkdir.M1-003 — implementation notes

**Issue:** #3 — first runnable: single-level `mkdir a` in TXN with
cap-tail as owner.
**Upstream doc:** `design/tooling/r49-r50-plan.md` §5.9 (paideia-os).

## What landed

- `src/mkdir_state.pdx`: added `mkdir_run_i : u64` singleton (loop
  counter storage that survives `mkdir_one` calls without a callee-save
  push). Extended `mkdir_state_reset` to zero the new slot.
- `src/mkdir.pdx`:
  - Four new stderr strings: `err_multi_msg` (38 B; multi-level path
    rejection), `err_txn_open_msg` (23 B), `err_mkdir_msg` (21 B),
    `err_commit_msg` (25 B).
  - `mkdir_one(path)` — single-level create sequence per
    `design/architecture.md` §4:
    1. Multi-level guard (any `/` → `MK_MULTI_LEVEL_UNSUPPORTED`).
    2. `-v` emission deferred (see M1-003 deferral note in doc §4/§5).
    3. `--dry-run` short-circuit (`MK_OK` without touching either cap).
    4. Placeholder `sys_cap_invoke(SLOT_TXN, PXT_OP_QUERY_ID)` — TXN
       open. Negative rax → `MK_TXN_OPEN_FAIL`.
    5. Placeholder `sys_cap_invoke(SLOT_PARENT_FILE, PFF_OP_DEBUG_PRINT)`
       — create-dir. Negative rax → `MK_MKDIR_FAIL`.
    6. Cap-tail owner write deferred to `mkdir.M2-003`.
    7. Placeholder `sys_cap_invoke(SLOT_TXN, PXT_OP_QUERY_STATE)` — TXN
       commit. Negative rax → `MK_TXN_COMMIT_FAIL`.
    SysV push/pop parity: one `push rbx` at prologue (aligns rsp to 16
    for nested `call`s AND preserves `path` across every
    `emit_stderr` / `syscall`); balanced by one `pop rbx` at the
    single epilogue label.
  - `mkdir_run()` — walks `pos_ptrs[0..pos_count]`, calls `mkdir_one`
    per positional. Loop counter lives in `MkdirState::mkdir_run_i`
    (survives `mkdir_one` clobber). Fail-fast: first non-`MK_OK`
    return records `exit_code + err_pos_index` and stops. No partial
    completion at M1.
  - `_start` Step 7 upgraded: `call mkdir_run` then
    `sys_exit(MkdirState::exit_code)`.
- `design/architecture.md`: §4 step 1 tightened (M1 rejects multi-level
  always, regardless of `-p`); §4 step 2 clarified (`-v` emission is
  M3-001, not M1-003).
- STATUS.md — M1-003 rollup bumped to LANDED.
- `.plans/m1-003-notes.md` (this file).

## Smoke witness at M1

The bootstrap argv `-v --dry-run a` produces:

```
parse_argv → OK; flag_count=2 (v, dry-run); pos_count=1 (a)
parse_flags_from_argv → flag_v=1; flag_dry_run=1; flag_p=0
mkdir_run
  i=0: mkdir_one("a")
    Step 1  path has no '/'                    → mkdir_one_scan_done
    Step 2  -v emission deferred (M3-001)      → no op
    Step 3  flag_dry_run == 1                  → return MK_OK (0)
  exit_code stays 0
sys_exit(0)                                    → MK_OK smoke
```

The `mkdir: M1 ring3 ok\n` marker prints from Step 0 of `_start`
before any of the above. Together the two witnesses — the marker and
the exit code — bracket the whole M1 flow: marker proves the task
reached its entry, exit 0 proves argv + flag walk + mkdir_run +
mkdir_one all worked end to end without touching the still-being-built
kernel-side pdxfs create-dir op.

## Design decisions

- **Multi-level always rejected at M1, even with `-p`.** The `-p`
  recognition path lives in `parse_flags_from_argv` (M1-002) but
  `mkdir_one`'s multi-level guard runs before the dry-run short-circuit
  and rejects any `/` unconditionally. This keeps M1 strictly aligned
  with the "single-level only" line in `design/architecture.md` §4 and
  puts the multi-level TXN atomicity entirely in `mkdir.M2-001`'s
  hands.
- **`-v` emission deferred.** Full verbose emission needs a
  `strlen` helper (`sys_debug_puts` wants a byte count, but paths are
  NUL-terminated with unknown length). Rather than add a `strlen`
  helper only to delete it at M3-001 (when semantic-pipe
  `send_record` carries a typed record instead of a raw byte string),
  M1-003 leaves the emit site empty. `flag_v` is still stored and
  honoured at read; the M4-001 test matrix confirms that.
- **`--dry-run` short-circuit as the M1 smoke path.** The placeholder
  `sys_cap_invoke` calls (Steps 4/5/7) go through the kernel's cap
  dispatch and would return negative errno at M1 (no shell exists yet
  to hand mkdir a real `KIND_PDXFS_TXN` cap). Using `--dry-run` in the
  bootstrap avoids that failure and gets us a green exit-0 smoke while
  still landing every line of the cap-invoke code path for M2 to
  activate.
- **`.bss` loop counter over callee-save push.** `mkdir_run` uses
  `MkdirState::mkdir_run_i` as its loop counter. Alternative was
  `push rbx` in `mkdir_run`'s prologue + counter in `rbx`. The `.bss`
  choice avoids the push and matches the `echo_idx` pattern in
  `src/user/dispatch.pdx` (paideia-os). Safe at M1 because `mkdir_run`
  runs at most once per `_start` invocation (no re-entrance possible).
- **`push rbx` in `mkdir_one` even though the counter is in `.bss`.**
  `mkdir_one` needs `path` to survive multiple `call emit_stderr` /
  `syscall` invocations. Options were: (a) a per-call `.bss` slot; (b)
  `push rbx` + `mov rbx, rdi`. Chose (b) because it also aligns `rsp`
  to 16 for the nested `call emit_stderr` (SysV requires 16-alignment
  at every `call` site). A `.bss` slot would still need an
  `sub rsp, 8` for alignment and would prevent `mkdir_one` from ever
  being called recursively (an M2 concern under `-p`).
- **Placeholder cap-invoke pattern from libpdx-cap M1.** `mkdir_one`'s
  Steps 4/5/7 use `PXT_OP_QUERY_ID` / `PFF_OP_DEBUG_PRINT` /
  `PXT_OP_QUERY_STATE` respectively as stand-ins for the real
  "open-TXN" / "create-dir" / "commit-TXN" ops that the kernel does
  not yet expose. The signature and call sites are frozen at M1 so
  `mkdir.M2` becomes a body edit rather than a signature change —
  exact analog of `cap_manifest_verify`'s M1 skeleton in
  `libpdx-cap/src/cap.pdx`. Filed for the kernel-side work as
  `mkdir.M2-substrate-001` at the paideia-os repo when M1 closes.

## paideia-as conformance

- No `test` mnemonic anywhere. Every zero-check is `cmp reg, 0` or
  `cmp reg, imm`.
- Every `cmp reg, imm` uses an immediate ≤ 0x7FFFFFFF (max seen at
  M1-003: `0x2F` — the `/` byte for the multi-level guard).
- Every byte load uses `xor rax, rax; mov_b rax, [ptr]` per the
  paideia-as #1248 mitigation. Instances: `mkdir_one_scan_head` (path
  walk in Step 1).
- `r11` used only as scratch (LEA temps). Never live across a call.
  `mkdir_run` reloads r11 at every `.bss` slot access rather than
  assuming a stale value survives `mkdir_one`.
- `mkdir_one` push/pop parity: one `push rbx` in the prologue matched
  by one `pop rbx` at the single epilogue label. Every `MK_*` return
  path jumps to `mkdir_one_epilogue` — no early `ret` bypasses the
  `pop`, verified by grep for `ret` in the function body (exactly one
  `ret`, after the `pop`).
- `mkdir_run` push/pop parity: zero pushes (loop counter in `.bss`).
- Labels avoid every paideia-as reserved keyword. Prefixes:
  `mkdir_one_` in `mkdir_one`, `mkdir_run_` in `mkdir_run`,
  `mkdir_pff_` in `parse_flags_from_argv` (unchanged from M1-002),
  `mkdir_` in `_start` (unchanged from M1-002).
- SysV alignment: `mkdir_one`'s single `push rbx` puts `rsp` at
  16-aligned + 0 (call pushed 8 → +16 with the push = aligned) —
  correct for nested `call emit_stderr`. `mkdir_run`'s zero pushes
  leave `rsp` at 16-aligned + 8 after the caller's `call` — the
  nested `call mkdir_one` re-enters with the same alignment
  `mkdir_one` expects (SysV standard).

## Cross-module linkage

New unqualified linker-name references:
- `mkdir_one` → local (called from `mkdir_run`).
- `mkdir_run` → local (called from `_start`).
- `pos_ptrs`, `pos_count` → libpdx-argv `ParsedArgs` (`.bss` reads).
- `exit_code`, `err_pos_index`, `mkdir_run_i`, `flag_dry_run` →
  local `MkdirState` (`.bss` reads / writes).

## What did not land (queued for M2 and beyond)

- Multi-level `-p` TXN atomic create — `mkdir.M2-001`.
- Pre-existing-dir idempotence under `-p` — `mkdir.M2-002`.
- Cap-tail owner write (KIND_USER_ref stamped into inode) —
  `mkdir.M2-003`.
- `CreatedDirRecord[]` schema bind + emit on `-v` — `mkdir.M3-001`.
- `CreateDirRecord` via libpdx-audit — `mkdir.M3-002`.
- PdxFS v1 undo record + `undo mkdir` — `mkdir.M3-003`.
- Kernel-side real ops (open-TXN, create-dir, commit-TXN) —
  `mkdir.M2-substrate-001` at paideia-os.
- Coreutil test matrix (single-level, multi-level -p, mixed, TXN-abort,
  cap-tail correctness) — `mkdir.M4-001` through `mkdir.M4-004`.

## Build note

Same as M1-001 / M1-002: mkdir has no local build script yet.
paideia-as ≥ v0.33 will build both modules once main invokes
`paideia-as build src/mkdir_state.pdx src/mkdir.pdx -o build/mkdir.elf`
alongside a libpdx-argv link. Build is main-only per the
no-background-builds memory note; this issue lands the source and
defers the build check to main's synchronous `bash tools/build.sh`
step.
