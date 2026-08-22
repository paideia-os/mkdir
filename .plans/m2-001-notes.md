# mkdir.M2-001 — implementation notes

**Issue:** #4 — `-p` create-parents: multi-level path atomic in single TXN.
**Upstream doc:** `design/tooling/r49-r50-plan.md` §5.9 (paideia-os).

## What landed

- `src/mkdir_state.pdx`:
  - Two new return codes: `MK_ABS_PATH_UNSUPPORTED = 7` and
    `MK_PATH_TOO_DEEP = 8`.
  - New bound `PATH_MAX_COMPONENTS = 16` (deep enough for every
    realistic paideia-os path; deepest at R48 is
    `/system/audit/user-events/pkg-<ts>-<id>.pdxevent` at 4 levels).
  - Five new `.bss` slots: `comp_start_offsets[16]`,
    `comp_lengths[16]`, `comp_count`, `newly_created_count`,
    `walk_i`. The two arrays are consumed by index up to
    `comp_count`, so only the counters are zeroed at
    `mkdir_state_reset`.
  - `mkdir_state_reset` extended to zero the three new counters.
- `src/mkdir.pdx`:
  - Two new stderr diagnostics: `err_abs_path_msg`,
    `err_too_deep_msg`.
  - New leaf helper `mkdir_split_path(path) -> rc` — single-pass
    byte walk that populates `comp_start_offsets`, `comp_lengths`,
    `comp_count`. Rejects absolute paths at byte 0
    (`MK_ABS_PATH_UNSUPPORTED`); rejects > 16 non-empty levels
    (`MK_PATH_TOO_DEEP`); silently skips empty components
    (`foo//bar` → 2 comps; `foo/` → 1 comp).
  - `mkdir_one` reworked around the `-p` branch:
    - Step 1 (multi-level `/` guard) now gated on `flag_p == 0`.
      Under `-p` the guard is bypassed and `mkdir_split_path`
      handles decomposition.
    - Non-`-p` branch sets `comp_count = 1` so the walk-loop
      shape is unified.
    - `--dry-run` short-circuit moved AFTER split validation, so
      `mkdir -p --dry-run /abs/path` still surfaces
      `MK_ABS_PATH_UNSUPPORTED` (POSIX validate-not-execute).
    - New Step 6 replaces the M1 single create call with a
      `walk_i`-driven loop over `comp_count`. `walk_i` lives in
      `.bss` so it survives the placeholder `sys_cap_invoke`
      caller-save clobber without a callee-save push (same idiom
      `mkdir_run_i` uses for the outer positional walk).
    - Each iteration increments `newly_created_count` after a
      successful placeholder create. M3-003's undo record reads
      this counter to unwind exactly the levels this invocation
      created (a pre-existing top level under M2-002 will not
      advance the counter so its removal is not queued for undo).
    - `push rbx` / `pop rbx` parity preserved: exactly one push in
      the prologue, exactly one `ret` in the epilogue.
  - Bootstrap argv upgraded from `-v --dry-run a` (3 args) to
    `-p -v --dry-run a/b/c` (4 args). New `.rodata` string
    `argv_flag_p` and `argv_positional_abc`; `argv_bootstrap_ptrs`
    grows to `[u64; 4]` and `argv_bootstrap_argc = 4`. The M2
    smoke witness exercises the `-p` split path end-to-end
    (`comp_count = 3` for a, b, c) then short-circuits on
    `--dry-run` for a clean `exit(0)`.
- `design/architecture.md`:
  - §4 title clarified as "M1 baseline + M2-001 `-p` walk".
  - New §4a documents `mkdir_split_path`.
  - New §4b documents the three `mkdir_one` M2-001 additions.
  - §5 flag-semantics table: `-p` row marked LANDED at M2-001;
    M2-002 and M2-003 pointers moved to the wire-up column.
  - §8 renamed to "What M2-001 explicitly does not do"; M1's
    "-p behaviour missing" bullet removed since it is now
    landed.
- `STATUS.md` — M2-001 row added, marked LANDED.
- `.plans/m2-001-notes.md` (this file).

