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
