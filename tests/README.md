# tests/

**Wave:** R50 coreutil
**Milestone:** M4 (tests + smoke) — IN PROGRESS.

Per `design/tooling/r49-r50-plan.md` §5.9 (paideia-os) M4 line, mkdir's
test matrix ships FOUR issues (M4-001..M4-004). Each issue lands one
test-driver `.pdx` module plus one or more `expected-*.txt` witness
files that the paideia-os smoke matrix greps for.

## Test matrix

| Issue       | Driver                                             | Witness(es)                                                     | State  |
|-------------|----------------------------------------------------|-----------------------------------------------------------------|--------|
| M4-001 (#10)| `test_m4_001_single_multi.pdx`                     | `expected-mkdir-m4-001-single.txt`, `expected-mkdir-m4-001-multi.txt` | LANDED |
| M4-002 (#11)| `test_m4_002_mixed_pre_existing.pdx`               | `expected-mkdir-m4-002-mixed.txt`                               | LANDED |
| M4-003 (#12)| `test_m4_003_txn_abort.pdx`                        | `expected-mkdir-m4-003-abort.txt`                               | LANDED |
| M4-004 (#13)| `test_m4_004_cap_tail.pdx`                         | `expected-mkdir-m4-004-owner.txt`                               | LANDED |
| ENH-012 (#27)| `test_enh_multi_positional.pdx`, `test_enh_record_fields.pdx`, `test_enh_path_guards.pdx` | `expected-mkdir-enh-multi-positional.txt`, `expected-mkdir-enh-fields-single.txt`, `expected-mkdir-enh-fields-multi.txt`, `expected-mkdir-enh-guards.txt` | LANDED |

## ENH-012 — the gap the M4 matrix left open

Every M4-* driver above invokes `mkdir_run` with exactly ONE positional
(`argc` of 1, or 2 for `-p` plus one path), and asserts record COUNTS
(`created_dir_records_count`, `rmdir_undo_records_count`) but never a
record FIELD. That gap is exactly why the mkdir.ENH-004 (`..`
containment escape), mkdir.ENH-005 (`path_len == 0` on the non-`-p`
path), and mkdir.ENH-006 (TXN double-commit + record clobber across
positionals) defects all shipped inside a green M4 matrix. `mkdir.ENH-012`
adds the three drivers a green M4 was missing:

- `test_enh_multi_positional.pdx` — `mkdir a b c` (three positionals,
  the first invocation shape in this repo's matrix with more than
  one). Asserts one shared TXN (`parent_txn_id` equal across all three
  `CreatedDirRecord` entries) and per-positional `path_len` fields —
  the mkdir.ENH-006 regression test.
- `test_enh_record_fields.pdx` — two scenarios: `mkdir a` (asserts
  `created_dir_records[0].path_len == 1`, not 0 — the mkdir.ENH-005
  regression test) and `mkdir -p a/b/c` (asserts the per-level
  cumulative `path_len` values 1/3/5 against one shared `path_ptr`).
- `test_enh_path_guards.pdx` — `mkdir -p ../x` and
  `mkdir -p a/../../x`, both asserting `MK_PARENT_REF_UNSUPPORTED`
  (10) and `newly_created_count == 0` — the mkdir.ENH-004 regression
  test.

## Test-driver convention (introduced at M4-001)

Each M4-* test lands as a `Mkdir<TestId>` module exporting one or more
`run_test_m4_<id>_<scenario> : () -> u64 !{mem, sysreg} @{cap}`
functions that return 0 on PASS or 1 on FAIL. The driver:

1. Calls `mkdir_state_reset()` + `reset()` (libpdx-argv `ParsedArgs`).
2. Seeds a bootstrap argv over its own `.rodata` strings + a
   `.bss argv_*_ptrs` array (built via `lea+mov` stores per entry).
3. Calls `parse_argv → parse_flags_from_argv → mkdir_run` identically
   to `src/mkdir.pdx _start` Steps 3..7. Some drivers (M4-003) invoke
   `mkdir_one` directly to exercise a specific failure branch.
4. Asserts on `MkdirState` singletons via `cmp reg, imm` against
   expected values. Every assert-fail emits a per-check FAIL
   diagnostic + returns 1.
5. On all-pass, emits its OK witness line + returns 0.

Drivers **do not define their own `_start`.** A test-runner binary
lives at paideia-os `tests/mkdir/mkdir_test_runner.pdx`
(`mkdir.M4-substrate-001` follow-up) — that binary owns `_start`,
calls each `run_test_m4_*` in sequence, sums the failures, and
`sys_exit`s with the sum.

Until the runner lands, the drivers stay shape-frozen and the
paideia-os smoke matrix greps the expected-*.txt witness files
directly for the OK lines (same seam paideia-os
`tests/r31/expected-spawn-pair.txt` uses at HEAD).

## Legacy M1 witness

The M1 first-runnable smoke witness is the `mkdir: M1 ring3 ok\n` line
that `_start` emits before its `sys_exit` — same proof-of-life shape as
`R31 ECHO CLIENT RING3 OK` in `src/user/echo_client.pdx` (paideia-os).
Its acceptance ships as part of the paideia-os smoke matrix, not from
this tests/ tree.
