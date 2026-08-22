# mkdir.M4-001 — implementation notes

**Issue:** #10 — single + multi-level test.
**Upstream doc:** `design/tooling/r49-r50-plan.md` §5.9 (paideia-os).

## What landed

- `tests/test_m4_001_single_multi.pdx` — `MkdirTestM4001` module with
  two test-driver functions:
    - `run_test_m4_001_single : () -> u64 !{mem, sysreg} @{cap}`
    - `run_test_m4_001_multi  : () -> u64 !{mem, sysreg} @{cap}`
  Each returns 0 on PASS or 1 on FAIL. Both share five per-check FAIL
  diagnostics (`fail_rc_msg`, `fail_comp_count_msg`,
  `fail_newly_created_msg`, `fail_cdr_count_msg`, `fail_rur_count_msg`)
  and each has its own OK witness (`ok_single_msg`, `ok_multi_msg`).
- `tests/expected-mkdir-m4-001-single.txt` — `mkdir.M4-001 single OK`
- `tests/expected-mkdir-m4-001-multi.txt` — `mkdir.M4-001 multi OK`
- `tests/README.md` — replaced the M1-era placeholder with the M4 test
  matrix table + the test-driver + smoke-harness convention.
- `STATUS.md` — appended the M4-001 row (LANDED) and bumped the current
  milestone to M4 (in progress).

## Test-driver convention (introduced here, referenced by M4-002..004)

Each M4-* test lands as a `Mkdir<TestId>` module exporting a
`run_test_m4_<id> : () -> u64 !{mem, sysreg} @{cap}` function that
returns 0 on PASS or 1 on FAIL. The driver:

1. Calls `mkdir_state_reset()` + `reset()` (libpdx-argv `ParsedArgs`).
2. Seeds a bootstrap argv over its own `.rodata` strings + a
   `.bss argv_*_ptrs` array (built via three `lea+mov` stores per
   entry).
3. Calls `parse_argv → parse_flags_from_argv → mkdir_run` identically
   to `src/mkdir.pdx _start` Steps 3..7.
4. Asserts on `MkdirState` singletons via `cmp reg, imm` against
   expected values. Every assert-fail emits a per-check FAIL diagnostic
   + returns 1.
5. On all-pass, emits its OK witness line + returns 0.

Drivers **do not define their own `_start`.** A test-runner binary
lives at paideia-os `tests/mkdir/mkdir_test_runner.pdx` (M4-substrate-001
follow-up) — that binary owns `_start`, calls each `run_test_m4_*` in
sequence, sums the failures, and `sys_exit`s with the sum. Until the
runner lands, the drivers stay shape-frozen and the paideia-os smoke
matrix greps the expected-*.txt witness files directly for the OK lines.

This is the same "shape here, substrate wire in paideia-os" seam that
M2 established for `PXT_OP_CREATE` / `PFF_OP_QUERY_INODE` /
`USER_OP_QUERY_FP0` placeholders and M3-001 established for the
`CreatedDirRecord[]` staging.

## Scenario 1 — single-level (`mkdir a`)

Bootstrap argv:
- `argv_single_ptrs[0]` = `"a"` (one positional, no flags).

Flow:
1. `mkdir_one("a")` skips the multi-level guard (single-component
   path, no `'/'`).
2. `mkdir_one_no_slash` branch sets `comp_count = 1`.
3. `--dry-run` flag is 0 → NO short-circuit; proceeds to Step 5.
4. Step 5: `sys_cap_invoke(SLOT_TXN, PXT_OP_QUERY_ID)` — placeholder
   returns positive txn_id → `current_txn_id` populated.
5. Step 6 walks once: pre-existence probe placeholder (returns
   not-exists) → `PXT_OP_CREATE` placeholder → `newly_created_count = 1`
   → cap-tail stamp placeholder → CDR + RUR stage at index 0.
6. Step 7 commit placeholder → freezes
   `created_dir_records_count = rmdir_undo_records_count = 1`.
7. Returns `MK_OK`.

Post-condition asserts:
- `exit_code                    == 0`
- `comp_count                   == 1`
- `newly_created_count          == 1`
- `created_dir_records_count    == 1`
- `rmdir_undo_records_count     == 1`

Smoke witness on all-pass: `mkdir.M4-001 single OK\n`.

## Scenario 2 — multi-level (`mkdir -p a/b/c`)