## Smoke witness at M2-001

The M2 bootstrap argv `-p -v --dry-run a/b/c` produces:

```
parse_argv → OK; flag_count=3 (p, v, dry-run); pos_count=1 (a/b/c)
parse_flags_from_argv → flag_p=1; flag_v=1; flag_dry_run=1
mkdir_run
  i=0: mkdir_one("a/b/c")
    Step 1  flag_p == 1                        → skip multi-level guard
    Step 2  mkdir_split_path("a/b/c")          → comp_count=3
              comp_start_offsets = [0, 2, 4]
              comp_lengths       = [1, 1, 1]
              rax = MK_OK
    Step 4  flag_dry_run == 1                  → return MK_OK (0)
  exit_code stays 0
sys_exit(0)                                    → MK_OK smoke
```

The `mkdir: M1 ring3 ok\n` marker still prints from Step 0 of
`_start` before any of the above. Together the marker and the exit
code bracket the M2-001 flow: marker proves ring3 entry, exit 0
proves the whole `-p` split path parses + validates + short-circuits
cleanly without touching the still-being-built kernel-side
create-dir op.

## Design decisions

- **`-p` guard bypass at Step 1, not at Step 2.** The M1 multi-level
  guard was structural — it decided which return code fires when a
  path contains `/`. Under `-p`, the intent is "multi-level is OK";
  the split then owns the path validation. Keeping the guard gated
  on `flag_p == 0` preserves POSIX-canonical `mkdir foo/bar` error
  behaviour (which every muscle-memory user expects to fail) while
  the `-p` path opts into the split.
- **`comp_count = 1` on the non-`-p` branch.** Alternative was to
  keep two code paths (single-component + multi-component). The
  unified walk-loop is simpler: one loop, one placeholder-create
  call site, one `newly_created_count` increment site. When M2-002
  and M2-003 land, they slot in as an extra branch and an extra
  syscall inside the loop rather than two mirrored patches to two
  branches.
- **`--dry-run` short-circuit AFTER split.** POSIX `mkdir --dry-run`
  (per the R49 D3 flag grammar) validates the arguments without
  executing them. Putting the short-circuit before the split would
  hide `MK_ABS_PATH_UNSUPPORTED` from the caller — a UX regression
  because `--dry-run` is exactly when the caller wants to know
  about validation errors. The trade-off: dry-run still walks the
  split (linear in path length), which is fine — split has no
  syscalls and no cap invocations.
- **Empty-path short-circuit to MK_OK.** POSIX-ish `mkdir -p ""` is
  idempotent (creates zero directories). `mkdir_split_path` returns
  `MK_OK` with `comp_count = 0` for an empty path, and `mkdir_one`
  short-circuits before the TXN open. This avoids opening a TXN
  scope just to close it empty.
