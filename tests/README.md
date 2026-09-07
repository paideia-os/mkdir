# tests/

**Wave:** R50 coreutil → v1.1-A extraction
**Milestone:** v1.1-A tests — DEFERRED.

## v1.1-A test state

The M1..M5 test drivers (`test_m4_00[1-4]_*.pdx`,
`test_enh_*.pdx`) and their `expected-*.txt` witness files were all
retired at v1.1-A. Every one of them asserted against the STUB
scaffolding's `MkdirState` singletons (`comp_count`,
`newly_created_count`, `created_dir_records`, `rmdir_undo_records`,
`current_txn_id`) and the shape of the placeholder `sys_cap_invoke`
dispatch — v1.1-A retires all of that, so those tests can no longer
even build against `src/`.

Concretely deleted at v1.1-A:

- `test_m4_001_single_multi.pdx` + `expected-mkdir-m4-001-{single,multi}.txt`
- `test_m4_002_mixed_pre_existing.pdx` + `expected-mkdir-m4-002-mixed.txt`
- `test_m4_003_txn_abort.pdx` + `expected-mkdir-m4-003-abort.txt`
- `test_m4_004_cap_tail.pdx` + `expected-mkdir-m4-004-owner.txt`
- `test_enh_multi_positional.pdx` + `expected-mkdir-enh-multi-positional.txt`
- `test_enh_record_fields.pdx` + `expected-mkdir-enh-{fields-single,fields-multi}.txt`
- `test_enh_path_guards.pdx` + `expected-mkdir-enh-guards.txt`

## v1.1-A test plan (open)

A v1.1-A test driver needs a different shape from M4: v1.1-A carries
no `.bss` state to assert against and does not vend a
`mkdir_run`-style entry point that a driver can call in-process.
Every assertion has to be observed from the outside — filesystem-side
via `sys_stat` after the tool exits, and exit-code side via the
paideia-os smoke matrix's `$?` render.

Two smoke witnesses are wanted (both filed as follow-up issues, not
this milestone):

1. **v1.1-A-T1** — `mkdir /tmp/mkdir-v1.1-A-t1`; assert `sys_stat`
   sees the directory afterwards and the tool's exit status is 0.
2. **v1.1-A-T2** — `mkdir` with zero positionals; assert the
   `mkdir: missing operand` line reaches the debug channel and the
   tool's exit status is 2.

The `-errno` first-failure path (v1.1-A-T3) waits on a repeatable
`sys_mkdir` failure fixture — a collision on an existing entry
returns `-EIO` per `sys_mkdir.pdx §Failure taxonomy`, which is the
simplest to script once T1's fixture is in place.

These belong in the paideia-os smoke matrix (external to this repo)
rather than the M4-era `run_test_m4_*` in-process driver shape;
`src/mkdir.pdx` no longer exposes any function other than `_start`
for a driver to call into.