Bootstrap argv:
- `argv_multi_ptrs[0]` = `"-p"` (short flag → `flag_p = 1`).
- `argv_multi_ptrs[1]` = `"a/b/c"` (positional).

The bootstrap deliberately DOES NOT include `--dry-run` (unlike
`src/mkdir.pdx`'s own `_start` bootstrap): the M4-001 multi scenario
must exercise Steps 5..7 (TXN open → walk → commit), not short-circuit
before Step 5.

Flow (with `flag_p == 1`):
1. `mkdir_one("a/b/c")` takes the `mkdir_one_split` branch.
2. `mkdir_split_path("a/b/c")` → `comp_count = 3`; component offsets
   +0, +2, +4 with lengths 1/1/1.
3. Step 5 TXN open → `current_txn_id` captured.
4. Step 6 walks THREE times: probe (all not-exists) → create (all
   succeed) → `newly_created_count = 3` → cap-tail stamp x3 → CDR
   stage x3 + RUR stage x3.
5. Step 7 commit → freezes both counts at 3.
6. Returns `MK_OK`.

Post-condition asserts:
- `exit_code                    == 0`
- `comp_count                   == 3`
- `newly_created_count          == 3`
- `created_dir_records_count    == 3`
- `rmdir_undo_records_count     == 3`

Smoke witness on all-pass: `mkdir.M4-001 multi OK\n`.

## paideia-as conformance

Follows the same rules as `src/mkdir.pdx`:

- Module name PascalCase basename (`MkdirTestM4001`).
- No `test` mnemonic — every zero-check is `cmp reg, 0`.
- Register `r11` scratch only (LEA temp); never live across calls.
- Byte loads: N/A (only u64 field reads in this module).
- Every `cmp reg, imm` uses an immediate ≤ 3 (max: `comp_count == 3`
  assertion). SC+ IDs are 12 (`sys_debug_puts` via `emit_stderr`) and
  60 (unused here — the runner owns `sys_exit`).
- No SysV push/pop parity — drivers are straight-line with only
  leaf-return calls (`mkdir_state_reset`, `reset`, `parse_argv`,
  `parse_flags_from_argv`, `mkdir_run`, `emit_stderr` all restore
  their own callee-save).
- Labels prefixed `t4s_` (test-m4-001-single) and `t4m_`
  (test-m4-001-multi). Avoids every paideia-as reserved keyword
  (`loop`, `if`, `let`, `fn`, `pub`, `mut`, `struct`, `structure`,
  `unsafe`, `block`).
- Cross-module calls by unqualified linker name per the paideia-as
  flat convention (`mkdir_state_reset` / `reset` /
  `parse_argv` / `parse_flags_from_argv` / `mkdir_run` /
  `emit_stderr`).

## What did NOT land at M4-001

- No `_start` for either driver — the test-runner binary lives in
  paideia-os at `mkdir.M4-substrate-001`. Until that lands, the two
  drivers are shape-frozen library-style functions.
- No CDR / RUR record-body assertion. M4-001 asserts only the counter
  invariants (`created_dir_records_count`, `rmdir_undo_records_count`).
  Full record-content verification (path_ptr, path_len, parent_txn_id,
  owner_slot_ref) is `mkdir.M4-004`'s scope (cap-tail correctness
  reads the owner field per record).
- No substrate-driven pre-existence discrimination. Every level in the
  multi scenario is treated as newly-created because the M2-002
  placeholder probe returns not-exists for every level. Mixed
  pre-existing + new is `mkdir.M4-002`'s scope.
- No TXN-abort assertion. The M3-001 placeholder create always
  succeeds; TXN-abort-mid-create is `mkdir.M4-003`'s scope.

## Cross-repo dependencies

- `mkdir.M4-001` direct: paideia-os kernel through R48b (KIND_USER,
  KIND_PDXFS_FILE, KIND_PDXFS_TXN, KIND_IPC_ENDPOINT); libpdx-argv
  M1-002 (`Parser::parse_argv`, `ParsedArgs::reset`); libpdx-cap
  M1-001; `src/mkdir.pdx` at M3-003 (exports every helper the drivers
  call).
- `mkdir.M4-001` shape: `mkdir.M4-substrate-001` at paideia-os (the
  `mkdir_test_runner.pdx` binary that provides `_start`, sequences
  the per-scenario drivers, and sums failures for `sys_exit`).

paideia-as ≥ v0.33 required for the module encoder (`mov_b` and
`@align`).

## Build note

No build here — main runs `bash tools/build.sh` and commits the
artefact once clean.