- **`walk_i` in `.bss`, matching `mkdir_run_i`.** The alternative
  was a callee-save push in `mkdir_one`'s prologue (`push rbp;
  mov rbp, walk_i_reg`). The `.bss` path avoids the push and
  matches the `mkdir_run_i` precedent. Safe because `mkdir_one`
  runs at most once per positional (`mkdir_run` calls it in
  sequence, never recursively).
- **`newly_created_count` in `.bss`, not a per-invocation stack
  slot.** M3-003's undo record reads this after `mkdir_one`
  returns; keeping it in `.bss` means the undo journal writer
  (added later at M3-003) can read it directly rather than
  threading it back through `mkdir_run`'s frame.
- **Placeholder create per level.** The kernel-side create-dir op
  is still being built (`mkdir.M2-substrate-001` at paideia-os).
  M2-001 issues `PFF_OP_DEBUG_PRINT` per component — this
  exercises the KIND_PDXFS_FILE dispatch path so a mis-seeded
  slot 1 surfaces at M2 rather than at the substrate landing.
  The `newly_created_count` increment is unconditional at M2-001
  because the placeholder never fails cleanly; M2-002 makes the
  increment conditional on "not pre-existing" — same call site,
  one extra branch.
- **`PATH_MAX_COMPONENTS = 16` bound checked at every close, not at
  end.** Alternative was to walk the whole path unbounded and
  refuse only if the final count exceeded 16. Checking at every
  close means over-limit paths surface at the first over-limit
  level without wasting cycles on the tail — matches the "fail
  fast" idiom `libpdx-argv` uses for its `MAX_FLAGS` overflow
  check.

## paideia-as conformance

- No `test` mnemonic anywhere in the new code.
- Every `cmp reg, imm` uses an immediate ≤ 0x7FFFFFFF (max seen at
  M2-001: `0x2F` for the `/` byte, `16` for `PATH_MAX_COMPONENTS`,
  `7`/`8` for MK_* value routing).
- Every byte load in `mkdir_split_path` uses `xor rax, rax;
  mov_b rax, [ptr]` per the paideia-as #1248 mitigation. Instances:
  the absolute-path prelude and the `msp_loop` body.
- `r11` used only as scratch (LEA temps for `.bss` slot addresses).
  Never live across a call.
- `mkdir_split_path` push/pop parity: zero pushes (leaf function).
- `mkdir_one` push/pop parity unchanged from M1: one `push rbx` in
  the prologue matched by one `pop rbx` at the single epilogue
  label. All new error paths (`mkdir_one_abs_path_fail`,
  `mkdir_one_too_deep_fail`) route through `mkdir_one_epilogue` —
  no early `ret` bypasses the pop.
- Labels avoid every paideia-as reserved keyword. New prefixes:
  `msp_` in `mkdir_split_path`; extended `mkdir_one_` prefix in
  `mkdir_one` for new labels (`mkdir_one_split`,
  `mkdir_one_after_split`, `mkdir_one_no_slash`,
  `mkdir_one_abs_path_fail`, `mkdir_one_too_deep_fail`,
  `mkdir_one_walk_head`, `mkdir_one_do_commit`, `mkdir_one_ok`).
- SysV alignment: `mkdir_one`'s `call mkdir_split_path` inherits the
  same aligned-rsp discipline as `call emit_stderr` (one prior push
  aligns rsp for every nested call). `mkdir_split_path` is a leaf
  so its rsp alignment is not consulted.

## Cross-module linkage

New unqualified linker-name references introduced by M2-001:
- `mkdir_split_path` → local (called from `mkdir_one`).
- `comp_start_offsets`, `comp_lengths`, `comp_count`,
  `newly_created_count`, `walk_i` → local `MkdirState` (`.bss`
  reads/writes).
- `argv_flag_p`, `argv_positional_abc` → local (`.rodata` reads
  from `_start`).

## What did not land (queued for M2-002/003 and beyond)

- Pre-existing-dir handling — `mkdir.M2-002` (#5). Adds the
  existence probe branch inside the walk loop; `newly_created_count`
  increment becomes conditional.
- Cap-tail owner write — `mkdir.M2-003` (#6). Adds a placeholder
  `sys_cap_invoke(SLOT_USER, USER_OP_QUERY_FP0)` after each
  successful create; `MK_CAP_TAIL_FAIL` return code added.
- Kernel-side real ops (open-TXN, create-dir, commit-TXN) —
  `mkdir.M2-substrate-001` at paideia-os.
- Dot-elimination (`a/./b`) — deferred to `mkdir.M2-substrate-001`
  (proper semantic normalisation belongs at the kernel-side
  create-dir op, not the userspace splitter).
- Real per-component existence probe — `mkdir.M2-substrate-002` at
  paideia-os. M2-002 lands the library-side branch; substrate wire
  fills the probe's return convention.
- `CreatedDirRecord[]` schema bind — `mkdir.M3-001`.
- `CreateDirRecord` via libpdx-audit — `mkdir.M3-002`.
- PdxFS v1 undo record — `mkdir.M3-003`.
- Coreutil test matrix — `mkdir.M4-001` through `mkdir.M4-004`.

## Build note

Same as M1-003: mkdir has no local build script. Build is main-only
per the no-background-builds memory note; this issue lands the
source and defers the build check to main's synchronous
`bash tools/build.sh` step alongside the libpdx-argv link.
